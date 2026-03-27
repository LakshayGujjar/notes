# 🔐 GCP Cloud IAM — Comprehensive Cheatsheet

> **Audience:** Security engineers, platform engineers, DevOps engineers, and architects managing access control on GCP.
> **Last updated:** March 2026 | Covers IAM policies, roles, service accounts, Workload Identity Federation, Org Policy, Access Context Manager, and security governance.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Principals & Identities](#2-principals--identities)
3. [Roles](#3-roles)
4. [IAM Policies & Bindings](#4-iam-policies--bindings)
5. [IAM Conditions](#5-iam-conditions)
6. [Deny Policies](#6-deny-policies)
7. [Service Accounts Deep Dive](#7-service-accounts-deep-dive)
8. [Workload Identity Federation](#8-workload-identity-federation)
9. [Organization Policy Service](#9-organization-policy-service)
10. [Access Context Manager & VPC Service Controls](#10-access-context-manager--vpc-service-controls)
11. [IAM Recommender & Policy Intelligence](#11-iam-recommender--policy-intelligence)
12. [Audit Logging](#12-audit-logging)
13. [IAM Best Practices & Security Patterns](#13-iam-best-practices--security-patterns)
14. [gcloud CLI & REST API Quick Reference](#14-gcloud-cli--rest-api-quick-reference)
15. [Pricing Summary](#15-pricing-summary)
16. [Quick Reference & Comparison Tables](#16-quick-reference--comparison-tables)

---

## 1. Overview & Key Concepts

### What is Cloud IAM?

**Cloud IAM** answers one question: **"Who (principal) can do what (role/permission) on which resource?"** It is GCP's unified access control system — every API call to a GCP service is evaluated against IAM policies.

**Core IAM model:**

```
Principal  +  Role  →  Binding  →  Policy  →  Resource
(who)         (what)    (pair)       (set)      (on what)

Example:
  user:alice@company.com  +  roles/storage.objectViewer  →
  binding on  projects/my-project/buckets/my-bucket
```

---

### IAM vs. Other GCP Access Controls

| Mechanism | Controls | Scope | When to Use |
|---|---|---|---|
| **IAM** | Who can act on resources | All GCP resources | Primary access control — always |
| **ACLs** (Cloud Storage) | Object-level access | GCS buckets/objects | Legacy; prefer IAM (uniform bucket-level access) |
| **VPC Firewall Rules** | Network traffic (IP, port, protocol) | VM network interfaces | Network-layer access between workloads |
| **Organization Policy** | What configurations are allowed | Org/folder/project | Guardrails on what can be created, regardless of IAM |
| **VPC Service Controls** | API access from trusted networks only | GCP APIs | Data exfiltration prevention |

---

### Resource Hierarchy & Policy Inheritance

```
Organization (org level policies)
    │
    ├── Folder: Production
    │       ├── Project: prod-app     ← inherits org + production folder policies
    │       └── Project: prod-data   ← inherits org + production folder policies
    │
    └── Folder: Development
            └── Project: dev-app     ← inherits org + development folder policies

Inheritance rules:
  ✅ Child inherits all ALLOW bindings from parent (additive only)
  ❌ Parent cannot DENY what a child has allowed (no "override upward")
  ✅ Deny policies ARE inherited and take precedence over allow policies
  ❌ You CANNOT remove inherited bindings from a child resource
```

> ⚠️ **Key insight:** IAM inheritance only ADDS permissions — a binding at the org level grants access to all resources beneath it. This is why the principle of least privilege is critical at higher levels.

---

### IAM Policy Evaluation Flow

```
API Request arrives
        │
        ▼
1. Is there a DENY POLICY that matches? ──YES──► ACCESS DENIED 🚫
        │NO
        ▼
2. Is there an ALLOW POLICY binding that matches?
   (Check org → folder → project → resource, union of all)
        │
        ├── YES, with IAM Condition → Does condition evaluate TRUE?
        │         YES → ACCESS GRANTED ✅
        │         NO  → continue to next binding
        │
        ├── YES, no condition → ACCESS GRANTED ✅
        │
        └── NO matching binding → IMPLICIT DENY 🚫
```

---

### Key Terminology

| Term | Definition |
|---|---|
| **Principal** | An identity that can be granted access (user, SA, group, domain) |
| **Role** | A named collection of permissions |
| **Permission** | The ability to perform a specific action (e.g., `storage.objects.get`) |
| **Binding** | Associates a principal with a role (optionally with a condition) |
| **Policy** | A collection of bindings attached to a resource |
| **Condition** | A CEL expression that restricts when a binding applies |
| **Service Account** | A machine identity for applications and workloads |
| **Audit Log** | Record of who did what, when, and where |

---

### Key Limits & Quotas

| Resource | Limit |
|---|---|
| Max bindings per IAM policy | 1,500 |
| Max members per binding | 1,500 |
| Max policy size | 64 KB |
| Max custom roles per org | 300 |
| Max custom roles per project | 300 |
| Max service accounts per project | 100 (default, requestable) |
| Max service account keys per SA | 10 |

---

## 2. 🔑 Principals & Identities

### Principal Types

| Type | Identifier Format | Use Case |
|---|---|---|
| **Google Account** | `user:alice@gmail.com` | Individual human users |
| **Service Account** | `serviceAccount:name@proj.iam.gserviceaccount.com` | Applications, workloads |
| **Google Group** | `group:dev-team@company.com` | Team access management |
| **Workspace Domain** | `domain:company.com` | All users in a Workspace domain |
| **Cloud Identity Domain** | `domain:company.com` | Cloud Identity users |
| **allAuthenticatedUsers** | `allAuthenticatedUsers` | Any Google-authenticated user |
| **allUsers** | `allUsers` | Anyone, including unauthenticated |

> 🚨 **Never use `allUsers` or `allAuthenticatedUsers` for sensitive resources.** These grant access to the entire internet or all Google accounts respectively. Only use for truly public content (e.g., public GCS websites).

---

### Service Account Types

| Type | Created By | Naming | Control |
|---|---|---|---|
| **User-managed** | You (gcloud/Console) | `name@project.iam.gserviceaccount.com` | Full control — recommended |
| **Default (Compute Engine)** | GCP automatically | `PROJECT_NUM-compute@developer.gserviceaccount.com` | Avoid — over-privileged |
| **Default (App Engine)** | GCP automatically | `PROJECT_ID@appspot.gserviceaccount.com` | Avoid — over-privileged |
| **Google-managed** | GCP services | `service-PROJECT_NUM@*.iam.gserviceaccount.com` | Do not modify |

> ⚠️ The **Compute Engine default SA** has `roles/editor` on the project by default — this is dangerously over-privileged. Always create a dedicated SA per workload with minimum required roles.

---

### gcloud: Service Account Management

```bash
# Create a service account
gcloud iam service-accounts create my-app-sa \
  --display-name="My Application Service Account" \
  --description="SA for my-app running in Cloud Run" \
  --project=MY_PROJECT

# List service accounts
gcloud iam service-accounts list \
  --project=MY_PROJECT \
  --format="table(email,displayName,disabled)"

# Describe a service account
gcloud iam service-accounts describe \
  my-app-sa@MY_PROJECT.iam.gserviceaccount.com

# Disable a service account (soft-disable, reversible)
gcloud iam service-accounts disable \
  my-app-sa@MY_PROJECT.iam.gserviceaccount.com

# Enable a service account
gcloud iam service-accounts enable \
  my-app-sa@MY_PROJECT.iam.gserviceaccount.com

# Delete a service account (irreversible after 30 days)
gcloud iam service-accounts delete \
  my-app-sa@MY_PROJECT.iam.gserviceaccount.com

# Create a service account key (⚠️ avoid if possible)
gcloud iam service-accounts keys create key.json \
  --iam-account=my-app-sa@MY_PROJECT.iam.gserviceaccount.com

# List keys for a service account
gcloud iam service-accounts keys list \
  --iam-account=my-app-sa@MY_PROJECT.iam.gserviceaccount.com \
  --format="table(name.basename(),validAfterTime,validBeforeTime,keyType)"

# Delete a key
gcloud iam service-accounts keys delete KEY_ID \
  --iam-account=my-app-sa@MY_PROJECT.iam.gserviceaccount.com

# Grant IAM role to a service account (on a project)
gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="serviceAccount:my-app-sa@MY_PROJECT.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
```

---

### Workload Identity for GKE

```bash
# Enable Workload Identity on a GKE cluster
gcloud container clusters update my-cluster \
  --workload-pool=MY_PROJECT.svc.id.goog \
  --region=us-central1

# Enable on a node pool
gcloud container node-pools update default-pool \
  --cluster=my-cluster \
  --workload-metadata=GKE_METADATA \
  --region=us-central1

# Bind K8s SA to GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  my-app-sa@MY_PROJECT.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:MY_PROJECT.svc.id.goog[NAMESPACE/KSA_NAME]"

# Annotate the Kubernetes SA (in your K8s manifest)
kubectl annotate serviceaccount KSA_NAME \
  --namespace=NAMESPACE \
  iam.gke.io/gcp-service-account=my-app-sa@MY_PROJECT.iam.gserviceaccount.com
```

---

### Python: Service Account Management

```python
from google.cloud import iam_admin_v1
from google.iam.admin.v1 import iam_pb2

client = iam_admin_v1.IAMClient()
PROJECT = "my-project"

# Create a service account
sa = client.create_service_account(
    request=iam_admin_v1.CreateServiceAccountRequest(
        name     = f"projects/{PROJECT}",
        account_id = "my-app-sa",
        service_account = iam_admin_v1.ServiceAccount(
            display_name = "My App Service Account",
            description  = "SA for my-app workload",
        ),
    )
)
print(f"Created SA: {sa.email}")

# List service accounts
sas = client.list_service_accounts(
    request=iam_admin_v1.ListServiceAccountsRequest(name=f"projects/{PROJECT}")
)
for account in sas:
    status = "DISABLED" if account.disabled else "active"
    print(f"  [{status}] {account.email}")

# Disable a service account
client.disable_service_account(
    request=iam_admin_v1.DisableServiceAccountRequest(
        name=f"projects/{PROJECT}/serviceAccounts/my-app-sa@{PROJECT}.iam.gserviceaccount.com"
    )
)
```

---

## 3. 🛡️ Roles

### Role Types

| Type | Naming | # of Permissions | Use in Production? |
|---|---|---|---|
| **Basic: Owner** | `roles/owner` | ~15,000+ | ❌ Never |
| **Basic: Editor** | `roles/editor` | ~10,000+ | ❌ Never |
| **Basic: Viewer** | `roles/viewer` | ~5,000+ | ❌ Avoid |
| **Predefined** | `roles/service.action` | Curated set | ✅ Preferred |
| **Custom** | `projects/P/roles/MyRole` | You define | ✅ For fine-grained control |

> ⚠️ **Basic roles grant access to ALL GCP services.** Granting `roles/editor` to a service account lets it read and write to BigQuery, GCS, Cloud SQL, and hundreds of other services. **Never use basic roles in production.**

---

### Predefined Role Naming Convention

```
roles/{service}.{role_type}

Examples:
  roles/storage.objectViewer          # Read GCS objects
  roles/storage.objectCreator         # Create GCS objects
  roles/bigquery.dataViewer           # Read BQ tables
  roles/bigquery.jobUser              # Run BQ queries
  roles/container.developer           # GKE developer
  roles/run.invoker                   # Invoke Cloud Run services
  roles/cloudsql.client               # Connect to Cloud SQL
  roles/pubsub.publisher              # Publish to Pub/Sub
  roles/secretmanager.secretAccessor  # Access secrets
  roles/logging.logWriter             # Write logs
```

---

### Custom Role YAML

```yaml
# custom-role.yaml
title:       "App Storage Reader"
description: "Read-only access to application GCS buckets"
stage:       "GA"                      # ALPHA | BETA | GA
includedPermissions:
  - storage.buckets.get
  - storage.objects.get
  - storage.objects.list
  - storage.objects.getIamPolicy
```

---

### gcloud: Roles

```bash
# List all predefined roles for a service
gcloud iam roles list \
  --filter="name:roles/storage" \
  --format="table(name,title)"

# Describe a predefined role (see its permissions)
gcloud iam roles describe roles/storage.objectViewer

# List testable permissions on a resource
gcloud iam list-testable-permissions \
  //cloudresourcemanager.googleapis.com/projects/MY_PROJECT \
  --format="table(name,stage)"

# Create a custom role from YAML (project-level)
gcloud iam roles create AppStorageReader \
  --project=MY_PROJECT \
  --file=custom-role.yaml

# Create a custom role at org level
gcloud iam roles create AppStorageReader \
  --organization=ORG_ID \
  --file=custom-role.yaml

# List custom roles
gcloud iam roles list \
  --project=MY_PROJECT \
  --filter="name:projects/MY_PROJECT" \
  --format="table(name,title,stage)"

# Describe a custom role
gcloud iam roles describe AppStorageReader \
  --project=MY_PROJECT

# Update a custom role (add a permission)
gcloud iam roles update AppStorageReader \
  --project=MY_PROJECT \
  --add-permissions=storage.objects.create

# Copy a predefined role as starting point for custom role
gcloud iam roles copy \
  --source=roles/storage.objectViewer \
  --destination=MyCustomViewer \
  --dest-project=MY_PROJECT

# Delete a custom role
gcloud iam roles delete AppStorageReader --project=MY_PROJECT
```

---

### Python: Custom Roles

```python
from google.cloud import iam_admin_v1
from google.iam.admin.v1 import iam_pb2

client = iam_admin_v1.IAMClient()
PROJECT = "my-project"

# Create a custom role
role = client.create_role(
    request=iam_admin_v1.CreateRoleRequest(
        parent  = f"projects/{PROJECT}",
        role_id = "AppStorageReader",
        role    = iam_admin_v1.Role(
            title       = "App Storage Reader",
            description = "Read-only access to application GCS buckets",
            stage       = iam_admin_v1.Role.RoleLaunchStage.GA,
            included_permissions = [
                "storage.buckets.get",
                "storage.objects.get",
                "storage.objects.list",
            ],
        ),
    )
)
print(f"Created role: {role.name}")

# List custom roles for a project
roles = client.list_roles(
    request=iam_admin_v1.ListRolesRequest(
        parent            = f"projects/{PROJECT}",
        show_deleted      = False,
        view              = iam_admin_v1.RoleView.FULL,
    )
)
for r in roles:
    print(f"  {r.name}: {r.title} ({len(r.included_permissions)} permissions)")

# Update role — add a permission
from google.protobuf import field_mask_pb2

updated_role = client.update_role(
    request=iam_admin_v1.UpdateRoleRequest(
        name = f"projects/{PROJECT}/roles/AppStorageReader",
        role = iam_admin_v1.Role(
            included_permissions = [
                "storage.buckets.get",
                "storage.objects.get",
                "storage.objects.list",
                "storage.objects.create",   # New permission
            ]
        ),
        update_mask = field_mask_pb2.FieldMask(paths=["included_permissions"]),
    )
)
print(f"Updated role: {updated_role.name}")
```

---

## 4. 📊 IAM Policies & Bindings

### Policy JSON Structure

```json
{
  "version": 3,
  "etag": "BwXXxx_eTAg=",
  "bindings": [
    {
      "role": "roles/storage.objectViewer",
      "members": [
        "user:alice@company.com",
        "serviceAccount:my-app-sa@proj.iam.gserviceaccount.com",
        "group:data-readers@company.com"
      ]
    },
    {
      "role": "roles/bigquery.dataEditor",
      "members": [
        "serviceAccount:etl-sa@proj.iam.gserviceaccount.com"
      ],
      "condition": {
        "title":       "Prod data access",
        "description": "Only during business hours UTC",
        "expression":  "request.time.getHours('UTC') >= 9 && request.time.getHours('UTC') <= 17"
      }
    }
  ],
  "auditConfigs": [
    {
      "service": "storage.googleapis.com",
      "auditLogConfigs": [
        {"logType": "DATA_READ"},
        {"logType": "DATA_WRITE"}
      ]
    }
  ]
}
```

**Policy version rules:**

| Version | Conditions | Required When |
|---|---|---|
| `1` | ❌ Not supported | No conditions needed |
| `2` | Reserved | (do not use) |
| `3` | ✅ Supported | Any conditional binding |

---

### gcloud: Policy Operations

```bash
# ── READ policies ─────────────────────────────────────────────────────
# Get project IAM policy
gcloud projects get-iam-policy MY_PROJECT \
  --format=json > policy.json

# Get folder IAM policy
gcloud resource-manager folders get-iam-policy FOLDER_ID

# Get org IAM policy
gcloud organizations get-iam-policy ORG_ID

# Get resource-level policy (e.g., GCS bucket)
gcloud storage buckets get-iam-policy gs://my-bucket

# Get BigQuery dataset policy
bq get-iam-policy my-project:my_dataset

# ── ADD bindings (additive — does not replace existing) ───────────────
# Grant a user a role on a project
gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="user:alice@company.com" \
  --role="roles/bigquery.dataViewer"

# Grant a SA a role on a folder
gcloud resource-manager folders add-iam-policy-binding FOLDER_ID \
  --member="serviceAccount:sa@proj.iam.gserviceaccount.com" \
  --role="roles/logging.logWriter"

# Grant a role on a specific GCS bucket
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="serviceAccount:reader-sa@proj.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# ── REMOVE bindings ────────────────────────────────────────────────────
gcloud projects remove-iam-policy-binding MY_PROJECT \
  --member="user:alice@company.com" \
  --role="roles/bigquery.dataViewer"

# ── SET policy (replaces entire policy — use with etag!) ─────────────
# ⚠️ DANGEROUS: replaces ALL bindings. Always use etag for safety.
gcloud projects set-iam-policy MY_PROJECT policy.json

# ── TROUBLESHOOT why a principal can/cannot do something ─────────────
gcloud policy-troubleshoot iam \
  //storage.googleapis.com/projects/MY_PROJECT/buckets/my-bucket \
  --principal-email=alice@company.com \
  --permission=storage.objects.get

# ── SIMULATE policy change before applying ────────────────────────────
gcloud policy-intelligence policy-simulate \
  --resource=//cloudresourcemanager.googleapis.com/projects/MY_PROJECT \
  --delta-file=delta.json
```

---

### Python: Policy Management

```python
from google.cloud import resourcemanager_v3
from google.iam.v1 import iam_policy_pb2, policy_pb2

client = resourcemanager_v3.ProjectsClient()
PROJECT_ID   = "my-project"
project_name = f"projects/{PROJECT_ID}"

# Get current IAM policy
get_request = iam_policy_pb2.GetIamPolicyRequest(
    resource = project_name,
    options  = iam_policy_pb2.GetPolicyOptions(requested_policy_version=3),
)
policy = client.get_iam_policy(request=get_request)
print(f"Policy version: {policy.version}, etag: {policy.etag}")
for binding in policy.bindings:
    print(f"  Role: {binding.role}")
    for member in binding.members:
        print(f"    {member}")

# Add a binding and update policy
new_binding = policy_pb2.Binding(
    role    = "roles/storage.objectViewer",
    members = ["serviceAccount:new-sa@my-project.iam.gserviceaccount.com"],
)
policy.bindings.append(new_binding)

set_request = iam_policy_pb2.SetIamPolicyRequest(
    resource = project_name,
    policy   = policy,   # etag in policy handles concurrency
)
updated = client.set_iam_policy(request=set_request)
print(f"Policy updated. New etag: {updated.etag}")

# Remove a binding
policy.bindings[:] = [
    b for b in policy.bindings
    if not (b.role == "roles/storage.objectViewer"
            and "user:old@company.com" in b.members)
]
client.set_iam_policy(request=iam_policy_pb2.SetIamPolicyRequest(
    resource = project_name, policy = policy
))
```

---

## 5. IAM Conditions

### Condition Anatomy

```json
{
  "condition": {
    "title":       "Temporary dev access",
    "description": "Access expires 2026-04-01",
    "expression":  "request.time < timestamp('2026-04-01T00:00:00Z')"
  }
}
```

---

### CEL Expression Syntax & Attributes

| Category | Attribute | Example Expression |
|---|---|---|
| **Time** | `request.time` | `request.time < timestamp('2026-04-01T00:00:00Z')` |
| **Time (hours)** | `request.time.getHours('UTC')` | `request.time.getHours('UTC') >= 9 && request.time.getHours('UTC') <= 17` |
| **Time (day of week)** | `request.time.getDayOfWeek('UTC')` | `request.time.getDayOfWeek('UTC') >= 1 && request.time.getDayOfWeek('UTC') <= 5` |
| **Resource name** | `resource.name` | `resource.name.startsWith('projects/my-project/buckets/dev-')` |
| **Resource type** | `resource.type` | `resource.type == 'storage.googleapis.com/Bucket'` |
| **Resource service** | `resource.service` | `resource.service == 'storage.googleapis.com'` |
| **Tag match** | `resource.matchTag()` | `resource.matchTag('env/production', 'true')` |
| **Tag ID** | `resource.matchTagId()` | `resource.matchTagId('tagValues/281482')` |
| **Access level** | `request.auth.access_levels` | `'accessPolicies/123/accessLevels/corp_vpn' in request.auth.access_levels` |

---

### gcloud: Conditional Bindings

```bash
# Time-based condition: access until a specific date
gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="user:contractor@company.com" \
  --role="roles/bigquery.dataViewer" \
  --condition="title=TempAccess,\
expression=request.time < timestamp('2026-06-01T00:00:00Z'),\
description=Temporary access until June 2026"

# Resource name condition: only access specific GCS paths
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="serviceAccount:sa@proj.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer" \
  --condition="title=DevFolderOnly,\
expression=resource.name.startsWith('projects/_/buckets/my-bucket/objects/dev/')"

# Business hours only (Mon-Fri, 9am-5pm UTC)
gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="serviceAccount:batch-sa@proj.iam.gserviceaccount.com" \
  --role="roles/bigquery.jobUser" \
  --condition='title=BusinessHoursOnly,expression=request.time.getHours("UTC") >= 9 && request.time.getHours("UTC") <= 17 && request.time.getDayOfWeek("UTC") >= 1 && request.time.getDayOfWeek("UTC") <= 5'

# Tag-based condition: only prod-tagged resources
gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="serviceAccount:prod-sa@proj.iam.gserviceaccount.com" \
  --role="roles/compute.viewer" \
  --condition='title=ProdTagOnly,expression=resource.matchTag("environment/env","production")'
```

---

### Python: Conditional Bindings

```python
from google.iam.v1 import policy_pb2

# Create a conditional binding for temporary access
conditional_binding = policy_pb2.Binding(
    role    = "roles/storage.objectAdmin",
    members = ["user:temp-user@company.com"],
    condition = policy_pb2.Expr(
        title       = "Temporary Admin Access",
        description = "Expires 2026-06-01",
        expression  = "request.time < timestamp('2026-06-01T00:00:00Z')",
    ),
)

# Add to policy and update
policy.version = 3   # Required for conditions
policy.bindings.append(conditional_binding)
```

---

### Common Condition Patterns

```python
# Just-in-time access (expires in 4 hours from now)
from datetime import datetime, timezone, timedelta
expiry = (datetime.now(timezone.utc) + timedelta(hours=4)).strftime('%Y-%m-%dT%H:%M:%SZ')
expression = f"request.time < timestamp('{expiry}')"

# Environment-based: restrict to dev resources only
expression = "resource.name.startsWith('projects/my-project/buckets/dev-')"

# Tag-gated: only resources tagged as non-sensitive
expression = "resource.matchTag('data-classification/sensitivity', 'low')"

# Access level: require corporate VPN
expression = "'accessPolicies/POLICY_ID/accessLevels/corp_vpn' in request.auth.access_levels"

# Combined: business hours AND corporate network
expression = """
    request.time.getHours('UTC') >= 9 &&
    request.time.getHours('UTC') <= 17 &&
    'accessPolicies/POLICY_ID/accessLevels/corp_vpn' in request.auth.access_levels
"""
```

---

## 6. 🚨 Deny Policies

### What are Deny Policies?

Deny policies provide **explicit deny rules** that are evaluated **before** allow policies. Even if a principal has an allow binding, a matching deny policy will block the action.

```
Evaluation order:
  1. DENY POLICIES  ← checked first
  2. Allow policies
  3. Implicit deny (default)
```

---

### Deny Policy JSON Structure

```json
{
  "name": "projects/MY_PROJECT/denypolicies/block-role-creation",
  "displayName": "Block IAM role creation org-wide",
  "rules": [
    {
      "denyRule": {
        "deniedPrincipals": [
          "principalSet://goog/public:all"
        ],
        "exceptionPrincipals": [
          "principal://iam.googleapis.com/projects/-/serviceAccounts/break-glass-sa@proj.iam.gserviceaccount.com"
        ],
        "deniedPermissions": [
          "iam.googleapis.com/roles.create",
          "iam.googleapis.com/roles.update",
          "iam.googleapis.com/roles.delete"
        ],
        "denialCondition": {
          "title": "Except during maintenance",
          "expression": "!request.time.between(timestamp('2026-03-20T02:00:00Z'), timestamp('2026-03-20T04:00:00Z'))"
        }
      }
    }
  ]
}
```

---

### gcloud: Deny Policies

```bash
# Create a deny policy at project level
gcloud iam deny-policies create block-role-creation \
  --project=MY_PROJECT \
  --policy-file=deny-policy.json \
  --display-name="Block IAM role creation"

# Create at org level
gcloud iam deny-policies create org-wide-deny \
  --organization=ORG_ID \
  --policy-file=org-deny-policy.json

# List deny policies
gcloud iam deny-policies list \
  --project=MY_PROJECT \
  --format="table(name,displayName)"

# Describe a deny policy
gcloud iam deny-policies get block-role-creation \
  --project=MY_PROJECT

# Update deny policy
gcloud iam deny-policies update block-role-creation \
  --project=MY_PROJECT \
  --policy-file=updated-deny.json

# Delete deny policy
gcloud iam deny-policies delete block-role-creation \
  --project=MY_PROJECT
```

---

### Deny Policy vs. Organization Policy

| Scenario | Use Deny Policy | Use Org Policy |
|---|---|---|
| Block specific IAM permissions | ✅ | ❌ |
| Prevent SA from calling `iam.roles.create` | ✅ | ❌ |
| Restrict which regions resources can be created | ❌ | ✅ |
| Block external IPs on VMs | ❌ | ✅ |
| Enforce uniform bucket access | ❌ | ✅ |
| Block SA key file creation org-wide | ❌ (use Org Policy) | ✅ |
| Prevent privilege escalation IAM actions | ✅ | ❌ |

> ⚠️ **Not all permissions are deniable.** The set of deniable permissions is a subset of all IAM permissions. Check the [deniable permissions reference](https://cloud.google.com/iam/docs/deny-permissions-reference) before designing deny policies.

---

## 7. 🔑 Service Accounts Deep Dive

### Service Account Use Cases

```
GCE Instance    → Attach SA to instance (instance metadata server provides token)
Cloud Run       → --service-account flag on deploy
Cloud Functions → --service-account flag on deploy
GKE Workload    → Workload Identity (K8s SA ↔ GCP SA binding)
Cloud Build     → Service account for build steps
On-prem/AWS     → Workload Identity Federation (no key file needed)
```

---

### gcloud: Service Account Operations

```bash
# ── CREATE & CONFIGURE ────────────────────────────────────────────────
# Create SA
gcloud iam service-accounts create app-sa \
  --display-name="App Service Account" \
  --project=MY_PROJECT

# Grant SA a role on the project
gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="serviceAccount:app-sa@MY_PROJECT.iam.gserviceaccount.com" \
  --role="roles/run.invoker"

# Attach SA to a Cloud Run service
gcloud run deploy my-service \
  --image=IMAGE_URI \
  --service-account=app-sa@MY_PROJECT.iam.gserviceaccount.com \
  --region=us-central1

# Attach SA to a GCE instance at creation
gcloud compute instances create my-vm \
  --service-account=app-sa@MY_PROJECT.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --zone=us-central1-a

# ── IMPERSONATION ─────────────────────────────────────────────────────
# Allow user to impersonate SA
gcloud iam service-accounts add-iam-policy-binding \
  app-sa@MY_PROJECT.iam.gserviceaccount.com \
  --member="user:developer@company.com" \
  --role="roles/iam.serviceAccountTokenCreator"

# Use impersonation in gcloud commands
gcloud storage ls gs://my-bucket \
  --impersonate-service-account=app-sa@MY_PROJECT.iam.gserviceaccount.com

# ── SHORT-LIVED CREDENTIALS ───────────────────────────────────────────
# Generate a short-lived access token (valid 1 hour)
gcloud auth print-access-token \
  --impersonate-service-account=app-sa@MY_PROJECT.iam.gserviceaccount.com

# Generate an ID token for authenticated Cloud Run/Functions calls
gcloud auth print-identity-token \
  --impersonate-service-account=app-sa@MY_PROJECT.iam.gserviceaccount.com \
  --audiences="https://my-service-hash-uc.a.run.app"

# ── KEY MANAGEMENT ────────────────────────────────────────────────────
# List all keys (including key creation dates)
gcloud iam service-accounts keys list \
  --iam-account=app-sa@MY_PROJECT.iam.gserviceaccount.com \
  --managed-by=user \
  --format="table(name.basename(),validAfterTime,validBeforeTime)"

# Disable a specific key
gcloud iam service-accounts keys disable KEY_ID \
  --iam-account=app-sa@MY_PROJECT.iam.gserviceaccount.com

# Delete a specific key
gcloud iam service-accounts keys delete KEY_ID \
  --iam-account=app-sa@MY_PROJECT.iam.gserviceaccount.com --quiet
```

---

### Python: Service Account Impersonation

```python
import google.auth
from google.auth import impersonated_credentials
from google.cloud import storage

# Impersonate a service account using Application Default Credentials
source_credentials, project = google.auth.default(
    scopes=["https://www.googleapis.com/auth/cloud-platform"]
)

target_credentials = impersonated_credentials.Credentials(
    source_credentials  = source_credentials,
    target_principal    = "app-sa@my-project.iam.gserviceaccount.com",
    target_scopes       = ["https://www.googleapis.com/auth/cloud-platform"],
    lifetime            = 3600,   # Max 3600 seconds (1 hour)
)

# Use impersonated credentials with any GCP client
storage_client = storage.Client(credentials=target_credentials)
for bucket in storage_client.list_buckets():
    print(f"Bucket: {bucket.name}")

# Generate a short-lived token for API calls
from google.auth.transport.requests import Request
target_credentials.refresh(Request())
print(f"Access token: {target_credentials.token[:20]}...")
print(f"Token expires: {target_credentials.expiry}")
```

---

## 8. Workload Identity Federation

### Architecture

```
External Workload (GitHub Actions / AWS / on-prem)
        │
        │ 1. Get token from IdP (OIDC JWT / AWS credentials)
        ▼
Google Security Token Service (STS)
        │ 2. Exchange external token for short-lived Google token
        │    (validates against Workload Identity Pool + Provider config)
        ▼
Workload Identity Pool
        │ 3. Map external claims → Google principal
        │    e.g., assertion.repository → "my-org/my-repo"
        ▼
GCP Service Account (via impersonation) OR Direct access
        │ 4. Use Google token to call GCP APIs
        ▼
GCP APIs (GCS, BigQuery, etc.)
```

---

### gcloud: Workload Identity Federation Setup

```bash
PROJECT=my-project
POOL_ID=github-actions-pool
PROVIDER_ID=github-actions-provider
SA_EMAIL=github-actions-sa@${PROJECT}.iam.gserviceaccount.com

# ── 1. Create Workload Identity Pool ─────────────────────────────────
gcloud iam workload-identity-pools create ${POOL_ID} \
  --project=${PROJECT} \
  --location=global \
  --display-name="GitHub Actions Pool" \
  --description="Trust pool for GitHub Actions CI/CD"

# ── 2a. Create OIDC Provider (GitHub Actions) ─────────────────────────
gcloud iam workload-identity-pools providers create-oidc ${PROVIDER_ID} \
  --project=${PROJECT} \
  --location=global \
  --workload-identity-pool=${POOL_ID} \
  --display-name="GitHub Actions Provider" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="\
google.subject=assertion.sub,\
attribute.actor=assertion.actor,\
attribute.repository=assertion.repository,\
attribute.repository_owner=assertion.repository_owner" \
  --attribute-condition="assertion.repository_owner == 'my-org'"

# ── 2b. Create AWS Provider ────────────────────────────────────────────
gcloud iam workload-identity-pools providers create-aws aws-provider \
  --project=${PROJECT} \
  --location=global \
  --workload-identity-pool=${POOL_ID} \
  --display-name="AWS Provider" \
  --account-id=123456789012 \
  --attribute-mapping="\
google.subject=assertion.arn,\
attribute.aws_role=assertion.arn.contains('assumed-role') ? assertion.arn.extract('{account_arn}assumed-role/') + 'assumed-role/' + assertion.arn.extract('assumed-role/{role_name}/') : assertion.arn"

# ── 3. Create the service account ─────────────────────────────────────
gcloud iam service-accounts create github-actions-sa \
  --display-name="GitHub Actions Service Account" \
  --project=${PROJECT}

# Grant it the permissions it needs
gcloud projects add-iam-policy-binding ${PROJECT} \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/storage.objectAdmin"

# ── 4. Allow the pool to impersonate the SA ────────────────────────────
# Allow ONLY the specific repo to impersonate
POOL_NAME="projects/$(gcloud projects describe ${PROJECT} --format='value(projectNumber)')/locations/global/workloadIdentityPools/${POOL_ID}"

gcloud iam service-accounts add-iam-policy-binding ${SA_EMAIL} \
  --project=${PROJECT} \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${POOL_NAME}/attribute.repository/my-org/my-repo"

# Get the pool's full resource name (needed for GitHub Actions workflow)
gcloud iam workload-identity-pools describe ${POOL_ID} \
  --project=${PROJECT} \
  --location=global \
  --format="value(name)"
```

---

### GitHub Actions Workflow (WIF)

```yaml
# .github/workflows/deploy.yml
name: Deploy to GCP

on:
  push:
    branches: [main]

permissions:
  id-token: write    # Required for Workload Identity Federation
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-actions-pool/providers/github-actions-provider'
          service_account: 'github-actions-sa@MY_PROJECT.iam.gserviceaccount.com'

      - name: Set up gcloud CLI
        uses: google-github-actions/setup-gcloud@v2

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy my-service \
            --image=us-central1-docker.pkg.dev/MY_PROJECT/my-repo/my-app:${{ github.sha }} \
            --region=us-central1 --quiet
```

---

### Python: WIF Authentication

```python
# Use WIF credential file for external workloads
import google.auth
from google.auth.external_account import Credentials

# Load credential config from WIF configuration JSON
# (download from Console: IAM → Workload Identity Federation → Download config)
credentials, project = google.auth.load_credentials_from_file(
    "wif-credential-config.json",
    scopes=["https://www.googleapis.com/auth/cloud-platform"]
)

# Use credentials with any GCP client
from google.cloud import storage
client = storage.Client(credentials=credentials, project=project)
```

---

## 9. Organization Policy Service

### Org Policy vs. IAM

```
IAM:         Controls WHO can perform actions
             "Alice can create VM instances"

Org Policy:  Controls WHAT configurations are allowed
             "No VM can have an external IP, regardless of who creates it"
```

---

### Key Built-in Constraints

| Constraint | Type | What It Controls |
|---|---|---|
| `constraints/compute.vmExternalIpAccess` | List | Which VMs can have external IPs |
| `constraints/compute.requireShieldedVm` | Boolean | Require Shielded VM for all VMs |
| `constraints/compute.restrictCloudNATUsage` | List | Restrict Cloud NAT usage |
| `constraints/iam.allowedPolicyMemberDomains` | List | Restrict IAM members to specific domains |
| `constraints/iam.disableServiceAccountKeyCreation` | Boolean | Block SA key file creation |
| `constraints/iam.disableServiceAccountKeyUpload` | Boolean | Block SA key upload |
| `constraints/gcp.resourceLocations` | List | Restrict resource creation to specific regions |
| `constraints/storage.uniformBucketLevelAccess` | Boolean | Enforce uniform bucket-level access |
| `constraints/storage.publicAccessPrevention` | Boolean | Prevent public GCS bucket access |
| `constraints/resourcemanager.allowedExportDestinations` | List | Restrict BQ data export destinations |
| `constraints/compute.skipDefaultNetworkCreation` | Boolean | Don't create default VPC in new projects |

---

### YAML Policy Format

```yaml
# boolean-policy.yaml — enforce no external IPs org-wide
name: organizations/ORG_ID/policies/compute.vmExternalIpAccess
spec:
  rules:
    - enforce: true

---
# list-policy.yaml — restrict resource locations
name: organizations/ORG_ID/policies/gcp.resourceLocations
spec:
  rules:
    - values:
        allowedValues:
          - in:us-locations
          - in:eu-locations
      allow: true

---
# domain-restriction.yaml — only allow company.com identities
name: organizations/ORG_ID/policies/iam.allowedPolicyMemberDomains
spec:
  rules:
    - values:
        allowedValues:
          - C03abc123   # Google Workspace/Cloud Identity customer ID
      allow: true
```

---

### gcloud: Org Policy

```bash
# List all available constraints
gcloud org-policies list-constraints \
  --organization=ORG_ID \
  --format="table(name,constraintDefault,displayName)"

# Describe a constraint
gcloud org-policies describe \
  constraints/iam.disableServiceAccountKeyCreation \
  --organization=ORG_ID

# Set a boolean policy (enforce = true)
gcloud org-policies set-policy boolean-policy.yaml

# Set via inline flag
gcloud org-policies enable-enforce \
  constraints/iam.disableServiceAccountKeyCreation \
  --organization=ORG_ID

# Set list policy from file
gcloud org-policies set-policy list-policy.yaml

# Get effective policy (shows inherited + override)
gcloud org-policies describe \
  constraints/gcp.resourceLocations \
  --project=MY_PROJECT \
  --effective

# Reset to default (remove any override)
gcloud org-policies reset \
  constraints/iam.disableServiceAccountKeyCreation \
  --organization=ORG_ID

# Delete a policy (revert to constraint default)
gcloud org-policies delete \
  constraints/iam.disableServiceAccountKeyCreation \
  --project=MY_PROJECT
```

---

### Custom Org Policy Constraint

```yaml
# custom-constraint.yaml — block VMs without labels
name: organizations/ORG_ID/customConstraints/custom.requireVmLabels
resourceTypes:
  - compute.googleapis.com/Instance
methodTypes:
  - CREATE
  - UPDATE
condition: >
  resource.labels.size() > 0 &&
  "env" in resource.labels &&
  "team" in resource.labels
actionType: ALLOW
displayName: "Require env and team labels on VMs"
description: "All VM instances must have env and team labels"
```

```bash
# Create custom constraint
gcloud org-policies set-custom-constraint custom-constraint.yaml

# Create policy to enforce it
gcloud org-policies set-policy policy-enforce-labels.yaml
```

---

## 10. Access Context Manager & VPC Service Controls

### Access Context Manager Overview

**Access Context Manager** defines **access levels** — conditions that must be met for a request to be considered "trusted". These levels are used in:
- IAM Conditions (`request.auth.access_levels`)
- VPC Service Controls perimeters

---

### gcloud: Access Context Manager

```bash
# ── Access Policy (one per org) ────────────────────────────────────────
# Create the access policy
gcloud access-context-manager policies create \
  --organization=ORG_ID \
  --title="My Corp Access Policy"

# ── Access Levels ──────────────────────────────────────────────────────
# Create an IP-based access level (corporate network)
cat > access-level.yaml << 'EOF'
name: accessPolicies/POLICY_ID/accessLevels/corp_vpn
title: Corporate VPN
description: Requests from corporate IP ranges
basic:
  conditions:
    - ipSubnetworks:
        - 203.0.113.0/24
        - 198.51.100.0/24
EOF

gcloud access-context-manager levels create corp_vpn \
  --policy=POLICY_ID \
  --title="Corporate VPN" \
  --basic-level-spec=access-level.yaml

# List access levels
gcloud access-context-manager levels list \
  --policy=POLICY_ID

# ── Service Perimeters ────────────────────────────────────────────────
# Create a VPC Service Controls perimeter
gcloud access-context-manager perimeters create my-perimeter \
  --policy=POLICY_ID \
  --title="Data Protection Perimeter" \
  --type=regular \
  --resources=projects/MY_PROJECT_NUMBER \
  --restricted-services=\
bigquery.googleapis.com,\
storage.googleapis.com,\
cloudfunctions.googleapis.com \
  --access-levels=accessPolicies/POLICY_ID/accessLevels/corp_vpn

# Create in dry-run mode first
gcloud access-context-manager perimeters dry-run create my-perimeter \
  --policy=POLICY_ID \
  --title="Data Protection Perimeter (dry run)" \
  --resources=projects/MY_PROJECT_NUMBER \
  --restricted-services=bigquery.googleapis.com,storage.googleapis.com

# Describe perimeter
gcloud access-context-manager perimeters describe my-perimeter \
  --policy=POLICY_ID

# Enforce dry-run perimeter
gcloud access-context-manager perimeters dry-run enforce my-perimeter \
  --policy=POLICY_ID
```

---

## 11. 📊 IAM Recommender & Policy Intelligence

### IAM Recommender

The IAM Recommender analyzes **90 days of activity logs** and compares granted permissions vs. actually used permissions. It surfaces three recommendation types:

| Recommendation Type | Description |
|---|---|
| **Excess permissions** | Role grants permissions never used in 90 days |
| **Service account key** | Recommends removing unused SA keys |
| **Lateral movement** | SA with `iam.serviceAccounts.actAs` that could be abused |

---

### gcloud: Recommender & Policy Intelligence

```bash
# List IAM recommendations for a project
gcloud recommender recommendations list \
  --recommender=google.iam.policy.Recommender \
  --project=MY_PROJECT \
  --location=global \
  --format="table(name,recommenderSubtype,primaryImpact.securityProjection.details.overview)"

# Get a specific recommendation
gcloud recommender recommendations describe RECOMMENDATION_ID \
  --recommender=google.iam.policy.Recommender \
  --project=MY_PROJECT \
  --location=global

# Apply a recommendation (replaces overly-broad role with narrower one)
gcloud recommender recommendations mark-claimed RECOMMENDATION_ID \
  --recommender=google.iam.policy.Recommender \
  --project=MY_PROJECT \
  --location=global \
  --etag=ETAG

gcloud recommender recommendations apply RECOMMENDATION_ID \
  --recommender=google.iam.policy.Recommender \
  --project=MY_PROJECT \
  --location=global \
  --etag=ETAG

# Dismiss a recommendation
gcloud recommender recommendations dismiss RECOMMENDATION_ID \
  --recommender=google.iam.policy.Recommender \
  --project=MY_PROJECT \
  --location=global \
  --etag=ETAG

# Policy Analyzer: who has access to what?
gcloud policy-intelligence query-activity \
  --project=MY_PROJECT \
  --activity-type=serviceAccountKeyLastAuthentication

# Analyze IAM policy: find all principals with access to a resource
gcloud policy-intelligence analyze-iam-policy \
  --project=MY_PROJECT \
  --full-resource-name="//storage.googleapis.com/projects/_/buckets/my-bucket" \
  --format=json

# Policy Troubleshooter: why can/can't a principal do X?
gcloud policy-troubleshoot iam \
  //storage.googleapis.com/projects/MY_PROJECT/buckets/my-bucket \
  --principal-email=alice@company.com \
  --permission=storage.objects.get
```

---

### Python: Policy Intelligence

```python
from google.cloud import policytroubleshooter_iam_v3
from google.cloud import asset_v1

# Policy Troubleshooter
troubleshooter = policytroubleshooter_iam_v3.PolicyTroubleshooterClient()

response = troubleshooter.troubleshoot_iam_policy(
    request=policytroubleshooter_iam_v3.TroubleshootIamPolicyRequest(
        access_tuple=policytroubleshooter_iam_v3.AccessTuple(
            principal            = "user:alice@company.com",
            full_resource_name   = "//storage.googleapis.com/projects/my-project/buckets/my-bucket",
            permission           = "storage.objects.get",
        )
    )
)
access = response.access
print(f"Access: {policytroubleshooter_iam_v3.AccessState(access).name}")
for explained in response.iam_policy_explanations:
    print(f"  Policy on: {explained.full_resource_name}")
    for binding in explained.binding_explanations:
        print(f"    Role: {binding.role}, Access: {binding.role_permission.name}")

# Policy Analyzer: find all principals with a permission
from google.cloud import asset_v1

asset_client = asset_v1.AssetServiceClient()

query = asset_v1.IamPolicyAnalysisQuery(
    scope = "projects/my-project",
    resource_selector = asset_v1.IamPolicyAnalysisQuery.ResourceSelector(
        full_resource_name = "//storage.googleapis.com/projects/my-project/buckets/my-bucket"
    ),
    access_selector = asset_v1.IamPolicyAnalysisQuery.AccessSelector(
        permissions = ["storage.objects.get"]
    ),
)

response = asset_client.analyze_iam_policy(
    request=asset_v1.AnalyzeIamPolicyRequest(analysis_query=query)
)
for result in response.main_analysis.analysis_results:
    for binding in result.iam_binding_explanations:
        for member in binding.memberships:
            print(f"  {member.member}: {binding.access_control_lists[0].accesses[0].permission}")
```

---

## 12. Audit Logging

### Cloud Audit Log Types

| Log Type | Always On? | What It Captures | IAM-Specific Examples |
|---|---|---|---|
| **Admin Activity** | ✅ Yes | Changes to resources and policies | `SetIamPolicy`, `CreateServiceAccount` |
| **Data Access** | ❌ Opt-in | Reads and writes to data | `GetIamPolicy`, `GetObject` |
| **System Event** | ✅ Yes | GCP system-initiated changes | Automatic SA operations |
| **Policy Denied** | ✅ Yes | IAM deny and VPC-SC denials | Denied API calls |

---

### Enabling Data Access Audit Logs

```yaml
# audit-config.yaml — add to IAM policy's auditConfigs
auditConfigs:
  - service: iam.googleapis.com
    auditLogConfigs:
      - logType: DATA_READ    # GetIamPolicy, ListRoles
      - logType: DATA_WRITE   # CreateServiceAccountKey
  - service: storage.googleapis.com
    auditLogConfigs:
      - logType: DATA_READ
      - logType: DATA_WRITE
  - service: bigquery.googleapis.com
    auditLogConfigs:
      - logType: DATA_READ
      - logType: DATA_WRITE
      - logType: ADMIN_READ
```

```bash
# Apply audit config to a project
gcloud projects get-iam-policy MY_PROJECT > current-policy.json
# Edit to add auditConfigs, then:
gcloud projects set-iam-policy MY_PROJECT updated-policy.json
```

---

### Cloud Logging Queries

```bash
# ── Who changed IAM policies in the last 24 hours? ────────────────────
gcloud logging read \
  'protoPayload.methodName="SetIamPolicy"
   protoPayload.serviceName="cloudresourcemanager.googleapis.com"
   timestamp>="2026-03-15T00:00:00Z"' \
  --project=MY_PROJECT \
  --limit=50 \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.resourceName)"

# ── Who created or deleted a service account? ─────────────────────────
gcloud logging read \
  'protoPayload.serviceName="iam.googleapis.com"
   (protoPayload.methodName="google.iam.admin.v1.IAM.CreateServiceAccount"
    OR protoPayload.methodName="google.iam.admin.v1.IAM.DeleteServiceAccount")' \
  --project=MY_PROJECT \
  --limit=50

# ── SA key creation (potential security risk) ─────────────────────────
gcloud logging read \
  'protoPayload.methodName="google.iam.admin.v1.IAM.CreateServiceAccountKey"' \
  --project=MY_PROJECT \
  --limit=50 \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.request.serviceAccountId)"

# ── Policy Denied events (VPC-SC or Deny Policy blocks) ───────────────
gcloud logging read \
  'logName="projects/MY_PROJECT/logs/cloudaudit.googleapis.com%2Fpolicy"' \
  --project=MY_PROJECT \
  --limit=50

# ── Access token generation (SA impersonation) ────────────────────────
gcloud logging read \
  'protoPayload.methodName="GenerateAccessToken"
   protoPayload.serviceName="iamcredentials.googleapis.com"' \
  --project=MY_PROJECT --limit=50

# ── Privilege escalation attempts ─────────────────────────────────────
gcloud logging read \
  '(protoPayload.methodName="SetIamPolicy"
    OR protoPayload.methodName="google.iam.admin.v1.IAM.CreateRole")
   protoPayload.authorizationInfo.granted=false' \
  --project=MY_PROJECT --limit=50
```

---

### BigQuery Log Analysis

```sql
-- Find all SetIamPolicy changes in the last 30 days
SELECT
  TIMESTAMP_TRUNC(timestamp, DAY)              AS day,
  protopayload_auditlog.authenticationinfo.principalemail AS actor,
  protopayload_auditlog.resourcename           AS resource,
  JSON_VALUE(protopayload_auditlog.requestjson, '$.policy.bindings') AS new_bindings,
  COUNT(*)                                     AS change_count
FROM `my-project.log_exports.cloudaudit_googleapis_com_activity_*`
WHERE
  protopayload_auditlog.methodname = "SetIamPolicy"
  AND _TABLE_SUFFIX >= FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY))
GROUP BY 1, 2, 3, 4
ORDER BY 5 DESC;

-- Find service accounts with the most permissions granted
SELECT
  JSON_VALUE(member) AS member,
  COUNT(DISTINCT role) AS role_count
FROM `my-project.log_exports.cloudaudit_googleapis_com_activity_*`,
  UNNEST(JSON_QUERY_ARRAY(
    protopayload_auditlog.requestjson, '$.policy.bindings'
  )) AS binding,
  UNNEST(JSON_VALUE_ARRAY(binding, '$.members')) AS member
WHERE protopayload_auditlog.methodname = "SetIamPolicy"
  AND member LIKE 'serviceAccount:%'
GROUP BY 1 ORDER BY 2 DESC
LIMIT 20;
```

---

### Setting Up IAM Change Alerts

```bash
# Create a log-based alert for SetIamPolicy changes
gcloud logging metrics create iam-policy-changes \
  --description="Count of IAM policy changes" \
  --log-filter='protoPayload.methodName="SetIamPolicy"'

# Create an alerting policy on this metric
gcloud alpha monitoring policies create \
  --display-name="IAM Policy Change Alert" \
  --condition-filter='metric.type="logging.googleapis.com/user/iam-policy-changes"' \
  --condition-threshold-value=0 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=0s \
  --notification-channels=NOTIFICATION_CHANNEL_ID \
  --documentation-content="An IAM policy was modified. Review immediately."
```

---

## 13. 🛡️ IAM Best Practices & Security Patterns

### Principle of Least Privilege

```bash
# Step 1: Check what permissions are actually being used
gcloud recommender recommendations list \
  --recommender=google.iam.policy.Recommender \
  --project=MY_PROJECT \
  --location=global

# Step 2: Replace over-privileged binding with minimal role
# Before: roles/editor → user:alice@company.com
# After:  roles/storage.objectViewer → user:alice@company.com

# Step 3: Use custom roles for fine-grained control
gcloud iam roles create MinimalBQReader \
  --project=MY_PROJECT \
  --file=minimal-bq-role.yaml
```

---

### Privileged Access Management (PAM)

```bash
# Create a PAM entitlement (just-in-time access)
gcloud pam entitlements create prod-admin-access \
  --project=MY_PROJECT \
  --location=global \
  --role=roles/container.admin \
  --max-request-duration=3600s \
  --requester-justification-required \
  --approvers="group:security-team@company.com"

# Request JIT access as a user
gcloud pam grants create \
  --entitlement=prod-admin-access \
  --project=MY_PROJECT \
  --location=global \
  --requested-duration=1800s \
  --justification="Emergency: investigating prod incident P1-2026-0316"

# Approve a grant request
gcloud pam grants approve GRANT_ID \
  --entitlement=prod-admin-access \
  --project=MY_PROJECT \
  --location=global
```

---

### IAM Governance Checklist

| # | Control | How to Implement |
|---|---|---|
| 1 | 🚨 No basic roles in production | Deny policies + Org Policy audit |
| 2 | 🔐 Workload Identity Federation for external workloads | Replace key files with WIF |
| 3 | 🔑 One service account per workload | Naming: `{app}-{env}-sa` |
| 4 | 🚫 Block SA key creation org-wide | Org Policy: `iam.disableServiceAccountKeyCreation` |
| 5 | 👥 Grant roles to groups, not users | Require Google Groups for all bindings |
| 6 | ⏰ Time-limited access for humans | IAM Conditions on all human bindings |
| 7 | 📊 Enable IAM Recommender | Review weekly; apply recommendations |
| 8 | 🛡️ Deny policies for critical permissions | Block `iam.roles.create` org-wide |
| 9 | 🔒 VPC Service Controls for sensitive projects | Perimeter around data projects |
| 10 | 📝 Data Access audit logs for sensitive services | Enable for BQ, GCS, CloudSQL |
| 11 | 🚨 Alert on SetIamPolicy changes | Log-based alert → Pub/Sub → PagerDuty |
| 12 | 🏷️ Resource Location org policy | Restrict to approved regions |
| 13 | 🌐 Domain restriction org policy | `iam.allowedPolicyMemberDomains` |
| 14 | 🔄 Regular access review | Quarterly review with Policy Analyzer |
| 15 | 🆘 Break-glass account process | Emergency SA with MFA, alerting |

---

### Anti-Patterns & Fixes

| ❌ Anti-Pattern | 🔴 Risk | ✅ Fix |
|---|---|---|
| Granting `roles/editor` to a SA | SA can modify any GCP resource | Create dedicated custom role with minimum permissions |
| Using default Compute Engine SA | 50+ services have read/write access | Disable default SA; create per-workload SAs |
| Sharing service account keys via email | Key leakage → persistent access | Use Workload Identity Federation; delete all key files |
| Binding roles to individual users | Hard to revoke when employee leaves | Use Google Groups for all bindings |
| `allAuthenticatedUsers` on GCS bucket | Any Google user can read data | Restrict to specific service accounts or groups |
| Owner/Editor role to CI/CD pipeline | Build system can delete prod resources | Grant only specific roles needed for deployment |
| No org-level domain restriction | External accounts can be granted access | `constraints/iam.allowedPolicyMemberDomains` |
| `set-iam-policy` without reading etag | Risk of overwriting concurrent changes | Always get-iam-policy first; use returned etag |
| No audit logging on Data Access | Cannot detect unauthorized reads | Enable Data Access logs for sensitive services |
| Long-lived SA keys for local dev | Developer machines get compromised | Use `gcloud auth application-default login` |
| Forgetting Org Policy blocks SA keys | Individual projects can still create keys | Apply Org Policy at org level, not just projects |
| SA with `iam.serviceAccounts.actAs` at project | Can impersonate any SA in project | Grant only specific SA impersonation |

---

## 14. gcloud CLI & REST API Quick Reference

### All Essential IAM Commands

```bash
# ── SERVICE ACCOUNTS ──────────────────────────────────────────────────
gcloud iam service-accounts create SA_NAME [--display-name --description]
gcloud iam service-accounts list [--project --filter]
gcloud iam service-accounts describe SA_EMAIL
gcloud iam service-accounts update SA_EMAIL [--display-name]
gcloud iam service-accounts enable SA_EMAIL
gcloud iam service-accounts disable SA_EMAIL
gcloud iam service-accounts delete SA_EMAIL
gcloud iam service-accounts get-iam-policy SA_EMAIL
gcloud iam service-accounts add-iam-policy-binding SA_EMAIL [--member --role]
gcloud iam service-accounts remove-iam-policy-binding SA_EMAIL [--member --role]
gcloud iam service-accounts keys create KEYFILE --iam-account=SA_EMAIL
gcloud iam service-accounts keys list --iam-account=SA_EMAIL
gcloud iam service-accounts keys delete KEY_ID --iam-account=SA_EMAIL
gcloud iam service-accounts keys disable KEY_ID --iam-account=SA_EMAIL
gcloud iam service-accounts keys enable KEY_ID --iam-account=SA_EMAIL

# ── ROLES ─────────────────────────────────────────────────────────────
gcloud iam roles list [--project --organization --filter]
gcloud iam roles describe ROLE_NAME [--project --organization]
gcloud iam roles create ROLE_ID [--project --organization --file]
gcloud iam roles copy [--source --destination --dest-project]
gcloud iam roles update ROLE_ID [--project --add-permissions --remove-permissions --stage]
gcloud iam roles delete ROLE_ID [--project --organization]
gcloud iam roles undelete ROLE_ID [--project]
gcloud iam list-testable-permissions RESOURCE

# ── PROJECT IAM ────────────────────────────────────────────────────────
gcloud projects get-iam-policy PROJECT_ID [--format=json]
gcloud projects set-iam-policy PROJECT_ID POLICY_FILE
gcloud projects add-iam-policy-binding PROJECT_ID [--member --role --condition]
gcloud projects remove-iam-policy-binding PROJECT_ID [--member --role]

# ── FOLDER IAM ─────────────────────────────────────────────────────────
gcloud resource-manager folders get-iam-policy FOLDER_ID
gcloud resource-manager folders set-iam-policy FOLDER_ID POLICY_FILE
gcloud resource-manager folders add-iam-policy-binding FOLDER_ID [--member --role]
gcloud resource-manager folders remove-iam-policy-binding FOLDER_ID [--member --role]

# ── ORGANIZATION IAM ───────────────────────────────────────────────────
gcloud organizations get-iam-policy ORG_ID
gcloud organizations set-iam-policy ORG_ID POLICY_FILE
gcloud organizations add-iam-policy-binding ORG_ID [--member --role]
gcloud organizations remove-iam-policy-binding ORG_ID [--member --role]

# ── DENY POLICIES ──────────────────────────────────────────────────────
gcloud iam deny-policies create POLICY_ID [--project --organization --policy-file]
gcloud iam deny-policies list [--project --organization]
gcloud iam deny-policies get POLICY_ID [--project --organization]
gcloud iam deny-policies update POLICY_ID [--project --organization --policy-file]
gcloud iam deny-policies delete POLICY_ID [--project --organization]

# ── WORKLOAD IDENTITY ─────────────────────────────────────────────────
gcloud iam workload-identity-pools create POOL_ID [--project --location]
gcloud iam workload-identity-pools list [--project --location]
gcloud iam workload-identity-pools describe POOL_ID [--project --location]
gcloud iam workload-identity-pools delete POOL_ID [--project --location]
gcloud iam workload-identity-pools providers create-oidc PROVIDER_ID [flags]
gcloud iam workload-identity-pools providers create-aws PROVIDER_ID [flags]
gcloud iam workload-identity-pools providers list --workload-identity-pool=POOL [flags]

# ── ORG POLICY ────────────────────────────────────────────────────────
gcloud org-policies list [--organization --folder --project]
gcloud org-policies describe CONSTRAINT [--organization --folder --project --effective]
gcloud org-policies set-policy POLICY_FILE
gcloud org-policies enable-enforce CONSTRAINT [--organization --folder --project]
gcloud org-policies disable-enforce CONSTRAINT [--organization --folder --project]
gcloud org-policies reset CONSTRAINT [--organization --folder --project]
gcloud org-policies delete CONSTRAINT [--organization --folder --project]
gcloud org-policies list-constraints [--organization]

# ── POLICY INTELLIGENCE ────────────────────────────────────────────────
gcloud policy-intelligence analyze-iam-policy [--project --full-resource-name --permissions]
gcloud policy-troubleshoot iam RESOURCE [--principal-email --permission]
gcloud policy-intelligence query-activity [--project --activity-type]

# ── IAM RECOMMENDER ───────────────────────────────────────────────────
gcloud recommender recommendations list [--recommender --project --location]
gcloud recommender recommendations describe REC_ID [--recommender --project --location]
gcloud recommender recommendations apply REC_ID [--recommender --project --location --etag]
gcloud recommender recommendations dismiss REC_ID [--recommender --project --location --etag]

# ── ACCESS CONTEXT MANAGER ────────────────────────────────────────────
gcloud access-context-manager policies create [--organization --title]
gcloud access-context-manager policies list --organization=ORG_ID
gcloud access-context-manager levels create LEVEL_ID [--policy --basic-level-spec]
gcloud access-context-manager levels list --policy=POLICY_ID
gcloud access-context-manager perimeters create PERIMETER [--policy --resources --restricted-services]
gcloud access-context-manager perimeters list --policy=POLICY_ID
gcloud access-context-manager perimeters dry-run create PERIMETER [flags]
gcloud access-context-manager perimeters dry-run enforce PERIMETER [--policy]
```

---

### REST API Base URLs

| Service | Base URL |
|---|---|
| IAM (resource manager) | `https://cloudresourcemanager.googleapis.com/v1/projects/{project}:setIamPolicy` |
| IAM (service accounts) | `https://iam.googleapis.com/v1/projects/{project}/serviceAccounts` |
| IAM credentials | `https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/{sa}:generateAccessToken` |
| Org Policy | `https://orgpolicy.googleapis.com/v2/projects/{project}/policies` |
| Policy Intelligence | `https://cloudasset.googleapis.com/v1/projects/{project}:analyzeIamPolicy` |
| Recommender | `https://recommender.googleapis.com/v1/projects/{project}/locations/global/recommenders/google.iam.policy.Recommender/recommendations` |
| Access Context Manager | `https://accesscontextmanager.googleapis.com/v1/accessPolicies` |

---

### Output Formatting Tips

```bash
# JSON output for scripting
gcloud projects get-iam-policy MY_PROJECT --format=json

# Custom table showing only service account bindings
gcloud projects get-iam-policy MY_PROJECT \
  --format="table(bindings.role,bindings.members)" \
  --flatten="bindings" \
  --filter="bindings.members:serviceAccount"

# Find all roles granted to a specific user
gcloud projects get-iam-policy MY_PROJECT \
  --format="json" | jq -r \
  '.bindings[] | select(.members[] | contains("user:alice@company.com")) | .role'

# Export bindings as CSV
gcloud projects get-iam-policy MY_PROJECT --format=json \
  | jq -r '.bindings[] | .role as $r | .members[] | [$r, .] | @csv' \
  > bindings.csv
```

---

## 15. 💰 Pricing Summary

| Service | Free? | Paid Component |
|---|---|---|
| **IAM core** (policies, roles, bindings) | ✅ Free | No charge |
| **Service Account** (create, manage) | ✅ Free | No charge |
| **Service Account keys** | ✅ Free to create | Cost is security risk, not monetary |
| **Workload Identity Federation** | ✅ Free | No charge |
| **Organization Policy** | ✅ Free | No charge |
| **Custom Org Policy constraints** | ✅ Free | No charge |
| **IAM Recommender (basic)** | ✅ Free | Basic recommendations free |
| **Recommender API** (programmatic) | 💰 Charged | Per API call beyond free tier |
| **Policy Analyzer API** | 💰 Charged | Per query |
| **Access Context Manager** (access levels) | ✅ Free | No charge for access levels |
| **VPC Service Controls** (perimeters) | 💰 Charged | Per project-month inside perimeter (~$18/project/month) |
| **Cloud Audit Logs — Admin Activity** | ✅ Free | Always on, no charge |
| **Cloud Audit Logs — System Event** | ✅ Free | Always on, no charge |
| **Cloud Audit Logs — Data Access** | 💰 Charged | Per GB ingested (first 50 GB/project/month free) |
| **Cloud Audit Logs — Policy Denied** | ✅ Free | Always on, no charge |
| **Privileged Access Management (PAM)** | 💰 Charged | Per entitlement-hour |

---

### 💰 Cost Optimization Tips

| Tip | Savings |
|---|---|
| Enable Data Access logs only for sensitive services (BQ, GCS, CloudSQL) | Reduce log ingestion costs significantly |
| Export logs to GCS (not BigQuery) for long-term archival | GCS storage is ~10x cheaper than BQ |
| Set log retention policies (shorter for non-critical) | Reduce Cloud Logging storage costs |
| Use log exclusion filters to skip noisy, low-value logs | Direct cost reduction |
| Only enable VPC Service Controls on high-risk data projects | ~$18/project/month per project in perimeter |

---

## 16. Quick Reference & Comparison Tables

### IAM Principal Types

| Type | Member Identifier Format | Use Case |
|---|---|---|
| Google Account | `user:alice@company.com` | Individual humans |
| Service Account | `serviceAccount:name@proj.iam.gserviceaccount.com` | Workloads, applications |
| Google Group | `group:team@company.com` | Team-based access management |
| Workspace Domain | `domain:company.com` | All org members |
| Cloud Identity Domain | `domain:company.com` | Cloud Identity users |
| WIF Principal Set | `principalSet://iam.googleapis.com/POOL_NAME/attribute.X/Y` | External workloads |
| All authenticated | `allAuthenticatedUsers` | ⚠️ Any Google account — use with extreme care |
| All users (public) | `allUsers` | 🚨 Public internet — only for public content |

---

### Role Types Comparison

| Aspect | Basic | Predefined | Custom |
|---|---|---|---|
| **Scope** | All services | One service | You define |
| **Permissions** | 1,000–15,000+ | Curated set | Exact set you need |
| **Maintainability** | Auto-updated | Auto-updated | You maintain |
| **Production use** | ❌ Never | ✅ Preferred | ✅ Fine-grained control |
| **Billing** | Free | Free | Free |
| **Max per project** | N/A | N/A | 300 |
| **Org-level creation** | N/A | N/A | ✅ |

---

### Key Predefined Roles Reference

| Service | Role | What It Grants |
|---|---|---|
| **Cloud Storage** | `roles/storage.objectViewer` | Read GCS objects |
| **Cloud Storage** | `roles/storage.objectCreator` | Create (not delete) objects |
| **Cloud Storage** | `roles/storage.objectAdmin` | Full object management |
| **BigQuery** | `roles/bigquery.dataViewer` | Read BQ tables/views |
| **BigQuery** | `roles/bigquery.dataEditor` | Read+write BQ tables |
| **BigQuery** | `roles/bigquery.jobUser` | Run BQ queries (but not read data) |
| **Compute Engine** | `roles/compute.instanceAdmin.v1` | Full VM management |
| **Compute Engine** | `roles/compute.viewer` | View VMs and network |
| **GKE** | `roles/container.developer` | Deploy workloads to GKE |
| **GKE** | `roles/container.clusterViewer` | View cluster config |
| **Cloud Run** | `roles/run.developer` | Deploy and manage Cloud Run |
| **Cloud Run** | `roles/run.invoker` | Invoke Cloud Run services |
| **Cloud SQL** | `roles/cloudsql.client` | Connect to Cloud SQL |
| **Cloud SQL** | `roles/cloudsql.instanceUser` | Log in to Cloud SQL |
| **Pub/Sub** | `roles/pubsub.publisher` | Publish messages |
| **Pub/Sub** | `roles/pubsub.subscriber` | Subscribe and consume |
| **Secret Manager** | `roles/secretmanager.secretAccessor` | Read secret values |
| **Logging** | `roles/logging.logWriter` | Write log entries |
| **Monitoring** | `roles/monitoring.metricWriter` | Write metrics |

---

### IAM Conditions Attribute Reference

| Category | Attribute | Example Expression |
|---|---|---|
| Time | `request.time` | `request.time < timestamp('2026-06-01T00:00:00Z')` |
| Hour of day | `request.time.getHours('UTC')` | `>= 9 && <= 17` |
| Day of week | `request.time.getDayOfWeek('UTC')` | `>= 1 && <= 5` (Mon–Fri) |
| Resource name | `resource.name` | `.startsWith('projects/proj/buckets/dev-')` |
| Resource name | `resource.name` | `.endsWith('/prod')` |
| Resource type | `resource.type` | `== 'storage.googleapis.com/Bucket'` |
| Tag match | `resource.matchTag()` | `resource.matchTag('env/environment', 'production')` |
| Tag ID | `resource.matchTagId()` | `resource.matchTagId('tagValues/281482')` |
| Access level | `request.auth.access_levels` | `'accessPolicies/POLICY/accessLevels/corp_vpn' in ...` |

---

### Organization Policy Key Constraints

| Constraint | Type | Enforcement |
|---|---|---|
| `compute.vmExternalIpAccess` | List | Which VMs can have external IPs |
| `compute.requireShieldedVm` | Boolean | Require Shielded VMs |
| `compute.skipDefaultNetworkCreation` | Boolean | No default VPC in new projects |
| `iam.allowedPolicyMemberDomains` | List | Restrict IAM member domains |
| `iam.disableServiceAccountKeyCreation` | Boolean | Block SA key file creation |
| `iam.disableServiceAccountKeyUpload` | Boolean | Block SA key upload |
| `gcp.resourceLocations` | List | Restrict resource creation regions |
| `storage.uniformBucketLevelAccess` | Boolean | Enforce uniform GCS bucket access |
| `storage.publicAccessPrevention` | Boolean | Block public GCS bucket access |
| `resourcemanager.allowedExportDestinations` | List | Restrict BQ export destinations |

---

### IAM vs. Organization Policy Decision Guide

| Question | Answer | Use |
|---|---|---|
| "Can Alice access this resource?" | Control who has access | IAM |
| "What resources can be created in this org?" | Control resource configuration | Org Policy |
| "Can any VM have an external IP?" | Configuration constraint | Org Policy |
| "Can this SA call the BigQuery API?" | Who can do what | IAM |
| "Block SA key creation for everyone" | Configuration guardrail | Org Policy (`iam.disableServiceAccountKeyCreation`) |
| "Block specific SA from calling `iam.roles.create`" | Explicit deny for principal | Deny Policy |
| "All GCS buckets must use uniform access" | Configuration enforcement | Org Policy |

---

### Service Account Types Comparison

| Type | Who Creates | Purpose | Key File Risk | Recommended? |
|---|---|---|---|---|
| **User-managed** | You | Application workloads | Avoid key files | ✅ Yes — with WIF |
| **Default (Compute)** | GCP automatically | Default VM access | High (Editor role) | ❌ No |
| **Default (App Engine)** | GCP automatically | GAE apps | High (Editor role) | ❌ No |
| **Google-managed** | GCP for its own services | GCP service operations | Not user-accessible | N/A — don't modify |

---

### Workload Identity Federation: OIDC vs. AWS

| Aspect | OIDC Provider | AWS Provider |
|---|---|---|
| **Use cases** | GitHub Actions, GitLab CI, on-prem | AWS EC2, Lambda, ECS |
| **Issuer** | OIDC issuer URL (e.g., token.actions.githubusercontent.com) | AWS account ID |
| **Attribute mapping** | Map JWT claims to Google attributes | Map AWS ARN to Google attributes |
| **Credential exchange** | JWT token → Google access token | AWS SigV4 signed request → Google token |
| **Key file needed** | ❌ No | ❌ No |

---

### IAM Governance Checklist (15 Controls)

| # | Control | Priority |
|---|---|---|
| 1 | Block basic roles (Owner/Editor) in production via deny policies | 🚨 Critical |
| 2 | Enable `iam.disableServiceAccountKeyCreation` org policy | 🚨 Critical |
| 3 | Enable `iam.allowedPolicyMemberDomains` org policy | 🚨 Critical |
| 4 | Require Workload Identity Federation for all external workloads | 🚨 Critical |
| 5 | Enable Admin Activity audit logging on all projects | 🚨 Critical |
| 6 | Alert on `SetIamPolicy` changes via log-based metrics | 🚨 Critical |
| 7 | One dedicated service account per workload/service | 🔴 High |
| 8 | Grant roles to Google Groups, not individual emails | 🔴 High |
| 9 | Apply IAM Recommender recommendations weekly | 🔴 High |
| 10 | Enable `storage.publicAccessPrevention` org policy | 🔴 High |
| 11 | Apply `gcp.resourceLocations` for data residency | 🔴 High |
| 12 | Enable Data Access audit logs for sensitive services (BQ, GCS, CloudSQL) | 🟡 Medium |
| 13 | Use VPC Service Controls for sensitive data projects | 🟡 Medium |
| 14 | Implement PAM (just-in-time access) for production admin roles | 🟡 Medium |
| 15 | Conduct quarterly IAM access reviews with Policy Analyzer | 🟡 Medium |

---

*Generated for GCP Cloud IAM | Official Docs: [cloud.google.com/iam/docs](https://cloud.google.com/iam/docs)*
