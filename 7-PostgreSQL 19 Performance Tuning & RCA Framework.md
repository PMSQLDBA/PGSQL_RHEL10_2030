# Phase 7 — PostgreSQL 19 Performance Tuning & RCA Framework

This phase turns the monitoring foundation from Phase 6 into a **production-style DBA troubleshooting workflow**.

I verified the core commands against the current PostgreSQL 19 documentation. PostgreSQL 19 remains a pre-release/testing version, so use this lab for learning and validation rather than production workloads. The PostgreSQL 19 documentation specifically recommends current statistics, `EXPLAIN`, autovacuum, and `pg_stat_statements` as key components of performance investigation. ([PostgreSQL][1])

---

# 1. PostgreSQL performance troubleshooting flow

Use this order instead of immediately changing parameters:

```text
                 PERFORMANCE ISSUE
                        |
                        v
                 1. DEFINE SYMPTOM
                        |
                        v
                 2. CPU / MEMORY / IO
                        |
                        v
                 3. CONNECTIONS
                        |
                        v
                 4. BLOCKING / WAITS
                        |
                        v
                 5. LONG TRANSACTIONS
                        |
                        v
                 6. pg_stat_statements
                        |
                        v
                 7. EXPLAIN ANALYZE
                        |
                        v
                 8. TABLE / INDEX / STATS
                        |
                        v
                 9. AUTOVACUUM
                        |
                        v
                 10. MEMORY / WAL
                        |
                        v
                 11. FIX
                        |
                        v
                 12. VALIDATE
                        |
                        v
                 13. DOCUMENT RCA
```

**Important DBA principle:**

> Do not change `work_mem`, `shared_buffers`, indexes, or query SQL merely because something looks high. First establish the workload symptom and evidence.

---

# 2. First question — Is PostgreSQL itself healthy?

```bash
systemctl is-active postgresql-19

systemctl is-enabled postgresql-19

sudo -u postgres psql -c "SELECT version();"

sudo -u postgres psql -c "SELECT now();"
```

Expected:

```text
active
enabled
PostgreSQL 19...
```

---

# 3. Check server load

On RHEL:

```bash
uptime

top

free -h

vmstat 1 5

iostat -xz 1 5
```

If `iostat` is unavailable:

```bash
sudo dnf install -y sysstat
```

Look at:

```text
CPU
 ├─ user
 ├─ system
 └─ iowait

Memory
 ├─ available
 ├─ swap
 └─ cache

Disk
 ├─ utilization
 ├─ latency
 ├─ IOPS
 └─ throughput
```

---

# 4. PostgreSQL session pressure

```sql
SELECT
    state,
    count(*) AS sessions
FROM pg_stat_activity
GROUP BY state
ORDER BY sessions DESC;
```

Then:

```sql
SELECT
    count(*) AS total_connections,
    current_setting('max_connections')::int AS max_connections
FROM pg_stat_activity;
```

### DBA interpretation

```text
High connections
       |
       +-- application connection leak?
       |
       +-- connection pool missing?
       |
       +-- traffic spike?
       |
       +-- long-running sessions?
       |
       +-- max_connections incorrectly sized?
```

Do **not** automatically solve high connections by increasing `max_connections`.

---

# 5. Find active queries

```sql
SELECT
    pid,
    usename,
    datname,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS duration,
    LEFT(query,500) AS query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

This is your first PostgreSQL equivalent of:

```text
sp_WhoIsActive
```

in a SQL Server environment.

---

# 6. Understand PostgreSQL waits

Look at:

```sql
SELECT
    wait_event_type,
    wait_event,
    count(*) AS sessions
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
GROUP BY wait_event_type, wait_event
ORDER BY sessions DESC;
```

Typical categories include:

```text
Lock
LWLock
IO
Client
IPC
Timeout
Activity
BufferPin
```

A query being slow does **not** automatically mean the execution plan is bad.

It may be waiting for:

```text
Lock
Disk I/O
Another backend
Client
Buffer/resource
```

Therefore:

```text
Slow query
   ↓
Check wait
   ↓
Then inspect execution plan
```

---

# 7. Blocking investigation

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    blocked.wait_event_type,
    blocked.wait_event,
    now() - blocked.query_start AS blocked_for,
    LEFT(blocked.query,300) AS blocked_query,
    LEFT(blocking.query,300) AS blocking_query
FROM pg_stat_activity blocked
JOIN LATERAL unnest(
    pg_blocking_pids(blocked.pid)
) AS b(pid) ON true
JOIN pg_stat_activity blocking
    ON blocking.pid = b.pid
ORDER BY blocked.query_start;
```

### RCA example

