# SOP-04: `pg_upgrade` when the primary has physical (streaming) standbys

| Item | Value |
|---|---|
| Purpose | Upgrade a primary + streaming standby(s) to a new major version |
| Worked example | PG **17 → 18**. `pg-primary` (10.0.0.11) and `pg-standby1` (10.0.0.12), streaming replication, optional tablespaces |
| Methods | **A. Link mode + rsync** (standbys "upgraded" in minutes, no full re-copy). **B. Copy mode + re-seed** standbys with `pg_basebackup` |
| Fact to remember | Standbys **can't** be upgraded by running `pg_upgrade` on them. A physical standby must be a block copy of the new primary |

---

## 1. Choosing the method

| | A. `--link` + rsync | B. `--copy` + pg_basebackup |
|---|---|---|
| Primary downtime | Minutes | Copy time (or minutes with `--clone`) |
| Time until standbys are back | Minutes (rsync sends only new catalog files plus hard-link metadata) | Full base backup time (network bound) |
| Network transfer to standby | Small | Entire database |
| Complexity / risk | Higher: strict ordering, both sides **must** be stopped at the same LSN | Low: standard rebuild |
| Rollback | Primary: only before the new primary starts. Standby: keep one standby **un-synced** as a fallback copy | Old primary intact (copy mode); old standbys can be kept until the new ones are seeded |

Rule of thumb: **< 500 GB or a fast network → B**. **Multi-TB → A** (rehearse it).

A practical safety net for both methods: **leave one old standby completely untouched** (stopped, not rsynced) until the new cluster is validated. It's your rollback copy at the old version.

---

## 2. Common variables (set on all hosts)

```bash
export OLD_BIN=/usr/pgsql-17/bin NEW_BIN=/usr/pgsql-18/bin
export OLD_DATA=/var/lib/pgsql/17/data NEW_DATA=/var/lib/pgsql/18/data
export PRIMARY=10.0.0.11 STANDBY=10.0.0.12
export REPL_USER=replicator
```

The `postgres` user on the primary needs passwordless SSH to the standby (for rsync).

---

## 3. Preparation (all hosts, before the window)

1. Install the **same** PostgreSQL 18 packages, and the same extension packages, on every host (`dnf install -y postgresql18-server postgresql18-contrib ...`).
2. On the **primary only**: `initdb` the new cluster (match locale/encoding/checksums, SOP-01 section 5), carry the configuration over, and run `pg_upgrade --check` (SOP-01 section 6).
   - New `postgresql.conf` must keep `wal_level = replica` (or logical), enough `max_wal_senders` / `max_replication_slots`, and `hot_standby = on`.
   - New `pg_hba.conf` must allow `replication` connections from the standbys.
3. On the **standbys**: do **not** run initdb (method A creates the directory via rsync; method B via pg_basebackup).
4. Check replication is healthy and caught up:

```sql
-- on primary
SELECT application_name, client_addr, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
SELECT slot_name, slot_type, active FROM pg_replication_slots;   -- physical slots must be recreated after upgrade
```

5. Record each standby's config: `primary_conninfo`, `primary_slot_name`, `recovery_min_apply_delay`, `restore_command`.

---

## 4. Method A: link mode + rsync (official fast procedure)

The order matters. Read the whole section before starting.

### 4.1 Stop the applications, then the primary

```bash
# primary
psql -X -c "CHECKPOINT;"
sudo systemctl stop postgresql-17
$OLD_BIN/pg_controldata -D $OLD_DATA | grep -E "cluster state|Latest checkpoint location"
#   Database cluster state:               shut down
#   Latest checkpoint location:           3/9A000028        <- note it
```

### 4.2 Let the standby catch up, then stop it

The standby must have received **and replayed** the primary's shutdown checkpoint.

```bash
# standby: wait a few seconds after the primary is down, then
sudo systemctl stop postgresql-17
$OLD_BIN/pg_controldata -D $OLD_DATA | grep -E "cluster state|Latest checkpoint location"
#   Database cluster state:               shut down in recovery
#   Latest checkpoint location:           3/9A000028        <- MUST equal the primary's value
```

