# SOP-10: Top 10 PostgreSQL extensions: install, create, use, verify

| # | Extension | Category | Preload + restart? | Package (RHEL PGDG, PG18) | Package (Debian/Ubuntu) |
|---|---|---|---|---|---|
| 1 | `pg_stat_statements` | Query performance | **yes** | `postgresql18-contrib` | `postgresql-18` (contrib built in) |
| 2 | `pgcrypto` | Hashing / encryption | no | `postgresql18-contrib` | built in |
| 3 | `pg_trgm` | Fuzzy & `LIKE '%x%'` search | no | `postgresql18-contrib` | built in |
| 4 | `postgres_fdw` | Query remote PostgreSQL | no | `postgresql18-contrib` | built in |
| 5 | `pg_partman` | Partition management | optional (bgw) | `pg_partman_18` | `postgresql-18-partman` |
| 6 | `pg_cron` | In-database job scheduler | **yes** | `pg_cron_18` | `postgresql-18-cron` |
| 7 | `pgaudit` | Audit logging | **yes** | `pgaudit_18` | `postgresql-18-pgaudit` |
| 8 | `pg_repack` | Online bloat removal | no | `pg_repack_18` | `postgresql-18-repack` |
| 9 | `pgvector` (`vector`) | Vector similarity / AI | no | `pgvector_18` | `postgresql-18-pgvector` |
| 10 | `postgis` | Geospatial | no | `postgis35_18` | `postgresql-18-postgis-3` |

Package names vary slightly between repositories. Confirm with `dnf search <name>` / `apt-cache search <name>`. **Every example below was run and verified** on a test cluster (PostgreSQL 16 builds of these extensions). The SQL is the same on 17/18.

### General procedure (applies to every extension)

```bash
# 1. Install the OS package for YOUR major version on every node (primary AND standbys)
sudo dnf install -y postgresql18-contrib                    # or the extension's package
# 2. If it needs preloading: add it to shared_preload_libraries and restart
psql -c "SHOW shared_preload_libraries;"
psql -c "ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements,pg_cron,pgaudit';"   # keep existing entries!
sudo systemctl restart postgresql-18
```

```sql
-- 3. Create it in each database that needs it (as a superuser or the DB owner for trusted extensions)
CREATE EXTENSION IF NOT EXISTS <name>;
-- 4. Verify
SELECT extname, extversion, extnamespace::regnamespace FROM pg_extension;
SELECT name, default_version, installed_version FROM pg_available_extensions WHERE name = '<name>';
-- 5. Later: upgrade the extension after a package update
ALTER EXTENSION <name> UPDATE;
```

`ALTER SYSTEM SET shared_preload_libraries` **replaces** the list. Always include the entries that are already there, or the server may start without a library another extension needs. Standbys need the same packages and preload list, or they fail to replay or start.

---

## 1. pg_stat_statements: which queries cost the most

**Why:** cumulative statistics for every normalised query (calls, total/mean time, rows, buffer hits/reads, temp usage, WAL). It's the first tool for "the database is slow".

```ini
# postgresql.conf (restart)
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000
pg_stat_statements.track = top          # 'all' to include statements inside functions
compute_query_id = on                   # default 'auto' is fine when the module is loaded
```

```sql
CREATE EXTENSION pg_stat_statements;

-- Top 10 by total time
SELECT queryid, calls, round(total_exec_time) AS total_ms, round(mean_exec_time::numeric, 2) AS mean_ms,
       rows, shared_blks_hit, shared_blks_read, left(regexp_replace(query, '\s+', ' ', 'g'), 80) AS query
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;

-- Worst cache hit ratio (I/O heavy)
SELECT left(query, 60), calls, shared_blks_read,
       round(100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0), 1) AS hit_pct
FROM pg_stat_statements WHERE shared_blks_read > 0 ORDER BY shared_blks_read DESC LIMIT 10;

-- Reset (e.g. before a load test)
SELECT pg_stat_statements_reset();
```

**Gotchas:** the statistics are reset by a restart only if the stats file is lost (they normally persist across clean restarts) and by a major version upgrade. `query` text is truncated by `track_activity_query_size`. Use `queryid` to correlate with `pg_stat_activity.query_id` and `auto_explain` logs.

---

