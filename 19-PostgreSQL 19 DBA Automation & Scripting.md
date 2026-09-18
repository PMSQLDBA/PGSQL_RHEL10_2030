# Phase 19 — PostgreSQL 19 DBA Automation & Scripting on RHEL 10

This phase focuses on reducing **manual DBA effort** through Bash, `psql`, systemd/cron, health checks, backup monitoring, and automated evidence collection.

> **Environment assumed:** RHEL 10 + PostgreSQL 19.

---

## 19.1 DBA Automation Architecture

```text
                 RHEL 10
                    |
          +---------+---------+
          |                   |
       systemd              cron
          |                   |
          +---------+---------+
                    |
                    v
              Bash Scripts
                    |
             +------+------+
             |             |
           psql        PostgreSQL
             |             |
             +------+------+
                    |
                    v
             Health Reports
                    |
          +---------+---------+
          |                   |
        Logs              Alerts/Email
```

The objective is:

```text
Manual DBA Work
       ↓
Script
       ↓
Schedule
       ↓
Monitor
       ↓
Alert
       ↓
DBA Intervention only when required
```

---

# 19.2 Standard DBA Script Directory

Create a dedicated structure:

```bash
sudo mkdir -p /opt/pgsql-dba/{bin,config,logs,reports,backup}
```

Set ownership:

```bash
sudo chown -R postgres:postgres /opt/pgsql-dba
```

Recommended layout:

```text
/opt/pgsql-dba/
├── bin/
│   ├── pg_health_check.sh
│   ├── pg_backup_check.sh
│   ├── pg_replication_check.sh
│   ├── pg_lock_check.sh
│   └── pg_disk_check.sh
│
├── config/
│   └── dba.conf
│
├── logs/
├── reports/
└── backup/
```

---

# 19.3 PostgreSQL Environment Configuration

Create:

```bash
sudo -u postgres vi /opt/pgsql-dba/config/dba.conf
```

Example:

```bash
PGHOST="localhost"
PGPORT="5432"
PGUSER="postgres"

PGDATA="/var/lib/pgsql/19/data"
PGLOG="/opt/pgsql-dba/logs"

ALERT_DISK_PERCENT=80
ALERT_CONNECTION_PERCENT=80
ALERT_REPLICATION_LAG_SECONDS=60
```

Keep environment-specific values in configuration rather than hard-coding them into every script.

---

# 19.4 Basic Connectivity Test

```bash
sudo -u postgres psql \
    -h localhost \
    -p 5432 \
    -d postgres \
    -c "SELECT now();"
```

Expected:

```text
              now
-------------------------------
 2026-09-17 ...
```

If this fails, investigate:

```text
PostgreSQL service
      ↓
Listener
      ↓
Authentication
      ↓
pg_hba.conf
```

---

# 19.5 PostgreSQL Service Health Script

Create:

```bash
sudo -u postgres vi /opt/pgsql-dba/bin/pg_service_check.sh
```

Script:

```bash
#!/bin/bash

SERVICE="postgresql-19"

if systemctl is-active --quiet "$SERVICE"; then
    echo "$(date '+%Y-%m-%d %H:%M:%S') PostgreSQL is UP"
    exit 0
else
    echo "$(date '+%Y-%m-%d %H:%M:%S') PostgreSQL is DOWN"
    exit 1
fi
```

Make executable:

```bash
sudo chmod 750 /opt/pgsql-dba/bin/pg_service_check.sh
```

Test:

```bash
sudo /opt/pgsql-dba/bin/pg_service_check.sh
```

---

# 19.6 Database Connectivity Health Check

```bash
sudo -u postgres psql \
    -d postgres \
    -tAc "SELECT 1;"
```

Expected:

```text
1
```

Automation:

```bash
RESULT=$(sudo -u postgres psql -d postgres -tAc "SELECT 1;" 2>/dev/null)

if [ "$RESULT" = "1" ]; then
    echo "PostgreSQL connectivity: PASS"
else
    echo "PostgreSQL connectivity: FAIL"
    exit 1
fi
```

---

# 19.7 Server Health Check

Create:

```text
/opt/pgsql-dba/bin/pg_health_check.sh
```

Recommended checks:

```text
1. PostgreSQL service
2. PostgreSQL connection
3. Version
4. Database availability
5. Connection utilization
6. Long-running transactions
7. Blocking sessions
8. Replication
9. WAL
10. Archive status
11. Autovacuum
12. Disk utilization
13. Memory
14. CPU
```

