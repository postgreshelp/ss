# SOP-07: Logical replication end to end, with limitations, workarounds, and a major-version upgrade cutover

| Item | Value |
|---|---|
| Purpose | Build, operate, troubleshoot and finally **use logical replication to upgrade** PostgreSQL with near-zero downtime |
| Worked example | `node1` = **PG17** publisher (current production), `node2` = **PG18** subscriber (future production). Database `bankdb`, schema `bank` |
| Ends with | Cutover from 17 to 18, optional reverse replication for rollback, cleanup |
| Scripts | All scripts are inline. Copy them into files named as in their headers |

Single-host lab variant: run 17 on port 5432 and 18 on port 5433 on the same machine, and replace `host=node1`/`host=node2` with `host=localhost port=5432/5433`.

---

## 1. Architecture in one picture

```
node1 (PG17, publisher)                                 node2 (PG18, subscriber)
┌─────────────────────────────┐                         ┌──────────────────────────────┐
│ tables ── WAL (wal_level=logical)                      │ apply worker ── tables       │
│                 │                                      │      ▲                       │
│   publication bank_pub (what to send)                  │ subscription bank_sub        │
│   slot bank_sub (pgoutput) ── walsender ═══TCP═══▶     │ origin pg_<oid> (where we are)│
└─────────────────────────────┘                         └──────────────────────────────┘
Initial sync: one tablesync worker per table (COPY), then streaming of changes.
```

Logical replication sends **row changes** (INSERT/UPDATE/DELETE/TRUNCATE) of published tables. It does **not** send DDL, sequence values, large objects, or materialized view contents. Section 6 shows how to handle each.

---

## 2. Configuration

### 2.1 Publisher node1 (PG17), `postgresql.conf` (needs a restart for wal_level)

```ini
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
max_slot_wal_keep_size = 50GB        # protect the disk if a subscriber stops (slot gets invalidated instead)
# PG18+ only: idle_replication_slot_timeout = '2d'   # auto-invalidate abandoned slots
```

`pg_hba.conf`: logical replication connects to a **database**, so the entry uses the database name, not the `replication` keyword:

```
host  bankdb  repuser  10.0.0.12/32  scram-sha-256
```

### 2.2 Subscriber node2 (PG18)

```ini
max_active_replication_origins = 10      # PG18 (PG17: governed by max_replication_slots)
max_logical_replication_workers = 8
max_sync_workers_per_subscription = 4    # parallel initial table copies
max_parallel_apply_workers_per_subscription = 2
max_worker_processes = 16
```

### 2.3 Replication user (node1)

```sql
CREATE ROLE repuser WITH LOGIN REPLICATION PASSWORD 'Rep#2026';
GRANT CONNECT ON DATABASE bankdb TO repuser;
GRANT USAGE ON SCHEMA bank TO repuser;
GRANT SELECT ON ALL TABLES IN SCHEMA bank TO repuser;              -- needed for the initial COPY
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA bank GRANT SELECT ON TABLES TO repuser;
```

`ALTER DEFAULT PRIVILEGES` only covers tables later created **by the role named in `FOR ROLE`** (default: the role running the command). Run it for every role that creates tables in `bank`. A new table that `repuser` can't read breaks replication (see 6.1).

On node2, create the subscription as a superuser, or as a role with `pg_create_subscription` (PG16+) that owns the target tables.

---

## 3. Sample data (node1)

