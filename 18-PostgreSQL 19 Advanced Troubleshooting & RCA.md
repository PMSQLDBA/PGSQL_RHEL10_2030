# Phase 18 — PostgreSQL 19 Advanced Troubleshooting & RCA Master SOP

This phase moves from **routine administration to production incident handling**.

One PostgreSQL 19-specific point is important: PostgreSQL 19 adds `pg_stat_lock`, `pg_stat_recovery`, and `pg_stat_autovacuum_scores`, and enables `log_lock_waits` by default. These are useful additions for DBA troubleshooting. ([PostgreSQL][1])

---

## 18.1 Production DBA Troubleshooting Methodology

When an incident occurs, don't immediately restart PostgreSQL.

Use:

```text
                    INCIDENT
                       |
                       v
                DEFINE SYMPTOM
                       |
                       v
                CHECK SCOPE
                       |
          +------------+------------+
          |            |            |
          v            v            v
       CLIENT        DATABASE       OS
          |            |            |
          +------------+------------+
                       |
                       v
                 IDENTIFY WAIT
                       |
                       v
                 FIND ROOT CAUSE
                       |
                       v
                 CORRECTIVE ACTION
                       |
                       v
                  VALIDATION
                       |
                       v
                      RCA
```

---

# 18.2 First 10 DBA Commands

When something is wrong, start here.

### 1. Is PostgreSQL running?

```bash
sudo systemctl status postgresql-19
```

### 2. What does the log say?

```bash
sudo journalctl -u postgresql-19 --since "30 minutes ago"
```

### 3. Is PostgreSQL listening?

```bash
sudo ss -lntp | grep 5432
```

### 4. Can the local server connect?

```bash
sudo -u postgres psql -d postgres
```

### 5. Is the server accepting connections?

```sql
SELECT version();
```

### 6. What sessions exist?

```sql
SELECT
    pid,
    usename,
    datname,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    query_start,
    query
FROM pg_stat_activity
ORDER BY query_start;
```

### 7. Are sessions waiting?

```sql
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;
```

### 8. Are there blocking locks?

```sql
SELECT *
FROM pg_locks;
```

### 9. Is the filesystem full?

```bash
df -h
```

### 10. Is memory/CPU under pressure?

```bash
top
```

PostgreSQL's official monitoring system specifically provides `pg_stat_activity`, `pg_stat_replication`, `pg_stat_wal_receiver`, `pg_stat_io`, `pg_stat_database`, `pg_stat_wal`, and other views for this purpose. ([PostgreSQL][2])

---

# 18.3 Issue 1 — PostgreSQL Will Not Start

### Symptom

```text
systemctl start postgresql-19
```

fails.

First:

```bash
sudo systemctl status postgresql-19 -l
```

Then:

```bash
sudo journalctl -u postgresql-19 -n 100 --no-pager
```

Also:

```bash
sudo -u postgres pg_ctl \
    -D /var/lib/pgsql/19/data \
    status
```

---

## 18.4 Startup Failure Decision Tree

```text
PostgreSQL won't start
        |
        v
systemctl status
        |
        v
Read journal
        |
 +------+------+------+------+
 |      |      |      |
 v      v      v      v
Config Port   Disk   Permission
```

Check configuration:

```bash
sudo -u postgres /usr/pgsql-19/bin/postgres \
    -D /var/lib/pgsql/19/data \
    -C config_file
```

---

# 18.5 Common Startup RCA

### Case A — Port already in use

```text
could not bind IPv4 address
Address already in use
```

Check:

```bash
sudo ss -lntp | grep 5432
```

Find competing process.

---

### Case B — Configuration syntax error

Example:

```text
invalid value for parameter
```

Validate configuration carefully.

Don't randomly modify multiple parameters.

---

### Case C — Permission problem

Check:

```bash
ls -ld /var/lib/pgsql/19/data
```

```bash
ls -l /var/lib/pgsql/19/data/postgresql.conf
```

Expected ownership should match the PostgreSQL service account.

---

# 18.6 Issue 2 — Client Cannot Connect

Test locally:

```bash
psql -h localhost -U postgres -d postgres
```

Then remotely:

```bash
psql -h DB_SERVER_IP -U app_user -d appdb
```

Troubleshooting:

```text
Client
  |
  v
DNS/IP
  |
  v
Network
  |
  v
Firewall
  |
  v
PostgreSQL listener
  |
  v
pg_hba.conf
  |
  v
Authentication
  |
  v
Authorization
```

---

