# Phase 14 — PostgreSQL 19 Monitoring & Enterprise DBA Health Check SOP

This phase turns the previous performance-tuning concepts into a **repeatable DBA health-check process** for PostgreSQL 19 on **RHEL 10**.

The goal is to answer, quickly:

> **Is PostgreSQL healthy, what is currently wrong, and what should the DBA investigate first?**

---

## 14.1 Enterprise Health-Check Architecture

```text
                    PostgreSQL 19
                          |
        +-----------------+------------------+
        |                 |                  |
        v                 v                  v
   Availability       Performance        Reliability
        |                 |                  |
   Service status     SQL performance    Backup
   Connections        I/O               WAL
   Database status    CPU               Replication
   Errors             Memory            Autovacuum
        |                 |                  |
        +-----------------+------------------+
                          |
                          v
                  DBA HEALTH REPORT
                          |
            +-------------+-------------+
            |             |             |
           PASS          WARN        CRITICAL
```

---

# 14.2 Health-Check Categories

A production DBA health check should cover at least:

| Category      | What to check                   |
| ------------- | ------------------------------- |
| PostgreSQL    | Version, uptime, configuration  |
| Availability  | Service/database availability   |
| Connections   | Current/max connections         |
| Sessions      | Active/idle/idle-in-transaction |
| Blocking      | Blocked and blocking sessions   |
| Transactions  | Long-running transactions       |
| Performance   | Expensive SQL                   |
| I/O           | Read/write activity and latency |
| Memory        | PostgreSQL/OS memory pressure   |
| WAL           | WAL generation and archiving    |
| Checkpoints   | Timed/requested checkpoints     |
| Replication   | Primary/standby state and lag   |
| Vacuum        | Autovacuum/VACUUM activity      |
| Bloat         | Dead tuples and table growth    |
| Statistics    | Analyze freshness               |
| Indexes       | Usage and suspicious indexes    |
| Database size | Growth                          |
| Backups       | Last successful backup          |
| Recovery      | Restore/PITR validation         |
| Security      | Roles, authentication, exposure |
| OS            | CPU, RAM, disk, filesystem      |

---

# 14.3 Step 1 — PostgreSQL Availability

On RHEL:

```bash
sudo systemctl status postgresql-19
```

Check:

```bash
sudo systemctl is-active postgresql-19
```

Expected:

```text
active
```

Check startup configuration:

```bash
sudo systemctl is-enabled postgresql-19
```

---

# 14.4 PostgreSQL Connectivity

```bash
sudo -u postgres psql -d postgres -c "SELECT now();"
```

Expected:

```text
current timestamp
```

If this fails:

```text
Service
   ↓
Port
   ↓
Listener
   ↓
Authentication
   ↓
Database
```

Investigate in that order.

---

# 14.5 Check PostgreSQL Port

```bash
sudo ss -lntp | grep 5432
```

Expected example:

```text
LISTEN ... 5432 ... postgres
```

Also:

```bash
sudo ss -lntp | grep postgres
```

---

# 14.6 Check PostgreSQL Listener

```sql
SHOW listen_addresses;
SHOW port;
```

Check:

```sql
SELECT
    inet_server_addr(),
    inet_server_port(),
    version();
```

For a server intended to accept remote connections, verify that `listen_addresses` and `pg_hba.conf` match the intended network design.

---

# 14.7 Connection Health

```sql
SELECT
    count(*) AS current_connections,
    current_setting('max_connections')::int AS max_connections
FROM pg_stat_activity;
```

Calculate utilization:

```sql
SELECT
    count(*) AS current_connections,
    current_setting('max_connections')::int AS max_connections,
    ROUND(
        100.0 * count(*) /
        current_setting('max_connections')::numeric,
        2
    ) AS connection_utilization_pct
FROM pg_stat_activity;
```

### Example interpretation

```text
30 / 200
```

is very different from:

```text
195 / 200
```

A server approaching its connection limit requires investigation.

---

# 14.8 Connection Breakdown

```sql
SELECT
    state,
    count(*) AS sessions
FROM pg_stat_activity
GROUP BY state
ORDER BY sessions DESC;
```

Typical states:

```text
active
idle
idle in transaction
idle in transaction (aborted)
```

Pay particular attention to:

```text
idle in transaction
idle in transaction (aborted)
```

---

