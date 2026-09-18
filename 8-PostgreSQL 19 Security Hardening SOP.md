# Phase 8 — PostgreSQL 19 Security Hardening SOP

This phase secures your **RHEL 10 + PostgreSQL 19 lab** at the database, network, authentication, TLS, privilege, and operating-system layers.

I checked the current PostgreSQL 19 documentation before laying out these steps. PostgreSQL 19 defaults `password_encryption` to `scram-sha-256`; MD5 password support is deprecated. PostgreSQL also supports `hostssl` rules for enforcing TLS and `verify-full` for client-side certificate/hostname verification. ([PostgreSQL][1])

> **Important:** PostgreSQL 19 is still a pre-release/testing version. Perform this security lab on your RHEL 10 test VM, not production.

---

## 1. Security architecture

```text
                    CLIENT
                      |
                      | TCP/5432
                      v
              +---------------+
              |   Firewalld   |
              +---------------+
                      |
                      v
              +---------------+
              | PostgreSQL 19 |
              +---------------+
                 |     |     |
          +------+     |     +------+
          |            |            |
          v            v            v
       TLS/SCRAM    pg_hba.conf   Roles
          |                         |
          |                         v
          |                    Privileges
          |                         |
          +------------+------------+
                       |
                       v
                     Data
                       |
             +---------+---------+
             |                   |
          SELinux              Audit
             |                   |
             +---------+---------+
                       |
                       v
                  DBA Monitoring
```

---

# 2. Security objectives

Your final configuration should achieve:

| Layer                | Target                                      |
| -------------------- | ------------------------------------------- |
| OS                   | RHEL SELinux enforcing                      |
| Network              | Port 5432 source restricted                 |
| PostgreSQL listener  | Specific interface/IP                       |
| Authentication       | SCRAM-SHA-256                               |
| Remote access        | `hostssl`                                   |
| TLS                  | TLS 1.2+                                    |
| PostgreSQL superuser | Restricted                                  |
| Application users    | No SUPERUSER                                |
| Database privileges  | Least privilege                             |
| Schema privileges    | Explicit                                    |
| `PUBLIC` privileges  | Reviewed                                    |
| Passwords            | SCRAM                                       |
| Audit                | PostgreSQL logging / pgaudit where required |
| Firewall             | Source restricted                           |
| Configuration        | Backed up and validated                     |

---

# 3. Check current security state

Run on RHEL:

```bash
cat /etc/redhat-release

getenforce

sudo firewall-cmd --state

sudo firewall-cmd --list-all

sudo ss -lntp | grep 5432
```

Then PostgreSQL:

```bash
sudo -u postgres psql
```

Run:

```sql
SELECT version();

SHOW listen_addresses;

SHOW port;

SHOW password_encryption;

SHOW ssl;

SHOW ssl_min_protocol_version;

SHOW hba_file;

SHOW config_file;
```

---

# 4. SELinux

Check:

```bash
getenforce
```

Target:

```text
Enforcing
```

If:

```text
Permissive
```

do **not** immediately disable SELinux.

For a properly configured RHEL environment, SELinux should remain enabled and PostgreSQL should operate within the appropriate SELinux policy.

PostgreSQL also provides `sepgsql`, an SELinux-based mandatory-access-control module, although that is a separate advanced security architecture and isn't required for your basic PostgreSQL hardening lab. ([PostgreSQL][2])

---

# 5. Firewalld

First inspect:

```bash
sudo firewall-cmd --list-all
```

Do **not** use:

```bash
--add-port=5432/tcp
```

as a permanent production rule if every network source does not need PostgreSQL access.

Instead, restrict the source.

For example, if your DBA workstation is:

```text
192.168.0.73
```

use:

```bash
sudo firewall-cmd \
--permanent \
--add-rich-rule='rule family="ipv4" source address="192.168.0.73/32" port port="5432" protocol="tcp" accept'
```

Reload:

```bash
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --list-rich-rules
```

---

# 6. PostgreSQL listener

Check:

```sql
SHOW listen_addresses;
```

For a hardened lab, instead of:

```text
*
```

prefer the server's actual required IP/interface.

For example:

```conf
listen_addresses = '192.168.0.145'
```

**Do not blindly use `192.168.0.145` unless that is actually the IP of your current RHEL VM.**

Verify first:

```bash
ip addr
```

Then:

```sql
SHOW listen_addresses;
```

