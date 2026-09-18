# Phase 23 — PostgreSQL 19 Advanced Backup & Recovery Operations on RHEL 10

This phase covers **new recovery/backup operations**, without repeating the backup fundamentals, HA/DR architecture, or basic replication already covered.

I verified the procedures against the current **PostgreSQL 19 documentation**. PostgreSQL 19 includes `pg_basebackup`, `pg_verifybackup`, `pg_combinebackup`, `pg_rewind`, `pg_amcheck`, and related recovery utilities. ([PostgreSQL][1])

---

## 23.1 Enterprise Backup Strategy

A production PostgreSQL backup strategy should provide:

```text
Logical Backup
      +
Physical Base Backup
      +
WAL Archive
      +
Backup Verification
      +
Restore Testing
      +
Offsite Copy
```

The objective is not merely:

> "We have a backup."

The objective is:

> **"We can recover the required data within the required RPO/RTO."**

---

# 23.2 Backup Types

| Backup             | Purpose                                    |
| ------------------ | ------------------------------------------ |
| `pg_dump`          | Individual database logical backup         |
| `pg_dumpall`       | Cluster-wide logical objects/roles/globals |
| `pg_basebackup`    | Physical cluster backup                    |
| WAL archive        | Point-in-time recovery                     |
| Incremental backup | Reduce repeated full-copy requirements     |
| Snapshot           | Infrastructure-level recovery mechanism    |

A physical base backup covers the **entire PostgreSQL cluster**, whereas `pg_dump` is used when you need database/object-level logical backup. ([PostgreSQL][1])

---

# 23.3 PostgreSQL 19 Incremental Backup Capability

PostgreSQL 19 includes incremental-backup support around:

```text
pg_basebackup
+
backup manifests
+
pg_combinebackup
```

`pg_combinebackup` can reconstruct a full backup from an incremental backup and its dependent backups. ([PostgreSQL][1])

Conceptually:

```text
FULL BACKUP
     |
     +---------+
     |         |
     v         v
Incremental  Incremental
   #1           #2
     |           |
     +-----+-----+
           |
           v
   pg_combinebackup
           |
           v
     Reconstructed
       full backup
```

This is particularly relevant for large PostgreSQL installations where repeatedly transferring the entire cluster is expensive.

---

# 23.4 Why Incremental Backups Matter

For a:

```text
20 TB PostgreSQL cluster
```

a traditional repeated full physical backup can create substantial:

```text
Network traffic
Storage consumption
Backup duration
I/O load
```

Incremental strategies can reduce the amount of changed data that needs to be captured.

But the backup chain must be managed carefully.

---

# 23.5 Backup Chain

Example:

```text
Day 1
FULL
 |
 +--> Day 2 Incremental
       |
       +--> Day 3 Incremental
             |
             +--> Day 4 Incremental
```

If your recovery design depends on this chain:

```text
FULL
+
required incrementals
+
required WAL
```

must all remain available.

Therefore:

> Backup retention must consider the dependency chain, not simply individual file age.

---

# 23.6 Backup Manifest

Modern PostgreSQL physical backups can generate:

```text
backup_manifest
```

The manifest records information about files included in the backup and supports verification.

This enables:

```text
Backup
   ↓
Manifest
   ↓
pg_verifybackup
   ↓
Integrity validation
```

---

# 23.7 `pg_verifybackup`

Run:

```bash
pg_verifybackup /backup/base_20260917
```

This verifies the backup against its manifest.

PostgreSQL explicitly notes that `pg_verifybackup` does **not** prove that every possible recovery problem has been eliminated; test restores remain necessary. ([PostgreSQL][2])

Therefore:

```text
Backup exists
      ≠
Backup verified
      ≠
Backup successfully restored
```

---

# 23.8 Enterprise Backup Validation

Use three levels:

```text
LEVEL 1
File existence

LEVEL 2
pg_verifybackup

LEVEL 3
Actual restore + application/data validation
```

The third level is the strongest operational test.

---

# 23.9 `pg_amcheck`

PostgreSQL provides:

```bash
pg_amcheck
```

for checking corruption/consistency of databases and relations.

This is useful as part of a scheduled integrity-validation strategy. ([PostgreSQL][3])

Example:

```bash
pg_amcheck \
    --database=appdb
```

For large environments, carefully schedule checks because they consume I/O and CPU.

---

# 23.10 Data Checksums

Check whether checksums are enabled:

```bash
pg_checksums \
    --check \
    -D /var/lib/pgsql/19/data
```

PostgreSQL documents `pg_checksums` for checking/enabling/disabling data checksums. ([PostgreSQL][4])

Important:

> Checksum operations on a large cluster can be expensive and enabling/disabling them requires the cluster to be stopped.

Do not casually execute checksum state changes on production.

---

