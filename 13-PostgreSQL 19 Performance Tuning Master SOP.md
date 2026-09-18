# Phase 13 — PostgreSQL 19 Performance Tuning Master SOP

> **Version note:** PostgreSQL 19 documentation is currently marked **unsupported/pre-release**, so use this as a PostgreSQL 19 lab/learning SOP and revalidate against the final PostgreSQL 19 release documentation before production deployment. ([PostgreSQL][1])

## 1. Performance Tuning Methodology

Do **not** start by changing `shared_buffers`, `work_mem`, or random indexes.

Use this sequence:

```text
1. Identify the symptom
        ↓
2. Establish baseline
        ↓
3. Identify bottleneck
        ↓
4. Identify expensive SQL
        ↓
5. Inspect execution plan
        ↓
6. Validate statistics/indexes
        ↓
7. Check memory / CPU / I/O
        ↓
8. Check locks / waits / transactions
        ↓
9. Apply ONE controlled change
        ↓
10. Re-test
        ↓
11. Compare before vs after
        ↓
12. Document RCA
```

PostgreSQL's `EXPLAIN` exposes the planner's execution plan, while `EXPLAIN ANALYZE` adds actual execution statistics. `BUFFERS`, `WAL`, `MEMORY`, and `IO` provide additional diagnostic information. ([PostgreSQL][2])

---

# 2. Establish the Performance Baseline

## 2.1 PostgreSQL version

```sql
SELECT version();

SHOW server_version;
SHOW server_version_num;
```

For your RHEL lab:

```bash
/usr/pgsql-19/bin/psql --version
```

---

# 3. Server Configuration Baseline

Run:

```sql
SELECT name,
       setting,
       unit,
       source,
       pending_restart
FROM pg_settings
WHERE name IN
(
    'max_connections',
    'shared_buffers',
    'work_mem',
    'maintenance_work_mem',
    'effective_cache_size',
    'effective_io_concurrency',
    'random_page_cost',
    'seq_page_cost',
    'max_wal_size',
    'min_wal_size',
    'checkpoint_timeout',
    'checkpoint_completion_target',
    'wal_compression',
    'track_io_timing',
    'autovacuum',
    'autovacuum_max_workers',
    'autovacuum_naptime'
)
ORDER BY name;
```

---

# 4. Memory Tuning

PostgreSQL has several different memory consumers.

The important distinction is:

```text
shared_buffers
      ↓
shared PostgreSQL buffer cache

work_mem
      ↓
per-query-operation memory

maintenance_work_mem
      ↓
VACUUM / CREATE INDEX / maintenance operations

effective_cache_size
      ↓
planner estimate only
```

### Critical point

`work_mem` is **not a server-wide memory allocation**.

A complex query can have multiple sort/hash operations, and multiple sessions can execute them concurrently. Therefore:

```text
Total possible memory
≠
work_mem
```

PostgreSQL documentation explicitly warns that total memory consumption can be many times the configured `work_mem`. ([PostgreSQL][3])

---

# 5. `shared_buffers`

Check:

```sql
SHOW shared_buffers;
```

For a dedicated PostgreSQL server with at least 1 GB RAM, PostgreSQL documentation gives approximately **25% of system RAM as a reasonable starting point**, while noting that workloads can benefit from different values and that going above roughly 40% is unlikely to be beneficial because PostgreSQL also relies on the OS cache. ([PostgreSQL][3])

Example for a lab VM:

```text
RAM = 8 GB
Starting point ≈ 2 GB
```

Do **not** blindly configure 2 GB just because the VM has 8 GB.

Consider:

* concurrent connections
* `work_mem`
* autovacuum
* background processes
* OS cache
* monitoring agents
* replication
* backup processes

---

# 6. `work_mem`

Check:

```sql
SHOW work_mem;
SHOW hash_mem_multiplier;
```

Find temporary-file-heavy queries:

```sql
SELECT datname,
       temp_files,
       pg_size_pretty(temp_bytes) AS temp_bytes
FROM pg_stat_database
ORDER BY temp_bytes DESC;
```

If a query performs:

```text
Sort
Hash Join
Hash Aggregate
Materialize
```

and spills to disk, investigate `work_mem`.

Use query-level testing instead of immediately changing the global value:

```sql
BEGIN;

SET LOCAL work_mem = '64MB';

EXPLAIN (ANALYZE, BUFFERS, WAL, SETTINGS)
SELECT ...;

ROLLBACK;
```

This is much safer for troubleshooting.

---

# 7. `effective_cache_size`

Check:

```sql
SHOW effective_cache_size;
```

This parameter **does not allocate memory**.

It tells the planner how much cache is realistically available from:

```text
PostgreSQL shared buffers
+
OS filesystem cache
```

It influences planner decisions such as index scans versus sequential scans. ([PostgreSQL][4])

Therefore:

```text
shared_buffers      = actual PostgreSQL memory
effective_cache_size = planner estimate
```

Do not confuse the two.

---

# 8. Query Performance Investigation

## 8.1 Find currently active queries

```sql
SELECT pid,
       usename,
       datname,
       application_name,
       client_addr,
       state,
       wait_event_type,
       wait_event,
       now() - query_start AS duration,
       query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

---

# 9. Find Long-Running Queries

```sql
SELECT pid,
       usename,
       datname,
       now() - query_start AS duration,
       wait_event_type,
       wait_event,
       state,
       LEFT(query, 1000) AS query
FROM pg_stat_activity
WHERE state <> 'idle'
  AND query_start IS NOT NULL
ORDER BY duration DESC;
```

---

# 10. Find Blocking Sessions

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    blocking.query AS blocking_query,
    now() - blocked.query_start AS blocked_duration
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
ORDER BY blocked.query_start;
```

### RCA pattern

```text
Slow query
   ↓
Actually waiting
   ↓
Lock contention
   ↓
Find blocking PID
   ↓
Find blocking transaction
   ↓
Find application/session
   ↓
Correct transaction/application behavior
```

Do **not** automatically kill the blocked session.

---

# 11. Find Idle-in-Transaction Sessions

These can be particularly problematic because an open transaction can prevent cleanup of old row versions.

```sql
SELECT pid,
       usename,
       datname,
       client_addr,
       now() - xact_start AS xact_age,
       now() - state_change AS state_age,
       query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

Investigate:

```text
idle in transaction
        ↓
old transaction snapshot
        ↓
dead tuples cannot be fully cleaned
        ↓
bloat / vacuum delay
        ↓
performance degradation
```

---

# 12. `EXPLAIN`

Start with:

```sql
EXPLAIN
SELECT ...
;
```

Then use:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
;
```

For deeper analysis:

```sql
EXPLAIN
(
    ANALYZE,
    BUFFERS,
    WAL,
    SETTINGS,
    VERBOSE
)
SELECT ...
;
```

PostgreSQL 19 also exposes `MEMORY` and `IO` options. ([PostgreSQL][2])

### Important warning

`EXPLAIN ANALYZE` actually executes the statement.

Therefore **do not casually run it against production `UPDATE`, `DELETE`, or other destructive statements**.

For example:

```sql
EXPLAIN ANALYZE
DELETE FROM orders
WHERE order_id = 100;
```

will execute the DELETE.

---

# 13. What to Look for in an Execution Plan

### A. Sequential scan

```text
Seq Scan
```

is **not automatically bad**.

If a query needs a large percentage of a table:

```text
Read 80% of table
```

a sequential scan may be cheaper than an index scan.

---

### B. Index Scan

```text
Index Scan
```

Usually useful when the query retrieves a relatively small portion of a table.

---

### C. Bitmap Heap Scan

```text
Bitmap Index Scan
        ↓
Bitmap Heap Scan
```

Common when multiple rows need to be retrieved using an index.

---

### D. Nested Loop

```text
Nested Loop
```

Can be excellent when the outer relation is small and the inner relation has an appropriate index.

Can become disastrous when:

```text
Outer rows = millions
Inner operation = repeated millions of times
```

---

### E. Hash Join

```text
Hash Join
```

Often useful for larger equality joins.

Investigate:

```text
Hash
Batches
Memory usage
Temp I/O
```

Multiple hash batches can indicate memory pressure.

---

### F. Sort

Look for:

```text
Sort Method: quicksort
```

versus:

```text
Sort Method: external merge
Disk: ...
```

External sorting means the operation spilled to temporary storage.

