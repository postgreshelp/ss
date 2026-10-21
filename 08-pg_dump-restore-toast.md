# SOP-08: `pg_dump` / `pg_restore` of tables with large TOASTed values

| Item | Value |
|---|---|
| Purpose | Reliably dump, restore and verify tables whose rows hold big `text` / `jsonb` / `bytea` values (stored out-of-line in TOAST), including the failure modes specific to them |
| Applies to | PostgreSQL 14+ (compression notes); the rest applies to all supported versions |
| Worked example | Database `docdb`, table `app.payloads` with ~2 KB–5 MB values |
| Key point | TOAST is **transparent** to pg_dump: there's nothing special to "include". The problems are **size limits, memory, time, compression choice, and corruption** |

---

## 1. TOAST in 60 seconds

- A row must fit in an 8 KB page. When a row exceeds ~2 KB (`TOAST_TUPLE_THRESHOLD`), PostgreSQL **compresses** large values and/or moves them **out of line** into the table's TOAST relation `pg_toast.pg_toast_<oid>`, in ~2 KB chunks.
- Per-column strategy (`attstorage`): `p` plain, `m` main (compress, avoid out-of-line), `x` extended (compress, then out-of-line; the default for text/jsonb/bytea), `e` external (out-of-line, **no compression**: fast substring access).
- Compression method per column (PG14+): `pglz` (default) or `lz4` (`default_toast_compression`, `ALTER TABLE ... ALTER COLUMN ... SET COMPRESSION lz4`).
- One field can hold at most **1 GB**.

pg_dump reads every row through `COPY ... TO STDOUT` (text format). Values are **de-TOASTed and decompressed** on the server, sent as text, and on restore **re-compressed and re-TOASTed** according to the target column's settings.

---

## 2. Lab setup (reproducible example)

```sql
CREATE DATABASE docdb;
\c docdb
CREATE SCHEMA app;
CREATE TABLE app.payloads (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  kind       text NOT NULL,
  body_text  text,          -- big text
  body_json  jsonb,         -- big jsonb
  body_bin   bytea,         -- big binary
  created_at timestamptz DEFAULT now()
);
-- compressible text (repetitive) and incompressible binary (random) of various sizes
INSERT INTO app.payloads(kind, body_text)
  SELECT 'text', repeat('MyBank statement line '||g||'. ', 100 * (1 + g % 50)) FROM generate_series(1,2000) g;
INSERT INTO app.payloads(kind, body_json)
  SELECT 'json', jsonb_build_object('id', g, 'items', (SELECT jsonb_agg(jsonb_build_object('n', i, 'v', md5(i::text))) FROM generate_series(1, 200) i))
  FROM generate_series(1,1000) g;
INSERT INTO app.payloads(kind, body_bin)
  SELECT 'bin', (SELECT string_agg(decode(md5(random()::text), 'hex'), ''::bytea) FROM generate_series(1, 2000 * (1 + g % 20)))
  FROM generate_series(1,300) g;
ANALYZE app.payloads;
```

---

## 3. Inspect TOAST usage before dumping

```sql
-- Heap vs TOAST vs index size
SELECT c.relname,
       pg_size_pretty(pg_relation_size(c.oid))                         AS heap,
       pg_size_pretty(pg_relation_size(c.reltoastrelid))               AS toast,
       pg_size_pretty(pg_indexes_size(c.oid))                          AS indexes,
       pg_size_pretty(pg_total_relation_size(c.oid))                   AS total,
       c.reltoastrelid::regclass                                       AS toast_table
FROM pg_class c WHERE c.oid = 'app.payloads'::regclass;

-- Per-column storage strategy and compression
SELECT attname, atttypid::regtype, attstorage, attcompression
FROM pg_attribute WHERE attrelid = 'app.payloads'::regclass AND attnum > 0 AND NOT attisdropped;

-- Value sizes: stored (compressed) vs logical, and the compression actually used (PG14+)
SELECT kind, count(*),
       pg_size_pretty(avg(coalesce(pg_column_size(body_text), pg_column_size(body_json), pg_column_size(body_bin)))::bigint) AS avg_stored,
       pg_size_pretty(max(octet_length(coalesce(body_text, body_json::text, encode(body_bin,'hex'))))::bigint) AS max_text_form,
       string_agg(DISTINCT coalesce(pg_column_compression(body_text), pg_column_compression(body_json), pg_column_compression(body_bin), 'none'), ',') AS compression
FROM app.payloads GROUP BY kind;

-- Largest rows: these decide memory/timeout needs and the 1 GB COPY risk
SELECT id, kind, pg_size_pretty(octet_length(body_bin)::bigint) AS bin_size,
       pg_size_pretty(octet_length(body_text)::bigint) AS text_size
FROM app.payloads ORDER BY greatest(coalesce(octet_length(body_bin),0), coalesce(octet_length(body_text),0)) DESC LIMIT 5;
```

---

## 4. Risks specific to TOAST-heavy tables