## 2. pgcrypto: password hashing and column encryption

```sql
CREATE EXTENSION pgcrypto;

-- (a) Password hashing (bcrypt). Store only the hash.
CREATE TABLE app_users (username text PRIMARY KEY, pw_hash text NOT NULL);
INSERT INTO app_users VALUES ('naresh', crypt('S3cret!', gen_salt('bf', 10)));
SELECT username FROM app_users WHERE username = 'naresh' AND pw_hash = crypt('S3cret!', pw_hash);  -- login check: 1 row
SELECT username FROM app_users WHERE username = 'naresh' AND pw_hash = crypt('wrong', pw_hash);    -- 0 rows

-- (b) Symmetric column encryption (PGP)
CREATE TABLE customer_pii (id int PRIMARY KEY, pan_enc bytea);
INSERT INTO customer_pii VALUES (1, pgp_sym_encrypt('ABCDE1234F', 'key-from-vault'));
SELECT id, pgp_sym_decrypt(pan_enc, 'key-from-vault') AS pan FROM customer_pii;

-- (c) Digests and random bytes
SELECT encode(digest('hello', 'sha256'), 'hex');
SELECT encode(gen_random_bytes(16), 'hex');
```

**Gotchas:** the key passes through SQL, so it can appear in logs (`log_statement`, pg_stat_statements). Pass it as a bind parameter from the application, and don't log parameters. `gen_random_uuid()` is in core since PG13 (no pgcrypto needed). PG18 adds core `uuidv7()`.

---

## 3. pg_trgm: fast `LIKE '%term%'`, `ILIKE` and fuzzy matching

```sql
CREATE EXTENSION pg_trgm;
CREATE TABLE customers (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, full_name text);
INSERT INTO customers(full_name) SELECT 'Customer '||g||' '||md5(g::text) FROM generate_series(1,200000) g;
INSERT INTO customers(full_name) VALUES ('Naresh Kumar'), ('Naresh Kumaar'), ('Nareesh Kumar');

CREATE INDEX customers_name_trgm ON customers USING gin (full_name gin_trgm_ops);
ANALYZE customers;

-- substring search uses the index (a plain btree can't)
EXPLAIN (COSTS OFF) SELECT * FROM customers WHERE full_name ILIKE '%kumar%';
--  Bitmap Heap Scan on customers ... -> Bitmap Index Scan on customers_name_trgm

-- fuzzy "did you mean"
SELECT full_name, similarity(full_name, 'Naresh Kumar') AS sim
FROM customers WHERE full_name % 'Naresh Kumar' ORDER BY sim DESC LIMIT 5;
SET pg_trgm.similarity_threshold = 0.5;   -- tune what % matches
```

**Gotchas:** GIN trigram indexes are large and slow down writes. Patterns shorter than 3 characters can't use them. Use `gist_trgm_ops` if you need `ORDER BY full_name <-> 'x'` (KNN).

---

## 4. postgres_fdw: query another PostgreSQL server as if it were local

```sql
CREATE EXTENSION postgres_fdw;
CREATE SERVER reporting_srv FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host 'node2', port '5432', dbname 'filesdb', fetch_size '10000');
CREATE USER MAPPING FOR CURRENT_USER SERVER reporting_srv OPTIONS (user 'report_ro', password 'xxxx');

CREATE SCHEMA remote;
IMPORT FOREIGN SCHEMA app LIMIT TO (documents) FROM SERVER reporting_srv INTO remote;

SELECT count(*) FROM remote.documents;
-- Check the WHERE is executed remotely ("pushdown"):
EXPLAIN (VERBOSE, COSTS OFF) SELECT file_name FROM remote.documents WHERE doc_id = 1;
--  Foreign Scan on remote.documents
--    Remote SQL: SELECT file_name FROM app.documents WHERE ((doc_id = 1))
```

**Gotchas:** joins, aggregates and sorts are pushed down only when safe (same server, shippable functions/operators). Use `EXPLAIN VERBOSE` to see the "Remote SQL". Non-superusers need a password in the user mapping. Set `use_remote_estimate 'true'` on the server for better plans on big remote tables. Store credentials in user mappings, never in views.

---

## 5. pg_partman: automatic creation and retention of time-based partitions