```sql
-- 03_sample_schema.sql   (run on node1 as the owner)
CREATE DATABASE bankdb;
\c bankdb
CREATE SCHEMA bank;

CREATE TABLE bank.customers (
  customer_id  bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  full_name    text NOT NULL,
  email        text UNIQUE,
  created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE bank.accounts (
  account_id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id  bigint NOT NULL REFERENCES bank.customers,
  acct_type    text NOT NULL CHECK (acct_type IN ('SAVINGS','CURRENT')),
  balance      numeric(14,2) NOT NULL DEFAULT 0,
  updated_at   timestamptz NOT NULL DEFAULT now()
);

-- partitioned table
CREATE TABLE bank.transactions (
  txn_id      bigint GENERATED ALWAYS AS IDENTITY,
  account_id  bigint NOT NULL,
  amount      numeric(14,2) NOT NULL,
  txn_time    timestamptz NOT NULL DEFAULT now(),
  note        text,
  PRIMARY KEY (txn_id, txn_time)
) PARTITION BY RANGE (txn_time);
CREATE TABLE bank.transactions_2026_09 PARTITION OF bank.transactions FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');
CREATE TABLE bank.transactions_2026_10 PARTITION OF bank.transactions FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');
CREATE TABLE bank.transactions_2026_11 PARTITION OF bank.transactions FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');

-- LIMITATION demo 1: no primary key
CREATE TABLE bank.audit_log (event_time timestamptz DEFAULT now(), username text, action text);

-- LIMITATION demo 2: a plain sequence used by the application
CREATE SEQUENCE bank.invoice_no_seq START 100000;
CREATE TABLE bank.invoices (
  invoice_no bigint PRIMARY KEY DEFAULT nextval('bank.invoice_no_seq'),
  account_id bigint NOT NULL, total numeric(14,2) NOT NULL);

-- LIMITATION demo 3: large objects
CREATE TABLE bank.documents (doc_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, file_name text, content_oid oid);

-- LIMITATION demo 4: materialized view
CREATE MATERIALIZED VIEW bank.branch_balance AS
  SELECT acct_type, count(*) AS accounts, sum(balance) AS total FROM bank.accounts GROUP BY acct_type;

-- data
INSERT INTO bank.customers(full_name, email)
  SELECT 'Customer '||g, 'cust'||g||'@mybank.example' FROM generate_series(1,10000) g;
INSERT INTO bank.accounts(customer_id, acct_type, balance)
  SELECT (g % 10000)+1, CASE WHEN g%3=0 THEN 'CURRENT' ELSE 'SAVINGS' END, round((random()*100000)::numeric,2)
  FROM generate_series(1,20000) g;
INSERT INTO bank.transactions(account_id, amount, txn_time, note)
  SELECT (g % 20000)+1, round((random()*2000-1000)::numeric,2),
         timestamptz '2026-09-01' + (g % (60*24*80)) * interval '1 minute', 'seed'
  FROM generate_series(1,300000) g;
INSERT INTO bank.audit_log(username, action) SELECT 'app', 'login' FROM generate_series(1,1000);
INSERT INTO bank.invoices(account_id, total) SELECT (g%20000)+1, 99.00 FROM generate_series(1,5000) g;
INSERT INTO bank.documents(file_name, content_oid)
  SELECT 'statement_'||g||'.txt', lo_from_bytea(0, convert_to('Statement '||g||' for MyBank', 'UTF8')) FROM generate_series(1,100) g;
REFRESH MATERIALIZED VIEW bank.branch_balance;
ANALYZE;
```

Optional **write load** while you practise (run on node1 in a separate shell):

```bash
# 03_load.sh: one transfer per 100 ms
while true; do
  psql -X -q -d bankdb -c "WITH a AS (SELECT (random()*19999)::int+1 id)
    UPDATE bank.accounts SET balance = balance + 1, updated_at = now() WHERE account_id = (SELECT id FROM a);
    INSERT INTO bank.transactions(account_id, amount, note) VALUES ((random()*19999)::int+1, 1, 'load');
    INSERT INTO bank.invoices(account_id, total) VALUES ((random()*19999)::int+1, 10);"
  sleep 0.1
done
```

---

## 4. Build replication

### 4.1 Copy the schema (DDL isn't replicated)

