# ☁️ GCP Cloud Storage — Comprehensive Cheatsheet

> **Audience:** Developers & Cloud Engineers | **Last updated:** 2025  
> A single-reference document covering theory, configuration, CLI, SDK, and best practices.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Storage Classes](#2-storage-classes)
3. [Bucket Configuration](#3-bucket-configuration)
4. [Access Control & Security](#4-access-control--security)
5. [Object Lifecycle Management](#5-object-lifecycle-management)
6. [Data Transfer & Upload Methods](#6-data-transfer--upload-methods)
7. [gsutil & gcloud CLI Quick Reference](#7-gsutil--gcloud-cli-quick-reference)
8. [Client Libraries — Python](#8-client-libraries--python)
9. [Notifications & Integrations](#9-notifications--integrations)
10. [Performance & Best Practices](#10-performance--best-practices)
11. [Pricing Model Summary](#11-pricing-model-summary)
12. [Common Errors & Troubleshooting](#12-common-errors--troubleshooting)

---

## 1. Overview & Key Concepts

**Google Cloud Storage (GCS)** is a globally unified, scalable, fully managed object storage service. It is designed for storing any amount of unstructured data — blobs, backups, media, ML datasets, logs — with high durability (11 nines) and flexible access patterns.

### Core Architecture

```
Project
└── Bucket (globally unique namespace)
    └── Object (key-value: name → data + metadata)
```

### Key Terminology

| Term | Description |
|---|---|
| **Bucket** | Top-level container for objects. Has a globally unique name and a fixed location. |
| **Object** | A file stored in a bucket. Identified by its **object name** (path-like key). Max size: **5 TB**. |
| **Object Name** | Acts as the object's key (e.g., `images/profile/user123.jpg`). No true directory structure — it's flat. |
| **Metadata** | Key-value pairs attached to an object. Includes system metadata (Content-Type, ETag, size) and custom metadata. |
| **ACL** | Access Control List — fine-grained permissions on individual buckets or objects. |
| **IAM** | Identity & Access Management — recommended, policy-based access control at project/bucket level. |
| **Generation** | A version number assigned to each object when created or overwritten. Used for versioning. |
| **Metageneration** | Tracks updates to an object's metadata (independent from data generation). |
| **ETag** | Entity tag for object integrity checking and conditional requests. |
| **Signed URL** | Time-limited URL granting temporary access to a private object without requiring a Google account. |
| **HMAC Key** | Credentials used for S3-compatible (XML API) authentication. |
| **Uniform Access** | Bucket-level IAM only — ACLs are disabled. Recommended. |
| **Fine-grained Access** | Allows per-object ACLs in addition to bucket-level IAM. |

### GCS in the GCP Ecosystem

- **Compute:** GCE instances, GKE pods mount GCS via FUSE or access via API
- **Analytics:** BigQuery reads GCS external tables; Dataflow pipelines stream to/from GCS
- **ML:** Vertex AI datasets, model artifacts, and training data stored in GCS
- **CI/CD:** Cloud Build stores artifacts; Cloud Deploy uses GCS as a staging area
- **Serverless:** Cloud Functions and Cloud Run triggered by GCS object events

---

## 2. Storage Classes

> **Tip:** Storage class affects cost-per-GB and retrieval fees, **not** latency or durability. All classes offer the same millisecond access speed and 11 nines durability.

| Class | Best For | Min Storage Duration | Retrieval Cost | Availability SLA |
|---|---|---|---|---|
| **Standard** | Frequently accessed data; hot storage | None | None | 99.99% (multi/dual-region), 99.9% (regional) |
| **Nearline** | Accessed ~once per month (backups, DR) | 30 days | Per-GB fee | 99.9% (multi/dual-region), 99.0% (regional) |
| **Coldline** | Accessed ~once per quarter (archives) | 90 days | Per-GB fee (higher) | 99.9% (multi/dual-region), 99.0% (regional) |
| **Archive** | Long-term archival; accessed < once/year | 365 days | Per-GB fee (highest) | No SLA (best-effort, same infra) |

### Storage Class Notes

- **Early deletion fee:** If you delete or move an object before its minimum duration, you are charged for the remaining days.
- **Default class:** Set at bucket creation; objects inherit it unless explicitly overridden per object.
- **Class change:** You can change an object's storage class via `Rewrite` or lifecycle rules — no data movement required.

---

## 3. Bucket Configuration

### Naming Rules

- Globally unique across **all** GCP projects
- 3–63 characters (dots allowed, max 222 chars if using dots)
- Lowercase letters, numbers, hyphens, underscores, dots only
- Cannot start or end with a hyphen or dot
- Cannot look like an IP address (e.g., `192.168.1.1`)
- Avoid dots if using HTTPS + SSL — causes SSL certificate mismatch

> ⚠️ **Warning:** Bucket names are **public** — do not embed sensitive info (project IDs, internal naming conventions) in them.

### Location Types

| Type | Description | Use Case |
|---|---|---|
| **Regional** | Single GCP region (e.g., `us-central1`) | Lowest latency for co-located compute |
| **Dual-region** | Two specific regions (e.g., `NAM4` = Iowa + S. Carolina) | High availability + geo-redundancy |
| **Multi-region** | Large geographic area (`US`, `EU`, `ASIA`) | Global distribution, highest availability |

> **Tip:** Once a bucket's location is set, it **cannot be changed**. You must migrate data to a new bucket.

### Versioning

```bash
# Enable versioning
gcloud storage buckets update gs://my-bucket --versioning

# List all versions of an object
gcloud storage ls -a gs://my-bucket/my-file.txt
```

- When enabled, overwriting or deleting an object creates a **noncurrent version** (kept indefinitely unless lifecycle rules apply).
- Restoring: copy a specific generation back to `#latest`.

### Retention Policies

- Prevents deletion or modification of objects for a defined period.
- Can be **locked** (irreversible — for compliance, e.g., WORM storage).

```bash
# Set a 1-year retention policy
gcloud storage buckets update gs://my-bucket --retention-period=365d

# Lock the policy (IRREVERSIBLE)
gcloud storage buckets update gs://my-bucket --lock-retention-policy
```

### Access Control Modes

| Mode | Description | Recommendation |
|---|---|---|
| **Uniform** | IAM only; ACLs are disabled bucket-wide | ✅ Recommended (simpler, more secure) |
| **Fine-grained** | IAM + per-object ACLs allowed | Legacy; use only when per-object ACLs are required |

```bash
# Switch bucket to uniform access
gcloud storage buckets update gs://my-bucket --uniform-bucket-level-access
```

---

## 4. Access Control & Security

### IAM Roles for Cloud Storage

| Role | Permissions |
|---|---|
| `roles/storage.admin` | Full control of buckets and objects |
| `roles/storage.objectAdmin` | Full control of objects (no bucket management) |
| `roles/storage.objectCreator` | Upload objects only |
| `roles/storage.objectViewer` | Read objects and metadata |
| `roles/storage.legacyBucketOwner` | Read/write bucket + objects (legacy ACL-based) |
| `roles/storage.legacyBucketReader` | List bucket + read objects (legacy ACL-based) |

```bash
# Grant a user objectViewer on a specific bucket
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="user:alice@example.com" \
  --role="roles/storage.objectViewer"
```

### ACLs (Fine-grained Mode Only)

```bash
# Make a single object publicly readable
gcloud storage objects update gs://my-bucket/file.txt --add-acl-grant=entity=AllUsers,role=READER

# View ACL on an object
gcloud storage objects describe gs://my-bucket/file.txt --format="value(acl)"
```

### Signed URLs

Grant **temporary, token-based access** to private objects without requiring a Google account.

```bash
# Generate a signed URL valid for 1 hour (using service account key)
gcloud storage sign-url gs://my-bucket/report.pdf \
  --duration=1h \
  --private-key-file=/path/to/sa-key.json
```

> ⚠️ **Warning:** Signed URLs are **not revocable** before expiry. Use short durations for sensitive data. Prefer **Workload Identity** over service account key files in production.

### HMAC Keys (S3-Compatible API)

```bash
# Create an HMAC key for a service account
gcloud storage hmac create sa@project.iam.gserviceaccount.com

# List HMAC keys
gcloud storage hmac list
```

### Public Access Prevention

```bash
# Prevent any object in a bucket from becoming public
gcloud storage buckets update gs://my-bucket --public-access-prevention
```

### Encryption Options

| Option | Key Management | Use Case |
|---|---|---|
| **Google-managed (GMEK)** | Google manages keys automatically | Default; no setup required |
| **CMEK** | Customer-managed via Cloud KMS | Compliance, audit trail of key usage |
| **CSEK** | Customer-supplied key sent with each request | Maximum control; key never stored by Google |

```bash
# Create a bucket with CMEK
gcloud storage buckets create gs://my-bucket \
  --default-encryption-key=projects/PROJECT/locations/REGION/keyRings/RING/cryptoKeys/KEY
```

---

## 5. Object Lifecycle Management

Lifecycle rules automatically transition or delete objects based on conditions — reducing manual overhead and storage costs.

### Rule Structure

Each rule has:
- **Condition(s):** When the rule applies
- **Action:** What to do when all conditions are met

### Supported Conditions

| Condition | Type | Description |
|---|---|---|
| `age` | int (days) | Object age since creation |
| `storageClass` | string | Current storage class of the object |
| `numNewerVersions` | int | Number of newer versions (versioned buckets) |
| `isLive` | bool | `true` = current version; `false` = noncurrent |
| `createdBefore` | date | Objects created before this date |
| `matchesPrefix` | string[] | Object name starts with prefix |
| `matchesSuffix` | string[] | Object name ends with suffix |

### Supported Actions

| Action | Description |
|---|---|
| `Delete` | Permanently deletes the object (or noncurrent version) |
| `SetStorageClass` | Changes the object's storage class |
| `AbortIncompleteMultipartUpload` | Cancels stuck multipart uploads after N days |

### Lifecycle Policy Example

```json
{
  "lifecycle": {
    "rule": [
      {
        "action": { "type": "SetStorageClass", "storageClass": "NEARLINE" },
        "condition": { "age": 30, "matchesStorageClass": ["STANDARD"] }
      },
      {
        "action": { "type": "SetStorageClass", "storageClass": "COLDLINE" },
        "condition": { "age": 90 }
      },
      {
        "action": { "type": "Delete" },
        "condition": { "age": 365 }
      },
      {
        "action": { "type": "Delete" },
        "condition": { "numNewerVersions": 3, "isLive": false }
      }
    ]
  }
}
```

```bash
# Apply the lifecycle policy to a bucket
gcloud storage buckets update gs://my-bucket --lifecycle-file=lifecycle.json

# View current lifecycle configuration
gcloud storage buckets describe gs://my-bucket --format="value(lifecycle)"
```

> **Tip:** Lifecycle rules are evaluated once per day. Changes may take up to 24 hours to take effect.

---

## 6. Data Transfer & Upload Methods

### Upload Method Comparison

| Method | Max Size | Best For |
|---|---|---|
| **Simple Upload** | 5 MB | Small files, single request |
| **Multipart Upload** | 5 MB metadata + media | Small-medium files with metadata |
| **Resumable Upload** | 5 TB | Large files; network-unreliable environments |
| **Parallel Composite Upload** | 5 TB | Very large files; high-bandwidth environments |
| **Transfer Service** | Unlimited | Bulk migration from S3, Azure, HTTP sources |
| **Storage Transfer Appliance** | Petabytes | Offline data transfer |

### Simple Upload (JSON API)

```bash
curl -X POST \
  "https://storage.googleapis.com/upload/storage/v1/b/my-bucket/o?uploadType=media&name=hello.txt" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: text/plain" \
  --data-binary @hello.txt
```

### Resumable Upload (JSON API)

```bash
# Step 1: Initiate the session — get upload URI
curl -X POST \
  "https://storage.googleapis.com/upload/storage/v1/b/my-bucket/o?uploadType=resumable&name=largefile.zip" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/zip" \
  -H "X-Upload-Content-Length: 10485760" \
  -D - | grep -i location

# Step 2: Upload data to the returned URI
curl -X PUT "UPLOAD_URI" \
  -H "Content-Range: bytes 0-10485759/10485760" \
  --data-binary @largefile.zip
```

### Parallel Composite Upload (gsutil)

```bash
# Automatically splits file and uploads in parallel
gsutil -o GSUtil:parallel_composite_upload_threshold=150M cp largefile.tar.gz gs://my-bucket/
```

> ⚠️ **Warning:** Parallel composite uploads create a composed object. If the bucket uses CSEK or the object is downloaded by `gsutil`, the component parts must also exist. Delete components after composition if not needed.

### gcloud CLI Upload

```bash
# Upload a single file
gcloud storage cp ./data.csv gs://my-bucket/data/data.csv

# Upload a directory recursively
gcloud storage cp -r ./dataset/ gs://my-bucket/dataset/

# Stream from stdin
cat data.txt | gcloud storage cp - gs://my-bucket/streamed.txt
```

---

## 7. gsutil & gcloud CLI Quick Reference

> **Note:** `gcloud storage` (newer) is preferred over `gsutil` for new scripts. Both are fully supported.

### Bucket Operations

| Operation | gcloud storage | gsutil |
|---|---|---|
| Create bucket | `gcloud storage buckets create gs://NAME --location=us-central1` | `gsutil mb -l us-central1 gs://NAME` |
| Delete bucket (empty) | `gcloud storage buckets delete gs://NAME` | `gsutil rb gs://NAME` |
| Delete bucket + all objects | `gcloud storage rm -r gs://NAME` | `gsutil rm -r gs://NAME` |
| List buckets | `gcloud storage buckets list` | `gsutil ls` |
| Describe bucket | `gcloud storage buckets describe gs://NAME` | `gsutil ls -L -b gs://NAME` |
| Enable versioning | `gcloud storage buckets update gs://NAME --versioning` | `gsutil versioning set on gs://NAME` |
| Set storage class | `gcloud storage buckets update gs://NAME --default-storage-class=NEARLINE` | `gsutil defstorageclass set NEARLINE gs://NAME` |

### Object Operations

| Operation | gcloud storage | gsutil |
|---|---|---|
| Upload file | `gcloud storage cp file.txt gs://NAME/` | `gsutil cp file.txt gs://NAME/` |
| Download file | `gcloud storage cp gs://NAME/file.txt .` | `gsutil cp gs://NAME/file.txt .` |
| List objects | `gcloud storage ls gs://NAME/` | `gsutil ls gs://NAME/` |
| List all versions | `gcloud storage ls -a gs://NAME/` | `gsutil ls -a gs://NAME/` |
| Delete object | `gcloud storage rm gs://NAME/file.txt` | `gsutil rm gs://NAME/file.txt` |
| Move/rename | `gcloud storage mv gs://NAME/old.txt gs://NAME/new.txt` | `gsutil mv gs://NAME/old.txt gs://NAME/new.txt` |
| Copy (server-side) | `gcloud storage cp gs://src/f.txt gs://dst/f.txt` | `gsutil cp gs://src/f.txt gs://dst/f.txt` |
| Sync directory | `gcloud storage rsync -r ./local gs://NAME/remote` | `gsutil rsync -r ./local gs://NAME/remote` |
| Get object metadata | `gcloud storage objects describe gs://NAME/file.txt` | `gsutil stat gs://NAME/file.txt` |
| Set metadata | `gcloud storage objects update gs://NAME/f.txt --content-type=image/png` | `gsutil setmeta -h "Content-Type:image/png" gs://NAME/f.txt` |

### Permission & Policy Operations

```bash
# View IAM policy on a bucket
gcloud storage buckets get-iam-policy gs://my-bucket

# Add IAM binding
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="serviceAccount:sa@project.iam.gserviceaccount.com" \
  --role="roles/storage.objectCreator"

# Remove IAM binding
gcloud storage buckets remove-iam-policy-binding gs://my-bucket \
  --member="user:alice@example.com" \
  --role="roles/storage.objectViewer"

# Apply lifecycle config
gcloud storage buckets update gs://my-bucket --lifecycle-file=lifecycle.json
```

---

## 8. Client Libraries — Python

### Installation

```bash
pip install google-cloud-storage
```

### Authentication

```python
# Option 1: Application Default Credentials (ADC) — recommended
# Run: gcloud auth application-default login (local dev)
# Or set GOOGLE_APPLICATION_CREDENTIALS env var (service account key)

from google.cloud import storage
client = storage.Client()  # Uses ADC automatically

# Option 2: Explicit service account
client = storage.Client.from_service_account_json("/path/to/key.json")
```

### Create a Bucket

```python
from google.cloud import storage

def create_bucket(bucket_name: str, location: str = "us-central1"):
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    bucket.storage_class = "STANDARD"
    new_bucket = client.create_bucket(bucket, location=location)
    print(f"Created bucket {new_bucket.name} in {new_bucket.location}")
    return new_bucket
```

### Upload a Blob

```python
def upload_blob(bucket_name: str, source_file: str, destination_blob: str):
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(destination_blob)

    blob.upload_from_filename(source_file)           # From file path
    # blob.upload_from_string("hello world")         # From string
    # blob.upload_from_file(file_obj)                # From file-like object

    print(f"Uploaded {source_file} → gs://{bucket_name}/{destination_blob}")
```

### Download a Blob

```python
def download_blob(bucket_name: str, source_blob: str, dest_file: str):
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(source_blob)

    blob.download_to_filename(dest_file)             # To file path
    # content = blob.download_as_bytes()             # To bytes
    # text = blob.download_as_text()                 # To string

    print(f"Downloaded gs://{bucket_name}/{source_blob} → {dest_file}")
```

### List Objects

```python
def list_blobs(bucket_name: str, prefix: str = None):
    client = storage.Client()
    blobs = client.list_blobs(bucket_name, prefix=prefix)

    for blob in blobs:
        print(f"{blob.name}  |  {blob.size} bytes  |  {blob.updated}")
```

### Generate a Signed URL (v4)

```python
import datetime
from google.cloud import storage

def generate_signed_url(bucket_name: str, blob_name: str, expiration_minutes: int = 60):
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(blob_name)

    url = blob.generate_signed_url(
        version="v4",
        expiration=datetime.timedelta(minutes=expiration_minutes),
        method="GET",
    )
    print(f"Signed URL (valid {expiration_minutes}m): {url}")
    return url
```

> ⚠️ **Warning:** `generate_signed_url` requires a service account with signing capability. When running on GCE/Cloud Run with a service account, use `google.auth.compute_engine.IDTokenCredentials` or `impersonated_credentials` instead of a key file.

### Delete an Object

```python
def delete_blob(bucket_name: str, blob_name: str):
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(blob_name)

    blob.delete()
    print(f"Deleted gs://{bucket_name}/{blob_name}")
```

### Copy Between Buckets (Server-side)

```python
def copy_blob(src_bucket: str, src_blob: str, dst_bucket: str, dst_blob: str):
    client = storage.Client()
    source_bucket = client.bucket(src_bucket)
    source_blob = source_bucket.blob(src_blob)
    dest_bucket = client.bucket(dst_bucket)

    source_bucket.copy_blob(source_blob, dest_bucket, dst_blob)
    print(f"Copied to gs://{dst_bucket}/{dst_blob}")
```

---

## 9. Notifications & Integrations

### Pub/Sub Notifications

Trigger a Pub/Sub message when objects are created, deleted, archived, or have metadata updated.

```bash
# Create a Pub/Sub topic
gcloud pubsub topics create gcs-notifications

# Set up GCS → Pub/Sub notification
gcloud storage buckets notifications create gs://my-bucket \
  --topic=gcs-notifications \
  --event-types=OBJECT_FINALIZE,OBJECT_DELETE \
  --object-prefix=uploads/

# List notifications on a bucket
gcloud storage buckets notifications list gs://my-bucket
```

**Notification payload includes:**

```json
{
  "kind": "storage#object",
  "id": "my-bucket/uploads/file.txt/1234567890",
  "name": "uploads/file.txt",
  "bucket": "my-bucket",
  "size": "1024",
  "contentType": "text/plain",
  "eventType": "OBJECT_FINALIZE"
}
```

### Cloud Functions / Eventarc Trigger

```bash
# Deploy a Cloud Function triggered on GCS object creation
gcloud functions deploy process-upload \
  --runtime=python311 \
  --trigger-resource=my-bucket \
  --trigger-event=google.storage.object.finalize \
  --region=us-central1
```

```python
# Cloud Function handler
def process_upload(event, context):
    bucket = event["bucket"]
    name = event["name"]
    print(f"New object: gs://{bucket}/{name}")
    # Add processing logic here
```

### BigQuery External Table

Query GCS data directly without loading it into BigQuery:

```sql
CREATE EXTERNAL TABLE my_dataset.events
OPTIONS (
  format = 'CSV',
  uris = ['gs://my-bucket/data/*.csv'],
  skip_leading_rows = 1
);

SELECT * FROM my_dataset.events LIMIT 100;
```

### Dataflow Integration

```python
import apache_beam as beam

with beam.Pipeline() as p:
    lines = (
        p
        | "Read from GCS" >> beam.io.ReadFromText("gs://my-bucket/input/*.txt")
        | "Process" >> beam.Map(str.upper)
        | "Write to GCS" >> beam.io.WriteToText("gs://my-bucket/output/result")
    )
```

---

## 10. Performance & Best Practices

### Object Naming for High Throughput

GCS distributes load based on object name prefixes. Avoid sequential or time-based prefixes for high-QPS workloads.

| ❌ Bad (hotspot risk) | ✅ Good (distributed) |
|---|---|
| `logs/2024-01-01/event1.json` | `a3f1/logs/2024-01-01/event1.json` |
| `img/000001.jpg`, `img/000002.jpg` | Use random hash prefix: `a1b2/img/000001.jpg` |
| `backup/20240101_120000.tar` | Prefix with hash of content or UUID |

### Request Rate Ramp-up

> ⚠️ New buckets support ~1,000 write ops/sec and ~5,000 read ops/sec per prefix by default. Ramp up gradually over several minutes to avoid `429 Too Many Requests`.

- GCS auto-scales after ~a few minutes of sustained load
- For sustained high QPS (>1,000 writes/prefix/sec), contact GCP support to pre-warm

### Avoiding Common Performance Pitfalls

- **Don't list objects in a hot loop** — list results are eventually consistent and expensive
- **Use `gsutil -m`** for parallel multi-threaded transfers
- **Compose large objects** from parts rather than re-uploading entirely
- **Enable `Transfer-Encoding: gzip`** for compressible data to reduce egress costs

### Parallel & Batch Operations

```bash
# Parallel copy with gsutil (-m flag)
gsutil -m cp -r ./large-dataset/ gs://my-bucket/dataset/

# Parallel sync
gsutil -m rsync -r -d ./local/ gs://my-bucket/remote/

# gcloud parallel transfers (automatic)
gcloud storage cp -r ./large-dataset/ gs://my-bucket/dataset/
```

### Cost Optimization Strategies

| Strategy | How |
|---|---|
| Use appropriate storage class | Run lifecycle rules to move to Nearline/Coldline/Archive as data ages |
| Delete incomplete multipart uploads | Use `AbortIncompleteMultipartUpload` lifecycle action (e.g., after 7 days) |
| Minimize egress | Co-locate compute and storage in the same region; use CDN for public assets |
| Enable object versioning selectively | Versioning stores all versions indefinitely — pair with lifecycle delete rules |
| Use `requester pays` | Shifts bandwidth/API costs to the requester for shared public datasets |
| Compress before storing | Reduce storage volume for text, JSON, logs, CSVs |

---

## 11. Pricing Model Summary

> Prices are approximate US multi-region rates as of 2025. Actual pricing varies by region. Always check [cloud.google.com/storage/pricing](https://cloud.google.com/storage/pricing).

### Storage Costs (per GB / month)

| Storage Class | Multi-Region | Dual-Region | Regional |
|---|---|---|---|
| Standard | $0.026 | $0.026 | $0.020 |
| Nearline | $0.010 | $0.010 | $0.010 |
| Coldline | $0.004 | $0.004 | $0.004 |
| Archive | $0.0012 | $0.0012 | $0.0012 |

### Operation Costs

| Category | Class A Ops (writes) | Class B Ops (reads) |
|---|---|---|
| Standard | $0.05 / 10K ops | $0.004 / 10K ops |
| Nearline | $0.10 / 10K ops | $0.01 / 10K ops |
| Coldline | $0.10 / 10K ops | $0.05 / 10K ops |
| Archive | $0.50 / 10K ops | $0.50 / 10K ops |

**Class A ops:** `PUT`, `POST`, `LIST`, `PATCH`, `REWRITE`  
**Class B ops:** `GET`, `HEAD`

### Retrieval Fees (per GB retrieved)

| Class | Cost |
|---|---|
| Standard | Free |
| Nearline | $0.01 |
| Coldline | $0.02 |
| Archive | $0.05 |

### Network Egress

| Destination | Cost |
|---|---|
| Same region (GCP service) | Free |
| Different GCP region | ~$0.01–0.08 / GB |
| Internet (first 1 TB/mo) | $0.085 / GB |
| Internet (10+ TB/mo) | $0.06 / GB |
| Using Cloud CDN | Free from GCS to CDN |

---

## 12. Common Errors & Troubleshooting

| HTTP Code | Error Name | Cause | Fix |
|---|---|---|---|
| **400** | Bad Request | Malformed request (invalid bucket name, bad lifecycle JSON, invalid header) | Validate request syntax; check bucket naming rules |
| **401** | Unauthorized | Missing or expired OAuth token | Re-authenticate: `gcloud auth application-default login` |
| **403** | Forbidden | Caller lacks IAM permissions; Public Access Prevention enabled; CMEK key access denied | Check IAM bindings; verify KMS key permissions; check bucket policy |
| **404** | Not Found | Bucket or object doesn't exist; incorrect bucket/object name | Verify the URI; check for typos; confirm object exists with `gcloud storage ls` |
| **409** | Conflict | Bucket already exists globally; trying to delete a non-empty bucket | Use a unique bucket name; delete objects before deleting bucket |
| **412** | Precondition Failed | Conditional request failed (e.g., `If-Match` ETag mismatch) | Re-fetch current object generation before retrying |
| **429** | Too Many Requests | Exceeded QPS per prefix; burst limit hit | Implement exponential backoff; diversify object name prefixes |
| **500** | Internal Server Error | Transient GCS backend error | Retry with exponential backoff |
| **503** | Service Unavailable | GCS overloaded or request rate too high | Back off and retry; ramp up request rate gradually |

### Exponential Backoff Pattern (Python)

```python
import time
import random
from google.api_core import retry
from google.cloud import storage

@retry.Retry(predicate=retry.if_exception_type(Exception))
def resilient_upload(bucket_name, source_file, destination_blob):
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(destination_blob)
    blob.upload_from_filename(source_file)
```

### Common `gsutil` Debugging Tips

```bash
# Enable debug logging
gsutil -D cp file.txt gs://my-bucket/

# Check why a request is failing (verbose)
gsutil -d ls gs://my-bucket/

# Test permissions without transferring data
gcloud storage objects describe gs://my-bucket/file.txt

# Check if a bucket exists and you have access
gcloud storage buckets describe gs://my-bucket
```

### IAM Propagation Delay

> **Tip:** IAM policy changes can take up to **60 seconds** to fully propagate. If you just granted a permission and still get a `403`, wait a moment and retry before debugging further.

---

## Quick Reference Card

```
GCS URI format:         gs://BUCKET_NAME/OBJECT_NAME
Max object size:        5 TB
Max bucket name length: 63 chars (222 with dots)
Durability:             99.999999999% (11 nines)
Default encryption:     Google-managed AES-256
IAM propagation:        Up to 60 seconds
Lifecycle eval:         Once per day (up to 24h delay)
Resumable upload chunk: Multiples of 256 KiB
```

---

*Reference: [GCS Documentation](https://cloud.google.com/storage/docs) | [Pricing](https://cloud.google.com/storage/pricing) | [Client Libraries](https://cloud.google.com/storage/docs/reference/libraries)*
