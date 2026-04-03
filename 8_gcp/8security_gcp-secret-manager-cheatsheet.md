# 🔑 GCP Secret Manager Cheatsheet

> **Audience:** Security Engineers · Platform Engineers · DevOps Engineers · Architects  
> **Scope:** Secret Manager — creation, versioning, access, rotation, replication, CMEK, integrations, IAM, monitoring

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Secrets & Secret Versions](#2-secrets--secret-versions)
3. [Accessing Secret Versions](#3-accessing-secret-versions)
4. [Secret Versioning & Lifecycle](#4-secret-versioning--lifecycle)
5. [Replication Policies](#5-replication-policies)
6. [IAM & Access Control](#6-iam--access-control)
7. [Secret Rotation](#7-secret-rotation)
8. [CMEK (Customer-Managed Encryption Keys)](#8-cmek-customer-managed-encryption-keys)
9. [Secret Manager Integrations](#9-secret-manager-integrations)
10. [Monitoring, Alerting & Audit Logging](#10-monitoring-alerting--audit-logging)
11. [Best Practices & Security Patterns](#11-best-practices--security-patterns)
12. [gcloud CLI & REST API Quick Reference](#12-gcloud-cli--rest-api-quick-reference)
13. [Pricing Summary](#13-pricing-summary)
14. [Quick Reference & Comparison Tables](#14-quick-reference--comparison-tables)

---

## 1. Overview & Key Concepts

### What is Secret Manager?

Secret Manager is Google Cloud's fully managed service for storing, accessing, and managing sensitive configuration data — API keys, passwords, TLS certificates, SSH keys, and any small blob of sensitive data. Core value proposition:

- **Centralized secret storage** — single source of truth for all secrets across services
- **Versioning** — immutable secret versions with full history and rollback capability
- **Fine-grained IAM** — per-secret access control; bind roles at the secret resource level
- **Automatic replication** — Google-managed global replication or user-specified regional replication
- **Audit logging** — every access and management operation logged to Cloud Audit Logs
- **Rotation support** — built-in rotation scheduling with Pub/Sub event notifications
- **CMEK** — optionally encrypt secret payloads with your own Cloud KMS keys

---

### Secret Manager vs. Alternatives

| | **Secret Manager** | **Env Variables** | **GCS / Firestore** | **HashiCorp Vault** | **AWS Secrets Manager** | **Cloud KMS alone** |
|---|---|---|---|---|---|---|
| **Purpose-built for secrets** | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| **Versioning** | ✅ | ❌ | Manual | ✅ | ✅ | Key versions only |
| **Automatic rotation** | ✅ (Pub/Sub) | ❌ | Manual | ✅ | ✅ (native) | ❌ |
| **Audit logging** | ✅ Cloud Audit Logs | ❌ | Partial | ✅ | ✅ CloudTrail | ✅ |
| **IAM granularity** | Per-secret | ❌ | Per-bucket/doc | ACL policies | Per-secret | Per-key |
| **GCP integration** | Native | Manual | Manual | Plugin/agent | ❌ | Native |
| **Operational burden** | None (managed) | None | Medium | High (self-host) or $$$ (HCP) | None (managed) | None |
| **Key material storage** | Payload bytes | In process | Object store | Pluggable backends | AWS managed | Never — ops only |
| **When to choose** | GCP-native workloads | Stateless, low-sensitivity | ❌ Avoid | Multi-cloud, complex policies | AWS-native | Encryption keys, not secrets |

> 💡 **Choose Secret Manager** for any GCP workload that needs centralized, audited, IAM-controlled storage of credentials, API keys, or certificates. Do not use raw environment variables for secrets in containerized or serverless workloads.

---

### Core Concepts

| Concept | Definition |
|---|---|
| **Secret** | Named container resource. Holds metadata, IAM policy, replication config, rotation schedule. Does NOT hold the actual data. |
| **Secret Version** | Immutable payload container. The actual secret bytes live here. Each add creates a new version. |
| **Secret Data** | The actual bytes stored in a version — up to 64 KB, binary or UTF-8. |
| **Secret Metadata** | Labels, annotations, create time, replication config, rotation config — lives on the Secret resource. |
| **Replication Policy** | How and where versions are stored: `AUTOMATIC` (Google-managed) or `USER_MANAGED` (your regions). |
| **Rotation** | The process of creating a new version with updated credentials and deprecating the old one. |
| **Annotation** | Unstructured key-value metadata on a secret (for tooling, not IAM filtering). |
| **Label** | Structured key-value metadata on a secret (for IAM conditions, billing, resource filtering). |
| **Alias** | Named pointer to a specific version (`latest` is built-in; custom aliases are in preview). |

---

### Resource Hierarchy

```
GCP Project
└── Secret  (named resource — metadata, IAM, replication, rotation schedule)
    ├── Secret Version 1  (DESTROYED — payload gone, metadata remains)
    ├── Secret Version 2  (DISABLED — payload intact, access blocked)
    ├── Secret Version 3  (ENABLED — non-primary, accessible)
    └── Secret Version 4  (ENABLED — "latest", used by default)
```

- **Project** → billing boundary, IAM inheritance root
- **Secret** → logical credential identity; IAM policies bind here for per-secret access control
- **Secret Version** → immutable payload; state controls access; new credentials = new version number

---

### Secret Version States

```
  Add Version
      │
      ▼
  ┌──────────┐
  │  ENABLED  │◄──────────────────┐
  └─────┬─────┘                   │
        │ Disable                 │ Re-enable
        ▼                         │
  ┌──────────┐                    │
  │ DISABLED  │────────────────────┘
  └─────┬─────┘
        │ Destroy (or version-destroy-ttl expires)
        ▼
  ┌──────────┐
  │ DESTROYED │  ← IRREVERSIBLE — payload permanently deleted
  └──────────┘
```

| State | Payload Accessible | Metadata Accessible | Can Transition To |
|---|---|---|---|
| `ENABLED` | ✅ | ✅ | DISABLED, DESTROYED |
| `DISABLED` | ❌ | ✅ | ENABLED, DESTROYED |
| `DESTROYED` | ❌ | ✅ (name, create time only) | None |

---

### Key Limits & Quotas

| Resource | Limit |
|---|---|
| Secrets per project | 10,000 |
| Versions per secret | 10,000 |
| Max secret payload size | **64 KB** |
| `accessSecretVersion` requests / minute / project | 60,000 |
| Admin API requests / minute / project | 600 |
| Replication locations (USER_MANAGED) | Up to 10 regions |

> ⚠️ **64 KB is a hard limit.** TLS certificates, SSH keys, and most API keys are well under this. If your payload exceeds 64 KB, consider splitting it or storing large blobs in GCS with only the encryption key in Secret Manager.

---

## 2. Secrets & Secret Versions

### Naming Conventions

```
Rules:
  - 1–255 characters
  - Letters, digits, hyphens, underscores only
  - Must start with a letter or digit
  - Case-sensitive

Convention: {service}-{env}-{purpose}
Examples:
  payments-prod-db-password
  api-gateway-prod-stripe-key
  kafka-staging-tls-cert
  gke-prod-registry-credentials
```

---

### Creating Secrets

```bash
# Create with AUTOMATIC replication (simplest, highest availability)
gcloud secrets create payments-prod-db-password \
  --replication-policy=automatic \
  --labels=env=prod,team=payments,service=postgres \
  --project=my-project

# Create with USER_MANAGED replication (data residency)
gcloud secrets create payments-prod-db-password \
  --replication-policy=user-managed \
  --locations=europe-west1,europe-west3 \
  --labels=env=prod,team=payments \
  --project=my-project

# Create with rotation schedule and Pub/Sub topic
gcloud secrets create payments-prod-db-password \
  --replication-policy=automatic \
  --rotation-period=7776000s \
  --next-rotation-time="2024-07-01T00:00:00Z" \
  --topics=projects/my-project/topics/secret-rotation \
  --project=my-project
```

---

### Adding Secret Versions (the Payload)

```bash
# From a string (pipe via stdin)
echo -n "my-super-secret-password" | gcloud secrets versions add \
  payments-prod-db-password \
  --data-file=- \
  --project=my-project

# From a file
gcloud secrets versions add payments-prod-db-password \
  --data-file=/path/to/secret.txt \
  --project=my-project

# From a certificate file
gcloud secrets versions add api-gateway-prod-tls-cert \
  --data-file=server.crt \
  --project=my-project
```

> ⚠️ **Never put secret values directly in the `--data-file` flag as a string literal or in shell history.** Always use `--data-file=-` with stdin piping or a file.

---

### Listing and Describing

```bash
# List all secrets in a project
gcloud secrets list --project=my-project
gcloud secrets list --filter="labels.env=prod" --format="table(name,createTime,labels)"

# Describe a secret (metadata, replication, rotation)
gcloud secrets describe payments-prod-db-password --project=my-project

# List versions of a secret
gcloud secrets versions list payments-prod-db-password \
  --project=my-project \
  --format="table(name, state, createTime)"

# Describe a specific version
gcloud secrets versions describe 3 \
  --secret=payments-prod-db-password \
  --project=my-project
```

---

### Managing Version State

```bash
# Disable a version (payload intact, access blocked)
gcloud secrets versions disable 2 \
  --secret=payments-prod-db-password \
  --project=my-project

# Re-enable a disabled version
gcloud secrets versions enable 2 \
  --secret=payments-prod-db-password \
  --project=my-project

# Destroy a version (IRREVERSIBLE — payload deleted permanently)
gcloud secrets versions destroy 1 \
  --secret=payments-prod-db-password \
  --project=my-project
```

---

### Deleting a Secret

```bash
# Delete a secret and ALL its versions (IRREVERSIBLE)
gcloud secrets delete payments-prod-db-password \
  --project=my-project

# With confirmation bypass (for automation)
gcloud secrets delete payments-prod-db-password \
  --project=my-project \
  --quiet
```

> 🚨 **Deleting a secret is permanent.** All versions and their payloads are deleted. There is no soft-delete or recycle bin.

---

### Labels and Annotations

```bash
# Update labels on an existing secret
gcloud secrets update payments-prod-db-password \
  --update-labels=env=prod,team=payments,rotation-frequency=90d \
  --project=my-project

# Add annotations (unstructured metadata for tooling)
gcloud secrets update payments-prod-db-password \
  --update-annotations=managed-by=terraform,last-rotation=2024-04-01 \
  --project=my-project
```

---

### Terraform: Secrets and Versions

```hcl
# Secret with automatic replication
resource "google_secret_manager_secret" "db_password" {
  secret_id = "payments-prod-db-password"
  project   = var.project_id

  replication {
    auto {}
  }

  labels = {
    env     = "prod"
    team    = "payments"
    service = "postgres"
  }

  lifecycle {
    prevent_destroy = true  # Safety: never accidentally delete production secrets
  }
}

# Secret version (the actual payload)
resource "google_secret_manager_secret_version" "db_password_v1" {
  secret      = google_secret_manager_secret.db_password.id
  secret_data = var.db_password  # From a sensitive variable — never hardcode

  lifecycle {
    ignore_changes = [secret_data]  # Prevent Terraform from re-applying on drift
  }
}

# Secret with user-managed replication (data residency)
resource "google_secret_manager_secret" "eu_secret" {
  secret_id = "payments-prod-eu-api-key"
  project   = var.project_id

  replication {
    user_managed {
      replicas {
        location = "europe-west1"
      }
      replicas {
        location = "europe-west3"
      }
    }
  }
}
```

> ⚠️ **Terraform stores secret values in state.** Use `secret_data` from a sensitive variable or a data source, and ensure your Terraform state is encrypted (e.g., GCS backend with CMEK) and access-controlled.

---

## 3. Accessing Secret Versions

### How Access Works

```
Client Application
      │
      │  accessSecretVersion(name="projects/P/secrets/S/versions/V")
      ▼
Secret Manager API
      │
      │  Returns: { payload: { data: <base64 bytes> }, name, createTime }
      ▼
Client Application
      │
      │  base64_decode(payload.data) → your raw secret bytes
      ▼
    Use secret (never log it!)
```

---

### gcloud: Access Secret Versions

```bash
# Access the latest enabled version
gcloud secrets versions access latest \
  --secret=payments-prod-db-password \
  --project=my-project

# Access a specific version number
gcloud secrets versions access 3 \
  --secret=payments-prod-db-password \
  --project=my-project

# Access and save to file (for certificates, SSH keys)
gcloud secrets versions access latest \
  --secret=api-gateway-prod-tls-cert \
  --project=my-project \
  --out-file=/tmp/server.crt

# Access and decode binary secret to file
gcloud secrets versions access latest \
  --secret=my-binary-secret \
  --project=my-project \
  --format="get(payload.data)" | base64 --decode > output.bin
```

---

### Python: Access Secret Versions

```python
from google.cloud import secretmanager
from google.api_core.exceptions import NotFound, PermissionDenied, FailedPrecondition


def access_secret_latest(project_id: str, secret_id: str) -> str:
    """Access the latest enabled version of a secret."""
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("utf-8")


def access_secret_version(project_id: str, secret_id: str, version_id: str) -> str:
    """Access a specific version of a secret."""
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/{version_id}"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("utf-8")


def access_secret_safe(project_id: str, secret_id: str, version: str = "latest") -> str | None:
    """Access a secret with full error handling."""
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/{version}"
    try:
        response = client.access_secret_version(request={"name": name})
        return response.payload.data.decode("utf-8")
    except NotFound:
        raise ValueError(f"Secret {secret_id} version {version} does not exist")
    except PermissionDenied:
        raise PermissionError(f"No access to secret {secret_id}. Check IAM bindings.")
    except FailedPrecondition as e:
        raise RuntimeError(f"Secret version is DISABLED or DESTROYED: {e}")


def access_binary_secret(project_id: str, secret_id: str) -> bytes:
    """Access a binary secret (certificate, key file, etc.)."""
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data  # raw bytes, no decode
```

---

### Go: Access Secret Version

```go
import (
    "context"
    "fmt"
    secretmanager "cloud.google.com/go/secretmanager/apiv1"
    secretmanagerpb "cloud.google.com/go/secretmanager/apiv1/secretmanagerpb"
)

func accessSecretVersion(projectID, secretID, versionID string) (string, error) {
    ctx := context.Background()
    client, err := secretmanager.NewClient(ctx)
    if err != nil {
        return "", fmt.Errorf("failed to create client: %w", err)
    }
    defer client.Close()

    name := fmt.Sprintf("projects/%s/secrets/%s/versions/%s",
        projectID, secretID, versionID)

    result, err := client.AccessSecretVersion(ctx,
        &secretmanagerpb.AccessSecretVersionRequest{Name: name})
    if err != nil {
        return "", fmt.Errorf("failed to access secret: %w", err)
    }
    return string(result.Payload.Data), nil
}
```

---

### Node.js: Access Secret Version

```javascript
const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');

const client = new SecretManagerServiceClient();

async function accessSecretVersion(projectId, secretId, version = 'latest') {
  const name = `projects/${projectId}/secrets/${secretId}/versions/${version}`;
  const [response] = await client.accessSecretVersion({ name });
  return response.payload.data.toString('utf8');
}

// Usage
const password = await accessSecretVersion('my-project', 'payments-prod-db-password');
// NEVER: console.log(password)  ← logs secret to stdout
```

---

### REST API: Access Secret Version

```bash
# Access via REST
curl -s \
  "https://secretmanager.googleapis.com/v1/projects/PROJECT_ID/secrets/SECRET_ID/versions/latest:access" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  | python3 -c "import sys,json,base64; d=json.load(sys.stdin); print(base64.b64decode(d['payload']['data']).decode())"
```

---

### In-Process Secret Caching

```python
import time
from functools import lru_cache
from google.cloud import secretmanager

# Simple TTL cache (refresh every 5 minutes)
_secret_cache: dict[str, tuple[str, float]] = {}
SECRET_TTL = 300  # seconds


def get_secret_cached(project_id: str, secret_id: str) -> str:
    cache_key = f"{project_id}/{secret_id}"
    cached_value, cached_at = _secret_cache.get(cache_key, (None, 0))

    if cached_value and (time.time() - cached_at) < SECRET_TTL:
        return cached_value

    # Cache miss or expired — fetch from Secret Manager
    value = access_secret_latest(project_id, secret_id)
    _secret_cache[cache_key] = (value, time.time())
    return value
```

> 💡 **Cache secrets in-process with a TTL.** One Secret Manager call per process startup (or per TTL expiry) is the right model — not one call per request. For rotation-aware caching, re-fetch when you receive a rotation Pub/Sub notification or when an auth error suggests the credential changed.

---

### Safe Secret Handling Patterns

```python
import logging

# ❌ NEVER do this
secret = get_secret("payments-prod-db-password")
logging.info(f"Connected with password: {secret}")          # leaks secret to logs
print(f"Secret value: {secret}")                             # leaks to stdout
os.environ["DB_PASSWORD"] = secret                           # leaks to /proc/environ

# ✅ Safe patterns
secret = get_secret("payments-prod-db-password")
logging.info("Successfully retrieved DB password from Secret Manager")  # log the action, not the value

# Use directly without intermediate variables where possible
connection = create_db_connection(
    host=DB_HOST,
    password=get_secret("payments-prod-db-password")         # used immediately, not stored
)

# When you must store, use a class that masks __repr__ and __str__
class SecretString:
    def __init__(self, value: str):
        self._value = value
    def __repr__(self): return "SecretString(***)"
    def __str__(self):  return "***"
    def get(self) -> str: return self._value
```

---

## 4. Secret Versioning & Lifecycle

### Immutability

Secret versions are **immutable** — once created, the payload cannot be changed. To update a secret, you always add a new version. This design:
- Ensures audit completeness (full history preserved)
- Enables rollback to a previous version by re-enabling it
- Makes rotation safe (old version stays accessible during transition)

---

### Version Aliases

The built-in `latest` alias always points to the **highest-numbered ENABLED version**.

**Custom aliases (preview):**
```bash
# Not yet GA — check current availability
# Intended pattern for blue/green rotation:
#   alias "prod" → version 5 (stable)
#   alias "canary" → version 6 (new, being tested)
#   promote: update "prod" alias to version 6
```

---

### Version Destroy TTL

```bash
# Create a secret with automatic delayed destruction after disable
gcloud secrets create payments-prod-db-password \
  --replication-policy=automatic \
  --version-destroy-ttl=86400s  # Auto-destroy disabled versions after 24 hours

# This means: when you DISABLE a version, it will automatically
# transition to DESTROYED after 24 hours — no manual cleanup needed
```

---

### Python: Version Lifecycle Management

```python
from google.cloud import secretmanager
from google.protobuf import field_mask_pb2


def list_versions_with_state(project_id: str, secret_id: str) -> list:
    client = secretmanager.SecretManagerServiceClient()
    parent = f"projects/{project_id}/secrets/{secret_id}"
    versions = []
    for version in client.list_secret_versions(request={"parent": parent}):
        versions.append({
            "name": version.name,
            "state": version.state.name,
            "create_time": version.create_time,
        })
    return versions


def disable_version(project_id: str, secret_id: str, version_id: str):
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/{version_id}"
    return client.disable_secret_version(request={"name": name})


def destroy_version(project_id: str, secret_id: str, version_id: str):
    """IRREVERSIBLE — permanently deletes the secret payload."""
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/{version_id}"
    return client.destroy_secret_version(request={"name": name})
```

---

### Version Hygiene Best Practices

| Scenario | Recommended Action |
|---|---|
| After successful rotation | Disable old version immediately; destroy after 24–48 hr window |
| Emergency rollback needed | Re-enable previous version; update consumers |
| Compliance audit | Keep version metadata (destroyed state) — payload gone but audit trail remains |
| Development secrets | Destroy old versions aggressively (no retention needed) |
| Production secrets | Retain 2 versions max (current + previous); destroy all others |

---

## 5. Replication Policies

### AUTOMATIC vs. USER_MANAGED

```
AUTOMATIC Replication
─────────────────────────────────────────────────────
  Secret Manager auto-replicates to multiple regions.
  You do NOT choose which regions.
  Google ensures availability and durability.

  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │  Region A   │     │  Region B   │     │  Region C   │
  │  (replica)  │     │  (replica)  │     │  (replica)  │
  └─────────────┘     └─────────────┘     └─────────────┘
        ▲                   ▲                   ▲
        └───────────────────┴───────────────────┘
                  Managed by Google

USER_MANAGED Replication
─────────────────────────────────────────────────────
  You specify exactly which GCP regions hold replicas.
  Explicit data residency guarantees.

  ┌─────────────────────┐    ┌─────────────────────┐
  │   europe-west1      │    │   europe-west3      │
  │  (GDPR region)      │    │  (GDPR region)      │
  │  Optional: KMS key  │    │  Optional: KMS key  │
  └─────────────────────┘    └─────────────────────┘
```

> ⚠️ **Replication policy is IMMUTABLE.** It is set at secret creation and **cannot be changed**. If you need to change replication, you must create a new secret, copy the version data, and update all consumers.

---

### Decision Guide

| Requirement | Use |
|---|---|
| Highest availability, no data residency constraints | `AUTOMATIC` |
| GDPR — data must stay in EU | `USER_MANAGED` with EU regions only |
| Single-region for compliance auditing | `USER_MANAGED` with one region |
| Per-region CMEK (different KMS key per region) | `USER_MANAGED` |
| US government workloads (data in US only) | `USER_MANAGED` with `us-central1` / `us-east1` |

---

### gcloud: Replication

```bash
# AUTOMATIC (default — simplest)
gcloud secrets create my-secret \
  --replication-policy=automatic

# USER_MANAGED — single region
gcloud secrets create my-secret \
  --replication-policy=user-managed \
  --locations=europe-west1

# USER_MANAGED — multi-region (same continent)
gcloud secrets create my-secret \
  --replication-policy=user-managed \
  --locations=europe-west1,europe-west3,europe-north1

# USER_MANAGED with per-replica CMEK
gcloud secrets create my-secret \
  --replication-policy=user-managed \
  --locations=europe-west1 \
  --kms-key-name=projects/P/locations/europe-west1/keyRings/RING/cryptoKeys/KEY
```

---

### Terraform: User-Managed Replication with CMEK

```hcl
resource "google_secret_manager_secret" "gdpr_secret" {
  secret_id = "payments-prod-eu-db-password"
  project   = var.project_id

  replication {
    user_managed {
      replicas {
        location = "europe-west1"
        customer_managed_encryption {
          kms_key_name = google_kms_crypto_key.eu_w1_key.id
        }
      }
      replicas {
        location = "europe-west3"
        customer_managed_encryption {
          kms_key_name = google_kms_crypto_key.eu_w3_key.id
        }
      }
    }
  }
}
```

---

## 6. IAM & Access Control

### IAM Roles Reference

| Role | Permissions | Who Needs It |
|---|---|---|
| `roles/secretmanager.admin` | Full control: create, delete, update secrets; manage versions; set IAM | Security/platform team |
| `roles/secretmanager.secretAccessor` | `accessSecretVersion` — read secret payloads | Applications, services, pipelines |
| `roles/secretmanager.secretVersionAdder` | `addSecretVersion` — add new versions, cannot read | CI/CD, rotation functions |
| `roles/secretmanager.secretVersionManager` | Enable, disable, destroy versions; add versions | Rotation automation, lifecycle mgmt |
| `roles/secretmanager.viewer` | List and describe secrets and versions; cannot read payloads | Auditors, monitoring tools |

> 🛡️ **Separation of duties:** The key insight — `secretVersionAdder` can write new secrets but CANNOT read them. `secretAccessor` can read but cannot create. Never assign both to the same principal unless required.

---

### Resource-Level IAM (Principle of Least Privilege)

```bash
# ✅ CORRECT: Grant accessor at the secret level (not project level)
gcloud secrets add-iam-policy-binding payments-prod-db-password \
  --project=my-project \
  --member="serviceAccount:payments-svc@my-project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# ❌ AVOID: Granting at project level gives access to ALL secrets
# gcloud projects add-iam-policy-binding my-project \
#   --member="serviceAccount:..." \
#   --role="roles/secretmanager.secretAccessor"

# Grant secretVersionAdder to CI/CD SA (can rotate, cannot read)
gcloud secrets add-iam-policy-binding payments-prod-db-password \
  --project=my-project \
  --member="serviceAccount:cicd-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretVersionAdder"

# View IAM policy for a secret
gcloud secrets get-iam-policy payments-prod-db-password \
  --project=my-project

# Remove a binding
gcloud secrets remove-iam-policy-binding payments-prod-db-password \
  --project=my-project \
  --member="serviceAccount:old-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

---

### Python: IAM Policy Management

```python
from google.cloud import secretmanager
from google.iam.v1 import iam_policy_pb2


def get_secret_iam_policy(project_id: str, secret_id: str):
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}"
    return client.get_iam_policy(request={"resource": name})


def grant_secret_accessor(project_id: str, secret_id: str, member: str):
    """Grant secretAccessor to a member on a specific secret."""
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}"

    policy = client.get_iam_policy(request={"resource": name})
    policy.bindings.add(
        role="roles/secretmanager.secretAccessor",
        members=[member]
    )
    return client.set_iam_policy(
        request={"resource": name, "policy": policy}
    )
```

---

### Conditional IAM

```bash
# Grant access only within business hours (example condition)
gcloud secrets add-iam-policy-binding my-secret \
  --project=my-project \
  --member="serviceAccount:my-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor" \
  --condition='expression=request.time.getHours("America/New_York") >= 9 && request.time.getHours("America/New_York") < 17,title=business-hours-only'

# Grant access based on resource label
gcloud secrets add-iam-policy-binding my-secret \
  --project=my-project \
  --member="serviceAccount:my-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor" \
  --condition='expression=resource.name.startsWith("projects/PROJECT_NUMBER/secrets/payments-"),title=payments-secrets-only'
```

---

### Audit Logging

| Log Type | Default | Operations Captured |
|---|---|---|
| **Admin Activity** | Always on | `CreateSecret`, `DeleteSecret`, `UpdateSecret`, `SetIamPolicy`, `AddSecretVersion`, `EnableSecretVersion`, `DisableSecretVersion`, `DestroySecretVersion` |
| **Data Access** | **Must enable** | `AccessSecretVersion`, `GetIamPolicy`, `ListSecrets`, `GetSecretVersion` |

> 🚨 **Enable Data Access audit logs in production.** Without them, you have no record of who accessed which secret — a critical gap for compliance and incident response.

```bash
# Enable Data Access audit logging for Secret Manager
# Add to your org/project IAM policy via Console: IAM → Audit Logs → Secret Manager API → tick Data Read + Data Write
# Or via gcloud (update the project's audit config):
cat > audit_policy.yaml << 'EOF'
auditConfigs:
- auditLogConfigs:
  - logType: DATA_READ
  - logType: DATA_WRITE
  service: secretmanager.googleapis.com
EOF
gcloud projects set-iam-policy my-project audit_policy.yaml
```

---

## 7. Secret Rotation

### Rotation Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Automatic Rotation Flow                       │
│                                                                   │
│  Secret Manager          Pub/Sub            Rotation Function    │
│  ┌──────────────┐   ┌───────────────┐   ┌────────────────────┐  │
│  │ rotation_    │──►│ secret-       │──►│  1. Get notification│  │
│  │ period fires │   │ rotation topic│   │  2. Generate new   │  │
│  │              │   │               │   │     credential     │  │
│  │ Publishes    │   │ Message:      │   │  3. Add new version│  │
│  │ notification │   │ - secret name │   │  4. Update target  │  │
│  │              │   │ - version num │   │     service        │  │
│  └──────────────┘   │ - project id  │   │  5. Disable old    │  │
│                      └───────────────┘   │     version        │  │
│                                          └────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

### Setting Up Rotation

```bash
# Create a Pub/Sub topic for rotation notifications
gcloud pubsub topics create secret-rotation --project=my-project

# Create secret with rotation schedule
gcloud secrets create payments-prod-db-password \
  --replication-policy=automatic \
  --rotation-period=7776000s \
  --next-rotation-time="2024-07-01T00:00:00Z" \
  --topics=projects/my-project/topics/secret-rotation

# Update rotation schedule on an existing secret
gcloud secrets update payments-prod-db-password \
  --rotation-period=2592000s \
  --next-rotation-time="2024-05-01T00:00:00Z" \
  --project=my-project

# Remove rotation schedule
gcloud secrets update payments-prod-db-password \
  --remove-rotation-schedule \
  --project=my-project
```

---

### Rotation Notification Payload

```json
{
  "name": "projects/123456789/secrets/payments-prod-db-password/versions/5",
  "labels": {
    "env": "prod",
    "team": "payments"
  },
  "notificationBody": {
    "secretId": "payments-prod-db-password",
    "versionId": "5",
    "eventType": "SECRET_ROTATE"
  }
}
```

---

### Terraform: Secret with Rotation

```hcl
resource "google_pubsub_topic" "rotation_topic" {
  name    = "secret-rotation"
  project = var.project_id
}

resource "google_secret_manager_secret" "db_password" {
  secret_id = "payments-prod-db-password"
  project   = var.project_id

  replication { auto {} }

  rotation {
    rotation_period    = "7776000s"  # 90 days
    next_rotation_time = "2024-07-01T00:00:00Z"
  }

  topics {
    name = google_pubsub_topic.rotation_topic.id
  }
}

# Allow Secret Manager to publish to the topic
resource "google_pubsub_topic_iam_member" "secret_manager_publisher" {
  topic   = google_pubsub_topic.rotation_topic.name
  project = var.project_id
  role    = "roles/pubsub.publisher"
  member  = "serviceAccount:service-${var.project_number}@gcp-sa-secretmanager.iam.gserviceaccount.com"
}
```

---

### Python: Cloud Function Rotation Handler

```python
import base64
import json
import functions_framework
from google.cloud import secretmanager
import secrets
import string


@functions_framework.cloud_event
def rotate_secret(cloud_event):
    """Cloud Function triggered by Secret Manager rotation notification."""
    # 1. Parse the Pub/Sub message
    pubsub_message = base64.b64decode(cloud_event.data["message"]["data"]).decode()
    payload = json.loads(pubsub_message)

    project_id = cloud_event.data["message"]["attributes"]["googproject"]
    secret_id = payload.get("secretId") or cloud_event.data["message"]["attributes"]["secretId"]

    print(f"Rotation triggered for secret: {secret_id}")

    client = secretmanager.SecretManagerServiceClient()
    parent = f"projects/{project_id}/secrets/{secret_id}"

    # 2. Generate new credential (example: random password)
    new_credential = _generate_new_password()

    # 3. Store current version number (to disable after rotation)
    current_versions = list(client.list_secret_versions(
        request={"parent": parent, "filter": "state:ENABLED"}
    ))
    old_version_names = [v.name for v in current_versions]

    # 4. Add new secret version
    new_version = client.add_secret_version(
        request={
            "parent": parent,
            "payload": {"data": new_credential.encode("utf-8")},
        }
    )
    print(f"Created new version: {new_version.name}")

    # 5. TODO: Update target service with the new credential
    # e.g., _update_database_password(new_credential)
    # e.g., _update_api_key_with_provider(new_credential)

    # 6. Disable old versions (after verifying new version works)
    for old_version_name in old_version_names:
        client.disable_secret_version(request={"name": old_version_name})
        print(f"Disabled old version: {old_version_name}")


def _generate_new_password(length: int = 32) -> str:
    """Generate a cryptographically secure random password."""
    alphabet = string.ascii_letters + string.digits + "!@#$%^&*()"
    return "".join(secrets.choice(alphabet) for _ in range(length))
```

---

### Zero-Downtime Rotation Pattern

```
Timeline:
─────────────────────────────────────────────────────────────────
 T+0:  Add new version (v6). Both v5 and v6 are ENABLED.
       Applications using "latest" automatically get v6.
       Applications pinned to "5" still get v5.

 T+0 to T+30min: Monitor — verify all services work with v6.
       No consumers are broken (v5 still accessible).

 T+30min: Disable v5 (old version).
       All consumers now use v6.
       If something breaks, re-enable v5 immediately.

 T+24hr: Destroy v5 (payload permanently deleted).
─────────────────────────────────────────────────────────────────
```

> 💡 **Never destroy the old version immediately.** Keep both ENABLED for a validation window (30 minutes to 24 hours depending on your deployment cadence) before disabling, then wait another window before destroying.

---

## 8. CMEK (Customer-Managed Encryption Keys)

### How Encryption Works

```
Default (Google-managed keys):
  Secret payload → AES-256 encryption → Google-managed KMS key
  You have no key management responsibility.

CMEK (Customer-managed keys):
  Secret payload → AES-256 encryption → YOUR Cloud KMS key
  You control key rotation, key destruction, key access.
  If your KMS key is destroyed → secret versions PERMANENTLY INACCESSIBLE.
```

---

### Setting Up CMEK

```bash
# Step 1: Get the Secret Manager service account for your project
# Format: service-PROJECT_NUMBER@gcp-sa-secretmanager.iam.gserviceaccount.com
PROJECT_NUMBER=$(gcloud projects describe my-project --format="value(projectNumber)")
SM_SA="service-${PROJECT_NUMBER}@gcp-sa-secretmanager.iam.gserviceaccount.com"

# Step 2: Grant cryptoKeyEncrypterDecrypter to Secret Manager SA
gcloud kms keys add-iam-policy-binding my-kms-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --member="serviceAccount:${SM_SA}" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Step 3: Create a CMEK-encrypted secret (AUTOMATIC replication)
gcloud secrets create my-cmek-secret \
  --replication-policy=automatic \
  --kms-key-name=projects/my-project/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-key

# Step 4: Create a CMEK-encrypted secret (USER_MANAGED — per-replica key)
gcloud secrets create my-cmek-eu-secret \
  --replication-policy=user-managed \
  --locations=europe-west1 \
  --kms-key-name=projects/my-project/locations/europe-west1/keyRings/eu-keyring/cryptoKeys/eu-key
```

---

### Terraform: CMEK Secret

```hcl
# Grant Secret Manager SA access to the KMS key
resource "google_kms_crypto_key_iam_member" "secret_manager_cmek" {
  crypto_key_id = google_kms_crypto_key.secret_key.id
  role          = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
  member        = "serviceAccount:service-${data.google_project.project.number}@gcp-sa-secretmanager.iam.gserviceaccount.com"
}

# CMEK-encrypted secret with user-managed replication and per-replica keys
resource "google_secret_manager_secret" "cmek_secret" {
  secret_id = "payments-prod-cmek-db-password"
  project   = var.project_id

  replication {
    user_managed {
      replicas {
        location = "europe-west1"
        customer_managed_encryption {
          kms_key_name = google_kms_crypto_key.eu_w1_key.id
        }
      }
      replicas {
        location = "europe-west3"
        customer_managed_encryption {
          kms_key_name = google_kms_crypto_key.eu_w3_key.id
        }
      }
    }
  }

  depends_on = [google_kms_crypto_key_iam_member.secret_manager_cmek]
}
```

> 🚨 **KMS key destruction = permanent data loss.** If you disable or destroy the KMS key used to encrypt a secret, all versions of that secret become permanently inaccessible. Secret Manager cannot decrypt them without the KMS key. Monitor your KMS keys with destruction alerts when CMEK is in use.

---

## 9. Secret Manager Integrations

### Cloud Run

```bash
# Environment variable injection (secret value available as env var)
gcloud run deploy my-service \
  --image=gcr.io/my-project/my-app \
  --set-secrets=DB_PASSWORD=payments-prod-db-password:latest \
  --set-secrets=API_KEY=api-gateway-prod-key:3 \
  --region=us-central1

# Volume mount (secret value available as file at /secrets/db-password)
gcloud run deploy my-service \
  --image=gcr.io/my-project/my-app \
  --set-secrets=/secrets/db-password=payments-prod-db-password:latest \
  --region=us-central1
```

**Cloud Run YAML spec:**
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-service
spec:
  template:
    spec:
      serviceAccountName: my-run-sa@my-project.iam.gserviceaccount.com
      containers:
      - image: gcr.io/my-project/my-app
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: payments-prod-db-password
              key: latest
        volumeMounts:
        - name: tls-cert
          mountPath: /secrets/tls
          readOnly: true
      volumes:
      - name: tls-cert
        secret:
          secretName: api-gateway-prod-tls-cert
```

> 💡 **Prefer volume mounts for large secrets** (certificates, key files). Prefer env var injection for small values (passwords, tokens) consumed by frameworks that expect env vars.

---

### GKE: Secrets Store CSI Driver

```bash
# Step 1: Enable the Secret Manager add-on on your GKE cluster
gcloud container clusters update my-cluster \
  --enable-secret-manager \
  --region=us-central1

# Step 2: Install Secrets Store CSI Driver (if not using add-on)
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true
```

**SecretProviderClass:**
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: payments-secrets
  namespace: default
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/my-project/secrets/payments-prod-db-password/versions/latest"
        path: "db-password"
      - resourceName: "projects/my-project/secrets/payments-prod-tls-cert/versions/latest"
        path: "tls.crt"
  secretObjects:                          # Sync to Kubernetes Secret
  - secretName: payments-k8s-secret
    type: Opaque
    data:
    - objectName: db-password
      key: DB_PASSWORD
```

**Pod spec using the CSI volume:**
```yaml
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: payments-ksa  # Must have secretAccessor IAM binding via Workload Identity
  containers:
  - name: app
    image: gcr.io/my-project/payments-app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: payments-k8s-secret
          key: DB_PASSWORD
    volumeMounts:
    - name: secrets-vol
      mountPath: "/var/secrets"
      readOnly: true
  volumes:
  - name: secrets-vol
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: payments-secrets
```

---

### Cloud Build: Secret Injection

```yaml
# cloudbuild.yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    docker build \
      --build-arg NPM_TOKEN=$$NPM_TOKEN \
      -t gcr.io/$PROJECT_ID/my-app .
  secretEnv:
  - 'NPM_TOKEN'

availableSecrets:
  secretManager:
  - versionName: projects/$PROJECT_ID/secrets/npm-registry-token/versions/latest
    env: 'NPM_TOKEN'
```

---

### Cloud Functions (Gen 2)

```bash
# Inject as environment variable at deploy time
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python311 \
  --entry-point=handler \
  --set-secrets=DB_PASSWORD=payments-prod-db-password:latest \
  --region=us-central1
```

---

### Compute Engine

```python
# Access at VM startup (e.g., in a startup script)
# SA must have secretAccessor IAM binding

from google.cloud import secretmanager
import subprocess

def get_secret(secret_id: str) -> str:
    project_id = subprocess.check_output(
        ["curl", "-s", "http://metadata.google.internal/computeMetadata/v1/project/project-id",
         "-H", "Metadata-Flavor: Google"]
    ).decode().strip()
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("utf-8")
```

---

### Terraform: Data Source (⚠️ State Caveat)

```hcl
# Reading a secret value in Terraform (warning: stored in state)
data "google_secret_manager_secret_version" "db_password" {
  secret  = "payments-prod-db-password"
  version = "latest"
  project = var.project_id
}

resource "google_cloud_run_service" "app" {
  # ...
  template {
    spec {
      containers {
        env {
          name  = "DB_PASSWORD"
          value = data.google_secret_manager_secret_version.db_password.secret_data
        }
      }
    }
  }
}
```

> ⚠️ **Secret values read via Terraform data sources are stored in Terraform state in plaintext.** Ensure your state backend (GCS bucket) is encrypted (CMEK recommended) and strictly access-controlled. Prefer using Cloud Run's native `--set-secrets` integration over Terraform data sources where possible.

---

## 10. Monitoring, Alerting & Audit Logging

### Cloud Logging Queries

```bash
# Who accessed a specific secret in the last 24 hours?
gcloud logging read \
  'resource.type="audited_resource"
   protoPayload.serviceName="secretmanager.googleapis.com"
   protoPayload.methodName="google.cloud.secretmanager.v1.SecretManagerService.AccessSecretVersion"
   protoPayload.resourceName:"secrets/payments-prod-db-password"' \
  --freshness=24h \
  --format="table(timestamp, protoPayload.authenticationInfo.principalEmail)"

# All failed secret access attempts (PERMISSION_DENIED)
gcloud logging read \
  'resource.type="audited_resource"
   protoPayload.serviceName="secretmanager.googleapis.com"
   protoPayload.status.code=7' \
  --freshness=1h

# Secrets created or deleted this week
gcloud logging read \
  'resource.type="audited_resource"
   protoPayload.serviceName="secretmanager.googleapis.com"
   protoPayload.methodName=~"CreateSecret|DeleteSecret"' \
  --freshness=7d

# All accesses by a specific service account
gcloud logging read \
  'resource.type="audited_resource"
   protoPayload.serviceName="secretmanager.googleapis.com"
   protoPayload.authenticationInfo.principalEmail="my-sa@my-project.iam.gserviceaccount.com"' \
  --freshness=7d

# Secret version destruction events
gcloud logging read \
  'resource.type="audited_resource"
   protoPayload.serviceName="secretmanager.googleapis.com"
   protoPayload.methodName="google.cloud.secretmanager.v1.SecretManagerService.DestroySecretVersion"' \
  --freshness=30d
```

---

### Log-Based Metric: Unauthorized Access Attempts

```bash
gcloud logging metrics create secret-manager-permission-denied \
  --description="Secret Manager PERMISSION_DENIED errors" \
  --log-filter='resource.type="audited_resource"
protoPayload.serviceName="secretmanager.googleapis.com"
protoPayload.status.code=7'
```

---

### Cloud Monitoring Alerts

```bash
# Alert on unauthorized access attempts
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="Secret Manager: Unauthorized Access Attempts" \
  --condition-filter='metric.type="logging.googleapis.com/user/secret-manager-permission-denied"' \
  --condition-threshold-value=5 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=60s

# Alert on high-sensitivity secret access (log-based metric for specific secret)
gcloud logging metrics create prod-db-password-access \
  --log-filter='protoPayload.resourceName:"secrets/payments-prod-db-password"
protoPayload.methodName:"AccessSecretVersion"'
```

---

### BigQuery Security Analytics

```sql
-- Export audit logs to BigQuery and query:
-- Which secrets are accessed most, by which identities?
SELECT
  REGEXP_EXTRACT(
    protopayload_auditlog.resourceName,
    r'secrets/([^/]+)'
  ) AS secret_id,
  protopayload_auditlog.authenticationInfo.principalEmail AS accessor,
  COUNT(*) AS access_count,
  MIN(timestamp) AS first_access,
  MAX(timestamp) AS last_access
FROM `PROJECT.DATASET.cloudaudit_googleapis_com_data_access_*`
WHERE
  protopayload_auditlog.serviceName = 'secretmanager.googleapis.com'
  AND protopayload_auditlog.methodName LIKE '%AccessSecretVersion%'
  AND _TABLE_SUFFIX BETWEEN '20240101' AND '20241231'
GROUP BY secret_id, accessor
ORDER BY access_count DESC
LIMIT 50;
```

---

## 11. Best Practices & Security Patterns

### ✅ Do This

| Practice | Why |
|---|---|
| **One secret per credential** | Easier rotation, narrower blast radius, granular IAM |
| **Separate secrets per environment** | `payments-dev-db-password` ≠ `payments-prod-db-password` |
| **Grant `secretAccessor` at secret level** | Not at project level — limits breach impact |
| **Enable Data Access audit logs** | Essential for compliance and incident response |
| **Set automatic rotation for long-lived credentials** | Compliance, reduces blast radius of compromise |
| **Use `version-destroy-ttl`** | Automatic cleanup after rotation; reduces version sprawl |
| **Cache secrets with TTL** | One SM call per process/startup — not per request |
| **Label all secrets** | `env`, `team`, `service`, `rotation-frequency` for governance |
| **Use Workload Identity Federation** | No service account key files — the most common secret to compromise |
| **Enable CMEK for regulated workloads** | PCI-DSS, GDPR — key material control |

---

### ❌ Anti-Patterns

| Anti-Pattern | Risk | Fix |
|---|---|---|
| Secrets in source code | Committed to git, leaked in PRs | Use Secret Manager; add `.env` to `.gitignore` |
| Secrets in Dockerfile `ENV` | Exposed in image layers, `docker inspect` | Use Secret Manager; inject at runtime |
| Secrets in env vars (prod containers) | Leaked via `/proc/environ`, logging | Use SM volume mounts or SDK access |
| `secretAccessor` at project level | One compromised SA = all secrets exposed | Bind at secret level |
| Same secret across environments | Dev compromise exposes prod | Separate secrets per env |
| Accessing `latest` on every request | Unnecessary SM API calls, latency, cost | Cache in-process with TTL |
| Never rotating secrets | Prolonged exposure if credential leaks | Automate rotation with Pub/Sub |
| Never destroying old versions | Version accumulation, cost, stale credentials accessible | Disable then destroy after rotation window |
| Storing secrets in GCS without CMEK | Bucket ACL misconfiguration = exposure | Use Secret Manager |
| Secrets in Terraform state (unencrypted) | State file = plaintext credentials | Encrypt state with CMEK; prefer runtime injection |

---

### The Secret Zero Problem

> *How do you authenticate to Secret Manager to get your first secret?*

```
Solutions (in order of preference):

1. Workload Identity (best for GCP):
   GKE/Cloud Run/Compute → Workload Identity → IAM → Secret Manager
   No credentials needed; identity is the VM/pod metadata itself.

2. Workload Identity Federation (for external workloads):
   GitHub Actions / AWS Lambda / on-prem service
   → OIDC token → Google STS → Short-lived token → Secret Manager
   No service account key files.

3. Secret Manager Bootstrap (for on-prem without WIF):
   Store a single, minimal-privilege service account key in your
   secure build system (e.g., HashiCorp Vault, CI/CD secret store)
   that can ONLY access a bootstrap secret which contains the
   credentials for full access.

4. Avoid: service account key files checked into repos or
   passed as environment variables.
```

---

### Security Checklist

```
Secrets Organization
  [ ] One secret per credential (not bundled JSON blobs)
  [ ] Separate secrets per environment (prod/staging/dev)
  [ ] All secrets labeled: env, team, service, rotation-frequency
  [ ] Naming convention enforced: {service}-{env}-{purpose}

IAM
  [ ] secretAccessor granted at secret level, not project level
  [ ] No human users have secretAccessor (use SA + Workload Identity)
  [ ] Separation: secretVersionAdder ≠ secretAccessor on same principal
  [ ] Regular IAM binding review (quarterly)
  [ ] VPC Service Controls configured for Secret Manager

Rotation
  [ ] All DB passwords have automatic rotation schedule (≤90 days)
  [ ] All API keys have rotation schedule (≤1 year)
  [ ] Zero-downtime rotation pattern in use (overlap window before disable)
  [ ] Old versions destroyed after rotation window (or version-destroy-ttl set)

Audit & Monitoring
  [ ] Data Access audit logs enabled for Secret Manager
  [ ] Alert on PERMISSION_DENIED for Secret Manager operations
  [ ] Alert on DestroySecretVersion events
  [ ] Audit logs exported to BigQuery for analytics
  [ ] Regular review of who accessed which secrets

Encryption
  [ ] CMEK enabled for secrets in regulated environments
  [ ] Terraform state encrypted (GCS backend with CMEK)
  [ ] Secret values never logged or printed in application code

Workloads
  [ ] No service account key files — Workload Identity everywhere
  [ ] Secrets cached in-process with appropriate TTL
  [ ] Cloud Run services use native --set-secrets integration
  [ ] GKE pods use CSI driver or SDK, not Kubernetes Secrets
```

---

## 12. gcloud CLI & REST API Quick Reference

### Secrets

```bash
# Create
gcloud secrets create SECRET_ID \
  [--replication-policy=automatic|user-managed] \
  [--locations=REGION1,REGION2] \
  [--kms-key-name=KMS_KEY] \
  [--rotation-period=DURATION] \
  [--next-rotation-time=DATETIME] \
  [--topics=PUBSUB_TOPIC] \
  [--labels=KEY=VALUE,...] \
  [--annotations=KEY=VALUE,...] \
  [--version-destroy-ttl=DURATION] \
  --project=PROJECT_ID

# List / Describe / Update / Delete
gcloud secrets list [--filter=FILTER] [--format=FORMAT] --project=PROJECT_ID
gcloud secrets describe SECRET_ID --project=PROJECT_ID
gcloud secrets update SECRET_ID [--update-labels=...] [--rotation-period=...] --project=PROJECT_ID
gcloud secrets delete SECRET_ID [--quiet] --project=PROJECT_ID

# IAM
gcloud secrets add-iam-policy-binding SECRET_ID --member=MEMBER --role=ROLE --project=PROJECT_ID
gcloud secrets remove-iam-policy-binding SECRET_ID --member=MEMBER --role=ROLE --project=PROJECT_ID
gcloud secrets get-iam-policy SECRET_ID --project=PROJECT_ID
gcloud secrets set-iam-policy SECRET_ID POLICY_FILE --project=PROJECT_ID
```

### Secret Versions

```bash
# Add / List / Describe
gcloud secrets versions add SECRET_ID --data-file=FILE_OR_- --project=PROJECT_ID
gcloud secrets versions list SECRET_ID [--filter=state:ENABLED] --project=PROJECT_ID
gcloud secrets versions describe VERSION --secret=SECRET_ID --project=PROJECT_ID

# Access
gcloud secrets versions access VERSION --secret=SECRET_ID [--out-file=FILE] --project=PROJECT_ID
gcloud secrets versions access latest --secret=SECRET_ID --project=PROJECT_ID

# Lifecycle
gcloud secrets versions enable VERSION --secret=SECRET_ID --project=PROJECT_ID
gcloud secrets versions disable VERSION --secret=SECRET_ID --project=PROJECT_ID
gcloud secrets versions destroy VERSION --secret=SECRET_ID --project=PROJECT_ID
```

### Key Flags Reference

| Flag | Command(s) | Description |
|---|---|---|
| `--replication-policy` | `create` | `automatic` or `user-managed` |
| `--locations` | `create` | Regions for user-managed replication |
| `--kms-key-name` | `create` | Full KMS key resource name for CMEK |
| `--rotation-period` | `create`, `update` | e.g., `7776000s` (90 days) |
| `--next-rotation-time` | `create`, `update` | RFC 3339 datetime |
| `--topics` | `create`, `update` | Pub/Sub topic for rotation notifications |
| `--labels` | `create`, `update` | `key=value,...` — for filtering and governance |
| `--annotations` | `create`, `update` | `key=value,...` — unstructured metadata |
| `--version-destroy-ttl` | `create`, `update` | Auto-destroy disabled versions after duration |
| `--data-file` | `versions add` | File path or `-` for stdin |
| `--out-file` | `versions access` | Write secret to file instead of stdout |
| `--filter` | `list`, `versions list` | CEL filter expression |
| `--format` | Any | `json`, `yaml`, `table(field,...)` |

---

### REST API Reference

```
Base URL: https://secretmanager.googleapis.com/v1/

Resource Paths:
  Project:        projects/{project}
  Secret:         projects/{project}/secrets/{secret}
  Secret Version: projects/{project}/secrets/{secret}/versions/{version}

Operations:
  POST   .../secrets                           Create secret
  GET    .../secrets/{s}                       Get secret metadata
  PATCH  .../secrets/{s}                       Update secret
  DELETE .../secrets/{s}                       Delete secret
  GET    .../secrets                           List secrets

  POST   .../secrets/{s}:addVersion            Add secret version
  GET    .../secrets/{s}/versions              List versions
  GET    .../secrets/{s}/versions/{v}          Get version metadata
  POST   .../secrets/{s}/versions/{v}:access   Access version payload
  POST   .../secrets/{s}/versions/{v}:enable   Enable version
  POST   .../secrets/{s}/versions/{v}:disable  Disable version
  POST   .../secrets/{s}/versions/{v}:destroy  Destroy version

  GET    .../secrets/{s}:getIamPolicy          Get IAM policy
  POST   .../secrets/{s}:setIamPolicy          Set IAM policy
  POST   .../secrets/{s}:testIamPermissions    Test permissions
```

---

### Python Client Library

```python
# Install
# pip install google-cloud-secret-manager

from google.cloud import secretmanager

client = secretmanager.SecretManagerServiceClient()

# Resource path builders
client.secret_path(project, secret)
client.secret_version_path(project, secret, version)

# Secret management
client.create_secret(request={"parent": f"projects/{p}", "secret_id": ..., "secret": {...}})
client.get_secret(request={"name": ...})
client.update_secret(request={"secret": ..., "update_mask": ...})
client.delete_secret(request={"name": ...})
client.list_secrets(request={"parent": f"projects/{p}"})

# Version management
client.add_secret_version(request={"parent": ..., "payload": {"data": bytes}})
client.get_secret_version(request={"name": ...})
client.list_secret_versions(request={"parent": ..., "filter": "state:ENABLED"})
client.access_secret_version(request={"name": ...})  # returns payload
client.enable_secret_version(request={"name": ...})
client.disable_secret_version(request={"name": ...})
client.destroy_secret_version(request={"name": ...})

# IAM
client.get_iam_policy(request={"resource": ...})
client.set_iam_policy(request={"resource": ..., "policy": ...})
```

---

## 13. Pricing Summary

> 💰 **No free tier for Secret Manager.** All active secret versions and all API operations are charged.

### Cost Components

| Component | Price |
|---|---|
| Active secret version (per month) | ~$0.06 per version |
| Secret access operations | $0.03 per 10,000 operations |
| Admin operations (create, list, describe, update) | Free |
| CMEK (Secret Manager) | No additional SM cost; KMS charges apply |

### Estimated Monthly Cost: Typical Production Setup

```
Scenario: 20 production services, each with 1 active secret + 1 previous version = 40 versions

Active Versions:
  40 versions × $0.06 = $2.40/month

API Operations:
  20 services × 10 instances × 1 fetch at startup × 2 restarts/day × 30 days
  = 12,000 access operations/month
  12,000 / 10,000 × $0.03 = $0.04/month

  Plus normal operation caching (one fetch per 5 min per instance):
  20 services × 10 instances × 12 fetches/hour × 24 hours × 30 days
  = 1,728,000 operations/month
  1,728,000 / 10,000 × $0.03 = $5.18/month

Total estimate: ~$7.62/month
```

> 💡 **The #1 cost driver is API operations.** Cache aggressively. A single fetch per process startup (or per TTL) instead of per request reduces operations by 99%+.

---

### 💰 Cost Optimization Tips

- **Cache in-process with TTL** — one SM call per startup/TTL instead of per request
- **Destroy old versions promptly** — disabled non-primary versions still cost $0.06/month
- **Use `version-destroy-ttl`** — auto-cleanup disabled versions without manual intervention
- **Batch secret fetches at startup** — retrieve all needed secrets in parallel at initialization
- **Avoid accessing `latest` in hot paths** — the `latest` resolution requires an extra metadata lookup; pin to a version number after fetching it once

---

## 14. Quick Reference & Comparison Tables

### Secret Manager vs. Alternatives

| Feature | **Secret Manager** | **Env Variables** | **GCS Object** | **HashiCorp Vault** | **AWS Secrets Manager** |
|---|---|---|---|---|---|
| Purpose-built storage | ✅ | ❌ | ❌ | ✅ | ✅ |
| Versioning | ✅ | ❌ | ❌ (object versions) | ✅ | ✅ |
| Automatic rotation | ✅ Pub/Sub | ❌ | ❌ | ✅ | ✅ Native |
| Audit logging | ✅ Cloud Audit | ❌ | Partial (GCS logs) | ✅ | ✅ CloudTrail |
| Per-resource IAM | ✅ Per-secret | ❌ | Per-bucket | ✅ Per-path | ✅ Per-secret |
| GCP native integration | ✅✅ | Manual | Partial | Plugin required | ❌ |
| Operational burden | None | None | Medium | High / $$$ HCP | None |
| CMEK support | ✅ | ❌ | ✅ | Depends on backend | AWS KMS only |
| Multi-region | ✅ (automatic) | N/A | ✅ | Manual | ✅ |

---

### IAM Roles Summary

| Role | Read Payload | Add Versions | Enable/Disable/Destroy | Create/Delete Secret | Set IAM |
|---|---|---|---|---|---|
| `secretmanager.admin` | ❌ | ✅ | ✅ | ✅ | ✅ |
| `secretmanager.secretAccessor` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `secretmanager.secretVersionAdder` | ❌ | ✅ | ❌ | ❌ | ❌ |
| `secretmanager.secretVersionManager` | ❌ | ✅ | ✅ | ❌ | ❌ |
| `secretmanager.viewer` | ❌ | ❌ | ❌ | ❌ | ❌ |

> 💡 `admin` cannot read payloads by default — this is intentional separation of duties. Assign `secretAccessor` separately to anyone who needs to read secret values.

---

### Secret Version State Machine

| State | Payload Accessible | Transitions To | Trigger |
|---|---|---|---|
| `ENABLED` | ✅ | DISABLED, DESTROYED | `disable`, `destroy` |
| `DISABLED` | ❌ | ENABLED, DESTROYED | `enable`, `destroy`, or `version-destroy-ttl` expires |
| `DESTROYED` | ❌ (permanent) | None | Irreversible |

---

### Replication Policy Decision Guide

| Requirement | Policy | Configuration |
|---|---|---|
| Highest availability, no data residency | `AUTOMATIC` | Default |
| GDPR — EU data only | `USER_MANAGED` | `europe-west1`, `europe-west3` |
| US government / FedRAMP | `USER_MANAGED` | `us-central1`, `us-east1` |
| Single-region compliance audit | `USER_MANAGED` | One region only |
| Per-region CMEK keys | `USER_MANAGED` | One KMS key per replica |
| Low latency in specific region | `USER_MANAGED` | Include target region |

---

### Rotation Patterns

| Pattern | Complexity | Zero-Downtime | Automation | Use Case |
|---|---|---|---|---|
| **Manual** | Low | ⚠️ If done carefully | None | One-off, low-frequency |
| **Automatic (Pub/Sub + Cloud Function)** | Medium | ✅ | Full | DB passwords, API keys |
| **Zero-downtime (overlap window)** | Medium | ✅ | Partial | Production, high availability |
| **Blue/Green (version aliases)** | High (preview) | ✅ | Full | Critical services, canary testing |

---

### Secret Manager Integrations

| GCP Service | Integration Method | Notes |
|---|---|---|
| Cloud Run | `--set-secrets` (env var or volume) | Native; recommended |
| GKE | Secrets Store CSI Driver | Volume mount + Kubernetes Secret sync |
| Cloud Functions Gen 2 | `--set-secrets` (env var) | Same as Cloud Run |
| Cloud Build | `availableSecrets` + `secretEnv` | Injected per build step |
| Compute Engine | SDK at startup | Service account must have IAM binding |
| App Engine | SDK in app code | Service account must have IAM binding |
| Config Connector | `SecretManagerSecret` CRD | GitOps workflow |
| Terraform | `data.google_secret_manager_secret_version` | ⚠️ Stored in state |

---

### Anti-Patterns and Fixes

| Anti-Pattern | Risk Level | Fix |
|---|---|---|
| Secrets in source code / `.env` files committed to git | 🔴 Critical | Use Secret Manager + `.gitignore` |
| Secrets in Dockerfile `ENV` instructions | 🔴 Critical | Inject at runtime via `--set-secrets` |
| `secretAccessor` at project level | 🔴 Critical | Bind at secret resource level |
| Accessing `latest` on every HTTP request | 🟡 Medium | Cache with TTL (5–60 min) |
| Not rotating credentials | 🔴 Critical | Set `rotation-period` + Pub/Sub function |
| Old versions never destroyed | 🟡 Medium | Use `version-destroy-ttl` or manual cleanup |
| Logging secret values | 🔴 Critical | Log actions, never values |
| Sharing secrets across environments | 🔴 Critical | Separate secret per environment |
| Service account key files as credentials | 🔴 Critical | Workload Identity Federation |
| Terraform state with plaintext secrets | 🟠 High | Encrypt state; prefer runtime injection |

---

*Generated: GCP Secret Manager Comprehensive Cheatsheet — covers Secret Manager API v1 | Rotation, CMEK, Replication, IAM, Integrations, Monitoring*