```text
Application query slow
        ↓
Wait event = Lock
        ↓
Blocking PID identified
        ↓
Blocking transaction is "idle in transaction"
        ↓
Application failed to COMMIT/ROLLBACK
        ↓
ROOT CAUSE = application transaction management
```

That is substantially different from:

```text
ROOT CAUSE = PostgreSQL needs more memory
```

---

# 8. Long-running transaction investigation

```sql
SELECT
    pid,
    usename,
    datname,
    state,
    now() - xact_start AS transaction_age,
    now() - query_start AS query_age,
    wait_event_type,
    wait_event,
    LEFT(query,300) AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Pay special attention to:

```text
idle in transaction
```

because an old transaction can interfere with cleanup of obsolete row versions.

---

# 9. `pg_stat_statements` — your primary SQL performance source

Check whether it exists:

```sql
SELECT
    extname,
    extversion
FROM pg_extension
WHERE extname = 'pg_stat_statements';
```

If not:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

Your Phase 2 configuration must also have:

```text
shared_preload_libraries = 'pg_stat_statements'
```

and PostgreSQL must be restarted after changing that startup parameter.

PostgreSQL 19 exposes execution metrics including `calls`, `total_exec_time`, `mean_exec_time`, minimum/maximum execution time and block-level statistics through `pg_stat_statements`. ([PostgreSQL][2])

---

# 10. Top SQL by total execution time

```sql
SELECT
    calls,
    round(total_exec_time::numeric,2) AS total_exec_ms,
    round(mean_exec_time::numeric,2) AS avg_exec_ms,
    rows,
    shared_blks_hit,
    shared_blks_read,
    temp_blks_read,
    temp_blks_written,
    LEFT(query,500) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

This answers:

> Which SQL statements are consuming the most cumulative database execution time?

---

# 11. Top SQL by average execution time

```sql
SELECT
    calls,
    round(mean_exec_time::numeric,2) AS avg_exec_ms,
    round(max_exec_time::numeric,2) AS max_exec_ms,
    rows,
    LEFT(query,500) AS query
FROM pg_stat_statements
WHERE calls >= 5
ORDER BY mean_exec_time DESC
LIMIT 20;
```

---

# 12. Top SQL by physical reads

```sql
SELECT
    calls,
    shared_blks_read,
    shared_blks_hit,
    round(
        100.0 * shared_blks_hit /
        NULLIF(shared_blks_hit + shared_blks_read,0),
        2
    ) AS cache_hit_percent,
    LEFT(query,500) AS query
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 20;
```

This helps identify queries generating substantial shared-buffer reads.

---

# 13. Top SQL generating temporary I/O

```sql
SELECT
    calls,
    temp_blks_read,
    temp_blks_written,
    round(mean_exec_time::numeric,2) AS avg_exec_ms,
    LEFT(query,500) AS query
FROM pg_stat_statements
WHERE temp_blks_read > 0
   OR temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 20;
```

Potential causes include:

```text
Sort
Hash
Materialization
Large aggregation
Insufficient work_mem
Poor execution plan
Large intermediate result
```

Do **not** automatically increase `work_mem`.

---

# 14. Execution plan — first level

Use:

```sql
EXPLAIN
SELECT ...
;
```

Example:

```sql
EXPLAIN
SELECT *
FROM customer_orders
WHERE customer_id = 1001;
```

This shows the optimizer's selected execution plan without actually executing the query.

---

# 15. Execution plan with actual runtime

For a controlled test:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM customer_orders
WHERE customer_id = 1001;
```

PostgreSQL documentation explicitly notes that `EXPLAIN ANALYZE` actually executes the statement and introduces profiling overhead. Therefore, use it carefully on production DML. ([PostgreSQL][1])

For destructive statements:

```sql
BEGIN;

EXPLAIN (ANALYZE, BUFFERS)
DELETE FROM customer_orders
WHERE customer_id = 1001;

