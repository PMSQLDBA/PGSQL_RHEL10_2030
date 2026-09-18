## Phase 4 — PostgreSQL 19 Backup, WAL Archiving & PITR

For your RHEL 10 lab, I recommend **pgBackRest 2.59.1** rather than building a backup solution around shell scripts and `pg_dump`. I verified that PGDG currently provides `pgBackRest 2.59.1` packages for RHEL 10, and pgBackRest 2.59 specifically added PostgreSQL 19 support. ([PostgreSQL FTP][1])

The design should be:

```text
                    PostgreSQL 19
                         |
                +--------+--------+
                |                 |
             DATA              WAL
                |                 |
                +--------+--------+
                         |
                         v
                    pgBackRest
                         |
              +----------+----------+
              |                     |
          Full Backup           WAL Archive
              |                     |
              +----------+----------+
                         |
                         v
                  Backup Repository
```

### 1. Install pgBackRest

Because you're already using PGDG:

```bash
sudo dnf install -y pgbackrest
```

Verify:

```bash
pgbackrest version
```

You should currently get the 2.59.x release available for your RHEL 10 package stream. ([PostgreSQL FTP][1])

---

# 2. Create the backup repository

For a **single-server lab**, start with a separate filesystem/directory:

```bash
sudo mkdir -p /backup/pgbackrest
```

Set ownership:

```bash
sudo chown -R postgres:postgres /backup/pgbackrest
sudo chmod 750 /backup/pgbackrest
```

For production, I would **not** keep the only backup repository on the same physical storage as the database. Use separate storage, another backup server, or object storage such as S3/Azure Blob/GCS.

pgBackRest supports multiple repositories and S3/Azure/GCS/SFTP targets. ([PostgreSQL][2])

---

# 3. Create pgBackRest configuration

Create:

```bash
sudo vi /etc/pgbackrest.conf
```

Use:

```ini
[global]
repo1-path=/backup/pgbackrest
repo1-retention-full=2
repo1-retention-diff=4

repo1-bundle=y
repo1-compress-type=zst
repo1-compress-level=3

start-fast=y

process-max=4

log-level-console=info
log-level-file=detail

log-path=/var/log/pgbackrest

[main]
pg1-path=/var/lib/pgsql/19/data
```

Create the log directory:

```bash
sudo mkdir -p /var/log/pgbackrest
sudo chown postgres:postgres /var/log/pgbackrest
sudo chmod 750 /var/log/pgbackrest
```

Set configuration permissions:

```bash
sudo chmod 640 /etc/pgbackrest.conf
sudo chown root:postgres /etc/pgbackrest.conf
```

---

# 4. Configure PostgreSQL WAL archiving

This is the critical part.

Check current settings:

```bash
sudo -u postgres psql -c "SHOW archive_mode;"
sudo -u postgres psql -c "SHOW archive_command;"
```

Configure:

```bash
sudo -u postgres psql -c \
"ALTER SYSTEM SET archive_mode = 'on';"

sudo -u postgres psql -c \
"ALTER SYSTEM SET archive_command = 'pgbackrest --stanza=main archive-push %p';"
```

Because `archive_mode` requires a restart, restart PostgreSQL:

```bash
sudo systemctl restart postgresql-19
```

Verify:

```bash
sudo -u postgres psql -c "SHOW archive_mode;"
```

Expected:

```text
on
```

And:

```bash
sudo -u postgres psql -c "SHOW archive_command;"
```

Expected:

```text
pgbackrest --stanza=main archive-push %p
```

PostgreSQL's WAL architecture is what makes continuous archiving and point-in-time recovery possible. PostgreSQL 19 also provides native backup-control functions such as `pg_backup_start()` and `pg_backup_stop()`, although pgBackRest manages the backup workflow for you. ([PostgreSQL][3])

---

# 5. Create the pgBackRest stanza

Run:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
stanza-create
```

Expected:

```text
P00   INFO: stanza-create command begin
...
P00   INFO: stanza-create command end: completed successfully
```

Then:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
check
```

This is an important DBA validation because it tests the configured repository and PostgreSQL/WAL archiving path.

