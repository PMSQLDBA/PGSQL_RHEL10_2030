# Phase 15 — PostgreSQL 19 High Availability, Failover & Switchover Master SOP

We continue from **Phase 15 of 20**.

This phase focuses on the area most important for an enterprise PostgreSQL DBA: **designing, operating, monitoring, and troubleshooting HA using PostgreSQL streaming replication**.

---

## 15.1 HA Architecture

A standard PostgreSQL HA design:

```text
                    APPLICATION
                         |
                         v
                  HAProxy / VIP
                         |
                 +-------+-------+
                 |               |
                 v               v
            PRIMARY          STANDBY
          PostgreSQL 19    PostgreSQL 19
                 |               |
                 +------ WAL ----+
                         |
                         v
                   DR / Replica
```

PostgreSQL's native physical streaming replication allows WAL generated on the primary to be streamed to standby servers. A standby can continuously replay WAL and can be promoted when required.

---

# 15.2 Primary vs Standby

### Primary

Responsible for:

```text
INSERT
UPDATE
DELETE
DDL
WAL generation
```

### Physical Standby

Normally:

```text
Read-only
Receives WAL
Replays WAL
Can serve read workloads
Can be promoted
```

Architecture:

```text
PRIMARY
  |
  | WAL streaming
  v
STANDBY
  |
  | WAL replay
  v
DATABASE STATE
```

---

# 15.3 Check Current Server Role

Run on PostgreSQL:

```sql
SELECT
    pg_is_in_recovery();
```

Result:

```text
false
```

means:

```text
PRIMARY
```

Result:

```text
true
```

means:

```text
STANDBY / RECOVERY
```

This is one of the most important PostgreSQL DBA commands.

---

# 15.4 Primary Configuration

Check:

```sql
SHOW wal_level;
SHOW max_wal_senders;
SHOW max_replication_slots;
SHOW hot_standby;
SHOW synchronous_standby_names;
```

For physical replication, the primary should have an appropriate WAL level and sufficient sender/slot capacity.

---

# 15.5 Create Replication User

On primary:

```sql
CREATE ROLE repl_user
WITH
    LOGIN
    REPLICATION
    PASSWORD 'CHANGE_THIS_PASSWORD';
```

Validate:

```sql
SELECT
    rolname,
    rolreplication,
    rolcanlogin
FROM pg_roles
WHERE rolname = 'repl_user';
```

Expected:

```text
rolreplication = true
rolcanlogin    = true
```

---

# 15.6 `pg_hba.conf`

On the primary:

```bash
sudo vi /var/lib/pgsql/19/data/pg_hba.conf
```

Example:

```conf
host    replication    repl_user    10.10.10.20/32    scram-sha-256
```

Use the **actual standby IP**, not a broad network range unless there is a specific reason.

Reload:

```bash
sudo systemctl reload postgresql-19
```

---

# 15.7 Primary Replication Settings

Example:

```conf
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
```

For a standby using a replication slot:

```conf
max_replication_slots = 10
```

The exact values depend on:

```text
Number of standbys
Backup tools
Logical replication
WAL workload
Connection requirements
```

Do not copy arbitrary values into production.

---

# 15.8 Create Physical Replication Slot

On primary:

```sql
SELECT pg_create_physical_replication_slot('standby1_slot');
```

Verify:

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

---

# 15.9 Why Replication Slots Matter

Without a slot:

```text
PRIMARY
   |
   +---- WAL
   |
   v
Standby
```

If the standby falls behind enough, required WAL may be recycled.

Then:

```text
Standby
   |
   X
Missing WAL
```

and the standby may require reinitialization.

With a replication slot:

```text
PRIMARY
   |
   +---- WAL retained
   |
   v
Replication Slot
   |
   v
Standby
```

The primary retains WAL needed by the slot.

### Critical warning

An abandoned replication slot can cause:

```text
WAL retention
     ↓
pg_wal growth
     ↓
filesystem exhaustion
     ↓
PostgreSQL outage
```

Therefore **replication slots must be monitored**.

---

# 15.10 Prepare Standby

Stop PostgreSQL on the standby:

```bash
sudo systemctl stop postgresql-19
```

Verify:

```bash
sudo systemctl is-active postgresql-19
```

Expected:

```text
inactive
```

---

# 15.11 Initialize Standby with `pg_basebackup`

From standby:

```bash
sudo -u postgres pg_basebackup \
    -h PRIMARY_IP \
    -U repl_user \
    -D /var/lib/pgsql/19/data \
    -Fp \
    -Xs \
    -P \
    -R \
    -S standby1_slot
```

Important options:

```text
-D
destination

-Xs
stream WAL during backup

-R
automatically create standby connection configuration

-S
use replication slot
```

