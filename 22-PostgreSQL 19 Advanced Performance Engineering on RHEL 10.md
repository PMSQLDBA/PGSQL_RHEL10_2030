# Phase 22 — PostgreSQL 19 Advanced Performance Engineering on RHEL 10

This phase covers **advanced performance engineering only**. I’ll avoid repeating installation, basic architecture, security, replication, backup, HA/DR, and basic monitoring already covered.

---

## 22.1 Performance Engineering Model

PostgreSQL performance should be analyzed across four layers:

```text
Application
    ↓
SQL / Query
    ↓
PostgreSQL Engine
    ↓
RHEL 10 / Storage
```

The DBA's job is to identify **where the bottleneck actually exists** before changing configuration.

---

# 22.2 First Rule — Measure Before Tuning

Never start with:

```text
"Increase shared_buffers"
"Add indexes"
"Increase work_mem"
"Increase CPU"
```

Start with:

```text
Symptom
   ↓
Measurement
   ↓
Bottleneck
   ↓
Root cause
   ↓
Change
   ↓
Measure again
```

---

# 22.3 PostgreSQL Performance Domains

```text
CPU
│
├── Query execution
├── Sorts
├── Aggregations
├── Joins
└── Parallel workers

Memory
│
├── shared_buffers
├── work_mem
├── maintenance_work_mem
└── OS cache

Storage
│
├── Random I/O
├── Sequential I/O
├── WAL I/O
└── Checkpoints

Concurrency
│
├── Locks
├── Connections
├── Transactions
└── Contention

SQL
│
├── Execution plans
├── Cardinality estimates
├── Indexes
└── Statistics
```

---

# 22.4 `EXPLAIN` — Core Performance Tool

Start with:

```sql
EXPLAIN
SELECT *
FROM orders
WHERE customer_id = 1001;
```

For actual execution:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE customer_id = 1001;
```

This allows comparison between:

```text
Estimated rows
vs
Actual rows
```

and:

```text
Estimated cost
vs
Actual execution
```

---

# 22.5 Critical Warning: `EXPLAIN ANALYZE`

`EXPLAIN ANALYZE` actually executes the statement.

Therefore:

```sql
EXPLAIN ANALYZE
UPDATE ...
```

can modify production data.

For DML testing, use an appropriate transaction strategy where safe:

```sql
BEGIN;

EXPLAIN (ANALYZE, BUFFERS)
UPDATE ...;

ROLLBACK;
```

But don't assume rollback makes every side effect harmless; triggers, external effects, sequence behavior, and other mechanisms require separate consideration.

---

# 22.6 Reading an Execution Plan

Example:

```text
Seq Scan
  ↓
Filter
  ↓
Rows Removed by Filter
```

Possible interpretation:

```text
Large table
+
Low selectivity
=
Sequential scan may be appropriate
```

Do not automatically classify every sequential scan as a problem.

---

# 22.7 Sequential Scan vs Index Scan

### Sequential Scan

```text
Table
 ↓
Read many/all pages
 ↓
Evaluate rows
```

### Index Scan

```text
Index
 ↓
Locate matching rows
 ↓
Visit table pages
```

For highly selective predicates, an index may be beneficial.

For queries returning a large percentage of a table, sequential scanning may be cheaper.

---

# 22.8 Bitmap Heap Scan

PostgreSQL can use:

```text
Bitmap Index Scan
       ↓
Bitmap Heap Scan
```

This is useful when many matching table rows exist but an index can still narrow the pages that need to be visited.

---

# 22.9 Cardinality Estimation

One of the most important performance concepts:

```text
Optimizer estimate
        vs
Actual rows
```

Example:

```text
Estimated: 100 rows
Actual:    5,000,000 rows
```

This can produce a poor execution plan.

Investigate:

```text
Statistics
Data distribution
Correlated columns
Skew
Outdated statistics
```

---

# 22.10 `ANALYZE`

Run:

```sql
ANALYZE orders;
```

For an entire database:

```sql
ANALYZE;
```

Autovacuum normally performs automatic analyze operations, but heavily changing or unusual data distributions may require DBA attention.

---

# 22.11 Extended Statistics

For correlated columns, PostgreSQL supports extended statistics.

Example:

```sql
CREATE STATISTICS orders_customer_status_stats
ON customer_id, status
FROM orders;
```

Then:

```sql
ANALYZE orders;
```

This can improve estimates where individual-column statistics don't adequately describe relationships between columns.

---

# 22.12 Statistics Targets

Check:

```sql
SELECT
    attrelid::regclass,
    attname,
    attstattarget
