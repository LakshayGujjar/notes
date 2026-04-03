# GCP Cloud Storage Cheatsheet

> **TL;DR:** Cloud Storage is GCP's infinitely scalable, fully managed object storage service — use it for everything from static website assets and data lake raw files to ML training datasets, backups, and archival, with four storage classes, strong consistency, and deep integration across the entire GCP ecosystem.

---

## Table of Contents

1. [Overview & Core Concepts](#1-overview--core-concepts)
2. [Storage Classes & Locations](#2-storage-classes--locations)
3. [Bucket Configuration & Management](#3-bucket-configuration--management)
4. [Object Operations](#4-object-operations)
5. [Access Control & IAM](#5-access-control--iam)
6. [Data Transfer & Migration](#6-data-transfer--migration)
7. [Security & Encryption](#7-security--encryption)
8. [Lifecycle Management & Cost Optimization](#8-lifecycle-management--cost-optimization)
9. [Notifications & Event-Driven Patterns](#9-notifications--event-driven-patterns)
10. [Performance & Best Practices](#10-performance--best-practices)
11. [Client Libraries & API Access](#11-client-libraries--api-access)
12. [HMAC Keys & S3 Compatibility](#12-hmac-keys--s3-compatibility)
13. [Data Governance & Compliance](#13-data-governance--compliance)
14. [Integrations with GCP Services](#14-integrations-with-gcp-services)
15. [gcloud storage CLI Quick Reference](#15-gcloud-storage-cli-quick-reference)
16. [Terraform Snippet](#16-terraform-snippet)
17. [Troubleshooting Quick Reference](#17-troubleshooting-quick-reference)

---

## 1. Overview & Core Concepts

**Cloud Storage (GCS)** is GCP's unified object storage service offering unlimited capacity, 99.999999999% (11 nines) annual durability, and strong global consistency. It stores arbitrary binary data as **objects** inside flat-namespace containers called **buckets**. There are no directories — folder-like paths are just `/`-delimited key prefixes.

### Key Terminology

| Term | Description |
|---|---|
| **Bucket** | A globally unique, flat-namespace container for objects. Created in a specific location. |
| **Object** | A blob of data (any file type, up to 5 TB) stored with a key (path) and metadata |
| **Key** | The full name of an object within a bucket, e.g., `data/2024/01/records.csv` |
| **Metadata** | Key-value pairs describing an object (content-type, cache-control, custom fields) |
| **Generation** | A unique integer stamped on every object version; enables versioning and CAS operations |
| **Metageneration** | Version counter for object metadata (increments separately from data generation) |
| **Storage Class** | Determines cost, availability, and minimum storage duration (Standard, Nearline, Coldline, Archive) |
| **Location** | Where bucket data physically resides: single-region, dual-region, or multi-region |
| **ACL** | Access Control List — legacy fine-grained permission system per object or bucket |
| **IAM** | Identity and Access Management — recommended unified permission system |
| **Uniform bucket-level access** | Disables per-object ACLs; enforces IAM-only access control |
| **Signed URL** | A time-limited URL granting temporary access to a specific object |
| **HMAC Key** | Access key pair for authenticating with the S3-compatible XML API |
| **Consistency** | GCS provides **strong read-after-write consistency** for all operations globally |
| **Autoclass** | Feature that automatically moves objects to the most cost-effective storage class |
| **Soft delete** | Deleted objects are retained for a configurable period before permanent deletion |

### Consistency Model

> **Note:** Unlike older object stores, Cloud Storage provides **strong consistency** for all operations: reads after writes, reads after deletes, reads after metadata updates, and list operations. You never need to add delays or retries to observe newly written data.

### GCS vs. Other GCP Storage Options

| Storage Service | Type | Max Object/File | Best For |
|---|---|---|---|
| **Cloud Storage** | Object store | 5 TB per object | Unstructured data, media, backups, data lake |
| **Filestore** | Managed NFS | Petabytes (share) | Shared filesystems for VMs, GKE, lift-and-shift |
| **Persistent Disk** | Block storage | 64 TB per disk | Boot disks, databases attached to GCE/GKE |
| **Cloud SQL** | Relational DB | N/A (rows) | OLTP: MySQL, PostgreSQL, SQL Server workloads |
| **Cloud Spanner** | Distributed RDBMS | N/A (rows) | Global OLTP with strong consistency at scale |
| **BigQuery** | Analytical DW | N/A (rows) | SQL analytics, petabyte-scale OLAP queries |
| **Bigtable** | Wide-column NoSQL | ~100 MB / cell | IoT, time-series, low-latency key-based lookup |
| **Firestore** | Document DB | 1 MB / document | Mobile/web apps, flexible document workloads |

### Use-Case Positioning

| Use Case | Recommended Service | Notes |
|---|---|---|
| Static website / CDN origin | Cloud Storage | Enable web hosting on bucket |
| ML training dataset store | Cloud Storage | GCS FUSE or direct Vertex AI integration |
| Data lake raw / landing zone | Cloud Storage | Partition by date in key prefix |
| Database backups | Cloud Storage (Nearline/Coldline) | Cost-optimized for infrequent restore |
| Long-term compliance archive | Cloud Storage (Archive) | 7+ year retention at lowest cost |
| Application file uploads | Cloud Storage | Resumable upload + Signed URLs |
| Log archival | Cloud Storage | Lifecycle auto-transitions |
| Shared filesystem across VMs | Filestore | POSIX-compatible NFS |
| Container image registry | Artifact Registry (not GCS) | Use AR instead of raw GCS |

---

## 2. Storage Classes & Locations

### Storage Class Comparison

| Attribute | Standard | Nearline | Coldline | Archive |
|---|---|---|---|---|
| **Use case** | Frequently accessed | Once/month or less | Once/quarter or less | Once/year or less |
| **Min storage duration** | None | 30 days | 90 days | 365 days |
| **Storage cost** | Highest | Lower | Lower still | Lowest |
| **Retrieval cost** | None | Per GB | Per GB (higher) | Per GB (highest) |
| **Per-op cost** | Lowest | Higher | Higher | Highest |
| **Availability SLA** | 99.99% (multi) | 99.9% | 99.9% | 99.9% |
| **Access latency** | Milliseconds | Milliseconds | Milliseconds | Milliseconds |
| **Autoclass eligible** | ✅ | ✅ | ✅ | ✅ |
| **Example workloads** | Active web assets, ML datasets | Backup accessed monthly | DR archives, quarterly reports | Legal/compliance archives |

> **Note:** Archive class has millisecond access latency — it is **not** like AWS Glacier. Objects in Archive are immediately available; you simply pay more per retrieval operation.

### Location Types

| Location Type | Description | Redundancy | Example Names | Best For |
|---|---|---|---|---|
| **Multi-region** | Data replicated across 2+ regions in a geo | Survives regional failure | `us`, `eu`, `asia` | Global applications, maximum availability |
| **Dual-region** | Data replicated across exactly 2 specific regions | Survives one region failure | `nam4` (Iowa+S.Carolina), `eur4` | Data sovereignty + HA (GDPR EU) |
| **Single region** | Data stored in one GCP region | Survives zone failure | `us-central1`, `europe-west1` | Lowest latency within a region; cost-sensitive |

### Turbo Replication

Turbo replication (dual-region only) guarantees **99% of new objects** replicated to both regions within 15 minutes. Default dual-region replication has no SLA on replication timing.

```bash
# Create a dual-region bucket with turbo replication
gcloud storage buckets create gs://my-bucket \
  --location=nam4 \
  --placement=us-central1,us-east1 \
  --rpo=ASYNC_TURBO
```

### Choosing Class + Location

| Scenario | Class | Location Type |
|---|---|---|
| Global SaaS serving images | Standard | Multi-region (`us` or `eu`) |
| EU data residency + HA | Standard | Dual-region (`eur4`) |
| Nightly DB backup, restore < 1/month | Nearline | Single region (same as DB) |
| Quarterly DR snapshots | Coldline | Single region or dual-region |
| 7-year compliance archive | Archive | Single region (lowest cost) |
| ML training data (accessed daily) | Standard | Single region (co-located with compute) |
| Autoclass (unknown access pattern) | Autoclass | Any |

---

## 3. Bucket Configuration & Management

### Bucket Naming Rules

- Globally unique across all GCP projects
- 3–63 characters; lowercase letters, numbers, hyphens, underscores, dots
- Cannot start or end with a hyphen; cannot look like an IP address
- Avoid dots if using SSL (wildcard certs don't cover `bucket.with.dots.storage.googleapis.com`)

### Create a Bucket

```bash
# Basic bucket creation
gcloud storage buckets create gs://my-project-data \
  --location=us-central1 \
  --storage-class=STANDARD \
  --uniform-bucket-level-access

# Create with versioning and public access prevention
gcloud storage buckets create gs://my-secure-bucket \
  --location=us-central1 \
  --uniform-bucket-level-access \
  --no-public-access-prevention=false   # enforce: prevent all public access
```

### Uniform Bucket-Level Access vs. Fine-Grained ACLs

| Aspect | Uniform (Recommended) | Fine-Grained (Legacy) |
|---|---|---|
| **Access model** | IAM only — applies to all objects | IAM + per-object ACLs |
| **Consistency** | One policy for all objects | Each object can have different ACL |
| **Signed URLs** | ✅ Supported | ✅ Supported |
| **Public objects** | Via IAM `allUsers` | Via ACL or IAM |
| **Recommended** | ✅ Yes | ❌ Avoid for new buckets |

```bash
# Enable uniform bucket-level access on existing bucket
gcloud storage buckets update gs://my-bucket \
  --uniform-bucket-level-access
```

### Versioning

```bash
# Enable versioning
gcloud storage buckets update gs://my-bucket --versioning

# List all versions of objects in a bucket
gcloud storage ls -a gs://my-bucket/

# Get a specific version by generation
gcloud storage cp \
  gs://my-bucket/data.csv#1698765432123456 \
  ./data-old.csv
```

### Soft Delete Policy

```bash
# Set soft delete retention to 14 days (default: 7 days; range: 0–90)
gcloud storage buckets update gs://my-bucket \
  --soft-delete-duration=1209600s   # 14 days in seconds

# Restore a soft-deleted object
gcloud storage restore gs://my-bucket/data.csv#GENERATION
```

### Retention Policies

```bash
# Set a 1-year retention policy (objects cannot be deleted before 365 days)
gcloud storage buckets update gs://my-bucket \
  --retention-period=365d

# Lock the retention policy (IRREVERSIBLE — prevents shortening or removal)
gcloud storage buckets update gs://my-bucket \
  --lock-retention-period
```

> **Note:** Locking a retention policy is permanent and irreversible. Once locked, you can only increase the retention period, never decrease it. Use this for WORM compliance.

### Lifecycle Management Rules

```bash
# Apply a lifecycle policy from a JSON file
gcloud storage buckets update gs://my-bucket \
  --lifecycle-file=lifecycle.json
```

```json
{
  "lifecycle": {
    "rule": [
      {
        "action": { "type": "SetStorageClass", "storageClass": "NEARLINE" },
        "condition": {
          "age": 30,
          "matchesStorageClass": ["STANDARD"]
        }
      },
      {
        "action": { "type": "SetStorageClass", "storageClass": "COLDLINE" },
        "condition": {
          "age": 90,
          "matchesStorageClass": ["NEARLINE"]
        }
      },
      {
        "action": { "type": "SetStorageClass", "storageClass": "ARCHIVE" },
        "condition": {
          "age": 365,
          "matchesStorageClass": ["COLDLINE"]
        }
      },
      {
        "action": { "type": "Delete" },
        "condition": {
          "age": 730,
          "isLive": false          
        }
      },
      {
        "action": { "type": "AbortIncompleteMultipartUpload" },
        "condition": { "age": 7 }
      }
    ]
  }
}
```

### Requester-Pays

```bash
# Enable requester-pays (callers pay for requests + egress, not the bucket owner)
gcloud storage buckets update gs://my-bucket \
  --requester-pays

# Accessing a requester-pays bucket (must specify billing project)
gcloud storage ls gs://requester-pays-bucket \
  --billing-project=my-billing-project
```

---

## 4. Object Operations

### Upload

```bash
# Simple upload
gcloud storage cp local-file.txt gs://my-bucket/path/file.txt

# Upload with explicit content-type and cache-control
gcloud storage cp image.png gs://my-bucket/images/image.png \
  --content-type=image/png \
  --cache-control="public, max-age=86400"

# Upload entire directory recursively
gcloud storage cp -r ./dist/ gs://my-bucket/website/

# Parallel composite upload (for large files > 150 MB)
gcloud storage cp large-file.tar.gz gs://my-bucket/ \
  --composite-upload-threshold=150MB
```

```python
# Python — simple upload
from google.cloud import storage

client = storage.Client()
bucket = client.bucket("my-bucket")
blob = bucket.blob("path/to/file.txt")

# Upload from local file
blob.upload_from_filename("local-file.txt", content_type="text/plain")

# Upload from string/bytes
blob.upload_from_string(b"Hello, GCS!", content_type="text/plain")

# Resumable upload (recommended for files > 5 MB)
with open("large-file.tar.gz", "rb") as f:
    blob.upload_from_file(f, content_type="application/gzip")
```

### Download

```bash
# Download an object
gcloud storage cp gs://my-bucket/path/file.txt ./local-file.txt

# Download directory
gcloud storage cp -r gs://my-bucket/data/ ./local-data/

# Stream to stdout (cat)
gcloud storage cat gs://my-bucket/config.json
```

```python
# Python — download
blob = bucket.blob("path/to/file.txt")

# Download to local file
blob.download_to_filename("local-file.txt")

# Download to string
content = blob.download_as_text()

# Download to bytes
data = blob.download_as_bytes()

# Streaming download
import io
buffer = io.BytesIO()
blob.download_to_file(buffer)
```

### Copy, Move, and Rename

```bash
# Copy within GCS (server-side, no egress cost)
gcloud storage cp gs://my-bucket/old-path/file.txt \
                  gs://my-bucket/new-path/file.txt

# Move (copy + delete source)
gcloud storage mv gs://my-bucket/old/file.txt \
                  gs://my-bucket/new/file.txt

# Cross-bucket copy
gcloud storage cp gs://source-bucket/file.txt \
                  gs://dest-bucket/file.txt
```

```python
# Python — copy object
source_blob = source_bucket.blob("old-path/file.txt")
destination_blob = bucket.copy_blob(
    source_blob,
    dest_bucket,
    "new-path/file.txt"
)

# Rename (copy + delete)
bucket.rename_blob(blob, "new-name.txt")
```

### Delete and Restore Versioned Objects

```bash
# Delete live version (becomes noncurrent if versioning enabled)
gcloud storage rm gs://my-bucket/file.txt

# Delete a specific version permanently
gcloud storage rm gs://my-bucket/file.txt#GENERATION

# Delete all versions of all objects (wipe bucket contents)
gcloud storage rm -r --all-versions gs://my-bucket/**
```

```python
# Python — delete
blob.delete()

# Delete specific generation
blob = bucket.blob("file.txt", generation=1698765432123456)
blob.delete()
```

### Object Composition

```bash
# Compose up to 32 source objects into one destination object
gcloud storage objects compose \
  gs://my-bucket/part-000 \
  gs://my-bucket/part-001 \
  gs://my-bucket/part-002 \
  gs://my-bucket/final-combined-file
```

```python
# Python — compose
destination = bucket.blob("final-combined-file")
sources = [
    bucket.blob("part-000"),
    bucket.blob("part-001"),
    bucket.blob("part-002"),
]
destination.compose(sources)
```

### Object Metadata Management

```python
# Python — set and update metadata
blob = bucket.blob("file.txt")

# Set standard metadata
blob.content_type = "text/plain"
blob.cache_control = "public, max-age=3600"
blob.content_encoding = "gzip"
blob.content_disposition = "attachment; filename=file.txt"

# Set custom metadata
blob.metadata = {
    "author": "data-team",
    "pipeline-run-id": "abc-123",
    "source": "etl-v2"
}
blob.patch()   # Push metadata updates to GCS
```

```bash
# Update metadata via gcloud
gcloud storage objects update gs://my-bucket/file.txt \
  --content-type=text/plain \
  --cache-control="public, max-age=3600" \
  --custom-metadata=author=data-team,pipeline=etl-v2
```

### Object Holds

```bash
# Place a temporary hold (prevents deletion/modification)
gcloud storage objects update gs://my-bucket/important.csv \
  --temporary-hold

# Release temporary hold
gcloud storage objects update gs://my-bucket/important.csv \
  --no-temporary-hold

# Place an event-based hold (released by bucket-level event trigger)
gcloud storage objects update gs://my-bucket/contract.pdf \
  --event-based-hold
```

### `gsutil` vs. `gcloud storage` CLI

| Operation | gsutil (legacy) | gcloud storage (current) |
|---|---|---|
| Copy | `gsutil cp` | `gcloud storage cp` |
| List | `gsutil ls` | `gcloud storage ls` |
| Remove | `gsutil rm` | `gcloud storage rm` |
| Move | `gsutil mv` | `gcloud storage mv` |
| Sync | `gsutil rsync` | `gcloud storage rsync` |
| Cat | `gsutil cat` | `gcloud storage cat` |
| Stat | `gsutil stat` | `gcloud storage objects describe` |
| Signed URL | `gsutil signurl` | `gcloud storage sign-url` |
| Performance | Slower (Python) | Faster (Go-based, parallelized) |

> **Note:** `gcloud storage` is the modern, Go-based replacement for `gsutil`. It is significantly faster for large transfers due to automatic parallelism. New scripts should use `gcloud storage`; `gsutil` still works but is in maintenance mode.

---

## 5. Access Control & IAM

### GCS IAM Roles

| Role | Scope | Key Permissions |
|---|---|---|
| `roles/storage.admin` | Project | Full control: buckets + objects + IAM |
| `roles/storage.objectAdmin` | Bucket/Project | Create, read, update, delete objects; no bucket create/delete |
| `roles/storage.objectCreator` | Bucket/Project | Upload objects only (no read, no delete) |
| `roles/storage.objectViewer` | Bucket/Project | Read/list objects only |
| `roles/storage.legacyBucketOwner` | Bucket | Legacy ACL equivalent of OWNER on bucket |
| `roles/storage.legacyBucketReader` | Bucket | Legacy ACL equivalent of READER on bucket |
| `roles/storage.hmacKeyAdmin` | Project | Create and manage HMAC keys |

```bash
# Grant object viewer on a bucket to a service account
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member=serviceAccount:my-sa@my-project.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer

# Grant access to a specific prefix (use IAM conditions)
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member=serviceAccount:pipeline-sa@my-project.iam.gserviceaccount.com \
  --role=roles/storage.objectAdmin \
  --condition="expression=resource.name.startsWith('projects/_/buckets/my-bucket/objects/data/team-a/'),title=team-a-prefix"

# View current IAM policy
gcloud storage buckets get-iam-policy gs://my-bucket
```

### Signed URLs (V4)

Signed URLs grant time-limited access to a specific object without requiring the requester to have a GCP account.

```python
# Python — generate a V4 signed URL (read access, 1 hour)
import datetime
from google.cloud import storage

client = storage.Client()
bucket = client.bucket("my-bucket")
blob = bucket.blob("private/report.pdf")

url = blob.generate_signed_url(
    version="v4",
    expiration=datetime.timedelta(hours=1),
    method="GET",
    # For service account key-based signing:
    # service_account_email=SA_EMAIL,
    # access_token=ACCESS_TOKEN,
)
print(url)
```

```bash
# CLI — generate a signed URL (requires service account key or impersonation)
gcloud storage sign-url gs://my-bucket/private/report.pdf \
  --duration=1h \
  --private-key-file=sa-key.json

# Or with service account impersonation (no key file needed)
gcloud storage sign-url gs://my-bucket/file.txt \
  --duration=15m \
  --impersonate-service-account=signing-sa@my-project.iam.gserviceaccount.com
```

### Signed POST Policy (Browser Uploads)

```python
# Generate a signed POST policy for direct browser uploads
import datetime, json
from google.cloud import storage

client = storage.Client()
bucket = client.bucket("my-bucket")

policy = bucket.generate_upload_policy(
    conditions=[
        ["starts-with", "$key", "uploads/"],
        {"bucket": "my-bucket"},
        ["content-length-range", 0, 10485760],   # max 10 MB
        {"x-goog-date": datetime.datetime.utcnow().strftime("%Y%m%dT%H%M%SZ")},
    ],
    expiration=datetime.timedelta(hours=1),
)

# Use policy fields in an HTML <form> for browser-direct upload
```

### Making Objects Publicly Readable

```bash
# Make ALL objects in a bucket publicly readable (use with caution)
gcloud storage buckets add-iam-policy-binding gs://my-public-bucket \
  --member=allUsers \
  --role=roles/storage.objectViewer

# Make a single object public
gcloud storage objects add-iam-policy-binding \
  gs://my-bucket/public/logo.png \
  --member=allUsers \
  --role=roles/storage.objectViewer
```

> **Note:** Before making objects public, ensure `Public access prevention` is NOT enabled on the bucket. To prevent accidental public exposure on sensitive buckets, enable public access prevention: `gcloud storage buckets update gs://my-bucket --public-access-prevention`.

### Cross-Account Access

```bash
# Grant access to a service account in another GCP project
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member=serviceAccount:external-sa@other-project.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer
```

---

## 6. Data Transfer & Migration

### rsync (Directory Synchronization)

```bash
# Sync local directory to GCS (upload only new/changed files)
gcloud storage rsync ./local-data/ gs://my-bucket/data/ \
  --recursive \
  --delete-unmatched-destination-objects  # delete GCS files not in source

# Sync GCS to GCS (cross-bucket)
gcloud storage rsync gs://source-bucket/ gs://dest-bucket/ \
  --recursive

# Sync with exclusion pattern
gcloud storage rsync ./data/ gs://my-bucket/data/ \
  --recursive \
  --exclude=".*\.tmp$|.*\.log$"
```

### Parallel Composite Uploads

```bash
# For files > 150 MB: split into parts, upload in parallel, compose
gcloud storage cp large-dataset.tar.gz gs://my-bucket/ \
  --composite-upload-threshold=150MB \
  --composite-upload-component-size=50MB

# Equivalent in Python — use resumable upload (client library handles this)
blob.chunk_size = 50 * 1024 * 1024   # 50 MB chunks
blob.upload_from_filename("large-dataset.tar.gz")
```

### Streaming Upload / Download

```python
# Streaming upload from a generator (never loads full file into memory)
import io
from google.cloud import storage

def data_generator():
    for i in range(1000):
        yield f"row {i},value {i*2}\n".encode()

client = storage.Client()
blob = client.bucket("my-bucket").blob("output/stream.csv")

with blob.open("wb") as f:
    for chunk in data_generator():
        f.write(chunk)

# Streaming download
with blob.open("rb") as f:
    for line in f:
        process(line)
```

### Storage Transfer Service

The managed service for scheduled, recurring bulk transfers from S3, Azure Blob, HTTP/HTTPS URLs, or on-premises POSIX filesystems.

```bash
# Create a transfer job from S3 to GCS (one-time)
gcloud transfer jobs create \
  s3://my-aws-bucket \
  gs://my-gcs-bucket \
  --source-creds-file=aws-creds.json \
  --description="S3 to GCS migration"

# Create a recurring nightly transfer
gcloud transfer jobs create \
  s3://my-aws-bucket/exports/ \
  gs://my-gcs-bucket/s3-exports/ \
  --source-creds-file=aws-creds.json \
  --schedule-repeats-every=24h \
  --schedule-starts=2024-01-01T02:00:00Z
```

```python
# Python — create a Storage Transfer Service job
from google.cloud import storage_transfer

client = storage_transfer.StorageTransferServiceClient()

job = client.create_transfer_job(
    request={
        "transfer_job": {
            "project_id": "my-project",
            "description": "Nightly S3 sync",
            "transfer_spec": {
                "aws_s3_data_source": {
                    "bucket_name": "my-aws-bucket",
                    "aws_access_key": {
                        "access_key_id": "AKID...",
                        "secret_access_key": "secret...",
                    },
                },
                "gcs_data_sink": {
                    "bucket_name": "my-gcs-bucket",
                    "path": "from-s3/",
                },
                "transfer_options": {
                    "delete_objects_from_source_after_transfer": False,
                    "overwrite_when": "DIFFERENT",
                },
            },
            "schedule": {
                "schedule_start_date": {"year": 2024, "month": 1, "day": 1},
                "start_time_of_day": {"hours": 2, "minutes": 0},
            },
            "status": "ENABLED",
        }
    }
)
print(f"Created job: {job.name}")
```

### Transfer Options Summary

| Method | Best For | Max Scale | Notes |
|---|---|---|---|
| `gcloud storage cp` | Ad-hoc file copies | TBs | Automatic parallelism |
| `gcloud storage rsync` | Directory sync / incremental | TBs | Compares checksums |
| Storage Transfer Service | S3, Azure, HTTP sources; scheduled | Exabytes | Managed, retries, monitoring |
| Transfer Appliance | Offline, > 20 TB, slow WAN | Petabytes | Physical device; weeks lead time |
| `gsutil -m` (parallel) | Legacy parallel transfers | TBs | Use `gcloud storage` instead |

---

## 7. Security & Encryption

### Encryption Options

| Method | Key Management | Use Case |
|---|---|---|
| **Google-managed (default)** | Google manages keys automatically | Default; no configuration needed |
| **CMEK (Cloud KMS)** | Customer creates and manages keys in Cloud KMS | Compliance, audit trail, key rotation control |
| **CSEK** | Customer generates and provides key per request | Customer retains 100% key custody; Google never sees the key |
| **Client-side encryption** | Customer encrypts before upload | Full client-side control; GCS stores ciphertext only |

### CMEK Configuration

```bash
# Step 1: Create a Cloud KMS key ring and key
gcloud kms keyrings create gcs-keyring \
  --location=us-central1

gcloud kms keys create gcs-encryption-key \
  --keyring=gcs-keyring \
  --location=us-central1 \
  --purpose=encryption

# Step 2: Grant Cloud Storage service account access to the key
gcloud kms keys add-iam-policy-binding gcs-encryption-key \
  --keyring=gcs-keyring \
  --location=us-central1 \
  --member="serviceAccount:$(gcloud storage service-agent --project=my-project)" \
  --role=roles/cloudkms.cryptoKeyEncrypterDecrypter

# Step 3: Set default encryption key on the bucket
gcloud storage buckets update gs://my-bucket \
  --default-encryption-key=projects/my-project/locations/us-central1/keyRings/gcs-keyring/cryptoKeys/gcs-encryption-key
```

```python
# Python — upload with CMEK
from google.cloud import storage

client = storage.Client()
kms_key = "projects/my-project/locations/us-central1/keyRings/gcs-keyring/cryptoKeys/gcs-encryption-key"

bucket = client.bucket("my-bucket")
bucket.default_kms_key_name = kms_key

blob = bucket.blob("encrypted-data.csv")
blob.upload_from_filename("data.csv")
```

### Customer-Supplied Encryption Keys (CSEK)

```python
# Python — upload with customer-supplied key (CSEK)
import os, base64

# Generate a 256-bit random key
raw_key = os.urandom(32)
encryption_key = base64.b64encode(raw_key).decode("utf-8")

blob = bucket.blob("csek-object.txt")
blob.upload_from_string(
    "sensitive data",
    encryption_key=encryption_key   # Customer keeps this key
)

# Must provide same key for all future reads
blob.download_as_text(encryption_key=encryption_key)
```

> **Note:** With CSEK, if you lose the encryption key, the data is **permanently unrecoverable**. Google cannot help. Store CSEK keys securely (e.g., in Cloud KMS or a hardware HSM).

### VPC Service Controls

VPC Service Controls create a security perimeter around GCS to prevent data exfiltration.

```bash
# Create a service perimeter including Cloud Storage
gcloud access-context-manager perimeters create my-perimeter \
  --policy=POLICY_ID \
  --title="GCS Data Perimeter" \
  --resources=projects/PROJECT_NUMBER \
  --restricted-services=storage.googleapis.com \
  --access-levels=accessPolicies/POLICY_ID/accessLevels/trusted-network
```

### Key Rotation

```bash
# Rotate the default CMEK key on a bucket (new objects use new key version)
gcloud kms keys versions create \
  --key=gcs-encryption-key \
  --keyring=gcs-keyring \
  --location=us-central1 \
  --primary   # Set as primary; old versions still decrypt existing objects

# Re-encrypt existing objects with new key version (rewrite)
gcloud storage objects rewrite gs://my-bucket/**  \
  --destination-kms-key=projects/my-project/locations/us-central1/keyRings/gcs-keyring/cryptoKeys/gcs-encryption-key
```

---

## 8. Lifecycle Management & Cost Optimization

### Lifecycle Rule Conditions

| Condition | Type | Description |
|---|---|---|
| `age` | Integer (days) | Object age in days since creation |
| `createdBefore` | Date string | Objects created before this date |
| `isLive` | Boolean | `true` = live version; `false` = noncurrent/archived version |
| `matchesStorageClass` | List of strings | Only apply to objects in these storage classes |
| `numNewerVersions` | Integer | Noncurrent versions with this many newer versions |
| `daysSinceNoncurrentTime` | Integer | Days since object became noncurrent |
| `matchesPrefix` / `matchesSuffix` | String | Apply only to objects matching prefix/suffix |

### Lifecycle Rule Actions

| Action | Description |
|---|---|
| `SetStorageClass` | Transition object to a different storage class |
| `Delete` | Permanently delete object (or noncurrent version) |
| `AbortIncompleteMultipartUpload` | Cancel stuck multipart uploads older than N days |

### Full Lifecycle Policy Example

```json
{
  "lifecycle": {
    "rule": [
      {
        "action": { "type": "SetStorageClass", "storageClass": "NEARLINE" },
        "condition": {
          "age": 30,
          "matchesStorageClass": ["STANDARD"],
          "isLive": true
        }
      },
      {
        "action": { "type": "SetStorageClass", "storageClass": "COLDLINE" },
        "condition": {
          "age": 90,
          "matchesStorageClass": ["NEARLINE"],
          "isLive": true
        }
      },
      {
        "action": { "type": "Delete" },
        "condition": {
          "numNewerVersions": 3,
          "isLive": false
        }
      },
      {
        "action": { "type": "Delete" },
        "condition": {
          "daysSinceNoncurrentTime": 30,
          "isLive": false
        }
      },
      {
        "action": { "type": "AbortIncompleteMultipartUpload" },
        "condition": { "age": 7 }
      }
    ]
  }
}
```

### Autoclass

Autoclass automatically moves objects to the most cost-effective storage class based on access patterns. Objects start in Standard, move to Nearline → Coldline → Archive when not accessed, and move back to Standard on access.

```bash
# Enable Autoclass on a bucket
gcloud storage buckets update gs://my-bucket \
  --enable-autoclass

# Enable Autoclass with terminal storage class (default: NEARLINE; upgrade to ARCHIVE)
gcloud storage buckets update gs://my-bucket \
  --enable-autoclass \
  --autoclass-terminal-storage-class=ARCHIVE
```

> **Note:** Autoclass adds a small per-object monitoring fee. It is ideal when access patterns are unpredictable. For well-understood patterns (e.g., "delete everything after 90 days"), explicit lifecycle rules are cheaper and more predictable.

### Cost Monitoring

```bash
# Export billing data to BigQuery and query GCS costs by bucket
# (after enabling billing export in Console)

# Sample BigQuery query to see GCS cost by bucket label
# SELECT
#   labels.value AS bucket_name,
#   SUM(cost) AS total_cost
# FROM `my-project.billing_export.gcp_billing_export_v1_*`,
#   UNNEST(labels) AS labels
# WHERE service.description = 'Cloud Storage'
#   AND labels.key = 'bucket'
#   AND _TABLE_SUFFIX BETWEEN '20240101' AND '20240131'
# GROUP BY 1 ORDER BY 2 DESC;
```

---

## 9. Notifications & Event-Driven Patterns

### Pub/Sub Notifications

```bash
# Create a Pub/Sub topic for GCS events
gcloud pubsub topics create gcs-events

# Grant GCS permission to publish to the topic
gcloud pubsub topics add-iam-policy-binding gcs-events \
  --member="serviceAccount:$(gcloud storage service-agent --project=my-project)" \
  --role=roles/pubsub.publisher

# Create notification — notify on all object creates
gcloud storage buckets notifications create gs://my-bucket \
  --topic=gcs-events \
  --event-types=OBJECT_FINALIZE \
  --payload-format=JSON_API_V1

# Notify on all event types, filter to specific prefix
gcloud storage buckets notifications create gs://my-bucket \
  --topic=gcs-events \
  --event-types=OBJECT_FINALIZE,OBJECT_DELETE,OBJECT_ARCHIVE \
  --object-prefix=uploads/ \
  --payload-format=JSON_API_V1

# List existing notifications
gcloud storage buckets notifications list gs://my-bucket
```

### Notification Event Types

| Event Type | Trigger |
|---|---|
| `OBJECT_FINALIZE` | New object created or overwritten (most common) |
| `OBJECT_DELETE` | Object permanently deleted (not just versioned away) |
| `OBJECT_ARCHIVE` | Live version replaced (versioning enabled) |
| `OBJECT_METADATA_UPDATE` | Object metadata changed |

### Processing Notifications (Python)

```python
# Cloud Function triggered by Pub/Sub notification from GCS
import base64, json
from google.cloud import storage, bigquery

def process_gcs_event(event, context):
    """Triggered by a Pub/Sub message from GCS notification."""
    message = base64.b64decode(event["data"]).decode("utf-8")
    notification = json.loads(message)

    bucket_name = notification["bucket"]
    object_name = notification["name"]
    event_type = notification["eventType"]
    size = notification.get("size", 0)

    if event_type != "OBJECT_FINALIZE":
        return

    print(f"New object: gs://{bucket_name}/{object_name} ({size} bytes)")

    # Load into BigQuery
    bq = bigquery.Client()
    job = bq.load_table_from_uri(
        f"gs://{bucket_name}/{object_name}",
        "my-project.my_dataset.my_table",
        job_config=bigquery.LoadJobConfig(
            source_format=bigquery.SourceFormat.NEWLINE_DELIMITED_JSON,
            write_disposition=bigquery.WriteDisposition.WRITE_APPEND,
        ),
    )
    job.result()
    print(f"Loaded {object_name} to BigQuery")
```

### Eventarc Trigger (Cloud Run)

```bash
# Trigger a Cloud Run service on every new object in a bucket
gcloud eventarc triggers create gcs-to-cloudrun \
  --location=us-central1 \
  --destination-run-service=my-processor \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=my-bucket" \
  --service-account=eventarc-sa@my-project.iam.gserviceaccount.com
```

### GCS FUSE (Mount as Filesystem)

```bash
# Install gcsfuse (Linux)
export GCSFUSE_REPO=gcsfuse-$(lsb_release -c -s)
echo "deb http://packages.cloud.google.com/apt $GCSFUSE_REPO main" \
  | sudo tee /etc/apt/sources.list.d/gcsfuse.list
sudo apt-get update && sudo apt-get install -y gcsfuse

# Mount a bucket
mkdir -p /mnt/my-bucket
gcsfuse --implicit-dirs my-bucket /mnt/my-bucket

# Mount with specific options (read-only, file caching)
gcsfuse \
  --implicit-dirs \
  --file-cache-enable-parallel-downloads \
  --file-cache-max-size-mb=1024 \
  my-bucket /mnt/my-bucket

# Unmount
fusermount -u /mnt/my-bucket
```

> **Note:** GCS FUSE is not a POSIX filesystem — it does not support atomic renames, hard links, or random-write appends efficiently. It's best for read-heavy workloads (ML training data) or sequential writes (log files). For random I/O workloads, use Persistent Disk.

### Event-Driven Pipeline Pattern

```
GCS bucket                Pub/Sub              Cloud Function          BigQuery
(uploads/)   ──────────►  gcs-events  ──────►  process_gcs_event  ──► my_dataset.events
  object                  topic                  (JSON parser)           (streaming insert)
  finalize
  event
```

---

## 10. Performance & Best Practices

### Object Naming for Performance

Cloud Storage distributes data across many servers based on key prefixes. Sequential or time-based prefixes (e.g., `2024/01/01/001.json`) concentrate load on a single shard.

```bash
# ❌ Bad — sequential prefix causes hotspot
gs://my-bucket/2024-01-01-000001.json
gs://my-bucket/2024-01-01-000002.json

# ✅ Good — hash prefix distributes load
gs://my-bucket/a3f2/2024-01-01-000001.json  # prepend first 4 chars of SHA256
gs://my-bucket/9b1e/2024-01-01-000002.json

# ✅ Good alternative — reverse timestamp
gs://my-bucket/10000000000000-20240101-record.json
```

```python
# Python — add hash prefix for high-throughput ingestion
import hashlib

def gcs_key(original_path: str) -> str:
    prefix = hashlib.sha256(original_path.encode()).hexdigest()[:4]
    return f"{prefix}/{original_path}"

key = gcs_key("2024/01/01/events.json")
# → "a3f2/2024/01/01/events.json"
```

### Request Rate Ramp-Up

GCS auto-scales to handle high request rates, but a **cold bucket** (no recent traffic) needs time to warm up:

```
Initial capacity: ~1,000 object writes/sec per key prefix
After ramp-up:    Unlimited (GCS scales horizontally automatically)
Ramp-up time:     ~20 minutes under sustained load

Best practice: If expecting sudden bursts > 1,000 req/s on a new bucket,
gradually ramp up traffic over 20 minutes, or use distributed key prefixes.
```

### Parallel Composite Upload (Python)

```python
# Python — parallel composite upload for large files
from google.cloud.storage import transfer_manager

client = storage.Client()
bucket = client.bucket("my-bucket")

# Upload a large file using parallel composite upload
transfer_manager.upload_chunks_concurrently(
    filename="large-file.tar.gz",
    blob=bucket.blob("large-file.tar.gz"),
    chunk_size=50 * 1024 * 1024,  # 50 MB chunks
    max_workers=8,
)
```

### Sliced (Parallel) Download

```python
# Python — download large file in parallel slices
from google.cloud.storage import transfer_manager

transfer_manager.download_chunks_concurrently(
    blob=bucket.blob("large-file.tar.gz"),
    filename="local-large-file.tar.gz",
    chunk_size=50 * 1024 * 1024,
    max_workers=8,
)
```

### Cloud CDN Integration

```bash
# Create a backend bucket (GCS) for Cloud CDN
gcloud compute backend-buckets create my-cdn-backend \
  --gcs-bucket-name=my-public-bucket \
  --enable-cdn \
  --cache-mode=CACHE_ALL_STATIC

# Attach to a URL map (existing load balancer)
gcloud compute url-maps import my-url-map \
  --source=url-map.yaml
```

### Best Practice Checklist

```
✅ Use regional bucket co-located with compute for lowest latency
✅ Set content-type and cache-control on all objects
✅ Use resumable uploads for files > 5 MB
✅ Enable Autoclass or explicit lifecycle rules to manage costs
✅ Use composite uploads for files > 150 MB
✅ Prefix object keys with a hash or reverse timestamp for > 1000 req/s
✅ Enable versioning + lifecycle delete-old-versions to control storage costs
✅ Use Uniform Bucket-Level Access (disable ACLs)
✅ Enable Public Access Prevention on non-public buckets
✅ Set soft-delete retention window appropriate to your RTO
✅ Use Signed URLs for time-limited access instead of making objects public
✅ Enable Cloud CDN for public static assets with high read traffic
```

---

## 11. Client Libraries & API Access

### Python — Initialization and ADC

```python
# Install: pip install google-cloud-storage

from google.cloud import storage

# Application Default Credentials (ADC) — preferred
# Automatically uses: Workload Identity > SA key > gcloud credentials
client = storage.Client()

# Explicit project
client = storage.Client(project="my-project")

# With service account key file (avoid in production; use ADC)
client = storage.Client.from_service_account_json("sa-key.json")
```

### Python — Full CRUD Pattern

```python
from google.cloud import storage
import datetime

client = storage.Client()

# ── Bucket operations ──────────────────────────────────────
bucket = client.bucket("my-bucket")
if not bucket.exists():
    bucket = client.create_bucket("my-bucket", location="us-central1")

# ── Upload ────────────────────────────────────────────────
blob = bucket.blob("data/records.json")
blob.upload_from_string(
    '{"id": 1, "value": "test"}',
    content_type="application/json"
)

# ── Download ──────────────────────────────────────────────
content = blob.download_as_text()
data = blob.download_as_bytes()
blob.download_to_filename("/tmp/records.json")

# ── List objects ──────────────────────────────────────────
for b in client.list_blobs("my-bucket", prefix="data/", delimiter="/"):
    print(b.name, b.size, b.updated)

# ── Copy ──────────────────────────────────────────────────
dest_bucket = client.bucket("dest-bucket")
bucket.copy_blob(blob, dest_bucket, "archive/records.json")

# ── Delete ────────────────────────────────────────────────
blob.delete()
```

### Python — Error Handling and Retries

```python
from google.cloud import storage, exceptions
from google.api_core import retry
import time

client = storage.Client()

# Automatic retry with exponential backoff (built-in for idempotent ops)
blob = client.bucket("my-bucket").blob("file.txt")

# Custom retry predicate
@retry.Retry(
    predicate=retry.if_exception_type(
        exceptions.TooManyRequests,
        exceptions.ServiceUnavailable,
    ),
    deadline=300.0,        # 5-minute total deadline
    initial=1.0,           # Start at 1s
    maximum=60.0,          # Cap at 60s
    multiplier=2.0,
)
def upload_with_retry(blob, data):
    blob.upload_from_string(data)

try:
    upload_with_retry(blob, b"my data")
except exceptions.Forbidden as e:
    print(f"Permission denied: {e}")
except exceptions.NotFound as e:
    print(f"Bucket or object not found: {e}")
```

### Node.js

```javascript
// Install: npm install @google-cloud/storage
const { Storage } = require('@google-cloud/storage');
const storage = new Storage();

// Upload a file
async function uploadFile(bucketName, filePath, destName) {
  await storage.bucket(bucketName).upload(filePath, {
    destination: destName,
    metadata: { contentType: 'text/plain' },
  });
  console.log(`${filePath} uploaded to gs://${bucketName}/${destName}`);
}

// Download a file
async function downloadFile(bucketName, srcName, destPath) {
  await storage.bucket(bucketName).file(srcName).download({
    destination: destPath,
  });
}
```

### Go

```go
// Install: go get cloud.google.com/go/storage

import (
    "context"
    "io"
    "cloud.google.com/go/storage"
)

func uploadObject(ctx context.Context, bucket, object string, data []byte) error {
    client, err := storage.NewClient(ctx)
    if err != nil { return err }
    defer client.Close()

    wc := client.Bucket(bucket).Object(object).NewWriter(ctx)
    wc.ContentType = "application/octet-stream"
    if _, err := wc.Write(data); err != nil { return err }
    return wc.Close()
}
```

### XML API (S3-Compatible Endpoint)

```python
# Use Python requests to call the XML API directly
import requests
from google.auth.transport.requests import AuthorizedSession
from google.oauth2 import service_account

creds = service_account.Credentials.from_service_account_file(
    "sa-key.json",
    scopes=["https://www.googleapis.com/auth/devstorage.full_control"]
)
session = AuthorizedSession(creds)

# List objects via XML API
response = session.get(
    "https://storage.googleapis.com/my-bucket",
    params={"list-type": "2", "prefix": "data/"}
)
print(response.text)
```

---

## 12. HMAC Keys & S3 Compatibility

### What Are HMAC Keys?

HMAC (Hash-based Message Authentication Code) keys are access key pairs (Access Key ID + Secret Access Key) associated with a service account. They enable Cloud Storage to work with tools and SDKs that use the S3-compatible XML API, including boto3, s3cmd, and any AWS-SDK-based application.

### Create and Manage HMAC Keys

```bash
# Create HMAC key for a service account
gcloud storage hmac create my-sa@my-project.iam.gserviceaccount.com \
  --project=my-project

# Output:
# accessId: GOOGABCDEF123456789
# secret:   abcdefghijklmnopqrstuvwxyz1234567890ABCD

# List HMAC keys
gcloud storage hmac list --project=my-project

# Deactivate an HMAC key
gcloud storage hmac update GOOGABCDEF123456789 \
  --deactivate \
  --project=my-project

# Delete an HMAC key (must deactivate first)
gcloud storage hmac delete GOOGABCDEF123456789 \
  --project=my-project
```

### boto3 with Cloud Storage

```python
# pip install boto3
import boto3
from botocore.config import Config

# Connect to Cloud Storage using HMAC credentials
s3 = boto3.client(
    "s3",
    endpoint_url="https://storage.googleapis.com",
    aws_access_key_id="GOOGABCDEF123456789",
    aws_secret_access_key="abcdefghijklmnopqrstuvwxyz1234567890ABCD",
    config=Config(signature_version="s3v4"),
    region_name="us-central1",
)

# List buckets
response = s3.list_buckets()
for b in response["Buckets"]:
    print(b["Name"])

# Upload a file
s3.upload_file("local-file.txt", "my-bucket", "path/to/remote-file.txt")

# Download a file
s3.download_file("my-bucket", "path/to/remote-file.txt", "local-copy.txt")

# Generate a presigned URL (equivalent to GCS Signed URL)
url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "my-bucket", "Key": "file.txt"},
    ExpiresIn=3600,
)
print(url)

# List objects with prefix
response = s3.list_objects_v2(Bucket="my-bucket", Prefix="data/2024/")
for obj in response.get("Contents", []):
    print(obj["Key"], obj["Size"])
```

### s3cmd Configuration

```ini
# ~/.s3cfg
[default]
access_key = GOOGABCDEF123456789
secret_key = abcdefghijklmnopqrstuvwxyz1234567890ABCD
host_base   = storage.googleapis.com
host_bucket = %(bucket)s.storage.googleapis.com
use_https   = True
```

```bash
# List buckets
s3cmd ls

# Upload
s3cmd put local-file.txt s3://my-bucket/remote-file.txt

# Sync directory
s3cmd sync ./local-dir/ s3://my-bucket/remote-dir/
```

### S3 → GCS Migration Pattern

```python
# Migrate objects from S3 to GCS using HMAC key + Storage Transfer Service
# (Preferred approach: see Section 6)
# For scripted migration using boto3 on both sides:

import boto3
from google.cloud import storage as gcs_client

s3 = boto3.client("s3", region_name="us-east-1")
gcs = gcs_client.Client()

def migrate_object(s3_bucket, s3_key, gcs_bucket_name):
    obj = s3.get_object(Bucket=s3_bucket, Key=s3_key)
    gcs_bucket = gcs.bucket(gcs_bucket_name)
    blob = gcs_bucket.blob(s3_key)
    blob.upload_from_file(
        obj["Body"],
        content_type=obj["ContentType"]
    )
    print(f"Migrated s3://{s3_bucket}/{s3_key} → gs://{gcs_bucket_name}/{s3_key}")
```

### S3 Compatibility Limitations

| Feature | S3 Behavior | GCS XML API Behavior |
|---|---|---|
| Bucket creation region | S3 CreateBucket LocationConstraint | Use GCS Console/gcloud |
| Object tagging | S3 native tagging | Use GCS object metadata instead |
| Multipart upload | S3 multipart API | Supported via XML API |
| Cross-region replication | S3 CRR | Use Storage Transfer Service |
| Object versioning via S3 API | Supported | Partially supported |
| Server-side encryption headers | `x-amz-server-side-encryption` | Use GCS CMEKs via Console/API |

---

## 13. Data Governance & Compliance

### Retention Policies and Locks

```bash
# Set a 7-year retention policy for compliance archive
gcloud storage buckets update gs://compliance-archive \
  --retention-period=2557d   # 7 years

# View retention policy details
gcloud storage buckets describe gs://compliance-archive \
  --format="yaml(retentionPolicy)"

# Lock the retention policy (IRREVERSIBLE)
gcloud storage buckets update gs://compliance-archive \
  --lock-retention-period
```

### Object Holds

```bash
# Place a temporary hold on a specific object (e.g., litigation)
gcloud storage objects update gs://compliance-bucket/contract-2024.pdf \
  --temporary-hold

# Release after litigation hold is lifted
gcloud storage objects update gs://compliance-bucket/contract-2024.pdf \
  --no-temporary-hold

# Place event-based hold (must release explicitly per retention-event-time)
gcloud storage objects update gs://compliance-bucket/policy-v2.pdf \
  --event-based-hold
```

### Audit Logging

```bash
# Enable Data Access audit logs for Cloud Storage (captures all object reads)
cat > audit-config.json << 'EOF'
{
  "auditConfigs": [
    {
      "service": "storage.googleapis.com",
      "auditLogConfigs": [
        {"logType": "DATA_READ"},
        {"logType": "DATA_WRITE"},
        {"logType": "ADMIN_READ"}
      ]
    }
  ]
}
EOF

gcloud projects set-iam-policy my-project audit-config.json
```

```bash
# Query audit logs for object access
gcloud logging read \
  'protoPayload.serviceName="storage.googleapis.com"
   AND protoPayload.methodName="storage.objects.get"
   AND resource.labels.bucket_name="compliance-bucket"' \
  --project=my-project \
  --limit=50 \
  --format=json
```

### Bucket Access Logs

```bash
# Enable access logs (writes a log object per hour of requests)
gcloud storage buckets update gs://my-bucket \
  --log-bucket=gs://my-access-logs-bucket \
  --log-object-prefix=my-bucket/
```

### Cloud Storage Insights (Inventory Reports)

```bash
# Create an inventory report config (daily snapshot of all objects + metadata)
gcloud storage insights inventory-reports create \
  --location=us-central1 \
  --source-bucket=gs://my-bucket \
  --destination=gs://my-reports-bucket/inventory/ \
  --metadata-fields=project,bucket,name,location,storageClass,size,timeCreated,updated,contentType,retentionExpirationTime \
  --schedule=daily
```

### Data Residency Controls

```bash
# Org policy: restrict bucket creation to specific regions (EU only)
gcloud resource-manager org-policies set-policy \
  --organization=ORG_ID \
  storage-location-constraint.json
```

```json
{
  "constraint": "constraints/gcp.resourceLocations",
  "listPolicy": {
    "allowedValues": [
      "in:europe-locations"
    ]
  }
}
```

### GDPR / HIPAA Considerations

| Requirement | GCS Feature |
|---|---|
| **Data residency** | Single-region or dual-region bucket; org policy location constraint |
| **Encryption at rest** | Default Google-managed; CMEK for audit trail |
| **Encryption in transit** | TLS 1.2+ enforced by default |
| **Access audit trail** | Data Access audit logs |
| **Right to erasure** | Delete objects + disable versioning, or expire via lifecycle |
| **Data minimization** | Lifecycle rules to auto-delete after retention period |
| **Breach notification** | Cloud Security Command Center integration |
| **BAA / HIPAA** | Google offers HIPAA BAA for Cloud Storage |

---

## 14. Integrations with GCP Services

### BigQuery External Tables

```bash
# Create a BigQuery external table over GCS (no data copy)
bq mk --external_table_definition=@CSV=gs://my-bucket/data/*.csv \
  my-project:my_dataset.external_gcs_table

# Or via SQL
```

```sql
CREATE OR REPLACE EXTERNAL TABLE `my_project.my_dataset.events`
OPTIONS (
  format = 'PARQUET',
  uris   = ['gs://my-bucket/events/year=2024/*/*.parquet']
);

SELECT COUNT(*) FROM `my_project.my_dataset.events`;
```

### Dataflow Reading/Writing GCS

```python
# Apache Beam / Dataflow — read CSV from GCS, write to GCS
import apache_beam as beam

with beam.Pipeline(options=pipeline_options) as p:
    (
        p
        | "Read CSV" >> beam.io.ReadFromText("gs://my-bucket/input/*.csv",
                                              skip_header_lines=1)
        | "Parse"   >> beam.Map(lambda line: line.split(","))
        | "Write"   >> beam.io.WriteToText("gs://my-bucket/output/result",
                                            file_name_suffix=".txt")
    )
```

### Cloud Composer (Airflow) GCS Operators

```python
# Airflow DAG — GCS operators
from airflow.providers.google.cloud.operators.gcs import (
    GCSCreateBucketOperator,
    GCSDeleteObjectsOperator,
    GCSSynchronizeBucketsOperator,
)
from airflow.providers.google.cloud.transfers.gcs_to_bigquery import \
    GCSToBigQueryOperator

with dag:
    load_to_bq = GCSToBigQueryOperator(
        task_id="load_gcs_to_bq",
        bucket="my-bucket",
        source_objects=["data/{{ ds }}/records.json"],
        destination_project_dataset_table="my_project.dataset.table",
        source_format="NEWLINE_DELIMITED_JSON",
        write_disposition="WRITE_APPEND",
    )
```

### Vertex AI Datasets from GCS

```python
# Create a Vertex AI dataset backed by GCS
from google.cloud import aiplatform

aiplatform.init(project="my-project", location="us-central1")

dataset = aiplatform.ImageDataset.create(
    display_name="my-image-dataset",
    gcs_source="gs://my-bucket/training-data/images.jsonl",
    import_schema_uri=aiplatform.schema.dataset.ioformat.image.single_label_classification,
)
```

### GKE GCS FUSE CSI Driver

```yaml
# Mount a GCS bucket as a volume in a GKE Pod
# (Cluster must have GcsFuseCsiDriver addon enabled)
spec:
  serviceAccountName: my-app-sa   # Must have storage.objectViewer
  containers:
    - name: ml-trainer
      image: tensorflow/tensorflow:latest
      volumeMounts:
        - name: training-data
          mountPath: /data
          readOnly: true
  volumes:
    - name: training-data
      csi:
        driver: gcsfuse.csi.storage.gke.io
        readOnly: true
        volumeAttributes:
          bucketName: my-ml-data-bucket
          mountOptions: "implicit-dirs,file-cache:enable-parallel-downloads:true"
```

### Cloud Functions GCS Trigger

```python
# Cloud Function triggered on every new object in a bucket
# (Deploy with --trigger-bucket flag)
import functions_framework
from google.cloud import storage

@functions_framework.cloud_event
def process_new_file(cloud_event):
    data = cloud_event.data
    bucket = data["bucket"]
    name   = data["name"]
    print(f"Processing: gs://{bucket}/{name}")

    client = storage.Client()
    blob = client.bucket(bucket).blob(name)
    content = blob.download_as_text()
    # ... process content
```

```bash
# Deploy the function
gcloud functions deploy process-new-file \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=process_new_file \
  --trigger-bucket=my-bucket
```

### Cloud CDN with GCS Backend Bucket

```bash
# Create a backend bucket backed by GCS
gcloud compute backend-buckets create gcs-cdn-backend \
  --gcs-bucket-name=my-public-bucket \
  --enable-cdn \
  --cache-mode=CACHE_ALL_STATIC \
  --default-ttl=86400 \      # 24 hours
  --max-ttl=604800           # 7 days

# Add to existing URL map
gcloud compute url-maps add-path-matcher my-url-map \
  --path-matcher-name=static-assets \
  --default-backend-bucket=gcs-cdn-backend \
  --path-rules="/static/*=gcs-cdn-backend"
```

---

## 15. `gcloud storage` CLI Quick Reference

### Bucket Commands

| Command | Description | Example |
|---|---|---|
| `buckets create` | Create a new bucket | `gcloud storage buckets create gs://my-bucket --location=us-central1` |
| `buckets delete` | Delete an empty bucket | `gcloud storage buckets delete gs://my-bucket` |
| `buckets describe` | Show bucket configuration | `gcloud storage buckets describe gs://my-bucket` |
| `buckets list` | List all buckets in project | `gcloud storage buckets list` |
| `buckets update` | Modify bucket settings | `gcloud storage buckets update gs://my-bucket --versioning` |
| `buckets get-iam-policy` | Show IAM policy | `gcloud storage buckets get-iam-policy gs://my-bucket` |
| `buckets add-iam-policy-binding` | Add IAM binding | `gcloud storage buckets add-iam-policy-binding gs://my-bucket --member=... --role=...` |
| `buckets notifications create` | Add Pub/Sub notification | `gcloud storage buckets notifications create gs://my-bucket --topic=my-topic` |

### Object Commands

| Command | Description | Example |
|---|---|---|
| `cp` | Copy objects | `gcloud storage cp file.txt gs://my-bucket/` |
| `cp -r` | Recursive copy | `gcloud storage cp -r ./dir/ gs://my-bucket/dir/` |
| `mv` | Move/rename objects | `gcloud storage mv gs://my-bucket/old.txt gs://my-bucket/new.txt` |
| `rm` | Delete objects | `gcloud storage rm gs://my-bucket/file.txt` |
| `rm -r` | Recursive delete | `gcloud storage rm -r gs://my-bucket/prefix/` |
| `ls` | List objects | `gcloud storage ls gs://my-bucket/` |
| `ls -l` | List with details | `gcloud storage ls -l gs://my-bucket/` |
| `ls -a` | List all versions | `gcloud storage ls -a gs://my-bucket/` |
| `cat` | Output object to stdout | `gcloud storage cat gs://my-bucket/config.json` |
| `rsync` | Sync directories | `gcloud storage rsync -r ./local/ gs://my-bucket/remote/` |
| `objects describe` | Show object metadata | `gcloud storage objects describe gs://my-bucket/file.txt` |
| `objects update` | Update object metadata | `gcloud storage objects update gs://my-bucket/file.txt --content-type=text/plain` |

### Signing & HMAC Commands

```bash
# Generate a signed URL
gcloud storage sign-url gs://my-bucket/file.txt \
  --duration=1h \
  --impersonate-service-account=sa@project.iam.gserviceaccount.com

# Create HMAC key
gcloud storage hmac create sa@project.iam.gserviceaccount.com

# List HMAC keys
gcloud storage hmac list

# Deactivate HMAC key
gcloud storage hmac update ACCESS_KEY_ID --deactivate
```

### `gsutil` → `gcloud storage` Equivalents

| gsutil | gcloud storage |
|---|---|
| `gsutil cp` | `gcloud storage cp` |
| `gsutil ls` | `gcloud storage ls` |
| `gsutil rm` | `gcloud storage rm` |
| `gsutil mv` | `gcloud storage mv` |
| `gsutil rsync` | `gcloud storage rsync` |
| `gsutil cat` | `gcloud storage cat` |
| `gsutil stat` | `gcloud storage objects describe` |
| `gsutil mb` | `gcloud storage buckets create` |
| `gsutil rb` | `gcloud storage buckets delete` |
| `gsutil du` | `gcloud storage du` |
| `gsutil acl ch` | `gcloud storage buckets/objects add-iam-policy-binding` |
| `gsutil signurl` | `gcloud storage sign-url` |
| `gsutil notification create` | `gcloud storage buckets notifications create` |

---

## 16. Terraform Snippet

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = "us-central1"
}

variable "project_id" {
  description = "GCP project ID"
  type        = string
}

variable "bucket_name" {
  description = "Globally unique bucket name"
  type        = string
}

# ── Cloud KMS Key for CMEK ────────────────────────────────
resource "google_kms_key_ring" "gcs_ring" {
  name     = "gcs-keyring"
  location = "us-central1"
}

resource "google_kms_crypto_key" "gcs_key" {
  name            = "gcs-encryption-key"
  key_ring        = google_kms_key_ring.gcs_ring.id
  rotation_period = "7776000s"   # 90 days
}

# Grant GCS service agent permission to use the key
resource "google_kms_crypto_key_iam_member" "gcs_sa_kms" {
  crypto_key_id = google_kms_crypto_key.gcs_key.id
  role          = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
  member        = "serviceAccount:${data.google_storage_project_service_account.gcs_sa.email_address}"
}

data "google_storage_project_service_account" "gcs_sa" {
  project = var.project_id
}

# ── Pub/Sub Topic for Notifications ──────────────────────
resource "google_pubsub_topic" "gcs_events" {
  name = "gcs-object-events"
}

resource "google_pubsub_topic_iam_member" "gcs_pubsub_publish" {
  topic  = google_pubsub_topic.gcs_events.name
  role   = "roles/pubsub.publisher"
  member = "serviceAccount:${data.google_storage_project_service_account.gcs_sa.email_address}"
}

# ── Cloud Storage Bucket ──────────────────────────────────
resource "google_storage_bucket" "main" {
  name          = var.bucket_name
  location      = "us-central1"
  storage_class = "STANDARD"
  force_destroy = false   # Prevent accidental deletion in production

  # Uniform bucket-level access (disable legacy ACLs)
  uniform_bucket_level_access = true

  # Prevent accidental public access
  public_access_prevention = "enforced"

  # Object versioning
  versioning {
    enabled = true
  }

  # Soft delete (retain deleted objects for 14 days)
  soft_delete_policy {
    retention_duration_seconds = 1209600   # 14 days
  }

  # CMEK — default encryption key for all objects
  encryption {
    default_kms_key_name = google_kms_crypto_key.gcs_key.id
  }

  # Lifecycle rules
  lifecycle_rule {
    # Transition to Nearline after 30 days
    action {
      type          = "SetStorageClass"
      storage_class = "NEARLINE"
    }
    condition {
      age                   = 30
      matches_storage_class = ["STANDARD"]
      with_state            = "LIVE"
    }
  }

  lifecycle_rule {
    # Transition to Coldline after 90 days
    action {
      type          = "SetStorageClass"
      storage_class = "COLDLINE"
    }
    condition {
      age                   = 90
      matches_storage_class = ["NEARLINE"]
      with_state            = "LIVE"
    }
  }

  lifecycle_rule {
    # Delete live objects after 365 days
    action {
      type = "Delete"
    }
    condition {
      age        = 365
      with_state = "LIVE"
    }
  }

  lifecycle_rule {
    # Delete noncurrent versions after 30 days
    action {
      type = "Delete"
    }
    condition {
      days_since_noncurrent_time = 30
      with_state                 = "ARCHIVED"
    }
  }

  lifecycle_rule {
    # Abort incomplete multipart uploads after 7 days
    action {
      type = "AbortIncompleteMultipartUpload"
    }
    condition {
      age = 7
    }
  }

  # CORS configuration (for browser uploads)
  cors {
    origin          = ["https://app.mycompany.com"]
    method          = ["GET", "PUT", "POST", "HEAD"]
    response_header = ["Content-Type", "Content-MD5", "x-goog-resumable"]
    max_age_seconds = 3600
  }

  # Labels for cost attribution
  labels = {
    env         = "production"
    team        = "data-engineering"
    cost-center = "analytics"
  }

  depends_on = [google_kms_crypto_key_iam_member.gcs_sa_kms]
}

# ── IAM Bindings on the Bucket ────────────────────────────
resource "google_storage_bucket_iam_member" "data_pipeline_writer" {
  bucket = google_storage_bucket.main.name
  role   = "roles/storage.objectCreator"
  member = "serviceAccount:data-pipeline@${var.project_id}.iam.gserviceaccount.com"
}

resource "google_storage_bucket_iam_member" "analytics_reader" {
  bucket = google_storage_bucket.main.name
  role   = "roles/storage.objectViewer"
  member = "serviceAccount:analytics-sa@${var.project_id}.iam.gserviceaccount.com"
}

# ── Pub/Sub Notification on the Bucket ───────────────────
resource "google_storage_notification" "object_created" {
  bucket         = google_storage_bucket.main.name
  payload_format = "JSON_API_V1"
  topic          = google_pubsub_topic.gcs_events.id
  event_types    = ["OBJECT_FINALIZE", "OBJECT_DELETE"]

  custom_attributes = {
    environment = "production"
  }

  depends_on = [google_pubsub_topic_iam_member.gcs_pubsub_publish]
}

# ── Outputs ───────────────────────────────────────────────
output "bucket_url" {
  value       = "gs://${google_storage_bucket.main.name}"
  description = "Cloud Storage bucket URL"
}

output "bucket_self_link" {
  value = google_storage_bucket.main.self_link
}

output "pubsub_topic" {
  value = google_pubsub_topic.gcs_events.name
}
```

```bash
terraform init
terraform plan  -var="project_id=my-project" -var="bucket_name=my-unique-bucket-12345"
terraform apply -var="project_id=my-project" -var="bucket_name=my-unique-bucket-12345"
```

---

## 17. Troubleshooting Quick Reference

| Issue | Likely Cause | Diagnostic Steps | Fix |
|---|---|---|---|
| **403 Forbidden on object read** | Missing IAM role; uniform access + no IAM binding; VPC SC perimeter | `gcloud storage objects describe gs://bucket/obj`; check `gcloud projects get-iam-policy` | Add `roles/storage.objectViewer`; check VPC SC perimeter allows the caller |
| **403 on bucket create** | Missing `storage.buckets.create` permission | `gcloud auth list`; check project-level IAM | Add `roles/storage.admin` or `roles/storage.legacyBucketOwner` |
| **404 Bucket Not Found** | Bucket doesn't exist; typo in bucket name; project mismatch | `gcloud storage buckets list`; check project | Create the bucket; verify `--project` flag |
| **SignedURL expired or invalid** | URL past expiration; clock skew; wrong signing service account | Check `X-Goog-Expires` in URL query params | Generate a new signed URL; ensure system clock is synchronized; verify SA has `iam.serviceAccounts.signBlob` |
| **CORS error in browser** | CORS policy not set; wrong origin; missing response-header | Check browser DevTools → Network → Response headers | Set CORS policy with correct `origin`, `method`, and `response_header` on the bucket |
| **Resumable upload stalled/incomplete** | Network interruption; client didn't resume; incomplete multipart > 7 days | Check upload session URI; `gcloud storage ls` for incomplete objects | Resume using the original session URI within 1 week; add `AbortIncompleteMultipartUpload` lifecycle rule |
| **Autoclass moving to Archive unexpectedly** | Object not accessed for required period; Autoclass terminal class set to ARCHIVE | Check object metadata: `storageClass`, `timeStorageClassUpdated` | Set `--autoclass-terminal-storage-class=NEARLINE` or disable Autoclass; add explicit lifecycle rule instead |
| **VPC Service Controls blocking GCS access** | Service perimeter does not include the project or access level | Check Cloud Audit Logs for `ACCESS_DENIED` with `VPC_SERVICE_CONTROLS` | Add project to perimeter; add access level for the caller's IP/identity |
| **Lifecycle rules not triggering** | Rules evaluated once daily; object does not match all conditions; storage class not listed in `matchesStorageClass` | Check bucket lifecycle config with `gcloud storage buckets describe` | Verify all conditions match; wait 24 hours; ensure `matchesStorageClass` includes the current class |
| **Transfer Service job failing** | Source credentials expired; source bucket permissions; destination bucket CMEK | Check Transfer Service job logs in Cloud Console → Storage Transfer | Rotate source credentials; grant transfer SA `storage.objectAdmin` on destination; grant KMS access |
| **CMEK key access denied** | GCS service agent lacks `cloudkms.cryptoKeyEncrypterDecrypter` | Check KMS IAM policy: `gcloud kms keys get-iam-policy` | Grant `roles/cloudkms.cryptoKeyEncrypterDecrypter` to the GCS service agent |
| **CMEK key disabled/destroyed** | Key was disabled or destroyed in Cloud KMS | Attempt any object operation → `KEY_DISABLED` error in logs | Re-enable the key in Cloud KMS; if destroyed, data is unrecoverable |
| **High unexpected egress costs** | Cross-region data movement; internet egress; CDN not caching | Check Cloud Billing export filtered by `network.egress`; check bucket region vs. consumer region | Co-locate bucket and consumers in same region; use Cloud CDN; check for accidental cross-region traffic |
| **gsutil/gcloud auth error** | Expired credentials; wrong account | `gcloud auth list`; `gcloud auth application-default print-access-token` | `gcloud auth login`; `gcloud auth application-default login` |
| **Object count/list incomplete** | Listing > 1,000 objects without pagination; eventual list consistency after mass delete | Use `--page-size` or paginate via API `pageToken` | Always paginate large lists; use GCS Inventory Reports for full snapshot |
| **Bucket name taken** | GCS bucket names are globally unique | Try another name | Use project ID as prefix: `my-project-data-bucket` |
| **Soft delete restoring wrong version** | Multiple versions exist; generation not specified | `gcloud storage ls -a gs://bucket/object` | Specify `#GENERATION` when restoring: `gcloud storage restore gs://bucket/obj#GEN` |

---

*Last updated: 2025 | Covers Cloud Storage GA features as of this date.*
*Refer to [cloud.google.com/storage/docs](https://cloud.google.com/storage/docs) for the latest information.*