---

# 19.8 Database Inventory

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS size,
    datallowconn
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

Automate this into:

```text
/opt/pgsql-dba/reports/database_inventory.txt
```

---

# 19.9 Database Size Monitoring

```sql
SELECT
    datname,
    pg_database_size(datname) AS bytes,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

This allows you to track:

```text
Current size
Previous size
Growth
Growth percentage
```

For a large production database, growth-rate monitoring is often more useful than a single absolute-size threshold.

---

# 19.10 Disk Monitoring

RHEL:

```bash
df -h
```

For PostgreSQL:

```bash
df -h /var/lib/pgsql/19/data
```

Automation:

```bash
df -P /var/lib/pgsql/19/data | awk 'NR==2 {print $5}'
```

Remove `%`:

```bash
df -P /var/lib/pgsql/19/data |
awk 'NR==2 {gsub("%","",$5); print $5}'
```

Then:

```bash
if [ "$USAGE" -ge 80 ]; then
    echo "WARNING: PostgreSQL filesystem usage is ${USAGE}%"
fi
```

---

# 19.11 Inode Monitoring

Don't monitor only:

```text
Disk space
```

Also monitor:

```bash
df -ih
```

Because:

```text
Free GB
+
Free inodes
```

are separate resources.

---

# 19.12 Connection Monitoring

```sql
SELECT
    count(*) AS current_connections,
    current_setting('max_connections')::int AS max_connections
FROM pg_stat_activity;
```

Percentage:

```sql
SELECT
    round(
        100.0 * count(*) /
        current_setting('max_connections')::int,
        2
    ) AS connection_usage_percent
FROM pg_stat_activity;
```

Alert example:

```text
>80%  → WARNING
>90%  → CRITICAL
```

Thresholds should be tuned to the environment rather than treated as universal PostgreSQL defaults.

---

# 19.13 Long-Running Query Monitoring

```sql
SELECT
    pid,
    usename,
    datname,
    now() - query_start AS duration,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND query_start IS NOT NULL
  AND now() - query_start > interval '10 minutes'
ORDER BY query_start;
```

Automation:

```text
No rows
   ↓
PASS

Rows returned
   ↓
WARNING
```

---

# 19.14 Long-Running Transactions

```sql
SELECT
    pid,
    usename,
    datname,
    now() - xact_start AS transaction_age,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Especially investigate:

```text
idle in transaction
```

---

# 19.15 Blocking Session Monitoring

```sql
SELECT
    pid,
    pg_blocking_pids(pid) AS blocking_pids,
    usename,
    datname,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

Automation:

```text
No blocked sessions → PASS

Blocked sessions → ALERT
```

---

# 19.16 Replication Health Script

Primary:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

Check:

```text
state
sync_state
write_lag
flush_lag
replay_lag
```

---

# 19.17 WAL Monitoring

```sql
SELECT
    wal_records,
    wal_fpi,
    wal_bytes,
    wal_buffers_full,
    stats_reset
FROM pg_stat_wal;
```

Track the trend rather than interpreting a single counter value in isolation.

---

# 19.18 WAL Archive Monitoring

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

Automation can alert when:

```text
failed_count increases
```

or when the archive has not progressed within an environment-specific SLA.

---

# 19.19 Replication Slot Monitoring

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    wal_status,
    safe_wal_size
FROM pg_replication_slots;
```

Pay particular attention to inactive slots retaining WAL.

A dangerous situation is:

```text
Inactive slot
     ↓
WAL retained
     ↓
pg_wal growth
     ↓
Disk exhaustion
```

---

# 19.20 Autovacuum Monitoring

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

