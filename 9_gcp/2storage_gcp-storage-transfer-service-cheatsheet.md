# 🔄 GCP Storage Transfer Service — Comprehensive Cheatsheet

> **Audience:** Developers & Cloud Engineers | **Last updated:** 2025
> A single-reference document covering theory, configuration, CLI, SDK, and best practices.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Transfer Sources & Destinations](#2-transfer-sources--destinations)
3. [Transfer Job Types & Configuration](#3-transfer-job-types--configuration)
4. [Agent Pools & On-Premises Transfers](#4-agent-pools--on-premises-transfers)
5. [Event-Driven Transfers](#5-event-driven-transfers)
6. [Scheduling & Filters](#6-scheduling--filters)
7. [IAM & Security](#7-iam--security)
8. [Monitoring, Logging & Notifications](#8-monitoring-logging--notifications)
9. [Client Library — Python](#9-client-library--python)
10. [gcloud CLI Quick Reference](#10-gcloud-cli-quick-reference)
11. [Pricing Model Summary](#11-pricing-model-summary)
12. [Common Errors & Troubleshooting](#12-common-errors--troubleshooting)

---

## 1. Overview & Key Concepts

**Google Cloud Storage Transfer Service (STS)** is a fully managed, scalable data movement service for ingesting, migrating, and synchronising large volumes of data into and between Google Cloud Storage buckets. It handles scheduling, retries, checksums, parallelism, and monitoring — removing the operational burden from self-managed copy scripts.

### How STS Fits in the GCP Data Movement Ecosystem

```
┌─────────────────────────────────────────────────────────────────────┐
│                  GCP Data Movement Landscape                        │
├───────────────────┬─────────────────┬───────────────┬──────────────┤
│  Storage Transfer │ gsutil / gcloud  │   Transfer    │  BigQuery    │
│     Service       │    storage cp    │   Appliance   │  Data Xfer   │
├───────────────────┼─────────────────┼───────────────┼──────────────┤
│ Managed service   │ CLI tool         │ Physical HW   │ SaaS sources │
│ TB–PB scale       │ GB–TB scale      │ PB offline    │ Analytics    │
│ Cross-cloud       │ GCS only (fast)  │ No network    │ BQ tables    │
│ Scheduled / event │ Manual / script  │ One-time      │ Recurring    │
│ Agent for on-prem │ Needs network    │ Ship device   │ No agents    │
│ Free service*     │ Free tool        │ Hardware fee  │ Free / paid  │
└───────────────────┴─────────────────┴───────────────┴──────────────┘
* Network egress costs still apply
```

### When to Use Storage Transfer Service

- Migrating **TB–PB** of data from AWS S3, Azure Blob, or HTTP sources to Cloud Storage
- **Scheduled recurring sync** from S3/Azure → GCS for cross-cloud pipelines
- **On-premises / POSIX / HDFS** data ingestion at scale using transfer agents
- **Near-real-time replication** between GCS buckets using event-driven transfers
- Any transfer needing **built-in retry, checksum validation, and audit logging**

### Core Terminology

| Term | Description |
|---|---|
| **Transfer Job** | Top-level resource defining source, destination, schedule, and filters. Has a stable job name. |
| **Transfer Run** | A single execution of a transfer job. Multiple runs can exist per job (one per schedule tick). |
| **Transfer Operation** | The underlying long-running operation executing a transfer run. Tracks bytes/objects moved. |
| **Transfer Spec** | The configuration block within a job defining what to transfer (source, sink, filters, options). |
| **Source** | Where data is read from (S3, Azure, GCS bucket, POSIX filesystem, HDFS, HTTP URL list). |
| **Sink** | The Cloud Storage bucket where data is written. Always a GCS bucket. |
| **Agent** | A Docker-based process installed on-premises or on a GCE VM that reads local data and sends it to GCS. Required for POSIX/HDFS sources. |
| **Agent Pool** | A named group of agents. All agents in a pool are treated as a single logical source. |
| **POSIX Transfer** | Transfer from a local filesystem (NFS, local disk, etc.) using agents. |
| **Event-Driven Transfer** | A transfer triggered by Pub/Sub notifications when new objects appear in a source bucket. |
| **Manifest File** | A CSV or TSV file listing specific objects to transfer. Enables selective transfers. |
| **Overwrite Condition** | Controls when destination objects are overwritten: always, never, or if different (by checksum). |
| **Delete Option** | Whether to delete objects from the source after transfer or delete destination objects not in source. |
| **Bandwidth Limit** | Cap on aggregate transfer throughput (MB/s) applied per agent pool or globally. |
| **STS Service Account** | Google-managed service account (`project-NNNN@storage-transfer-service.iam.gserviceaccount.com`) that performs transfers. Must be granted access to source and sink. |

---

## 2. Transfer Sources & Destinations

### Source → Destination Support Matrix

| Source | Destination | Supported | Mechanism | Key Limitations |
|---|---|---|---|---|
| **AWS S3** | Cloud Storage | ✅ | Agentless | Requires AWS access key + secret or IAM role |
| **Azure Blob Storage** | Cloud Storage | ✅ | Agentless | Requires SAS token or connection string |
| **HTTP/HTTPS URL list** | Cloud Storage | ✅ | Agentless | URLs must be publicly accessible; no auth supported |
| **Cloud Storage bucket** | Cloud Storage | ✅ | Agentless | Intra- and inter-project; inter-region egress costs apply |
| **POSIX filesystem** | Cloud Storage | ✅ | Agent-based | Agents must be installed on source host |
| **POSIX filesystem** | POSIX filesystem | ✅ | Agent-based | Both src and dst agents required; experimental |
| **HDFS** | Cloud Storage | ✅ | Agent-based | Agents run on HDFS cluster nodes |
| **AWS S3** | Cloud Storage (event-driven) | ✅ | Agentless + Pub/Sub | Requires S3 event → SQS → Pub/Sub bridge |
| **Cloud Storage** | Cloud Storage (event-driven) | ✅ | Agentless + Pub/Sub | Native GCS Pub/Sub notification |
| **Cloud Storage** | AWS S3 | ✗ | — | Not supported; use gsutil or custom pipeline |
| **Cloud Storage** | Azure Blob | ✗ | — | Not supported |
| **Cloud Storage** | POSIX filesystem | ✗ | — | Not supported by STS; use gsutil |
| **Azure** | AWS S3 | ✗ | — | Cloud Storage is always the sink |

> ⚠️ **Warning:** Cloud Storage is **always the sink** (destination) for Storage Transfer Service. STS cannot write to AWS S3, Azure, or other non-GCS destinations.

### Agentless vs Agent-Based Transfers

| Property | Agentless | Agent-Based |
|---|---|---|
| **Sources** | S3, Azure, HTTP, GCS | POSIX, HDFS |
| **Infrastructure needed** | None — fully managed | Docker agent on source machines |
| **Scaling** | Automatic | Proportional to agent count |
| **Network path** | Google's network → GCS | Agent machine → GCS |
| **On-premises support** | ✗ | ✅ |
| **Setup complexity** | Low | Medium |

---

## 3. Transfer Job Types & Configuration

### Job Types Overview

| Type | Trigger | Use Case |
|---|---|---|
| **One-time batch** | Runs once at a specified time (or immediately) | Migrations, one-off syncs |
| **Scheduled recurring** | Repeats on a defined schedule (daily, hourly, etc.) | Ongoing cross-cloud sync |
| **Event-driven** | Pub/Sub notification when new objects arrive | Near-real-time replication |

### Creating a One-Time GCS → GCS Transfer (CLI)

```bash
gcloud transfer jobs create \
  gs://source-bucket \
  gs://destination-bucket \
  --name="my-gcs-migration" \
  --source-agent-pool="" \
  --description="One-time full migration" \
  --no-async
```

### Creating a Scheduled S3 → GCS Transfer (CLI)

```bash
gcloud transfer jobs create \
  s3://my-aws-bucket \
  gs://my-gcs-bucket \
  --name="s3-to-gcs-daily" \
  --source-creds-file=aws-creds.json \
  --schedule-starts="2025-01-01T02:00:00Z" \
  --schedule-repeats-every=1d \
  --description="Daily S3 sync"
```

AWS credentials file format (`aws-creds.json`):

```json
{
  "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
  "secretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}
```

### Creating a Transfer via REST API

```bash
curl -X POST \
  "https://storagetransfer.googleapis.com/v1/transferJobs" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "transferJobs/my-s3-job",
    "projectId": "my-gcp-project",
    "transferSpec": {
      "awsS3DataSource": {
        "bucketName": "my-aws-bucket",
        "awsAccessKey": {
          "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
          "secretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
        }
      },
      "gcsDataSink": {
        "bucketName": "my-gcs-bucket"
      },
      "transferOptions": {
        "overwriteObjectsAlreadyExistingInSink": false,
        "deleteObjectsFromSourceAfterTransfer": false,
        "deleteObjectsUniqueInSink": false,
        "metadataOptions": {
          "storageClass": "PRESERVE",
          "timeCreated": "PRESERVE_AS_CUSTOM_TIME"
        }
      }
    },
    "schedule": {
      "scheduleStartDate": { "year": 2025, "month": 1, "day": 1 },
      "startTimeOfDay": { "hours": 2, "minutes": 0 }
    },
    "status": "ENABLED"
  }'
```

### Transfer Spec Configuration Options

#### Overwrite Conditions

| Setting | Behaviour |
|---|---|
| `overwriteObjectsAlreadyExistingInSink: false` | Skip objects that already exist at destination (default) |
| `overwriteObjectsAlreadyExistingInSink: true` | Always overwrite destination objects |
| `overwriteWhen: DIFFERENT` | Overwrite only if checksum or size differs |
| `overwriteWhen: NEVER` | Never overwrite — skip if destination object exists |
| `overwriteWhen: ALWAYS` | Always overwrite destination objects |

#### Deletion Options

| Setting | Behaviour |
|---|---|
| `deleteObjectsFromSourceAfterTransfer: false` | Source objects are kept after transfer (default) |
| `deleteObjectsFromSourceAfterTransfer: true` | Source objects deleted after successful transfer (**move** semantics) |
| `deleteObjectsUniqueInSink: true` | Destination objects NOT in source are deleted (mirror/sync semantics) |

> ⚠️ **Warning:** `deleteObjectsUniqueInSink: true` turns the transfer into a **destructive mirror**. Any object in the destination not present in the source will be permanently deleted. Use with extreme caution.

#### Metadata Preservation

```json
"metadataOptions": {
  "acl": "PRESERVE",
  "gid": "PRESERVE",
  "kmsKey": "PRESERVE",
  "mode": "PRESERVE",
  "storageClass": "PRESERVE",
  "symlink": "PRESERVE",
  "timeCreated": "PRESERVE_AS_CUSTOM_TIME",
  "uid": "PRESERVE"
}
```

#### Bandwidth Limits

```bash
# Set bandwidth limit on a transfer job (in MB/s)
gcloud transfer jobs update my-s3-job \
  --bandwidth-limit=100  # 100 MB/s cap
```

#### Pub/Sub Notification on Completion

```json
"notificationConfig": {
  "pubsubTopic": "projects/my-project/topics/transfer-notifications",
  "eventTypes": ["TRANSFER_OPERATION_SUCCESS", "TRANSFER_OPERATION_FAILED"],
  "payloadFormat": "JSON"
}
```

### Manifest-Based Transfers

A manifest file explicitly lists which objects to transfer — useful for partial migrations.

Manifest CSV format (no header, one object path per line):

```
data/file1.csv
data/file2.csv
images/photo.jpg
```

```bash
# Upload manifest to GCS first, then reference it in the job
gsutil cp manifest.csv gs://my-control-bucket/manifests/manifest.csv

gcloud transfer jobs create \
  gs://source-bucket \
  gs://destination-bucket \
  --manifest-file=gs://my-control-bucket/manifests/manifest.csv
```

---

## 4. Agent Pools & On-Premises Transfers

### What Are Transfer Agents?

Transfer agents are **Docker containers** that run on machines with access to the source data (on-premises servers, GCE VMs with NFS mounts, HDFS cluster nodes). They:

- Read data from the local POSIX filesystem or HDFS
- Upload objects directly to the Cloud Storage sink
- Report progress back to the Storage Transfer Service control plane
- Operate in pools — multiple agents work in parallel on the same job

### When Agents Are Required

| Source Type | Agents Required? |
|---|---|
| AWS S3 | ✗ — agentless |
| Azure Blob | ✗ — agentless |
| HTTP/HTTPS | ✗ — agentless |
| Cloud Storage | ✗ — agentless |
| POSIX filesystem (on-prem) | ✅ — required |
| POSIX filesystem (GCE VM) | ✅ — required |
| HDFS | ✅ — required |

### Agent Pool Architecture

```
On-Premises / GCE VM
┌─────────────────────────────────────────┐
│  Agent Pool: "on-prem-pool"             │
│  ┌──────────┐ ┌──────────┐ ┌────────┐  │
│  │ Agent 1  │ │ Agent 2  │ │Agent N │  │
│  │(Docker)  │ │(Docker)  │ │(Docker)│  │
│  └────┬─────┘ └────┬─────┘ └───┬────┘  │
│       │             │           │       │
│       └─────────────┴───────────┘       │
│              reads from                 │
│         /mnt/data or HDFS               │
└──────────────────┬──────────────────────┘
                   │ HTTPS upload
                   ▼
           Cloud Storage Bucket
```

### Creating an Agent Pool

```bash
# Create a named agent pool
gcloud transfer agent-pools create on-prem-pool \
  --project=my-project \
  --display-name="On-premises NFS agents" \
  --bandwidth-limit=500  # Optional: 500 MB/s cap across all agents

# List agent pools
gcloud transfer agent-pools list

# Describe an agent pool
gcloud transfer agent-pools describe on-prem-pool
```

### Installing and Running Transfer Agents (Docker)

```bash
# Step 1: Authenticate — get a service account key or use ADC
# (On GCE, Workload Identity is preferred over key files)

# Step 2: Pull the agent image
docker pull gcr.io/cloud-ingest/tsop-agent:latest

# Step 3: Run the agent (mounts local directory /data into container)
docker run --ulimit memlock=64000000 \
  -d \
  --rm \
  -v /data:/data \
  -v /tmp:/tmp \
  gcr.io/cloud-ingest/tsop-agent:latest \
  --project-id=my-project \
  --agent-pool=on-prem-pool \
  --creds-file=/path/to/sa-key.json

# Step 4: Run multiple agents for parallelism (on the same machine)
for i in {1..4}; do
  docker run --ulimit memlock=64000000 \
    -d --rm \
    -v /data:/data \
    gcr.io/cloud-ingest/tsop-agent:latest \
    --project-id=my-project \
    --agent-pool=on-prem-pool \
    --creds-file=/path/to/sa-key.json
done
```

### Agent Machine Type Recommendations

| Data Volume per Day | Recommended Agents | Machine Type | Notes |
|---|---|---|---|
| < 1 TB | 1–2 | 4 vCPU / 8 GB RAM | Dev / small migrations |
| 1–10 TB | 4–8 | 8 vCPU / 16 GB RAM | Production ingestion |
| 10–100 TB | 8–16 | 16 vCPU / 32 GB RAM | Large-scale migration |
| > 100 TB | 16+ | 32 vCPU / 64 GB RAM | Consider multiple pools |

> **Tip:** Each agent saturates roughly **200–400 MB/s** of upload bandwidth under ideal conditions. Add more agents (up to the network/disk limit) for higher aggregate throughput, not larger machines.

### Creating a POSIX Transfer Job

```bash
# Create a transfer job from on-prem POSIX filesystem to GCS
gcloud transfer jobs create \
  posix:///data/exports \
  gs://my-gcs-bucket/imports/ \
  --name="onprem-to-gcs" \
  --source-agent-pool=on-prem-pool \
  --description="Daily on-prem data ingestion"
```

### Service Account Requirements for Agents

```bash
# Create a dedicated service account for agents
gcloud iam service-accounts create sts-agent-sa \
  --display-name="STS Transfer Agent SA"

# Grant required roles on the destination bucket
gcloud storage buckets add-iam-policy-binding gs://my-gcs-bucket \
  --member="serviceAccount:sts-agent-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

# Grant STS agent role at project level
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:sts-agent-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/storagetransfer.transferAgent"

# Create and download a key for the agent
gcloud iam service-accounts keys create sa-key.json \
  --iam-account=sts-agent-sa@my-project.iam.gserviceaccount.com
```

### Monitoring Agent Health

```bash
# List agents registered to a pool (shows connected/disconnected state)
gcloud transfer agent-pools describe on-prem-pool \
  --format="json(transferAgents)"

# Check agent pool bandwidth utilisation
gcloud transfer agent-pools describe on-prem-pool \
  --format="value(bandwidthLimit)"
```

---

## 5. Event-Driven Transfers

### How Event-Driven Transfers Work

```
Source Bucket (GCS or S3)
        │
        │ New object created
        ▼
   Pub/Sub Topic
   (object notification)
        │
        ▼
  Pub/Sub Subscription
  (STS subscribes here)
        │
        ▼
  Storage Transfer Service
  (processes notification,
   copies object to sink)
        │
        ▼
  Destination GCS Bucket
```

### Supported Sources for Event-Driven Transfers

| Source | Pub/Sub Integration | Notes |
|---|---|---|
| **Cloud Storage** | Native GCS Pub/Sub notifications | Direct integration; easiest setup |
| **AWS S3** | S3 Event → SNS → SQS → Pub/Sub bridge | Requires AWS-side setup; higher latency |

### Event-Driven vs Scheduled Transfers

| Property | Event-Driven | Scheduled |
|---|---|---|
| **Latency** | Seconds to minutes | Minutes to hours (schedule dependent) |
| **Trigger** | Object creation event | Clock/schedule |
| **Object coverage** | Only new objects | All objects matching filters |
| **Backfill support** | ✗ (only new events) | ✅ (scans all objects) |
| **Use case** | Real-time replication, streaming ingest | Bulk sync, periodic migration |

### End-to-End Setup: GCS → GCS Event-Driven Transfer

```bash
# Step 1: Create a Pub/Sub topic for GCS notifications
gcloud pubsub topics create gcs-source-events

# Step 2: Grant the GCS service account permission to publish to the topic
GCS_SA=$(gsutil kms serviceaccount -p my-project)
gcloud pubsub topics add-iam-policy-binding gcs-source-events \
  --member="serviceAccount:${GCS_SA}" \
  --role="roles/pubsub.publisher"

# Step 3: Enable GCS Pub/Sub notifications on the source bucket
gcloud storage buckets notifications create gs://source-bucket \
  --topic=gcs-source-events \
  --event-types=OBJECT_FINALIZE

# Step 4: Grant STS service account access to subscribe to the topic
STS_SA="project-$(gcloud projects describe my-project \
  --format='value(projectNumber)')@storage-transfer-service.iam.gserviceaccount.com"

gcloud pubsub topics add-iam-policy-binding gcs-source-events \
  --member="serviceAccount:${STS_SA}" \
  --role="roles/pubsub.subscriber"

gcloud pubsub topics add-iam-policy-binding gcs-source-events \
  --member="serviceAccount:${STS_SA}" \
  --role="roles/pubsub.viewer"

# Step 5: Create the event-driven transfer job
gcloud transfer jobs create \
  gs://source-bucket \
  gs://destination-bucket \
  --name="event-driven-gcs-sync" \
  --event-stream-name=projects/my-project/topics/gcs-source-events
```

### End-to-End Setup: AWS S3 → GCS Event-Driven Transfer

```bash
# AWS-side setup (in AWS Console or CLI):
# 1. Create an SQS queue
# 2. Configure S3 bucket to send ObjectCreated events to SQS
# 3. Create a Pub/Sub topic in GCP
# 4. Set up a Pub/Sub push subscription that bridges SQS → Pub/Sub
#    (use a Cloud Function or Eventarc for the bridge)

# GCP-side: create event-driven job referencing the Pub/Sub topic
gcloud transfer jobs create \
  s3://my-aws-bucket \
  gs://my-gcs-bucket \
  --name="s3-event-driven-sync" \
  --source-creds-file=aws-creds.json \
  --event-stream-name=projects/my-project/topics/s3-object-events
```

> **Tip:** Event-driven transfers do **not** backfill historical data. Run a one-time batch job first to transfer existing objects, then enable event-driven for ongoing new objects.

---

## 6. Scheduling & Filters

### Schedule Configuration

#### One-Time Transfer (Immediate)

```bash
gcloud transfer jobs create \
  gs://source-bucket gs://dest-bucket \
  --name="one-time-now"
  # No --schedule-* flags = runs once immediately
```

#### One-Time Transfer at a Future Time

```bash
gcloud transfer jobs create \
  gs://source-bucket gs://dest-bucket \
  --name="one-time-future" \
  --schedule-starts="2025-06-01T00:00:00Z"
```

#### Recurring Transfer

```bash
# Every 24 hours starting from a specific date
gcloud transfer jobs create \
  s3://my-bucket gs://my-gcs-bucket \
  --source-creds-file=aws-creds.json \
  --name="daily-s3-sync" \
  --schedule-starts="2025-01-01T02:00:00Z" \
  --schedule-repeats-every=24h

# Every 6 hours with an end date
gcloud transfer jobs create \
  gs://source-bucket gs://dest-bucket \
  --name="6h-sync" \
  --schedule-starts="2025-01-01T00:00:00Z" \
  --schedule-repeats-every=6h \
  --schedule-ends="2025-12-31T23:59:59Z"
```

### Schedule via REST API (JSON)

```json
"schedule": {
  "scheduleStartDate": {
    "year": 2025,
    "month": 1,
    "day": 1
  },
  "scheduleEndDate": {
    "year": 2025,
    "month": 12,
    "day": 31
  },
  "startTimeOfDay": {
    "hours": 2,
    "minutes": 0,
    "seconds": 0
  },
  "repeatInterval": "86400s"
}
```

### Object Filters

Filters are specified in the `objectConditions` block of the transfer spec.

#### Prefix / Suffix Filters

```bash
# Include only objects with specific prefix
gcloud transfer jobs create \
  gs://source-bucket gs://dest-bucket \
  --include-prefixes="data/2025/,reports/" \
  --name="filtered-transfer"

# Exclude specific prefixes
gcloud transfer jobs create \
  gs://source-bucket gs://dest-bucket \
  --exclude-prefixes="tmp/,staging/" \
  --name="filtered-transfer"
```

#### Time-Based Filters

```bash
# Transfer only objects modified in the last 7 days
gcloud transfer jobs create \
  gs://source-bucket gs://dest-bucket \
  --include-modified-after-relative=7d \
  --name="recent-objects"

# Transfer only objects modified before a specific date
gcloud transfer jobs create \
  gs://source-bucket gs://dest-bucket \
  --include-modified-before-absolute="2025-01-01T00:00:00Z" \
  --name="archive-transfer"
```

#### Filter Options via REST API (JSON)

```json
"objectConditions": {
  "includePrefixes": ["data/", "exports/"],
  "excludePrefixes": ["tmp/", "staging/"],
  "minTimeElapsedSinceLastModification": "86400s",
  "maxTimeElapsedSinceLastModification": "604800s",
  "lastModifiedBefore": "2025-01-01T00:00:00Z",
  "lastModifiedSince": "2024-01-01T00:00:00Z"
}
```

### Manifest File Format

A manifest file contains one object path per line (relative to the source root). Stored as a CSV in a GCS bucket.

```
# manifest.csv — no header required
reports/2024/q4/sales.csv
reports/2024/q4/revenue.csv
data/archive/2023-backup.tar.gz
images/banner.png
```

```bash
# Upload manifest and create manifest-based job
gsutil cp manifest.csv gs://control-bucket/manifests/

gcloud transfer jobs create \
  gs://source-bucket \
  gs://dest-bucket \
  --manifest-file=gs://control-bucket/manifests/manifest.csv \
  --name="manifest-transfer"
```

> **Tip:** Manifest-based transfers are the most efficient way to transfer a specific known subset of objects without relying on prefix filters. Useful for data reconciliation and selective re-migrations.

---

## 7. IAM & Security

### STS Service Account

Every GCP project using Storage Transfer Service has a Google-managed service account:

```
project-PROJECT_NUMBER@storage-transfer-service.iam.gserviceaccount.com
```

```bash
# Get your project's STS service account
gcloud transfer google-service-accounts describe --project=my-project
# OR
PROJECT_NUMBER=$(gcloud projects describe my-project --format="value(projectNumber)")
STS_SA="project-${PROJECT_NUMBER}@storage-transfer-service.iam.gserviceaccount.com"
echo $STS_SA
```

### Required IAM Roles

#### On the Destination (Sink) GCS Bucket

```bash
# Grant STS SA write access to the destination bucket
gcloud storage buckets add-iam-policy-binding gs://destination-bucket \
  --member="serviceAccount:${STS_SA}" \
  --role="roles/storage.objectAdmin"

# If preserving object ACLs, also grant legacyBucketOwner
gcloud storage buckets add-iam-policy-binding gs://destination-bucket \
  --member="serviceAccount:${STS_SA}" \
  --role="roles/storage.legacyBucketOwner"
```

#### On the Source GCS Bucket (for GCS → GCS)

```bash
gcloud storage buckets add-iam-policy-binding gs://source-bucket \
  --member="serviceAccount:${STS_SA}" \
  --role="roles/storage.objectViewer"

gcloud storage buckets add-iam-policy-binding gs://source-bucket \
  --member="serviceAccount:${STS_SA}" \
  --role="roles/storage.legacyBucketReader"
```

#### User / Service Account Managing Jobs

```bash
# Role to create and manage transfer jobs
gcloud projects add-iam-policy-binding my-project \
  --member="user:engineer@example.com" \
  --role="roles/storagetransfer.admin"

# Read-only job viewer
gcloud projects add-iam-policy-binding my-project \
  --member="user:readonly@example.com" \
  --role="roles/storagetransfer.viewer"
```

### IAM Role Reference

| Role | Who Uses It | Permissions |
|---|---|---|
| `roles/storagetransfer.admin` | Job owners, automation SAs | Create, update, delete, run jobs |
| `roles/storagetransfer.user` | Developers | Create and run jobs; cannot delete |
| `roles/storagetransfer.viewer` | Monitoring | View jobs and operations only |
| `roles/storagetransfer.transferAgent` | Agent SAs | Agent registration and heartbeat |
| `roles/storage.objectAdmin` | STS SA on sink | Write objects to destination |
| `roles/storage.objectViewer` | STS SA on source | Read objects from source GCS bucket |

### AWS S3 Cross-Account Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::my-aws-bucket",
        "arn:aws:s3:::my-aws-bucket/*"
      ]
    }
  ]
}
```

> **Tip:** For AWS, use an **IAM user with programmatic access** (access key + secret). The key is stored in the transfer job spec. Rotate keys regularly and use least-privilege policies — STS only needs `s3:GetObject`, `s3:ListBucket`, and `s3:GetBucketLocation`.

### Azure Blob Storage — SAS Token

Generate a Shared Access Signature (SAS) token with the following permissions:
- Service: `Blob`
- Resource type: `Container` + `Object`
- Permissions: `Read` + `List`

```bash
# Create Azure transfer job using SAS token
gcloud transfer jobs create \
  "https://MY_ACCOUNT.blob.core.windows.net/MY_CONTAINER" \
  gs://my-gcs-bucket \
  --name="azure-migration" \
  --source-creds-file=azure-creds.json
```

Azure credentials file (`azure-creds.json`):

```json
{
  "azureStorageAccount": "mystorageaccount",
  "azureSasToken": "sv=2021-06-08&ss=b&srt=co&sp=rl&se=2025-12-31T00:00:00Z&st=2025-01-01T00:00:00Z&spr=https&sig=XXXXX"
}
```

> ⚠️ **Warning:** SAS tokens have an expiry date. If the token expires mid-transfer or before a scheduled job runs, the transfer will fail with a 403 error. Set token expiry well beyond the last scheduled transfer date.

### VPC Service Controls

To restrict Storage Transfer Service to a VPC Service Controls perimeter:

```bash
# Add storagetransfer.googleapis.com to your service perimeter
gcloud access-context-manager perimeters update my-perimeter \
  --add-restricted-services=storagetransfer.googleapis.com \
  --policy=POLICY_ID
```

Ensure the STS service account is listed as an **access level** member in the perimeter to allow it to access protected resources.

### Encryption (CMEK)

```bash
# Create transfer job that writes CMEK-encrypted objects to GCS
gcloud transfer jobs create \
  s3://my-bucket \
  gs://my-encrypted-bucket \
  --name="cmek-transfer"
# Note: CMEK is configured on the destination GCS bucket, not on the job itself.
# Objects written by STS inherit the bucket's default CMEK if configured.
```

---

## 8. Monitoring, Logging & Notifications

### Viewing Job Status (CLI)

```bash
# List all transfer jobs with status
gcloud transfer jobs list

# Describe a specific job
gcloud transfer jobs describe my-s3-job

# List transfer operations (runs) for a job
gcloud transfer operations list \
  --job-names="transferJobs/my-s3-job"

# Describe a specific operation (shows bytes, objects, errors)
gcloud transfer operations describe \
  "transferOperations/transferJobs%2Fmy-s3-job-TIMESTAMP"
```

### Job and Operation Status Values

| Status | Meaning |
|---|---|
| `ENABLED` | Job is active and will run on schedule |
| `DISABLED` | Job is paused; no new runs triggered |
| `DELETED` | Job has been deleted |
| `QUEUED` | Run is waiting to be picked up |
| `IN_PROGRESS` | Run is actively transferring data |
| `PAUSED` | Run was manually paused |
| `SUCCESS` | Run completed with no errors |
| `FAILED` | Run failed (check operation counters) |
| `ABORTED` | Run was manually stopped |

### Cloud Logging — Key Log Fields

Transfer logs appear in Cloud Logging under resource type `storage_transfer_job`.

```bash
# Query transfer logs in Cloud Logging (gcloud)
gcloud logging read \
  'resource.type="storage_transfer_job" AND
   resource.labels.job_id="my-s3-job"' \
  --limit=50 \
  --format=json

# Query for errors only
gcloud logging read \
  'resource.type="storage_transfer_job" AND
   severity>=ERROR AND
   resource.labels.job_id="my-s3-job"' \
  --limit=20
```

### Cloud Logging — Log Explorer Query (copy-paste)

```
resource.type="storage_transfer_job"
resource.labels.job_id="transferJobs/my-s3-job"
severity>=WARNING
timestamp>="2025-01-01T00:00:00Z"
```

### Interpreting Common Log Entries

| Log Field | Value | Meaning |
|---|---|---|
| `transferredBytes` | Increasing | Transfer in progress |
| `objectsFoundFromSource` | Count | Objects scanned from source |
| `objectsCopiedToSink` | Count | Successfully transferred |
| `objectsFromSourceSkippedBySync` | Count | Skipped (already in destination, overwrite=false) |
| `objectsFromSourceFailed` | > 0 | Objects that failed to transfer — check error details |
| `status` | `SUCCESS` | Operation completed cleanly |
| `status` | `FAILED` | Check `errorBreakdowns` for per-object error codes |

### Cloud Monitoring Metrics

| Metric | Description |
|---|---|
| `storagetransfer.googleapis.com/transferjob/transferred_bytes_count` | Bytes transferred per run |
| `storagetransfer.googleapis.com/transferjob/transferred_objects_count` | Objects transferred per run |
| `storagetransfer.googleapis.com/transferjob/found_objects_count` | Objects discovered at source |
| `storagetransfer.googleapis.com/transferjob/skipped_objects_count` | Objects skipped at destination |
| `storagetransfer.googleapis.com/transferjob/failed_objects_count` | Transfer failures |

```bash
# Example: alert if failed objects > 0 in last 1 hour
# (create in Cloud Monitoring > Alerting > Create Policy)
# Metric: storagetransfer.googleapis.com/transferjob/failed_objects_count
# Condition: sum > 0 for 60 minutes
```

### Pub/Sub Notifications

```json
{
  "notificationConfig": {
    "pubsubTopic": "projects/my-project/topics/sts-notifications",
    "eventTypes": [
      "TRANSFER_OPERATION_SUCCESS",
      "TRANSFER_OPERATION_FAILED",
      "TRANSFER_OPERATION_ABORTED"
    ],
    "payloadFormat": "JSON"
  }
}
```

```bash
# Grant STS SA permission to publish to the notification topic
gcloud pubsub topics add-iam-policy-binding sts-notifications \
  --member="serviceAccount:${STS_SA}" \
  --role="roles/pubsub.publisher"
```

Sample notification payload:

```json
{
  "name": "transferOperations/transferJobs%2Fmy-s3-job-1234567890",
  "projectId": "my-project",
  "transferJobName": "transferJobs/my-s3-job",
  "status": "SUCCESS",
  "counters": {
    "objectsFoundFromSource": "15432",
    "objectsCopiedToSink": "15430",
    "bytesFoundFromSource": "1073741824",
    "bytesCopiedToSink": "1073200000"
  },
  "notificationType": "TRANSFER_OPERATION_SUCCESS"
}
```

---

## 9. Client Library — Python

### Installation

```bash
pip install google-cloud-storage-transfer
```

### Authentication

```python
# Uses Application Default Credentials (ADC)
# Run: gcloud auth application-default login (local)
# Or set GOOGLE_APPLICATION_CREDENTIALS=/path/to/sa-key.json

from google.cloud import storage_transfer_v1 as storagetransfer
client = storagetransfer.StorageTransferServiceClient()
```

### Create a GCS → GCS Transfer Job

```python
from google.cloud import storage_transfer_v1 as storagetransfer
from google.protobuf import timestamp_pb2
import datetime

def create_gcs_to_gcs_job(project_id: str, src_bucket: str, dst_bucket: str):
    client = storagetransfer.StorageTransferServiceClient()

    now = datetime.datetime.utcnow()
    job = storagetransfer.TransferJob(
        project_id=project_id,
        description="GCS to GCS daily sync",
        transfer_spec=storagetransfer.TransferSpec(
            gcs_data_source=storagetransfer.GcsData(bucket_name=src_bucket),
            gcs_data_sink=storagetransfer.GcsData(bucket_name=dst_bucket),
            transfer_options=storagetransfer.TransferOptions(
                overwrite_objects_already_existing_in_sink=False,
                delete_objects_from_source_after_transfer=False,
            ),
        ),
        schedule=storagetransfer.Schedule(
            schedule_start_date={"year": now.year, "month": now.month, "day": now.day},
            start_time_of_day={"hours": 2, "minutes": 0},
            repeat_interval={"seconds": 86400},
        ),
        status=storagetransfer.TransferJob.Status.ENABLED,
    )

    response = client.create_transfer_job(request={"transfer_job": job})
    print(f"Created job: {response.name}")
    return response
```

### Create an AWS S3 → GCS Transfer Job

```python
def create_s3_to_gcs_job(
    project_id: str, s3_bucket: str, gcs_bucket: str,
    aws_access_key: str, aws_secret_key: str
):
    client = storagetransfer.StorageTransferServiceClient()

    job = storagetransfer.TransferJob(
        project_id=project_id,
        description="S3 to GCS migration",
        transfer_spec=storagetransfer.TransferSpec(
            aws_s3_data_source=storagetransfer.AwsS3Data(
                bucket_name=s3_bucket,
                aws_access_key=storagetransfer.AwsAccessKey(
                    access_key_id=aws_access_key,
                    secret_access_key=aws_secret_key,
                ),
            ),
            gcs_data_sink=storagetransfer.GcsData(bucket_name=gcs_bucket),
            transfer_options=storagetransfer.TransferOptions(
                overwrite_when=storagetransfer.TransferOptions.OverwriteWhen.DIFFERENT,
            ),
        ),
        status=storagetransfer.TransferJob.Status.ENABLED,
    )

    response = client.create_transfer_job(request={"transfer_job": job})
    print(f"Created S3 job: {response.name}")
    return response
```

### Check Job Status

```python
def get_job_status(project_id: str, job_name: str):
    client = storagetransfer.StorageTransferServiceClient()

    job = client.get_transfer_job(
        request={"job_name": job_name, "project_id": project_id}
    )
    print(f"Job: {job.name}")
    print(f"Status: {job.status.name}")
    print(f"Description: {job.description}")
    return job
```

### List Transfer Runs (Operations)

```python
def list_transfer_operations(project_id: str, job_name: str):
    client = storagetransfer.StorageTransferServiceClient()

    filter_str = f'{{"project_id": "{project_id}", "job_names": ["{job_name}"]}}'

    operations = client.transport._operations_client.list_operations(
        name="transferOperations",
        filter=filter_str,
    )

    for op in operations:
        metadata = storagetransfer.TransferOperation()
        op.metadata.Unpack(metadata)
        print(f"Operation: {op.name}")
        print(f"  Status: {metadata.status.name}")
        print(f"  Objects copied: {metadata.counters.objects_copied_to_sink}")
        print(f"  Bytes copied: {metadata.counters.bytes_copied_to_sink}")
```

### Pause and Resume a Job

```python
def pause_job(project_id: str, job_name: str):
    client = storagetransfer.StorageTransferServiceClient()
    client.update_transfer_job(request={
        "job_name": job_name,
        "project_id": project_id,
        "transfer_job": {"status": storagetransfer.TransferJob.Status.DISABLED},
        "update_transfer_job_field_mask": {"paths": ["status"]},
    })
    print(f"Job {job_name} paused.")

def resume_job(project_id: str, job_name: str):
    client = storagetransfer.StorageTransferServiceClient()
    client.update_transfer_job(request={
        "job_name": job_name,
        "project_id": project_id,
        "transfer_job": {"status": storagetransfer.TransferJob.Status.ENABLED},
        "update_transfer_job_field_mask": {"paths": ["status"]},
    })
    print(f"Job {job_name} resumed.")
```

### Run a Job On Demand (Immediate Trigger)

```python
def run_job_now(project_id: str, job_name: str):
    client = storagetransfer.StorageTransferServiceClient()

    operation = client.run_transfer_job(
        request={"job_name": job_name, "project_id": project_id}
    )
    print(f"Transfer started. Operation: {operation.operation.name}")

    # Wait for completion (blocking)
    result = operation.result(timeout=3600)
    print(f"Transfer complete: {result}")
    return result
```

---

## 10. gcloud CLI Quick Reference

### Transfer Job Operations

```bash
# CREATE — GCS to GCS
gcloud transfer jobs create gs://SRC gs://DST --name=JOB_NAME

# CREATE — S3 to GCS (with schedule)
gcloud transfer jobs create s3://BUCKET gs://DST \
  --source-creds-file=aws-creds.json \
  --name=JOB_NAME \
  --schedule-starts="2025-01-01T00:00:00Z" \
  --schedule-repeats-every=24h

# CREATE — Azure to GCS
gcloud transfer jobs create \
  "https://ACCOUNT.blob.core.windows.net/CONTAINER" gs://DST \
  --source-creds-file=azure-creds.json \
  --name=JOB_NAME

# CREATE — On-premises POSIX to GCS
gcloud transfer jobs create posix:///local/path gs://DST \
  --source-agent-pool=POOL_NAME \
  --name=JOB_NAME

# LIST all jobs
gcloud transfer jobs list

# DESCRIBE a job
gcloud transfer jobs describe JOB_NAME

# UPDATE job (change description, schedule, status)
gcloud transfer jobs update JOB_NAME \
  --status=disabled \
  --description="Updated description"

# DELETE a job
gcloud transfer jobs delete JOB_NAME

# RUN a job immediately (on-demand)
gcloud transfer jobs run JOB_NAME

# PAUSE (disable)
gcloud transfer jobs update JOB_NAME --status=disabled

# RESUME (enable)
gcloud transfer jobs update JOB_NAME --status=enabled
```

### Transfer Operation (Run) Commands

```bash
# LIST operations for a job
gcloud transfer operations list --job-names="transferJobs/JOB_NAME"

# LIST all recent operations
gcloud transfer operations list

# DESCRIBE an operation (full counters and status)
gcloud transfer operations describe OPERATION_NAME

# PAUSE a running operation
gcloud transfer operations pause OPERATION_NAME

# RESUME a paused operation
gcloud transfer operations resume OPERATION_NAME

# CANCEL an operation
gcloud transfer operations cancel OPERATION_NAME
```

### Agent Pool Commands

```bash
# CREATE an agent pool
gcloud transfer agent-pools create POOL_NAME \
  --bandwidth-limit=LIMIT_MBPS

# LIST agent pools
gcloud transfer agent-pools list

# DESCRIBE an agent pool
gcloud transfer agent-pools describe POOL_NAME

# UPDATE pool bandwidth limit
gcloud transfer agent-pools update POOL_NAME \
  --bandwidth-limit=200

# DELETE an agent pool (all agents must be disconnected first)
gcloud transfer agent-pools delete POOL_NAME
```

### Agent Install & Run (Docker)

```bash
# Pull latest agent image
docker pull gcr.io/cloud-ingest/tsop-agent:latest

# Run a single agent (ADC auth on GCE)
docker run -d --rm \
  --ulimit memlock=64000000 \
  -v /data:/data \
  gcr.io/cloud-ingest/tsop-agent:latest \
  --project-id=PROJECT_ID \
  --agent-pool=POOL_NAME

# Run with explicit service account key
docker run -d --rm \
  --ulimit memlock=64000000 \
  -v /data:/data \
  -v /path/to/sa-key.json:/sa-key.json \
  gcr.io/cloud-ingest/tsop-agent:latest \
  --project-id=PROJECT_ID \
  --agent-pool=POOL_NAME \
  --creds-file=/sa-key.json

# List running agents on host
docker ps | grep tsop-agent
```

---

## 11. Pricing Model Summary

> Prices are approximate as of 2025. Always verify at [cloud.google.com/storage-transfer/pricing](https://cloud.google.com/storage-transfer/pricing).

### Transfer Service Operations Cost

| Source | Destination | STS Fee | Notes |
|---|---|---|---|
| AWS S3 | Cloud Storage | **Free** | Network egress from AWS still applies |
| Azure Blob | Cloud Storage | **Free** | Azure egress costs still apply |
| HTTP/HTTPS | Cloud Storage | **Free** | Standard internet egress |
| Cloud Storage | Cloud Storage (same region) | **Free** | No egress fee either |
| Cloud Storage | Cloud Storage (cross-region) | **Free** | GCS cross-region egress applies |
| POSIX (on-prem) | Cloud Storage | **Free** | Network ingress to GCS is free |
| HDFS | Cloud Storage | **Free** | Network ingress to GCS is free |

> **Tip:** Storage Transfer Service itself has **no per-GB transfer fee**. You only pay for **network egress** from the source and GCS **storage + operations** at the destination.

### Network Egress Costs (approximate)

| Egress Path | Cost |
|---|---|
| AWS → GCS (internet) | ~$0.09 / GB (AWS standard egress) |
| Azure → GCS (internet) | ~$0.08 / GB (Azure standard egress) |
| GCS → GCS (same region) | **Free** |
| GCS → GCS (different region, same continent) | ~$0.01–$0.02 / GB |
| GCS → GCS (inter-continent) | ~$0.05–$0.08 / GB |
| On-premises → GCS | **Free** (ingress to GCS) |

### GCS Storage & Operation Costs at Destination

| Cost Type | Price |
|---|---|
| Storage (Standard) | $0.020–$0.026 / GB / month |
| Class A ops (writes) | $0.05 / 10,000 ops |
| Class B ops (reads) | $0.004 / 10,000 ops |

### Cost Optimisation Strategies

| Strategy | Saving |
|---|---|
| Schedule transfers during AWS/Azure free egress windows (if any) | Reduces source-side egress |
| Use `overwriteObjectsAlreadyExistingInSink: false` | Avoids redundant writes and Class A operation charges |
| Set `deleteObjectsUniqueInSink` only when needed | Prevents accidental costly deletions triggering re-uploads |
| Use same-region GCS buckets for GCS→GCS | Eliminates cross-region egress entirely |
| Compress data before transfer at source | Reduces bytes transferred and storage cost |
| Use Archive snapshot backups instead of full copies | Use STS incremental sync patterns rather than full daily copies |
| Right-size agent pool | Avoid idle agent VMs — use preemptible/Spot VMs for agent hosts |
| Use `--include-modified-after-relative` filters | Transfer only new/changed objects, not full bucket every run |

---

## 12. Common Errors & Troubleshooting

| Error / Issue | Cause | Fix |
|---|---|---|
| `403 Permission denied` on destination GCS | STS service account lacks `storage.objectAdmin` on sink bucket | Grant `roles/storage.objectAdmin` to `project-NNNN@storage-transfer-service.iam.gserviceaccount.com` on the bucket |
| `403 Permission denied` on source GCS | STS SA lacks `storage.objectViewer` on source | Add `roles/storage.objectViewer` and `roles/storage.legacyBucketReader` on source bucket |
| `403 InvalidAccessKeyId` (AWS S3) | AWS access key is wrong or expired | Verify key in `aws-creds.json`; rotate and update the transfer job |
| `403` from Azure | SAS token expired or lacks List permission | Regenerate SAS token with `Read + List` and future expiry; update job creds |
| Agent not connecting to pool | Docker container can't reach `storagetransfer.googleapis.com` | Check firewall/proxy allows HTTPS egress; verify SA has `roles/storagetransfer.transferAgent` |
| Transfer job stuck in `QUEUED` | No agents registered (POSIX job) or internal GCP delay | For POSIX: start Docker agents; for agentless: wait or check quotas |
| High `objectsFromSourceFailed` count | Object permissions, encoding issues, or source deletion mid-transfer | Check Cloud Logging `errorBreakdowns`; re-run for failed objects using a manifest |
| Transfer bandwidth lower than expected | Bandwidth limit set on job or agent pool; small agent count; VM network cap | Increase `--bandwidth-limit`, add more agents, or upgrade agent VM |
| Event-driven transfer not triggering | Pub/Sub topic misconfigured; STS SA lacks subscriber role; GCS notifications not set up | Verify `gcloud pubsub subscriptions list`; confirm STS SA has `pubsub.subscriber`; check GCS notification exists |
| Manifest transfer skipping objects | Object path in manifest doesn't match exact GCS key (case-sensitive, no leading `/`) | Ensure paths in manifest are relative to bucket root with no leading slash |
| `deleteObjectsUniqueInSink` deleting unexpected objects | Sync semantics delete destination objects not in source — source scan incomplete | Use `--include-prefixes` carefully; test with `overwrite=false` first before enabling delete |
| Transfer fails after Azure SAS expiry mid-run | SAS token hard expires during a multi-hour transfer | Generate SAS tokens with expiry at least 48 hours beyond the last scheduled run |
| Cross-region egress cost spike | GCS→GCS transfer between distant regions generating unexpected bills | Confirm source/destination bucket regions; use same-region buckets where possible |
| `RESOURCE_EXHAUSTED` quota errors | Too many concurrent transfer operations or API calls | Request quota increase; stagger job schedules to reduce concurrency |
| GKE / Workload Identity agent auth failure | Agent container can't exchange Workload Identity token | Annotate the Kubernetes SA; bind it to the GCP SA with `roles/storagetransfer.transferAgent` |

### Diagnostic Commands

```bash
# Get full operation details including error breakdown
gcloud transfer operations describe OPERATION_NAME \
  --format=json | jq '.metadata.errorBreakdowns'

# Tail transfer logs from Cloud Logging
gcloud logging read \
  'resource.type="storage_transfer_job"' \
  --limit=20 \
  --order=desc \
  --format="table(timestamp,severity,jsonPayload.message)"

# Check STS service account exists
gcloud transfer google-service-accounts describe \
  --project=my-project

# Verify agent pool has active agents
gcloud transfer agent-pools describe POOL_NAME --format=json

# Test S3 credentials manually (AWS CLI)
aws s3 ls s3://my-bucket --region us-east-1

# Check GCS bucket permissions for STS SA
gcloud storage buckets get-iam-policy gs://my-bucket \
  | grep "storage-transfer-service"
```

---

## Quick Reference Card

```
Service account:  project-PROJECT_NUMBER@storage-transfer-service.iam.gserviceaccount.com
Always the sink:  Cloud Storage (GCS) — STS cannot write to S3, Azure, or POSIX
Agent required:   POSIX filesystem, HDFS sources only
Agent image:      gcr.io/cloud-ingest/tsop-agent:latest
Min agent count:  1 (more for parallelism — ~200–400 MB/s per agent)
STS service fee:  Free — pay only for network egress and GCS ops
Event-driven:     Requires Pub/Sub notification on source bucket
Backfill:         Event-driven does NOT backfill — run batch job first
Manifest format:  One object path per line, no header, no leading slash
Max schedule interval: No minimum — can repeat every 1 second (not recommended)
Overwrite default: false (skip existing destination objects)
Bandwidth limit:  Set per job or per agent pool (in MB/s)
```

---

*Reference: [STS Docs](https://cloud.google.com/storage-transfer/docs) | [Pricing](https://cloud.google.com/storage-transfer/pricing) | [Python SDK](https://cloud.google.com/python/docs/reference/storagetransfer/latest) | [REST API](https://cloud.google.com/storage-transfer/docs/reference/rest)*
