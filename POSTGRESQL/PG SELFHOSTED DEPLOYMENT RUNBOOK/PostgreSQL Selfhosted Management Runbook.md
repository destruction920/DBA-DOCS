# Executive Runbook: Self-Hosted PostgreSQL Deployment

## 1. Executive Summary
This document outlines the high-level workflow, strategic decisions, and Business Continuity Planning (BCP) implemented for the self-hosted PostgreSQL database cluster. It is designed for management and architectural review, omitting the low-level shell scripts, code, and terminal commands found in the technical runbook.

The deployment leverages a self-hosted architecture on an Azure Red Hat Enterprise Linux (RHEL) 8 Virtual Machine to ensure maximum performance isolation, deeper hardware control, and custom Business Continuity pipelines.


## 2. Strategic Rationale: Why Self-Hosted?
While managed cloud databases (like Azure Flexible Server) offer convenience, our strategic requirements dictate a self-hosted approach:
- **Public IP & Accessibility:** Accommodates tools like Qlik which require public IP endpoints (Azure PaaS typically restricts databases to private IPs).
- **Direct Read Replica Access:** Fulfills strict customer demands for direct access to read replicas.
- **Hardware & Performance Isolation:** Ensures large databases are isolated on dedicated VMs to prevent "noisy neighbor" latency.
- **Cost Efficiency (IaaS vs. PaaS):** Leveraging an Infrastructure-as-a-Service (IaaS) model to manage a self-hosted VM is significantly more cost-effective than equivalent fully managed Platform-as-a-Service (PaaS) tiers.


## 3. Deployment Flow & Phases

### Phase 1: Infrastructure & Operating System Configuration
**Objective:** Prepare the underlying server for high-performance database operations.
- **Storage Expansion:** Disks are managed using Logical Volume Management (LVM), allowing storage to be dynamically expanded on-the-fly without system downtime.
- **Resource Allocation Check:** CPU, RAM, and Disk allocations are audited before any configuration to ensure they meet the projected database sizing.
- **Process Isolation:** Legacy default cron jobs [Linux Schedulers] are suspended to ensure a clean deployment slate.

### Phase 2: Automated Resource Tuning (Auto-Tuning)
**Objective:** Ensure the database always utilizes optimal hardware resources.
- PostgreSQL natively does not automatically scale its memory parameters if the VM's RAM or CPU is upgraded. 
- To solve this, a custom **Auto-Tuning Hook** is integrated into the server's boot sequence. Every time the server starts, it checks the currently allocated RAM and CPU, calculates optimal memory limits (e.g., `shared_buffers`, `work_mem`), and reconfigures the database dynamically before it comes online.

### Phase 3: Engine Parameter Tuning & Optimization
**Objective:** Configure the PostgreSQL engine for optimal workloads.
- **Performance Tuning:** Core database parameters (such as memory caching and processing limits) are customized to ensure fast, consistent response times for high-concurrency enterprise workloads.
- **Maintenance Automation:** Background cleanup processes are optimized to maintain long-term database health and prevent performance degradation without manual intervention.
- **Data Integration Preparedness:** The database is pre-configured to easily and securely stream real-time data changes to downstream analytics tools.

### Phase 4: Operational & Application Extensions
**Objective:** Provision necessary database plugins to support security auditing, performance monitoring, and application functionality.
- **Security & Performance Monitoring:** Core extensions (such as `pgaudit`, `pg_stat_statements`, and `pg_cron`) are configured to enforce strict compliance logging, track real-time query execution metrics, and automate internal maintenance jobs.
- **Application Functionality:** A standardized suite of application-facing extensions (including cryptographic functions, foreign-data wrappers, and advanced string matching) are installed and created to fulfill all application dependencies, as detailed in the technical deployment runbook.


## 4. Business Continuity & Disaster Recovery (BCP)

The disaster recovery strategy is designed to ensure near zero data loss and rapid recovery capability. All backups are stored locally for immediate access and continuously synced to an offsite Azure Blob Storage container. To facilitate this, custom automated DBA scripts are prepared and deployed to autonomously manage all BCP, monitoring, backups, and storage purging tasks.

### 4.1 Backup Architecture
- **Continuous Archiving (Point-in-Time Recovery):** A dedicated WAL pipeline continuously streams transaction logs to Azure Blob Storage, enabling database restoration to the exact minute of failure. *(Note: PITR is manually enabled post-migration to avoid archiving massive initial migration data).*
- **Full Physical Backups:** A complete daily backup is taken automatically to serve as a solid restoration baseline.

