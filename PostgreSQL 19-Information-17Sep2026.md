As of **September 17, 2026**, **PostgreSQL 19 has not reached GA (General Availability) yet**.

* **PostgreSQL 19 GA target:** **September 2026** ([PostgreSQL][1])
* **Latest available:** **PostgreSQL 19 Beta 3**, released **August 13, 2026** ([PostgreSQL][2])
* **Beta 4:** scheduled for **September 24, 2026** ([PostgreSQL Wiki][3])
* **Exact GA date:** **Not officially announced yet**. The project currently lists September 2026 as the planned release window. ([PostgreSQL][1])

### Expected timeline

| Version              | Status      | Date               |
| -------------------- | ----------- | ------------------ |
| PostgreSQL 18        | GA          | Sep 25, 2025       |
| PostgreSQL 19 Beta 1 | Released    | Jun 4, 2026        |
| PostgreSQL 19 Beta 3 | Released    | Aug 13, 2026       |
| PostgreSQL 19 Beta 4 | Planned     | Sep 24, 2026       |
| **PostgreSQL 19 GA** | **Planned** | **September 2026** |

PostgreSQL officially follows an approximately **annual major-release cycle around September**. ([PostgreSQL][4])

**For your PostgreSQL lab:** since you're currently working with PostgreSQL 18, I would treat **19 Beta as testing-only** and use **PostgreSQL 18.x for production/production-like SOPs** until PostgreSQL 19 GA is released.

