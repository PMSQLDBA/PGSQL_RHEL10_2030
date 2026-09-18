## Phase 5 — PostgreSQL 19 PITR Disaster-Recovery Lab

Now we'll perform an actual **backup → data change → accidental data loss → PITR → validation** exercise.

I verified the PostgreSQL 19 documentation first. PostgreSQL 19 supports `recovery_target_time`, `recovery_target_name`, `recovery_target_lsn`, and `recovery_target_xid`; a `recovery.signal` file starts targeted recovery. ([PostgreSQL][1])

### Lab scenario

We'll simulate:

```text
10:00  Full backup
        |
10:10  Important data created
        |
10:15  Restore point created
        |
10:20  More data created
        |
10:25  DBA accidentally drops table
        |
        X  DATA LOSS
        |
        v
Restore FULL backup
        +
Replay WAL
        |
        v
Recover to 10:15 restore point
        |
        v
Validate recovered data
```

For this lab, **do not perform the destructive step on any production database**.

---

# 1. Create the lab database

Connect:

```bash
sudo -u postgres psql
```

Create:

```sql
CREATE DATABASE pitr_lab;
```

Connect:

```sql
\c pitr_lab
```

Create the table:

```sql
CREATE TABLE customer_orders
(
    order_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_name text NOT NULL,
    order_amount numeric(12,2) NOT NULL,
    created_at timestamptz DEFAULT clock_timestamp()
);
```

Insert initial data:

```sql
INSERT INTO customer_orders
(customer_name, order_amount)
VALUES
('Customer-A', 100.00),
('Customer-B', 200.00),
('Customer-C', 300.00);
```

Verify:

```sql
SELECT * FROM customer_orders ORDER BY order_id;
```

---

# 2. Force WAL archiving

```sql
SELECT pg_switch_wal();
```

Then:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
check
```

The `check` must complete successfully before continuing.

---

# 3. Take a FULL backup

```bash
sudo -u postgres pgbackrest \
--stanza=main \
--type=full \
backup
```

Verify:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
info
```

You now have a recovery baseline.

---

# 4. Create a named PostgreSQL restore point

This is an excellent DBA technique for planned recovery markers.

Run:

```bash
sudo -u postgres psql -d pitr_lab
```

Then:

```sql
SELECT pg_create_restore_point('BEFORE_BAD_DEPLOYMENT');
```

PostgreSQL writes a named marker into WAL, and that marker can later be used with `recovery_target_name`. ([PostgreSQL][2])

Record the returned LSN.

For example:

```text
pg_create_restore_point
-----------------------
0/5000120
```

Your value will be different.

---

# 5. Add more business data

```sql
INSERT INTO customer_orders
(customer_name, order_amount)
VALUES
('Customer-D', 400.00),
('Customer-E', 500.00),
('Customer-F', 600.00);
```

Check:

```sql
SELECT
    order_id,
    customer_name,
    order_amount,
    created_at
FROM customer_orders
ORDER BY order_id;
```

You should now have six rows.

---

# 6. Create another WAL switch

```sql
SELECT pg_switch_wal();
```

Exit:

```text
\q
```

Then:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
check
```

---

# 7. Simulate DBA error

Now deliberately destroy the table:

```bash
sudo -u postgres psql -d pitr_lab
```

Run:

```sql
DROP TABLE customer_orders;
```

Verify:

```sql
\dt
```

The table is gone.

This represents:

```text
Bad deployment
      OR
Accidental DROP
      OR
Application mistake
      OR
DBA mistake
```

---

# 8. Record the incident time

Immediately record:

```bash
date --iso-8601=seconds
```

For example:

```text
2026-09-17T20:25:31-05:00
```

In a real incident, the recovery target would normally be chosen **just before the destructive transaction**, based on database/application evidence.

---

# 9. Stop PostgreSQL

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

# 10. Protect the damaged cluster

**Do not delete it.**

Rename it:

```bash
sudo mv \
/var/lib/pgsql/19/data \
/var/lib/pgsql/19/data_failed_$(date +%Y%m%d_%H%M%S)
```

Create a new directory:

```bash
sudo mkdir -p /var/lib/pgsql/19/data
```

Set ownership:

```bash
sudo chown postgres:postgres \
/var/lib/pgsql/19/data

sudo chmod 700 \
/var/lib/pgsql/19/data
```

---

# 11. Restore the full backup

Use pgBackRest:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
--type=immediate \
restore
```

This restores the backup to the configured PostgreSQL data directory.

### Important

The exact pgBackRest restore options should be adapted to your repository and recovery objective. **Do not run a restore against an active data directory.**

---

# 12. Configure PITR target

For PostgreSQL 19, targeted recovery uses `recovery.signal`. PostgreSQL then replays archived WAL until the configured recovery target is reached. ([PostgreSQL][1])

For the named restore point:

```bash
sudo touch /var/lib/pgsql/19/data/recovery.signal
```

Add the recovery configuration to `postgresql.auto.conf`:

```bash
sudo -u postgres vi \
/var/lib/pgsql/19/data/postgresql.auto.conf
```

Add:

```text
restore_command = 'pgbackrest --stanza=main archive-get %f "%p"'

recovery_target_name = 'BEFORE_BAD_DEPLOYMENT'

recovery_target_action = 'pause'
```

The `pause` action is useful during a lab because PostgreSQL can stop at the recovery target so you can inspect the recovered state before allowing recovery to continue. PostgreSQL 19 documents `recovery_target_action` values including `pause`, `promote`, and `shutdown`. ([PostgreSQL][1])

---

# 13. Start PostgreSQL

