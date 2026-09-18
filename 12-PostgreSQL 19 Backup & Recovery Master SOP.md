# Phase 12 — PostgreSQL 19 Backup & Recovery Master SOP

This phase establishes a **production-style backup and recovery strategy** for your RHEL 10 + PostgreSQL 19 lab.

The key principle is:

> **A backup is not successful until you can restore it and validate the recovered database.**

PostgreSQL's official documentation separates backup/recovery into SQL dumps, filesystem/base backups, and continuous WAL archiving with PITR. For enterprise recovery, the combination of a **base backup + continuous WAL archiving** is the foundation for point-in-time recovery. ([PostgreSQL][1])

> **PG19 status:** The official PG19 documentation currently identifies the version as unsupported/pre-release. The commands below should therefore be validated against the exact PG19 build in your lab before production use.

---

# 12.1 Backup Architecture

Recommended architecture:

```text
                         PostgreSQL 19
                              |
                    +---------+---------+
                    |                   |
                    v                   v
              Base Backups          WAL Archive
                    |                   |
                    |                   |
                    +---------+---------+
                              |
                              v
                       Backup Repository
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
          Local Disk      NFS/Object       DR Location
                          Storage
```

For production:

```text
PRIMARY
   |
   +---- WAL ---> Backup Repository
   |
   +---- Base Backup ---> Backup Repository
   |
   +---- Physical Standby
```

---

# 12.2 Four Backup Technologies You Need to Know

| Method          | Scope                            |              PITR | Typical Use             |
| --------------- | -------------------------------- | ----------------: | ----------------------- |
| `pg_dump`       | One database                     |                No | Logical/object recovery |
| `pg_dumpall`    | Cluster globals + DB definitions |                No | Roles/globals           |
| `pg_basebackup` | Entire cluster                   |      Yes with WAL | Physical backup         |
| pgBackRest      | Enterprise physical backup       |               Yes | Production              |
| WAL archive     | WAL changes                      | Required for PITR | Continuous recovery     |

`pg_dump` and `pg_dumpall` are **logical backups** and cannot by themselves participate in continuous WAL-based recovery. PostgreSQL explicitly distinguishes them from filesystem/base backups used with WAL archiving. ([PostgreSQL][2])

---

# 12.3 Backup Strategy I Recommend

For your DBA lab and production-oriented learning:

```text
                BACKUP STRATEGY
                      |
       +--------------+--------------+
       |                             |
 Logical Backup                 Physical Backup
       |                             |
 pg_dump/pg_dumpall              pgBackRest
                                     |
                         +-----------+-----------+
                         |                       |
                    Base Backup              WAL Archive
                         |                       |
                         +-----------+-----------+
                                     |
                                     v
                                    PITR
```

### Example schedule

```text
Daily
  |
  +-- Full backup

During day
  |
  +-- Continuous WAL archive

Weekly
  |
  +-- Restore validation

Monthly
  |
  +-- Full DR exercise
```

The exact schedule should be driven by:

* RPO
* RTO
* database size
* WAL generation rate
* backup window
* storage cost
* compliance requirements.

---

# 12.4 Phase 12A — PostgreSQL Native Backup

Before pgBackRest, understand PostgreSQL's native mechanisms.

Check:

```bash
/usr/pgsql-19/bin/pg_basebackup --version
```

```bash
/usr/pgsql-19/bin/pg_verifybackup --version
```

---

# 12.5 `pg_basebackup`

`pg_basebackup` takes a physical base backup of the **entire PostgreSQL cluster**. PostgreSQL 19 also exposes base-backup progress through `pg_stat_progress_basebackup`. ([PostgreSQL][3])

Example:

```bash
sudo mkdir -p /backup/pg_basebackup
sudo chown postgres:postgres /backup/pg_basebackup
```

Run:

```bash
sudo -u postgres /usr/pgsql-19/bin/pg_basebackup \
    -h 127.0.0.1 \
    -D /backup/pg_basebackup/full_$(date +%Y%m%d_%H%M%S) \
    -U repl_user \
    -Fp \
    -Xs \
    -P \
    -v
```

---

# 12.6 What Does `-Xs` Do?

```text
-Xs
```

means:

```text
stream WAL
```

while the base backup is being taken.

Conceptually:

```text
Database files
      +
WAL generated during backup
      |
      v
Consistent physical backup
```

---

# 12.7 Monitor Base Backup

On PostgreSQL:

```sql
SELECT
    pid,
    phase,
    backup_total,
    backup_streamed,
    tablespaces_total,
    tablespaces_streamed,
    backup_type
FROM pg_stat_progress_basebackup;
```

PostgreSQL 19 reports phases such as:

```text
initializing
waiting for checkpoint to finish
estimating backup size
streaming database files
waiting for WAL archiving to finish
transferring WAL files
```

The official documentation notes that `backup_total` is an estimate rather than an exact final size. ([PostgreSQL][3])

---

# 12.8 Backup Manifest

Modern PostgreSQL base backups can include:

```text
backup_manifest
```

Example:

```bash
ls -lh /backup/pg_basebackup/full_*/
```

You should see files/directories including:

```text
PG_VERSION
backup_label
backup_manifest
base/
global/
pg_wal/
```

The manifest is important for integrity verification.

---

# 12.9 Verify `pg_basebackup`

Run:

```bash
/usr/pgsql-19/bin/pg_verifybackup \
    /backup/pg_basebackup/full_YYYYMMDD_HHMMSS
```

Expected:

```text
backup successfully verified
```

PostgreSQL documents `pg_verifybackup` as checking the backup against the generated `backup_manifest`. However, PostgreSQL explicitly warns that this is **not equivalent to a real restore test**. ([PostgreSQL][4])

Therefore:

```text
pg_verifybackup
       ≠
Successful Recovery Test
```

---

# 12.10 Backup Validation Hierarchy

Use three levels:

```text
LEVEL 1
File exists
     ↓
LEVEL 2
pg_verifybackup
     ↓
LEVEL 3
Actual restore
     ↓
LEVEL 4
Application validation
```

Production standard:

```text
Backup PASS
+
Restore PASS
+
Application validation PASS
```

---

# 12.11 Phase 12B — WAL Archiving

WAL is central to PostgreSQL recovery.

PostgreSQL continuously writes WAL under:

```text
$PGDATA/pg_wal
```

WAL records changes needed for crash recovery and can also be archived for PITR. ([PostgreSQL][2])

Check:

```sql
SHOW archive_mode;
SHOW archive_command;
SHOW archive_timeout;
```

---

# 12.12 Enable WAL Archiving

Edit:

```bash
sudo vi /var/lib/pgsql/19/data/postgresql.conf
```

For a simple lab repository:

```conf
archive_mode = on
archive_command = 'test ! -f /backup/wal_archive/%f && cp %p /backup/wal_archive/%f'
```

Create directory:

```bash
sudo mkdir -p /backup/wal_archive
sudo chown postgres:postgres /backup/wal_archive
sudo chmod 700 /backup/wal_archive
```

Restart:

```bash
sudo systemctl restart postgresql-19
```

Validate:

```sql
SHOW archive_mode;
SHOW archive_command;
```

---

# 12.13 Test WAL Archiving

Generate WAL:

```sql
CREATE TABLE wal_test
(
    id BIGINT,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp()
);
```

```sql
INSERT INTO wal_test(id)
SELECT generate_series(1,10000);
```

Force WAL switch:

```sql
SELECT pg_switch_wal();
```

PostgreSQL documents `pg_switch_wal()` as forcing a WAL segment switch so the completed segment can be archived. ([PostgreSQL][5])

Check:

```bash
ls -lh /backup/wal_archive/
```

---

# 12.14 Check Archiver Health

