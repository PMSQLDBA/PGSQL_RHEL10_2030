# Phase 11 — PostgreSQL 19 Advanced HA & DR SOP

This phase builds on **Phase 9 physical streaming replication** and **Phase 10 logical replication**. The goal is to design and validate a PostgreSQL HA/DR architecture that covers **streaming replication, cascading replication, failover, switchover, logical failover slots, `pg_createsubscriber`, RPO/RTO testing, and operational runbooks**.

> **Version status:** PostgreSQL 19 documentation is still marked **unsupported/pre-release** by PostgreSQL. The PG19-specific features below should therefore be treated as lab/testing material until the release is officially supported. ([PostgreSQL][1])

---

# 11.1 Enterprise HA Architecture

A practical PostgreSQL 19 architecture:

```text
                         APPLICATIONS
                              |
                              v
                    +-------------------+
                    | VIP / LB / DNS    |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    | PG19 PRIMARY      |
                    | 192.168.0.145     |
                    +----+---------+----+
                         |         |
              Physical  |         | Physical
             Streaming  |         | Streaming
                         |         |
                         v         v
                +------------+  +------------+
                | STANDBY 01 |  | STANDBY 02 |
                | .146       |  | .147       |
                +------+-----+  +------------+
                       |
                       | Cascading / DR
                       v
                +------------+
                | DR STANDBY |
                | .148       |
                +------------+

                         |
                         | Logical Replication
                         v
                +----------------+
                | Reporting /    |
                | Migration /    |
                | Integration    |
                +----------------+
```

PostgreSQL physical standbys can themselves act as senders when **cascading replication** is used. ([PostgreSQL][1])

---

# 11.2 HA vs DR

Do not treat HA and DR as identical.

| Requirement        | HA                    | DR                       |
| ------------------ | --------------------- | ------------------------ |
| Primary failure    | Yes                   | Yes                      |
| Local standby      | Yes                   | Optional                 |
| Remote site        | Optional              | Yes                      |
| Automatic failover | Possible              | Possible                 |
| RPO                | Very low              | Depends on design        |
| RTO                | Seconds/minutes       | Minutes/hours            |
| Main purpose       | Availability          | Disaster recovery        |
| Typical technology | Streaming replication | Streaming + archive/PITR |

Example:

```text
HA:
Primary → Standby

DR:
Primary → Remote DR Standby
        → WAL archive
```

---

# 11.3 RPO and RTO

## RPO

**Recovery Point Objective**

How much data can potentially be lost?

```text
RPO = 0
```

means no committed transaction loss is acceptable.

This generally requires synchronous replication or another equivalent durability design.

---

## RTO

**Recovery Time Objective**

How long can the service remain unavailable?

Example:

```text
RTO = 5 minutes
```

means the service must be restored within five minutes.

### DBA planning matrix

| Design            | Typical RPO characteristic  | RTO characteristic |
| ----------------- | --------------------------- | ------------------ |
| Async standby     | Possible recent WAL loss    | Low                |
| Sync standby      | Can approach zero data loss | Low                |
| Archive/PITR only | Archive interval dependent  | Higher             |
| HA + DR           | Depends on failover path    | Low/medium         |

Never claim a specific RPO/RTO merely because streaming replication exists. It must be **tested**.

---

# 11.4 Primary Configuration

On primary:

```sql
SHOW wal_level;
SHOW max_wal_senders;
SHOW max_replication_slots;
SHOW synchronous_standby_names;
SHOW synchronous_commit;
```

Recommended lab baseline:

```conf
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
```

PostgreSQL documents `max_replication_slots` as controlling the maximum number of replication slots, with a default of 10. ([PostgreSQL][1])

---

# 11.5 Physical Replication Slots

Create a slot for each physical standby.

```sql
SELECT *
FROM pg_create_physical_replication_slot('standby01_slot');

SELECT *
FROM pg_create_physical_replication_slot('standby02_slot');
```

Validate:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    wal_status
FROM pg_replication_slots;
```

Expected:

```text
standby01_slot
standby02_slot
```

---

# 11.6 Why Replication Slots Matter

Without a replication slot, the primary can remove WAL before a disconnected standby receives it.

With a slot:

```text
Primary
   |
   | WAL
   v