```bash
sudo systemctl start postgresql-19
```

Watch the log:

```bash
sudo journalctl \
-u postgresql-19 \
-f
```

Or inspect PostgreSQL logs.

You should see recovery activity.

---

# 14. Check recovery status

```bash
sudo -u postgres psql -c "
SELECT
    pg_is_in_recovery();
"
```

Expected:

```text
t
```

Check:

```bash
sudo -u postgres psql -c "
SELECT
    pg_last_wal_replay_lsn(),
    pg_last_xact_replay_timestamp();
"
```

---

# 15. Check the recovered database

Connect:

```bash
sudo -u postgres psql -d pitr_lab
```

Run:

```sql
\dt
```

Then:

```sql
SELECT *
FROM customer_orders
ORDER BY order_id;
```

The important question is:

> **Does `customer_orders` exist again?**

If yes, PITR successfully recovered the database to the selected recovery point.

---

# 16. Validate the expected rows

Run:

```sql
SELECT COUNT(*)
FROM customer_orders;
```

The expected result depends on exactly where your named restore point was created.

This is why **recovery validation must be based on business/application checkpoints**, not simply "PostgreSQL started successfully."

---

# 17. Verify recovery state

Run:

```sql
SELECT
    pg_is_in_recovery();
```

If it returns:

```text
t
```

the server is still in recovery.

Because we configured:

```text
recovery_target_action = 'pause'
```

this is expected.

---

# 18. Resume recovery

If you've validated the recovered state:

```sql
SELECT pg_wal_replay_resume();
```

Then:

```sql
SELECT pg_is_in_recovery();
```

Eventually:

```text
f
```

The database has exited recovery.

---

# 19. Verify timeline

Run:

```sql
SELECT
    timeline_id,
    redo_lsn,
    redo_wal_start_lsn
FROM pg_control_checkpoint();
```

Also:

```bash
sudo -u postgres pg_controldata \
/var/lib/pgsql/19/data | grep -E \
"Latest checkpoint's TimeLineID|Database system identifier"
```

PITR can create a new timeline because you're recovering to an earlier point and then continuing from there. PostgreSQL's recovery documentation explicitly describes timelines as separate histories created through recovery/PITR. ([PostgreSQL][1])

---

# 20. Critical DBA lesson — PITR is not the same as `pg_dump`

This distinction is important:

| Capability                     |     `pg_dump` |   pgBackRest + WAL |
| ------------------------------ | ------------: | -----------------: |
| Logical backup                 |             ✅ |                  ❌ |
| Physical backup                |             ❌ |                  ✅ |
| WAL archiving                  |             ❌ |                  ✅ |
| PITR                           |             ❌ |                  ✅ |
| Full cluster recovery          |             ❌ |                  ✅ |
| Individual object restore      |      Possible | Different workflow |
| Large DB recovery              | Less suitable |                  ✅ |
| Continuous recovery capability |             ❌ |                  ✅ |

For a multi-terabyte PostgreSQL environment, **physical backup + WAL archiving + PITR** is the core DR mechanism.

---

# 21. Automate the PITR test

Once your manual test succeeds, create:

```text
pitr_test.sh
```

The automation should perform:

```text
1. Validate PostgreSQL
2. Validate pgBackRest
3. Validate latest FULL backup
4. Validate WAL archive
5. Record recovery target
6. Stop PostgreSQL
7. Restore backup
8. Create recovery.signal
9. Configure recovery target
10. Start PostgreSQL
11. Wait for recovery
12. Validate database
13. Validate expected objects
14. Validate row counts
15. Validate pg_is_in_recovery()
16. Resume/promote
17. Generate PASS/FAIL report
```

That gives you a repeatable **DR test**, rather than a one-time restore demonstration.

---

## Production DBA recovery model

Your eventual architecture should be:

```text
                     APPLICATION
                          |
                          v
                  PostgreSQL 19
                          |
             +------------+------------+
             |                         |
            DATA                      WAL
             |                         |
             |                    pgBackRest
             |                         |
             +-------------+-----------+
                           |
                  Backup Repository
                           |
             +-------------+-------------+
             |                           |
        Local Backup              Off-site/Object
        Repository                  Storage
             |                           |
             +-------------+-------------+
                           |
                       DR Server
                           |
                           v
                     PITR Recovery
```

### Recovery objectives

You should define these before calling the DR solution complete:

**RPO — Recovery Point Objective**

> How much data can the business afford to lose?

With continuous WAL archiving, the theoretical recovery point can be very close to the failure point, subject to successful WAL archival and infrastructure constraints.

**RTO — Recovery Time Objective**

> How long can the business tolerate the database being unavailable?

RTO depends on database size, backup type, storage throughput, WAL volume, network bandwidth and restore architecture.

### The most important DBA rule

**Never declare "backup successful" until you have successfully restored it and validated the application data.**

PostgreSQL's official documentation explicitly treats continuous archiving and PITR as the mechanism for recovering to an earlier point in time, and PostgreSQL 19 supports named restore points and time/LSN/XID recovery targets. ([PostgreSQL][1])

**Next Phase 6:** build the **PostgreSQL 19 monitoring and health-check framework** — `pg_stat_activity`, blocking/locking, long-running transactions, replication/WAL health, database and table growth, autovacuum/bloat, checkpoints, cache hit ratio, connection utilization, OS checks, and an automated **HTML DBA health report**.

[1]: https://www.postgresql.org/docs/19/runtime-config-wal.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 19.5. Write Ahead Log"
[2]: https://www.postgresql.org/docs/19/functions-admin.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 9.29. System Administration Functions"
