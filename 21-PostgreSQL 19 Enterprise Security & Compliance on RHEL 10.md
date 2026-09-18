# Phase 21 — PostgreSQL 19 Enterprise Security & Compliance on RHEL 10

**Scope:** PostgreSQL 19 + RHEL 10.
**Phase limit:** We will continue through **Phase 25 only**. No Phase 26 will be introduced.

---

## 21.1 PostgreSQL Security Layers

A production PostgreSQL deployment should be secured at multiple layers:

```text
                    APPLICATION
                         |
                         v
                 NETWORK SECURITY
                         |
                         v
                  RHEL 10 / SELinux
                         |
                         v
                PostgreSQL LISTENER
                         |
                         v
                    pg_hba.conf
                         |
                         v
              AUTHENTICATION / SCRAM
                         |
                         v
                 ROLES / PRIVILEGES
                         |
                         v
                  DATABASE OBJECTS
                         |
                         v
                   DATA / STORAGE
                         |
                         v
                  BACKUP / WAL
```

Security is therefore not just a PostgreSQL configuration task.

---

# 21.2 Authentication vs Authorization

### Authentication

Answers:

> **Who are you?**

Examples:

```text
SCRAM-SHA-256
Certificate authentication
Peer authentication
```

### Authorization

Answers:

> **What are you allowed to do?**

Examples:

```text
CONNECT
SELECT
INSERT
UPDATE
DELETE
EXECUTE
CREATE
```

---

# 21.3 Recommended Authentication

For password-based remote authentication:

```text
SCRAM-SHA-256
```

Check:

```sql
SHOW password_encryption;
```

Set for future password changes:

```sql
ALTER SYSTEM SET password_encryption = 'scram-sha-256';
```

Reload/restart requirements depend on the parameter and PostgreSQL version; verify the effective setting with:

```sql
SHOW password_encryption;
```

---

# 21.4 Create Application Role

Do not use:

```text
postgres
```

for an application.

Example:

```sql
CREATE ROLE app_user
LOGIN
PASSWORD 'CHANGE_ME';
```

Better production architecture:

```text
Application
     |
     v
app_login
     |
     v
app_role
     |
     v
Database objects
```

Separate login identities from privilege roles where practical.

---

# 21.5 Role Hierarchy

Example:

```sql
CREATE ROLE app_read;
CREATE ROLE app_write;
CREATE ROLE app_admin;
```

Grant:

```sql
GRANT app_read TO app_user;
```

This makes privilege management easier than individually granting permissions to every user.

---

# 21.6 Least Privilege

Avoid:

```sql
GRANT ALL PRIVILEGES
ON DATABASE appdb
TO app_user;
```

unless there is a documented reason.

Prefer:

```sql
GRANT CONNECT
ON DATABASE appdb
TO app_user;
```

Then grant only required schema/table privileges.

---

# 21.7 Schema Permissions

Example:

```sql
GRANT USAGE
ON SCHEMA app
TO app_read;
```

Then:

```sql
GRANT SELECT
ON ALL TABLES IN SCHEMA app
TO app_read;
```

For future tables:

```sql
ALTER DEFAULT PRIVILEGES
IN SCHEMA app
GRANT SELECT ON TABLES TO app_read;
```

Be careful about **which role executes `ALTER DEFAULT PRIVILEGES`** because default privileges apply to objects subsequently created by that object-creating role.

---

# 21.8 Restrict `PUBLIC`

PostgreSQL grants some privileges to the special pseudo-role:

```text
PUBLIC
```

Review:

```sql
\dp
```

and:

```sql
\dn+
```

Avoid assuming that removing all `PUBLIC` privileges is automatically correct; review application dependencies first.

---

# 21.9 `pg_hba.conf`

This file controls client authentication.

Typical structure:

```text
TYPE    DATABASE    USER    ADDRESS       METHOD
```

Example:

```conf
host    appdb       app_user    10.10.10.0/24    scram-sha-256
```

Meaning:

```text
TCP connection
     ↓
appdb
     ↓
app_user
     ↓
10.10.10.0/24
     ↓
SCRAM authentication
```

---

# 21.10 HBA Rule Ordering

This is critical:

```text
PostgreSQL reads pg_hba.conf
from top to bottom.
```

The first matching rule is used.

Therefore:

```conf
host all all 0.0.0.0/0 scram-sha-256
```

placed above a restrictive rule can unintentionally broaden access.

Avoid unnecessarily broad rules.

---

# 21.11 Network Restriction

RHEL firewall:

```bash
sudo firewall-cmd --list-all
```

PostgreSQL should generally be reachable only from authorized application/administration networks.

