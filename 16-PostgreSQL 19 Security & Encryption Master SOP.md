# Phase 16 — PostgreSQL 19 Security & Encryption Master SOP

### RHEL 10 Enterprise DBA Track

This phase covers **authentication, authorization, TLS, encryption at rest, column-level encryption, auditing, RHEL hardening, and security validation**.

I verified the current PostgreSQL 19 and RHEL 10 documentation before laying this out. PostgreSQL 19 documents `pg_hba.conf`, SCRAM, certificate authentication, LDAP, PAM and OAuth among its authentication mechanisms. ([PostgreSQL][1])

---

## 16.1 PostgreSQL Security Architecture

Think of PostgreSQL security as **five layers**:

```text
                 PostgreSQL SECURITY
                        |
       +----------------+----------------+
       |                |                |
       v                v                v
 Authentication    Authorization     Encryption
       |                |                |
   pg_hba.conf       Roles/ACLs       TLS
   SCRAM             GRANT/REVOKE     LUKS
   Certificates                       pgcrypto
       |                |                |
       +----------------+----------------+
                        |
                        v
                 Auditing / Monitoring
                        |
                        v
                  RHEL 10 Security
```

---

# 16.2 Authentication vs Authorization

This distinction is fundamental.

### Authentication

> **Who are you?**

Examples:

```text
SCRAM
Certificate
LDAP
PAM
OAuth
Peer
```

### Authorization

> **What are you allowed to do?**

Examples:

```text
CONNECT
SELECT
INSERT
UPDATE
DELETE
CREATE
EXECUTE
```

Architecture:

```text
Client
  |
  v
Authentication
  |
  v
PostgreSQL Role
  |
  v
Authorization / ACL
  |
  v
Database Object
```

---

# 16.3 `pg_hba.conf`

PostgreSQL determines client authentication through `pg_hba.conf`.

Check its location:

```sql
SHOW hba_file;
```

Example:

```text
/var/lib/pgsql/19/data/pg_hba.conf
```

PostgreSQL evaluates HBA records in order, so **the first matching rule is important**.

---

# 16.4 Example Secure HBA

Avoid broad rules such as:

```conf
host all all 0.0.0.0/0 trust
```

A more restrictive pattern is:

```conf
hostssl appdb app_user 10.10.20.50/32 scram-sha-256
```

Meaning:

```text
hostssl
   ↓
TCP connection using TLS

appdb
   ↓
Only this database

app_user
   ↓
Only this role

10.10.20.50/32
   ↓
Specific client

scram-sha-256
   ↓
Password authentication
```

---

# 16.5 SCRAM Authentication

PostgreSQL 19 supports SCRAM-SHA-256 password authentication.

Check:

```sql
SHOW password_encryption;
```

Recommended:

```text
scram-sha-256
```

PostgreSQL documentation identifies `scram-sha-256` as the modern password storage/authentication mechanism and notes that older clients may not support it. ([PostgreSQL][2])

---

# 16.6 Configure SCRAM

```sql
ALTER SYSTEM SET password_encryption = 'scram-sha-256';
```

Reload:

```sql
SELECT pg_reload_conf();
```

Create/change password:

```sql
ALTER ROLE app_user
PASSWORD 'Strong-Password-Here';
```

Validate:

```sql
SHOW password_encryption;
```

---

# 16.7 Important DBA Point

Changing:

```text
password_encryption
```

does **not automatically convert every existing stored password**.

The password needs to be reset so PostgreSQL stores the new verifier according to the configured method.

---

# 16.8 Role Security

List roles:

```sql
SELECT
    rolname,
    rolsuper,
    rolcreaterole,
    rolcreatedb,
    rolreplication,
    rolbypassrls,
    rolcanlogin
FROM pg_roles
ORDER BY rolname;
```

Look particularly for:

```text
SUPERUSER
CREATEROLE
CREATEDB
REPLICATION
BYPASSRLS
```

---

# 16.9 Principle of Least Privilege

Avoid:

```text
Application
   |
   v
SUPERUSER
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
   +--> required operations
```

For example:

```sql
CREATE ROLE app_user
LOGIN
PASSWORD 'StrongPassword';
```

Then grant only what is required.