PostgreSQL 19 additionally provides `pg_stat_autovacuum_scores`, which can be incorporated into automated monitoring. ([postgresql.org](https://www.postgresql.org/docs/19/monitoring-stats.html?utm_source=chatgpt.com))

---

# 19.21 Backup Automation

A production backup framework should include:

```text
Backup
  ↓
Backup completion verification
  ↓
Backup integrity validation
  ↓
Retention
  ↓
Restore testing
  ↓
Monitoring
```

Do **not** define backup success simply as:

```text
backup file exists
```

---

# 19.22 PostgreSQL Logical Backup

Database:

```bash
pg_dump \
    -d appdb \
    -F c \
    -f /backup/appdb_$(date +%Y%m%d_%H%M%S).dump
```

`-F c` creates PostgreSQL's custom archive format, which can be restored with `pg_restore`.

---

# 19.23 Restore Test

Create a separate validation database:

```bash
createdb backup_validation
```

Restore:

```bash
pg_restore \
    -d backup_validation \
    /backup/appdb_YYYYMMDD_HHMMSS.dump
```

Then:

```bash
psql -d backup_validation \
    -c "SELECT count(*) FROM important_table;"
```

This is far more meaningful than merely checking whether the backup file exists.

---

# 19.24 Physical Backup

For production PostgreSQL environments, physical backup tooling should be designed around your:

```text
RPO
RTO
Database size
WAL volume
Retention
Storage
DR requirements
```

PostgreSQL's native physical backup interfaces include `pg_basebackup` and continuous WAL archiving. ([postgresql.org](https://www.postgresql.org/docs/19/backup.html?utm_source=chatgpt.com))

Example:

```bash
pg_basebackup \
    -D /backup/base \
    -Fp \
    -Xs \
    -P
```

Do not use this example as a complete production backup architecture without configuring WAL retention/archive and recovery requirements.

---

# 19.25 Cron Automation

Edit PostgreSQL user's crontab:

```bash
sudo -u postgres crontab -e
```

Example:

```cron
*/5 * * * * /opt/pgsql-dba/bin/pg_health_check.sh >> /opt/pgsql-dba/logs/health.log 2>&1
```

Meaning:

```text
Every 5 minutes
       ↓
Health check
       ↓
Log result
```

---

# 19.26 Why systemd Timers Can Be Better

For enterprise RHEL environments, systemd timers provide:

```text
Service integration
Logging
Dependency handling
Execution status
Better operational visibility
```

Architecture:

```text
pg-health.service
       ↑
       |
pg-health.timer
       |
       v
systemd
```

---

# 19.27 Example systemd Service

Create:

```bash
sudo vi /etc/systemd/system/pg-health.service
```

```ini
[Unit]
Description=PostgreSQL DBA Health Check
After=postgresql-19.service

[Service]
Type=oneshot
User=postgres
ExecStart=/opt/pgsql-dba/bin/pg_health_check.sh
```

---

# 19.28 Example systemd Timer

```bash
sudo vi /etc/systemd/system/pg-health.timer
```

```ini
[Unit]
Description=Run PostgreSQL DBA Health Check

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min
Unit=pg-health.service

[Install]
WantedBy=timers.target
```

Reload:

```bash
sudo systemctl daemon-reload
```

Enable:

```bash
sudo systemctl enable --now pg-health.timer
```

Check:

```bash
systemctl list-timers pg-health.timer
```

---

# 19.29 Central Health Report

Your automated report should ideally contain:

```text
==================================================
POSTGRESQL DBA HEALTH REPORT
==================================================

Server:
RHEL Version:
PostgreSQL Version:
Date/Time:

SERVICE
-------
Status:

DATABASE
--------
Database count:
Largest database:

CONNECTIONS
-----------
Current:
Maximum:
Usage:

LONG RUNNING QUERIES
--------------------
Count:

BLOCKING
--------
Count:

REPLICATION
-----------
Status:
Lag:

WAL
---
WAL status:
Archive status:

AUTOVACUUM
----------
Dead tuples:
Autovacuum status:

STORAGE
-------
PGDATA:
Usage:

RESULT
------
PASS / WARNING / CRITICAL
==================================================
```

---

# 19.30 Exit Codes

Standardize your scripts:

```text
0 = PASS
1 = WARNING
2 = CRITICAL
```

Example:

```bash
exit 0
```

for healthy.

This makes the scripts easier to integrate with monitoring systems.

---

# 19.31 Logging Standard

Every script should record:

```text
Timestamp
Hostname
PostgreSQL version
Script name
Result
Error
Execution duration
```

Example:

```text
2026-09-17 20:15:01 | db01 | pg_health_check | PASS
```

---

# 19.32 Automated Incident Evidence Collection

Create:

```text
pg_collect_evidence.sh
```

Collect:

```text
OS version
Kernel
CPU
Memory
Disk
PostgreSQL version
Service status
PostgreSQL logs
Active sessions
Locks
Replication
WAL
Archiving
Database sizes
```

Output:

```text
/opt/pgsql-dba/reports/
    incident_20260917_201500/
```

This is extremely useful during production incidents.

---

# 19.33 Automated RCA Evidence Flow

```text
ALERT
  |
  v
Health Check
  |
  v
Failure detected
  |
  v
Evidence collection
  |
  +--> PostgreSQL
  +--> OS
  +--> Storage
  +--> Network
  +--> Replication
  |
  v
Timestamped report
  |
  v
DBA investigation
```

The automation should **collect evidence**, not automatically execute destructive remediation.

---

# 19.34 What NOT to Automate Blindly

Avoid automatic:

```text
DROP DATABASE
DROP TABLE
DROP REPLICATION SLOT
pg_terminate_backend()
VACUUM FULL
REINDEX DATABASE
DELETE WAL
Restart PostgreSQL
```

unless the action has a carefully designed, tested operational control.

Automation should normally be:

```text
Detect
+
Collect
+
Alert
```

before:

```text
Modify
+
Terminate
+
Restart
```

---

# 19.35 PostgreSQL DBA Automation Levels

| Level | Automation                    |
| ----- | ----------------------------- |
| L1    | Manual commands               |
| L2    | Bash scripts                  |
| L3    | Scheduled scripts             |
| L4    | Monitoring + alerts           |
| L5    | Automated evidence collection |
| L6    | Controlled remediation        |
| L7    | Self-healing                  |

For production, build progressively.

---

# 19.36 Recommended DBA Automation Stack

```text
RHEL 10
   |
   +-- Bash
   |
   +-- psql
   |
   +-- systemd
   |
   +-- PostgreSQL statistics
   |
   +-- pg_stat_statements
   |
   +-- PostgreSQL logs
   |
   +-- OS metrics
   |
   +-- Monitoring platform
```

The database remains the authoritative source for PostgreSQL-specific health information, while RHEL provides OS/resource evidence.

---

# 19.37 Automation Best Practices

### DO

```text
Use configuration files
Use absolute paths
Validate commands
Use exit codes
Timestamp logs
Run with least privilege
Test scripts manually
Test in non-production
Version-control scripts
Capture evidence
```

### DON'T

```text
Hard-code passwords
Run everything as root
Ignore errors
Delete logs
Automatically kill sessions
Automatically restart production
Assume backup = recoverable
```

---

# 19.38 Password Security

Avoid:

```bash
psql postgresql://postgres:Password123@localhost/postgres
```

Passwords can become exposed through:

```text
Shell history
Process listings
Logs
Scripts
```

Prefer PostgreSQL-supported authentication mechanisms and protected credential storage such as `.pgpass` where appropriate.

Ensure:

```bash
chmod 600 ~/.pgpass
```

PostgreSQL documents `.pgpass` as a supported password-file mechanism with strict file-permission requirements. ([postgresql.org](https://www.postgresql.org/docs/19/libpq-pgpass.html?utm_source=chatgpt.com))

---

# 19.39 Version Control

Put DBA scripts under Git:

```text
postgresql-dba-automation/
├── health/
├── backup/
├── replication/
├── monitoring/
├── troubleshooting/
├── reports/
└── README.md
```

Version:

```text
v1.0
v1.1
v2.0
```

This prevents the common enterprise problem of:

```text
"Which version of the DBA script did we run?"
```

---

# 19.40 Final DBA Automation Model

```text
                    PostgreSQL 19
                          |
             +------------+------------+
             |                         |
          Statistics                 Logs
             |                         |
             +------------+------------+
                          |
                          v
                    DBA Scripts
                          |
              +-----------+-----------+
              |           |           |
             Bash       psql       systemd
              |           |           |
              +-----------+-----------+
                          |
                          v
                    Health Report
                          |
                    +-----+-----+
                    |           |
                  PASS        ALERT
                                |
                                v
                          DBA Investigation
                                |
                                v
                              RCA
```

---

# Phase 19 Completed

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
17  Upgrades / Patching / Migration
18  Advanced Troubleshooting & RCA
19  DBA Automation & Scripting       ← COMPLETED
```

## Next — Phase 20

**PostgreSQL 19 Enterprise HA-DR Architecture**

This will cover **Primary → Standby architecture, synchronous vs asynchronous replication, quorum, cascading replication, replication slots, failover, switchover, split-brain prevention, fencing, RPO/RTO, DR site design, connection routing, DNS/VIP considerations, Patroni/repmgr concepts, and a complete production HA/DR runbook.**
