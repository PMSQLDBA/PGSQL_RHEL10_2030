# Phase 17 — PostgreSQL 19 Upgrades, Patching & Version Migration

**Important current-status correction:** As of **September 17, 2026**, PostgreSQL 19 is still in **beta**; PostgreSQL's official release archive shows PostgreSQL 19 Beta 3 released August 13, 2026, while PostgreSQL 18 is the current stable major release. Therefore, the procedures below are the **PostgreSQL 19 documented upgrade procedures**, but PostgreSQL 19 should not be treated as a production-stable release yet. ([PostgreSQL][1])

[PostgreSQL 19 official documentation](https://www.postgresql.org/docs/19/?utm_source=chatgpt.com)

---

## 17.1 PostgreSQL Upgrade Types

There are **two fundamentally different upgrade operations**:

```text
                 PostgreSQL Upgrade
                         |
              +----------+----------+
              |                     |
              v                     v
         Minor Upgrade         Major Upgrade
              |                     |
        18.5 → 18.6            17 → 18
        18.6 → 18.7            18 → 19
              |                     |
         Usually binary        Requires migration
         replacement           mechanism
```

### Minor release

Example:

```text
PostgreSQL 18.5
      ↓
PostgreSQL 18.6
```

Minor releases primarily contain bug/security fixes and do **not require `pg_upgrade`**. ([PostgreSQL][2])

### Major release

Example:

```text
PostgreSQL 18
      ↓
PostgreSQL 19
```

Requires a major-version upgrade procedure such as:

```text
pg_upgrade
pg_dump / pg_restore
logical replication
```

PostgreSQL's official documentation specifically identifies these as major upgrade approaches. ([PostgreSQL][3])

---

# 17.2 DBA Upgrade Decision Matrix

| Requirement                          | Recommended approach                  |
| ------------------------------------ | ------------------------------------- |
| Minor patch                          | Package/binary update                 |
| Major upgrade, low downtime          | `pg_upgrade`                          |
| Major upgrade, migration flexibility | Logical replication                   |
| Small database                       | Dump/restore                          |
| Very large database                  | `pg_upgrade`                          |
| Cross-platform migration             | Dump/restore or logical replication   |
| Cross-architecture migration         | Carefully evaluate method             |
| HA environment                       | Upgrade topology-aware                |
| Need rollback after cutover          | Keep old environment until validation |

---

# 17.3 PostgreSQL 18 → 19

For your RHEL 10 lab:

```text
RHEL 10
   |
   +----------------+
   |                |
PostgreSQL 18   PostgreSQL 19
   |                |
 OLD CLUSTER     NEW CLUSTER
```

You should **not overwrite the PostgreSQL 18 data directory**.

Instead:

```text
18 binaries + 18 data
          |
          | pg_upgrade
          v
19 binaries + 19 data
```

---

# 17.4 Pre-Upgrade Assessment

Before touching production:

```text
[ ] PostgreSQL version
[ ] RHEL version
[ ] CPU architecture
[ ] Disk space
[ ] Database sizes
[ ] Tablespaces
[ ] Extensions
[ ] Replication
[ ] Replication slots
[ ] Backups
[ ] WAL archiving
[ ] Application compatibility
[ ] Driver compatibility
[ ] Authentication
[ ] TLS
[ ] Custom configuration
[ ] Cron/systemd jobs
[ ] Monitoring
```

---

# 17.5 Capture PostgreSQL Version

```sql
SELECT version();
```

And:

```sql
SHOW server_version;
```

Also:

```sql
SELECT
    current_setting('server_version') AS server_version,
    current_setting('data_directory') AS data_directory,
    current_setting('config_file') AS config_file;
```

---

# 17.6 Capture Database Inventory

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS database_size
FROM pg_database
WHERE datallowconn
ORDER BY pg_database_size(datname) DESC;
```

This tells you where the migration effort is concentrated.

---

# 17.7 Capture Tablespaces

```sql
SELECT
    spcname,
    pg_tablespace_location(oid)
FROM pg_tablespace;
```

Do not forget tablespaces.

A common upgrade failure is:

```text
PGDATA migrated
       |
       X
tablespace forgotten
```

---

# 17.8 Capture Extensions

```sql
SELECT
    extname,
    extversion
FROM pg_extension
ORDER BY extname;
```

Then verify that the required extension libraries are available for PostgreSQL 19.

`pg_upgrade` can check many compatibility issues, but external modules/shared libraries require administrator validation. ([PostgreSQL][2])

---

# 17.9 Capture Roles

```sql
SELECT
    rolname,
    rolsuper,
    rolcreaterole,
    rolcreatedb,
    rolreplication,
    rolcanlogin
FROM pg_roles
ORDER BY rolname;
```

Roles are cluster-level objects, so they must be preserved during a migration.

---

# 17.10 Capture Configuration

```sql
SELECT
    name,
    setting,
    unit,
    source
FROM pg_settings
WHERE source <> 'default'
ORDER BY name;
```

Also save:

```bash
sudo cp /var/lib/pgsql/18/data/postgresql.conf \
        /var/tmp/postgresql.conf.pg18

sudo cp /var/lib/pgsql/18/data/pg_hba.conf \
        /var/tmp/pg_hba.conf.pg18
```

Adjust paths to your actual installation.

---

# 17.11 Check Replication

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

Also:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    wal_status
FROM pg_replication_slots;
```

---

# 17.12 Backup Before Upgrade

Your upgrade plan should have **at least one independently validated recovery path**.

Do not consider:

```text
pg_upgrade completed successfully
```

as your backup.

Before production:

```text
FULL BACKUP
    ↓
BACKUP VALIDATION
    ↓
RESTORE TEST
    ↓
UPGRADE
```

---

# 17.13 `pg_upgrade`

`pg_upgrade` upgrades a PostgreSQL cluster to a later major version without the full dump/restore process normally required for a major upgrade. ([PostgreSQL][2])

Concept:

```text
PostgreSQL 18
     |
     | pg_upgrade
     |
     v
PostgreSQL 19
```

---

# 17.14 Why `pg_upgrade` Is Fast

Normally:

```text
Database
   ↓
Dump
   ↓
SQL/text/archive
   ↓
Restore
   ↓
Database
```

Large databases can take significant time.

`pg_upgrade` can reuse existing user-data files while creating the new-version system catalogs.

Therefore:

```text
OLD DATA FILES
      |
      +----------+
                 |
                 v
          NEW CLUSTER
```

This is why `pg_upgrade` can dramatically reduce major-upgrade duration. ([PostgreSQL][2])

---

# 17.15 `pg_upgrade --check`

**Always start here.**

Example:

```bash
sudo -u postgres /usr/pgsql-19/bin/pg_upgrade \
    --check \
    --old-datadir=/var/lib/pgsql/18/data \
    --new-datadir=/var/lib/pgsql/19/data \
    --old-bindir=/usr/pgsql-18/bin \
    --new-bindir=/usr/pgsql-19/bin
```

`--check` performs compatibility checks without modifying the clusters. ([PostgreSQL][2])

---

# 17.16 What `--check` Protects You From

Potential problems include:

```text
Extension incompatibility
Configuration incompatibility
Unsupported objects
Binary incompatibility
Missing libraries
System catalog issues
```

Do not proceed until all actionable errors are resolved.

---

# 17.17 `--jobs`

For large environments:

```bash
--jobs=8
```

Example:

```bash
sudo -u postgres /usr/pgsql-19/bin/pg_upgrade \
    --jobs=8 \
    ...
```

PostgreSQL documentation states that multiple jobs can process databases/tablespaces in parallel and can substantially reduce upgrade time. ([PostgreSQL][2])

A reasonable starting point is based on available CPU capacity, but benchmark in your environment.

---

# 17.18 Copy vs Link Mode

Default behavior:

```text
OLD DATA
   |
   | copy
   v
NEW DATA
```

Link:

```bash
--link
```

Concept:

```text
OLD DATA
   |
   | hard links
   v
NEW DATA
```

Advantages:

```text
Much faster
Less additional disk space
```

But there is an important rollback consequence.

Once the new cluster starts after a link-mode upgrade, the old cluster's data files may be shared/modified and the old cluster may no longer be safely restartable. ([PostgreSQL][2])

---

# 17.19 Clone Mode

Where supported:

```text
--clone
```

Clone mode can provide speed/disk advantages while preserving the old cluster's usability better than link mode.

The exact availability depends on filesystem/OS capabilities. ([PostgreSQL][2])

For a DBA production runbook:

> **Do not select `--link` merely because it is faster. Select it only after your rollback design has explicitly accounted for its behavior.**

---

# 17.20 Production Upgrade Flow

```text
             PRE-CHECK
                 |
                 v
          BACKUP + RESTORE TEST
                 |
                 v
          APPLICATION TEST
                 |
                 v
          pg_upgrade --check
                 |
                 v
          SCHEDULE DOWNTIME
                 |
                 v
           STOP APPLICATION
                 |
                 v
           STOP PG 18
                 |
                 v
             pg_upgrade
                 |
                 v
           START PG 19
                 |
                 v
        POST-UPGRADE SCRIPTS
                 |
                 v
          ANALYZE / STATISTICS
                 |
                 v
       APPLICATION VALIDATION
                 |
                 v
          MONITOR / OBSERVE
```

---

# 17.21 Standby Servers

This is where HA upgrades become more complicated.

PostgreSQL's official `pg_upgrade` documentation provides procedures for upgrading streaming-replication standbys, including installing new binaries and using `rsync` in applicable scenarios. Alternatively, standbys can be recreated after the new primary is running. ([PostgreSQL][2])

A simpler operational strategy is often:

```text
OLD PRIMARY
     |
     v
Upgrade primary
     |
     v
NEW PRIMARY
     |
     v
Rebuild standby
```

This increases rebuild time but reduces operational complexity.

---

# 17.22 Logical Replication Upgrade

Another approach:

```text
PostgreSQL 18
     |
     | logical replication
     v
PostgreSQL 19
```

Then:

```text
Application
     |
     v
Cutover
     |
     v
PostgreSQL 19
```

This can minimize downtime because the target can be synchronized while the source remains available.

---

# 17.23 Logical Replication Advantages

```text
Very low downtime potential
Cross-version migration
Flexible topology
Can migrate selected databases/tables
Can support staged migration
```

Challenges:

```text
Sequences
DDL
Large objects
Extensions
Replication conflicts
Application cutover
Data validation
```

PostgreSQL 19 has also introduced changes to logical replication, including sequence-value replication improvements. ([PostgreSQL][4])

---

# 17.24 Dump/Restore

Traditional:

```bash
pg_dump
pg_restore
```

or:

```bash
pg_dumpall
```

Architecture:

```text
PG18
 |
 | dump
 v
Backup
 |
 | restore
 v
PG19
```

Advantages:

```text
Clean migration
Good cross-platform option
Good for smaller databases
```

Disadvantages:

```text
Longer downtime
Potentially slow for very large databases
Requires more migration time
```

---

# 17.25 PostgreSQL 19 Compatibility Point

PostgreSQL 19 has documented compatibility changes.

For example:

* RADIUS support has been removed.
* MD5 authentication is deprecated and successful MD5 authentication generates a warning.
* `standard_conforming_strings` is forced on.
* PostgreSQL 19 introduces additional password-expiration warning behavior. ([PostgreSQL][4])

Therefore, don't treat a major upgrade as merely:

```text
Install binaries
+
Run pg_upgrade
```

You must perform an **application and security compatibility assessment**.

---

# 17.26 Post-Upgrade Validation

Immediately check:

```sql
SELECT version();
```

```sql
SELECT current_setting('server_version');
```

Then:

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname))
FROM pg_database
WHERE datallowconn;
```

Compare against pre-upgrade values.

---

# 17.27 Check Extensions

```sql
SELECT
    extname,
    extversion