Conceptually:

```text
Internet
   X
   |
Firewall
   |
Authorized network
   |
PostgreSQL :5432
```

Do not expose port 5432 publicly unless there is a compelling, secured architecture.

---

# 21.12 PostgreSQL Listener

Check:

```sql
SHOW listen_addresses;
SHOW port;
```

For example:

```conf
listen_addresses = '10.10.10.20'
port = 5432
```

Avoid:

```conf
listen_addresses = '*'
```

unless the network and HBA configuration intentionally restrict access.

---

# 21.13 TLS/SSL

Check:

```sql
SHOW ssl;
SHOW ssl_cert_file;
SHOW ssl_key_file;
```

Client sessions:

```sql
SELECT
    pid,
    usename,
    client_addr,
    ssl
FROM pg_stat_ssl;
```

A secure architecture can be:

```text
Application
     |
    TLS
     |
     v
PostgreSQL
```

---

# 21.14 TLS Certificate Architecture

```text
              Certificate Authority
                       |
              +--------+--------+
              |                 |
              v                 v
        PostgreSQL Cert    Client Cert
              |                 |
              +--------+--------+
                       |
                       v
                 Mutual TLS
```

For higher-assurance environments, client certificate authentication can complement password authentication.

---

# 21.15 Encryption in Transit vs At Rest

### In transit

Protects:

```text
Application → PostgreSQL
```

using TLS.

### At rest

Protects:

```text
PostgreSQL data files
WAL
Backups
```

from unauthorized access to the underlying storage.

These solve different security problems.

---

# 21.16 RHEL 10 Encryption at Rest

A common RHEL approach is:

```text
PostgreSQL
     |
     v
Filesystem
     |
     v
LUKS / dm-crypt
     |
     v
Physical / virtual storage
```

This provides storage-level encryption.

It is **not PostgreSQL Transparent Data Encryption**.

---

# 21.17 LUKS Architecture

```text
PostgreSQL
     |
     v
PGDATA
     |
     v
Filesystem
     |
     v
LUKS
     |
     v
Encrypted Block Device
     |
     v
Storage
```

LUKS protects data when the encrypted storage is not unlocked.

Once the filesystem is mounted and PostgreSQL can read it, PostgreSQL itself sees normal plaintext filesystem operations.

---

# 21.18 PostgreSQL Native Encryption

PostgreSQL does not provide a general SQL Server-style built-in TDE mechanism for encrypting the entire database transparently at the PostgreSQL storage-engine level.

Therefore distinguish:

```text
PostgreSQL encryption
≠
RHEL LUKS
≠
TLS
≠
Backup encryption
≠
Column encryption
```

Each protects a different layer.

---

# 21.19 Column-Level Encryption

For application-sensitive data, PostgreSQL provides the `pgcrypto` extension.

Example:

```sql
CREATE EXTENSION pgcrypto;
```

Encryption functions include:

```text
pgp_sym_encrypt()
pgp_sym_decrypt()
```

Conceptually:

```text
Application
     |
     v
Sensitive data
     |
     v
Encryption
     |
     v
Encrypted column
```

Key management is the critical operational problem.

Do not hard-code encryption keys into SQL scripts.

---

# 21.20 Encryption Strategy

A mature architecture can look like:

```text
                    DATA
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
     Column         Database       Backup
   Encryption       Storage       Encryption
        |             |             |
     pgcrypto       LUKS         Backup tool
                      |
                      v
                    TLS
                      |
                 Network path
```

---

# 21.21 SELinux

RHEL uses SELinux as a mandatory access-control mechanism.

Check:

```bash
getenforce
```

Possible results:

```text
Enforcing
Permissive
Disabled
```

For production:

```text
Enforcing
```

is generally the target security posture, subject to application validation.

---

# 21.22 SELinux Troubleshooting

Do **not** immediately disable SELinux when PostgreSQL encounters an access problem.

Check:

```bash
sudo ausearch -m AVC -ts recent
```

Also:

```bash
sudo journalctl -t setroubleshoot --since "30 minutes ago"
```

Determine:

```text
What was denied?
Which process?
Which file/context?
Why?
```

Then implement the correct SELinux policy/context.

---

# 21.23 File Permissions

PostgreSQL data directory should be tightly restricted.

Example:

```bash
sudo ls -ld /var/lib/pgsql/19/data
```

Check:

```bash
sudo find /var/lib/pgsql/19/data -maxdepth 1 -type f -printf '%M %u:%g %p\n'
```

The PostgreSQL service account should control its database files.

Avoid allowing ordinary OS users to read PGDATA.