FROM pg_attribute
WHERE attstattarget >= 0;
```

A column can have a customized statistics target:

```sql
ALTER TABLE orders
ALTER COLUMN customer_id
SET STATISTICS 500;
```

Then:

```sql
ANALYZE orders;
```

Higher statistics targets can improve estimates but increase statistics collection/storage overhead.

Don't globally increase every column without evidence.

---

# 22.13 `pg_stat_statements`

This is one of the most valuable tools for workload-level analysis.

Check whether installed:

```sql
SELECT *
FROM pg_extension
WHERE extname = 'pg_stat_statements';
```

Create if required:

```sql
CREATE EXTENSION pg_stat_statements;
```

It requires the appropriate PostgreSQL configuration for tracking.

---

# 22.14 Find Expensive Queries

Example:

```sql
SELECT
    queryid,
    calls,
    total_exec_time,
    mean_exec_time,
    rows,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

This answers:

> Which queries consume the most total execution time?

---

# 22.15 Find Slow Queries

```sql
SELECT
    queryid,
    calls,
    mean_exec_time,
    total_exec_time,
    query
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

Important distinction:

```text
Total execution time
≠
Average execution time
```

A query taking 50 ms and running 10 million times can matter more than a 30-second query executed once.

---

# 22.16 Find I/O-Heavy Queries

Where available in the statistics view:

```sql
SELECT
    queryid,
    calls,
    shared_blks_read,
    shared_blks_hit,
    temp_blks_read,
    temp_blks_written,
    query
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 20;
```

This helps identify queries generating significant physical reads.

---

# 22.17 Cache Hit Ratio

A basic database-level view:

```sql
SELECT
    datname,
    blks_read,
    blks_hit,
    round(
        100.0 * blks_hit /
        NULLIF(blks_hit + blks_read, 0),
        2
    ) AS cache_hit_ratio
FROM pg_stat_database
ORDER BY cache_hit_ratio;
```

Do **not** use a target such as "99%" as a universal performance requirement.

A high hit ratio can coexist with poor performance, and a lower ratio can be perfectly reasonable for certain workloads.

---

# 22.18 `shared_buffers`

Check:

```sql
SHOW shared_buffers;
```

This is PostgreSQL's shared buffer cache.

Conceptually:

```text
Application
    ↓
PostgreSQL
    ↓
shared_buffers
    ↓
OS page cache
    ↓
Storage
```

The correct value depends on workload, RAM, OS behavior, concurrent services, and storage characteristics.

Avoid copying a percentage blindly from another server.

---

# 22.19 `work_mem`

Check:

```sql
SHOW work_mem;
```

`work_mem` is **not simply "memory per database."**

It can be consumed by individual query operations such as:

```text
Sort
Hash
Hash join
Hash aggregation
```

and a single query can have multiple memory-consuming operations.

Therefore:

```text
work_mem × operations × concurrent sessions
```

can become substantial.

---

# 22.20 Temporary File Monitoring

Large sorts/hashes may spill to temporary files.

Check:

```sql
SELECT
    datname,
    temp_files,
    pg_size_pretty(temp_bytes) AS temp_bytes
FROM pg_stat_database
ORDER BY temp_bytes DESC;
```

If temporary I/O is significant, investigate:

```text
work_mem
Query plan
Sort strategy
Join strategy
Indexes
Concurrency
```

Don't simply increase `work_mem` first.

---

# 22.21 Detect Sort Spills

Use:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
ORDER BY ...
```

Look for:

```text
Sort Method: external merge
Disk: ...
```

That indicates disk-based sorting.

---

# 22.22 Hash Operations

For hash joins/aggregations, inspect:

```text
Buckets
Batches
Memory usage
Disk usage
```

If the operation uses many batches, investigate whether memory pressure or inaccurate cardinality estimates are contributing.