---

# 14. PostgreSQL Statistics

Check database activity:

```sql
SELECT datname,
       numbackends,
       xact_commit,
       xact_rollback,
       blks_read,
       blks_hit,
       temp_files,
       temp_bytes,
       deadlocks
FROM pg_stat_database
ORDER BY datname;
```

---

# 15. Cache Hit Ratio

A simple database-level calculation:

```sql
SELECT datname,
       blks_hit,
       blks_read,
       ROUND(
           100.0 * blks_hit /
           NULLIF(blks_hit + blks_read, 0),
           2
       ) AS cache_hit_ratio_pct
FROM pg_stat_database
WHERE blks_hit + blks_read > 0
ORDER BY cache_hit_ratio_pct;
```

### Important

Do **not** use cache-hit ratio alone to declare a PostgreSQL server healthy or unhealthy.

PostgreSQL's documentation recommends combining PostgreSQL I/O statistics with operating-system-level monitoring because PostgreSQL's statistics do not distinguish every disk-versus-kernel-cache scenario. ([PostgreSQL][5])

---

# 16. PostgreSQL 19 `pg_stat_io`

This is particularly important for your PG19 learning.

```sql
SELECT *
FROM pg_stat_io;
```

Useful fields include I/O activity by:

```text
backend type
object
context
reads
writes
extends
hits
read_time
write_time
```

You can also inspect:

```sql
SELECT backend_type,
       object,
       context,
       reads,
       writes,
       extends,
       hits,
       read_time,
       write_time
FROM pg_stat_io
ORDER BY read_time DESC NULLS LAST;
```

`pg_stat_io` is specifically intended to help analyze PostgreSQL I/O behavior and should be interpreted together with OS-level I/O information. ([PostgreSQL][5])

---

# 17. Enable I/O Timing for Diagnostics

Check:

```sql
SHOW track_io_timing;
```

For temporary diagnostic testing:

```sql
SET track_io_timing = on;
```

Then:

```sql
EXPLAIN
(
    ANALYZE,
    BUFFERS
)
SELECT ...;
```

PostgreSQL 19 exposes I/O timing in several monitoring facilities, including `pg_stat_database`, `pg_stat_io`, `EXPLAIN`, `VACUUM`, and `pg_stat_statements`. ([PostgreSQL][6])

Because timing collection has overhead, test its impact on your workload before enabling it permanently.

---

# 18. `pg_stat_statements`

For your DBA lab, this should be one of the primary performance tools.

Check:

```sql
SELECT *
FROM pg_extension
WHERE extname = 'pg_stat_statements';
```

If already configured:

```sql
SELECT
    queryid,
    calls,
    total_exec_time,
    mean_exec_time,
    rows,
    shared_blks_hit,
    shared_blks_read,
    temp_blks_read,
    temp_blks_written,
    LEFT(query, 500) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Find highest average execution time

```sql
SELECT
    queryid,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    rows,
    LEFT(query, 500) AS query
FROM pg_stat_statements
WHERE calls > 0
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### Find highest total workload

```sql
SELECT
    queryid,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    LEFT(query, 500) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

This distinction matters:

```text
Query A
1 call × 30 seconds
= 30 seconds total

Query B
1,000,000 calls × 10 ms
= 10,000 seconds total
```

The second query can have a much larger impact on the database even though its individual execution is fast.

---

# 19. Table-Level Performance

Find the most frequently scanned tables:

```sql
SELECT
    schemaname,
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
ORDER BY seq_tup_read DESC
LIMIT 20;
```

Find tables with many dead tuples:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    ROUND(
        100.0 * n_dead_tup /
        NULLIF(n_live_tup + n_dead_tup, 0),
        2
    ) AS dead_tuple_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 20;
```

---

# 20. VACUUM and Autovacuum

PostgreSQL relies heavily on routine vacuuming.

`VACUUM`:

* reclaims/reuses dead-row storage
* updates planner statistics in conjunction with `ANALYZE`
* maintains visibility information
* helps prevent transaction-ID wraparound

PostgreSQL recommends regular vacuuming and provides autovacuum to automate routine maintenance. ([PostgreSQL][1])

Check:

```sql
SHOW autovacuum;
SHOW autovacuum_max_workers;
SHOW autovacuum_naptime;
SHOW autovacuum_vacuum_threshold;
SHOW autovacuum_vacuum_scale_factor;
SHOW autovacuum_analyze_threshold;
SHOW autovacuum_analyze_scale_factor;
```

---

# 21. Autovacuum Tuning

For a heavily updated table, database-wide defaults may not be optimal.

Example:

```sql
ALTER TABLE public.orders
SET
(
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_analyze_scale_factor = 0.01
);
```

This means the table can be vacuumed/analyzed after a smaller percentage of rows changes.

### Do not blindly apply these values globally.

Tune based on:

```text
table size
+
update/delete frequency
+
dead tuple growth
+
query workload
+
vacuum duration
+
I/O capacity
```

PostgreSQL 19 also introduces changes to autovacuum table prioritization and exposes `pg_stat_autovacuum_scores`. ([PostgreSQL][1])

---

# 22. `ANALYZE`

Check when tables were last analyzed:

```sql
SELECT
    schemaname,
    relname,
    last_analyze,
    last_autoanalyze,
    analyze_count,
    autoanalyze_count
FROM pg_stat_user_tables
ORDER BY last_autoanalyze NULLS FIRST;
```

Run manually:

```sql
ANALYZE VERBOSE public.orders;
```

`ANALYZE` updates statistics in `pg_statistic`, which the planner uses to estimate row counts and choose execution plans. ([PostgreSQL][7])

---

# 23. Detect Statistics Problems

A classic problem:

```text
Estimated rows = 10
Actual rows    = 5,000,000
```

This can produce a bad execution plan.

Use:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
;
```

Compare:

```text
rows=
```

with:

```text
actual rows=
```

Large discrepancies should trigger investigation of:

* stale statistics
* data skew
* correlated columns
* insufficient statistics target
* missing extended statistics
* parameter-sensitive behavior
* query predicates

---

# 24. Increase Column Statistics Target

Default:

```sql
SHOW default_statistics_target;
```

For a problematic column:

```sql
ALTER TABLE public.orders
ALTER COLUMN customer_id
SET STATISTICS 500;
```

Then:

```sql
ANALYZE public.orders;
```

Do not increase statistics targets on every column without evidence.

---

# 25. Index Investigation

Find indexes:

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename, indexname;
```

Find potentially unused indexes:

```sql
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC,
         pg_relation_size(indexrelid) DESC;
```

### WARNING

`idx_scan = 0` does **not** automatically mean:

> DROP THIS INDEX.

Consider:

* statistics reset
* recent index creation
* reporting/maintenance workloads
* seasonal workloads
* uniqueness enforcement
* foreign-key support
* failover/DR behavior

---

# 26. Index Bloat

Do not immediately rebuild every large index.

First establish:

```text
Is the index actually bloated?
        ↓
Is it affecting performance?
        ↓
Is disk space a problem?
        ↓
Can REINDEX be performed safely?
```

Possible operation:

```sql
REINDEX INDEX index_name;
```

For lower-impact production maintenance where supported by your operational requirements:

```sql
REINDEX INDEX CONCURRENTLY index_name;
```

Always test operational impact first.

---

# 27. Table Bloat Investigation

PostgreSQL's MVCC model means updates/deletes can leave dead row versions until vacuum removes/reclaims them.

Investigate:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Potential RCA:

```text
High UPDATE/DELETE rate
        +
Long-running transactions
        +
Insufficient autovacuum
        ↓
Dead tuples accumulate
        ↓
Table/index growth
        ↓
More I/O
        ↓
Query performance degradation
```

---

# 28. `VACUUM FULL` — Do Not Use as Routine Tuning

Normal:

```sql
VACUUM;
```

or:

```sql
VACUUM (ANALYZE);
```

`VACUUM FULL` behaves differently: it rewrites the table, requires an `ACCESS EXCLUSIVE` lock, and is much more disruptive. PostgreSQL documentation recommends routine standard `VACUUM` rather than relying on `VACUUM FULL`. ([PostgreSQL][1])

Therefore:

```text
Routine maintenance
        ↓
VACUUM / autovacuum

Exceptional space-reclamation operation
        ↓
VACUUM FULL
```

---

# 29. CPU Troubleshooting

Check OS:

```bash
top
```

or:

```bash
htop
```

Check CPU:

```bash
mpstat -P ALL 1 10
```

Check PostgreSQL sessions:

```sql
SELECT pid,
       usename,
       state,
       now() - query_start AS duration,
       query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

Then correlate:

```text
OS CPU
  +
PostgreSQL active sessions
  +
pg_stat_statements
  +
EXPLAIN ANALYZE
```

Do not conclude:

```text
High CPU = PostgreSQL problem
```

The workload itself may simply be CPU-intensive.

---

# 30. I/O Troubleshooting

Linux:

```bash
iostat -xz 1 10
```

Memory:

```bash
free -h
```

Disk:

```bash
df -h
```

Disk latency:

```bash
iostat -xz
```

PostgreSQL:

```sql
SELECT *
FROM pg_stat_io;
```

Correlate:

```text
PostgreSQL I/O
       +
Linux I/O
       +
storage latency
       +
query execution plan
```

---

# 31. WAL Performance

Check:

```sql
SELECT *
FROM pg_stat_wal;
```

Check WAL generation over time by sampling the statistics.

Also:

```sql
SHOW wal_level;
SHOW wal_compression;
SHOW max_wal_size;
SHOW min_wal_size;
SHOW checkpoint_timeout;
SHOW checkpoint_completion_target;
```

Look for:

```text
Excessive WAL generation
Frequent checkpoints
Replication lag
Slow storage
Large bulk operations
```

---

# 32. Checkpoint Troubleshooting

Potential symptoms:

```text
Periodic I/O spikes
Query latency spikes
High write activity
Checkpoint warnings
```

Review:

```sql
SELECT
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint
FROM pg_stat_bgwriter;
```

If requested checkpoints are unusually high:

```text
WAL generated rapidly
        ↓
max_wal_size reached
        ↓
requested checkpoint
        ↓
I/O burst
```

Investigate workload and WAL configuration rather than simply increasing parameters blindly.

---

# 33. Performance RCA Example

### Incident

```text
Application reports:
"Orders query became slow."
```

### Step 1 — Verify

```sql
SELECT now();
```

Check application/query timing.

### Step 2 — Identify query

```sql
SELECT
    queryid,
    calls,
    mean_exec_time,
    total_exec_time,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Step 3 — Execution plan

```sql
EXPLAIN
(
    ANALYZE,
    BUFFERS,
    WAL,
    SETTINGS
)
SELECT ...
;
```

### Step 4 — Compare estimates

```text
Estimated rows: 1,000
Actual rows:    5,000,000
```

### Step 5 — Investigate statistics

```sql
ANALYZE public.orders;
```

### Step 6 — Re-test

```sql
EXPLAIN
(
    ANALYZE,
    BUFFERS,
    WAL,
    SETTINGS
)
SELECT ...
;
```

### Step 7 — If still slow

Investigate:

```text
Index
Join strategy
Sort/hash spilling
I/O
CPU
Blocking
Long transaction
Autovacuum
Data distribution
```

---

# 34. Performance RCA Template

```text
INCIDENT:
Query response time increased from 500 ms to 8 seconds.

IMPACT:
Application order processing latency increased.

DETECTION:
Application monitoring / pg_stat_statements.

ROOT CAUSE:
Planner underestimated result cardinality because statistics
were stale after a large data load.

EVIDENCE:
EXPLAIN ANALYZE showed:
estimated rows = 100
actual rows    = 2,500,000

ACTION:
ANALYZE was executed.

VALIDATION:
Execution plan changed and execution time returned to baseline.

PREVENTION:
Review statistics maintenance after bulk loads and tune
autovacuum/analyze behavior for the affected table.

LESSON:
Do not change indexes or memory parameters before validating
the execution plan and planner statistics.
```

---

# 35. DBA Performance Decision Tree