```bash
# from node2
# 1. Roles first: the schema dump contains GRANTs/OWNERs that fail if the roles don't exist yet
pg_dumpall -h node1 --roles-only --no-role-passwords -f roles.sql   # drop --no-role-passwords if you are superuser on node1 and want the hashes
psql -h node2 -d postgres -f roles.sql          # "role already exists" errors for postgres etc. are harmless
# 2. Schema
pg_dump -h node1 -d bankdb --schema-only --no-publications --no-subscriptions -f bankdb_schema.sql
createdb -h node2 bankdb
psql -h node2 -d bankdb -v ON_ERROR_STOP=1 -f bankdb_schema.sql
```

Using the **PG18** `pg_dump` against the 17 server is correct: always use the newer client version.

### 4.2 Fix the tables without a primary key **before** publishing (limitation L3)

```sql
-- node1: find them
SELECT n.nspname||'.'||c.relname AS table_without_pk
FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
WHERE c.relkind IN ('r','p') AND n.nspname='bank'
  AND NOT EXISTS (SELECT 1 FROM pg_index i WHERE i.indrelid=c.oid AND i.indisprimary)
  AND c.relreplident <> 'f';
```

UPDATE/DELETE on a published table without a replica identity **fail on the publisher** (`cannot update table "audit_log" because it does not have a replica identity and publishes updates`). Choose one:

```sql
-- best: add a surrogate PK (run on BOTH nodes, subscriber first)
ALTER TABLE bank.audit_log ADD COLUMN audit_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY;
-- or: an existing unique, NOT NULL index
-- ALTER TABLE t REPLICA IDENTITY USING INDEX t_unique_idx;
-- last resort (whole old row is logged; slower on large tables):
-- ALTER TABLE bank.audit_log REPLICA IDENTITY FULL;
```

### 4.3 Publication (node1)

```sql
CREATE PUBLICATION bank_pub FOR TABLES IN SCHEMA bank        -- PG15+: whole schema, new tables auto-included
  WITH (publish = 'insert, update, delete, truncate', publish_via_partition_root = true);
SELECT * FROM pg_publication_tables WHERE pubname = 'bank_pub';
```

- `publish_via_partition_root = true`: changes are published as if they belong to `bank.transactions`, so the subscriber can partition **differently** (or not at all).
- Alternatives: `FOR TABLE a, b` (explicit list), `FOR ALL TABLES` (superuser), row filters `FOR TABLE bank.accounts WHERE (acct_type = 'SAVINGS')` and column lists `FOR TABLE bank.customers (customer_id, full_name)` (PG15+).

### 4.4 Subscription (node2)

```sql
CREATE SUBSCRIPTION bank_sub
  CONNECTION 'host=node1 port=5432 dbname=bankdb user=repuser password=Rep#2026'
  PUBLICATION bank_pub
  WITH (copy_data = true, streaming = parallel, disable_on_error = true);
```

- Creates slot `bank_sub` on node1 and starts one tablesync worker per table (up to `max_sync_workers_per_subscription` at once).
- `disable_on_error = true`: on an apply error, the subscription **disables itself** instead of retrying in a tight loop and filling the log.

### 4.5 Watch the initial sync

```sql
-- node2: i = init, d = copying data, f = finished copy, s = synchronized, r = ready (streaming)
SELECT srrelid::regclass AS tbl, srsubstate FROM pg_subscription_rel ORDER BY 2, 1;
SELECT * FROM pg_stat_progress_copy;                                    -- running COPYs
-- node1
SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
FROM pg_replication_slots;
```

Done when every row shows `r`.

### 4.6 Verify the data (script)

