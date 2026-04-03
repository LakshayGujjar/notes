# GCP BeyondCorp Enterprise (BCE) — Comprehensive Cheatsheet

> **Audience:** Senior GCP engineers. No hand-holding — straight to operational depth.

---

## Table of Contents

1. [Overview & Core Philosophy](#1-overview--core-philosophy)
2. [Architecture & Components](#2-architecture--components)
3. [Identity & Access Layer](#3-identity--access-layer)
4. [Access Context Manager (ACM)](#4-access-context-manager-acm)
5. [Endpoint Verification & Device Trust](#5-endpoint-verification--device-trust)
6. [IAP Configuration & Code Examples](#6-iap-configuration--code-examples)
7. [Access Context Manager — Code Examples](#7-access-context-manager--code-examples)
8. [Terraform Examples](#8-terraform-examples)
9. [Workforce Identity Federation](#9-workforce-identity-federation)
10. [Workload Identity Federation](#10-workload-identity-federation)
11. [Context-Aware Access for Google Workspace & SaaS](#11-context-aware-access-for-google-workspace--saas)
12. [Threat & Data Protection](#12-threat--data-protection)
13. [IAP Connector — On-Premises & Hybrid](#13-iap-connector--on-premises--hybrid)
14. [Observability & Audit](#14-observability--audit)
15. [Security Hardening & Best Practices](#15-security-hardening--best-practices)
16. [Limits & Quotas](#16-limits--quotas)
17. [Common Gotchas & Troubleshooting](#17-common-gotchas--troubleshooting)
18. [Quick Reference — gcloud Commands](#18-quick-reference--gcloud-commands)
19. [Decision Tree](#19-decision-tree)

---

## 1. Overview & Core Philosophy

### What BeyondCorp Enterprise Solves

Traditional perimeter security assumes anything inside the corporate network is trusted. This collapses under:
- Remote/hybrid work (users are never "inside")
- Cloud-first architectures (no single network perimeter)
- Lateral movement after perimeter breach (trust is binary, not contextual)

**BeyondCorp Enterprise** is Google's enterprise Zero Trust platform that shifts access decisions from network location to **identity + device posture + context**, enforced at every request.

### Zero Trust Principles in BCE

| Principle | BCE Implementation |
|---|---|
| Never trust, always verify | IAP enforces auth on every request; no implicit network trust |
| Assume breach | VPC Service Controls isolates blast radius per resource |
| Least-privilege access | Access Levels + IAM conditions restrict beyond role alone |
| Continuous verification | Context re-evaluated per request (not per session) |
| Device health matters | Endpoint Verification feeds real-time device signals into policy |

### BCE vs. Traditional VPN

| Dimension | VPN | BeyondCorp Enterprise |
|---|---|---|
| Trust boundary | Network perimeter | Identity + device context per request |
| Access granularity | Network-level (IP/port) | Application/resource-level |
| Lateral movement risk | High (full network access after VPN auth) | Minimal (per-app, per-resource control) |
| User experience | VPN client required; latency overhead | Direct browser/app access; Google GFE proximity |
| Scalability | VPN concentrator bottleneck | Google's global GFE infrastructure |
| Visibility | Limited (NetFlow) | Full request-level audit logs |
| Device enforcement | Usually none or basic | Chrome-native, MDM-integrated device signals |

### BCE vs. BCRA vs. Cloud IAP — Disambiguation

| Product | Scope | Key Use Case |
|---|---|---|
| **Cloud IAP** | GCP resources only | Protect GCE VMs, App Engine, GKE, Cloud Run with OAuth 2.0 |
| **BeyondCorp Remote Access (BCRA)** | Deprecated/rebranded | Predecessor; functionality absorbed into BCE |
| **BeyondCorp Enterprise (BCE)** | GCP + on-prem + SaaS + Workspace | Full Zero Trust platform: IAP + ACM + Endpoint Verification + Chrome DLP + Threat Protection |

> **💡 Best Practice:** IAP is a component *within* BCE. "BeyondCorp Enterprise" is the full product suite. Use "IAP" when referring to the proxy layer specifically, "BCE" when discussing the holistic Zero Trust posture.

---

## 2. Architecture & Components

### Full BCE Request Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        BeyondCorp Enterprise Request Flow                │
└─────────────────────────────────────────────────────────────────────────┘

  ┌────────────────┐     HTTPS      ┌─────────────────────────────────────┐
  │  User + Device │────────────────▶│     Google Front End (GFE)          │
  │  (Chrome/App)  │                │  Global Anycast, TLS Termination     │
  └────────────────┘                └──────────────┬──────────────────────┘
         │                                         │
         │  Endpoint Verification                  │ Auth Challenge
         │  (device signals)                       ▼
         │                          ┌─────────────────────────────────────┐
         │                          │   Identity-Aware Proxy (IAP)        │
         │                          │                                     │
         │                          │  ① Validate Google Identity (OIDC)  │
         │                          │  ② Check IAP IAM bindings           │
         │                          │  ③ Extract JWT claims               │
         │                          └──────────────┬──────────────────────┘
         │                                         │
         │                                         │ Context check
         ▼                                         ▼
  ┌────────────────┐                ┌─────────────────────────────────────┐
  │ Chrome Endpoint│                │   Access Context Manager (ACM)      │
  │ Verification   │──device data──▶│                                     │
  │ Extension      │                │  ④ Evaluate Access Levels           │
  └────────────────┘                │     • IP subnetwork                 │
                                    │     • Device policy (MDM signals)   │
                                    │     • Region / OS version           │
                                    │     • CEL custom expressions        │
                                    │  ⑤ VPC Service Controls check       │
                                    └──────────────┬──────────────────────┘
                                                   │
                                        ALLOW / DENY decision
                                                   │
                          ┌────────────────────────┼──────────────────┐
                          │                        │                   │
                          ▼                        ▼                   ▼
               ┌──────────────────┐   ┌─────────────────┐  ┌────────────────────┐
               │  GCE / GKE /     │   │  App Engine /   │  │  On-Prem App via   │
               │  Cloud Run       │   │  Cloud Run      │  │  IAP Connector     │
               └──────────────────┘   └─────────────────┘  └────────────────────┘
```

### Component Breakdown

| Component | Role | Key Capability |
|---|---|---|
| **Google Front End (GFE)** | Global entry point | TLS termination, DDoS mitigation, anycast routing |
| **Identity-Aware Proxy (IAP)** | AuthN/AuthZ proxy | OAuth 2.0, JWT validation, TCP/HTTP/HTTPS tunneling |
| **Access Context Manager (ACM)** | Policy engine | Access Levels, Access Policies, VPC Service Controls |
| **Chrome Enterprise** | Browser trust anchor | Native endpoint signals, managed browser policy |
| **Endpoint Verification** | Device inventory | OS compliance, MDM status, screen lock, encryption |
| **Cloud Armor** | WAF / edge security | OWASP rule sets, rate limiting, geo-based blocking |
| **Certificate Authority Service (CAS)** | PKI | mTLS cert issuance for workload-to-workload auth |
| **BeyondCorp Threat & Data Protection** | Browser DLP + TI | Real-time content inspection, phishing/malware blocking |
| **Google SecOps (Chronicle)** | SIEM/SOAR | Centralized log ingestion, threat detection, playbooks |

### Request Lifecycle

```
① User opens app URL
② GFE receives request; routes to IAP
③ IAP checks: Is the user authenticated?
     No  → Redirect to Google Sign-In (OIDC flow)
     Yes → Validate ID token (audience = IAP client ID)
④ IAP checks: Does the user have roles/iap.httpsResourceAccessor?
     No  → 403 Forbidden
     Yes → Proceed
⑤ ACM evaluates bound Access Level:
     Fail → 403 with access level violation
     Pass → Forward request to backend
⑥ Backend receives request with injected headers:
     X-Goog-Authenticated-User-Email: accounts.google.com:user@example.com
     X-Goog-Authenticated-User-Id: accounts.google.com:1234567890
     X-Goog-Iap-Jwt-Assertion: <signed JWT>
```

> **⚠️ Gotcha:** Backends **must** validate the `X-Goog-Iap-Jwt-Assertion` header to prevent IAP bypass. If a backend is reachable directly (without going through IAP/LB), an attacker can spoof the `X-Goog-Authenticated-User-Email` header. Always firewall backends to only accept traffic from Google's GFE IP ranges.

---

## 3. Identity & Access Layer

### Google Identity Integration

| Identity Type | Mechanism | Notes |
|---|---|---|
| Google Workspace users | Native OIDC | Managed via Google Admin Console |
| Cloud Identity users | Native OIDC | Lightweight identity without Workspace apps |
| External users (personal Google) | OIDC | `google.com` domain; no organizational control |
| External IdP (SAML/OIDC) | Workforce Identity Federation | Pool + Provider config; see §9 |
| Service accounts | Service account credentials | Use Workload Identity Federation instead of keys |

### IAP-Secured Resource Types

| Resource Type | IAP Resource Type Flag | Notes |
|---|---|---|
| HTTPS Load Balancer backend | `backend-services` | Most common; GCE/GKE/NEGs |
| App Engine | `app-engine` | Per-service or default service |
| Cloud Run | Via LB backend (serverless NEG) | Cloud Run direct URL bypasses IAP |
| GKE Ingress | Via LB backend | Requires BackendConfig CR |
| TCP (SSH/RDP/custom ports) | `--tunnel-through-iap` | IAP TCP tunneling; no HTTP |
| On-prem apps | Via IAP Connector | Outbound-only tunnel from on-prem |

### IAP IAM Roles

| Role | Description | Scope |
|---|---|---|
| `roles/iap.httpsResourceAccessor` | Access IAP-protected HTTPS resource | Per backend, per project, or org |
| `roles/iap.tunnelResourceAccessor` | SSH/TCP tunnel through IAP | Per tunnel resource |
| `roles/iap.settingsAdmin` | Configure IAP settings | Project-level |
| `roles/iap.admin` | Full IAP admin including OAuth config | Project-level |
| `roles/iap.webServiceVersionViewer` | View IAP-protected App Engine versions | App Engine-specific |

> **💡 Best Practice:** Grant `roles/iap.httpsResourceAccessor` at the **backend-service level**, not the project level, to enforce least-privilege access per application.

---

## 4. Access Context Manager (ACM)

### Hierarchy

```
Organization
  └── Access Policy (one org-level + scoped policies)
        ├── Access Levels
        │     ├── Basic (IP, device policy, region, OS)
        │     └── Custom (CEL expressions)
        ├── Access Bindings (Access Level → Google Group)
        └── Service Perimeters (VPC Service Controls)
              ├── Restricted services
              ├── Ingress rules
              └── Egress rules
```

### Access Levels: Basic vs. Custom

**Basic Conditions (AND within a condition, OR across conditions):**

| Condition Attribute | Description | Example Value |
|---|---|---|
| `ipSubnetworks` | CIDR ranges for allowed source IPs | `["203.0.113.0/24", "10.0.0.0/8"]` |
| `regions` | ISO 3166-1 alpha-2 country codes | `["IN", "US", "DE"]` |
| `devicePolicy.requireScreenLock` | Screen lock must be enabled | `true` |
| `devicePolicy.allowedEncryptionStatuses` | Storage encryption state | `["ENCRYPTED"]` |
| `devicePolicy.requireCorpOwned` | Device must be corp-managed | `true` |
| `devicePolicy.requireAdminApproval` | Requires admin-approved device | `false` |
| `devicePolicy.allowedDeviceManagementLevels` | MDM level | `["COMPLETE"]` |
| `requiredAccessLevels` | Chained access level dependency | Other level names |

**Custom Conditions (CEL):**

```cel
# Example: Encrypted corp device in approved region, not compromised
device.encryption_status == DeviceEncryptionStatus.ENCRYPTED &&
device.is_admin_approved_device &&
device.management_state == DeviceManagementState.COMPLETE_DEVICE_MANAGEMENT &&
!device.is_compromised &&
origin.region_code in ["IN", "US", "DE", "GB"]
```

### Access Policies

| Type | Scope | Use Case |
|---|---|---|
| Org-level policy | All projects in org | Single policy for centralized governance |
| Scoped policy | Specific folders/projects | Delegated admin; team-level ACM management |

> **⚠️ Gotcha:** There is **exactly one org-level access policy** per organization. You cannot create a second one. Scoped policies are additive, not replacing the org policy.

### VPC Service Controls — Key Concepts

| Concept | Description |
|---|---|
| **Service Perimeter** | Logical security boundary around GCP projects/resources |
| **Restricted Services** | GCP APIs blocked from outside the perimeter (e.g., BigQuery, GCS, Spanner) |
| **Ingress Rules** | Allow specific identities/sources into the perimeter |
| **Egress Rules** | Allow specific identities/targets to reach outside the perimeter |
| **Dry-Run Mode** | Log violations without enforcement; safe for initial rollout |
| **Access Bridge** | Allows two perimeters to share resources (peering perimeters) |

### Comparison: Access Levels vs. VPC SC vs. IAP Conditions

| Feature | Access Levels | VPC Service Controls | IAP Conditions |
|---|---|---|---|
| Enforced at | IAP + CAA binding | GCP API layer (data plane) | IAP proxy (HTTP/TCP) |
| Protects | Application access | GCP API/data access | App frontend access |
| Scope | Per identity/group | Per project/resource | Per IAP resource |
| Works without IAP | Yes (CAA for Workspace) | Yes (independent) | N/A |
| Device enforcement | Yes | No | No |
| CEL support | Yes (custom levels) | Ingress/egress rules | Yes (IAM conditions) |

---

## 5. Endpoint Verification & Device Trust

### How Endpoint Verification Works

```
Chrome Browser
  └── Endpoint Verification Extension
        ├── Collects: OS type, OS version, screen lock, encryption status,
        │            managed browser, corp-owned flag, Chrome version
        ├── Sends signals to: Google Admin Console (Device Inventory)
        └── Signals used in: ACM Access Level device policy evaluation
```

### Device Trust Tiers

| Trust Level | Description | Requirements |
|---|---|---|
| **No verification** | Anonymous device; no signals collected | None |
| **Basic** | Device is registered in Google's inventory | EV Chrome extension installed, user signed in |
| **OS-verified** | Device meets OS-level compliance | MDM enrollment, screen lock enabled, storage encrypted |
| **Enterprise-verified** | Fully corp-managed device | Corporate MDM (Intune/Jamf/Workspace ONE), cert-based auth, admin-approved |

### Device Policy Attributes

| Attribute | Type | Description |
|---|---|---|
| `isCompromised` | bool | Device flagged as compromised by MDM |
| `screenLockEnabled` | bool | Screen lock/PIN required on device |
| `encryptedStorageEnabled` | bool | Full-disk encryption active |
| `osType` | enum | `DESKTOP_CHROME_OS`, `DESKTOP_WINDOWS`, `DESKTOP_MAC`, `ANDROID`, `IOS` |
| `osVersion` | string | Minimum OS version required (e.g., `"14.0"`) |
| `requireAdminApproval` | bool | Admin must explicitly approve device |
| `requireCorpOwned` | bool | Device must be in corp inventory |
| `allowedDeviceManagementLevels` | enum | `NONE`, `BASIC`, `COMPLETE` |

### MDM Integration Matrix

| MDM Platform | Integration Method | Signals Available |
|---|---|---|
| **Google Workspace MDM** | Native; no config needed | All attributes |
| **Microsoft Intune** | Endpoint Verification + Intune connector | `isCompromised`, `requireCorpOwned`, management level |
| **Jamf Pro** | Endpoint Verification + Jamf connector | `requireCorpOwned`, `encryptedStorageEnabled`, `osVersion` |
| **VMware Workspace ONE** | Endpoint Verification + WS1 connector | `requireCorpOwned`, management level |
| **Other MDMs** | Chrome Browser Cloud Management | Limited signals; OS-level data only |

> **💡 Best Practice:** For enterprise environments, always configure **at minimum** `requireScreenLock: true` and `allowedEncryptionStatuses: [ENCRYPTED]` in your production access levels. Anonymous device access to production resources is a critical Zero Trust violation.

---

## 6. IAP Configuration & Code Examples

### Enable IAP and Configure Backend (gcloud)

```bash
# Enable IAP API
gcloud services enable iap.googleapis.com --project=PROJECT_ID

# Enable IAP on a Global HTTPS Load Balancer backend service
gcloud compute backend-services update BACKEND_SERVICE_NAME \
  --global \
  --iap=enabled,oauth2-client-id=CLIENT_ID,oauth2-client-secret=CLIENT_SECRET

# Enable IAP on a Regional HTTPS Load Balancer backend service
gcloud compute backend-services update BACKEND_SERVICE_NAME \
  --region=REGION \
  --iap=enabled,oauth2-client-id=CLIENT_ID,oauth2-client-secret=CLIENT_SECRET

# Grant IAP HTTPS access to a user
gcloud iap web add-iam-policy-binding \
  --resource-type=backend-services \
  --service=BACKEND_SERVICE_NAME \
  --member="user:user@example.com" \
  --role="roles/iap.httpsResourceAccessor"

# Grant IAP access to a Google Group
gcloud iap web add-iam-policy-binding \
  --resource-type=backend-services \
  --service=BACKEND_SERVICE_NAME \
  --member="group:eng-team@example.com" \
  --role="roles/iap.httpsResourceAccessor"

# Grant IAP access with an access level condition
gcloud iap web add-iam-policy-binding \
  --resource-type=backend-services \
  --service=BACKEND_SERVICE_NAME \
  --member="group:eng-team@example.com" \
  --role="roles/iap.httpsResourceAccessor" \
  --condition="expression=request.auth.access_levels.exists(x, x=='accessPolicies/POLICY_ID/accessLevels/LEVEL_ID'),title=TrustedDevice"
```

### IAP TCP Tunneling (SSH & Port Forwarding)

```bash
# SSH to a VM through IAP (no external IP required)
gcloud compute ssh VM_NAME \
  --tunnel-through-iap \
  --project=PROJECT_ID \
  --zone=ZONE

# SSH with custom flags via IAP
gcloud compute ssh VM_NAME \
  --tunnel-through-iap \
  --project=PROJECT_ID \
  --zone=ZONE \
  -- -L 5432:localhost:5432  # Forward local 5432 → VM's PostgreSQL

# RDP tunnel (Windows VM)
gcloud compute start-iap-tunnel VM_NAME 3389 \
  --local-host-port=localhost:3389 \
  --project=PROJECT_ID \
  --zone=ZONE
# Then connect RDP client to localhost:3389

# Custom port tunnel (e.g., internal app on port 8080)
gcloud compute start-iap-tunnel VM_NAME 8080 \
  --local-host-port=localhost:8080 \
  --project=PROJECT_ID \
  --zone=ZONE

# Grant IAP TCP tunnel access
gcloud compute instances add-iam-policy-binding VM_NAME \
  --zone=ZONE \
  --member="user:user@example.com" \
  --role="roles/iap.tunnelResourceAccessor"
```

> **⚠️ Gotcha:** For IAP TCP to work, your GCE firewall must allow ingress from `35.235.240.0/20` on the target port. Without this rule, IAP TCP tunnels silently fail at the connection stage.

```bash
# Required firewall rule for IAP TCP
gcloud compute firewall-rules create allow-iap-tcp \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22,tcp:3389 \
  --source-ranges=35.235.240.0/20 \
  --network=VPC_NETWORK_NAME
```

### IAP JWT Validation (Backend — Python)

```python
# Validate the IAP JWT in your backend to prevent header spoofing
import jwt
import requests
from cachetools import TTLCache

_KEY_CACHE = TTLCache(maxsize=10, ttl=3600)

def validate_iap_jwt(iap_jwt: str, expected_audience: str) -> dict:
    """
    Validates an IAP-issued JWT.
    expected_audience format: /projects/PROJECT_NUMBER/apps/PROJECT_ID (App Engine)
                              /projects/PROJECT_NUMBER/global/backendServices/SERVICE_ID (GCE)
    """
    certs_url = "https://www.gstatic.com/iap/verify/public_key"
    
    if certs_url not in _KEY_CACHE:
        _KEY_CACHE[certs_url] = requests.get(certs_url).json()
    
    certs = _KEY_CACHE[certs_url]
    
    # Decode header to get kid
    header = jwt.get_unverified_header(iap_jwt)
    key_id = header.get("kid")
    
    if key_id not in certs:
        raise ValueError(f"Key ID {key_id} not found in IAP public keys")
    
    public_key = certs[key_id]
    
    payload = jwt.decode(
        iap_jwt,
        public_key,
        algorithms=["ES256"],
        audience=expected_audience,
    )
    return payload


# Usage in a Flask app
from flask import Flask, request, abort

app = Flask(__name__)
IAP_AUDIENCE = "/projects/PROJECT_NUMBER/global/backendServices/BACKEND_SERVICE_ID"

@app.before_request
def verify_iap():
    iap_jwt = request.headers.get("X-Goog-Iap-Jwt-Assertion")
    if not iap_jwt:
        abort(403, "Missing IAP JWT")
    try:
        payload = validate_iap_jwt(iap_jwt, IAP_AUDIENCE)
        request.user_email = payload.get("email")
    except Exception as e:
        abort(403, f"Invalid IAP JWT: {e}")
```

### Programmatic IAP Request (Service-to-Service)

```python
import google.oauth2.id_token
import google.auth.transport.requests
import requests

IAP_CLIENT_ID = "YOUR_IAP_OAUTH_CLIENT_ID"
IAP_PROTECTED_URL = "https://your-app.example.com/api/resource"

def make_iap_request(url: str, client_id: str, method: str = "GET", **kwargs) -> requests.Response:
    """
    Makes an authenticated request to an IAP-protected resource using ADC.
    The service account running this code must have roles/iap.httpsResourceAccessor.
    """
    auth_req = google.auth.transport.requests.Request()
    # fetch_id_token uses ADC; works with service accounts, Workload Identity, etc.
    token = google.oauth2.id_token.fetch_id_token(auth_req, client_id)
    headers = kwargs.pop("headers", {})
    headers["Authorization"] = f"Bearer {token}"
    response = requests.request(method, url, headers=headers, **kwargs)
    response.raise_for_status()
    return response

# Example usage
resp = make_iap_request(IAP_PROTECTED_URL, IAP_CLIENT_ID)
print(resp.json())
```

> **💡 Best Practice:** Use **Application Default Credentials (ADC)** + `fetch_id_token`. Never hardcode service account JSON keys. In GKE, use Workload Identity; in Cloud Run/GCE, use the attached service account.

### BackendConfig for IAP on GKE

```yaml
# k8s/backend-config.yaml
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: my-app-iap-config
  namespace: default
spec:
  iap:
    enabled: true
    oauthclientCredentials:
      secretName: iap-oauth-secret  # k8s Secret with client_id and client_secret
---
# k8s/service.yaml — annotate the Service to use the BackendConfig
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
  annotations:
    cloud.google.com/backend-config: '{"default": "my-app-iap-config"}'
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# Create the OAuth secret in k8s
kubectl create secret generic iap-oauth-secret \
  --from-literal=client_id=CLIENT_ID \
  --from-literal=client_secret=CLIENT_SECRET
```

---

## 7. Access Context Manager — Code Examples

### Access Level Management (gcloud)

```bash
# ── Access Policy ────────────────────────────────────────────────────────────

# Get the org-level access policy ID
gcloud access-context-manager policies list --organization=ORG_ID

# Create a scoped access policy (delegated admin)
gcloud access-context-manager policies create \
  --organization=ORG_ID \
  --scopes=projects/PROJECT_ID \
  --title="Project Scoped Policy"

# ── Basic Access Level ────────────────────────────────────────────────────────

# conditions.yaml
cat > /tmp/conditions.yaml << 'EOF'
conditions:
- ipSubnetworks:
  - 203.0.113.0/24
  - 10.100.0.0/16
  devicePolicy:
    requireScreenLock: true
    requireAdminApproval: false
    requireCorpOwned: true
    allowedEncryptionStatuses:
    - ENCRYPTED
    allowedDeviceManagementLevels:
    - COMPLETE
  regions:
  - IN
  - US
  - DE
EOF

gcloud access-context-manager levels create TRUSTED_CORP_LEVEL \
  --policy=POLICY_ID \
  --title="Trusted Corporate Devices" \
  --basic-level-spec=/tmp/conditions.yaml \
  --combine-function=AND

# ── Custom (CEL) Access Level ─────────────────────────────────────────────────

cat > /tmp/cel_spec.yaml << 'EOF'
expression: >
  device.encryption_status == DeviceEncryptionStatus.ENCRYPTED &&
  device.management_state == DeviceManagementState.COMPLETE_DEVICE_MANAGEMENT &&
  !device.is_compromised &&
  device.is_admin_approved_device &&
  origin.region_code in ["IN", "US", "DE", "GB", "SG"]
EOF

gcloud access-context-manager levels create CUSTOM_CEL_LEVEL \
  --policy=POLICY_ID \
  --title="CEL-Enforced Device Trust" \
  --custom-level-spec=/tmp/cel_spec.yaml

# ── List & Describe ───────────────────────────────────────────────────────────

gcloud access-context-manager levels list --policy=POLICY_ID

gcloud access-context-manager levels describe TRUSTED_CORP_LEVEL \
  --policy=POLICY_ID \
  --format=yaml

# ── Delete ────────────────────────────────────────────────────────────────────

gcloud access-context-manager levels delete TRUSTED_CORP_LEVEL \
  --policy=POLICY_ID
```

### VPC Service Controls — Perimeter Lifecycle

```bash
# ── Create perimeter in DRY-RUN mode (safe first step) ────────────────────────

gcloud access-context-manager perimeters dry-run create PROD_DATA_PERIMETER \
  --policy=POLICY_ID \
  --title="Production Data Perimeter" \
  --type=PERIMETER_TYPE_REGULAR \
  --resources=projects/PROJECT_NUMBER_1,projects/PROJECT_NUMBER_2 \
  --restricted-services=bigquery.googleapis.com,storage.googleapis.com,spanner.googleapis.com \
  --access-levels=accessPolicies/POLICY_ID/accessLevels/TRUSTED_CORP_LEVEL

# ── Check what dry-run would block ────────────────────────────────────────────

# Filter logs for VPC SC dry-run violations (won't block, just logs)
# Use Cloud Logging filter:
# resource.type="audited_resource"
# protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
# protoPayload.metadata.vpcServiceControlsUniqueId:*
# severity="ERROR"  ← for dry-run violations

# ── Enforce the perimeter ────────────────────────────────────────────────────

gcloud access-context-manager perimeters dry-run enforce PROD_DATA_PERIMETER \
  --policy=POLICY_ID

# ── Add ingress rule (allow specific SA from outside perimeter) ────────────────

cat > /tmp/ingress_policy.yaml << 'EOF'
- ingressFrom:
    identities:
    - serviceAccount:etl-pipeline-sa@PROJECT_ID.iam.gserviceaccount.com
    sources:
    - accessLevel: accessPolicies/POLICY_ID/accessLevels/TRUSTED_CORP_LEVEL
  ingressTo:
    resources:
    - '*'
    operations:
    - serviceName: bigquery.googleapis.com
      methodSelectors:
      - method: '*'
EOF

gcloud access-context-manager perimeters update PROD_DATA_PERIMETER \
  --policy=POLICY_ID \
  --set-ingress-policies=/tmp/ingress_policy.yaml

# ── Add egress rule (allow data export to specific destination) ────────────────

cat > /tmp/egress_policy.yaml << 'EOF'
- egressFrom:
    identities:
    - serviceAccount:data-export-sa@PROJECT_ID.iam.gserviceaccount.com
  egressTo:
    resources:
    - projects/DESTINATION_PROJECT_NUMBER
    operations:
    - serviceName: storage.googleapis.com
      methodSelectors:
      - method: google.storage.objects.create
EOF

gcloud access-context-manager perimeters update PROD_DATA_PERIMETER \
  --policy=POLICY_ID \
  --set-egress-policies=/tmp/egress_policy.yaml

# ── Describe perimeter ────────────────────────────────────────────────────────

gcloud access-context-manager perimeters describe PROD_DATA_PERIMETER \
  --policy=POLICY_ID \
  --format=yaml
```

> **⚠️ Gotcha:** Enforcing a VPC Service Controls perimeter **without proper ingress rules** will lock out legitimate service accounts and automation pipelines. Always run in dry-run mode for at least 48 hours and analyze violations in Cloud Logging before enforcing.

---

## 8. Terraform Examples

### IAP OAuth Brand & Client

```hcl
# Only one IAP brand per project allowed. This resource is NOT importable if
# already created via Console — you must import it manually.
resource "google_iap_brand" "project_brand" {
  support_email     = "support@example.com"
  application_title = "MyApp — BeyondCorp Protected"
  project           = var.project_id
}

resource "google_iap_client" "project_client" {
  display_name = "MyApp IAP OAuth Client"
  brand        = google_iap_brand.project_brand.name
}

# Store client secret in Secret Manager
resource "google_secret_manager_secret" "iap_client_secret" {
  secret_id = "iap-oauth-client-secret"
  project   = var.project_id

  replication {
    auto {}
  }
}

resource "google_secret_manager_secret_version" "iap_client_secret_v" {
  secret      = google_secret_manager_secret.iap_client_secret.id
  secret_data = google_iap_client.project_client.secret
}
```

### HTTPS Load Balancer with IAP

```hcl
# Backend service with IAP enabled
resource "google_compute_backend_service" "app_backend" {
  name        = "my-app-backend"
  project     = var.project_id
  protocol    = "HTTP"
  timeout_sec = 30

  backend {
    group = google_compute_instance_group_manager.app.instance_group
  }

  health_checks = [google_compute_health_check.app.id]

  iap {
    enabled              = true
    oauth2_client_id     = google_iap_client.project_client.client_id
    oauth2_client_secret = google_iap_client.project_client.secret
  }

  log_config {
    enable      = true
    sample_rate = 1.0
  }
}

# IAP IAM binding for a group
resource "google_iap_web_backend_service_iam_binding" "iap_access" {
  project             = var.project_id
  web_backend_service = google_compute_backend_service.app_backend.name
  role                = "roles/iap.httpsResourceAccessor"

  members = [
    "group:eng-team@example.com",
    "group:qa-team@example.com",
  ]
}

# IAP IAM binding WITH access level condition (context-aware)
resource "google_iap_web_backend_service_iam_binding" "iap_access_prod" {
  project             = var.project_id
  web_backend_service = google_compute_backend_service.app_backend.name
  role                = "roles/iap.httpsResourceAccessor"

  members = ["group:sre-team@example.com"]

  condition {
    title       = "TrustedCorpDevice"
    description = "Only allow access from managed corp devices"
    expression  = "request.auth.access_levels.exists(x, x == 'accessPolicies/${var.access_policy_id}/accessLevels/corp_trusted')"
  }
}
```

### IAP Tunnel Resource IAM (for SSH/TCP)

```hcl
resource "google_iap_tunnel_instance_iam_binding" "ssh_access" {
  project  = var.project_id
  zone     = var.zone
  instance = google_compute_instance.bastion.name
  role     = "roles/iap.tunnelResourceAccessor"

  members = [
    "group:infra-team@example.com",
  ]
}
```

### Access Context Manager Access Level

```hcl
resource "google_access_context_manager_access_level" "corp_trusted" {
  parent = "accessPolicies/${var.access_policy_id}"
  name   = "accessPolicies/${var.access_policy_id}/accessLevels/corp_trusted"
  title  = "Corporate Trusted Devices"

  basic {
    combining_function = "AND"

    conditions {
      ip_subnetworks = ["203.0.113.0/24", "10.100.0.0/16"]
      regions        = ["IN", "US", "DE"]

      device_policy {
        require_screen_lock            = true
        require_corp_owned             = true
        allowed_encryption_statuses    = ["ENCRYPTED"]
        allowed_device_management_levels = ["COMPLETE"]
      }
    }
  }
}

# Custom CEL-based access level
resource "google_access_context_manager_access_level" "cel_trusted" {
  parent = "accessPolicies/${var.access_policy_id}"
  name   = "accessPolicies/${var.access_policy_id}/accessLevels/cel_trusted"
  title  = "CEL — Encrypted Managed Device"

  custom {
    expr {
      expression = <<-CEL
        device.encryption_status == DeviceEncryptionStatus.ENCRYPTED &&
        device.management_state == DeviceManagementState.COMPLETE_DEVICE_MANAGEMENT &&
        !device.is_compromised &&
        origin.region_code in ["IN", "US", "DE"]
      CEL
    }
  }
}
```

### VPC Service Controls Service Perimeter

```hcl
resource "google_access_context_manager_service_perimeter" "data_perimeter" {
  parent = "accessPolicies/${var.access_policy_id}"
  name   = "accessPolicies/${var.access_policy_id}/servicePerimeters/data_perimeter"
  title  = "Production Data Perimeter"
  type   = "PERIMETER_TYPE_REGULAR"

  # use_explicit_dry_run_spec = true  # Uncomment for dry-run enforcement

  status {
    resources = [
      "projects/${var.prod_project_number}",
      "projects/${var.data_project_number}",
    ]

    restricted_services = [
      "bigquery.googleapis.com",
      "storage.googleapis.com",
      "spanner.googleapis.com",
      "secretmanager.googleapis.com",
    ]

    access_levels = [
      google_access_context_manager_access_level.corp_trusted.name,
    ]

    # Ingress: allow ETL SA from any source with corp access level
    ingress_policies {
      ingress_from {
        identity_type = "SERVICE_ACCOUNTS"
        identities    = ["serviceAccount:etl-sa@${var.project_id}.iam.gserviceaccount.com"]
        sources {
          access_level = google_access_context_manager_access_level.corp_trusted.name
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

    # Egress: allow export SA to write to archive bucket in another project
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
        }
      }
    }
  }
}
```

### Workload Identity Federation (Terraform)

```hcl
resource "google_iam_workload_identity_pool" "github_pool" {
  workload_identity_pool_id = "github-actions-pool"
  project                   = var.project_id
  display_name              = "GitHub Actions"
}

resource "google_iam_workload_identity_pool_provider" "github_provider" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github_pool.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-oidc"
  project                            = var.project_id
  display_name                       = "GitHub OIDC"

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }

  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.actor"      = "assertion.actor"
    "attribute.repository" = "assertion.repository"
    "attribute.ref"        = "assertion.ref"
  }

  attribute_condition = "assertion.repository == 'my-org/my-repo'"
}

resource "google_service_account_iam_binding" "github_wif_binding" {
  service_account_id = google_service_account.deploy_sa.name
  role               = "roles/iam.workloadIdentityUser"

  members = [
    "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github_pool.name}/attribute.repository/my-org/my-repo",
  ]
}
```

---

## 9. Workforce Identity Federation

### Key Concepts

| Concept | Description |
|---|---|
| **Workforce Pool** | Container for external identities at org level |
| **Provider** | OIDC or SAML 2.0 IdP configuration within a pool |
| **Attribute Mapping** | Maps IdP token claims to Google attributes (`google.subject`, `google.groups`, `attribute.*`) |
| **Attribute Condition** | CEL expression to restrict which federated identities are allowed |
| **Principal** | `principal://iam.googleapis.com/locations/global/workforcePools/POOL/subject/SUB` |
| **Principal Set** | `principalSet://...` — wildcard matching groups/attributes |

### Supported IdPs

| IdP | Protocol | Notes |
|---|---|---|
| Azure Active Directory | OIDC or SAML 2.0 | Use OIDC for group claims; SAML for richer attributes |
| Okta | OIDC or SAML 2.0 | Add `groups` scope in Okta app for group claims |
| Ping Identity | SAML 2.0 | Upload metadata XML |
| ADFS | SAML 2.0 | Requires ADFS metadata endpoint |
| Any OIDC 1.0 IdP | OIDC | Must expose `/.well-known/openid-configuration` |
| Any SAML 2.0 IdP | SAML 2.0 | Upload metadata XML |

### Setup (gcloud)

```bash
# ── Create Workforce Pool ─────────────────────────────────────────────────────

gcloud iam workforce-pools create POOL_ID \
  --organization=ORG_ID \
  --location=global \
  --display-name="Corporate Workforce Pool" \
  --description="Federated identities from Okta"

# ── Create OIDC Provider (Okta example) ──────────────────────────────────────

gcloud iam workforce-pools providers create-oidc PROVIDER_ID \
  --workforce-pool=POOL_ID \
  --location=global \
  --display-name="Okta OIDC Provider" \
  --issuer-uri="https://YOUR_ORG.okta.com" \
  --client-id="OKTA_CLIENT_ID" \
  --client-secret-value="OKTA_CLIENT_SECRET" \
  --web-sso-response-type=code \
  --web-sso-assertion-claims-behavior=merge-user-info-over-id-token-claims \
  --attribute-mapping="google.subject=assertion.sub,google.display_name=assertion.name,google.groups=assertion.groups,attribute.department=assertion.department,attribute.cost_center=assertion.cost_center" \
  --attribute-condition="'gcp-users' in assertion.groups"

# ── Create SAML Provider (Azure AD example) ───────────────────────────────────

gcloud iam workforce-pools providers create-saml SAML_PROVIDER_ID \
  --workforce-pool=POOL_ID \
  --location=global \
  --display-name="Azure AD SAML" \
  --idp-metadata-path=/path/to/azure-ad-metadata.xml \
  --attribute-mapping="google.subject=assertion.subject,google.groups=assertion.attributes['http://schemas.microsoft.com/ws/2008/06/identity/claims/groups'],attribute.department=assertion.attributes.department" \
  --attribute-condition="assertion.subject.endsWith('@example.com')"

# ── Grant IAM roles to federated users ───────────────────────────────────────

# Single user
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="principal://iam.googleapis.com/locations/global/workforcePools/POOL_ID/subject/USER_SUBJECT" \
  --role="roles/viewer"

# All users in a group (groups must be in attribute mapping)
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="principalSet://iam.googleapis.com/locations/global/workforcePools/POOL_ID/group/gcp-admins" \
  --role="roles/editor"

# All users with a specific attribute value
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="principalSet://iam.googleapis.com/locations/global/workforcePools/POOL_ID/attribute.department/engineering" \
  --role="roles/compute.viewer"

# All users in the pool
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="principalSet://iam.googleapis.com/locations/global/workforcePools/POOL_ID/*" \
  --role="roles/logging.viewer"
```

> **⚠️ Gotcha:** Workforce Identity Federation identities are **not** Google accounts. They cannot be used to log into `console.cloud.google.com` directly — they must use the workforce pool sign-in URL: `https://console.cloud.google.com/auth/federation/signin?pool_id=POOL_ID&organization_id=ORG_ID`

---

## 10. Workload Identity Federation

### Architecture

```
External Workload                        Google Cloud
(GitHub Actions / AWS EC2 / Azure VM)
                                         ┌──────────────────────────────┐
┌──────────────────┐   OIDC/SAML token   │  Workload Identity Pool      │
│  CI/CD Pipeline  │────────────────────▶│  Provider (OIDC/AWS/SAML)    │
│  or External VM  │                     │  ↓ token exchange            │
└──────────────────┘                     │  STS short-lived token       │
                                         │  ↓ impersonate SA            │
                                         │  Service Account             │
                                         │  ↓ access GCP APIs           │
                                         └──────────────────────────────┘
```

### Setup (gcloud)

```bash
# ── Workload Identity Pool ────────────────────────────────────────────────────

gcloud iam workload-identity-pools create POOL_ID \
  --project=PROJECT_ID \
  --location=global \
  --display-name="External Workloads"

# ── GitHub Actions OIDC Provider ─────────────────────────────────────────────

gcloud iam workload-identity-pools providers create-oidc PROVIDER_ID \
  --workload-identity-pool=POOL_ID \
  --project=PROJECT_ID \
  --location=global \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --allowed-audiences="https://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/providers/PROVIDER_ID" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.ref=assertion.ref,attribute.event_name=assertion.event_name" \
  --attribute-condition="attribute.repository=='my-org/my-repo' && attribute.ref=='refs/heads/main'"

# ── AWS Provider ──────────────────────────────────────────────────────────────

gcloud iam workload-identity-pools providers create-aws AWS_PROVIDER_ID \
  --workload-identity-pool=POOL_ID \
  --project=PROJECT_ID \
  --location=global \
  --account-id="AWS_ACCOUNT_ID" \
  --attribute-mapping="google.subject=assertion.arn,attribute.aws_role=assertion.arn.extract('assumed-role/{role}/')" \
  --attribute-condition="attribute.aws_role=='my-deployment-role'"

# ── Bind SA to external identity ─────────────────────────────────────────────

gcloud iam service-accounts add-iam-policy-binding DEPLOY_SA@PROJECT_ID.iam.gserviceaccount.com \
  --project=PROJECT_ID \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/attribute.repository/my-org/my-repo"
```

### GitHub Actions Workflow (No Keys)

```yaml
# .github/workflows/deploy.yaml
name: Deploy to GCP

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write  # Required for OIDC token generation

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v2
      with:
        workload_identity_provider: "projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/providers/PROVIDER_ID"
        service_account: "deploy-sa@PROJECT_ID.iam.gserviceaccount.com"

    - name: Deploy
      run: gcloud run deploy my-service --image gcr.io/PROJECT_ID/my-image --region us-central1
```

> **💡 Best Practice:** Scope the `attribute-condition` to a specific branch (`assertion.ref == 'refs/heads/main'`) to prevent any branch in the repository from impersonating the deploy service account.

---

## 11. Context-Aware Access for Google Workspace & SaaS

### How CAA Extends BCE to Workspace

```
Google Admin Console
  └── Security → Access & Data Control → Context-Aware Access
        ├── Access Levels (from ACM — shared with BCE)
        └── Access Bindings
              ├── Google Group: sre-team@example.com
              └── Access Level: corp_trusted
                    → Enforced on: Gmail, Drive, Docs, Meet, Calendar
```

### CAA Configuration (gcloud)

```bash
# Create an access binding (group ↔ access level ↔ Workspace enforcement)
gcloud access-context-manager cloud-bindings create \
  --group-key=sre-team@example.com \
  --policy=POLICY_ID \
  --level=accessPolicies/POLICY_ID/accessLevels/corp_trusted

# List existing bindings
gcloud access-context-manager cloud-bindings list --policy=POLICY_ID

# Update a binding
gcloud access-context-manager cloud-bindings update BINDING_ID \
  --policy=POLICY_ID \
  --level=accessPolicies/POLICY_ID/accessLevels/higher_trust_level

# Delete a binding
gcloud access-context-manager cloud-bindings delete BINDING_ID \
  --policy=POLICY_ID
```

### CAA Scope

| Resource Type | CAA Enforced? | Notes |
|---|---|---|
| Gmail | ✅ Yes | Browser + mobile apps |
| Google Drive | ✅ Yes | Web; mobile enforcement requires MDM |
| Google Docs/Sheets/Slides | ✅ Yes | Per-binding group |
| Google Meet | ✅ Yes | |
| Google Admin Console | ✅ Yes | Admin access control |
| SAML-federated SaaS apps | ✅ Yes | Apps configured in Cloud Identity |
| Cloud Console (GCP) | ✅ Yes | Via Cloud Identity IdP |
| Non-Google SaaS (Salesforce, Workday) | ✅ If SAML-federated via Cloud Identity | Requires Cloud Identity Premium |

> **⚠️ Gotcha:** CAA access bindings apply to the **entire group**. If a user is in multiple groups with conflicting access levels, the **most permissive** binding wins. Design your groups carefully and use restrictive access levels rather than relying on group exclusions.

---

## 12. Threat & Data Protection

### Chrome Enterprise Integration

```
Chrome Browser (Managed)
  ├── Chrome Enterprise Connectors
  │     ├── Content Analysis Connector → DLP scanning (upload/download/paste/print)
  │     ├── Real-time URL filtering → Blocks phishing/malware URLs
  │     └── Extension risk assessment → Blocks risky extensions
  ├── Safe Browsing (Enterprise) → Enhanced phishing/malware protection
  └── Reporting → Chrome browser data → Google SecOps (Chronicle)
```

### DLP Rules Scope

| Action Intercepted | DLP Applicable | Example Policy |
|---|---|---|
| File upload to web | ✅ | Block PII upload to personal cloud storage |
| File download | ✅ | Warn on confidential file download |
| Copy-paste content | ✅ | Block copy of credit card patterns |
| Print | ✅ | Block print of classified documents |
| Screenshot | ✅ (managed device) | Warn on screenshot of sensitive apps |
| Web form submit | ✅ | Block SSN submission to non-approved domains |

### Integration with Google SecOps (Chronicle)

```
BeyondCorp Events  ──────────────────▶  Google SecOps (Chronicle)
  ├── IAP access logs                     ├── Threat detection (YARA-L rules)
  ├── ACM policy violations               ├── SOAR playbooks (auto-remediation)
  ├── VPC SC violations                   ├── UBA (user behavior analytics)
  ├── Chrome DLP events                   └── Investigation timelines
  └── Endpoint Verification events
```

---

## 13. IAP Connector — On-Premises & Hybrid

### Architecture

```
Internet / User
      │ HTTPS (authenticated via IAP)
      ▼
┌──────────────────────────────┐
│   Google GFE + IAP           │
│   (Identity enforcement)     │
└──────────────┬───────────────┘
               │ Encrypted tunnel (outbound from on-prem)
               ▼
┌──────────────────────────────┐      ┌──────────────────────────────┐
│   IAP Connector              │────▶ │   On-Premises App            │
│   (GKE pod / GCE VM)        │      │   (internal-app.corp:443)    │
│   connector.cloudproxy.app   │      └──────────────────────────────┘
└──────────────────────────────┘

Key: The connector establishes OUTBOUND connections to Google.
     No inbound ports need to be opened on the corporate firewall.
```

### Connector Deployment (Helm on GKE)

```bash
# ── Prerequisites ─────────────────────────────────────────────────────────────

# 1. Create a service account for the connector
gcloud iam service-accounts create iap-connector-sa \
  --display-name="IAP Connector SA" \
  --project=PROJECT_ID

# 2. Grant required roles to the SA
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:iap-connector-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/iap.httpsResourceAccessor"

# 3. Bind Workload Identity (for GKE-based connector)
gcloud iam service-accounts add-iam-policy-binding iap-connector-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[iap-connector/iap-connector]"

# ── Helm Install ──────────────────────────────────────────────────────────────

helm repo add iap-connector https://storage.googleapis.com/iap-connector-helm/
helm repo update

helm install iap-connector iap-connector/iap-connector \
  --namespace iap-connector \
  --create-namespace \
  --set connector.projectId=PROJECT_ID \
  --set connector.tunnelEndpoint="tunnel.cloudproxy.app:443" \
  --set backend.host=internal-app.corp.local \
  --set backend.port=443 \
  --set backend.protocol=HTTPS \
  --set serviceAccount.annotations."iam\.gke\.io/gcp-service-account"="iap-connector-sa@PROJECT_ID.iam.gserviceaccount.com"

# ── Register connector with a backend service ─────────────────────────────────
# After connector is running, create a serverless NEG pointing to the connector URL
# and attach it to an HTTPS Load Balancer with IAP enabled
```

> **💡 Best Practice:** Run multiple connector replicas (`--set replicaCount=3`) behind a GKE Service for HA. The connector maintains stateless outbound WebSocket connections to Google's infrastructure — losing one replica does not terminate active sessions.

---

## 14. Observability & Audit

### Key Log Types

| Log Type | Log Name (Cloud Logging) | Key Fields | Purpose |
|---|---|---|---|
| IAP access | `cloudaudit.googleapis.com/data_access` | `callerIp`, `principalEmail`, `resourceName`, `status.code` | Audit who accessed what |
| IAP config changes | `cloudaudit.googleapis.com/activity` | `methodName`, `request`, `principalEmail` | Track IAP setting changes |
| ACM policy changes | `cloudaudit.googleapis.com/activity` | `methodName`, `request.accessLevel`, `request.perimeter` | Track access level/policy changes |
| VPC SC violations | `cloudaudit.googleapis.com/policy` | `servicePerimeter`, `violationReason`, `callerIp` | Detect unauthorized API access |
| VPC SC dry-run | `cloudaudit.googleapis.com/policy` | `dryRun: true`, `violationReason` | Pre-enforcement validation |
| Endpoint Verification | Admin Console Reports API | `deviceId`, `osVersion`, `policyStatus`, `userId` | Device compliance tracking |

### Cloud Logging Queries

```bash
# ── IAP: All access denials (403/PERMISSION_DENIED) ──────────────────────────
resource.type="gce_backend_service"
protoPayload.serviceName="iap.googleapis.com"
protoPayload.status.code=7

# ── IAP: Access by specific user ─────────────────────────────────────────────
resource.type="gce_backend_service"
protoPayload.authenticationInfo.principalEmail="user@example.com"

# ── IAP: TCP tunnel connections ───────────────────────────────────────────────
resource.type="gce_instance"
protoPayload.methodName="AuthorizeUser"
protoPayload.request.@type="type.googleapis.com/google.cloud.iap.v1.TunnelDestGroup"

# ── VPC SC: Enforced violations ───────────────────────────────────────────────
protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
protoPayload.status.code!=0
NOT protoPayload.metadata.dryRun=true

# ── VPC SC: Dry-run violations only ──────────────────────────────────────────
protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
protoPayload.metadata.dryRun=true

# ── ACM: Access level evaluation failures ─────────────────────────────────────
resource.type="audited_resource"
protoPayload.authorizationInfo.granted=false
protoPayload.metadata.accessLevels:*

# ── ACM: Policy or access level modifications ─────────────────────────────────
resource.type="audited_resource"
protoPayload.serviceName="accesscontextmanager.googleapis.com"
protoPayload.methodName=~"Create|Update|Delete"

# ── All BCE-related denials (combined) ────────────────────────────────────────
(
  protoPayload.serviceName="iap.googleapis.com" AND protoPayload.status.code=7
) OR (
  protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
  AND protoPayload.status.code!=0
  AND NOT protoPayload.metadata.dryRun=true
)
```

### Key Cloud Monitoring Metrics

| Metric | Description | Alert Threshold Suggestion |
|---|---|---|
| `iap.googleapis.com/https/request_count` | Total IAP-proxied HTTP requests | Spike > 2x baseline (potential DDoS) |
| `iap.googleapis.com/https/backend_latencies` | p99 latency to backend | > 2000ms |
| `iap.googleapis.com/tunnel/sent_bytes_count` | TCP tunnel data transfer | Unusual exfil patterns |
| `iap.googleapis.com/tunnel/num_connections` | Active IAP TCP connections | Sudden spike |
| `accesscontextmanager.googleapis.com/policy_evaluation_count` | ACM evaluations | Spike with high deny rate |
| `logging.googleapis.com/log_entry_count` | VPC SC violation log count | Any non-zero count in enforced mode |

### Log Export for SIEM

```bash
# Create a log sink for BCE-related logs to BigQuery
gcloud logging sinks create bce-audit-sink \
  bigquery.googleapis.com/projects/PROJECT_ID/datasets/bce_audit_logs \
  --log-filter='
    protoPayload.serviceName="iap.googleapis.com" OR
    protoPayload.serviceName="accesscontextmanager.googleapis.com" OR
    protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"
  ' \
  --project=PROJECT_ID

# Grant sink SA write access to BigQuery dataset
# (gcloud will output the sink's service account — grant it roles/bigquery.dataEditor)
```

---

## 15. Security Hardening & Best Practices

### Defense-in-Depth Layers

```
Layer 1 — Edge:          Cloud Armor (WAF, DDoS, geo-blocking)
Layer 2 — AuthN:         IAP + Google Identity / Workforce Federation
Layer 3 — AuthZ:         IAP IAM bindings (roles/iap.httpsResourceAccessor)
Layer 4 — Context:       ACM Access Levels (device + IP + region + OS)
Layer 5 — Data:          VPC Service Controls (API-level perimeter)
Layer 6 — Transport:     mTLS via Certificate Authority Service
Layer 7 — Content:       Chrome DLP + BeyondCorp Threat Protection
Layer 8 — Detection:     Google SecOps (Chronicle) + Cloud Logging
```

### Key Hardening Checklist

- ✅ **Never expose backends directly** — Block all external ingress except from GFE IP ranges (`130.211.0.0/22`, `35.191.0.0/16`) and IAP TCP range (`35.235.240.0/20`)
- ✅ **Validate IAP JWT on every backend request** — Don't trust `X-Goog-Authenticated-User-Email` without JWT validation; it can be spoofed
- ✅ **Combine IAP + ACM Access Levels** — IAP IAM alone isn't Zero Trust; add device/context enforcement via access levels
- ✅ **Always dry-run VPC Service Controls** — Minimum 48–72 hours in dry-run before enforcing; log and resolve all violations
- ✅ **Use Workload Identity Federation** for all CI/CD — No service account key files; keys don't expire and leak silently
- ✅ **Enforce Endpoint Verification at OS-verified tier minimum** for production access
- ✅ **Store OAuth client secrets in Secret Manager** — Never in Terraform state or environment variables
- ✅ **Enable Cloud Armor** with OWASP Top 10 ruleset on all load balancers fronting IAP
- ✅ **Deploy IAP Connector in HA mode** (3+ replicas) for on-prem app access
- ✅ **Use break-glass accounts** — Define emergency access service accounts with enhanced alerting, outside BCE policy scope, reviewed quarterly
- ✅ **Export all BCE logs to a locked log bucket** — `_Default` log sink with retention lock for compliance
- ✅ **Rotate OAuth credentials quarterly** — Use Secret Manager + Cloud Run/Cloud Build for automated rotation
- ✅ **Use scoped access policies for delegated admin** — Don't give all teams access to the org-level ACM policy

> **⚠️ Gotcha:** **Cloud Run direct URLs (`*.run.app`) bypass IAP entirely.** When protecting Cloud Run with IAP, you must: (1) set `--no-allow-unauthenticated` on the Cloud Run service, AND (2) only expose it through a Global HTTPS LB with IAP enabled on the NEG backend. Users who know the `.run.app` URL can bypass your IAP policy.

> **⚠️ Gotcha:** **IAP does not protect WebSocket connections by default** in all configurations. Verify WebSocket support is enabled in your backend service when your app uses WebSockets.

---

## 16. Limits & Quotas

| Resource | Limit | Notes |
|---|---|---|
| Access Policies per Organization | 1 org-level + N scoped | Scoped policies: no documented hard limit |
| Access Levels per Policy | 1,000 | Combine with CEL for complex logic |
| Service Perimeters per Policy | 100 | Includes bridge perimeters |
| Projects per Service Perimeter | 1,000 | Across all `resources` entries |
| Restricted services per Perimeter | 100 | |
| Ingress policies per Perimeter | 100 | |
| Egress policies per Perimeter | 100 | |
| Workforce Identity Pools per Organization | 10 | Quota increase requestable |
| Workforce Providers per Pool | 10 | |
| Workload Identity Pools per Project | 10 | |
| Workload Identity Providers per Pool | 10 | |
| IAP OAuth Brands per Project | 1 | Cannot be deleted once created |
| IAP-secured Backend Services per Project | No hard limit | Subject to LB backend limits |
| ACM Custom Level CEL expression size | 1,000 characters | Per expression; use functions for complexity |
| Access Bindings (CAA) per Policy | 500 | Per group + access level pair |
| IAP TCP tunnel idle timeout | 1 hour | Connection closed after 1hr of inactivity |

---

## 17. Common Gotchas & Troubleshooting

| Issue | Likely Cause | Diagnosis | Fix |
|---|---|---|---|
| **403 on IAP-protected resource** | Missing IAP IAM binding | Check `gcloud iap web get-iam-policy` | Add `roles/iap.httpsResourceAccessor` to user/group |
| **Access level not applied** | ACM propagation delay | Wait 5–10 min; check `gcloud access-context-manager levels describe` | Re-test; levels take up to 10 min to propagate globally |
| **"PERMISSION_DENIED: Request is prohibited by organization policy"** | VPC SC blocking the API call | Filter VPC SC violation logs; check perimeter config | Add ingress rule for the blocked SA/identity |
| **IAP TCP tunnel instantly disconnects** | Firewall blocking IAP IP range | Check GCE firewall logs | Allow ingress from `35.235.240.0/20` on target port |
| **Federated user gets 403 after OIDC redirect** | Attribute mapping mismatch or condition too strict | Log `google.subject` from token; compare with pool attribute mapping | Fix `--attribute-mapping` to match actual IdP token claims |
| **Endpoint Verification device policy not enforced** | `devicePolicy` missing from access level conditions | Check access level spec with `gcloud acm levels describe` | Add `devicePolicy` block to access level |
| **IAP JWT `audience` mismatch** | Backend validating wrong audience string | Print expected audience; compare with `aud` claim in JWT | Use the format `/projects/PROJECT_NUMBER/global/backendServices/SERVICE_ID` |
| **CAA blocks all Workspace access** | Access level too restrictive (e.g., no device in inventory) | Check Admin Console → Directory → Devices | Use monitoring mode first; verify device is enrolled in EV |
| **Cloud Run app accessible via `.run.app` URL despite IAP** | IAP only on LB; Cloud Run allows unauthenticated | Test `.run.app` URL directly | Set `--no-allow-unauthenticated` on Cloud Run service |
| **VPC SC lockout during enforcement** | Legitimate SA not in ingress rules | Switch to dry-run: `gcloud acm perimeters dry-run create --copy-from=ENFORCED` | Add ingress rule for locked-out SA; re-enforce |
| **Workload Identity Federation returns 403** | SA binding uses wrong principalSet format | Print effective binding with `gcloud iam sa get-iam-policy` | Verify `projects/PROJECT_NUMBER` not `projects/PROJECT_ID` in principalSet |
| **IAP header spoofing on direct backend access** | Backend reachable without going through IAP/LB | Test with `curl -H "X-Goog-Authenticated-User-Email: ..."` | Firewall backend to only accept GFE IPs; validate IAP JWT in code |

---

## 18. Quick Reference — gcloud Commands

```bash
# ── IAP ───────────────────────────────────────────────────────────────────────

# List IAP-enabled backends
gcloud compute backend-services list --filter="iap.enabled=true" --global

# Get IAP IAM policy for a backend
gcloud iap web get-iam-policy \
  --resource-type=backend-services \
  --service=BACKEND_NAME \
  --project=PROJECT_ID

# Get IAP settings
gcloud iap settings get \
  --resource-type=backend-services \
  --service=BACKEND_NAME

# Disable IAP on a backend
gcloud compute backend-services update BACKEND_NAME \
  --global \
  --iap=disabled

# SSH through IAP (no external IP)
gcloud compute ssh VM_NAME --tunnel-through-iap --zone=ZONE

# ── Access Context Manager ────────────────────────────────────────────────────

# Get the org-level access policy
gcloud access-context-manager policies list --organization=ORG_ID

# List all access levels
gcloud access-context-manager levels list --policy=POLICY_ID

# Describe an access level (YAML output)
gcloud access-context-manager levels describe LEVEL_ID \
  --policy=POLICY_ID --format=yaml

# List all service perimeters
gcloud access-context-manager perimeters list --policy=POLICY_ID

# Describe a service perimeter
gcloud access-context-manager perimeters describe PERIMETER_ID \
  --policy=POLICY_ID --format=yaml

# List CAA access bindings
gcloud access-context-manager cloud-bindings list --policy=POLICY_ID

# ── Workforce Identity ────────────────────────────────────────────────────────

# List workforce pools
gcloud iam workforce-pools list --organization=ORG_ID --location=global

# Describe a workforce pool
gcloud iam workforce-pools describe POOL_ID \
  --organization=ORG_ID --location=global

# List providers in a pool
gcloud iam workforce-pools providers list \
  --workforce-pool=POOL_ID --location=global --organization=ORG_ID

# ── Workload Identity ─────────────────────────────────────────────────────────

# List workload identity pools
gcloud iam workload-identity-pools list --location=global --project=PROJECT_ID

# Describe a pool
gcloud iam workload-identity-pools describe POOL_ID \
  --location=global --project=PROJECT_ID

# List providers in a pool
gcloud iam workload-identity-pools providers list \
  --workload-identity-pool=POOL_ID --location=global --project=PROJECT_ID

# ── Diagnostic ───────────────────────────────────────────────────────────────

# Check effective IAM policy (including inherited)
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.role=roles/iap.httpsResourceAccessor"

# Test IAM permissions for a principal
gcloud iam list-testable-permissions //cloudresourcemanager.googleapis.com/projects/PROJECT_ID \
  --filter="name:iap"

# Decode an IAP JWT manually (for debugging)
echo "JWT_TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
```

---

## 19. Decision Tree

```
START: I need to protect access to an application or GCP resource
│
├─► Is the resource a GCP API (BigQuery, GCS, Spanner, Secret Manager)?
│     YES → Add VPC Service Controls perimeter around the project/resource
│           ↳ + Add IAP for human access to any frontend; enforce access levels
│           ↳ + Use ingress/egress rules for service account access
│
├─► Is the resource a web application (App Engine, GCE, GKE, Cloud Run, on-prem)?
│     YES → Enable IAP on the backend service / load balancer
│           │
│           ├─► Do I only need to restrict by identity (user must be in a group)?
│           │     → IAP only + roles/iap.httpsResourceAccessor binding
│           │       [Simplest; no device enforcement; network-level access still possible]
│           │
│           ├─► Do I need device trust / network / region enforcement?
│           │     → IAP + ACM Access Levels (basic or CEL)
│           │       Bind access level to IAP IAM condition
│           │       [Adds context-awareness; recommended for production]
│           │
│           ├─► Do I need to protect the underlying GCP data APIs too?
│           │     → IAP + ACM Access Levels + VPC Service Controls
│           │       [Full data-plane protection; required for regulated workloads]
│           │
│           └─► Do I need browser-level DLP, threat protection, and SIEM?
│                 → Full BCE stack:
│                   IAP + ACM Access Levels + VPC SC + Cloud Armor
│                   + Chrome Enterprise + Endpoint Verification
│                   + BeyondCorp Threat & Data Protection + Google SecOps
│                   [Maximum Zero Trust posture; required for highly sensitive data]
│
├─► Is the resource accessed via SSH/RDP/TCP (not HTTP)?
│     YES → Enable IAP TCP Tunneling
│           ↳ Grant roles/iap.tunnelResourceAccessor
│           ↳ Allow 35.235.240.0/20 in GCE firewall
│           ↳ No public IP required on VM
│
├─► Is the resource on-premises or in another cloud?
│     YES → Deploy IAP Connector (GKE/VM) in the target environment
│           ↳ Connector establishes outbound tunnel to Google
│           ↳ Enable IAP on the LB backend pointing to connector
│
├─► Are my users not Google accounts (external IdP — Okta, Azure AD, ADFS)?
│     YES → Set up Workforce Identity Federation
│           ↳ Create workforce pool + OIDC/SAML provider
│           ↳ Map attributes; set attribute conditions
│           ↳ Grant IAM using principalSet:// syntax
│
└─► Is the workload non-human (CI/CD, external VMs, AWS/Azure services)?
      YES → Set up Workload Identity Federation
            ↳ Create workload pool + provider (OIDC/AWS)
            ↳ Bind pool identities to a service account
            ↳ No service account key files needed
```

### Summary: When to Use What

| Scenario | Solution |
|---|---|
| Protect a web app for Google users only | IAP on backend service |
| Add device trust + context enforcement | IAP + ACM Access Levels |
| Protect GCP API data access | VPC Service Controls |
| Full Zero Trust for sensitive workloads | IAP + ACM + VPC SC + Cloud Armor + BCE |
| SSH/RDP to VMs without public IPs | IAP TCP Tunneling |
| Extend Zero Trust to on-prem apps | IAP Connector |
| Federate external human users | Workforce Identity Federation |
| Allow keyless CI/CD access to GCP | Workload Identity Federation |
| Enforce Zero Trust in Chrome browser | Chrome Enterprise + CAA bindings |
| DLP + Threat Protection in browser | BeyondCorp Threat & Data Protection |