---

# 6. Take your first FULL backup

Run:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
--type=full \
backup
```

Monitor:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
info
```

You should see something conceptually like:

```text
stanza: main
status: ok

full backup:
    timestamp
    database size
    backup size
    repository size
```

---

# 7. Enable page checksums

For a new lab cluster, I recommend enabling PostgreSQL page checksums because they provide an additional mechanism for detecting data-page corruption.

Check:

```bash
sudo -u postgres psql -c "
SELECT
    current_setting('data_checksums') AS data_checksums;
"
```

If your cluster was initialized without checksums, don't simply change a configuration parameter—page checksums are a cluster-level property.

For a **fresh lab**, the cleanest approach is to initialize the cluster with:

```bash
sudo -u postgres /usr/pgsql-19/bin/initdb \
--data-checksums \
-D /var/lib/pgsql/19/data
```

But **do not execute this against your existing cluster**, because it would recreate the database directory.

For your current lab, first determine:

```bash
sudo -u postgres psql -c "
SELECT pg_catalog.pg_is_in_recovery(),
       current_setting('data_checksums');
"
```

---

# 8. Test WAL archiving

Generate database activity:

```bash
sudo -u postgres psql <<'EOF'
CREATE DATABASE backup_lab;
EOF
```

Connect:

```bash
sudo -u postgres psql -d backup_lab
```

Create test data:

```sql
CREATE TABLE backup_test
(
    id bigint GENERATED ALWAYS AS IDENTITY,
    message text,
    created_at timestamptz DEFAULT now()
);

INSERT INTO backup_test(message)
SELECT 'PostgreSQL backup test'
FROM generate_series(1,10000);
```

Exit:

```text
\q
```

Force a WAL switch:

```bash
sudo -u postgres psql -c "SELECT pg_switch_wal();"
```

Then:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
check
```

If the archive is functioning correctly, `check` should complete successfully.

---

# 9. Take a differential backup

After your full backup:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
--type=diff \
backup
```

Then:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
info
```

You should now have:

```text
FULL
  +
DIFFERENTIAL
```

pgBackRest supports full, differential and incremental backups, backup retention, WAL archiving, encryption and parallel processing. ([PostgreSQL][2])

---

# 10. Take an incremental backup

```bash
sudo -u postgres pgbackrest \
--stanza=main \
--type=incr \
backup
```

Check:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
info
```

Your backup chain becomes:

```text
FULL
  |
  +---- DIFFERENTIAL
  |
  +---- INCREMENTAL
```

---

# 11. Verify the backup

Run:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
verify
```

The current pgBackRest release specifically includes improvements around backup verification and PostgreSQL 19 support. ([PostgreSQL][2])

---

# 12. Backup encryption

For a production-style lab, I recommend repository encryption.

Generate a key:

```bash
sudo openssl rand -base64 48
```

Store the resulting secret securely.

Then configure pgBackRest encryption using its repository encryption settings rather than putting a plaintext key into shell scripts.

**Do not store the encryption key in GitHub, your installation script, or `/etc/pgbackrest.conf` with world-readable permissions.**

---

# 13. The most important test: RESTORE

A backup that has never been restored is **not a validated recovery strategy**.

For the lab, create a separate restore directory:

```bash
sudo mkdir -p /restore/pgsql19
sudo chown postgres:postgres /restore/pgsql19
sudo chmod 700 /restore/pgsql19
```

List available backups:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
info
```

Then restore into the test location.

For example:

```bash
sudo -u postgres pgbackrest \
--stanza=main \
--delta \
restore
```

**Do not run this against your active production data directory.**

For a real restore exercise, we'll stop PostgreSQL and use a dedicated recovery VM/data directory.

---

# 14. PITR architecture

The final recovery architecture should be:

```text
                  PostgreSQL
                       |
                       |
                 WAL generation
                       |
                       v
              pgBackRest archive
                       |
                       v
              Backup Repository
                       |
           +-----------+-----------+
           |                       |
        FULL                    WAL
        Backup                 Archive
           |                       |
           +-----------+-----------+
                       |
                       v
                 PITR RESTORE
                       |
                       v
              Desired timestamp
```

