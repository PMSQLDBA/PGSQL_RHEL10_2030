# Phase 24 — PostgreSQL 19 Enterprise Monitoring, Observability & Automation on RHEL 10

This phase covers **new enterprise operational material only**. No repetition of the security, performance, backup/recovery, or installation topics already covered.

---

## 24.1 Monitoring vs Observability

### Monitoring

Answers:

> **Is the system healthy?**

Examples:

```text
CPU
Memory
Disk
Connections
Replication status
Database availability
```

### Observability

Answers:

> **Why is the system behaving this way?**

It combines:

```text
Metrics
Logs
Traces
Events
Query statistics
System telemetry
```

Enterprise PostgreSQL operations need both.

---

# 24.2 PostgreSQL Observability Architecture

```text
                    PostgreSQL 19
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
      Metrics          Logs          Query Statistics
        |                |                |
        +----------------+----------------+
                         |
                         v
                Monitoring Platform
                         |
              +----------+----------+
              |                     |
              v                     v
          Dashboard               Alerts
              |
              v
             DBA
```

---

# 24.3 What Should a DBA Monitor?

Organize monitoring into:

```text
1. Availability
2. Capacity
3. Connections
4. Transactions
5. Locks
6. Sessions
7. Replication
8. WAL
9. Autovacuum
10. Query workload
11. Storage
12. OS
13. Errors
14. Backups
15. Security events
```

---

# 24.4 Availability Monitoring

Basic test:

```bash
pg_isready -h <hostname> -p 5432
```

Example:

```bash
pg_isready -h 192.168.0.145 -p 5432
```

Expected:

```text
accepting connections
```

This is useful for a simple availability probe.

It does **not** prove that the application is functioning correctly.

---

# 24.5 Application-Level Health Check

A stronger health check should perform:

```text
DNS
 ↓
TCP
 ↓
PostgreSQL authentication
 ↓
SELECT test
 ↓
Required schema/object test
 ↓
Application transaction test
```

Therefore:

```text
Port 5432 open
       ≠
PostgreSQL healthy
       ≠
Application healthy
```

---

# 24.6 Connection Monitoring

```sql
SELECT
    datname,
    usename,
    application_name,
    client_addr,
    state,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY
    datname,
    usename,
    application_name,
    client_addr,
    state
ORDER BY connections DESC;
```

This provides a workload-oriented connection view.

---

# 24.7 Connection Saturation

Monitor:

```sql
SHOW max_connections;
```

Current sessions:

```sql
SELECT count(*)
FROM pg_stat_activity;
```

Calculate utilization:

```sql
SELECT
    count(*) AS current_connections,
    current_setting('max_connections')::int AS max_connections,
    round(
        100.0 * count(*) /
        current_setting('max_connections')::int,
        2
    ) AS utilization_pct
FROM pg_stat_activity;
```

Don't automatically increase `max_connections` when utilization is high.

Investigate connection pooling and application behavior first.

---

# 24.8 Long-Running Transactions

```sql
SELECT
    pid,
    usename,
    datname,
    xact_start,
    now() - xact_start AS transaction_age,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Long transactions can interfere with:

```text
VACUUM
Dead-tuple cleanup
Transaction ID advancement
Storage utilization
```

---

# 24.9 Idle in Transaction

Find:

```sql
SELECT
    pid,
    usename,
    datname,
    xact_start,
    state_change,
    now() - state_change AS idle_duration,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY state_change;
```

This is a particularly important application behavior to monitor.

---

# 24.10 Blocking Sessions

Find sessions waiting on another session:

```sql
SELECT
    pid,
    usename,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

For production diagnosis, correlate the waiting session with the blocking session rather than killing sessions blindly.

---

# 24.11 Blocking Chain

Conceptually:

```text
Session 101
    |
    | holds lock
    v
Session 202
    |
    | waits
    v
Session 303
```

One transaction can therefore create a chain of blocked sessions.

The DBA should identify:

```text
Root blocker
      ↓
Affected sessions
      ↓
Business impact
```

---

# 24.12 Wait Events

PostgreSQL exposes wait information through:

```sql
pg_stat_activity
```

Example:

```sql
SELECT
    pid,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;
```

Wait-event analysis helps distinguish:

```text
Lock waits
I/O waits
Client waits
IPC waits
Other backend waits
```