Replication Slot
   |
   | protects required WAL
   v
Standby
```

But there is a major risk:

```text
Standby DOWN
      |
      v
Slot remains
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

PostgreSQL explicitly warns that replication slots can retain required WAL and catalog rows even when their consumers are no longer connected. ([PostgreSQL][2])

Therefore monitor:

```sql
SELECT
    slot_name,
    active,
    restart_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS retained_wal
FROM pg_replication_slots;
```

---

# 11.7 Cascading Replication

Instead of:

```text
PRIMARY
 | | |
 | | +---- Standby 3
 | +------ Standby 2
 +-------- Standby 1
```

you can use:

```text
PRIMARY
   |
   v
STANDBY 1
   |
   +----> STANDBY 2
             |
             +----> DR STANDBY
```

This is **cascading replication**.

### Advantages

* Reduces WAL sender connections on primary.
* Reduces network traffic from primary.
* Useful for remote-site replication.
* Useful for large environments.

### Disadvantages

* Adds another dependency.
* Failure of an upstream standby affects downstream replicas.
* Monitoring becomes more complicated.
* RPO can differ between replicas.

---

# 11.8 Cascading Standby Configuration

On downstream standby:

```conf
primary_conninfo =
'host=192.168.0.146 port=5432 user=repl_user application_name=standby02'

primary_slot_name = 'standby02_slot'
```

Architecture:

```text
192.168.0.145
PRIMARY
    |
    v
192.168.0.146
STANDBY01
    |
    v
192.168.0.147
STANDBY02
```

Verify on the upstream standby:

```sql
SELECT
    client_addr,
    application_name,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

---

# 11.9 Synchronous Replication

Suppose:

```text
PRIMARY
   |
   | synchronous
   v
STANDBY01
```

Configure:

```conf
synchronous_standby_names = 'standby01'
```

The standby's `application_name` must match.

For example:

```conf
primary_conninfo =
'host=192.168.0.145 port=5432 user=repl_user application_name=standby01'
```

PostgreSQL supports:

```conf
synchronous_standby_names = 'FIRST 1 (standby01, standby02)'
```

or:

```conf
synchronous_standby_names = 'ANY 1 (standby01, standby02)'
```

The `FIRST` and `ANY` forms determine how synchronous standby selection occurs. ([PostgreSQL][1])

---

# 11.10 `remote_write` vs `remote_flush` vs `remote_apply`

Important DBA concept:

```text
remote_write
    ↓
Standby received WAL and wrote it

remote_flush
    ↓
Standby flushed WAL to durable storage

remote_apply
    ↓
Standby replayed WAL
```

For example:

```conf
synchronous_commit = remote_apply
```

means a commit waits until the synchronous standby has applied the WAL.

PostgreSQL specifically notes that `remote_apply` causes commits to wait for replication and application on the standby. ([PostgreSQL][1])

### Trade-off

```text
Higher durability
       ↓
Higher commit latency
```

---

# 11.11 Failover vs Switchover

These terms must be clearly separated.

## Switchover

Planned event:

```text
PRIMARY
   |
   | planned shutdown
   v
STANDBY
   |
   v
PROMOTE
```

Used for:

* maintenance
* OS patching
* PostgreSQL upgrades
* planned DR testing

---

## Failover

Unplanned event:

```text
PRIMARY
   X
   |
   X FAILURE
   |
   v
STANDBY
   |
   v
PROMOTE
```

Used for:

* server failure
* storage failure
* network/site failure
* unrecoverable PostgreSQL failure

---

# 11.12 Planned Switchover SOP

### Step 1 — Check replication

On primary:

```sql
SELECT
    client_addr,
    application_name,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

You want:

```text
state     = streaming
sync_state = sync/async as designed
```

---

### Step 2 — Generate final WAL

```sql
SELECT pg_switch_wal();
```

Then confirm the standby has received/replayed it.

---

# 11.13 PostgreSQL 19 `WAIT FOR`

PG19 introduces the `WAIT FOR` command for waiting until a standby reaches an LSN at a specified state such as written, flushed, or replayed. ([PostgreSQL][3])