```bash
#!/usr/bin/env bash
# 04_compare.sh: row count + content hash per table on both nodes
# usage: bash 04_compare.sh "host=node1 dbname=bankdb" "host=node2 dbname=bankdb"
PUB="$1"; SUB="$2"
TABLES=$(psql "$PUB" -X -At -c "select format('%I.%I', schemaname, tablename) from pg_publication_tables where pubname='bank_pub'
          and tablename not like '%\_20%' order by 1")      # partitions are compared via their parent
printf '%-28s %12s %12s %s\n' TABLE PUB_ROWS SUB_ROWS RESULT
for t in $TABLES; do
  pk=$(psql "$PUB" -X -At -c "select string_agg(quote_ident(a.attname), ',') from pg_index i
        join pg_attribute a on a.attrelid=i.indrelid and a.attnum = any(i.indkey) where i.indrelid='$t'::regclass and i.indisprimary")
  q="select count(*)||' '||coalesce(md5(string_agg(x::text, '|' order by ${pk:-x::text})),'-') from $t x"
  read -r pc ph <<<"$(psql "$PUB" -X -At -c "$q")"; read -r sc sh <<<"$(psql "$SUB" -X -At -c "$q")"
  [[ "$pc" == "$sc" && "$ph" == "$sh" ]] && r=OK || r=DIFF
  printf '%-28s %12s %12s %s\n' "$t" "$pc" "$sc" "$r"
done
```

With the write load running, small DIFFs on busy tables are expected (timing). Stop the load and rerun: everything must be OK. (md5 over a whole table is fine for lab-sized data. On TB tables, compare per PK range or use sampling.)

---

## 5. Day-to-day monitoring

```sql
-- node1: per-subscriber lag
SELECT application_name, state, sent_lsn, replay_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS replay_lag,
       reply_time
FROM pg_stat_replication;

-- node1: slot health (alert if wal_status <> 'reserved' or retained WAL grows)
SELECT slot_name, active, wal_status, safe_wal_size,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;

-- node2: apply workers and errors
SELECT subname, pid, received_lsn, latest_end_lsn, last_msg_receipt_time FROM pg_stat_subscription;
SELECT * FROM pg_stat_subscription_stats;     -- apply_error_count, sync_error_count (+ conflict counters on PG18)
SELECT subname, subenabled FROM pg_subscription;   -- disabled = an error happened (disable_on_error)
```

Alert rules (suggested):

| Metric | Warn | Critical |
|---|---|---|
| `retained_wal` on node1 | > 10 GB | > 80% of `max_slot_wal_keep_size` |
| `wal_status` | `extended` | `unreserved` / `lost` |
| `subenabled = false` on node2 | any | — |
| `now() - last_msg_receipt_time` | > 1 min | > 5 min |

---

## 6. Limitations and workarounds

### 6.1 L1: DDL is not replicated

Symptom: you add a column on node1, the next INSERT replicates, and node2's apply fails with `logical replication target relation "bank.accounts" is missing replicated column "x"`.

**Rule of order:**
- Additive changes (new column, new table, wider type): apply to the **subscriber first**, then the publisher.
- Destructive changes (drop column/table, narrow type): apply to the **publisher first**, then the subscriber.
- New tables: create them on both nodes, then add them to the publication (automatic with `FOR TABLES IN SCHEMA`), then refresh on the subscriber.

```bash
#!/usr/bin/env bash
# 06_apply_ddl.sh: run one DDL file on both nodes in the safe order
# usage: bash 06_apply_ddl.sh additive|destructive change.sql
set -euo pipefail
MODE="$1"; FILE="$2"
PUB="host=node1 dbname=bankdb"; SUB="host=node2 dbname=bankdb"
run() { echo ">> $1"; psql "$2" -X -v ON_ERROR_STOP=1 -1 -f "$FILE"; }
if [[ "$MODE" == additive ]]; then run "subscriber" "$SUB"; run "publisher" "$PUB"
else run "publisher" "$PUB"; run "subscriber" "$SUB"; fi
# new tables need SELECT for the replication user BEFORE the refresh, otherwise their initial copy fails and
# (with disable_on_error) the WHOLE subscription is disabled, so every other table stops replicating too
psql "$PUB" -X -c "GRANT SELECT ON ALL TABLES IN SCHEMA bank TO repuser;"
# pick up new tables (no-op if none)
psql "$SUB" -X -c "ALTER SUBSCRIPTION bank_sub REFRESH PUBLICATION WITH (copy_data = true);"
echo "DDL applied in $MODE order"
```