---

# 24.13 Transaction Throughput

Useful database-level metrics include:

```sql
SELECT
    datname,
    xact_commit,
    xact_rollback
FROM pg_stat_database
ORDER BY datname;
```

Track trends rather than relying on a single snapshot.

---

# 24.14 Rollback Rate

A rising rollback rate can indicate:

```text
Application errors
Constraint failures
Deadlocks
Timeouts
Transaction management problems
```

Monitor the **trend** and correlate it with application deployments/events.

---

# 24.15 Deadlocks

PostgreSQL can log deadlock information when configured appropriately.

A deadlock generally means:

```text
Transaction A
    holds Lock 1
    waits for Lock 2

Transaction B
    holds Lock 2
    waits for Lock 1
```

Architecture:

```text
A ──holds──> Lock 1
A ──waits──> Lock 2

B ──holds──> Lock 2
B ──waits──> Lock 1
```

PostgreSQL detects the cycle and aborts one transaction.

---

# 24.16 Autovacuum Monitoring

Check:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    autovacuum_count,
    last_autoanalyze,
    autoanalyze_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Monitor:

```text
Dead tuples
Autovacuum frequency
Autoanalyze frequency
Tables not receiving maintenance
```

---

# 24.17 Autovacuum Alert Conditions

Don't use a single universal threshold.

Instead identify:

```text
Rapid dead-tuple growth
Oldest transaction
Autovacuum not keeping up
Large frequently updated tables
Transaction ID age
```

Thresholds should be based on the workload and table characteristics.

---

# 24.18 Database Size Monitoring

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS database_size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

Trend this over time.

The important metric is:

```text
Growth rate
```

not merely current size.

---

# 24.19 Table Growth

Example:

```sql
SELECT
    schemaname,
    relname,
    pg_size_pretty(
        pg_total_relation_size(relid)
    ) AS total_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;
```

This identifies the largest user tables/indexes combined.

---

# 24.20 Capacity Forecasting

Suppose:

```text
Current database = 4 TB
Monthly growth   = 150 GB
```

A simple forecast:

```text
6 months ≈ 4 TB + 900 GB
```

But production forecasting should account for:

```text
Data growth
Index growth
WAL
Temporary files
Maintenance overhead
Backup copies
Free-space requirements
Storage performance
```

---

# 24.21 WAL Monitoring

PostgreSQL 19 exposes WAL statistics through:

```sql
SELECT *
FROM pg_stat_wal;
```

Track:

```text
wal_records
wal_fpi
wal_bytes
wal_buffers_full
```

Trend these metrics rather than evaluating isolated values.

---

# 24.22 WAL Growth Alert

A sudden WAL increase can correlate with:

```text
Large batch operation
Bulk UPDATE
Index rebuild
Application deployment
High write workload
Replication/archive issue
```

The correct response is investigation, not automatically increasing storage.

---

# 24.23 Log Monitoring

PostgreSQL logs should be centrally collected where possible.

Architecture:

```text
PostgreSQL
    |
    v
RHEL journaling / log files
    |
    v
Log collector
    |
    v
Central logging / SIEM
    |
    v
Alerting
```

Monitor for:

```text
FATAL
ERROR
PANIC
deadlock
authentication failures
connection failures
checkpoint issues
archive failures
```

---

# 24.24 RHEL Service Monitoring

Check:

```bash
systemctl status postgresql-19
```

Depending on the RHEL packaging/service layout, the exact service unit name can differ.

First identify it:

```bash
systemctl list-units --type=service | grep -i postgres
```

Then monitor the actual PostgreSQL service unit.

---

# 24.25 RHEL Journal Monitoring

```bash
journalctl --since "1 hour ago" | grep -i postgres
```

For continuous operational troubleshooting:

```bash
journalctl -f
```

Use more targeted filters in production to avoid drowning useful events in unrelated system messages.

---

# 24.26 OS Monitoring

Core commands:

```bash
uptime
free -h
vmstat 5
iostat -xz 5
df -h
df -i
```

Each answers a different question:

| Command  | Primary use                |
| -------- | -------------------------- |
| `uptime` | Load                       |
| `free`   | Memory                     |
| `vmstat` | CPU/memory/system activity |
| `iostat` | Storage                    |
| `df -h`  | Capacity                   |
| `df -i`  | Inode utilization          |

