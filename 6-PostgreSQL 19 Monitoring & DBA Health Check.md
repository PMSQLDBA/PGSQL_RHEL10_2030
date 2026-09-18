# Phase 6 — PostgreSQL 19 Monitoring & DBA Health Check

I verified this against the current PostgreSQL 19 documentation. PostgreSQL 19 adds several useful monitoring capabilities, including `pg_stat_lock`, `pg_stat_recovery`, `pg_stat_autovacuum_scores`, and additional vacuum/analyze progress information. ([PostgreSQL][1])

For your DBA lab, I recommend building the monitoring layer **without immediately introducing Prometheus/Grafana**. First establish a strong native PostgreSQL health-check framework. Then Grafana/Prometheus can consume the same metrics later.

---

## 1. Monitoring architecture

```text
                 PostgreSQL 19
                       |
       +---------------+----------------+
       |               |                |
       v               v                v
 pg_stat_*         pg_settings       pg_locks
       |               |                |
       +---------------+----------------+
                       |
                       v
             DBA Health Check
                       |
          +------------+------------+
          |                         |
       SQL report               OS report
          |                         |
          +------------+------------+
                       |
                       v
                 HTML Report
```

PostgreSQL's cumulative statistics system provides views for sessions, tables, indexes, WAL, I/O, replication, archiving and progress reporting. ([PostgreSQL][1])

---

# 2. Session monitoring

### Active sessions

```sql
SELECT
    pid,
    usename,
    datname,
    client_addr,
    application_name,
    state,
    backend_start,
    xact_start,
    query_start,
    wait_event_type,
    wait_event,
    LEFT(query, 200) AS query
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
ORDER BY query_start NULLS LAST;
```

This is your PostgreSQL equivalent of a first-level SQL Server session investigation.

---

# 3. Long-running queries

```sql
SELECT
    pid,
    usename,
    datname,
    now() - query_start AS duration,
    state,
    wait_event_type,
    wait_event,
    LEFT(query, 300) AS query
FROM pg_stat_activity
WHERE query_start IS NOT NULL
  AND state <> 'idle'
ORDER BY query_start;
```

For a DBA health check, flag sessions exceeding your chosen threshold rather than automatically killing them.

For example:

```sql
AND now() - query_start > interval '15 minutes'
```

---

# 4. Long-running transactions

This is particularly important because an old transaction can prevent cleanup of dead tuples.

```sql
SELECT
    pid,
    usename,
    datname,
    now() - xact_start AS xact_age,
    state,
    wait_event_type,
    wait_event,
    LEFT(query, 300) AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Investigate especially:

```text
idle in transaction
```

combined with a very old `xact_start`.

---

# 5. Blocking and blocked sessions

Use:

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    now() - blocked.query_start AS blocked_duration,
    LEFT(blocked.query, 200) AS blocked_query,
    LEFT(blocking.query, 200) AS blocking_query
FROM pg_stat_activity blocked
JOIN LATERAL unnest(
    pg_blocking_pids(blocked.pid)
) AS blocker_pid(pid) ON true
JOIN pg_stat_activity blocking
    ON blocking.pid = blocker_pid.pid
WHERE blocked.pid <> blocking.pid
ORDER BY blocked.query_start;
```

This should become one of your **critical DBA alerts**.

---

# 6. PostgreSQL 19 lock monitoring

PostgreSQL 19 introduces `pg_stat_lock`, which provides per-lock-type statistics. ([PostgreSQL][2])

Check:

```sql
SELECT *
FROM pg_stat_lock;
```

For individual lock information:

```sql
SELECT
    locktype,
    database,
    relation::regclass AS relation,
    pid,
    mode,
    granted
FROM pg_locks
ORDER BY granted, pid;
```

Remember:

```text
pg_locks
   =
current lock state

pg_stat_lock
   =
PostgreSQL 19 lock statistics
```

---

# 7. Connection utilization

```sql
SELECT
    count(*) AS current_connections,
    current_setting('max_connections')::int AS max_connections,
    round(
        100.0 * count(*) /
        current_setting('max_connections')::int,
        2
    ) AS utilization_percent
FROM pg_stat_activity;
```

A DBA report should flag unusually high connection utilization.

Do not simply increase `max_connections` whenever it gets high. Connection pooling is often the appropriate architectural solution.

---

# 8. Database-level statistics

```sql
SELECT
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit,
    tup_returned,
    tup_fetched,
    tup_inserted,
    tup_updated,
    tup_deleted,
    temp_files,
    temp_bytes,
    deadlocks
FROM pg_stat_database
WHERE datname IS NOT NULL
ORDER BY datname;
```