Example:

```sql
-- change.sql
ALTER TABLE bank.customers ADD COLUMN phone text;
CREATE TABLE bank.loans (loan_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, account_id bigint, principal numeric);
```

`bash 06_apply_ddl.sh additive change.sql`

**Detect forgotten DDL**: an event trigger on the publisher that records every DDL, so you can compare and alert:

```sql
-- node1
CREATE TABLE public.ddl_history (id bigserial PRIMARY KEY, at timestamptz DEFAULT now(), username text DEFAULT current_user, tag text, query text);
CREATE OR REPLACE FUNCTION public.log_ddl() RETURNS event_trigger LANGUAGE plpgsql AS $$
BEGIN INSERT INTO public.ddl_history(tag, query) VALUES (tg_tag, current_query()); END $$;
CREATE EVENT TRIGGER trg_log_ddl ON ddl_command_end EXECUTE FUNCTION public.log_ddl();
-- review: SELECT * FROM public.ddl_history ORDER BY id DESC;
```

### 6.2 L2: Sequences are not replicated

The subscriber's sequences stay at their initial values. After cutover, the first `nextval()` would produce a **duplicate key**. Sync them just before cutover (and periodically in rehearsals):

```bash
#!/usr/bin/env bash
# 06_sync_sequences.sh: copy every sequence's position from publisher to subscriber, plus a safety gap
# usage: bash 06_sync_sequences.sh "host=node1 dbname=bankdb" "host=node2 dbname=bankdb" [gap]
set -euo pipefail
PUB="$1"; SUB="$2"; GAP="${3:-1000}"
psql "$PUB" -X -At -c "
  SELECT format('SELECT setval(%L, %s, true);', format('%I.%I', schemaname, sequencename),
                coalesce(last_value, start_value) + $GAP * increment_by)
  FROM pg_sequences" > /tmp/setval.sql
wc -l < /tmp/setval.sql | xargs echo "sequences:"
psql "$SUB" -X -v ON_ERROR_STOP=1 -1 -f /tmp/setval.sql >/dev/null
psql "$SUB" -X -c "SELECT schemaname, sequencename, last_value FROM pg_sequences ORDER BY 1,2;"
```

