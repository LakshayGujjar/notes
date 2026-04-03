# 🔭 GCP Network Intelligence Center — Comprehensive Cheatsheet

> **Audience:** Cloud network engineers, SREs, and architects managing GCP networking.
> **Last updated:** March 2026 | Covers all 6 NIC modules.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Module 1 — Network Topology](#2-module-1--network-topology)
3. [Module 2 — Connectivity Tests](#3-module-2--connectivity-tests)
4. [Module 3 — Performance Dashboard](#4-module-3--performance-dashboard)
5. [Module 4 — Firewall Insights](#5-module-4--firewall-insights)
6. [Module 5 — Network Analyzer](#6-module-5--network-analyzer)
7. [Module 6 — Organization Policies for Networking](#7-module-6--organization-policies-for-networking)
8. [IAM Roles & Required APIs](#8-iam-roles--required-apis)
9. [gcloud CLI Quick Reference](#9-gcloud-cli-quick-reference)
10. [Integration Patterns & Automation](#10-integration-patterns--automation)
11. [Pricing Summary](#11-pricing-summary)
12. [Quick Reference & Comparison Table](#12-quick-reference--comparison-table)

---

## 1. Overview & Key Concepts

### What is Network Intelligence Center?

**Network Intelligence Center (NIC)** is GCP's unified network observability and diagnostics platform. It provides tools to **visualize**, **verify**, **monitor**, and **optimize** your cloud network — without requiring packet captures or manual trace analysis.

**Why it exists:**
- Cloud networks are complex (VPCs, peering, VPNs, Interconnect, PSC, GKE, etc.)
- Traditional debugging (ping, traceroute) doesn't work inside GCP's SDN
- Engineers need control-plane visibility and data-plane metrics in one place

### Core Value Proposition

| Pillar | Description |
|---|---|
| 🔭 **Visibility** | Real-time topology graphs of your entire network |
| ✅ **Verification** | Simulate packet paths and confirm reachability without sending traffic |
| 📊 **Monitoring** | Latency and packet loss heatmaps across GCP zones/regions |
| 🔍 **Optimization** | Firewall rule analysis, misconfiguration detection, policy enforcement |

---

### The 6 Core Modules

| Module | Purpose | Cost |
|---|---|---|
| **Network Topology** | Visual graph of VPC entities and live traffic metrics | Free |
| **Connectivity Tests** | Control-plane reachability simulation between endpoints | Per test |
| **Performance Dashboard** | Inter-zone/region latency & packet loss heatmap | Free |
| **Firewall Insights** | Firewall rule usage analysis and cleanup recommendations | Tied to logging |
| **Network Analyzer** | Automated misconfiguration and best-practice scanning | Free |
| **Organization Policies** | Network governance enforcement across org/projects | Free |

---

### Required APIs (Enable Before Use)

```bash
# Enable all Network Intelligence Center APIs at once
gcloud services enable \
  networkmanagement.googleapis.com \
  recommender.googleapis.com \
  networksecurity.googleapis.com \
  logging.googleapis.com \
  monitoring.googleapis.com \
  compute.googleapis.com \
  --project=MY_PROJECT
```

---

## 2. Module 1 — Network Topology

### 📡 What It Shows

Network Topology renders a **real-time, interactive graph** of your GCP network — visualizing entities, their relationships, and live traffic metrics between them.

**Entities visualized:**

| Category | Entities |
|---|---|
| Compute | VMs, Managed Instance Groups |
| Networking | VPCs, Subnets, VPC Peering, Shared VPC |
| Hybrid | Cloud Interconnect (VLAN attachments), Cloud VPN tunnels |
| Services | Cloud NAT, Cloud Load Balancers (L4/L7), Private Service Connect (PSC) |
| Containers | GKE clusters (as grouped entities) |

**Key metrics shown per entity link:**

| Metric | Description |
|---|---|
| **Throughput (bps)** | Bits per second flowing over the link |
| **Packets per second** | Traffic volume |
| **Packet loss %** | Measured drop rate on the path |
| **Latency (ms)** | RTT between entity pairs |

> 💡 Metrics are sampled — not every packet is measured. Throughput data has a **~1-minute lag** and is based on **Andromeda flow sampling**.

---

### Accessing Network Topology

**Console:**
```
GCP Console → Network Intelligence Center → Network Topology
```

**gcloud CLI:** Network Topology is **console-only** for the visual graph. Use Cloud Monitoring for metric queries:

```bash
# Query VPC flow metrics used by Topology via Monitoring API
gcloud monitoring time-series list \
  --filter='metric.type="networking.googleapis.com/vm_flow/egress_bytes_count"' \
  --project=MY_PROJECT \
  --interval-start-time=$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --interval-end-time=$(date -u +%Y-%m-%dT%H:%M:%SZ)
```

---

### Limitations

| Limitation | Detail |
|---|---|
| Sampling rate | Flow data is sampled, not 100% captured |
| Supported protocols | TCP and UDP only (ICMP not shown in flow metrics) |
| Entity freshness | Topology graph updates every ~5 minutes |
| No historical playback | Snapshot is always current state |
| Subnet-level only | Individual VM IPs not always visible for large subnets |
| VPC limit | Very large VPCs (1000s of VMs) may render slowly |

> ⚠️ Network Topology does **not** show traffic contents — only metadata (bytes, packets, direction).

---

## 3. Module 2 — Connectivity Tests

### ✅ What It Does

Connectivity Tests simulate the **control-plane packet path** between two endpoints and report whether traffic would be `REACHABLE` or `UNREACHABLE` — and **exactly why**, at each hop.

> 💡 This is **not** live traffic injection — it's a **logical simulation** of GCP's routing and firewall rules at the moment the test runs. No packets traverse the network.

---

### Supported Endpoint Types

| Type | Source | Destination |
|---|---|---|
| GCE VM instance | ✅ | ✅ |
| IP address (internal/external) | ✅ | ✅ |
| Cloud SQL instance | ❌ | ✅ |
| GKE cluster/node/pod | ✅ | ✅ |
| App Engine (Flex) | ✅ | ✅ |
| Cloud Run | ✅ | ✅ |
| Forwarding rule (Load Balancer) | ❌ | ✅ |
| Cloud VPN tunnel | ✅ | ✅ |
| PSC endpoint | ✅ | ✅ |

---

### Test Result States

| State | Meaning |
|---|---|
| ✅ `REACHABLE` | Simulation confirms traffic can reach the destination |
| ❌ `UNREACHABLE` | A firewall rule, route, or config blocks the traffic |
| ⚠️ `AMBIGUOUS` | Multiple paths exist with conflicting outcomes |
| 🔍 `UNDETERMINED` | Insufficient information to simulate (e.g., third-party appliance) |

---

### gcloud CLI Examples

```bash
# ── CREATE a connectivity test (VM-to-VM, TCP port 22) ────────────────
gcloud network-management connectivity-tests create SSH_TEST \
  --source-instance=projects/MY_PROJECT/zones/us-central1-a/instances/SOURCE_VM \
  --destination-instance=projects/MY_PROJECT/zones/us-central1-b/instances/DEST_VM \
  --protocol=TCP \
  --destination-port=22 \
  --description="Test SSH reachability between VMs" \
  --project=MY_PROJECT

# ── CREATE a test using IP addresses (external to internal) ───────────
gcloud network-management connectivity-tests create EXT_TO_VM \
  --source-ip-address=203.0.113.10 \
  --destination-instance=projects/MY_PROJECT/zones/us-central1-a/instances/WEB_VM \
  --protocol=TCP \
  --destination-port=443 \
  --project=MY_PROJECT

# ── CREATE a test to Cloud SQL ────────────────────────────────────────
gcloud network-management connectivity-tests create VM_TO_SQL \
  --source-instance=projects/MY_PROJECT/zones/us-central1-a/instances/APP_VM \
  --destination-cloud-sql-instance=projects/MY_PROJECT/instances/MY_SQL_INSTANCE \
  --protocol=TCP \
  --destination-port=3306 \
  --project=MY_PROJECT

# ── LIST all connectivity tests ───────────────────────────────────────
gcloud network-management connectivity-tests list \
  --project=MY_PROJECT \
  --format="table(name,reachabilityDetails.result,createTime)"

# ── DESCRIBE a test (includes full trace) ────────────────────────────
gcloud network-management connectivity-tests describe SSH_TEST \
  --project=MY_PROJECT \
  --format=json

# ── RE-RUN an existing test (re-evaluates current config) ─────────────
gcloud network-management connectivity-tests rerun SSH_TEST \
  --project=MY_PROJECT

# ── DELETE a connectivity test ────────────────────────────────────────
gcloud network-management connectivity-tests delete SSH_TEST \
  --project=MY_PROJECT
```

---

### Interpreting Trace Results

A test result contains a **trace** — a list of steps showing the simulated packet path:

```json
{
  "steps": [
    { "state": "START_FROM_INSTANCE",     "description": "Starting at SOURCE_VM" },
    { "state": "APPLY_EGRESS_FIREWALL",   "description": "Egress firewall: ALLOWED by allow-internal" },
    { "state": "ROUTE",                   "description": "Routed via subnet route 10.0.0.0/24" },
    { "state": "APPLY_INGRESS_FIREWALL",  "description": "Ingress firewall: DENIED by default-deny-all" },
    { "state": "DROP",                    "description": "Packet dropped — no matching allow rule" }
  ],
  "deliveryStep": {
    "action": "DROP",
    "cause": "DENIED_BY_INGRESS_FIREWALL"
  }
}
```

**Common drop causes:**

| Cause | Meaning | Fix |
|---|---|---|
| `DENIED_BY_INGRESS_FIREWALL` | Ingress rule blocks inbound traffic | Add allow rule on destination |
| `DENIED_BY_EGRESS_FIREWALL` | Egress rule blocks outbound traffic | Add egress allow rule on source |
| `NO_ROUTE` | No matching route in the routing table | Add a subnet or custom route |
| `WRONG_NEXT_HOP` | Route exists but next-hop is wrong/down | Fix next-hop (VPN/Interconnect) |
| `VPN_TUNNEL_NOT_ESTABLISHED` | VPN tunnel is down | Check tunnel status |

---

### Common Use Cases

- 🔍 **Debug firewall rules** — Find exactly which rule is blocking traffic
- 🔍 **Validate VPN reachability** — Confirm on-prem → GCP path is functional
- 🔍 **Test PSC connectivity** — Verify Private Service Connect endpoint is routable
- 🔍 **Post-deployment validation** — Run in CI/CD to catch network regressions
- 🔍 **Cross-project Shared VPC** — Check subnet reachability across service projects

---

## 4. Module 3 — Performance Dashboard

### 📊 What It Measures

The Performance Dashboard shows **active probe-based** measurements of:
- **Round-trip latency** (p50, p95, p99 in milliseconds)
- **Packet loss percentage**

Between **GCP zones** (intra-region) and **GCP regions** (inter-region), using Google's own infrastructure probes.

> 💡 Data comes from **active synthetic probing** between GCP's infrastructure nodes — not from your VMs. This reflects Google's backbone performance, not your application traffic.

---

### Reading the Heatmap

```
              ┌─────────────────────────────────────────────────────┐
              │              DESTINATION ZONE                        │
              │  us-c1-a   us-c1-b   us-c1-c   us-e1-b   europe-w1 │
SOURCE  ──────┼──────────────────────────────────────────────────────┤
us-c1-a       │    —       0.4ms     0.5ms     33ms       100ms     │
us-c1-b       │  0.4ms       —       0.3ms     34ms       101ms     │
us-e1-b       │  33ms      34ms      35ms        —         85ms     │
europe-w1     │  100ms    101ms      99ms      85ms          —      │
              └─────────────────────────────────────────────────────┘
```

- **Green cells** → Low latency / no loss (healthy)
- **Yellow cells** → Elevated latency or slight loss (watch)
- **Red cells** → High latency or significant loss (investigate)

---

### Available Metrics

| Metric | Description | Typical Values |
|---|---|---|
| `RTT p50` | Median round-trip latency | Intra-zone: <1ms, Cross-region: 30–200ms |
| `RTT p95` | 95th percentile RTT | Should be close to p50 for stable paths |
| `RTT p99` | 99th percentile RTT | Spikes here indicate intermittent congestion |
| `Packet loss %` | % of probes dropped | Should be 0%; >0.1% warrants investigation |

---

### gcloud CLI Examples

```bash
# ── List available latency data (inter-region) ────────────────────────
gcloud beta network-management location-based-service-quality list \
  --project=MY_PROJECT

# ── Query inter-zone latency metrics via Cloud Monitoring ─────────────
gcloud monitoring time-series list \
  --filter='metric.type="networkmanagement.googleapis.com/node_to_node/rtt_latencies"' \
  --project=MY_PROJECT \
  --interval-start-time=$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --interval-end-time=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --format="table(metric.labels.source_location,metric.labels.destination_location,points[0].value.distributionValue.mean)"

# ── Query packet loss metrics ─────────────────────────────────────────
gcloud monitoring time-series list \
  --filter='metric.type="networkmanagement.googleapis.com/node_to_node/packet_loss_rate"' \
  --project=MY_PROJECT \
  --interval-start-time=$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --interval-end-time=$(date -u +%Y-%m-%dT%H:%M:%SZ)
```

> 💡 The Performance Dashboard is **console-first**. For programmatic access, query metrics via the Cloud Monitoring API using the `networkmanagement.googleapis.com/node_to_node/*` metric prefix.

---

### Use Cases

| Scenario | How to Use |
|---|---|
| **Choose optimal region** | Compare p99 latency for your app's primary user regions |
| **Diagnose inter-region degradation** | Look for sudden latency spikes or packet loss in heatmap |
| **SLA validation** | Confirm baseline latency is within acceptable bounds |
| **Capacity planning** | Identify zones with consistently higher RTT before deploying |

---

## 5. Module 4 — Firewall Insights

### 🔍 What It Analyzes

Firewall Insights uses **Firewall Rules Logging data** to surface:
- Which rules are **actively used** (hit counts)
- Which rules are **shadowed** (never matched)
- Which rules are **overly permissive** (allow all / broad ranges)
- Which allow rules have **zero hits** (cleanup candidates)

> ⚠️ **Prerequisite:** [Firewall Rules Logging](https://cloud.google.com/vpc/docs/firewall-rules-logging) must be enabled on the rules you want to analyze. Logging incurs Cloud Logging ingestion costs.

```bash
# Enable logging on an existing firewall rule
gcloud compute firewall-rules update MY_RULE \
  --enable-logging \
  --project=MY_PROJECT
```

---

### Insight Types

| Insight Type | ID | Description |
|---|---|---|
| 🔴 **Shadowed Rules** | `SHADOWED_RULE` | Rule is never matched — a higher-priority rule always fires first |
| 🟡 **Overly Permissive** | `OVERLY_PERMISSIVE_RULE` | Rule allows `0.0.0.0/0` or very broad port ranges with no hits |
| 🟢 **Deny Rules with Hits** | `DENY_RULE_WITH_HITS` | Deny rule is actively blocking traffic — useful for security audits |
| ⚪ **Allow Rules with No Hits** | `ALLOW_RULE_WITH_NO_HITS` | Allow rule never matched in observation window — safe to review for deletion |

---

### gcloud CLI Examples

```bash
# ── List ALL Firewall Insights for a project ──────────────────────────
gcloud recommender insights list \
  --project=MY_PROJECT \
  --location=global \
  --insight-type=google.compute.firewall.Insight \
  --format="table(name,insightSubtype,stateInfo.state,description)"

# ── Filter: only SHADOWED_RULE insights ───────────────────────────────
gcloud recommender insights list \
  --project=MY_PROJECT \
  --location=global \
  --insight-type=google.compute.firewall.Insight \
  --filter="insightSubtype=SHADOWED_RULE" \
  --format="table(name,insightSubtype,content.shadowingFirewalls)"

# ── Filter: only ALLOW rules with no hits ─────────────────────────────
gcloud recommender insights list \
  --project=MY_PROJECT \
  --location=global \
  --insight-type=google.compute.firewall.Insight \
  --filter="insightSubtype=ALLOW_RULE_WITH_NO_HITS"

# ── Describe a specific insight (full detail) ─────────────────────────
gcloud recommender insights describe INSIGHT_ID \
  --project=MY_PROJECT \
  --location=global \
  --insight-type=google.compute.firewall.Insight \
  --format=json

# ── List Recommender RECOMMENDATIONS for firewall cleanup ─────────────
gcloud recommender recommendations list \
  --project=MY_PROJECT \
  --location=global \
  --recommender=google.compute.firewall.Recommender \
  --format="table(name,recommenderSubtype,stateInfo.state,description)"

# ── Mark a recommendation as CLAIMED (in-progress) ───────────────────
gcloud recommender recommendations mark-claimed RECOMMENDATION_ID \
  --project=MY_PROJECT \
  --location=global \
  --recommender=google.compute.firewall.Recommender \
  --etag=ETAG_FROM_DESCRIBE

# ── Mark a recommendation as SUCCEEDED (after applying fix) ──────────
gcloud recommender recommendations mark-succeeded RECOMMENDATION_ID \
  --project=MY_PROJECT \
  --location=global \
  --recommender=google.compute.firewall.Recommender \
  --etag=ETAG_FROM_DESCRIBE
```

---

### Recommender API Integration

Firewall Insights is tightly integrated with the **Recommender API**, which generates actionable recommendations:

| Recommendation | Action |
|---|---|
| Delete shadowed rule | Remove redundant rule that is never evaluated |
| Restrict IP ranges | Narrow `0.0.0.0/0` to specific source ranges |
| Delete unused allow rule | Remove rule with no hits in the past 30 days |
| Restrict ports | Narrow broad port ranges to only required ports |

> 💡 The observation window for "no hits" analysis defaults to **30 days**. Newly created rules need this window to accumulate data before appearing in insights.

---

## 6. Module 5 — Network Analyzer

### 🔍 What It Does

Network Analyzer performs **automated, continuous, read-only scanning** of your GCP network configuration. It detects misconfigurations, potential connectivity failures, and deviations from best practices — **without any traffic being sent**.

Findings are regenerated approximately every **10 minutes**.

---

### Finding Categories

| Category | Description |
|---|---|
| **Connectivity** | Configs that break or impair reachability |
| **Security** | Configs that expose resources or violate security posture |
| **Best Practices** | Deviations from GCP recommended configurations |
| **Configuration** | Invalid or conflicting settings |

---

### Key Finding Types

| Finding | Severity | Description |
|---|---|---|
| `MISCONFIGURED_CLOUD_NAT` | 🔴 CRITICAL | NAT gateway has no valid address or subnet mapping |
| `BROKEN_VPC_PEERING` | 🔴 CRITICAL | Peering connection exists but routes are not exchanged |
| `SUBOPTIMAL_BGP_ROUTE` | 🟡 WARNING | BGP routes not propagated correctly via Cloud Router |
| `LB_HEALTH_CHECK_FIREWALL_MISSING` | 🔴 CRITICAL | Load balancer health checks blocked by firewall rules |
| `GKE_CLUSTER_CONNECTIVITY_ISSUE` | 🔴 CRITICAL | GKE master cannot reach nodes or vice versa |
| `SUBOPTIMAL_CLOUD_NAT_CONFIG` | 🟡 WARNING | NAT configured but no VMs benefit from it |
| `SHADOW_DENY_ROUTE` | 🟡 WARNING | A deny route shadows legitimate traffic paths |
| `UNUSED_RESERVED_ADDRESS` | ℹ️ INFO | Static IP reserved but not attached |
| `PRIVATE_GOOGLE_ACCESS_DISABLED` | 🟡 WARNING | Subnet has no access to Google APIs privately |

---

### Severity Levels

| Severity | Icon | Meaning |
|---|---|---|
| `CRITICAL` | 🔴 | Active connectivity or security failure — fix immediately |
| `WARNING` | 🟡 | Degraded or at-risk configuration |
| `INFO` | ℹ️ | Optimization opportunity or informational note |

---

### gcloud CLI Examples

```bash
# ── List ALL Network Analyzer reports ────────────────────────────────
gcloud network-management reports list \
  --project=MY_PROJECT \
  --format="table(name,createTime)"

# ── Describe a specific report (contains all findings) ────────────────
gcloud network-management reports describe REPORT_NAME \
  --project=MY_PROJECT \
  --format=json

# ── List findings filtered by severity (using --filter) ───────────────
gcloud network-management reports list \
  --project=MY_PROJECT \
  --format=json | jq '.[] | .findings[] | select(.severity=="CRITICAL")'

# ── Access Analyzer via Recommender API (findings as insights) ─────────
gcloud recommender insights list \
  --project=MY_PROJECT \
  --location=global \
  --insight-type=google.networkanalyzer.networkservices.loadbalancers.Insight \
  --format="table(name,insightSubtype,stateInfo.state)"

# ── Other Network Analyzer insight types ──────────────────────────────
# google.networkanalyzer.vpcnetwork.connectivityInsight
# google.networkanalyzer.hybridconnectivity.Insight
# google.networkanalyzer.gke.networkInsight
```

> 💡 Network Analyzer findings are also surfaced in the **Security Command Center** (SCC) for organizations using SCC Standard or Premium tier.

---

### Findings → Recommendations Mapping

| Finding | Linked Recommendation |
|---|---|
| `LB_HEALTH_CHECK_FIREWALL_MISSING` | Create firewall rule allowing `35.191.0.0/16`, `130.211.0.0/22` |
| `BROKEN_VPC_PEERING` | Re-establish peering or fix route export/import settings |
| `MISCONFIGURED_CLOUD_NAT` | Attach valid external IP or adjust subnet mapping |
| `GKE_CLUSTER_CONNECTIVITY_ISSUE` | Add firewall rules for master CIDR to node port ranges |
| `PRIVATE_GOOGLE_ACCESS_DISABLED` | Enable Private Google Access on the affected subnet |

---

## 7. Module 6 — Organization Policies for Networking

### 🏛️ Role of Org Policies

Organization Policies enforce **guardrails** on GCP resource configurations across an entire organization, folder, or project — preventing non-compliant network resources from being created.

They complement IAM (which controls *who* can act) by controlling *what* configurations are allowed.

---

### Key Networking Org Policy Constraints

| Constraint | Type | Description |
|---|---|---|
| `compute.vmExternalIpAccess` | List | Restricts which VMs can have external (public) IPs |
| `compute.restrictCloudNATUsage` | List | Limits which VPCs can create Cloud NAT gateways |
| `compute.restrictVpnPeerIPs` | List | Whitelist of allowed VPN peer IP addresses |
| `compute.restrictSharedVpcHostProjects` | List | Limits which projects can be Shared VPC hosts |
| `compute.restrictSharedVpcSubnetworks` | List | Limits which subnets can be shared via Shared VPC |
| `compute.restrictPartnerInterconnectUsage` | Boolean | Block/allow Partner Interconnect usage |
| `compute.skipDefaultNetworkCreation` | Boolean | Prevent default VPC creation in new projects |
| `compute.restrictCloudInterconnectUsage` | List | Restrict Interconnect to specific organizations |
| `compute.requireVpcFlowLogs` | List | Enforce VPC Flow Logs on subnets |

---

### gcloud CLI Examples

```bash
# ── LIST all org policies set on a project ────────────────────────────
gcloud resource-manager org-policies list \
  --project=MY_PROJECT

# ── LIST all policies set at org level ────────────────────────────────
gcloud resource-manager org-policies list \
  --organization=MY_ORG_ID

# ── DESCRIBE a specific org policy ───────────────────────────────────
gcloud resource-manager org-policies describe \
  compute.vmExternalIpAccess \
  --project=MY_PROJECT

# ── SET policy: deny all external IPs in a project ────────────────────
gcloud resource-manager org-policies deny \
  compute.vmExternalIpAccess \
  --project=MY_PROJECT \
  allValues    # denies all values (no VMs can have external IPs)

# ── SET policy: allow only specific VPN peer IPs org-wide ─────────────
cat > /tmp/vpn-peer-policy.yaml << 'EOF'
constraint: constraints/compute.restrictVpnPeerIPs
listPolicy:
  allowedValues:
    - "203.0.113.1"
    - "203.0.113.2"
EOF

gcloud resource-manager org-policies set-policy /tmp/vpn-peer-policy.yaml \
  --organization=MY_ORG_ID

# ── RESET a policy to inherit from parent ─────────────────────────────
gcloud resource-manager org-policies reset \
  compute.vmExternalIpAccess \
  --project=MY_PROJECT

# ── AUDIT: Find all projects violating vmExternalIpAccess ─────────────
gcloud asset search-all-resources \
  --organization=MY_ORG_ID \
  --asset-types="compute.googleapis.com/Instance" \
  --query="networkInterfaces:accessConfigs.natIP:*" \
  --format="table(name,networkInterfaces.accessConfigs.natIP)"
```

---

### Auditing Policy Violations

```bash
# Use Cloud Asset Inventory to find resources violating policies
# Find all VMs with external IPs across the org
gcloud asset search-all-resources \
  --organization=MY_ORG_ID \
  --asset-types="compute.googleapis.com/Instance" \
  --query='networkInterfaces.accessConfigs.type="ONE_TO_ONE_NAT"' \
  --format="table(name,project,location)"

# Check effective policy for a specific resource
gcloud resource-manager org-policies describe \
  compute.vmExternalIpAccess \
  --effective \
  --project=MY_PROJECT
```

> 💡 Use **`--effective`** flag to see the final resolved policy after inheritance from org → folder → project hierarchy.

---

## 8. IAM Roles & Required APIs

### IAM Roles Reference

| Role | Level | Permissions |
|---|---|---|
| `roles/networkmanagement.admin` | Project | Full access to Connectivity Tests, Network Analyzer, Topology |
| `roles/networkmanagement.viewer` | Project | Read-only access to all NIC modules |
| `roles/recommender.computeFirewallAdmin` | Project | View + update Firewall Insights recommendations |
| `roles/recommender.computeFirewallViewer` | Project | View-only Firewall Insights |
| `roles/recommender.viewer` | Project | View any recommender insight (read-only) |
| `roles/compute.networkAdmin` | Project | Manage VPC, firewall rules, routes (needed for remediation) |
| `roles/compute.networkViewer` | Project | Read-only view of network resources |
| `roles/compute.securityAdmin` | Project | Manage firewall rules and SSL certs |
| `roles/orgpolicy.policyAdmin` | Org/Folder | Create and modify org policies |
| `roles/orgpolicy.policyViewer` | Org/Folder | View org policies |
| `roles/cloudasset.viewer` | Org/Project | Asset inventory search (for auditing) |

---

### Least Privilege Recommendations Per Module

| Module | Minimum Role(s) Needed |
|---|---|
| Network Topology (view) | `roles/networkmanagement.viewer` |
| Connectivity Tests (create/run) | `roles/networkmanagement.admin` |
| Connectivity Tests (view only) | `roles/networkmanagement.viewer` |
| Firewall Insights (view) | `roles/recommender.computeFirewallViewer` |
| Firewall Insights (apply fixes) | `roles/recommender.computeFirewallAdmin` + `roles/compute.securityAdmin` |
| Network Analyzer (view) | `roles/networkmanagement.viewer` |
| Org Policies (view) | `roles/orgpolicy.policyViewer` |
| Org Policies (set) | `roles/orgpolicy.policyAdmin` |

---

### Assign IAM Roles via gcloud

```bash
# Grant Connectivity Tests admin to a user
gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="user:engineer@example.com" \
  --role="roles/networkmanagement.admin"

# Grant Firewall Insights viewer to a service account (CI/CD)
gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="serviceAccount:cicd-sa@MY_PROJECT.iam.gserviceaccount.com" \
  --role="roles/recommender.computeFirewallViewer"

# Grant org policy admin at folder level
gcloud resource-manager folders add-iam-policy-binding FOLDER_ID \
  --member="user:netadmin@example.com" \
  --role="roles/orgpolicy.policyAdmin"

# Enable all required APIs
gcloud services enable \
  networkmanagement.googleapis.com \
  recommender.googleapis.com \
  cloudasset.googleapis.com \
  logging.googleapis.com \
  monitoring.googleapis.com \
  --project=MY_PROJECT
```

---

## 9. gcloud CLI Quick Reference

### Module 2 — Connectivity Tests

```bash
# Create test
gcloud network-management connectivity-tests create TEST_NAME \
  --source-instance=projects/P/zones/Z/instances/SRC \
  --destination-instance=projects/P/zones/Z/instances/DST \
  --protocol=TCP --destination-port=PORT --project=MY_PROJECT

# List tests (table format)
gcloud network-management connectivity-tests list --project=MY_PROJECT \
  --format="table(name,reachabilityDetails.result,updateTime)"

# Describe (JSON for trace parsing)
gcloud network-management connectivity-tests describe TEST_NAME \
  --project=MY_PROJECT --format=json

# Re-run
gcloud network-management connectivity-tests rerun TEST_NAME \
  --project=MY_PROJECT

# Delete
gcloud network-management connectivity-tests delete TEST_NAME \
  --project=MY_PROJECT --quiet
```

### Module 4 — Firewall Insights

```bash
# List all firewall insights
gcloud recommender insights list \
  --project=MY_PROJECT --location=global \
  --insight-type=google.compute.firewall.Insight \
  --format="table(name,insightSubtype,stateInfo.state)"

# Filter by subtype
gcloud recommender insights list \
  --project=MY_PROJECT --location=global \
  --insight-type=google.compute.firewall.Insight \
  --filter="insightSubtype=SHADOWED_RULE"

# List firewall recommendations
gcloud recommender recommendations list \
  --project=MY_PROJECT --location=global \
  --recommender=google.compute.firewall.Recommender

# Mark recommendation claimed
gcloud recommender recommendations mark-claimed REC_ID \
  --project=MY_PROJECT --location=global \
  --recommender=google.compute.firewall.Recommender --etag=ETAG
```

### Module 5 — Network Analyzer

```bash
# List analyzer reports
gcloud network-management reports list --project=MY_PROJECT

# Describe a report
gcloud network-management reports describe REPORT_NAME \
  --project=MY_PROJECT --format=json

# List GKE network insights
gcloud recommender insights list \
  --project=MY_PROJECT --location=global \
  --insight-type=google.networkanalyzer.gke.networkInsight
```

### Org Policies

```bash
# List all constraints available
gcloud org-policies list-constraints --organization=MY_ORG_ID

# List policies set on project
gcloud resource-manager org-policies list --project=MY_PROJECT

# Get effective policy
gcloud resource-manager org-policies describe CONSTRAINT \
  --effective --project=MY_PROJECT

# Deny a constraint
gcloud resource-manager org-policies deny CONSTRAINT \
  --project=MY_PROJECT allValues
```

### Formatting Tips

```bash
# Output as JSON (best for scripting/jq)
--format=json

# Custom table output
--format="table(name,status,createTime.date('%Y-%m-%d'))"

# Filter results
--filter="status=ACTIVE AND createTime>'2026-01-01'"

# Flatten nested fields
--format="table[box](name,reachabilityDetails.result:label=RESULT)"

# YAML output
--format=yaml

# Limit results
--limit=20 --sort-by=createTime
```

---

## 10. Integration Patterns & Automation

### CI/CD — Post-Terraform Connectivity Validation

Run Connectivity Tests automatically after infrastructure changes to catch network regressions:

```bash
#!/bin/bash
# post-deploy-test.sh — Run after terraform apply

PROJECT="my-gcp-project"
TEST_NAME="post-deploy-web-to-db-$(date +%s)"

# Create and run test
gcloud network-management connectivity-tests create "$TEST_NAME" \
  --source-instance="projects/$PROJECT/zones/us-central1-a/instances/web-server" \
  --destination-instance="projects/$PROJECT/zones/us-central1-a/instances/db-server" \
  --protocol=TCP \
  --destination-port=5432 \
  --project="$PROJECT"

# Poll for result (wait up to 60s)
for i in {1..12}; do
  sleep 5
  RESULT=$(gcloud network-management connectivity-tests describe "$TEST_NAME" \
    --project="$PROJECT" \
    --format="value(reachabilityDetails.result)")

  if [ "$RESULT" = "REACHABLE" ]; then
    echo "✅ Connectivity test PASSED: $RESULT"
    gcloud network-management connectivity-tests delete "$TEST_NAME" \
      --project="$PROJECT" --quiet
    exit 0
  elif [ "$RESULT" = "UNREACHABLE" ]; then
    echo "❌ Connectivity test FAILED: $RESULT"
    gcloud network-management connectivity-tests delete "$TEST_NAME" \
      --project="$PROJECT" --quiet
    exit 1
  fi
  echo "⏳ Waiting for result... ($i/12)"
done

echo "⚠️ Test timed out"
exit 2
```

---

### Python — Network Management API Client

```python
"""Run a connectivity test via the Python client library."""

from google.cloud import network_management_v1
from google.cloud.network_management_v1.types import ConnectivityTest, Endpoint
import time

PROJECT = "my-gcp-project"

def run_connectivity_test(project: str, src_instance: str, dst_instance: str, port: int) -> str:
    client = network_management_v1.ReachabilityServiceClient()

    test = ConnectivityTest(
        name=f"projects/{project}/locations/global/connectivityTests/auto-test",
        source=Endpoint(instance=src_instance),
        destination=Endpoint(instance=dst_instance, port=port),
        protocol="TCP",
    )

    parent = f"projects/{project}/locations/global"

    # Create test (returns a long-running operation)
    operation = client.create_connectivity_test(
        parent=parent,
        connectivity_test_id="auto-test",
        resource=test,
    )

    # Wait for operation to complete
    result = operation.result(timeout=60)
    reachability = result.reachability_details.result.name
    return reachability


if __name__ == "__main__":
    status = run_connectivity_test(
        project=PROJECT,
        src_instance=f"projects/{PROJECT}/zones/us-central1-a/instances/web-vm",
        dst_instance=f"projects/{PROJECT}/zones/us-central1-a/instances/db-vm",
        port=5432,
    )
    print(f"Result: {status}")
    # Output: "REACHABLE" or "UNREACHABLE"
```

---

### Automating Firewall Insights Remediation

```python
"""List and auto-apply ALLOW_RULE_WITH_NO_HITS recommendations."""

from google.cloud import recommender_v1

PROJECT = "my-gcp-project"
LOCATION = "global"
RECOMMENDER = "google.compute.firewall.Recommender"

def list_firewall_recommendations(project: str) -> list:
    client = recommender_v1.RecommenderClient()
    parent = client.recommender_path(project, LOCATION, RECOMMENDER)

    recommendations = list(client.list_recommendations(parent=parent))
    return [r for r in recommendations if r.state_info.state.name == "ACTIVE"]


def mark_recommendation_claimed(client, rec_name: str, etag: str):
    request = recommender_v1.MarkRecommendationClaimedRequest(
        name=rec_name,
        etag=etag,
    )
    return client.mark_recommendation_claimed(request=request)


if __name__ == "__main__":
    client = recommender_v1.RecommenderClient()
    recs = list_firewall_recommendations(PROJECT)

    for rec in recs:
        if "DELETE_FIREWALL_RULE" in rec.recommender_subtype:
            print(f"🔍 Found cleanup candidate: {rec.name}")
            print(f"   Description: {rec.description}")
            # In production: apply the recommended operation via Compute API
            # Here we just mark as claimed (in-review)
            mark_recommendation_claimed(client, rec.name, rec.etag)
            print(f"   ✅ Marked as CLAIMED for review")
```

---

### Event-Driven Connectivity Testing with Cloud Functions + Pub/Sub

```python
"""Cloud Function triggered by Pub/Sub to run connectivity tests on VM creation."""

import functions_framework
import base64
import json
from google.cloud import network_management_v1

@functions_framework.cloud_event
def run_test_on_vm_create(cloud_event):
    """Triggered by Pub/Sub message from Cloud Audit Logs when a VM is created."""
    
    data = base64.b64decode(cloud_event.data["message"]["data"]).decode()
    event = json.loads(data)

    # Extract new VM details from audit log event
    resource = event.get("resource", {})
    project = resource.get("labels", {}).get("project_id")
    zone = resource.get("labels", {}).get("zone")
    instance_name = event.get("protoPayload", {}).get("resourceName", "").split("/")[-1]

    if not all([project, zone, instance_name]):
        return

    print(f"🔍 Running connectivity test for new VM: {instance_name}")

    client = network_management_v1.ReachabilityServiceClient()
    parent = f"projects/{project}/locations/global"

    test = network_management_v1.ConnectivityTest(
        source=network_management_v1.Endpoint(
            instance=f"projects/{project}/zones/{zone}/instances/{instance_name}"
        ),
        destination=network_management_v1.Endpoint(
            ip_address="8.8.8.8",
            port=443,
        ),
        protocol="TCP",
    )

    operation = client.create_connectivity_test(
        parent=parent,
        connectivity_test_id=f"auto-{instance_name[:20]}",
        resource=test,
    )
    print(f"✅ Test created. Operation: {operation.operation.name}")
```

**Pub/Sub trigger setup:**

```bash
# Create Pub/Sub topic for VM creation events
gcloud pubsub topics create vm-creation-events --project=MY_PROJECT

# Create Log Sink → Pub/Sub for VM insert audit logs
gcloud logging sinks create vm-create-sink \
  pubsub.googleapis.com/projects/MY_PROJECT/topics/vm-creation-events \
  --log-filter='protoPayload.methodName="v1.compute.instances.insert"
    AND protoPayload.response.status="DONE"' \
  --project=MY_PROJECT

# Deploy the Cloud Function
gcloud functions deploy connectivity-test-on-vm-create \
  --gen2 \
  --runtime=python311 \
  --trigger-topic=vm-creation-events \
  --entry-point=run_test_on_vm_create \
  --region=us-central1 \
  --project=MY_PROJECT
```

---

### Export Network Topology Metrics to Cloud Monitoring

```bash
# Create an alerting policy for inter-zone packet loss > 0.1%
cat > /tmp/packet-loss-alert.yaml << 'EOF'
displayName: "Inter-zone Packet Loss Alert"
conditions:
  - displayName: "Packet loss > 0.1%"
    conditionThreshold:
      filter: 'metric.type="networkmanagement.googleapis.com/node_to_node/packet_loss_rate"'
      comparison: COMPARISON_GT
      thresholdValue: 0.001
      duration: 300s
      aggregations:
        - alignmentPeriod: 60s
          perSeriesAligner: ALIGN_MEAN
alertStrategy:
  autoClose: 1800s
combiner: OR
EOF

gcloud alpha monitoring policies create \
  --policy-from-file=/tmp/packet-loss-alert.yaml \
  --project=MY_PROJECT
```

---

## 11. Pricing Summary

> ⚠️ Prices are approximate as of early 2026. Always verify at [cloud.google.com/network-intelligence-center/pricing](https://cloud.google.com/network-intelligence-center/pricing).

### Per-Module Breakdown

| Module | Pricing Model | Cost |
|---|---|---|
| **Network Topology** | ✅ **Free** | No charge |
| **Connectivity Tests** | Per test run | ~**$0.15 per test** |
| **Performance Dashboard** | ✅ **Free** | No charge |
| **Firewall Insights** | Tied to Firewall Rules Logging | See below |
| **Network Analyzer** | ✅ **Free** | No charge |
| **Org Policies** | ✅ **Free** | No charge |

---

### Connectivity Tests Pricing

| Item | Cost |
|---|---|
| Create a test (no run) | Free |
| Test run (each execution) | ~$0.15 |
| First N tests/month (free tier) | ❌ No free tier |

```bash
# Each "rerun" counts as a new billable test run
gcloud network-management connectivity-tests rerun TEST_NAME  # ~$0.15
```

---

### Firewall Insights Pricing

Firewall Insights itself is **free**, but it depends on **Firewall Rules Logging**, which incurs costs:

| Component | Cost |
|---|---|
| Firewall Rules Logging | ~$0.50 per GB ingested into Cloud Logging |
| Firewall Insights analysis | **Free** |
| Recommender API calls | **Free** |

> 💡 Enable Firewall Rules Logging **selectively** (only on rules you want to analyze) to control logging costs. Logging all rules in a large VPC can generate significant log volume.

---

### Monthly Cost Estimate Example

```
Scenario: 50 connectivity tests/day, firewall logging on 20 rules (5GB/month)

Connectivity Tests : 50 × 30 × $0.15  = $225/month
Firewall Logging   : 5 GB × $0.50     = $2.50/month
Other modules      :                    $0.00/month
─────────────────────────────────────────────────────
TOTAL              :                  ~$227.50/month
```

---

## 12. Quick Reference & Comparison Table

### All 6 Modules Side-by-Side

| Module | Purpose | Data Source | Cost | Requires Logging | Key Use Case |
|---|---|---|---|---|---|
| **Network Topology** | Visualize VPC graph + live metrics | Flow sampling (Andromeda) | Free | No | Topology discovery, traffic visualization |
| **Connectivity Tests** | Simulate packet path + verify reachability | Control-plane analysis | Per test (~$0.15) | No | Debug firewall/routing, CI/CD validation |
| **Performance Dashboard** | Latency & packet loss heatmap | Active GCP probing | Free | No | Region placement, SLA monitoring |
| **Firewall Insights** | Analyze rule usage, find waste | Firewall Rules Logs | Free (logging costs) | ✅ Yes | Firewall hygiene, rule cleanup |
| **Network Analyzer** | Auto-detect misconfigurations | Config analysis | Free | No | Proactive health checks, security posture |
| **Org Policies** | Enforce network governance constraints | Policy engine | Free | No | Compliance, prevent external IPs/VPN |

---

### Troubleshooting Scenario → Right Module

| Scenario | Best Module(s) |
|---|---|
| "Why can't VM A reach VM B?" | **Connectivity Tests** |
| "Which firewall rule is blocking port 443?" | **Connectivity Tests** → trace, then **Firewall Insights** |
| "Are any firewall rules unused/redundant?" | **Firewall Insights** |
| "What does my VPC topology look like?" | **Network Topology** |
| "Is there high latency between us-east1 and europe-west1?" | **Performance Dashboard** |
| "Why is my load balancer health check failing?" | **Network Analyzer** → `LB_HEALTH_CHECK_FIREWALL_MISSING` |
| "Is my Cloud NAT configured correctly?" | **Network Analyzer** |
| "Is VPC peering working between Project A and B?" | **Connectivity Tests** + **Network Analyzer** |
| "Are any VMs in my org using external IPs they shouldn't?" | **Org Policies** + Asset Inventory |
| "Why did a BGP route stop propagating?" | **Network Analyzer** |
| "Which region has the lowest latency to us-central1?" | **Performance Dashboard** |
| "Is my GKE master reachable from worker nodes?" | **Network Analyzer** |

---

### Connectivity Test Result → Next Action

| Result | Next Action |
|---|---|
| `REACHABLE` | ✅ No action needed — path is clear |
| `UNREACHABLE` + firewall drop | Add or modify firewall allow rule |
| `UNREACHABLE` + no route | Add subnet route or custom route |
| `UNREACHABLE` + VPN tunnel down | Check VPN tunnel status, peer device |
| `UNREACHABLE` + wrong next-hop | Fix route next-hop (Interconnect/VPN) |
| `AMBIGUOUS` | Review multiple conflicting paths; use Network Topology |
| `UNDETERMINED` | Manual investigation required (third-party appliance in path) |

---

*Generated for GCP Network Intelligence Center | Official Docs: [cloud.google.com/network-intelligence-center](https://cloud.google.com/network-intelligence-center/docs/overview)*