For example:

```text
10:00  Full backup
10:15  Application changes
10:30  Application changes
10:45  Bad deployment
10:46  Data corruption discovered

              ↓

Restore FULL backup
              +
Replay WAL
              ↓
Recover to 10:44:59
```

That is the core PostgreSQL **Point-in-Time Recovery** model.

---

# 15. DBA backup validation script

Create:

```bash
sudo vi /usr/local/bin/postgresql-backup-health.sh
```

Use:

```bash
#!/bin/bash

set -u

PGSERVICE="postgresql-19"
STANZA="main"

echo "============================================================"
echo " PostgreSQL Backup Health Check"
echo "============================================================"

echo
echo "DATE:"
date

echo
echo "POSTGRESQL SERVICE:"
systemctl is-active "$PGSERVICE"

echo
echo "POSTGRESQL VERSION:"
sudo -u postgres psql -tAc "SELECT version();"

echo
echo "ARCHIVE MODE:"
sudo -u postgres psql -tAc "SHOW archive_mode;"

echo
echo "ARCHIVE COMMAND:"
sudo -u postgres psql -tAc "SHOW archive_command;"

echo
echo "PGDATA:"
sudo -u postgres psql -tAc "SHOW data_directory;"

echo
echo "PGPORT:"
sudo -u postgres psql -tAc "SHOW port;"

echo
echo "BACKUP STATUS:"
sudo -u postgres pgbackrest \
    --stanza="$STANZA" \
    check

echo
echo "BACKUP INVENTORY:"
sudo -u postgres pgbackrest \
    --stanza="$STANZA" \
    info

echo
echo "============================================================"
echo " Backup health check completed"
echo "============================================================"
```

Make executable:

```bash
sudo chmod 750 /usr/local/bin/postgresql-backup-health.sh
```

Run:

```bash
sudo /usr/local/bin/postgresql-backup-health.sh
```

---

## Recommended DBA backup policy

For your lab, start with:

| Component              | Lab setting                    |
| ---------------------- | ------------------------------ |
| Backup tool            | **pgBackRest 2.59.x**          |
| Full backup            | Weekly                         |
| Differential           | Daily                          |
| Incremental            | Optional / workload-dependent  |
| WAL archiving          | Continuous                     |
| Full retention         | 2                              |
| Differential retention | 4                              |
| Compression            | Zstandard                      |
| Encryption             | Recommended                    |
| Backup verification    | Regular                        |
| Restore test           | Mandatory                      |
| PITR                   | Mandatory exercise             |
| Repository             | Separate filesystem            |
| Production repository  | Separate server/object storage |

pgBackRest 2.59 supports PostgreSQL 19 and provides the capabilities we need here, including full/differential/incremental backups, WAL archiving, encryption, multiple repositories and cloud/object-storage targets. ([PostgreSQL][2])

### One architectural change I recommend for the next stage

Don't stop at a local `/backup/pgbackrest` repository. For the **DBA/enterprise version**, we should build:

```text
RHEL 10
PostgreSQL 19
      |
      | WAL
      v
pgBackRest
      |
      +----------> Local backup repository
      |
      +----------> S3-compatible/object storage
                         |
                         v
                    Off-site backup
```

That gives you the foundation for **RPO/RTO, ransomware resilience, PITR, backup retention, restore testing and DR** rather than merely having a local backup.

**Phase 5:** I recommend we next build the **complete automated PostgreSQL 19 backup + PITR lab**, including a deliberate data-loss scenario, exact restore commands, `recovery_target_time`, validation queries, and an automated backup/restore success report.

[1]: https://ftp.postgresql.org/pub/repos/yum/common/redhat/rhel-10-x86_64/?utm_source=chatgpt.com "Index of /pub/repos/yum/common/redhat/rhel-10-x86_64/"
[2]: https://www.postgresql.org/about/news/pgbackrest-2590-released-3355/?utm_source=chatgpt.com "PostgreSQL: pgBackRest 2.59.0 Released"
[3]: https://www.postgresql.org/docs/19/functions-admin.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 9.29. System Administration Functions"
