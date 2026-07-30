# Comprehensive PostgreSQL Self-Hosted Deployment Runbook

This runbook describes the end-to-end installation, tuning, configuration, and Business Continuity Planning (BCP) setup for a self-hosted PostgreSQL database cluster on Azure's Red Hat Enterprise Linux (RHEL) 8 VM.

---

## 1. Runbook Overview

### Why Self-Hosted PostgreSQL (SH PG)?
While managed cloud databases (such as Azure Database for PostgreSQL Flexible Server) offer convenience, a self-hosted deployment on a virtual machine (VM) provides:
- **Deeper Hardware Control:** Direct tuning of the file system, disk configurations (LVM), and kernel/memory optimizations.
- **Customized Extensions & Versioning:** Integration of specialized packages (e.g., custom builds of `pg_proctab` or `pg_stat_monitor`) that might be restricted or unsupported on PaaS.
- **Advanced Hook Scripts:** Capability to run custom script triggers, such as dynamically tuning configuration parameters during system boot based on the actual allocated VM size.
- **Cost & Network Topology Control:** Custom BCP pipelines utilizing utilities like `azcopy` integrated directly with local backup policies, bypassing cloud-managed vendor storage lock-ins.
- **Public IP Requirements:** Accommodates tools like Qlik which require public IP endpoints, whereas managed Azure instances typically restrict endpoints to private IPs.
- **Read Replica Access:** Fulfills strict customer demands for direct access to read replicas.
- **Performance Isolation:** Isolates larger databases on dedicated VMs to avoid "noisy neighbor" latency issues common in shared PaaS environments.

---

## 2. Prerequisites & Assumptions

Before starting the deployment, verify the following prerequisites are met:
1. **VM Provisioning:** A VM provisioned from a RHEL 8 base/clone template containing:
   - Core operating system installed (Red Hat Enterprise Linux 8).
   - Minimal PostgreSQL configuration files and binaries installed.
   - Primary OS hard disk (`/dev/<os_disk>` or equivalent, 64 GB).
   - Secondary hard disk allocated for PostgreSQL data storage (`/dev/<pg_data_disk>` or equivalent).
2. **Environment Variables:** The `postgres` user must have `$PGDATA` configured pointing to the data directory (typically `/u01/pgsql/<version>/data`).
3. **Permissions:** Root access (`sudo`) is required for OS level adjustments.

---

## 3. Phase 1: Operating System Configuration

### 3.1 Cron Job Suspend (Pre-configuration Safety)
To prevent legacy background scripts from executing during VM setup:
1. Log in as the `root` and `postgres` user.
2. Comment/Disable existing cron jobs by commenting them out:
   ```bash
   crontab -e
   # Prepend a '#' character to all active cron entries
   ```

### 3.2 LVM Partition Extension (PGDATA Disk Expansion)
#### Theory: Logical Volume Management (LVM)
LVM abstracts physical storage devices, allowing the operating system to resize storage volumes dynamically without shutting down services. It consists of:
- **Physical Volume (PV):** The raw hard disk partition (e.g., `/dev/<pg_data_disk>`).
- **Volume Group (VG):** A storage pool combining multiple PVs.
- **Logical Volume (LV):** The virtual partition carved from the VG, which hosts the filesystem (e.g., ext4, xfs).

#### Commands [To Be Executed as Root User only]:
To extend the volume hosting `/u01/pgsql/<version>/data` to use newly allocated VM storage:

1. Scan and detect the newly allocated disk space:
   ```bash
   # Resize the physical volume to match the virtual disk capacity
     pvresize /dev/{filesystem_device}
   ```
   *(Replace `{filesystem_device}` with your actual device name, e.g., `sdb` or `nvme0n1`)*.

2. Extend the logical volume and resize the underlying filesystem concurrently:
   ```bash
      lvextend -L +[Size]G -r /dev/[vg_name]/[lv_name]
   ```
   *Note: The `-r` flag triggers an online resize of the filesystem (XFS or EXT4) automatically.*

### 3.3 Hostname and Network Configuration
1. Update the hostname to reflect the environment naming conventions (e.g., `vm-psql-erp-prod-02`):
   ```bash
   sudo vi /etc/hostname
   # Enter the new hostname, save, and exit (Esc, :wq)

   or

   hostnamectl set-hostname vm-erpdb-psql-prod-09 [set hostname without system reboot]
   ```

### 3.4 Cleanup of Legacy PostgreSQL Directories
If the VM template was provisioned with an older PostgreSQL version (e.g., version 16), remove its leftover data directory to avoid conflicts:
```bash
# Navigate to the postgres home directory
cd /var/lib/pgsql

# Remove the old version directory
rm -rf 16
```


2. Validate CPU, memory, and disk allocations:
   ```bash
   df -h / lsblk      # Check filesystem capacity
   free -m    # Validate RAM size
   lscpu      # Validate CPU core count
   ```
3. Restart the virtual machine to apply OS-level configurations:
   ```bash
   sudo reboot
   ```

---

## 4. Phase 2: PostgreSQL Systemd Auto-Tuning Configuration

### 4.1 Theory: Dynamic Tuning on VM Boot
PostgreSQL does not natively adjust its memory/CPU parameters when host system specifications change (such as adding/Reducing RAM or CPU cores to a VM).
To resolve this, we configure a systemd `ExecStartPre` hook. This script executes prior to the engine startup, analyzing the current VM hardware resources, dynamically tuning PostgreSQL configuration variables, and writing them to `postgresql.conf`.

### 4.2 Deploying the Auto-Tune Script
Create the directory structure for backups and logs, and create the auto-tune script:
```bash
sudo mkdir -p /u01/backup/logs/autotune
sudo mkdir -p /u01/backup/logs/autotune/config_backup
sudo mkdir -p /u01/Gsl
sudo vi /u01/Gsl/pg_autotune_systemd.sh
```

Insert the following script content (`pg_autotune_systemd.sh` v4):