# 18.7 Check Listener

```bash
sudo ss -lntp | grep 5432
```

Check PostgreSQL:

```sql
SHOW listen_addresses;
SHOW port;
```

If:

```text
listen_addresses = localhost
```

remote clients cannot normally connect through the server's external IP.

---

# 18.8 Issue 3 — Authentication Failure

Symptom:

```text
FATAL:
password authentication failed
```

Check role:

```sql
SELECT
    rolname,
    rolcanlogin
FROM pg_roles
WHERE rolname = 'app_user';
```

Check authentication configuration:

```sql
SHOW hba_file;
```

Then inspect the matching HBA rule.

Remember:

> PostgreSQL uses the **first matching `pg_hba.conf` rule**.

---

# 18.9 Issue 4 — Permission Denied

Example:

```text
ERROR:
permission denied for table customer
```

Check:

```sql
SELECT
    grantee,
    privilege_type
FROM information_schema.role_table_grants
WHERE table_name = 'customer';
```

Check role memberships:

```sql
SELECT
    member.rolname AS member,
    role.rolname AS granted_role
FROM pg_auth_members m
JOIN pg_roles role
    ON role.oid = m.roleid
JOIN pg_roles member
    ON member.oid = m.member;
```

---

# 18.10 Issue 5 — Blocking

This is one of the most important DBA incidents.

Example:

```text
Session A
   |
   | UPDATE
   v
Table

Session B
   |
   | UPDATE same resource
   v
WAITING
```

Find waiting sessions:

```sql
SELECT
    pid,
    usename,
    datname,
    state,
    wait_event_type,
    wait_event,
    query_start,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

PostgreSQL 19's monitoring documentation identifies `Lock` waits and specific events such as `relation`, `transactionid`, `tuple`, and `virtualxid`. ([PostgreSQL][2])

---

# 18.11 Find Blocking PID

Use:

```sql
SELECT
    pid,
    pg_blocking_pids(pid) AS blocking_pids,
    usename,
    state,
    query_start,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

Example:

```text
PID 2050
blocking_pids = {1980}
```

Therefore:

```text
1980 → blocker
2050 → blocked
```

---

# 18.12 Blocking RCA

Typical root causes:

```text
Long transaction
Uncommitted UPDATE
DDL holding locks
Application transaction leak
Idle-in-transaction session
Explicit LOCK
```

Check:

```sql
SELECT
    pid,
    usename,
    state,
    xact_start,
    query_start,
    state_change,
    query
FROM pg_stat_activity
ORDER BY xact_start NULLS LAST;
```

---

# 18.13 `idle in transaction`

This state deserves special attention:

```text
idle in transaction
```

means the session has an open transaction but is currently not executing a query.

Potential consequences:

```text
Locks remain held
VACUUM effectiveness affected
Long transaction
Transaction ID age
Blocking
Bloat
```

Find them:

```sql
SELECT
    pid,
    usename,
    datname,
    xact_start,
    state,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

---

# 18.14 Terminating a Problem Session

First attempt:

```sql
SELECT pg_cancel_backend(2050);
```

This requests cancellation of the currently running query.

If necessary:

```sql
SELECT pg_terminate_backend(2050);
```

This terminates the backend/session.

### DBA rule

Do not terminate a backend simply because it has been running "a long time."

First determine:

```text
What is it doing?
Who owns it?
What is it blocking?
Is it a critical transaction?
What happens if terminated?
```

---

# 18.15 Issue 6 — Deadlock

Example:

```text
Transaction A
   |
   +--> Lock Table 1
   |
   +--> waits for Table 2

Transaction B
   |
   +--> Lock Table 2
   |
   +--> waits for Table 1
```

```text
A → B
↑   ↓
+---+
```

PostgreSQL detects the deadlock and aborts one transaction.

---

# 18.16 Deadlock Investigation

Look in PostgreSQL logs.

PostgreSQL 19 enables:

```conf
log_lock_waits = on
```

by default. ([PostgreSQL][1])

Also review:

```text
deadlock_timeout
```

and application transaction ordering.

The solution is usually not:

```text
increase timeout
```

but:

```text
identify conflicting transaction patterns
+
standardize lock acquisition order
+
shorten transactions
```

---

# 18.17 Issue 7 — High CPU

Do not immediately increase CPU.

First identify database workload:

```sql
SELECT
    pid,
    usename,
    datname,
    state,
    query_start,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
