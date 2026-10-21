# SOP-06: `pg_upgrade` when the cluster is a logical replication publisher and/or subscriber

| Item | Value |
|---|---|
| Purpose | Major-upgrade nodes in a logical replication topology without re-copying data |
| Worked example | `node1` publisher (PG17, `bankdb`, publication `bank_pub`) → `node2` subscriber (PG17, subscription `bank_sub`). Both upgraded to **PG18** |
| Key version fact | Since **PostgreSQL 17**, pg_upgrade **migrates logical replication slots** (publisher) and **subscriptions with their state** (subscriber), **if the OLD cluster is 17 or newer**. Older sources: see section 7 |
| Prerequisite reading | SOP-01/02 (the mechanics of pg_upgrade), SOP-07 (logical replication basics) |

---

## 1. What pg_upgrade carries over (old cluster ≥ 17)

| Object | Carried over? | Requirement |
|---|---|---|
| Publications | Yes (schema) | none |
| Logical replication **slots** on the publisher | **Yes** | old ≥ 17; every slot has consumed all WAL (caught up); new cluster `wal_level=logical`; enough `max_replication_slots`; output plugin present |
| Subscriptions + per-table sync state + replication origin position | **Yes** | old ≥ 17; every table in `pg_subscription_rel` in state `r` (ready) or `i` (init); origins exist; enough origin slots on the new cluster |
| Physical slots | No | recreate |
| Data | Yes (pg_upgrade copies/links all data) | |

---

## 2. Topology and order of operations (two nodes)

The supported order (per the PostgreSQL docs, "Upgrading logical replication clusters"):

```
1. node2: ALTER SUBSCRIPTION bank_sub DISABLE         (stop applying; slot on node1 stays)
2. node1: stop writes, wait until the slot is caught up, stop node1 (17)
3. node1: pg_upgrade to 18, start node1 (18)          slot migrated, at the same position
4. node2: stop node2 (17), pg_upgrade to 18, start node2 (18)   subscription migrated (disabled)
5. node2: ALTER SUBSCRIPTION bank_sub ENABLE          streaming resumes from the saved position
```

You can also upgrade only one side (any combination of 17/18 works: logical replication works across major versions). The steps below cover both; skip the half you don't need.

Variables (on each node):

```bash
export OLD_BIN=/usr/pgsql-17/bin NEW_BIN=/usr/pgsql-18/bin
export OLD_DATA=/var/lib/pgsql/17/data NEW_DATA=/var/lib/pgsql/18/data
```

---

## 3. Pre-checks

### 3.1 Publisher (node1)

```sql
-- Slots, their plugin, and how far behind they are
SELECT slot_name, plugin, slot_type, database, active, wal_status, conflicting,
       confirmed_flush_lsn, pg_current_wal_lsn(),
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS behind
FROM pg_replication_slots WHERE slot_type = 'logical';
-- No 'lost' wal_status and no conflicting = true allowed (pg_upgrade refuses them; drop/recreate those).

SHOW wal_level;                  -- logical (needed on the NEW cluster too)
SHOW max_replication_slots;      -- new cluster must have >= number of logical slots
SELECT pubname, puballtables, pubinsert, pubupdate, pubdelete, pubtruncate FROM pg_publication;
SELECT * FROM pg_publication_tables;
```

### 3.2 Subscriber (node2)

```sql
SELECT subname, subenabled, subslotname, subconninfo FROM pg_subscription;
-- Every table must be ready (r) or init (i), not d (data copy) / f / s:
SELECT srsubid::regclass, srrelid::regclass, srsubstate FROM pg_subscription_rel WHERE srsubstate NOT IN ('r','i');
-- (expect 0 rows)
SELECT roname, remote_lsn FROM pg_replication_origin_status;   -- one origin per subscription (pg_<subid>)
SELECT * FROM pg_stat_subscription;
```

### 3.3 New-cluster settings (both nodes' `postgresql.conf` for 18)

```ini
# publisher (node1, 18)
wal_level = logical
max_replication_slots = 10          # >= logical slots on old node1 + headroom
max_wal_senders = 10
# subscriber (node2, 18)
max_active_replication_origins = 10 # PG18: limits replication origins (>= number of subscriptions). On PG17 this was max_replication_slots
max_logical_replication_workers = 8
max_worker_processes = 16
```

Carry everything else over as in SOP-01 section 5, and run `pg_upgrade --check` on each node beforehand. It checks the slot and subscription requirements and reports them in `pg_upgrade_output.d/*/`.

---

## 4. Procedure

### 4.1 node2: disable the subscription

