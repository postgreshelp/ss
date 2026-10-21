# SOP-01: Major version upgrade with `pg_upgrade` (copy mode)

| Item | Value |
|---|---|
| Purpose | Upgrade a PostgreSQL cluster to a new major version, keeping the old cluster intact for rollback |
| Method | `pg_upgrade --copy` (the default mode) |
| Worked example | PostgreSQL **17 → 18** on RHEL / Rocky 9 with PGDG packages |
| Downtime | Proportional to data size (every data file is copied). Rough guide: 100–300 GB/hour on local SSD |
| Rollback | Simple: stop the new cluster and start the old one. The old cluster is never modified |
| Disk needed | Free space ≥ size of the old cluster (`du -sh $OLD_DATA` plus every tablespace) |
| Related | SOP-02 link mode (fast, harder rollback), SOP-03 tablespaces, SOP-04 replicas, SOP-05 extensions |

---

## 1. When to use copy mode

Use copy mode when:
- **rollback safety matters more than downtime**: the old cluster stays byte-for-byte untouched;
- the data is small or medium (copying is acceptable within the maintenance window);
- the new data directory is on a **different filesystem** (link mode needs the same filesystem).

Faster alternatives: `--link` (seconds, SOP-02), `--clone` (reflinks on XFS with reflink=1, Btrfs, APFS; as fast as link and keeps the old cluster safe), `--copy-file-range` (PG17+, kernel-assisted copy), `--swap` (PG18+, moves directories, no rollback).

---

## 2. Variables (set these first, as the `postgres` OS user)

```bash
export OLD_VER=17
export NEW_VER=18
export OLD_BIN=/usr/pgsql-${OLD_VER}/bin
export NEW_BIN=/usr/pgsql-${NEW_VER}/bin
export OLD_DATA=/var/lib/pgsql/${OLD_VER}/data
export NEW_DATA=/var/lib/pgsql/${NEW_VER}/data
export PGPORT=5432
export UPG_DIR=/var/lib/pgsql/upgrade_$(date +%Y%m%d)   # working dir for logs & scripts
mkdir -p "$UPG_DIR" && cd "$UPG_DIR"
```

> Debian/Ubuntu paths: binaries `/usr/lib/postgresql/18/bin`, data `/var/lib/postgresql/18/main`, configuration in `/etc/postgresql/18/main/`. Debian also offers `pg_upgradecluster`, a wrapper. This SOP uses plain `pg_upgrade`, which behaves the same everywhere.

---

## 3. Pre-upgrade checklist (days before)

| # | Check | Command / action | Pass criteria |
|---|---|---|---|
| 1 | Read the release notes "Migration" section for **every** major version skipped | postgresql.org/docs/release | Incompatibilities understood |
| 2 | Recent verified backup exists | pgBackRest / Barman / `pg_basebackup` + restore test | Restorable |
| 3 | Old cluster health | `SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;` | Far below 2 billion; no wraparound warnings |
| 4 | Encoding / locale / provider | `SELECT datname, pg_encoding_to_char(encoding), datcollate, datctype, datlocprovider FROM pg_database;` | Noted, to use in initdb |
| 5 | Data checksums on the old cluster | `SHOW data_checksums;` | Noted (PG18 initdb enables checksums **by default**) |
| 6 | Extensions in every database | `SELECT current_database(), extname, extversion FROM pg_extension;` (run in each DB) | Every one has a package built for NEW_VER (SOP-05) |
| 7 | Non-default settings | `SELECT name, setting, source FROM pg_settings WHERE source NOT IN ('default','override');` | Saved to a file |
| 8 | ALTER SYSTEM settings | `cat $OLD_DATA/postgresql.auto.conf` | Saved. **These are not carried over** |
| 9 | Prepared transactions | `SELECT * FROM pg_prepared_xacts;` | Empty |
| 10 | Replication slots | `SELECT slot_name, slot_type, active FROM pg_replication_slots;` | Physical slots are not migrated; plan to recreate them. Logical: SOP-06 |
| 11 | Disk space | `df -h $(dirname $NEW_DATA)` vs `du -sh $OLD_DATA` | Free > used |
| 12 | Rehearsal | Run the full procedure on a restored copy | Timings recorded |

Save the evidence:

```bash
psql -X -At -c "SELECT version();"                                   > pre_version.txt
psql -X -c "SELECT name, setting, unit, source FROM pg_settings WHERE source NOT IN ('default','override') ORDER BY 1;" > pre_settings.txt
cp $OLD_DATA/postgresql.conf $OLD_DATA/postgresql.auto.conf $OLD_DATA/pg_hba.conf $OLD_DATA/pg_ident.conf "$UPG_DIR"/
for db in $(psql -X -At -c "SELECT datname FROM pg_database WHERE datallowconn"); do
  psql -X -At -d "$db" -c "SELECT '$db', extname, extversion FROM pg_extension ORDER BY 2;"
done > pre_extensions.txt
psql -X -At -c "SELECT datname, pg_database_size(datname) FROM pg_database ORDER BY 1;"   > pre_db_sizes.txt
```

**Row-count fingerprint** (for validation after the upgrade). Run in each application database:

```sql
-- save as fingerprint.sql
SELECT schemaname || '.' || relname AS tbl, n_live_tup
FROM pg_stat_user_tables ORDER BY 1;
-- For critical tables, use an exact count and checksum:
SELECT count(*), md5(string_agg(t::text, '' ORDER BY id)) FROM accounts t;
```

---

## 4. Install the new binaries (no downtime)

```bash
# as root
dnf install -y postgresql18-server postgresql18-contrib
# plus every extension package used on the old cluster, built for 18 (SOP-05), e.g.:
# dnf install -y postgis35_18 pg_partman_18 pgaudit_18 pg_repack_18 pgvector_18
```

Never let the package manager start or initialise anything automatically against the old data directory.

---

## 5. Initialise the new cluster (no downtime)

Match the **encoding, locale, locale provider and checksum setting** of the old cluster. Otherwise `pg_upgrade --check` fails.

```bash
# as postgres. Example: old cluster is UTF8, en_US.UTF-8, libc provider, NO checksums
$NEW_BIN/initdb -D "$NEW_DATA" \
  --encoding=UTF8 --locale=en_US.UTF-8 --locale-provider=libc \
  --no-data-checksums \
  --username=postgres
```

- If the old cluster **has** checksums, leave out `--no-data-checksums` (PG18 enables them by default).
- Want checksums but the old cluster lacks them? Enable them on the old cluster first, offline: `pg_checksums --enable -D $OLD_DATA` (cluster stopped; reads every block, so time it). Then upgrade.
- If the old cluster uses the ICU provider: `--locale-provider=icu --icu-locale=<same>`.
- Pre-PG18 targets: checksums default to off; add `--data-checksums` only if the old cluster has them.

### Carry the configuration over (by hand, not by copying the whole file)

```bash
# Review the differences between old and new defaults:
diff <(grep -Ev '^\s*(#|$)' "$UPG_DIR/postgresql.conf") <(grep -Ev '^\s*(#|$)' "$NEW_DATA/postgresql.conf")
```

Put your tuned parameters into `$NEW_DATA/postgresql.conf`: `shared_buffers`, `work_mem`, `max_connections`, `wal_level`, `shared_preload_libraries`, logging, and so on. Also:
- re-apply everything from the old `postgresql.auto.conf` (or copy that file after reviewing it);
- copy `pg_hba.conf` and `pg_ident.conf`;
- check that **no removed or renamed parameter** remains (the new server refuses to start on an unknown parameter);
- `shared_preload_libraries` must list the same libraries, and they must be installed for 18.

---

## 6. Compatibility check (no downtime: the old cluster may keep running)

```bash
cd "$UPG_DIR"
$NEW_BIN/pg_upgrade \
  --old-bindir="$OLD_BIN"  --new-bindir="$NEW_BIN" \
  --old-datadir="$OLD_DATA" --new-datadir="$NEW_DATA" \
  --copy --jobs=4 --check
```

Expected ending:

```
Checking for ...                                             ok
...
*Clusters are compatible*
```

If something fails, pg_upgrade names a file in `$NEW_DATA/pg_upgrade_output.d/<timestamp>/` that lists the offending objects. Common findings:

| Message | Fix |
|---|---|
| `loadable_libraries.txt` / "could not load library" | Install the extension package for NEW_VER (SOP-05) |
| `encodings/lc_collate values ... do not match` | Re-run initdb with the matching locale |
| `old cluster does not use data checksums but the new one does` | initdb with `--no-data-checksums`, or enable them on the old cluster |
| `tables_using_reg.txt` (reg* data types) | Change those columns to text/oid |
| `prepared transactions` | `COMMIT PREPARED` / `ROLLBACK PREPARED` |
| `contrib_isn_and_int8_pass_by_value` | Only when moving between 32/64-bit builds |
| `new cluster database "x" is not empty` | Never create objects in the new cluster before upgrading |

Repeat until you see `*Clusters are compatible*`.

---

## 7. Maintenance window: the upgrade

### 7.1 Stop the applications and the old cluster cleanly

```bash
# 1. Stop applications / pgbouncer pools (PAUSE), or block app logins in pg_hba.conf (reject line for the app role, then reload).
#    Do NOT use ALTER DATABASE ... ALLOW_CONNECTIONS false: pg_upgrade refuses to run when any
#    non-template0 database has datallowconn = false (tested: "All non-template0 databases must allow connections").
# 2. Final checks
psql -X -c "SELECT count(*) FROM pg_stat_activity WHERE backend_type='client backend' AND pid<>pg_backend_pid();"
psql -X -c "SELECT * FROM pg_prepared_xacts;"
psql -X -c "CHECKPOINT;"
# 3. Stop (as root)
systemctl stop postgresql-17
systemctl disable postgresql-17
# 4. Confirm a clean shutdown
$OLD_BIN/pg_controldata -D "$OLD_DATA" | grep -E "cluster state|Latest checkpoint location"
#   Database cluster state: shut down        <-- must say "shut down"
```

### 7.2 Run pg_upgrade

```bash
cd "$UPG_DIR"
time $NEW_BIN/pg_upgrade \
  --old-bindir="$OLD_BIN"  --new-bindir="$NEW_BIN" \
  --old-datadir="$OLD_DATA" --new-datadir="$NEW_DATA" \
  --copy --jobs=4 2>&1 | tee pg_upgrade_run.log
```

- `--jobs` parallelises per database (copying and dump/restore of the schema). Use about the number of CPU cores or tablespaces.
- On a reflink-capable filesystem, `--clone` instead of `--copy` makes this nearly instant and keeps the same rollback safety.
- PG18 also carries **optimizer statistics** over (except extended statistics). `--no-statistics` turns that off.

Expected ending:

```
Upgrade Complete
----------------
Some statistics are not transferred by pg_upgrade.
Once you start the new server, consider running these two commands:
    .../vacuumdb --all --analyze-in-stages --missing-stats-only
    .../vacuumdb --all --analyze-only
Running this script will delete the old cluster's data files:
    ./delete_old_cluster.sh
```

Follow the exact commands printed by your pg_upgrade version (older versions print `vacuumdb --all --analyze-in-stages`).

### 7.3 Start the new cluster

```bash
# as root. If the old and new versions both use port 5432, the old one must stay stopped.
systemctl enable --now postgresql-18
systemctl status postgresql-18 --no-pager
```

### 7.4 Rebuild statistics

```bash
$NEW_BIN/vacuumdb --all --analyze-in-stages --missing-stats-only -j 4
$NEW_BIN/vacuumdb --all --analyze-only -j 4         # if pg_upgrade suggested it
```

On PG ≤17 targets, statistics are **not** carried over: run `vacuumdb --all --analyze-in-stages` before opening to users, or plans will be poor.

---

## 8. Validation (before handing back to the application)

```bash
psql -X -c "SELECT version();"                          # 18.x
psql -X -c "SHOW data_checksums;"
psql -X -c "SELECT datname, datallowconn FROM pg_database;"
```

```sql
-- in each application database
SELECT extname, extversion FROM pg_extension ORDER BY 1;       -- compare with pre_extensions.txt
\i fingerprint.sql                                              -- compare counts and checksums
SELECT count(*) FROM pg_stat_user_tables WHERE last_analyze IS NULL AND last_autoanalyze IS NULL; -- after vacuumdb
-- smoke test: a representative read and write transaction from the application
```

