# PostgreSQL DBA SOP pack: upgrades, logical replication, dump/restore, extensions

Prepared October 2026. Worked examples use **PostgreSQL 17 → 18** (18 is the current major release; 19 is in release-candidate stage, with GA scheduled for 19 Nov 2026). Version-specific behaviour is called out where it differs for older sources.

| # | SOP | Use it when |
|---|---|---|
| 01 | [pg_upgrade: copy mode](01-pg_upgrade-copy-mode.md) | You want the safest rollback (old cluster untouched) and can afford copy time |
| 02 | [pg_upgrade: link mode](02-pg_upgrade-link-mode.md) | Large clusters, downtime of minutes; rollback only before the new cluster first starts |
| 03 | [pg_upgrade: custom tablespaces](03-pg_upgrade-custom-tablespaces.md) | Tables/indexes live on other mount points |
| 04 | [pg_upgrade: physical standbys (copy & link)](04-pg_upgrade-physical-replicas.md) | Primary + streaming replicas; rsync method vs re-seed |
| 05 | [pg_upgrade: custom extensions](05-pg_upgrade-custom-extensions.md) | PostGIS, pg_partman, pg_cron, pgaudit, TimescaleDB, in-house C extensions |
| 06 | [pg_upgrade: logical replication nodes](06-pg_upgrade-logical-replication.md) | The cluster is a publisher and/or subscriber (slot/subscription migration, PG17+) |
| 07 | [Logical replication end to end + upgrade cutover](07-logical-replication-end-to-end.md) | Build, operate, work around limitations, then upgrade with near-zero downtime |
| 08 | [pg_dump / pg_restore with TOAST data](08-pg_dump-restore-toast.md) | Big text/jsonb/bytea values; 1 GB COPY limit; corrupt TOAST |
| 09 | [pg_dump / pg_restore with large objects](09-pg_dump-restore-large-objects.md) | Uploaded PDFs/text files stored as LOs |
| 10 | [Top 10 extensions](10-top-10-extensions.md) | Install, create, use and verify the extensions most teams need |

## Which upgrade method?

```
Need near-zero downtime or changing hardware/OS/layout? ── yes ──▶ SOP-07 (logical replication cutover)
        │ no
Old cluster must stay usable for rollback? ── yes ──▶ reflink FS? ── yes ──▶ pg_upgrade --clone (SOP-01/02)
        │ no                                          └── no ───▶ pg_upgrade --copy (SOP-01)
        ▼
pg_upgrade --link (SOP-02)   + tablespaces? SOP-03   + standbys? SOP-04   + extensions? SOP-05   + logical? SOP-06
```

## Conventions used in every SOP
- RHEL/Rocky 9 + PGDG paths (`/usr/pgsql-18/bin`, `/var/lib/pgsql/18/data`, service `postgresql-18`). Debian/Ubuntu equivalents are noted where they differ.
- Commands run as the `postgres` OS user unless marked `root`/`sudo`.
- Every SOP has: purpose, prerequisites, variables, pre-checks, procedure, validation, rollback, troubleshooting.

## How these SOPs were checked

| SOP | Verification |
|---|---|
| 07, 08, 09, 10 | **Executed end to end** on a test system (PostgreSQL 16 binaries plus PG16 builds of pg_partman 5.0, pg_cron 1.6, pgaudit 16, pg_repack 1.5, pgvector 0.6, PostGIS 3.4). Testing found and fixed several issues, recorded as gotchas in the SOPs: roles must be copied before the schema; a new table without `SELECT` for the replication user disables the **whole** subscription; `length()` doesn't detect missing TOAST chunks (use `md5()`); `-t` dumps omit schema and extensions; HNSW recall can miss isolated vectors |
| 01–06 | Mechanics verified with same-version `pg_upgrade --link` runs (check output, hard links sharing inodes, `pg_control.old` rollback). This caught one error before delivery: blocking logins with `ALLOW_CONNECTIONS false` makes pg_upgrade refuse to run (SOP-01 now uses pg_hba instead). 17→18-specific behaviour (statistics carry-over, `--swap`, checksums default, logical slot and subscription migration) is written from the PostgreSQL 17/18 documentation and release notes. **Rehearse in your environment before production use** |

Treat every SOP as a template: replace host names, versions, paths and timings with your own rehearsal results.
