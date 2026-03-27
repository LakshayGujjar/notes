# 🗄️ GCP Cloud SQL — Comprehensive Cheatsheet

> **Audience:** Developers & Cloud Engineers | **Last updated:** 2025
> A single-reference document covering theory, configuration, CLI, SDK, and best practices.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Database Engine Support & Editions](#2-database-engine-support--editions)
3. [Instance Lifecycle & Configuration](#3-instance-lifecycle--configuration)
4. [Networking & Connectivity](#4-networking--connectivity)
5. [High Availability & Failover](#5-high-availability--failover)
6. [Read Replicas & Cross-Region Replicas](#6-read-replicas--cross-region-replicas)
7. [Backups & Recovery](#7-backups--recovery)
8. [Security](#8-security)
9. [Performance & Flags](#9-performance--flags)
10. [Client Library Connections — Python](#10-client-library-connections--python)
11. [gcloud CLI Quick Reference](#11-gcloud-cli-quick-reference)
12. [Pricing Model Summary](#12-pricing-model-summary)
13. [Common Errors & Troubleshooting](#13-common-errors--troubleshooting)

---

## 1. Overview & Key Concepts

**Google Cloud SQL** is a fully managed relational database service supporting MySQL, PostgreSQL, and SQL Server. Google handles provisioning, patching, backups, replication, and failover — letting teams focus on data rather than database operations.

### Cloud SQL vs Other GCP Database Options

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                      GCP Relational Database Landscape                       │
├─────────────────┬──────────────┬─────────────┬──────────────┬───────────────┤
│   Cloud SQL     │  AlloyDB     │Cloud Spanner │  GCE + DB    │ Bare Metal    │
├─────────────────┼──────────────┼─────────────┼──────────────┼───────────────┤
│ Managed MySQL / │ Managed PG   │ Managed      │ Self-managed │ Self-managed  │
│ PG / SQL Server │ (compatible) │ NewSQL       │ any DB       │ Oracle/SAP    │
│ Regional HA     │ Regional HA  │ Global HA    │ Manual       │ Manual        │
│ Up to 96 vCPU   │ Up to 128vCP │ Unlimited    │ Any size     │ Physical HW   │
│ Standard OLTP   │ HTAP/OLTP    │ Globally     │ Full control │ Lift & shift  │
│                 │              │ distributed  │              │               │
│ $ – $$$         │ $$$ – $$$$   │ $$$$ – $$$$$ │ $$           │ $$$$$         │
└─────────────────┴──────────────┴─────────────┴──────────────┴───────────────┘
```

### When to Use Cloud SQL

- Traditional OLTP workloads requiring MySQL, PostgreSQL, or SQL Server compatibility
- Applications migrating from on-premises relational databases to GCP
- Workloads needing managed HA, backups, and patching without operational overhead
- Multi-region read scaling via cross-region replicas

### Core Terminology

| Term | Description |
|---|---|
| **Instance** | A Cloud SQL database server. The top-level resource (VM + storage + config). |
| **Database** | A logical database within an instance (e.g., `myapp_db` inside the instance). |
| **User** | A database-level user account. Separate from GCP IAM users. |
| **Connection Name** | Unique identifier: `PROJECT:REGION:INSTANCE`. Used by Auth Proxy and connectors. |
| **Tier / Machine Type** | The CPU and RAM spec of the instance (e.g., `db-n1-standard-4`). |
| **Edition** | `Enterprise` (standard) or `Enterprise Plus` (higher perf, more features). |
| **Private IP** | Instance accessible only via VPC (via Private Services Access). Recommended. |
| **Public IP** | Instance accessible over the internet. Requires authorized networks or proxy. |
| **Authorized Network** | An IP CIDR range explicitly allowed to connect to the instance's public IP. |
| **Cloud SQL Auth Proxy** | A local proxy binary that handles IAM auth and TLS for Cloud SQL connections. |
| **Cloud SQL Connector** | Language SDK (Python, Java, Go) that embeds Auth Proxy logic directly in code. |
| **HA** | High Availability — standby instance in a secondary zone; auto-failover. |
| **Read Replica** | A read-only copy of the primary instance for read scaling or DR. |
| **Maintenance Window** | Scheduled window during which Cloud SQL applies patches/updates. |
| **Flag** | A database engine configuration parameter applied to the instance. |
| **PITR** | Point-In-Time Recovery — restore to any second within the retention window. |
| **Storage Auto-resize** | Automatically increases disk size when usage exceeds 95% capacity. |
| **Deletion Protection** | Prevents accidental instance deletion via gcloud or Console. |

---

## 2. Database Engine Support & Editions

### Supported Engines & Versions

| Property | MySQL | PostgreSQL | SQL Server |
|---|---|---|---|
| **Supported Versions** | 5.7, 8.0, 8.4 | 13, 14, 15, 16 | 2017, 2019, 2022 |
| **Max Storage (SSD)** | 64 TB | 64 TB | 64 TB |
| **Max vCPUs** | 96 | 96 | 96 |
| **Max RAM** | 624 GB | 624 GB | 624 GB |
| **Max Connections** | ~4,000 (varies by RAM) | ~1,000 (varies by RAM) | 32,767 |
| **HA Model** | Regional (primary + standby) | Regional (primary + standby) | Regional (primary + standby) |
| **PITR Mechanism** | Binary log | WAL archiving | Transaction log backup |
| **Extensions** | Plugins (limited) | 100+ extensions | N/A (SQL Server features) |
| **Notable Extensions/Plugins** | `audit_log`, `validate_password` | `pg_stat_statements`, `postgis`, `pgvector`, `uuid-ossp` | Full-text search, CLR (limited) |
| **IAM Auth** | ✅ | ✅ | ✗ |
| **Max DB per Instance** | Unlimited | Unlimited | 100 |

### Enterprise vs Enterprise Plus

| Feature | Enterprise | Enterprise Plus |
|---|---|---|
| **vCPU range** | 1–96 | 2–96 |
| **Max RAM** | 624 GB | 624 GB |
| **SSD storage** | Up to 64 TB | Up to 64 TB |
| **Data cache (in-memory)** | ✗ | ✅ — dedicated cache tier |
| **PITR retention** | Up to 7 days | Up to 35 days |
| **Maintenance window** | ~1 hour | < 10 seconds (near-zero downtime) |
| **Read replicas** | ✅ | ✅ |
| **Zonal failover RTO** | ~60 seconds | ~30 seconds |
| **Customer-Managed Encryption (CMEK)** | ✅ | ✅ |
| **Relative cost** | $ | $$ |
| **Best for** | Dev, test, standard production | Mission-critical, latency-sensitive production |

> **Tip:** Use **Enterprise Plus** for production workloads that cannot tolerate maintenance downtime or need PITR retention beyond 7 days. The near-zero downtime maintenance is the most impactful feature for production SLAs.

---

## 3. Instance Lifecycle & Configuration

### Creating an Instance

```bash
# PostgreSQL 15 — private IP, 4 vCPU, SSD, HA enabled
gcloud sql instances create my-pg-instance \
  --database-version=POSTGRES_15 \
  --tier=db-custom-4-15360 \
  --region=us-central1 \
  --availability-type=REGIONAL \
  --storage-type=SSD \
  --storage-size=100GB \
  --storage-auto-increase \
  --no-assign-ip \
  --network=projects/my-project/global/networks/my-vpc \
  --deletion-protection \
  --edition=ENTERPRISE \
  --backup \
  --backup-start-time=02:00 \
  --retained-backups-count=7 \
  --enable-point-in-time-recovery

# MySQL 8.0 — public IP with authorized network
gcloud sql instances create my-mysql-instance \
  --database-version=MYSQL_8_0 \
  --tier=db-n1-standard-4 \
  --region=us-central1 \
  --storage-type=SSD \
  --storage-size=50GB \
  --storage-auto-increase \
  --authorized-networks=203.0.113.0/24 \
  --root-password=MyRootPass123!

# SQL Server 2022 Enterprise
gcloud sql instances create my-sqlserver-instance \
  --database-version=SQLSERVER_2022_ENTERPRISE \
  --tier=db-custom-8-30720 \
  --region=europe-west1 \
  --storage-type=SSD \
  --storage-size=200GB \
  --root-password=MyAdminPass123!
```

### Machine Type Naming

```
db-custom-VCPUS-MEMORY_MB    # Custom (recommended)
  e.g. db-custom-4-15360     # 4 vCPU, 15 GB RAM

db-n1-standard-N             # Standard (legacy)
db-n1-highmem-N              # High memory (legacy)
db-f1-micro                  # Shared core (dev only)
db-g1-small                  # Shared core (dev only)
```

### Updating an Instance

```bash
# Resize CPU and RAM
gcloud sql instances patch my-pg-instance \
  --tier=db-custom-8-30720

# Enable storage auto-resize
gcloud sql instances patch my-pg-instance \
  --storage-auto-increase

# Manually increase storage (cannot decrease)
gcloud sql instances patch my-pg-instance \
  --storage-size=200GB

# Change maintenance window to Sunday 3 AM
gcloud sql instances patch my-pg-instance \
  --maintenance-window-day=SUN \
  --maintenance-window-hour=3

# Enable deletion protection
gcloud sql instances patch my-pg-instance \
  --deletion-protection

# Disable deletion protection (before deleting)
gcloud sql instances patch my-pg-instance \
  --no-deletion-protection
```

### Cloning an Instance

```bash
# Clone to a new instance (same region, same data at this moment)
gcloud sql instances clone my-pg-instance my-pg-clone \
  --point-in-time=2025-01-15T10:00:00.000Z  # optional PITR clone
```

> **Tip:** Cloning is the fastest way to create a staging environment from production data. The clone is independent — changes to the clone do not affect the original.

### Restarting and Deleting

```bash
# Restart an instance (brief downtime ~30–60 seconds)
gcloud sql instances restart my-pg-instance

# Delete an instance (irreversible — remove deletion protection first)
gcloud sql instances patch my-pg-instance --no-deletion-protection
gcloud sql instances delete my-pg-instance
```

> ⚠️ **Warning:** Deleting an instance **permanently deletes all databases, users, and automated backups** associated with it. On-demand backups are retained. Always verify you have a recent backup before deletion.

---

## 4. Networking & Connectivity

### Public IP vs Private IP

| Property | Public IP | Private IP |
|---|---|---|
| **Accessibility** | Internet-reachable | VPC-only (Private Services Access) |
| **Access control** | Authorized networks (IP allowlist) + SSL | VPC firewall rules + IAM |
| **Auth Proxy required** | Recommended (replaces IP allowlist) | Recommended (for IAM auth and TLS) |
| **Latency** | Higher (internet round-trip) | Lower (internal Google network) |
| **Security posture** | Lower (exposed surface) | Higher (no public endpoint) |
| **Recommendation** | Dev/test only | ✅ Production |

### Setting Up Private IP (Private Services Access)

```bash
# Step 1: Allocate IP range for PSA
gcloud compute addresses create google-managed-services-my-vpc \
  --global \
  --purpose=VPC_PEERING \
  --prefix-length=24 \
  --network=my-vpc

# Step 2: Create VPC peering for service networking
gcloud services vpc-peerings connect \
  --service=servicenetworking.googleapis.com \
  --ranges=google-managed-services-my-vpc \
  --network=my-vpc

# Step 3: Create instance with private IP only
gcloud sql instances create my-pg-instance \
  --database-version=POSTGRES_15 \
  --tier=db-custom-4-15360 \
  --region=us-central1 \
  --no-assign-ip \
  --network=my-vpc
```

### Cloud SQL Auth Proxy

The Auth Proxy authenticates via IAM, encrypts connections with TLS, and eliminates the need for IP allowlisting.

```
Application
    │
    │ connects to localhost:5432
    ▼
Cloud SQL Auth Proxy (local process)
    │
    │ IAM-authenticated, TLS-encrypted tunnel
    ▼
Cloud SQL Instance
```

```bash
# Download the proxy binary
curl -o cloud-sql-proxy \
  https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.14.1/cloud-sql-proxy.linux.amd64
chmod +x cloud-sql-proxy

# Run the proxy for a PostgreSQL instance (TCP mode)
./cloud-sql-proxy \
  --port=5432 \
  my-project:us-central1:my-pg-instance

# Run with Unix socket (recommended for local apps)
./cloud-sql-proxy \
  --unix-socket=/cloudsql \
  my-project:us-central1:my-pg-instance

# Run as background daemon with service account
./cloud-sql-proxy \
  --credentials-file=/path/to/sa-key.json \
  --port=5432 \
  my-project:us-central1:my-pg-instance &
```

### Connecting via gcloud

```bash
# Connect to PostgreSQL (opens psql session)
gcloud sql connect my-pg-instance --user=postgres --database=mydb

# Connect to MySQL
gcloud sql connect my-mysql-instance --user=root

# Connect to SQL Server (uses sqlcmd internally)
gcloud sql connect my-sqlserver-instance --user=sqlserver
```

> **Tip:** `gcloud sql connect` temporarily adds your current public IP to authorized networks for 1 minute. It is meant for interactive debugging only — not application connections.

### Connection Strings by Engine

```bash
# PostgreSQL (via Auth Proxy on localhost)
postgresql://USER:PASSWORD@127.0.0.1:5432/DATABASE

# PostgreSQL (via Unix socket from Auth Proxy)
postgresql://USER:PASSWORD@/DATABASE?host=/cloudsql/PROJECT:REGION:INSTANCE

# MySQL (via Auth Proxy on localhost)
mysql://USER:PASSWORD@127.0.0.1:3306/DATABASE

# MySQL (via Unix socket)
mysql://USER:PASSWORD@/DATABASE?unix_socket=/cloudsql/PROJECT:REGION:INSTANCE

# SQL Server (via Auth Proxy on localhost)
Server=127.0.0.1,1433;Database=DATABASE;User Id=USER;Password=PASSWORD;
```

### GKE Sidecar Proxy Pattern

```yaml
# Pod spec with Cloud SQL Auth Proxy as a sidecar container
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: my-app:latest
      env:
        - name: DB_HOST
          value: "127.0.0.1"
        - name: DB_PORT
          value: "5432"
      volumeMounts:
        - name: cloudsql-unix-socket
          mountPath: /cloudsql

    - name: cloud-sql-proxy                         # Sidecar container
      image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2
      args:
        - "--unix-socket=/cloudsql"
        - "--credentials-file=/secrets/sa-key.json"
        - "my-project:us-central1:my-pg-instance"
      securityContext:
        runAsNonRoot: true
      volumeMounts:
        - name: cloudsql-unix-socket
          mountPath: /cloudsql
        - name: sa-secret
          mountPath: /secrets
          readOnly: true

  volumes:
    - name: cloudsql-unix-socket
      emptyDir: {}
    - name: sa-secret
      secret:
        secretName: cloudsql-sa-key
```

### Serverless VPC Access (Cloud Run / Cloud Functions)

```bash
# Create a Serverless VPC Access connector
gcloud compute networks vpc-access connectors create my-connector \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.8.0.0/28

# Deploy Cloud Run service using the connector
gcloud run deploy my-service \
  --image=gcr.io/my-project/my-app \
  --vpc-connector=my-connector \
  --set-env-vars="DB_HOST=PRIVATE_IP_OF_CLOUD_SQL"
```

---

## 5. High Availability & Failover

### HA Architecture

```
                    Primary Zone (us-central1-a)
                    ┌──────────────────────────┐
                    │   Primary Instance        │
                    │   ┌──────┐  ┌─────────┐  │
                    │   │ Data │  │  Write  │  │
                    │   └──┬───┘  └────┬────┘  │
                    └──────┼───────────┼────────┘
                           │  Sync     │ (regional persistent disk)
                    ┌──────┼───────────┼────────┐
                    │   Standby Zone (us-central1-b)
                    │   ┌──────┐  ┌─────────┐  │
                    │   │Standby│  │(passive)│  │
                    └──────────────────────────┘
                    Shared regional disk — no data loss on failover
```

### HA vs Read Replica

| Property | HA Standby | Read Replica |
|---|---|---|
| **Purpose** | Automatic failover (availability) | Read scaling / DR |
| **Readable** | ✗ (passive standby only) | ✅ |
| **Failover** | Automatic (~60s) | Manual promotion only |
| **Location** | Same region, different zone | Same or different region |
| **Cost** | 2× instance cost | Full replica instance cost |
| **Data loss on failure** | Near-zero (shared disk) | Replica lag (seconds) |

### Enabling HA

```bash
# Enable HA at creation
gcloud sql instances create my-pg-instance \
  --database-version=POSTGRES_15 \
  --tier=db-custom-4-15360 \
  --region=us-central1 \
  --availability-type=REGIONAL  # REGIONAL = HA; ZONAL = no HA

# Enable HA on existing instance (causes restart)
gcloud sql instances patch my-pg-instance \
  --availability-type=REGIONAL

# Verify HA configuration
gcloud sql instances describe my-pg-instance \
  --format="value(settings.availabilityType)"
```

### Manual Failover (Testing)

```bash
# Trigger a manual failover to the standby zone
gcloud sql instances failover my-pg-instance
```

> **Tip:** Test manual failover regularly in staging. During failover, connections are dropped for ~30–60 seconds (Enterprise) or ~30 seconds (Enterprise Plus). Ensure your application has connection retry logic.

### RTO / RPO

| Scenario | Enterprise RTO | Enterprise Plus RTO | RPO |
|---|---|---|---|
| Zone failure (HA enabled) | ~60 seconds | ~30 seconds | Near-zero (shared disk) |
| Zone failure (HA disabled) | Manual recovery | Manual recovery | Last backup |
| Regional failure | Manual (cross-region replica promotion) | Manual | Replica lag |

---

## 6. Read Replicas & Cross-Region Replicas

### Replica Types

| Type | Zone | Use Case |
|---|---|---|
| **In-region replica** | Same region, different zone | Read scaling |
| **Cross-region replica** | Different region | DR, geo-distributed reads |
| **Cascade replica** | Replica of a replica | Isolate replica load; DR chains |

### Creating Read Replicas

```bash
# In-region read replica
gcloud sql instances create my-pg-replica \
  --master-instance-name=my-pg-instance \
  --region=us-central1 \
  --tier=db-custom-4-15360

# Cross-region read replica (different region)
gcloud sql instances create my-pg-replica-eu \
  --master-instance-name=my-pg-instance \
  --region=europe-west1 \
  --tier=db-custom-4-15360

# Cascade replica (replica of a replica)
gcloud sql instances create my-pg-cascade-replica \
  --master-instance-name=my-pg-replica \
  --region=us-east1 \
  --tier=db-custom-2-7680
```

### Monitoring Replica Lag

```bash
# Check replica lag via Cloud Monitoring metric
# Metric: cloudsql.googleapis.com/database/replication/replica_lag

# Using gcloud to view metrics (last 1 hour)
gcloud monitoring metrics list \
  --filter="metric.type=cloudsql.googleapis.com/database/replication/replica_lag"

# Via SQL (PostgreSQL replica)
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;

# Via SQL (MySQL replica)
SHOW SLAVE STATUS\G
# Look at: Seconds_Behind_Master
```

### Promoting a Replica to Standalone

```bash
# Promote replica to a standalone primary instance
# (breaks replication — replica becomes independent)
gcloud sql instances promote-replica my-pg-replica

# Verify it's now a standalone primary
gcloud sql instances describe my-pg-replica \
  --format="value(instanceType)"
# Should return: CLOUD_SQL_INSTANCE (not READ_REPLICA_INSTANCE)
```

> ⚠️ **Warning:** Promoting a cross-region replica during a DR event is a **manual, irreversible operation**. Replication from the original primary does not resume automatically. Update your application's connection string after promotion.

### Cross-Region DR Pattern

```
Region: us-central1 (Primary)
  my-pg-instance  ──replication──►  my-pg-replica-eu  (europe-west1)

DR Event:
  1. Verify replica lag is acceptable
  2. gcloud sql instances promote-replica my-pg-replica-eu
  3. Update DNS/connection string to point to my-pg-replica-eu
  4. my-pg-replica-eu is now the primary — no longer receives replication
```

---

## 7. Backups & Recovery

### Automated Backups

```bash
# Enable automated backups with 7-day retention
gcloud sql instances patch my-pg-instance \
  --backup \
  --backup-start-time=03:00 \
  --retained-backups-count=7 \
  --retained-transaction-log-days=7  # for PITR (max 35 for Enterprise Plus)

# View backup configuration
gcloud sql instances describe my-pg-instance \
  --format="json(settings.backupConfiguration)"
```

### On-Demand Backups

```bash
# Create an on-demand backup (persists even if instance is deleted)
gcloud sql backups create \
  --instance=my-pg-instance \
  --description="Pre-deployment backup"

# List backups
gcloud sql backups list --instance=my-pg-instance

# Describe a backup
gcloud sql backups describe BACKUP_ID --instance=my-pg-instance
```

### Point-In-Time Recovery (PITR)

PITR uses continuous WAL (PostgreSQL) or binary log (MySQL) archiving to restore to any second within the retention window.

```bash
# Enable PITR (must have automated backups enabled)
gcloud sql instances patch my-pg-instance \
  --enable-point-in-time-recovery

# Restore to a specific point in time (creates a NEW instance)
gcloud sql instances restore-backup my-pg-instance \
  --backup-instance=my-pg-instance \
  --restore-instance=my-pg-restored \
  --point-in-time=2025-01-15T14:30:00.000Z
```

> ⚠️ **Warning:** PITR **always restores to a new instance**, not in-place. You must update your application's connection string to point to the restored instance. Plan your DR runbook accordingly.

### Restoring from a Backup

```bash
# Restore to the same instance (overwrites all current data)
gcloud sql instances restore-backup my-pg-instance \
  --backup-run-id=BACKUP_RUN_ID \
  --backup-instance=my-pg-instance

# Restore to a different (new) instance
gcloud sql instances restore-backup my-pg-restored \
  --backup-run-id=BACKUP_RUN_ID \
  --backup-instance=my-pg-instance
```

### Exporting to Cloud Storage

```bash
# Export entire instance as SQL dump (PostgreSQL)
gcloud sql export sql my-pg-instance \
  gs://my-backup-bucket/exports/mydb-$(date +%Y%m%d).sql \
  --database=mydb \
  --offload  # runs in background, doesn't block instance

# Export specific table as CSV
gcloud sql export csv my-pg-instance \
  gs://my-backup-bucket/exports/users-$(date +%Y%m%d).csv \
  --database=mydb \
  --query="SELECT * FROM users WHERE created_at > '2025-01-01'"

# Export MySQL database
gcloud sql export sql my-mysql-instance \
  gs://my-backup-bucket/exports/mydb.sql \
  --database=mydb
```

### Importing from Cloud Storage

```bash
# Import SQL dump into PostgreSQL instance
gcloud sql import sql my-pg-instance \
  gs://my-backup-bucket/exports/mydb-20250101.sql \
  --database=mydb

# Import CSV into a specific table
gcloud sql import csv my-pg-instance \
  gs://my-backup-bucket/exports/users-20250101.csv \
  --database=mydb \
  --table=users

# Import SQL dump into MySQL instance
gcloud sql import sql my-mysql-instance \
  gs://my-backup-bucket/exports/mydb.sql \
  --database=mydb
```

> **Tip:** For large imports/exports, use `--offload` (SQL export) to avoid locking the instance. The operation runs asynchronously and the instance remains available. Grant the Cloud SQL service account `storage.objectAdmin` on the GCS bucket.

### Cross-Region Backup Replication

```bash
# Enable backup replication to another region
gcloud sql instances patch my-pg-instance \
  --backup-location=us-east1  # store backups in a secondary region
```

---

## 8. Security

### IAM Roles Reference

| Role | Who Uses It | Permissions |
|---|---|---|
| `roles/cloudsql.admin` | DBAs, automation SAs | Full control of all Cloud SQL resources |
| `roles/cloudsql.editor` | Developers | Create/modify instances and databases; no delete |
| `roles/cloudsql.viewer` | Monitoring, read-only | View instances and configurations |
| `roles/cloudsql.client` | Application service accounts | Connect via Auth Proxy / connector |
| `roles/cloudsql.instanceUser` | IAM database users | Login to database via IAM auth |

```bash
# Grant an app service account connect access
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-app-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

# Grant IAM database user role
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-app-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/cloudsql.instanceUser"
```

### IAM Database Authentication

IAM auth allows GCP service accounts and users to authenticate to Cloud SQL using their IAM credentials — no static passwords.

```bash
# Enable IAM auth on an instance
gcloud sql instances patch my-pg-instance \
  --database-flags=cloudsql.iam_authentication=on

# Create an IAM database user (PostgreSQL)
gcloud sql users create my-app-sa@my-project.iam \
  --instance=my-pg-instance \
  --type=CLOUD_IAM_SERVICE_ACCOUNT

# Create an IAM user (human user)
gcloud sql users create user@example.com \
  --instance=my-pg-instance \
  --type=CLOUD_IAM_USER
```

```sql
-- Grant connect privilege to IAM user in PostgreSQL
GRANT CONNECT ON DATABASE mydb TO "my-app-sa@my-project.iam";
GRANT USAGE ON SCHEMA public TO "my-app-sa@my-project.iam";
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public
  TO "my-app-sa@my-project.iam";
```

### Built-In Database Users

```bash
# Create a standard database user with password
gcloud sql users create appuser \
  --instance=my-pg-instance \
  --password=SecurePassword123!

# Change password
gcloud sql users set-password appuser \
  --instance=my-pg-instance \
  --password=NewSecurePassword456!

# Delete a user
gcloud sql users delete appuser \
  --instance=my-pg-instance

# List users
gcloud sql users list --instance=my-pg-instance
```

### SSL/TLS Enforcement

```bash
# Require SSL for all connections
gcloud sql instances patch my-pg-instance \
  --ssl-mode=ENCRYPTED_ONLY  # refuse non-SSL connections

# Require SSL with client certificate verification
gcloud sql instances patch my-pg-instance \
  --ssl-mode=TRUSTED_CLIENT_CERTIFICATE_REQUIRED

# Create a client certificate
gcloud sql ssl client-certs create my-client-cert client.key \
  --instance=my-pg-instance

# Download server CA cert
gcloud sql ssl server-ca-certs list \
  --instance=my-pg-instance \
  --format="value(cert)" > server-ca.pem
```

### Customer-Managed Encryption Keys (CMEK)

```bash
# Step 1: Create a KMS key
gcloud kms keyrings create my-cloudsql-keyring --location=us-central1
gcloud kms keys create my-cloudsql-key \
  --location=us-central1 \
  --keyring=my-cloudsql-keyring \
  --purpose=encryption

# Step 2: Grant Cloud SQL SA access to the key
SQL_SA=$(gcloud sql instances describe my-pg-instance \
  --format="value(serviceAccountEmailAddress)")

gcloud kms keys add-iam-policy-binding my-cloudsql-key \
  --location=us-central1 \
  --keyring=my-cloudsql-keyring \
  --member="serviceAccount:${SQL_SA}" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Step 3: Create instance with CMEK
gcloud sql instances create my-cmek-instance \
  --database-version=POSTGRES_15 \
  --tier=db-custom-4-15360 \
  --region=us-central1 \
  --disk-encryption-key=projects/my-project/locations/us-central1/keyRings/my-cloudsql-keyring/cryptoKeys/my-cloudsql-key
```

### Secret Manager Integration (Python)

```python
from google.cloud import secretmanager

def get_db_password(project_id: str, secret_name: str) -> str:
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_name}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")

# Usage
password = get_db_password("my-project", "cloudsql-db-password")
```

### Audit Logging

```bash
# Enable Data Access audit logs for Cloud SQL
# (done at project level in IAM > Audit Logs)
gcloud projects get-iam-policy my-project > policy.yaml
# Add to policy.yaml:
# auditConfigs:
# - auditLogConfigs:
#   - logType: DATA_READ
#   - logType: DATA_WRITE
#   service: cloudsql.googleapis.com

gcloud projects set-iam-policy my-project policy.yaml
```

---

## 9. Performance & Flags

### Setting Database Flags

```bash
# Set a single flag
gcloud sql instances patch my-pg-instance \
  --database-flags=max_connections=200

# Set multiple flags at once (replaces ALL existing flags)
gcloud sql instances patch my-pg-instance \
  --database-flags=max_connections=200,shared_buffers=512000,work_mem=16384

# Clear all flags (reset to defaults)
gcloud sql instances patch my-pg-instance \
  --clear-database-flags
```

> ⚠️ **Warning:** `--database-flags` **replaces all existing flags**. Always include all desired flags in a single command to avoid accidentally resetting others. Retrieve current flags first with `gcloud sql instances describe`.

### Key PostgreSQL Flags

| Flag | Recommended Value | Effect |
|---|---|---|
| `max_connections` | 100–500 (depends on RAM) | Max concurrent connections; use PgBouncer to multiplex |
| `shared_buffers` | 25% of instance RAM (in KB) | Main PostgreSQL cache — most impactful setting |
| `work_mem` | 4–64 MB per connection | Memory per sort/hash operation; high value risks OOM |
| `effective_cache_size` | 75% of instance RAM | Planner hint for available cache |
| `wal_buffers` | `16MB` | Write-ahead log buffer; increase for write-heavy workloads |
| `pg_stat_statements` | `on` | Enable query statistics (required for Query Insights) |
| `log_min_duration_statement` | `1000` (ms) | Log queries slower than 1 second |
| `autovacuum_vacuum_cost_delay` | `2` (ms) | Speed up autovacuum for high-churn tables |
| `random_page_cost` | `1.1` (SSD) | Planner cost model — set low for SSD storage |

```bash
# Example: optimise PostgreSQL for a 16 GB RAM instance
gcloud sql instances patch my-pg-instance \
  --database-flags=\
shared_buffers=4194304,\
effective_cache_size=12582912,\
max_connections=200,\
work_mem=16384,\
pg_stat_statements.track=all,\
log_min_duration_statement=1000,\
random_page_cost=1.1
```

### Key MySQL Flags

| Flag | Recommended Value | Effect |
|---|---|---|
| `innodb_buffer_pool_size` | 70–80% of RAM | InnoDB data/index cache — most impactful |
| `max_connections` | 150–500 | Max concurrent connections |
| `slow_query_log` | `on` | Enable slow query logging |
| `long_query_time` | `1` (seconds) | Log queries taking > 1 second |
| `innodb_io_capacity` | `2000` (SSD) | I/O operations per second for InnoDB background tasks |
| `innodb_flush_method` | `O_DIRECT` | Bypass OS cache for InnoDB data files |
| `query_cache_type` | `0` | Disable query cache (deprecated, causes contention) |
| `wait_timeout` | `28800` (8h) | Idle connection timeout (seconds) |

```bash
# Example: optimise MySQL for 8 GB RAM instance
gcloud sql instances patch my-mysql-instance \
  --database-flags=\
innodb_buffer_pool_size=6442450944,\
max_connections=200,\
slow_query_log=on,\
long_query_time=1,\
innodb_io_capacity=2000
```

### Key SQL Server Flags

| Flag | Recommended Value | Effect |
|---|---|---|
| `max degree of parallelism` | `0` (auto) or `= vCPU count` | Parallelism for query execution |
| `cost threshold for parallelism` | `50` | Minimum query cost before parallelism used |
| `max server memory (MB)` | Leave to Cloud SQL | Managed automatically |
| `optimize for ad hoc workloads` | `1` | Cache single-use query plans more efficiently |

### Query Insights

Query Insights provides execution plans, top queries, and wait event analysis in the Cloud Console.

```bash
# Enable Query Insights
gcloud sql instances patch my-pg-instance \
  --insights-config-query-insights-enabled \
  --insights-config-query-plans-per-minute=5 \
  --insights-config-query-string-length=1024 \
  --insights-config-record-application-tags \
  --insights-config-record-client-address
```

### Connection Pooling with PgBouncer

Cloud SQL does not include PgBouncer natively. Run it as a sidecar or separate service:

```yaml
# PgBouncer sidecar in Kubernetes
- name: pgbouncer
  image: pgbouncer/pgbouncer:1.21.0
  env:
    - name: DATABASES_HOST
      value: "127.0.0.1"          # Cloud SQL Auth Proxy on localhost
    - name: DATABASES_PORT
      value: "5432"
    - name: DATABASES_DBNAME
      value: "mydb"
    - name: PGBOUNCER_POOL_MODE
      value: "transaction"        # transaction pooling for stateless apps
    - name: PGBOUNCER_MAX_CLIENT_CONN
      value: "1000"
    - name: PGBOUNCER_DEFAULT_POOL_SIZE
      value: "20"                 # 20 real connections to Cloud SQL
  ports:
    - containerPort: 5432
```

### CPU and RAM Sizing Guidelines

| Workload Type | vCPUs | RAM | Notes |
|---|---|---|---|
| Dev / test | 1–2 | 2–4 GB | Use `db-f1-micro` or `db-g1-small` for cost |
| Small production | 2–4 | 8–16 GB | `db-custom-4-15360` |
| Medium production | 4–8 | 16–32 GB | Ensure `shared_buffers` = 25% RAM |
| Large OLTP | 16–32 | 64–128 GB | Consider Enterprise Plus for maintenance SLA |
| Data warehouse / analytics | 32–96 | 128–624 GB | Consider AlloyDB or BigQuery instead |

---

## 10. Client Library Connections — Python

### Installation

```bash
pip install cloud-sql-python-connector[pg8000]   # PostgreSQL
pip install cloud-sql-python-connector[pymysql]  # MySQL
pip install sqlalchemy
```

### PostgreSQL — Synchronous (SQLAlchemy + pg8000)

```python
from google.cloud.sql.connector import Connector
import sqlalchemy

def create_pg_engine():
    connector = Connector()

    def getconn():
        return connector.connect(
            "my-project:us-central1:my-pg-instance",  # connection name
            "pg8000",                                   # driver
            user="appuser",
            password="SecurePassword123!",
            db="mydb",
        )

    engine = sqlalchemy.create_engine(
        "postgresql+pg8000://",
        creator=getconn,
        pool_size=5,
        max_overflow=2,
        pool_timeout=30,
        pool_recycle=1800,
    )
    return engine

# Usage
engine = create_pg_engine()
with engine.connect() as conn:
    result = conn.execute(sqlalchemy.text("SELECT version()"))
    print(result.fetchone())
```

### PostgreSQL — IAM Authentication

```python
from google.cloud.sql.connector import Connector, IPTypes
import google.auth

def create_iam_pg_engine():
    connector = Connector()
    credentials, _ = google.auth.default()   # ADC — uses service account automatically

    def getconn():
        return connector.connect(
            "my-project:us-central1:my-pg-instance",
            "pg8000",
            user="my-app-sa@my-project.iam",  # IAM SA username
            enable_iam_auth=True,              # use IAM token instead of password
            db="mydb",
            ip_type=IPTypes.PRIVATE,           # use private IP
        )

    return sqlalchemy.create_engine(
        "postgresql+pg8000://", creator=getconn
    )
```

### MySQL — Synchronous (SQLAlchemy + PyMySQL)

```python
from google.cloud.sql.connector import Connector
import sqlalchemy

def create_mysql_engine():
    connector = Connector()

    def getconn():
        return connector.connect(
            "my-project:us-central1:my-mysql-instance",
            "pymysql",
            user="appuser",
            password="SecurePassword123!",
            db="mydb",
        )

    return sqlalchemy.create_engine(
        "mysql+pymysql://",
        creator=getconn,
        pool_size=5,
        pool_recycle=1800,
    )
```

### PostgreSQL — Async (asyncpg + SQLAlchemy async)

```python
from google.cloud.sql.connector import AsyncConnector
import sqlalchemy.ext.asyncio as async_sa

async def create_async_pg_engine():
    async_connector = AsyncConnector()

    async def getconn():
        return await async_connector.connect(
            "my-project:us-central1:my-pg-instance",
            "asyncpg",
            user="appuser",
            password="SecurePassword123!",
            db="mydb",
        )

    engine = async_sa.create_async_engine(
        "postgresql+asyncpg://",
        async_creator=getconn,
    )
    return engine

# Usage in async context
import asyncio

async def main():
    engine = await create_async_pg_engine()
    async with engine.connect() as conn:
        result = await conn.execute(sqlalchemy.text("SELECT now()"))
        print(result.fetchone())

asyncio.run(main())
```

### Unix Socket Connection (Auth Proxy Running Locally)

```python
import sqlalchemy

# PostgreSQL via Unix socket (Auth Proxy must be running with --unix-socket=/cloudsql)
engine = sqlalchemy.create_engine(
    "postgresql+psycopg2://appuser:SecurePassword123!@/mydb"
    "?host=/cloudsql/my-project:us-central1:my-pg-instance"
)

# MySQL via Unix socket
engine = sqlalchemy.create_engine(
    "mysql+pymysql://appuser:SecurePassword123!@/mydb"
    "?unix_socket=/cloudsql/my-project:us-central1:my-mysql-instance"
)
```

### Environment Variable Pattern (12-Factor App)

```python
import os
import sqlalchemy
from google.cloud.sql.connector import Connector

def create_engine_from_env():
    instance_connection = os.environ["INSTANCE_CONNECTION_NAME"]
    db_user = os.environ["DB_USER"]
    db_pass = os.environ["DB_PASS"]
    db_name = os.environ["DB_NAME"]
    db_driver = os.environ.get("DB_DRIVER", "pg8000")

    connector = Connector()

    def getconn():
        return connector.connect(
            instance_connection, db_driver,
            user=db_user, password=db_pass, db=db_name,
        )

    return sqlalchemy.create_engine("postgresql+pg8000://", creator=getconn)
```

---

## 11. gcloud CLI Quick Reference

### Instance Management

```bash
# CREATE
gcloud sql instances create INSTANCE \
  --database-version=POSTGRES_15 \
  --tier=db-custom-4-15360 \
  --region=REGION

# LIST
gcloud sql instances list

# DESCRIBE
gcloud sql instances describe INSTANCE

# PATCH (update)
gcloud sql instances patch INSTANCE --tier=db-custom-8-30720

# DELETE
gcloud sql instances delete INSTANCE

# RESTART
gcloud sql instances restart INSTANCE

# CLONE
gcloud sql instances clone SOURCE_INSTANCE TARGET_INSTANCE

# FAILOVER (manual HA test)
gcloud sql instances failover INSTANCE

# CONNECT (interactive session)
gcloud sql connect INSTANCE --user=postgres --database=mydb
```

### Database & User Management

```bash
# CREATE database
gcloud sql databases create mydb --instance=INSTANCE

# LIST databases
gcloud sql databases list --instance=INSTANCE

# DELETE database
gcloud sql databases delete mydb --instance=INSTANCE

# CREATE user
gcloud sql users create appuser --instance=INSTANCE --password=PASS

# SET user password
gcloud sql users set-password appuser --instance=INSTANCE --password=NEWPASS

# LIST users
gcloud sql users list --instance=INSTANCE

# DELETE user
gcloud sql users delete appuser --instance=INSTANCE

# CREATE IAM user
gcloud sql users create user@example.com \
  --instance=INSTANCE \
  --type=CLOUD_IAM_USER
```

### Backup & Restore

```bash
# CREATE on-demand backup
gcloud sql backups create --instance=INSTANCE --description="Manual backup"

# LIST backups
gcloud sql backups list --instance=INSTANCE

# DESCRIBE backup
gcloud sql backups describe BACKUP_ID --instance=INSTANCE

# DELETE backup
gcloud sql backups delete BACKUP_ID --instance=INSTANCE

# RESTORE to same instance
gcloud sql instances restore-backup INSTANCE \
  --backup-run-id=BACKUP_ID \
  --backup-instance=INSTANCE

# RESTORE PITR to new instance
gcloud sql instances restore-backup NEW_INSTANCE \
  --backup-instance=SOURCE_INSTANCE \
  --point-in-time=2025-01-15T14:30:00.000Z
```

### Read Replicas

```bash
# CREATE in-region replica
gcloud sql instances create REPLICA \
  --master-instance-name=PRIMARY \
  --region=REGION \
  --tier=db-custom-4-15360

# CREATE cross-region replica
gcloud sql instances create REPLICA \
  --master-instance-name=PRIMARY \
  --region=OTHER_REGION

# PROMOTE replica to standalone
gcloud sql instances promote-replica REPLICA

# DELETE replica
gcloud sql instances delete REPLICA
```

### Import & Export

```bash
# EXPORT SQL dump
gcloud sql export sql INSTANCE gs://BUCKET/file.sql --database=DB

# EXPORT CSV
gcloud sql export csv INSTANCE gs://BUCKET/file.csv \
  --database=DB \
  --query="SELECT * FROM TABLE"

# IMPORT SQL dump
gcloud sql import sql INSTANCE gs://BUCKET/file.sql --database=DB

# IMPORT CSV
gcloud sql import csv INSTANCE gs://BUCKET/file.csv \
  --database=DB \
  --table=TABLE
```

### Flags & SSL

```bash
# SET flags
gcloud sql instances patch INSTANCE \
  --database-flags=FLAG1=VALUE1,FLAG2=VALUE2

# CLEAR all flags
gcloud sql instances patch INSTANCE --clear-database-flags

# CREATE client SSL cert
gcloud sql ssl client-certs create CERT_NAME KEY_FILE \
  --instance=INSTANCE

# LIST SSL certs
gcloud sql ssl client-certs list --instance=INSTANCE

# DELETE SSL cert
gcloud sql ssl client-certs delete CERT_NAME --instance=INSTANCE

# LIST server CA certs
gcloud sql ssl server-ca-certs list --instance=INSTANCE
```

---

## 12. Pricing Model Summary

> Prices are approximate US region rates as of 2025. Verify at [cloud.google.com/sql/pricing](https://cloud.google.com/sql/pricing).

### Instance Pricing (per vCPU and GB RAM per hour)

| Component | MySQL / PostgreSQL | SQL Server |
|---|---|---|
| vCPU | $0.0413 / vCPU-hour | $0.0413 / vCPU-hour + license |
| RAM | $0.0070 / GB-hour | $0.0070 / GB-hour + license |
| SQL Server license (SE) | — | ~$0.33 / vCPU-hour |
| SQL Server license (EE) | — | ~$1.10 / vCPU-hour |

### Storage Pricing

| Storage Type | Price |
|---|---|
| SSD | $0.170 / GB / month |
| HDD | $0.090 / GB / month |
| Backup storage (automated) | $0.080 / GB / month |
| On-demand backup | $0.080 / GB / month |

### HA & Replica Pricing

| Feature | Cost |
|---|---|
| HA (Regional availability) | ~2× instance cost (standby instance) |
| Read replica | Full instance cost per replica |
| Cross-region replica | Full instance cost + cross-region egress |

### Enterprise vs Enterprise Plus Pricing

| Edition | Approximate Premium |
|---|---|
| Enterprise | Base price |
| Enterprise Plus | ~35–50% premium over Enterprise |

### Network Egress

| Destination | Cost |
|---|---|
| Same region (GCE → Cloud SQL) | Free |
| Cross-region replica sync | ~$0.01–$0.08 / GB |
| Cloud SQL → internet | Standard GCP egress |

### Cost Optimisation Strategies

| Strategy | Savings |
|---|---|
| Use `db-f1-micro` / `db-g1-small` for dev/test | 80–90% vs production tiers |
| Stop dev instances outside business hours | Up to 65% (no compute charge when stopped; storage still billed) |
| Right-size RAM before enabling `shared_buffers` | Avoid paying for RAM that isn't used by the DB |
| Use HDD for non-performance-sensitive workloads | 47% cheaper than SSD |
| Set backup retention to minimum required | Backup storage billed per GB |
| Use Committed Use Discounts (1 or 3 year) | Up to 25–52% off instance cost |
| Use cross-region replicas instead of dual-region instances | More flexible DR at similar cost |
| Enable storage auto-resize conservatively (set size alerts) | Prevents paying for unused provisioned storage |

---

## 13. Common Errors & Troubleshooting

| Error / Issue | Cause | Fix |
|---|---|---|
| `connection refused` on port 5432/3306 | Auth Proxy not running; wrong port; firewall blocking | Start Cloud SQL Auth Proxy; verify `--port` flag; check VPC firewall rules |
| `connection timeout` | Instance private IP unreachable; VPC peering not set up; wrong region | Verify PSA setup; confirm instance region matches app region; use `gcloud sql connect` to test |
| `FATAL: password authentication failed` | Wrong password; user doesn't exist; IAM auth misconfigured | Verify credentials; `gcloud sql users list`; check `cloudsql.iam_authentication` flag |
| `too many connections` | `max_connections` limit reached | Increase `max_connections` flag; implement PgBouncer connection pooling; close idle connections |
| `SSL SYSCALL error` / `SSL handshake failed` | Client not using SSL; certificate mismatch; `ssl-mode=ENCRYPTED_ONLY` set | Use SSL in connection string; download server CA cert; check `--ssl-mode` setting |
| `replica lag too high` | Replica can't keep up with primary write volume; replica under-resourced | Scale up replica tier; reduce write load; check `Seconds_Behind_Master` or `pg_last_xact_replay_timestamp()` |
| `PITR restore failed` | PITR not enabled on source; target timestamp outside retention window; logs missing | Verify `--enable-point-in-time-recovery` is on; check `retained-transaction-log-days`; restore from nearest backup |
| `instance is read-only` | Storage usage > 95% and auto-resize failed or disabled | Immediately increase storage: `gcloud sql instances patch --storage-size=NEWSIZE`; enable `--storage-auto-increase` |
| `slow queries` | Missing indexes; `shared_buffers` too low; `work_mem` too low; connection pooling absent | Enable Query Insights; run `EXPLAIN ANALYZE`; add indexes; tune memory flags; use PgBouncer |
| `IAM auth login failed` | IAM user not created in DB; missing `cloudsql.instanceUser` role; wrong username format | Create IAM user: `gcloud sql users create`; verify IAM role binding; PostgreSQL IAM username must include `@DOMAIN` |
| `ERROR 1419: Super privilege required` (MySQL) | Binary logging enabled but user lacks SUPER privilege for triggers/functions | Set flag `log_bin_trust_function_creators=on` for MySQL |
| `export/import fails with 403` | Cloud SQL SA lacks permissions on GCS bucket | Grant `roles/storage.objectAdmin` on the bucket to the Cloud SQL SA |
| `instance upgrade failed` | Incompatible flags for new version; insufficient storage | Review version-specific incompatible flags; ensure adequate free storage before upgrade |
| `ERROR: extension does not exist` (PostgreSQL) | Extension not enabled in the database | `CREATE EXTENSION IF NOT EXISTS "uuid-ossp";` — verify extension is whitelisted by Cloud SQL |
| `could not connect to server` (Auth Proxy) | Service account lacks `roles/cloudsql.client`; wrong connection name | Add `roles/cloudsql.client` to SA; verify connection name format `PROJECT:REGION:INSTANCE` |

### Diagnostic Commands

```bash
# Check instance state
gcloud sql instances describe INSTANCE --format="value(state)"

# Get connection name
gcloud sql instances describe INSTANCE \
  --format="value(connectionName)"

# Check current flags
gcloud sql instances describe INSTANCE \
  --format="json(settings.databaseFlags)"

# Check storage usage
gcloud sql instances describe INSTANCE \
  --format="json(diskEncryptionConfiguration,settings.dataDiskSizeGb)"

# List recent operations on an instance
gcloud sql operations list --instance=INSTANCE --limit=10

# Check replica lag (PostgreSQL — run on replica)
gcloud sql connect REPLICA --user=postgres \
  --database=postgres \
  --execute="SELECT now() - pg_last_xact_replay_timestamp() AS lag;"

# View Cloud Logging for Cloud SQL errors
gcloud logging read \
  'resource.type="cloudsql_database" AND severity>=ERROR' \
  --limit=20 \
  --format="table(timestamp,severity,jsonPayload.message)"
```

---

## Quick Reference Card

```
Connection name format:   PROJECT:REGION:INSTANCE
Auth Proxy binary:        cloud-sql-proxy (v2+)
Auth Proxy TCP port:      5432 (PG), 3306 (MySQL), 1433 (SQL Server)
IAM user format (PG):     serviceaccount@project.iam  (no @gserviceaccount.com)
Max storage:              64 TB (all engines)
Max vCPUs:                96
HA type:                  REGIONAL (2 zones) or ZONAL (no HA)
HA cost:                  ~2× instance cost
PITR retention:           Up to 7 days (Enterprise), 35 days (Enterprise Plus)
Storage change:           Increase only — cannot decrease
Backup SA perm needed:    storage.objectAdmin on GCS bucket for export/import
Deletion protection:      Enabled by default on new instances (recommended)
Connector install (PG):   pip install cloud-sql-python-connector[pg8000]
Edition upgrade:          Enterprise → Enterprise Plus: patch --edition=ENTERPRISE_PLUS
```

---

*Reference: [Cloud SQL Docs](https://cloud.google.com/sql/docs) | [Pricing](https://cloud.google.com/sql/pricing) | [Python Connector](https://github.com/GoogleCloudPlatform/cloud-sql-python-connector) | [Auth Proxy](https://cloud.google.com/sql/docs/postgres/sql-proxy)*