PostgreSQL must be listening on an address reachable by the authorized client.

---

# 7. SCRAM authentication

Check:

```sql
SHOW password_encryption;
```

Target:

```text
scram-sha-256
```

Set:

```sql
ALTER SYSTEM SET password_encryption = 'scram-sha-256';
```

Reload:

```bash
sudo systemctl reload postgresql-19
```

Verify:

```sql
SHOW password_encryption;
```

PostgreSQL 19 documents `scram-sha-256` as the password-encryption setting and warns that MD5-encrypted passwords are deprecated. ([PostgreSQL][1])

---

# 8. Create a DBA role

Don't use the PostgreSQL `postgres` superuser for normal daily DBA activity.

Create a dedicated administrative role:

```sql
CREATE ROLE dba_admin
LOGIN
CREATEDB
CREATEROLE
PASSWORD 'CHANGE_THIS_PASSWORD';
```

Verify:

```sql
SELECT
    rolname,
    rolsuper,
    rolcreatedb,
    rolcreaterole,
    rolcanlogin
FROM pg_roles
WHERE rolname = 'dba_admin';
```

Expected:

```text
rolsuper     = false
rolcreatedb  = true
rolcreaterole = true
rolcanlogin  = true
```

For a production environment, administrative role design should be more granular than simply granting `CREATEROLE`.

---

# 9. Application role

Create a separate application login:

```sql
CREATE ROLE app_user
LOGIN
PASSWORD 'CHANGE_THIS_PASSWORD';
```

Verify:

```sql
SELECT
    rolname,
    rolsuper,
    rolcreatedb,
    rolcreaterole,
    rolcanlogin
FROM pg_roles
WHERE rolname = 'app_user';
```

Target:

```text
SUPERUSER     false
CREATEDB      false
CREATEROLE    false
LOGIN         true
```

---

# 10. Why role separation matters

Avoid:

```text
Application
     |
     v
postgres SUPERUSER
```

Use:

```text
Application
     |
     v
app_user
     |
     +--> required database
     +--> required schema
     +--> required tables
     +--> required sequences
```

This is the PostgreSQL equivalent of enforcing least privilege.

---

# 11. PostgreSQL roles vs OS users

These are different security identities.

```text
RHEL
  |
  +-- postgres       <-- OS account
  |
  +-- root           <-- OS administrator

PostgreSQL
  |
  +-- postgres       <-- database role
  |
  +-- dba_admin      <-- database role
  |
  +-- app_user       <-- database role
```

PostgreSQL explicitly separates database roles from operating-system users. ([PostgreSQL][3])

---

# 12. Secure `pg_hba.conf`

Find the file:

```sql
SHOW hba_file;
```

On your PGDG installation it will normally be under the PostgreSQL data directory.

Back it up:

```bash
sudo cp \
$(sudo -u postgres psql -Atc "SHOW hba_file") \
/var/backups/pg_hba.conf.$(date +%Y%m%d_%H%M%S)
```

---

# 13. Understand `pg_hba.conf`

Typical syntax:

```text
TYPE      DATABASE    USER       ADDRESS          METHOD
```

Example:

```text
hostssl   all         dba_admin  192.168.0.73/32  scram-sha-256
```

PostgreSQL processes HBA rules in order and uses the first matching record. There is no fallback to a later matching rule after authentication fails. ([PostgreSQL][3])

That makes rule ordering extremely important.

---

# 14. Recommended lab HBA

A simplified hardened configuration could contain:

```conf
# Local PostgreSQL administration
local   all             postgres                                peer

# DBA workstation
hostssl all             dba_admin       192.168.0.73/32         scram-sha-256

# Application server - replace with actual application subnet/IP
# hostssl appdb          app_user        10.10.10.20/32          scram-sha-256

# Reject non-TLS remote connections
hostnossl all            all             0.0.0.0/0               reject
```

Do **not** copy the application IP from this example.

Use your real source address.

---

# 15. Validate HBA configuration

PostgreSQL exposes parsed HBA information:

```sql
SELECT
    line_number,
    type,
    database,
    user_name,
    address,
    auth_method,
    error
FROM pg_hba_file_rules
ORDER BY line_number;
```

This is extremely useful.

Check for:

```text
error IS NOT NULL
```

For example:

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

# 16. Reload HBA

HBA changes normally require a reload, not a full PostgreSQL restart:

```bash
sudo systemctl reload postgresql-19
```

Verify:

```sql
SELECT pg_reload_conf();
```

---

# 17. TLS configuration

Check:

```sql
SHOW ssl;

SHOW ssl_min_protocol_version;

SHOW ssl_cert_file;

SHOW ssl_key_file;
```

Enable:

```sql
ALTER SYSTEM SET ssl = 'on';
```

Set minimum protocol:

```sql
ALTER SYSTEM SET ssl_min_protocol_version = 'TLSv1.2';
```

Restart:

```bash
sudo systemctl restart postgresql-19
```

PostgreSQL 19's default minimum TLS protocol is TLS 1.2. ([PostgreSQL][1])

---

# 18. TLS certificate

For your lab, create a self-signed certificate.

First identify the data directory:

```bash
sudo -u postgres psql -Atc "SHOW data_directory;"
```

Then:

```bash
sudo -u postgres openssl req \
-new \
-x509 \
-days 365 \
-nodes \
-out /var/lib/pgsql/19/data/server.crt \
-keyout /var/lib/pgsql/19/data/server.key \
-subj "/CN=postgresql19"
```

Secure the key:

```bash
sudo chown postgres:postgres \
/var/lib/pgsql/19/data/server.crt \
/var/lib/pgsql/19/data/server.key

sudo chmod 600 \
/var/lib/pgsql/19/data/server.key
```

PostgreSQL specifically requires the private key to prevent world/group access; `0600` is the standard Unix permission documented by PostgreSQL. ([PostgreSQL][4])

---

# 19. Verify TLS

Connect:

```bash
sudo -u postgres psql
```

Then:

```sql
SELECT
    ssl,
    version,
    cipher
FROM pg_stat_ssl
WHERE pid = pg_backend_pid();
```

Expected:

```text
ssl = true
```

---

# 20. Force TLS from the client

From a PostgreSQL client:

```bash
psql \
"host=192.168.0.145 \
port=5432 \
dbname=postgres \
user=dba_admin \
sslmode=require"
```

For production-grade certificate validation, prefer:

```text
sslmode=verify-full
```

with a trusted CA and a certificate whose identity matches the hostname.

PostgreSQL documents `verify-full` as verifying both the CA chain and that the server hostname matches the certificate. ([PostgreSQL][5])

---

# 21. `require` vs `verify-full`

| Mode          |      TLS | Certificate validation | Hostname validation |
| ------------- | -------: | ---------------------: | ------------------: |
| `disable`     |        ❌ |                      ❌ |                   ❌ |
| `prefer`      | Optional |                      ❌ |                   ❌ |
| `require`     |        ✅ |                Limited |                   ❌ |
| `verify-ca`   |        ✅ |                      ✅ |                   ❌ |
| `verify-full` |        ✅ |                      ✅ |                   ✅ |

For a serious production deployment:

```text
verify-full
```

is the preferred target when your PKI and client configuration support it. ([PostgreSQL][5])

---

# 22. Channel binding

PostgreSQL 19/libpq supports SCRAM channel binding.

Check the client:

```text
channel_binding=require
```

Example connection:

```bash
psql \
"host=postgresql19.example.com \
dbname=appdb \
user=app_user \
sslmode=verify-full \
channel_binding=require"
```

PostgreSQL documents channel binding as a mechanism that helps authenticate the server to the client over SSL/TLS when SCRAM is used. ([PostgreSQL][5])

---

# 23. Database privileges

Create a lab database:

```sql
CREATE DATABASE appdb;
```

Connect:

```bash
sudo -u postgres psql -d appdb
```

Create schema:

```sql
CREATE SCHEMA app AUTHORIZATION postgres;
```

Grant only required access:

```sql
GRANT CONNECT ON DATABASE appdb TO app_user;

GRANT USAGE ON SCHEMA app TO app_user;
```

Then:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO app_user;
```

And sequences:

```sql
GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA app
TO app_user;
```

---

# 24. Default privileges

This is frequently missed by DBAs.

Existing grants do not automatically protect future objects.

Configure:

```sql
ALTER DEFAULT PRIVILEGES
IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES
TO app_user;
```

For sequences:

```sql
ALTER DEFAULT PRIVILEGES
IN SCHEMA app
GRANT USAGE, SELECT, UPDATE
ON SEQUENCES
TO app_user;
```

Now future objects receive the intended privileges.

---

# 25. Review `PUBLIC`

Check database privileges:

```sql
SELECT
    datname,
    datacl
