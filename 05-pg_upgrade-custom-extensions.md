# SOP-05: `pg_upgrade` with custom / third-party extensions

| Item | Value |
|---|---|
| Purpose | Upgrade clusters that use contrib and third-party extensions (PostGIS, pg_partman, pg_cron, pgaudit, pgvector, TimescaleDB, in-house C extensions) |
| Worked example | PG **17 → 18**, RHEL/Rocky 9 + PGDG |
| Key point | pg_upgrade copies **catalog objects**. The extension's **shared library (.so) and SQL scripts** must already be installed for the **new** version, and in a version that can load the old extension version |

---

## 1. How extensions move through pg_upgrade

1. pg_upgrade dumps the schema with `pg_dump --binary-upgrade`. Extensions are recreated **at their current version** (for example `postgis 3.4.2` stays `3.4.2`) with the same OIDs.
2. Every C function in the old catalog refers to a library such as `$libdir/postgis-3`. pg_upgrade `--check` tries to **load each library in the new binaries** and lists failures in `loadable_libraries.txt`.
3. After the upgrade, the extension's **catalog version** is still the old one, even if the newer package ships a newer default version. You update it explicitly with `ALTER EXTENSION ... UPDATE` (pg_upgrade writes `update_extensions.sql` for you).

So for each extension you need, **built for PG18**:
- a library that loads, and
- the SQL upgrade path from your current extension version (`ext--old--new.sql`) if you want to move to a newer version.

---

## 2. Inventory (every database, including `template1`)

```bash
for db in $(psql -X -At -c "select datname from pg_database where datallowconn"); do
  psql -X -At -d "$db" -F ' | ' -c "select '$db', e.extname, e.extversion, n.nspname,
         (select default_version from pg_available_extensions a where a.name=e.extname)
       from pg_extension e join pg_namespace n on n.oid=e.extnamespace order by 2"
done | tee pre_extensions.txt
psql -X -At -c "show shared_preload_libraries"
psql -X -At -c "select name, setting from pg_settings where name like '%.%' order by 1"   # extension GUCs (pg_cron.*, pgaudit.*, ...)
```

Also find **C functions that aren't part of any extension** (in-house functions created with `CREATE FUNCTION ... LANGUAGE C`):

```sql
SELECT p.oid::regprocedure, p.probin
FROM pg_proc p JOIN pg_language l ON l.oid = p.prolang
WHERE l.lanname = 'c' AND p.probin IS NOT NULL
  AND NOT EXISTS (SELECT 1 FROM pg_depend d WHERE d.classid='pg_proc'::regclass AND d.objid=p.oid AND d.deptype='e');
```

Their `probin` libraries must also exist (recompiled against PG18) in the new `$libdir`.

### Fill in the extension matrix

| Extension | Old ver (in DB) | PG18 package | Version(s) in package | Upgrade path exists? | preload? | Notes |
|---|---|---|---|---|---|---|
| plpgsql | 1.0 | core | 1.0 | n/a | no | |
| pg_stat_statements | 1.11 | postgresql18-contrib | 1.12 | yes | **yes** | stats reset by upgrade |
| postgis | 3.4.2 | postgis35_18 | 3.5.x | yes (3.4→3.5) | no | also postgis_raster/topology |
| pg_partman | 5.1.0 | pg_partman_18 | 5.2.x | yes | bgw optional | |
| pg_cron | 1.6 | pg_cron_18 | 1.6 | n/a | **yes** | `cron.database_name` |
| pgaudit | 17.0 | pgaudit_18 | 18.0 | yes | **yes** | pgaudit major version tracks PG major |
| vector | 0.7.4 | pgvector_18 | 0.8.x | yes | no | |
| timescaledb | 2.x | timescaledb-2-postgresql-18 | must include **your exact** old version | special | **yes** | see 6.3 |

How to see what a package ships (after installing it):

```bash
ls /usr/pgsql-18/share/extension/ | grep -E '^postgis--' | sort -V | tail
ls /usr/pgsql-18/lib/ | grep -i postgis
```

---

## 3. Install extension packages for the new version (before the window)