| Risk | Why | Mitigation |
|---|---|---|
| **Dump fails for `bytea` values > ~500 MB** | COPY text output of bytea is **hex**, which doubles the size, and one COPY row can't exceed 1 GB. Error: `invalid memory alloc request size` / `out of memory` | Section 7 (binary COPY of those rows), or move such files to large objects / object storage |
| Long dump, CPU bound | Every value is decompressed on dump and re-compressed on restore | Parallel directory format `-Fd -j`; lz4 compression for columns; dump compression `zstd` or `lz4` (PG16+) |
| Huge dump file | Decompressed data, then dump-compressed again | `-Z zstd:3` (PG16+), `-Z 6` gzip, or `-Z 0` when the data is already incompressible (binary) |
| Corrupt TOAST stops the dump | `ERROR: missing chunk number 0 for toast value 123 in pg_toast_16400` | Section 8: find and isolate the bad rows, dump the rest |
| Restore WAL burst / slow | TOAST writes are large | `-j`, `--no-tablespaces`, restore into a fresh cluster with `wal_level=minimal` (if no replicas), `maintenance_work_mem` high |
| `statement_timeout` / idle timeouts kill a long COPY | One table's COPY is a single statement | pg_dump sets `statement_timeout=0` for its own session; check **pgbouncer / load balancer** idle timeouts on the path |

---

## 5. Standard procedure: dump

```bash
export PGHOST=db1 PGDATABASE=docdb PGUSER=postgres
DUMP_DIR=/backup/docdb_$(date +%Y%m%d_%H%M)

# Directory format, 4 parallel workers, zstd compression (PG16+; use -Z 6 on older pg_dump)
time pg_dump -Fd -j 4 -Z zstd:3 -f "$DUMP_DIR" -d docdb -v 2> "$DUMP_DIR.log"
grep -iE "error|warning" "$DUMP_DIR.log"
du -sh "$DUMP_DIR"
pg_restore -l "$DUMP_DIR" | grep -c "TABLE DATA"
```

Notes:
- `-j` parallelises **by table**. One huge TOAST table is still dumped by a single worker. Split it with `--table` runs if needed (section 7).
- `--no-toast-compression` (PG14+): leave the per-column `SET COMPRESSION` out of the dump, so restored columns use the **target's** `default_toast_compression`. Use it when moving to a server where you want `lz4` everywhere.
- For a dump of just this table: `pg_dump -Fc -t app.payloads -f payloads.dump`.

---

## 6. Standard procedure: restore and verify

### 6.1 Restore

```bash
createdb -h db2 docdb
# Optional: choose lz4 for restored values (affects columns without an explicit SET COMPRESSION, or all of them if dumped with --no-toast-compression)
psql -h db2 -d docdb -c "ALTER DATABASE docdb SET default_toast_compression = 'lz4';"
time pg_restore -h db2 -d docdb -j 4 --no-owner --exit-on-error -v "$DUMP_DIR" 2> restore.log
```

A restore **re-compresses** every value, which makes it the one moment you can switch TOAST compression for free. `VACUUM FULL` / `CLUSTER` don't recompress existing values (since PG14), but inserting the data again does.

### 6.2 Verify (counts, content hashes, compression)

```sql
-- run on BOTH source and target and compare outputs
SELECT kind, count(*),
       sum(octet_length(body_text)) AS text_bytes,
       sum(octet_length(body_bin))  AS bin_bytes,
       md5(string_agg(md5(coalesce(body_text,'') || coalesce(body_json::text,'') || encode(coalesce(body_bin,''::bytea),'hex')), '' ORDER BY id)) AS content_hash
FROM app.payloads GROUP BY kind ORDER BY kind;

-- target only: confirm the compression in use
SELECT kind, coalesce(pg_column_compression(body_text), pg_column_compression(body_json), 'not compressed') AS method, count(*)
FROM app.payloads GROUP BY 1,2 ORDER BY 1,2;
-- note: pass the COLUMN itself; pg_column_compression(col::text) inspects a computed copy and returns NULL
```

Script form:

```bash
#!/usr/bin/env bash
# verify_toast_copy.sh SRC_CONNINFO DST_CONNINFO
Q="SELECT kind||' '||count(*)||' '||md5(string_agg(md5(coalesce(body_text,'')||coalesce(body_json::text,'')||encode(coalesce(body_bin,''::bytea),'hex')),'' ORDER BY id)) FROM app.payloads GROUP BY kind ORDER BY kind"
diff <(psql "$1" -X -At -c "$Q") <(psql "$2" -X -At -c "$Q") && echo "TOAST data identical" || { echo "DIFFERENCE FOUND"; exit 1; }
```

Also run `ANALYZE` on the target (pg_restore doesn't restore statistics before PG18; PG18's pg_dump can include statistics with `--statistics`).

---

## 7. Values too big for text COPY (bytea > ~500 MB, text near 1 GB)

Symptom during pg_dump:

```
pg_dump: error: Dumping the contents of table "payloads" failed: PQgetResult() failed.
pg_dump: detail: Error message from server: ERROR:  invalid memory alloc request size 1073741824
```