`pg_basebackup` is designed to create a base backup suitable for creating a standby server.

---

# 15.12 Verify Standby Configuration

Check:

```bash
sudo ls -l /var/lib/pgsql/19/data/
```

Look for:

```text
standby.signal
postgresql.auto.conf
```

`-R` causes the standby connection configuration to be written automatically.

---

# 15.13 Start Standby

```bash
sudo systemctl start postgresql-19
```

Check:

```bash
sudo systemctl status postgresql-19
```

Then:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
t
```

---

# 15.14 Verify Replication from Primary

On primary:

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

Expected:

```text
state = streaming
```

---

# 15.15 Verify WAL Receiver on Standby

On standby:

```sql
SELECT
    status,
    sender_host,
    sender_port,
    slot_name,
    written_lsn,
    flushed_lsn,
    latest_end_lsn,
    latest_end_time
FROM pg_stat_wal_receiver;
```

Expected:

```text
status = streaming
```

---

# 15.16 Test Replication

On primary:

```sql
CREATE TABLE ha_test
(
    id integer PRIMARY KEY,
    message text,
    created_at timestamptz DEFAULT now()
);
```

```sql
INSERT INTO ha_test(id, message)
VALUES
(1, 'Replication Test');
```

On standby:

```sql
SELECT *
FROM ha_test;
```

Expected:

```text
1 | Replication Test
```

This confirms:

```text
Primary
   ↓
WAL
   ↓
Standby
   ↓
Replay
   ↓
Data visible
```

---

# 15.17 Monitor Replication Lag

On primary:

```sql
SELECT
    application_name,
    client_addr,
    state,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

Also calculate LSN differences when appropriate.

Example:

```sql
SELECT
    application_name,
    pg_size_pretty(
        pg_wal_lsn_diff(sent_lsn, replay_lsn)
    ) AS replay_lag_bytes
FROM pg_stat_replication;
```

---

# 15.18 Important — Lag Has Multiple Meanings

You should distinguish:

```text
Write lag
Flush lag
Replay lag
```

Conceptually:

```text
Primary WAL
    |
    +--> sent
           |
           +--> written
                  |
                  +--> flushed
                         |
                         +--> replayed
```

Therefore:

```text
Network delay
Disk flush delay
Recovery/replay delay
```

can have different root causes.

---

# 15.19 Replication Troubleshooting

If:

```text
state != streaming
```

investigate:

```text
1. PostgreSQL service
2. Network connectivity
3. Port 5432
4. pg_hba.conf
5. Replication user
6. Password
7. TLS requirements
8. Replication slot
9. WAL availability
10. PostgreSQL logs
```

Test network:

```bash
nc -zv PRIMARY_IP 5432
```

or:

```bash
psql -h PRIMARY_IP -U repl_user -d postgres
```

---

# 15.20 Standby Logs

On RHEL:

```bash
sudo journalctl -u postgresql-19 --since "30 minutes ago"
```

Look for:

```text
FATAL
ERROR
PANIC
replication
WAL
streaming
authentication
connection
```

---

# 15.21 Planned Switchover

A **switchover** is a controlled role change.

Example:

```text
BEFORE

PRIMARY
   |
   v
STANDBY
```

After:

```text
AFTER

OLD PRIMARY → STANDBY
       ^
       |
NEW PRIMARY
```

This is normally used for:

```text
Planned maintenance
OS patching
PostgreSQL maintenance
Infrastructure migration
Data-center maintenance
```

---

# 15.22 Switchover Preconditions

Before switchover:

```text
[ ] Standby healthy
[ ] Standby connected
[ ] WAL caught up
[ ] No unexpected replication errors
[ ] Application connections understood
[ ] Backup healthy
[ ] DNS/VIP/HAProxy plan ready
[ ] Rollback plan documented
```

---

# 15.23 Check Standby Replay Position

On primary:

```sql
SELECT
    pg_current_wal_lsn();
```

On standby:

```sql
SELECT
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();
```

For a controlled switchover, you want the standby synchronized to the required point.

---

# 15.24 Promote Standby

On the standby:

```sql
SELECT pg_promote();
```

Check:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
f
```

The standby is now primary.

---

# 15.25 What Happens to the Old Primary?

After promotion:

```text
OLD PRIMARY
     |
     X
No longer primary
```

Do **not** simply start it as though nothing happened.

You must prevent split brain.

---

# 15.26 Split-Brain

The most dangerous HA failure:

```text
             APPLICATION
               /      \
              /        \
             v          v
        PRIMARY A    PRIMARY B
             |          |
             +----X-----+
