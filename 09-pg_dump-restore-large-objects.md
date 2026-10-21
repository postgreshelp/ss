# SOP-09: `pg_dump` / `pg_restore` with large objects (uploaded PDFs, text files, images)

| Item | Value |
|---|---|
| Purpose | Store, back up, restore, migrate and verify **large objects** (LOs): files kept in `pg_largeobject` and referenced by an `oid` column |
| Worked example | Database `filesdb`, table `app.documents(doc_id, file_name, mime_type, content_oid oid)` holding uploaded `.pdf` / `.txt` files |
| Key points | LOs are **not rows of your table**: they're separate objects with their own owner, privileges and lifecycle. Selective dumps (`-t`, `-n`) **leave them out** unless you ask for them. Deleting a row doesn't delete its LO |

---

## 1. Large objects in 60 seconds

```
app.documents                         pg_largeobject_metadata      pg_largeobject
doc_id | file_name   | content_oid    oid   | owner | acl           loid  | pageno | data (2 KB)
-------+-------------+------------    ------+-------+-----          ------+--------+-----
1      | invoice.pdf | 24601  ──────▶ 24601 | app   |        ──────▶ 24601 | 0      | ...
                                                                    24601 | 1      | ...
```

- An LO is up to **4 TB**, stored in 2 KB pages, and supports **random access and streaming** (`lo_open`, `lo_lseek`, `loread`). `bytea` is limited to 1 GB per value and is read whole.
- The `oid` column is just a number. There's **no foreign key**, so orphans are common.
- Each LO has an **owner** and an **ACL** (`GRANT SELECT ON LARGE OBJECT 24601 TO reader;`).
- **Not replicated** by logical replication (SOP-07 section 6.4). Physical replication and backups include them.

---

## 2. Lab setup: store real files

```bash
# sample files (the client side)
mkdir -p /tmp/uploads && cd /tmp/uploads
printf 'MyBank statement for account 1001\nBalance: 12,500.00\n' > statement_1001.txt
seq 1 200000 > big_report.txt                                   # ~1.2 MB text
head -c 3000000 /dev/urandom > scan_0001.pdf                     # stands in for a 3 MB PDF
# a minimal real PDF (optional, if a tool is available): echo "hello" | enscript -p - | ps2pdf - real.pdf
sha256sum * > /tmp/uploads.sha256
```

```sql
CREATE DATABASE filesdb;
\c filesdb
CREATE EXTENSION IF NOT EXISTS lo;                                -- 'lo' type + lo_manage trigger
CREATE SCHEMA app;
CREATE TABLE app.documents (
  doc_id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  file_name   text NOT NULL,
  mime_type   text,
  content_oid lo,                                                 -- domain over oid from the 'lo' extension
  uploaded_at timestamptz DEFAULT now()
);
-- Delete the LO automatically when its row is deleted or the oid is changed:
CREATE TRIGGER t_documents_lo BEFORE UPDATE OR DELETE ON app.documents
  FOR EACH ROW EXECUTE FUNCTION lo_manage(content_oid);
```

### 2.1 Upload: client-side (`\lo_import`, no special privileges; the file is on the psql machine)

```bash
cd /tmp/uploads
for f in statement_1001.txt big_report.txt scan_0001.pdf; do
  psql -X -d filesdb -v ON_ERROR_STOP=1 <<SQL
\lo_import '$f' 'uploaded $f'
INSERT INTO app.documents(file_name, mime_type, content_oid)
VALUES ('$f', CASE WHEN '$f' LIKE '%.pdf' THEN 'application/pdf' ELSE 'text/plain' END, :LASTOID);
SQL
done
```

`\lo_import` sets the psql variable `LASTOID` to the new LO's oid.

### 2.2 Upload: server-side (`lo_import()`, the file must be on the DB server; needs superuser or `pg_read_server_files`)

```sql
INSERT INTO app.documents(file_name, mime_type, content_oid)
VALUES ('policy.txt', 'text/plain', lo_import('/var/lib/pgsql/import/policy.txt'));
```

### 2.3 From SQL data (`lo_from_bytea`) and read back (`lo_get`)

```sql
INSERT INTO app.documents(file_name, mime_type, content_oid)
VALUES ('note.txt', 'text/plain', lo_from_bytea(0, convert_to('Generated inside SQL', 'UTF8')));
SELECT file_name, convert_from(lo_get(content_oid), 'UTF8') FROM app.documents WHERE file_name = 'note.txt';
SELECT file_name, pg_size_pretty(length(lo_get(content_oid))::bigint) FROM app.documents;    -- reads whole LO: fine for small files
```

### 2.4 Download (export)