```

Then inspect expensive SQL with:

```text
pg_stat_statements
```

PostgreSQL 19 documents `pg_stat_statements` as the standard extension for collecting planning and execution statistics across SQL statements. ([PostgreSQL][3])

---

# 18.18 Enable `pg_stat_statements`

In:

```conf
shared_preload_libraries = 'pg_stat_statements'
```

Then restart PostgreSQL.

In the required database:

```sql
CREATE EXTENSION pg_stat_statements;
```

Check:

```sql
SELECT
    calls,
    total_exec_time,
    mean_exec_time,
    rows,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

---

# 18.19 High CPU RCA

Possible causes:

```text
Bad execution plan
Missing index
Large sequential scans
Excessive concurrency
CPU-heavy functions
Poor join strategy
Data growth
Statistics problems
```

Next:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

**Do not run `EXPLAIN ANALYZE` blindly against destructive SQL in production.**

---

# 18.20 Issue 8 — High I/O

PostgreSQL 19 provides:

```text
pg_stat_io
```

which exposes cluster-wide I/O statistics. ([PostgreSQL][2])

Query:

```sql
SELECT
    backend_type,
    object,
    context,
    reads,
    writes,
    read_time,
    write_time,
    fsyncs,
    fsync_time
FROM pg_stat_io
ORDER BY read_time DESC NULLS LAST;
```

For I/O timing to be populated, the relevant timing configuration must be enabled. ([PostgreSQL][2])

---

# 18.21 OS-Level I/O

RHEL:

```bash
iostat -xz 1
```

Also:

```bash
vmstat 1
```

```bash
sar -n DEV 1
```

Check:

```text
CPU
IOPS
await
utilization
memory
swap
network
```

Database-level and OS-level evidence should be correlated.

---

# 18.22 Issue 9 — Disk Full

Check:

```bash
df -h
```

Then:

```bash
df -ih
```

Find large directories:

```bash
sudo du -xh /var/lib/pgsql/19 | sort -h | tail -20
```

Potential PostgreSQL causes:

```text
WAL accumulation
Replication slot
Large table/index
Temporary files
Logs
Backup files
Archived WAL
```

---

# 18.23 WAL Explosion

Check:

```sql
SELECT
    slot_name,
    active,
    restart_lsn,
    wal_status,
    safe_wal_size
FROM pg_replication_slots;
```

A stale physical replication slot can prevent WAL from being recycled.

Architecture:

```text
Standby offline
      |
      v
Replication slot
      |
      v
WAL retained
      |
      v
pg_wal grows
      |
      v
Filesystem fills
```

---

# 18.24 Issue 10 — Replication Lag

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

Standby:

```sql
SELECT *
FROM pg_stat_wal_receiver;
```

Investigate:

```text
Network
Primary WAL generation
Standby disk I/O
Replay workload
CPU
Locks
Replication configuration
```

---

# 18.25 Issue 11 — WAL Archiving Failure

Check:

```sql
SELECT *
FROM pg_stat_archiver;
```

Look at:

```text
archived_count
failed_count
last_archived_wal
last_failed_wal
last_failed_time
```

Important:

> A healthy PostgreSQL primary does not necessarily mean your WAL archive is healthy.

---

# 18.26 Issue 12 — Autovacuum Problem

Check:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

PostgreSQL 19 adds:

```text
pg_stat_autovacuum_scores
```

for per-table autovacuum-related details. ([PostgreSQL][1])

---

# 18.27 Autovacuum RCA

Possible causes:

```text
Long-running transaction
High update/delete workload
Insufficient autovacuum workers
Poor per-table settings
I/O contention
Locks
Large tables
Wraparound pressure
```

Check:

```sql
SELECT
    pid,
    usename,
    xact_start,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

---

# 18.28 Issue 13 — Table Bloat

Symptoms:

```text
Table grows rapidly
Disk usage increases
Queries become slower
VACUUM struggles
```

Investigate:

```text
Dead tuples
Long transactions
Autovacuum
Index bloat
Update/delete workload
```

Don't automatically run:

```text
VACUUM FULL
```

on a large production table.

It can require substantial locking and additional operational planning.

---

# 18.29 Issue 14 — Slow Query

Standard workflow:

```text
Slow query
   |
   v
pg_stat_statements
   |
   v
Identify SQL
   |
   v
EXPLAIN
   |
   v
EXPLAIN ANALYZE
   |
   v
BUFFERS
   |
   v
Check statistics
   |
   v
Indexes / joins / predicates
   |
   v
Tune
```

Example:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM customer
WHERE customer_id = 1001;
```