---

# 22.23 Parallel Query

Check:

```sql
SHOW max_worker_processes;
SHOW max_parallel_workers;
SHOW max_parallel_workers_per_gather;
```

A parallel plan may look like:

```text
Gather
  |
  +-- Parallel Seq Scan
  |
  +-- Parallel Seq Scan
```

Parallelism can improve large analytical workloads but may increase CPU pressure.

---

# 22.24 Don't Maximize Parallelism

More workers do not automatically mean:

```text
More workers
   =
Faster query
```

Possible result:

```text
Too many workers
      ↓
CPU saturation
      ↓
More contention
      ↓
Other queries slow down
```

Tune based on workload.

---

# 22.25 JIT Compilation

PostgreSQL supports LLVM-based JIT compilation.

Check:

```sql
SHOW jit;
```

JIT can help some CPU-intensive analytical workloads.

For short OLTP queries, JIT compilation overhead may outweigh its benefits.

Measure before changing it.

---

# 22.26 Index Engineering

An index should answer a specific access pattern.

Example:

```sql
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);
```

Validate with:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE customer_id = 1001;
```

---

# 22.27 Composite Indexes

Example:

```sql
CREATE INDEX idx_orders_customer_status
ON orders(customer_id, status);
```

This can support queries using:

```text
customer_id
```

and combinations involving:

```text
customer_id + status
```

But index column order matters.

---

# 22.28 Partial Index

Example:

```sql
CREATE INDEX idx_orders_active_customer
ON orders(customer_id)
WHERE status = 'ACTIVE';
```

Useful when:

```text
ACTIVE rows
```

represent a relatively small portion of the table and queries repeatedly target that predicate.

---

# 22.29 Expression Index

Example:

```sql
CREATE INDEX idx_users_lower_email
ON users(lower(email));
```

This can support:

```sql
SELECT *
FROM users
WHERE lower(email) = 'user@example.com';
```

without requiring a full-table evaluation for every lookup.

---

# 22.30 Covering Index / `INCLUDE`

Example:

```sql
CREATE INDEX idx_orders_customer
ON orders(customer_id)
INCLUDE (order_date, amount);
```

This can enable index-only scans in suitable circumstances.

However, whether an index-only scan is actually used depends on the query plan and visibility-map state.

---

# 22.31 Index-Only Scan

Conceptually:

```text
Query
 ↓
Index
 ↓
Required columns available
 ↓
Avoid many heap visits
```

This can significantly reduce I/O for appropriate workloads.

---

# 22.32 Index Bloat

Indexes can grow due to updates/deletes and workload patterns.

Investigate using PostgreSQL catalog/statistics information and appropriate extensions/tools.

Don't blindly run:

```sql
REINDEX DATABASE;
```

on production.

Large indexes can require substantial I/O and locking considerations.

---

# 22.33 `REINDEX CONCURRENTLY`

For suitable production scenarios:

```sql
REINDEX INDEX CONCURRENTLY index_name;
```

This reduces blocking compared with a conventional rebuild, but it still consumes resources and has operational considerations.

Test before production deployment.

---

# 22.34 Table Bloat

Dead tuples are generated by PostgreSQL's MVCC behavior.

Monitor:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    last_vacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

High dead tuples can contribute to:

```text
Storage growth
More pages scanned
Poor cache utilization
Longer queries
```

---

# 22.35 VACUUM vs VACUUM FULL

### VACUUM

```text
Reclaims reusable space
Updates visibility information
Supports MVCC maintenance
```

### VACUUM FULL

```text
Rewrites table
Can return space to filesystem
Requires substantially stronger locking
Can require significant temporary disk space
```

Do not use `VACUUM FULL` as routine maintenance.

---

# 22.36 Transaction ID / Wraparound Risk

PostgreSQL's MVCC architecture requires transaction ID maintenance.

Monitor database age:

```sql
SELECT
    datname,
    age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

Very high transaction age requires immediate DBA investigation.

Autovacuum/freeze behavior is therefore not merely about reclaiming space.

---

# 22.37 Checkpoint Tuning

Inspect:

```sql
SHOW checkpoint_timeout;
SHOW checkpoint_completion_target;
SHOW max_wal_size;
SHOW min_wal_size;
```

