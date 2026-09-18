# Phase 9 — PostgreSQL 19 High Availability & Streaming Replication SOP

This phase builds a **Primary → Physical Standby** architecture and then validates **replication, lag, failover, promotion, timeline changes, and standby rebuild**.

PostgreSQL 19 continues to use physical streaming replication for primary/standby HA. PostgreSQL 19 also adds replication-related capabilities such as `WAIT FOR`, improved replication-slot synchronization, and additional failover support for logical replication slots. ([PostgreSQL][1])

---

## 1. Target architecture

For your lab, use two RHEL 10 VMs:

```text
                 PostgreSQL 19 HA LAB

        PRIMARY                         STANDBY
   +---------------+               +---------------+
   | RHEL 10       |               | RHEL 10       |
   | PostgreSQL 19 |               | PostgreSQL 19 |
   |               |               |               |
   |  PG-PRIMARY   |==============>|  PG-STANDBY   |
   |               |   Streaming   |               |
   +---------------+  Replication  +---------------+
          |                                |
          |                                |
       Read/Write                        Read Only
          |                                |
          +---------------+----------------+
                          |
                       WAL/PITR
```

The important distinction:

```text
Primary
  └── accepts writes

Physical standby
  └── receives WAL
  └── replays WAL
  └── can serve read-only queries when hot standby is enabled
```

PostgreSQL describes physical replication as a primary sending WAL and a standby receiving/replaying it. ([PostgreSQL][2])

---

# 2. RPO and RTO

Before configuring HA, define the objectives.

### RPO

**Recovery Point Objective**

> How much data can potentially be lost?

Example:

```text
RPO = 0
```

would require synchronous replication and careful failure semantics.

### RTO

**Recovery Time Objective**

> How quickly must service be restored?

Example:

```text
RTO = 5 minutes
```

means your operational failover process must restore service within five minutes.

Do not assume:

```text
Streaming replication = zero data loss
```

Asynchronous replication can have replication lag.

---

# 3. Example lab IP design

Use placeholders until you confirm the actual IPs:

```text
PRIMARY
Hostname: pg19-primary
IP:      <PRIMARY_IP>

STANDBY
Hostname: pg19-standby
IP:      <STANDBY_IP>

DBA
IP:      <DBA_IP>
```

For example only:

```text
PRIMARY   192.168.0.145
STANDBY   192.168.0.146
DBA       192.168.0.73
```

**Do not configure those example addresses unless they are actually assigned to your VMs.**

---

# 4. Verify both servers

Run on both:

```bash
hostnamectl

cat /etc/redhat-release

getenforce

/usr/pgsql-19/bin/psql --version

systemctl status postgresql-19 --no-pager
```

Expected:

```text
RHEL 10
PostgreSQL 19
SELinux Enforcing
postgresql-19 active
```

---

# 5. Verify PostgreSQL data directory

On each server:

```bash
sudo -u postgres psql -Atc \
"SHOW data_directory;"
```

For the PGDG installation used in our earlier phases, it should normally resemble:

```text
/var/lib/pgsql/19/data
```

But **always verify rather than hard-code it**.

---

# 6. Primary — configure WAL

On the primary:

```sql
SHOW wal_level;

SHOW max_wal_senders;

SHOW max_replication_slots;
```

For physical streaming replication:

```sql
ALTER SYSTEM SET wal_level = 'replica';

ALTER SYSTEM SET max_wal_senders = '10';

ALTER SYSTEM SET max_replication_slots = '10';
```

These are startup parameters, so restart:

```bash
sudo systemctl restart postgresql-19
```

Verify:

```sql
SHOW wal_level;

SHOW max_wal_senders;

SHOW max_replication_slots;
```

PostgreSQL documents that `wal_level` must be `replica` or higher for physical standby connections, while `max_replication_slots` controls the number of replication slots available. ([PostgreSQL][2])

---

# 7. Create replication user

On the primary:

```sql
CREATE ROLE repl_user
WITH
    REPLICATION
    LOGIN
    PASSWORD 'CHANGE_THIS_PASSWORD';
```

Verify:

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

Do not grant:

```text
SUPERUSER
CREATEDB
CREATEROLE
```

to this role unless there is a documented requirement.

---

# 8. Secure the replication password

Rather than putting a password directly into shell history or repeatedly into commands, use `.pgpass` on the standby.

On standby:

```bash
sudo -u postgres vi /var/lib/pgsql/.pgpass
```

Example:

```text
<PRIMARY_IP>:5432:replication:repl_user:CHANGE_THIS_PASSWORD
```

Secure it:

```bash
sudo chown postgres:postgres /var/lib/pgsql/.pgpass

sudo chmod 600 /var/lib/pgsql/.pgpass
```

PostgreSQL supports supplying the replication password through `.pgpass`, using `replication` as the database name. ([PostgreSQL][2])

---

# 9. Primary — configure `pg_hba.conf`

Find the file:

```sql
SHOW hba_file;
```

Add a tightly scoped replication rule:

```conf
host    replication    repl_user    <STANDBY_IP>/32    scram-sha-256
```

For example:

```conf
host    replication    repl_user    192.168.0.146/32    scram-sha-256
```

**Do not use:**

```conf
host replication repl_user 0.0.0.0/0 scram-sha-256
```

unless this is an isolated temporary lab.

Reload:

```bash
sudo systemctl reload postgresql-19
```

Validate:

```sql
SELECT *
FROM pg_hba_file_rules
WHERE error IS NOT NULL;
```

Expected:

```text
0 rows
```

---

# 10. Firewall on primary

Allow TCP/5432 only from the standby:

```bash
sudo firewall-cmd \
--permanent \
--add-rich-rule='rule family="ipv4" source address="<STANDBY_IP>/32" port port="5432" protocol="tcp" accept'
```

Then:

```bash
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --list-rich-rules
```

---

# 11. Test network connectivity

From standby:

```bash
nc -zv <PRIMARY_IP> 5432
```

or:

```bash
psql \
-h <PRIMARY_IP> \
-U repl_user \
-d postgres
```

The second command also tests authentication.

---

# 12. Create a physical replication slot

On primary:

```sql
SELECT *
FROM pg_create_physical_replication_slot('pg19_standby_slot');
```

Verify:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn
FROM pg_replication_slots;
```

You should see:

```text
pg19_standby_slot
physical
false
```

A physical slot prevents required WAL from being removed before the standby has consumed it.

### Critical warning

A replication slot is **not free storage**.

If the standby disappears while the slot remains active, WAL can accumulate on the primary.

PostgreSQL explicitly warns that replication slots persist independently of their consumers and can cause storage consumption if abandoned. ([PostgreSQL][3])

---

# 13. Check WAL retention

```sql
SELECT
    slot_name,
    active,
    restart_lsn,
    wal_status,
    safe_wal_size,
    inactive_since,
    invalidation_reason
FROM pg_replication_slots;
```

PostgreSQL 19 exposes additional slot status information including `wal_status`, `safe_wal_size`, and invalidation information. ([PostgreSQL][4])

This should be part of your Phase 6 monitoring script.

---

# 14. Prepare the standby

Stop PostgreSQL:

```bash
sudo systemctl stop postgresql-19
```

Verify:

```bash
systemctl is-active postgresql-19
```

Expected:

```text
inactive
```

---

# 15. Protect the existing standby data directory

Find it:

```bash
sudo -u postgres psql -Atc \
"SHOW data_directory;"
```

If this is a fresh lab standby, the data directory can be emptied/reinitialized.

**Do not delete an existing database cluster until you have verified that it is the correct standby data directory.**

For a disposable lab:

```bash
sudo mv /var/lib/pgsql/19/data \
/var/lib/pgsql/19/data.before-replication
```

Then:

```bash
sudo mkdir -p /var/lib/pgsql/19/data