```bash
# examples (PGDG RHEL naming: <name>_<pgmajor>)
sudo dnf install -y postgresql18-contrib
sudo dnf install -y postgis35_18 pg_partman_18 pg_cron_18 pgaudit_18 pgvector_18 pg_repack_18
# In-house extension: rebuild against PG18
make PG_CONFIG=/usr/pgsql-18/bin/pg_config clean all && sudo make PG_CONFIG=/usr/pgsql-18/bin/pg_config install
```

Rules:
- **The old extension version must be loadable by the new package.** For PostGIS the library name is major-based (`postgis-3`), so any 3.x package for PG18 can serve a 3.4 catalog. For extensions with versioned library names (TimescaleDB `timescaledb-2.14.2.so`), the new package must contain **the exact old version's library**.
- If the PG18 package doesn't support the old extension version, **first update the extension on the old cluster** to a version that exists for both (`ALTER EXTENSION x UPDATE TO 'y'` on 17), then upgrade PostgreSQL.

---

## 4. New cluster configuration

```bash
$NEW_BIN/initdb -D $NEW_DATA --encoding=UTF8 --locale=en_US.UTF-8 [--no-data-checksums]
```

In `$NEW_DATA/postgresql.conf`, copy the preload list and every extension GUC:

```ini
shared_preload_libraries = 'pg_stat_statements,pg_cron,pgaudit'   # same as old (libraries must exist for 18)
cron.database_name = 'appdb'
pgaudit.log = 'ddl,role'
pg_stat_statements.max = 10000
```

Do **not** `CREATE EXTENSION` in the new cluster yourself (pg_upgrade requires the new cluster to be empty).

---

## 5. Check, upgrade, update extensions

```bash
$NEW_BIN/pg_upgrade -b $OLD_BIN -B $NEW_BIN -d $OLD_DATA -D $NEW_DATA --link --jobs=4 --check
```

If you see:

```
Checking for presence of required libraries                 fatal
Your installation references loadable libraries that are missing from the
new installation. ... A list of problem libraries is in the file:
    .../pg_upgrade_output.d/.../loadable_libraries.txt
```

then open the file: it names each missing library and the databases that use it. Install the right package (section 3), or for an extension you no longer need, `DROP EXTENSION` it on the **old** cluster.

Upgrade (window) as in SOP-01/02, then start the new cluster.

### 5.1 Update extensions to the versions shipped for 18

pg_upgrade writes `update_extensions.sql` in the directory it ran from when updates are available:

```bash
cat update_extensions.sql            # contains \connect + ALTER EXTENSION ... UPDATE; per DB
psql -X -f update_extensions.sql
```

Or do it explicitly, one database at a time, after reading each extension's release notes:

```sql
SELECT name, installed_version, default_version FROM pg_available_extensions
WHERE installed_version IS NOT NULL AND installed_version <> default_version;

ALTER EXTENSION pg_stat_statements UPDATE;
ALTER EXTENSION vector UPDATE;
ALTER EXTENSION pg_partman UPDATE;
-- PostGIS has its own helper that updates all its sub-extensions:
SELECT postgis_extensions_upgrade();
SELECT postgis_full_version();
```

---

## 6. Extension-specific notes

### 6.1 PostGIS
- Check `postgis_full_version()` before and after. After `postgis_extensions_upgrade()` it should not show "procs need upgrade".
- `postgis_raster`, `postgis_topology`, `postgis_sfcgal`, `address_standardizer` must be in the PG18 package set too.
- Spatial indexes (GiST) are binary-compatible. No reindex needed for pg_upgrade itself, but **an OS upgrade that changes glibc collations needs `REINDEX` for text indexes** (unrelated to PostGIS, often done in the same window).

### 6.2 pg_cron
- Jobs live in the `cron.job` table in `cron.database_name`, so they're upgraded with the data.
- Jobs **start running as soon as the new server starts**. Pause risky jobs before the window: `UPDATE cron.job SET active = false WHERE jobname IN (...);`, then re-enable after validation.