```bash
#!/bin/bash
# pg_autotune_systemd.sh — Version 4
# Detects hardware changes, modifies postgresql.conf.
# Designed exclusively to be run via Systemd ExecStartPre hook.
#
# V4 Changes:
#   - max_parallel_workers_per_gather fallback formula updated to: min(4, CPU/2)
#   - Email functionality removed entirely
# V3 Changes from V2:
#   - shared_buffers      : 25% → 26% of RAM
#   - effective_cache_size: 75% → 76% of RAM
#   - systemctl log message now uses dynamic PG_VERSION (no hardcoded '16')

# --- USER CONFIGURATION ---
MAX_CONNECTIONS=5000
WORK_MEM_MB=1024
WAL_BUFFERS_MB=32
TEST_SLEEP_SECONDS=30
# --------------------------

# --- LOGGING CONFIGURATION ---
LOG_DIR="/u01/backup/logs/autotune"
LOG_FILE="${LOG_DIR}/pg_autotune_$(date +'%Y%m%d_%H%M%S').log"

# Auto-create log directory if it does not exist
if [ ! -d "$LOG_DIR" ]; then
    mkdir -p "$LOG_DIR" 2>/dev/null
fi

# Redirect all output to the log file
exec >> "$LOG_FILE" 2>&1

# Verify the log file was actually created (exec fails silently if dir is unwritable)
if [ ! -f "$LOG_FILE" ]; then
    # Fall back to stderr — visible via: journalctl -u postgresql-PG_VERSION
    exec >&2
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] CRITICAL: Cannot write to log file: $LOG_FILE"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] CRITICAL: Check that $LOG_DIR exists and is writable by root."
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] CRITICAL: Continuing — all output will appear in journalctl."
fi

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

log "================================================================="
log "Executing PostgreSQL Auto-Tune Hook  (v4)"
# -----------------------------

# --- INTERNAL STATE (Do not modify manually) ---
LAST_KNOWN_STATE="First Run"
# -----------------------------------------------

# 1. Dynamically Locate postgresql.conf
log "Locating postgresql.conf dynamically using PGDATA..."

# Check if PGDATA is available in the current environment
if [ -n "$PGDATA" ] && [ -f "$PGDATA/postgresql.conf" ]; then
    PG_CONF="$PGDATA/postgresql.conf"
    PGDATA_DIR="$PGDATA"
else
    # Attempt to load the postgres user's environment variables (useful if running from root crontab)
    PGDATA_ENV=$(su - postgres -c 'echo $PGDATA' 2>/dev/null | grep -v '^\s*$' | tail -n 1)
    if [ -n "$PGDATA_ENV" ] && [ -f "$PGDATA_ENV/postgresql.conf" ]; then
        PG_CONF="$PGDATA_ENV/postgresql.conf"
        PGDATA_DIR="$PGDATA_ENV"
    else
        log "CRITICAL: \$PGDATA environment variable is not set or postgresql.conf not found inside it."
        log "Please ensure \$PGDATA is exported, or set PG_CONF manually."
        exit 1
    fi
fi
log "Found PostgreSQL Configuration: $PG_CONF"

# Dynamically Detect PostgreSQL Version
# Note: postgres -V reads the installed binary — does NOT require the service to be running.
PG_VERSION=$(postgres -V 2>/dev/null | awk '{print $NF}' | cut -d. -f1)
if [ -z "$PG_VERSION" ]; then
    PG_VERSION=$(psql -V 2>/dev/null | awk '{print $3}' | cut -d. -f1)
fi
if [ -z "$PG_VERSION" ]; then
    PG_VERSION="17" # Fallback
fi
log "Detected PostgreSQL Version: $PG_VERSION"

# 2. Detect Hardware
TOTAL_RAM_KB=$(grep MemTotal /proc/meminfo | awk '{print $2}')
TOTAL_RAM_MB=$((TOTAL_RAM_KB / 1024))
TOTAL_RAM_GB=$((TOTAL_RAM_MB / 1024))
CPU_CORES=$(nproc)

log "Detected Hardware: ${TOTAL_RAM_MB}MB RAM  |  ${CPU_CORES} CPU Cores"

CURRENT_STATE="${TOTAL_RAM_MB}MB_RAM-${CPU_CORES}_CPUS"

# 3. Hardware Change Validation (Self-contained)
if [ "$CURRENT_STATE" == "$LAST_KNOWN_STATE" ]; then
    log "Hardware unchanged ($CURRENT_STATE). Skipping auto-tune process."
    exit 0
fi

log "================================================================="
log "HARDWARE CHANGE DETECTED!"
log "Previous Hardware: $LAST_KNOWN_STATE"
log "New Hardware:      $CURRENT_STATE"
log "================================================================="

# 4. Memory Parameters (Local Math)
# shared_buffers      = 26% of OS-visible physical RAM (MemTotal)
# effective_cache_size= 76% of OS-visible physical RAM (MemTotal)  [planner hint only]
# maintenance_work_mem= 5% of RAM (RAM_GB / 20), minimum 1 GB
SHARED_BUFFERS_MB=$(( (TOTAL_RAM_MB * 26) / 100 ))
EFFECTIVE_CACHE_MB=$(( (TOTAL_RAM_MB * 76) / 100 ))

MAINT_WORK_MEM_GB=$((TOTAL_RAM_GB / 20))
if [ "$MAINT_WORK_MEM_GB" -lt 1 ]; then MAINT_WORK_MEM_GB=1; fi
MAINT_WORK_MEM_MB=$((MAINT_WORK_MEM_GB * 1024))

# work_mem and wal_buffers are static — set via USER CONFIGURATION block above

# 5. CPU/Parallel Parameters (API with Local Fallback)
API_URL="https://api.pgconfig.org/v1/tuning/get-config?format=conf&max_connections=${MAX_CONNECTIONS}&pg_version=${PG_VERSION}&environment_name=OLTP&total_ram=${TOTAL_RAM_GB}&cpus=${CPU_CORES}&drive_type=SSD&arch=x86-64&os_type=linux"
log "Calling pgconfig.org API for CPU/parallel parameters..."
API_CONF=$(curl -s --max-time 5 "$API_URL" 2>/dev/null)

if [ -n "$API_CONF" ] && echo "$API_CONF" | grep -q "max_parallel_workers"; then
    log "API call succeeded. Parsing CPU parameters from response."
    MAX_WORKER_PROCESSES=$(echo "$API_CONF" | grep "^max_worker_processes" | awk '{print $3}')
    MAX_PARALLEL_WORKERS=$(echo "$API_CONF" | grep "^max_parallel_workers " | awk '{print $3}')
    MAX_PARALLEL_WORKERS_PER_GATHER=$(echo "$API_CONF" | grep "^max_parallel_workers_per_gather" | awk '{print $3}')
else
    log "API call failed or timed out. Using local fallback math for CPU parameters."
    MAX_WORKER_PROCESSES=$CPU_CORES
    MAX_PARALLEL_WORKERS=$CPU_CORES
    # Formula: min(4, Total CPU Cores / 2)
    MAX_PARALLEL_WORKERS_PER_GATHER=$(( (CPU_CORES / 2) < 4 ? (CPU_CORES / 2) : 4 ))
    if [ "$MAX_PARALLEL_WORKERS_PER_GATHER" -eq 0 ]; then MAX_PARALLEL_WORKERS_PER_GATHER=1; fi
fi

# 6. Backup the Configuration
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_PATH="/u01/backup/logs/autotune/config_backup/postgresql.conf.backup.${TIMESTAMP}"
cp "$PG_CONF" "$BACKUP_PATH"
log "Backed up $PG_CONF to $BACKUP_PATH"

# 7. Direct File Modification Function
# Logs a before-and-after trace for every parameter change.
set_pg_param() {
    local key=$1
    local val=$2
    local prev_val
    if grep -qE "^\s*#?\s*${key}\s*=" "$PG_CONF"; then
        prev_val=$(grep -E "^\s*#?\s*${key}\s*=" "$PG_CONF" | head -n 1)
        sed -i -E "s/^\s*#?\s*${key}\s*=.*/${key} = ${val}/g" "$PG_CONF"
        log "Updated parameter:  [${prev_val}]   --->   [${key} = ${val}]"
    else
        echo "${key} = ${val}" >> "$PG_CONF"
        log "Added NEW parameter:   [${key} = ${val}]"
    fi
}

# Apply parameters
log "Applying optimized configurations to postgresql.conf..."
set_pg_param "shared_buffers"               "'${SHARED_BUFFERS_MB}MB'"
set_pg_param "effective_cache_size"         "'${EFFECTIVE_CACHE_MB}MB'"
set_pg_param "maintenance_work_mem"         "'${MAINT_WORK_MEM_MB}MB'"
set_pg_param "work_mem"                     "'${WORK_MEM_MB}MB'"
set_pg_param "wal_buffers"                  "'${WAL_BUFFERS_MB}MB'"
set_pg_param "max_worker_processes"         "$MAX_WORKER_PROCESSES"
set_pg_param "max_parallel_workers"         "$MAX_PARALLEL_WORKERS"
set_pg_param "max_parallel_workers_per_gather" "$MAX_PARALLEL_WORKERS_PER_GATHER"

# 8. (Restart logic removed - handled natively by systemd)

# 9. Update Self-Contained State
SCRIPT_PATH=$(readlink -f "$0")
sed -i -E "s/^LAST_KNOWN_STATE=.*/LAST_KNOWN_STATE=\"$CURRENT_STATE\"/" "$SCRIPT_PATH"
log "Process complete. State saved inside script."

# 10. Testing Gap (set TEST_SLEEP_SECONDS=0 in production)
if [ "$TEST_SLEEP_SECONDS" -gt 0 ]; then
    log "TESTING MODE: Sleeping for ${TEST_SLEEP_SECONDS} seconds to demonstrate ExecStartPre gap."
    log "You can check 'systemctl status postgresql-${PG_VERSION}' in another terminal now."
    log "The database engine will not start until this sleep finishes."
    sleep "$TEST_SLEEP_SECONDS"
    log "Sleep finished. Handing control back to systemd native auto-startup..."
fi
```