# 23.11 PITR Architecture

Point-in-Time Recovery:

```text
BASE BACKUP
     |
     +----------------------+
                            |
                         WAL ARCHIVE
                            |
                            v
                    Recovery Process
                            |
                            v
                    Target Timestamp
```

Example:

```text
Base backup:
Sunday 01:00

WAL:
Sunday 01:00 → Wednesday 15:30
```

You can recover to an appropriate point within the available recovery history.

---

# 23.12 PITR Use Cases

PITR is especially valuable for:

```text
Accidental DELETE
Accidental DROP
Bad deployment
Application corruption
Data corruption
Human error
Logical damage
```

Example:

```text
14:00  Normal
14:05  Bad deployment
14:10  DELETE executed
14:15  Incident detected
```

You may recover to a point immediately before the destructive event, assuming the necessary backup/WAL history exists.

---

# 23.13 Named Restore Point

Before a high-risk deployment:

```sql
SELECT pg_create_restore_point('before_release_20260917');
```

PostgreSQL records a named WAL marker that can later be used as a recovery target. ([PostgreSQL][5])

Architecture:

```text
             WAL
              |
14:00 --------+---------
              |
              v
      before_release
              |
14:05 --------+---------
              |
              v
        Deployment
```

This can simplify recovery targeting.

---

# 23.14 Restore Point Naming Standard

Use a predictable naming convention:

```text
before_release_YYYYMMDD
before_schema_change_YYYYMMDD
before_major_upgrade_YYYYMMDD
before_data_fix_YYYYMMDD
```

Avoid reusing the same restore-point name because PostgreSQL recovery by name stops at the first matching restore point encountered. ([PostgreSQL][5])

---

# 23.15 Recovery Target Types

PITR can target concepts such as:

```text
Timestamp
Transaction ID
LSN
Named restore point
Immediate recovery point
```

The correct target depends on what happened and what recovery evidence is available.

---

# 23.16 PITR Recovery Flow

```text
                 INCIDENT
                    |
                    v
             Determine target
                    |
                    v
             Select base backup
                    |
                    v
              Restore backup
                    |
                    v
              Supply WAL
                    |
                    v
             Recovery process
                    |
                    v
              Target reached
                    |
                    v
             Validate database
                    |
                    v
               Open service
```

---

# 23.17 Never Guess the Recovery Point

If a destructive operation occurred at:

```text
15:43:27
```

don't arbitrarily recover to:

```text
15:40:00
```

unless that satisfies the business requirement.

Determine:

```text
Exact incident time
Last known good transaction
Required business recovery point
```

Then select the appropriate recovery target.

---

# 23.18 Recovery Validation

After recovery:

```sql
SELECT now();

SELECT pg_is_in_recovery();
```

Then validate:

```text
Database exists
Critical schemas exist
Critical tables exist
Expected row counts
Recent transactions
Application connectivity
Permissions
Extensions
Sequences
Constraints
```

---

# 23.19 Application-Level Validation

Database recovery is incomplete until the application is validated.

Example:

```text
Database
   ↓
Connection
   ↓
Application login
   ↓
Read test
   ↓
Write test
   ↓
Business transaction
```

This is especially important because a technically successful PostgreSQL recovery can still produce an application-level failure.

---

# 23.20 Backup Storage Architecture

Do not keep every backup on the same server.

Bad:

```text
PostgreSQL
   |
   +-- /backup
```

If the server/storage fails:

```text
Database lost
+
Backup lost
```

Better:

```text
PostgreSQL
     |
     v
Local backup
     |
     v
Remote backup repository
     |
     v
Offsite / DR storage
```

---

# 23.21 Backup Immutability

For ransomware resilience, consider:

```text
Immutable storage
Object lock
Write protection
Separate credentials
Separate administrative boundary
```

The exact mechanism depends on the storage platform.

The principle is:

```text
Production admin compromise
        X
        |
        v
Backup deletion
```

should be difficult or impossible within the defined retention window.

---

# 23.22 Encryption of Backups

Your backup strategy should consider:

```text
Encryption at rest
Encryption in transit
Key management
Access control
Retention
Deletion
```

A backup containing sensitive data is itself sensitive data.

---

# 23.23 Backup Access Control

Recommended separation:

```text
DBA
 |
 +--> Can create backups
 |
Backup system
 |
 +--> Can store backups
 |
Security/Backup admin
 |
 +--> Controls retention/deletion
```

This supports separation of duties.

---

# 23.24 Recovery Testing

Minimum practical model:

```text
Daily
    Backup verification

Weekly
    Restore sample

Monthly
    Full recovery test

Quarterly
    DR recovery exercise
```

Actual frequency should be based on business requirements and risk.

---

# 23.25 Recovery Drill

Example:

```text
START
  |
  v
Declare recovery event
  |
  v
Identify latest valid backup
  |
  v
Verify backup
  |
  v
Restore
  |
  v
Apply WAL
  |
  v
Reach target
  |
  v
Validate
  |
  v
Application test
  |
  v
Record RTO/RPO
```

---

# 23.26 Measuring Recovery Time

Record:

```text
T0 = Incident
T1 = Recovery started
T2 = Base backup restored
T3 = WAL replay started/completed
T4 = Database available
T5 = Application available
```

Then:

```text
RTO = T5 - T0
```

This gives you an actual measured recovery time.

---

# 23.27 `pg_rewind`

After a failover, the old primary may have diverged from the new primary.

PostgreSQL provides:

```text
pg_rewind
```

to synchronize a cluster whose timeline has diverged with another copy of the same cluster. ([PostgreSQL][1])

Conceptually:

```text
OLD PRIMARY
     |
     | diverged
     v
NEW PRIMARY
```

Then:

```text
OLD PRIMARY
     |
     | pg_rewind
     v
NEW PRIMARY timeline
     |
     v
OLD PRIMARY becomes standby
```

This can be significantly faster than rebuilding an enormous cluster from scratch when its prerequisites are satisfied.

---

# 23.28 `pg_rewind` vs Fresh Base Backup

|                    | `pg_rewind`                 | Fresh `pg_basebackup`             |
| ------------------ | --------------------------- | --------------------------------- |
| Purpose            | Reconcile divergent cluster | Build new physical copy           |
| Data movement      | Potentially much smaller    | Full base backup                  |
| Speed              | Can be much faster          | Potentially slower                |
| Prerequisites      | Important                   | Simpler operational model         |
| Use after failover | Often useful                | Always viable if resources permit |

Never assume `pg_rewind` is always possible; validate the PostgreSQL prerequisites before using it.

---

# 23.29 Recovery Corruption Investigation

If corruption is suspected:

```text
Symptoms
   ↓
Database errors
   ↓
pg_amcheck
   ↓
pg_checksums
   ↓
PostgreSQL logs
   ↓
Storage/kernel logs
   ↓
Backup comparison
```

Investigate the infrastructure as well as PostgreSQL.

---

# 23.30 RHEL Storage Evidence

During corruption incidents collect:

```bash
dmesg -T
```

and:

```bash
journalctl -k
```

Also:

```bash
lsblk
```

and:

```bash
df -h
```

and:

```bash
mount
```

The objective is to determine whether the underlying storage layer contributed to the problem.

---

# 23.31 Backup Incident RCA

### Symptom

```text
Backup job reported SUCCESS
but restore failed.
```

### Investigation

```text
Backup file exists
       ↓
pg_verifybackup
       ↓
Integrity failure
```

Possible causes:

```text
Incomplete backup transfer
Storage corruption
Missing files
Backup-process failure
Incorrect retention
```

### Corrective action

```text
Fix backup pipeline
+
Add verification
+
Add restore testing
+
Monitor failures
```

---

# 23.32 Enterprise Backup SLA

Define:

```text
Backup frequency
Retention
RPO
RTO
Backup window
Restore window
Offsite copy requirement
Encryption
Immutability
Verification frequency
Restore-test frequency
```

Without these, "backup strategy" is incomplete.

---

# 23.33 DBA Backup Dashboard

Recommended metrics:

```text
Last successful base backup
Last successful incremental backup
Last WAL archived
Last backup verification
Backup size
Backup duration
WAL volume
Backup repository capacity
Restore-test result
Recovery duration
RPO achieved
RTO achieved
```

---

# 23.34 Phase 23 Production Checklist

```text
[ ] Logical backups configured where required
[ ] Physical backups configured
[ ] WAL archiving configured
[ ] Backup manifests generated
[ ] pg_verifybackup implemented
[ ] Restore tests implemented
[ ] pg_amcheck strategy defined
[ ] Data checksums assessed
[ ] PITR tested
[ ] Restore points documented
[ ] Backup encryption implemented
[ ] Offsite copy implemented
[ ] Immutable retention considered
[ ] Recovery runbook documented
[ ] pg_rewind procedure documented
[ ] RPO measured
[ ] RTO measured
[ ] Evidence retained
```

---

# Phase 23 Completed

```

[1]: https://www.postgresql.org/docs/19/reference.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: Part VI. Reference"
[2]: https://www.postgresql.org/docs/18/app-pgverifybackup.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: pg_verifybackup"
[3]: https://www.postgresql.org/docs/19/contrib.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: Appendix F. Additional Supplied Modules and Extensions"
[4]: https://www.postgresql.org/docs/19/app-pgchecksums.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: pg_checksums"
[5]: https://www.postgresql.org/docs/19/functions-admin.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 9.29. System Administration Functions"
