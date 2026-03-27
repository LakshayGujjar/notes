# 🔐 GCP Cloud KMS Cheatsheet

> **Audience:** Security Engineers · Platform Engineers · DevOps Engineers · Architects  
> **Scope:** Cloud KMS, Cloud HSM, Cloud EKM — key management, encryption, signing, IAM, CMEK, compliance, monitoring

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Key Rings & Crypto Keys](#2-key-rings--crypto-keys)
3. [Encryption & Decryption (Symmetric)](#3-encryption--decryption-symmetric)
4. [Asymmetric Signing & Verification](#4-asymmetric-signing--verification)
5. [Asymmetric Encryption & Decryption](#5-asymmetric-encryption--decryption)
6. [MAC (Message Authentication Codes)](#6-mac-message-authentication-codes)
7. [Key Rotation](#7-key-rotation)
8. [Key Version Lifecycle & Destruction](#8-key-version-lifecycle--destruction)
9. [Key Import (BYOK)](#9-key-import-byok--bring-your-own-key)
10. [Cloud HSM](#10-cloud-hsm)
11. [Cloud External Key Manager (EKM)](#11-cloud-external-key-manager-ekm)
12. [IAM & Access Control for KMS](#12-iam--access-control-for-kms)
13. [Customer-Managed Encryption Keys (CMEK)](#13-customer-managed-encryption-keys-cmek-integration)
14. [Key Access Justifications](#14-key-access-justifications)
15. [Monitoring, Alerting & Audit Logging](#15-monitoring-alerting--audit-logging)
16. [Best Practices & Security Patterns](#16-best-practices--security-patterns)
17. [gcloud CLI & REST API Quick Reference](#17-gcloud-cli--rest-api-quick-reference)
18. [Pricing Summary](#18-pricing-summary)
19. [Quick Reference & Comparison Tables](#19-quick-reference--comparison-tables)

---

## 1. Overview & Key Concepts

### What is Cloud KMS?

Cloud KMS (Key Management Service) is Google Cloud's hosted key management service for creating, managing, rotating, and using cryptographic keys. It provides:

- **Centralized key management** across all GCP services and custom applications
- **Encryption at rest** via CMEK (Customer-Managed Encryption Keys)
- **Digital signing** for JWTs, container images, documents
- **FIPS 140-2 compliance** (Level 1 via software keys; Level 3 via Cloud HSM)
- **Audit trails** for every key operation via Cloud Audit Logs
- **Envelope encryption** to protect arbitrary amounts of data efficiently

---

### Cloud KMS vs. Alternatives

| | **Cloud KMS** | **Self-Managed Keys** | **HashiCorp Vault** | **AWS KMS** |
|---|---|---|---|---|
| **Operational burden** | Low (managed service) | High (your HSMs/servers) | Medium (self-hosted or HCP) | Low (managed service) |
| **GCP CMEK integration** | Native | ❌ | ❌ | ❌ |
| **FIPS 140-2 Level 3** | ✅ (Cloud HSM) | Depends | ✅ (with HSM backend) | ✅ |
| **Key material leaves GCP** | Never (except EKM) | You control | You control | Never |
| **Multi-cloud** | Limited | ✅ | ✅ (true multi-cloud) | AWS-native |
| **Cost model** | Per key version + per operation | CapEx / OpEx | License + infrastructure | Per key + per operation |
| **When to choose** | GCP-centric workloads, CMEK | Custom compliance needs | Multi-cloud or complex workflows | AWS workloads |

> 💡 **Choose Cloud KMS** when your workload is primarily GCP, you need CMEK for GCP services, or you require a managed, audit-logged key store with no operational overhead.

---

### Cloud KMS Product Family

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Cloud KMS Product Family                        │
│                                                                       │
│  ┌────────────────┐  ┌─────────────────┐  ┌──────────────────────┐  │
│  │   Cloud KMS    │  │   Cloud HSM     │  │  Cloud EKM           │  │
│  │  (Software)    │  │  (Hardware)     │  │  (External)          │  │
│  │                │  │                 │  │                      │  │
│  │ FIPS 140-2 L1  │  │ FIPS 140-2 L3  │  │ Key never enters GCP │  │
│  │ Lower cost     │  │ HSM attestation │  │ Sovereign cloud use  │  │
│  │ Higher throughput│ │ Higher latency  │  │ Partner KMS required │  │
│  └────────────────┘  └─────────────────┘  └──────────────────────┘  │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │               Confidential Computing                           │  │
│  │  Confidential VMs / GKE Nodes — hardware-level memory encryption│ │
│  │  Complements KMS but is a separate product (AMD SEV / Intel TDX)│ │
│  └────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Core Concepts

| Concept | Definition |
|---|---|
| **Key Ring** | Logical grouping of crypto keys. Regional resource. Cannot be deleted. |
| **Crypto Key** | Named key resource within a key ring. Defines purpose, algorithm, protection level, rotation period. |
| **Key Version** | Actual cryptographic key material (the secret bits). Multiple versions per key. |
| **Key State** | Lifecycle state of a key version (ENABLED, DISABLED, SCHEDULED_FOR_DESTRUCTION, DESTROYED). |
| **Key Material** | The underlying cryptographic bytes. Never directly accessible to you for software keys. |
| **Key Purpose** | What the key can do: ENCRYPT_DECRYPT, ASYMMETRIC_SIGN, ASYMMETRIC_DECRYPT, MAC. |
| **Key Algorithm** | The specific cryptographic algorithm (e.g., `GOOGLE_SYMMETRIC_ENCRYPTION`, `RSA_SIGN_PSS_4096_SHA512`). |
| **Protection Level** | Where key operations occur: SOFTWARE, HSM, EXTERNAL, EXTERNAL_VPC. |

---

### Key Hierarchy

```
GCP Project
└── Key Ring  (regional resource, e.g., us-central1/my-keyring)
    └── Crypto Key  (defines purpose, algorithm, protection level)
        ├── Key Version 1  (DESTROYED — old material)
        ├── Key Version 2  (ENABLED — primary, used for new encryptions)
        └── Key Version 3  (ENABLED — non-primary, used for decryption only)
```

- **Project** → billing, IAM boundary
- **Key Ring** → regional grouping, cannot be moved or deleted; IAM policies apply here
- **Crypto Key** → logical key; rotation policy lives here; IAM policies can override key ring
- **Key Version** → the actual secret bytes; each rotation creates a new version

---

### Key Purposes

| Purpose | Use Case | Algorithms |
|---|---|---|
| `ENCRYPT_DECRYPT` | Symmetric encryption (data at rest, CMEK) | AES-256-GCM (`GOOGLE_SYMMETRIC_ENCRYPTION`) |
| `ASYMMETRIC_SIGN` | Digital signatures: JWTs, code signing, container image signing | RSA-PSS, RSA-PKCS1, EC P-256, EC P-384 |
| `ASYMMETRIC_DECRYPT` | Wrapping symmetric keys, secure transport of small secrets | RSA-OAEP (2048/3072/4096) |
| `MAC` | Message authentication, API request signing, webhook verification | HMAC-SHA256, HMAC-SHA384, HMAC-SHA512 |

---

### Protection Levels

| Protection Level | Where Operations Run | FIPS 140-2 | Cost | Key Material Location |
|---|---|---|---|---|
| `SOFTWARE` | Google-managed software | Level 1 | $ | Google's infrastructure |
| `HSM` | Dedicated FIPS 140-2 Level 3 HSM | **Level 3** | $$$ | HSM hardware only |
| `EXTERNAL` | Your external KMS (internet) | Depends on partner | $$ + partner cost | Never enters GCP |
| `EXTERNAL_VPC` | Your external KMS (private VPC) | Depends on partner | $$ + partner cost | Never enters GCP |

> ⚠️ **HSM keys have higher latency** (~1–5 ms vs ~0.1 ms for SOFTWARE). Plan throughput accordingly.

---

### Key Version States

```
         Import/Create
              │
              ▼
    ┌──────────────────┐
    │ PENDING_GENERATION│  (HSM key generation in progress)
    │ PENDING_IMPORT    │  (BYOK: awaiting import)
    └──────────┬────────┘
               │
               ▼
         ┌──────────┐
    ┌───►│  ENABLED  │◄──┐  (actively usable: encrypt, sign, decrypt)
    │    └────┬──────┘   │
    │         │ Disable  │ Re-enable
    │         ▼          │
    │    ┌──────────┐    │
    │    │ DISABLED  │───┘  (key material intact; no crypto ops)
    │    └────┬──────┘
    │         │ Schedule destruction (min 24hr wait)
    │         ▼
    │  ┌──────────────────────┐
    │  │SCHEDULED_FOR_DESTRUCT│  (can be cancelled)
    │  └──────────┬───────────┘
    │             │ After scheduled time
    │             ▼
    │       ┌──────────┐
    └───────│ DESTROYED │  (key material gone; IRREVERSIBLE)
            └──────────┘
```

> 🚨 **DESTROYED is permanent.** Any data encrypted only with a destroyed key version is **permanently inaccessible**. There is no recovery path — not even by Google.

---

### Envelope Encryption Pattern

> 💡 Direct KMS encryption is limited to **64 KB**. All large-data encryption must use the envelope pattern.

```
ENCRYPTION
──────────────────────────────────────────────────────────
  1. Generate a random DEK (Data Encryption Key) locally
     e.g., 256-bit AES key via os.urandom(32)

  2. Encrypt your DATA with the DEK (locally, e.g., AES-GCM)
     → Produces: encrypted_data

  3. Encrypt the DEK with KMS KEK (Key Encryption Key)
     → KMS.Encrypt(plaintext=DEK) → encrypted_DEK

  4. Store:  { encrypted_DEK + encrypted_data }
     (discard the plaintext DEK immediately)

DECRYPTION
──────────────────────────────────────────────────────────
  1. Call KMS.Decrypt(ciphertext=encrypted_DEK)
     → Returns plaintext DEK

  2. Decrypt DATA locally using the DEK
     → Returns original plaintext

  Benefit: KMS only ever sees the small DEK (~32 bytes).
           Your data stays local. Only one KMS call needed.
```

---

### Key Rotation

- **Automatic rotation**: set `rotation_period` and `next_rotation_time` on a key. Cloud KMS creates a new primary version on schedule; old versions remain enabled for decryption.
- **Manual rotation**: use `gcloud kms keys versions create` + `set-primary-version`
- Old key versions are **never automatically destroyed** — they remain available for decryption until you explicitly destroy them.

> 💡 **Rotation does not re-encrypt existing data.** If you rotate, old DEKs remain decryptable via old key versions. Re-encryption is a separate operational step.

---

### Supported Algorithms

| Category | Algorithm Identifier | Key Size | Hash | Use Case |
|---|---|---|---|---|
| Symmetric | `GOOGLE_SYMMETRIC_ENCRYPTION` | 256-bit AES | GCM (AEAD) | CMEK, data encryption |
| RSA Sign PSS | `RSA_SIGN_PSS_2048_SHA256` | 2048 | SHA-256 | Signatures |
| RSA Sign PSS | `RSA_SIGN_PSS_4096_SHA512` | 4096 | SHA-512 | High-security signatures |
| RSA Sign PKCS1 | `RSA_SIGN_PKCS1_2048_SHA256` | 2048 | SHA-256 | Legacy/TLS compatibility |
| RSA Sign PKCS1 | `RSA_SIGN_PKCS1_4096_SHA512` | 4096 | SHA-512 | High-security signatures |
| EC Sign | `EC_SIGN_P256_SHA256` | P-256 | SHA-256 | JWTs, container signing |
| EC Sign | `EC_SIGN_P384_SHA384` | P-384 | SHA-384 | Higher assurance signing |
| RSA Decrypt OAEP | `RSA_DECRYPT_OAEP_2048_SHA256` | 2048 | SHA-256 | Key wrapping |
| RSA Decrypt OAEP | `RSA_DECRYPT_OAEP_4096_SHA512` | 4096 | SHA-512 | Key wrapping |
| HMAC | `HMAC_SHA256` | 256 | SHA-256 | MAC, webhook verification |
| HMAC | `HMAC_SHA512` | 512 | SHA-512 | High-security MAC |

---

### Key Limits & Quotas

| Resource | Default Limit |
|---|---|
| Key rings per project per location | 1,000 |
| Crypto keys per key ring | 1,000 |
| Key versions per crypto key | 200 |
| Crypto key operations (encrypt/decrypt) per second | 60,000 per project |
| Asymmetric sign/decrypt operations per second | 1,000 per project |
| Import jobs per project | 50 active at a time |

> 💡 Request quota increases via Cloud Console → IAM & Admin → Quotas.

---

## 2. Key Rings & Crypto Keys

### Key Ring Concepts

- **Regional resource**: key ring and all its keys live in one location. Location is **immutable**.
- **Cannot be deleted**: key rings are permanent. Plan naming carefully.
- **IAM policies**: set at key ring level apply to all keys within it.
- **Location matters** for data residency: always co-locate key rings with the data they protect.

### Key Ring Locations

| Location Type | Examples | Use Case | Trade-offs |
|---|---|---|---|
| Regional | `us-central1`, `europe-west1`, `asia-east1` | Data residency requirements | Single region; lower latency |
| Multi-regional | `us`, `europe`, `asia` | Redundancy within continent | No single-region guarantees |
| Global | `global` | Signing keys, global services | No data residency guarantees; avoid for sensitive data |

> ⚠️ **Never use `global` for CMEK keys** unless you have no data residency requirements. Prefer regional keys that match your data's region.

---

### Creating Key Rings

```bash
# Create a key ring
gcloud kms keyrings create prod-keyring \
  --location=us-central1

# List key rings
gcloud kms keyrings list --location=us-central1

# Describe a key ring
gcloud kms keyrings describe prod-keyring \
  --location=us-central1
```

---

### Creating Crypto Keys

```bash
# Symmetric encryption key (AES-256-GCM) — most common for CMEK
gcloud kms keys create bigquery-prod-cmek \
  --keyring=prod-keyring \
  --location=us-central1 \
  --purpose=encryption \
  --rotation-period=90d \
  --next-rotation-time=$(date -u -d "+1 day" +%Y-%m-%dT%H:%M:%SZ) \
  --labels=env=prod,team=platform

# Asymmetric signing key — EC P-256 (for JWTs, container signing)
gcloud kms keys create jwt-signing-key \
  --keyring=signing-keyring \
  --location=us-central1 \
  --purpose=asymmetric-signing \
  --default-algorithm=ec-sign-p256-sha256 \
  --protection-level=software

# Asymmetric signing key — RSA 4096 PSS (for document signing)
gcloud kms keys create doc-signing-key \
  --keyring=signing-keyring \
  --location=us-central1 \
  --purpose=asymmetric-signing \
  --default-algorithm=rsa-sign-pss-4096-sha512 \
  --protection-level=hsm

# Asymmetric decryption key — RSA OAEP (for wrapping secrets)
gcloud kms keys create key-wrapping-key \
  --keyring=prod-keyring \
  --location=us-central1 \
  --purpose=asymmetric-encryption \
  --default-algorithm=rsa-decrypt-oaep-4096-sha512

# MAC key — HMAC-SHA256 (for webhook verification)
gcloud kms keys create webhook-mac-key \
  --keyring=prod-keyring \
  --location=us-central1 \
  --purpose=mac \
  --default-algorithm=hmac-sha256

# Key management operations
gcloud kms keys list --keyring=prod-keyring --location=us-central1
gcloud kms keys describe bigquery-prod-cmek --keyring=prod-keyring --location=us-central1
gcloud kms keys update bigquery-prod-cmek \
  --keyring=prod-keyring --location=us-central1 \
  --rotation-period=30d
gcloud kms keys disable bigquery-prod-cmek --keyring=prod-keyring --location=us-central1 --version=1
gcloud kms keys enable bigquery-prod-cmek --keyring=prod-keyring --location=us-central1 --version=1
```

---

### Terraform: Key Rings & Crypto Keys

```hcl
# Key ring
resource "google_kms_key_ring" "prod_keyring" {
  name     = "prod-keyring"
  location = "us-central1"
  project  = var.project_id
}

# Symmetric key for CMEK (BigQuery, GCS, etc.)
resource "google_kms_crypto_key" "bigquery_cmek" {
  name            = "bigquery-prod-cmek"
  key_ring        = google_kms_key_ring.prod_keyring.id
  purpose         = "ENCRYPT_DECRYPT"
  rotation_period = "7776000s"  # 90 days

  version_template {
    algorithm        = "GOOGLE_SYMMETRIC_ENCRYPTION"
    protection_level = "SOFTWARE"
  }

  labels = {
    env     = "prod"
    service = "bigquery"
  }

  lifecycle {
    prevent_destroy = true  # Safety: never accidentally destroy production keys
  }
}

# Asymmetric signing key (EC P-256 for JWTs)
resource "google_kms_crypto_key" "jwt_signing" {
  name     = "jwt-signing-key"
  key_ring = google_kms_key_ring.prod_keyring.id
  purpose  = "ASYMMETRIC_SIGN"

  version_template {
    algorithm        = "EC_SIGN_P256_SHA256"
    protection_level = "HSM"
  }
}

# MAC key
resource "google_kms_crypto_key" "webhook_mac" {
  name     = "webhook-mac-key"
  key_ring = google_kms_key_ring.prod_keyring.id
  purpose  = "MAC"

  version_template {
    algorithm        = "HMAC_SHA256"
    protection_level = "SOFTWARE"
  }
}
```

---

## 3. Encryption & Decryption (Symmetric)

### How It Works

```
Client → KMS Encrypt(plaintext, key) → ciphertext (base64-encoded blob)
Client → KMS Decrypt(ciphertext, key) → plaintext

Max direct encrypt payload: 64 KB (65,536 bytes)
For larger data: use envelope encryption (see Section 1)
```

### Additional Authenticated Data (AAD)

AAD binds ciphertext to a specific context (e.g., a resource ID). Decryption **fails** if the AAD doesn't match.

```
Encrypt(plaintext, aad="projects/my-project/datasets/my-dataset")
Decrypt(ciphertext, aad="projects/my-project/datasets/my-dataset")  ← must match exactly
```

> 💡 **Use AAD** to prevent ciphertext from being moved or replayed in a different context. Example: bind a DEK to the GCS object path that stores its encrypted data.

---

### gcloud CLI Examples

```bash
# Encrypt a file (plaintext → ciphertext)
gcloud kms encrypt \
  --keyring=prod-keyring \
  --key=bigquery-prod-cmek \
  --location=us-central1 \
  --plaintext-file=secret.txt \
  --ciphertext-file=secret.enc

# Encrypt with AAD
gcloud kms encrypt \
  --keyring=prod-keyring \
  --key=bigquery-prod-cmek \
  --location=us-central1 \
  --plaintext-file=secret.txt \
  --ciphertext-file=secret.enc \
  --additional-authenticated-data="my-resource-id"

# Decrypt
gcloud kms decrypt \
  --keyring=prod-keyring \
  --key=bigquery-prod-cmek \
  --location=us-central1 \
  --ciphertext-file=secret.enc \
  --plaintext-file=decrypted.txt

# Decrypt with AAD (must match what was used during encryption)
gcloud kms decrypt \
  --keyring=prod-keyring \
  --key=bigquery-prod-cmek \
  --location=us-central1 \
  --ciphertext-file=secret.enc \
  --plaintext-file=decrypted.txt \
  --additional-authenticated-data="my-resource-id"
```

---

### Python: Direct Encrypt / Decrypt

```python
import base64
from google.cloud import kms

def encrypt_symmetric(project_id, location_id, key_ring_id, key_id, plaintext: bytes) -> bytes:
    client = kms.KeyManagementServiceClient()
    key_name = client.crypto_key_path(project_id, location_id, key_ring_id, key_id)

    response = client.encrypt(
        request={"name": key_name, "plaintext": plaintext}
    )
    return response.ciphertext  # raw bytes (not base64)


def decrypt_symmetric(project_id, location_id, key_ring_id, key_id, ciphertext: bytes) -> bytes:
    client = kms.KeyManagementServiceClient()
    key_name = client.crypto_key_path(project_id, location_id, key_ring_id, key_id)

    response = client.decrypt(
        request={"name": key_name, "ciphertext": ciphertext}
    )
    return response.plaintext


def encrypt_with_aad(project_id, location_id, key_ring_id, key_id,
                     plaintext: bytes, aad: bytes) -> bytes:
    client = kms.KeyManagementServiceClient()
    key_name = client.crypto_key_path(project_id, location_id, key_ring_id, key_id)

    response = client.encrypt(
        request={
            "name": key_name,
            "plaintext": plaintext,
            "additional_authenticated_data": aad,
        }
    )
    return response.ciphertext
```

---

### Python: Envelope Encryption

```python
import os
import base64
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
from google.cloud import kms

# ── ENCRYPTION ──────────────────────────────────────────────────────────────

def envelope_encrypt(project_id, location_id, key_ring_id, key_id,
                     plaintext: bytes) -> dict:
    """Encrypt data using envelope encryption with Cloud KMS."""
    # 1. Generate a random 256-bit DEK
    dek = os.urandom(32)

    # 2. Encrypt the data locally with the DEK (AES-256-GCM)
    aesgcm = AESGCM(dek)
    nonce = os.urandom(12)
    encrypted_data = nonce + aesgcm.encrypt(nonce, plaintext, None)

    # 3. Encrypt the DEK with KMS (KEK)
    kms_client = kms.KeyManagementServiceClient()
    key_name = kms_client.crypto_key_path(project_id, location_id, key_ring_id, key_id)
    response = kms_client.encrypt(request={"name": key_name, "plaintext": dek})
    encrypted_dek = response.ciphertext

    # 4. Return the bundle (store this together)
    return {
        "encrypted_dek": base64.b64encode(encrypted_dek).decode(),
        "encrypted_data": base64.b64encode(encrypted_data).decode(),
    }


# ── DECRYPTION ──────────────────────────────────────────────────────────────

def envelope_decrypt(project_id, location_id, key_ring_id, key_id,
                     bundle: dict) -> bytes:
    """Decrypt data using envelope decryption with Cloud KMS."""
    encrypted_dek = base64.b64decode(bundle["encrypted_dek"])
    encrypted_data = base64.b64decode(bundle["encrypted_data"])

    # 1. Decrypt the DEK with KMS
    kms_client = kms.KeyManagementServiceClient()
    key_name = kms_client.crypto_key_path(project_id, location_id, key_ring_id, key_id)
    response = kms_client.decrypt(
        request={"name": key_name, "ciphertext": encrypted_dek}
    )
    dek = response.plaintext

    # 2. Decrypt the data locally
    aesgcm = AESGCM(dek)
    nonce = encrypted_data[:12]
    ciphertext = encrypted_data[12:]
    return aesgcm.decrypt(nonce, ciphertext, None)
```

---

### Go: Encrypt and Decrypt

```go
import (
    "context"
    kms "cloud.google.com/go/kms/apiv1"
    kmspb "cloud.google.com/go/kms/apiv1/kmspb"
)

func encryptSymmetric(projectID, locationID, keyRingID, keyID string, plaintext []byte) ([]byte, error) {
    ctx := context.Background()
    client, _ := kms.NewKeyManagementClient(ctx)
    defer client.Close()

    keyName := fmt.Sprintf("projects/%s/locations/%s/keyRings/%s/cryptoKeys/%s",
        projectID, locationID, keyRingID, keyID)

    result, err := client.Encrypt(ctx, &kmspb.EncryptRequest{
        Name:      keyName,
        Plaintext: plaintext,
    })
    return result.Ciphertext, err
}

func decryptSymmetric(projectID, locationID, keyRingID, keyID string, ciphertext []byte) ([]byte, error) {
    ctx := context.Background()
    client, _ := kms.NewKeyManagementClient(ctx)
    defer client.Close()

    keyName := fmt.Sprintf("projects/%s/locations/%s/keyRings/%s/cryptoKeys/%s",
        projectID, locationID, keyRingID, keyID)

    result, err := client.Decrypt(ctx, &kmspb.DecryptRequest{
        Name:       keyName,
        Ciphertext: ciphertext,
    })
    return result.Plaintext, err
}
```

---

### REST API: Encrypt / Decrypt

```bash
# Encrypt via REST
curl -s -X POST \
  "https://cloudkms.googleapis.com/v1/projects/PROJECT_ID/locations/LOCATION/keyRings/KEYRING/cryptoKeys/KEY:encrypt" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "plaintext": "'"$(echo -n 'my secret' | base64)"'"
  }'

# Decrypt via REST (ciphertext comes from encrypt response)
curl -s -X POST \
  "https://cloudkms.googleapis.com/v1/projects/PROJECT_ID/locations/LOCATION/keyRings/KEYRING/cryptoKeys/KEY:decrypt" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "ciphertext": "CIPHERTEXT_FROM_ENCRYPT_RESPONSE"
  }'
```

---

### CMEK Integration Snippets

```bash
# GCS: Set default CMEK key on a bucket
gsutil kms authorize -k projects/PROJECT/locations/LOCATION/keyRings/RING/cryptoKeys/KEY -p PROJECT
gsutil kms encryption -k projects/PROJECT/locations/LOCATION/keyRings/RING/cryptoKeys/KEY gs://my-bucket

# BigQuery: Create dataset with CMEK
bq mk \
  --dataset \
  --default_kms_key=projects/PROJECT/locations/LOCATION/keyRings/RING/cryptoKeys/KEY \
  my_project:my_dataset

# GCE: Create a persistent disk with CMEK
gcloud compute disks create my-disk \
  --zone=us-central1-a \
  --kms-key=projects/PROJECT/locations/LOCATION/keyRings/RING/cryptoKeys/KEY

# Cloud SQL: Create instance with CMEK
gcloud sql instances create my-sql-instance \
  --region=us-central1 \
  --disk-encryption-key=projects/PROJECT/locations/LOCATION/keyRings/RING/cryptoKeys/KEY
```

---

## 4. Asymmetric Signing & Verification

### How It Works

```
KMS holds the PRIVATE key → never exported
You get the PUBLIC key → verify offline (no KMS call needed)

Sign:   KMS.AsymmetricSign(hash_of_message) → signature
Verify: pubkey.verify(message, signature)   → local operation
```

---

### Signing Algorithms Reference

| Algorithm | Key Type | Hash | Typical Use Case |
|---|---|---|---|
| `RSA_SIGN_PSS_2048_SHA256` | RSA 2048 | SHA-256 | Standard document signing |
| `RSA_SIGN_PSS_4096_SHA512` | RSA 4096 | SHA-512 | High-security signing |
| `RSA_SIGN_PKCS1_2048_SHA256` | RSA 2048 | SHA-256 | Legacy/TLS compatibility |
| `RSA_SIGN_PKCS1_4096_SHA512` | RSA 4096 | SHA-512 | High-security legacy |
| `EC_SIGN_P256_SHA256` | EC P-256 | SHA-256 | JWTs, container image signing |
| `EC_SIGN_P384_SHA384` | EC P-384 | SHA-384 | Higher assurance signing |

---

### gcloud: Sign and Get Public Key

```bash
# Get the public key (PEM format)
gcloud kms keys versions get-public-key 1 \
  --key=jwt-signing-key \
  --keyring=signing-keyring \
  --location=us-central1 \
  --output-file=public_key.pem

# Sign a file
gcloud kms asymmetric-sign \
  --key=jwt-signing-key \
  --keyring=signing-keyring \
  --location=us-central1 \
  --version=1 \
  --algorithm=ec-sign-p256-sha256 \
  --input-file=message.txt \
  --signature-file=message.sig
```

---

### Python: Sign with EC / RSA

```python
import hashlib
from google.cloud import kms


def sign_asymmetric_ec(project_id, location_id, key_ring_id, key_id,
                       version_id: str, message: bytes) -> bytes:
    """Sign a message using an EC key in Cloud KMS."""
    client = kms.KeyManagementServiceClient()
    version_name = client.crypto_key_version_path(
        project_id, location_id, key_ring_id, key_id, version_id
    )

    # Hash the message first (KMS signs the hash, not the raw message)
    digest = hashlib.sha256(message).digest()

    response = client.asymmetric_sign(
        request={
            "name": version_name,
            "digest": {"sha256": digest},
        }
    )
    return response.signature  # DER-encoded signature bytes


def sign_asymmetric_rsa_pss(project_id, location_id, key_ring_id, key_id,
                             version_id: str, message: bytes) -> bytes:
    """Sign a message using an RSA-PSS key in Cloud KMS."""
    client = kms.KeyManagementServiceClient()
    version_name = client.crypto_key_version_path(
        project_id, location_id, key_ring_id, key_id, version_id
    )

    digest = hashlib.sha512(message).digest()

    response = client.asymmetric_sign(
        request={
            "name": version_name,
            "digest": {"sha512": digest},
        }
    )
    return response.signature
```

---

### Python: Get Public Key and Verify

```python
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import ec, padding
from cryptography.hazmat.primitives.asymmetric.utils import decode_dss_signature
from google.cloud import kms
import functools


@functools.lru_cache(maxsize=32)
def get_public_key(project_id, location_id, key_ring_id, key_id, version_id: str):
    """Fetch and cache the public key from Cloud KMS."""
    client = kms.KeyManagementServiceClient()
    version_name = client.crypto_key_version_path(
        project_id, location_id, key_ring_id, key_id, version_id
    )
    response = client.get_public_key(request={"name": version_name})
    return serialization.load_pem_public_key(response.pem.encode())


def verify_signature_ec(public_key, message: bytes, signature: bytes) -> bool:
    """Verify an EC signature offline (no KMS API call)."""
    try:
        public_key.verify(
            signature,
            message,
            ec.ECDSA(hashes.SHA256())
        )
        return True
    except Exception:
        return False


def verify_signature_rsa_pss(public_key, message: bytes, signature: bytes) -> bool:
    """Verify an RSA-PSS signature offline."""
    try:
        public_key.verify(
            signature,
            message,
            padding.PSS(
                mgf=padding.MGF1(hashes.SHA512()),
                salt_length=padding.PSS.MAX_LENGTH,
            ),
            hashes.SHA512()
        )
        return True
    except Exception:
        return False
```

> 💡 **Cache the public key.** It only changes when you rotate the signing key. Use `lru_cache` or a short-lived in-memory cache to avoid repeated KMS API calls for signature verification.

---

### Common Use Cases

| Use Case | Algorithm | Notes |
|---|---|---|
| **JWT signing** (RS256) | `RSA_SIGN_PKCS1_2048_SHA256` | Compatible with standard JWT libraries |
| **JWT signing** (ES256) | `EC_SIGN_P256_SHA256` | Smaller signatures; preferred for modern systems |
| **Container image signing** | `EC_SIGN_P256_SHA256` | Used with Binary Authorization |
| **Document signing** | `RSA_SIGN_PSS_4096_SHA512` | Higher security margin |
| **Code signing** | `EC_SIGN_P384_SHA384` | Common for software distribution |

---

## 5. Asymmetric Encryption & Decryption

### How It Works

```
PUBLIC key:  anyone can use it to ENCRYPT (no KMS call needed)
PRIVATE key: only KMS can use it to DECRYPT (KMS.AsymmetricDecrypt)

Use case: securely deliver a symmetric key to a recipient who has an RSA public key
```

---

### Python: Encrypt with Public Key / Decrypt with KMS

```python
from cryptography.hazmat.primitives.asymmetric import padding as asym_padding
from cryptography.hazmat.primitives import hashes, serialization
from google.cloud import kms


def encrypt_with_public_key(pem_public_key: str, plaintext: bytes) -> bytes:
    """Encrypt data locally using the KMS public key — no KMS call required."""
    public_key = serialization.load_pem_public_key(pem_public_key.encode())
    return public_key.encrypt(
        plaintext,
        asym_padding.OAEP(
            mgf=asym_padding.MGF1(algorithm=hashes.SHA256()),
            algorithm=hashes.SHA256(),
            label=None,
        )
    )


def decrypt_asymmetric(project_id, location_id, key_ring_id, key_id,
                       version_id: str, ciphertext: bytes) -> bytes:
    """Decrypt using the KMS private key."""
    client = kms.KeyManagementServiceClient()
    version_name = client.crypto_key_version_path(
        project_id, location_id, key_ring_id, key_id, version_id
    )

    response = client.asymmetric_decrypt(
        request={"name": version_name, "ciphertext": ciphertext}
    )
    return response.plaintext
```

---

### gcloud: Asymmetric Decrypt

```bash
# Get the public key and encrypt data locally
gcloud kms keys versions get-public-key 1 \
  --key=key-wrapping-key --keyring=prod-keyring \
  --location=us-central1 --output-file=pub.pem

# Encrypt locally with openssl (RSA-OAEP)
openssl pkeyutl -encrypt \
  -inkey pub.pem -pubin \
  -pkeyopt rsa_padding_mode:oaep \
  -pkeyopt rsa_oaep_md:sha256 \
  -in secret_dek.bin -out encrypted_dek.bin

# Decrypt with KMS
gcloud kms asymmetric-decrypt \
  --key=key-wrapping-key --keyring=prod-keyring \
  --location=us-central1 --version=1 \
  --ciphertext-file=encrypted_dek.bin \
  --plaintext-file=decrypted_dek.bin
```

---

### RSA OAEP Size Limits

| Key Size | Max Plaintext (SHA-256 OAEP) | Max Plaintext (SHA-512 OAEP) |
|---|---|---|
| RSA-2048 | 190 bytes | 126 bytes |
| RSA-3072 | 318 bytes | 254 bytes |
| RSA-4096 | 446 bytes | 382 bytes |

> ⚠️ **Only suitable for small secrets** (e.g., 32-byte AES keys). Do not use for large data — use envelope encryption instead.

---

## 6. MAC (Message Authentication Codes)

### What MAC Is

A MAC (HMAC) proves both **integrity** and **authenticity** of a message — it ensures the message was not tampered with and originated from a party with access to the MAC key.

```
MacSign(key, message) → mac_tag
MacVerify(key, message, mac_tag) → VALID / INVALID
```

---

### Python: Generate and Verify MAC

```python
import base64
from google.cloud import kms


def generate_mac(project_id, location_id, key_ring_id, key_id,
                 version_id: str, data: bytes) -> bytes:
    client = kms.KeyManagementServiceClient()
    version_name = client.crypto_key_version_path(
        project_id, location_id, key_ring_id, key_id, version_id
    )
    response = client.mac_sign(
        request={"name": version_name, "data": data}
    )
    return response.mac


def verify_mac(project_id, location_id, key_ring_id, key_id,
               version_id: str, data: bytes, mac: bytes) -> bool:
    client = kms.KeyManagementServiceClient()
    version_name = client.crypto_key_version_path(
        project_id, location_id, key_ring_id, key_id, version_id
    )
    response = client.mac_verify(
        request={"name": version_name, "data": data, "mac": mac}
    )
    return response.success
```

---

### gcloud: MAC Sign and Verify

```bash
# Generate a MAC tag
gcloud kms mac-sign \
  --key=webhook-mac-key --keyring=prod-keyring \
  --location=us-central1 --version=1 \
  --input-file=payload.json \
  --signature-file=payload.mac

# Verify a MAC tag
gcloud kms mac-verify \
  --key=webhook-mac-key --keyring=prod-keyring \
  --location=us-central1 --version=1 \
  --input-file=payload.json \
  --signature-file=payload.mac
```

---

### MAC vs. Asymmetric Signing

| | **MAC (HMAC)** | **Asymmetric Signing** |
|---|---|---|
| **Key type** | Symmetric (shared secret) | Asymmetric (public/private) |
| **Who can verify** | Only parties with the MAC key | Anyone with the public key |
| **Performance** | Faster | Slower (RSA) or comparable (EC) |
| **Use case** | Internal service-to-service, webhooks | Public verification, PKI, JWTs |
| **Non-repudiation** | ❌ (shared secret) | ✅ (only KMS holds private key) |

---

## 7. Key Rotation

### Why Rotate?

Key rotation limits the cryptographic exposure window. If a DEK encrypted with an old KEK is somehow compromised, newer data (encrypted with a newer DEK under a new KEK version) is not affected.

---

### Automatic Rotation

```bash
# Set rotation period on an existing key
gcloud kms keys update my-key \
  --keyring=prod-keyring \
  --location=us-central1 \
  --rotation-period=90d \
  --next-rotation-time=2024-04-01T00:00:00Z

# Create a key with rotation from the start
gcloud kms keys create my-key \
  --keyring=prod-keyring --location=us-central1 \
  --purpose=encryption \
  --rotation-period=7776000s  # 90 days in seconds
```

> 💡 After automatic rotation, the **new version becomes primary** (used for new encryptions). Old versions remain ENABLED and are used automatically for decryption of old ciphertexts — no action required by you.

---

### Manual Rotation

```bash
# Create a new key version
gcloud kms keys versions create \
  --key=my-key \
  --keyring=prod-keyring \
  --location=us-central1

# List versions to find the new version number
gcloud kms keys versions list \
  --key=my-key \
  --keyring=prod-keyring \
  --location=us-central1

# Promote to primary
gcloud kms keys set-primary-version my-key \
  --keyring=prod-keyring \
  --location=us-central1 \
  --version=3
```

---

### Python: Trigger Manual Rotation

```python
from google.cloud import kms


def rotate_key_manual(project_id, location_id, key_ring_id, key_id) -> str:
    """Create a new key version and set it as primary."""
    client = kms.KeyManagementServiceClient()
    key_name = client.crypto_key_path(project_id, location_id, key_ring_id, key_id)

    # Create new version
    new_version = client.create_crypto_key_version(
        request={"parent": key_name, "crypto_key_version": {}}
    )
    print(f"Created: {new_version.name}")

    # Update primary version
    from google.protobuf import field_mask_pb2
    updated_key = client.update_crypto_key_primary_version(
        request={
            "name": key_name,
            "crypto_key_version_id": new_version.name.split("/")[-1],
        }
    )
    return updated_key.primary.name
```

---

### Rotation Best Practices

| Sensitivity | Recommended Rotation Period |
|---|---|
| High-sensitivity (PCI, HIPAA) | 30–90 days |
| Standard production workloads | 90 days |
| Low-sensitivity / dev environments | 1 year |
| Signing keys (JWTs, code) | 1 year (with key version pinning) |

> ⚠️ **Rotation ≠ Re-encryption.** Old data remains encrypted under old key versions. To fully re-encrypt, you must explicitly decrypt with old version and re-encrypt with new version. For CMEK resources, contact the GCP service — some support re-keying.

---

## 8. Key Version Lifecycle & Destruction

### State Transitions and Commands

```bash
# Disable a key version (key material intact; crypto ops blocked)
gcloud kms keys versions disable VERSION \
  --key=my-key --keyring=prod-keyring --location=us-central1

# Re-enable a disabled key version
gcloud kms keys versions enable VERSION \
  --key=my-key --keyring=prod-keyring --location=us-central1

# Schedule a version for destruction (minimum 24-hour wait)
gcloud kms keys versions schedule-destroy VERSION \
  --key=my-key --keyring=prod-keyring --location=us-central1

# Cancel scheduled destruction (before the timer expires)
gcloud kms keys versions cancel-scheduled-destroy VERSION \
  --key=my-key --keyring=prod-keyring --location=us-central1

# List versions with state
gcloud kms keys versions list \
  --key=my-key --keyring=prod-keyring --location=us-central1 \
  --format="table(name, state, createTime, destroyScheduledTime)"
```

---

### Python: Key Version Management

```python
from google.cloud import kms


def disable_key_version(project_id, location_id, key_ring_id, key_id, version_id: str):
    client = kms.KeyManagementServiceClient()
    version_name = client.crypto_key_version_path(
        project_id, location_id, key_ring_id, key_id, version_id
    )
    version = client.get_crypto_key_version(request={"name": version_name})
    version.state = kms.CryptoKeyVersion.CryptoKeyVersionState.DISABLED

    from google.protobuf import field_mask_pb2
    update_mask = field_mask_pb2.FieldMask(paths=["state"])
    return client.update_crypto_key_version(
        request={"crypto_key_version": version, "update_mask": update_mask}
    )


def schedule_destroy(project_id, location_id, key_ring_id, key_id, version_id: str):
    client = kms.KeyManagementServiceClient()
    version_name = client.crypto_key_version_path(
        project_id, location_id, key_ring_id, key_id, version_id
    )
    return client.destroy_crypto_key_version(request={"name": version_name})


def list_versions_with_state(project_id, location_id, key_ring_id, key_id):
    client = kms.KeyManagementServiceClient()
    key_name = client.crypto_key_path(project_id, location_id, key_ring_id, key_id)
    return list(client.list_crypto_key_versions(request={"parent": key_name}))
```

---

### 🚨 Key Destruction: Critical Points

- **24-hour default waiting period** before destruction becomes permanent (configurable up to 120 days via `--destroy-scheduled-duration`)
- **Permanent data loss**: data encrypted only with a destroyed key version is **unrecoverable**. Google cannot help.
- **CMEK impact**: if a CMEK key version is destroyed, GCP services using it lose access to their data permanently.
- **Best practice**: use `prevent_destroy = true` in Terraform for production keys; use IAM Deny policies to restrict `cloudkms.cryptoKeyVersions.destroy`.

---

## 9. Key Import (BYOK — Bring Your Own Key)

### Why BYOK?

- Regulatory requirements mandating proof of key provenance
- Existing key infrastructure or HSMs you want to leverage
- Controlled key generation audits

> ⚠️ **BYOK transfers responsibility.** If you import a key, you are responsible for the key's generation quality and prior handling. Cloud HSM attestation only applies to operations after import.

---

### Import Job States

| State | Meaning |
|---|---|
| `PENDING_GENERATION` | Import job's wrapping key is being generated |
| `ACTIVE` | Import job is ready; wrapping key available (3-day window) |
| `EXPIRED` | Import job expired; create a new one |

---

### Step-by-Step BYOK Workflow

```bash
# Step 1: Create an import job
gcloud kms import-jobs create my-import-job \
  --location=us-central1 \
  --keyring=prod-keyring \
  --import-method=rsa-oaep-3072-sha256-aes-256 \
  --protection-level=hsm

# Step 2: Wait for ACTIVE state and download the wrapping key
gcloud kms import-jobs describe my-import-job \
  --location=us-central1 \
  --keyring=prod-keyring

# Download the public wrapping key (PEM)
gcloud kms import-jobs describe my-import-job \
  --location=us-central1 \
  --keyring=prod-keyring \
  --format="value(publicKey.pem)" > wrapping_key.pem

# Step 3: Generate your key material locally
openssl rand -out my_aes_key.bin 32  # 256-bit AES key

# Step 4: Wrap the key material (RSA-OAEP + AES-KWP double wrap)
# 4a. Generate a temporary AES wrapping key
openssl rand -out temp_aes_wrap.bin 32

# 4b. Wrap your key with AES-KWP
openssl enc -id-aes256-wrap-pad -K $(xxd -p -c 256 temp_aes_wrap.bin) \
  -iv A65959A6 -in my_aes_key.bin -out wrapped_target_key.bin

# 4c. Wrap the AES key with the RSA-OAEP public key
openssl pkeyutl -encrypt \
  -inkey wrapping_key.pem -pubin \
  -pkeyopt rsa_padding_mode:oaep \
  -pkeyopt rsa_oaep_md:sha256 \
  -in temp_aes_wrap.bin -out wrapped_aes_key.bin

# 4d. Concatenate: wrapped_aes_key || wrapped_target_key
cat wrapped_aes_key.bin wrapped_target_key.bin > final_wrapped_key.bin

# Step 5: Import the wrapped key
gcloud kms keys versions import \
  --location=us-central1 \
  --keyring=prod-keyring \
  --key=my-imported-key \
  --import-job=my-import-job \
  --algorithm=google-symmetric-encryption \
  --wrapped-key-file=final_wrapped_key.bin
```

---

### gcloud: Import Job Management

```bash
# List import jobs
gcloud kms import-jobs list \
  --location=us-central1 \
  --keyring=prod-keyring

# Describe import job (check state)
gcloud kms import-jobs describe my-import-job \
  --location=us-central1 \
  --keyring=prod-keyring
```

---

## 10. Cloud HSM

### Cloud KMS vs. Cloud HSM

| Feature | Cloud KMS (SOFTWARE) | Cloud HSM |
|---|---|---|
| **Protection level** | SOFTWARE | HSM |
| **FIPS 140-2** | Level 1 | **Level 3** |
| **Key operation location** | Google servers | Dedicated HSM hardware |
| **HSM attestation** | ❌ | ✅ |
| **Latency** | ~0.1–0.5 ms | ~1–5 ms |
| **Cost** | $ | $$$ (~3–4x SOFTWARE cost) |
| **Supported algorithms** | Full | Subset (not all MAC/HMAC) |
| **Use cases** | General workloads | PCI-DSS, FIPS requirements, government |

---

### Creating HSM Keys

```bash
# Create an HSM-protected symmetric key
gcloud kms keys create pci-cmek-key \
  --keyring=pci-keyring \
  --location=us-central1 \
  --purpose=encryption \
  --protection-level=hsm \
  --rotation-period=90d

# Create an HSM-protected asymmetric signing key
gcloud kms keys create pci-signing-key \
  --keyring=pci-keyring \
  --location=us-central1 \
  --purpose=asymmetric-signing \
  --default-algorithm=ec-sign-p256-sha256 \
  --protection-level=hsm
```

---

### HSM Attestation

Attestation provides cryptographic proof that a key operation occurred inside a FIPS 140-2 Level 3 HSM.

```bash
# Get the attestation for a key version
gcloud kms keys versions describe 1 \
  --key=pci-cmek-key \
  --keyring=pci-keyring \
  --location=us-central1 \
  --format=json | jq .attestation

# Download the attestation bundle
gcloud kms keys versions get-certificate-chain 1 \
  --key=pci-cmek-key \
  --keyring=pci-keyring \
  --location=us-central1 \
  --output-file=attestation_bundle.json
```

> 💡 **Verify attestation chain** to prove key operations occurred in Google's HSMs. Google provides a verification tool and publishes HSM CA certificates at https://cloud.google.com/kms/docs/attest-key.

---

## 11. Cloud External Key Manager (EKM)

### Architecture

```
GCP Service (e.g., BigQuery)
      │ CMEK operation
      ▼
Cloud KMS (as proxy)
      │ Forward key operation
      ▼
External Key Manager (Thales, Fortanix, Futurex, etc.)
      │
      ▼
Key material NEVER leaves external KMS
```

**EKM vs. BYOK:**
- **BYOK**: key material is imported INTO Cloud KMS/HSM. Google's infrastructure holds the key.
- **EKM**: key material NEVER enters GCP. Every crypto operation is proxied to the external KMS.

---

### Protection Levels

| Level | Connectivity | Use Case |
|---|---|---|
| `EXTERNAL` | Public internet to partner KMS | Simpler setup; internet connectivity |
| `EXTERNAL_VPC` | Private VPC peering to partner KMS | Regulatory isolation; no internet path |

> ⚠️ **EKM availability = GCP service availability.** If your external KMS is down, GCP services using EKM-backed CMEK keys cannot perform key operations (new encryptions fail; existing data may become temporarily inaccessible).

---

### EKM Connection Management

```bash
# Create an EKM connection (EXTERNAL_VPC)
gcloud kms ekm-connections create my-ekm-connection \
  --location=us-central1 \
  --service-resolvers=hostname=ekm.example.com,serverCertificates=server_cert.pem,\
uris=https://ekm.example.com/v0/cryptoKeys/my-key \
  --key-management-mode=manual \
  --etag=""

# List EKM connections
gcloud kms ekm-connections list --location=us-central1

# Describe an EKM connection
gcloud kms ekm-connections describe my-ekm-connection --location=us-central1

# Create a KMS key backed by EKM
gcloud kms keys create my-ekm-key \
  --keyring=prod-keyring \
  --location=us-central1 \
  --purpose=encryption \
  --protection-level=external \
  --ekm-connection=my-ekm-connection \
  --external-key-uri=https://ekm.example.com/v0/cryptoKeys/my-key
```

---

## 12. IAM & Access Control for KMS

### IAM Roles Reference

| Role | Permissions | Who Needs It |
|---|---|---|
| `roles/cloudkms.admin` | Create/delete keys, manage key rings, set IAM | KMS platform team |
| `roles/cloudkms.cryptoKeyEncrypterDecrypter` | Encrypt + Decrypt | GCP service SAs for CMEK; app encrypt/decrypt |
| `roles/cloudkms.cryptoKeyEncrypter` | Encrypt only | Write-only services |
| `roles/cloudkms.cryptoKeyDecrypter` | Decrypt only | Read-only/restore services |
| `roles/cloudkms.signerVerifier` | AsymmetricSign + GetPublicKey + Verify | Signing services |
| `roles/cloudkms.publicKeyViewer` | GetPublicKey only | Services that only verify signatures |
| `roles/cloudkms.viewer` | Read key metadata, no crypto ops | Auditors, monitoring |
| `roles/cloudkms.importer` | Create import jobs, import key versions | Key import pipelines |

---

### Resource-Level IAM

IAM can be bound at three levels (most specific wins for deny; union applies for allow):

```
Project  →  Key Ring  →  Crypto Key  →  Key Version (limited)
```

**Best practice**: Grant `cryptoKeyEncrypterDecrypter` at the **crypto key level**, not the project or key ring level.

---

### gcloud: IAM Management

```bash
# Grant encrypter/decrypter on a specific key (CMEK service SA)
gcloud kms keys add-iam-policy-binding my-key \
  --keyring=prod-keyring \
  --location=us-central1 \
  --member="serviceAccount:bq-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Grant at key ring level (applies to all keys in ring)
gcloud kms keyrings add-iam-policy-binding prod-keyring \
  --location=us-central1 \
  --member="serviceAccount:my-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/cloudkms.viewer"

# View IAM policy on a key
gcloud kms keys get-iam-policy my-key \
  --keyring=prod-keyring \
  --location=us-central1

# Remove a binding
gcloud kms keys remove-iam-policy-binding my-key \
  --keyring=prod-keyring \
  --location=us-central1 \
  --member="serviceAccount:bq-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"
```

---

### Python: IAM Policy Management

```python
from google.cloud import kms
from google.iam.v1 import iam_policy_pb2, policy_pb2


def add_encrypter_decrypter(project_id, location_id, key_ring_id, key_id,
                             member: str):
    """Grant cryptoKeyEncrypterDecrypter on a specific key."""
    client = kms.KeyManagementServiceClient()
    key_name = client.crypto_key_path(project_id, location_id, key_ring_id, key_id)

    policy = client.get_iam_policy(request={"resource": key_name})
    policy.bindings.add(
        role="roles/cloudkms.cryptoKeyEncrypterDecrypter",
        members=[member]
    )
    return client.set_iam_policy(
        request={"resource": key_name, "policy": policy}
    )
```

---

### Separation of Duties

```
┌──────────────────┐     ┌─────────────────────────────┐
│   KMS Admin      │     │   Encrypter/Decrypter        │
│                  │     │                              │
│ • Create keys    │     │ • Encrypt data               │
│ • Rotate keys    │     │ • Decrypt data               │
│ • Delete keys    │     │ • Cannot manage keys         │
│ • Set IAM        │     │ • Cannot view IAM            │
│ Cannot encrypt   │     │ Cannot rotate/delete keys    │
└──────────────────┘     └─────────────────────────────┘
```

> 🛡️ **Never assign both `cloudkms.admin` and `cloudkms.cryptoKeyEncrypterDecrypter` to the same principal.** This would allow a compromised account to both access data AND manipulate key lifecycle.

---

### VPC Service Controls

Restrict Cloud KMS to trusted VPC networks to prevent exfiltration:

```bash
# Create a service perimeter including Cloud KMS
gcloud access-context-manager perimeters create my-perimeter \
  --policy=POLICY_ID \
  --title="KMS Perimeter" \
  --resources=projects/PROJECT_NUMBER \
  --restricted-services=cloudkms.googleapis.com \
  --access-levels=accessPolicies/POLICY/accessLevels/my-level
```

---

### Audit Logging

| Log Type | Operations Captured | Enable By Default |
|---|---|---|
| **Admin Activity** | Key creation, rotation, IAM changes, key destruction | Always on |
| **Data Access** | Encrypt, Decrypt, Sign, MacSign, GetPublicKey | Must be enabled |

> ⚠️ **Enable Data Access logs** for Cloud KMS in production. Without them, you cannot audit who encrypted or decrypted data.

```bash
# Enable Data Access audit logging for Cloud KMS
gcloud projects get-iam-policy PROJECT_ID --format=json > policy.json
# Add to auditConfigs: { service: "cloudkms.googleapis.com", auditLogConfigs: [DATA_READ, DATA_WRITE] }
gcloud projects set-iam-policy PROJECT_ID policy.json
```

---

## 13. Customer-Managed Encryption Keys (CMEK) Integration

### CMEK vs. Other Key Types

| Key Type | Who Manages | Control Level | Use Case |
|---|---|---|---|
| **Google-managed** | Google | None | Default; most services |
| **CMEK** (Cloud KMS) | You + Cloud KMS | High | Regulated workloads; key rotation control |
| **CSEK** (Customer-supplied) | You entirely | Maximum | Specific GCS/Compute use cases; you provide key bytes |

---

### CMEK-Supported GCP Services

| Service | Granularity | Grant to |
|---|---|---|
| BigQuery | Dataset, Table | `bq-PROJECT_NUMBER@bigquery-encryption.iam.gserviceaccount.com` |
| Cloud Storage | Bucket | `service-PROJECT_NUMBER@gs-project-accounts.iam.gserviceaccount.com` |
| Compute Engine (PD) | Disk, Image, Snapshot | `service-PROJECT_NUMBER@compute-system.iam.gserviceaccount.com` |
| Cloud SQL | Instance | `service-PROJECT_NUMBER@gcp-sa-cloud-sql.iam.gserviceaccount.com` |
| GKE (etcd) | Cluster (app-layer) | `service-PROJECT_NUMBER@container-engine-robot.iam.gserviceaccount.com` |
| Pub/Sub | Topic, Subscription | `service-PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com` |
| Dataflow | Job | `service-PROJECT_NUMBER@dataflow-service-producer-prod.iam.gserviceaccount.com` |
| Cloud Spanner | Instance, Database | `service-PROJECT_NUMBER@gcp-sa-spanner.iam.gserviceaccount.com` |
| Vertex AI | Dataset, Model | `service-PROJECT_NUMBER@gcp-sa-aiplatform.iam.gserviceaccount.com` |
| Secret Manager | Secret | `service-PROJECT_NUMBER@gcp-sa-secretmanager.iam.gserviceaccount.com` |
| Cloud Run | Service | `service-PROJECT_NUMBER@serverless-robot-prod.iam.gserviceaccount.com` |

---

### CMEK Setup Pattern

```bash
# Step 1: Create a KMS key
gcloud kms keys create SERVICE-env-cmek \
  --keyring=prod-keyring --location=LOCATION \
  --purpose=encryption --rotation-period=90d

# Step 2: Get the service's encryption service account
# (Pattern varies by service — see table above)
SERVICE_SA="service-$(gcloud projects describe PROJECT_ID --format='value(projectNumber)')@gs-project-accounts.iam.gserviceaccount.com"

# Step 3: Grant the SA cryptoKeyEncrypterDecrypter
gcloud kms keys add-iam-policy-binding SERVICE-env-cmek \
  --keyring=prod-keyring --location=LOCATION \
  --member="serviceAccount:${SERVICE_SA}" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Step 4: Create the resource with CMEK (service-specific — see below)
```

---

### BigQuery CMEK

```bash
# Grant BigQuery SA access to the KMS key
BQ_SA="bq-$(gcloud projects describe PROJECT_ID --format='value(projectNumber)')@bigquery-encryption.iam.gserviceaccount.com"
gcloud kms keys add-iam-policy-binding bigquery-prod-cmek \
  --keyring=prod-keyring --location=us-central1 \
  --member="serviceAccount:${BQ_SA}" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Create dataset with CMEK
bq --location=US mk \
  --dataset \
  --default_kms_key="projects/PROJECT/locations/us-central1/keyRings/prod-keyring/cryptoKeys/bigquery-prod-cmek" \
  PROJECT:my_secure_dataset
```

```python
from google.cloud import bigquery

def create_cmek_dataset(project_id, dataset_id, key_name):
    client = bigquery.Client(project=project_id)
    dataset = bigquery.Dataset(f"{project_id}.{dataset_id}")
    dataset.location = "US"
    dataset.default_encryption_configuration = \
        bigquery.EncryptionConfiguration(kms_key_name=key_name)
    return client.create_dataset(dataset)
```

---

### Cloud Storage CMEK

```bash
# Authorize GCS SA to use the key
gsutil kms authorize \
  -k projects/PROJECT/locations/us-central1/keyRings/prod-keyring/cryptoKeys/gcs-prod-cmek \
  -p PROJECT_ID

# Set bucket-level default CMEK key
gsutil kms encryption \
  -k projects/PROJECT/locations/us-central1/keyRings/prod-keyring/cryptoKeys/gcs-prod-cmek \
  gs://my-secure-bucket
```

---

### Compute Engine CMEK

```bash
# Create encrypted persistent disk
gcloud compute disks create my-encrypted-disk \
  --zone=us-central1-a \
  --size=100GB \
  --type=pd-ssd \
  --kms-key=projects/PROJECT/locations/us-central1/keyRings/prod-keyring/cryptoKeys/gce-prod-cmek

# Create an encrypted VM boot disk
gcloud compute instances create my-secure-vm \
  --zone=us-central1-a \
  --machine-type=n2-standard-4 \
  --boot-disk-kms-key=projects/PROJECT/locations/us-central1/keyRings/prod-keyring/cryptoKeys/gce-prod-cmek
```

---

### GKE Application-Layer Encryption (etcd CMEK)

```bash
# Create GKE cluster with application-layer encryption
gcloud container clusters create my-secure-cluster \
  --region=us-central1 \
  --database-encryption-key=projects/PROJECT/locations/us-central1/keyRings/prod-keyring/cryptoKeys/gke-etcd-cmek
```

---

### Organization Policy for CMEK

```bash
# Require CMEK for all new resources (organization-level)
gcloud resource-manager org-policies set-policy \
  --organization=ORG_ID policy.yaml
```

```yaml
# policy.yaml — restrict non-CMEK services
name: organizations/ORG_ID/policies/gcp.restrictNonCmekServices
spec:
  rules:
  - values:
      deniedValues:
      - is:storage.googleapis.com
      - is:bigquery.googleapis.com
      - is:compute.googleapis.com
```

> 🚨 **CMEK Key Destruction Impact:** If you destroy the CMEK key version protecting a GCS bucket, BigQuery dataset, or GCE disk, that resource's data becomes **permanently inaccessible**. There is no recovery — not even by Google.

---

## 14. Key Access Justifications

### What It Is

Key Access Justifications (KAJ) is a premium feature (part of Assured Workloads / Sovereignty Controls) that requires every request to access your KMS key to include a human-readable justification. You can configure policies to **automatically deny** certain justification types.

### How It Works

```
GCP internal operation (e.g., Google support access)
    │
    ▼
Cloud KMS (with KAJ enabled)
    │ Key access request includes justification code
    ▼
Your policy: ALLOW or DENY based on justification type
    │
    ▼
Key operation proceeds or is denied
    └─ Audit log always records the justification
```

### Justification Types

| Code | Meaning |
|---|---|
| `CUSTOMER_INITIATED_SUPPORT` | Customer opened a support ticket requesting access |
| `GOOGLE_INITIATED_SERVICE` | Google needs access for service maintenance |
| `GOOGLE_INITIATED_REVIEW` | Google security team review |
| `GOOGLE_RESPONSE_TO_PRODUCTION_ALERT` | Automated incident response |
| `THIRD_PARTY_DATA_REQUEST` | Legal/government request |

> 💡 **KAJ requires EKM.** Key Access Justifications only works with `EXTERNAL` or `EXTERNAL_VPC` protection levels — because the external KMS makes the allow/deny decision based on the justification sent by Google.

---

## 15. Monitoring, Alerting & Audit Logging

### Cloud Monitoring Metrics

| Metric | Description |
|---|---|
| `cloudkms.googleapis.com/request/count` | Total API requests (by method, key) |
| `cloudkms.googleapis.com/request/latencies` | Per-request latency |
| `cloudkms.googleapis.com/crypto_operation_count` | Encrypt/decrypt/sign operations |

---

### Key Rotation Alert

```bash
# Create alert: fire when next_rotation_time is within 7 days
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="KMS Key Rotation Due Soon" \
  --condition-display-name="next_rotation_time < 7 days" \
  --condition-filter='resource.type="cloudkms_cryptokey" metric.type="cloudkms.googleapis.com/key/next_rotation_time"' \
  --condition-threshold-value=604800 \
  --condition-threshold-comparison=COMPARISON_LT \
  --condition-duration=0s
```

---

### Key Destruction Alert (Log-Based Metric)

```bash
# Create a log-based metric for scheduled destruction events
gcloud logging metrics create kms-scheduled-destruction \
  --description="KMS key version scheduled for destruction" \
  --log-filter='resource.type="cloudkms_cryptokey" protoPayload.methodName="google.cloud.kms.v1.KeyManagementService.DestroyCryptoKeyVersion"'

# Create alert on the metric
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="KMS Key Version Scheduled for Destruction" \
  --condition-filter='metric.type="logging.googleapis.com/user/kms-scheduled-destruction"' \
  --condition-threshold-value=0 \
  --condition-threshold-comparison=COMPARISON_GT
```

---

### Unauthorized Access Alert

```bash
# Alert on PERMISSION_DENIED for KMS operations
gcloud logging metrics create kms-permission-denied \
  --log-filter='resource.type="cloudkms_cryptokey" severity=ERROR protoPayload.status.code=7'
```

---

### Useful Cloud Logging Queries

```bash
# Who encrypted/decrypted data in the last hour?
gcloud logging read \
  'resource.type="cloudkms_cryptokey" protoPayload.methodName=~"Encrypt|Decrypt"' \
  --freshness=1h \
  --format="table(timestamp, protoPayload.authenticationInfo.principalEmail, protoPayload.resourceName)"

# Who modified a key (rotation, disable, IAM change)?
gcloud logging read \
  'resource.type="cloudkms_cryptokey" protoPayload.methodName=~"UpdateCryptoKey|SetIamPolicy|DestroyCryptoKeyVersion"' \
  --freshness=24h

# Key version state changes
gcloud logging read \
  'resource.type="cloudkms_cryptokey" protoPayload.request.state!=""' \
  --freshness=7d
```

---

### BigQuery Security Analytics

```sql
-- Export audit logs to BigQuery and query with this
SELECT
  timestamp,
  protopayload_auditlog.authenticationInfo.principalEmail AS actor,
  protopayload_auditlog.methodName AS method,
  protopayload_auditlog.resourceName AS key_resource,
  protopayload_auditlog.status.code AS status_code
FROM `PROJECT.DATASET.cloudaudit_googleapis_com_data_access_*`
WHERE
  resource.type = 'cloudkms_cryptokey'
  AND protopayload_auditlog.methodName IN ('Encrypt', 'Decrypt', 'AsymmetricSign')
  AND _TABLE_SUFFIX BETWEEN '20240101' AND '20241231'
ORDER BY timestamp DESC
LIMIT 1000;
```

---

## 16. Best Practices & Security Patterns

### ✅ Do This

| Practice | Why |
|---|---|
| **Always use envelope encryption** | KMS 64 KB limit; better performance; DEK compromise is isolated |
| **Key ring per environment and purpose** | `prod-keyring`, `dev-keyring`, `signing-keyring` — isolate blast radius |
| **Regional key ring = data's region** | Data residency compliance; lower latency |
| **Separate KMS Admin from Encrypter/Decrypter** | Compromise of one doesn't give access to the other |
| **Enable automatic rotation (90 days)** | Limits cryptographic exposure window |
| **Use HSM for PCI-DSS, FIPS 140-2 workloads** | Required by most compliance frameworks |
| **Enable Data Access audit logs** | Essential for forensics and compliance |
| **Use VPC Service Controls** | Prevent KMS API calls from outside trusted networks |
| **Tag all KMS keys with labels** | Cost attribution, governance, resource tracking |
| **Use Workload Identity Federation** | Avoid service account key files entirely |

---

### ❌ Anti-Patterns to Avoid

| Anti-Pattern | Fix |
|---|---|
| Encrypting large payloads directly with KMS | Use envelope encryption |
| One key ring for all services and environments | Separate key rings by env and service |
| Granting `cryptoKeyEncrypterDecrypter` at project level | Scope to specific key resource |
| Combining KMS Admin + Encrypter/Decrypter on same SA | Separate roles, separate accounts |
| No key rotation policy | Set `rotation_period` on all symmetric keys |
| Not monitoring key expiry or destruction events | Set up Cloud Monitoring alerts |
| Using `global` location for CMEK keys | Use regional keys that match your data |
| No `prevent_destroy = true` in Terraform for prod keys | Always protect production keys |
| Not enabling Data Access audit logs | Required for security forensics |
| Storing DEKs in plaintext alongside encrypted data | Always store encrypted DEKs only |

---

### Key Naming Convention

```
{service}-{environment}-{purpose}

Examples:
  bigquery-prod-cmek
  gcs-staging-cmek
  api-prod-signing
  payments-prod-hmac
  gke-prod-etcd-cmek
```

---

### KMS Security Checklist

```
Key Management
  [ ] All symmetric keys have automatic rotation enabled (≤90 days for prod)
  [ ] Key rings are environment-separated (prod/staging/dev)
  [ ] Production keys have Terraform prevent_destroy = true
  [ ] No unused key versions (non-primary disabled or scheduled for destruction)
  [ ] HSM protection level for PCI/HIPAA/FIPS workloads

IAM
  [ ] KMS Admin role separated from Encrypter/Decrypter
  [ ] CMEK: cryptoKeyEncrypterDecrypter granted at key level, not project
  [ ] VPC Service Controls configured for Cloud KMS
  [ ] Periodic IAM review for KMS key bindings

Audit & Monitoring
  [ ] Data Access audit logs enabled for Cloud KMS
  [ ] Alert on SCHEDULED_FOR_DESTRUCTION key version events
  [ ] Alert on PERMISSION_DENIED for KMS operations
  [ ] Alert on key rotation due (next_rotation_time approaching)
  [ ] KMS audit logs exported to BigQuery for analytics

CMEK
  [ ] All prod data stores using CMEK (not Google-managed keys)
  [ ] Org policy constraining non-CMEK services
  [ ] CMEK key region matches data region
  [ ] Impact analysis for each CMEK key (which resources depend on it)

Envelope Encryption
  [ ] DEKs never stored in plaintext
  [ ] DEKs rotated or re-wrapped after KMS key rotation
  [ ] AAD used to bind DEKs to their data context
```

---

## 17. gcloud CLI & REST API Quick Reference

### Key Rings

```bash
gcloud kms keyrings create RING --location=LOC
gcloud kms keyrings list --location=LOC
gcloud kms keyrings describe RING --location=LOC
gcloud kms keyrings add-iam-policy-binding RING --location=LOC --member=MEMBER --role=ROLE
gcloud kms keyrings get-iam-policy RING --location=LOC
```

### Keys

```bash
gcloud kms keys create KEY --keyring=RING --location=LOC --purpose=PURPOSE [--default-algorithm=ALG] [--protection-level=LEVEL] [--rotation-period=PERIOD]
gcloud kms keys list --keyring=RING --location=LOC
gcloud kms keys describe KEY --keyring=RING --location=LOC
gcloud kms keys update KEY --keyring=RING --location=LOC --rotation-period=PERIOD
gcloud kms keys set-primary-version KEY --keyring=RING --location=LOC --version=VER
gcloud kms keys add-iam-policy-binding KEY --keyring=RING --location=LOC --member=MEMBER --role=ROLE
gcloud kms keys get-iam-policy KEY --keyring=RING --location=LOC
```

### Key Versions

```bash
gcloud kms keys versions create --key=KEY --keyring=RING --location=LOC
gcloud kms keys versions list --key=KEY --keyring=RING --location=LOC
gcloud kms keys versions describe VER --key=KEY --keyring=RING --location=LOC
gcloud kms keys versions enable VER --key=KEY --keyring=RING --location=LOC
gcloud kms keys versions disable VER --key=KEY --keyring=RING --location=LOC
gcloud kms keys versions schedule-destroy VER --key=KEY --keyring=RING --location=LOC
gcloud kms keys versions cancel-scheduled-destroy VER --key=KEY --keyring=RING --location=LOC
gcloud kms keys versions get-public-key VER --key=KEY --keyring=RING --location=LOC --output-file=pub.pem
gcloud kms keys versions get-certificate-chain VER --key=KEY --keyring=RING --location=LOC  # HSM attestation
gcloud kms keys versions import --key=KEY --keyring=RING --location=LOC --import-job=JOB --algorithm=ALG --wrapped-key-file=FILE
```

### Crypto Operations

```bash
gcloud kms encrypt --key=KEY --keyring=RING --location=LOC --plaintext-file=IN --ciphertext-file=OUT [--additional-authenticated-data=AAD]
gcloud kms decrypt --key=KEY --keyring=RING --location=LOC --ciphertext-file=IN --plaintext-file=OUT [--additional-authenticated-data=AAD]
gcloud kms asymmetric-sign --key=KEY --keyring=RING --location=LOC --version=VER --algorithm=ALG --input-file=IN --signature-file=OUT
gcloud kms asymmetric-decrypt --key=KEY --keyring=RING --location=LOC --version=VER --ciphertext-file=IN --plaintext-file=OUT
gcloud kms mac-sign --key=KEY --keyring=RING --location=LOC --version=VER --input-file=IN --signature-file=OUT
gcloud kms mac-verify --key=KEY --keyring=RING --location=LOC --version=VER --input-file=IN --signature-file=SIG
```

### Import Jobs & EKM Connections

```bash
gcloud kms import-jobs create JOB --location=LOC --keyring=RING --import-method=METHOD --protection-level=LEVEL
gcloud kms import-jobs list --location=LOC --keyring=RING
gcloud kms import-jobs describe JOB --location=LOC --keyring=RING

gcloud kms ekm-connections create CONN --location=LOC [options]
gcloud kms ekm-connections list --location=LOC
gcloud kms ekm-connections describe CONN --location=LOC
```

---

### Key Flags Reference

| Flag | Used With | Description |
|---|---|---|
| `--keyring` | Most commands | Key ring name |
| `--key` | Version commands, crypto ops | Crypto key name |
| `--version` | Version commands, asymmetric ops | Key version number |
| `--location` | All commands | GCP region (e.g., `us-central1`) |
| `--purpose` | `keys create` | `encryption`, `asymmetric-signing`, `asymmetric-encryption`, `mac` |
| `--algorithm` | `keys create`, `asymmetric-sign` | Algorithm identifier (e.g., `ec-sign-p256-sha256`) |
| `--protection-level` | `keys create`, `import-jobs create` | `software`, `hsm`, `external`, `external-vpc` |
| `--rotation-period` | `keys create`, `keys update` | e.g., `90d`, `7776000s` |
| `--plaintext-file` | `encrypt`, `decrypt` | Input/output file path; use `-` for stdin |
| `--ciphertext-file` | `encrypt`, `decrypt`, `asymmetric-decrypt` | Input/output file path |
| `--output-file` | `get-public-key` | PEM output file path |
| `--format` | Any | `json`, `yaml`, `table(field,...)` |

---

### REST API Base URLs

```
Base: https://cloudkms.googleapis.com/v1/

Key Rings:    projects/{p}/locations/{l}/keyRings/{kr}
Crypto Keys:  projects/{p}/locations/{l}/keyRings/{kr}/cryptoKeys/{k}
Key Versions: projects/{p}/locations/{l}/keyRings/{kr}/cryptoKeys/{k}/cryptoKeyVersions/{v}
Import Jobs:  projects/{p}/locations/{l}/keyRings/{kr}/importJobs/{j}
EKM Conns:    projects/{p}/locations/{l}/ekmConnections/{e}

Operations:
  POST .../cryptoKeys/{k}:encrypt
  POST .../cryptoKeys/{k}:decrypt
  POST .../cryptoKeyVersions/{v}:asymmetricSign
  POST .../cryptoKeyVersions/{v}:asymmetricDecrypt
  POST .../cryptoKeyVersions/{v}:macSign
  POST .../cryptoKeyVersions/{v}:macVerify
  GET  .../cryptoKeyVersions/{v}/publicKey
```

---

### Python Client Library

```python
# Install
pip install google-cloud-kms

# Import
from google.cloud import kms

# Client
client = kms.KeyManagementServiceClient()

# Key resource path builders
client.key_ring_path(project, location, key_ring)
client.crypto_key_path(project, location, key_ring, key)
client.crypto_key_version_path(project, location, key_ring, key, version)
client.import_job_path(project, location, key_ring, import_job)

# Key management
client.create_key_ring(request={"parent": ..., "key_ring_id": ..., "key_ring": {}})
client.create_crypto_key(request={"parent": ..., "crypto_key_id": ..., "crypto_key": {}})
client.create_crypto_key_version(request={"parent": ..., "crypto_key_version": {}})
client.list_crypto_keys(request={"parent": ...})
client.list_crypto_key_versions(request={"parent": ...})
client.update_crypto_key(request={"crypto_key": ..., "update_mask": ...})
client.update_crypto_key_primary_version(request={"name": ..., "crypto_key_version_id": ...})
client.destroy_crypto_key_version(request={"name": ...})

# Crypto operations
client.encrypt(request={"name": ..., "plaintext": ..., "additional_authenticated_data": ...})
client.decrypt(request={"name": ..., "ciphertext": ..., "additional_authenticated_data": ...})
client.asymmetric_sign(request={"name": ..., "digest": {"sha256": ...}})
client.asymmetric_decrypt(request={"name": ..., "ciphertext": ...})
client.mac_sign(request={"name": ..., "data": ...})
client.mac_verify(request={"name": ..., "data": ..., "mac": ...})
client.get_public_key(request={"name": ...})
```

---

## 18. Pricing Summary

> 💰 **No free tier.** All key versions and all API operations are charged.

### Key Version Cost (per month)

| Protection Level | Cost per Active Key Version |
|---|---|
| SOFTWARE | ~$0.06 / month |
| HSM | ~$1.00–$2.50 / month (varies by region) |
| EXTERNAL | ~$0.06 / month + partner KMS costs |
| EXTERNAL_VPC | ~$0.06 / month + partner KMS costs |

### API Operation Cost

| Operation | Price |
|---|---|
| Symmetric encrypt/decrypt | $0.03 per 10,000 operations |
| Asymmetric sign/decrypt | $0.15 per 10,000 operations |
| MAC sign/verify | $0.03 per 10,000 operations |
| Key version management (create, destroy) | Free |

### Estimated Monthly Cost: Typical CMEK Setup

```
Scenario: BigQuery + GCS + GCE with CMEK in prod + staging

Key Versions:
  8 keys × SOFTWARE   = 8 × $0.06  = $0.48/month
  2 keys × HSM        = 2 × $1.00  = $2.00/month

API Operations (est.):
  ~100K encrypt/decrypt ops/day × 30 days = 3M ops
  Symmetric: 3,000,000 / 10,000 × $0.03  = $9.00/month
  Asymmetric signing: 50K ops/month
  50,000 / 10,000 × $0.15               = $0.75/month

Total estimate: ~$12–15/month
```

---

### 💰 Cost Optimization Tips

- **Destroy unused key versions** — disabled versions with no decryption traffic still cost money
- **Use SOFTWARE protection** for dev/staging; reserve HSM for production compliance workloads
- **Minimize key versions** — clean up old non-primary versions after data has been re-encrypted
- **Batch encrypt/decrypt** at the application layer where possible (process multiple records per KMS call)
- **Cache public keys** for asymmetric signature verification — avoid GetPublicKey calls per request
- **Use envelope encryption** — one KMS call per DEK, not per data record

---

## 19. Quick Reference & Comparison Tables

### Cloud KMS Product Family Comparison

| Feature | Cloud KMS (SOFTWARE) | Cloud HSM | Cloud EKM |
|---|---|---|---|
| **Protection level** | SOFTWARE | HSM | EXTERNAL / EXTERNAL_VPC |
| **FIPS 140-2** | Level 1 | **Level 3** | Depends on partner |
| **Key material location** | Google infrastructure | Google HSM hardware | External KMS (never GCP) |
| **HSM attestation** | ❌ | ✅ | ❌ (partner-dependent) |
| **Latency** | ~0.1–0.5 ms | ~1–5 ms | ~10–100 ms (network dependent) |
| **Cost** | $ | $$$ | $$ + partner |
| **GCP availability dependency** | GCP | GCP | GCP + External KMS |
| **Compliance** | General | PCI-DSS, FIPS, FedRAMP High | Sovereignty, KAJ |
| **BYOK** | ✅ | ✅ | Key never imported |

---

### Key Purpose Reference

| Purpose | Allowed Algorithms | Typical Use Case |
|---|---|---|
| `ENCRYPT_DECRYPT` | `GOOGLE_SYMMETRIC_ENCRYPTION` | CMEK, data encryption, envelope KEK |
| `ASYMMETRIC_SIGN` | RSA-PSS, RSA-PKCS1, EC P-256, EC P-384 | JWTs, code signing, container signing |
| `ASYMMETRIC_DECRYPT` | RSA-OAEP (2048, 3072, 4096) | Key wrapping, secure secret transport |
| `MAC` | HMAC-SHA256, HMAC-SHA384, HMAC-SHA512 | API signing, webhook verification |

---

### Key Version State Machine

| State | Crypto Ops Allowed | How to Enter | How to Exit |
|---|---|---|---|
| `PENDING_GENERATION` | ❌ | Key version created (HSM) | Automatic → ENABLED |
| `PENDING_IMPORT` | ❌ | Import started | `import` → ENABLED |
| `ENABLED` | ✅ All | Create/import, re-enable | Disable |
| `DISABLED` | ❌ | Disable command | Re-enable or schedule-destroy |
| `SCHEDULED_FOR_DESTRUCTION` | ❌ | schedule-destroy | cancel (→ DISABLED) or timer expires (→ DESTROYED) |
| `DESTROYED` | ❌ | Timer expires after schedule | **Irreversible** |

---

### IAM Roles Summary

| Role | Encrypt | Decrypt | Sign | Create Keys | Delete Keys | Set IAM | View Keys |
|---|---|---|---|---|---|---|---|
| `cloudkms.admin` | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| `cloudkms.cryptoKeyEncrypterDecrypter` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| `cloudkms.cryptoKeyEncrypter` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| `cloudkms.cryptoKeyDecrypter` | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| `cloudkms.signerVerifier` | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ |
| `cloudkms.publicKeyViewer` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | Pub key only |
| `cloudkms.viewer` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ (metadata) |
| `cloudkms.importer` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

---

### CMEK-Supported Services

| Service | CMEK Granularity | Org Policy Constraint |
|---|---|---|
| BigQuery | Dataset, Table | `constraints/gcp.restrictNonCmekServices` |
| Cloud Storage | Bucket (default key) | `constraints/gcp.restrictNonCmekServices` |
| Compute Engine | Disk, Image, Snapshot | `constraints/gcp.restrictNonCmekServices` |
| Cloud SQL | Instance | `constraints/gcp.restrictNonCmekServices` |
| GKE | Cluster (etcd app-layer) | `constraints/gcp.restrictNonCmekServices` |
| Cloud Spanner | Instance, Database | `constraints/gcp.restrictNonCmekServices` |
| Pub/Sub | Topic, Subscription | `constraints/gcp.restrictNonCmekServices` |
| Dataflow | Job | — |
| Vertex AI | Dataset, Model, Pipeline | — |
| Cloud Run | Service | — |
| Secret Manager | Secret | — |

---

### Envelope Encryption vs. Direct Encryption

| | **Direct Encryption** | **Envelope Encryption** |
|---|---|---|
| **Max data size** | 64 KB | Unlimited |
| **KMS calls per operation** | 1 (per record) | 1 (per DEK, not per record) |
| **Performance** | High latency for large volume | Low latency (bulk ops local) |
| **Key compromise impact** | All records exposed | Only records using that DEK |
| **Use case** | Tiny secrets (passwords, tokens) | Files, DB rows, large payloads |
| **Complexity** | Low | Medium |

---

### Key Ring Location Decision Guide

| Requirement | Recommended Location Type |
|---|---|
| Data residency in a specific region | Regional (`us-central1`, `europe-west1`) |
| Redundancy within a continent | Multi-regional (`us`, `europe`) |
| Global signing keys (low residency requirements) | `global` (only for signing/MAC, not CMEK) |
| GDPR / data sovereignty | Regional in EU (`europe-west1`, `europe-west3`, `europe-west4`) |
| US government (FedRAMP) | `us-central1` or `us-east1` |

---

*Generated: GCP Cloud KMS Comprehensive Cheatsheet — covers Cloud KMS, Cloud HSM, Cloud EKM | Versions: Cloud KMS API v1*