---

# 16.10 Separate DBA and Application Accounts

Recommended:

```text
postgres
   |
   +--> emergency/bootstrap administration

dba_admin
   |
   +--> DBA administration

app_owner
   |
   +--> owns application objects

app_user
   |
   +--> application runtime

report_user
   |
   +--> reporting/read-only access
```

Do not run applications as `postgres`.

---

# 16.11 Database Access

```sql
REVOKE CONNECT ON DATABASE appdb
FROM PUBLIC;
```

Then:

```sql
GRANT CONNECT ON DATABASE appdb
TO app_user;
```

This creates explicit database access.

---

# 16.12 Schema Security

```sql
REVOKE CREATE ON SCHEMA public
FROM PUBLIC;
```

Then create application schema:

```sql
CREATE SCHEMA app AUTHORIZATION app_owner;
```

Grant:

```sql
GRANT USAGE ON SCHEMA app
TO app_user;
```

---

# 16.13 Table Permissions

Example:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO app_user;
```

For sequences:

```sql
GRANT USAGE, SELECT
ON ALL SEQUENCES IN SCHEMA app
TO app_user;
```

---

# 16.14 Default Privileges

This is frequently missed by DBAs.

Existing permissions:

```sql
GRANT ...
```

do not automatically apply to objects created later.

Use:

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE app_owner
IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES TO app_user;
```

---

# 16.15 SSL vs TLS

PostgreSQL configuration historically calls the feature:

```text
ssl
```

but modern secure connections use **TLS**.

PostgreSQL 19 documents `ssl` as the configuration parameter for TLS connections; the terminology "SSL" remains for historical reasons. ([PostgreSQL][2])

---

# 16.16 Enable TLS

Check:

```sql
SHOW ssl;
```

Configure:

```conf
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
```

Restart PostgreSQL if required by the configuration change.

---

# 16.17 Certificate Permissions

The private key is extremely sensitive.

Example:

```bash
sudo chown postgres:postgres server.key
sudo chmod 600 server.key
```

The PostgreSQL service account should be able to access the key while ordinary users should not.

---

# 16.18 Force TLS

Instead of:

```conf
host all all 10.10.20.0/24 scram-sha-256
```

use:

```conf
hostssl all all 10.10.20.0/24 scram-sha-256
```

This means the matching TCP connection must use TLS.

PostgreSQL specifically supports `hostssl` HBA entries for SSL/TLS connections. ([PostgreSQL][3])

---

# 16.19 Verify TLS

From SQL:

```sql
SELECT
    pid,
    usename,
    client_addr,
    ssl,
    version,
    cipher
FROM pg_stat_ssl
JOIN pg_stat_activity
USING (pid)
WHERE usename IS NOT NULL;
```

You want application connections to show:

```text
ssl = true
```

---

# 16.20 Client-Side TLS Verification

For a client connection:

```bash
psql "host=dbserver.example.com port=5432 dbname=appdb user=app_user sslmode=verify-full"
```

`verify-full` provides stronger server identity verification than simply encrypting the connection.

---

# 16.21 Certificate Authentication

PostgreSQL supports certificate-based client authentication.

Example HBA:

```conf
hostssl appdb app_user 10.10.20.50/32 cert
```

The certificate can authenticate the client instead of using a password.

PostgreSQL 19 documents certificate authentication and also supports `clientcert` validation options for `hostssl` entries. ([PostgreSQL][3])

---

# 16.22 TLS Security Model

```text
             Client
                |
          Client Certificate
                |
                v
             TLS 1.2+
                |
                v
          Server Certificate
                |
                v
          PostgreSQL 19
```

RHEL 10's system-wide cryptographic policies support TLS 1.2 and 1.3; TLS versions earlier than 1.2 are not supported by the RHEL 10 policy framework. ([Red Hat Documentation][4])

---

# 16.23 RHEL 10 Cryptographic Policy

Check:

```bash
update-crypto-policies --show
```

Typical default:

```text
DEFAULT
```

RHEL 10 provides predefined policies including:

```text
DEFAULT
LEGACY
FUTURE
FIPS
```

Red Hat recommends choosing the strictest policy compatible with your requirements rather than weakening security unnecessarily. ([Red Hat Documentation][5])

