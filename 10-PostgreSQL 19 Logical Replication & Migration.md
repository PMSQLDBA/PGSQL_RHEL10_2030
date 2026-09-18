# Phase 10 — PostgreSQL 19 Logical Replication & Migration SOP

**Lab focus:** Publisher → Subscriber, selective replication, sequence replication, monitoring, conflict handling, and **PostgreSQL 17/18 → 19 migration**.

> **Current-version note:** PostgreSQL 19 documentation is currently marked **unsupported/pre-release** by the official PostgreSQL site, and the release page still shows an unreleased date. Treat this Phase 10 lab as a PostgreSQL 19 testing/lab exercise until PostgreSQL 19 is officially released. ([PostgreSQL][1])

---

## 10.1 Logical Replication Architecture

```text
                  PostgreSQL 19
                  PUBLISHER
               +----------------+
               | Source DB      |
               | salesdb        |
               |                |
               | Publication    |
               | logical slot   |
               +-------+--------+
                       |
                 WAL / Logical
                  Decoding
                       |
                       v
               +-------+--------+
               | SUBSCRIBER     |
               | PostgreSQL 19  |
               |                |
               | Subscription   |
               | Apply Workers  |
               +----------------+
                       |
                       v
                Application /
                Reporting /
                Migration
```

PostgreSQL logical replication uses a **publisher** and **subscriber** model. A server can simultaneously act as both publisher and subscriber. ([PostgreSQL][2])

### Physical vs Logical Replication

| Feature              | Physical Replication                         | Logical Replication                           |
| -------------------- | -------------------------------------------- | --------------------------------------------- |
| Replication level    | WAL/block level                              | Logical row changes                           |
| Scope                | Entire cluster                               | Selected objects/data                         |
| Cross-major-version  | Generally no                                 | **Yes**                                       |
| Selected tables      | No                                           | Yes                                           |
| Row filtering        | No                                           | Yes                                           |
| Column filtering     | No                                           | Yes                                           |
| Subscriber writable  | Read-only standby                            | Yes                                           |
| HA/DR                | Excellent use case                           | Not primary purpose                           |
| Migration            | Possible but not normal cross-version method | Excellent                                     |
| DDL replication      | N/A as logical DDL                           | Not automatic                                 |
| Sequence replication | Physical WAL                                 | **PG19 adds sequence replication**            |
| Replication slot     | Physical                                     | Logical                                       |
| Typical use          | HA/DR                                        | Migration, selective replication, integration |

PostgreSQL explicitly documents logical replication as a method for upgrading between major versions. ([PostgreSQL][3])

---

# 10.2 Phase 10A — Prepare Publisher

Assume:

```text
Publisher : 192.168.0.145
Subscriber: 192.168.0.146
Database  : appdb
User      : logical_repl
```

Replace the IP addresses with your actual lab addresses.

### Check current version

```bash
psql --version
```

```sql
SELECT version();
```

### Check WAL configuration

```sql
SHOW wal_level;
SHOW max_replication_slots;
SHOW max_wal_senders;
SHOW max_logical_replication_workers;
SHOW max_worker_processes;
```

For a basic logical replication setup:

```text
wal_level = logical
```

is the traditional configuration. PostgreSQL 19 introduces behavior allowing logical replication to be automatically enabled when needed with `wal_level=replica`; `effective_wal_level` reports the effective level. For this lab, explicitly setting `wal_level=logical` keeps the configuration unambiguous. ([PostgreSQL][1])

---

# 10.3 Create Replication User

On the **publisher**:

```sql
CREATE ROLE logical_repl
WITH LOGIN
REPLICATION
PASSWORD 'CHANGE_THIS_STRONG_PASSWORD';
```

For production, use a strong secret and preferably SSL/TLS-protected connections.

Verify:

```sql
SELECT
    rolname,
    rolreplication,
    rolcanlogin
FROM pg_roles
WHERE rolname = 'logical_repl';
```

---

# 10.4 Configure pg_hba.conf

On publisher:

```bash
sudo vi /var/lib/pgsql/19/data/pg_hba.conf
```

Add a **source-restricted** rule:

```text
host    appdb    logical_repl    192.168.0.146/32    scram-sha-256
```

Do **not** use:

```text
0.0.0.0/0
```

in a production configuration.

Reload:

```bash
sudo systemctl reload postgresql-19
```

Validate from subscriber:

```bash
psql -h 192.168.0.145 -U logical_repl -d appdb
```

---

# 10.5 Create Test Tables

On publisher:

```sql
CREATE TABLE public.customers
(
    customer_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_name TEXT NOT NULL,
    email TEXT,
    state_code CHAR(2),
    created_at TIMESTAMPTZ DEFAULT clock_timestamp()
);
```

Insert test data:

```sql
INSERT INTO public.customers
(customer_name, email, state_code)
VALUES
('John Smith', 'john@example.com', 'TX'),
('Mary Jones', 'mary@example.com', 'NJ'),
('David Brown', 'david@example.com', 'CA'),
('Robert Wilson', 'robert@example.com', 'NY');
```

---

# 10.6 Create Publication

The simplest publication:

```sql
CREATE PUBLICATION pub_customers
FOR TABLE public.customers;
```

Check:

```sql
SELECT *
FROM pg_publication;
```

Check publication tables:

```sql
SELECT *
FROM pg_publication_tables
WHERE pubname = 'pub_customers';
```

---

# 10.7 Row Filtering

PostgreSQL supports row-filter expressions in publications. Rows for which the expression evaluates to false or NULL are not published. ([PostgreSQL][4])

Example:

```sql
CREATE PUBLICATION pub_texas_customers
FOR TABLE public.customers
WHERE (state_code = 'TX');
```

Only Texas rows are published.

This is useful for:

* regional databases
* data distribution
* reporting environments
* tenant-specific replication
* reducing replication volume

### Important

A row filter is **not a security boundary**.

Do not treat logical replication filtering as a replacement for:

* database authorization
* RLS
* network controls
* encryption
* application security

---

# 10.8 Column Filtering

You can publish selected columns.

Example:

```sql
CREATE PUBLICATION pub_customer_basic
FOR TABLE public.customers
(
    customer_id,
    customer_name,
    state_code
);
```

The subscriber therefore receives only the published columns.

This is useful when the subscriber doesn't require sensitive or unnecessary attributes.

---

# 10.9 PostgreSQL 19 — Excluding Tables

One of the new PG19 logical replication capabilities is the ability to exclude tables while using broader publication definitions. ([PostgreSQL][1])

Conceptually:

```sql
CREATE PUBLICATION pub_all_except
FOR ALL TABLES
EXCEPT TABLE public.audit_log;
```

This is particularly useful for large databases where you want:

```text
ALL TABLES
      |
      +---- audit_log       excluded
      +---- temp_data       excluded
      +---- application     replicated
      +---- customers       replicated
      +---- orders          replicated
```

**Before deploying this syntax in your lab, validate the exact PG19 build's `CREATE PUBLICATION` grammar with:**

```sql
\h CREATE PUBLICATION
```

because PG19 is still pre-release/testing.

---

# 10.10 Subscriber Configuration

Create the target database:

```sql
CREATE DATABASE appdb;
```

The target schema should exist before replication.

For example:

```sql
CREATE TABLE public.customers
(
    customer_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_name TEXT NOT NULL,
    email TEXT,
    state_code CHAR(2),
    created_at TIMESTAMPTZ DEFAULT clock_timestamp()
);
```

### Important DBA rule

Logical replication primarily replicates **data changes**, not your complete database definition.

Plan separately for:

* schemas
* tables
* indexes
* constraints
* functions
* procedures
* views
* materialized views
* extensions
* roles
* permissions
* database-level configuration

DDL changes are not automatically replicated by normal logical replication. PostgreSQL specifically warns about this in the `pg_createsubscriber` documentation. ([PostgreSQL][5])

---

# 10.11 Create Subscription

