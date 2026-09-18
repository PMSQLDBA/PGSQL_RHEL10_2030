# Phase 25 — PostgreSQL 19 DBA Capstone & Production Runbook

**PostgreSQL 19 + RHEL 10**

This is the **final phase**. It consolidates the operational model into a practical DBA runbook without repeating the detailed material from Phases 1–24.

---

# 25.1 Production DBA Operating Model

A PostgreSQL production environment should be operated through five continuous activities:

```text
BUILD
  ↓
VALIDATE
  ↓
OPERATE
  ↓
PROTECT
  ↓
IMPROVE
  ↺
```

Where:

| Area     | DBA responsibility                |
| -------- | --------------------------------- |
| Build    | Installation/configuration        |
| Validate | Functional & technical validation |
| Operate  | Monitoring & incident response    |
| Protect  | Security, backup, recovery        |
| Improve  | Performance, capacity, automation |

---

# 25.2 Production Architecture

A typical enterprise design:

```text
                         USERS
                           |
                           v
                    APPLICATION TIER
                           |
                    Connection Pool
                           |
                           v
                 +-------------------+
                 | PostgreSQL 19     |
                 | Primary           |
                 +-------------------+
                    |             |
             Replication       WAL Archive
                    |             |
                    v             v
             Standby/DR       Backup Store
                                  |
                                  v
                           Offsite / Immutable
```

RHEL 10 provides:

```text
OS
├── systemd
├── SELinux
├── Firewall
├── Filesystem
├── Storage
├── Networking
└── OS monitoring
```

PostgreSQL provides:

```text
Database Engine
├── SQL
├── Transactions
├── WAL
├── MVCC
├── Authentication
├── Roles
├── Statistics
├── Replication
└── Recovery
```

---

# 25.3 Production Build Standard

Before accepting a PostgreSQL server into production:

```text
[ ] RHEL 10 baseline complete
[ ] Hostname configured
[ ] DNS validated
[ ] Time synchronization validated
[ ] Storage provisioned
[ ] PostgreSQL 19 installed
[ ] Cluster initialized
[ ] Service configured
[ ] Database connectivity validated
[ ] Authentication configured
[ ] Network access restricted
[ ] Logging configured
[ ] Monitoring configured
[ ] Backup configured
[ ] Recovery tested
[ ] Documentation completed
```

---

# 25.4 PostgreSQL Installation Validation

After installation:

```bash
psql --version
```

Then:

```bash
systemctl status <postgresql-service>
```

Validate:

```sql
SELECT version();

SELECT current_database();

SELECT current_user;

SELECT now();

SELECT pg_is_in_recovery();
```

This confirms the fundamental operating state.

---

# 25.5 PostgreSQL Production Acceptance Test

Use the following sequence:

```text
OS
 ↓
PostgreSQL Service
 ↓
Port
 ↓
Authentication
 ↓
Database
 ↓
Schema
 ↓
SQL
 ↓
Application
 ↓
Monitoring
 ↓
Backup
 ↓
Recovery
```

A server should not be considered production-ready merely because:

```text
systemctl status = active
```

---

# 25.6 Database Creation Standard

Example:

```sql
CREATE DATABASE appdb;
```

Then establish:

```text
Database
   ↓
Schema
   ↓
Application roles
   ↓
Objects
   ↓
Privileges
```

Avoid putting every application object directly into the default `public` schema without a deliberate design.

---

# 25.7 Production Role Model

Recommended conceptual structure:

```text
                Application
                     |
                     v
               Login Role
                     |
                     v
             Privilege Role
              /     |      \
             /      |       \
          SELECT   EXECUTE   DML
```

Separate:

```text
Application identity
DBA identity
Monitoring identity
Replication identity
Backup identity
```

where practical.

---

# 25.8 Database Naming Standard

Use consistent naming.

Example:

```text
Environment:
PROD
QA
DEV
DR

Database:
sales_prod
sales_qa
sales_dev
sales_dr
```

Actual organizational standards may differ.

Consistency is more important than the particular naming convention.

---

# 25.9 Schema Standard

Example:

```text
app
reporting
staging
audit
```

Avoid allowing unrelated applications to share one uncontrolled schema.

---

# 25.10 Production Configuration Management

Treat PostgreSQL configuration as code/configuration management.

```text
Configuration
     |
     v
Version Control
     |
     v
Peer Review
     |
     v
Testing
     |
     v
Approval
     |
     v
Production
```

Capture the effective configuration:

