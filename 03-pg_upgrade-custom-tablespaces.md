# SOP-03: `pg_upgrade` with custom tablespaces

| Item | Value |
|---|---|
| Purpose | Upgrade a cluster whose tables/indexes live in user-defined tablespaces on other mount points |
| Worked example | PG **17 → 18**, two tablespaces: `ts_data` on `/pgdata_ts/data`, `ts_index` on `/pgdata_ts/index` |
| Modes covered | copy, link (and the clone variant) |
| Key point | pg_upgrade **does not move tablespaces**. It creates a **new version subdirectory inside each existing tablespace location** |

---

## 1. How tablespaces look on disk, before and after

```
Before (17 only)                                After pg_upgrade (17 + 18 side by side)
$OLD_DATA/pg_tblspc/16390 -> /pgdata_ts/data    $NEW_DATA/pg_tblspc/16390 -> /pgdata_ts/data
/pgdata_ts/data/                                /pgdata_ts/data/
└── PG_17_202406281/                            ├── PG_17_202406281/     <- old cluster's files
    └── 16384/24601 ...                         └── PG_18_202506291/     <- new cluster's files
                                                    └── 16384/24601 ...
```

- `PG_<major>_<catalog version>` is the per-version subdirectory. Two versions can therefore share one tablespace location.
- The new cluster's `pg_tblspc/<oid>` symlinks point to the **same locations** as the old ones. pg_upgrade preserves tablespace OIDs and paths.
- **Copy mode:** data is copied into `PG_18_*` on the same mount, so **each tablespace mount needs free space ≥ its used space**.
- **Link mode:** `PG_17_*` → `PG_18_*` hard links, always on the same filesystem (same mount), so link mode works as long as the main data directories also share a filesystem (SOP-02 section 3.1).
- `delete_old_cluster.sh` removes the `PG_17_*` subdirectories as well.

---

## 2. Variables

```bash
export OLD_BIN=/usr/pgsql-17/bin NEW_BIN=/usr/pgsql-18/bin
export OLD_DATA=/var/lib/pgsql/17/data NEW_DATA=/var/lib/pgsql/18/data
export UPG_DIR=/var/lib/pgsql/upgrade_$(date +%Y%m%d); mkdir -p $UPG_DIR; cd $UPG_DIR
```

---

## 3. Inventory the tablespaces

```sql
-- Locations and sizes
SELECT oid, spcname, pg_tablespace_location(oid) AS location,
       pg_size_pretty(pg_tablespace_size(oid)) AS size, spcoptions
FROM pg_tablespace ORDER BY 1;

-- What lives in each tablespace (run in every database)
SELECT coalesce(t.spcname, '(db default)') AS tablespace, c.relkind, count(*),
       pg_size_pretty(sum(pg_relation_size(c.oid))) AS size
FROM pg_class c LEFT JOIN pg_tablespace t ON t.oid = c.reltablespace
WHERE c.relkind IN ('r','i','t','m','p') GROUP BY 1,2 ORDER BY 1,2;

-- Databases whose DEFAULT tablespace is custom
SELECT datname, t.spcname FROM pg_database d JOIN pg_tablespace t ON t.oid = d.dattablespace;
```

```bash
# On disk: symlinks and the version subdirectories
ls -l $OLD_DATA/pg_tblspc/
for l in $OLD_DATA/pg_tblspc/*; do echo "$(basename $l) -> $(readlink $l)"; ls -1 "$(readlink $l)"; done
# Free space per tablespace mount (copy mode needs >= used)
for l in $OLD_DATA/pg_tblspc/*; do d=$(readlink $l); echo "$d: used $(du -sh $d | cut -f1)"; df -h $d | tail -1; done
```

### Red flags to fix BEFORE the upgrade