On subscriber:

```sql
CREATE SUBSCRIPTION sub_customers
CONNECTION 'host=192.168.0.145 port=5432 dbname=appdb user=logical_repl password=CHANGE_THIS_STRONG_PASSWORD'
PUBLICATION pub_customers;
```

By default, the initial table contents are synchronized and then ongoing changes are applied. ([PostgreSQL][6])

---

# 10.12 Validate Subscription

On subscriber:

```sql
SELECT
    subname,
    subenabled,
    subslotname,
    subpublications
FROM pg_subscription;
```

Check workers:

```sql
SELECT *
FROM pg_stat_subscription;
```

Check data:

```sql
SELECT *
FROM public.customers
ORDER BY customer_id;
```

You should see the publisher's rows.

---

# 10.13 Test DML Replication

### Publisher

```sql
INSERT INTO public.customers
(customer_name, email, state_code)
VALUES
('Alice Green', 'alice@example.com', 'TX');
```

Subscriber:

```sql
SELECT *
FROM public.customers
WHERE customer_name = 'Alice Green';
```

---

### UPDATE

Publisher:

```sql
UPDATE public.customers
SET state_code = 'FL'
WHERE customer_name = 'Alice Green';
```

Subscriber:

```sql
SELECT *
FROM public.customers
WHERE customer_name = 'Alice Green';
```

---

### DELETE

Publisher:

```sql
DELETE FROM public.customers
WHERE customer_name = 'Alice Green';
```

Subscriber:

```sql
SELECT *
FROM public.customers
WHERE customer_name = 'Alice Green';
```

Expected:

```text
0 rows
```

---

# 10.14 Replication Slot Monitoring

On publisher:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    confirmed_flush_lsn,
    wal_status,
    invalidation_reason,
    failover,
    synced
FROM pg_replication_slots;
```

Logical replication depends on replication slots.

A forgotten logical slot can retain WAL and potentially consume significant disk space. PostgreSQL documents invalidation reasons including:

```text
wal_removed
rows_removed
wal_level_insufficient
idle_timeout
```

and PG19 adds visibility into `failover`, `synced`, and slot synchronization state. ([PostgreSQL][7])

### DBA alert

Monitor:

```text
slot inactive
        ↓
consumer stopped
        ↓
WAL retained
        ↓
pg_wal grows
        ↓
filesystem fills
        ↓
DATABASE OUTAGE
```

This is one of the most important logical replication operational risks.

---

# 10.15 Subscription Monitoring

On subscriber:

```sql
SELECT
    subname,
    pid,
    relid,
    received_lsn,
    last_msg_send_time,
    last_msg_receipt_time,
    latest_end_lsn,
    latest_end_time
FROM pg_stat_subscription;
```

PG19 also provides:

```sql
SELECT *
FROM pg_stat_subscription_stats;
```

This view includes apply errors, table synchronization errors, sequence synchronization errors, and several logical replication conflict counters. ([PostgreSQL][8])

---

# 10.16 Conflict Investigation

Check:

```sql
SELECT *
FROM pg_stat_subscription_stats;
```

Important conflict categories include:

```text
confl_insert_exists
confl_update_origin_differs
confl_delete_missing
confl_multiple_unique_conflicts
```

A common cause is modifying replicated tables directly on the subscriber.

Example:

```text
Publisher
   |
   | INSERT id=100
   v
Subscriber
   |
   +-- local INSERT id=100
           |
           v
       UNIQUE conflict
```

### Recommended design

For a normal migration subscriber:

```text
Publisher = READ/WRITE
Subscriber = REPLICATION TARGET
```

Avoid application writes to replicated tables until your architecture explicitly requires bidirectional or multi-primary behavior.

---

# 10.17 PostgreSQL 19 Sequence Replication

This is a major PG19 enhancement.

PostgreSQL 19 adds native logical replication of sequence values. A publication can include:

```sql
CREATE PUBLICATION pub_sequences
FOR ALL SEQUENCES;
```

Sequence synchronization requires the **publisher to be PostgreSQL 19 or later**. ([PostgreSQL][9])

You can also combine table and sequence replication.

For example:

```sql
CREATE PUBLICATION pub_app
FOR TABLE public.customers,
          public.orders