```sql
CREATE SCHEMA partman;
CREATE EXTENSION pg_partman SCHEMA partman;

CREATE TABLE public.events (
  id         bigint GENERATED ALWAYS AS IDENTITY,
  created_at timestamptz NOT NULL,
  payload    jsonb,
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- pg_partman 5.x: native range partitioning, one partition per day, 4 days premade ahead
SELECT partman.create_parent(p_parent_table := 'public.events', p_control := 'created_at',
                             p_interval := '1 day', p_premake := 4);

-- retention: drop partitions older than 30 days on each maintenance run
UPDATE partman.part_config SET retention = '30 days', retention_keep_table = false
WHERE parent_table = 'public.events';

-- maintenance creates future partitions and applies retention: schedule it (pg_cron, section 6) or use the bgw
CALL partman.run_maintenance_proc();

SELECT * FROM partman.show_partitions('public.events');
INSERT INTO public.events(created_at, payload) VALUES (now(), '{"k":1}');
SELECT tableoid::regclass, * FROM public.events;          -- row landed in today's partition
```

Background worker alternative (no cron needed): `shared_preload_libraries = '...,pg_partman_bgw'`, `pg_partman_bgw.interval = 3600`, `pg_partman_bgw.dbname = 'appdb'`.

**Gotchas:** rows outside every partition go to the **default partition** (created by partman) and block creating a partition for that range later. Monitor `SELECT count(*) FROM public.events_default;`. pg_partman 4.x → 5.x is a major change (trigger-based partitioning removed; `create_parent` arguments changed).

---

## 6. pg_cron: schedule SQL inside the database

```ini
# postgresql.conf (restart)
shared_preload_libraries = 'pg_cron'
cron.database_name = 'appdb'          # the ONE database that holds the cron schema/jobs
cron.timezone = 'Asia/Kolkata'        # default GMT
# cron.use_background_workers = on    # run jobs as background workers instead of libpq connections
```

```sql
-- in appdb
CREATE EXTENSION pg_cron;

SELECT cron.schedule('nightly-analyze', '30 2 * * *', 'ANALYZE');
SELECT cron.schedule('partman-maintenance', '*/15 * * * *', 'CALL partman.run_maintenance_proc()');
SELECT cron.schedule('purge-old-audit', '0 3 * * 0', $$DELETE FROM audit_log WHERE event_time < now() - interval '90 days'$$);
-- a job in ANOTHER database:
SELECT cron.schedule_in_database('vacuum-filesdb', '0 4 * * *', 'VACUUM (ANALYZE) app.documents', 'filesdb');
-- every 30 seconds (pg_cron 1.5+ supports seconds):
SELECT cron.schedule('heartbeat', '30 seconds', $$INSERT INTO heartbeat(ts) VALUES (now())$$);

SELECT jobid, jobname, schedule, command, active FROM cron.job;
SELECT jobid, status, return_message, start_time, end_time FROM cron.job_run_details ORDER BY start_time DESC LIMIT 10;

SELECT cron.alter_job(job_id := (SELECT jobid FROM cron.job WHERE jobname = 'heartbeat'), active := false);
SELECT cron.unschedule('heartbeat');
```

**Gotchas:** by default jobs connect through libpq to `cron.host` (localhost), so pg_hba must allow it (or set `cron.use_background_workers = on`). Jobs run as the user who scheduled them. `cron.job_run_details` grows forever: schedule its cleanup (`DELETE FROM cron.job_run_details WHERE end_time < now() - interval '7 days'`). Jobs only run on the primary.

---

## 7. pgaudit: who did what (session and object audit logging)

```ini
# postgresql.conf (restart)
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'ddl, role'            # classes: read, write, function, role, ddl, misc, misc_set, all
pgaudit.log_relation = on            # one log line per table touched
pgaudit.log_parameter = off          # on = log bind values (may expose sensitive data)
log_line_prefix = '%m [%p] %u@%d '   # who/where in each line
```

```sql
CREATE EXTENSION pgaudit;

-- Session audit for one role only (e.g. DBAs): everything they write or change
ALTER ROLE dba_naresh SET pgaudit.log = 'write, ddl, role';

-- Object audit: log every SELECT/UPDATE on a sensitive table, by anyone
CREATE ROLE auditor NOLOGIN;
SET pgaudit.role = 'auditor';                       -- normally set in postgresql.conf
GRANT SELECT, UPDATE ON customer_pii TO auditor;    -- grants to the audit role define WHAT is audited
SELECT * FROM customer_pii;                         -- produces an AUDIT: OBJECT,... line
```