FROM pg_extension
ORDER BY extname;
```

If PostgreSQL reports extension updates are required, execute the generated post-upgrade instructions.

`pg_upgrade` can generate scripts for required post-upgrade processing. ([PostgreSQL][2])

---

# 17.28 Refresh Statistics

After upgrade:

```bash
sudo -u postgres vacuumdb \
    --all \
    --analyze-in-stages \
    --missing-stats-only
```

Then:

```bash
sudo -u postgres vacuumdb \
    --all \
    --analyze-only
```

PostgreSQL specifically recommends regenerating statistics because not all statistics are transferred by `pg_upgrade`. ([PostgreSQL][2])

---

# 17.29 Application Validation

Test:

```text
[ ] Connection
[ ] Login
[ ] SELECT
[ ] INSERT
[ ] UPDATE
[ ] DELETE
[ ] Stored procedures/functions
[ ] Application transactions
[ ] Connection pooling
[ ] TLS
[ ] Authentication
[ ] Reporting
[ ] Batch jobs
[ ] Monitoring
```

---

# 17.30 Performance Validation

Compare before/after:

```text
CPU
Memory
I/O
TPS
Query latency
Connections
Cache hit ratio
WAL generation
Autovacuum
Locks
Replication
```

Don't assume:

```text
Upgrade successful
       =