ROLLBACK;
```

Even then, be careful with side effects, triggers, sequences and external functions.

---

# 16. Add WAL information

For deeper analysis:

```sql
EXPLAIN (
    ANALYZE,
    BUFFERS,
    WAL
)
SELECT *
FROM customer_orders
WHERE customer_id = 1001;
```

For write workloads:

```sql
EXPLAIN (
    ANALYZE,
    BUFFERS,
    WAL
)
UPDATE customer_orders
SET status = 'COMPLETE'
WHERE customer_id = 1001;
```

This allows you to correlate:

```text
Execution time
+
Buffer activity
+
WAL generation
```

---

# 17. What to look for in an execution plan

Example:

```text
Seq Scan
Index Scan
Index Only Scan
Bitmap Heap Scan
Bitmap Index Scan
Nested Loop
Hash Join
Merge Join
Sort
Aggregate
HashAggregate
```

Do **not** use this rule:

```text
Seq Scan = BAD
Index Scan = GOOD
```

A sequential scan can be exactly correct when a large percentage of the table is required.

---

# 18. Actual rows vs estimated rows

One of the most important DBA checks:

```text
estimated rows = 10
actual rows    = 1,500,000
```

That is a major cardinality-estimation problem.

Possible causes:

```text
Outdated statistics
Data distribution changed
Insufficient statistics target
Correlated columns
Skewed data
Missing extended statistics
Parameter-sensitive workload
```

First investigate statistics before blindly adding an index.

---

# 19. Update statistics

For a specific table:

```sql
ANALYZE customer_orders;
```

Or:

```sql
VACUUM (ANALYZE) customer_orders;
```

PostgreSQL documentation states that current planner statistics are important for good plans, and `ANALYZE` updates those statistics. ([PostgreSQL][1])

---

# 20. Check statistics freshness

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_mod_since_analyze,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_mod_since_analyze DESC
LIMIT 25;
```

A large:

```text
n_mod_since_analyze
```

combined with an old:

```text
last_autoanalyze
```

is an investigation candidate.

---

# 21. Autovacuum performance

PostgreSQL's autovacuum is not optional housekeeping in a normal production architecture; it performs both `VACUUM` and `ANALYZE` automatically and protects against transaction-ID wraparound. ([PostgreSQL][3])

Check:

```sql
SHOW autovacuum;
SHOW autovacuum_max_workers;
SHOW autovacuum_naptime;
SHOW autovacuum_vacuum_scale_factor;
SHOW autovacuum_analyze_scale_factor;
SHOW autovacuum_work_mem;
```

PostgreSQL 19 also introduces autovacuum worker slots and scoring controls. ([PostgreSQL][4])

---

# 22. PostgreSQL 19 autovacuum scoring

```sql
SELECT
    schemaname,
    relname,
    score
FROM pg_stat_autovacuum_scores
ORDER BY score DESC
LIMIT 25;
```

This is particularly useful in PostgreSQL 19 because autovacuum prioritization now uses scoring components for:

```text
Transaction age
Dead tuples
Inserted tuples
Analyze requirements
```

The PostgreSQL 19 documentation describes this new prioritization mechanism. ([PostgreSQL][3])

---

# 23. Vacuum — standard vs FULL

### Standard VACUUM

```sql
VACUUM customer_orders;
```

### VACUUM ANALYZE

```sql
VACUUM (ANALYZE) customer_orders;
```

### VACUUM FULL

```sql
VACUUM FULL customer_orders;
```

Do **not** make `VACUUM FULL` your normal maintenance command.

PostgreSQL documents that ordinary `VACUUM` can operate alongside normal reads/writes, while `VACUUM FULL` rewrites the table and requires an `ACCESS EXCLUSIVE` lock. ([PostgreSQL][5])

Think:

```text
Routine maintenance
        ↓
VACUUM / ANALYZE
        ↓
Special space-reclamation situation
        ↓
VACUUM FULL
```

---

# 24. Memory tuning

Check current settings:

```sql
SELECT
    name,
    setting,
    unit,
    context
FROM pg_settings
WHERE name IN
(
    'shared_buffers',
    'work_mem',
    'maintenance_work_mem',
    'autovacuum_work_mem',
    'effective_cache_size',
    'temp_buffers'
)
ORDER BY name;
```

### Important distinction

```text
shared_buffers
      ↓
PostgreSQL shared buffer cache

work_mem
      ↓
per-operation memory
```

`work_mem` is **not** simply "memory allocated to PostgreSQL."

Multiple sort/hash operations and multiple sessions can consume it concurrently.

PostgreSQL's resource documentation also warns that autovacuum workers can independently consume memory through `autovacuum_work_mem`. ([PostgreSQL][6])

---

# 25. `effective_cache_size`

Check:

```sql
SHOW effective_cache_size;
```

This does **not** allocate memory.

It is a planner estimate of cache available to PostgreSQL queries and influences planner cost calculations. PostgreSQL explicitly documents that it does not reserve kernel cache or allocate shared memory. ([PostgreSQL][7])

---

# 26. Checkpoint/WAL investigation

```sql
SELECT *
FROM pg_stat_wal;
```

And:

```sql
SELECT
    name,
    setting,
    unit
FROM pg_settings
WHERE name IN
(
    'checkpoint_timeout',
    'checkpoint_completion_target',
    'max_wal_size',
    'min_wal_size',
    'wal_compression'
);
```

