# 🐘 GCP AlloyDB for PostgreSQL — Comprehensive Cheatsheet

> **Audience:** Developers & Cloud Engineers | **Last updated:** 2025
> A single-reference document covering architecture, AI/vector search, columnar engine, HA, migrations, security, and best practices.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Architecture Deep Dive](#2-architecture-deep-dive)
3. [Cluster & Instance Types](#3-cluster--instance-types)
4. [Networking & Connectivity](#4-networking--connectivity)
5. [Creating & Managing Clusters and Instances](#5-creating--managing-clusters-and-instances)
6. [AlloyDB Auth Proxy](#6-alloydb-auth-proxy)
7. [PostgreSQL Compatibility & Extensions](#7-postgresql-compatibility--extensions)
8. [AlloyDB AI & Vector Search (pgvector)](#8-alloydb-ai--vector-search-pgvector)
9. [Columnar Engine](#9-columnar-engine)
10. [High Availability & Failover](#10-high-availability--failover)
11. [Backups & Point-In-Time Recovery](#11-backups--point-in-time-recovery)
12. [Security — IAM, Auth & Encryption](#12-security--iam-auth--encryption)
13. [Performance Tuning & Best Practices](#13-performance-tuning--best-practices)
14. [Migrations to AlloyDB](#14-migrations-to-alloydb)
15. [Client Library — Python](#15-client-library--python)
16. [gcloud CLI Quick Reference](#16-gcloud-cli-quick-reference)
17. [Monitoring & Observability](#17-monitoring--observability)
18. [Pricing Model Summary](#18-pricing-model-summary)
19. [Common Errors & Troubleshooting](#19-common-errors--troubleshooting)

---

## 1. Overview & Key Concepts

**AlloyDB for PostgreSQL** is Google's fully managed, PostgreSQL-compatible database service built on a disaggregated storage-compute architecture. It delivers up to 4× faster transactional throughput and 100× faster analytical queries than standard PostgreSQL, with sub-60-second failover, built-in ML inference, and a hybrid columnar engine — all without leaving the PostgreSQL ecosystem.

### AlloyDB vs Other Databases

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                              PostgreSQL-Compatible Landscape                             │
├──────────────┬──────────────┬──────────────┬──────────────┬──────────────┬──────────────┤
│  AlloyDB     │  Cloud SQL PG│  Cloud Spanner│  Aurora PG  │  Self-managed│  RDS PG      │
│  (GCP)       │  (GCP)       │  (GCP)       │  (AWS)       │  PostgreSQL  │  (AWS)       │
├──────────────┼──────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ 100% PG compat│ 100% PG compat│ PG wire compat│ PG-compat  │ Native PG    │ 100% PG compat│
│ Managed      │ Managed      │ Managed      │ Managed      │ Self-managed │ Managed      │
│ Distrib. storage│ Local storage│ Distrib.   │ Distrib.    │ Local/SAN    │ EBS storage  │
│ Columnar engine│ ✗           │ ✗            │ Parallel q.  │ Manual FDW   │ ✗            │
│ Built-in AI  │ ✗            │ ✗            │ ✗            │ Manual       │ ✗            │
│ pgvector     │ pgvector     │ ✗            │ pgvector     │ pgvector     │ pgvector     │
│ < 60s failover│ 60-120s     │ 99.999%      │ < 30s        │ Manual       │ 60-120s      │
│ Up to 64 TB  │ Up to 64 TB  │ Unlimited    │ Up to 128 TB │ Disk-limited │ Up to 64 TB  │
│ $$–$$$       │ $–$$         │ $$$$         │ $$$          │ $–$$         │ $$–$$$       │
└──────────────┴──────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

### When to Choose AlloyDB

- **HTAP workloads** — mix of heavy OLTP + real-time analytics on the same database
- **AI/ML applications** — vector similarity search, RAG pipelines, in-database ML inference
- **High-throughput PostgreSQL** — outgrowing Cloud SQL; need faster writes and reads
- **Sub-60s failover** — stricter RTO than Cloud SQL can provide
- **Large PostgreSQL databases** — auto-scaling storage up to 64 TB without provisioning
- **Replacing Aurora** — moving GCP-native with full PostgreSQL compatibility

### Core Terminology

| Term | Description |
|---|---|
| **Cluster** | Top-level AlloyDB resource. Contains one primary instance + optional read pools. All billing and storage is per cluster. |
| **Primary Instance** | The read-write PostgreSQL instance. One per cluster. Handles all DML and DDL. |
| **Read Pool Instance** | One or more read-only replicas sharing cluster storage. Scale independently from primary. |
| **AlloyDB Omni** | Self-managed AlloyDB that runs anywhere (on-prem, other clouds, laptops). Subscription-based. |
| **Columnar Engine** | Adaptive in-memory columnar cache that accelerates analytical queries automatically. |
| **pgvector** | PostgreSQL extension for vector similarity search; natively supported and optimised in AlloyDB. |
| **AlloyDB AI** | Suite of AI features: embedding generation, vector search (ScaNN), and Vertex AI integration. |
| **ScaNN Index** | AlloyDB's high-performance approximate nearest neighbour vector index (faster than HNSW for large datasets). |
| **Trial Cluster** | Free 30-day evaluation cluster with full AlloyDB capabilities. |
| **Secondary Cluster** | Cross-region read-only replica of a primary cluster for DR. Can be promoted to primary. |
| **Auth Proxy** | Recommended connection broker — handles IAM auth, TLS, and connection pooling. |
| **PSA** | Private Service Access — VPC peering model for AlloyDB networking. |
| **PITR** | Point-In-Time Recovery — restore to any second within the backup retention window. |
| **REGIONAL** | Availability type with automatic HA: primary + standby in different zones, < 60s failover. |
| **ZONAL** | Single-zone instance; no automatic failover. Lower cost, for dev/test. |
| **CMEK** | Customer-Managed Encryption Key — bring-your-own KMS key for cluster encryption. |

---

## 2. Architecture Deep Dive

### Disaggregated Storage-Compute Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          AlloyDB Architecture                                   │
│                                                                                 │
│  COMPUTE LAYER                                                                  │
│  ┌─────────────────────────┐    ┌─────────────────────────────────────────┐    │
│  │   Primary Instance      │    │   Read Pool (0–N nodes)                 │    │
│  │   (Read + Write)        │    │   (Read only — independent scaling)     │    │
│  │                         │    │                                         │    │
│  │  PostgreSQL Engine      │    │  PostgreSQL Engine × N                  │    │
│  │  + Columnar Engine Cache│    │  + Columnar Engine Cache per node       │    │
│  │  + Ultra-fast log buf.  │    │                                         │    │
│  └──────────┬──────────────┘    └───────────────┬─────────────────────────┘    │
│             │ WAL log writes                     │ reads (shared storage)       │
│             ▼                                    ▼                              │
│  ─────────────────────────────────────────────────────────────────────────     │
│  STORAGE LAYER (Google-managed, distributed across 6 copies in 2 regions)      │
│                                                                                 │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Region A (3 copies)          │  Region B (3 copies)                    │  │
│  │  Zone 1: ████  Zone 2: ████   │  Zone 1: ████  Zone 2: ████             │  │
│  │                                                                          │  │
│  │  Log Storage  →  SSTable Storage  ←  Distributed Read Cache              │  │
│  │  (WAL landing)    (compacted data)   (hot pages shared across instances) │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  KEY INSIGHT: Compute nodes share the SAME storage layer.                       │
│  Read pool nodes serve reads from shared distributed cache — no replication lag.│
└─────────────────────────────────────────────────────────────────────────────┘
```

### How Writes and Reads Work

| Operation | Path |
|---|---|
| **Write (DML)** | Client → Primary instance → WAL written to log storage → Acknowledged to client → Background compaction builds SSTables |
| **Read (primary)** | Client → Primary instance → Reads from distributed read cache (shared storage) |
| **Read (read pool)** | Client → Read pool node → Reads from the **same** shared distributed cache → Zero replication lag for reads |
| **Failover** | Standby instance (REGIONAL) attaches to existing storage — no data copy needed → < 60s |

### Columnar Engine

```
Row Store (default)        Columnar Cache (overlay)
┌────┬────┬────┬────┐      ┌────────────────┐
│ id │name│amt │ts  │      │ amt (columnar) │  ← AlloyDB auto-detects
│  1 │ A  │100 │... │  +   │ ts  (columnar) │    analytical patterns and
│  2 │ B  │200 │... │      │ ...            │    populates columnar cache
│  3 │ C  │ 50 │... │      └────────────────┘    for hot analytical columns
└────┴────┴────┴────┘
        ↓                            ↓
  OLTP queries              OLAP/aggregation queries
  use row store             use columnar cache
  (point lookups)           (GROUP BY, SUM, COUNT)
```

### AlloyDB vs Standard PostgreSQL Streaming Replication

| Aspect | AlloyDB | PostgreSQL Streaming Replication |
|---|---|---|
| **Replica lag** | Zero for reads (shared storage) | WAL streaming delay (milliseconds to seconds) |
| **Failover time** | < 60 seconds | Minutes (depends on WAL apply lag) |
| **Storage cost** | Shared (1× storage for all replicas) | N× storage (full copy per replica) |
| **Read scale-out** | Add read pool nodes (minutes) | Provision new replica VM + full base backup |
| **Write path** | Log-based (fast commit) | WAL written to local disk |
| **Replica freshness** | Immediately consistent reads | Replication lag |

---

## 3. Cluster & Instance Types

### Instance Type Comparison

| Type | Read/Write | HA | Scaling | Max vCPUs | Use Case |
|---|---|---|---|---|---|
| **Primary (REGIONAL)** | Read + Write | ✅ Auto failover < 60s | Vertical | 96 vCPU | Production OLTP |
| **Primary (ZONAL)** | Read + Write | ✗ Manual only | Vertical | 96 vCPU | Dev/test, cost-sensitive |
| **Read Pool** | Read only | ✅ (adds/removes nodes) | Horizontal (node count) | 96 vCPU/node | Read scale-out, analytics |
| **Secondary Cluster** | Read only | ✅ Can be promoted | Cross-region | Matches primary | DR, cross-region reads |
| **Trial Cluster** | Read + Write | ✅ REGIONAL | Vertical | 96 vCPU | Free 30-day evaluation |

### Primary Instance Configuration

| Parameter | Range | Notes |
|---|---|---|
| vCPUs | 2, 4, 8, 16, 32, 64, 96 | 2 vCPU minimum; scale online |
| Memory | 16–624 GB | 8 GB RAM per vCPU (fixed ratio) |
| Storage | Auto-scales 10 GB → 64 TB | Pay per GiB used; no provisioning |
| Availability type | REGIONAL or ZONAL | REGIONAL = HA; set at creation, changeable |

### Read Pool Configuration

| Parameter | Range | Notes |
|---|---|---|
| Node count | 1–20 nodes | Each node is a full read replica |
| vCPUs per node | 2–96 | Independent of primary |
| Memory per node | Matches vCPU ratio | 8 GB per vCPU |
| Load balancing | Automatic | Built-in; no external LB needed |

> **Tip:** Read pool nodes share the same distributed storage as the primary — they see writes immediately (no replication lag). Use read pools to offload heavy SELECT workloads, analytical queries, and reporting from the primary instance.

### Cross-Region Secondary Clusters

```bash
# Relationships:
Primary Cluster (us-central1)
    └── Replicates to →  Secondary Cluster (us-east1)
                             └── Read-only until promoted
                             └── Can be promoted to standalone primary
                             └── Promotion is irreversible (severs replication)
```

### Trial Clusters

- **Free for 30 days** — full AlloyDB capabilities including REGIONAL HA, read pools, AI features
- After 30 days: automatically converts to a billed cluster (or delete to avoid charges)
- One trial cluster per project per region
- Cannot be converted back to trial after billing starts

---

## 4. Networking & Connectivity

### Network Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                            GCP Project VPC                                │
│                                                                           │
│  ┌──────────────┐  ┌─────────────────┐  ┌──────────────────────────────┐ │
│  │  GKE Pod     │  │  Cloud Run      │  │  Compute Engine VM           │ │
│  │  (same VPC)  │  │  (needs VPC     │  │  (same VPC subnet)           │ │
│  └──────┬───────┘  │  Access conn.)  │  └──────────────┬───────────────┘ │
│         │          └────────┬────────┘                 │                 │
│         │ Auth Proxy        │ Auth Proxy                │ Auth Proxy      │
│         │ (sidecar)         │ (sidecar)                 │ (local process) │
│         └───────────────────┼───────────────────────────┘                 │
│                             │ VPC internal (PSA peering)                  │
│                  ┌──────────▼──────────────┐                              │
│                  │  AlloyDB Cluster        │                              │
│                  │  Primary: 10.0.1.2:5432 │  ← PSA allocated IP         │
│                  │  Read pool: 10.0.1.3    │  ← No public IP by default  │
│                  └─────────────────────────┘                              │
│                                                                           │
│  On-premises ──► Cloud VPN / Interconnect ──► VPC ──► AlloyDB             │
└───────────────────────────────────────────────────────────────────────────┘
```

### Connection Methods by Platform

| Platform | Method | Extra Config |
|---|---|---|
| Compute Engine | Auth Proxy (local) or direct PSA IP | Same VPC; service account with `alloydb.client` |
| GKE | Auth Proxy sidecar container | Same VPC; Workload Identity |
| Cloud Run | Auth Proxy sidecar or Serverless VPC Access | VPC Access connector in same region |
| Cloud Functions | Auth Proxy or Serverless VPC Access | Connector required for PSA access |
| App Engine Standard | Serverless VPC Access connector | Connector in same region as AlloyDB |
| On-premises | Cloud VPN / Interconnect + Auth Proxy | PSA route must be advertised |
| Public internet | Public IP + authorised networks | Explicit enable required; SSL enforced |

### Private Service Access (PSA) Setup

```bash
# Allocate an IP range for PSA (if not already done)
gcloud compute addresses create alloydb-psa-range \
  --global \
  --purpose=VPC_PEERING \
  --prefix-length=16 \
  --network=projects/my-project/global/networks/default

# Create PSA connection
gcloud services vpc-peerings connect \
  --service=servicenetworking.googleapis.com \
  --ranges=alloydb-psa-range \
  --network=default \
  --project=my-project
```

### Connection String Formats

```bash
# psql via Auth Proxy (Unix socket)
psql "host=/tmp/alloydb-proxy/my-project:us-central1:my-cluster dbname=mydb user=myuser"

# psql via Auth Proxy (TCP)
psql "host=127.0.0.1 port=5432 dbname=mydb user=myuser sslmode=disable"

# SQLAlchemy via Auth Proxy (Unix socket)
postgresql+psycopg2://myuser:password@/mydb?host=/tmp/alloydb-proxy/my-project:us-central1:my-cluster

# SQLAlchemy via Auth Proxy (TCP)
postgresql+psycopg2://myuser:password@127.0.0.1:5432/mydb

# asyncpg via Auth Proxy (TCP)
postgresql://myuser:password@127.0.0.1:5432/mydb

# JDBC via Auth Proxy (TCP)
jdbc:postgresql://127.0.0.1:5432/mydb?user=myuser&password=secret
```

---

## 5. Creating & Managing Clusters and Instances

### Create Primary Cluster + Instance

```bash
# ── Step 1: Create the cluster ──────────────────────────────────────
gcloud alloydb clusters create my-cluster \
  --region=us-central1 \
  --network=projects/my-project/global/networks/default \
  --database-version=POSTGRES_15 \
  --password=StrongPassword123! \
  --allocated-ip-range-name=alloydb-psa-range \
  --backup-window-start=02:00 \
  --continuous-backup-enabled \
  --continuous-backup-retention-period=14

# ── Step 2: Create the primary instance ─────────────────────────────
gcloud alloydb instances create my-primary \
  --instance-type=PRIMARY \
  --cpu-count=8 \
  --region=us-central1 \
  --cluster=my-cluster \
  --availability-type=REGIONAL \
  --ssl-mode=ENCRYPTED_ONLY \
  --database-flags=max_connections=500,work_mem=16384

# ── Production with deletion protection ──────────────────────────────
gcloud alloydb clusters create my-prod-cluster \
  --region=us-central1 \
  --network=projects/my-project/global/networks/default \
  --database-version=POSTGRES_15 \
  --password=StrongPassword123! \
  --deletion-protection

gcloud alloydb instances create my-prod-primary \
  --instance-type=PRIMARY \
  --cpu-count=16 \
  --region=us-central1 \
  --cluster=my-prod-cluster \
  --availability-type=REGIONAL \
  --ssl-mode=ENCRYPTED_ONLY
```

### Create Read Pool Instance

```bash
# Add a read pool with 3 nodes (each 4 vCPUs)
gcloud alloydb instances create my-read-pool \
  --instance-type=READ_POOL \
  --cpu-count=4 \
  --read-pool-node-count=3 \
  --region=us-central1 \
  --cluster=my-cluster \
  --ssl-mode=ENCRYPTED_ONLY

# Scale read pool (add nodes)
gcloud alloydb instances update my-read-pool \
  --read-pool-node-count=5 \
  --region=us-central1 \
  --cluster=my-cluster
```

### Create Secondary Cluster (Cross-Region DR)

```bash
# Create secondary cluster in a different region
gcloud alloydb clusters create my-secondary \
  --region=us-east1 \
  --network=projects/my-project/global/networks/default \
  --primary-cluster=projects/my-project/locations/us-central1/clusters/my-cluster

# Create a read instance in the secondary cluster
gcloud alloydb instances create my-secondary-instance \
  --instance-type=SECONDARY \
  --cpu-count=8 \
  --region=us-east1 \
  --cluster=my-secondary

# Promote secondary to standalone primary (irreversible)
gcloud alloydb clusters promote my-secondary \
  --region=us-east1
```

### Update & Manage Instances

```bash
# Scale primary instance vCPUs (online operation)
gcloud alloydb instances update my-primary \
  --cpu-count=16 \
  --region=us-central1 \
  --cluster=my-cluster

# Update database flags
gcloud alloydb instances update my-primary \
  --database-flags=max_connections=1000,shared_buffers=4096 \
  --region=us-central1 \
  --cluster=my-cluster

# Enable public IP with authorised network
gcloud alloydb instances update my-primary \
  --assign-inbound-public-ip=ASSIGN_IPV4 \
  --authorized-external-networks=203.0.113.0/24 \
  --region=us-central1 \
  --cluster=my-cluster

# Disable public IP
gcloud alloydb instances update my-primary \
  --no-assign-inbound-public-ip \
  --region=us-central1 \
  --cluster=my-cluster

# Delete instance (delete read pool before deleting cluster)
gcloud alloydb instances delete my-read-pool \
  --region=us-central1 \
  --cluster=my-cluster

# Delete cluster
gcloud alloydb clusters delete my-cluster \
  --region=us-central1 \
  --force   # deletes all instances within
```

---

## 6. AlloyDB Auth Proxy

### What is the Auth Proxy?

The AlloyDB Auth Proxy is a lightweight binary that runs alongside your application and proxies connections to AlloyDB. It handles:

- **IAM authentication** — exchanges Google credentials for short-lived database tokens
- **TLS enforcement** — all connections encrypted even if application uses `sslmode=disable`
- **Connection lifecycle** — manages keepalives and reconnection
- **No firewall rules needed** — uses IAM-authenticated HTTPS tunnel

### Installation

```bash
# Download the latest Auth Proxy binary (Linux amd64)
curl -o alloydb-auth-proxy \
  https://storage.googleapis.com/alloydb-auth-proxy/v1.x.x/alloydb-auth-proxy.linux.amd64
chmod +x alloydb-auth-proxy

# Or run via Docker
docker pull gcr.io/alloydb-connectors/alloydb-auth-proxy:latest
```

### Running as a Standalone Process

```bash
# Get the instance connection name
INSTANCE_URI="projects/my-project/locations/us-central1/clusters/my-cluster/instances/my-primary"

# Run on default TCP port 5432
./alloydb-auth-proxy \
  --port=5432 \
  --credentials-file=/path/to/sa-key.json \
  "$INSTANCE_URI"

# Run with Unix socket (preferred — faster than TCP)
./alloydb-auth-proxy \
  --unix-socket=/tmp/alloydb-proxy \
  --credentials-file=/path/to/sa-key.json \
  "$INSTANCE_URI"

# Run using Application Default Credentials (ADC) — recommended on GCP
./alloydb-auth-proxy \
  --port=5432 \
  "$INSTANCE_URI"

# Run multiple instances on different ports
./alloydb-auth-proxy \
  "projects/my-project/locations/us-central1/clusters/cluster-a/instances/primary?port=5432" \
  "projects/my-project/locations/us-central1/clusters/cluster-b/instances/primary?port=5433"

# Enable auto-IAM auth (bypasses password — uses IAM identity)
./alloydb-auth-proxy \
  --auto-iam-authn \
  --port=5432 \
  "$INSTANCE_URI"
```

### Required IAM Permissions

```bash
# Grant alloydb.client to the application service account
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-app@my-project.iam.gserviceaccount.com" \
  --role="roles/alloydb.client"

# Also required (to call AlloyDB API)
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-app@my-project.iam.gserviceaccount.com" \
  --role="roles/serviceusage.serviceUsageConsumer"
```

### Kubernetes Sidecar Pattern

```yaml
# kubernetes-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      serviceAccountName: my-app-ksa  # Workload Identity bound to GCP SA
      containers:
        # ── Application container ─────────────────────────────────────
        - name: my-app
          image: gcr.io/my-project/my-app:latest
          env:
            - name: DB_HOST
              value: "127.0.0.1"
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: "mydb"
          # Application connects to localhost:5432 → Auth Proxy sidecar

        # ── AlloyDB Auth Proxy sidecar ────────────────────────────────
        - name: alloydb-auth-proxy
          image: gcr.io/alloydb-connectors/alloydb-auth-proxy:latest
          args:
            - "--port=5432"
            - "projects/my-project/locations/us-central1/clusters/my-cluster/instances/my-primary"
          securityContext:
            runAsNonRoot: true
          resources:
            requests:
              memory: "32Mi"
              cpu:    "10m"
```

---

## 7. PostgreSQL Compatibility & Extensions

### Supported PostgreSQL Versions

| AlloyDB Version | PostgreSQL Base | Status |
|---|---|---|
| POSTGRES_14 | PostgreSQL 14 | Supported |
| POSTGRES_15 | PostgreSQL 15 | Supported (recommended) |
| POSTGRES_16 | PostgreSQL 16 | Supported |

### Supported Extensions

| Extension | Purpose | Notes |
|---|---|---|
| `pgvector` | Vector similarity search | Enhanced with ScaNN index in AlloyDB |
| `pg_stat_statements` | Query performance tracking | Enabled by default |
| `pg_cron` | In-database cron job scheduling | Schedule SQL jobs without external tools |
| `PostGIS` | Geospatial data types and functions | Full PostGIS support |
| `uuid-ossp` | UUID generation functions | `uuid_generate_v4()` |
| `btree_gin` | GIN indexes for B-tree types | Composite index support |
| `pg_trgm` | Trigram similarity search | Full-text fuzzy matching |
| `hstore` | Key-value pairs in a column | Flexible schema fields |
| `pgcrypto` | Cryptographic functions | `crypt()`, `pgp_sym_encrypt()` |
| `pgaudit` | Statement-level audit logging | Compliance auditing |
| `auto_explain` | Automatic slow query plan logging | Query diagnosis |
| `tablefunc` | Pivot/crosstab functions | `crosstab()` |
| `google_ml_integration` | Vertex AI ML inference | AlloyDB-specific |
| `alloydb_scann` | ScaNN high-performance vector index | AlloyDB-specific |

### AlloyDB-Specific SQL Functions

```sql
-- Generate embeddings via Vertex AI (requires google_ml_integration)
SELECT embedding('text-embedding-004', 'What is AlloyDB?');

-- Perform ML prediction from a Vertex AI model endpoint
SELECT ml_predict_row(
  'projects/my-project/locations/us-central1/endpoints/my-endpoint',
  '{"instances": [{"feature1": 1.0, "feature2": 2.0}]}'
);
```

### Enabling Extensions

```sql
-- List all available extensions
SELECT name, default_version, installed_version
FROM pg_available_extensions
ORDER BY name;

-- Enable pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- Enable PostGIS
CREATE EXTENSION IF NOT EXISTS postgis;

-- Enable pg_cron
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Enable pgaudit (also requires database flag: cloudsql.enable_pgaudit=on)
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- Enable google_ml_integration
CREATE EXTENSION IF NOT EXISTS google_ml_integration CASCADE;
```

### Unsupported / Limited Features

| Feature | Status | Alternative |
|---|---|---|
| Logical replication (as publisher) | Limited | Use DMS for CDC |
| Custom background workers | Not supported | Use pg_cron for scheduling |
| `LOAD` command (custom shared libraries) | Not supported | Use built-in extensions |
| Superuser access | Not granted to customers | `alloydbsuperuser` role provided instead |
| `pg_upgrade` in-place | Not applicable | Use DMS for major version migration |

---

## 8. AlloyDB AI & Vector Search (pgvector)

### Setup

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Enable AlloyDB ScaNN extension (high-performance ANN index)
CREATE EXTENSION IF NOT EXISTS alloydb_scann;

-- Create a table with a vector column (1536 dimensions for text-embedding-004)
CREATE TABLE documents (
    id          BIGSERIAL PRIMARY KEY,
    content     TEXT       NOT NULL,
    metadata    JSONB,
    embedding   VECTOR(768) -- dimensions must match your embedding model
);
```

### Inserting Embeddings

```sql
-- Insert with pre-computed embedding from application
INSERT INTO documents (content, metadata, embedding)
VALUES (
    'AlloyDB is a PostgreSQL-compatible database',
    '{"source": "docs", "category": "database"}',
    '[0.021, -0.033, 0.089, ...]'::vector   -- 768-dim vector literal
);

-- Generate embedding inline using Vertex AI (google_ml_integration)
INSERT INTO documents (content, metadata, embedding)
VALUES (
    'AlloyDB supports vector search natively',
    '{"source": "docs"}',
    embedding('text-embedding-004', 'AlloyDB supports vector search natively')
);
```

### Vector Similarity Search Operators

| Operator | Distance Metric | Use Case |
|---|---|---|
| `<->` | L2 (Euclidean) distance | Geometric similarity |
| `<=>` | Cosine similarity (1 - cosine) | Text embeddings (most common) |
| `<#>` | Negative inner product | When vectors are normalised |

```sql
-- Cosine similarity search — find top 5 most similar documents
SELECT id, content, metadata,
       (embedding <=> '[0.021, -0.033, ...]'::vector) AS distance
FROM documents
ORDER BY embedding <=> '[0.021, -0.033, ...]'::vector
LIMIT 5;

-- With metadata filter (pre-filter then rerank)
SELECT id, content, distance
FROM (
    SELECT id, content,
           (embedding <=> query_vec) AS distance
    FROM documents,
    LATERAL (SELECT embedding('text-embedding-004', 'my search query') AS query_vec) q
    WHERE metadata->>'source' = 'docs'
    ORDER BY distance
    LIMIT 20
) ranked
LIMIT 5;
```

### Vector Indexes

```sql
-- IVFFlat index (faster build, lower recall — good baseline)
CREATE INDEX ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);   -- lists ≈ sqrt(row_count)

-- HNSW index (slower build, higher recall — better for < 1M vectors)
CREATE INDEX ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- ScaNN index (AlloyDB-specific — fastest for > 1M vectors)
CREATE INDEX ON documents
USING scann (embedding vector_cosine_ops)
WITH (num_leaves = 500);

-- Set ef_search for HNSW at query time (trade recall vs speed)
SET hnsw.ef_search = 100;
```

### Vertex AI Embedding Integration

```sql
-- Grant permissions to use Vertex AI (run as alloydbsuperuser)
GRANT EXECUTE ON FUNCTION embedding TO my_app_user;

-- Generate embeddings directly in SQL
SELECT
    id,
    content,
    embedding('text-embedding-004', content) AS vec
FROM documents
WHERE id = 1;

-- Batch embed on insert (use in ETL pipeline)
UPDATE documents
SET embedding = embedding('text-embedding-004', content)
WHERE embedding IS NULL;
```

### RAG (Retrieval Augmented Generation) Pattern

```python
import psycopg2
import vertexai
from vertexai.language_models import TextEmbeddingModel, TextGenerationModel

# Step 1: Embed the user query
model = TextEmbeddingModel.from_pretrained("text-embedding-004")
query_embedding = model.get_embeddings(["What is AlloyDB?"])[0].values

# Step 2: Retrieve relevant documents from AlloyDB
conn = psycopg2.connect("host=127.0.0.1 port=5432 dbname=mydb user=myuser")
with conn.cursor() as cur:
    cur.execute("""
        SELECT content, metadata,
               (embedding <=> %s::vector) AS distance
        FROM documents
        ORDER BY embedding <=> %s::vector
        LIMIT 5
    """, (query_embedding, query_embedding))
    context_docs = cur.fetchall()

# Step 3: Generate answer using retrieved context
context = "\n".join([doc[0] for doc in context_docs])
llm = TextGenerationModel.from_pretrained("text-bison")
response = llm.predict(
    f"Context:\n{context}\n\nQuestion: What is AlloyDB?\nAnswer:"
)
print(response.text)
```

---

## 9. Columnar Engine

### What Is the Columnar Engine?

The columnar engine is an **adaptive in-memory columnar cache** that coexists with AlloyDB's row store. It automatically detects analytical query patterns and transparently caches frequently scanned columns in a columnar format — no ETL, no separate tables, no configuration required.

```
Query: SELECT region, SUM(amount), COUNT(*) FROM orders GROUP BY region

Without columnar engine:          With columnar engine:
Read all columns for all rows →   Read only "region" + "amount" columns
Sequential page scan               from columnar cache →
(high I/O, slow)                  100× faster (OLAP workloads)
```

### Enabling & Configuring

```sql
-- Check if columnar engine is enabled (enabled by default)
SHOW google_columnar_engine.enabled;

-- Manually pin a column into the columnar cache
SELECT google_columnar_engine_add('orders', 'amount');
SELECT google_columnar_engine_add('orders', 'region');

-- Remove a column from columnar cache
SELECT google_columnar_engine_delete('orders', 'amount');

-- List columns currently in the columnar cache
SELECT * FROM g_columnar_columns;

-- Check columnar engine memory usage
SELECT * FROM g_columnar_heap_info;

-- Session-level hint: force columnar engine for this query
SET google_columnar_engine.use_columnar_scan = on;

-- Disable columnar engine for a specific query
SET google_columnar_engine.use_columnar_scan = off;
```

### Queries That Benefit (and Don't)

| Query Pattern | Columnar Engine? | Why |
|---|---|---|
| `SELECT SUM(amount) FROM orders` | ✅ Yes | Full column scan → columnar optimal |
| `SELECT region, COUNT(*) GROUP BY region` | ✅ Yes | Aggregation on single column |
| `SELECT * FROM orders WHERE date > '2025-01-01'` (large scan) | ✅ Yes | Range scan on many rows |
| `SELECT * FROM orders WHERE id = 12345` | ✗ No | Point lookup — row store faster |
| `INSERT / UPDATE / DELETE` | ✗ No | DML always uses row store |
| `SELECT * FROM orders LIMIT 10` | ✗ No | Small result — row store optimal |

### EXPLAIN Output with Columnar Scan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT region, SUM(amount), COUNT(*)
FROM orders
WHERE order_date >= '2025-01-01'
GROUP BY region;

-- Look for "Custom Scan (columnar scan)" in the output:
-- Custom Scan (columnar scan) on orders
--   Output: region, amount
--   Columnar cache hit ratio: 98.7%
--   Rows scanned: 10,000,000
--   (actual time=0.032..1.241 rows=8 loops=1)
```

### Monitoring Columnar Engine

```sql
-- Overall columnar cache statistics
SELECT * FROM g_columnar_heap_info;

-- Per-column cache status
SELECT table_name, column_name, state, size_bytes
FROM g_columnar_columns;

-- Query-level columnar usage
SELECT query, columnar_scans, row_scans
FROM pg_stat_statements
JOIN g_columnar_query_stats USING (queryid)
ORDER BY columnar_scans DESC
LIMIT 10;
```

### HTAP Pattern

AlloyDB enables **Hybrid Transactional/Analytical Processing** on a single database:

```sql
-- OLTP: concurrent transactional write (row store)
BEGIN;
INSERT INTO orders (customer_id, amount, region) VALUES (123, 49.99, 'US-WEST');
COMMIT;

-- OLAP: real-time analytical query on the same table (columnar cache)
-- No ETL, no data sync, no separate data warehouse needed for moderate analytics
SELECT
    region,
    DATE_TRUNC('month', order_date) AS month,
    SUM(amount) AS revenue,
    COUNT(*) AS order_count
FROM orders
GROUP BY region, month
ORDER BY month DESC, revenue DESC;
```

---

## 10. High Availability & Failover

### REGIONAL vs ZONAL Architecture

```
REGIONAL (HA) — recommended for production:

  Zone: us-central1-a              Zone: us-central1-b
  ┌──────────────────┐             ┌──────────────────┐
  │   PRIMARY        │             │   STANDBY        │
  │   (active)       │             │   (hot standby)  │
  │   Handles all    │             │   Attached to    │
  │   reads/writes   │             │   shared storage │
  └──────────────────┘             └──────────────────┘
          │                                │
          └──── Shared distributed ────────┘
                storage layer

  Failover: Standby already has storage access → promote in < 60s
  (No data copy needed — storage is shared)

ZONAL — dev/test only:

  Zone: us-central1-a only
  ┌──────────────────┐
  │   PRIMARY        │  ← No standby; zone failure = downtime
  │   (active)       │
  └──────────────────┘
```

### Failover Timeline

```
T+0s:   Primary becomes unhealthy (crash, zone outage)
T+0-5s: AlloyDB health checks detect failure
T+5-30s: Standby instance promoted to primary (shared storage — no copy)
T+30-60s: DNS/VIP updated; clients reconnect
T+60s+: Service restored; new standby being provisioned
```

### Manual Switchover (Planned Failover)

```bash
# Trigger a planned switchover (primary → standby promotion)
# Use for maintenance or testing HA; minimal downtime
gcloud alloydb instances failover my-primary \
  --region=us-central1 \
  --cluster=my-cluster
```

### Python Retry Pattern for Failover

```python
import psycopg2
import time
from psycopg2 import OperationalError

DSN = "host=127.0.0.1 port=5432 dbname=mydb user=myuser password=secret"

def get_connection(retries: int = 10, delay: float = 2.0):
    """Retry connection during AlloyDB failover (~60s window)."""
    for attempt in range(retries):
        try:
            conn = psycopg2.connect(DSN, connect_timeout=5)
            conn.autocommit = False
            return conn
        except OperationalError as e:
            if attempt == retries - 1:
                raise
            wait = min(delay * (1.5 ** attempt), 30)
            print(f"Connection failed (attempt {attempt+1}): {e}. Retrying in {wait:.1f}s")
            time.sleep(wait)

def execute_with_retry(query: str, params=None):
    """Execute query with connection retry on failover."""
    conn = get_connection()
    try:
        with conn.cursor() as cur:
            cur.execute(query, params)
            conn.commit()
            return cur.fetchall() if cur.description else None
    except OperationalError:
        conn.close()
        # Reconnect after failover
        conn = get_connection()
        with conn.cursor() as cur:
            cur.execute(query, params)
            conn.commit()
    finally:
        conn.close()
```

### AlloyDB vs Cloud SQL HA Comparison

| Metric | AlloyDB REGIONAL | Cloud SQL HA |
|---|---|---|
| Failover time | < 60 seconds | 60–120 seconds |
| Failover mechanism | Storage-native promotion (no data copy) | Replica promotion + WAL apply |
| Storage during failover | Shared (standby already has access) | Secondary has own storage copy |
| Read replicas during primary failover | Continue serving reads | Brief disruption |
| Cost of standby | Included in primary pricing | Separate instance charge |

---

## 11. Backups & Point-In-Time Recovery

### Backup Types

| Type | Trigger | Retention | Use Case |
|---|---|---|---|
| **Automated backup** | Daily (configurable window) | 1–365 days | Scheduled disaster recovery |
| **On-demand backup** | Manual (`gcloud alloydb backups create`) | Until manually deleted | Pre-migration, pre-maintenance snapshot |
| **Continuous WAL (PITR)** | Always-on (WAL archiving) | 1–35 days | Recovery to any second |

> ⚠️ **Warning:** Restoring a backup or performing PITR **creates a NEW cluster** — it does not overwrite the existing cluster. You must update your application's connection string to point to the restored cluster.

### Backup Management CLI

```bash
# Create an on-demand backup
gcloud alloydb backups create my-backup-20250115 \
  --cluster=my-cluster \
  --region=us-central1 \
  --description="Pre-migration snapshot"

# List backups
gcloud alloydb backups list --region=us-central1

# Describe a backup
gcloud alloydb backups describe my-backup-20250115 --region=us-central1

# Delete a backup
gcloud alloydb backups delete my-backup-20250115 --region=us-central1

# Update backup retention period on cluster
gcloud alloydb clusters update my-cluster \
  --region=us-central1 \
  --automated-backup-retention-period=30d \
  --backup-window-start=03:00
```

### Restoring from a Backup

```bash
# Restore backup to a NEW cluster
gcloud alloydb clusters restore my-cluster-restored \
  --source-backup=projects/my-project/locations/us-central1/backups/my-backup-20250115 \
  --region=us-central1 \
  --network=projects/my-project/global/networks/default

# Then create a primary instance in the restored cluster
gcloud alloydb instances create my-restored-primary \
  --instance-type=PRIMARY \
  --cpu-count=8 \
  --region=us-central1 \
  --cluster=my-cluster-restored \
  --availability-type=REGIONAL
```

### Point-In-Time Recovery (PITR)

```bash
# Restore cluster to a specific point in time (creates a new cluster)
gcloud alloydb clusters restore my-cluster-pitr \
  --source-cluster=projects/my-project/locations/us-central1/clusters/my-cluster \
  --restore-point-in-time=2025-01-15T10:30:00Z \
  --region=us-central1 \
  --network=projects/my-project/global/networks/default

# Enable/configure continuous backup (PITR)
gcloud alloydb clusters update my-cluster \
  --region=us-central1 \
  --enable-continuous-backup \
  --continuous-backup-retention-period=14   # days; max 35
```

### Backup Encryption

```bash
# Create cluster with CMEK backup encryption
gcloud alloydb clusters create my-cmek-cluster \
  --region=us-central1 \
  --network=projects/my-project/global/networks/default \
  --database-version=POSTGRES_15 \
  --password=StrongPassword123! \
  --kms-key=projects/my-project/locations/us-central1/keyRings/my-ring/cryptoKeys/my-key
```

---

## 12. Security — IAM, Auth & Encryption

### IAM Roles

| Role | Resource Level | Permissions |
|---|---|---|
| `roles/alloydb.admin` | Project | Full control: create, update, delete, manage all AlloyDB resources |
| `roles/alloydb.editor` | Project | Create/update clusters and instances; no delete |
| `roles/alloydb.client` | Project | Connect via Auth Proxy; required for all applications |
| `roles/alloydb.viewer` | Project | View cluster/instance metadata; no data or connection access |
| `roles/alloydb.databaseUser` | Project | IAM database user — can be granted DB-level privileges |

```bash
# Grant alloydb.client to app service account
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-app@my-project.iam.gserviceaccount.com" \
  --role="roles/alloydb.client"

# Grant alloydb.databaseUser for IAM DB auth
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-app@my-project.iam.gserviceaccount.com" \
  --role="roles/alloydb.databaseUser"
```

### IAM Database Authentication

```bash
# Enable IAM authentication on the cluster (database flag)
gcloud alloydb instances update my-primary \
  --database-flags=alloydb.iam_authentication=on \
  --region=us-central1 \
  --cluster=my-cluster

# Create an IAM user in the database
gcloud alloydb users create my-app@my-project.iam \
  --cluster=my-cluster \
  --region=us-central1 \
  --type=CLOUD_IAM_SERVICE_ACCOUNT

# Create an IAM human user
gcloud alloydb users create user@example.com \
  --cluster=my-cluster \
  --region=us-central1 \
  --type=CLOUD_IAM_USER
```

```sql
-- Grant database privileges to IAM user
GRANT CONNECT ON DATABASE mydb TO "my-app@my-project.iam";
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public
  TO "my-app@my-project.iam";
```

### SSL/TLS Modes

| Mode | Description | Use Case |
|---|---|---|
| `ALLOW_UNENCRYPTED_AND_ENCRYPTED` | Both SSL and non-SSL allowed | Dev/test only |
| `ENCRYPTED_ONLY` | SSL required; all connections encrypted | Production default |
| `REQUIRE_TRUSTED_CLIENT_CERT` | Client must present a certificate | mTLS; highest security |

### Row-Level Security (RLS)

```sql
-- Enable RLS on a table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Create policy: users can only see their own orders
CREATE POLICY orders_isolation ON orders
    FOR ALL
    TO PUBLIC
    USING (customer_id = current_setting('app.current_user_id')::bigint);

-- Policy for admin role (bypass RLS)
CREATE POLICY orders_admin ON orders
    FOR ALL
    TO admin_role
    USING (true);

-- Set the application user context before queries
SET app.current_user_id = '12345';
SELECT * FROM orders;  -- returns only rows for customer 12345
```

### pgaudit for Compliance Logging

```bash
# Enable pgaudit via database flag
gcloud alloydb instances update my-primary \
  --database-flags=pgaudit.log=all,pgaudit.log_catalog=on \
  --region=us-central1 \
  --cluster=my-cluster
```

```sql
-- Enable pgaudit extension
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- Configure audit for specific role
ALTER ROLE my_app_user SET pgaudit.log = 'read, write';

-- View recent audit log entries (in Cloud Logging)
-- Logs appear at: projects/my-project/logs/cloudaudit.googleapis.com%2Fdata_access
```

### CMEK

```bash
# Create KMS key for AlloyDB
gcloud kms keyrings create alloydb-keyring --location=us-central1
gcloud kms keys create alloydb-key \
  --location=us-central1 \
  --keyring=alloydb-keyring \
  --purpose=encryption

# Grant AlloyDB service account access to the key
gcloud kms keys add-iam-policy-binding alloydb-key \
  --location=us-central1 \
  --keyring=alloydb-keyring \
  --member="serviceAccount:service-PROJECT_NUMBER@gcp-sa-alloydb.iam.gserviceaccount.com" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"
```

---

## 13. Performance Tuning & Best Practices

### AlloyDB-Specific Tuning

```sql
-- Check current PostgreSQL settings
SHOW max_connections;
SHOW shared_buffers;
SHOW work_mem;

-- Update flags via gcloud (cannot SET most parameters directly in AlloyDB)
-- gcloud alloydb instances update my-primary \
--   --database-flags=max_connections=500,work_mem=16384 \
--   --region=us-central1 --cluster=my-cluster

-- Check autovacuum status
SELECT schemaname, tablename, last_autovacuum, last_autoanalyze,
       n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 10;
```

### Index Types and When to Use Them

```sql
-- B-tree (default) — equality and range queries
CREATE INDEX idx_orders_customer ON orders (customer_id);

-- Partial index — reduces index size for filtered queries
CREATE INDEX idx_orders_pending ON orders (created_at)
WHERE status = 'pending';

-- Covering index — include non-key columns to avoid heap fetch
CREATE INDEX idx_orders_covering ON orders (customer_id)
INCLUDE (amount, status, created_at);

-- GIN — for JSONB, array, and full-text search
CREATE INDEX idx_orders_metadata ON orders USING GIN (metadata);
CREATE INDEX idx_orders_fts ON orders USING GIN (to_tsvector('english', description));

-- GiST — for geospatial, range types
CREATE INDEX idx_locations_geo ON locations USING GIST (coordinates);

-- BRIN — for naturally ordered large tables (e.g., append-only time-series)
CREATE INDEX idx_events_time_brin ON events USING BRIN (event_time)
WITH (pages_per_range = 128);

-- Analyse index usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC   -- low scans = potentially unused index
LIMIT 10;
```

### Query Analysis

```sql
-- Enable pg_stat_statements (enabled by default in AlloyDB)
-- Top 10 slowest queries by mean execution time
SELECT
    LEFT(query, 80) AS query_snippet,
    calls,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round(total_exec_time::numeric / 1000, 2) AS total_sec,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows / calls AS avg_rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- EXPLAIN ANALYZE a slow query
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT c.name, SUM(o.amount)
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.created_at >= NOW() - INTERVAL '30 days'
GROUP BY c.name
ORDER BY 2 DESC;
```

### Bulk Write Performance

```sql
-- Use COPY for bulk inserts (10–100× faster than multi-row INSERT)
COPY orders (customer_id, amount, status, created_at)
FROM '/tmp/orders.csv'
WITH (FORMAT csv, HEADER true);

-- Multi-row INSERT (batch — acceptable for moderate volumes)
INSERT INTO orders (customer_id, amount, status)
VALUES
    (1, 49.99, 'pending'),
    (2, 29.99, 'confirmed'),
    (3, 99.99, 'shipped')
ON CONFLICT (id) DO UPDATE SET amount = EXCLUDED.amount;

-- Defer index maintenance for bulk loads
BEGIN;
ALTER TABLE orders DISABLE TRIGGER ALL;
-- ... bulk insert ...
ALTER TABLE orders ENABLE TRIGGER ALL;
COMMIT;
ANALYZE orders;
```

### Partitioning Strategies

```sql
-- Range partitioning (time-series — most common)
CREATE TABLE orders (
    id          BIGSERIAL,
    customer_id BIGINT,
    amount      NUMERIC,
    order_date  DATE NOT NULL
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2025_q1 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');

CREATE TABLE orders_2025_q2 PARTITION OF orders
    FOR VALUES FROM ('2025-04-01') TO ('2025-07-01');

-- Hash partitioning (distribute load evenly)
CREATE TABLE users (
    id    BIGSERIAL,
    email TEXT
) PARTITION BY HASH (id);

CREATE TABLE users_0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE users_1 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 1);

-- Detach old partition (instant, no data movement)
ALTER TABLE orders DETACH PARTITION orders_2024_q1;
```

### Connection Pooling

```bash
# PgBouncer is the recommended pooler; run alongside Auth Proxy
# Auth Proxy handles IAM auth + TLS → PgBouncer → AlloyDB
docker run -d \
  --name pgbouncer \
  -e DATABASE_URL="postgres://user:pass@127.0.0.1:5432/mydb" \
  -e POOL_MODE=transaction \
  -e MAX_CLIENT_CONN=1000 \
  -e DEFAULT_POOL_SIZE=20 \
  -p 6432:6432 \
  bitnami/pgbouncer:latest
```

---

## 14. Migrations to AlloyDB

### Migration Paths

| Source | Method | Downtime | Notes |
|---|---|---|---|
| Cloud SQL for PostgreSQL | DMS (online migration) | Minimal (CDC cutover) | Same project/region simplest |
| Self-managed PostgreSQL | DMS + connectivity setup | Minimal | Requires network access to source |
| Amazon RDS PostgreSQL | DMS + VPN/Interconnect | Minimal | Network setup required |
| Amazon Aurora PostgreSQL | DMS + VPN/Interconnect | Minimal | Aurora-specific extensions must be replaced |
| Oracle | Schema Conversion Tool + DMS | Significant (schema conversion) | Complex; schema rewriting needed |
| MySQL | Schema Conversion Tool + DMS | Significant | Schema conversion required |

### Pre-Migration Checklist

```
Schema & Extensions:
  ✅ Audit source extensions — verify AlloyDB support
  ✅ Replace Aurora-specific functions (e.g., aurora_version())
  ✅ Review unsupported features (logical replication as publisher, custom workers)
  ✅ Check enum type usage and ordering

Connectivity:
  ✅ PSA set up in target VPC
  ✅ AlloyDB cluster and primary instance created
  ✅ Auth Proxy installed and tested
  ✅ Connection strings updated in application config

Data Validation:
  ✅ Row count comparison script prepared
  ✅ Checksum validation for critical tables
  ✅ Application-level smoke test queries prepared

Performance:
  ✅ Target AlloyDB sized appropriately (match or exceed source)
  ✅ Indexes recreated on target
  ✅ ANALYZE run on all tables after migration
```

### DMS Migration Setup

```bash
# Create a DMS connection profile for the source (Cloud SQL)
gcloud database-migration connection-profiles create source-cloudsql \
  --region=us-central1 \
  --type=CLOUDSQL \
  --cloudsql-instance=projects/my-project/instances/my-cloudsql

# Create a DMS connection profile for AlloyDB target
gcloud database-migration connection-profiles create target-alloydb \
  --region=us-central1 \
  --type=ALLOYDB \
  --alloydb-cluster=projects/my-project/locations/us-central1/clusters/my-cluster

# Create a migration job (continuous replication — minimal downtime)
gcloud database-migration migration-jobs create my-migration \
  --region=us-central1 \
  --type=CONTINUOUS \
  --source=source-cloudsql \
  --destination=target-alloydb \
  --dump-type=LOGICAL

# Start the migration job
gcloud database-migration migration-jobs start my-migration --region=us-central1

# Monitor migration status
gcloud database-migration migration-jobs describe my-migration --region=us-central1

# Promote (cutover) — stops replication and makes AlloyDB writable
gcloud database-migration migration-jobs promote my-migration --region=us-central1
```

### Minimal-Downtime Migration Pattern

```
1. Create AlloyDB cluster and set up Auth Proxy
2. Create DMS migration job (CONTINUOUS type)
3. Initial full dump loads into AlloyDB (background)
4. CDC (Change Data Capture) replication catches up
5. Monitor lag → wait until lag < 5 seconds
6. Application maintenance window starts
7. Stop writes to source DB (app → read-only mode)
8. Wait for lag = 0 on DMS console
9. Promote DMS job (AlloyDB becomes writable)
10. Update connection strings → restart app
11. Validate row counts and smoke tests
12. Maintenance window ends
```

### Post-Migration Validation

```sql
-- Compare row counts (run on both source and target)
SELECT
    schemaname,
    relname AS table_name,
    n_live_tup AS estimated_rows
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;

-- Verify indexes were migrated
SELECT indexname, indexdef
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY tablename, indexname;

-- Run ANALYZE to update statistics
ANALYZE VERBOSE;

-- Check for invalid indexes
SELECT indexname, tablename
FROM pg_indexes
WHERE indexname IN (
    SELECT idxname FROM pg_catalog.pg_index i
    JOIN pg_catalog.pg_class c ON c.oid = i.indrelid
    WHERE NOT i.indisvalid
);
```

---

## 15. Client Library — Python

### Installation

```bash
pip install psycopg2-binary asyncpg sqlalchemy pgvector
```

### Auth Proxy Connection Hierarchy

```
Application code
    └── connects to → Auth Proxy (localhost:5432 or Unix socket)
                          └── authenticates via → IAM / password
                                └── proxies to → AlloyDB Primary (10.x.x.x:5432)
```

### psycopg2 — Sync Connections

```python
import psycopg2
from psycopg2 import pool, OperationalError
import io, os

# ── Connect via Auth Proxy (Unix socket — preferred) ──────────────────
conn = psycopg2.connect(
    host="/tmp/alloydb-proxy/my-project:us-central1:my-cluster",
    dbname="mydb",
    user="myuser",
    password=os.environ["DB_PASSWORD"],
)

# ── Connect via Auth Proxy (TCP) ──────────────────────────────────────
conn = psycopg2.connect(
    host="127.0.0.1",
    port=5432,
    dbname="mydb",
    user="myuser",
    password=os.environ["DB_PASSWORD"],
    sslmode="disable",  # Auth Proxy handles TLS; disable redundant SSL
)

# ── Connection pool ───────────────────────────────────────────────────
db_pool = pool.ThreadedConnectionPool(
    minconn=2,
    maxconn=20,
    host="127.0.0.1",
    port=5432,
    dbname="mydb",
    user="myuser",
    password=os.environ["DB_PASSWORD"],
    connect_timeout=5,
)

def get_conn():
    return db_pool.getconn()

def release_conn(conn):
    db_pool.putconn(conn)

# ── Parameterised query (always use %s to prevent SQL injection) ──────
with conn.cursor() as cur:
    cur.execute(
        "SELECT id, name, amount FROM orders WHERE customer_id = %s AND status = %s",
        (customer_id, "pending")
    )
    rows = cur.fetchall()

# ── Transaction with rollback ─────────────────────────────────────────
conn.autocommit = False
try:
    with conn.cursor() as cur:
        cur.execute("UPDATE accounts SET balance = balance - %s WHERE id = %s", (100, 1))
        cur.execute("UPDATE accounts SET balance = balance + %s WHERE id = %s", (100, 2))
    conn.commit()
except Exception as e:
    conn.rollback()
    raise

# ── Bulk insert with COPY (fastest method) ────────────────────────────
csv_data = "1,Alice,99.99\n2,Bob,49.99\n3,Carol,79.99\n"
with conn.cursor() as cur:
    cur.copy_expert(
        "COPY orders (customer_id, name, amount) FROM STDIN WITH (FORMAT csv)",
        io.StringIO(csv_data)
    )
conn.commit()
```

### asyncpg — Async Connections

```python
import asyncpg
import asyncio
import os

async def main():
    # ── Single async connection ───────────────────────────────────────
    conn = await asyncpg.connect(
        host="127.0.0.1",
        port=5432,
        database="mydb",
        user="myuser",
        password=os.environ["DB_PASSWORD"],
    )

    # ── Async connection pool (recommended for async apps) ────────────
    pool = await asyncpg.create_pool(
        dsn="postgresql://myuser:password@127.0.0.1:5432/mydb",
        min_size=2,
        max_size=20,
        command_timeout=30,
    )

    async with pool.acquire() as conn:
        rows = await conn.fetch(
            "SELECT * FROM orders WHERE customer_id = $1", 123
        )
        for row in rows:
            print(dict(row))

    await pool.close()

asyncio.run(main())
```

### SQLAlchemy — ORM & Connection Pool

```python
from sqlalchemy import create_engine, text
from sqlalchemy.orm import Session
import os

# ── Engine via Auth Proxy (Unix socket) ───────────────────────────────
engine = create_engine(
    "postgresql+psycopg2://myuser:password@/mydb"
    "?host=/tmp/alloydb-proxy/my-project:us-central1:my-cluster",
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,          # test connection health before use
    pool_recycle=1800,           # recycle connections every 30 min
    connect_args={"connect_timeout": 5},
)

# ── Engine via Auth Proxy (TCP) ────────────────────────────────────────
engine = create_engine(
    f"postgresql+psycopg2://myuser:{os.environ['DB_PASSWORD']}@127.0.0.1:5432/mydb",
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,
    pool_recycle=1800,
)

# ── Async engine (asyncpg + SQLAlchemy 2.0) ───────────────────────────
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

async_engine = create_async_engine(
    "postgresql+asyncpg://myuser:password@127.0.0.1:5432/mydb",
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,
)

# ── Session usage ──────────────────────────────────────────────────────
with Session(engine) as session:
    result = session.execute(
        text("SELECT id, amount FROM orders WHERE customer_id = :cid"),
        {"cid": 123}
    )
    orders = result.fetchall()
```

### Vector Search with pgvector

```python
from pgvector.psycopg2 import register_vector
import psycopg2
import numpy as np

conn = psycopg2.connect("host=127.0.0.1 port=5432 dbname=mydb user=myuser")
register_vector(conn)   # register vector type with psycopg2

# Insert a vector
embedding = np.array([0.1, 0.2, 0.3, ...])  # 768-dim numpy array
with conn.cursor() as cur:
    cur.execute(
        "INSERT INTO documents (content, embedding) VALUES (%s, %s)",
        ("my document text", embedding)
    )
conn.commit()

# Vector similarity search
query_embedding = np.array([0.11, 0.19, 0.31, ...])
with conn.cursor() as cur:
    cur.execute("""
        SELECT id, content,
               (embedding <=> %s) AS distance
        FROM documents
        ORDER BY embedding <=> %s
        LIMIT 10
    """, (query_embedding, query_embedding))
    results = cur.fetchall()
    for doc_id, content, distance in results:
        print(f"[{distance:.4f}] {content[:80]}")
```

### IAM Authentication Connection

```python
import google.auth
import google.auth.transport.requests
import psycopg2

# Get IAM token (auto-refreshed by Google Auth library)
credentials, project = google.auth.default(
    scopes=["https://www.googleapis.com/auth/cloud-platform"]
)
credentials.refresh(google.auth.transport.requests.Request())

# Use IAM token as password with Auth Proxy (--auto-iam-authn)
conn = psycopg2.connect(
    host="127.0.0.1",
    port=5432,
    dbname="mydb",
    user="my-app@my-project.iam",   # IAM service account email
    password=credentials.token,     # short-lived IAM token
    sslmode="disable",
)
```

---

## 16. gcloud CLI Quick Reference

### Cluster Operations

```bash
# CREATE cluster
gcloud alloydb clusters create CLUSTER \
  --region=REGION \
  --network=NETWORK \
  --database-version=POSTGRES_15 \
  --password=PASSWORD

# LIST clusters
gcloud alloydb clusters list --region=REGION

# DESCRIBE cluster
gcloud alloydb clusters describe CLUSTER --region=REGION

# UPDATE cluster (backup config, deletion protection)
gcloud alloydb clusters update CLUSTER --region=REGION \
  --automated-backup-retention-period=30d \
  --deletion-protection

# DELETE cluster (force removes all instances)
gcloud alloydb clusters delete CLUSTER --region=REGION --force
```

### Instance Operations

```bash
# CREATE primary instance
gcloud alloydb instances create INSTANCE \
  --instance-type=PRIMARY \
  --cpu-count=8 \
  --region=REGION \
  --cluster=CLUSTER \
  --availability-type=REGIONAL

# CREATE read pool
gcloud alloydb instances create READ_POOL \
  --instance-type=READ_POOL \
  --cpu-count=4 \
  --read-pool-node-count=3 \
  --region=REGION \
  --cluster=CLUSTER

# LIST instances
gcloud alloydb instances list --region=REGION --cluster=CLUSTER

# DESCRIBE instance (get IP address, state)
gcloud alloydb instances describe INSTANCE \
  --region=REGION --cluster=CLUSTER

# UPDATE CPU count (online for primary)
gcloud alloydb instances update INSTANCE \
  --cpu-count=16 --region=REGION --cluster=CLUSTER

# UPDATE database flags
gcloud alloydb instances update INSTANCE \
  --database-flags=max_connections=500,work_mem=16384 \
  --region=REGION --cluster=CLUSTER

# FAILOVER (manual switchover)
gcloud alloydb instances failover INSTANCE \
  --region=REGION --cluster=CLUSTER

# DELETE instance
gcloud alloydb instances delete INSTANCE \
  --region=REGION --cluster=CLUSTER
```

### Backup Operations

```bash
# CREATE on-demand backup
gcloud alloydb backups create BACKUP \
  --cluster=CLUSTER --region=REGION

# LIST backups
gcloud alloydb backups list --region=REGION

# RESTORE from backup (creates new cluster)
gcloud alloydb clusters restore NEW_CLUSTER \
  --source-backup=projects/PROJECT/locations/REGION/backups/BACKUP \
  --region=REGION --network=NETWORK

# PITR restore (creates new cluster)
gcloud alloydb clusters restore NEW_CLUSTER \
  --source-cluster=projects/PROJECT/locations/REGION/clusters/CLUSTER \
  --restore-point-in-time=TIMESTAMP \
  --region=REGION --network=NETWORK

# DELETE backup
gcloud alloydb backups delete BACKUP --region=REGION
```

### User Management

```bash
# CREATE built-in user (password auth)
gcloud alloydb users create myuser \
  --cluster=CLUSTER --region=REGION \
  --password=StrongPassword123! \
  --type=BUILT_IN

# CREATE IAM user (service account)
gcloud alloydb users create sa@project.iam \
  --cluster=CLUSTER --region=REGION \
  --type=CLOUD_IAM_SERVICE_ACCOUNT

# CREATE IAM user (human user)
gcloud alloydb users create user@example.com \
  --cluster=CLUSTER --region=REGION \
  --type=CLOUD_IAM_USER

# LIST users
gcloud alloydb users list --cluster=CLUSTER --region=REGION

# DELETE user
gcloud alloydb users delete myuser --cluster=CLUSTER --region=REGION

# SET password
gcloud alloydb users set-password myuser \
  --cluster=CLUSTER --region=REGION \
  --password=NewPassword123!
```

### Secondary Cluster Operations

```bash
# CREATE secondary cluster
gcloud alloydb clusters create SECONDARY \
  --region=SECONDARY_REGION \
  --network=NETWORK \
  --primary-cluster=projects/PROJECT/locations/PRIMARY_REGION/clusters/PRIMARY

# PROMOTE secondary to primary
gcloud alloydb clusters promote SECONDARY --region=SECONDARY_REGION

# LIST operations
gcloud alloydb operations list --region=REGION
gcloud alloydb operations describe OPERATION_ID --region=REGION
```

---

## 17. Monitoring & Observability

### Key Metrics Reference

| Metric | Alert Threshold | Description |
|---|---|---|
| `alloydb.googleapis.com/instance/cpu/utilization` | > 0.80 | CPU usage fraction; consider scaling vCPUs |
| `alloydb.googleapis.com/instance/memory/components/cache` | — | Buffer cache hit ratio; low = I/O bound queries |
| `alloydb.googleapis.com/instance/postgres/insights/aggregate/execution_count` | — | Total query executions per second |
| `alloydb.googleapis.com/instance/postgres/insights/aggregate/latencies` | p99 > 500ms | Query latency percentiles |
| `alloydb.googleapis.com/instance/replication/replica_lag` | > 30s | Replication delay to secondary cluster |
| `alloydb.googleapis.com/instance/storage/used_bytes` | > 80% of planned | Storage growth rate monitoring |
| `alloydb.googleapis.com/instance/postgres/connections` | > 80% of max | Connected clients vs `max_connections` |
| `alloydb.googleapis.com/instance/memory/components/usage` | > 0.90 | Overall memory pressure |

### Query Insights

AlloyDB includes a built-in **Query Insights** dashboard in the GCP Console (AlloyDB → your instance → **Query Insights** tab) with:

- Top queries ranked by CPU, latency, and execution count
- Query plan visualisation per query
- Latency percentiles (p50, p95, p99)
- Lock wait analysis
- Tag-based query grouping (using `pg_query_tag`)

```sql
-- Tag queries for Query Insights grouping
SET pg_query_tag = 'checkout-flow';
SELECT * FROM orders WHERE status = 'pending';
RESET pg_query_tag;
```

### pg_stat_statements Analysis

```sql
-- Reset statistics
SELECT pg_stat_statements_reset();

-- Top queries by total time
SELECT
    LEFT(query, 100) AS query,
    calls,
    round(total_exec_time::numeric, 0) AS total_ms,
    round(mean_exec_time::numeric, 2)  AS mean_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Queries with high variability (potential index issues)
SELECT LEFT(query, 80), calls,
       round(mean_exec_time::numeric, 2) AS mean_ms,
       round(stddev_exec_time::numeric, 2) AS stddev_ms,
       round(stddev_exec_time / NULLIF(mean_exec_time, 0) * 100, 1) AS cv_pct
FROM pg_stat_statements
WHERE calls > 100
ORDER BY cv_pct DESC LIMIT 10;
```

### Setting Up Alerts

```bash
# Alert: CPU > 80% for 5 minutes
gcloud alpha monitoring policies create \
  --display-name="AlloyDB High CPU" \
  --condition-display-name="CPU > 80%" \
  --condition-filter='resource.type="alloydb.googleapis.com/Instance"
    metric.type="alloydb.googleapis.com/instance/cpu/utilization"' \
  --condition-threshold-value=0.80 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=300s \
  --notification-channels=CHANNEL_ID

# Alert: Replica lag > 30 seconds
gcloud alpha monitoring policies create \
  --display-name="AlloyDB Replica Lag" \
  --condition-filter='metric.type="alloydb.googleapis.com/instance/replication/replica_lag"' \
  --condition-threshold-value=30 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=60s \
  --notification-channels=CHANNEL_ID
```

---

## 18. Pricing Model Summary

> Prices are approximate US regional rates as of 2025. Multi-region and other regions differ. Always verify at [cloud.google.com/alloydb/pricing](https://cloud.google.com/alloydb/pricing).

### Compute Pricing (per vCPU-hour)

| Instance Type | vCPU Price/hr | Memory (GB/hr) | Notes |
|---|---|---|---|
| Primary instance | $0.0815 / vCPU | $0.0137 / GB | Billed per vCPU and GB of RAM |
| Read pool (per node) | $0.0815 / vCPU | $0.0137 / GB | Same rate as primary; billed per node |
| Secondary cluster primary | $0.0815 / vCPU | $0.0137 / GB | Compute charged; storage shared |

### Storage & Backup Pricing

| Resource | Price |
|---|---|
| Storage (actual data used) | $0.10 / GiB / month |
| Automated backup storage | $0.088 / GiB / month |
| Continuous backup (PITR) | $0.088 / GiB / month (WAL storage) |
| On-demand backup storage | $0.088 / GiB / month |
| Cross-region replication egress | $0.08–$0.12 / GiB |

### Trial Cluster

- **Free for 30 days** — full AlloyDB features including REGIONAL HA
- One trial cluster per project per region
- After 30 days: auto-converts to billed cluster

### AlloyDB vs Cloud SQL PostgreSQL Enterprise Plus

| Configuration | AlloyDB | Cloud SQL Enterprise Plus | Notes |
|---|---|---|---|
| 8 vCPU, 64 GB RAM, 1 TB storage | ~$1,400/mo | ~$1,100/mo | AlloyDB ~25% higher compute |
| Failover time | < 60s | 60–120s | AlloyDB faster |
| Read replicas | Shared storage (cheaper) | Full storage copy (more expensive at scale) | AlloyDB cheaper at scale |
| Columnar engine | Included | Not available | AlloyDB unique |
| AI/vector search | Built-in ScaNN | pgvector only | AlloyDB better perf |

### Cost Optimisation Strategies

| Strategy | Savings |
|---|---|
| Right-size primary vCPUs (use Query Insights to find idle capacity) | Large reduction for over-provisioned instances |
| Use read pool instead of Cloud Spanner for read scale-out | Cheaper than separate database tier |
| Use columnar engine instead of exporting to BigQuery for light analytics | Eliminates BigQuery + ETL cost for moderate OLAP |
| Schedule heavy reports to read pool | Offload CPU from primary; avoid scaling primary |
| Set backup retention to match compliance need (not maximum) | Reduce backup storage cost |
| Use ZONAL for dev/test instances | ~No standby cost; 50% less than REGIONAL |
| Use trial cluster for POC | 30 days free |
| Autoscale read pool nodes based on load | Use node count 0 during off-hours (if read pool supports scale-to-zero) |

---

## 19. Common Errors & Troubleshooting

| Error / Issue | Cause | Fix |
|---|---|---|
| Connection refused / timeout | Auth Proxy not running; PSA not configured; Serverless VPC Access connector missing for Cloud Run | Start Auth Proxy; verify PSA peering (`gcloud services vpc-peerings list`); add VPC connector for serverless; check firewall rules |
| `FATAL: password authentication failed for user` | Wrong password; IAM user created in DB but no password set; `alloydb.iam_authentication` not enabled | Reset password via `gcloud alloydb users set-password`; enable IAM auth flag; verify user type (BUILT_IN vs IAM) |
| `FATAL: remaining connection slots are reserved for non-replication superuser connections` | `max_connections` exhausted | Increase `max_connections` flag; deploy PgBouncer connection pooler; audit for connection leaks; check connection pool `pool_pre_ping` |
| `ERROR: extension "X" does not exist` | Extension not created with `CREATE EXTENSION`; extension not on allowlist | Run `CREATE EXTENSION IF NOT EXISTS X`; verify extension is in `pg_available_extensions`; enable via database flags if required |
| Slow queries after migration | Missing indexes; stale table statistics; `work_mem` too low | Run `ANALYZE VERBOSE;`; recreate indexes from source; review `EXPLAIN ANALYZE` plans; tune `work_mem` |
| High replication lag on secondary cluster | Write-heavy primary saturating secondary CPU; network between regions congested | Increase vCPUs on secondary instance; monitor `replication/replica_lag` metric; avoid large batch operations during peak |
| Columnar engine not used for analytical query | Query not columnar-eligible (uses functions columnar engine can't handle); cache not yet warmed | Check `g_columnar_columns` — manually pin columns with `google_columnar_engine_add()`; verify with `EXPLAIN` for "Custom Scan (columnar scan)"; run workload to warm cache |
| PITR restore fails | WAL gap in archive (backup retention expired for requested timestamp); restore timestamp outside retention window | Check continuous backup retention period; choose a timestamp within the retention window; create a new on-demand backup before disabling PITR |
| IAM authentication errors (`pg_iam_auth` / token rejected) | `alloydb.iam_authentication` flag not set; service account not added as DB user; IAM token expired | Enable `alloydb.iam_authentication=on` flag; run `gcloud alloydb users create SA --type=CLOUD_IAM_SERVICE_ACCOUNT`; refresh token; verify `alloydb.client` IAM role |
| High autovacuum CPU | Table with very high delete/update rate; autovacuum not keeping up; bloat accumulating | Tune `autovacuum_vacuum_cost_delay` and `autovacuum_vacuum_scale_factor`; run manual `VACUUM ANALYZE`; consider table partitioning to limit vacuumed scope |
| `ERROR: invalid input syntax for type vector` | pgvector not enabled; vector literal format wrong | Run `CREATE EXTENSION IF NOT EXISTS vector`; use correct syntax `'[0.1, 0.2, ...]'::vector` |
| Auth Proxy exits immediately after start | Wrong instance URI format; service account lacks `alloydb.client` role; AlloyDB API not enabled | Verify URI format `projects/P/locations/R/clusters/C/instances/I`; grant `roles/alloydb.client`; enable `alloydb.googleapis.com` API |
| `ERROR: operator does not exist: vector <=> unknown` | pgvector operator requires explicit cast | Cast the literal: `embedding <=> '[0.1, ...]'::vector` |

### Diagnostic Queries

```sql
-- Active connections breakdown
SELECT state, count(*), usename
FROM pg_stat_activity
GROUP BY state, usename
ORDER BY count DESC;

-- Long-running queries (> 30 seconds)
SELECT pid, now() - query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start < now() - interval '30 seconds'
ORDER BY duration DESC;

-- Bloat estimation per table
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
       n_dead_tup,
       round(n_dead_tup::numeric / NULLIF(n_live_tup,0)*100, 1) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_pct DESC
LIMIT 10;

-- Lock wait analysis
SELECT
    blocked.pid     AS blocked_pid,
    blocked.query   AS blocked_query,
    blocking.pid    AS blocking_pid,
    blocking.query  AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;
```

```bash
# Check Auth Proxy connectivity
./alloydb-auth-proxy \
  --port=5432 \
  "projects/my-project/locations/us-central1/clusters/my-cluster/instances/my-primary" &
psql "host=127.0.0.1 port=5432 dbname=mydb user=myuser" -c "SELECT version();"

# Check AlloyDB instance status
gcloud alloydb instances describe my-primary \
  --region=us-central1 --cluster=my-cluster \
  --format="value(state)"

# List recent operations (to check for in-progress changes)
gcloud alloydb operations list --region=us-central1 \
  --filter="done=false"
```

---

## Quick Reference Card

```
Hierarchy:           Project → Cluster → Instance(s)
Instance types:      PRIMARY (read+write) | READ_POOL (read-only) | SECONDARY (DR)
Availability types:  REGIONAL (HA, < 60s failover) | ZONAL (no HA)
Storage:             Auto-scales 10 GB → 64 TB; billed per GiB used (not provisioned)
vCPU options:        2, 4, 8, 16, 32, 64, 96 vCPUs (8 GB RAM per vCPU)
Failover time:       < 60 seconds (REGIONAL) — faster than Cloud SQL due to shared storage
Read replicas:       Zero replication lag — shared distributed storage
Default port:        5432 (standard PostgreSQL)
TLS port override:   6378 when REQUIRE_TRUSTED_CLIENT_CERT (check per deployment)
Auth Proxy port:     5432 (TCP) or Unix socket /tmp/alloydb-proxy/...
Columnar engine:     Automatic HTAP; manually pin with google_columnar_engine_add()
pgvector ops:        <-> L2 | <=> cosine | <#> inner product
ScaNN index:         Best for > 1M vectors (AlloyDB-specific, faster than HNSW)
Backup retention:    1–365 days automated; PITR up to 35 days continuous WAL
Restore behaviour:   Always creates a NEW cluster (never overwrites existing)
CMEK:                Set at cluster creation time; cannot be changed later
max_connections:     Managed via database flag; default 1000; pool with PgBouncer
IAM auth:            alloydb.iam_authentication=on + create user with --type=CLOUD_IAM_*
No free tier:        Trial cluster: 30 days free; billing starts after
PG compatibility:    100% — supports PG 14, 15, 16; full DDL/DML/extensions
```

---

*Reference: [AlloyDB Docs](https://cloud.google.com/alloydb/docs) | [Pricing](https://cloud.google.com/alloydb/pricing) | [Auth Proxy](https://cloud.google.com/alloydb/docs/auth-proxy/overview) | [pgvector](https://cloud.google.com/alloydb/docs/ai/work-with-embeddings) | [DMS Migration](https://cloud.google.com/database-migration/docs/alloydb)*