This is particularly useful for controlled switchover validation.

Conceptually:

```text
Primary transaction
       |
       v
Generate LSN
       |
       v
WAIT FOR standby
       |
       v
Standby confirmed
       |
       v
Promote standby
```

Before using exact syntax in your lab, validate it directly:

```sql
\h WAIT
```

because PG19 is still pre-release.

---

# 11.14 Promote Standby

On standby:

```sql
SELECT pg_promote();
```

Validate:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
f
```

Then:

```sql
SELECT
    pg_current_wal_lsn(),
    pg_is_in_recovery();
```

The promoted standby is now the new primary.

---

# 11.15 Critical Step — Fence the Old Primary

This is one of the most important HA rules.

Never simply promote a standby while the old primary might still accept writes.

Bad:

```text
OLD PRIMARY  → WRITE
       |
       X
       |
STANDBY → WRITE
```

You now have:

```text
           SPLIT BRAIN
```

Correct:

```text
OLD PRIMARY
     |
     v
FENCE / ISOLATE
     |
     X
     |
NEW PRIMARY
     |
     v
APPLICATION
```

Fencing may involve:

* powering off the old host
* disabling PostgreSQL
* removing network access
* fencing through cluster management
* preventing application connections
* cloud/VM orchestration controls

---

# 11.16 Split-Brain RCA

### Symptom

Two servers accept application writes.

### Evidence

```text
Primary A:
transaction IDs increasing

Primary B:
transaction IDs increasing
```

### Root cause

The old primary was not fenced before standby promotion.

### Impact

Potential:

* divergent data
* conflicting transactions
* inconsistent application state
* difficult reconciliation
* possible data loss

### Corrective action

```text
1. Fence old primary
2. Confirm only one writable primary
3. Promote standby
4. Rebuild old primary
5. Reintroduce it as standby
```

---

# 11.17 Rebuild Old Primary

After failover, the old primary generally should not simply be restarted and pointed back at the new primary.

Use a controlled rebuild.

Example:

```bash
sudo systemctl stop postgresql-19
```

Move the old data directory:

```bash
sudo mv /var/lib/pgsql/19/data \
        /var/lib/pgsql/19/data.old
```

Then take a fresh base backup:

```bash
pg_basebackup \
  -h NEW_PRIMARY_IP \
  -U repl_user \
  -D /var/lib/pgsql/19/data \
  -Fp \
  -Xs \
  -P \
  -R \
  -S standby_slot
```

Then:

```bash
sudo systemctl start postgresql-19
```

Validate:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
t
```

---

# 11.18 `pg_rewind`

When appropriate, `pg_rewind` can synchronize a former primary with the new primary after timeline divergence.

Typical scenario:

```text
OLD PRIMARY
     |
     | divergence
     v
NEW PRIMARY
```

Instead of transferring the entire database again, `pg_rewind` can rewind the old primary to the new primary's timeline when its prerequisites are satisfied.

Typical command:

```bash
pg_rewind \
  --target-pgdata=/var/lib/pgsql/19/data \
  --source-server="host=NEW_PRIMARY_IP port=5432 user=rewind_user dbname=postgres"
```

### Critical

Do not blindly run `pg_rewind`.

Validate:

```text
[ ] Target is no longer the active primary
[ ] New primary identified
[ ] Required WAL/history available
[ ] wal_log_hints enabled OR data checksums available
[ ] Correct permissions
[ ] Backup exists
[ ] Application cannot connect to target
```

For uncertain cases, rebuild using `pg_basebackup`.

---

# 11.19 PostgreSQL 19 Logical Failover Slots

Now combine Phase 10 with Phase 11.

Architecture:

```text
                  PRIMARY
                     |
        +------------+------------+
        |                         |
 Physical standby          Logical subscriber
        |                         |
        |                         |
 sync logical slots              |
        |                         |
        v                         |
   Standby slot                  |
        |                         |
        +------------+------------+
                     |
                  FAILOVER
                     |
                     v
                NEW PRIMARY
                     |
                     v
          Logical subscriber
          continues from slot
```