# 14.9 Idle-in-Transaction Health Check

```sql
SELECT
    pid,
    usename,
    datname,
    client_addr,
    application_name,
    now() - xact_start AS transaction_age,
    now() - state_change AS idle_age,
    LEFT(query, 500) AS query
FROM pg_stat_activity
WHERE state LIKE 'idle in transaction%'
ORDER BY xact_start;
```

Potential impact:

```text
Open transaction
       ↓
Old snapshot
       ↓
VACUUM cannot remove some dead tuples
       ↓
Table growth
       ↓
Performance degradation
```

---

# 14.10 Long-Running Transactions

```sql
SELECT
    pid,
    usename,
    datname,
    now() - xact_start AS transaction_age,
    state,
    wait_event_type,
    wait_event,
    LEFT(query, 500) AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

A long transaction is **not automatically a problem**.

Investigate:

```text
Duration
+
Transaction purpose
+
Application
+
Locks
+
Vacuum impact
```

---

# 14.11 Blocking Detection

Use:

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocked.datname AS blocked_database,
    now() - blocked.query_start AS blocked_duration,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    now() - blocking.query_start AS blocking_duration,
    LEFT(blocked.query, 500) AS blocked_query,
    LEFT(blocking.query, 500) AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
ORDER BY blocked.query_start;
```

---

# 14.12 Blocking RCA

```text
Blocked Session
      |
      v
Find blocking PID
      |
      v
Find transaction
      |
      v
Identify application
      |
      v
Identify SQL
      |
      v
Determine why transaction remains open
      |
      v
Correct application/transaction behavior
```

Do **not** make:

```text
blocked = kill session
```

your default troubleshooting procedure.

---

# 14.13 Database Size

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS database_size
FROM pg_database
WHERE datallowconn
ORDER BY pg_database_size(datname) DESC;
```

For a single database:

```sql
SELECT pg_size_pretty(pg_database_size(current_database()));
```

---

# 14.14 Database Growth

Capture the result periodically.

Example:

```text
2026-09-01 → 500 GB
2026-09-08 → 525 GB
2026-09-15 → 570 GB
```

Growth:

```text
70 GB / 14 days
```

This is much more useful than knowing only the current size.

For enterprise monitoring, store daily measurements in a monitoring repository.

---

# 14.15 Largest Tables

```sql
SELECT
    schemaname,
    relname,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(
        pg_indexes_size(relid)
    ) AS indexes_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;
```

This tells you:

```text
Table
+
Table data
+
Indexes
```

---

# 14.16 Dead Tuple Monitoring

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

Look for:

```text
High n_dead_tup
+
old last_autovacuum
+
high UPDATE/DELETE workload
```

---

# 14.17 Autovacuum Health

```sql
SELECT
    schemaname,
    relname,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Investigate tables where:

```text
dead tuples increasing
```

but:

```text
autovacuum not keeping up
```

---

# 14.18 PostgreSQL 19 Autovacuum Monitoring

PostgreSQL 19 adds `pg_stat_autovacuum_scores`, which provides additional visibility into autovacuum prioritization.

Check:

```sql
SELECT *
FROM pg_stat_autovacuum_scores;
```

Use this together with:

```text
pg_stat_user_tables
+
autovacuum logs
+
transaction age
+
table modification rate
```

rather than relying on one metric.

---

# 14.19 Vacuum Progress

When a vacuum is actively running:

```sql
SELECT
    pid,
    datname,
    relid::regclass AS table_name,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    heap_blks_vacuumed,
    index_vacuum_count,
    num_dead_tuples
FROM pg_stat_progress_vacuum;
```

This is useful during an active maintenance operation.

---

# 14.20 Index Health

```sql
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 30;
```

Do not automatically remove indexes with low usage.

Validate:

```text
Usage
+
Uniqueness
+
Foreign keys
+
Application workload
+
Reporting workload
+
Statistics collection period
```

---

# 14.21 Expensive SQL

If `pg_stat_statements` is enabled:

```sql
SELECT
    queryid,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    rows,
    shared_blks_read,
    shared_blks_hit,
    temp_blks_read,
    temp_blks_written,
    LEFT(query, 500) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

Create separate reports for:

```text
Highest total execution time
Highest average execution time
Highest calls
Highest shared reads
Highest temporary I/O
```

---

# 14.22 I/O Health — PostgreSQL 19

```sql
SELECT
    backend_type,
    object,
    context,
    reads,
    writes,
    extends,
    hits,
    read_time,
    write_time