---

# 16.24 FIPS — Important Production DBA Note

If an environment requires FIPS compliance, do **not** casually enable FIPS after PostgreSQL/RHEL deployment.

RHEL 10 documentation states that FIPS mode must be enabled **during RHEL installation**; enabling it after installation is not supported. ([Red Hat Documentation][5])

Therefore:

```text
FIPS requirement identified
        |
        v
Design/build RHEL accordingly
        |
        v
Install with FIPS enabled
        |
        v
Deploy PostgreSQL
```

not:

```text
Existing production server
        |
        v
Enable FIPS
```

---

# 16.25 Encryption at Rest

This is where many DBAs misunderstand PostgreSQL.

PostgreSQL does **not provide a SQL Server TDE-style native transparent database encryption mechanism** for an ordinary PostgreSQL cluster.

For RHEL, a common infrastructure-level approach is:

```text
PostgreSQL
     |
     v
Filesystem
     |
     v
LUKS-encrypted block device
     |
     v
Storage
```

RHEL 10 uses **LUKS/LUKS2** for block-device encryption; LUKS2 is the default format. ([Red Hat Documentation][6])

---

# 16.26 Recommended PostgreSQL Storage Layout

Example:

```text
/dev/mapper/pgdata
       |
       v
/var/lib/pgsql/19/data

/dev/mapper/pgwal
       |
       v
Dedicated WAL filesystem

/dev/mapper/pgbackup
       |
       v
Backup repository
```

This provides infrastructure-level protection for data stored on those encrypted volumes.

---

# 16.27 Check Current Filesystems

```bash
lsblk -f
```

```bash
df -hT
```

Check encryption:

```bash
lsblk
```

Look for:

```text
crypt
```

Example:

```text
sdb
└─crypt-pgdata
   └─vg_pg-lv_data
      └─/var/lib/pgsql/19/data
```

---

# 16.28 LUKS Architecture

```text
PostgreSQL
    |
    v
XFS filesystem
    |
    v
LVM
    |
    v
LUKS2
    |
    v
Physical / Virtual Disk
```

Encryption happens below PostgreSQL.

PostgreSQL sees:

```text
normal filesystem
```

while the storage layer handles encryption/decryption.

---

# 16.29 LUKS Is Not Database-Level Encryption

Important distinction:

```text
LUKS
  ↓
Protects data when storage is not unlocked/accessed normally
```

It does **not** mean:

```text
SELECT
```

automatically returns encrypted data.

Once the filesystem is mounted and PostgreSQL is running:

```text
PostgreSQL
     |
     v
Plain database pages in memory
```

Therefore LUKS primarily addresses **storage/media protection**, not application-level confidentiality.

---

# 16.30 Automated LUKS Unlock

RHEL 10 supports mechanisms such as:

```text
Clevis
Tang
TPM 2.0
PKCS#11
Trustee
```

for automated/unattended unlocking scenarios. ([Red Hat Documentation][5])

This is particularly relevant for enterprise servers where manual passphrase entry during every reboot is undesirable.

---

# 16.31 PostgreSQL Data Encryption Options

Think in layers:

| Layer          | Technology             | Protects                   |
| -------------- | ---------------------- | -------------------------- |
| Network        | TLS                    | Data in transit            |
| Storage        | LUKS2                  | Data at rest               |
| Column/data    | `pgcrypto`             | Selected sensitive values  |
| Backup         | Backup-tool encryption | Backup repository          |
| Application    | App-level encryption   | Sensitive application data |
| Key management | KMS/HSM                | Encryption keys            |

---

# 16.32 `pgcrypto`

PostgreSQL provides the `pgcrypto` extension for cryptographic functions.

Enable:

```sql
CREATE EXTENSION pgcrypto;
```

Verify:

```sql
SELECT extname, extversion
FROM pg_extension
WHERE extname = 'pgcrypto';
```

---

# 16.33 Example — Hashing

```sql
SELECT crypt(
    'MyPassword',
    gen_salt('bf')
);
```

Important:

> Password hashing and encryption are not the same thing.

Hashing:

```text
Plaintext
   |
   v
Hash
```

Encryption:

```text
Plaintext
   |
   v
Ciphertext
   |
   v
Decryption
   |
   v
Plaintext
```

---

# 16.34 Example — Symmetric Encryption

Conceptually:

```sql
SELECT
    pgp_sym_encrypt(
        'Sensitive Data',
        'Encryption-Key'
    );
```

Decrypt:

```sql
SELECT
    pgp_sym_decrypt(
        pgp_sym_encrypt(
            'Sensitive Data',
            'Encryption-Key'
        ),
        'Encryption-Key'
    );
```

For production, **never hard-code encryption keys inside SQL scripts, stored procedures, source code, or application configuration committed to Git**.

---

# 16.35 Key Management

Bad design:

```text
Database
   |
   +--> Encryption key
   |
   +--> Same server
```

Better:

```text
Application / DBA
       |
       v
KMS / HSM
       |
       v
Encryption Key
       |
       v
Database encryption operation
```

Enterprise key-management systems should control:

```text
Key generation
Key storage
Access
Rotation
Revocation
Auditing
Separation of duties
```

---

# 16.36 Encryption and Backups

This is critical.

Even if:

```text
Production database
       ↓
LUKS encrypted
```

your backup may be:

```text
Backup repository
       ↓
Unencrypted
```

That defeats part of the security objective.

Therefore evaluate:

```text
Production encryption
+
WAL archive encryption
+
Backup encryption
+
Offsite copy encryption
+
Snapshot encryption
```

---

# 16.37 Sensitive Data Classification

Before deciding encryption:

```text
PUBLIC
INTERNAL
CONFIDENTIAL
RESTRICTED
```

Examples of potentially restricted information:

```text
Passwords
Financial information
PII
Tokens
API secrets
Government identifiers
Healthcare information
```

Do not automatically encrypt every column.

Encryption introduces:

```text
CPU cost
Key management
Operational complexity
Index/query considerations
```

---

# 16.38 Auditing

PostgreSQL's core statistics views provide operational visibility, but enterprise security auditing often requires additional controls.

A common PostgreSQL ecosystem option is:

```text
pgAudit
```

Use auditing to answer:

```text
Who?
What?
When?
From where?
Against which object?
```

Example:

```text
User: app_user
Database: appdb
Table: customer
Operation: UPDATE
Time: 2026-09-17 18:30
Client: 10.10.20.50
```

---

# 16.39 Security Logging

Review:

```bash
sudo journalctl -u postgresql-19
```

Look for:

```text
authentication failure
connection rejected
permission denied
role changes
configuration changes
TLS errors
```

Do not rely on logs alone for complete auditing.

---

# 16.40 RHEL Firewall

Check:

```bash
sudo firewall-cmd --state
```

List services:

```bash
sudo firewall-cmd --list-all
```

For PostgreSQL, expose port 5432 only to required networks/hosts.

Example concept:

```text
Application subnet
       |
       v
TCP/5432
       |
       v
PostgreSQL
```

Avoid:

```text
Internet
   |
   v
0.0.0.0/0:5432
```

unless there is an exceptional, explicitly secured architecture.

---

# 16.41 SELinux

Check:

```bash
getenforce
```

Expected enterprise posture:

```text
Enforcing
```

Check PostgreSQL-related contexts:

```bash
ls -Z /var/lib/pgsql/19/data
```

Do not disable SELinux simply because PostgreSQL encounters a permission problem.

Instead investigate:

```text
SELinux context
+
audit log
+
file ownership
+
permissions
```

---

# 16.42 Security Incident RCA

### Scenario

Application reports:

```text
FATAL: password authentication failed
```

Investigation:

```text
Client
  |
  v
Network
  |
  v
PostgreSQL
  |
  v
pg_hba.conf
  |
  v
Authentication method
  |
  v
Role
  |
  v
Password
```

Check:

```sql
SELECT rolname, rolcanlogin
FROM pg_roles
WHERE rolname = 'app_user';
```

Then review the matching HBA rule.

---

# 16.43 Another RCA — "Connection Is Not Encrypted"

Check:

```sql
SELECT
    pid,
    usename,
    client_addr,
    ssl,
    version,
    cipher
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid);
```

If:

```text
ssl = false
```

investigate:

```text
pg_hba.conf
ssl setting
client sslmode
connection string
TLS certificates
```

---

# 16.44 Security Validation Checklist

```text
[ ] SCRAM configured
[ ] No unnecessary trust authentication
[ ] No unnecessary PUBLIC privileges
[ ] No application SUPERUSER
[ ] Least privilege implemented
[ ] TLS enabled
[ ] TLS connections validated
[ ] Certificates protected
[ ] RHEL crypto policy reviewed
[ ] Firewall restricted
[ ] SELinux enforcing
[ ] PostgreSQL data volume encrypted
[ ] Backup repository encrypted
[ ] WAL/archive protected
[ ] Sensitive columns reviewed
[ ] Key management documented
[ ] Audit requirements implemented
[ ] Security logs monitored
[ ] Security incident procedure documented
```

---

# 16.45 PostgreSQL Security Golden Architecture

```text
                 APPLICATION
                      |
                      | TLS 1.2/1.3
                      v
                RHEL FIREWALL
                      |
                      v
              PostgreSQL 19
                      |
          +-----------+-----------+
          |                       |
          v                       v
   Authentication          Authorization
      SCRAM/TLS              Roles/ACL
          |                       |
          +-----------+-----------+
                      |
                      v
                 DATA FILES
                      |
                      v
                   LUKS2
                      |
                      v
                  STORAGE

Backup
   |
   v
Encrypted Repository

Keys
   |
   v
KMS / HSM / Enterprise Key Management
```

---

# 16.46 DBA Security Rules

1. **Never use `trust` casually in production.**
2. **Prefer SCRAM-SHA-256 for password authentication.**
3. **Use `hostssl` when TCP connections must be encrypted.**
4. **Use certificate authentication where it provides operational value.**
5. **Do not use `postgres` for application connectivity.**
6. **Apply least privilege.**
7. **Restrict `pg_hba.conf` by database, user, network and authentication method.**
8. **Do not expose port 5432 broadly.**
9. **Keep SELinux enforcing unless there is a documented reason otherwise.**
10. **Use LUKS2 for RHEL storage encryption where appropriate.**
11. **Remember that LUKS is storage encryption, not SQL-level TDE.**
12. **Protect backups independently from production storage.**
13. **Never store encryption keys beside encrypted data without appropriate protection.**
14. **Do not hard-code cryptographic keys in SQL/application source.**
15. **Treat FIPS as a deployment architecture requirement, not a switch to flip later.** ([Red Hat Documentation][5])

---

## Phase 16 Completed

Current track:

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
15  HA / Failover / Switchover
16  Security & Encryption             ← COMPLETED
```

### Next — Phase 17

**PostgreSQL 19 Upgrades, Patching & Version Migration**

We will cover:

* Minor-version patching
* Major-version upgrade
* PostgreSQL 18 → 19
* `pg_upgrade`
* Dump/restore
* Logical replication migration
* Physical replication migration
* Extension compatibility
* Pre-upgrade assessment
* Rollback strategy
* Downtime calculation
* HA upgrade strategy
* Production upgrade runbook
* Upgrade validation
* RCA scenarios
* DBA upgrade checklist.

[1]: https://www.postgresql.org/docs/19/client-authentication.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: Chapter 20. Client Authentication"
[2]: https://www.postgresql.org/docs/19/runtime-config-connection.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 19.3. Connections and Authentication"
[3]: https://www.postgresql.org/docs/19/ssl-tcp.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 19: 18.9. Secure TCP/IP Connections with SSL"
[4]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/securing_networks/planning-and-implementing-tls?utm_source=chatgpt.com "Chapter 4. Planning and implementing TLS | Securing networks | Red Hat Enterprise Linux | 10 | Red Hat Documentation"
[5]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/security_hardening/index?utm_source=chatgpt.com "Security hardening | Red Hat Enterprise Linux | 10 | Red Hat Documentation"
[6]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_storage_devices/encrypting-block-devices-by-using-luks?utm_source=chatgpt.com "Chapter 17. Encrypting block devices by using LUKS | Managing storage devices | Red Hat Enterprise Linux | 10 | Red Hat Documentation"