PG19 supports logical replication slots configured with:

```text
failover = true
```

so they can be synchronized to physical standbys. ([PostgreSQL][4])

---

# 11.20 Configure Slot Synchronization

On physical standby:

```conf
sync_replication_slots = on
```

This allows logical failover slots from the primary to be synchronized to the standby. ([PostgreSQL][1])

The PostgreSQL documentation recommends automatic synchronization through `sync_replication_slots` rather than relying only on manual `pg_sync_replication_slots()`. ([PostgreSQL][2])

---

# 11.21 `synchronized_standby_slots`

On primary:

```conf
synchronized_standby_slots = 'standby01_slot'
```

This tells logical WAL sender processes to wait for the specified physical replication slot to confirm WAL receipt before allowing logical failover-slot consumers to proceed. ([PostgreSQL][1])

Architecture:

```text
Application
    |
    v
PRIMARY
    |
    +------ Physical WAL ------> STANDBY
    |
    +------ Logical WAL -------> Subscriber
                  |
                  |
        waits for physical
        standby confirmation
```

This helps ensure the logical subscriber does not advance beyond WAL that has safely reached the physical standby.

---

# 11.22 Verify Failover Slots

On primary:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    confirmed_flush_lsn,
    failover,
    synced,
    invalidation_reason,
    slotsync_skip_reason
FROM pg_replication_slots;
```

Expected logical slot:

```text
failover = true
```

On standby:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    failover,
    synced
FROM pg_replication_slots;
```

Expected:

```text
synced = true
```

PG19 exposes these fields specifically for monitoring slot synchronization. ([PostgreSQL][2])

---

# 11.23 `pg_sync_replication_slots()`

Manual synchronization:

```sql
SELECT pg_sync_replication_slots();
```

However, PostgreSQL's documentation distinguishes this from continuous automatic synchronization.

For production HA:

```text
sync_replication_slots = on
```

is the preferred mechanism for continuous synchronization. ([PostgreSQL][2])

---

# 11.24 `pg_createsubscriber`

This is a major PG19 feature.

It can convert a **physical standby into a logical replica**.

Instead of:

```text
10 TB database
      |
      v
Initial logical copy
      |
      v
10 TB transferred again
```

you can use:

```text
PRIMARY
   |
physical replication
   |
   v
PHYSICAL STANDBY
   |
pg_createsubscriber
   |
   v
LOGICAL REPLICA
```

`pg_createsubscriber` does **not** repeat the initial table data copy; it converts the physical standby and performs the required synchronization. PostgreSQL specifically documents it as useful for large databases where the initial logical copy would otherwise take substantial time. ([PostgreSQL][5])

---

# 11.25 `pg_createsubscriber` Prerequisites

Important requirements include:

```text
[ ] Same major version
[ ] Target is physical standby
[ ] Same system identifier initially
[ ] Source is not in recovery
[ ] wal_level = replica or logical
[ ] Enough replication slots
[ ] Enough WAL senders
[ ] Enough logical replication workers
[ ] Enough replication origins
[ ] Target accepts local connections
```

The official PG19 documentation provides these prerequisites and warns that a failed conversion can leave the target needing to be recreated as a standby. ([PostgreSQL][5])

---

# 11.26 Example `pg_createsubscriber`

On the target physical standby:

```bash
/usr/pgsql-19/bin/pg_createsubscriber \
  -D /var/lib/pgsql/19/data \
  -P "host=192.168.0.145 port=5432 user=repl_user dbname=postgres" \
  -d appdb \
  --verbose
```

The command creates the logical replication objects and synchronizes from the appropriate point rather than copying the entire database again. ([PostgreSQL][5])

### Important warning

Do **not** experiment with this on your only working standby.

Use:

```text
PRIMARY
   |
   +---- STANDBY-A   ← production HA
   |
   +---- STANDBY-B   ← lab for pg_createsubscriber
```

---

# 11.27 Advanced HA Monitoring Dashboard

Monitor these categories.

### Primary

```sql
SELECT
    client_addr,
    application_name,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

### Slots

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    wal_status,
    failover,
    synced,
    invalidation_reason
FROM pg_replication_slots;
```