sudo chown postgres:postgres \
/var/lib/pgsql/19/data

sudo chmod 700 \
/var/lib/pgsql/19/data
```

---

# 16. Take the base backup

From the standby:

```bash
sudo -u postgres \
/usr/pgsql-19/bin/pg_basebackup \
-h <PRIMARY_IP> \
-D /var/lib/pgsql/19/data \
-U repl_user \
-P \
-R \
-X stream \
-S pg19_standby_slot
```

Important options:

```text
-D
  standby data directory

-R
  automatically creates standby connection configuration

-X stream
  stream required WAL during backup

-S
  use physical replication slot

-P
  progress display
```

---

# 17. What `pg_basebackup -R` does

The `-R` option prepares the backup to operate as a standby.

This is preferable to manually constructing every standby configuration file because it reduces configuration mistakes.

After completion:

```bash
sudo ls -l /var/lib/pgsql/19/data/
```

Look for:

```text
standby.signal
```

and the generated standby connection configuration.

---

# 18. Start standby

```bash
sudo systemctl start postgresql-19
```

Verify:

```bash
systemctl status postgresql-19 --no-pager
```

Then:

```bash
sudo -u postgres psql \
-c "SELECT pg_is_in_recovery();"
```

Expected:

```text
t
```

That confirms the server is operating as a standby.

---

# 19. Verify replication from primary

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
state      = streaming
sync_state = async
```

assuming asynchronous replication.

---

# 20. Verify standby

On standby:

```sql
SELECT
    pg_is_in_recovery();
```

Then:

```sql
SELECT
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn(),
    pg_last_xact_replay_timestamp();
```

You should see WAL receive/replay activity.

---

# 21. Create test data

Primary:

```sql
CREATE DATABASE ha_lab;
```

Connect:

```bash
sudo -u postgres psql -d ha_lab
```

Create:

```sql
CREATE TABLE ha_test
(
    id          bigint GENERATED ALWAYS AS IDENTITY,
    server_name text,
    created_at  timestamptz DEFAULT clock_timestamp()
);
```

Insert:

```sql
INSERT INTO ha_test(server_name)
VALUES
('PRIMARY-TEST-01'),
('PRIMARY-TEST-02'),
('PRIMARY-TEST-03');
```

---

# 22. Check the standby

On standby:

```bash
sudo -u postgres psql -d ha_lab
```

Run:

```sql
SELECT *
FROM ha_test
ORDER BY id;
```

You should see the rows replicated.

Then:

```sql
INSERT INTO ha_test(server_name)
VALUES ('SHOULD_FAIL');
```

Expected:

```text
ERROR:
cannot execute INSERT in a read-only transaction
```

That's correct.

---

# 23. Verify read-only behavior

```sql
SHOW transaction_read_only;
```

On the standby it should normally report:

```text
on
```

Check:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
t
```

---

# 24. Measure replication lag

On primary:

```sql
SELECT
    application_name,
    client_addr,
    state,
    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_current_wal_lsn(),
            replay_lsn
        )
    ) AS replay_lag_bytes
FROM pg_stat_replication;
```

This is generally more useful than relying on a timestamp alone.

---

# 25. WAL positions

Primary:

```sql
SELECT pg_current_wal_lsn();
```

Standby:

```sql
SELECT
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();
```

Conceptually:

```text
PRIMARY
current WAL
     |
     v
WAL sender
     |
     v
network
     |
     v
WAL receiver
     |
     v
received WAL
     |
     v
replay
     |
     v
standby database
```

---

# 26. Synchronous vs asynchronous replication

Your current configuration:

```text
Primary
   |
   | WAL
   v
Standby
```

is asynchronous unless you configure synchronous replication.

### Asynchronous

```text
COMMIT
  |
  +--> Primary confirms
  |
  +--> Standby receives later