### 4.3 Script Permissions & SELinux Configuration
#### Theory: SELinux Policies on Custom Services
SELinux (Security-Enhanced Linux) enforces Mandatory Access Control. When systemd attempts to run a script located in custom mounts (like `/u01/`), SELinux blocks execution unless the script is assigned the standard system binary domain context (`bin_t`).

```bash
# Grant execution permissions
sudo chmod +x /u01/Gsl/pg_autotune_systemd.sh

# Apply SELinux binary execution labeling
sudo chcon -t bin_t /u01/Gsl/pg_autotune_systemd.sh

# Ensure the postgres user owns the entire /u01 directory tree
sudo chown -R postgres:postgres /u01
```


### 4.4 Hooking Script into PostgreSQL Systemd Unit
1. Edit the systemd service file for the PostgreSQL engine:
   ```bash
   sudo vi /usr/lib/systemd/system/postgresql-<version>.service
   ```
2. Locate the line beginning with `ExecStartPre=`.
3. Add the custom hook execution rule **directly above** the default directory check line:
   ```ini
   # Add this line (the '+' instructs systemd to execute it with root permissions):
   ExecStartPre=+/u01/Gsl/pg_autotune_systemd.sh
   
   # Just above:
   ExecStartPre=/usr/pgsql-<version>/bin/postgresql-<version>-check-db-dir ${PGDATA}
   ```
4. Reload the systemd daemon to pick up configuration changes:
   ```bash
   sudo systemctl daemon-reload
   ```

---

## 5. Phase 3: Database Engine Parameter Tuning

### 5.1 Theory: Key Parameter Tuning in PostgreSQL
By default, PostgreSQL installation configurations are very conservative (designed for low-resource environments). For large, high-capacity databases, adjusting parameters optimizes CPU cycle distribution, memory usage, write traffic, and vacuum behavior:

- **Memory Settings (`work_mem`, `shared_buffers`):** Determines the maximum RAM allocated for internal sort operations, hash tables, and database caching.
- **Autovacuum Optimization:** In high-write environments, default values fail to keep up with dead tuple generation, leading to table bloat. Reducing scale factors triggers autovacuuming much faster.
- **Logical Replication (For Insightx Sync):** Configures logical replication parameters, allowing external consumers to stream data changes (CDC) from WAL logs.

#### Configuration Management Breakdown
The following table outlines the total parameters modified in this deployment and specifies whether they are handled dynamically by the auto-tune hook script or manually during deployment in `postgresql.conf`:

| Parameter                               | Managed By           | Description                                                |
|-----------------------------------------|----------------------|------------------------------------------------------------|
| `shared_buffers`                        | `pgautotune.sh`      | RAM allocated for database caching.                        |
| `effective_cache_size`                  | `pgautotune.sh`      | Planner hint for disk cache size.                          |
| `maintenance_work_mem`                  | `pgautotune.sh`      | Memory for indexes/maintenance (vacuum).                   |
| `work_mem`                              | `pgautotune.sh`      | Memory limit for sorting/joins.                            |
| `wal_buffers`                           | `pgautotune.sh`      | Memory allocation for WAL data.                            |
| `max_worker_processes`                  | `pgautotune.sh`      | Upper limit on concurrent background workers.              |
| `max_parallel_workers`                  | `pgautotune.sh`      | Limit on active parallel query workers.                    |
| `max_parallel_workers_per_gather`       | `pgautotune.sh`      | Parallel workers allowed per execution query.              |
| `track_io_timing`                       | Manual (Section 5.2) | Collect stats on read/write latency.                       |
| `track_activity_query_size`             | Manual (Section 5.2) | Allocate larger buffer for SQL query logging.              |
| `temp_buffers`                          | Manual (Section 5.2) | Cache size allocated for temporary tables.                 |
| `effective_io_concurrency`              | Manual (Section 5.2) | Number of concurrent disk operations (for SSDs).           |
| `random_page_cost`                      | Manual (Section 5.2) | Relative index disk read cost estimate.                    |
| `seq_page_cost`                         | Manual (Section 5.2) | Relative sequential page access cost.                      |
| `autovacuum_max_workers`                | Manual (Section 5.2) | Max concurrent autovacuum processes.                       |
| `autovacuum_vacuum_scale_factor`        | Manual (Section 5.2) | Clean dead tuples after % rows change limit.               |
| `autovacuum_vacuum_insert_scale_factor` | Manual (Section 5.2) | Clean tables after % inserts limit.                        |
| `autovacuum_vacuum_cost_delay`          | Manual (Section 5.2) | Sleep time limit when vacuum reaches limit.                |
| `log_autovacuum_min_duration`           | Manual (Section 5.2) | Log autovacuum runs taking longer than limit.              |
| `max_prepared_transactions`             | Manual (Section 5.2) | Support for concurrent prepared/2PC transactions.          |
| `max_connections`                       | Manual (Section 5.2) | Concurrent connection capacity limit.                      |
| `shared_preload_libraries`              | Manual (Section 5.2) | Preload extension libraries.                               |
| `cron.database_name`                    | Manual (Section 5.2) | Target database for pg_cron extension.                     |
| `cron.timezone`                         | Manual (Section 5.2) | Timezone configured for pg_cron jobs.                      |
| `cron.max_running_jobs`                 | Manual (Section 5.2) | Maximum concurrent pg_cron jobs allowed.                   |
| `wal_level`                             | Manual (Section 5.2) | Enable CDC logical replication streams.                    |
| `max_replication_slots`                 | Manual (Section 5.2) | Concurrent replication stream connections limit.           |
| `max_wal_senders`                       | Manual (Section 5.2) | Senders running parallel logical transactions.             |
| `sync_replication_slots`                | Manual (Section 5.2) | Sync replication slot details across restarts.             |
| `min_wal_size` / `max_wal_size`         | Manual (Section 5.2) | Sizing limits for WAL segments.                            |
| `archive_mode`                          | Post-Migration       | Enable continuous WAL archiving for PITR.                  |
| `archive_command`                       | Post-Migration       | Shell command (`BLOB.sh`) to transfer WAL segments.        |
| `archive_timeout`                       | Post-Migration       | Force WAL rotation after timeout (120s).                   |

> **Note:** Point-in-Time Recovery (PITR) parameters (`archive_mode`, `archive_command`, and `archive_timeout`) are documented here for completeness, but they must be **manually enabled post-migration** to avoid heavy archiving operations during the initial bulk data migration.

### 5.2 Modifying parameters in `postgresql.conf`
Open the PostgreSQL configuration file:
```bash
sudo vi $PGDATA/postgresql.conf
```
Update or append the following values:

```ini
# --- Performance & Resource Configurations ---
track_io_timing = on                    # Collect stats on read/write latency
track_activity_query_size = 102400       # Allocate larger buffer for SQL query logging
temp_buffers = '1GB'                    # Cache size allocated for temporary tables
effective_io_concurrency = 100          # Number of concurrent disk operations (for SSDs)
random_page_cost = 1.1                  # Relative index disk read cost estimate (1.1 for SSD)
seq_page_cost = 2.0                     # Relative sequential page access cost

# --- Autovacuum Tuning (Reduce Bloat) ---
autovacuum_max_workers = 6               # Max concurrent autovacuum processes
autovacuum_vacuum_scale_factor = 0.05    # Clean dead tuples after 5% of rows change (default 20%)
autovacuum_vacuum_insert_scale_factor = 0.05 # Clean tables after 5% of inserts
autovacuum_vacuum_cost_delay = 1         # Sleep time limit (ms) when vacuum reaches limit
log_autovacuum_min_duration = 60000     # Log autovacuum runs taking longer than 60s

# --- System-Wide Limits ---
max_prepared_transactions = 1750        # Support up to 1750 concurrent prepared/2PC transactions
max_connections = 5000                   # Concurrent connection capacity limit

# --- Extensions preloading ---
shared_preload_libraries = 'pg_cron,pgaudit,pg_stat_statements'

# --- pg_cron Extension Settings ---
cron.database_name = 'postgres'
cron.timezone = 'Asia/Kolkata'
cron.max_running_jobs = 100

# --- Insightx / Replication Configurations ---
wal_level = replica                     # Enable CDC logical replication streams
max_replication_slots = 100             # Allow up to 100 concurrent replication stream connections
max_wal_senders = 100                   # Senders running parallel logical transactions
sync_replication_slots = true           # Sync replication slot details to prevent loss
min_wal_size = '1GB'
max_wal_size = '10GB'

# --- Point-in-Time Recovery (PITR) / Archiving ---
# NOTE: Enable these parameters MANUALLY POST-MIGRATION to avoid excessive wal accumulation and server storage exhaustion
# archive generation during initial bulk data loads.
 archive_mode = on
 archive_command = '/u01/Gsl/BLOB.sh %p %f'
 archive_timeout = 120
```

Restart the PostgreSQL service to apply the values:
```bash
sudo systemctl restart postgresql-<version>
```

---

## 6. Phase 4: PostgreSQL Extensions Installation

### 6.1 Theoretical Overview of Selected Extensions
- **pg_cron:** A cron job scheduler that runs directly inside the database, enabling `VACUUM` scheduling or operational SQL tasks.
- **pg_stat_statements:** Tracks runtime statistics of all queries run on the database (essential for database performance diagnostics).
- **pgaudit:** Provides structured logging of select operations (DML/DDL) for security auditing.

### 6.2 Step 1: Downloading & Installing packages (YUM/RPM) [it will be available with template]
Run the following commands as the `root` user to fetch matching versions of extensions from the PGDG Yum repository:

```bash
# 1. Download extension packages (Ensure version matches PostgreSQL main engine)
wget https://ftp.postgresql.org/pub/repos/yum/<version>/redhat/rhel-8-x86_64/pgaudit_<version>-17.1-1PGDG.rhel8.x86_64.rpm
wget https://ftp.postgresql.org/pub/repos/yum/<version>/redhat/rhel-8-x86_64/pg_cron_<version>-1.6.7-1PGDG.rhel8.x86_64.rpm

# 2. Local RPM package installation
sudo rpm -ivh pgaudit_<version>-17.1-1PGDG.rhel8.x86_64.rpm
sudo rpm -ivh pg_cron_<version>-1.6.7-1PGDG.rhel8.x86_64.rpm
```

### 6.3 Step 2: Creating Extensions inside Database
Log in to PostgreSQL using the `psql` console as a superuser:
```bash
psql -U postgres -d postgres
```
Run the following DDL statements to load the extensions:
```sql
-- Core extensions setup
CREATE EXTENSION DICT_INT;
CREATE EXTENSION DBLINK;
CREATE EXTENSION FUZZYSTRMATCH;
CREATE EXTENSION PG_BUFFERCACHE;
CREATE EXTENSION PG_CRON;
CREATE EXTENSION PG_STAT_STATEMENTS;
CREATE EXTENSION PGAUDIT;
CREATE EXTENSION PGCRYPTO;
CREATE EXTENSION PGSTATTUPLE;        
CREATE EXTENSION POSTGRES_FDW;

alter role postgres with password 'postgres';
```

---

## 7. Phase 5: Backup & Business Continuity (BCP) Setup

### 7.1 Theory: Backup Architecture
The BCP setup is divided into:
1. **WAL Archiving:** Copying transaction logs (WALs) continuously to Azure Blob Storage using `azcopy`.
2. **Physical Backups (`pg_basebackup`):** Generating a full physical snapshot of the database cluster on a schedule, copying it to cloud storage, and verifying backups.
3. **Log & Agent Housekeeping:** System health validation scripts and automated retention cleanups.

### 7.2 Directory Configuration [login with postgres user]
Create the operational hierarchy:
```bash
mkdir -p /u01/Gsl/SQL
mkdir -p /u01/backup/logs/arc_sync
mkdir -p /u01/backup/logs/list_delete_selfhosted
mkdir -p /u01/backup/logs/long_query
mkdir -p /u01/backup/logs/AZCOPY_DEL
mkdir -p /u01/backup/logs/MONITOR_AGENT
mkdir -p /u01/backup/arc_list
mkdir -p /u01/backup/PG
```

---

### 7.3 BCP Script Deployment [login as postgres user]

Create all scripts under the `/u01/Gsl/` directory.

#### Script 1: WAL Archiving (`/u01/Gsl/BLOB.sh`)
**Purpose:** Copy transaction log files (WAL) to Azure Blob Storage as they are filled or timeout.
```bash
vi /u01/Gsl/BLOB.sh
```
```bash
#!/bin/bash
# BLOB.sh - Archive single WAL file to Azure Storage

# --- User Configuration Variables ---
BASE_URL="https://storageerpbackupprodjiw.blob.core.windows.net/pgdbbackup"
VM_HOSTNAME="<vm_hostname>"
SAS_TOKEN="<SAS_TOKEN>"
# ------------------------------------

date_dir=$(date +%Y_%m_%d)
WAL_FILE_PATH=$1   
WAL_FILE_NAME=$2   
logfile="/u01/backup/logs/arc_sync"

BLOB_STORAGE_URL="${BASE_URL}/self-hosted/${VM_HOSTNAME}/archive/${date_dir}/${WAL_FILE_NAME}?${SAS_TOKEN}"

/usr/bin/azcopy copy "$WAL_FILE_PATH" "$BLOB_STORAGE_URL" --log-level "ERROR" --overwrite=true >> $logfile/blob_archive.log

if [ $? -ne 0 ]; then
    echo "Failed to archive WAL file to Blob Storage: $WAL_FILE_NAME" >&2 >> $logfile/blob_archive.log
    exit 1
fi
exit 0
```
```bash
chmod 777 /u01/Gsl/BLOB.sh
```

#### Script 2: Base Physical Backup (`/u01/Gsl/BASEBKP.sh`)
**Purpose:** Create a physical backup snapshot, sync files to Azure container, and send success/fail alert email.
```bash
vi /u01/Gsl/BASEBKP.sh
```
```bash
#!/bin/bash
# BASEBKP.sh - Database physical base backup

# --- User Configuration Variables ---
BASE_URL="https://storageerpbackupprodjiw.blob.core.windows.net/pgdbbackup"
VM_HOSTNAME="<vm_hostname>"
SAS_TOKEN="<SAS_TOKEN>"
PG_VERSION="16"

PGPASSWORD="postgres"
SMTP_APP_PASSWORD="oegmwnlfdtdpmvix"
ALERT_TO="dbsrvalerts@gsl.in"
ALERT_FROM="dbsrvalerts@gsl.in"
SMTP_SERVER="smtp.gmail.com:587"
# ------------------------------------

yy=$(date +%Y)
mm=$(date +%m)
dd=$(date +%d)
MYDATE="${yy}_${mm}_${dd}"
MYTIME="$(date +%H%M%S)"
DD=$(date +%d%m%y)

PGUSER="postgres"
export PGPASSWORD
export PGUSER

CMD_M=mailx
PG_BASEBACKUP_LOG="/u01/backup/logs"
Azure_Log="$PG_BASEBACKUP_LOG/GINESYS_backuplog.log"
POSTGRESQL_BIN="/usr/pgsql-${PG_VERSION}/bin"

mkdir -p "$PG_BASEBACKUP_LOG"

start_time=$(date +%s)
start_time_human=$(date '+%Y-%m-%d %H:%M:%S')

echo "Starting Pg_basebackup at $start_time_human" >> "$PG_BASEBACKUP_LOG/pg_basebackup_${MYDATE}_${MYTIME}.log"
echo "Running pg_basebackup command..." >> "$PG_BASEBACKUP_LOG/pg_basebackup_${MYDATE}_${MYTIME}.log"

$POSTGRESQL_BIN/pg_basebackup -D "/u01/backup/PG/$MYDATE" -Ft -z -P -Xs >> "$PG_BASEBACKUP_LOG/pg_basebackup_${MYDATE}_${MYTIME}.log" 2>&1

end_time=$(date +%s)
end_time_human=$(date '+%Y-%m-%d %H:%M:%S')
elapsed_time=$((end_time - start_time))
elapsed_time_human=$(date -ud @${elapsed_time} +'%H:%M:%S')

echo "Ending Pg_basebackup at $end_time_human" >> "$PG_BASEBACKUP_LOG/pg_basebackup_${MYDATE}_${MYTIME}.log"
echo "Elapsed time: $elapsed_time_human" >> "$PG_BASEBACKUP_LOG/pg_basebackup_${MYDATE}_${MYTIME}.log"

echo "---------------------------------" >> "$Azure_Log"
echo "Syncing files to Azure storage" >> "$Azure_Log"
echo "---------------------------------" >> "$Azure_Log"

sourcePath="/u01/backup/PG"
destinationUrl="${BASE_URL}/self-hosted/${VM_HOSTNAME}/fullbackup/?${SAS_TOKEN}"

/usr/bin/azcopy sync "$sourcePath" "$destinationUrl" --put-md5 --delete-destination=false >> "$Azure_Log" 2>&1

echo "${VM_HOSTNAME} STANDALONE: PG BASE BACKUP" | $CMD_M -a $PG_BASEBACKUP_LOG/pg_basebackup_${MYDATE}_${MYTIME}.log -v -r "$ALERT_FROM" -s "${VM_HOSTNAME} STANDALONE: PG BASEBACKUP INFORMATION ${DD}" -S smtp="$SMTP_SERVER" -S smtp-use-starttls -S smtp-auth=login -S smtp-auth-user="$ALERT_FROM" -S smtp-auth-password="$SMTP_APP_PASSWORD" -S ssl-verify=ignore $ALERT_TO
```
```bash
chmod 777 /u01/Gsl/BASEBKP.sh
```

