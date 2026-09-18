## Phase 3 — Remote DBA Connectivity + Security Hardening

For the RHEL 10 + PostgreSQL 19 lab, this phase should establish a **controlled Windows/pgAdmin → PostgreSQL connection** rather than exposing PostgreSQL to the entire network.

PostgreSQL 19 supports SCRAM-SHA-256 authentication, and its TLS implementation can encrypt client/server traffic. RHEL 10 `firewalld` supports source-restricted rules, which is what we should use here. ([PostgreSQL][1])

### Target architecture

```text
Windows DBA workstation
        |
        | TCP 5432
        |
        v
+---------------------------+
| RHEL 10                   |
|                           |
| firewalld                 |
|    ↓ only approved IP     |
| PostgreSQL 19             |
|    ↓                      |
| pg_hba.conf               |
|    ↓                      |
| SCRAM-SHA-256             |
|    ↓                      |
| PostgreSQL database       |
+---------------------------+
```

### 1. First identify the two IP addresses

On RHEL:

```bash
ip -br addr
```

Example:

```text
RHEL 10       192.168.0.145
Windows       192.168.0.73
```

**Do not use those example addresses unless they are actually your lab addresses.**

---

# 2. Create a dedicated DBA role

Do **not** use the `postgres` superuser for normal DBA connectivity.

Connect locally:

```bash
sudo -u postgres psql
```

Create a DBA login:

```sql
CREATE ROLE dba_admin
WITH
    LOGIN
    CREATEDB
    CREATEROLE
    PASSWORD 'CHANGE_THIS_STRONG_PASSWORD';
```

Check:

```sql
\du dba_admin
```

For your lab this gives you a separate administrative login while retaining `postgres` for emergency/local administrative work.

---

# 3. Create an application login

This demonstrates the important separation between DBA and application accounts:

```sql
CREATE ROLE app_user
WITH
    LOGIN
    PASSWORD 'CHANGE_THIS_STRONG_PASSWORD';
```

Do **not** give it `SUPERUSER`.

Verify:

```sql
SELECT
    rolname,
    rolsuper,
    rolcreatedb,
    rolcreaterole,
    rolcanlogin
FROM pg_roles
WHERE rolname IN ('postgres','dba_admin','app_user');
```

Expected conceptually:

```text
postgres    true    ...
dba_admin   false   true    true    true
app_user    false   false   false   true
```

---

# 4. Configure SCRAM

PostgreSQL 19 uses `scram-sha-256` as the default `password_encryption` setting. ([PostgreSQL][1])

Check:

```bash
sudo -u postgres psql -c "SHOW password_encryption;"
```

You want:

```text
scram-sha-256
```

If necessary:

```bash
sudo -u postgres psql -c \
"ALTER SYSTEM SET password_encryption = 'scram-sha-256';"
```

Restart:

```bash
sudo systemctl restart postgresql-19
```

---

# 5. Configure `pg_hba.conf`

This is where we restrict **who can authenticate**.

Find the file:

```bash
sudo -u postgres psql -c "SHOW hba_file;"
```

Edit it:

```bash
sudo vi /var/lib/pgsql/19/data/pg_hba.conf
```

For a Windows DBA workstation at `192.168.0.73`, use:

```text
# Local Unix socket
local   all             all                             peer

# Localhost TCP
host    all             all             127.0.0.1/32    scram-sha-256
host    all             all             ::1/128         scram-sha-256

# DBA workstation
host    all             dba_admin       192.168.0.73/32 scram-sha-256
```

Notice that we are **not** doing this:

```text
host all all 0.0.0.0/0 scram-sha-256
```

That would permit authentication attempts from every IPv4 source that can reach the server.

PostgreSQL's HBA rules are evaluated according to the entries in `pg_hba.conf`, so the ordering of rules matters. ([PostgreSQL][2])

---

# 6. Configure `listen_addresses`

Check:

```bash
sudo -u postgres psql -c "SHOW listen_addresses;"
```

For a lab:

```text
*
```

is convenient.

However, for a hardened server, I'd prefer the actual server address:

```text
listen_addresses = '192.168.0.145'
```

This limits PostgreSQL to that interface.

PostgreSQL documents `listen_addresses` as controlling which interfaces accept TCP/IP connection attempts. ([PostgreSQL][1])

After changing it:

```bash
sudo systemctl restart postgresql-19
```

---

# 7. Restrict RHEL 10 firewalld

First inspect the current configuration:

```bash
sudo firewall-cmd --get-active-zones
```

Then:

```bash
sudo firewall-cmd --list-all
```

For example, if your active zone is `public`, don't simply expose:

```bash
sudo firewall-cmd --permanent --add-port=5432/tcp
```

Instead, use a source-restricted rule.

For Windows workstation:

```bash
sudo firewall-cmd \
--permanent \
--zone=public \
--add-rich-rule='rule family="ipv4" source address="192.168.0.73/32" port port="5432" protocol="tcp" accept'
```

Reload:

```bash
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --zone=public --list-rich-rules
```

RHEL 10 documents source-based and rich-rule firewall controls and recommends making persistent rules with `--permanent` followed by a reload. ([Red Hat Documentation][3])

---

# 8. Verify PostgreSQL is listening

On RHEL:

```bash
sudo ss -lntp | grep 5432
```

You should see something similar to:

```text
LISTEN 0 244 192.168.0.145:5432 ...
```

Then from Windows PowerShell:

```powershell
Test-NetConnection 192.168.0.145 -Port 5432
```

You want:

```text
TcpTestSucceeded : True
```