```

Advantages:

* lower commit latency
* less sensitivity to network latency
* geographically flexible

Risk:

* recent transactions can potentially be missing from the standby during an abrupt primary failure.

---

# 27. Synchronous replication

PostgreSQL supports synchronous standby selection through:

```sql
SHOW synchronous_standby_names;
```

For example:

```sql
ALTER SYSTEM SET synchronous_standby_names =
'FIRST 1 (pg19_standby)';
```

The standby must use:

```text
application_name=pg19_standby
```

in `primary_conninfo`.

PostgreSQL 19 supports both priority-based `FIRST` and quorum-based `ANY` synchronous standby configurations. ([PostgreSQL][2])

---

# 28. `synchronous_commit`

Check:

```sql
SHOW synchronous_commit;
```

Possible values include:

```text
on
remote_apply
remote_write
local
off
```

For the strongest standby-apply confirmation:

```sql
ALTER SYSTEM SET synchronous_commit = 'remote_apply';
```

But do not enable this without understanding the latency implications.

With `remote_apply`, a transaction commit waits until the synchronous standby has applied the corresponding WAL. PostgreSQL documents that this can make every commit wait for standby replay. ([PostgreSQL][2])

---

# 29. HA design comparison

| Mode              | Commit waits for standby? | Typical RPO |            Commit latency |
| ----------------- | ------------------------: | ----------: | ------------------------: |
| Async             |                        No | >0 possible |                     Lower |
| Sync remote write |                       Yes |    Very low |                    Higher |
| Sync remote flush |                       Yes |    Very low |                    Higher |
| Sync remote apply |                       Yes |    Very low | Highest potential latency |

The actual RPO depends on the failure scenario and architecture; don't equate a configuration label with a guaranteed business RPO.

---

# 30. Hot standby

On standby:

```sql
SHOW hot_standby;
```

Normally:

```text
on
```

This permits read-only queries while the standby is recovering.

However, long-running standby queries can conflict with WAL replay.

Relevant parameters include:

```sql
SHOW max_standby_streaming_delay;

SHOW max_standby_archive_delay;

SHOW hot_standby_feedback;
```

PostgreSQL documents that standby queries can conflict with recovery and may be canceled when replay needs to apply conflicting WAL. ([PostgreSQL][2])

---

# 31. `hot_standby_feedback`

Check:

```sql
SHOW hot_standby_feedback;
```

If enabled, the standby can communicate feedback about old snapshots to the primary.

Potential benefit:

```text
fewer standby query cancellations
```

Potential downside:

```text
old xmin held on primary
        ↓
dead rows cannot be removed
        ↓
table/index bloat
```

Therefore:

```text
hot_standby_feedback = performance/availability tradeoff
```

Do not enable it blindly.

---

# 32. Planned switchover

A **switchover** is planned.

Architecture:

```text
PRIMARY
   |
   | controlled shutdown
   v
STANDBY
   |
   v
PROMOTE
   |
   v
NEW PRIMARY
```

Before promotion, stop application writes if necessary and ensure the standby is caught up.

Check:

```sql
SELECT
    application_name,
    state,
    sync_state,
    sent_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

---

# 33. Promote standby

On standby:

```bash
sudo -u postgres \
/usr/pgsql-19/bin/pg_ctl \
-D /var/lib/pgsql/19/data \
promote
```

Or:

```sql
SELECT pg_promote();
```

Then:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
f
```

The standby is now the primary.

---

# 34. Timeline change

After promotion:

```sql
SELECT
    timeline_id,
    redo_lsn,
    checkpoint_lsn