If the LSNs differ, **stop**: start the old standby again, let it catch up (the old primary must be running for that, so restart it too), and repeat 4.1–4.2. Rsyncing a standby that isn't at the same point produces a corrupt standby.

Also check `wal_level` was not `minimal` on the old primary (impossible with standbys, but check `pg_controldata | grep wal_level`).

### 4.3 Upgrade the primary in link mode (do NOT start it)

```bash
# primary
cd /var/lib/pgsql
$NEW_BIN/pg_upgrade -b $OLD_BIN -B $NEW_BIN -d $OLD_DATA -D $NEW_DATA --link --jobs=4 | tee upgrade.log
```

**Do not start the new primary yet.** Once it starts, it writes WAL and changes files, and the standby copy would no longer match.

### 4.4 Rsync to each standby (from the primary, as postgres)

Run from the directory that **contains** both the old and new data directories, so rsync sees both trees and can recreate the hard links on the standby:

```bash
# primary, as postgres, with $OLD_DATA=/var/lib/pgsql/17/data and $NEW_DATA=/var/lib/pgsql/18/data
cd /var/lib/pgsql
rsync --archive --delete --hard-links --size-only --no-inc-recursive \
      /var/lib/pgsql/17 /var/lib/pgsql/18 \
      $STANDBY:/var/lib/pgsql
```