```

Both nodes accept writes.

This can create:

```text
Data divergence
Conflicting transactions
Data loss
Difficult reconciliation
```

### Golden Rule

> **Never allow two PostgreSQL nodes to independently believe they are the authoritative primary.**

---

# 15.27 Rebuild Old Primary

After a failover/switchover, the old primary generally needs to be converted into a standby using an appropriate rejoin process.

One common method is:

```bash
pg_rewind
```

provided its prerequisites are satisfied.

Concept:

```text
OLD PRIMARY
     |
     v
pg_rewind
     |
     v
NEW PRIMARY timeline
     |
     v
OLD PRIMARY becomes STANDBY
```

Then configure:

```text
standby.signal
primary_conninfo
```

and restart.

---

# 15.28 Timeline Concept

PostgreSQL uses timelines to distinguish divergent histories after recovery or promotion.

Before promotion:

```text
Timeline 1
   |
   +---- Primary
   |
   +---- Standby
```

After standby promotion:

```text
Timeline 1
       |
       +---- old history
       |
       +---- Timeline 2
                |
                +---- New Primary
```

This is essential for understanding:

```text
PITR
Failover
pg_rewind
Backup/recovery
```

---

# 15.29 Automatic Failover

Native PostgreSQL streaming replication provides the replication mechanism, but **automatic leader election/failover orchestration is not provided simply by enabling streaming replication**.

For automated HA, common architecture includes:

```text
PostgreSQL
     |
 Patroni
     |
 +---+---+
 |       |
etcd   Consul
 |
Leader election
 |
HAProxy
 |
Application
```

The exact orchestration technology should be selected based on operational requirements.

---

# 15.30 Patroni Architecture

Typical:

```text
              APPLICATION
                   |
                HAProxy
                   |
          +--------+--------+
          |                 |
          v                 v
      PostgreSQL A      PostgreSQL B
          |                 |
          +--------+--------+
                   |
                Patroni
                   |
             Distributed
             Configuration
             Store
```

Patroni manages PostgreSQL HA state and coordinates leader election/failover with a distributed consensus store.

---

# 15.31 HAProxy Role

HAProxy sits between:

```text
Application
     |
     v
HAProxy
     |
 +---+---+
 |       |
 v       v
Primary Standby
```

The application connects to:

```text
HAProxy
```

rather than directly embedding a database server IP.

This allows infrastructure to change underneath the application.

---

# 15.32 Read/Write Routing

Enterprise architecture can separate:

```text
WRITE
  |
  v
PRIMARY
```

and:

```text
READ
  |
  v
READ REPLICAS
```

But application-level read-after-write consistency must be considered.

A recently committed transaction on primary may not yet have replayed on a standby.

Therefore:

```text
WRITE primary
   ↓
Immediately READ standby
```

can produce stale-read behavior depending on replication state.

---

# 15.33 Synchronous Replication

Asynchronous:

```text
Primary
   |
   | WAL
   v
Standby
```

Primary does not normally wait for standby replay before confirming a commit.

Synchronous:

```text
Primary
   |
   | WAL
   v
Synchronous Standby
   |
   v
Acknowledgement
   |
   v
COMMIT returns
```

This can reduce the amount of committed data potentially lost during certain failures, but introduces dependency on standby availability and network/storage latency.

---

# 15.34 Configure Synchronous Standby

Example:

```conf
synchronous_standby_names = 'FIRST 1 (standby1)'
```

The exact syntax and quorum design should be selected carefully for the topology.

Then:

```sql
SELECT
    application_name,
    sync_state,
    sync_priority
FROM pg_stat_replication;
```

Expected:

```text
sync_state = sync
```

or another state appropriate to your configuration.

---

# 15.35 Synchronous vs Asynchronous

| Feature               | Async      | Sync                           |
| --------------------- | ---------- | ------------------------------ |
| Commit latency        | Lower      | Potentially higher             |
| Dependency on standby | Lower      | Higher                         |
| Potential data loss   | Greater    | Reduced                        |
| Network sensitivity   | Lower      | Higher                         |
| HA complexity         | Lower      | Higher                         |
| Typical use           | General HA | Strict durability requirements |

Neither is universally appropriate.

The choice should follow:

```text
RPO
RTO
network latency
application workload
availability requirements
```

---

# 15.36 Multiple Standbys

Example:

```text
                 PRIMARY
                /   |   \
               /    |    \
              v     v     v
            S1      S2     S3
```

Possible roles:

```text
S1 = synchronous standby
S2 = asynchronous standby
S3 = DR standby
```

This can provide:

```text
HA
Read scaling
DR
Maintenance flexibility
```

---

# 15.37 Cascading Replication

Instead of:

```text
PRIMARY
 | | |
 v v v
S1 S2 S3
```

you can use:

```text
PRIMARY
   |
   v
  S1
 /  \
