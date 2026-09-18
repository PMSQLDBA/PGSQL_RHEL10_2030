Below is a **one-command DBA lab installer** for **RHEL 10 + PostgreSQL 19 Beta**. It is designed to minimize manual work while keeping the important PostgreSQL/RHEL configuration explicit and auditable.

Before running it, note that PostgreSQL 19 is pre-GA as of **September 17, 2026**, so use this for a lab/testing environment, not production. The PGDG testing repository currently provides PostgreSQL 19 packages for RHEL 10 x86_64.

### `install-postgresql19-rhel10.sh`

```bash
#!/bin/bash

set -euo pipefail

# ============================================================
# PostgreSQL 19 Beta - RHEL 10 Automated Lab Installation
# ============================================================

PG_MAJOR="19"
PG_SERVICE="postgresql-${PG_MAJOR}"
PG_BIN="/usr/pgsql-${PG_MAJOR}/bin"
PGDATA="/var/lib/pgsql/${PG_MAJOR}/data"
PG_PORT="5432"

echo "============================================================"
echo " PostgreSQL ${PG_MAJOR} - RHEL 10 Lab Installation"
echo "============================================================"

# ------------------------------------------------------------
# 1. Root validation
# ------------------------------------------------------------

if [[ $EUID -ne 0 ]]; then
    echo "ERROR: Run this script as root or with sudo."
    exit 1
fi

# ------------------------------------------------------------
# 2. OS validation
# ------------------------------------------------------------

echo
echo "[1/12] Validating operating system..."

if ! grep -q "Red Hat Enterprise Linux" /etc/redhat-release; then
    echo "ERROR: This script is intended for RHEL."
    exit 1
fi

RHEL_MAJOR=$(rpm -E %rhel)

if [[ "$RHEL_MAJOR" != "10" ]]; then
    echo "ERROR: RHEL 10 is required. Detected RHEL ${RHEL_MAJOR}."
    exit 1
fi

ARCH=$(uname -m)

if [[ "$ARCH" != "x86_64" ]]; then
    echo "ERROR: This lab script currently expects x86_64."
    exit 1
fi

cat /etc/redhat-release
echo "Architecture: ${ARCH}"

# ------------------------------------------------------------
# 3. Update metadata
# ------------------------------------------------------------

echo
echo "[2/12] Refreshing DNF metadata..."

dnf clean all
dnf makecache

# ------------------------------------------------------------
# 4. Install PGDG repository
# ------------------------------------------------------------

echo
echo "[3/12] Installing official PostgreSQL PGDG repository..."

dnf install -y \
https://download.postgresql.org/pub/repos/yum/reporpms/EL-10-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# ------------------------------------------------------------
# 5. Disable RHEL PostgreSQL module
# ------------------------------------------------------------

echo
echo "[4/12] Disabling RHEL PostgreSQL module..."

dnf -qy module disable postgresql || true

# ------------------------------------------------------------
# 6. Enable PostgreSQL 19 testing repository
# ------------------------------------------------------------

echo
echo "[5/12] Enabling PostgreSQL 19 testing repository..."

dnf install -y dnf-plugins-core

dnf config-manager --enable pgdg19-updates-testing

# ------------------------------------------------------------
# 7. Install PostgreSQL
# ------------------------------------------------------------

echo
echo "[6/12] Installing PostgreSQL ${PG_MAJOR}..."

dnf install -y \
postgresql${PG_MAJOR} \
postgresql${PG_MAJOR}-server \
postgresql${PG_MAJOR}-contrib

# ------------------------------------------------------------
# 8. Verify binaries
# ------------------------------------------------------------

echo
echo "[7/12] Verifying PostgreSQL binaries..."

${PG_BIN}/psql --version

# ------------------------------------------------------------
# 9. Initialize database cluster
# ------------------------------------------------------------

echo
echo "[8/12] Initializing PostgreSQL cluster..."

if [[ ! -f "${PGDATA}/PG_VERSION" ]]; then

    ${PG_BIN}/postgresql-${PG_MAJOR}-setup initdb

else

    echo "Database cluster already initialized."
fi

# ------------------------------------------------------------
# 10. Start PostgreSQL
# ------------------------------------------------------------

echo
echo "[9/12] Enabling and starting PostgreSQL..."

systemctl enable "${PG_SERVICE}"
systemctl start "${PG_SERVICE}"

sleep 3

if systemctl is-active --quiet "${PG_SERVICE}"; then
    echo "PostgreSQL service is RUNNING."
else
    echo "ERROR: PostgreSQL failed to start."
    systemctl status "${PG_SERVICE}" --no-pager
    exit 1
fi

# ------------------------------------------------------------
# 11. Basic PostgreSQL configuration
# ------------------------------------------------------------

echo
echo "[10/12] Applying basic lab configuration..."

# Backup configuration
cp "${PGDATA}/postgresql.conf" \
   "${PGDATA}/postgresql.conf.bak.$(date +%Y%m%d%H%M%S)"

cp "${PGDATA}/pg_hba.conf" \
   "${PGDATA}/pg_hba.conf.bak.$(date +%Y%m%d%H%M%S)"

# Listen on all interfaces for lab connectivity
sed -i \
"s/^#\?listen_addresses.*/listen_addresses = '*'/" \
"${PGDATA}/postgresql.conf"

# Explicitly set PostgreSQL port
sed -i \
"s/^#\?port.*/port = ${PG_PORT}/" \
"${PGDATA}/postgresql.conf"

# ------------------------------------------------------------
# 12. Configure local firewall
# ------------------------------------------------------------

echo
echo "[11/12] Configuring firewalld..."

if systemctl is-active --quiet firewalld; then

    firewall-cmd --permanent \
        --add-port=${PG_PORT}/tcp

    firewall-cmd --reload

    echo "Port ${PG_PORT}/tcp opened in firewalld."

else

    echo "firewalld is not running. Firewall configuration skipped."

fi

# ------------------------------------------------------------
# Configure PostgreSQL host authentication
# ------------------------------------------------------------

echo
echo "[12/12] Configuring pg_hba.conf..."

cat >> "${PGDATA}/pg_hba.conf" <<EOF

# PostgreSQL 19 RHEL 10 LAB
# Added by automated installation script
host    all             all             0.0.0.0/0               scram-sha-256
host    all             all             ::0/0                   scram-sha-256
EOF

# Restart PostgreSQL
systemctl restart "${PG_SERVICE}"

sleep 3

# ------------------------------------------------------------
# Final validation
# ------------------------------------------------------------

echo
echo "============================================================"
echo " FINAL VALIDATION"
echo "============================================================"

echo
echo "OS:"
cat /etc/redhat-release

echo
echo "Kernel:"
uname -r

echo
echo "Architecture:"
uname -m

echo
echo "PostgreSQL:"
${PG_BIN}/psql --version

echo
echo "Service:"
systemctl is-active "${PG_SERVICE}"

echo
echo "Database version:"
sudo -u postgres ${PG_BIN}/psql -c "SELECT version();"

echo
echo "Data directory:"
sudo -u postgres ${PG_BIN}/psql -c "SHOW data_directory;"

echo
echo "Port:"
sudo -u postgres ${PG_BIN}/psql -c "SHOW port;"

echo
echo "Listen addresses:"
sudo -u postgres ${PG_BIN}/psql -c "SHOW listen_addresses;"

echo
echo "Server status:"
ss -lntp | grep ":${PG_PORT}" || true

echo
echo "============================================================"
echo " PostgreSQL ${PG_MAJOR} LAB INSTALLATION COMPLETE"
echo "============================================================"

echo
echo "Service:"
echo "  systemctl status ${PG_SERVICE}"

echo
echo "Connect locally:"
echo "  sudo -u postgres ${PG_BIN}/psql"

echo
echo "Configuration:"
echo "  ${PGDATA}/postgresql.conf"
echo "  ${PGDATA}/pg_hba.conf"

echo
echo "============================================================"
```