FROM pg_stat_io
ORDER BY
    COALESCE(read_time, 0) +
    COALESCE(write_time, 0) DESC;
```

This should be correlated with Linux:

```bash
iostat -xz 1 10
```

and:

```bash
vmstat 1 10
```

---

# 14.23 WAL Health

```sql
SELECT *
FROM pg_stat_wal;
```

Important areas include:

```text
WAL records
WAL bytes
WAL buffers full
WAL writes
WAL syncs
```

High WAL generation can be normal for a write-heavy system.

The important question is:

> **Has WAL generation changed unexpectedly relative to the normal workload?**

---

# 14.24 WAL Archive Health

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

Health logic:

```text
archived_count increasing
+
last_archived_time recent
+
failed_count not increasing
```

---

# 14.25 WAL Archive Alert

Potential warning:

```text
last_archived_time
```

is unexpectedly old.

Potential critical condition:

```text
failed_count increasing
```

or the archive destination is unavailable.

Investigate:

```bash
df -h
df -i
```

Then:

```bash
sudo journalctl -u postgresql-19 --since "1 hour ago"
```

---

# 14.26 Checkpoint Health

```sql
SELECT
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint
FROM pg_stat_bgwriter;
```

Monitor the ratio between:

```text
checkpoints_req
```

and:

```text
checkpoints_timed
```

A large number of requested checkpoints may indicate WAL pressure or configuration/workload issues.

---

# 14.27 Replication Health

On a primary:

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    sync_state,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

This provides visibility into connected physical replication clients.

---

# 14.28 Replication Slot Health

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    confirmed_flush_lsn,
    wal_status,
    safe_wal_size
FROM pg_replication_slots;
```

Pay special attention to inactive slots.

Potential problem:

```text
Replication slot inactive
       ↓
WAL retained
       ↓
pg_wal grows
       ↓
Disk fills
       ↓
Database outage
```

---

# 14.29 Replication Slot RCA

If `pg_wal` grows unexpectedly:

```text
Check disk
   ↓
Check replication slots
   ↓
Check standby
   ↓
Check archive
   ↓
Check WAL generation
```

Do **not** simply delete replication slots without understanding why they exist.

---

# 14.30 Transaction ID Age

Check:

```sql
SELECT
    datname,
    age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY xid_age DESC;
```

Also inspect database-level vacuum-related information.

Transaction ID wraparound protection is one of PostgreSQL's most important maintenance requirements.

---

# 14.31 Security Health Check

List roles:

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

Investigate unnecessary:

```text
SUPERUSER
CREATEROLE
CREATEDB
REPLICATION
```

privileges.

---

# 14.32 Login Roles

```sql
SELECT
    rolname,
    rolcanlogin,
    rolvaliduntil
FROM pg_roles
WHERE rolcanlogin
ORDER BY rolname;
```

Check expired accounts:

```sql
SELECT
    rolname,
    rolvaliduntil
FROM pg_roles
WHERE rolcanlogin
  AND rolvaliduntil IS NOT NULL
ORDER BY rolvaliduntil;
```

---

# 14.33 Configuration Audit

Important:

```sql
SELECT
    name,
    setting,
    unit,
    source,
    sourcefile,
    sourceline,
    pending_restart
FROM pg_settings
WHERE source <> 'default'
ORDER BY name;
```

This gives you configuration overrides.

Pay attention to:

```text
pending_restart = true
```

because the configuration change has not yet taken effect.

---

# 14.34 Check Pending Restart

```sql
SELECT
    name,
    setting,
    pending_restart
FROM pg_settings
WHERE pending_restart
ORDER BY name;
```

If results exist:

```text
Configuration changed
        ↓
Restart required
        ↓
Plan maintenance window
```

Do not blindly restart production.

---

# 14.35 OS Health Check

RHEL:

```bash
uptime
```

```bash
free -h
```

```bash
df -h
```

```bash
df -i
```

```bash
iostat -xz 1 10
```

```bash
vmstat 1 10
```

CPU:

```bash
mpstat -P ALL 1 10
```

Processes:

```bash
ps -ef | grep '[p]ostgres'
```

---

# 14.36 Disk Health