Performance successful
```

---

# 17.31 Rollback Strategy

### Before cutover

Old cluster:

```text
PG18
 |
 +--> Available
```

If `pg_upgrade --check` fails:

```text
Fix issue
   |
Retry
```

If a normal copy-mode upgrade fails before the new cluster is started:

```text
PG18
 |
 +--> Can generally be restarted
```

PostgreSQL documents that the old cluster remains unmodified when neither `--link` nor `--swap` is used, subject to the documented failure conditions. ([PostgreSQL][2])

---

# 17.32 After New PostgreSQL Starts

Rollback becomes much more serious.

Especially with:

```text
--link
--swap
```

the old cluster may no longer be safely restartable after the new cluster begins using modified/shared data.

Therefore:

```text
BACKUP
   +
ROLLBACK PLAN
   +
TESTED RECOVERY
```

are mandatory.

---

# 17.33 Production Cutover Checklist

```text
[ ] Change ticket approved
[ ] Maintenance window approved
[ ] Backup completed
[ ] Backup restore verified
[ ] Application team notified
[ ] Connection pool drained
[ ] Application stopped
[ ] PostgreSQL 18 stopped
[ ] pg_upgrade --check passed
[ ] PostgreSQL 19 binaries verified
[ ] Extensions installed
[ ] pg_upgrade completed
[ ] Configuration reviewed
[ ] pg_hba.conf reviewed
[ ] TLS reviewed
[ ] PostgreSQL 19 started
[ ] Logs clean
[ ] Databases accessible
[ ] Roles validated
[ ] Extensions validated
[ ] Statistics refreshed
[ ] Application tested
[ ] Monitoring healthy
[ ] HA re-established
[ ] DR validated
```

---

# 17.34 Upgrade RCA Template

### Incident

```text
PostgreSQL major-version upgrade failed.
```

### Symptoms

```text
Application unable to connect.
```

### Investigation

```text
PostgreSQL service
       ↓