```bash
psql -X -d filesdb -c "\lo_export $(psql -X -At -d filesdb -c "select content_oid from app.documents where file_name='scan_0001.pdf'") '/tmp/scan_0001.out.pdf'"
sha256sum /tmp/scan_0001.out.pdf; grep scan_0001.pdf /tmp/uploads.sha256          # must match
```

---

## 3. Inventory before a dump

```sql
SELECT count(*) AS lo_count FROM pg_largeobject_metadata;
SELECT pg_size_pretty(pg_total_relation_size('pg_largeobject')) AS lo_storage;
-- owners (important for restore)
SELECT pg_get_userbyid(lomowner) AS owner, count(*) FROM pg_largeobject_metadata GROUP BY 1;
-- LOs with explicit grants
SELECT oid, lomacl FROM pg_largeobject_metadata WHERE lomacl IS NOT NULL LIMIT 10;
-- ORPHANS: LOs that no table references (here: only app.documents.content_oid holds LO oids)
SELECT m.oid FROM pg_largeobject_metadata m
WHERE NOT EXISTS (SELECT 1 FROM app.documents d WHERE d.content_oid = m.oid);
-- DANGLING references: rows pointing at LOs that don't exist
SELECT d.doc_id, d.content_oid FROM app.documents d
WHERE d.content_oid IS NOT NULL AND NOT EXISTS (SELECT 1 FROM pg_largeobject_metadata m WHERE m.oid = d.content_oid);
```

### 3.1 Clean up orphans with `vacuumlo`

`vacuumlo` deletes every LO whose oid isn't found in **any** `oid`/`lo` column of the database.

```bash
vacuumlo -n -v filesdb        # dry run: lists what it would remove
vacuumlo -v filesdb           # remove
```

Caution: an LO referenced only from a `text`/`bigint` column, or from another database, looks orphaned to vacuumlo. Check the dry-run list.

---

## 4. Dump: what includes LOs and what doesn't

| Command | LOs included? |
|---|---|
| `pg_dump -d filesdb` (whole database) | **Yes** |
| `pg_dump -d filesdb -n app` or `-t app.documents` | **No**: the table comes with dangling oids |
| `... -t app.documents --large-objects` (`-b`; PG≤15: `--blobs`) | **Yes**, all LOs of the database (they belong to no schema) |
| `... --no-large-objects` (`-B`) | No |
| `pg_dump --schema-only` | No |
| `pg_dumpall` | Yes (per database) |

```bash
export PGDATABASE=filesdb
# Whole database, custom format, parallel-restorable (LOs restore in parallel batches on PG17+)
pg_dump -Fc -Z 6 -f filesdb.dump -d filesdb
# Directory format + parallel dump
pg_dump -Fd -j 4 -f filesdb_dir -d filesdb
# Only the documents table AND the LOs
pg_dump -Fc -t app.documents --large-objects -f documents_with_files.dump -d filesdb

# What's inside?
pg_restore -l filesdb.dump | grep -Ei 'BLOB|LARGE OBJECT' | head
pg_restore -l filesdb.dump | grep -ciE 'BLOB'
```

A `-t`/`-n` dump contains **only the named objects**: not their schema (`CREATE SCHEMA app`), and not extensions they depend on (the `lo` type). To restore it into an empty database, first run `CREATE SCHEMA app; CREATE EXTENSION lo;` (tested: without them the restore fails with `schema "app" does not exist`).

The TOC has an entry per LO (PG ≤16: `BLOB <oid>`), grouped in batches since PG17 (`BLOB METADATA`), plus the `BLOBS` data entry.

---

## 5. Restore

```bash
createdb -h db2 filesdb
pg_restore -h db2 -d filesdb -j 4 --exit-on-error filesdb.dump
```

### 5.1 Owners and privileges

LOs are restored with `ALTER LARGE OBJECT <oid> OWNER TO <role>` and their GRANTs:
- If the roles don't exist on the target, these statements fail. Create the roles first (`pg_dumpall --roles-only`), **or**
- use `--no-owner` (the LOs become owned by the restoring user) and `--no-privileges` (drop the LO GRANTs). Then grant access deliberately:

```sql
-- give the application role read/write on every LO (run on the target after the restore)
DO $$ DECLARE o oid; BEGIN
  FOR o IN SELECT oid FROM pg_largeobject_metadata LOOP
    EXECUTE format('GRANT SELECT, UPDATE ON LARGE OBJECT %s TO app_rw', o);
  END LOOP; END $$;
```

If the application suddenly gets `permission denied for large object 24601` after a restore, ownership/ACL changed. Fix with the loop above, or as a temporary escape hatch `ALTER SYSTEM SET lo_compat_privileges = on;` (disables LO permission checks).

### 5.2 OID collisions

