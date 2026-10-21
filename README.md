# Movie & File Vault: TOAST + Large Object demo

A small Flask web app that stores data the way your `pg_dump` / `pg_restore` SOPs talk about it:

| What you do in the browser | What PostgreSQL does |
|---|---|
| Paste a big movie synopsis | `movies.synopsis` (plain `text`) is compressed and moved to the hidden **TOAST** table |
| Upload a PDF / image as **Large Object** | bytes go to `pg_largeobject`; `files.lo_oid` holds only the pointer |
| Upload a PDF / image as **bytea** | bytes go into `files.data` and are TOASTed |
| View image on screen / download PDF | the app reads the bytes back from the database, nothing is on the web server's disk |
| **Verify** page | the DB recomputes md5 of every synopsis and file and compares with the md5 saved at upload |

The test you asked for: load data, `pg_dump`, `pg_restore` into another database, change only the connection
properties, and see the same synopsis, images and PDFs. The Verify page proves it with one fingerprint.

## 1. Install (once)

```bash
cd toast-lo-demo
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt            # flask + psycopg[binary]
```

## 2. Source database

Edit `db.properties` (host, dbname, user, password; use `sslmode = require` for Aurora), then:

```bash
createdb -h <host> -U <user> moviedb
psql -h <host> -U <user> -d moviedb -f schema.sql
python3 seed.py                  # optional: 3 movies (1.6 KB / 95 KB / 1.2 MB) + PDF + PNG (as LO and as bytea)
PORT=5001 python3 app.py         # open http://<this-server>:5001
```

Try it by hand too: Movies > paste or upload a long text > open it again. Files > upload your own PDF and image.
Look at **TOAST & LO internals** to see the stored size vs raw size, the hidden `pg_toast.pg_toast_<oid>` table
and the large object pages. Then open **Verify** and note the `FINGERPRINT`.

## 3. Export and import

```bash
bash scripts/dump_restore.sh                       # local default: moviedb -> moviedb_uat on the same server
# Aurora / another server:
SRC_HOST=... SRC_DB=moviedb SRC_USER=... SRC_PASS=... \
DST_HOST=... DST_DB=moviedb_uat DST_USER=... DST_PASS=... bash scripts/dump_restore.sh
```

It runs `pg_dump -Fc`, lists the archive (you will see `TABLE DATA`, `BLOB`/`BLOBS`, `SEQUENCE SET`), creates the
destination database, runs `pg_restore --no-owner --no-privileges`, and prints row and large-object counts both sides.

## 4. Point the app at the destination ("UAT")

```bash
cp db.properties db.uat.properties      # change: env_label = UAT, host, dbname, user, password
APP_CONFIG=db.uat.properties PORT=5002 python3 app.py
```

The banner changes colour (blue SOURCE, orange UAT). Compare:

```bash
diff <(curl -s localhost:5001/verify.txt) <(curl -s localhost:5002/verify.txt) && echo IDENTICAL
```

Or simply edit `db.properties` of the single running app, save, and refresh the browser (the file is re-read on every
request, no restart).

## 5. Things worth showing in class

1. **Dump without large objects.** `pg_dump -Fc -t files -t movies ...` skips large objects unless you add
   `--large-objects` (`-b`). Restore it and open Verify: the LO files show **MISSING** while the movies and the
   bytea file (all TOAST) are fine. The `files` rows survive, but their `lo_oid` points at nothing. (Tested: this is
   exactly what the app reports.)
2. **TOAST is invisible to dump/restore.** There is no separate step for the synopsis; `pg_dump` reads the
   detoasted value and `pg_restore` re-TOASTs it. The OID of the TOAST table differs on the destination.
3. **Large object OIDs are preserved** by `pg_restore`, so `files.lo_oid` still matches. That is why this app can store
   the OID in a normal column. If you migrate by another route (for example logical replication), large objects are not
   replicated at all.
4. **Deleting a file unlinks its large object** (`lo_unlink`). Deleting only the row would leave an orphan in
   `pg_largeobject`; `vacuumlo` cleans such orphans.
5. **Sequences:** after restore, new inserts continue after the old max id (the dump includes `SEQUENCE SET`).

## Notes for Aurora / RDS

- `sslmode = require` in the properties file. The security group must allow the app server to reach port 5432.
- Use a `pg_dump` client that is the same major version as the server or newer.
- Restoring as a non-superuser works with `--no-owner --no-privileges` (already in the script). Reading
  `pg_largeobject` directly on the internals page may show "n/a" for a non-superuser; the rest of the app does not need it.
- On PostgreSQL 17+ the internals page also shows the exact `pg_column_toast_chunk_id()` per synopsis. This part
  was not exercised in my test (local PostgreSQL 16); on 16 and earlier it falls back to "stored size > 2 KB".

## Files

| File | Purpose |
|---|---|
| `app.py` | the web app (Flask + psycopg 3) |
| `schema.sql` | the two tables |
| `db.properties` | connection settings, re-read on every request |
| `seed.py`, `samples/make_samples.py` | sample data generator (no extra libraries) |
| `scripts/dump_restore.sh` | pg_dump + pg_restore walk-through with counts |
| `templates/` | the HTML pages |

## What was tested

Against PostgreSQL 16 (local): schema, seed, every page (HTTP 200), byte-for-byte equality of downloaded PDF and PNG
(LO and bytea), web uploads, dump and restore into a second database, identical fingerprint on source and
destination, a new insert on the destination, and the dump-without-large-objects case (reported as MISSING).
