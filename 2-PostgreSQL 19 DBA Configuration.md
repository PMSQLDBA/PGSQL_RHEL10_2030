For a **DBA-grade RHEL 10 + PostgreSQL 19 lab**, Phase 2 should configure the server after the base installation rather than mixing everything into the initial installer.

One correction to the previous script: **do not leave `0.0.0.0/0` and `::0/0` in `pg_hba.conf` on anything beyond an isolated lab network.** We'll make the hardened configuration source-restricted.

## Phase 2 — PostgreSQL 19 DBA Configuration

### What this phase will automate

```text
RHEL 10
   │
   ├── OS validation
   ├── PostgreSQL service validation
   ├── Filesystem validation
   ├── PostgreSQL configuration backup
   │
   ├── Memory configuration
   │      ├── shared_buffers
   │      ├── effective_cache_size
   │      ├── work_mem
   │      └── maintenance_work_mem
   │
   ├── Connection configuration
   │      ├── max_connections
   │      └── listen_addresses
   │
   ├── WAL / checkpoint configuration
   │
   ├── Logging
   │
   ├── pg_stat_statements
   │
   ├── SCRAM authentication
   │
   ├── Firewall
   │
   ├── SELinux validation
   │
   └── DBA health check
```

### Recommended approach

I would **not blindly hard-code memory values** into the script. The script should detect available RAM and calculate reasonable lab defaults.

For example:

```text
RAM detected
     ↓
Calculate PostgreSQL memory parameters
     ↓
Generate configuration
     ↓
Restart PostgreSQL
     ↓
Validate
```

That makes the script reusable across your different RHEL VMs.

### Phase 2 script

Save as:

```bash
vi configure-postgresql19-dba.sh
```

Then use:

```bash
#!/bin/bash

set -euo pipefail

PG_MAJOR="19"
PGDATA="/var/lib/pgsql/${PG_MAJOR}/data"
PGSERVICE="postgresql-${PG_MAJOR}"
PGPORT="5432"

echo "============================================================"
echo " PostgreSQL ${PG_MAJOR} DBA Configuration - RHEL 10"
echo "============================================================"

if [[ $EUID -ne 0 ]]; then
    echo "ERROR: Run with sudo/root."
    exit 1
fi

if [[ ! -d "$PGDATA" ]]; then
    echo "ERROR: PostgreSQL data directory not found: $PGDATA"
    exit 1
fi

# ------------------------------------------------------------
# 1. Backup configuration
# ------------------------------------------------------------

echo
echo "[1/10] Backing up PostgreSQL configuration..."

BACKUP_DIR="${PGDATA}/config_backup_$(date +%Y%m%d_%H%M%S)"

mkdir -p "$BACKUP_DIR"

cp "${PGDATA}/postgresql.conf" "$BACKUP_DIR/"
cp "${PGDATA}/pg_hba.conf" "$BACKUP_DIR/"

echo "Backup: $BACKUP_DIR"

# ------------------------------------------------------------
# 2. Detect RAM
# ------------------------------------------------------------

echo
echo "[2/10] Detecting system memory..."

TOTAL_RAM_MB=$(free -m | awk '/^Mem:/ {print $2}')

echo "Total RAM: ${TOTAL_RAM_MB} MB"

# Lab-oriented defaults
if (( TOTAL_RAM_MB <= 4096 )); then

    SHARED_BUFFERS="512MB"
    EFFECTIVE_CACHE="2GB"
    WORK_MEM="8MB"
    MAINTENANCE_WORK_MEM="128MB"

elif (( TOTAL_RAM_MB <= 8192 )); then

    SHARED_BUFFERS="1GB"
    EFFECTIVE_CACHE="4GB"
    WORK_MEM="16MB"
    MAINTENANCE_WORK_MEM="256MB"

elif (( TOTAL_RAM_MB <= 16384 )); then

    SHARED_BUFFERS="4GB"
    EFFECTIVE_CACHE="10GB"
    WORK_MEM="32MB"
    MAINTENANCE_WORK_MEM="512MB"

else

    SHARED_BUFFERS="8GB"
    EFFECTIVE_CACHE="20GB"
    WORK_MEM="64MB"
    MAINTENANCE_WORK_MEM="1GB"

fi

echo "shared_buffers          = $SHARED_BUFFERS"
echo "effective_cache_size   = $EFFECTIVE_CACHE"
echo "work_mem               = $WORK_MEM"
echo "maintenance_work_mem   = $MAINTENANCE_WORK_MEM"

# ------------------------------------------------------------
# 3. Configure PostgreSQL
# ------------------------------------------------------------

echo
echo "[3/10] Configuring PostgreSQL..."

CONF="${PGDATA}/postgresql.conf"

set_parameter()
{
    PARAM="$1"
    VALUE="$2"

    if grep -Eq "^[#[:space:]]*${PARAM}[[:space:]]*=" "$CONF"; then

        sed -i -E \
        "s|^[#[:space:]]*${PARAM}[[:space:]]*=.*|${PARAM} = ${VALUE}|" \
        "$CONF"

    else

        echo "${PARAM} = ${VALUE}" >> "$CONF"

    fi
}

set_parameter "listen_addresses" "'*'"
set_parameter "port" "5432"

set_parameter "shared_buffers" "'${SHARED_BUFFERS}'"
set_parameter "effective_cache_size" "'${EFFECTIVE_CACHE}'"
set_parameter "work_mem" "'${WORK_MEM}'"
set_parameter "maintenance_work_mem" "'${MAINTENANCE_WORK_MEM}'"

set_parameter "max_connections" "200"

# ------------------------------------------------------------
# 4. WAL configuration
# ------------------------------------------------------------

echo
echo "[4/10] Configuring WAL/checkpoints..."

set_parameter "wal_compression" "on"
set_parameter "checkpoint_completion_target" "0.9"

# ------------------------------------------------------------
# 5. Logging
# ------------------------------------------------------------

echo
echo "[5/10] Configuring logging..."

set_parameter "logging_collector" "on"
set_parameter "log_filename" "'postgresql-%Y-%m-%d_%H%M%S.log'"
set_parameter "log_truncate_on_rotation" "on"
set_parameter "log_rotation_age" "'1d'"
set_parameter "log_rotation_size" "'100MB'"

set_parameter "log_connections" "on"
set_parameter "log_disconnections" "on"

# ------------------------------------------------------------
# 6. pg_stat_statements
# ------------------------------------------------------------

echo
echo "[6/10] Configuring pg_stat_statements..."

set_parameter "shared_preload_libraries" "'pg_stat_statements'"

# ------------------------------------------------------------
# 7. PostgreSQL authentication
# ------------------------------------------------------------

echo
echo "[7/10] Configuring SCRAM authentication..."

HBA="${PGDATA}/pg_hba.conf"

# Preserve existing file and add controlled lab rules
cat >> "$HBA" <<'EOF'

# ============================================================
# DBA LAB - SCRAM AUTHENTICATION
# ============================================================

host    all    all    127.0.0.1/32    scram-sha-256
host    all    all    ::1/128         scram-sha-256

EOF

# ------------------------------------------------------------
# 8. Permissions
# ------------------------------------------------------------

echo
echo "[8/10] Correcting PostgreSQL ownership/permissions..."

chown postgres:postgres "$PGDATA"
chmod 700 "$PGDATA"

chmod 600 "${PGDATA}/pg_hba.conf"
chmod 600 "${PGDATA}/postgresql.conf"

# ------------------------------------------------------------
# 9. Restart PostgreSQL
# ------------------------------------------------------------

echo
echo "[9/10] Restarting PostgreSQL..."

systemctl restart "$PGSERVICE"

sleep 5

if ! systemctl is-active --quiet "$PGSERVICE"; then

    echo
    echo "ERROR: PostgreSQL failed to start."
    systemctl status "$PGSERVICE" --no-pager

    exit 1
fi

# ------------------------------------------------------------
# 10. Final validation
# ------------------------------------------------------------

echo
echo "[10/10] Running DBA validation..."

echo
echo "Service:"
systemctl is-active "$PGSERVICE"

echo
echo "PostgreSQL:"
sudo -u postgres /usr/pgsql-${PG_MAJOR}/bin/psql \
-c "SELECT version();"

echo
echo "Memory configuration:"

sudo -u postgres /usr/pgsql-${PG_MAJOR}/bin/psql \
-c "SHOW shared_buffers;"

sudo -u postgres /usr/pgsql-${PG_MAJOR}/bin/psql \
-c "SHOW effective_cache_size;"

sudo -u postgres /usr/pgsql-${PG_MAJOR}/bin/psql \
-c "SHOW work_mem;"

echo
echo "Connections:"

sudo -u postgres /usr/pgsql-${PG_MAJOR}/bin/psql \
-c "SHOW max_connections;"

echo
echo "WAL:"

sudo -u postgres /usr/pgsql-${PG_MAJOR}/bin/psql \
-c "SHOW wal_compression;"

echo
echo "Logging:"

sudo -u postgres /usr/pgsql-${PG_MAJOR}/bin/psql \
-c "SHOW logging_collector;"

echo
echo "Listener:"

ss -lntp | grep ":${PGPORT}" || true

echo
echo "============================================================"
echo " PostgreSQL ${PG_MAJOR} DBA CONFIGURATION COMPLETE"
echo "============================================================"
```

### Run

```bash
chmod +x configure-postgresql19-dba.sh
sudo ./configure-postgresql19-dba.sh
```

### Then verify `pg_stat_statements`

Connect:

```bash
sudo -u postgres psql
```

Run:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

Verify:

```sql
SELECT
    extname,
    extversion
FROM pg_extension
WHERE extname = 'pg_stat_statements';
```

### DBA health check

Run:

```bash
sudo -u postgres psql -x -c "
SELECT
    version(),
    current_database(),
    current_user,
    inet_server_addr(),
    inet_server_port();
"
```

And:

```bash
sudo -u postgres psql -c "
SELECT
    name,
    setting,
    unit
FROM pg_settings
WHERE name IN
(
    'max_connections',
    'shared_buffers',
    'effective_cache_size',
    'work_mem',
    'maintenance_work_mem',
    'wal_compression',
    'checkpoint_completion_target',
    'logging_collector'
)
ORDER BY name;
"
```

### Important DBA note

I would **not yet automate `postgres` password creation** inside the script. Putting a PostgreSQL superuser password directly into an installation script creates a credential-management problem. For a real DBA automation framework, the password should come from **Ansible Vault, a secrets manager, or another protected credential mechanism**.

The next logical phase is **Phase 3: remote DBA connectivity + security hardening**, covering `pg_hba.conf`, RHEL `firewalld`, SELinux, DNS/hostname, SSL/TLS, SCRAM, a dedicated DBA role, a non-superuser application role, and Windows `psql/pgAdmin` connectivity.