#### Script 3: Old Backup Deletion (`/u01/Gsl/DELBASEBKP.sh`)
**Purpose:** Delete local backups older than 24 hours (1440 mins) to prevent disk space exhaustion.
```bash
vi /u01/Gsl/DELBASEBKP.sh
```
```bash
#!/bin/bash
# DELBASEBKP.sh - Housekeeping backup deletion script

# --- User Configuration Variables ---
VM_HOSTNAME="<vm_hostname>"
SMTP_APP_PASSWORD="oegmwnlfdtdpmvix"
ALERT_TO="dbsrvalerts@gsl.in"
ALERT_FROM="dbsrvalerts@gsl.in"
SMTP_SERVER="smtp.gmail.com:587"
# ------------------------------------

CMD_M=mailx
DD=$(date +%d%m%y)
export LOG_DIR=/u01/backup/logs/list_delete_selfhosted
export LOG_FILE=$LOG_DIR/list_delete_selfhosted_$(date +%Y-%m-%d_%H_%M).log

cd /u01/backup/PG

echo "/backup mountpoint size before deletion" > ${LOG_FILE}
df -h /u01 >> ${LOG_FILE}
echo "" >> ${LOG_FILE}

echo "THE LIST OF THE BACKUP FILES THAT ARE DELETED" >> ${LOG_FILE}
echo "*********************************************" >> ${LOG_FILE}
find /u01/backup/PG/* -type d -empty -delete
find /u01/backup/PG/* -mmin +1440 >> ${LOG_FILE}
find /u01/backup/PG/* -mmin +1440 -exec rm -rf {} \;

echo "" >> ${LOG_FILE}
echo "/backup mountpoint size after deletion" >> ${LOG_FILE}
df -h /u01 >> ${LOG_FILE}

if [ $? -ne 0 ]; then
  echo "Error: file delete did not complete."
  exit 1
else
  echo "Success."
  echo "${VM_HOSTNAME} STANDALONE: OLD PG BASE BACKUP DELETION" | $CMD_M -a $LOG_FILE -v -r "$ALERT_FROM" -s "${VM_HOSTNAME} STANDALONE: OLD PG BASEBACKUP DELETION INFORMATION ${DD}" -S smtp="$SMTP_SERVER" -S smtp-use-starttls -S smtp-auth=login -S smtp-auth-user="$ALERT_FROM" -S smtp-auth-password="$SMTP_APP_PASSWORD" -S ssl-verify=ignore $ALERT_TO
fi
```
```bash
chmod 777 /u01/Gsl/DELBASEBKP.sh
```

#### Script 4: Remote Archive Verification (`/u01/Gsl/ARCNAME.sh`)
**Purpose:** Verify files stored in Azure against internal Postgres database state using `pg_split_walfile_name`.
```bash
vi /u01/Gsl/ARCNAME.sh
```
```bash
#!/bin/bash
# ARCNAME.sh - Compare remote storage logs with database segment logs

# --- User Configuration Variables ---
BASE_URL="https://storageerpbackupprodjiw.blob.core.windows.net/pgdbbackup"
VM_HOSTNAME="<vm_hostname>"
SAS_TOKEN="<SAS_TOKEN>"
# ------------------------------------

date_dir=$(date +%Y_%m_%d)
log_file="/u01/backup/arc_list/arcname_${date_dir}.log"
AZ_LIST_URL="${BASE_URL}/self-hosted/${VM_HOSTNAME}/archive/${date_dir}?${SAS_TOKEN}"

files=$(/usr/bin/azcopy list "$AZ_LIST_URL" | cut -d';' -f1)

while IFS= read -r file; do
    if [[ -z "$file" ]]; then
        continue
    fi

    if grep -q "$file" "$log_file"; then
        continue
    fi

    pg_command="select pg_split_walfile_name('$file');"
    output=$(psql -U postgres -d postgres -t -c "$pg_command")

    file_creation_time_utc=$(date -u '+%Y-%m-%d %H:%M:%S')
    file_creation_time_ist=$(date -d "$file_creation_time_utc UTC +5 hours 30 minutes" '+%Y-%m-%d %H:%M:%S')

    echo "[$file_creation_time_utc UTC | $file_creation_time_ist IST] $file || $output" >> "$log_file"
done <<< "$files"

AZ_COPY_URL="${BASE_URL}/self-hosted/${VM_HOSTNAME}/archive/${date_dir}/?${SAS_TOKEN}"
/usr/bin/azcopy copy "$log_file" "$AZ_COPY_URL"
```
```bash
chmod 777 /u01/Gsl/ARCNAME.sh
```