---

# 21.24 Secrets Management

Avoid storing:

```text
Passwords
TLS private keys
Encryption keys
Cloud credentials
```

inside:

```text
Git
Shell scripts
SQL scripts
Public configuration repositories
```

Enterprise options can include:

```text
Vault
Cloud secret-management services
Protected OS credential files
Certificate management systems
```

The correct solution depends on the organization's infrastructure.

---

# 21.25 Audit Logging

Security monitoring should answer:

```text
Who?
What?
When?
From where?
Against which database/object?
Was it successful?
```

PostgreSQL's native logging can capture connection and statement information according to configured logging parameters.

Examples:

```conf
log_connections = on
log_disconnections = on
```

For statement logging, choose settings carefully because broad statement logging can create significant volume and potentially expose sensitive information.

---

# 21.26 `pgaudit`

`pgaudit` is a PostgreSQL extension/project designed to provide detailed session/object audit logging.

Typical architecture:

```text
User
 |
 v
PostgreSQL
 |
 v
pgaudit
 |
 v
Audit Logs
 |
 v
SIEM
```

Audit configuration should be designed around compliance requirements rather than simply enabling every possible audit category.

---

# 21.27 SOX Security Controls

Typical database controls include:

```text
Access management
Privileged-user management
Change management
Audit logging
Separation of duties
Backup controls
Evidence retention
Periodic access review
```

PostgreSQL provides technical mechanisms, but the final SOX control design is an organizational/process requirement.

---

# 21.28 HIPAA-Oriented Controls

For environments handling regulated health information, technical controls commonly include:

```text
Access control
Authentication
Encryption
Audit controls
Integrity controls
Transmission security
Backup/recovery
Incident response
```

PostgreSQL itself does not make an environment "HIPAA compliant."

Compliance is an end-to-end organizational responsibility.

---

# 21.29 GDPR-Oriented Controls

Relevant technical areas can include:

```text
Data minimization
Access control
Encryption
Pseudonymization
Auditability
Retention
Deletion processes
Backup lifecycle
Data discovery
```

Again:

```text
PostgreSQL feature
        ≠
Compliance certification
```

---

# 21.30 Security Validation Script

A DBA can automate checks such as:

```text
PostgreSQL version
listen_addresses
port
SSL
password_encryption
HBA configuration
superuser count
LOGIN roles
Role memberships
PUBLIC privileges
Database CONNECT privileges
Replication privileges
PGDATA permissions
SELinux status
Firewall status
Backup status
Audit configuration
```

---

# 21.31 Superuser Audit

```sql
SELECT
    rolname,
    rolsuper,
    rolcreaterole,
    rolcreatedb,
    rolreplication,
    rolcanlogin
FROM pg_roles
ORDER BY rolname;
```

Investigate unexpected:

```text
rolsuper = true
```

accounts.

---

# 21.32 Login Role Audit

```sql
SELECT
    rolname,
    rolcanlogin,
    rolsuper,
    rolcreaterole,
    rolcreatedb,
    rolreplication
FROM pg_roles
WHERE rolcanlogin
ORDER BY rolname;
```

Review regularly.

---

# 21.33 Security Baseline

A PostgreSQL 19 + RHEL 10 production baseline should include:

```text
[ ] SELinux enforcing
[ ] Firewall enabled
[ ] PostgreSQL not publicly exposed
[ ] SCRAM authentication
[ ] TLS where required
[ ] Least privilege
[ ] Dedicated application roles
[ ] Dedicated replication role
[ ] Limited superusers
[ ] Restricted pg_hba.conf
[ ] Protected PGDATA
[ ] Encrypted storage where required
[ ] Protected backups
[ ] Audit logging
[ ] Monitoring
[ ] Security patching
[ ] Tested recovery
```

---

# 21.34 Security Incident Response

If credentials are compromised:

```text
1. Identify affected role
2. Disable/restrict access
3. Rotate credentials
4. Review active sessions
5. Review audit/log records
6. Determine accessed resources
7. Check for unauthorized changes
8. Validate application
9. Preserve evidence
10. Document incident
```

Don't simply change a password and close the incident.

---

# 21.35 Phase 21 Summary

```text
PostgreSQL 19
      +
RHEL 10
      |
      +-- SCRAM
      +-- TLS
      +-- pg_hba.conf
      +-- Roles
      +-- Least privilege
      +-- SELinux
      +-- Firewall
      +-- LUKS
      +-- pgcrypto
      +-- Audit logging
      +-- pgaudit
      +-- Backup security
      +-- Compliance controls
```

**Phase 21 — COMPLETED.**