---

# 18.30 Bad Plan Investigation

Look for:

```text
Seq Scan
Nested Loop with huge row counts
Incorrect cardinality estimate
Large sort
Hash spilling
High shared/local/temp reads
```

Then compare:

```text
estimated rows
vs
actual rows
```

A large difference can indicate stale or insufficient statistics.

---

# 18.31 `ANALYZE`

Run:

```sql
ANALYZE VERBOSE customer;
```

PostgreSQL uses statistics collected by `ANALYZE` to help the optimizer choose execution plans. ([PostgreSQL][4])

For a specific column:

```sql
ANALYZE customer(customer_id);
```

---

# 18.32 Issue 15 — Connection Exhaustion

Check:

```sql
SHOW max_connections;
```

Current connections:

```sql
SELECT
    count(*)
FROM pg_stat_activity;
```

By database:

```sql
SELECT
    datname,
    count(*)
FROM pg_stat_activity
GROUP BY datname
ORDER BY count(*) DESC;
```

By user:

```sql
SELECT
    usename,
    count(*)
FROM pg_stat_activity
GROUP BY usename
ORDER BY count(*) DESC;
```

---

# 18.33 Connection Pooling

Architecture:

```text
Application
     |
     v
Connection Pool
     |
     v
PostgreSQL
```

Without pooling:

```text
1000 application requests
        |
        v
1000 PostgreSQL sessions
```

This can create significant memory and process overhead.

A connection pooler such as PgBouncer can reduce direct backend connection pressure.

---

# 18.34 Issue 16 — PostgreSQL Crash

Immediate actions:

```bash
sudo systemctl status postgresql-19
```

```bash
sudo journalctl -u postgresql-19 --since "1 hour ago"
```

Look for:

```text
PANIC
FATAL
segmentation fault
OOM
I/O errors
checksum failures
WAL errors
```

Also check kernel:

```bash
sudo journalctl -k --since "1 hour ago"
```

---

# 18.35 OOM Investigation

Check:

```bash
free -h
```

```bash
sudo journalctl -k | grep -i -E 'oom|out of memory|killed process'
```

Potential causes:

```text
Too many connections
Excessive work_mem
Large maintenance_work_mem
Memory pressure outside PostgreSQL
Operating-system configuration
```

Remember:

> `work_mem` is not simply "memory allocated once per server."

It can be consumed by individual operations within queries, so multiplying it naively by `max_connections` can produce misleading capacity calculations—but concurrent workload can still make aggregate memory consumption very large.

---

# 18.36 Issue 17 — Checksum Failure

PostgreSQL 19 exposes checksum failure information through database statistics. ([PostgreSQL][2])

Check:

```sql
SELECT
    datname,
    checksum_failures,
    checksum_last_failure
FROM pg_stat_database;
```

If checksum failures are detected:

```text
STOP
  |
  v
Do not blindly repair
  |
  v
Collect evidence
  |
  v
Check storage
  |
  v
Check PostgreSQL logs
  |
  v
Check backups
  |
  v
Perform recovery analysis
```

This is a potentially serious storage/data-integrity incident.

---

# 18.37 Issue 18 — PostgreSQL Won't Accept New Connections

Possible states:

```text
PostgreSQL stopped
Port inaccessible
max_connections reached
pg_hba rejection
Authentication failure
Filesystem full
Resource exhaustion
Recovery mode
```

Check:

```sql
SELECT
    count(*) AS connections
FROM pg_stat_activity;
```

Then:

```sql
SHOW max_connections;
```

---

# 18.38 Emergency DBA Checklist

When production is degraded:

```text
1. Confirm incident
2. Identify affected application
3. Check PostgreSQL availability
4. Check active sessions
5. Check locks
6. Check CPU
7. Check memory
8. Check disk
9. Check WAL
10. Check replication
11. Check recent deployments
12. Check recent configuration changes
13. Capture evidence
14. Apply smallest safe corrective action
15. Validate
16. Document RCA
```

---

# 18.39 Evidence Collection

Before making major changes, capture:

```bash
date
hostname
uptime
free -h
df -h
df -ih
```

```bash
ps aux | grep postgres
```

```bash
sudo ss -lntp
```

```bash
sudo journalctl -u postgresql-19 --since "1 hour ago"
```

Database:

```sql
SELECT now();

SELECT version();

SELECT * FROM pg_stat_activity;

SELECT * FROM pg_stat_replication;

SELECT * FROM pg_replication_slots;

SELECT * FROM pg_stat_wal;

SELECT * FROM pg_stat_archiver;

SELECT * FROM pg_stat_database;
```