FROM pg_database
WHERE datname = 'appdb';
```

Check schema privileges:

```sql
SELECT
    nspname,
    nspacl
FROM pg_namespace
WHERE nspname NOT LIKE 'pg_%'
  AND nspname <> 'information_schema';
```

Remember that `PUBLIC` represents all roles.

Do not blindly revoke every `PUBLIC` privilege without understanding the consequences.

---

# 26. Check role memberships

```sql
SELECT
    member.rolname AS member,
    role.rolname AS granted_role
FROM pg_auth_members m
JOIN pg_roles role
    ON role.oid = m.roleid
JOIN pg_roles member
    ON member.oid = m.member;
```

Look specifically for:

```text
app_user
     ↓
postgres
```

or other privileged roles.

---

# 27. Find superusers

```sql
SELECT
    rolname,
    rolcanlogin
FROM pg_roles
WHERE rolsuper = true
ORDER BY rolname;
```

Every superuser should be explicitly justified.

---

# 28. Find login roles with elevated privileges

```sql
SELECT
    rolname,
    rolsuper,
    rolcreatedb,
    rolcreaterole,
    rolreplication,
    rolbypassrls,
    rolcanlogin
FROM pg_roles
WHERE rolcanlogin
ORDER BY rolname;
```

This should become part of your regular security health check.

---

# 29. Password expiration

For privileged accounts:

```sql
ALTER ROLE dba_admin
VALID UNTIL '2027-12-31';
```

Check:

```sql
SELECT
    rolname,
    rolvaliduntil
FROM pg_roles
WHERE rolcanlogin;
```

PostgreSQL 19 supports password expiration through `VALID UNTIL`; its password-expiration warning parameter can warn when an expiration date is approaching. ([PostgreSQL][1])

---

# 30. Audit logging

Start with PostgreSQL native logging.

Check:

```sql
SHOW logging_collector;

SHOW log_connections;

SHOW log_disconnections;

SHOW log_line_prefix;

SHOW log_statement;

SHOW log_min_duration_statement;
```

A useful lab configuration:

```sql
ALTER SYSTEM SET log_connections = 'on';

ALTER SYSTEM SET log_disconnections = 'on';

ALTER SYSTEM SET log_line_prefix =
'%m [%p] user=%u,db=%d,app=%a,client=%h ';
```

Reload:

```bash
sudo systemctl reload postgresql-19
```

---

# 31. Don't use `log_statement = 'all'` blindly

Although:

```sql
ALTER SYSTEM SET log_statement = 'all';
```

is useful for a short troubleshooting exercise, it can generate substantial log volume and may expose sensitive statement content.

For normal operations, consider:

```sql
ALTER SYSTEM SET log_statement = 'none';
```

and use targeted logging such as:

```sql
ALTER SYSTEM SET log_min_duration_statement = '1000';
```

for queries taking at least one second.

Tune the threshold according to workload.

---

# 32. pgaudit

For environments requiring detailed database auditing, evaluate the `pgaudit` extension.

Conceptually:

```text
PostgreSQL native logging
        +
pgaudit
        |
        v
Detailed audit trail
```

Use pgaudit when compliance requirements justify it; don't install an auditing extension simply because it is available.

Your final design should define:

```text
WHO
WHAT
WHEN
WHERE
RESULT
```

and establish log retention and protection outside PostgreSQL as well.

---

# 33. Security validation script

Create:

```bash
sudo mkdir -p /opt/postgresql19-security
```

Then:

```bash
sudo vi /opt/postgresql19-security/security_check.sql
```

Use:

```sql
\pset pager off

SELECT '=== VERSION ===' AS section;
SELECT version();

SELECT '=== AUTHENTICATION ===' AS section;

SHOW password_encryption;

SELECT
    name,
    setting
FROM pg_settings
WHERE name IN
(
    'ssl',
    'ssl_min_protocol_version',
    'listen_addresses',
    'port'
);

SELECT '=== ROLE SECURITY ===' AS section;

SELECT
    rolname,
    rolsuper,
    rolcreatedb,
    rolcreaterole,
    rolreplication,
    rolbypassrls,
    rolcanlogin,
    rolvaliduntil
FROM pg_roles
ORDER BY rolname;

SELECT '=== HBA ERRORS ===' AS section;