FOR ALL SEQUENCES;
```

**Validate the exact grammar for your PG19 build:**

```sql
\h CREATE PUBLICATION
```

---

# 10.18 Refresh Sequences

PG19 provides:

```sql
ALTER SUBSCRIPTION sub_app
REFRESH SEQUENCES;
```

This re-synchronizes sequence values.

PostgreSQL documents `REFRESH SEQUENCES` specifically for sequence synchronization. ([PostgreSQL][9])

Inspect sequence state:

```sql
SELECT *
FROM pg_get_sequence_data('public.customers_customer_id_seq');
```

PG19 also provides `pg_get_sequence_data()` for examining sequence synchronization state. ([PostgreSQL][9])

---

# 10.19 Initial Copy Options

When creating a subscription:

```sql
CREATE SUBSCRIPTION sub_app
CONNECTION '...'
PUBLICATION pub_app
WITH (
    copy_data = true
);
```

`copy_data=true` means:

```text
Existing publisher data
        ↓
Initial table synchronization
        ↓
Subscriber
        ↓
Ongoing changes
```

For a pre-seeded target:

```sql
WITH (
    copy_data = false
);
```

This is useful when you have independently loaded the target with a consistent copy.

---

# 10.20 Logical Replication for PostgreSQL 17/18 → 19 Migration

This is the major DBA use case.

```text
PostgreSQL 17/18
      |
      | Logical Replication
      v
PostgreSQL 19
      |
      | Initial sync
      v
Catch-up
      |
      v
Application freeze
      |
      v
Final synchronization
      |
      v
Application points to PG19
```

PostgreSQL's official upgrade documentation states that logical replication can be used to migrate between different major versions and can reduce upgrade downtime to only several seconds after synchronization has completed. ([PostgreSQL][3])

---

# 10.21 Migration Runbook

## Step 1 — Source Assessment

On PostgreSQL 17/18:

```sql
SELECT version();
```

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname))
FROM pg_database
WHERE datistemplate = false
ORDER BY pg_database_size(datname) DESC;
```

Check extensions:

```sql
SELECT
    extname,
    extversion
FROM pg_extension
ORDER BY extname;
```

Check tables:

```sql
SELECT
    schemaname,
    count(*) AS table_count
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
GROUP BY schemaname
ORDER BY schemaname;
```

---

# 10.22 Step 2 — Schema Migration

Use:

```bash
pg_dump \
  -h SOURCE_IP \
  -U postgres \
  -d appdb \
  --schema-only \
  > appdb_schema.sql
```

Copy to PG19:

```bash
scp appdb_schema.sql user@192.168.0.146:/tmp/
```

Restore:

```bash
psql \
  -h 192.168.0.146 \
  -U postgres \
  -d appdb \
  -f /tmp/appdb_schema.sql
```

Validate:

```sql
SELECT
    schemaname,
    count(*)
FROM pg_tables
WHERE schemaname NOT IN
(
    'pg_catalog',
    'information_schema'
)
GROUP BY schemaname;
```

---

# 10.23 Step 3 — Create Publication on PG17/18

Example:

```sql
CREATE PUBLICATION migration_pub
FOR ALL TABLES;
```

For a controlled migration, I recommend explicitly listing required tables rather than blindly publishing everything.

---

# 10.24 Step 4 — Create Subscription on PG19

On PG19:

```sql
CREATE SUBSCRIPTION migration_sub
CONNECTION
'host=SOURCE_IP port=5432 dbname=appdb user=logical_repl password=PASSWORD'
PUBLICATION migration_pub
WITH (
    copy_data = true
);
```

Initial synchronization starts.

Monitor:

```sql
SELECT *
FROM pg_stat_subscription;
```

---

# 10.25 Step 5 — Monitor Data Synchronization