What the flags do:
- `--hard-links`: preserves the link structure, so on the standby `18/data/...` becomes a hard link to the standby's **existing** `17/data/...` file; data isn't re-sent.
- `--size-only`: files of the same size are assumed identical (true for the shared data files, because they're the same files). **That's why the standby must be at exactly the same checkpoint.**
- `--no-inc-recursive`: rsync needs the whole file list up front to detect hard links across the two trees.
- `--delete`: removes files on the standby that are not on the primary.

**Tablespaces:** repeat for each tablespace location, syncing both version subdirectories:

```bash
# for /pgdata_ts/data (PG_17_* and PG_18_* inside)
rsync --archive --delete --hard-links --size-only --no-inc-recursive \
      /pgdata_ts/data/PG_17_202406281 /pgdata_ts/data/PG_18_202506291 \
      $STANDBY:/pgdata_ts/data
```

**pg_wal on another disk (symlink):** the new `pg_wal` is a fresh directory, and rsync copies it as-is. If you keep WAL on a separate mount, recreate the symlink layout on the standby to match the primary (rsync copies the symlink itself).

**Config files outside the data directory** (Debian `/etc/postgresql/...`): copy them separately.

### 4.5 Configure the standby (before starting anything)

```bash
# standby, as postgres
touch $NEW_DATA/standby.signal
cat >> $NEW_DATA/postgresql.auto.conf <<EOF
primary_conninfo = 'host=$PRIMARY port=5432 user=$REPL_USER application_name=pg-standby1'
primary_slot_name = 'standby1_slot'
EOF
```

(The data directory came from the primary, so `postgresql.conf` is the primary's. Adjust anything host-specific, for example `listen_addresses`.)

### 4.6 Start the new primary, create slots, then start the standbys

```bash
# primary
sudo systemctl enable --now postgresql-18
psql -X -c "SELECT pg_create_physical_replication_slot('standby1_slot');"   # slots are NOT migrated by pg_upgrade
# standby
sudo systemctl disable postgresql-17
sudo systemctl enable --now postgresql-18
```

### 4.7 Validate replication

```sql
-- primary
SELECT application_name, state, sync_state, pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag
FROM pg_stat_replication;                                 -- state = streaming
-- standby
SELECT pg_is_in_recovery(), pg_last_wal_replay_lsn(), now() - pg_last_xact_replay_timestamp() AS delay;
```

Functional test: `CREATE TABLE upg_repl_test AS SELECT 1 AS x;` on the primary, then `SELECT * FROM upg_repl_test;` on the standby, then drop it.

Then: statistics (`vacuumdb --all --analyze-in-stages --missing-stats-only`) on the primary (they replicate), a new **full backup**, and cleanup (`delete_old_cluster.sh` on the primary; `rm -rf /var/lib/pgsql/17` on standbys after the soak period).

---

## 5. Method B: copy mode + re-seed standbys with pg_basebackup

### 5.1 Upgrade the primary

Follow SOP-01 (copy) or SOP-02 (link) for the primary. Old standbys can simply be **stopped and kept as-is** as a fallback.

```bash
# standbys first (optional: keep them running until the primary is stopped, then stop them)
sudo systemctl stop postgresql-17      # on each standby
# primary
sudo systemctl stop postgresql-17
$NEW_BIN/pg_upgrade -b $OLD_BIN -B $NEW_BIN -d $OLD_DATA -D $NEW_DATA --copy --jobs=4
sudo systemctl enable --now postgresql-18
$NEW_BIN/vacuumdb --all --analyze-in-stages --missing-stats-only -j 4
```

The application can go live on the primary now, before the standbys are back (decide whether running without HA for a while is acceptable).

### 5.2 Seed each standby

```bash
# standby, as postgres. NEW_DATA must not exist or must be empty
$NEW_BIN/pg_basebackup -h $PRIMARY -U $REPL_USER -D $NEW_DATA \
  -X stream -R -C -S standby1_slot --checkpoint=fast -P -v
```

- `-R` writes `standby.signal` + `primary_conninfo`; `-C -S` creates the physical slot on the primary.
- **Tablespaces:** pg_basebackup restores each tablespace to the **same path** as on the primary and needs that location to be **empty** (or missing). The old `PG_17_*` dir is still there, so either:
  - map to new, empty paths: `-T /pgdata_ts/data=/pgdata_ts/data18 -T /pgdata_ts/index=/pgdata_ts/index18`, or
  - move the old content aside first: `mv /pgdata_ts/data /pgdata_ts/data_old17 && mkdir -m 700 /pgdata_ts/data`.

```bash
sudo systemctl disable postgresql-17
sudo systemctl enable --now postgresql-18
```

Validate as in 4.7.

### 5.3 Speed-ups for big databases
- `--compress=server-zstd` (PG15+) when the network is the bottleneck.
- Seed one standby, then seed the others **from that standby** (`-h <standby1>`), which takes load off the primary.
- Use your backup tool's restore (pgBackRest `--type=standby`) from a backup taken **after** the upgrade.

---

## 6. Cascading standbys, delayed standbys, archive

- **Cascading** (standby2 streams from standby1): Method A, rsync from the primary to all of them, then point `primary_conninfo` at the upstream standby. Method B: seed from the upstream.
- **Delayed standby** (`recovery_min_apply_delay`): it's never at the same LSN as the primary, so it **can't** use Method A. Re-seed it (Method B).
- **WAL archive / PITR:** the new cluster starts a new timeline history and a new system identifier. Configure a **new** archive location or stanza (`pgbackrest stanza-upgrade` for pgBackRest), and take a full backup right away.
- **Patroni / repmgr:** use the tool's major-upgrade procedure (Patroni: pause the cluster, run this SOP per node, update DCS config, resume). Don't let an HA manager restart old nodes mid-upgrade.

---

## 7. Rollback

| Situation | Rollback |
|---|---|
| Method A, new primary **not started** | Primary: rename `pg_control.old` → `pg_control`, start 17. Rsynced standbys: **do not reuse their 17 tree**. rsync `--delete` made it a mirror of the primary's (including `pg_control.old`). Re-seed them, or use the untouched spare standby |
| Method A, new primary started | Restore from backup, or promote the **untouched spare 17 standby** (the data loss is everything written since the upgrade) |
| Method B | Stop 18, start the old 17 primary and the untouched old standbys |

---

## 8. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Standby on 18 fails: `requested timeline ... is not a child of this server's history` or `database system identifier differs` | The standby wasn't rsynced from this exact upgrade, or the primary started before the rsync. Re-seed with pg_basebackup |
| rsync transfers the entire database | `--hard-links`/`--no-inc-recursive` missing, rsync run from the wrong directory (both trees must be in the same command), or the standby has different paths |
| Standby: `could not open file pg_tblspc/...` | A tablespace wasn't rsynced. Run the tablespace rsync, or re-seed |
| `replication slot "standby1_slot" does not exist` | Physical slots aren't migrated. Create them on the new primary |
| Checkpoint LSNs differ between primary and standby | Don't continue with Method A. Restart both on 17, let the standby catch up, repeat, or use Method B |