```sql
SELECT
    name,
    setting,
    unit,
    source
FROM pg_settings
ORDER BY name;
```

---

# 25.11 Configuration Change Record

Every significant change should record:

```text
Change ID
Date/time
Server
Parameter
Old value
New value
Reason
Expected impact
Implementation
Validation
Rollback
Owner
```

This becomes valuable during RCA.

---

# 25.12 DBA Daily Runbook

### Start of day

```text
1. Check PostgreSQL availability
2. Check critical alerts
3. Check failed jobs
4. Check filesystem capacity
5. Check connections
6. Check long-running transactions
7. Check blocking
8. Check replication
9. Check WAL/archive health
10. Check previous backup status
```

Then investigate exceptions.

The DBA should spend time on **exceptions**, not manually inspecting every normal metric.

---

# 25.13 DBA Weekly Runbook

```text
1. Review top SQL
2. Review query latency trends
3. Review deadlocks
4. Review autovacuum behavior
5. Review database growth
6. Review table/index growth
7. Review WAL growth
8. Review backup verification
9. Review restore-test results
10. Review configuration drift
```

---

# 25.14 DBA Monthly Runbook

```text
1. Capacity forecast
2. Performance trend
3. Storage forecast
4. Role/access review
5. Configuration review
6. Backup/recovery review
7. Incident review
8. Patch review
9. Monitoring review
10. Automation review
```

---

# 25.15 Incident Management

Use:

```text
DETECT
  ↓
ASSESS
  ↓
CONTAIN
  ↓
RECOVER
  ↓
VALIDATE
  ↓
RCA
  ↓
PREVENT
```

Don't immediately make configuration changes before establishing the failure mode.

---

# 25.16 P1 Database Down

### Step 1

Check:

```bash
systemctl status <postgresql-service>
```

### Step 2

Check logs:

```bash
journalctl -u <postgresql-service> --since "30 minutes ago"
```

### Step 3

Check OS:

```bash
df -h
free -h
uptime
```

### Step 4

Determine:

```text
Process failure?
Storage?
Memory?
Configuration?
Corruption?
OS failure?
Infrastructure failure?
```

### Step 5

Recover according to the documented incident/recovery procedure.

---

# 25.17 P1 Storage Full

Never immediately delete PostgreSQL files.

Investigate:

```bash
df -h
du -xhd1 <filesystem>
```

Determine whether growth comes from:

```text
PGDATA
WAL
Archive
Temporary files
Logs
Backups
Other OS files
```

Then remediate the actual cause.

---

# 25.18 P1 WAL Growth

Flow:

```text
WAL growing
   ↓
Check generation rate
   ↓
Check archiving
   ↓
Check replication
   ↓
Check long transactions
   ↓
Check workload
   ↓
Identify root cause
```

Possible causes differ substantially, so avoid deleting WAL manually.

---

# 25.19 P1 Blocking Incident

Flow:

```text
Application slowdown
       ↓
Check pg_stat_activity
       ↓
Identify waiting sessions
       ↓
Identify root blocker
       ↓
Identify transaction
       ↓
Determine business impact
       ↓
Controlled remediation
```

Do not terminate a session simply because it appears in a blocking query.

First establish what the transaction is doing.

---

# 25.20 P1 Performance Incident

Use:

```text
Current symptoms
      ↓
pg_stat_activity
      ↓
pg_stat_statements
      ↓
EXPLAIN
      ↓
Locks
      ↓
CPU
      ↓
I/O
      ↓
Memory
      ↓
Root cause
```

Compare against a known-good baseline.

---

# 25.21 P1 Database Corruption

Treat suspected corruption as a serious incident.

```text
Stop unnecessary changes
       ↓
Preserve evidence
       ↓
Review PostgreSQL logs
       ↓
Review OS/storage logs
       ↓
Validate affected objects
       ↓
Check backups
       ↓
Determine recovery strategy
       ↓
Validate restored environment
```

Don't repeatedly modify a potentially damaged database before understanding the failure.

---

# 25.22 Emergency Recovery Decision

Use this decision model:

```text
Data damaged?
     |
     +-- NO → Fix service/problem
     |
     +-- YES
          |
          v
    Latest good point known?
          |
      +---+---+
      |       |
     YES      NO
      |       |
      v       v
    PITR    Investigate
```

Then evaluate:

```text
RPO
RTO
Data loss
Business impact
Backup availability
WAL availability
```

---

# 25.23 Recovery Validation Gate

Recovery is not complete until:

```text
[ ] PostgreSQL starts
[ ] Expected database exists
[ ] Expected schemas exist
[ ] Critical tables exist
[ ] Data validation passes
[ ] Roles exist
[ ] Permissions work
[ ] Extensions work
[ ] Application connects
[ ] Application transaction succeeds
[ ] Monitoring is active
[ ] Backup resumes
```

---

# 25.24 Performance Change Gate

Before changing a performance parameter:

```text
Baseline
   ↓
Evidence
   ↓
Hypothesis
   ↓
Test
   ↓
Change
   ↓
Measure
   ↓
Keep / Rollback
```

This prevents configuration "tuning by folklore."

---

# 25.25 Index Change Gate

Before creating an index:

```text
Query identified
      ↓
Execution plan reviewed
      ↓
Selectivity understood
      ↓
Existing indexes reviewed
      ↓
Write overhead considered
      ↓
Storage overhead considered
      ↓
Index created
      ↓
Plan validated
```

---

# 25.26 Automation Standard

Production scripts should contain:

```text
#!/bin/bash

set -euo pipefail
```

and ideally implement:

```text
Input validation
Logging
Error handling
Timeouts
Exit codes
Credential protection
Dry-run capability
Audit trail
```

For destructive operations:

```text
Confirmation
+
Change approval
+
Rollback
```

should be considered mandatory operational controls.

---

# 25.27 DBA Automation Repository

Recommended structure:

```text
postgresql-dba/
│
├── installation/
├── configuration/
├── health-check/
├── monitoring/
├── performance/
├── backup/
├── recovery/
├── security/
├── maintenance/
├── incident-response/
├── reporting/
└── documentation/
```

Each script should have:

```text
Purpose
Prerequisites
Inputs
Expected output
Exit codes
Examples
Rollback
Owner
Version
```

---

# 25.28 Production Health Check

A comprehensive health check should produce:

```text
========================================
POSTGRESQL PRODUCTION HEALTH CHECK
========================================

Server:
PostgreSQL:
Uptime:
Recovery state:

Availability:
Connections:
Transactions:
Locks:
Deadlocks:
Long transactions:

Database growth:
Largest databases:
Largest tables:

Autovacuum:
Transaction age:

WAL:
Replication:
Archiving:

Filesystem:
CPU:
Memory:
I/O:

Backup:
Backup verification:
Last restore test:

Errors:
Critical alerts:

Overall status:
========================================
```

---

# 25.29 Health Status

Use:

```text
GREEN
Normal

AMBER
Warning / DBA attention

RED
Critical / Immediate action
```

Avoid assigning "green" solely because PostgreSQL responds to a connection test.

---

# 25.30 Production Documentation

Maintain these documents:

```text
1. Architecture document
2. Installation SOP
3. Configuration baseline
4. Security baseline
5. Backup SOP
6. Recovery SOP
7. HA/DR SOP
8. Monitoring SOP
9. Performance SOP
10. Incident response SOP
11. Upgrade SOP
12. Patch SOP
13. Capacity plan
14. Application dependency map
15. Contact/escalation matrix
```

---

# 25.31 PostgreSQL Upgrade Runbook

For a major-version upgrade:

```text
Planning
   ↓
Compatibility assessment
   ↓
Application testing
   ↓
Extension validation
   ↓
Backup validation
   ↓
Upgrade rehearsal
   ↓
Downtime/RTO validation
   ↓
Production upgrade
   ↓
Post-upgrade validation
   ↓
Performance comparison
```

Never treat:

```text
PostgreSQL 18 → PostgreSQL 19
```

as equivalent to a routine minor update.

A major-version upgrade requires an upgrade strategy and compatibility testing.

---

# 25.32 Minor Release / Security Update Runbook

```text
Review release
    ↓
Assess impact
    ↓
Test
    ↓
Backup verification
    ↓
Schedule maintenance
    ↓
Apply update
    ↓
Restart if required
    ↓
Validate
    ↓
Monitor
```

Keep PostgreSQL and RHEL patching under controlled change management.

---

# 25.33 DBA Production Readiness Scorecard

Don't use this as a quality ranking; use it as a **readiness checklist**:

| Domain        | Required evidence          |
| ------------- | -------------------------- |
| Installation  | Tested build               |
| Configuration | Approved baseline          |
| Security      | Access controls            |
| Monitoring    | Actionable alerts          |
| Performance   | Baseline                   |
| Backup        | Successful backups         |
| Recovery      | Successful restore         |
| DR            | Tested procedure           |
| Automation    | Version-controlled scripts |
| Documentation | Current runbooks           |
| Capacity      | Forecast                   |
| Operations    | Incident procedures        |