Source:

```sql
SELECT
    slot_name,
    active,
    restart_lsn,
    confirmed_flush_lsn
FROM pg_replication_slots;
```

Target:

```sql
SELECT *
FROM pg_stat_subscription;
```

Check table synchronization:

```sql
SELECT
    srrelid::regclass,
    srsubstate,
    srsublsn
FROM pg_subscription_rel;
```

Typical states include synchronization stages before reaching the ready state.

---

# 10.26 Step 6 — Validate Row Counts

Source:

```sql
SELECT count(*)
FROM public.customers;
```

Target:

```sql
SELECT count(*)
FROM public.customers;
```

For large migrations, build an automated validation framework containing:

```text
Database
Schema
Table
Source Count
Target Count
Difference
Status
```

Example:

```text
customers      10,000,000   10,000,000   0       PASS
orders         50,000,000   50,000,000   0       PASS
order_lines   150,000,000  149,999,990  10      FAIL
```

Do not perform the cutover until discrepancies are understood.

---

# 10.27 Step 7 — Application Freeze

At the migration window:

```text
STOP APPLICATION WRITES
        ↓
Allow replication to catch up
        ↓
Confirm subscriber is current
        ↓
Run final validation
        ↓
Switch application connection
        ↓
START APPLICATION
```

This is the key downtime-reduction mechanism.

---

# 10.28 Step 8 — Sequence Validation

For PG19:

```sql
SELECT *
FROM pg_get_sequence_data('public.customers_customer_id_seq');
```

Also inspect application-generated IDs after cutover.

Sequence problems can cause:

```text
duplicate key
unique constraint violation
unexpected ID generation
application failure
```

PG19's sequence replication is specifically designed to address synchronization of sequence values between publisher and subscriber. ([PostgreSQL][9])

---

# 10.29 Step 9 — Application Cutover

Change:

```text
OLD:
PostgreSQL 17/18
192.168.0.145
```

to:

```text
NEW:
PostgreSQL 19
192.168.0.146
```

Better production design:

```text
Application
     |
     v
DNS / VIP / Load Balancer
     |
     v
PostgreSQL endpoint
```

Then the database IP does not need to be hardcoded into every application.

---

# 10.30 Step 10 — Post-Cutover Validation

Run:

```sql
SELECT version();
```

```sql
SELECT current_database();
```

```sql
SELECT now();
```

Check:

```text
Application connectivity
Application transactions
Reads
Writes
Sequences
Indexes
Constraints
Extensions
Scheduled jobs
Monitoring
Backups
Replication
Security
Performance
```

---

# 10.31 Important Migration Limitation — DDL

Logical replication does **not** mean:

```text
CREATE TABLE
ALTER TABLE
CREATE INDEX
CREATE FUNCTION
ALTER FUNCTION
```

will automatically appear on the subscriber.

Therefore:

```text
DDL
 |
 +----> Migration process
 |
 +----> Manual/schema deployment
```

must be handled separately.

This is especially important during long-running migrations where developers continue changing schemas.

### Recommended process

```text
DDL change request
       ↓
Apply to source
       ↓
Apply same DDL to target
       ↓
Validate schema
       ↓
Continue replication
```

---

# 10.32 PostgreSQL 19 `pg_createsubscriber`

PG19 introduces another important migration-oriented tool:

```bash
pg_createsubscriber
```

It converts a **physical standby into a logical replica**.

The official documentation describes it as particularly useful for large databases because it avoids repeating the complete initial logical data copy. Instead, the physical standby already contains the data and logical replication synchronizes changes from an appropriate starting point. ([PostgreSQL][5])

Architecture:

```text
PG19 Primary
     |
     | Physical Streaming
     v
PG19 Standby
     |
     | pg_createsubscriber
     v
Logical Replica
```

This can be extremely valuable for very large databases.

However, there are important prerequisites:

* source and target must use the same major version as `pg_createsubscriber`
* target must be a physical standby
* target must have the required logical replication worker/origin capacity
* source must have sufficient replication slots/WAL senders
* DDL should not change during conversion
* failure during conversion can leave the target needing to be rebuilt