Check:
- [ ] Version is 18.x; all databases present; sizes are similar (`pre_db_sizes.txt`).
- [ ] Extensions present. Run `ALTER EXTENSION ... UPDATE` where pg_upgrade produced `update_extensions.sql` (SOP-05).
- [ ] Row counts and checksums of critical tables match.
- [ ] The application connects (pg_hba OK), and a key read and write transaction works.
- [ ] Replication / backups reconfigured: a new stanza or backup for the new version, and **a full backup taken now** (old backups can't be used with the new major version).
- [ ] Monitoring points at the new version (port, data directory, service name).

If you blocked app logins in `pg_hba.conf`, remove the reject line from the **new** cluster's pg_hba.conf and reload.

---

## 9. Rollback (copy mode)

Possible at any time **until you delete the old cluster**, because the old cluster was never modified:

```bash
systemctl stop postgresql-18 && systemctl disable postgresql-18
systemctl enable --now postgresql-17
```

Caveat: any writes made on the new cluster after go-live are **lost** on rollback. Decide the rollback cut-off point (for example, before reopening to users) in advance.

---

## 10. Post-upgrade housekeeping (after the agreed soak period, e.g. 1–2 weeks)

```bash
# Take a fresh full backup of the new cluster FIRST
cd "$UPG_DIR" && ./delete_old_cluster.sh      # removes the old data dir (and old tablespace subdirs)
dnf remove -y 'postgresql17*'                  # optional, once nothing depends on it
```

- Recreate physical replication slots and standbys (SOP-04).
- Update `PATH`, `PGDATA`, cron jobs, backup configuration, monitoring and runbooks to version 18.
- Review new PG18 features and defaults with the application team.

---

## 11. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `There seems to be a postmaster servicing the old cluster` | The old cluster is still running. Stop it (only `--check` may run against a live cluster) |
| `could not connect to source postmaster` | pg_upgrade connects through a Unix socket in the current directory. Run it from a directory postgres can write to, with local `peer`/`trust` in both pg_hba files |
| `New cluster database "postgres" is not empty` | Objects were created in the new cluster. Re-initdb it |
| `All non-template0 databases must allow connections` | A database has `datallowconn = false`. On the old cluster: `ALTER DATABASE x ALLOW_CONNECTIONS true;` (block logins via pg_hba instead) |
| Failure halfway through the copy | The old cluster is still intact. Fix the cause, `rm -rf $NEW_DATA`, re-initdb, re-run |
| New server won't start: `unrecognized configuration parameter` | Remove or rename that parameter in postgresql.conf / postgresql.auto.conf |
| Poor performance right after the upgrade | Statistics missing. Run the vacuumdb commands from 7.4 |
| Logs | `$NEW_DATA/pg_upgrade_output.d/` (kept on failure, or with `--retain`) |

---

## Appendix A: one-shot pre-check script

```bash
#!/usr/bin/env bash
# precheck_upgrade.sh: run as postgres against the OLD cluster
set -euo pipefail
echo "== version";            psql -X -At -c "select version()"
echo "== checksums";          psql -X -At -c "show data_checksums"
echo "== locale/encoding";    psql -X -At -c "select datname, pg_encoding_to_char(encoding), datcollate, datctype, datlocprovider from pg_database"
echo "== xid age (top 5)";    psql -X -At -c "select datname, age(datfrozenxid) from pg_database order by 2 desc limit 5"
echo "== prepared xacts";     psql -X -At -c "select count(*) from pg_prepared_xacts"
echo "== slots";              psql -X -At -c "select slot_name, slot_type, active, wal_status from pg_replication_slots"
echo "== preload libs";       psql -X -At -c "show shared_preload_libraries"
echo "== tablespaces";        psql -X -At -c "select spcname, pg_tablespace_location(oid) from pg_tablespace"
for db in $(psql -X -At -c "select datname from pg_database where datallowconn"); do
  echo "== extensions in $db"; psql -X -At -d "$db" -c "select extname, extversion from pg_extension order by 1"
  echo "== reg* columns in $db"
  psql -X -At -d "$db" -c "select c.oid::regclass, a.attname, a.atttypid::regtype from pg_attribute a join pg_class c on c.oid=a.attrelid
    where a.attnum>0 and not a.attisdropped and a.atttypid in ('regproc'::regtype,'regprocedure'::regtype,'regoper'::regtype,'regoperator'::regtype,'regconfig'::regtype,'regdictionary'::regtype,'regnamespace'::regtype,'regcollation'::regtype)
    and c.relnamespace not in ('pg_catalog'::regnamespace,'information_schema'::regnamespace)"
done
echo "== disk";  du -sh "$PGDATA"; df -h "$(dirname "$PGDATA")"
```