### 6.3 TimescaleDB
- The PG18 package must include the **exact TimescaleDB version installed in the old cluster**. Often you first `ALTER EXTENSION timescaledb UPDATE` on 17 to a version supported on both 17 and 18.
- Run `timescaledb_pre_restore()`/`post_restore()` **only** for dump/restore, not for pg_upgrade.
- After the upgrade: `ALTER EXTENSION timescaledb UPDATE;` as the first command in a fresh session (`psql -X`).

### 6.4 pg_partman
- Partitioned tables are native, so they're upgraded like any table. `partman.part_config` comes along.
- 4.x → 5.x is a **major extension upgrade** with breaking changes (trigger-based partitioning removed). Do it separately from the PostgreSQL upgrade, in a test cycle.
- If you use the background worker: `shared_preload_libraries` includes `pg_partman_bgw`, with its GUCs.

### 6.5 pg_stat_statements / pg_buffercache / auto_explain
- Statistics in pg_stat_statements are **reset** (they live in shared memory and a stats file; not carried over). Snapshot the top queries before the upgrade if you need a baseline.

### 6.6 pgaudit
- The pgaudit major version matches the PG major version (pgaudit 18.x for PG18). The library must be installed **and preloaded**, or the server won't start with `pgaudit` in `shared_preload_libraries`.

### 6.7 In-house C extensions
- Rebuild against PG18 headers. Watch for API changes listed in the release notes (for example, hook signatures).
- Bump the extension version if you changed SQL objects, and ship an `ext--old--new.sql` upgrade script.

### 6.8 Extensions that block pg_upgrade (handle on the OLD cluster first)

| Situation | Action before upgrading |
|---|---|
| Extension discontinued / no PG18 build | `DROP EXTENSION` (and dependent objects) or replace it (e.g. `uuid-ossp` → built-in `gen_random_uuid()` / PG18 `uuidv7()`) |
| Data types from an extension stored in user tables, with no PG18 build | Migrate the columns to core types before the upgrade |
| `reg*` types stored in tables (`regproc` etc.) | Convert to `text`/`oid` (`tables_using_reg.txt`) |

---

## 7. Validation

```bash
# compare extension inventory before/after
for db in $(psql -X -At -c "select datname from pg_database where datallowconn"); do
  psql -X -At -d "$db" -F ' | ' -c "select '$db', extname, extversion from pg_extension order by 2"
done > post_extensions.txt
diff <(cut -d'|' -f1,2 pre_extensions.txt) <(cut -d'|' -f1,2 post_extensions.txt) && echo "Same extensions in every DB"
```

Functional smoke tests (adapt to what you use):

```sql
SELECT count(*) FROM pg_stat_statements;                                -- 0 right after start; grows
SELECT postgis_full_version();                                          -- no "need upgrade"
SELECT ST_AsText(ST_Buffer('POINT(0 0)'::geometry, 1));
SELECT jobid, jobname, active FROM cron.job;                            -- re-enable paused jobs
SELECT * FROM partman.show_partitions('public.events') LIMIT 3;
SELECT '[1,2,3]'::vector <-> '[1,2,4]';
SHOW pgaudit.log;
```

---

## 8. Rollback
Same as the mode you used (SOP-01 section 9 copy / SOP-02 section 6.4 link). **`ALTER EXTENSION ... UPDATE` can't be undone in place.** If you need extension-level rollback ability, keep the extension versions unchanged during the PostgreSQL upgrade and update them in a separate change.

---

## 9. Troubleshooting

| Error | Fix |
|---|---|
| `could not load library "$libdir/postgis-3": ... No such file` | PostGIS package for 18 not installed (`postgis35_18`) |
| `could not access file "timescaledb-2.14.2": No such file` | The new package lacks that exact version. Update the extension on 17 first, or install a package that contains it |
| `FATAL: could not access file "pg_cron": No such file` at startup | Preloaded library not installed for 18 |
| `extension "x" has no update path from version "a" to version "b"` | Update in steps (a→intermediate→b), or check that the package includes the scripts |
| Functions error after the upgrade: `could not find function "x" in file ...` | Library version mismatch. Install the matching package, then `ALTER EXTENSION ... UPDATE` |