---

# 9. Cache hit ratio

A commonly useful diagnostic calculation:

```sql
SELECT
    datname,
    blks_hit,
    blks_read,
    round(
        100.0 * blks_hit /
        NULLIF(blks_hit + blks_read, 0),
        2
    ) AS cache_hit_ratio
FROM pg_stat_database
WHERE datname IS NOT NULL
ORDER BY cache_hit_ratio;
```

Don't treat a single cache-hit percentage as a universal performance threshold. It must be interpreted with workload, query plans and I/O behavior.

---

# 10. WAL monitoring

```sql
SELECT *
FROM pg_stat_wal;
```

Important values include WAL generation and write/sync activity.

Also:

```sql
SELECT
    pg_current_wal_lsn() AS current_wal_lsn;
```

For your backup environment:

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

### Critical condition

```text
failed_count increasing
        +
last_failed_time recent
        =
investigate WAL archiving
```

This is especially important because your PITR architecture depends on successful WAL archiving.

---

# 11. Autovacuum monitoring

PostgreSQL recommends regular vacuuming and provides autovacuum to automate routine maintenance. ([PostgreSQL][3])

Check:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 25;
```

Look for:

```text
High n_dead_tup
+
Old last_autovacuum
+
High transaction activity
```

---

# 12. PostgreSQL 19 autovacuum scoring

This is one of the useful PostgreSQL 19 additions.

```sql
SELECT
    schemaname,
    relname,
    score
FROM pg_stat_autovacuum_scores
ORDER BY score DESC
LIMIT 25;
```

PostgreSQL 19 exposes the autovacuum scoring information through this new view. The score helps show which tables are currently considered more eligible for autovacuum processing. ([PostgreSQL][1])

---

# 13. Vacuum currently running

```sql
SELECT
    pid,
    datname,
    relid::regclass AS table_name,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    heap_blks_vacuumed
FROM pg_stat_progress_vacuum;
```

PostgreSQL exposes one row for each backend currently running `VACUUM`, including autovacuum workers. ([PostgreSQL][4])

---

# 14. Table growth

```sql
SELECT
    schemaname,
    relname,
    pg_size_pretty(
        pg_total_relation_size(relid)
    ) AS total_size,
    pg_size_pretty(
        pg_relation_size(relid)
    ) AS table_size,
    pg_size_pretty(
        pg_indexes_size(relid)
    ) AS index_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 25;
```

This should become your **top 25 largest tables** report.

---

# 15. Database size

```sql
SELECT
    datname,
    pg_size_pretty(
        pg_database_size(datname)
    ) AS database_size
FROM pg_database
WHERE datistemplate = false
ORDER BY pg_database_size(datname) DESC;
```

---

# 16. Index usage

```sql
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

Be careful with unused-index analysis: an index with zero scans isn't automatically safe to drop. Evaluate workload cycles, reporting jobs, constraints, uniqueness and query plans first.

---

# 17. Invalid indexes

```sql
SELECT
    n.nspname AS schema_name,
    c.relname AS index_name,
    i.indisvalid,
    i.indisready
FROM pg_index i
JOIN pg_class c
    ON c.oid = i.indexrelid
JOIN pg_namespace n
    ON n.oid = c.relnamespace
WHERE NOT i.indisvalid
   OR NOT i.indisready;
```

Any unexpected result deserves investigation.

---

# 18. Replication monitoring

For a primary:

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

For a standby:

```sql
SELECT
    pg_is_in_recovery(),
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn(),
    pg_last_xact_replay_timestamp();
```

If you later build your PostgreSQL 19 HA lab, these queries become the foundation for replication monitoring.

---

# 19. I/O monitoring

PostgreSQL 19's statistics system includes `pg_stat_io`, which is particularly useful for understanding backend and background-process I/O behavior. ([PostgreSQL][1])

Run:

```sql
SELECT
    backend_type,
    object,
    context,
    reads,
    writes,
    extends,
    hits,
    evictions
FROM pg_stat_io
ORDER BY reads DESC;
```

This is one area where I would **not** rely solely on old PostgreSQL monitoring scripts. PostgreSQL 19 has additional native statistics that should be incorporated into your DBA toolkit.

---

# 20. `pg_stat_statements`

You configured this in Phase 2.

Verify:

```sql
SELECT
    queryid,
    calls,
    total_exec_time,
    mean_exec_time,
    rows,
    LEFT(query, 300) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

The extension tracks planning/execution statistics for SQL statements and requires `shared_preload_libraries`; PostgreSQL documents this explicitly. ([PostgreSQL][5])

### Top queries by total execution time

```sql
SELECT
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS avg_ms,
    rows,
    LEFT(query, 250) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Top queries by average execution time

```sql
SELECT
    calls,
    round(mean_exec_time::numeric, 2) AS avg_ms,
    round(total_exec_time::numeric, 2) AS total_ms,
    LEFT(query, 250) AS query
FROM pg_stat_statements
WHERE calls > 5
ORDER BY mean_exec_time DESC
LIMIT 20;
```

---

# 21. PostgreSQL configuration drift

This is very useful for a DBA health report:

```sql
SELECT
    name,
    setting,
    unit,
    source,
    sourcefile,
    sourceline
FROM pg_settings
WHERE source NOT IN ('default', 'override')
ORDER BY name;
```

Also inspect configuration errors:

```sql
SELECT
    name,
    setting,
    applied,
    error
FROM pg_file_settings
WHERE error IS NOT NULL
   OR NOT applied;
```

This can identify configuration changes that failed to apply.

---

# 22. OS-level checks

From RHEL:

```bash
echo "===== HOST ====="
hostnamectl

echo
echo "===== UPTIME ====="
uptime

echo
echo "===== MEMORY ====="
free -h

echo
echo "===== FILESYSTEM ====="
df -h

echo
echo "===== INODES ====="
df -ih

echo
echo "===== LOAD ====="
cat /proc/loadavg

echo
echo "===== POSTGRESQL ====="
systemctl status postgresql-19 --no-pager
```

The `df -ih` check is important because a filesystem can run out of **inodes** even when it still has free capacity.

---

# 23. Create the DBA health-check SQL

Create:

```bash
sudo mkdir -p /opt/postgresql19-monitor
```

Then:

```bash
sudo vi /opt/postgresql19-monitor/postgresql_health.sql
```

Use this consolidated report:

```sql
\pset pager off
\timing on

SELECT '=== DATABASE ===' AS section;

SELECT
    current_database() AS database_name,
    current_user,
    inet_server_addr() AS server_ip,
    inet_server_port() AS port,
    version();

SELECT '=== CONNECTIONS ===' AS section;

SELECT
    count(*) AS current_connections,
    current_setting('max_connections')::int AS max_connections,
    round(
        100.0 * count(*) /
        current_setting('max_connections')::int,
        2
    ) AS utilization_percent
FROM pg_stat_activity;

SELECT '=== BLOCKING ===' AS section;

SELECT
    blocked.pid AS blocked_pid,
    blocking.pid AS blocking_pid,
    blocked.usename AS blocked_user,
    blocking.usename AS blocking_user,
    now() - blocked.query_start AS blocked_duration,
    LEFT(blocked.query,200) AS blocked_query
FROM pg_stat_activity blocked
JOIN LATERAL unnest(
    pg_blocking_pids(blocked.pid)
) AS b(pid) ON true
JOIN pg_stat_activity blocking
    ON blocking.pid = b.pid;

SELECT '=== LONG RUNNING TRANSACTIONS ===' AS section;

SELECT
    pid,
    usename,
    datname,
    now() - xact_start AS transaction_age,
    state,
    LEFT(query,200) AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;

SELECT '=== DATABASE SIZE ===' AS section;

SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
WHERE datistemplate = false
ORDER BY pg_database_size(datname) DESC;

SELECT '=== CACHE HIT RATIO ===' AS section;

SELECT
    datname,
    round(
        100.0 * blks_hit /
        NULLIF(blks_hit + blks_read,0),
        2
    ) AS cache_hit_ratio
FROM pg_stat_database
WHERE datname IS NOT NULL;

SELECT '=== WAL ARCHIVER ===' AS section;

SELECT
    archived_count,
    failed_count,
    last_archived_wal,
    last_archived_time,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;

SELECT '=== AUTOVACUUM ===' AS section;

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

SELECT '=== LARGEST TABLES ===' AS section;

SELECT
    schemaname,
    relname,
    pg_size_pretty(
        pg_total_relation_size(relid)
    ) AS total_size
FROM pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;

SELECT '=== INVALID INDEXES ===' AS section;

SELECT
    n.nspname AS schema_name,
    c.relname AS index_name,
    i.indisvalid,
    i.indisready
FROM pg_index i
JOIN pg_class c
    ON c.oid = i.indexrelid
JOIN pg_namespace n
    ON n.oid = c.relnamespace
WHERE NOT i.indisvalid
   OR NOT i.indisready;

SELECT '=== REPLICATION ===' AS section;

SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;

SELECT '=== CONFIGURATION ERRORS ===' AS section;

SELECT
    name,
    setting,
    applied,
    error
FROM pg_file_settings
WHERE error IS NOT NULL
   OR NOT applied;

SELECT '=== POSTGRESQL 19 I/O ===' AS section;

SELECT
    backend_type,
    object,
    context,
    reads,
    writes,
    extends,
    hits,
    evictions
FROM pg_stat_io
ORDER BY reads DESC;
```