These are explicitly documented by PostgreSQL. ([PostgreSQL][5])

---

# 10.33 Logical Replication + HA

A production architecture can become:

```text
                 PostgreSQL 19
                    PRIMARY
                       |
             Physical Streaming
                       |
              +--------+--------+
              |                 |
          Standby 1         Standby 2
              |
       Logical Failover
          Slot Sync
              |
              v
        Subscriber
```

PG19 supports logical **failover slots** that can be synchronized to physical standbys.

Relevant settings include:

```text
sync_replication_slots = on
```

on the standby.

The purpose is to allow logical subscribers to resume from the new primary after failover. ([PostgreSQL][2])

Check:

```sql
SELECT
    slot_name,
    slot_type,
    failover,
    synced,
    invalidation_reason,
    slotsync_skip_reason
FROM pg_replication_slots;
```

---

# 10.34 RCA — Logical Replication Failure

### Scenario

Application data is not appearing on subscriber.

### Investigation

#### 1. Subscription status

```sql
SELECT *
FROM pg_stat_subscription;
```

#### 2. Subscription errors

```sql
SELECT *
FROM pg_stat_subscription_stats;
```

#### 3. Replication slot

Publisher:

```sql
SELECT
    slot_name,
    active,
    restart_lsn,
    confirmed_flush_lsn,
    wal_status,
    invalidation_reason
FROM pg_replication_slots;
```

#### 4. PostgreSQL logs

```bash
sudo journalctl -u postgresql-19 --since "30 minutes ago"
```

#### 5. Network

```bash
nc -zv 192.168.0.145 5432
```

#### 6. Authentication

```bash
psql \
  -h 192.168.0.145 \
  -U logical_repl \
  -d appdb
```

---

## RCA Matrix

| Symptom            | Likely Cause                  | Validation                   |
| ------------------ | ----------------------------- | ---------------------------- |
| Subscription down  | Network/authentication        | `pg_stat_subscription`       |
| Apply error        | Data conflict                 | `pg_stat_subscription_stats` |
| WAL growing        | Inactive slot                 | `pg_replication_slots`       |
| Table missing      | Publication/schema issue      | `pg_publication_tables`      |
| Sequence incorrect | Sequence sync issue           | `pg_get_sequence_data()`     |
| DDL mismatch       | Schema not synchronized       | Schema comparison            |
| Subscriber lag     | Apply bottleneck              | Subscription statistics      |
| Slot invalidated   | Required WAL/rows removed     | `invalidation_reason`        |
| Migration fails    | Source/target incompatibility | Version/schema checks        |

---

# 10.35 Phase 10 Production Checklist

### Publisher

```text
[ ] PostgreSQL version validated
[ ] wal_level validated
[ ] max_replication_slots sized
[ ] max_wal_senders sized
[ ] Replication user created
[ ] pg_hba source restricted
[ ] TLS enabled
[ ] Publication created
[ ] Publication tables validated
```

### Subscriber

```text
[ ] PostgreSQL version validated
[ ] Schema deployed
[ ] Required extensions installed
[ ] Subscription created
[ ] pg_stat_subscription monitored
[ ] pg_stat_subscription_stats monitored
[ ] Initial synchronization completed
[ ] Row counts validated
[ ] Sequence values validated
```

### Migration

```text
[ ] Source assessment
[ ] Schema migration
[ ] Publication
[ ] Subscription
[ ] Initial copy
[ ] Continuous replication
[ ] Data validation
[ ] Application freeze
[ ] Final synchronization
[ ] Sequence validation
[ ] Application cutover
[ ] Functional validation
[ ] Performance validation
[ ] Backup validation
[ ] Old source retained for rollback window
```

---

# 10.36 DBA Interview Questions

### Q1. Physical vs logical replication?

**Physical replication** replicates the database cluster at the WAL/storage level and is primarily used for HA/DR.