### WAL

```sql
SELECT
    pg_current_wal_lsn();
```

### Standby

```sql
SELECT
    pg_is_in_recovery(),
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn(),
    pg_last_xact_replay_timestamp();
```

---

# 11.28 Replication Lag

Basic byte lag:

```sql
SELECT
    application_name,
    client_addr,
    pg_size_pretty(
        pg_wal_lsn_diff(
            sent_lsn,
            replay_lsn
        )
    ) AS replay_lag
FROM pg_stat_replication;
```

Interpret together with:

```text
write_lsn
flush_lsn
replay_lsn
```

because a standby may have:

```text
received WAL
but not flushed it
```

or:

```text
flushed WAL
but not replayed it
```

---

# 11.29 HA Health Status

Create a simple decision tree:

```text
Replication connection?
        |
      YES
        |
        v
WAL streaming?
        |
      YES
        |
        v
Replay progressing?
        |
      YES
        |
        v
Lag within SLA?
        |
      YES
        |
        v
          HEALTHY
```

If:

```text
NO
```

investigate:

```text
Network
Authentication
pg_hba.conf
WAL retention
Replication slot
Disk space
Standby recovery
Long-running transaction
I/O
CPU
Replication worker
```

---

# 11.30 Disaster Simulation

## Scenario

Primary:

```text
192.168.0.145
```

Standby:

```text
192.168.0.146
```

Simulate failure:

```bash
sudo systemctl stop postgresql-19
```

From standby:

```sql
SELECT pg_is_in_recovery();
```

Then:

```sql
SELECT pg_promote();
```

Validate:

```sql
SELECT
    pg_is_in_recovery(),
    pg_current_wal_lsn();
```

Expected:

```text
pg_is_in_recovery
-----------------
f
```

Now test application connectivity against the new primary.

---

# 11.31 RPO Test

Before failure:

```sql
CREATE TABLE rpo_test
(
    id BIGINT PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp()
);
```

Insert:

```sql
INSERT INTO rpo_test(id)
VALUES (1);
```

Immediately record:

```sql
SELECT
    clock_timestamp(),
    pg_current_wal_lsn();
```

Promote standby.

Then:

```sql
SELECT *
FROM rpo_test
WHERE id = 1;
```

Result:

```text
FOUND
```

means the transaction reached the standby before promotion.

Repeat this test under:

* async replication
* synchronous replication

and document the results.

---

# 11.32 RTO Test

Record:

```text
T1 = failure detected
T2 = standby promoted
T3 = application endpoint changed
T4 = application connected
T5 = first successful transaction
```

Calculate:

```text
RTO = T5 - T1
```

Do not claim an RTO until this has actually been measured.

---

# 11.33 Complete HA Test Matrix

| Test                  | Expected Result               |
| --------------------- | ----------------------------- |
| Primary restart       | Standby remains available     |
| Primary crash         | Standby can be promoted       |
| Network interruption  | Standby reconnects            |
| WAL backlog           | Alert generated               |
| Slot inactive         | Alert generated               |
| Disk full             | Alert generated               |
| Planned switchover    | No unexpected data loss       |
| Unplanned failover    | RPO within SLA                |
| Application reconnect | Successful                    |
| Old primary recovery  | Rejoins as standby            |
| Logical subscriber    | Continues replication         |
| Logical failover slot | Available on promoted primary |
| Cascading standby     | Receives WAL                  |
| DR standby            | Meets lag requirement         |

---

# 11.34 Production HA Failure Decision Tree

```text
PRIMARY FAILURE
      |
      v
Is primary recoverable quickly?
      |
   +--+--+
   |     |
  YES    NO
   |     |
   v     v
Repair  FAILOVER
         |
         v
Fence old primary
         |
         v
Confirm standby health
         |
         v
Promote standby
         |
         v
Change application endpoint
         |
         v
Validate transactions
         |
         v
Rebuild old primary
         |
         v
Rejoin as standby
```

---

# 11.35 DBA Golden Rules

### Rule 1

**Never promote without considering split-brain.**

### Rule 2

**Never leave replication slots unmanaged.**

### Rule 3