| Finding | Why it matters | Fix |
|---|---|---|
| Tablespace location **inside** `$OLD_DATA` (e.g. `/var/lib/pgsql/17/data/ts1`) | pg_upgrade warns; `delete_old_cluster.sh` would remove the tablespace with the old data directory | Move it out: `ALTER TABLE ... SET TABLESPACE` to a temporary location, drop and recreate the tablespace outside PGDATA, move back |
| In-place tablespaces (`allow_in_place_tablespaces`, stored under `pg_tblspc/` as directories) | Testing feature only | Move objects to a real tablespace first |
| Symlink target that is a mount point **owned by root** | `initdb`/pg_upgrade can't create `PG_18_*` | Location directory owned by `postgres`, mode 700 |
| A tablespace mount missing on the server | The upgrade fails with "could not create directory" | Mount everything in `/etc/fstab`, as during normal operation |
| Not enough free space on a tablespace mount (copy mode) | The copy fails midway | Use link/clone mode, or add space |
| Someone already created `PG_18_*` dirs (failed earlier attempt) | pg_upgrade refuses: "directory already exists" | Remove the stale `PG_18_*` dirs (after confirming the new cluster isn't in use) |

```bash
# Detect tablespaces inside the data directory
for l in $OLD_DATA/pg_tblspc/*; do case "$(readlink -f $l)" in "$(readlink -f $OLD_DATA)"/*) echo "INSIDE PGDATA: $l -> $(readlink $l)";; esac; done
# Detect stale new-version dirs
for l in $OLD_DATA/pg_tblspc/*; do ls -d "$(readlink $l)"/PG_18_* 2>/dev/null && echo "STALE in $(readlink $l)"; done
```

---

## 4. Build the new cluster

```bash
sudo dnf install -y postgresql18-server postgresql18-contrib
$NEW_BIN/initdb -D $NEW_DATA --encoding=UTF8 --locale=en_US.UTF-8 [--no-data-checksums]   # match old (SOP-01 section 5)
```

Do **not** create the tablespaces in the new cluster yourself: pg_upgrade restores them (with the same OIDs and locations) as part of the schema.

---

## 5. Check

```bash
cd $UPG_DIR
$NEW_BIN/pg_upgrade -b $OLD_BIN -B $NEW_BIN -d $OLD_DATA -D $NEW_DATA --link --jobs=4 --check   # or --copy
```

- `--jobs` also parallelises **per tablespace** for the file transfer. Set it to at least the number of tablespaces.

---

## 6. Upgrade (window)

```bash
sudo systemctl stop postgresql-17; sudo systemctl disable postgresql-17
$OLD_BIN/pg_controldata -D $OLD_DATA | grep "cluster state"        # shut down
cd $UPG_DIR
time $NEW_BIN/pg_upgrade -b $OLD_BIN -B $NEW_BIN -d $OLD_DATA -D $NEW_DATA --link --jobs=4 | tee upgrade.log
```

Verify the tablespace layout **before** starting the new server:

```bash
ls -l $NEW_DATA/pg_tblspc/                       # same OIDs, same targets as the old cluster
for l in $NEW_DATA/pg_tblspc/*; do echo "== $(readlink $l)"; ls -1 "$(readlink $l)"; done
# expect both PG_17_* and PG_18_* in every location
```

Link mode: same go/no-go and rollback rule as SOP-02 section 6.4 (rename `pg_control.old` back **only if** the new cluster has never been started).

```bash
sudo systemctl enable --now postgresql-18
$NEW_BIN/vacuumdb --all --analyze-in-stages --missing-stats-only -j 4
```

---

## 7. Validation

```sql
SELECT spcname, pg_tablespace_location(oid), pg_size_pretty(pg_tablespace_size(oid)) FROM pg_tablespace;
-- Objects are still in their tablespaces:
SELECT c.relname, t.spcname FROM pg_class c JOIN pg_tablespace t ON t.oid=c.reltablespace
WHERE c.relname IN ('accounts','accounts_pkey','transactions') ;
-- Read every block of a big table in a custom tablespace (catches a missing file immediately):
SELECT count(*) FROM transactions;
```

```bash
# Sizes: PG_18_* should be close to PG_17_* (copy), or the same inodes (link)
for l in $NEW_DATA/pg_tblspc/*; do d=$(readlink $l); du -sh $d/PG_17_* $d/PG_18_*; done
```

---

## 8. Cleanup

```bash
cd $UPG_DIR && cat delete_old_cluster.sh      # read it: it lists $OLD_DATA AND every /tablespace/PG_17_* dir
./delete_old_cluster.sh
```

After cleanup, each location contains only `PG_18_*`.

---

## 9. Variants

### 9.1 You want the new version on NEW tablespace disks

pg_upgrade can't relocate tablespaces. Two approaches:
1. Upgrade in place (this SOP), then move objects online-ish with `ALTER TABLE ... SET TABLESPACE` (takes an ACCESS EXCLUSIVE lock and rewrites the table) or `pg_repack --tablespace` (minimal locking; SOP-10).
2. Upgrade via logical replication (SOP-07) to a new server where you create the tablespaces wherever you like.

### 9.2 Standbys with tablespaces

Each tablespace's `PG_17_*` and `PG_18_*` must also be rsynced to the standby. See SOP-04 section 4.4.

### 9.3 Temporary tablespaces (`temp_tablespaces`)

They're upgraded like any other tablespace (they normally contain no persistent files). Make sure the setting is carried into the new `postgresql.conf`.

---

## 10. Troubleshooting

| Error | Fix |
|---|---|
| `could not create directory "/pgdata_ts/data/PG_18_…": File exists` | Stale directory from a previous attempt. Remove it if it's not in use |
| `could not create directory ...: Permission denied` | `chown postgres:postgres /pgdata_ts/data && chmod 700 /pgdata_ts/data` |
| `Invalid cross-device link` in link mode for a tablespace | The tablespace symlink target changed mounts between checks, or a bind mount is involved. Use copy/clone for that layout |
| New server: `could not open file "pg_tblspc/16390/PG_18_.../...": No such file` | The tablespace mount wasn't mounted when the server started. Mount it, restart |