```sql
SELECT
    archived_count,
    failed_count,
    last_archived_wal,
    last_archived_time,
    last_failed_wal,
    last_failed_time,
    stats_reset
FROM pg_stat_archiver;
```

### Healthy

```text
archived_count increasing
failed_count stable
last_archived_time recent
```

### Problem

```text
failed_count increasing
last_archived_time old
```

---

# 12.15 WAL Archive Failure RCA

If:

```text
failed_count > 0
```

investigate:

```bash
df -h
```

```bash
df -i
```

Permissions:

```bash
ls -ld /backup/wal_archive
```

Test as postgres:

```bash
sudo -u postgres touch /backup/wal_archive/test_file
```

Then:

```bash
sudo -u postgres rm /backup/wal_archive/test_file
```

Check PostgreSQL logs:

```bash
sudo journalctl -u postgresql-19 --since "30 minutes ago"
```

---

# 12.16 Critical Warning — `archive_command`

Never use a command that silently overwrites or loses WAL.

Bad:

```conf
archive_command = 'cp %p /backup/wal_archive/%f'
```

if your process does not safely handle duplicate WAL filenames.

A safer simple lab example is:

```conf
archive_command = 'test ! -f /backup/wal_archive/%f && cp %p /backup/wal_archive/%f'
```

For production, use a dedicated backup system such as pgBackRest rather than building a large backup framework from shell commands.

---

# 12.17 Phase 12C — pgBackRest Architecture

For your lab:

```text
PostgreSQL 19
     |
     | pgBackRest
     v
/backup/pgbackrest
     |
     +-- repo1
     |
     +-- backups
     |
     +-- archive
```

Production:

```text
PostgreSQL
     |
 pgBackRest
     |
     +------ Local Repository
     |
     +------ Object Storage
     |
     +------ DR Repository
```

This provides a much stronger operational model than ad-hoc `cp` scripts.

---

# 12.18 pgBackRest Configuration

Example:

```bash
sudo mkdir -p /etc/pgbackrest
sudo mkdir -p /var/log/pgbackrest
sudo mkdir -p /var/lib/pgbackrest

sudo chown -R postgres:postgres /var/log/pgbackrest
sudo chown -R postgres:postgres /var/lib/pgbackrest
```

Create:

```bash
sudo vi /etc/pgbackrest/pgbackrest.conf
```

Example lab configuration:

```ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-diff=4
start-fast=y
log-level-console=info
log-level-file=detail

[main]
pg1-path=/var/lib/pgsql/19/data
```

---

# 12.19 Configure PostgreSQL for pgBackRest

Use:

```conf
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
```

Restart:

```bash
sudo systemctl restart postgresql-19
```

---

# 12.20 Create pgBackRest Stanza

```bash
sudo -u postgres pgbackrest \
    --stanza=main \
    stanza-create
```

Then:

```bash
sudo -u postgres pgbackrest \
    --stanza=main \
    check
```

Expected:

```text
P00   INFO: check command end: completed successfully
```

---

# 12.21 pgBackRest `check`

`check` validates the backup configuration and WAL archiving path.

Run regularly:

```bash
sudo -u postgres pgbackrest \
    --stanza=main \
    check
```

This should be part of monitoring.

---

# 12.22 Full Backup

```bash
sudo -u postgres pgbackrest \
    --stanza=main \
    --type=full \
    backup
```

Check:

```bash
sudo -u postgres pgbackrest \
    --stanza=main \
    info
```

---

# 12.23 Differential Backup

```bash
sudo -u postgres pgbackrest \
    --stanza=main \
    --type=diff \
    backup
```

Differential backup means:

```text
Latest FULL
     +
Changes since FULL
```

---

# 12.24 Incremental Backup

```bash
sudo -u postgres pgbackrest \
    --stanza=main \
    --type=incr \
    backup
```

Conceptually:

```text
FULL
 |
 +--- DIFF
 |
 +--- INCR
 |
 +--- INCR
```

The backup chain must be managed correctly; don't delete individual files manually.