### Run it

On your fresh RHEL 10 VM:

```bash
vi install-postgresql19-rhel10.sh
```

Paste the script, save it, then:

```bash
chmod +x install-postgresql19-rhel10.sh
```

Run:

```bash
sudo ./install-postgresql19-rhel10.sh
```

At the end, your DBA validation should show:

```text
PostgreSQL 19.x
active
5432
listen_addresses = *
```

### One security change I strongly recommend

The script above opens PostgreSQL to all IPv4/IPv6 sources **because this is a lab and is intended to make remote connectivity easy**.

For an enterprise environment, **do not use**:

```text
0.0.0.0/0
::0/0
```

Instead, restrict `pg_hba.conf` to your application/server subnet, for example:

```text
host    all    all    192.168.10.0/24    scram-sha-256
```

and restrict the RHEL firewall to the required source IP/subnet.

Also, before exposing PostgreSQL beyond a trusted lab network, set a password for the `postgres` role:

```bash
sudo -u postgres psql
```

Then:

```sql
ALTER ROLE postgres WITH PASSWORD 'YOUR_STRONG_PASSWORD';
```

For your DBA lab, I would use this script as **Phase 1** and then build **Phase 2** with automated DBA configuration: dedicated data/WAL filesystems, `shared_buffers`, `effective_cache_size`, `work_mem`, `maintenance_work_mem`, logging, `log_rotation`, `pg_stat_statements`, password authentication, NTP/time synchronization, SELinux considerations, firewall restriction, backup directory, and a complete post-install health-check report.