#### Script 5: Long Query & System Performance Analysis (`/u01/Gsl/LONG_QUERY.sh`)
**Purpose:** Record daily active SQL operations, OS load averages, and disk metrics, and email details to the admin team.
```bash
vi /u01/Gsl/LONG_QUERY.sh
```
```bash
#!/bin/bash
# LONG_QUERY.sh - System diagnostics and active SQL monitor

# --- User Configuration Variables ---
VM_HOSTNAME="<vm_hostname>"
PGPASSWORD="<postgres_user_password>"

SMTP_APP_PASSWORD="dbsrvalerts@gsl.in"
ALERT_TO="dbsrvalerts@gsl.in"
ALERT_FROM="dbsrvalerts@gsl.in"
SMTP_SERVER="smtp.gmail.com:587"
# ------------------------------------

export PGPASSWORD

yy=$(date +%Y)
mm=$(date +%m)
dd=$(date +%d)
MYDATE="${yy}_${mm}_${dd}"
MYTIME="$(date +%H%M%S)"
LOGFILE="/u01/backup/logs/long_query/Long_Query_${MYDATE}_${MYTIME}.log"
DD=$(date +%d%m%y)
CMD_M=mailx

echo "PRODUCTION SERVER DISK SPACE START
***********************************************************************************************************************" >> ${LOGFILE}
df -h >> $LOGFILE
echo "PRODUCTION SERVER DISK SPACE END
***********************************************************************************************************************" >> ${LOGFILE}

echo "SYSTEM TOP LOAD PROCESSES
***********************************************************************************************************************" >> ${LOGFILE}
top -b -n 1 | head -n 20 >> $LOGFILE

psql -h localhost -U postgres -d postgres -f /u01/Gsl/SQL/daily_query.sql >> "$LOGFILE" 2>&1

echo "${VM_HOSTNAME} STANDALONE: PG LONG QUERY INFO" | $CMD_M -a $LOGFILE -v -r "$ALERT_FROM" -s "${VM_HOSTNAME} STANDALONE: PG LONG QUERY INFORMATION ${DD}" -S smtp="$SMTP_SERVER" -S smtp-use-starttls -S smtp-auth=login -S smtp-auth-user="$ALERT_FROM" -S smtp-auth-password="$SMTP_APP_PASSWORD" -S ssl-verify=ignore $ALERT_TO

exit 0
```
*(Ensure that `/u01/Gsl/SQL/daily_query.sql` exists and contains performance checking SQL statements).*
```bash
chmod 777 /u01/Gsl/LONG_QUERY.sh
```

#### Script 6: AzCopy Execution Logs Purge (`/u01/Gsl/DEL_AZCOPY_LOG.sh`)
**Purpose:** Remove historical log output created by `azcopy` runs older than 1 day to prevent file index exhaustion inside `/var/lib/pgsql/.azcopy/`.
```bash
vi /u01/Gsl/DEL_AZCOPY_LOG.sh
```
```bash
#!/bin/bash
# DEL_AZCOPY_LOG.sh - Clean up AzCopy run directory logs

export LOG_DIR=/u01/backup/logs/AZCOPY_DEL
export LOG_FILE=$LOG_DIR/list_delete_azcopy_logs_$(date +%Y-%m-%d_%H_%M).log

SEARCH_DIR="/var/lib/pgsql/.azcopy/"

echo "/.azcopy size before deletion" > ${LOG_FILE}
du -sh $SEARCH_DIR >> ${LOG_FILE}

find $SEARCH_DIR -type f -mtime +1 -exec rm -f {} \; -exec echo "Removed file: {}" >> $LOG_FILE \;

echo "/.azcopy size after deletion" >> ${LOG_FILE}
du -sh $SEARCH_DIR >> ${LOG_FILE}

echo "File listing completed and saved to $LOG_FILE"
```
```bash
chmod 777 /u01/Gsl/DEL_AZCOPY_LOG.sh
```

#### Script 7: Azure Monitor Agent Log Purge (`/u01/Gsl/DEL_MONITOR_AGENT.sh`)
**Purpose:** Purge event metrics (.sst) generated by Azure Monitor Agent older than 14 days.
```bash
vi /u01/Gsl/DEL_MONITOR_AGENT.sh
```
```bash
#!/bin/bash
# DEL_MONITOR_AGENT.sh - Clean old SST telemetry files

LOG_DIR="/u01/backup/logs/MONITOR_AGENT"
mkdir -p $LOG_DIR
LOG_FILE=$LOG_DIR/list_delete_sst_logs_$(date +%Y-%m-%d_%H_%M).log
SEARCH_DIR="/var/opt/microsoft/azuremonitoragent/events"

{
echo "===================================================="
echo " SST CLEANUP SCRIPT LOG - $(date)"
echo "===================================================="
echo "[1] $SEARCH_DIR size BEFORE deletion:"
du -sh "$SEARCH_DIR"
echo ""
echo "[2] Files to be deleted (older than 14 days):"
find "$SEARCH_DIR" -type f -name "*.sst" -mtime +14 -path "*/c*/*" -ls
} >> "$LOG_FILE"

find "$SEARCH_DIR" -type f -name "*.sst" -mtime +14 -path "*/c*/*" -exec rm -f {} \; >> "$LOG_FILE" 2>&1

{
echo "[3] $SEARCH_DIR size AFTER deletion:"
du -sh "$SEARCH_DIR"
echo ""
echo "Cleanup completed at $(date)"
echo "===================================================="
} >> "$LOG_FILE"
```
```bash
sudo chmod +x /u01/Gsl/DEL_MONITOR_AGENT.sh
```

---

## 8. Phase 6: Automated Operations Setup (Cron Jobs)

We configure the cron scheduler to trigger our scripts.

### 8.1 Postgres User Crontab Setup
Open the postgres user crontab editor:
```bash
sudo -u postgres crontab -e
```
Insert the following schedule entries (uncomment when deployment goes active):
```cron
# Trigger database backup every evening at 18:35 (6:35 PM)
35 18 * * * sh /u01/Gsl/BASEBKP.sh

# Run local backup housekeeping cleanups at 18:31 (6:31 PM)
31 18 * * * sh /u01/Gsl/DELBASEBKP.sh

# Run active long query diagnostics execution every 30 minutes
*/30 * * * * sh /u01/Gsl/LONG_QUERY.sh

# Run verification of remote archived files against postgres database state every minute
*/1 * * * * sh /u01/Gsl/ARCNAME.sh

# Purge old AzCopy logs every night at 20:00 (8:00 PM)
00 20 * * * sh /u01/Gsl/DEL_AZCOPY_LOG.sh

# Track storage and disk spaces every 30 minutes
*/30 * * * * sh /u01/Gsl/Disk_usage.sh
```