Checkpoint activity can affect:

```text
Write I/O
Latency
WAL volume
Recovery behavior
```

Avoid tuning checkpoints solely from generic recommendations.

---

# 22.38 Checkpoint Monitoring

Database statistics can expose checkpoint activity.

```sql
SELECT
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint
FROM pg_stat_bgwriter;
```

A high rate of requested checkpoints can be a signal worth investigating.

---

# 22.39 WAL Performance

Monitor:

```sql
SELECT
    wal_records,
    wal_fpi,
    wal_bytes,
    wal_buffers_full
FROM pg_stat_wal;
```

High WAL generation can result from:

```text
Heavy DML
Bulk loads
Index maintenance
Full-page writes
Large transactions
```

The correct response depends on the workload.

---

# 22.40 Storage Latency

On RHEL:

```bash
iostat -xz 5
```

Look at:

```text
await
%util
r/s
w/s
rkB/s
wkB/s
```

Interpretation must consider the storage architecture and device type.

High `%util` does not have exactly the same meaning across every modern storage stack.

---

# 22.41 CPU Analysis

RHEL:

```bash
top
```

or:

```bash
vmstat 5
```

Look for:

```text
us
sy
wa
id
```

A database with high CPU usage requires query/workload investigation before simply adding CPU.

---

# 22.42 Memory Analysis

Check:

```bash
free -h
```

and:

```bash
vmstat 5
```

Investigate:

```text
Available memory
Swap activity
Page reclaim
Memory pressure
PostgreSQL memory configuration
Other processes
```

Avoid treating Linux's `free` column alone as evidence of memory shortage.

---

# 22.43 NUMA Considerations

Large RHEL servers may use NUMA.

Check:

```bash
numactl --hardware
```

and:

```bash
lscpu
```

For large PostgreSQL systems, NUMA topology can affect:

```text
Memory locality
CPU scheduling
Latency
Throughput
```

This becomes increasingly relevant on large multi-socket systems.

---

# 22.44 Connection Scaling

More PostgreSQL connections are not automatically better.

Architecture:

```text
1000 Clients
     |
     v
Connection Pool
     |
     v
Controlled PostgreSQL connections
```

Connection pooling can reduce:

```text
Backend process overhead
Memory consumption
Connection churn
```

The exact pool size must be workload-tested.

---

# 22.45 Performance Troubleshooting Decision Tree

```text
Query slow?
    |
    +-- One query?
    |      |
    |      +-- EXPLAIN ANALYZE
    |      +-- BUFFERS
    |      +-- Statistics
    |
    +-- Many queries?
           |
           +-- pg_stat_statements
           +-- CPU
           +-- I/O
           +-- Memory
           +-- Locks
           +-- Connections
```

Then:

```text
Evidence
   ↓
Root Cause
   ↓
Change
   ↓
Benchmark
   ↓
Production validation
```

---

# 22.46 DBA Performance RCA Example

### Symptom

```text
Orders query increased from 300 ms
to 12 seconds.
```

### Investigation

```text
EXPLAIN ANALYZE
      ↓
Actual rows far exceed estimate
      ↓
Statistics inaccurate
      ↓
Poor join strategy
```

### Corrective action

```text
ANALYZE
+
extended statistics if justified
```

### Validation

```text
EXPLAIN ANALYZE again
      ↓
Execution time
+
buffer reads
+
actual/estimated rows
```

The important point is:

> **Tune the root cause, not the symptom.**

---

# 22.47 Production Performance Checklist

```text
[ ] Capture baseline
[ ] Identify workload
[ ] Identify slow/expensive SQL
[ ] Review EXPLAIN plans
[ ] Compare estimated vs actual rows
[ ] Review statistics
[ ] Review indexes
[ ] Review temporary I/O
[ ] Review locks
[ ] Review CPU
[ ] Review memory
[ ] Review storage latency
[ ] Review WAL/checkpoints
[ ] Review autovacuum
[ ] Implement one controlled change
[ ] Measure again
[ ] Document RCA
```

---

# Phase 22 Completed

```text
21  Enterprise Security & Compliance
22  Advanced Performance Engineering   ← COMPLETED
```