**Never call asynchronous replication "zero data loss."**

### Rule 4

**Never assume a standby is healthy because `pg_is_in_recovery()` returns true.**

Check:

```text
WAL receive
WAL replay
lag
disk
slot
errors
```

### Rule 5

**Never test failover only theoretically.**

Perform an actual controlled test.

### Rule 6

**Never return the old primary directly to production after an unplanned failover without validating timeline/data consistency.**

### Rule 7

**HA without endpoint management is incomplete.**

Your application must know where the active primary is.

---

# 11.36 Phase 11 Final Architecture

```text
                         APPLICATION
                              |
                         VIP / DNS / LB
                              |
                              v
                    +-------------------+
                    | PG19 PRIMARY      |
                    | .145              |
                    +---------+---------+
                              |
                +-------------+-------------+
                |                           |
          Physical WAL                 Logical WAL
                |                           |
                v                           v
        +---------------+           +---------------+
        | STANDBY 01    |           | SUBSCRIBER    |
        | .146          |           | Reporting     |
        +-------+-------+           +---------------+
                |
                | Cascading
                v
        +---------------+
        | STANDBY 02    |
        | .147          |
        +-------+-------+
                |
                | DR
                v
        +---------------+
        | DR STANDBY    |
        | .148          |
        +---------------+

        Logical failover slots
                |
                v
        Slot synchronization
                |
                v
          Failover continuity
```

## Phase 11 Key Takeaways

1. **Physical streaming replication** → HA/DR.
2. **Cascading replication** → reduce primary network/WAL-sender load.
3. **Synchronous replication** → stronger durability with commit-latency tradeoff.
4. **Failover** → unplanned promotion.
5. **Switchover** → planned role transition.
6. **Fencing** → critical to prevent split-brain.
7. **Replication slots** → protect required WAL but can consume disk.
8. **Logical failover slots** → PG19 capability for maintaining logical replication across physical failover.
9. **`sync_replication_slots`** → continuously synchronize logical failover slots to standby.
10. **`synchronized_standby_slots`** → coordinate logical WAL availability with physical standby receipt.
11. **`pg_createsubscriber`** → PG19 mechanism to convert a physical standby into a logical replica without repeating a full initial data copy.
12. **RPO/RTO must be measured**, not assumed.

The PG19 documentation confirms the failover-slot synchronization model and the `pg_createsubscriber` workflow; it also explicitly warns that `pg_createsubscriber` failure can leave the target requiring a fresh standby rebuild. ([PostgreSQL][5])

### Official PostgreSQL references

* [PostgreSQL 19 Replication Configuration](https://www.postgresql.org/docs/19/runtime-config-replication.html?utm_source=chatgpt.com)
* [PostgreSQL 19 pg_createsubscriber](https://www.postgresql.org/docs/19/app-pgcreatesubscriber.html?utm_source=chatgpt.com)
* [PostgreSQL 19 Logical Decoding](https://www.postgresql.org/docs/19/logicaldecoding-explanation.html?utm_source=chatgpt.com)
* [PostgreSQL 19 Streaming Replication Protocol](https://www.postgresql.org/docs/19/protocol-replication.html?utm_source=chatgpt.com)
* [PostgreSQL 19 Release Notes](https://www.postgresql.org/docs/19/release-19.html?utm_source=chatgpt.com)

**Next phase: Phase 12 — PostgreSQL 19 Backup & Recovery Master SOP: pgBackRest architecture, full/differential/incremental backups, WAL archiving, encryption, retention, restore, PITR, corruption recovery, backup validation, `verify`, disaster recovery drills, and production backup strategy.**

[1]: https://www.postgresql.org/docs/19/runtime-config-replication.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 19.6. Replication"
[2]: https://www.postgresql.org/docs/19/logicaldecoding-explanation.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 47.2. Logical Decoding Concepts"
[3]: https://www.postgresql.org/docs/19/release-19.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: E.1. Release 19"
[4]: https://www.postgresql.org/docs/19/protocol-replication.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 54.4. Streaming Replication Protocol"
[5]: https://www.postgresql.org/docs/19/app-pgcreatesubscriber.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: pg_createsubscriber"