### 4.2 Automated Housekeeping
- **Operational Automation:** Automated housekeeping scripts are deployed to the newly provisioned VM to autonomously manage all routine DBA tasks.

### 4.3 High Availability via Physical Streaming Replication
- **Real-Time Redundancy:** To ensure high availability (HA) and rapid failover capabilities, a physical standby (replica) node is configured to synchronize with the primary database.
- **Continuous Syncing:** The architecture utilizes Physical Streaming Replication to stream Write-Ahead Log (WAL) records in real-time. This guarantees that the standby remains continuously up-to-date, drastically minimizing potential data loss (RPO) compared to traditional batch log shipping.

### 4.4 Recovery Objectives (RTO & RPO)
Based on the implemented business continuity architecture, the targeted recovery metrics are defined below:

| Architecture Model                                         | Recovery Point Objective (RPO)           | Recovery Time Objective (RTO)       | Failover Type         |
|------------------------------------------------------------|------------------------------------------|-------------------------------------|-----------------------|
| **Self-Hosted PostgreSQL (without Streaming Replication)** | 5 Minutes (via continuous WAL archiving) | 1 - 3 Hours                         | Restore + PITR        |
| **Self-Hosted PostgreSQL (with Streaming Replication)**    |        Near-Zero                         | 5 - 15 Minutes (Ideal Conditions)   | Manual Promotion      |


## 5. Observability & Monitoring

### 5.1 Monitoring & Telemetry Stack
- **Proactive Observability:** The monitoring stack is fully configured using Node Exporter, PostgreSQL Exporter, SQL Exporter, Prometheus and Grafana. Additional establishment includes: Heartbeat mechanisms , alerting logs and custom scripting deployment for proactive system monitoring.

### 5.2 Historical Data Archiving
- **PGGDBA Configuration:** A dedicated local database (`pggdba`) is provisioned to capture, store, and analyze historical data regarding database growth, size, and operational trends.


## 6. Work Governance & Ownership Matrix

To ensure clear accountability and seamless hand-offs during the deployment lifecycle, the following matrix defines the ownership and operational responsibilities for each implementation phase.

| Implementation Phase                         | Primary Owner      | Key Responsibilities                                                                                                            |
|----------------------------------------------|--------------------|---------------------------------------------------------------------------------------------------------------------------------|
| **Phase 1: Clone Template Provisioning**     | Cloud Team         | Provisioning and cloning the initial RHEL 8 VM template, assigning network IPs, and attaching secondary data disks.             |
| **Phase 2: OS Host & Storage Config**        | DBA Team           | Performing OS host changes and extending LVM partitions.                                                                        |
| **Phase 3: PostgreSQL Autotune Setup**       | DBA Team           | Deploying the systemd startup hooks to allow the database to dynamically adjust memory limits based on allocated VM hardware.   |
| **Phase 4: Engine Parameter Tuning**         | DBA Team           | Configuring core PostgreSQL parameters for optimized memory caching, parallel workers, and autovacuum limits.                   |
| **Phase 5: Extensions**                      | DBA Team           | Downloading and installing Linux RPM packages for PostgreSQL extensions.                                                        |
| **Phase 6: Business Continuity (BCP)**       | DBA Team           | Deploying backup scripts, setting up Azure Storage integration, and configuring local log housekeeping.                         |
| **Phase 7: Operations & Monitoring**         | DBA Team           | Configuring Prometheus, Grafana, and Exporters; setting up daily diagnostic panels and Heartbeat alerting mechanisms.           |
| **Phase 8: PGGDBA Configuration**            | DBA Team           | Configuring a local database (`pggdba`) to store historical archiving data for database size, growth, and object tracking.      |
| **Phase 9: Service Verification**            | DBA Team           | Verifying configurations, parameters, database availability, and ensuring all Exporter and Prometheus services are running.     |
| **Phase 10: HA Replica VM Provisioning**     | Cloud Team         | *(optional)* Provisioning and cloning the secondary RHEL 8 VM template for the replica server.                                  |
| **Phase 11: HA Replication Configuration**   | DBA Team           | *(optional)* Configuring physical streaming replication for continuous syncing between primary and standby nodes.               |


> **Note:** For comprehensive technical details and command-line instructions regarding the execution of each phase, please refer to the **PostgreSQL Selfhosted Technical Deployment Runbook**.