### 8.2 Root User Crontab Setup
Open the root user crontab editor:
```bash
sudo crontab -e
```
Insert the following scheduling policy:
```cron
# Clean Azure Monitor Agent events folder every evening at 20:05 (8:05 PM)
05 20 * * * sh /u01/Gsl/DEL_MONITOR_AGENT.sh
```

---

## 9. Phase 7: Exporter & Monitoring Service Verification

Since monitoring agents are pre-installed via the initial VM template, this phase verifies that the necessary telemetry services are active and running correctly before handing over to operations.

### 9.1 Verify Node Exporter
Ensure the Node Exporter service (for OS-level metrics) is active and listening:
```bash
sudo systemctl status node_exporter
# Output should show "active (running)"
```

### 9.2 Verify PostgreSQL Exporter
Ensure the PostgreSQL Exporter service (for database-level metrics) is active:
```bash
sudo systemctl status postgres_exporter
# Output should show "active (running)"
```

### 9.3 Verify Prometheus Service
Ensure the Prometheus time-series database is active:
```bash
sudo systemctl status prometheus
# Output should show "active (running)"
```

---

## 10. Phase 8: Post-Migration WAL Archiving Activation

### 10.1 Theory: Point In Time Recovery (PITR)
To achieve data durability, transaction records (WAL files) must be continuously archived. By setting `archive_mode = on` and configuring `archive_command` to execute our `BLOB.sh` script, PostgreSQL ensures every transaction log segment is copied to remote cloud storage before it is recycled locally. This enables recovering the database up to the exact millisecond of a failure.

### 10.2 Activation Process (Post Oracle Migration)
1. Complete migration data verification checkouts.
2. Edit `postgresql.conf`:
   ```bash
   sudo vi $PGDATA/postgresql.conf
   ```
3. Update the parameters to enable the pipeline:
   ```ini
   archive_mode = on
   archive_command = '/u01/Gsl/BLOB.sh %p %f'
   archive_timeout = 120
   ```
4. Restart the database engine to start logging to Azure Storage:
   ```bash
   sudo systemctl restart postgresql-<version>
   ```
5. Verify archives are successfully transferring to cloud container storage by checking:
   - `/u01/backup/logs/arc_sync/blob_archive.log`
   - Remote Azure Storage container path `/archive/`.

---
## Additional Configuration
> *   **PGGDBA Configuration:** To configure the `pggdba` database for tracking historical size and growth data, please refer to the independent **PGGDBA Runbook Documentation**.
> *   **Grafana Configuration:** To configure Grafana for live monitoring of the new instance, please refer to the independent **Grafana Dashboard Runbook Documentation**.



## 11. Phase 9: Setting up Physical Streaming Replication after go-Live [OPTIONAL]

### 11.1 Streaming Replication Theory
- WAL record chunks are streamed by database servers to keep data in sync.
- The standby server connects to the master to receive the WAL chunks.
- The WAL records are streamed as they are generated.
- The streaming of WAL records need not wait for the WAL file to be filled.
- This allows a standby server to stay more up-to-date than is possible with file-based log shipping.
- By default, streaming replication is asynchronous even though it also supports synchronous replication.

### 11.2 Replication Parameters

| Parameter             | Value       | Description                                                                                  |
|-----------------------|-------------|----------------------------------------------------------------------------------------------|
| `wal_level`           | `replica`   | Level of info written to WAL; `replica` supports archiving and read-only queries on standby. |
| `wal_log_hints`       | `on`        | Required for `pg_rewind` capability when standby goes out of sync with master.               |
| `max_wal_senders`     | `integer`   | Max concurrent running WAL sender processes (default 10; 0 disables replication).            |
| `wal_keep_segments`   | `integer`   | Min number of past log file segments kept in the `pg_wal` directory for standby fetch.       |
| `hot_standby`         | `on`        | Enables read-only connection on the node when it is in standby role.                         |

### 11.3 Replication Slots
- A replication slot is used to retain the WAL files when the standby goes offline or disconnected.
- The master server uses replication slots to keep track of how much the standby lags and retain the WAL files it needs until the standby reconnects again.
- Replication slots came in with PostgreSQL 9.4, before that the `wal_keep_segments` parameter used to govern how many wal files need to be maintained.
- Replication slots have to be created manually and the default value is 10.
- PostgreSQL Replication slots are of two types:
  - Physical replication slots
  - Logical replication slots

### 11.4 Streaming Replication Setup Steps

**On Master Node:**
**Step 1:** Setup the following postgres config in `postgresql.conf`:
```ini
listen_addresses = '*'
wal_level = replica
hot_standby = on
```

**Step 2:** Create or alter a user to use replication encrypted password:
```sql
ALTER USER postgres WITH REPLICATION ENCRYPTED PASSWORD '<password>';
```

**Step 3:** Modify `pg_hba.conf` to allow replication connections. Add the IP addresses of both master and standby with the `md5` method:
```text
host    replication    postgres    <primary_ip>/32      md5
host    replication    postgres    <standby_ip>/32      md5
```

**Step 4:** Reload/restart the configuration.
```bash
sudo systemctl reload postgresql-<version>
```

**On Standby Node:**
**Step 5:** Stop the postgresql service on the standby node.
```bash
sudo systemctl stop postgresql-<version>
```

**Step 6:** Delete all the files under the data directory.
```bash
rm -rf /u01/pgsql/<version>/data/*
```

**Step 7:** Run `pg_basebackup` to clone the standby instance:
```bash
pg_basebackup -h <primary_ip> -U postgres -p 5432 -D /u01/pgsql/<version>/data -Fp -Xs -P -R -C -S pgstandby
```
> **Note:** Here `-S pgstandby` represents a replication slot.

**Step 8:** Start the postgresql service on the standby node.
```bash
sudo systemctl start postgresql-<version>
```

**Step 9:** Test replication by creating a table in primary and checking standby.

### 11.5 Monitoring Primary and Standby Streaming Replication

**On Primary:**
- Check stream replication slots information:
```sql
SELECT * FROM pg_replication_slots;
```

- Check PG replication status:
```sql
SELECT * FROM pg_stat_replication;
```
> **Note:** 0 lag indicates no gap between primary and secondary.

**On Standby:**
- Check if secondary cluster is in recovery mode or not:
```sql
SELECT pg_is_in_recovery();
```