This covers identity columns too (they're backed by sequences). The gap absorbs values used between the snapshot and the final freeze. Run it **after** writes are frozen at cutover.

### 6.3 L3: Tables without a primary key

Covered in 4.2. Re-check before every cutover with the query in 4.2. `REPLICA IDENTITY FULL` works but makes each UPDATE/DELETE on the subscriber search for the row, which is slow without an index. PG16+ can use a suitable non-unique btree index on the subscriber.

### 6.4 L4: Large objects are not replicated

`bank.documents.content_oid` replicates as a number, but the object itself (`pg_largeobject`) doesn't. Options:

**A. Copy the LOs at cutover** (writes frozen):

```bash
#!/usr/bin/env bash
# 06_copy_large_objects.sh: dump only the large objects from node1 and load them into node2 (same OIDs)
set -euo pipefail
pg_dump -h node1 -d bankdb -Fc --large-objects --exclude-table-data='*.*' -f /tmp/lo.dump   # LOs + (tiny) schema, no table rows
pg_restore -l /tmp/lo.dump | grep -Ei 'BLOB|LARGE OBJECT' > /tmp/lo.list     # keep only LO entries
pg_restore -h node2 -d bankdb -L /tmp/lo.list /tmp/lo.dump
psql -h node1 -d bankdb -X -At -c "select count(*) from pg_largeobject_metadata"
psql -h node2 -d bankdb -X -At -c "select count(*) from pg_largeobject_metadata"
```

The OIDs are preserved, so `content_oid` values on node2 resolve. LOs that already exist on node2 (from a previous run) make restore fail: run on an LO-free target, or `SELECT lo_unlink(oid) FROM pg_largeobject_metadata;` on node2 first (in the lab only!).

**B. Migrate to `bytea`** (replicates natively; recommended long term):

```sql
-- on BOTH nodes (subscriber first): new column
ALTER TABLE bank.documents ADD COLUMN content bytea;
-- on the publisher: backfill (replicates as UPDATEs), then switch the app to the column
UPDATE bank.documents SET content = lo_get(content_oid) WHERE content IS NULL;
```

See SOP-09 for LO details.

### 6.5 L5: Materialized views are not replicated

The subscriber has the matview definition (from the schema dump) but its own data. Refresh it after cutover, and/or on a schedule:

```sql
REFRESH MATERIALIZED VIEW bank.branch_balance;             -- node2
-- or keep it current with pg_cron on node2 (SOP-10):
-- SELECT cron.schedule('refresh-branch-balance', '*/15 * * * *', 'REFRESH MATERIALIZED VIEW bank.branch_balance');
```

### 6.6 L6: Conflicts (apply errors on the subscriber)

Typical cause: someone wrote to the subscriber, or a row was inserted on both sides.

```sql
-- node2 (as a test): create a conflict
INSERT INTO bank.customers(customer_id, full_name) OVERRIDING SYSTEM VALUE VALUES (999999, 'local row');
-- node1
INSERT INTO bank.customers(customer_id, full_name) OVERRIDING SYSTEM VALUE VALUES (999999, 'publisher row');
```

The node2 log shows something like:

```
ERROR:  duplicate key value violates unique constraint "customers_pkey"
CONTEXT:  processing remote data for replication origin "pg_16999" during message type "INSERT"
          for replication target relation "bank.customers" in transaction 812, finished at 0/3A1B2C8
LOG:  subscription "bank_sub" has been disabled because of an error
```

Resolution, either:

```sql
-- (a) fix the data so the remote change can apply, then re-enable
DELETE FROM bank.customers WHERE customer_id = 999999;          -- node2
ALTER SUBSCRIPTION bank_sub ENABLE;

-- (b) skip the whole remote transaction (it is NOT applied, so the nodes now differ for that txn)
ALTER SUBSCRIPTION bank_sub SKIP (lsn = '0/3A1B2C8');            -- "finished at" LSN from the log
ALTER SUBSCRIPTION bank_sub ENABLE;
```

Prevention: make the subscriber read-only for applications until cutover:

```sql
-- node2: block APPLICATION writes but not replication. Use role-level settings only:
-- the apply worker runs as the subscription owner. A DATABASE-level default_transaction_read_only
-- would also hit the apply worker and break replication.
ALTER ROLE app_role SET default_transaction_read_only = on;
REVOKE INSERT, UPDATE, DELETE, TRUNCATE ON ALL TABLES IN SCHEMA bank FROM app_role;
```

### 6.7 L7: The slot holds WAL when the subscriber is down

A stopped or disabled subscription keeps its slot, and node1 keeps **all WAL since then**. That can fill the disk.

- Set `max_slot_wal_keep_size` (2.1). Past the limit, the slot is invalidated (`wal_status = lost`) and you must re-sync, but the publisher survives.
- PG18: `idle_replication_slot_timeout` invalidates slots that stay inactive too long.
- Remove a slot you no longer need: on node2 `DROP SUBSCRIPTION bank_sub;`. If node2 is gone: on node1 `SELECT pg_drop_replication_slot('bank_sub');`.

### 6.8 L8: Very large initial copy

- Parallel table syncs: `max_sync_workers_per_subscription`.
- Big tables are copied by one worker each. To split them, publish them in a separate publication and subscription, or pre-load them.
- **Pre-load + `copy_data = false`**: restore a consistent copy first, then subscribe from that exact point. The robust way in PG17+ is **`pg_createsubscriber`**, which converts a **physical standby** into a logical subscriber with no data copy:

```bash
# node2 is a stopped physical standby of node1, same major version
pg_createsubscriber -D /var/lib/pgsql/17/data -P "host=node1 dbname=bankdb user=postgres" \
  -d bankdb --publication=bank_pub --subscription=bank_sub --dry-run   # then without --dry-run
```

(It needs the same major version, so for a 17→18 upgrade you'd run it on 17, then pg_upgrade the converted subscriber, which SOP-06 supports.)

### 6.9 L9: Partitioned tables

With `publish_via_partition_root = true` (4.3), the subscriber can use a different partition layout. Without it, every partition must exist on the subscriber with the same name. New partitions: create them on **both** sides **before** rows arrive (use `06_apply_ddl.sh additive`, or pg_partman on both nodes with the same premake).

### 6.10 L10: Triggers, generated columns, and other behaviour

- Ordinary triggers **don't fire** on the subscriber for replicated changes (the apply worker uses `session_replication_role = replica`). To make one fire: `ALTER TABLE t ENABLE ALWAYS TRIGGER trg;` (or `ENABLE REPLICA TRIGGER`).
- Stored generated columns are computed **on the subscriber** from replicated base columns. PG18 can publish them instead (`publish_generated_columns = stored`).
- `TRUNCATE` is replicated (if published). With FKs, every table in the FK group must be in the same subscription.
- Column types may differ if the text representation converts (int → bigint is fine).

---

## 7. Operational tasks

| Task | Command |
|---|---|
| Pause / resume apply | `ALTER SUBSCRIPTION bank_sub DISABLE;` / `ENABLE;` |
| Add a table explicitly | node1 `ALTER PUBLICATION bank_pub ADD TABLE bank.loans;` node2 `ALTER SUBSCRIPTION bank_sub REFRESH PUBLICATION;` |
| Remove a table | node1 `ALTER PUBLICATION bank_pub DROP TABLE bank.loans;` node2 `REFRESH PUBLICATION` |
| Re-copy one table | node2 `TRUNCATE` it, then node1 drop it from + re-add it to the publication, then node2 `REFRESH PUBLICATION WITH (copy_data = true)` |
| Change connection (password/host) | `ALTER SUBSCRIPTION bank_sub CONNECTION '...';` |
| Who's behind? | queries in section 5 |
| Drop everything | node2 `DROP SUBSCRIPTION bank_sub;` (drops the remote slot) then node1 `DROP PUBLICATION bank_pub;` |

---

## 8. Upgrade cutover: PG17 (node1) → PG18 (node2)

### 8.1 Rehearsal checklist (days before)
- [ ] Every table in `bank` is in `pg_publication_tables`, and every non-PK table is fixed (4.2).
- [ ] `04_compare.sh` OK with the load stopped.
- [ ] The sequence sync script tested (6.2). LO copy tested (6.4). Matview refresh tested (6.5).
- [ ] Application tested against node2 (read-only queries during replication; full tests on a clone).
- [ ] Roles and passwords exist on node2 (copied in 4.1; re-run for roles added since, and set passwords if you used `--no-role-passwords`).
- [ ] Reverse replication prepared (8.4) if a rollback path is required.
- [ ] Cutover timings measured. Typically 2–10 minutes of write freeze.

### 8.2 Cutover runbook

```text
T-30  Announce; check lag < 1 MB and subenabled = t; run 04_compare.sh (expect only small DIFFs from live traffic).
T-0   FREEZE WRITES on node1:
        - pgbouncer PAUSE bankdb (or stop app writers), and
        - ALTER DATABASE bankdb SET default_transaction_read_only = on;  then terminate existing sessions:
          SELECT pg_terminate_backend(pid) FROM pg_stat_activity
          WHERE datname='bankdb' AND backend_type='client backend' AND usename <> 'repuser' AND pid <> pg_backend_pid();
T+1   Wait for zero lag:
        node1: SELECT pg_current_wal_lsn();          -- note L
        node2: SELECT latest_end_lsn FROM pg_stat_subscription WHERE subname='bank_sub';   -- until >= L
T+2   bash 04_compare.sh ...                         -- ALL OK (no live traffic now)
T+3   bash 06_sync_sequences.sh "host=node1 dbname=bankdb" "host=node2 dbname=bankdb" 1000
T+4   bash 06_copy_large_objects.sh                  -- if LOs are used
T+5   node2: REFRESH MATERIALIZED VIEW bank.branch_balance;
T+6   node2: ALTER SUBSCRIPTION bank_sub DISABLE;    -- node2 stops following node1
      node2: ALTER ROLE app_role RESET default_transaction_read_only;  re-GRANT app write privileges (6.6)
T+7   (optional) start reverse replication node2 -> node1 (8.4)
T+8   Repoint the application to node2 (DNS / pgbouncer config / VIP), RESUME.
T+10  Smoke tests; watch errors; check pg_stat_activity on node2.
```

### 8.3 Post-cutover
- Run `ANALYZE;` on node2 (it already had statistics from replication activity, but refresh them anyway).
- Take a **full backup** of node2 (PG18) and set up its standbys (pg_basebackup).
- Keep node1 read-only and intact for the rollback window.

### 8.4 Rollback path: reverse replication 18 → 17 (set up at T+7)

```sql
-- node2 (new primary, PG18)
CREATE PUBLICATION bank_pub_rev FOR TABLES IN SCHEMA bank WITH (publish_via_partition_root = true);
-- node1 (old, PG17): data is identical at T+7, so don't copy
ALTER DATABASE bankdb RESET default_transaction_read_only;   -- the apply worker needs writes; keep apps OFF node1
CREATE SUBSCRIPTION bank_sub_rev
  CONNECTION 'host=node2 dbname=bankdb user=repuser password=Rep#2026'
  PUBLICATION bank_pub_rev WITH (copy_data = false, origin = none);
```

If you roll back: freeze node2, wait for bank_sub_rev to catch up, sync sequences **node2 → node1** (`06_sync_sequences.sh` with the arguments swapped), repoint the app to node1.

### 8.5 Cleanup (after the rollback window)

```sql
-- node1
DROP SUBSCRIPTION IF EXISTS bank_sub_rev;
-- node2
DROP SUBSCRIPTION bank_sub;          -- drops slot bank_sub on node1 (if reachable); else on node1: SELECT pg_drop_replication_slot('bank_sub');
DROP PUBLICATION IF EXISTS bank_pub_rev;
-- node1
DROP PUBLICATION bank_pub;
```

Then decommission node1 (final backup, power off, archive).

---

## 9. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `could not connect to the publisher: ... no pg_hba.conf entry for replication connection` | hba line uses `replication` keyword | Logical needs the **database name** in pg_hba |
| `permission denied for table x` during initial sync, and **all** tables stop replicating | repuser lacks SELECT on a newly added table; with `disable_on_error` the whole subscription is disabled | node1 `GRANT SELECT ON ALL TABLES IN SCHEMA bank TO repuser;` node2 `ALTER SUBSCRIPTION bank_sub ENABLE;` (tested: the backlog then applies) |
| Table stuck in `d` | large table still copying, or COPY errors | `pg_stat_progress_copy`; subscriber log |
| `relation "bank.x" does not exist` on the subscriber | DDL not applied there | Create it, then `REFRESH PUBLICATION` |
| `cannot update table ... does not have a replica identity` (on the **publisher**!) | Table without a PK in a publication that publishes updates | 4.2 |
| Subscription keeps disabling itself | Apply error + `disable_on_error` | Read the log, fix, or `SKIP` (6.6), then `ENABLE` |
| Publisher disk filling with WAL | Inactive slot | 6.7 |
| `replication slot "bank_sub" is active for PID x` when dropping | The subscription is still connected | `ALTER SUBSCRIPTION ... DISABLE` first |
| Duplicate keys after cutover | Sequences not synced | 6.2, with a bigger gap |