---

# 24. Run the complete health check

```bash
sudo -u postgres psql \
-f /opt/postgresql19-monitor/postgresql_health.sql
```

This gives you a single DBA-oriented diagnostic output.

---

# 25. Generate HTML report

For your workflow, I'd take this one step further.

```bash
sudo -u postgres psql \
-X \
-f /opt/postgresql19-monitor/postgresql_health.sql \
--html \
> /opt/postgresql19-monitor/postgresql_health.html
```

Then:

```bash
ls -lh /opt/postgresql19-monitor/postgresql_health.html
```

You now have:

```text
PostgreSQL
     |
     v
Health SQL
     |
     v
HTML report
```

---

# 26. Monitoring thresholds

Don't use arbitrary thresholds as automatic diagnoses. Use them as **investigation triggers**.

| Area                   | Investigation trigger                          |
| ---------------------- | ---------------------------------------------- |
| PostgreSQL service     | Not active                                     |
| Connection utilization | Sustained high utilization                     |
| Blocking               | Any business-impacting blocker                 |
| Long transaction       | Unusually old transaction                      |
| WAL archiving          | Increasing `failed_count`                      |
| Autovacuum             | Persistent dead-tuple accumulation             |
| Disk                   | Low free space                                 |
| Inodes                 | Low inode availability                         |
| Replication            | Increasing replay/flush lag                    |
| Invalid index          | Unexpected invalid index                       |
| Configuration          | `pg_file_settings` errors                      |
| Backup                 | Failed pgBackRest check                        |
| PITR                   | Restore test failure                           |
| I/O                    | Unexpected read/write pressure                 |
| Query performance      | Significant regression in `pg_stat_statements` |

---

## 27. What PostgreSQL 19 gives you that we should exploit

For this lab, don't build the monitoring framework as if you're running PostgreSQL 12/13.

PostgreSQL 19 adds:

* `pg_stat_lock`
* `pg_stat_recovery`
* `pg_stat_autovacuum_scores`
* enhanced `pg_stat_progress_vacuum`
* enhanced `pg_stat_progress_analyze`

These are documented PostgreSQL 19 additions. ([PostgreSQL][2])

So your monitoring stack should specifically include these PostgreSQL 19 views.

---

# Phase 6 outcome

You now have the foundation for:

```text
                    DBA MONITORING
                         |
        +----------------+----------------+
        |                |                |
      Sessions         Locks             I/O
        |                |                |
   Connections       Blocking        pg_stat_io
   Long queries      Deadlocks
        |                |
        +--------+-------+
                 |
          Maintenance
                 |
       +---------+---------+
       |                   |
   Autovacuum           Analyze
       |                   |
       +---------+---------+
                 |
              Storage
                 |
        +--------+--------+
        |                 |
     Database           Tables
       size              size
        |                 |
        +--------+--------+
                 |
             Backup/DR
                 |
       +---------+---------+
       |                   |
   WAL Archive          pgBackRest
       |                   |
       +---------+---------+
                 |
              PITR
```

The next logical step is **Phase 7 — PostgreSQL 19 Performance Tuning**, where we'll build a DBA troubleshooting workflow around **CPU/I/O → waits → blocking → execution plans → `pg_stat_statements` → indexes → statistics → autovacuum → memory → WAL/checkpoints**, including `EXPLAIN (ANALYZE, BUFFERS, WAL)` and practical RCA scenarios.

[1]: https://www.postgresql.org/docs/19/monitoring-stats.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 27.2. The Cumulative Statistics System"
[2]: https://www.postgresql.org/docs/19/release-19.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: E.1. Release 19"
[3]: https://www.postgresql.org/docs/19/sql-vacuum.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: VACUUM"
[4]: https://www.postgresql.org/docs/19/progress-reporting.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 27.4. Progress Reporting"
[5]: https://www.postgresql.org/docs/19/pgstatstatements.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: F.34. pg_stat_statements — track statistics of SQL planning and execution"
