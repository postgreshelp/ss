# SOP-02: Major version upgrade with `pg_upgrade --link` (hard-link mode)

| Item | Value |
|---|---|
| Purpose | Upgrade large clusters with minimal downtime (seconds to minutes, regardless of size) |
| Method | `pg_upgrade --link`: new cluster files are **hard links** to the old data files |
| Worked example | PostgreSQL **17 → 18**, RHEL/Rocky 9, PGDG packages |
| Downtime | Schema dump/restore time + seconds of linking. Usually < 5 min, even for multi-TB clusters |
| Disk needed | Very little (new catalog files + logs) |
| Rollback | **Only until the new cluster is first started.** After that: restore from backup |
| Hard requirement | Old and new data directories (and each tablespace's old/new subdirectories) on the **same filesystem** |

---

## 1. How link mode works (and why rollback is limited)

```
$OLD_DATA/base/16384/24576   ──hard link──┐
                                         ├── same inode, same data blocks on disk
$NEW_DATA/base/16384/24576   ────────────┘
```

- pg_upgrade restores the schema into the new cluster, then **hard-links** each user data file instead of copying it.
- After the new cluster starts, it writes to those shared files with the new version's format, so the old cluster can no longer use them safely.
- To stop someone starting the old cluster by mistake, pg_upgrade renames `$OLD_DATA/global/pg_control` to `pg_control.old`.

| Mode | Speed | Old cluster after upgrade | Notes |
|---|---|---|---|
| `--copy` | slow | intact | SOP-01 |
| `--clone` | fast | intact | needs reflink FS (XFS reflink=1, Btrfs, APFS) — **prefer it if available** |
| `--link` | fast | unusable once new starts | same filesystem |
| `--swap` (PG18+) | fastest | destroyed during the upgrade | moves directories; no rollback without backup |

Check whether `--clone` is possible: `xfs_info /var/lib/pgsql | grep reflink` (reflink=1 means yes).

---

## 2. Variables

```bash
export OLD_VER=17 NEW_VER=18
export OLD_BIN=/usr/pgsql-17/bin NEW_BIN=/usr/pgsql-18/bin
export OLD_DATA=/var/lib/pgsql/17/data NEW_DATA=/var/lib/pgsql/18/data
export UPG_DIR=/var/lib/pgsql/upgrade_$(date +%Y%m%d); mkdir -p $UPG_DIR; cd $UPG_DIR
```

---

## 3. Pre-checks (in addition to the SOP-01 section 3 checklist)

### 3.1 Same-filesystem check (mandatory)

```bash
stat -c '%d %n' "$OLD_DATA" "$(dirname "$NEW_DATA")"
# Both lines must show the SAME device number.
df -P "$OLD_DATA" "$(dirname "$NEW_DATA")" | awk 'NR>1{print $1, $6}'
```

Each tablespace location also needs to be checked (see SOP-03): pg_upgrade creates the new subdirectory **inside the same location**, so it is on the same filesystem automatically. But if `pg_wal` is a symlink to another disk, that's fine: WAL isn't linked.

### 3.2 A backup you have actually restored

Because rollback after go-live means **restore from backup**, this is not optional:

```bash
pgbackrest --stanza=main --type=full backup     # or your tool
pgbackrest --stanza=main info
```

Record the backup label and how long a restore takes (your real RTO if link mode goes wrong).

### 3.3 Rehearse on a clone

Restore the backup to a test host and run this whole SOP there. Record:
- the time `pg_upgrade --check` takes;
- the time `pg_upgrade --link` takes (mostly schema dump/restore, which is **proportional to the number of objects**, not data size);
- the time the post-upgrade analyze takes.

---

## 4. Install binaries and initdb the new cluster (no downtime)

```bash
# root
dnf install -y postgresql18-server postgresql18-contrib   # + extension packages for 18 (SOP-05)

# postgres: match encoding/locale/provider/checksums of the old cluster
psql -X -At -c "select pg_encoding_to_char(encoding), datcollate, datlocprovider from pg_database where datname='template1'"
psql -X -At -c "show data_checksums"
$NEW_BIN/initdb -D "$NEW_DATA" --encoding=UTF8 --locale=en_US.UTF-8 --locale-provider=libc \
  --no-data-checksums      # ONLY if the old cluster has checksums off (PG18 initdb default = on)
```

Move the configuration over (SOP-01 section 5): tuned `postgresql.conf` values, `postgresql.auto.conf` contents, `pg_hba.conf`, `pg_ident.conf`, `shared_preload_libraries`.

---

## 5. Compatibility check (old cluster may be running)

```bash
cd $UPG_DIR
$NEW_BIN/pg_upgrade -b $OLD_BIN -B $NEW_BIN -d $OLD_DATA -D $NEW_DATA --link --jobs=8 --check
# -> *Clusters are compatible*
```

With `--link --check`, pg_upgrade also checks that hard links can be created between the two directories. Fix every reported issue (SOP-01 section 6 table) before the window.

---

## 6. Maintenance window

### 6.1 Freeze traffic and stop the old cluster

```bash
# pgbouncer: PAUSE; or stop app servers
psql -X -c "SELECT count(*) FROM pg_stat_activity WHERE backend_type='client backend' AND pid<>pg_backend_pid();"
psql -X -c "CHECKPOINT;"
sudo systemctl stop postgresql-17 && sudo systemctl disable postgresql-17
$OLD_BIN/pg_controldata -D $OLD_DATA | grep "cluster state"     # shut down
```

### 6.2 Last-chance safety copy of the old catalog (optional, cheap)

`global/` and the small catalog files are tiny. Keeping a copy makes forensic comparison easier:

```bash
tar czf $UPG_DIR/old_global_backup.tgz -C $OLD_DATA global PG_VERSION postgresql.conf postgresql.auto.conf pg_hba.conf
```

### 6.3 Upgrade

```bash
cd $UPG_DIR
time $NEW_BIN/pg_upgrade -b $OLD_BIN -B $NEW_BIN -d $OLD_DATA -D $NEW_DATA --link --jobs=8 2>&1 | tee pg_upgrade_link.log
```

Expected ending (wording varies by version):

```
Adding ".old" suffix to old global/pg_control               ok

If you want to start the old cluster, you will need to remove
the ".old" suffix from /var/lib/pgsql/17/data/global/pg_control.old.
Because "link" mode was used, the old cluster cannot be safely
started once the new cluster has been started.
...
Upgrade Complete
```

### 6.4 DECISION POINT: go / no-go **before** starting the new cluster

This is the last moment a cheap rollback is possible. Check:
- [ ] `pg_upgrade_link.log` ends with "Upgrade Complete" and has no errors.
- [ ] `ls -li $OLD_DATA/base/<db_oid>/<relfilenode> $NEW_DATA/base/<db_oid>/<relfilenode>` show the **same inode** (links were created).
- [ ] The new `postgresql.conf` / `pg_hba.conf` are in place.

**No-go → rollback without data loss** (only valid if the new cluster has NEVER been started):

```bash
mv $OLD_DATA/global/pg_control.old $OLD_DATA/global/pg_control
sudo systemctl enable --now postgresql-17
rm -rf $NEW_DATA        # discard the new cluster
```

### 6.5 Go: start the new cluster (point of no return)

```bash
sudo systemctl enable --now postgresql-18
psql -X -c "select version();"
```

From now on, the old cluster is invalid. Do not start it.

### 6.6 Statistics

```bash
$NEW_BIN/vacuumdb --all --analyze-in-stages --missing-stats-only -j 8
$NEW_BIN/vacuumdb --all --analyze-only -j 8     # if pg_upgrade suggested it
```

On PG18, the main optimizer statistics are preserved, so the application can usually return **before** this finishes. For targets ≤17, run `vacuumdb --all --analyze-in-stages` first: stage 1 is quick and makes plans usable.

---

## 7. Validation

Same as SOP-01 section 8: version, databases, extensions (`update_extensions.sql`), row-count fingerprints, application smoke test. Then:
- [ ] **Take a new full backup immediately.** Backups of the 17 cluster can't restore an 18 cluster. Until this backup completes, your only recovery path is the pre-upgrade backup **plus losing everything written since**.
- [ ] Rebuild standbys (SOP-04: the rsync method works **only** with link mode and must be done **before** the new primary starts. If you didn't plan for it, rebuild them with `pg_basebackup`).

---

## 8. Rollback after the new cluster has started

The old cluster can't be used. The options:
1. **Restore the pre-upgrade backup** onto the 17 binaries (data written since the upgrade is lost unless you re-apply it).
2. **Logical replication back to a 17 cluster** (set up in advance; see SOP-07 section 8: "reverse replication for rollback").
3. Fix forward on 18 (most common: most post-upgrade problems are configuration or statistics, not data).

---

## 9. Cleanup

```bash
# after the soak period and a new full backup:
cd $UPG_DIR && ./delete_old_cluster.sh
```

In link mode, `delete_old_cluster.sh` removes the old directory entries. Data blocks still linked from the new cluster are not freed, which is expected: the new cluster owns them now. `du` on the old directory **overstates** what you'll free, because hard-linked files are counted in both trees.

---

## 10. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `could not create hard link between old and new data directories: Invalid cross-device link` | Different filesystems. Move `$NEW_DATA` onto the same mount, or use `--copy` / `--clone` |
| Link mode took much longer than expected | Time goes into the schema dump/restore: many objects (100k+ tables, many large objects before PG17). Use `--jobs`; rehearse |
| Someone started the old cluster after the new one | Treat the old cluster as corrupt. Never use it. Check the new one with `amcheck` (SOP-10) and restore if in doubt |
| `pg_control.old` missing on rollback | It's only renamed in link mode; in copy mode the old cluster is untouched |
| `du` shows no space freed after deleting the old cluster | Expected (hard links). Freed space equals only the old catalog files |

---

## Appendix: verify links were created

```bash
DBOID=$(psql -X -At -c "select oid from pg_database where datname='appdb'")
REL=$(psql -X -At -d appdb -c "select pg_relation_filenode('accounts')")
ls -li $NEW_DATA/base/$DBOID/$REL $OLD_DATA/base/$DBOID/$REL* 2>/dev/null | head
# The inode numbers (first column) should match and the link count (third column) should be 2.
# Note: pg_upgrade preserves relfilenodes, so the file name is the same in old and new.
```