Configuration
       ↓
Authentication
       ↓
Extensions
       ↓
Database objects
       ↓
Application compatibility
```

### Root Cause Examples

```text
Unsupported extension
Missing shared library
Incorrect pg_hba.conf
Incompatible configuration
Application driver incompatibility
Insufficient disk space
Invalid tablespace path
```

### Corrective Action

```text
Resolve compatibility issue
Restore/rollback if necessary
Repeat upgrade in non-production
Validate
Reschedule production upgrade
```

---

# 17.35 DBA Recommendation Matrix

For your PostgreSQL DBA lab:

| Scenario                 | Method                                                   |
| ------------------------ | -------------------------------------------------------- |
| 18.x → 18.x patch        | Package update                                           |
| 18 → 19 small DB         | Dump/restore                                             |
| 18 → 19 large DB         | `pg_upgrade`                                             |
| 18 → 19 minimal downtime | Logical replication                                      |
| HA cluster               | Planned topology-specific upgrade                        |
| Cross-platform           | Dump/restore / logical replication                       |
| Production               | `pg_upgrade` or logical replication after full rehearsal |

---

# 17.36 Most Important Commands

### Version

```bash
psql --version
```

```sql
SELECT version();
```

### Cluster

```bash
pg_controldata $PGDATA
```

### Upgrade validation

```bash
pg_upgrade --check
```

### Upgrade

```bash
pg_upgrade
```

### Statistics

```bash
vacuumdb --all --analyze-only
```

### Database size

```sql
SELECT pg_size_pretty(pg_database_size(current_database()));
```

---

# 17.37 PostgreSQL 19 Upgrade Golden Rules

1. **Minor upgrade ≠ major upgrade.**
2. **Do not use `pg_upgrade` for a normal minor release.**
3. **Always run `pg_upgrade --check` first.**
4. **Validate every extension.**
5. **Account for tablespaces.**
6. **Back up before major upgrade.**
7. **Test restore—not just backup completion.**
8. **Do not blindly use `--link` because it is faster.**
9. **Refresh statistics after the upgrade.**
10. **Validate application drivers and connection pools.**
11. **Re-establish HA and replication after migration.**
12. **Keep the rollback strategy explicit.**
13. **Test the complete procedure in a production-sized rehearsal.**
14. **Do not delete the old cluster immediately.**
15. **For PostgreSQL 19 specifically, review the documented compatibility changes before migration.** ([PostgreSQL][2])

---

## Phase 17 Status

```text
01  Installation
02  Architecture
03  Configuration
04  Users / Roles / Security
05  Database / Schema / Objects
06  SQL / DDL / DML
07  Storage / Tablespaces
08  WAL
09  HA Fundamentals
10  Streaming Replication
11  Backup Fundamentals
12  Backup / Recovery / PITR
13  Performance Tuning
14  Monitoring / Health Check
15  HA / Failover / Switchover
16  Security & Encryption
17  Upgrades / Patching / Migration   ← COMPLETED
```

### Next — Phase 18

**PostgreSQL 19 Advanced Troubleshooting & RCA**

That phase will be much more operational and will cover **startup failures, connection failures, authentication failures, blocking/locks, WAL problems, replication failures, disk-full incidents, corruption symptoms, autovacuum problems, high CPU, high memory, slow queries, emergency DBA commands, and complete production RCA methodology.**

[1]: https://www.postgresql.org/docs/release/?utm_source=chatgpt.com "PostgreSQL: Release Notes"
[2]: https://www.postgresql.org/docs/19/pgupgrade.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: pg_upgrade"
[3]: https://www.postgresql.org/docs/19/runtime.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: Chapter 18. Server Setup and Operation"
[4]: https://www.postgresql.org/docs/19/release-19.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: E.1. Release 19"