**Logical replication** publishes logical changes from selected objects and is especially useful for selective replication, integrations, and major-version migration.

---

### Q2. Can PostgreSQL logical replication support major-version migration?

**Yes.** PostgreSQL documents logical replication as a method for migrating between different major versions. ([PostgreSQL][3])

---

### Q3. Does logical replication replicate DDL?

**No, not automatically.**

DDL must be managed separately.

---

### Q4. What happens if a logical replication slot is abandoned?

WAL can be retained because the slot protects required WAL from removal. Eventually this can consume filesystem capacity.

---

### Q5. What is new in PostgreSQL 19 logical replication?

Important additions include:

* sequence replication
* `FOR ALL SEQUENCES`
* sequence synchronization
* `REFRESH SEQUENCES`
* publication table exclusion
* logical failover-slot improvements
* `pg_createsubscriber`
* improvements around logical replication configuration. ([PostgreSQL][1])

---

### Q6. Is logical replication an automatic HA solution?

**No.**

Logical replication is not equivalent to PostgreSQL physical streaming HA. Automatic failover requires additional architecture for:

```text
Failure detection
Fencing
Promotion
Endpoint management
Application reconnection
Replication continuity
```

---

### Q7. What is the biggest operational risk with logical slots?

**Unbounded WAL retention caused by a stalled or abandoned consumer.**

---

# Phase 10 Summary

```text
                 LOGICAL REPLICATION
                         |
        +----------------+----------------+
        |                                 |
   SELECTIVE DATA                    MIGRATION
        |                                 |
        v                                 v
 Publication                       PG17/18 → PG19
        |                                 |
        v                                 v
 Logical Slot                     Initial Copy
        |                                 |
        v                                 v
 Subscriber                       Catch-up
        |                                 |
        +---------------+-----------------+
                        |
                        v
                   VALIDATION
                        |
                        v
                     CUTOVER
```

**Most important DBA takeaway:** use **physical streaming replication for HA/DR**, and use **logical replication for selective data movement, integrations, and cross-major-version migration**. PostgreSQL 19 substantially expands logical replication capabilities, particularly around **sequence synchronization and failover-aware logical slots**, but because the official PostgreSQL 19 documentation is still marked pre-release/unsupported, validate all PG19-specific syntax against the exact build in your lab before using it in production. ([PostgreSQL][1])

**Official references:** [PostgreSQL 19 Logical Replication documentation](https://www.postgresql.org/docs/19/logical-replication.html?utm_source=chatgpt.com) · [PostgreSQL 19 Upgrade documentation](https://www.postgresql.org/docs/19/upgrading.html?utm_source=chatgpt.com) · [PostgreSQL 19 Release Notes](https://www.postgresql.org/docs/19/release-19.html?utm_source=chatgpt.com)

**Next phase:** **Phase 11 — PostgreSQL 19 Advanced HA: Physical + Logical Replication, Failover Slots, Cascading Replication, Switchover/Failover, `pg_createsubscriber`, RPO/RTO testing, and production HA architecture.**

[1]: https://www.postgresql.org/docs/19/release-19.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: E.1. Release 19"
[2]: https://www.postgresql.org/docs/19/runtime-config-replication.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 19.6. Replication"
[3]: https://www.postgresql.org/docs/19/upgrading.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 18.6. Upgrading a PostgreSQL Cluster"
[4]: https://www.postgresql.org/docs/18/sql-createpublication.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: CREATE PUBLICATION"
[5]: https://www.postgresql.org/docs/19/app-pgcreatesubscriber.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: pg_createsubscriber"
[6]: https://www.postgresql.org/docs/19/logical-replication-quick-setup.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 29.15. Quick Setup"
[7]: https://www.postgresql.org/docs/19/view-pg-replication-slots.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 53.22. pg_replication_slots"
[8]: https://www.postgresql.org/docs/19/monitoring-stats.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 27.2. The Cumulative Statistics System"
[9]: https://www.postgresql.org/docs/19/logical-replication-sequences.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 29.7. Replicating Sequences"