[PostgreSQL Official Roadmap](https://www.postgresql.org/developer/roadmap/?utm_source=chatgpt.com)

[1]: https://www.postgresql.org/developer/roadmap/?utm_source=chatgpt.com "PostgreSQL: Roadmap"
[2]: https://www.postgresql.org/?utm_source=chatgpt.com "PostgreSQL: The world's most advanced open source database"
[3]: https://wiki.postgresql.org/wiki/PostgreSQL_19_Open_Items?utm_source=chatgpt.com "PostgreSQL 19 Open Items - PostgreSQL wiki"
[4]: https://www.postgresql.org/about/press/faq/?utm_source=chatgpt.com "PostgreSQL: PostgreSQL Press FAQ"

Because **Red Hat Enterprise Linux (RHEL) is Linux**, not Windows. The `.exe` format is a **Windows executable/package convention**, while RHEL uses **RPM packages** for software distribution.

### PostgreSQL on RHEL

| Platform                     | Typical PostgreSQL package | Package manager                        |
| ---------------------------- | -------------------------- | -------------------------------------- |
| **RHEL / Rocky / AlmaLinux** | `.rpm`                     | `dnf`                                  |
| **Fedora**                   | `.rpm`                     | `dnf`                                  |
| **Ubuntu / Debian**          | `.deb`                     | `apt`                                  |
| **Windows**                  | `.exe` installer           | Windows Installer/installer executable |
| **macOS**                    | `.dmg` / Homebrew          | Finder/Homebrew                        |

For example, on RHEL you might install PostgreSQL using:

```bash
sudo dnf install postgresql19-server
```

The package itself is an RPM, conceptually similar to a Windows installer package, but it integrates with the Linux operating system's package-management system.

### Why RPM specifically?

RPM provides more than just the PostgreSQL executable. It can manage:

* PostgreSQL binaries
* Configuration files
* Dependencies
* Libraries
* User/group creation
* Service integration
* File ownership and permissions
* Versioning
* Upgrades
* Package verification
* Uninstallation

For example:

```bash
rpm -qa | grep postgresql
```

can show installed PostgreSQL RPMs.

And:

```bash
dnf info postgresql19-server
```

can show package information.

After installation, PostgreSQL is normally managed as a Linux service rather than by launching an `.exe`:

```bash
sudo systemctl enable postgresql-19
sudo systemctl start postgresql-19
sudo systemctl status postgresql-19
```

### Important distinction

**RPM is not the PostgreSQL executable itself.**

Think of it this way:

```text
Windows
   |
   +-- PostgreSQL binaries
   |
   +-- PostgreSQL installer
   |
   +-- PostgreSQL.exe / installer.exe
   |
   +-- Windows Services
```

versus:

```text
RHEL
   |
   +-- PostgreSQL binaries
   |
   +-- PostgreSQL RPM packages
   |
   +-- dnf
   |
   +-- systemd
   |
   +-- PostgreSQL service
```

So when you download PostgreSQL for **RHEL**, seeing `.rpm` files is **normal and expected**. You would not normally expect PostgreSQL to be distributed as a Windows-style `.exe` installer for RHEL.

If you're specifically asking about **PostgreSQL 19 on RHEL**, I can also give you the **current PostgreSQL 19 RPM installation procedure for RHEL 9/10, including the official PGDG repository, initialization, `systemctl`, `postgresql.conf`, `pg_hba.conf`, firewall, and verification commands**.

For a DBA who wants **minimum manual effort, repeatability, and easy patching**, I recommend using the **official PostgreSQL Yum/PGDG repository + `dnf`**, rather than downloading individual RPMs and installing them manually. PostgreSQL officially provides the PGDG repository for RHEL-family systems and integrates it with normal package/patch management. ([PostgreSQL][1])

### My recommended DBA approach

```text
RHEL
  │
  ├── Add official PGDG repository
  │
  ├── dnf install PostgreSQL
  │
  ├── initdb
  │
  ├── systemctl enable/start
  │
  └── Verify
```

For example, once the appropriate PGDG repository is configured:

```bash
sudo dnf install -y postgresql19-server postgresql19
```

Then initialize and start:

```bash
sudo /usr/pgsql-19/bin/postgresql-19-setup initdb

sudo systemctl enable postgresql-19
sudo systemctl start postgresql-19

sudo systemctl status postgresql-19
```

Verify:

```bash
/usr/pgsql-19/bin/psql --version
```

and:

```bash
sudo -u postgres psql -c "SELECT version();"
```

### Why I prefer this for a DBA

| Method                   |   DBA effort | Patching |  Dependencies | Recommendation             |
| ------------------------ | -----------: | -------: | ------------: | -------------------------- |
| Manual RPM downloads     |         High |   Manual |        Manual | ❌                          |
| Compile from source      |    Very high |   Manual |        Manual | ❌                          |
| RHEL built-in PostgreSQL |          Low |     Easy |          Easy | ⚠️ Version depends on RHEL |
| **Official PGDG + DNF**  | **Very low** | **Easy** | **Automatic** | **✅ Preferred**            |
| Container                |     Very low |     Easy |      Isolated | Depends on architecture    |

The important advantage is that PGDG maintains packages for supported PostgreSQL versions and integrates them with the normal RHEL package-management workflow. ([PostgreSQL][1])

### One DBA improvement I'd make

Don't make the installation a series of commands you manually type every time. Create a **single idempotent installation script**:

```text
01. Validate RHEL version
02. Validate CPU architecture
03. Configure PGDG repository
04. Install PostgreSQL packages
05. Initialize cluster
06. Configure service
07. Enable at boot
08. Start PostgreSQL
09. Validate listener
10. Validate database connection
11. Display PostgreSQL version
12. Display PGDATA
```

Then your entire PostgreSQL deployment becomes essentially:

```bash
sudo ./install-postgresql19.sh
```

That is the approach I would use for your DBA lab and eventually convert into **Ansible** for multiple RHEL servers.

One important point: **PostgreSQL's official RHEL instructions currently show the PGDG repository as the preferred way to obtain versions that may differ from the PostgreSQL version bundled with a particular RHEL release.** ([PostgreSQL][1])

If your target is specifically **RHEL 9**, I can give you a **copy/paste PostgreSQL 19 automated installation script** that performs the complete installation and basic DBA configuration with virtually no manual work.

[1]: https://www.postgresql.org/download/linux/redhat/?utm_source=chatgpt.com "PostgreSQL: Linux downloads (Red Hat family)"

If your target is **RHEL 10**, then I would slightly change the recommendation.

### RHEL 10 + PostgreSQL 19

As of **September 17, 2026**, PostgreSQL's official RHEL-family page says that the **RHEL 10 distribution repositories provide PostgreSQL 16**, while the PGDG repository supports RHEL 10. ([PostgreSQL][1])

However, **PostgreSQL 19 is still in beta** at this point. The official PGDG testing repository contains PostgreSQL 19 beta RPMs for RHEL 10, including Beta 3 builds. ([PostgreSQL Download][2])

So for your DBA lab:

| Requirement                  | Recommendation                     |
| ---------------------------- | ---------------------------------- |
| RHEL 10                      | ✅                                  |
| PostgreSQL 16 from RHEL repo | Production option                  |
| PostgreSQL 18                | **Recommended current GA version** |
| PostgreSQL 19 Beta           | Lab/testing only                   |
| PostgreSQL 19 GA             | Wait for official GA               |
| Installation method          | **PGDG + DNF**                     |
| Manual RPM installation      | ❌ Avoid                            |

### Simplest DBA installation

For **PostgreSQL 18 on RHEL 10**, I'd use:

```bash
dnf install -y postgresql18-server postgresql18
```

Then:

```bash
/usr/pgsql-18/bin/postgresql-18-setup initdb
systemctl enable --now postgresql-18
```

Verify:

```bash
/usr/pgsql-18/bin/psql --version

sudo -u postgres psql -c "SELECT version();"
```

This is preferable to manually downloading `.rpm` files because `dnf` handles dependencies and updates for you.

### For PostgreSQL 19

Don't use the `testing/19` repository for a production installation today. The official repository currently exposes PostgreSQL 19 beta packages for RHEL 10, which confirms that RHEL 10 packaging is available, but these are **pre-GA packages**. ([PostgreSQL Download][2])

**My DBA recommendation for your RHEL 10 lab:**

```text
RHEL 10
   ↓
Official PGDG Repository
   ↓
PostgreSQL 18
   ↓
DNF installation
   ↓
Automated initdb
   ↓
systemctl enable --now
   ↓
Post-install DBA configuration
```

The PostgreSQL project itself provides the RHEL installation procedure and specifically notes that the RHEL-family installation requires initialization and service enable/start after package installation. ([PostgreSQL][1])

If your objective is **learning PostgreSQL 19 specifically**, we can instead build a **RHEL 10 + PostgreSQL 19 Beta lab**, clearly separated from a production installation.

[1]: https://www.postgresql.org/download/linux/redhat/?utm_source=chatgpt.com "PostgreSQL: Linux downloads (Red Hat family)"
[2]: https://download.postgresql.org/pub/repos/yum/testing/19/redhat/rhel-10-x86_64/?utm_source=chatgpt.com "Index of /pub/repos/yum/testing/19/redhat/rhel-10-x86_64/"

Yes. Since you want a **RHEL 10 + PostgreSQL 19 lab**, the cleanest DBA approach is to use the **official PGDG testing repository and `dnf`**, not manually download individual RPMs. The PostgreSQL project explicitly says beta/RC builds are for testing/evaluation, not production. ([PostgreSQL][1])

I also verified the current repository: PostgreSQL 19 **Beta 3 build** packages are available for **RHEL 10 x86_64**, including `postgresql19` and `postgresql19-server`. ([PostgreSQL Download][2])

## RHEL 10 + PostgreSQL 19 Beta Lab

### 1. Confirm RHEL version

```bash
cat /etc/redhat-release
uname -m
```

You want something similar to:

```text
Red Hat Enterprise Linux release 10.x
x86_64
```

### 2. Install the PGDG repository

For minimum DBA effort, use the official PostgreSQL repository RPM:

```bash
sudo dnf install -y \
https://download.postgresql.org/pub/repos/yum/reporpms/EL-10-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

Then:

```bash
sudo dnf repolist
```

### 3. Disable the RHEL PostgreSQL module

For the PGDG packages, disable the distribution PostgreSQL module so that `dnf` doesn't accidentally resolve the RHEL-provided PostgreSQL version:

```bash
sudo dnf -qy module disable postgresql
```

This is consistent with the PostgreSQL RPM installation guidance. ([PostgreSQL Wiki][3])

### 4. Enable the PostgreSQL 19 testing repository

Because PostgreSQL 19 is currently pre-GA, enable its testing repository:

```bash
sudo dnf config-manager --enable pgdg19-updates-testing
```

Check:

```bash
sudo dnf repolist | grep pgdg
```

### 5. Install PostgreSQL 19

```bash
sudo dnf install -y postgresql19 postgresql19-server postgresql19-contrib
```

Verify:

```bash
/usr/pgsql-19/bin/psql --version
```

You should see a PostgreSQL 19 beta version.

### 6. Initialize the database

```bash
sudo /usr/pgsql-19/bin/postgresql-19-setup initdb
```

This creates the PostgreSQL cluster/data directory.

### 7. Enable and start PostgreSQL

```bash
sudo systemctl enable --now postgresql-19
```

Check:

```bash
sudo systemctl status postgresql-19
```

You want:

```text
Active: active (running)
```

### 8. Verify the database

```bash
sudo -u postgres psql -c "SELECT version();"
```

Also:

```bash
sudo -u postgres psql -c "SELECT current_database(), current_user;"
```

### 9. Check PostgreSQL listener

```bash
sudo ss -lntp | grep 5432
```

By default, you should see PostgreSQL listening on the local interface.

### 10. Check PGDATA

```bash
sudo -u postgres psql -c "SHOW data_directory;"
```

Typically with PGDG packages this will be under:

```text
/var/lib/pgsql/19/data
```

---

## DBA verification checklist

After installation, I would run this single block:

```bash
echo "===== OS ====="
cat /etc/redhat-release

echo "===== KERNEL ====="
uname -r

echo "===== ARCHITECTURE ====="
uname -m

echo "===== PostgreSQL ====="
/usr/pgsql-19/bin/psql --version

echo "===== SERVICE ====="
systemctl is-active postgresql-19

echo "===== DATABASE ====="
sudo -u postgres psql -c "SELECT version();"

echo "===== DATA DIRECTORY ====="
sudo -u postgres psql -c "SHOW data_directory;"

echo "===== PORT ====="
sudo -u postgres psql -c "SHOW port;"

echo "===== LISTEN ADDRESSES ====="
sudo -u postgres psql -c "SHOW listen_addresses;"
```

That gives you a basic **installation + service + database + configuration validation** in one execution.

### One important correction to the earlier discussion

RHEL 10's **built-in repository currently provides PostgreSQL 16**, whereas the PostgreSQL Yum Repository provides multiple supported PostgreSQL versions. ([PostgreSQL][4])

For your lab, therefore:

```text
RHEL 10
   │
   ├── RHEL repository
   │      └── PostgreSQL 16
   │
   └── Official PGDG
          └── PostgreSQL 19 Beta
                 └── Testing only
```

The PostgreSQL project currently has the RHEL 10 PGDG testing repository populated with PostgreSQL 19 beta packages, so this is a valid lab setup. ([PostgreSQL Download][2])

**For your DBA learning, I'd actually build this as an automated `install-postgresql19-rhel10.sh` script next**, so you can provision a fresh RHEL 10 VM and install/configure PostgreSQL 19 with **one command**, including firewall, `pg_hba.conf`, remote connectivity, PostgreSQL service validation, and a final health check.

[1]: https://www.postgresql.org/download/?utm_source=chatgpt.com "PostgreSQL: Downloads"
[2]: https://download.postgresql.org/pub/repos/yum/testing/19/redhat/rhel-10-x86_64/?utm_source=chatgpt.com "Index of /pub/repos/yum/testing/19/redhat/rhel-10-x86_64/"
[3]: https://wiki.postgresql.org/wiki/RPM_Installation?utm_source=chatgpt.com "YUM Installation - PostgreSQL wiki"
[4]: https://www.postgresql.org/download/linux/redhat/?utm_source=chatgpt.com "PostgreSQL: Linux downloads (Red Hat family)"