The PostgreSQL DBA should always check:

```bash
df -h
```

and:

```bash
df -i
```

because either:

```text
Disk capacity exhausted
```

or:

```text
Inode exhaustion
```

can prevent PostgreSQL from functioning normally.

---

# 14.37 PostgreSQL Log Health

On systemd-based RHEL:

```bash
sudo journalctl -u postgresql-19 --since "24 hours ago"
```

Search errors:

```bash
sudo journalctl -u postgresql-19 --since "24 hours ago" |
grep -Ei 'error|fatal|panic|could not|failed'
```

Do not interpret every `ERROR` as a server failure; some errors are expected application-level events.

---

# 14.38 Backup Health

For pgBackRest:

```bash
sudo -u postgres pgbackrest \
    --stanza=main \
    check
```

Then:

```bash
sudo -u postgres pgbackrest \
    --stanza=main \
    info
```

You want to know:

```text
Last full backup
Last differential
Last incremental
Backup size
Repository status
Archive status
Retention
```

---

# 14.39 Restore-Test Health

Your monitoring report should include:

```text
Last successful backup:
Last successful restore test:
Last PITR test:
Last DR test:
Measured RTO:
Measured RPO:
```

This is much more meaningful than:

```text
Backup job = SUCCESS
```

---

# 14.40 Single DBA Health Query

Here is a useful starting point for a daily PostgreSQL report:

```sql
SELECT
    now() AS check_time,
    current_database() AS database_name,
    version() AS postgres_version,
    current_setting('server_version') AS server_version,
    current_setting('max_connections') AS max_connections,
    (
        SELECT count(*)
        FROM pg_stat_activity
    ) AS current_connections,
    (
        SELECT count(*)
        FROM pg_stat_activity
        WHERE state = 'active'
    ) AS active_sessions,
    (
        SELECT count(*)
        FROM pg_stat_activity
        WHERE state = 'idle in transaction'
    ) AS idle_in_transaction,
    pg_size_pretty(
        pg_database_size(current_database())
    ) AS database_size;
```

---

# 14.41 DBA Health Report — PASS/WARN/CRITICAL

Use a simple framework:

```text
GREEN
PASS
     ↓
Normal

YELLOW
WARN
     ↓
Investigate

RED
CRITICAL
     ↓
Immediate DBA action
```

Example:

| Check             | PASS    | WARN     | CRITICAL           |
| ----------------- | ------- | -------- | ------------------ |
| PostgreSQL        | Running | —        | Down               |
| Connections       | Normal  | High     | Exhausted          |
| Blocking          | None    | Some     | Severe/long        |
| WAL archive       | Healthy | Delayed  | Failed             |
| Disk              | Normal  | >70–80%  | Near/full          |
| Replication       | Healthy | Lagging  | Broken             |
| Autovacuum        | Healthy | Behind   | Severe             |
| Backup            | Current | Delayed  | Missing            |
| Restore test      | Recent  | Old      | Never tested       |
| Long transaction  | None    | Long     | Severe             |
| XID age           | Normal  | Elevated | Critical           |
| Replication slots | Healthy | Inactive | WAL retention risk |

These thresholds should be **defined for your environment**, not copied blindly from another production system.

---

# 14.42 Daily DBA Health-Check Workflow

```text
08:00
  |
  v
PostgreSQL service
  |
  v
Connectivity
  |
  v
Disk / filesystem
  |
  v
Connections
  |
  v
Blocking
  |
  v
Long transactions
  |
  v
WAL / archive
  |
  v
Replication
  |
  v
Autovacuum
  |
  v
Expensive SQL
  |
  v
Database growth
  |
  v
Backup
  |
  v
Security/configuration
  |
  v
HEALTH REPORT
```

---

# 14.43 Enterprise Incident Priority

### P1 — Immediate

Examples:

```text
PostgreSQL unavailable
Disk full
Data corruption
Primary + DR unavailable
WAL archive completely broken with imminent storage risk
```

### P2 — High

```text
Replication severely behind
Backup failures
Severe blocking
Connection exhaustion
Rapid unexplained WAL growth
```

### P3 — Medium

```text
Autovacuum falling behind
Query performance degradation
Increasing bloat
Growing database unusually fast
```

### P4 — Normal

```text
Configuration optimization
Index review
Capacity planning
Routine statistics maintenance
```

---