```sql
-- node2 (17)
ALTER SUBSCRIPTION bank_sub DISABLE;
SELECT subname, subenabled FROM pg_subscription;       -- subenabled = f
```

Writes on node1 now accumulate in the slot (WAL is retained). Keep this phase short, and watch `pg_wal` growth on node1.

### 4.2 node1: stop writes and make sure the slot has consumed **everything**

pg_upgrade requires each logical slot's `confirmed_flush_lsn` to equal the end of WAL (only the shutdown checkpoint may follow). The subscriber has to have confirmed every change, but it's disabled now. So the right sequence is:

1. Stop application writes on node1.
2. **Temporarily re-enable** the subscription on node2 to drain the remaining changes, then disable it again:

```sql
-- node2
ALTER SUBSCRIPTION bank_sub ENABLE;
-- node1: repeat until lag_bytes = 0 (no writes are happening now)
SELECT slot_name, pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) AS lag_bytes FROM pg_replication_slots WHERE slot_type='logical';
-- node2
ALTER SUBSCRIPTION bank_sub DISABLE;
```

(If writes were already stopped before 4.1, the slot is already caught up and you can skip the re-enable.)

### 4.3 node1: upgrade the publisher

```bash
# node1
sudo systemctl stop postgresql-17
$NEW_BIN/pg_upgrade -b $OLD_BIN -B $NEW_BIN -d $OLD_DATA -D $NEW_DATA --link --jobs=4 --check   # final check, cluster stopped
$NEW_BIN/pg_upgrade -b $OLD_BIN -B $NEW_BIN -d $OLD_DATA -D $NEW_DATA --link --jobs=4
sudo systemctl enable --now postgresql-18
```

If the check says:
- `The replication slot "bank_sub" has not consumed the WAL yet`: go back to 4.2. Writes happened after draining, or the subscription wasn't drained.
- `logical replication slots ... is invalid / lost`: the slot is unusable. Drop it on the old cluster; you'll re-sync that subscription (section 7).

Verify on node1 (18):

```sql
SELECT slot_name, plugin, confirmed_flush_lsn, wal_status FROM pg_replication_slots;   -- bank_sub present
SELECT * FROM pg_publication;
```

The application can resume writing to node1 now. Changes queue up in the migrated slot until node2 is back.

### 4.4 node2: upgrade the subscriber

```bash
# node2
sudo systemctl stop postgresql-17
$NEW_BIN/pg_upgrade -b $OLD_BIN -B $NEW_BIN -d $OLD_DATA -D $NEW_DATA --link --jobs=4 --check
$NEW_BIN/pg_upgrade -b $OLD_BIN -B $NEW_BIN -d $OLD_DATA -D $NEW_DATA --link --jobs=4
sudo systemctl enable --now postgresql-18
```

```sql
-- node2 (18)
SELECT subname, subenabled, subslotname FROM pg_subscription;       -- bank_sub, disabled
SELECT srrelid::regclass, srsubstate FROM pg_subscription_rel;      -- all r
SELECT roname, remote_lsn FROM pg_replication_origin_status;        -- position preserved
```

### 4.5 node2: re-enable and verify catch-up

```sql
-- node2
ALTER SUBSCRIPTION bank_sub ENABLE;
SELECT subname, received_lsn, latest_end_lsn, last_msg_receipt_time FROM pg_stat_subscription;
-- node1
SELECT application_name, state, pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag FROM pg_stat_replication;
```

Functional test: insert a marker row on node1 and read it on node2.

```sql
-- node1
INSERT INTO accounts(customer, balance) VALUES ('upgrade-marker', 0) RETURNING id;
-- node2 (a few seconds later)
SELECT * FROM accounts WHERE customer = 'upgrade-marker';
```

Then delete the marker on node1, run statistics on both nodes (`vacuumdb --all --analyze-in-stages --missing-stats-only`), and take full backups.

---

## 5. Other topologies

| Topology | Order |
|---|---|
| **Cascaded** (node1 → node2 → node3) | Disable subscriptions downstream-first (node3, then node2). Upgrade node1, then node2, then node3. Enable subscriptions upstream-first |
| **Bidirectional** (node1 ⇄ node2, `origin = none`) | Disable both subscriptions. Stop writes on both, drain both directions. Upgrade node1 (its subscription is migrated too), then node2. Enable both |
| **Publisher only** | Drain the slot (4.2), upgrade node1, subscribers on any version reconnect automatically |
| **Subscriber only** | Disable, upgrade, enable. The publisher slot simply waits (watch WAL retention) |
| Publisher is also a **physical primary** | Combine with SOP-04. Logical slots migrate only on the node pg_upgrade runs on; standby **failover slots** (PG17 `failover = true`) must be re-synced on the new standbys |