Workaround: dump the table **without those rows' data**, and move the big rows with **binary COPY**, which has no hex expansion.

```bash
# 1. Everything except the big table's data
pg_dump -Fd -j 4 -f dump_main --exclude-table-data=app.payloads -d docdb
# 2. Big table: normal rows via text COPY, huge rows via binary COPY
psql -d docdb -X -c "\copy (SELECT * FROM app.payloads WHERE coalesce(octet_length(body_bin),0) < 400*1024*1024) TO 'payloads_normal.copy'"
psql -d docdb -X -c "\copy (SELECT * FROM app.payloads WHERE octet_length(body_bin) >= 400*1024*1024) TO 'payloads_huge.bin' WITH (FORMAT binary)"
# 3. Restore
pg_restore -d docdb_new -j 4 dump_main
psql -d docdb_new -X -c "\copy app.payloads FROM 'payloads_normal.copy'"
psql -d docdb_new -X -c "\copy app.payloads FROM 'payloads_huge.bin' WITH (FORMAT binary)"
psql -d docdb_new -X -c "SELECT setval(pg_get_serial_sequence('app.payloads','id'), (SELECT max(id) FROM app.payloads));"
```

Binary COPY needs the **same column types and column order** on both sides (it's fine across major versions for built-in types). `COPY FROM` loads identity values as-is, even into `GENERATED ALWAYS` columns (tested: no `OVERRIDING` needed), so always reset the sequence afterwards with the `setval` shown above. Verify with the hash query from 6.2: the test produced identical hashes for the binary-copied rows. Long term: store files > 100 MB outside the table (object storage + path), or as large objects (SOP-09: LO data is dumped in chunks, so it has no 1 GB COPY row problem).

---

## 8. Corrupt TOAST: dump fails with "missing chunk number"

Symptom:

```
pg_dump: error: Dumping the contents of table "payloads" failed: PQgetResult() failed.
pg_dump: detail: Error message from server: ERROR:  missing chunk number 0 for toast value 16589 in pg_toast_16402
```

### 8.1 Confirm and locate (PG14+: amcheck)

```sql
CREATE EXTENSION IF NOT EXISTS amcheck;
SELECT blkno, offnum, attnum, msg
FROM verify_heapam('app.payloads', check_toast => true) LIMIT 20;
```

### 8.2 Find the bad rows by reading each one (works on any version)

```sql
-- find_bad_toast.sql: lists ids whose TOASTed columns can't be read
DO $$
DECLARE r record; bad int := 0;
BEGIN
  FOR r IN SELECT id FROM app.payloads ORDER BY id LOOP
    BEGIN
      -- md5() forces a full read of every chunk. Do NOT use length(): it can be answered from the
      -- TOAST pointer header without reading the chunks, so it misses missing chunks.
      PERFORM md5(body_text), md5(body_json::text), md5(body_bin) FROM app.payloads WHERE id = r.id;
    EXCEPTION WHEN OTHERS THEN
      bad := bad + 1;
      RAISE NOTICE 'BAD ROW id=% : %', r.id, SQLERRM;
    END;
  END LOOP;
  RAISE NOTICE 'scan done, % bad row(s)', bad;
END $$;
```

### 8.3 Dump everything else, then deal with the bad rows

```bash
# take a file-level copy of the cluster FIRST (pg_basebackup or a stopped-server copy): the evidence
pg_dump -Fd -j 4 -f dump_ok --exclude-table-data=app.payloads -d docdb
psql -d docdb -X -c "\copy (SELECT * FROM app.payloads WHERE id NOT IN (16589 /* bad ids */)) TO 'payloads_good.copy'"
```

Then choose per bad row: restore its value from an older backup, recreate it from the application, or (with business approval) null the column or delete the row on the source: `UPDATE app.payloads SET body_bin = NULL WHERE id = 16589;`. Investigate the cause (storage/firmware, `fsync=off`, an earlier crash, faulty replication) with `pg_amcheck --heapallindexed --rootdescend -d docdb`, and check `data_checksums`.

---

## 9. Quick reference

| Task | Command |
|---|---|
| Size of TOAST for a table | `select pg_size_pretty(pg_relation_size(reltoastrelid)) from pg_class where oid='app.payloads'::regclass;` |
| Compression used by a value | `select pg_column_compression(body_text) from app.payloads limit 1;` |
| Switch future values to lz4 | `ALTER TABLE app.payloads ALTER COLUMN body_text SET COMPRESSION lz4;` (existing values unchanged until rewritten by an UPDATE / dump-restore) |
| Avoid compressing incompressible data | `ALTER TABLE app.payloads ALTER COLUMN body_bin SET STORAGE EXTERNAL;` (also speeds up `substring()` on it) |
| Dump without per-column compression clauses | `pg_dump --no-toast-compression ...` |
| Fastest parallel dump | `pg_dump -Fd -j N -Z zstd:1` (PG16+) |
| Check TOAST integrity | `select * from verify_heapam('app.payloads', check_toast => true);` |