```text
QUERY SLOW
   |
   +-- Is it currently running?
   |       |
   |       +-- NO → pg_stat_statements
   |       |
   |       +-- YES
   |            |
   |            +-- Waiting?
   |                 |
   |                 +-- Lock → pg_blocking_pids()
   |                 |
   |                 +-- I/O → pg_stat_io + iostat
   |                 |
   |                 +-- CPU → OS + query analysis
   |
   +-- Execution Plan
           |
           +-- Bad estimates
           |       → ANALYZE/statistics
           |
           +-- Seq Scan
           |       → determine whether expected
           |
           +-- Bad Join
           |       → indexes/statistics/query design
           |
           +-- Sort/Hash spill
           |       → work_mem/query design
           |
           +-- Excessive I/O
           |       → indexes/cache/storage/query
           |
           +-- Dead tuples
                   → autovacuum/VACUUM/transactions
```

---

# 36. PostgreSQL DBA Golden Rules

### Rule 1

**Measure before changing.**

### Rule 2

**Never treat a sequential scan as automatically bad.**

### Rule 3

**Never increase `work_mem` globally just because one query spills.**

### Rule 4

**Never drop an index solely because `idx_scan = 0`.**

### Rule 5

**Never use `VACUUM FULL` as routine maintenance.**

### Rule 6

**Always compare estimated rows versus actual rows.**

### Rule 7

**Investigate blocking before killing sessions.**

### Rule 8

**Investigate long transactions when vacuum/bloat problems appear.**

### Rule 9

**Correlate PostgreSQL metrics with Linux CPU, memory, and storage metrics.**

### Rule 10

**Change one important variable at a time and validate the result.**

---

# 37. Practical Lab — Your RHEL 10 / PostgreSQL 19 Server

Run this first:

```sql
SELECT version();

SHOW shared_buffers;
SHOW work_mem;
SHOW maintenance_work_mem;
SHOW effective_cache_size;

SHOW max_connections;

SHOW track_io_timing;
SHOW compute_query_id;

SHOW autovacuum;
SHOW autovacuum_max_workers;
SHOW autovacuum_naptime;

SELECT *
FROM pg_stat_io;

SELECT
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit,
    temp_files,
    temp_bytes,
    deadlocks
FROM pg_stat_database;
```

Then:

```sql
SELECT
    pid,
    usename,
    datname,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS duration,
    LEFT(query, 500) AS query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY duration DESC;
```

And:

```sql
SELECT
    schemaname,
    relname,
    seq_scan,
    idx_scan,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

These commands give you the **initial performance baseline** before making any tuning changes.

---

## Phase 13 — Key Takeaway

The PostgreSQL performance hierarchy should be:

```text
                 PERFORMANCE
                      |
        +-------------+-------------+
        |             |             |
       SQL           DATA          SERVER
        |             |             |
   Query Plan     Statistics       CPU
   Joins          Indexes          RAM
   Predicates     Bloat            I/O
   Sorts          Vacuum           Storage
   Hashes         Analyze          WAL
        |             |             |
        +-------------+-------------+
                      |
                 MEASURE AGAIN
```

PostgreSQL's own documentation emphasizes current planner statistics, `EXPLAIN`, routine vacuuming/autovacuum, and the cumulative statistics/I/O facilities as core components of diagnosing performance. ([PostgreSQL][1])

**Next phase: Phase 14 — PostgreSQL 19 Monitoring & Enterprise DBA Health Check**, including a production-style **single SQL health-check script**, replication/WAL checks, blocking, long transactions, autovacuum, bloat indicators, database growth, configuration drift, and PASS/WARN/CRITICAL logic.

[1]: https://www.postgresql.org/docs/19/routine-vacuuming.html "https://www.postgresql.org/docs/19/routine-vacuuming.html"
[2]: https://www.postgresql.org/docs/19/sql-explain.html "https://www.postgresql.org/docs/19/sql-explain.html"
[3]: https://www.postgresql.org/docs/19/runtime-config-resource.html "https://www.postgresql.org/docs/19/runtime-config-resource.html"
[4]: https://www.postgresql.org/docs/19/runtime-config-query.html "https://www.postgresql.org/docs/19/runtime-config-query.html"
[5]: https://www.postgresql.org/docs/19/monitoring-stats.html "https://www.postgresql.org/docs/19/monitoring-stats.html"
[6]: https://www.postgresql.org/docs/19/runtime-config-statistics.html "https://www.postgresql.org/docs/19/runtime-config-statistics.html"
[7]: https://www.postgresql.org/docs/19/sql-analyze.html "https://www.postgresql.org/docs/19/sql-analyze.html"