---

# 9. Test with `psql` from Windows

From a Windows machine with PostgreSQL client installed:

```powershell
psql -h 192.168.0.145 -p 5432 -U dba_admin -d postgres
```

Enter the password.

Then:

```sql
SELECT
    current_user,
    current_database(),
    inet_client_addr(),
    inet_server_addr(),
    inet_server_port();
```

This confirms the complete path:

```text
Windows
   ↓
TCP 5432
   ↓
RHEL firewalld
   ↓
PostgreSQL listener
   ↓
pg_hba.conf
   ↓
SCRAM
   ↓
dba_admin
```

---

# 10. Configure TLS — next security layer

PostgreSQL has native TLS support. PostgreSQL 19 documents enabling it with:

```text
ssl = on
```

and using `server.crt` / `server.key` or explicitly configured certificate paths. ([PostgreSQL][4])

For your **lab**, we can create a self-signed certificate.

For production, I would use a certificate issued by your organization's internal CA or an appropriate trusted PKI.

### Generate lab certificate

On RHEL:

```bash
cd /var/lib/pgsql/19/data

sudo -u postgres openssl req \
-new \
-x509 \
-days 365 \
-nodes \
-newkey rsa:4096 \
-keyout server.key \
-out server.crt \
-subj "/CN=$(hostname -f)"
```

Set permissions:

```bash
sudo chown postgres:postgres \
/var/lib/pgsql/19/data/server.key \
/var/lib/pgsql/19/data/server.crt

sudo chmod 600 /var/lib/pgsql/19/data/server.key
sudo chmod 644 /var/lib/pgsql/19/data/server.crt
```

PostgreSQL specifically requires restrictive permissions on the server private key; `0600` is the standard Unix setup described in the PostgreSQL documentation. ([PostgreSQL][4])

Enable TLS:

```bash
sudo -u postgres psql -c \
"ALTER SYSTEM SET ssl = 'on';"
```

Restart:

```bash
sudo systemctl restart postgresql-19
```

Verify:

```bash
sudo -u postgres psql -c "SHOW ssl;"
```

Expected:

```text
on
```

---

# 11. Require TLS for remote DBA connections

Change the DBA workstation rule in `pg_hba.conf` from:

```text
host    all    dba_admin    192.168.0.73/32    scram-sha-256
```

to:

```text
hostssl    all    dba_admin    192.168.0.73/32    scram-sha-256
```

Reload:

```bash
sudo systemctl reload postgresql-19
```

Now:

```text
host
```

means TCP/IP authentication, while:

```text
hostssl
```

requires an SSL/TLS connection. PostgreSQL 19 documents this mechanism for requiring encrypted connections for selected HBA rules. ([PostgreSQL][4])

---

# 12. Verify TLS

From PostgreSQL:

```bash
sudo -u postgres psql -c "
SELECT
    datname,
    usename,
    client_addr,
    ssl,
    version
FROM pg_stat_ssl
JOIN pg_stat_activity
ON pg_stat_ssl.pid = pg_stat_activity.pid;
"
```

For your remote DBA session you should see:

```text
ssl = t
```

You can also connect from `psql` with:

```powershell
psql "host=192.168.0.145 port=5432 dbname=postgres user=dba_admin sslmode=require"
```

---

## Final security model

Your lab should now look like this:

```text
                 Windows DBA
                 192.168.0.73
                       |
                       | TCP/5432
                       | TLS
                       v
              +------------------+
              | RHEL 10          |
              |                  |
              | firewalld         |
              | Source allowlist  |
              |       ↓           |
              | PostgreSQL 19     |
              |       ↓           |
              | hostssl           |
              |       ↓           |
              | SCRAM-SHA-256     |
              |       ↓           |
              | dba_admin         |
              +------------------+
```

### Phase 3 validation checklist

| Test                  | Command                             | Expected             |
| --------------------- | ----------------------------------- | -------------------- |
| PostgreSQL service    | `systemctl is-active postgresql-19` | `active`             |
| Listener              | `ss -lntp \| grep 5432`             | Listening            |
| Firewall              | `firewall-cmd --list-rich-rules`    | DBA IP allowed       |
| Windows connectivity  | `Test-NetConnection ... -Port 5432` | `True`               |
| Authentication        | `psql ...`                          | Successful           |
| Authentication method | `pg_hba.conf`                       | `scram-sha-256`      |
| TLS                   | `SHOW ssl;`                         | `on`                 |
| Remote TLS            | `pg_stat_ssl`                       | `ssl = t`            |
| DBA account           | `\du dba_admin`                     | Login/DBA privileges |
| Application account   | `\du app_user`                      | Non-superuser        |

**Next Phase 4 should be storage + backup architecture**: separate PostgreSQL data/WAL/backup locations, `pgBackRest`, WAL archiving, retention, restore testing, PITR, backup encryption, and an automated backup-health report. That is the point where this becomes a proper DBA lab rather than just a PostgreSQL installation.

[1]: https://www.postgresql.org/docs/19/runtime-config-connection.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 19.3. Connections and Authentication"
[2]: https://www.postgresql.org/docs/10/auth-pg-hba-conf.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 10: 20.1. The pg_hba.conf File"
[3]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/securing_networks/securing-network-services?utm_source=chatgpt.com "Chapter 8. Securing network services | Securing networks | Red Hat Enterprise Linux | 10 | Red Hat Documentation"
[4]: https://www.postgresql.org/docs/19/ssl-tcp.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 18.9. Secure TCP/IP Connections with SSL"
