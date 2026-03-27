# GCP VPC Service Controls (VPC SC) — Comprehensive Cheatsheet

> **Audience:** Senior GCP engineers. No hand-holding — straight to operational depth.

---

## Table of Contents

1. [Overview & Core Philosophy](#1-overview--core-philosophy)
2. [Architecture & Enforcement Model](#2-architecture--enforcement-model)
3. [Access Policy](#3-access-policy)
4. [Service Perimeters — Deep Dive](#4-service-perimeters--deep-dive)
5. [Access Levels (Integration with ACM)](#5-access-levels-integration-with-acm)
6. [Ingress & Egress Rules — Deep Dive](#6-ingress--egress-rules--deep-dive)
7. [VPC Accessible Services](#7-vpc-accessible-services)
8. [Configuration & Code Examples](#8-configuration--code-examples)
9. [Terraform Examples](#9-terraform-examples)
10. [Private Google Access & DNS Configuration](#10-private-google-access--dns-configuration)
11. [Multi-Tenant & Multi-Project Perimeter Patterns](#11-multi-tenant--multi-project-perimeter-patterns)
12. [Common Integration Patterns & Code Examples](#12-common-integration-patterns--code-examples)
13. [Audit Logs & Observability](#13-audit-logs--observability)
14. [Troubleshooting & Lockout Recovery](#14-troubleshooting--lockout-recovery)
15. [Security Hardening & Best Practices](#15-security-hardening--best-practices)
16. [Limits & Quotas](#16-limits--quotas)
17. [Common Gotchas](#17-common-gotchas)
18. [Quick Reference — gcloud Commands](#18-quick-reference--gcloud-commands)
19. [Decision Tree](#19-decision-tree)

---

## 1. Overview & Core Philosophy

### What VPC Service Controls Solves

VPC firewall rules protect network traffic. IAM controls who can call an API. **VPC Service Controls adds a third layer: it controls *from where* an API can be called**, regardless of whether the caller has valid credentials.

Core threats addressed:

| Threat | How VPC SC Mitigates |
|---|---|
| **Credential theft** | Stolen token used from unauthorized network → perimeter blocks the call |
| **Misconfigured IAM** | Over-permissive role on a bucket → external caller hits perimeter before IAM |
| **Insider data exfiltration** | Employee copies data to personal GCS → egress rules block cross-project writes |
| **Supply chain attacks** | Compromised dependency calls GCP APIs → perimeter restricts callable services |
| **Confused deputy attacks** | A GCP service acting on behalf of an attacker → ingress rules limit which SAs can cross |

### What VPC SC Does NOT Protect Against

- Data already outside the perimeter (VPC SC is not DLP)
- Lateral movement **within** the perimeter between projects
- Threats to services not on the [VPC SC supported services list](https://cloud.google.com/vpc-service-controls/docs/supported-products)
- Network-level attacks (use VPC firewall rules + Cloud Armor for those)
- IAM misconfiguration **within** the perimeter (VPC SC + IAM are complementary, not substitutes)

### Disambiguation Table

| Control | Enforces At | Protects Against | Granularity |
|---|---|---|---|
| **VPC Firewall Rules** | Network layer (L3/L4) | Unauthorized TCP/UDP connections | IP, port, protocol |
| **Cloud IAM** | GCP API authorization layer | Unauthorized identity actions | Identity + resource + action |
| **VPC Service Controls** | GCP API data layer (perimeter) | Unauthorized access context (network origin, identity source) | Project + service + method + identity |
| **Cloud IAP** | HTTP proxy layer | Unauthenticated application access | User identity + device context |

### Key Terminology

| Term | Definition |
|---|---|
| **Access Policy** | Container for all ACM resources (access levels, perimeters); one per org |
| **Service Perimeter** | Logical boundary around GCP projects; restricts API calls crossing the boundary |
| **Restricted Services** | GCP APIs blocked from outside the perimeter (e.g., `bigquery.googleapis.com`) |
| **Access Level** | Conditions (IP, device, region) that allow crossing the perimeter |
| **Ingress Rule** | Explicit allow for inbound calls into the perimeter from outside |
| **Egress Rule** | Explicit allow for outbound calls from inside the perimeter to outside |
| **VPC Accessible Services** | Restricts which APIs VMs *inside the perimeter's VPC* can call |
| **Bridge Perimeter** | Connects two regular perimeters to allow resource sharing |
| **Dry-Run Mode** | Shadow enforcement: logs violations without blocking traffic |

### Data Plane vs. Control Plane

VPC SC enforces at the **data plane** — the actual API calls to read/write data:

```
Control Plane (IAM):  Who is allowed to call this API?
                      ↓
Data Plane (VPC SC):  Is this call coming from an allowed context?
                      ↓
Resource Policy:      Does the resource itself allow this operation?
```

VPC SC does **not** protect the Google Cloud control plane operations (project creation, billing, org policies).

---

## 2. Architecture & Enforcement Model

### Perimeter Enforcement Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        VPC SC Enforcement Model                              │
└─────────────────────────────────────────────────────────────────────────────┘

  OUTSIDE PERIMETER                    │  INSIDE PERIMETER
                                       │
  ┌──────────────────┐                 │  ┌──────────────────────────────────┐
  │  Developer       │  ────────────── │ ─▶│  BigQuery Dataset                │
  │  (corp laptop)   │  Access Level   │  │  GCS Bucket                      │
  │  ✅ if access     │  satisfied      │  │  Spanner Instance                │
  │  level matches   │                 │  │  Secret Manager Secret           │
  └──────────────────┘                 │  └──────────────────────────────────┘
                                       │
  ┌──────────────────┐                 │
  │  External Script │  ────────────── X  ❌ BLOCKED (no access level match)
  │  (unknown IP)    │                 │
  └──────────────────┘                 │
                                       │
  ┌──────────────────┐  Ingress Rule   │  ┌──────────────────────────────────┐
  │  ETL Service     │  ─────────────▶ │ ▶│  BigQuery (specific methods only) │
  │  Account (SA)    │  SA + source    │  └──────────────────────────────────┘
  └──────────────────┘  access level   │
                                       │
  ┌──────────────────────────────────┐ │  ┌──────────────────┐
  │  Inside: Archive SA              │ │  │  External Archive │
  │  ──── Egress Rule ──────────────▶│ X ▶│  GCS Bucket      │
  │  (specific project + method only)│ │  │  (limited write)  │
  └──────────────────────────────────┘ │  └──────────────────┘
                                       │
                        ─── VPC Accessible Services ───
  ┌──────────────────┐                 │
  │  VM inside VPC   │  Can only call  │  bigquery.googleapis.com ✅
  │  (compromised?)  │  allowed list:  │  storage.googleapis.com  ✅
  └──────────────────┘                 │  compute.googleapis.com  ❌ (not in list)
```

### Private Google API Access Path

```
VM (no public IP)
  │
  │  DNS: *.googleapis.com → restricted.googleapis.com (199.36.153.4/30)
  │  Route: 199.36.153.4/30 → default-internet-gateway
  ▼
restricted.googleapis.com  ──▶  VPC SC enforcement  ──▶  GCP API
      (for restricted services)

private.googleapis.com     ──▶  NO VPC SC enforcement ──▶  GCP API
(199.36.153.8/30)               (for non-restricted services only)
```

> **⚠️ Gotcha:** If a VM resolves `storage.googleapis.com` to `private.googleapis.com` (199.36.153.8/30) instead of `restricted.googleapis.com`, VPC SC enforcement is **bypassed**. Always configure DNS to route to `restricted.googleapis.com` for services in your restricted list.

### Request Evaluation Order

```
Incoming API Request
        │
        ▼
┌───────────────────┐
│  1. Authentication │  Is the caller authenticated? (valid token)
│     (Google STS)  │  NO → 401 Unauthenticated
└────────┬──────────┘
         │ YES
         ▼
┌───────────────────┐
│  2. IAM AuthZ     │  Does the caller have the required IAM role/permission?
│     Check         │  NO → 403 Permission Denied (IAM)
└────────┬──────────┘
         │ YES
         ▼
┌───────────────────┐
│  3. VPC SC        │  Is the caller's context (network, device, identity)
│     Perimeter     │  allowed to cross the perimeter boundary?
│     Evaluation    │  NO → 403 PERMISSION_DENIED (VPC SC violation logged)
└────────┬──────────┘
         │ YES (access level satisfied OR ingress/egress rule matches)
         ▼
┌───────────────────┐
│  4. Resource      │  Does the resource policy (bucket ACL, BQ dataset
│     Policy Check  │  policy) allow this operation?
└────────┬──────────┘
         │ YES
         ▼
     API Response
```

> **💡 Best Practice:** VPC SC violations return the same `403 PERMISSION_DENIED` as IAM denials. Always check Cloud Logging for `VpcServiceControlAuditMetadata` before assuming an IAM issue.

### Supported Services

Not all GCP services support VPC SC. Only services in the [supported products list](https://cloud.google.com/vpc-service-controls/docs/supported-products) can be restricted. Adding an unsupported service to `restrictedServices` has **no effect** — the service remains unprotected. Always verify support before relying on VPC SC for a given API.

High-risk services with VPC SC support (restrict these first):

| Service | API Name | Risk Level |
|---|---|---|
| Cloud Storage | `storage.googleapis.com` | 🔴 Critical |
| BigQuery | `bigquery.googleapis.com` | 🔴 Critical |
| Cloud Spanner | `spanner.googleapis.com` | 🔴 Critical |
| Secret Manager | `secretmanager.googleapis.com` | 🔴 Critical |
| Cloud SQL Admin | `sqladmin.googleapis.com` | 🟠 High |
| Pub/Sub | `pubsub.googleapis.com` | 🟠 High |
| Artifact Registry | `artifactregistry.googleapis.com` | 🟠 High |
| Vertex AI | `aiplatform.googleapis.com` | 🟠 High |
| Cloud KMS | `cloudkms.googleapis.com` | 🟠 High |
| Compute Engine | `compute.googleapis.com` | 🟡 Medium |
| Cloud Run | `run.googleapis.com` | 🟡 Medium |
| GKE | `container.googleapis.com` | 🟡 Medium |

---

## 3. Access Policy

### Hierarchy

```
Google Cloud Organization
  └── Access Policy (exactly ONE org-level; resource: accessPolicies/POLICY_NUMBER)
        ├── Access Levels
        │     ├── basic-level-1
        │     └── custom-cel-level-2
        ├── Service Perimeters
        │     ├── prod-data-perimeter (REGULAR)
        │     ├── analytics-perimeter (REGULAR)
        │     └── bridge-prod-analytics (BRIDGE)
        └── Access Bindings (for Context-Aware Access / Google Workspace)

  └── Scoped Access Policies (per folder or project — delegated admin)
        └── team-alpha-policy (scope: projects/PROJECT_ID)
              └── (own access levels + perimeters, independent of org policy)
```

### Org-Level vs. Scoped Policy

| Dimension | Org-Level Policy | Scoped Policy |
|---|---|---|
| Scope | Entire organization | Specific folder(s) or project(s) |
| Count | Exactly 1 per org | Multiple allowed |
| Admin delegation | Organization admins only | Can delegate to team/project admins |
| Perimeter reach | Any project in org | Only projects within the scope |
| Access level sharing | Available to all perimeters | Only available within the scoped policy |
| Use case | Central governance | Team-level self-service with guardrails |

### Retrieve Access Policy Number

```bash
# Get the org-level access policy (returns accessPolicies/POLICY_NUMBER)
gcloud access-context-manager policies list --organization=ORG_ID

# Extract just the policy number
POLICY_ID=$(gcloud access-context-manager policies list \
  --organization=ORG_ID \
  --format="value(name)" | sed 's/accessPolicies\///')
echo "Policy ID: $POLICY_ID"
```

### Access Policy IAM Roles

| Role | Description | Assign To |
|---|---|---|
| `roles/accesscontextmanager.policyAdmin` | Full CRUD on access levels, perimeters, bindings | Security architects |
| `roles/accesscontextmanager.policyReader` | Read-only view of all ACM resources | Auditors, developers |
| `roles/accesscontextmanager.policyEditor` | Create/update/delete but not set IAM | Security engineers |
| `roles/accesscontextmanager.gcpSecurityAdmin` | Manage ACM + org policy | Org admins |

```bash
# Grant policyAdmin on the org-level access policy
gcloud access-context-manager policies add-iam-policy-binding POLICY_NUMBER \
  --member="group:security-team@example.com" \
  --role="roles/accesscontextmanager.policyAdmin"
```

> **💡 Best Practice:** Separate the `policyAdmin` role from project-level IAM admins. The engineer who manages BigQuery IAM should not be able to modify the perimeter protecting it.

---

## 4. Service Perimeters — Deep Dive

### Perimeter Types

| Type | Behavior | When to Use |
|---|---|---|
| **PERIMETER_TYPE_REGULAR** | Hard boundary; unauthorized access blocked and logged | All production use cases |
| **PERIMETER_TYPE_BRIDGE** | Connects two regular perimeters; allows resource sharing between them | Cross-team projects that need data exchange without merging perimeters |

> **⚠️ Gotcha:** Bridge perimeters do **not** enforce access levels. They are purely a connectivity mechanism. Any project in either connected regular perimeter can access resources in the other. Use ingress/egress rules on the regular perimeters instead where possible.

### Perimeter Components (Complete Reference)

```yaml
# Conceptual perimeter structure (not literal API format)
servicePerimeter:
  name: "accessPolicies/POLICY_ID/servicePerimeters/PERIMETER_NAME"
  title: "Production Data Perimeter"
  type: PERIMETER_TYPE_REGULAR

  # Which GCP projects are INSIDE this perimeter
  resources:
  - projects/111111111111
  - projects/222222222222

  # Which GCP APIs are blocked for callers OUTSIDE the perimeter
  restrictedServices:
  - bigquery.googleapis.com
  - storage.googleapis.com
  - spanner.googleapis.com

  # Access levels that allow external callers to cross the perimeter
  # (applies to ALL identities that don't match an ingress rule)
  accessLevels:
  - accessPolicies/POLICY_ID/accessLevels/trusted_corp

  # Fine-grained allow rules for specific external identities/sources
  ingressPolicies:
  - ingressFrom: { ... }
    ingressTo: { ... }

  # Fine-grained allow rules for data flowing OUT of the perimeter
  egressPolicies:
  - egressFrom: { ... }
    egressTo: { ... }

  # Controls which APIs VMs inside the perimeter's VPC can call
  vpcAccessibleServices:
    enableRestriction: true
    allowedServices:
    - bigquery.googleapis.com
    - storage.googleapis.com

  # Dry-run: log violations without enforcing
  useExplicitDryRunSpec: true  # when true, spec below is dry-run; status is enforced
```

### Dry-Run Mode Explained

```
Enforced Perimeter (status field):
  → Blocks unauthorized requests
  → Logs violations to Cloud Audit Logs

Dry-Run Config (spec field, useExplicitDryRunSpec: true):
  → Does NOT block requests
  → Logs what WOULD be blocked as dry-run violations
  → Both can exist simultaneously (different configs)

Workflow:
  CREATE dry-run config → analyze violations → fix ingress/egress → ENFORCE
```

```bash
# A perimeter can have BOTH enforced config AND dry-run config simultaneously
# Enforced config: current production policy
# Dry-run config: testing a proposed new policy change

# Promote dry-run to enforced (replaces enforced with dry-run spec)
gcloud access-context-manager perimeters dry-run enforce PERIMETER_NAME \
  --policy=POLICY_ID

# Delete only the dry-run config (keep enforced)
gcloud access-context-manager perimeters dry-run delete PERIMETER_NAME \
  --policy=POLICY_ID
```

---

## 5. Access Levels (Integration with ACM)

### How VPC SC Uses Access Levels

Access levels attached to a perimeter's `accessLevels` field act as a **blanket pass** for any external caller who satisfies the conditions. They complement ingress rules:

- **Access Levels on perimeter**: "If you're from the corporate IP range on an encrypted device, you can cross the perimeter regardless of which SA you're using"
- **Ingress Rules**: "This specific SA from this specific source is allowed to call these specific methods"

### Access Level Types

```yaml
# Basic Access Level — AND-within-condition, OR-across-conditions
basicLevel:
  conditions:
  - ipSubnetworks: ["203.0.113.0/24", "10.0.0.0/8"]
    regions: ["IN", "US", "DE"]
    devicePolicy:
      requireScreenLock: true
      allowedEncryptionStatuses: [ENCRYPTED]
      requireCorpOwned: true
      allowedDeviceManagementLevels: [COMPLETE]
  combiningFunction: AND

# Custom Access Level — CEL expression
customLevel:
  expr:
    expression: >
      device.encryption_status == DeviceEncryptionStatus.ENCRYPTED &&
      device.management_state == DeviceManagementState.COMPLETE_DEVICE_MANAGEMENT &&
      !device.is_compromised &&
      origin.region_code in ["IN", "US", "DE"]
```

### Referencing Access Levels in Perimeters

```bash
# Full resource name format (required everywhere)
accessPolicies/POLICY_NUMBER/accessLevels/LEVEL_NAME

# Example:
accessPolicies/123456789/accessLevels/trusted_corp_network
```

> **⚠️ Gotcha:** Access level propagation takes **5–10 minutes** after creation or update. If you enforce a perimeter immediately after creating an access level, users may be incorrectly blocked during the propagation window. Always wait and verify with dry-run before enforcing.

---

## 6. Ingress & Egress Rules — Deep Dive

### Why Ingress/Egress Rules Exist

Access levels on a perimeter apply to **all** callers — you can't attach an access level only to a specific SA. Ingress rules solve this: they allow specific identities from specific sources to access specific resources and methods inside the perimeter, without needing to satisfy the perimeter-wide access level.

```
Access Level on perimeter:  Blanket pass for any caller satisfying conditions
Ingress Rule:               Targeted pass for specific SA + source + method combination
```

### Ingress Rule Structure (Complete)

```yaml
- ingressFrom:
    # Option A: Specific identities
    identities:
    - serviceAccount:etl-sa@PROJECT_ID.iam.gserviceaccount.com
    - user:admin@example.com

    # Option B: Identity type (mutually exclusive with identities list)
    # identityType: ANY_IDENTITY | ANY_USER_ACCOUNT | ANY_SERVICE_ACCOUNT

    # Sources: where the request originates from (AND logic with identities)
    sources:
    # Source option 1: must satisfy an access level
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
    # Source option 2: must originate from a specific VPC network
    - resource: //compute.googleapis.com/projects/PROJECT_ID/global/networks/NETWORK_NAME
    # Source option 3: allow from ANY source (use with caution)
    # - accessLevel: "*"

  ingressTo:
    # Which projects inside the perimeter are targeted
    resources:
    - '*'                          # All projects inside the perimeter
    # - projects/PROJECT_NUMBER    # Specific project only

    # Which APIs and methods are allowed
    operations:
    - serviceName: bigquery.googleapis.com
      methodSelectors:
      - method: '*'                # All BigQuery methods
    - serviceName: storage.googleapis.com
      methodSelectors:
      - method: google.storage.objects.get
      - method: google.storage.objects.list
      # Permission-based selector (alternative to method name)
      # - permission: storage.objects.get
```

### Egress Rule Structure (Complete)

```yaml
- egressFrom:
    identities:
    - serviceAccount:archive-sa@PROJECT_ID.iam.gserviceaccount.com
    # identityType: ANY_SERVICE_ACCOUNT  # Alternative

  egressTo:
    # Which external projects are allowed as destination
    resources:
    - projects/ARCHIVE_PROJECT_NUMBER
    # - '*'  # Any external project (rarely appropriate)

    operations:
    - serviceName: storage.googleapis.com
      methodSelectors:
      - method: google.storage.objects.create
      # - method: '*'  # All methods for this service
```

### Identity Type Reference

| `identityType` Value | What It Matches | Security Risk |
|---|---|---|
| `ANY_IDENTITY` | All users + all SAs | 🔴 Never use in production |
| `ANY_USER_ACCOUNT` | All human user accounts | 🟠 Use only for public-facing data |
| `ANY_SERVICE_ACCOUNT` | All service accounts in any project | 🟠 Moderately risky |
| *(omit — use identities list)* | Specific named users/SAs | ✅ Recommended |

> **⚠️ Gotcha:** Using `ANY_IDENTITY` as `identityType` in an ingress rule effectively makes the perimeter publicly accessible for the specified methods. This is almost never the intended behavior. Always use the explicit `identities` list.

### Common Ingress/Egress Patterns

#### Pattern 1: External SA reads from GCS inside perimeter

```yaml
- ingressFrom:
    identities:
    - serviceAccount:reader-sa@EXTERNAL_PROJECT.iam.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
  ingressTo:
    resources:
    - '*'
    operations:
    - serviceName: storage.googleapis.com
      methodSelectors:
      - method: google.storage.objects.get
      - method: google.storage.objects.list
      - method: google.storage.buckets.list
```

#### Pattern 2: Dataflow SA accesses BigQuery + GCS inside perimeter

```yaml
- ingressFrom:
    identities:
    - serviceAccount:dataflow-sa@PIPELINE_PROJECT.iam.gserviceaccount.com
    # Dataflow also uses a Dataflow service agent — include both
    - serviceAccount:service-PIPELINE_PROJECT_NUMBER@dataflow-service-account.iam.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
  ingressTo:
    resources:
    - '*'
    operations:
    - serviceName: bigquery.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: storage.googleapis.com
      methodSelectors:
      - method: '*'
```

#### Pattern 3: Cloud Build deploys to Cloud Run inside perimeter

```yaml
- ingressFrom:
    identities:
    - serviceAccount:BUILD_PROJECT_NUMBER@cloudbuild.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
  ingressTo:
    resources:
    - projects/TARGET_PROJECT_NUMBER
    operations:
    - serviceName: run.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: artifactregistry.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: cloudresourcemanager.googleapis.com
      methodSelectors:
      - method: '*'
```

#### Pattern 4: Data export to external archive project

```yaml
- egressFrom:
    identities:
    - serviceAccount:archive-sa@INSIDE_PROJECT.iam.gserviceaccount.com
  egressTo:
    resources:
    - projects/ARCHIVE_PROJECT_NUMBER
    operations:
    - serviceName: storage.googleapis.com
      methodSelectors:
      - method: google.storage.objects.create
      - method: google.storage.objects.get  # Needed for resumable upload
```

---

## 7. VPC Accessible Services

### What It Controls

`vpcAccessibleServices` is a **second layer of restriction** that applies to API calls originating from **VM instances inside the perimeter's VPC networks**. It answers: "Even if a VM is inside the perimeter, which APIs is it allowed to call?"

```
Without vpcAccessibleServices:
  VM inside perimeter → can call ANY GCP API (including unrestricted ones)

With vpcAccessibleServices (enableRestriction: true):
  VM inside perimeter → can ONLY call APIs in the allowedServices list
```

### Configuration

```yaml
vpcAccessibleServices:
  enableRestriction: true
  allowedServices:
  - bigquery.googleapis.com
  - storage.googleapis.com
  - spanner.googleapis.com
  - secretmanager.googleapis.com
  # Do NOT include services the VMs don't need
```

### `restrictedServices` vs. `vpcAccessibleServices`

| Dimension | `restrictedServices` | `vpcAccessibleServices` |
|---|---|---|
| Controls | Calls **from outside** the perimeter | Calls **from VMs inside** the perimeter's VPC |
| Default behavior | Services are NOT restricted (must opt-in) | Restriction disabled by default |
| Direction | Inbound to perimeter | Outbound from VMs |
| Use case | Block external/credential-theft access | Limit blast radius of compromised VM |
| Requires `enableRestriction` | No | Yes |

> **💡 Best Practice:** Enable `vpcAccessibleServices` with a tight allowlist even if `restrictedServices` is already configured. A compromised VM inside the perimeter should not be able to exfiltrate data via unrestricted GCP services like `translate.googleapis.com` or `vision.googleapis.com`.

### DNS Requirement for `restricted.googleapis.com`

When `vpcAccessibleServices` is enabled, VMs must reach APIs through `restricted.googleapis.com` (199.36.153.4/30) to have VPC SC enforced. If they resolve to `private.googleapis.com` (199.36.153.8/30) instead, VPC SC does not apply.

```
VM (VPC inside perimeter)
  │
  │  DNS query: bigquery.googleapis.com
  ▼
Cloud DNS (private zone override):
  *.googleapis.com → CNAME → restricted.googleapis.com
  restricted.googleapis.com → A → 199.36.153.4 (VPC SC enforced ✅)

  (NOT private.googleapis.com → 199.36.153.8 — VPC SC bypassed ❌)
```

---

## 8. Configuration & Code Examples

### Access Policy (gcloud)

```bash
# Get the org-level access policy (returns full resource name)
gcloud access-context-manager policies list --organization=ORG_ID

# Extract the numeric policy ID
POLICY_ID=$(gcloud access-context-manager policies list \
  --organization=ORG_ID \
  --format="value(name)" | sed 's|accessPolicies/||')

# Create a scoped access policy for delegated admin
gcloud access-context-manager policies create \
  --organization=ORG_ID \
  --scopes=projects/PROJECT_ID \
  --title="Team Alpha Scoped Policy"

# Grant policy admin to a group
gcloud access-context-manager policies add-iam-policy-binding $POLICY_ID \
  --member="group:security-team@example.com" \
  --role="roles/accesscontextmanager.policyAdmin"
```

### Service Perimeter Full Lifecycle (gcloud)

```bash
# ── Step 1: Create perimeter in DRY-RUN mode ──────────────────────────────────

gcloud access-context-manager perimeters dry-run create PROD_DATA_PERIMETER \
  --policy=$POLICY_ID \
  --title="Production Data Perimeter" \
  --type=PERIMETER_TYPE_REGULAR \
  --resources=projects/111111111111,projects/222222222222 \
  --restricted-services=\
bigquery.googleapis.com,\
storage.googleapis.com,\
spanner.googleapis.com,\
secretmanager.googleapis.com \
  --access-levels=accessPolicies/$POLICY_ID/accessLevels/trusted_corp

# ── Step 2: Monitor dry-run violations for 48–72 hours ────────────────────────
# (Use Cloud Logging filters from §13)

# ── Step 3: Add ingress rules for identified legitimate violations ─────────────

gcloud access-context-manager perimeters dry-run update PROD_DATA_PERIMETER \
  --policy=$POLICY_ID \
  --set-ingress-policies=ingress-policy.yaml

# ── Step 4: Enforce ───────────────────────────────────────────────────────────

gcloud access-context-manager perimeters dry-run enforce PROD_DATA_PERIMETER \
  --policy=$POLICY_ID

# ── Incremental updates after enforcement ─────────────────────────────────────

# Add a restricted service
gcloud access-context-manager perimeters update PROD_DATA_PERIMETER \
  --policy=$POLICY_ID \
  --add-restricted-services=pubsub.googleapis.com

# Add a project to the perimeter
gcloud access-context-manager perimeters update PROD_DATA_PERIMETER \
  --policy=$POLICY_ID \
  --add-resources=projects/333333333333

# Update ingress policies (replaces all existing ingress policies)
gcloud access-context-manager perimeters update PROD_DATA_PERIMETER \
  --policy=$POLICY_ID \
  --set-ingress-policies=updated-ingress.yaml

# Update egress policies
gcloud access-context-manager perimeters update PROD_DATA_PERIMETER \
  --policy=$POLICY_ID \
  --set-egress-policies=updated-egress.yaml

# Describe perimeter (full YAML — use for IaC reconciliation)
gcloud access-context-manager perimeters describe PROD_DATA_PERIMETER \
  --policy=$POLICY_ID \
  --format=yaml
```

### Ingress Policy YAML Examples

```yaml
# ingress-policy.yaml — Multi-rule ingress policy file
# Rule 1: ETL SA from corp network
- ingressFrom:
    identities:
    - serviceAccount:etl-sa@DATA_PROJECT.iam.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
  ingressTo:
    resources:
    - '*'
    operations:
    - serviceName: bigquery.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: storage.googleapis.com
      methodSelectors:
      - method: google.storage.objects.get
      - method: google.storage.objects.list

# Rule 2: Cloud Composer (Airflow) orchestrator SA
- ingressFrom:
    identities:
    - serviceAccount:composer-sa@ORCHESTRATION_PROJECT.iam.gserviceaccount.com
    - serviceAccount:service-ORCHESTRATION_PROJECT_NUMBER@cloudcomposer-accounts.iam.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
  ingressTo:
    resources:
    - '*'
    operations:
    - serviceName: bigquery.googleapis.com
      methodSelectors:
      - method: '*'

# Rule 3: Terraform CI/CD runner SA
- ingressFrom:
    identities:
    - serviceAccount:terraform-runner@CICD_PROJECT.iam.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
  ingressTo:
    resources:
    - '*'
    operations:
    - serviceName: bigquery.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: storage.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: cloudresourcemanager.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: secretmanager.googleapis.com
      methodSelectors:
      - method: '*'
```

```yaml
# egress-policy.yaml — Multi-rule egress policy file
# Rule 1: Archive SA writes to external archive project
- egressFrom:
    identities:
    - serviceAccount:archive-sa@DATA_PROJECT.iam.gserviceaccount.com
  egressTo:
    resources:
    - projects/ARCHIVE_PROJECT_NUMBER
    operations:
    - serviceName: storage.googleapis.com
      methodSelectors:
      - method: google.storage.objects.create
      - method: google.storage.objects.get

# Rule 2: ML model export to shared model registry project
- egressFrom:
    identities:
    - serviceAccount:ml-trainer-sa@DATA_PROJECT.iam.gserviceaccount.com
  egressTo:
    resources:
    - projects/MODEL_REGISTRY_PROJECT_NUMBER
    operations:
    - serviceName: artifactregistry.googleapis.com
      methodSelectors:
      - method: '*'
```

```bash
# Apply both policy files
gcloud access-context-manager perimeters update PROD_DATA_PERIMETER \
  --policy=$POLICY_ID \
  --set-ingress-policies=ingress-policy.yaml

gcloud access-context-manager perimeters update PROD_DATA_PERIMETER \
  --policy=$POLICY_ID \
  --set-egress-policies=egress-policy.yaml
```

---

## 9. Terraform Examples

### Provider & Variables

```hcl
# variables.tf
variable "org_id" {
  description = "GCP Organization ID"
  type        = string
}

variable "access_policy_id" {
  description = "Numeric ACM access policy ID (without 'accessPolicies/' prefix)"
  type        = string
}

variable "prod_project_number" {
  description = "Numeric project number of the production project"
  type        = string
}

variable "analytics_project_number" {
  description = "Numeric project number of the analytics project"
  type        = string
}

variable "archive_project_number" {
  description = "Numeric project number of the external archive project"
  type        = string
}

variable "project_id" {
  description = "Project ID (for service account references)"
  type        = string
}
```

### Access Level

```hcl
# Basic access level: IP + device policy
resource "google_access_context_manager_access_level" "trusted_corp" {
  parent = "accessPolicies/${var.access_policy_id}"
  name   = "accessPolicies/${var.access_policy_id}/accessLevels/trusted_corp"
  title  = "Trusted Corporate Network"

  basic {
    combining_function = "AND"

    conditions {
      ip_subnetworks = ["203.0.113.0/24", "10.100.0.0/16"]
      regions        = ["IN", "US", "DE", "GB"]

      device_policy {
        require_screen_lock              = true
        require_corp_owned               = true
        allowed_encryption_statuses      = ["ENCRYPTED"]
        allowed_device_management_levels = ["COMPLETE"]
      }
    }
  }
}

# Custom CEL-based access level
resource "google_access_context_manager_access_level" "cel_trusted" {
  parent = "accessPolicies/${var.access_policy_id}"
  name   = "accessPolicies/${var.access_policy_id}/accessLevels/cel_trusted"
  title  = "CEL — Managed Encrypted Device"

  custom {
    expr {
      expression = <<-CEL
        device.encryption_status == DeviceEncryptionStatus.ENCRYPTED &&
        device.management_state == DeviceManagementState.COMPLETE_DEVICE_MANAGEMENT &&
        !device.is_compromised &&
        origin.region_code in ["IN", "US", "DE", "GB"]
      CEL
    }
  }
}
```

### Regular Perimeter with All Components

```hcl
resource "google_access_context_manager_service_perimeter" "prod_perimeter" {
  parent = "accessPolicies/${var.access_policy_id}"
  name   = "accessPolicies/${var.access_policy_id}/servicePerimeters/prod_data_perimeter"
  title  = "Production Data Perimeter"
  type   = "PERIMETER_TYPE_REGULAR"

  # Prevent accidental perimeter deletion
  lifecycle {
    prevent_destroy = true
  }

  status {
    resources = [
      "projects/${var.prod_project_number}",
      "projects/${var.analytics_project_number}",
    ]

    restricted_services = [
      "bigquery.googleapis.com",
      "storage.googleapis.com",
      "spanner.googleapis.com",
      "secretmanager.googleapis.com",
      "cloudkms.googleapis.com",
      "pubsub.googleapis.com",
    ]

    # Blanket access level: any caller satisfying this can cross the perimeter
    access_levels = [
      google_access_context_manager_access_level.trusted_corp.name,
    ]

    # VMs inside the perimeter can only call these APIs
    vpc_accessible_services {
      enable_restriction = true
      allowed_services = [
        "bigquery.googleapis.com",
        "storage.googleapis.com",
        "spanner.googleapis.com",
        "secretmanager.googleapis.com",
      ]
    }

    # Ingress rule 1: ETL pipeline SA from corp-level access
    ingress_policies {
      ingress_from {
        identity_type = "SERVICE_ACCOUNTS"
        identities    = ["serviceAccount:etl-sa@${var.project_id}.iam.gserviceaccount.com"]
        sources {
          access_level = google_access_context_manager_access_level.trusted_corp.name
        }
      }
      ingress_to {
        resources = ["*"]
        operations {
          service_name = "bigquery.googleapis.com"
          method_selectors { method = "*" }
        }
        operations {
          service_name = "storage.googleapis.com"
          method_selectors { method = "*" }
        }
      }
    }

    # Ingress rule 2: Terraform CI/CD runner
    ingress_policies {
      ingress_from {
        identity_type = "SERVICE_ACCOUNTS"
        identities    = ["serviceAccount:terraform-runner@${var.project_id}.iam.gserviceaccount.com"]
        sources {
          access_level = google_access_context_manager_access_level.trusted_corp.name
        }
      }
      ingress_to {
        resources = ["*"]
        operations {
          service_name = "bigquery.googleapis.com"
          method_selectors { method = "*" }
        }
        operations {
          service_name = "storage.googleapis.com"
          method_selectors { method = "*" }
        }
        operations {
          service_name = "cloudresourcemanager.googleapis.com"
          method_selectors { method = "*" }
        }
        operations {
          service_name = "secretmanager.googleapis.com"
          method_selectors { method = "*" }
        }
      }
    }

    # Egress rule: archive SA can write to external archive project
    egress_policies {
      egress_from {
        identity_type = "SERVICE_ACCOUNTS"
        identities    = ["serviceAccount:archive-sa@${var.project_id}.iam.gserviceaccount.com"]
      }
      egress_to {
        resources = ["projects/${var.archive_project_number}"]
        operations {
          service_name = "storage.googleapis.com"
          method_selectors { method = "google.storage.objects.create" }
          method_selectors { method = "google.storage.objects.get" }
        }
      }
    }
  }

  depends_on = [
    google_access_context_manager_access_level.trusted_corp,
  ]
}
```

### Bridge Perimeter

```hcl
# Bridge: allows prod-perimeter projects to access analytics-perimeter resources
# Both perimeters must already exist before creating the bridge
resource "google_access_context_manager_service_perimeter" "prod_analytics_bridge" {
  parent = "accessPolicies/${var.access_policy_id}"
  name   = "accessPolicies/${var.access_policy_id}/servicePerimeters/bridge_prod_analytics"
  title  = "Bridge: Prod ↔ Analytics"
  type   = "PERIMETER_TYPE_BRIDGE"

  status {
    resources = [
      "projects/${var.prod_project_number}",
      "projects/${var.analytics_project_number}",
    ]
  }

  depends_on = [
    google_access_context_manager_service_perimeter.prod_perimeter,
  ]
}
```

### Dry-Run Perimeter (Terraform)

```hcl
resource "google_access_context_manager_service_perimeter" "staging_dryrun" {
  parent = "accessPolicies/${var.access_policy_id}"
  name   = "accessPolicies/${var.access_policy_id}/servicePerimeters/staging_dryrun"
  title  = "Staging Perimeter (Dry-Run)"
  type   = "PERIMETER_TYPE_REGULAR"

  # Use explicit dry-run spec — spec = dry-run, status = enforced (can be empty)
  use_explicit_dry_run_spec = true

  # Enforced config (empty = no enforcement)
  status {}

  # Dry-run config (logged but not enforced)
  spec {
    resources           = ["projects/${var.prod_project_number}"]
    restricted_services = ["bigquery.googleapis.com", "storage.googleapis.com"]
    access_levels       = [google_access_context_manager_access_level.trusted_corp.name]
  }
}
```

---

## 10. Private Google Access & DNS Configuration

### Overview: `restricted.googleapis.com` vs. `private.googleapis.com`

| Endpoint | IP Range | VPC SC Enforced? | Use For |
|---|---|---|---|
| `restricted.googleapis.com` | 199.36.153.4/30 | ✅ Yes | APIs in `restrictedServices` list |
| `private.googleapis.com` | 199.36.153.8/30 | ❌ No | APIs NOT in restricted services list |
| `*.googleapis.com` (default) | Public IPs | ❌ No (uses internet) | Default; VMs with public IP |

### DNS Configuration (Cloud DNS)

```bash
# ── Step 1: Create private DNS zone overriding googleapis.com ─────────────────

gcloud dns managed-zones create restricted-googleapis \
  --dns-name="googleapis.com." \
  --description="Route all googleapis.com to restricted.googleapis.com for VPC SC" \
  --visibility=private \
  --networks=VPC_NETWORK_NAME \
  --project=HOST_PROJECT_ID

# ── Step 2: A record for restricted.googleapis.com ───────────────────────────

gcloud dns record-sets create restricted.googleapis.com. \
  --zone=restricted-googleapis \
  --type=A \
  --ttl=300 \
  --rrdatas="199.36.153.4,199.36.153.5,199.36.153.6,199.36.153.7" \
  --project=HOST_PROJECT_ID

# ── Step 3: Wildcard CNAME routing all *.googleapis.com to restricted ─────────

gcloud dns record-sets create "*.googleapis.com." \
  --zone=restricted-googleapis \
  --type=CNAME \
  --ttl=300 \
  --rrdatas="restricted.googleapis.com." \
  --project=HOST_PROJECT_ID

# ── Step 4: gcr.io and pkg.dev (container registry) ─────────────────────────

# For container registries — separate zones needed
gcloud dns managed-zones create restricted-gcr \
  --dns-name="gcr.io." \
  --visibility=private \
  --networks=VPC_NETWORK_NAME \
  --project=HOST_PROJECT_ID

gcloud dns record-sets create "*.gcr.io." \
  --zone=restricted-gcr \
  --type=CNAME \
  --ttl=300 \
  --rrdatas="restricted.googleapis.com." \
  --project=HOST_PROJECT_ID

gcloud dns managed-zones create restricted-pkg-dev \
  --dns-name="pkg.dev." \
  --visibility=private \
  --networks=VPC_NETWORK_NAME \
  --project=HOST_PROJECT_ID

gcloud dns record-sets create "*.pkg.dev." \
  --zone=restricted-pkg-dev \
  --type=CNAME \
  --ttl=300 \
  --rrdatas="restricted.googleapis.com." \
  --project=HOST_PROJECT_ID
```

### Route Configuration

```bash
# ── Route for restricted.googleapis.com range ─────────────────────────────────

gcloud compute routes create restricted-googleapis-route \
  --network=VPC_NETWORK_NAME \
  --destination-range=199.36.153.4/30 \
  --next-hop-gateway=default-internet-gateway \
  --priority=100 \
  --project=HOST_PROJECT_ID

# ── Route for private.googleapis.com range (non-restricted APIs) ──────────────

gcloud compute routes create private-googleapis-route \
  --network=VPC_NETWORK_NAME \
  --destination-range=199.36.153.8/30 \
  --next-hop-gateway=default-internet-gateway \
  --priority=100 \
  --project=HOST_PROJECT_ID
```

### Terraform: DNS + Routes

```hcl
resource "google_dns_managed_zone" "restricted_googleapis" {
  name        = "restricted-googleapis"
  dns_name    = "googleapis.com."
  description = "Route googleapis.com to restricted.googleapis.com for VPC SC"
  visibility  = "private"
  project     = var.project_id

  private_visibility_config {
    networks {
      network_url = google_compute_network.vpc.id
    }
  }
}

resource "google_dns_record_set" "restricted_a" {
  name         = "restricted.googleapis.com."
  type         = "A"
  ttl          = 300
  managed_zone = google_dns_managed_zone.restricted_googleapis.name
  project      = var.project_id
  rrdatas      = ["199.36.153.4", "199.36.153.5", "199.36.153.6", "199.36.153.7"]
}

resource "google_dns_record_set" "wildcard_googleapis_cname" {
  name         = "*.googleapis.com."
  type         = "CNAME"
  ttl          = 300
  managed_zone = google_dns_managed_zone.restricted_googleapis.name
  project      = var.project_id
  rrdatas      = ["restricted.googleapis.com."]
}

resource "google_compute_route" "restricted_googleapis" {
  name             = "restricted-googleapis-route"
  network          = google_compute_network.vpc.name
  project          = var.project_id
  dest_range       = "199.36.153.4/30"
  next_hop_gateway = "default-internet-gateway"
  priority         = 100
}
```

> **💡 Best Practice:** In Shared VPC environments, create the DNS zones in the **host project** and set them as private zones visible to all service project VPCs. This ensures consistent routing across all projects in the perimeter.

---

## 11. Multi-Tenant & Multi-Project Perimeter Patterns

### Pattern 1: Single-Project Perimeter

```
┌─────────────────────────────┐
│  Perimeter: data-perimeter  │
│                             │
│  ┌─────────────────────┐    │
│  │  data-project       │    │
│  │  BigQuery / GCS     │    │
│  └─────────────────────┘    │
└─────────────────────────────┘
Ingress: specific SAs from ETL project
Egress:  archive SA to archive project
```

Simplest topology. One project = one blast radius.

### Pattern 2: Multi-Project + Shared VPC Perimeter

```
┌──────────────────────────────────────────────────────────────────┐
│  Service Perimeter: prod-data-perimeter                          │
│                                                                  │
│  ┌──────────────────┐    ┌──────────────────┐                   │
│  │  host-project    │    │  data-project    │                   │
│  │  (Shared VPC)    │◀──▶│  BigQuery / GCS  │                   │
│  │  MUST be in      │    │  Spanner         │                   │
│  │  perimeter       │    └──────────────────┘                   │
│  └──────────────────┘                                            │
│                                                                  │
│  ┌──────────────────┐    ┌──────────────────┐                   │
│  │  analytics-proj  │    │  ml-project      │                   │
│  │  Dataflow / BQ   │    │  Vertex AI       │                   │
│  └──────────────────┘    └──────────────────┘                   │
└──────────────────────────────────────────────────────────────────┘
    ▲  Ingress: etl-sa from corp network with trusted_corp access level
    ▼  Egress: archive-sa → archive-project (GCS write only)

Outside perimeter:
  cicd-project (Cloud Build SA → ingress rule required)
  archive-project (egress rule required)
```

> **⚠️ Gotcha:** The **Shared VPC host project MUST be included** in the service perimeter. If it's omitted, VMs in service projects cannot reach restricted APIs because VPC SC sees their requests as originating from outside the perimeter.

### Pattern 3: Hub-and-Spoke with Bridge

```
┌────────────────────────────────┐    ┌────────────────────────────────┐
│  Perimeter: central-data       │    │  Perimeter: team-analytics     │
│                                │    │                                │
│  central-data-project          │    │  analytics-project-a           │
│  (BigQuery, GCS data lake)     │    │  analytics-project-b           │
│                                │    │  (Looker, Dataform)            │
└────────────────────────────────┘    └────────────────────────────────┘
              │                                      │
              └──────────── Bridge Perimeter ────────┘
                    (projects from both perimeters)

Use case: central data team manages the lake; analytics team has own perimeter
Bridge allows cross-perimeter BigQuery reads without merging the perimeters
```

### Pattern 4: Cross-Organization (No Native Support)

VPC SC does not support cross-organization perimeters. Workaround:

```yaml
# Ingress rule: allow SA from external org's project
- ingressFrom:
    identities:
    - serviceAccount:partner-etl@PARTNER_PROJECT.iam.gserviceaccount.com
    sources:
    # Cannot use access levels from other org — use VPC network source instead
    - resource: //compute.googleapis.com/projects/PARTNER_PROJECT/global/networks/PARTNER_NETWORK
  ingressTo:
    resources:
    - projects/YOUR_PROJECT_NUMBER
    operations:
    - serviceName: bigquery.googleapis.com
      methodSelectors:
      - method: google.cloud.bigquery.v2.JobService.InsertJob
```

### Perimeter Design Checklist

Before creating a perimeter, map:

- [ ] All GCP projects that contain the restricted services
- [ ] All GCP projects whose VMs or services access those APIs (include these in perimeter OR create ingress rules)
- [ ] The Shared VPC host project (always in perimeter if using Shared VPC)
- [ ] All service accounts that cross the perimeter boundary
- [ ] All Google-managed SAs (Dataflow, Composer, Cloud Build, Vertex AI)
- [ ] CI/CD pipeline SAs (Terraform runner, Cloud Build SA, GitHub Actions WIF SA)
- [ ] Which methods each SA needs (minimize method wildcards)
- [ ] Which external projects need egress rule access (archive, partner projects)

---

## 12. Common Integration Patterns & Code Examples

### Pattern 1: Dataflow — GCS → BigQuery (All Inside Perimeter)

If the Dataflow worker project is **inside the perimeter**, no ingress/egress rules are needed. The critical requirement: include the Dataflow worker project in `resources`.

```yaml
# perimeter resources must include Dataflow worker project
resources:
- projects/DATA_PROJECT_NUMBER
- projects/DATAFLOW_WORKER_PROJECT_NUMBER  # ← often forgotten
- projects/HOST_PROJECT_NUMBER             # ← Shared VPC host
```

If Dataflow workers run in a **separate project outside the perimeter**:

```yaml
# ingress-dataflow.yaml
- ingressFrom:
    identities:
    - serviceAccount:dataflow-sa@WORKER_PROJECT.iam.gserviceaccount.com
    - serviceAccount:service-WORKER_PROJECT_NUMBER@dataflow-service-account.iam.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
  ingressTo:
    resources:
    - '*'
    operations:
    - serviceName: bigquery.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: storage.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: dataflow.googleapis.com
      methodSelectors:
      - method: '*'
```

### Pattern 2: Cloud Build Deploying Inside Perimeter

```yaml
# ingress-cloudbuild.yaml
- ingressFrom:
    identities:
    # Cloud Build service account format: PROJECT_NUMBER@cloudbuild.gserviceaccount.com
    - serviceAccount:CICD_PROJECT_NUMBER@cloudbuild.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
  ingressTo:
    resources:
    - projects/TARGET_PROJECT_NUMBER
    operations:
    - serviceName: run.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: artifactregistry.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: cloudbuild.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: cloudresourcemanager.googleapis.com
      methodSelectors:
      - method: '*'
```

> **💡 Best Practice:** For Cloud Build accessing perimeter resources, consider using [**Cloud Build Private Pools**](https://cloud.google.com/build/docs/private-pools/private-pools-overview) running inside the perimeter. Then Cloud Build workers are inside the perimeter and no ingress rules are needed.

### Pattern 3: Looker Studio / BI Tool Reading BigQuery

**Option A: Looker SA ingress rule** (for Looker Enterprise with a dedicated SA):

```yaml
- ingressFrom:
    identities:
    - serviceAccount:looker-sa@LOOKER_PROJECT.iam.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
  ingressTo:
    resources:
    - projects/DATA_PROJECT_NUMBER
    operations:
    - serviceName: bigquery.googleapis.com
      methodSelectors:
      - method: google.cloud.bigquery.v2.JobService.InsertJob
      - method: google.cloud.bigquery.v2.JobService.GetJob
      - method: google.cloud.bigquery.v2.TableService.GetTable
      - method: google.cloud.bigquery.v2.DatasetService.GetDataset
```

**Option B: User-level access level** (for Looker Studio with individual user auth):

Attach the `trusted_corp` access level directly to the perimeter — users accessing Looker Studio from the corp network will satisfy the perimeter's access level check with their own identity.

### Pattern 4: Pub/Sub → Dataflow → BigQuery

Pub/Sub itself may not be restricted, but BigQuery is:

```yaml
# The Dataflow SA processing Pub/Sub messages and writing to BigQuery
# needs ingress for BigQuery (and GCS for temp/staging)
- ingressFrom:
    identities:
    - serviceAccount:pipeline-sa@PIPELINE_PROJECT.iam.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
  ingressTo:
    resources:
    - projects/DATA_PROJECT_NUMBER
    operations:
    - serviceName: bigquery.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: storage.googleapis.com
      methodSelectors:
      - method: '*'
```

### Pattern 5: Secret Manager Inside Perimeter — Cloud Run Access

If Cloud Run is in a **different project** outside the perimeter:

```yaml
- ingressFrom:
    identities:
    - serviceAccount:cloudrun-sa@CLOUDRUN_PROJECT.iam.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
  ingressTo:
    resources:
    - projects/DATA_PROJECT_NUMBER
    operations:
    - serviceName: secretmanager.googleapis.com
      methodSelectors:
      - method: google.cloud.secretmanager.v1.SecretManagerService.AccessSecretVersion
```

If Cloud Run is in the **same project** as Secret Manager → no ingress rule needed; the project is already inside the perimeter.

### Pattern 6: Terraform / CI-CD Pipeline

Terraform runners need access to multiple GCP management APIs:

```yaml
- ingressFrom:
    identities:
    - serviceAccount:terraform-runner@CICD_PROJECT.iam.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
  ingressTo:
    resources:
    - '*'
    operations:
    - serviceName: cloudresourcemanager.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: bigquery.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: storage.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: secretmanager.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: iam.googleapis.com
      methodSelectors:
      - method: '*'
    - serviceName: spanner.googleapis.com
      methodSelectors:
      - method: '*'
```

---

## 13. Audit Logs & Observability

### VPC SC Violation Log Structure

```json
{
  "protoPayload": {
    "@type": "type.googleapis.com/google.cloud.audit.AuditLog",
    "status": {
      "code": 7,
      "message": "Request is prohibited by organization's policy."
    },
    "authenticationInfo": {
      "principalEmail": "etl-sa@PROJECT_ID.iam.gserviceaccount.com"
    },
    "requestMetadata": {
      "callerIp": "203.0.113.42",
      "callerSuppliedUserAgent": "google-cloud-sdk"
    },
    "resourceName": "projects/DATA_PROJECT/datasets/my_dataset",
    "methodName": "google.cloud.bigquery.v2.JobService.InsertJob",
    "metadata": {
      "@type": "type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata",
      "vpcServiceControlsUniqueId": "abc123xyz",
      "resourceNames": [
        "accessPolicies/POLICY_ID/servicePerimeters/PROD_DATA_PERIMETER"
      ],
      "dryRun": false,
      "violationReason": "NO_MATCHING_ACCESS_LEVEL",
      "ingressViolations": [
        {
          "servicePerimeter": "accessPolicies/POLICY_ID/servicePerimeters/PROD_DATA_PERIMETER",
          "accessPolicyTitle": "Org Access Policy",
          "checkedAccessLevels": ["accessPolicies/POLICY_ID/accessLevels/trusted_corp"]
        }
      ]
    }
  }
}
```

### Key Log Fields

| Field | Description |
|---|---|
| `protoPayload.status.code` | `7` = PERMISSION_DENIED (VPC SC block) |
| `protoPayload.authenticationInfo.principalEmail` | Who made the blocked call |
| `protoPayload.requestMetadata.callerIp` | Source IP of the blocked request |
| `protoPayload.methodName` | Which API method was blocked |
| `protoPayload.resourceName` | Which resource was the target |
| `metadata.dryRun` | `true` = dry-run violation (not enforced); `false` = enforced block |
| `metadata.violationReason` | Why it was blocked (see below) |
| `metadata.ingressViolations` | Which perimeter(s) blocked the call and which access levels were checked |

### Violation Reason Codes

| `violationReason` | Meaning |
|---|---|
| `NO_MATCHING_ACCESS_LEVEL` | Caller's context doesn't satisfy any attached access level |
| `ACCESS_POLICY_NOT_MET` | General policy violation |
| `INVALID_CREDENTIAL` | Issue with the caller's credential |
| `SERVICE_NOT_ENABLED` | Service is restricted but API is not enabled |
| `FROM_OUTSIDE_PERIMETER` | Request originates from outside the perimeter without an ingress rule |
| `NO_MATCHING_INGRESS_POLICY` | No ingress rule matches the caller + source + method combination |

### Cloud Logging Queries

```bash
# ── All enforced VPC SC violations ────────────────────────────────────────────
resource.type="audited_resource"
protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
protoPayload.status.code!=0
NOT protoPayload.metadata.dryRun=true

# ── Dry-run violations only ───────────────────────────────────────────────────
resource.type="audited_resource"
protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
protoPayload.metadata.dryRun=true

# ── Violations for a specific perimeter ──────────────────────────────────────
resource.type="audited_resource"
protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
protoPayload.metadata.resourceNames=~"servicePerimeters/PROD_DATA_PERIMETER"

# ── Violations by a specific service account ──────────────────────────────────
resource.type="audited_resource"
protoPayload.authenticationInfo.principalEmail="etl-sa@PROJECT_ID.iam.gserviceaccount.com"
protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"

# ── Violations for a specific GCP service (BigQuery) ─────────────────────────
resource.type="audited_resource"
protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
protoPayload.serviceName="bigquery.googleapis.com"
protoPayload.status.code!=0

# ── All BCE + VPC SC combined denials ─────────────────────────────────────────
protoPayload.status.code=7
(
  protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
  OR protoPayload.serviceName="iap.googleapis.com"
)
```

### Log-Based Alert (gcloud)

```bash
# Create a log-based alert for any enforced VPC SC violation
gcloud logging metrics create vpc-sc-enforced-violations \
  --description="Count of enforced VPC SC violations" \
  --log-filter='
    resource.type="audited_resource"
    protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
    protoPayload.status.code!=0
    NOT protoPayload.metadata.dryRun=true
  '

# Create an alerting policy on this metric
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="VPC SC Enforced Violation Alert" \
  --condition-display-name="Any enforced VPC SC violation" \
  --condition-filter='metric.type="logging.googleapis.com/user/vpc-sc-enforced-violations"' \
  --condition-threshold-value=0 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-aggregations-per-series-aligner=ALIGN_COUNT \
  --condition-aggregations-alignment-period=60s \
  --combiner=OR
```

### BigQuery Export for SIEM

```bash
# Create a log sink to BigQuery for historical VPC SC analysis
gcloud logging sinks create vpc-sc-audit-sink \
  "bigquery.googleapis.com/projects/PROJECT_ID/datasets/vpc_sc_audit" \
  --log-filter='
    protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
  ' \
  --project=PROJECT_ID

# Grant the sink's SA write access to the dataset
# (gcloud outputs the sink SA — grant it roles/bigquery.dataEditor)
SINK_SA=$(gcloud logging sinks describe vpc-sc-audit-sink \
  --format="value(writerIdentity)")
bq add-iam-policy-binding \
  --member="$SINK_SA" \
  --role="roles/bigquery.dataEditor" \
  PROJECT_ID:vpc_sc_audit
```

---

## 14. Troubleshooting & Lockout Recovery

### Step-by-Step: Diagnosing a Violation

```
Step 1: Find the violation log entry
  Filter: protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
  Look for: status.code = 7

Step 2: Extract key fields from the log entry
  • principalEmail → WHO was blocked
  • callerIp → FROM WHERE
  • methodName → WHAT operation
  • resourceName → ON WHAT resource
  • metadata.resourceNames → WHICH perimeter blocked it
  • metadata.violationReason → WHY it was blocked

Step 3: Determine the fix
  violationReason = NO_MATCHING_ACCESS_LEVEL
    → Caller's IP/device doesn't satisfy any access level
    → Fix: Add an ingress rule for this SA + source combination

  violationReason = FROM_OUTSIDE_PERIMETER
    → Caller is outside the perimeter and doesn't satisfy any access level
    → Fix: Add an ingress rule, OR add the caller's project to the perimeter

  violationReason = NO_MATCHING_INGRESS_POLICY
    → Ingress rule exists but doesn't cover this specific method or resource
    → Fix: Expand the ingress rule's methodSelectors or resources

Step 4: Add the ingress/egress rule
  → Update ingress-policy.yaml with the new rule
  → Apply with: gcloud access-context-manager perimeters update ... --set-ingress-policies=...

Step 5: Verify the fix
  → Wait 5–10 minutes for propagation
  → Re-test the blocked operation
  → Confirm no new violation in logs
```

### Lockout Recovery Procedure

> **⚠️ Gotcha:** If you enforce a perimeter and lock out your Terraform runner or Cloud Console access, you need org-level ACM admin rights to recover. Always keep a break-glass org admin account that's exempt from all perimeter restrictions.

```bash
# Emergency: Switch enforced perimeter back to dry-run (stops blocking)
# This copies the enforced config to a dry-run spec and removes enforcement
gcloud access-context-manager perimeters update PROD_DATA_PERIMETER \
  --policy=$POLICY_ID \
  --use-explicit-dry-run-spec

# OR: Delete the perimeter entirely (nuclear option — only for total lockout)
gcloud access-context-manager perimeters delete PROD_DATA_PERIMETER \
  --policy=$POLICY_ID

# After recovery: add the missing ingress rule, then re-enforce
cat >> ingress-policy.yaml << 'EOF'
- ingressFrom:
    identities:
    - serviceAccount:LOCKED_OUT_SA@PROJECT_ID.iam.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/trusted_corp
  ingressTo:
    resources:
    - '*'
    operations:
    - serviceName: BLOCKED_SERVICE.googleapis.com
      methodSelectors:
      - method: '*'
EOF

gcloud access-context-manager perimeters update PROD_DATA_PERIMETER \
  --policy=$POLICY_ID \
  --set-ingress-policies=ingress-policy.yaml

# Re-enforce after fixing
gcloud access-context-manager perimeters dry-run enforce PROD_DATA_PERIMETER \
  --policy=$POLICY_ID
```

### VPC SC Policy Troubleshooter

In Cloud Console: **VPC Service Controls → [Perimeter] → Troubleshoot**

The troubleshooter takes: principal email + source IP + service + method → simulates the evaluation and shows which rule would allow/deny.

### Troubleshooting Reference Table

| Issue | Likely Cause | Diagnosis | Fix |
|---|---|---|---|
| SA gets `PERMISSION_DENIED` on BigQuery | SA's project is outside the perimeter | Filter logs for SA email + `VpcServiceControlAuditMetadata` | Add ingress rule for the SA with BigQuery methods |
| Cloud Build pipeline fails on GCS access | Cloud Build SA not covered by any ingress rule | Filter logs for `PROJECT_NUMBER@cloudbuild.gserviceaccount.com` | Add ingress for Cloud Build SA with `storage.googleapis.com` |
| Terraform `apply` fails for perimeter resources | Terraform runner SA missing ingress; `SetIamPolicy` or `Create` blocked | Check violation `methodName` in logs | Add ingress for Terraform SA with `cloudresourcemanager.googleapis.com` + target services |
| VMs cannot reach BigQuery API (internal call) | `vpcAccessibleServices` too restrictive; BigQuery not in allowedServices | Check if `bigquery.googleapis.com` is in `vpc_accessible_services.allowed_services` | Add `bigquery.googleapis.com` to `allowedServices` |
| Dry-run shows zero violations but enforcement blocks | Access level propagation timing; level not yet active | Check access level `updateTime`; compare to enforcement time | Wait 10+ min after access level creation before enforcing |
| Cross-project Dataflow job blocked | Dataflow worker project not in perimeter `resources` | Check worker project number in violation log `callerIp` or `resourceName` | Add Dataflow worker project to perimeter `resources` OR add ingress rule for Dataflow SA |
| Secret Manager access blocked from Cloud Run | Cloud Run project is outside the perimeter | Check `principalEmail` = Cloud Run SA; `serviceName` = `secretmanager.googleapis.com` | Add ingress for Cloud Run SA with `secretmanager.googleapis.com` AccessSecretVersion method |
| Cloud Console blocked from viewing BigQuery | User's browser doesn't satisfy access level | User IP not in access level or device not enrolled | Ensure corp network + Endpoint Verification; use VPN if needed |
| Cloud Composer DAG fails on BigQuery | Composer SA and Composer worker project outside perimeter | Filter logs for Composer service account | Add ingress for Composer SA + `service-PROJECT_NUMBER@cloudcomposer-accounts.iam.gserviceaccount.com` |
| `gcloud storage cp` blocked even with access level | Using public googleapis.com endpoint; VPC SC sees public IP | Check `callerIp` in violation log — it's a Google IP, not your IP | Configure DNS → `restricted.googleapis.com`; use private Google access on the VM |

---

## 15. Security Hardening & Best Practices

### Enforcement Approach

- **Always start with dry-run** — minimum 48–72 hours of observation; longer for complex environments
- **Enforce incrementally** — restrict one service at a time; add `bigquery.googleapis.com` first, then `storage.googleapis.com`, then `secretmanager.googleapis.com`
- **Test in non-prod first** — apply the exact same perimeter config to a staging environment and run integration tests before enforcing in production
- **Never remove and re-add** — deleting a perimeter removes all protection instantly; update in place instead

### Ingress/Egress Rule Design

- **Never use `ANY_IDENTITY`** in production ingress rules — it nullifies the perimeter for the specified methods
- **Always use `sources`** in `ingressFrom` — combine identity with an access level or VPC network source; identity alone is not enough
- **Minimize method wildcards** — prefer specific method selectors over `method: '*'`; use `'*'` only for administrative SAs (Terraform, Cloud Build)
- **Document every rule** — add `title` and `description` to each ingress/egress rule block; rules become opaque otherwise
- **Audit ingress rules quarterly** — remove rules for decommissioned SAs; stale rules are a standing door in the perimeter

### Defense in Depth

- **Enable `vpcAccessibleServices`** alongside `restrictedServices` — they protect different attack surfaces
- **Combine with IAM** — apply least-privilege IAM roles within the perimeter; VPC SC + overly permissive IAM = weak posture
- **Layer with Cloud Armor** — for web-accessible applications, use Cloud Armor + IAP + VPC SC together
- **Use KMS for data-at-rest** — VPC SC protects API-level access; KMS protects stored data from raw storage access

### Infrastructure as Code

```hcl
# Always protect perimeters from accidental Terraform destroy
resource "google_access_context_manager_service_perimeter" "prod" {
  lifecycle {
    prevent_destroy = true
    # Ignore access_level propagation timing issues during plan
    ignore_changes = [status[0].access_levels]
  }
  # ...
}
```

- **Pin ACM Terraform provider version** — ACM resources are sensitive to provider API changes
- **Use separate Terraform state for ACM resources** — keep perimeter config in its own state file with restricted backend access
- **Never store access policy IDs in plaintext** — use Secret Manager or Terraform variables with `.tfvars` files excluded from VCS

### Observability

- **Export all VPC SC logs to BigQuery** — use a log sink with 365-day retention for compliance
- **Alert on any enforced violation** — zero violations in steady state is the goal; every violation warrants investigation
- **Alert on perimeter config changes** — monitor `accesscontextmanager.googleapis.com` admin activity logs for unauthorized perimeter modifications

---

## 16. Limits & Quotas

| Resource | Limit | Notes |
|---|---|---|
| Access Policies per Organization | 1 org-level + N scoped | Scoped policies: no documented hard limit |
| Service Perimeters per Policy | 100 | Includes both regular and bridge perimeters |
| Projects per Service Perimeter | 1,000 | Across all entries in `resources` |
| Restricted Services per Perimeter | 100 | Only services in the supported list have effect |
| Ingress Policies per Perimeter | 100 | Each entry in the `ingressPolicies` array |
| Egress Policies per Perimeter | 100 | Each entry in the `egressPolicies` array |
| Identities per Ingress/Egress Rule | 100 | Named identities in the `identities` list |
| Method Selectors per Operation | 100 | `methodSelectors` entries per `operations` block |
| Access Levels per Policy | 1,000 | Both basic and custom levels |
| Bridge Perimeters per Policy | 10 | |
| VPC Accessible Services per Perimeter | 100 | Entries in `allowedServices` |
| ACM API propagation delay | ~5–10 minutes | After create/update; allow before enforcing |
| Scoped access policies per folder/project | 1 per scope | |
| Access bindings (CAA) per policy | 500 | Per group + access level pair |

---

## 17. Common Gotchas

**Unsupported services are silently unprotected.** Adding `translate.googleapis.com` to `restrictedServices` has no effect if it's not on the VPC SC supported services list. Always verify service support before building compliance posture around it.

**Shared VPC host project omission.** The single most common lockout cause. VM networking flows through the host project — if it's outside the perimeter, VMs look like external callers.

**VPC SC violations log to the target project, not the caller's project.** If `etl-sa@project-a` is blocked calling BigQuery in `project-b`, the violation log appears in `project-b`'s logs. Many teams look in the wrong project.

**`ANY_IDENTITY` is an open door.** It matches all service accounts AND all user accounts. Even scoping it to specific methods gives any authenticated identity those methods.

**Bridge perimeters don't enforce access levels.** If you bridge two perimeters expecting access level enforcement between them, that enforcement is absent. Use ingress/egress rules on the regular perimeters instead.

**Dry-run mode provides zero protection.** Violations are logged but all requests are allowed through. Never treat a dry-run perimeter as a security control.

**IAM `resourcemanager.projects.get` is often blocked.** Terraform and `gcloud` frequently call `cloudresourcemanager.googleapis.com` for project metadata. If this API is in `restrictedServices`, Terraform plan/apply fails even for non-data operations. Add it to ingress rules for all CI/CD SAs.

**Cloud Console requires access level satisfaction.** Humans browsing BigQuery datasets in the Cloud Console from outside the corp network (e.g., from home without VPN) will get `PERMISSION_DENIED` even with correct IAM roles.

**Method selector format varies by service.** Some services use `google.storage.objects.get` format; others use `google.cloud.bigquery.v2.JobService.InsertJob`. Verify exact method names in the [GCP service proto definitions](https://github.com/googleapis/googleapis) or use `method: '*'` initially, then narrow down.

**Google-managed service agent SAs need ingress rules too.** Dataflow, Composer, Vertex AI, Cloud Build all use Google-managed service agents (different from user-created SAs) that also need ingress rules when running in projects outside the perimeter.

**Egress rules block writes to external buckets even with IAM access.** A SA with `roles/storage.objectCreator` on an external bucket will be blocked by VPC SC if the source project is inside a perimeter without an egress rule. IAM says "allowed"; VPC SC says "no".

**Perimeter config changes are not instant.** After updating `--set-ingress-policies`, there is a ~2–5 minute propagation delay. Automated tests run immediately after `gcloud ... update` will still see the old behavior.

---

## 18. Quick Reference — gcloud Commands

```bash
# ── Access Policy ─────────────────────────────────────────────────────────────

# Get the org-level access policy number
gcloud access-context-manager policies list \
  --organization=ORG_ID \
  --format="value(name)"

# Create a scoped access policy
gcloud access-context-manager policies create \
  --organization=ORG_ID \
  --scopes=projects/PROJECT_ID \
  --title="Team Policy"

# ── Perimeters ────────────────────────────────────────────────────────────────

# List all perimeters
gcloud access-context-manager perimeters list --policy=POLICY_ID

# Describe a perimeter (full YAML)
gcloud access-context-manager perimeters describe PERIMETER_NAME \
  --policy=POLICY_ID --format=yaml

# Create perimeter in dry-run mode
gcloud access-context-manager perimeters dry-run create PERIMETER_NAME \
  --policy=POLICY_ID \
  --title="Perimeter Title" \
  --type=PERIMETER_TYPE_REGULAR \
  --resources=projects/PROJECT_NUMBER \
  --restricted-services=bigquery.googleapis.com,storage.googleapis.com

# Describe dry-run config
gcloud access-context-manager perimeters dry-run describe PERIMETER_NAME \
  --policy=POLICY_ID

# List dry-run configs for all perimeters
gcloud access-context-manager perimeters dry-run list --policy=POLICY_ID

# Enforce a dry-run perimeter (promotes dry-run → enforced)
gcloud access-context-manager perimeters dry-run enforce PERIMETER_NAME \
  --policy=POLICY_ID

# Delete the dry-run config only (keep enforcement)
gcloud access-context-manager perimeters dry-run delete PERIMETER_NAME \
  --policy=POLICY_ID

# Add a project to an existing enforced perimeter
gcloud access-context-manager perimeters update PERIMETER_NAME \
  --policy=POLICY_ID \
  --add-resources=projects/NEW_PROJECT_NUMBER

# Remove a project from a perimeter
gcloud access-context-manager perimeters update PERIMETER_NAME \
  --policy=POLICY_ID \
  --remove-resources=projects/OLD_PROJECT_NUMBER

# Add a restricted service
gcloud access-context-manager perimeters update PERIMETER_NAME \
  --policy=POLICY_ID \
  --add-restricted-services=pubsub.googleapis.com

# Remove a restricted service
gcloud access-context-manager perimeters update PERIMETER_NAME \
  --policy=POLICY_ID \
  --remove-restricted-services=compute.googleapis.com

# Set ingress policies (replaces all existing ingress rules)
gcloud access-context-manager perimeters update PERIMETER_NAME \
  --policy=POLICY_ID \
  --set-ingress-policies=ingress-policy.yaml

# Set egress policies (replaces all existing egress rules)
gcloud access-context-manager perimeters update PERIMETER_NAME \
  --policy=POLICY_ID \
  --set-egress-policies=egress-policy.yaml

# Clear all ingress policies
gcloud access-context-manager perimeters update PERIMETER_NAME \
  --policy=POLICY_ID \
  --clear-ingress-policies

# Clear all egress policies
gcloud access-context-manager perimeters update PERIMETER_NAME \
  --policy=POLICY_ID \
  --clear-egress-policies

# Emergency rollback: switch enforced → dry-run (stops blocking)
gcloud access-context-manager perimeters update PERIMETER_NAME \
  --policy=POLICY_ID \
  --use-explicit-dry-run-spec

# Delete a perimeter entirely
gcloud access-context-manager perimeters delete PERIMETER_NAME \
  --policy=POLICY_ID

# ── Access Levels ─────────────────────────────────────────────────────────────

# List all access levels
gcloud access-context-manager levels list --policy=POLICY_ID

# Describe an access level
gcloud access-context-manager levels describe LEVEL_NAME \
  --policy=POLICY_ID --format=yaml

# Create a basic access level from YAML spec
gcloud access-context-manager levels create LEVEL_NAME \
  --policy=POLICY_ID \
  --title="Level Title" \
  --basic-level-spec=conditions.yaml

# Delete an access level
gcloud access-context-manager levels delete LEVEL_NAME \
  --policy=POLICY_ID
```

---

## 19. Decision Tree

```
START: I need to protect GCP data or API access
│
├─► Is this a network-level threat (unauthorized TCP/UDP connections)?
│     → Use VPC Firewall Rules (NOT VPC SC — VPC SC is API-layer, not network-layer)
│
├─► Is this about protecting GCP APIs from unauthorized callers?
│     → Use VPC Service Controls (service perimeter)
│     │
│     ├─► Do I also need to protect web application access (not just APIs)?
│     │     → Add Cloud IAP in front of the application
│     │       VPC SC protects the data APIs; IAP protects the app frontend
│     │
│     └─► Do I also need VM network-level isolation?
│           → VPC SC + VPC Firewall Rules are complementary; use both
│             VPC SC: controls which APIs VMs can call
│             Firewall: controls which VMs can talk to each other
│
├─► Which perimeter type?
│     │
│     ├─► Do I need a hard boundary that blocks unauthorized access?
│     │     → PERIMETER_TYPE_REGULAR (use for all production perimeters)
│     │
│     └─► Do I need two regular perimeters to exchange data without merging them?
│           → PERIMETER_TYPE_BRIDGE
│             ⚠️ Bridge does NOT enforce access levels — use ingress/egress rules
│             on the regular perimeters instead where possible
│
├─► How to allow external access to the perimeter?
│     │
│     ├─► Should ALL callers from a trusted network be allowed?
│     │     → Attach an Access Level to the perimeter
│     │       (blanket pass for any identity satisfying the conditions)
│     │
│     ├─► Should only SPECIFIC service accounts be allowed?
│     │     → Use Ingress Rules with explicit identities + source access level
│     │       (targeted pass: SA + network context + specific methods)
│     │
│     └─► Should BOTH apply?
│           → Use Access Levels for human users (corp network + device)
│             AND Ingress Rules for service accounts (explicit SA + method scope)
│
├─► Which services should I restrict first?
│     Priority order (highest data exfiltration risk first):
│     1. storage.googleapis.com (GCS — direct data access)
│     2. bigquery.googleapis.com (BQ — analytical data)
│     3. spanner.googleapis.com (transactional data)
│     4. secretmanager.googleapis.com (credentials)
│     5. cloudkms.googleapis.com (encryption keys)
│     6. pubsub.googleapis.com (data streaming)
│     7. aiplatform.googleapis.com (ML data + models)
│     8. artifactregistry.googleapis.com (container images, packages)
│
├─► Should I also use `vpcAccessibleServices`?
│     YES if ANY of these apply:
│     ✅ You have VMs inside the perimeter's VPC
│     ✅ You want defense-in-depth (compromised VM can't call unrestricted APIs)
│     ✅ Compliance requires restricting ALL API calls from perimeter VMs
│     ✅ You have sensitive data and want maximum blast radius reduction
│
│     NO if:
│     ❌ No VMs inside the perimeter (serverless-only architecture)
│     ❌ VMs need to call a wide variety of APIs and allowlist management is impractical
│
└─► Summary: Which controls should I combine?

    Scenario                              → Controls
    ─────────────────────────────────────────────────────────────────────
    Protect GCP data APIs from exfil      → VPC SC (regular perimeter)
    Add human access control (device/net) → + Access Levels (ACM)
    Allow specific pipeline SAs           → + Ingress Rules
    Allow data export to archive          → + Egress Rules
    Restrict VM API access too            → + vpcAccessibleServices
    Protect web app frontend              → + Cloud IAP
    WAF + DDoS protection                 → + Cloud Armor (at LB)
    Full Zero Trust posture               → All of the above + BeyondCorp Enterprise
```

### Summary Table

| Requirement | Solution |
|---|---|
| Block external access to BigQuery/GCS APIs | VPC SC `restrictedServices` |
| Allow specific SA from outside perimeter | Ingress Rule with explicit identity |
| Allow data export to external project | Egress Rule with specific project + method |
| Restrict VMs to only needed APIs | `vpcAccessibleServices` with allowlist |
| Allow human users from corp network | Access Level on perimeter (IP + device) |
| Share data between two perimeters | Bridge Perimeter OR ingress/egress rules |
| Test before enforcing | Dry-run mode (48–72hr minimum) |
| Protect web app AND data APIs | Cloud IAP + VPC SC |
| Protect network traffic too | VPC SC + VPC Firewall Rules |
| Full Zero Trust | BeyondCorp Enterprise (IAP + ACM + VPC SC + Chrome) |