Log output (server log):

```
... LOG:  AUDIT: SESSION,1,1,DDL,CREATE TABLE,TABLE,public.t1,CREATE TABLE t1(id int),<not logged>
... LOG:  AUDIT: OBJECT,1,1,READ,SELECT,TABLE,public.customer_pii,SELECT * FROM customer_pii;,<not logged>
```

**Gotchas:** `pgaudit.log = 'all'` or `'read'` on a busy system produces huge logs. Scope it by role, database, or object audit. The extension version tracks the PG major (pgaudit 18.x for PG18). Ship the logs off the server (they're only as trustworthy as their storage).

---

## 8. pg_repack: remove table/index bloat without long locks

**Why:** `VACUUM FULL` / `CLUSTER` hold an ACCESS EXCLUSIVE lock for the whole rewrite. pg_repack rebuilds the table in the background and only takes short locks at the start and end.

```sql
CREATE EXTENSION pg_repack;            -- in the target database
-- requirement: the table needs a PRIMARY KEY or a UNIQUE index on NOT NULL columns
SELECT pg_size_pretty(pg_total_relation_size('public.big_table'));
```

```bash
# The client binary must match the extension version (installed by the same package)
pg_repack --version
pg_repack -d appdb -t public.big_table --dry-run          # check
pg_repack -d appdb -t public.big_table -j 2               # rebuild table + indexes, 2 parallel index builds
pg_repack -d appdb -t public.big_table --only-indexes     # just the indexes
pg_repack -d appdb -t public.big_table --tablespace=fast_ssd --moveidx   # also move it to another tablespace
```

**Gotchas:** it needs free disk ≈ the table + indexes size. It captures concurrent changes with a trigger and a log table, so very write-heavy tables need a quiet period. DDL on the table while repack runs will fail or be blocked. Don't run two repacks on the same table. Check after an interruption: `DROP EXTENSION pg_repack CASCADE; CREATE EXTENSION pg_repack;` cleans up leftovers.

---

## 9. pgvector: similarity search for embeddings (AI / RAG)

```sql
CREATE EXTENSION vector;

CREATE TABLE kb_chunks (
  id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  doc       text,
  embedding vector(3)                       -- real models: 384, 768, 1536 ... dimensions
);
INSERT INTO kb_chunks(doc, embedding) VALUES
  ('How to reset a PostgreSQL password', '[0.9, 0.1, 0.0]'),
  ('Vacuum and bloat explained',         '[0.1, 0.8, 0.2]'),
  ('pg_upgrade link mode SOP',           '[0.2, 0.1, 0.9]'),
  ('Replication slot monitoring',        '[0.3, 0.7, 0.3]'),
  ('Resetting a forgotten role password','[0.8, 0.2, 0.1]');

-- HNSW index (best recall/speed trade-off); pick the operator class matching your distance
CREATE INDEX kb_chunks_hnsw ON kb_chunks USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);

-- nearest 3 by cosine distance (<=>). Others: <-> L2, <#> negative inner product
SET hnsw.ef_search = 40;                    -- higher = better recall, slower
SELECT id, doc, round((embedding <=> '[0.85, 0.15, 0.05]')::numeric, 4) AS cosine_distance
FROM kb_chunks ORDER BY embedding <=> '[0.85, 0.15, 0.05]' LIMIT 3;
--  id |                 doc                  | cosine_distance
--   1 | How to reset a PostgreSQL password   |          0.0037
--   5 | Resetting a forgotten role password  |          0.0044
--   4 | Replication slot monitoring          |          0.4701

-- Show that the index is used (on 5 rows the planner prefers a seq scan, so disable it for the demo)
SET enable_seqscan = off;
EXPLAIN (COSTS OFF) SELECT id FROM kb_chunks ORDER BY embedding <=> '[0.85, 0.15, 0.05]' LIMIT 3;
--  Limit -> Index Scan using kb_chunks_hnsw on kb_chunks   Order By: (embedding <=> ...)
RESET enable_seqscan;

-- RECALL CHECK: compare the index answer with the exact answer on sample queries
SET enable_indexscan = off;                 -- exact (brute force) search
SELECT id FROM kb_chunks ORDER BY embedding <=> '[0.85, 0.15, 0.05]' LIMIT 3;
RESET enable_indexscan;
```

**Gotchas:**
- HNSW/IVFFlat are **approximate**: always measure recall (the exact vs index comparison above) on real queries before going live. In testing we reproduced a miss: 3 isolated vectors among 50,000 clustered ones were **never** returned by the HNSW index, even with `ef_search = 400` and a rebuild, while exact search ranked them first. Real embedding sets are better distributed, but outliers do get missed.
- The index is used only for `ORDER BY <distance operator> ... LIMIT n` with the **same operator** as its opclass.
- HNSW builds are memory hungry (`maintenance_work_mem`).
- With a `WHERE` filter, approximate search can return fewer than n rows (pgvector 0.8 adds iterative scans: `SET hnsw.iterative_scan = relaxed_order`).
- `ivfflat` builds faster and is smaller, but needs representative data present before the build, and `lists`/`probes` tuning.

---

## 10. PostGIS: geospatial data and queries

```sql
CREATE EXTENSION postgis;
SELECT postgis_full_version();

CREATE TABLE branches (
  id    int PRIMARY KEY,
  name  text,
  geog  geography(Point, 4326)              -- lon/lat on the WGS84 sphere; distances in metres
);
INSERT INTO branches VALUES
  (1, 'MG Road Bengaluru',   ST_MakePoint(77.6070, 12.9752)::geography),
  (2, 'Koramangala',         ST_MakePoint(77.6245, 12.9352)::geography),
  (3, 'Hyderabad Hitech City', ST_MakePoint(78.3772, 17.4435)::geography);
CREATE INDEX branches_geog_gist ON branches USING gist (geog);

-- Branches within 10 km of a customer, nearest first
SELECT name, round(ST_Distance(geog, ST_MakePoint(77.6100, 12.9700)::geography)) AS metres
FROM branches
WHERE ST_DWithin(geog, ST_MakePoint(77.6100, 12.9700)::geography, 10000)
ORDER BY geog <-> ST_MakePoint(77.6100, 12.9700)::geography;

-- GeoJSON for the application / map
SELECT name, ST_AsGeoJSON(geog) FROM branches WHERE id = 1;
```

**Gotchas:** `ST_MakePoint(longitude, latitude)`: **longitude first**. Use `geography` for distances in metres around the globe, and `geometry` (projected SRID) for heavy planar analysis. `ST_DWithin` uses the index. `ST_Distance(...) < x` in a WHERE doesn't. Major upgrades: `SELECT postgis_extensions_upgrade();` (SOP-05).

---

## Bonus: small contrib modules every DBA should know

| Module | Use | Example |
|---|---|---|
| `amcheck` | Detect index/heap corruption | `SELECT bt_index_check('accounts_pkey', true);` / `pg_amcheck -d appdb` |
| `pg_buffercache` | What's in shared_buffers | `SELECT c.relname, count(*) FROM pg_buffercache b JOIN pg_class c ON c.relfilenode=b.relfilenode GROUP BY 1 ORDER BY 2 DESC LIMIT 10;` |
| `auto_explain` (module, no CREATE EXTENSION) | Log plans of slow queries | `session_preload_libraries='auto_explain'`, `auto_explain.log_min_duration='2s'` |
| `pgstattuple` | Exact bloat measurement | `SELECT * FROM pgstattuple('accounts');` |
| `citext` | Case-insensitive text | `email citext UNIQUE` |
| `pg_prewarm` | Warm the cache after a restart | `SELECT pg_prewarm('accounts');` |
| `btree_gist` | Exclusion constraints on scalars | `EXCLUDE USING gist (room WITH =, during WITH &&)` |

---

## Removing an extension

```sql
\dx+ pg_trgm                        -- objects owned by it
DROP EXTENSION pg_trgm;             -- fails if objects depend on it (e.g. an index)
DROP EXTENSION pg_trgm CASCADE;     -- also drops dependent indexes/columns: review first!
```

Then remove it from `shared_preload_libraries` (if listed) and restart. Remove the package last.