# 14.44 DBA Morning Report

A practical report can look like:

```text
=========================================================
 PostgreSQL 19 DAILY DBA HEALTH CHECK
=========================================================

SERVER
Status                 : PASS
Version                : PostgreSQL 19
Uptime                 : XX days

CONNECTIONS
Current                : XX
Maximum                : XXX
Utilization            : XX%

SESSIONS
Active                 : XX
Idle                   : XX
Idle in Transaction    : XX

BLOCKING
Blocked Sessions       : XX
Blocking Sessions      : XX

DATABASE
Largest Database       : XXX GB
Growth                 : XX GB/day

WAL
Archive Status         : PASS
Archive Failures       : 0

REPLICATION
Standby Status         : PASS
Replay Lag             : XX

VACUUM
Autovacuum Status      : PASS
High Dead Tuples       : XX

PERFORMANCE
Top Query              : Query ID XXXXX
Average Execution      : XXX ms
Total Execution        : XXX sec

STORAGE
Filesystem Usage       : XX%
PostgreSQL Data        : XXX GB

BACKUP
Last Full              : YYYY-MM-DD HH:MM
Last Incremental       : YYYY-MM-DD HH:MM
Backup Check           : PASS

RESTORE
Last Restore Test      : YYYY-MM-DD
PITR Test              : PASS

SECURITY
Unexpected Superusers  : 0

OVERALL
STATUS                 : PASS / WARN / CRITICAL
=========================================================
```

---

# 14.45 Most Important PostgreSQL 19 Views for a DBA

Memorize these:

```text
pg_stat_activity
pg_stat_database
pg_stat_user_tables
pg_stat_user_indexes
pg_stat_io
pg_stat_wal
pg_stat_archiver
pg_stat_replication
pg_replication_slots
pg_stat_progress_vacuum
pg_stat_progress_basebackup
pg_stat_statements
pg_settings
pg_roles
```

These views form a major part of day-to-day PostgreSQL troubleshooting and monitoring.

---

# 14.46 Complete DBA Troubleshooting Model

```text
                         INCIDENT
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
         AVAILABLE?      SLOW?          ERROR?
             |              |              |
             v              v              v
          Service       SQL/Waits       Logs
          Listener      Locks           RCA
          Network       I/O
                         CPU
                         Memory
             |              |
             +--------------+
                    |
                    v
               PostgreSQL
                Evidence
                    |
                    v
                Root Cause
                    |
                    v
              Corrective Action
                    |
                    v
                 Validate
                    |
                    v
               Prevent Repeat
```

---

## Phase 14 — DBA Golden Rules

1. **Monitor symptoms and evidence, not just server availability.**
2. **`pg_stat_activity` is your first stop for live incidents.**
3. **Use `pg_blocking_pids()` for lock investigation.**
4. **Use `pg_stat_statements` for workload-level SQL analysis.**
5. **Use `pg_stat_io` plus Linux I/O metrics for storage analysis.**
6. **Monitor replication slots because abandoned slots can retain WAL.**
7. **Monitor autovacuum and long-running transactions together.**
8. **Monitor WAL archiving continuously.**
9. **Backup success does not prove recoverability.**
10. **A restore test is the strongest evidence that your backup strategy works.**
11. **Never make a configuration change without measuring the before/after state.**
12. **Keep a baseline so that "normal" is defined for your specific workload.**

### Your PostgreSQL 19 DBA progression so far

```text
Phase 1  → RHEL 10 / PostgreSQL installation
Phase 2  → PostgreSQL architecture
Phase 3  → Configuration
Phase 4  → Users / roles / security
Phase 5  → Database & schema administration
Phase 6  → SQL / DML / DDL
Phase 7  → Storage / tablespaces
Phase 8  → WAL
Phase 9  → HA / replication
Phase 10 → Backup concepts
Phase 11 → Recovery / PITR
Phase 12 → Backup & Recovery Master SOP
Phase 13 → Performance Tuning
Phase 14 → Monitoring & Health Check
             ↓
        NEXT
             ↓
Phase 15 → PostgreSQL 19
           High Availability &
           Streaming Replication
           + Failover
           + Switchover
           + Replication Slots
           + Synchronous Replication
           + Quorum
           + Cascading Replicas
           + Patroni architecture
           + HAProxy
           + etcd
           + Automated Failover
```