---

# 12.25 Recommended Lab Schedule

```text
Sunday
  FULL

Monday
  INCR

Tuesday
  INCR

Wednesday
  DIFF

Thursday
  INCR

Friday
  INCR

Saturday
  DIFF
```

This is only an example. Choose the actual schedule from:

```text
RPO
RTO
backup duration
WAL volume
storage
business requirements
```

---

# 12.26 Backup Retention

Example:

```ini
repo1-retention-full=2
repo1-retention-diff=4
```

Do not manually delete:

```bash
rm -rf /var/lib/pgbackrest/*
```

Use pgBackRest:

```bash
sudo -u postgres pgbackrest \
    --stanza=main \
    expire
```

The backup repository must remain internally consistent.

---

# 12.27 Backup Encryption

For production backup repositories, consider repository encryption.

Architecture:

```text
PostgreSQL
     |
     v
pgBackRest
     |
     v
Encrypted Repository
     |
     v
Object Storage / DR
```

Protect the encryption key separately from the backup repository.

### Critical rule

```text
Backup + encryption key
```

must be recoverable together.

If:

```text
Backup exists
BUT
encryption key is lost
```

then:

```text
BACKUP = UNRECOVERABLE
```

---

# 12.28 PITR Architecture

Point-in-time recovery:

```text
FULL BACKUP
     |
     + WAL
     + WAL
     + WAL
     + WAL
     |
     v
Recovery Target
     |
     v
Recovered Database
```

Example:

```text
09:00 Full Backup
10:00 Data inserted
10:30 Data inserted
11:00 Accidental DELETE
11:30 DBA discovers issue
```

You can recover to:

```text
10:59:59
```

rather than restoring only the 09:00 backup.

PostgreSQL's continuous-archiving model is specifically designed for this use case. ([PostgreSQL][2])

---

# 12.29 Create a Recovery Marker

Before a planned change:

```sql
SELECT pg_create_restore_point('before_major_change');
```

Record the returned LSN.

PostgreSQL documents `pg_create_restore_point()` as creating a named WAL marker that can later be used with `recovery_target_name`. ([PostgreSQL][5])

---

# 12.30 PITR Test Scenario

Create:

```sql
CREATE TABLE recovery_test
(
    id INTEGER PRIMARY KEY,
    message TEXT,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp()
);
```

Insert:

```sql
INSERT INTO recovery_test(id,message)
VALUES
(1,'Before incident');
```

Create restore point:

```sql
SELECT pg_create_restore_point('before_incident');
```

Then simulate accidental data:

```sql
INSERT INTO recovery_test(id,message)
VALUES
(2,'Bad data'),
(3,'Bad data');
```

Now simulate accidental deletion:

```sql
DELETE FROM recovery_test;
```

---

# 12.31 PITR Recovery Target

Restore the base backup to a separate recovery server.

Create:

```bash
touch /var/lib/pgsql/19/data/recovery.signal
```

Configure:

```conf
restore_command = 'pgbackrest --stanza=main archive-get %f "%p"'
recovery_target_name = 'before_incident'
recovery_target_action = 'pause'
```

PostgreSQL uses `recovery.signal` for targeted recovery, while `standby.signal` is used for continuous standby mode. If both exist, standby mode takes precedence. ([PostgreSQL][6])

---

# 12.32 Start Recovery

```bash
sudo systemctl start postgresql-19
```

Monitor:

```bash
sudo journalctl -u postgresql-19 -f
```

Validate:

```sql
SELECT pg_is_in_recovery();
```

Check:

```sql
SELECT *
FROM recovery_test
ORDER BY id;
```

Expected:

```text
1 | Before incident
```

and the bad records should not be present if recovery stopped at the named restore point.

---

# 12.33 Recovery Target Types

PostgreSQL supports recovery targets including:

```text
recovery_target_time
recovery_target_lsn
recovery_target_name
recovery_target_xid
```

You can also control whether recovery stops before or after the matching transaction with:

```text
recovery_target_inclusive
```

and what PostgreSQL does after reaching the target with:

```text
recovery_target_action
```

The official PostgreSQL 19 recovery documentation covers these targeted-recovery parameters. ([PostgreSQL][6])

---

# 12.34 PITR by Timestamp

Example:

```conf
recovery_target_time = '2026-09-17 14:30:00-05'
recovery_target_action = 'pause'
```

This is useful when the exact WAL restore point isn't known.

However, timestamp-based recovery requires careful timezone handling.

Always use:

```text
UTC or explicit timezone
```

rather than relying on ambiguous local timestamps.

---

# 12.35 PITR by LSN

Example:

```conf
recovery_target_lsn = '0/50000A0'
recovery_target_action = 'pause'
```

This is more deterministic when you have captured the exact LSN.

---

# 12.36 PITR by Named Restore Point

Preferred for planned operations:

```sql
SELECT pg_create_restore_point('before_release_20260917');
```

Then:

```conf
recovery_target_name = 'before_release_20260917'
```

This is especially useful for:

```text
application deployments
schema migrations
major upgrades
batch processing
data corrections
```

---

# 12.37 Important PITR Limitation

PITR restores the **entire cluster**, not a single table.

PostgreSQL's continuous archive recovery is a cluster-level recovery mechanism. ([PostgreSQL][2])

If the requirement is:

> "Recover only one dropped table."

Typical approach:

```text
PITR to temporary server
        |
        v
Recover table
        |
        v
pg_dump table
        |
        v
Restore table to production
```

---

# 12.38 Table-Level Recovery

Architecture:

```text
PRODUCTION
    |
    | PITR
    v
TEMPORARY RECOVERY SERVER
    |
    | pg_dump
    v
table.sql
    |
    v
PRODUCTION
```

Example:

```bash
pg_dump \
    -h recovery-server \
    -U postgres \
    -d appdb \
    -t public.customers \
    -Fc \
    -f customers.dump
```

Restore:

```bash
pg_restore \
    -h production-server \
    -U postgres \
    -d appdb \
    customers.dump
```

This is much safer than trying to manipulate production data files manually.

---

# 12.39 `pg_dump` Backup

For logical database backup:

```bash
pg_dump \
    -h 127.0.0.1 \
    -U postgres \
    -d appdb \
    -Fc \
    -f /backup/appdb_$(date +%Y%m%d).dump
```

Check:

```bash
ls -lh /backup/*.dump
```

---

# 12.40 Logical Restore

Create database:

```sql
CREATE DATABASE appdb_restore;
```

Restore:

```bash
pg_restore \
    -h 127.0.0.1 \
    -U postgres \
    -d appdb_restore \
    /backup/appdb_YYYYMMDD.dump
```

Validate:

```sql
SELECT count(*)
FROM public.customers;
```

---

# 12.41 Cluster Globals Backup

`pg_dump` doesn't capture all cluster-wide objects such as roles.

Use:

```bash
pg_dumpall \
    -h 127.0.0.1 \
    -U postgres \
    --globals-only \
    > /backup/globals_$(date +%Y%m%d).sql
```

This protects:

```text
roles
role memberships
tablespace definitions
other cluster-level global objects
```

---

# 12.42 Disaster Recovery Hierarchy

When something goes wrong:

```text
What happened?
      |
      +-- Single object?
      |       |
      |       v
      |    Logical restore
      |
      +-- Database corruption?
      |       |
      |       v
      |    PITR temporary server
      |
      +-- Server loss?
      |       |
      |       v
      |    Restore base backup + WAL
      |
      +-- Site loss?
              |
              v
           DR standby
```

---

# 12.43 Backup Monitoring

Monitor:

```text
Backup success
Backup duration
Backup size
WAL archive success
WAL archive failures
Repository storage
Retention
Last successful backup
Last successful restore test
RPO
RTO
```

Example:

```sql
SELECT
    archived_count,
    failed_count,
    last_archived_wal,
    last_archived_time,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;
```

---

# 12.44 Linux Backup Monitoring

```bash
df -h
```

```bash
df -i
```

```bash
du -sh /var/lib/pgbackrest
```

```bash
du -sh /backup/*
```

Check PostgreSQL:

```bash
systemctl status postgresql-19
```

Check recent errors:

```bash
journalctl -u postgresql-19 --since "1 hour ago"
```

---

# 12.45 Backup Health Dashboard

Build these status categories:

```text
BACKUP STATUS
   |
   +-- Last Full Backup
   +-- Last Diff Backup
   +-- Last Incremental Backup
   +-- Last Successful WAL Archive
   +-- Last Failed WAL Archive
   +-- Repository Usage
   +-- Backup Verification
   +-- Last Restore Test
```

Example:

```text
FULL BACKUP           PASS
DIFF BACKUP           PASS
INCR BACKUP           PASS
WAL ARCHIVE           PASS
REPOSITORY            62%
VERIFY                PASS
RESTORE TEST          PASS
RPO                    5 min
RTO                   18 min
```

---

# 12.46 Backup RCA — Common Failures

| Symptom                        | Likely Cause                | Investigation             |
| ------------------------------ | --------------------------- | ------------------------- |
| Backup fails                   | Disk/storage                | `df -h`                   |
| WAL archive failing            | Permissions                 | `pg_stat_archiver`        |
| Repository full                | Retention/storage           | pgBackRest `info`         |
| Restore fails                  | Missing WAL                 | Archive verification      |
| PITR stops early               | WAL gap                     | Archive chain             |
| `pg_verifybackup` fails        | Backup corruption           | Manifest/checksum         |
| Restore succeeds but app fails | Missing roles/extensions    | Application validation    |
| Backup is slow                 | I/O/network                 | OS + pgBackRest logs      |
| WAL grows rapidly              | Slot/archive/consumer issue | replication slots         |
| Encryption restore fails       | Missing key                 | Key-management validation |

---

# 12.47 Production Backup Policy

A practical policy:

```text
FULL:
Weekly

DIFFERENTIAL:
Daily

INCREMENTAL:
Multiple times/day where required

WAL:
Continuous

Logical:
Daily/weekly depending on recovery requirements

Restore test:
Weekly

Full DR exercise:
Monthly/quarterly
```

But do not copy this schedule blindly.

For example:

```text
High-volume OLTP
      ↓
WAL generation = high
      ↓
Need frequent base backups
or efficient incremental strategy
```

---

# 12.48 3-2-1 Backup Principle

Use:

```text
3 copies
2 different storage/media types
1 off-site copy
```

Example:

```text
Copy 1 → PostgreSQL backup repository
Copy 2 → Object storage
Copy 3 → DR region/site
```

Do not treat:

```text
Primary server + backup directory on same disk
```

as a complete disaster recovery strategy.

---

# 12.49 Backup Security

Protect:

```text
Backup repository
WAL archive
Encryption keys
Database credentials
pgBackRest configuration
Object storage credentials
```

Permissions:

```bash
chmod 700 /var/lib/pgbackrest
```

Do not expose backup repositories over unrestricted network access.

Backups frequently contain the **entire database**, so backup security is effectively database security.

---

# 12.50 Restore Validation Checklist

After every major restore:

```text
[ ] PostgreSQL starts
[ ] Database exists
[ ] Roles exist
[ ] Extensions exist
[ ] Tables exist
[ ] Indexes exist
[ ] Constraints exist
[ ] Row counts match
[ ] Application connects
[ ] Application reads work
[ ] Application writes work
[ ] Sequences validated
[ ] Permissions validated
[ ] PostgreSQL logs clean
[ ] Backup source documented
[ ] Recovery target documented
[ ] Recovery duration recorded
```

---

# 12.51 The Most Important Backup Test

Do this regularly:

```text
TAKE BACKUP
     |
     v
VERIFY BACKUP
     |
     v
DELETE/ISOLATE TEST SERVER
     |
     v
RESTORE
     |
     v
START POSTGRESQL
     |
     v
VALIDATE DATABASE
     |
     v
VALIDATE APPLICATION
     |
     v
RECORD RTO
```

That produces actual evidence that your backup strategy works.

---

# 12.52 Phase 12 Complete Recovery Architecture

```text
                         APPLICATION
                              |
                              v
                       PG19 PRIMARY
                              |
             +----------------+----------------+
             |                                 |
             v                                 v
       Physical HA                       WAL Archive
             |                                 |
             v                                 v
        STANDBY / DR                  pgBackRest Repository
                                               |
                              +----------------+----------------+
                              |                |                |
                              v                v                v
                            FULL             DIFF             INCR
                              \                |                /
                               \               |               /
                                +--------------+--------------+
                                               |
                                               v
                                            PITR
                                               |
                              +----------------+----------------+
                              |                                 |
                              v                                 v
                       Full Cluster                       Object/Table
                         Recovery                           Recovery
                                                              |
                                                              v
                                                         pg_dump /
                                                         pg_restore
```

## Phase 12 — DBA Golden Rules

1. **`pg_dump` is not a replacement for physical/PITR backups.**
2. **`pg_basebackup` is a cluster-level physical backup.**
3. **WAL archiving is required for continuous PITR.**
4. **A replication standby is not a substitute for an independent backup.**
5. **A backup that has never been restored is an assumption, not evidence.**
6. **Monitor `pg_stat_archiver`; don't discover WAL-archive failure during an outage.**
7. **Never manually delete pgBackRest repository files.**
8. **Keep backup credentials and encryption keys separately protected.**
9. **Test PITR to a separate recovery server before attempting production recovery.**
10. **Measure RPO and RTO during actual recovery drills.**

PostgreSQL's own documentation makes the same fundamental distinction: `pg_verifybackup` checks the integrity of a base backup, but **only a test restore can establish that the backup can actually be used successfully**. ([PostgreSQL][4])

### Official references

* [PostgreSQL 19 Backup and Restore](https://www.postgresql.org/docs/19/backup.html?utm_source=chatgpt.com)
* [PostgreSQL 19 Continuous Archiving and PITR](https://www.postgresql.org/docs/19/continuous-archiving.html?utm_source=chatgpt.com)
* [PostgreSQL 19 pg_basebackup](https://www.postgresql.org/docs/19/app-pgbasebackup.html?utm_source=chatgpt.com)
* [PostgreSQL 19 pg_verifybackup](https://www.postgresql.org/docs/19/app-pgverifybackup.html?utm_source=chatgpt.com)
* [PostgreSQL 19 Backup Control Functions](https://www.postgresql.org/docs/19/functions-admin.html?utm_source=chatgpt.com)

**Next: Phase 13 — PostgreSQL 19 Performance Tuning Master SOP: `EXPLAIN (ANALYZE, BUFFERS, WAL)`, `pg_stat_statements`, `pg_stat_io`, memory/work_mem, indexes, vacuum/autovacuum, bloat, locks/waits, I/O tuning, CPU analysis, parallelism, and end-to-end performance RCA.**

[1]: https://www.postgresql.org/docs/19/admin.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: Part III. Server Administration"
[2]: https://www.postgresql.org/docs/17/continuous-archiving.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 17: 25.3. Continuous Archiving and Point-in-Time Recovery (PITR)"
[3]: https://www.postgresql.org/docs/19/progress-reporting.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 27.4. Progress Reporting"
[4]: https://www.postgresql.org/docs/19/app-pgverifybackup.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: pg_verifybackup"
[5]: https://www.postgresql.org/docs/19/functions-admin.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 9.29. System Administration Functions"
[6]: https://www.postgresql.org/docs/19/runtime-config-wal.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 19.5. Write Ahead Log"