A production system is ready when the required evidence exists for its applicable controls.

---

# 25.34 Complete DBA Lifecycle

```text
              DESIGN
                 |
                 v
             INSTALL
                 |
                 v
             CONFIGURE
                 |
                 v
             SECURE
                 |
                 v
             VALIDATE
                 |
                 v
              DEPLOY
                 |
                 v
              MONITOR
                 |
                 v
              OPTIMIZE
                 |
                 v
              PROTECT
                 |
                 v
              RECOVER
                 |
                 v
              IMPROVE
                 |
                 +----------+
                            |
                            v
                           DESIGN
```

---

# 25.35 Final DBA Command Map

| Requirement                 | Primary tool                      |
| --------------------------- | --------------------------------- |
| Server availability         | `pg_isready`                      |
| PostgreSQL service          | `systemctl`                       |
| PostgreSQL logs             | `journalctl` / PostgreSQL logging |
| SQL sessions                | `pg_stat_activity`                |
| Query workload              | `pg_stat_statements`              |
| Execution plan              | `EXPLAIN`                         |
| Statistics                  | `ANALYZE`                         |
| Database size               | `pg_database_size()`              |
| Table size                  | `pg_total_relation_size()`        |
| Locks                       | `pg_locks` + `pg_stat_activity`   |
| WAL                         | `pg_stat_wal`                     |
| Table maintenance           | `pg_stat_user_tables`             |
| Physical backup             | `pg_basebackup`                   |
| Backup validation           | `pg_verifybackup`                 |
| Incremental backup assembly | `pg_combinebackup`                |
| Integrity checking          | `pg_amcheck`                      |
| Data checksums              | `pg_checksums`                    |
| Diverged standby recovery   | `pg_rewind`                       |
| RHEL CPU/memory             | `vmstat`, `top`, `free`           |
| RHEL storage                | `iostat`                          |
| Filesystem                  | `df`, `du`                        |

---

# 25.36 Final Production DBA Golden Rules

### Rule 1

**Never change production without evidence.**

### Rule 2

**Never consider a backup successful until it has been verified and recovery has been tested.**

### Rule 3

**Never troubleshoot PostgreSQL without checking both PostgreSQL and RHEL.**

### Rule 4

**Never assume high CPU, high memory, or high I/O is the root cause.**

### Rule 5

**Never terminate a database session without understanding its transaction and business impact.**

### Rule 6

**Never delete PostgreSQL WAL/database files manually as a space-management shortcut.**

### Rule 7

**Never use `EXPLAIN ANALYZE` on destructive SQL casually in production.**

### Rule 8

**Never treat an HA standby as a substitute for backup.**

### Rule 9

**Never treat successful PostgreSQL startup as successful application recovery.**

### Rule 10

**Automate repeatable work; keep high-risk decisions under controlled DBA procedures.**

---

# 25.37 Final PostgreSQL 19 + RHEL 10 DBA Model

```text
                     PostgreSQL 19 DBA
                           |
       +-------------------+-------------------+
       |                   |                   |
   PLATFORM             DATABASE            OPERATIONS
       |                   |                   |
     RHEL 10             SQL/MVCC           Monitoring
     Storage             WAL                Automation
     Network             Indexes            Incident
     SELinux             Statistics         Capacity
     systemd             Transactions       Documentation
       |                   |                   |
       +-------------------+-------------------+
                           |
                           v
                    BUSINESS SERVICE
                           |
                           v
                  Availability + Data
```

---

# Phase 25 — COMPLETED

You now have the complete **25-phase PostgreSQL DBA learning/operations track**:

```text
01 → PostgreSQL Fundamentals
02 → Architecture
03 → RHEL/PostgreSQL Environment
04 → Installation
05 → Configuration
06 → Database & Schema Administration
07 → Roles & Access
08 → SQL Administration
09 → Storage
10 → Transactions & MVCC
11 → Maintenance
12 → Indexing
13 → Query Optimization
14 → Monitoring
15 → Backup
16 → Recovery
17 → Replication
18 → HA/DR
19 → Upgrades & Migration
20 → Enterprise Administration
21 → Security & Compliance
22 → Advanced Performance
23 → Advanced Backup & Recovery
24 → Monitoring / Observability / Automation
25 → **DBA Capstone & Production Runbook**
```

## **PHASE 25 IS THE FINAL PHASE.**
