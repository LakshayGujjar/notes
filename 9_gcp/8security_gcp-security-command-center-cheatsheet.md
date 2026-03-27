# 🛡️ GCP Security Command Center (SCC) Cheatsheet

> **Audience:** Security Engineers · Cloud Architects · DevSecOps Engineers · Platform Engineers  
> **Scope:** SCC Standard/Premium/Enterprise — threat detection, findings, muting, notifications, BigQuery export, compliance, remediation automation, IAM

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [SCC Tiers: Standard vs. Premium vs. Enterprise](#2-scc-tiers-standard-vs-premium-vs-enterprise)
3. [Built-in Security Sources & Detectors](#3-built-in-security-sources--detectors)
4. [Findings: Structure, Filtering & Management](#4-findings-structure-filtering--management)
5. [Muting Findings](#5-muting-findings)
6. [Security Marks](#6-security-marks)
7. [Notifications & Pub/Sub Integration](#7-notifications--pubsub-integration)
8. [BigQuery Export (Continuous Export)](#8-bigquery-export-continuous-export)
9. [Compliance & Posture Management](#9-compliance--posture-management)
10. [Custom Sources & Custom Findings](#10-custom-sources--custom-findings)
11. [IAM & Access Control for SCC](#11-iam--access-control-for-scc)
12. [Attack Path Simulation (Premium)](#12-attack-path-simulation-premium)
13. [Event Threat Detection Deep Dive](#13-event-threat-detection-deep-dive)
14. [Automating Remediation](#14-automating-remediation)
15. [Multi-Cloud Security (Enterprise)](#15-multi-cloud-security-enterprise)
16. [Monitoring, Alerting & Audit Logging](#16-monitoring-alerting--audit-logging)
17. [gcloud CLI Quick Reference](#17-gcloud-cli-quick-reference)
18. [Best Practices & Operational Patterns](#18-best-practices--operational-patterns)
19. [Quick Reference & Comparison Tables](#19-quick-reference--comparison-tables)

---

## 1. Overview & Key Concepts

### What is Security Command Center?

Security Command Center (SCC) is Google Cloud's unified security and risk management platform. It provides a centralized view of your GCP security posture across asset inventory, vulnerability detection, threat detection, compliance posture, and misconfiguration findings — all in one place.

Core value proposition:
- **Asset discovery** — continuous inventory of all GCP resources across org/folder/project
- **Vulnerability detection** — misconfigurations, CVEs, exposed services across all GCP services
- **Threat detection** — near-real-time log analysis, runtime container threats, VM memory scanning
- **Compliance posture** — automated mapping of findings to CIS, PCI-DSS, NIST, ISO 27001, and more
- **Single pane of glass** — all security signals from native and third-party sources in one API

---

### SCC Resource Hierarchy

```
Organization  (SCC enabled here — org-level is strongly preferred)
├── Folder A
│   ├── Project 1  ←── findings from this project roll up to org
│   └── Project 2
├── Folder B
│   └── Project 3
└── Project 4 (direct)

SCC Sources (Security Health Analytics, ETD, CTD, custom, etc.)
└── Findings  (attached to a source, reference a GCP resource in the hierarchy)
```

> ⚠️ **Always enable SCC at organization level.** Project-level SCC misses cross-project threat correlation, org-wide compliance views, and many ETD detectors that require org-scope log analysis.

---

### Core Concepts

| Concept | Definition |
|---|---|
| **Finding** | A security signal — a detected vulnerability, misconfiguration, or threat. Has state, severity, class, and resource reference. |
| **Source** | The detector that produced a finding (e.g., Security Health Analytics, Event Threat Detection, custom source). |
| **Asset** | A GCP resource tracked in SCC's asset inventory (VM, bucket, firewall, SA, etc.). |
| **Resource** | The GCP resource a finding is about — referenced by `resourceName`. |
| **Severity** | CRITICAL / HIGH / MEDIUM / LOW / INFORMATIONAL |
| **Finding Class** | Categorizes the type of finding: VULNERABILITY, MISCONFIGURATION, THREAT, etc. |
| **Finding State** | ACTIVE (open) or INACTIVE (resolved / no longer detected). |
| **Mute** | Suppresses a finding from dashboards without resolving it. |
| **Notification** | Real-time Pub/Sub stream of findings matching a filter. |
| **Posture** | Security configuration-as-code that defines desired state and detects drift. |
| **Attack Path** | Simulated path an attacker could follow from an exposure to a high-value resource. |

---

### Finding Classes

| Class | Description | Example Categories |
|---|---|---|
| `VULNERABILITY` | Exploitable weakness or CVE in software/config | OS CVE, container image CVE, weak credentials |
| `MISCONFIGURATION` | GCP resource configured in a security-weak way | `PUBLIC_BUCKET_ACL`, `OPEN_FIREWALL`, `SQL_NO_ROOT_PASSWORD` |
| `THREAT` | Active or suspected attack in progress | Crypto mining, data exfiltration, anomalous IAM |
| `OBSERVATION` | Noteworthy event, not necessarily malicious | Service account key created, excessive permissions |
| `SENSITIVE_DATA_EXPOSURE` | Sensitive data found in an exposed location | PII in public bucket, secrets in logs |
| `ERROR` | SCC configuration or scanning error | Missing permissions, scan failure |
| `SCC_ERROR` | Internal SCC service error | Configuration misconfiguration in SCC itself |

---

### Finding Severity Levels

| Severity | Description | Typical SLA |
|---|---|---|
| `CRITICAL` | Immediate risk of data breach or full compromise | 24 hours |
| `HIGH` | Significant risk; exploitation likely or easy | 7 days |
| `MEDIUM` | Moderate risk; requires context to evaluate | 30 days |
| `LOW` | Minor risk; defense-in-depth gap | 90 days |
| `INFORMATIONAL` | No direct risk; situational awareness only | Best effort |

---

### Finding States

| State | Meaning | Triggered By |
|---|---|---|
| `ACTIVE` | Finding is open and unresolved | Initial detection; re-detection after INACTIVE |
| `INACTIVE` | Finding resolved — resource no longer vulnerable / manually closed | Re-scan confirms remediation; manual `UpdateFinding` API call |

> 💡 INACTIVE ≠ muted. INACTIVE means the issue is resolved. Muted means the issue exists but is suppressed from views.

---

### Key Limits & Quotas

| Resource | Limit |
|---|---|
| Finding retention | 13 months (rolling) |
| Notification configs per org | 500 |
| Mute configs per org | 1,000 |
| BigQuery export configs per org | 100 |
| `ListFindings` API (requests/min) | 1,000 |
| `UpdateFinding` API (requests/min) | 1,000 |
| Custom SHA modules per org | 100 |
| Custom ETD modules per org | 100 |

---

## 2. SCC Tiers: Standard vs. Premium vs. Enterprise

### Standard Tier (Free)

Included with all GCP organizations at no additional cost:
- Security Health Analytics — **subset** of checks (~20 categories, no compliance mapping)
- Asset inventory and asset change history
- Findings API (create, read, update findings)
- Basic IAM anomaly detection (limited)
- Web Security Scanner — custom scan configs only (no managed scans)
- Integration with Cloud Logging and Cloud Asset Inventory

**Limitations:** No ETD, no Container Threat Detection, no compliance dashboards, no Attack Path Simulation, no managed Web Security Scanner.

---

### Premium Tier

Everything in Standard, plus:
- **Security Health Analytics** — full 150+ check categories with compliance mapping
- **Event Threat Detection (ETD)** — near-real-time log-based threat detection
- **Container Threat Detection (CTD)** — GKE runtime threat detection via eBPF
- **Web Security Scanner** — managed scans (auto-scheduled, no configuration required)
- **VM Manager** — OS vulnerability findings with CVE/CVSS scores
- **Rapid Vulnerability Detection (RVD)** — agentless network-based vuln scanning
- **Virtual Machine Threat Detection (VMTD)** — hypervisor-level memory scanning
- **Attack Path Simulation** — attacker path modeling and exposure scoring
- **Compliance dashboards** — CIS, PCI-DSS, NIST SP 800-53, ISO 27001, HIPAA, SOC 2, FedRAMP
- **Custom SHA modules** — CEL-based custom misconfiguration detectors
- **Custom ETD modules** — YARA-L custom threat detectors

**Pricing:** Per-project per-month billing based on project compute/service usage. Billed at the project level; activated at org level.

---

### Enterprise Tier

Everything in Premium, plus:
- **Security Operations (SecOps)** — built-in case management and SOAR capabilities
- **Mandiant Threat Intelligence** — IOC enrichment for ETD findings
- **Chronicle SIEM integration** — native ingestion of SCC findings into Chronicle
- **Gemini AI security insights** — AI-powered finding summaries, remediation suggestions, attack path explanations
- **Multi-cloud support** — ingest findings from AWS Security Hub and Microsoft Defender for Cloud
- **Posture management** — security posture-as-code with drift detection
- **Security operations workflows** — automated case creation, playbook triggers

**When to choose Enterprise:** Large organizations with a dedicated SOC, multi-cloud environments, Mandiant threat intelligence requirements, or needing integrated SOAR capabilities.

---

### Feature Comparison Table

| Feature | Standard | Premium | Enterprise |
|---|---|---|---|
| Security Health Analytics (basic) | ✅ ~20 checks | ✅ 150+ checks | ✅ 150+ checks |
| Event Threat Detection | ❌ | ✅ | ✅ + Mandiant Intel |
| Container Threat Detection | ❌ | ✅ | ✅ |
| Web Security Scanner (managed) | ❌ | ✅ | ✅ |
| VM Manager / RVD / VMTD | ❌ | ✅ | ✅ |
| Attack Path Simulation | ❌ | ✅ | ✅ |
| Compliance dashboards | ❌ | ✅ | ✅ |
| Custom SHA / ETD modules | ❌ | ✅ | ✅ |
| Posture management | ❌ | ❌ | ✅ |
| Multi-cloud (AWS / Azure) | ❌ | ❌ | ✅ |
| Chronicle SIEM integration | ❌ | ❌ | ✅ |
| Gemini AI insights | ❌ | ❌ | ✅ |
| Case management / SOAR | ❌ | ❌ | ✅ |
| Cost | Free | Per-project/mo | Per-project/mo (higher) |

---

### Enabling Premium/Enterprise

```bash
# Enable SCC Premium at organization level
gcloud services enable securitycenter.googleapis.com \
  --organization=ORG_ID

# Enable SCC Premium tier (via Security Command Center Settings API)
gcloud scc settings update \
  --organization=ORG_ID \
  --enable-asset-discovery

# Or via Console: Security → Security Command Center → Settings → Tier → Upgrade
```

---

## 3. Built-in Security Sources & Detectors

### Security Health Analytics (SHA)

SHA continuously scans GCP resource configurations for misconfigurations. It runs:
- **Batch scans**: every ~24 hours across all resources
- **Real-time triggers**: when a resource changes (via Cloud Asset Inventory feed)

**Key SHA Categories:**

| Category | Severity | Description |
|---|---|---|
| `PUBLIC_BUCKET_ACL` | CRITICAL | GCS bucket is publicly accessible |
| `SQL_NO_ROOT_PASSWORD` | CRITICAL | Cloud SQL root user has no password |
| `OPEN_FIREWALL` | HIGH | Firewall rule allows `0.0.0.0/0` ingress |
| `KMS_KEY_NOT_ROTATED` | MEDIUM | KMS key has not been rotated in 90 days |
| `LEGACY_NETWORK` | MEDIUM | Project uses legacy (non-VPC) network |
| `MFA_NOT_ENFORCED` | HIGH | No MFA required for admin accounts |
| `AUDIT_LOGGING_DISABLED` | HIGH | Data Access audit logging not enabled |
| `SERVICE_ACCOUNT_KEY_CREATED` | MEDIUM | External SA key created |
| `OVER_PRIVILEGED_SERVICE_ACCOUNT` | HIGH | SA has owner/editor on project |
| `CLUSTER_LOGGING_DISABLED` | MEDIUM | GKE cluster audit logging disabled |
| `BINARY_AUTHORIZATION_DISABLED` | MEDIUM | Binary Authorization not enabled on GKE |
| `SHIELDED_VM_DISABLED` | LOW | Shielded VM not enabled on Compute instance |
| `VPC_FLOW_LOGS_DISABLED` | MEDIUM | VPC flow logs not enabled on subnet |
| `DNSSEC_DISABLED` | MEDIUM | DNSSEC not enabled on Cloud DNS zone |
| `LOCK_RETENTION_POLICY_NOT_SET` | MEDIUM | GCS bucket has no retention lock |

---

### Event Threat Detection (ETD) — Premium

```
Cloud Logging (Audit Logs, VPC Flow, DNS, GKE Audit)
       │
       ▼
   Log Router
       │
       ▼
 ETD Analysis Engine  (near-real-time, 1–5 min latency)
       │
       ▼
   SCC Findings  (THREAT class)
```

**Log sources required for full ETD coverage:**
- Cloud Admin Activity Audit Logs (always on)
- Cloud Data Access Audit Logs (**must enable** — not on by default)
- VPC Flow Logs (subnet-level enablement)
- DNS query logs (Cloud DNS → Cloud Logging)
- GKE Audit Logs

**ETD Detector Categories:**

| MITRE Category | Example Detectors |
|---|---|
| `INITIAL_ACCESS` | Leaked SA key used externally, anomalous service account use |
| `PRIVILEGE_ESCALATION` | Anomalous IAM grant, sensitive role binding added |
| `DEFENSE_EVASION` | Audit log disabled, bucket logging disabled, SCC notifications disabled |
| `DISCOVERY` | Abnormal resource enumeration by principal |
| `LATERAL_MOVEMENT` | SSH tunneling from GCE to GCE, unusual service account impersonation |
| `CREDENTIAL_ACCESS` | SA key created by anomalous user, secret accessed outside normal pattern |
| `EXFILTRATION` | Data moved to external bucket, BigQuery data exfiltration |
| `IMPACT` | Crypto mining on Compute/GKE, mass resource deletion, ransom activity |

> ⚠️ **Enable Data Access audit logs for all critical services.** Without them, ETD cannot detect most credential access, exfiltration, and privilege escalation threats.

---

### Container Threat Detection (CTD) — Premium

CTD uses **eBPF kernel instrumentation** on GKE nodes — no agent required, minimal performance overhead.

**Detectors:**

| Detector | Description |
|---|---|
| `ADDED_BINARY_EXECUTED` | Unexpected binary executed inside container (not in original image) |
| `ADDED_LIBRARY_LOADED` | Unexpected shared library loaded |
| `REVERSE_SHELL` | Container process established a reverse shell connection |
| `UNEXPECTED_CHILD_SHELL` | Shell spawned from unexpected parent process |
| `MODIFIED_CRITICAL_FILE` | Critical system file (e.g., `/etc/passwd`) modified inside container |

**CTD Finding Fields:** `processes[].name`, `processes[].args`, `kubernetes.pods[].name`, `kubernetes.pods[].ns`, `kubernetes.pods[].containers[].name`, `kubernetes.pods[].containers[].imageId`

---

### Web Security Scanner (WSS)

- **Managed scans** (Premium): automatically scheduled DAST scans for App Engine, GKE Ingress, and Compute Engine HTTP(S) endpoints
- **Custom scans**: manually configured scan targets with auth, blocklists, entry points

**Finding Categories:** `XSS_CALLBACK`, `XSS_ERROR`, `MIXED_CONTENT`, `OUTDATED_LIBRARY`, `CLEAR_TEXT_PASSWORD`, `INVALID_CONTENT_TYPE`, `CSRF`, `ACCESSIBLE_GIT_REPOSITORY`, `HTTP_REDIRECT_TO_HTTPS`

---

### VM Manager & Rapid Vulnerability Detection

- **VM Manager**: OS-level vulnerability scanning via OS Configuration agent → CVE findings with CVSS scores, patch status
- **Rapid Vulnerability Detection (RVD)**: agentless network scanner — detects exposed services, default credentials, known CVEs on network-accessible services (no installation required)
- **Virtual Machine Threat Detection (VMTD)**: hypervisor-level memory scanning → detects crypto mining malware, rootkits operating below the OS

---

## 4. Findings: Structure, Filtering & Management

### Finding Data Model

```
Finding
├── name               "organizations/ORG/sources/SRC/findings/FINDING_ID"
├── canonicalName      "organizations/ORG/sources/SRC/locations/global/findings/ID"
├── parent             "organizations/ORG/sources/SRC"
├── resourceName       Full resource URI (e.g., "//storage.googleapis.com/my-bucket")
├── state              ACTIVE | INACTIVE
├── severity           CRITICAL | HIGH | MEDIUM | LOW | INFORMATIONAL
├── findingClass       VULNERABILITY | MISCONFIGURATION | THREAT | OBSERVATION | ...
├── category           Detector-specific string (e.g., "PUBLIC_BUCKET_ACL")
├── description        Human-readable description
├── nextSteps          Remediation guidance
├── externalUri        Link to source tool or documentation
├── eventTime          When the event occurred (not when finding was created)
├── createTime         When the finding was first created in SCC
├── updateTime         When the finding was last updated
├── mute               MUTED | UNMUTED | UNDEFINED
├── muteInitiator      Who/what applied the mute
├── muteUpdateTime     When mute state last changed
├── sourceProperties   Map of source-specific key-value data
├── securityMarks      User-managed key-value annotations
├── indicator          { ipAddresses, domains, signatures } — for threat findings
├── vulnerability      { cve { id, cvssv3 { baseScore } }, offendingPackage } — for vuln findings
├── access             { callerIp, callerIpGeo, userAgentFamily, principalEmail, serviceAccountDelegationInfo }
├── processes          [ { name, args, pid, parentPid, binaryPath } ]
├── connections        [ { ip, port, protocol, direction } ]
├── kubernetes         { pods, nodes, roles, bindings }
└── iamBindings        [ { action, role, member } ]
```

---

### Finding Filter Syntax (CEL-based)

```bash
# All CRITICAL and HIGH active findings
state="ACTIVE" AND (severity="CRITICAL" OR severity="HIGH")

# All active findings for a specific project
state="ACTIVE" AND resource.project_display_name="my-project"

# All THREAT class findings in the last 7 days
findingClass="THREAT" AND eventTime>="2024-01-01T00:00:00Z"

# All open PUBLIC_BUCKET_ACL findings
state="ACTIVE" AND category="PUBLIC_BUCKET_ACL"

# All unmuted VULNERABILITY findings
findingClass="VULNERABILITY" AND mute="UNMUTED" AND state="ACTIVE"

# Findings for a specific resource type
resource.type="google.storage.Bucket" AND state="ACTIVE"

# CRITICAL findings not yet muted, with no security mark "jira_ticket"
severity="CRITICAL" AND mute!="MUTED" AND NOT securityMarks.marks.jira_ticket:*

# All ETD findings from a specific project
source.display_name="Event Threat Detection" AND resource.project_name:"projects/my-project"
```

---

### gcloud: Findings Management

```bash
# List CRITICAL and HIGH active findings for an org
gcloud scc findings list \
  --organization=ORG_ID \
  --filter='state="ACTIVE" AND (severity="CRITICAL" OR severity="HIGH")' \
  --format="table(name, category, severity, resource.displayName, eventTime)"

# List findings from a specific source
gcloud scc findings list \
  --organization=ORG_ID \
  --source=SOURCE_ID \
  --filter='state="ACTIVE"'

# Update finding state to INACTIVE (manual resolution)
gcloud scc findings update FINDING_ID \
  --organization=ORG_ID \
  --source=SOURCE_ID \
  --state=INACTIVE

# Group findings by category (aggregation)
gcloud scc findings group \
  --organization=ORG_ID \
  --group-by=category \
  --filter='state="ACTIVE"' \
  --format="table(groupByFieldValue, count)"

# Group by severity
gcloud scc findings group \
  --organization=ORG_ID \
  --group-by=severity \
  --filter='state="ACTIVE"'
```

---

### Python: Findings Management

```python
from google.cloud import securitycenter
from google.protobuf import field_mask_pb2


def list_critical_findings(org_id: str) -> list:
    """List all CRITICAL active findings across an organization."""
    client = securitycenter.SecurityCenterClient()
    org_name = f"organizations/{org_id}"

    findings = []
    for finding_result in client.list_findings(
        request={
            "parent": f"{org_name}/sources/-",  # "-" = all sources
            "filter": 'state="ACTIVE" AND severity="CRITICAL"',
            "order_by": "eventTime desc",
        }
    ):
        findings.append(finding_result.finding)
    return findings


def list_findings_by_source(org_id: str, source_id: str) -> list:
    """List active findings from a specific source."""
    client = securitycenter.SecurityCenterClient()
    source_name = f"organizations/{org_id}/sources/{source_id}"

    return [
        r.finding
        for r in client.list_findings(
            request={
                "parent": source_name,
                "filter": 'state="ACTIVE"',
            }
        )
    ]


def update_finding_state(org_id: str, source_id: str, finding_id: str,
                          new_state: str = "INACTIVE"):
    """Update a finding state to INACTIVE (resolved)."""
    from google.cloud.securitycenter_v1 import Finding
    from google.protobuf.timestamp_pb2 import Timestamp
    import time

    client = securitycenter.SecurityCenterClient()
    finding_name = f"organizations/{org_id}/sources/{source_id}/findings/{finding_id}"

    ts = Timestamp()
    ts.FromSeconds(int(time.time()))

    finding = Finding(
        name=finding_name,
        state=Finding.State[new_state],
        event_time=ts,
    )
    update_mask = field_mask_pb2.FieldMask(paths=["state"])
    return client.update_finding(
        request={"finding": finding, "update_mask": update_mask}
    )
```

---

## 5. Muting Findings

### Mute vs. Resolve

| Action | Finding State | Finding Still in DB | Visible in Dashboard | When to Use |
|---|---|---|---|---|
| **Resolve (INACTIVE)** | INACTIVE | ✅ | No | Actual remediation confirmed |
| **Static Mute** | ACTIVE (muted) | ✅ | No (suppressed) | Accepted risk on specific instance |
| **Dynamic Mute Rule** | ACTIVE (auto-muted) | ✅ | No (suppressed) | Known false positive pattern / accepted risk class |

---

### Dynamic Mute Configs

Mute configs are persistent, filter-based rules that **automatically mute all future findings** matching the filter. They also retroactively mute existing matching findings.

```bash
# Create a dynamic mute config (suppress all LOW severity findings in dev projects)
gcloud scc muteconfigs create dev-low-severity-mute \
  --organization=ORG_ID \
  --filter='severity="LOW" AND resource.project_display_name:"dev-"' \
  --description="Suppress LOW findings in dev projects — reviewed quarterly"

# Create mute config for a known false positive pattern
gcloud scc muteconfigs create wss-test-xss-mute \
  --organization=ORG_ID \
  --filter='category="XSS_CALLBACK" AND resource.project_display_name="web-test-project"' \
  --description="Known false positive in web-test-project sandbox — owner: security@example.com"

# List all mute configs
gcloud scc muteconfigs list --organization=ORG_ID

# Describe a mute config
gcloud scc muteconfigs describe dev-low-severity-mute --organization=ORG_ID

# Update mute config filter
gcloud scc muteconfigs update dev-low-severity-mute \
  --organization=ORG_ID \
  --filter='severity="LOW" AND resource.project_display_name:"dev-"'

# Delete a mute config (unmutes all findings that were only muted by this config)
gcloud scc muteconfigs delete dev-low-severity-mute --organization=ORG_ID
```

---

### Python: Mute Management

```python
from google.cloud import securitycenter
from google.cloud.securitycenter_v1 import MuteConfig


def create_mute_config(org_id: str, mute_id: str, filter_str: str, description: str):
    """Create a dynamic mute rule for an organization."""
    client = securitycenter.SecurityCenterClient()
    parent = f"organizations/{org_id}"

    mute_config = MuteConfig(
        description=description,
        filter=filter_str,
        type_=MuteConfig.MuteConfigType.DYNAMIC,
    )
    return client.create_mute_config(
        request={
            "parent": parent,
            "mute_config": mute_config,
            "mute_config_id": mute_id,
        }
    )


def bulk_mute_findings(org_id: str, filter_str: str):
    """Bulk mute all findings matching a filter (one-time static mute)."""
    client = securitycenter.SecurityCenterClient()
    parent = f"organizations/{org_id}/sources/-"

    operation = client.bulk_mute_findings(
        request={"parent": parent, "filter": filter_str}
    )
    return operation.result()  # blocks until complete


def unmute_finding(org_id: str, source_id: str, finding_id: str):
    """Unmute a specific finding."""
    from google.cloud.securitycenter_v1 import Finding

    client = securitycenter.SecurityCenterClient()
    finding_name = f"organizations/{org_id}/sources/{source_id}/findings/{finding_id}"

    return client.set_mute(
        request={"name": finding_name, "mute": Finding.Mute.UNMUTED}
    )
```

---

### Mute Governance

> ⚠️ **Muting without governance creates security debt.** Every mute config should have: (1) a description explaining the reason, (2) an owner (`owner: security@example.com`), (3) a review date annotation. Review mute configs quarterly.

```bash
# Good mute config description pattern:
# "REASON: Known false positive in sandbox env | OWNER: security@co.com | REVIEW: 2024-Q3"
```

> 💡 Muted findings are **excluded from some compliance dashboard scores**. Review mute impact before muting findings that map to PCI-DSS or CIS controls.

---

## 6. Security Marks

### What Security Marks Are

Security marks are user-managed key-value pairs attached to **SCC findings** or **SCC assets** (not the GCP resource itself). They live entirely within SCC and are used for:
- **Workflow tracking**: `jira_ticket=PROJ-1234`, `reviewed=true`, `reviewed_by=john@`
- **Risk acceptance**: `accepted_risk=true`, `exception_expires=2024-12-31`
- **Custom filtering**: `data_classification=PII`, `environment=prod`
- **Automation triggers**: `auto_remediated=true`, `remediation_attempted=true`

**Constraints:** String keys and values only; max 64 marks per resource; keys max 800 chars; values max 800 chars.

---

### gcloud: Security Marks

```bash
# Add marks to a finding
gcloud scc findings update FINDING_ID \
  --organization=ORG_ID \
  --source=SOURCE_ID \
  --security-marks="jira_ticket=SEC-42,reviewed=true,reviewed_by=alice@example.com"

# Remove specific marks (set them to empty string to remove)
gcloud scc findings update FINDING_ID \
  --organization=ORG_ID \
  --source=SOURCE_ID \
  --security-marks="jira_ticket=" \
  --update-mask="securityMarks.marks.jira_ticket"

# Add marks to an asset
gcloud scc assets update-marks \
  --organization=ORG_ID \
  --asset=ASSET_ID \
  --security-marks="environment=prod,data_classification=PII"

# Filter findings by security mark
gcloud scc findings list \
  --organization=ORG_ID \
  --filter='securityMarks.marks.reviewed="true" AND state="ACTIVE"'
```

---

### Python: Security Marks

```python
from google.cloud import securitycenter
from google.cloud.securitycenter_v1 import SecurityMarks
from google.protobuf import field_mask_pb2


def add_marks_to_finding(org_id: str, source_id: str, finding_id: str,
                          marks: dict):
    """Add or update security marks on a finding without removing existing marks."""
    client = securitycenter.SecurityCenterClient()
    finding_name = f"organizations/{org_id}/sources/{source_id}/findings/{finding_id}"
    marks_name = f"{finding_name}/securityMarks"

    security_marks = SecurityMarks(
        name=marks_name,
        marks=marks,
    )
    # Update only the specified marks (field mask per-key)
    update_mask = field_mask_pb2.FieldMask(
        paths=[f"marks.{key}" for key in marks.keys()]
    )
    return client.update_security_marks(
        request={
            "security_marks": security_marks,
            "update_mask": update_mask,
        }
    )


# Usage
add_marks_to_finding(
    org_id="123456789",
    source_id="SOURCE_ID",
    finding_id="FINDING_ID",
    marks={
        "jira_ticket": "SEC-1234",
        "reviewed": "true",
        "reviewed_by": "alice@example.com",
    }
)
```

---

## 7. Notifications & Pub/Sub Integration

### How Notifications Work

```
SCC Finding (created or updated)
       │
       │  Evaluate against notification config filter
       ▼
  Filter matches?
       │
       ├── YES → Publish JSON finding to Pub/Sub topic (within seconds)
       │              │
       │              ├── Subscriber: Cloud Function → Jira/PagerDuty/Slack
       │              ├── Subscriber: Dataflow → BigQuery
       │              ├── Subscriber: Chronicle SIEM
       │              └── Subscriber: Cloud Function → automated remediation
       │
       └── NO  → Discarded (finding still in SCC)
```

---

### Setting Up Notification Configs

```bash
# Step 1: Create a Pub/Sub topic
gcloud pubsub topics create scc-critical-high-findings --project=my-project

# Step 2: Grant SCC service account publish rights
PROJECT_NUMBER=$(gcloud projects describe my-project --format="value(projectNumber)")
SCC_SA="service-org-ORG_ID@gcp-sa-scc-notification.iam.gserviceaccount.com"

gcloud pubsub topics add-iam-policy-binding scc-critical-high-findings \
  --project=my-project \
  --member="serviceAccount:${SCC_SA}" \
  --role="roles/pubsub.publisher"

# Step 3: Create the notification config
gcloud scc notifications create critical-high-alerts \
  --organization=ORG_ID \
  --pubsub-topic=projects/my-project/topics/scc-critical-high-findings \
  --filter='state="ACTIVE" AND (severity="CRITICAL" OR severity="HIGH")' \
  --description="Page on-call for CRITICAL/HIGH findings"

# List notification configs
gcloud scc notifications list --organization=ORG_ID

# Describe a config
gcloud scc notifications describe critical-high-alerts --organization=ORG_ID

# Update filter
gcloud scc notifications update critical-high-alerts \
  --organization=ORG_ID \
  --filter='state="ACTIVE" AND severity="CRITICAL"'

# Delete
gcloud scc notifications delete critical-high-alerts --organization=ORG_ID
```

---

### Notification Message Format

```json
{
  "notificationConfigName": "organizations/ORG/notificationConfigs/critical-high-alerts",
  "finding": {
    "name": "organizations/ORG/sources/SRC/findings/FINDING_ID",
    "parent": "organizations/ORG/sources/SRC",
    "resourceName": "//storage.googleapis.com/my-bucket",
    "state": "ACTIVE",
    "category": "PUBLIC_BUCKET_ACL",
    "severity": "CRITICAL",
    "findingClass": "MISCONFIGURATION",
    "eventTime": "2024-06-01T12:00:00Z",
    "createTime": "2024-06-01T12:00:05Z",
    "sourceProperties": {
      "ReactivationCount": "0",
      "ResourcePath": "//storage.googleapis.com/my-bucket"
    }
  },
  "resource": {
    "name": "//storage.googleapis.com/my-bucket",
    "projectName": "//cloudresourcemanager.googleapis.com/projects/12345",
    "projectDisplayName": "my-project",
    "type": "google.storage.Bucket"
  }
}
```

---

### Python: Create Notification Config and Process Messages

```python
from google.cloud import securitycenter, pubsub_v1
import json
import base64


def create_notification_config(org_id: str, config_id: str,
                                 pubsub_topic: str, filter_str: str):
    """Create an SCC notification config."""
    client = securitycenter.SecurityCenterClient()

    return client.create_notification_config(
        request={
            "parent": f"organizations/{org_id}",
            "config_id": config_id,
            "notification_config": {
                "description": f"Notification config: {config_id}",
                "pubsub_topic": pubsub_topic,
                "streaming_config": {"filter": filter_str},
            },
        }
    )


def process_scc_notification(message: pubsub_v1.types.ReceivedMessage) -> dict:
    """Parse an SCC notification Pub/Sub message."""
    data = json.loads(base64.b64decode(message.message.data).decode("utf-8"))
    finding = data.get("finding", {})
    resource = data.get("resource", {})

    return {
        "finding_name": finding.get("name"),
        "category": finding.get("category"),
        "severity": finding.get("severity"),
        "resource_name": finding.get("resourceName"),
        "project": resource.get("projectDisplayName"),
        "finding_class": finding.get("findingClass"),
        "event_time": finding.get("eventTime"),
    }
```

---

### Common Integration Patterns

```
Pattern 1: Pub/Sub → Cloud Function → Jira
  Filter: severity="CRITICAL" OR severity="HIGH"
  Function: create Jira issue, add jira_ticket security mark to finding

Pattern 2: Pub/Sub → Cloud Function → PagerDuty/Slack
  Filter: severity="CRITICAL" AND state="ACTIVE"
  Function: POST to PagerDuty Events API or Slack incoming webhook

Pattern 3: Pub/Sub → Dataflow → BigQuery
  Use Dataflow's Pub/Sub to BigQuery template
  Retains all finding updates for indefinite analytics

Pattern 4: Pub/Sub → Cloud Function → Automated Remediation
  Filter: category="PUBLIC_BUCKET_ACL" AND state="ACTIVE"
  Function: remove allUsers IAM binding from GCS bucket, mark finding INACTIVE
```

> 💡 **Use multiple notification configs** — one per severity tier (CRITICAL pages on-call, HIGH creates Jira ticket, MEDIUM/LOW goes to BigQuery only). Avoid one catch-all config that routes everything to Slack.

---

## 8. BigQuery Export (Continuous Export)

### Why Use BigQuery Export

SCC retains findings for **13 months rolling**. BigQuery export provides:
- Indefinite retention
- SQL-based ad-hoc querying
- Historical trend analysis and MTTR calculations
- Looker Studio / Data Studio dashboards
- Compliance evidence over multi-year periods

---

### Setting Up BigQuery Export

```bash
# Create a BigQuery dataset
bq mk --dataset \
  --location=US \
  --description="SCC findings continuous export" \
  my-project:scc_findings

# Create the BigQuery export config
gcloud scc bqexports create scc-all-findings-export \
  --organization=ORG_ID \
  --dataset=projects/my-project/datasets/scc_findings \
  --description="Export all SCC findings to BigQuery" \
  --filter=""  # empty filter = export everything

# Export only CRITICAL and HIGH findings
gcloud scc bqexports create scc-critical-high-export \
  --organization=ORG_ID \
  --dataset=projects/my-project/datasets/scc_findings_critical \
  --filter='severity="CRITICAL" OR severity="HIGH"'

# List / describe / update / delete
gcloud scc bqexports list --organization=ORG_ID
gcloud scc bqexports describe scc-all-findings-export --organization=ORG_ID
gcloud scc bqexports update scc-all-findings-export \
  --organization=ORG_ID \
  --description="Updated description"
gcloud scc bqexports delete scc-all-findings-export --organization=ORG_ID
```

---

### Useful BigQuery SQL Queries

```sql
-- 1. Active findings by severity and category (last 30 days)
SELECT
  severity,
  category,
  COUNT(*) AS finding_count
FROM `my-project.scc_findings.findings`
WHERE
  state = 'ACTIVE'
  AND event_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY severity, category
ORDER BY
  CASE severity WHEN 'CRITICAL' THEN 1 WHEN 'HIGH' THEN 2
    WHEN 'MEDIUM' THEN 3 WHEN 'LOW' THEN 4 ELSE 5 END,
  finding_count DESC;

-- 2. Mean time to resolve CRITICAL findings (in hours) per project
SELECT
  resource.project_display_name AS project,
  AVG(TIMESTAMP_DIFF(
    (SELECT MIN(f2.update_time) FROM `my-project.scc_findings.findings` f2
     WHERE f2.name = f.name AND f2.state = 'INACTIVE'),
    f.create_time,
    HOUR
  )) AS avg_resolution_hours
FROM `my-project.scc_findings.findings` f
WHERE
  f.severity = 'CRITICAL'
  AND f.state = 'INACTIVE'
GROUP BY project
ORDER BY avg_resolution_hours DESC;

-- 3. Top 10 projects by open HIGH/CRITICAL findings
SELECT
  resource.project_display_name AS project,
  COUNTIF(severity = 'CRITICAL') AS critical_count,
  COUNTIF(severity = 'HIGH') AS high_count,
  COUNT(*) AS total
FROM `my-project.scc_findings.findings`
WHERE
  state = 'ACTIVE'
  AND mute != 'MUTED'
  AND severity IN ('CRITICAL', 'HIGH')
GROUP BY project
ORDER BY critical_count DESC, high_count DESC
LIMIT 10;

-- 4. Findings trend over time (weekly aggregation)
SELECT
  DATE_TRUNC(DATE(event_time), WEEK) AS week,
  severity,
  COUNT(*) AS new_findings
FROM `my-project.scc_findings.findings`
WHERE event_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
GROUP BY week, severity
ORDER BY week, severity;

-- 5. THREAT class findings in the last 7 days with full detail
SELECT
  event_time,
  category,
  severity,
  resource.project_display_name AS project,
  resource.display_name AS resource,
  description,
  next_steps
FROM `my-project.scc_findings.findings`
WHERE
  finding_class = 'THREAT'
  AND state = 'ACTIVE'
  AND event_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
ORDER BY
  CASE severity WHEN 'CRITICAL' THEN 1 WHEN 'HIGH' THEN 2 ELSE 3 END,
  event_time DESC;

-- 6. Most common misconfiguration categories across the org
SELECT
  category,
  COUNT(DISTINCT resource.project_display_name) AS affected_projects,
  COUNT(*) AS total_findings
FROM `my-project.scc_findings.findings`
WHERE
  finding_class = 'MISCONFIGURATION'
  AND state = 'ACTIVE'
  AND mute != 'MUTED'
GROUP BY category
ORDER BY total_findings DESC
LIMIT 20;

-- 7. Projects with no improvement in finding count over 30 days
WITH
  snapshot_30d_ago AS (
    SELECT resource.project_display_name AS project, COUNT(*) AS count_30d
    FROM `my-project.scc_findings.findings`
    WHERE state = 'ACTIVE'
      AND event_time <= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
    GROUP BY project
  ),
  current_snapshot AS (
    SELECT resource.project_display_name AS project, COUNT(*) AS count_now
    FROM `my-project.scc_findings.findings`
    WHERE state = 'ACTIVE'
    GROUP BY project
  )
SELECT
  c.project,
  s.count_30d AS findings_30d_ago,
  c.count_now AS findings_today,
  c.count_now - s.count_30d AS delta
FROM current_snapshot c
JOIN snapshot_30d_ago s USING (project)
WHERE c.count_now >= s.count_30d
ORDER BY delta DESC;
```

---

## 9. Compliance & Posture Management

### Compliance Frameworks (Premium)

| Framework | Key Controls Covered |
|---|---|
| CIS GCP Foundations Benchmark | IAM, logging, networking, VM config, storage, KMS |
| PCI-DSS | Access control, audit logging, encryption, network security |
| NIST SP 800-53 | Access control (AC), audit (AU), configuration management (CM), incident response (IR) |
| ISO 27001 | Information security policies, asset management, access control, cryptography |
| HIPAA | PHI access controls, audit controls, transmission security |
| SOC 2 | Availability, confidentiality, processing integrity, security, privacy |
| FedRAMP | Based on NIST 800-53 moderate baseline |

Compliance scores are calculated as: `(passing controls / total controls) × 100%`. A finding mapped to a control causes that control to fail.

---

### Custom SHA Modules (Premium)

Custom SHA modules let you write your own misconfiguration detectors using **Common Expression Language (CEL)** on resource configuration data.

```yaml
# custom-sha-module.yaml
# Detect GCS buckets without uniform bucket-level access (UBLA)
displayName: "GCS Bucket Without Uniform Bucket-Level Access"
description: "Detects GCS buckets that do not have uniform bucket-level access enabled."
severity: MEDIUM
enablementState: ENABLED
resourceSelector:
  resourceTypes:
    - storage.googleapis.com/Bucket
predicate:
  expression: >
    !(resource.iamConfiguration.uniformBucketLevelAccess.enabled == true)
recommendation: "Enable uniform bucket-level access: gsutil uniformbucketlevelaccess set on gs://BUCKET"
```

```bash
# Create custom SHA module
gcloud scc custom-modules sha create \
  --organization=ORG_ID \
  --display-name="GCS Bucket Without UBLA" \
  --enablement-state=ENABLED \
  --custom-config-from-file=custom-sha-module.yaml

# List custom SHA modules
gcloud scc custom-modules sha list --organization=ORG_ID

# Test a custom module (without creating findings)
gcloud scc custom-modules sha test \
  --organization=ORG_ID \
  --custom-config-from-file=custom-sha-module.yaml \
  --resource=//storage.googleapis.com/my-test-bucket
```

---

### Posture Management (Enterprise)

```yaml
# Example posture YAML (security-posture.yaml)
name: organizations/ORG_ID/locations/global/postures/my-security-posture
state: ACTIVE
description: "Standard GCP security posture for production workloads"
policySets:
  - policySetId: cis-l1-gcp
    description: "CIS GCP Foundations Benchmark Level 1"
    policies:
      - policyId: audit-logging-enabled
        complianceStandards:
          - standard: CIS_GCP
            control: "2.1"
        constraint:
          securityHealthAnalyticsModule:
            moduleName: AUDIT_LOGGING_DISABLED
            moduleEnablementState: ENABLED
      - policyId: no-public-buckets
        constraint:
          securityHealthAnalyticsModule:
            moduleName: PUBLIC_BUCKET_ACL
            moduleEnablementState: ENABLED
```

```bash
# Create a posture definition
gcloud scc postures create my-security-posture \
  --organization=ORG_ID \
  --location=global \
  --posture-from-file=security-posture.yaml

# Deploy posture to a project
gcloud scc postures deploy \
  --organization=ORG_ID \
  --posture-name=organizations/ORG_ID/locations/global/postures/my-security-posture \
  --posture-revision-id=REVISION_ID \
  --workload=projects/PROJECT_NUMBER

# List posture deployments
gcloud scc postures list-deployments --organization=ORG_ID --location=global
```

---

## 10. Custom Sources & Custom Findings

### Registering a Custom Source

```python
from google.cloud import securitycenter
from google.cloud.securitycenter_v1 import Source


def register_custom_source(org_id: str, display_name: str, description: str) -> str:
    """Register a custom security source in SCC."""
    client = securitycenter.SecurityCenterClient()
    org_name = f"organizations/{org_id}"

    source = client.create_source(
        request={
            "parent": org_name,
            "source": Source(
                display_name=display_name,
                description=description,
            ),
        }
    )
    print(f"Created source: {source.name}")
    return source.name  # "organizations/ORG/sources/SOURCE_ID"
```

---

### Creating Custom Findings

```python
from google.cloud import securitycenter
from google.cloud.securitycenter_v1 import Finding
from google.protobuf.timestamp_pb2 import Timestamp
import time
import uuid


def create_custom_finding(org_id: str, source_id: str,
                           resource_name: str, category: str,
                           severity: str, description: str,
                           next_steps: str, source_properties: dict) -> Finding:
    """Create a custom finding under a registered source."""
    client = securitycenter.SecurityCenterClient()
    source_name = f"organizations/{org_id}/sources/{source_id}"
    finding_id = uuid.uuid4().hex[:32]

    ts = Timestamp()
    ts.FromSeconds(int(time.time()))

    finding = Finding(
        state=Finding.State.ACTIVE,
        resource_name=resource_name,
        category=category,
        severity=Finding.Severity[severity],     # "CRITICAL", "HIGH", etc.
        finding_class=Finding.FindingClass.VULNERABILITY,
        description=description,
        next_steps=next_steps,
        event_time=ts,
        source_properties={k: str(v) for k, v in source_properties.items()},
    )
    return client.create_finding(
        request={
            "parent": source_name,
            "finding_id": finding_id,
            "finding": finding,
        }
    )
```

---

### gcloud: Source Management

```bash
# Create a custom source
gcloud scc sources create \
  --organization=ORG_ID \
  --display-name="Internal Pentest Tool" \
  --description="Findings from internal penetration testing tool"

# List sources
gcloud scc sources list --organization=ORG_ID

# Describe a source
gcloud scc sources describe SOURCE_ID --organization=ORG_ID

# Grant findingsEditor to SA that publishes findings
gcloud scc sources set-iam-policy SOURCE_ID \
  --organization=ORG_ID \
  --policy-file=source-policy.json
```

---

### Custom Findings Best Practices

> 💡 **Naming conventions:** Use `CATEGORY_IN_CAPS_WITH_UNDERSCORES` matching your tool's finding type. Populate `nextSteps` with specific remediation commands. Set `externalUri` to the finding in your source tool (e.g., Qualys, Tenable URL). Always set `eventTime` to when the vulnerability was detected, not when you published it.

---

## 11. IAM & Access Control for SCC

### IAM Roles Reference

| Role | Permissions | Who Needs It |
|---|---|---|
| `roles/securitycenter.admin` | Full control — configure SCC, manage all resources | SCC platform owner |
| `roles/securitycenter.adminViewer` | Read-only admin view — view all SCC config and findings | Security team, CISO |
| `roles/securitycenter.findingsEditor` | Update finding state, security marks | Remediation teams, automation SAs |
| `roles/securitycenter.findingsViewer` | Read findings only | SOC analysts, read-only dashboards |
| `roles/securitycenter.sourcesEditor` | Create and manage sources | Custom source publishers |
| `roles/securitycenter.sourcesViewer` | View sources | Auditors |
| `roles/securitycenter.notificationConfigEditor` | Create/manage notification configs | Security automation team |
| `roles/securitycenter.settingsEditor` | Update SCC settings | SCC admin |
| `roles/securitycenter.settingsViewer` | View SCC settings | Auditors |

---

### gcloud: IAM for SCC

```bash
# Grant adminViewer at org level (security team — can see everything, cannot modify config)
gcloud organizations add-iam-policy-binding ORG_ID \
  --member="group:security-team@example.com" \
  --role="roles/securitycenter.adminViewer"

# Grant findingsViewer at folder level (SOC team scoped to a folder)
gcloud resource-manager folders add-iam-policy-binding FOLDER_ID \
  --member="group:soc-team@example.com" \
  --role="roles/securitycenter.findingsViewer"

# Grant findingsEditor to a remediation automation SA
gcloud organizations add-iam-policy-binding ORG_ID \
  --member="serviceAccount:scc-remediation-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/securitycenter.findingsEditor"

# Grant sourcesEditor + findingsEditor to custom source publisher SA
gcloud scc sources set-iam-policy SOURCE_ID \
  --organization=ORG_ID \
  --policy-file=- <<EOF
{
  "bindings": [{
    "role": "roles/securitycenter.findingsEditor",
    "members": ["serviceAccount:pentest-tool-sa@my-project.iam.gserviceaccount.com"]
  }]
}
EOF

# View IAM policy for SCC at org level
gcloud organizations get-iam-policy ORG_ID \
  --filter="bindings.role:securitycenter"
```

---

## 12. Attack Path Simulation (Premium)

### How It Works

```
SCC ingests:
  ├── IAM bindings (who can access what)
  ├── Network topology (firewall rules, VPC peering, load balancers)
  ├── Vulnerability findings (CVEs, misconfigurations, exposed services)
  └── Public exposure points (external IPs, public load balancers, public buckets)
          │
          ▼
  Graph-based attack simulation
  "If an attacker compromises this exposed resource, where can they go?"
          │
          ▼
  Attack paths with exposure scores (0–10)
  Exposed findings sorted by attack exposure score
```

**Key Concepts:**

| Concept | Definition |
|---|---|
| **Attack path** | Sequence of resource nodes an attacker could traverse from entry point to target |
| **Attack exposure score** | 0–10 score indicating how easily a resource is reachable via attack paths |
| **High-value resource** | Resource designated as a sensitive target (data stores, secret holders, admin VMs) |
| **Attack path node** | A GCP resource appearing in a simulated attack path |
| **Attack path edge** | The lateral movement vector between two nodes (IAM permission, network route) |

---

### Designating High-Value Resources

SCC automatically considers some resources high-value (Cloud SQL, GCS buckets with sensitive data labels, Secret Manager secrets). You can also designate resources via labels:

```bash
# Label a GCS bucket as high-value
gsutil label ch -l "scc-high-value:true" gs://my-sensitive-bucket

# Label a Compute VM as high-value
gcloud compute instances add-labels my-critical-vm \
  --zone=us-central1-a \
  --labels=scc-high-value=true
```

> 💡 **Prioritize remediation by exposure score**, not just severity. A MEDIUM finding with an attack exposure score of 9.5 should be fixed before a HIGH finding with a score of 0.2.

---

## 13. Event Threat Detection Deep Dive

### ETD Architecture

```
Cloud Audit Logs (Admin Activity + Data Access)
VPC Flow Logs
DNS Query Logs
GKE Audit Logs
Cloud Firewall Logs
        │
        ▼
  Cloud Logging → Log Router
        │
        ▼
  ETD Analysis Engine (near-real-time, 1–5 min)
  ├── Built-in ML + rule-based detectors
  └── Custom YARA-L modules (Premium)
        │
        ▼
  SCC Findings (findingClass=THREAT)
```

---

### ETD Detector Categories

| Category | Example Detectors | Primary Log Source |
|---|---|---|
| `INITIAL_ACCESS` | Leaked SA key used externally, anomalous access from Tor IP | Admin Activity, Data Access |
| `PRIVILEGE_ESCALATION` | Anomalous IAM grant (role binding added to high-privilege role) | Admin Activity |
| `DEFENSE_EVASION` | Audit log disabled, GCS bucket logging disabled, SCC notifications deleted | Admin Activity |
| `DISCOVERY` | Abnormal API enumeration (ListBuckets, ListInstances by anomalous principal) | Admin Activity |
| `LATERAL_MOVEMENT` | SA impersonation chain, cross-project SA token usage | Admin Activity, Data Access |
| `CREDENTIAL_ACCESS` | SA key created by unexpected user, secret accessed abnormally | Admin Activity, Data Access |
| `EXFILTRATION` | Data copied to external GCS bucket, BigQuery data exported externally | Data Access, Admin Activity |
| `IMPACT` | Crypto mining (Compute/GKE), mass resource deletion, ransomware indicators | Admin Activity, VMTD |

---

### ETD Custom Modules (YARA-L)

```yaml
# custom-etd-module.yaml
# Detect API calls from untrusted IP ranges
displayName: "Access from Untrusted IP Range"
description: "Detects GCP API calls originating from IPs outside trusted CIDR ranges"
severity: HIGH
enablementState: ENABLED
type: CLOUD_AUDIT_LOG
metadata:
  rule: |
    rule detect_untrusted_ip {
      meta:
        author = "security@example.com"
        severity = "HIGH"
      events:
        $e.metadata.event_type = "USER_RESOURCE_ACCESS"
        $e.principal.ip != /^(10\.|192\.168\.|172\.(1[6-9]|2[0-9]|3[01])\.).*$/
        $e.principal.ip != "203.0.113.0/24"  # trusted office IP
      condition: $e
    }
```

```bash
# Create custom ETD module
gcloud scc custom-modules etd create \
  --organization=ORG_ID \
  --display-name="Access from Untrusted IP" \
  --enablement-state=ENABLED \
  --custom-config-from-file=custom-etd-module.yaml

# Test a custom ETD module against existing logs
gcloud scc custom-modules etd test \
  --organization=ORG_ID \
  --custom-config-from-file=custom-etd-module.yaml

# List all custom ETD modules
gcloud scc custom-modules etd list --organization=ORG_ID
```

---

### Required Log Enablement for Full ETD Coverage

```bash
# Enable Data Access audit logs (CRITICAL — not on by default)
# Do this for all critical APIs via Console: IAM → Audit Logs → select all services

# Enable VPC Flow Logs on subnets
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-flow-logs \
  --logging-aggregation-interval=interval-5-sec \
  --logging-flow-sampling=1.0

# Enable DNS logging on Cloud DNS
gcloud dns policies update my-policy \
  --enable-logging \
  --networks=my-vpc

# Enable GKE audit logging
gcloud container clusters update my-cluster \
  --region=us-central1 \
  --logging=SYSTEM,WORKLOAD,API_SERVER
```

---

## 14. Automating Remediation

### Architecture

```
SCC Finding (new or updated)
       │
       ▼
SCC Notification → Pub/Sub Topic
       │
       ▼
Cloud Function / Cloud Run (triggered by Pub/Sub)
       │
       ├── Parse finding JSON (category, severity, resourceName, project)
       ├── Check: is this category auto-remediable?
       ├── Check: is finding already muted/remediated? (idempotency)
       ├── DRY_RUN mode? → log action, skip execution
       ├── Execute remediation action
       ├── Update finding: state=INACTIVE or add security mark
       └── On failure: create Jira ticket + alert
```

---

### Python: Cloud Function Remediation Handler

```python
import base64
import json
import functions_framework
from google.cloud import storage, securitycenter
import logging

logger = logging.getLogger(__name__)

REMEDIATORS = {}  # registry of category → handler function

def remediator(category: str):
    """Decorator to register a remediation handler for a finding category."""
    def decorator(fn):
        REMEDIATORS[category] = fn
        return fn
    return decorator


@functions_framework.cloud_event
def handle_scc_finding(cloud_event):
    """Cloud Function entry point: processes SCC findings from Pub/Sub."""
    data = json.loads(base64.b64decode(cloud_event.data["message"]["data"]))
    finding = data.get("finding", {})

    category = finding.get("category", "")
    severity = finding.get("severity", "")
    resource_name = finding.get("resourceName", "")
    finding_name = finding.get("name", "")
    mute_state = finding.get("mute", "UNMUTED")

    # Idempotency: skip already-muted or already-remediated findings
    marks = finding.get("securityMarks", {}).get("marks", {})
    if mute_state == "MUTED" or marks.get("auto_remediated") == "true":
        logger.info(f"Skipping already-processed finding: {finding_name}")
        return

    logger.info(f"Processing {severity} finding: {category} on {resource_name}")

    handler = REMEDIATORS.get(category)
    if not handler:
        logger.info(f"No auto-remediation for category: {category}")
        return

    try:
        handler(finding)
        _mark_remediated(finding_name)
    except Exception as e:
        logger.error(f"Remediation failed for {finding_name}: {e}")
        _create_ticket(finding, error=str(e))
        raise


@remediator("PUBLIC_BUCKET_ACL")
def remediate_public_bucket(finding: dict):
    """Remove public access from a GCS bucket."""
    resource_name = finding["resourceName"]
    # Extract bucket name from resource: //storage.googleapis.com/BUCKET_NAME
    bucket_name = resource_name.split("/")[-1]

    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    policy = bucket.get_iam_policy(requested_policy_version=3)

    original_count = len(policy.bindings)
    policy.bindings = [
        b for b in policy.bindings
        if "allUsers" not in b["members"]
        and "allAuthenticatedUsers" not in b["members"]
    ]
    bucket.set_iam_policy(policy)
    logger.info(f"Removed {original_count - len(policy.bindings)} public bindings from {bucket_name}")


def _mark_remediated(finding_name: str):
    """Add auto_remediated security mark to the finding."""
    from google.cloud.securitycenter_v1 import SecurityMarks
    from google.protobuf import field_mask_pb2
    import time

    client = securitycenter.SecurityCenterClient()
    client.update_security_marks(
        request={
            "security_marks": SecurityMarks(
                name=f"{finding_name}/securityMarks",
                marks={"auto_remediated": "true",
                       "remediation_time": str(int(time.time()))},
            ),
            "update_mask": field_mask_pb2.FieldMask(
                paths=["marks.auto_remediated", "marks.remediation_time"]
            ),
        }
    )


def _create_ticket(finding: dict, error: str = ""):
    """Create a Jira ticket for findings that cannot be auto-remediated."""
    # Implement your Jira/ServiceNow/PagerDuty integration here
    logger.warning(f"Manual intervention needed: {finding.get('category')} | Error: {error}")
```

---

### Common Remediation Patterns

| Finding Category | Auto-Remediable? | Remediation Action | Safety Consideration |
|---|---|---|---|
| `PUBLIC_BUCKET_ACL` | ✅ | Remove `allUsers`/`allAuthenticatedUsers` IAM bindings | Verify bucket is not a public website |
| `OPEN_FIREWALL` | ⚠️ With approval | Delete overly permissive rule or add specific deny | May break application access — use approval gate |
| `SQL_NO_ROOT_PASSWORD` | ❌ | Alert DBA team + create ticket | Cannot auto-set passwords |
| `OVER_PRIVILEGED_SERVICE_ACCOUNT` | ⚠️ | Remove excess role bindings | Risk of breaking dependent services |
| `AUDIT_LOGGING_DISABLED` | ✅ | Re-enable audit logging via API | Low risk |
| `CRYPTO_MINING` (ETD) | ⚠️ | Stop VM, capture disk snapshot for forensics | Verify it's not a false positive first |
| `ANOMALOUS_IAM_GRANT` (ETD) | ⚠️ | Revoke suspicious binding, quarantine SA | High risk — requires human review |

---

## 15. Multi-Cloud Security (Enterprise)

### Architecture

```
AWS Security Hub findings
        │  (SCC Enterprise connector)
        ▼
Microsoft Defender for Cloud findings
        │  (SCC Enterprise connector)
        ▼
┌──────────────────────────────────────────┐
│         SCC Enterprise                   │
│  ┌────────────┐  ┌─────────┐  ┌───────┐ │
│  │ GCP native │  │   AWS   │  │ Azure │ │
│  │  findings  │  │findings │  │finding│ │
│  └────────────┘  └─────────┘  └───────┘ │
│       Unified view, filtering, compliance│
└──────────────────────────────────────────┘
```

---

### Multi-Cloud Finding Source Names

```bash
# Filter for only AWS findings
gcloud scc findings list --organization=ORG_ID \
  --filter='source.display_name="Amazon Web Services"'

# Filter for only Azure findings
gcloud scc findings list --organization=ORG_ID \
  --filter='source.display_name="Microsoft Azure"'

# Filter for all non-GCP findings
gcloud scc findings list --organization=ORG_ID \
  --filter='source.display_name!="Security Health Analytics" AND source.display_name!="Event Threat Detection"'
```

> ⚠️ **Multi-cloud findings are ingested, not native.** SCC Enterprise displays AWS and Azure findings but cannot directly remediate them. Use AWS Console/CLI or Azure Portal for remediation. SCC serves as the unified visibility layer.

---

## 16. Monitoring, Alerting & Audit Logging

### Audit Logging for SCC Operations

| Log Type | SCC Operations Captured |
|---|---|
| **Admin Activity** (always on) | `CreateNotificationConfig`, `DeleteNotificationConfig`, `CreateMuteConfig`, `UpdateMuteConfig`, `CreateSource`, `SetIamPolicy`, `UpdateSettings`, `CreateBigQueryExport` |
| **Data Access** (must enable) | `GetFinding`, `ListFindings`, `GetSource`, `ListSources`, `GetNotificationConfig` |

---

### Cloud Logging Queries for SCC Operations

```bash
# Who accessed which finding in the last 24 hours
gcloud logging read \
  'resource.type="audited_resource"
   protoPayload.serviceName="securitycenter.googleapis.com"
   protoPayload.methodName="google.cloud.securitycenter.v1.SecurityCenter.GetFinding"' \
  --freshness=24h \
  --format="table(timestamp, protoPayload.authenticationInfo.principalEmail, protoPayload.resourceName)"

# Who created or modified a mute config
gcloud logging read \
  'protoPayload.serviceName="securitycenter.googleapis.com"
   protoPayload.methodName=~"CreateMuteConfig|UpdateMuteConfig|DeleteMuteConfig"' \
  --freshness=7d

# Who changed IAM policy on SCC sources
gcloud logging read \
  'protoPayload.serviceName="securitycenter.googleapis.com"
   protoPayload.methodName="google.cloud.securitycenter.v1.SecurityCenter.SetIamPolicy"' \
  --freshness=30d

# All finding state changes by automated systems (SA-driven)
gcloud logging read \
  'protoPayload.serviceName="securitycenter.googleapis.com"
   protoPayload.methodName="google.cloud.securitycenter.v1.SecurityCenter.UpdateFinding"
   protoPayload.authenticationInfo.principalEmail:"@.iam.gserviceaccount.com"' \
  --freshness=7d
```

---

### Alerting on CRITICAL Finding Spikes

```bash
# 1. Create a log-based metric for new CRITICAL SCC findings
gcloud logging metrics create scc-new-critical-findings \
  --description="New CRITICAL findings in SCC" \
  --log-filter='resource.type="audited_resource"
protoPayload.serviceName="securitycenter.googleapis.com"
protoPayload.methodName="google.cloud.securitycenter.v1.SecurityCenter.CreateFinding"
protoPayload.request.finding.severity="CRITICAL"'

# 2. Create a Cloud Monitoring alerting policy on the metric
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="SCC: New CRITICAL Finding Detected" \
  --condition-filter='metric.type="logging.googleapis.com/user/scc-new-critical-findings"' \
  --condition-threshold-value=0 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=0s
```

---

### Weekly Findings Summary Report

```
Cloud Scheduler (weekly cron)
       │
       ▼
Cloud Function
       │
       ├── Query BigQuery: open CRITICAL/HIGH findings by project
       ├── Query BigQuery: new findings this week vs. last week
       ├── Query BigQuery: MTTR for closed findings
       ├── Format HTML summary table
       └── Send via SendGrid / Gmail API to security-team@example.com
```

---

## 17. gcloud CLI Quick Reference

### Findings

```bash
gcloud scc findings list --organization=ORG_ID [--source=SRC_ID] [--filter=FILTER] [--format=FORMAT] [--page-size=N]
gcloud scc findings update FINDING_ID --organization=ORG_ID --source=SRC_ID [--state=ACTIVE|INACTIVE] [--security-marks=KEY=VALUE,...]
gcloud scc findings group --organization=ORG_ID --group-by=FIELD [--filter=FILTER]
```

### Sources

```bash
gcloud scc sources create --organization=ORG_ID --display-name=NAME --description=DESC
gcloud scc sources list --organization=ORG_ID
gcloud scc sources describe SRC_ID --organization=ORG_ID
gcloud scc sources update SRC_ID --organization=ORG_ID --display-name=NEW_NAME
gcloud scc sources get-iam-policy SRC_ID --organization=ORG_ID
gcloud scc sources set-iam-policy SRC_ID --organization=ORG_ID --policy-file=FILE
```

### Notifications

```bash
gcloud scc notifications create CONFIG_ID --organization=ORG_ID --pubsub-topic=TOPIC --filter=FILTER [--description=DESC]
gcloud scc notifications list --organization=ORG_ID
gcloud scc notifications describe CONFIG_ID --organization=ORG_ID
gcloud scc notifications update CONFIG_ID --organization=ORG_ID [--filter=FILTER] [--pubsub-topic=TOPIC]
gcloud scc notifications delete CONFIG_ID --organization=ORG_ID
```

### Mute Configs

```bash
gcloud scc muteconfigs create CONFIG_ID --organization=ORG_ID --filter=FILTER --description=DESC
gcloud scc muteconfigs list --organization=ORG_ID
gcloud scc muteconfigs describe CONFIG_ID --organization=ORG_ID
gcloud scc muteconfigs update CONFIG_ID --organization=ORG_ID [--filter=FILTER] [--description=DESC]
gcloud scc muteconfigs delete CONFIG_ID --organization=ORG_ID
```

### BigQuery Exports

```bash
gcloud scc bqexports create EXPORT_ID --organization=ORG_ID --dataset=DATASET [--filter=FILTER] [--description=DESC]
gcloud scc bqexports list --organization=ORG_ID
gcloud scc bqexports describe EXPORT_ID --organization=ORG_ID
gcloud scc bqexports update EXPORT_ID --organization=ORG_ID [--filter=FILTER]
gcloud scc bqexports delete EXPORT_ID --organization=ORG_ID
```

### Postures (Enterprise)

```bash
gcloud scc postures create POSTURE_ID --organization=ORG_ID --location=global --posture-from-file=FILE
gcloud scc postures list --organization=ORG_ID --location=global
gcloud scc postures describe POSTURE_ID --organization=ORG_ID --location=global
gcloud scc postures update POSTURE_ID --organization=ORG_ID --location=global --posture-from-file=FILE
gcloud scc postures delete POSTURE_ID --organization=ORG_ID --location=global
gcloud scc postures deploy --organization=ORG_ID --posture-name=NAME --posture-revision-id=ID --workload=RESOURCE
gcloud scc postures list-deployments --organization=ORG_ID --location=global
```

### Custom Modules

```bash
# SHA custom modules
gcloud scc custom-modules sha create --organization=ORG_ID --display-name=NAME --enablement-state=ENABLED --custom-config-from-file=FILE
gcloud scc custom-modules sha list --organization=ORG_ID
gcloud scc custom-modules sha describe MODULE_ID --organization=ORG_ID
gcloud scc custom-modules sha update MODULE_ID --organization=ORG_ID --custom-config-from-file=FILE
gcloud scc custom-modules sha delete MODULE_ID --organization=ORG_ID
gcloud scc custom-modules sha test --organization=ORG_ID --custom-config-from-file=FILE --resource=RESOURCE_NAME

# ETD custom modules
gcloud scc custom-modules etd create --organization=ORG_ID --display-name=NAME --enablement-state=ENABLED --custom-config-from-file=FILE
gcloud scc custom-modules etd list --organization=ORG_ID
gcloud scc custom-modules etd describe MODULE_ID --organization=ORG_ID
gcloud scc custom-modules etd test --organization=ORG_ID --custom-config-from-file=FILE
gcloud scc custom-modules etd delete MODULE_ID --organization=ORG_ID
```

### Settings & Key Flags

```bash
gcloud scc settings describe --organization=ORG_ID
gcloud scc settings update --organization=ORG_ID --enable-asset-discovery
```

| Flag | Applies To | Description |
|---|---|---|
| `--organization` | Most commands | Organization ID (e.g., `123456789`) |
| `--folder` | findings, muteconfigs | Scope to a specific folder |
| `--project` | findings, muteconfigs | Scope to a specific project |
| `--source` | findings | Source ID or `-` for all sources |
| `--filter` | findings, notifications, bqexports | CEL filter expression |
| `--location` | postures | `global` for most posture operations |
| `--format` | Any | `json`, `yaml`, `table(field,...)` |
| `--page-size` | List commands | Results per page (max 1000 for findings) |
| `--sort-by` | findings list | `eventTime`, `severity`, `createTime` |

---

### REST API Base URLs

```
Base: https://securitycenter.googleapis.com/v1/

Organizations:    organizations/{org}
Sources:          organizations/{org}/sources/{source}
Findings:         organizations/{org}/sources/{source}/findings/{finding}
Assets:           organizations/{org}/assets
Notifications:    organizations/{org}/notificationConfigs/{config}
Mute configs:     organizations/{org}/muteConfigs/{config}
BQ exports:       organizations/{org}/bigQueryExports/{export}
Settings:         organizations/{org}/organizationSettings

Operations:
  POST .../sources/{s}/findings/{f}:setState
  POST .../sources/{s}/findings/{f}:setMute
  POST .../sources/-/findings:bulkMute
  POST .../sources/{s}/findings/{f}/securityMarks:updateSecurityMarks
  GET  .../sources/-/findings?filter=...&orderBy=...
  POST .../notificationConfigs
  POST .../bigQueryExports
```

---

## 18. Best Practices & Operational Patterns

### ✅ Do This

| Practice | Why |
|---|---|
| **Enable SCC at org level** | Cross-project threat correlation, unified compliance, org-wide findings |
| **Enable Premium for production** | ETD and CTD are table-stakes for threat detection on GCP |
| **Enable Data Access audit logs for all critical APIs** | ETD depends on these — without them, most threat detections are blind |
| **Set up BigQuery export from day 1** | 13-month rolling retention is insufficient for compliance and forensics |
| **Create notification configs for CRITICAL/HIGH** | Page on-call; don't wait for daily dashboard review |
| **Define mute governance: justification + owner + review date** | Prevents security debt accumulation via unreviewed mutes |
| **Tag high-value resources** | Influences Attack Path Simulation prioritization |
| **Integrate SCC API into CI/CD pipelines** | Surface SHA findings during PRs and deployments |
| **Pair SCC with preventive controls** | SCC is detection — pair with Org Policy, VPC-SC, Binary Authorization for prevention |
| **Use SLA targets: CRITICAL=24h, HIGH=7d, MEDIUM=30d, LOW=90d** | Drives accountability and measurable security improvement |

---

### ❌ Anti-Patterns

| Anti-Pattern | Risk | Fix |
|---|---|---|
| Project-level SCC only | No cross-project threat correlation, no org compliance | Enable at org level |
| Standard tier in production | No ETD, no CTD, no compliance dashboards | Upgrade to Premium |
| Data Access logs not enabled | ETD misses exfiltration and privilege escalation threats | Enable for all critical GCP APIs |
| No BigQuery export | Findings lost after 13 months | Set up continuous BigQuery export on day 1 |
| Muting without governance | Silent security debt accumulation | Require description + owner + review date for all mutes |
| Ignoring MEDIUM findings | MEDIUM findings accumulate into systemic risk | Include MEDIUM in regular triage cadence |
| Not enabling VPC Flow Logs | ETD lateral movement detection requires them | Enable on all production subnets |
| Automated remediation without dry-run mode | Accidental service disruption | Always implement dry-run mode first |
| CRITICAL notifications going to a low-signal channel | Missed critical alerts | Use PagerDuty / paging-only channel for CRITICAL |
| Not reviewing mute configs quarterly | Muted findings never reviewed; security posture degrades | Schedule quarterly mute config reviews |

---

### SCC Findings Triage Workflow

```
New CRITICAL/HIGH Finding Appears
           │
           ▼
1. Identify: What is the finding category? Which resource? Which project?
           │
           ▼
2. Assess: Is this a real issue? (False positive? Already fixed? Accepted risk?)
           │
    ┌──────┴──────┐
   Yes            No (false positive)
    │              └─→ Create mute config with justification
    ▼
3. Blast radius: What data/services could be compromised?
           │
           ▼
4. Prioritize: Severity × Attack Exposure Score × Blast Radius
           │
           ▼
5. Remediate: Apply fix → verify → mark INACTIVE
           │
           ▼
6. Track: Create Jira ticket, add security mark, update SLA metrics
```

---

### SCC Operational Checklist

```
Initial Setup
  [ ] SCC enabled at organization level (not just project level)
  [ ] Premium tier enabled for production org
  [ ] SCC data access audit logs enabled for SCC itself

Detection Coverage
  [ ] Data Access audit logs enabled for BigQuery, GCS, Cloud SQL, Secret Manager, KMS
  [ ] VPC Flow Logs enabled on all production subnets
  [ ] DNS query logging enabled on all VPCs
  [ ] GKE audit logging enabled (API_SERVER, SYSTEM, WORKLOAD)
  [ ] Container Threat Detection confirmed enabled on GKE clusters

Alerting & Notifications
  [ ] Notification config for CRITICAL findings → PagerDuty / on-call
  [ ] Notification config for HIGH findings → Jira auto-creation
  [ ] Notification config for THREAT class findings → SOC channel

Data Retention & Analytics
  [ ] BigQuery continuous export configured (no filter — export everything)
  [ ] BigQuery dataset access-controlled and encrypted (CMEK)
  [ ] Looker Studio dashboard connected to BigQuery export

Mute Governance
  [ ] Mute config governance policy documented
  [ ] All mute configs have description + owner + review date
  [ ] Quarterly mute config review scheduled in calendar

Remediation
  [ ] Remediation SLAs defined and tracked: CRITICAL=24h, HIGH=7d, MEDIUM=30d
  [ ] Automated remediation Cloud Function deployed (PUBLIC_BUCKET_ACL, AUDIT_LOGGING_DISABLED)
  [ ] All automated remediations have dry-run mode tested before live execution

IAM
  [ ] Security team has adminViewer at org level (not admin)
  [ ] SOC analysts have findingsViewer (scoped to folder if applicable)
  [ ] No human accounts have securitycenter.admin without break-glass justification
  [ ] Remediation SAs have findingsEditor but NOT sourcesEditor or admin
```

---

## 19. Quick Reference & Comparison Tables

### SCC Tiers Comparison

| Feature | Standard | Premium | Enterprise |
|---|---|---|---|
| Security Health Analytics | ~20 basic checks | ✅ 150+ checks | ✅ 150+ checks |
| Event Threat Detection | ❌ | ✅ | ✅ + Mandiant Intel |
| Container Threat Detection | ❌ | ✅ | ✅ |
| Web Security Scanner (managed) | ❌ | ✅ | ✅ |
| VM Manager / RVD / VMTD | ❌ | ✅ | ✅ |
| Attack Path Simulation | ❌ | ✅ | ✅ |
| Compliance dashboards | ❌ | ✅ | ✅ |
| Custom SHA + ETD modules | ❌ | ✅ | ✅ |
| Posture management | ❌ | ❌ | ✅ |
| Multi-cloud (AWS / Azure) | ❌ | ❌ | ✅ |
| Chronicle + Gemini + SOAR | ❌ | ❌ | ✅ |
| Cost | Free | Per project/mo | Per project/mo (higher) |

---

### Built-in Security Sources

| Source | Finding Class | Detection Method | Tier | Key Categories |
|---|---|---|---|---|
| Security Health Analytics | MISCONFIGURATION | Config scanning (batch + real-time) | Standard/Premium | PUBLIC_BUCKET_ACL, OPEN_FIREWALL, KMS_KEY_NOT_ROTATED |
| Event Threat Detection | THREAT | Log stream analysis | Premium | Crypto mining, IAM anomaly, exfiltration |
| Container Threat Detection | THREAT | eBPF kernel instrumentation | Premium | Reverse shell, added binary, modified file |
| Web Security Scanner | VULNERABILITY | DAST scanning | Standard (custom) / Premium (managed) | XSS, CSRF, cleartext password |
| VM Manager | VULNERABILITY | OS agent + CVE database | Premium | CVE findings with CVSS scores |
| Rapid Vulnerability Detection | VULNERABILITY | Agentless network scan | Premium | Exposed services, default credentials |
| Virtual Machine Threat Detection | THREAT | Hypervisor memory scan | Premium | Crypto mining, rootkits |

---

### Finding Severity and SLA

| Severity | Description | Recommended SLA | Example Findings |
|---|---|---|---|
| CRITICAL | Immediate data breach risk | 24 hours | `PUBLIC_BUCKET_ACL`, leaked SA key in use, crypto mining |
| HIGH | Significant risk, exploitation likely | 7 days | `OPEN_FIREWALL`, `SQL_NO_ROOT_PASSWORD`, anomalous IAM grant |
| MEDIUM | Moderate risk, needs context | 30 days | `KMS_KEY_NOT_ROTATED`, `VPC_FLOW_LOGS_DISABLED`, SA key exists |
| LOW | Minor defense gap | 90 days | `SHIELDED_VM_DISABLED`, `DNSSEC_DISABLED` |
| INFORMATIONAL | No direct risk | Best effort | Configuration observation, SA created |

---

### IAM Roles Summary

| Role | Read Findings | Update Findings | Manage Sources | Manage Notifications | Manage Settings | Who Needs It |
|---|---|---|---|---|---|---|
| `securitycenter.admin` | ✅ | ✅ | ✅ | ✅ | ✅ | SCC platform owner |
| `securitycenter.adminViewer` | ✅ | ❌ | ❌ | ❌ | ❌ | Security team, CISO |
| `securitycenter.findingsEditor` | ✅ | ✅ | ❌ | ❌ | ❌ | Remediation teams, automation SAs |
| `securitycenter.findingsViewer` | ✅ | ❌ | ❌ | ❌ | ❌ | SOC analysts |
| `securitycenter.sourcesEditor` | ❌ | ❌ | ✅ | ❌ | ❌ | Custom source publishers |
| `securitycenter.notificationConfigEditor` | ❌ | ❌ | ❌ | ✅ | ❌ | Security automation team |
| `securitycenter.settingsEditor` | ❌ | ❌ | ❌ | ❌ | ✅ | SCC admin |

---

### SCC Notification vs. BigQuery Export

| | **Notification Config (Pub/Sub)** | **BigQuery Export** |
|---|---|---|
| **Use case** | Real-time alerting, SIEM, auto-remediation | Long-term retention, SQL analytics, dashboards |
| **Latency** | Seconds | Seconds to minutes |
| **Retention** | Pub/Sub subscription TTL (max 7 days) | Indefinite (BigQuery storage) |
| **Query capability** | None (raw JSON) | Full SQL |
| **Filtering** | CEL filter on notification config | CEL filter on export config |
| **Setup complexity** | Low | Low |
| **Best for** | Operational workflows | Historical analysis and compliance reporting |

> 💡 **Use both.** Notification configs for operational response; BigQuery export for analytics, compliance, and retention beyond SCC's 13-month window.

---

### ETD Detector Categories

| Category | MITRE ATT&CK Tactic | Log Source | Typical Severity |
|---|---|---|---|
| `INITIAL_ACCESS` | Initial Access (TA0001) | Admin Activity, Data Access | HIGH–CRITICAL |
| `PRIVILEGE_ESCALATION` | Privilege Escalation (TA0004) | Admin Activity | HIGH–CRITICAL |
| `DEFENSE_EVASION` | Defense Evasion (TA0005) | Admin Activity | MEDIUM–HIGH |
| `DISCOVERY` | Discovery (TA0007) | Admin Activity | LOW–MEDIUM |
| `LATERAL_MOVEMENT` | Lateral Movement (TA0008) | Admin Activity, VPC Flow | MEDIUM–HIGH |
| `CREDENTIAL_ACCESS` | Credential Access (TA0006) | Admin Activity, Data Access | HIGH |
| `EXFILTRATION` | Exfiltration (TA0010) | Data Access, Admin Activity | CRITICAL |
| `IMPACT` | Impact (TA0040) | Admin Activity, VMTD | HIGH–CRITICAL |

---

### Compliance Frameworks (Premium)

| Framework | Key GCP Controls | Common Failing SHA Categories |
|---|---|---|
| CIS GCP Foundations | IAM, logging, networking, storage, KMS | `AUDIT_LOGGING_DISABLED`, `VPC_FLOW_LOGS_DISABLED`, `PUBLIC_BUCKET_ACL` |
| PCI-DSS | Access control, encryption, audit logs, network | `OPEN_FIREWALL`, `SQL_NO_ROOT_PASSWORD`, `KMS_KEY_NOT_ROTATED` |
| NIST SP 800-53 | AC, AU, CM, SC, SI | `MFA_NOT_ENFORCED`, `AUDIT_LOGGING_DISABLED`, `LEGACY_NETWORK` |
| ISO 27001 | Asset mgmt, access control, cryptography | `OVER_PRIVILEGED_SERVICE_ACCOUNT`, `SERVICE_ACCOUNT_KEY_CREATED` |
| HIPAA | PHI access, audit, transmission security | `PUBLIC_BUCKET_ACL`, `SQL_NO_ROOT_PASSWORD`, `OPEN_FIREWALL` |
| FedRAMP | NIST 800-53 moderate baseline | `AUDIT_LOGGING_DISABLED`, `MFA_NOT_ENFORCED`, `OPEN_FIREWALL` |

---

*Generated: GCP Security Command Center Comprehensive Cheatsheet — covers SCC Standard / Premium / Enterprise | API v1*