SELECT *
FROM pg_hba_file_rules
WHERE error IS NOT NULL;

SELECT '=== CONFIGURATION ERRORS ===' AS section;

SELECT
    name,
    setting,
    applied,
    error
FROM pg_file_settings
WHERE error IS NOT NULL
   OR NOT applied;

SELECT '=== ACTIVE TLS CONNECTIONS ===' AS section;

SELECT
    a.pid,
    a.usename,
    a.datname,
    a.client_addr,
    s.ssl,
    s.version,
    s.cipher
FROM pg_stat_activity a
LEFT JOIN pg_stat_ssl s
    ON s.pid = a.pid
WHERE a.client_addr IS NOT NULL;
```

Run:

```bash
sudo -u postgres psql \
-X \
-f /opt/postgresql19-security/security_check.sql
```

---

# 34. Final security validation

You want this overall state:

```text
                         SECURITY
                            |
        +-------------------+-------------------+
        |                   |                   |
       OS                Network             DB
        |                   |                   |
    SELinux             Firewalld          Roles
    Enforcing              |               Privileges
        |                5432 restricted      |
        |                   |              SCRAM
        +-------------------+                   |
                            |                  TLS
                            +------------------+
                                      |
                                  Audit/Logs
```

### Validation checklist

```text
[ ] SELinux = Enforcing
[ ] Firewalld active
[ ] 5432 source restricted
[ ] listen_addresses restricted
[ ] SCRAM enabled
[ ] No unnecessary MD5 authentication
[ ] postgres superuser not used for applications
[ ] Application role is non-superuser
[ ] pg_hba.conf reviewed
[ ] pg_hba_file_rules has no errors
[ ] TLS enabled
[ ] TLS 1.2+ minimum
[ ] server.key permissions secured
[ ] hostssl rules implemented
[ ] Production clients use verify-full
[ ] Database privileges explicitly granted
[ ] Default privileges configured
[ ] PUBLIC privileges reviewed
[ ] Superuser list reviewed
[ ] Role memberships reviewed
[ ] Password expiration reviewed
[ ] Connection/disconnection logging configured
[ ] Slow-query logging configured
[ ] Audit requirements documented
```

---

## Phase 8 RCA example

**Issue:** DBA can connect remotely but application connection is rejected.

```text
Application
     |
     v
5432 reachable
     |
     v
PostgreSQL receives connection
     |
     v
pg_hba.conf
     |
     +---- hostssl required
     |
     +---- Client uses non-SSL
     |
     v
REJECT
```

### Investigation

```bash
Test-NetConnection <server-ip> -Port 5432
```

Then:

```sql
SELECT *
FROM pg_hba_file_rules
ORDER BY line_number;
```

Then check client:

```text
sslmode=require
```

Then server:

```sql
SHOW ssl;
```

Then:

```sql
SELECT
    ssl,
    version,
    cipher
FROM pg_stat_ssl
WHERE pid = pg_backend_pid();
```

### RCA

```text
Network connectivity = OK
PostgreSQL service = OK
Authentication = OK
TLS requirement = NOT MET
```

Therefore the issue is **authentication/TLS policy**, not firewall or PostgreSQL availability.

---

# Phase 8 complete

Your PostgreSQL 19 lab now has:

```text
01 Installation
02 DBA Configuration
03 Remote Connectivity
04 Backup / WAL
05 PITR / DR
06 Monitoring / Health Check
07 Performance / RCA
08 Security Hardening
```

The next logical phase is **Phase 9 — PostgreSQL 19 High Availability**, covering **Streaming Replication → Physical Standby → Replication Slots → WAL retention → Synchronous vs Asynchronous replication → Failover → Promotion → Timeline → Switchover → Rebuild standby → Replication monitoring → RTO/RPO validation**.

[1]: https://www.postgresql.org/docs/19/runtime-config-connection.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 19.3. Connections and Authentication"
[2]: https://www.postgresql.org/docs/19/contrib.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: Appendix F. Additional Supplied Modules and Extensions"
[3]: https://www.postgresql.org/docs/19/client-authentication.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: Chapter 20. Client Authentication"
[4]: https://www.postgresql.org/docs/19/ssl-tcp.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 18.9. Secure TCP/IP Connections with SSL"
[5]: https://www.postgresql.org/docs/19/libpq-connect.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 32.1. Database Connection Control Functions"