---

# 24.27 Filesystem Capacity

Monitor both:

```text
Disk space
+
Inodes
```

A filesystem can have free GBs but still encounter problems if inode utilization is exhausted.

Check:

```bash
df -h
df -i
```

---

# 24.28 Monitoring WAL Disk

Do not monitor only:

```text
PGDATA total size
```

Also understand:

```text
WAL growth
Archive destination
Temporary files
Backup destination
Filesystem free space
```

A WAL/archive destination filling up can create a serious database incident.

---

# 24.29 Alert Severity Model

A practical model:

```text
P1
Database unavailable
Critical data risk

P2
Major degradation
Replication/recovery risk
Critical capacity issue

P3
Performance degradation
Non-critical failures

P4
Informational
Capacity trend
Maintenance warning
```

The organization should define the final SLA/incident classification.

---

# 24.30 Alert Fatigue

Bad monitoring:

```text
1000 alerts/day
↓
DBA ignores alerts
↓
Real incident missed
```

Better:

```text
Meaningful threshold
+
Correlation
+
Deduplication
+
Severity
+
Actionable message
```

Every alert should ideally answer:

```text
What happened?
Why does it matter?
What should the DBA check?
```

---

# 24.31 Alert Example

Bad:

```text
CPU > 80%
```

Better:

```text
PostgreSQL CPU sustained >80% for 10 minutes
AND
query latency increased
AND
top workload identified
```

Correlated alerts provide more operational value.

---

# 24.32 PostgreSQL Metrics Export

Enterprise environments commonly integrate PostgreSQL with monitoring systems through exporters/agents.

Typical architecture:

```text
PostgreSQL
     |
     v
Metrics Exporter
     |
     v
Metrics Collector
     |
     v
Dashboard / Alert Manager
```

The exact exporter should be selected based on the organization's monitoring platform and security requirements.

---

# 24.33 DBA Automation Philosophy

Automate:

```text
Repeatable
Deterministic
Low-risk
Well-tested
Auditable
```

Do not initially automate:

```text
Ambiguous recovery decisions
Destructive actions
Automatic termination of arbitrary sessions
Uncontrolled configuration changes
```

---

# 24.34 Bash Health-Check Skeleton

Example:

```bash
#!/bin/bash

HOST="127.0.0.1"
PORT="5432"

if pg_isready -h "$HOST" -p "$PORT" >/dev/null 2>&1
then
    echo "PostgreSQL: UP"
else
    echo "PostgreSQL: DOWN"
    exit 1
fi
```

This is deliberately simple.

Enterprise automation should add logging, authentication handling, exit codes, monitoring integration, and error handling.

---

# 24.35 Automated SQL Health Checks

Example:

```bash
psql -d postgres -Atc "
SELECT
    now(),
    version(),
    pg_is_in_recovery();
"
```

This can be integrated into an operational health-check framework.

---

# 24.36 DBA Automation Framework

A mature design:

```text
Scheduler
    |
    v
Automation Script
    |
    +---- PostgreSQL checks
    |
    +---- RHEL checks
    |
    +---- Backup checks
    |
    +---- Capacity checks
    |
    v
Result
    |
 +--+--+
 |     |
PASS  FAIL
 |     |
 v     v
Log   Alert
```

---

# 24.37 Idempotent Automation

A script should safely run repeatedly.

Bad:

```bash
create user
```

every time.

Better:

```text
Check
  ↓
Does desired state exist?
  ↓
No → create
Yes → validate
```

This is the foundation of reliable DBA automation.

---

# 24.38 Configuration Management

For enterprise PostgreSQL, consider managing configuration through:

```text
Ansible
Infrastructure-as-Code
Configuration repositories
Controlled deployment pipelines
```

Example lifecycle:

```text
Git
 ↓
Review
 ↓
Test
 ↓
Approval
 ↓
Deployment
 ↓
Validation
```

---

# 24.39 Configuration Drift

Example:

```text
Production standard:
max_connections = 300
```

Server A:

```text
300
```

Server B:

```text
500
```

Server C:

```text
300
```

This is configuration drift.

Automated compliance checks can detect it.

---

# 24.40 PostgreSQL Configuration Inventory

Capture:

```sql
SELECT
    name,
    setting,
    unit,
    source
FROM pg_settings
ORDER BY name;
```

This is useful for:

```text
Baseline
Audit
Troubleshooting
Change comparison
Configuration drift
```

---

# 24.41 Before/After Change Capture

Before a major configuration change:

```bash
psql -d postgres -c \
"SELECT name, setting, source FROM pg_settings ORDER BY name;" \
> before_config.txt
```

After change:

```bash
psql -d postgres -c \
"SELECT name, setting, source FROM pg_settings ORDER BY name;" \
> after_config.txt
```

Then compare:

```bash
diff before_config.txt after_config.txt
```

This creates a simple operational audit trail.

---

# 24.42 Change Automation

Recommended:

```text
Request
  ↓
Risk assessment
  ↓
Backup/recovery validation
  ↓
Test
  ↓
Approval
  ↓
Implementation
  ↓
Validation
  ↓
Monitoring
  ↓
Documentation
```

Automation should enforce this workflow rather than bypass it.

---

# 24.43 DBA Daily Automation

Automate a daily report containing:

```text
Database availability
Database sizes
Connection utilization
Long transactions
Blocking sessions
Deadlocks
Autovacuum health
Transaction age
WAL statistics
Filesystem capacity
Backup status
Errors
Configuration drift
```

---

# 24.44 DBA Weekly Automation

Weekly report:

```text
Top SQL
Database growth
Table growth
Index growth
Autovacuum trends
Dead tuples
WAL growth
Backup validation
Restore-test results
Capacity forecast
Security events
```

---

# 24.45 DBA Monthly Automation

Monthly:

```text
Capacity forecast
Configuration review
Role review
Database growth
Storage forecast
Performance trend
Backup/recovery test
Incident trend
Patch review
Operational KPI review
```

---

# 24.46 Operational Dashboard

A useful PostgreSQL DBA dashboard:

```text
+-------------------+-------------------+
| Availability      | Connections       |
+-------------------+-------------------+
| Query Latency     | Transactions      |
+-------------------+-------------------+
| Locks             | Autovacuum        |
+-------------------+-------------------+
| WAL               | Storage           |
+-------------------+-------------------+
| Errors            | Backups           |
+-------------------+-------------------+
```

Drill-down should lead from:

```text
Alert
 ↓
Metric
 ↓
Database
 ↓
Session
 ↓
Query
 ↓
Root cause
```

---

# 24.47 SRE-Style PostgreSQL KPIs

Track:

```text
Availability %
P95/P99 query latency
Error rate
Connection saturation
Transaction throughput
Deadlock rate
WAL generation
Storage growth
Backup success %
Restore-test success %
RPO achieved
RTO achieved
```

These provide a much stronger operational picture than simply monitoring CPU.

---

# 24.48 Automation Safety Controls

Every production DBA automation script should ideally have:

```text
Logging
Exit codes
Timeout
Error handling
Dry-run mode where applicable
Configuration validation
Credential protection
Audit trail
Rollback procedure
Alerting
```

---

# 24.49 Automation RCA Example

### Problem

```text
Filesystem reached 100%.
```

### Weak automation

```text
Automatically delete files.
```

### Safe automation

```text
Detect
 ↓
Identify filesystem
 ↓
Identify PostgreSQL-related growth
 ↓
Check WAL/archive/backup conditions
 ↓
Alert DBA
 ↓
Execute approved remediation
```

Never automatically delete PostgreSQL WAL or database files merely because they are old.

---

# 24.50 Phase 24 Production Checklist

```text
[ ] Availability monitoring
[ ] Connection monitoring
[ ] Long-transaction monitoring
[ ] Lock monitoring
[ ] Deadlock monitoring
[ ] Autovacuum monitoring
[ ] Database-growth monitoring
[ ] WAL monitoring
[ ] Filesystem monitoring
[ ] OS monitoring
[ ] PostgreSQL log monitoring
[ ] Centralized logging
[ ] Alert severity model
[ ] Alert deduplication
[ ] Configuration baseline
[ ] Configuration-drift detection
[ ] DBA health-check automation
[ ] Daily report
[ ] Weekly report
[ ] Monthly KPI report
[ ] Automation audit trail
[ ] Recovery/rollback procedures
```

---

# Phase 24 Completed