---

## 6. Rollback

- **Copy/clone mode:** stop 18, start 17 on that node. The old slot / subscription is exactly where it was **before** the upgrade.
  - Rolling back the **publisher** after the subscriber consumed changes from the 18 node: the old slot's position is behind what the subscriber already applied, so changes are re-sent → **duplicate key conflicts** on the subscriber. Handle them with `ALTER SUBSCRIPTION ... SKIP` (SOP-07 section 6.6), or re-sync.
- **Link mode:** rollback only before the new cluster first starts (SOP-02 section 6.4).

---

## 7. Old cluster < 17 (slots/subscriptions are NOT migrated)

On PG16 and earlier, pg_upgrade **doesn't** migrate logical slots, and subscriptions come back **without** their replication progress. The procedure is "upgrade, then re-attach without re-copying":

**Publisher < 17:**
1. Stop writes on the publisher. Drain the subscriber (lag = 0).
2. On the subscriber: `ALTER SUBSCRIPTION bank_sub DISABLE; ALTER SUBSCRIPTION bank_sub SET (slot_name = NONE);` (detach it from the slot that will disappear).
3. Upgrade the publisher (slots are dropped by the upgrade; pg_upgrade --check on old versions may ask you to drop them first: `SELECT pg_drop_replication_slot('bank_sub');`).
4. Recreate the slot on the new publisher: `SELECT pg_create_logical_replication_slot('bank_sub', 'pgoutput');`
5. Subscriber: `ALTER SUBSCRIPTION bank_sub SET (slot_name = 'bank_sub'); ALTER SUBSCRIPTION bank_sub ENABLE;`
6. Resume writes. **No changes may happen between steps 1 and 5**, or they're lost for the subscriber.

**Subscriber < 17:** pg_upgrade restores the subscription **disabled**, with no table list and no origin position. Publisher writes must be stopped, and the subscriber fully caught up, **before** you stopped it. Then:

```sql
-- new subscriber (18)
ALTER SUBSCRIPTION bank_sub REFRESH PUBLICATION WITH (copy_data = false);  -- re-register tables WITHOUT re-copying
ALTER SUBSCRIPTION bank_sub ENABLE;   -- streams from the publisher slot's confirmed_flush_lsn
```

Never run a plain `REFRESH PUBLICATION` here: the default `copy_data = true` re-copies every table, producing duplicate rows or key errors.
If the publisher slot is gone or unusable, re-attach with a fresh slot:

```sql
-- publisher:  SELECT pg_drop_replication_slot('bank_sub');   -- if it still exists but is invalid
-- subscriber:
ALTER SUBSCRIPTION bank_sub SET (slot_name = NONE);
DROP SUBSCRIPTION bank_sub;
CREATE SUBSCRIPTION bank_sub CONNECTION 'host=node1 dbname=bankdb user=repuser'
  PUBLICATION bank_pub WITH (copy_data = false);   -- creates slot bank_sub; no re-copy
```

Also re-sync sequences if the subscriber may become the primary (SOP-07 section 6.2).

---

## 8. Troubleshooting

| Message / symptom | Cause | Fix |
|---|---|---|
| `The replication slot "x" has not consumed the WAL yet` | Writes after draining, or the subscriber was disabled before catching up | Stop writes, enable the subscription until lag = 0, disable, retry |
| `logical replication slots with wal_status lost / invalid` | WAL removed (`max_slot_wal_keep_size`) or an invalidated slot | Drop the slot; re-create the subscription with a re-sync (`copy_data = true`) |
| `new cluster: wal_level must be "logical"` | New postgresql.conf | Set `wal_level=logical` in the **new** config |
| `new cluster: max_replication_slots (N) must be >= ...` | Too low | Raise it in the new config |
| Subscriber check: `relation ... is in state 'd'` | Initial sync not finished | Wait for all tables to reach `r` before upgrading |
| Subscriber check about origins / `max_active_replication_origins` | Too low on PG18 | Raise `max_active_replication_origins` |
| After enable: `could not start WAL streaming: replication slot "x" does not exist` | Publisher was < 17 or the slot was dropped | Section 7: recreate the slot, set slot_name, enable |
| Duplicate key errors after re-enable | Changes applied twice (rollback or manual slot recreation) | `ALTER SUBSCRIPTION ... SKIP (lsn = ...)`, or fix the rows (SOP-07 section 6.6) |