If checkpoint/WAL pressure is suspected, correlate:

```text
WAL generation
+
checkpoint activity
+
disk I/O
+
query latency
```

Don't change `max_wal_size` in isolation.

---

# 27. A practical RCA example

### Problem

```text
Application reports:
"Orders query suddenly became slow."
```

### Step 1

Check active sessions.

```sql
SELECT
    pid,
    wait_event_type,
    wait_event,
    now() - query_start AS duration,
    LEFT(query,300)
FROM pg_stat_activity
WHERE state <> 'idle';
```

Suppose:

```text
wait_event_type = Lock
```

### Step 2

Find blocker.

```sql
SELECT pg_blocking_pids(<PID>);
```

### Step 3

Inspect blocker.

```sql
SELECT
    pid,
    state,
    xact_start,
    query_start,
    LEFT(query,500)
FROM pg_stat_activity
WHERE pid = <BLOCKING_PID>;
```

Suppose:

```text
state = idle in transaction
xact_age = 2 hours
```

### RCA

```text
Application opened transaction
        ↓
UPDATE executed
        ↓
COMMIT/ROLLBACK not issued
        ↓
Transaction remained open
        ↓
Locks remained
        ↓
Other sessions blocked
        ↓
Application observed slow queries
```

### Corrective action

Fix the transaction-management problem rather than increasing:

```text
shared_buffers
work_mem
max_connections
```

That is the difference between **RCA-driven tuning** and parameter guessing.

---

# 28. Performance tuning decision matrix

| Symptom                      | First investigation                    |
| ---------------------------- | -------------------------------------- |
| Query slow                   | `pg_stat_activity` + wait event        |
| Query waiting                | Lock/wait investigation                |
| CPU high                     | `pg_stat_statements`                   |
| Disk I/O high                | `pg_stat_io` + execution plans         |
| Many temp files              | `pg_stat_statements` + plan            |
| Poor plan                    | `EXPLAIN (ANALYZE, BUFFERS)`           |
| Estimated/actual rows differ | Statistics                             |
| Dead tuples high             | Autovacuum                             |
| Table continually grows      | Vacuum + workload                      |
| Connections high             | Pooling/application behavior           |
| WAL growing                  | WAL generation + replication/archiving |
| Replication lag              | `pg_stat_replication`                  |
| Checkpoint pressure          | WAL/checkpoint metrics                 |
| Blocking                     | `pg_blocking_pids()`                   |
| Long transaction             | `pg_stat_activity`                     |
| Invalid index                | `pg_index`                             |
| Query regression             | `pg_stat_statements` comparison        |

---

# 29. DBA golden rule

Use this sequence:

```text
OBSERVE
   ↓
MEASURE
   ↓
IDENTIFY
   ↓
REPRODUCE
   ↓
EXPLAIN
   ↓
CHANGE
   ↓
VALIDATE
   ↓
DOCUMENT
```

Never:

```text
CPU high
   ↓
increase memory
```

or:

```text
query slow
   ↓
create index
```

without evidence.

---

# 30. Phase 7 deliverable

Your PostgreSQL 19 DBA toolkit now has:

```text
Phase 1  Installation
Phase 2  DBA Configuration
Phase 3  Security / Remote Connectivity
Phase 4  Backup / WAL
Phase 5  PITR / DR
Phase 6  Monitoring / Health Check
Phase 7  Performance / RCA
```

The next phase should be **Phase 8 — PostgreSQL 19 Security Hardening**, covering:

* RHEL SELinux
* `pg_hba.conf` design
* SCRAM authentication
* TLS/SSL
* certificate management
* roles and memberships
* `SUPERUSER` separation
* `CREATEDB` / `CREATEROLE`
* `PUBLIC` privileges
* database/schema/table privilege model
* default privileges
* audit logging
* `pgaudit`
* password policy
* network/firewalld controls
* security validation queries
* DBA security checklist.

[1]: https://www.postgresql.org/docs/19/sql-explain.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: EXPLAIN"
[2]: https://www.postgresql.org/docs/19/pgstatstatements.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: F.34. pg_stat_statements — track statistics of SQL planning and execution"
[3]: https://www.postgresql.org/docs/19/routine-vacuuming.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 24.1. Routine Vacuuming"
[4]: https://www.postgresql.org/docs/19/runtime-config-vacuum.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 19.10. Vacuuming"
[5]: https://www.postgresql.org/docs/19/sql-vacuum.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: VACUUM"
[6]: https://www.postgresql.org/docs/19/runtime-config-resource.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 19.4. Resource Consumption"
[7]: https://www.postgresql.org/docs/19/runtime-config-query.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 19.7. Query Planning"