LO oids are **preserved** by pg_restore. Restoring into a database that already has an LO with the same oid fails (`duplicate key value violates unique constraint "pg_largeobject_metadata_oid_index"`). Restore into an **empty** database, or remove the conflicting LOs first.

### 5.3 Restoring only the LOs (e.g. after copying the table another way)

```bash
pg_restore -l filesdb.dump | grep -Ei 'BLOB|LARGE OBJECT' > lo_only.list
pg_restore -h db2 -d filesdb -L lo_only.list filesdb.dump
```

---

## 6. Verification (source vs target)

```sql
-- 1. counts and total size
SELECT count(*), pg_size_pretty(sum(length(lo_get(oid)))::bigint) FROM pg_largeobject_metadata;
-- 2. per-file content hash, joined to the table (the evidence the files survived byte-for-byte)
SELECT d.file_name, md5(lo_get(d.content_oid)) FROM app.documents d ORDER BY d.doc_id;
-- 3. no dangling references
SELECT count(*) FROM app.documents d
WHERE NOT EXISTS (SELECT 1 FROM pg_largeobject_metadata m WHERE m.oid = d.content_oid);
```

```bash
#!/usr/bin/env bash
# verify_lo_copy.sh SRC_CONNINFO DST_CONNINFO
Q="SELECT d.file_name||' '||md5(lo_get(d.content_oid)) FROM app.documents d ORDER BY d.doc_id"
diff <(psql "$1" -X -At -c "$Q") <(psql "$2" -X -At -c "$Q") \
  && echo "All $(psql "$2" -X -At -c 'select count(*) from app.documents') files identical" || { echo "MISMATCH"; exit 1; }
```

`lo_get()` loads the whole object into memory, which is fine for files up to a few hundred MB. For bigger LOs, export them (`\lo_export`) on both sides and compare `sha256sum`.

End-to-end file check: export a few files from the **target** and compare them with the original upload hashes (2.4).

---

## 7. Moving off large objects (optional)

### 7.1 LO → bytea (replicates logically, simpler permissions; values ≤ 1 GB, comfortable ≤ ~100 MB)

```sql
ALTER TABLE app.documents ADD COLUMN content bytea;
UPDATE app.documents SET content = lo_get(content_oid) WHERE content IS NULL;     -- batch by doc_id range on big tables
-- after the application switches to "content":
ALTER TABLE app.documents DISABLE TRIGGER t_documents_lo;   -- keep the trigger from double-unlinking
SELECT lo_unlink(content_oid) FROM app.documents WHERE content_oid IS NOT NULL;
ALTER TABLE app.documents DROP COLUMN content_oid;
DROP TRIGGER t_documents_lo ON app.documents;
```

### 7.2 bytea → LO (for files > 1 GB or where streaming access is needed)

```sql
UPDATE app.documents SET content_oid = lo_from_bytea(0, content) WHERE content_oid IS NULL;
```

---

## 8. Performance and scale notes

| Situation | Note |
|---|---|
| Millions of LOs, PG ≤16 | pg_dump keeps one TOC entry per LO (memory, slow), and pg_upgrade was slow too. PG17's pg_dump/pg_upgrade batch them. Use a PG17+ client for the dump |
| Parallel restore | PG17+ restores LO batches in parallel with `-j` |
| Huge single LOs (GBs) | Dumped in chunks, so there's no 1 GB COPY limit (unlike bytea, SOP-08 section 7) |
| Disk space never comes back after deleting files | Orphans: run `vacuumlo`, then `VACUUM pg_largeobject` (as superuser). `VACUUM FULL pg_largeobject` returns space to the OS but locks it |
| Application deletes rows but storage grows | No `lo_manage` trigger, or the app doesn't call `lo_unlink`. Add the trigger (2) and run vacuumlo periodically |

---

## 9. Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| Restored app shows "file not found" / `large object 24601 does not exist` | The dump was made with `-t`/`-n` without `--large-objects` | Re-dump with `--large-objects`, restore with `-L lo_only.list` (5.3) |
| `permission denied for large object 24601` | LO owner/ACL changed (`--no-owner`) or a different app role | GRANT loop (5.1) |
| `must be owner of large object` on unlink/update | Same | `ALTER LARGE OBJECT 24601 OWNER TO app_rw;` |
| `duplicate key ... pg_largeobject_metadata_oid_index` | Target already has LOs with those oids | Restore into an empty DB (5.2) |
| `could not open server file ... Permission denied` on `lo_import('/path')` | The server can't read the file, or the role lacks `pg_read_server_files` | Use client-side `\lo_import` |
| pg_dump very slow, high memory with many LOs | One TOC entry per LO (PG ≤16 client) | Use a PG17+ pg_dump |