FROM pg_control_checkpoint();
```

Also inspect:

```bash
ls -l /var/lib/pgsql/19/data/pg_wal
```

and timeline history files:

```bash
ls -l /var/lib/pgsql/19/data/*history
```

Conceptually:

```text
Timeline 1
    |
    | failure
    v
Timeline 2
    |
    v
NEW PRIMARY
```

This timeline concept is essential when rebuilding the old primary.

---

# 35. Old primary must NOT simply be started

After promotion:

```text
OLD PRIMARY
      |
      X
Do not simply start it
```

If you start both nodes independently:

```text
OLD PRIMARY
      |
      | writes
      v
   timeline 1

NEW PRIMARY
      |
      | writes
      v
   timeline 2
```

You have **split brain**.

That is a serious HA failure.

---

# 36. Rebuild the old primary

The safest general approach for your lab is to rebuild the old primary as a new standby.

On old primary:

```bash
sudo systemctl stop postgresql-19
```

Then protect its data directory.

After confirming the correct data directory:

```bash
sudo mv /var/lib/pgsql/19/data \
/var/lib/pgsql/19/data.old-primary
```

Create a fresh standby using:

```bash
pg_basebackup
```

against the **new primary**.

Example:

```bash
sudo -u postgres \
/usr/pgsql-19/bin/pg_basebackup \
-h <NEW_PRIMARY_IP> \
-D /var/lib/pgsql/19/data \
-U repl_user \
-P \
-R \
-X stream \
-S pg19_oldprimary_slot
```

Then:

```bash
sudo systemctl start postgresql-19
```

Verify:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
t
```

---

# 37. `pg_rewind`

For some failover scenarios, you can use:

```bash
pg_rewind
```

instead of taking a complete new base backup.

It can synchronize a former primary with the new primary when the servers have diverged timelines and the necessary WAL is available.

Before using it:

```bash
pg_rewind --help
```

and validate your exact topology.

Do not treat `pg_rewind` as a universal replacement for `pg_basebackup`.

---

# 38. PostgreSQL 19 HA improvement — `WAIT FOR`

PostgreSQL 19 introduces a `WAIT FOR` command that can wait for a standby to reach a specified WAL position. ([PostgreSQL][1])

This is useful for:

```text
Application write
       |
       v
Obtain LSN
       |
       v
WAIT FOR standby replay
       |
       v
Read from standby
```

This supports read-your-writes patterns where the application needs to ensure the standby has caught up before reading there.

---

# 39. Replication health query

Put this on your Phase 6 monitoring framework:

```sql
SELECT
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

And:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    wal_status,
    safe_wal_size,
    invalidation_reason
FROM pg_replication_slots;
```

---

# 40. Standby health check

Run on standby:

```sql
SELECT
    pg_is_in_recovery() AS is_standby,
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn() AS replayed_lsn,
    pg_last_xact_replay_timestamp() AS last_replayed_transaction;
```

Then:

```sql
SELECT
    now() - pg_last_xact_replay_timestamp()
        AS replay_time_delay;
```

Treat this timestamp as an indicator rather than the sole definition of replication lag; a quiet primary may have no recent transaction timestamp even when replication is completely healthy.

---

# 41. HA failure scenario

### Scenario

Primary crashes.

```text
PG19-PRIMARY
     X
     |
     X
  FAILURE
```

Standby:

```text
PG19-STANDBY
      |
      v
Check:
pg_is_in_recovery()
```

If the standby is healthy:

```bash
sudo -u postgres \
/usr/pgsql-19/bin/pg_ctl \
-D /var/lib/pgsql/19/data \
promote
```

Then:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
f
```

Application connection must then be redirected to the new primary.

---

# 42. Critical HA distinction

PostgreSQL streaming replication **does not automatically provide an end-to-end automatic failover solution by itself**.

You need an operational mechanism for:

```text
Failure detection
       +
Leader election / fencing
       +
Promotion
       +
Application endpoint change
       +
Old-primary fencing
       +
Rebuild
```

For an enterprise HA implementation, this is where technologies such as **Patroni**, **repmgr**, load balancers/connection pools, or cloud-managed HA services can become relevant.

---

# 43. HA validation test

Run these tests deliberately.

### Test 1 — replication

```text
Primary INSERT
      ↓
Standby SELECT
```

### Test 2 — standby read-only

```text
Standby INSERT
      ↓
Must fail
```

### Test 3 — replication lag

```text
Generate WAL
      ↓
Measure replay
```

### Test 4 — planned switchover

```text
Primary
   ↓
Standby promotion
   ↓
New Primary
```

### Test 5 — rebuild

```text
Old Primary
    ↓
Reinitialize
    ↓
New Standby
```

### Test 6 — failover

```text
Primary failure
       ↓
Standby promotion
       ↓
Application reconnect
```

---

# 44. RPO validation

Create a test transaction:

```sql
INSERT INTO ha_test(server_name)
VALUES ('RPO-TEST');
```

Record the LSN:

```sql
SELECT pg_current_wal_lsn();
```

Check standby:

```sql
SELECT pg_last_wal_replay_lsn();
```

Then compare.

For a synchronous architecture, define exactly which synchronous acknowledgment mode is being used before claiming a particular RPO.

---

# 45. RTO validation

Record:

```text
T1 = primary failure detected
T2 = standby promotion started
T3 = promotion completed
T4 = application reconnect
T5 = application transaction succeeds
```

Calculate:

```text
RTO = T5 - T1
```

This is much more meaningful than simply saying:

```text
"HA is configured."
```

---

# 46. Final PostgreSQL 19 HA checklist

```text
[ ] Primary installed
[ ] Standby installed
[ ] PostgreSQL versions compatible
[ ] wal_level = replica
[ ] max_wal_senders configured
[ ] max_replication_slots configured
[ ] Replication role created
[ ] SCRAM authentication configured
[ ] pg_hba replication rule restricted
[ ] Firewall restricted
[ ] Physical replication slot created
[ ] pg_basebackup completed
[ ] standby.signal exists
[ ] primary_conninfo configured
[ ] primary_slot_name configured
[ ] Standby streaming
[ ] Hot standby verified
[ ] Replication lag monitored
[ ] WAL slot retention monitored
[ ] Async replication tested
[ ] Sync replication understood/tested
[ ] Failover tested
[ ] Promotion tested
[ ] Timeline change validated
[ ] Old primary fenced
[ ] Old primary rebuilt
[ ] RPO measured
[ ] RTO measured
[ ] Application reconnection tested
```

---

## Phase 9 architecture outcome

Your lab is now progressing toward:

```text
                  PostgreSQL 19 DBA LAB

                       APPLICATION
                            |
                            v
                  +-------------------+
                  | Connection Layer  |
                  +-------------------+
                     /             \
                    /               \
                   v                 v
          +---------------+   +---------------+
          | PG19 PRIMARY  |==>| PG19 STANDBY  |
          |               |   |               |
          | READ/WRITE    |   | READ ONLY     |
          +---------------+   +---------------+
                  |                  |
                  | WAL              | Replay
                  v                  v
            +-----------+      +-----------+
            | pgBackRest|      | Read Work |
            |   PITR    |      |  Loads    |
            +-----------+      +-----------+
                  |
                  v
              DR / RPO
```

**Next phase: Phase 10 — PostgreSQL 19 Logical Replication & Migration**, covering **publication/subscription, row filtering, column lists, sequence replication in PostgreSQL 19, replication origins, logical replication slots, failover slots, upgrade/migration use cases, monitoring, conflict handling, and PostgreSQL 17/18 → 19 migration design**. PostgreSQL 19 specifically adds logical replication of sequence values and enhanced failover-slot synchronization, making this a useful next step for the current lab. ([PostgreSQL][5])

[1]: https://www.postgresql.org/docs/19/release-19.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: E.1. Release 19"
[2]: https://www.postgresql.org/docs/19/runtime-config-replication.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 19.6. Replication"
[3]: https://www.postgresql.org/docs/19/logicaldecoding-explanation.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 47.2. Logical Decoding Concepts"
[4]: https://www.postgresql.org/docs/19/view-pg-replication-slots.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 53.22. pg_replication_slots"
[5]: https://www.postgresql.org/docs/release/19.0/?utm_source=chatgpt.com "PostgreSQL: Release Notes"