This gives you a defensible incident evidence set.

---

# 18.40 Production RCA Template

## Incident

```text
PostgreSQL production performance degradation.
```

## Start Time

```text
YYYY-MM-DD HH:MM UTC
```

## Impact

```text
Application response time increased.
```

## Symptoms

```text
High CPU
Long-running queries
Lock waits
```

## Investigation

```text
pg_stat_activity
        ↓
pg_stat_statements
        ↓
pg_locks
        ↓
OS metrics
        ↓
PostgreSQL logs
```

## Root Cause

```text
Actual evidence-based root cause.
```

## Corrective Action

```text
Immediate remediation.
```

## Preventive Action

```text
Monitoring
Configuration
Application fix
Index/statistics
Capacity
Process improvement
```

## Validation

```text
CPU normal
Queries normal
Locks cleared
Application healthy
```

---

# 18.41 DBA RCA — Example

### Problem

```text
Application requests are timing out.
```

### Evidence

```text
pg_stat_activity
```

shows:

```text
Several sessions waiting on Lock.
```

### Blocking PID

```text
PID 4500
```

has:

```text
idle in transaction
```

### Root Cause

Application transaction remained open.

### Immediate Action

After application-owner validation:

```sql
SELECT pg_terminate_backend(4500);
```

### Result

Blocked transactions proceed.

### Preventive Action

```text
Application transaction management
idle-in-transaction monitoring
lock monitoring
appropriate timeout controls
```

That is a proper RCA because it identifies the **chain of evidence**, not merely the symptom.

---

# 18.42 PostgreSQL 19 Troubleshooting Toolkit

| Area              | Primary Tool/View           |
| ----------------- | --------------------------- |
| Sessions          | `pg_stat_activity`          |
| Locks             | `pg_locks`                  |
| Lock statistics   | `pg_stat_lock`              |
| Replication       | `pg_stat_replication`       |
| WAL receiver      | `pg_stat_wal_receiver`      |
| WAL               | `pg_stat_wal`               |
| Archiving         | `pg_stat_archiver`          |
| I/O               | `pg_stat_io`                |
| Database          | `pg_stat_database`          |
| Autovacuum        | `pg_stat_autovacuum_scores` |
| Query performance | `pg_stat_statements`        |
| Recovery          | `pg_stat_recovery`          |
| SSL               | `pg_stat_ssl`               |
| OS CPU            | `top`, `vmstat`             |
| OS I/O            | `iostat`                    |
| Filesystem        | `df`, `du`                  |
| PostgreSQL logs   | `journalctl`                |

The PostgreSQL 19 monitoring documentation lists these statistics facilities and their purposes. ([PostgreSQL][2])

---

# 18.43 The DBA Golden Rule

For every production problem:

```text
SYMPTOM
   ↓
EVIDENCE
   ↓
WAIT / RESOURCE
   ↓
ROOT CAUSE
   ↓
CORRECTIVE ACTION
   ↓
VALIDATION
   ↓
PREVENTION
```

Never:

```text
Problem
   ↓
Restart PostgreSQL
   ↓
Problem disappeared
   ↓
RCA = PostgreSQL issue
```

A restart may remove the symptom while destroying valuable diagnostic evidence.

---

# Phase 18 Completed

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
18  Advanced Troubleshooting & RCA  ← COMPLETED
```

## Next — Phase 19

**PostgreSQL 19 DBA Automation & Scripting**

We will build practical **RHEL 10 + PostgreSQL 19 automation**, including:

* Bash DBA scripts
* `psql` automation
* SQL health-check scripts
* backup automation
* WAL/archive monitoring
* replication monitoring
* disk-space alerts
* long-running transaction alerts
* blocking/lock alerts
* autovacuum monitoring
* PostgreSQL service monitoring
* cron/systemd timers
* email notifications
* centralized health reports
* automated RCA evidence collection
* production-ready DBA script framework.

[1]: https://www.postgresql.org/docs/19/release-19.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: E.1. Release 19"
[2]: https://www.postgresql.org/docs/19/monitoring-stats.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 27.2. The Cumulative Statistics System"
[3]: https://www.postgresql.org/docs/19/pgstatstatements.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: F.34. pg_stat_statements — track statistics of SQL planning and execution"
[4]: https://www.postgresql.org/docs/19/sql-analyze.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: ANALYZE"