v    v
S2   S3
```

This is cascading replication.

Benefits:

```text
Reduced primary outbound replication connections
Reduced network traffic from primary
Useful for geographically distributed topologies
```

Trade-off:

```text
Additional dependency
```

If S1 fails, S2/S3 may be affected depending on topology.

---

# 15.38 Three-Node Enterprise Example

```text
                 APPLICATION
                      |
                    HAProxy
                      |
                      v
                   PRIMARY
                 /    |    \
                /     |     \
               v      v      v
             HA-1    HA-2    DR
           sync     async    async
```

Possible deployment:

```text
PRIMARY
New Jersey

HA-1
New York

HA-2
Virginia

DR
Texas
```

The actual locations depend on the organization's network and DR architecture.

---

# 15.39 HA Failure Scenario

### Incident

Primary server crashes.

```text
PRIMARY
   X
```

Standby:

```text
STANDBY
   |
   v
PROMOTE
```

New architecture:

```text
APPLICATION
     |
     v
NEW PRIMARY
     |
     v
OLD PRIMARY
(rebuild as standby)
```

---

# 15.40 Failover RCA

Document:

```text
INCIDENT:
Primary unavailable.

DETECTION:
HA monitoring detected primary failure.

IMPACT:
Write workload unavailable for X minutes.

ACTION:
Standby promoted.

VALIDATION:
pg_is_in_recovery() = false.

APPLICATION:
Connections redirected to new primary.

RECOVERY:
Old primary rebuilt/rejoined as standby.

RPO:
Measured value.

RTO:
Measured value.

ROOT CAUSE:
Document actual infrastructure/database cause.

PREVENTION:
Corrective action.
```

---

# 15.41 HA Validation Checklist

Before declaring HA production-ready:

```text
[ ] Streaming replication works
[ ] WAL receiver healthy
[ ] Replication slots monitored
[ ] Lag monitored
[ ] Backup independent of standby
[ ] Switchover tested
[ ] Failover tested
[ ] Application reconnection tested
[ ] DNS/VIP/HAProxy tested
[ ] Old primary rejoin tested
[ ] Split-brain protection tested
[ ] PITR tested
[ ] DR recovery tested
[ ] Monitoring alerts tested
[ ] RPO measured
[ ] RTO measured
```

---

# 15.42 The Most Important HA Metrics

Monitor:

```text
pg_stat_replication
pg_stat_wal_receiver
pg_replication_slots
pg_stat_wal
pg_stat_archiver
pg_stat_database
pg_stat_io
```

Also monitor OS:

```text
CPU
Memory
Disk
Network
Filesystem
```

---

# 15.43 HA DBA Decision Tree

```text
REPLICATION PROBLEM
        |
        v
Primary reachable?
    |
   NO ----------------> Primary outage
    |                         |
   YES                        v
    |                     Failover
    v
Standby connected?
    |
   NO
    |
    +--> Network
    +--> Authentication
    +--> pg_hba.conf
    +--> PostgreSQL service
    +--> Logs
    +--> Slot
    |
   YES
    |
    v
Streaming?
    |
   NO
    |
    +--> WAL availability
    +--> Receiver error
    +--> Slot
    +--> Storage
    |
   YES
    |
    v
Lagging?
    |
   YES
    |
    +--> Network
    +--> Standby I/O
    +--> Replay workload
    +--> Primary WAL rate
    +--> Locks/resources
```

---

# 15.44 Production HA Golden Rules

1. **Replication is not backup.**
2. **A standby is not automatically a DR strategy.**
3. **Monitor replication lag continuously.**
4. **Monitor replication slots continuously.**
5. **Never allow split brain.**
6. **Test failover—not just replication.**
7. **Test application reconnection.**
8. **Test rejoining the old primary.**
9. **Measure actual RPO/RTO.**
10. **Keep an independent backup repository.**
11. **Understand timelines before using `pg_rewind` or PITR.**
12. **Choose synchronous replication based on business RPO/RTO requirements, not simply because it sounds safer.**

---

## Phase 15 Completed

Your PostgreSQL 19 DBA track now stands at:

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
15  HA / Failover / Switchover       ← COMPLETED
```

### Next: Phase 16

**PostgreSQL 19 Security & Encryption Master SOP**

It will cover:

* PostgreSQL authentication architecture
* `pg_hba.conf`
* SCRAM
* TLS/SSL
* certificate-based authentication
* encryption in transit
* encryption at rest on **RHEL 10**
* LUKS
* filesystem/database separation
* backup encryption
* column/data encryption
* `pgcrypto`
* KMS concepts
* key management
* role/privilege design
* auditing
* security hardening
* security RCA scenarios
* production security checklist.
