# ☁️ GCP Cloud VPN — Comprehensive Cheatsheet

> **Audience:** Cloud engineers setting up, managing, or troubleshooting GCP Cloud VPN.
> **Last updated:** March 2026 | Covers HA VPN (recommended) and Classic VPN (legacy).

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Architecture & Topology](#2-architecture--topology)
3. [Routing Options](#3-routing-options)
4. [Quotas & Limits](#4-quotas--limits)
5. [Setup & Configuration](#5-setup--configuration)
6. [Terraform Example](#6-terraform-example)
7. [Monitoring & Troubleshooting](#7-monitoring--troubleshooting)
8. [Security Best Practices](#8-security-best-practices)
9. [Pricing Summary](#9-pricing-summary)
10. [Quick Reference Table](#10-quick-reference-table)

---

## 1. Overview & Key Concepts

### What is Cloud VPN?

Cloud VPN securely connects your on-premises network or another cloud provider's network to your GCP **Virtual Private Cloud (VPC)** over an **IPsec** encrypted tunnel traversing the public internet.

**When to use Cloud VPN:**
- Hybrid connectivity when **Cloud Interconnect** is too costly or unavailable
- Dev/staging environments needing secure access to GCP resources
- Connecting multiple VPCs across regions or organizations
- Disaster recovery paths alongside Interconnect

> ⚠️ Cloud VPN is **not** suitable for workloads requiring guaranteed bandwidth, sub-10ms latency, or >3 Gbps sustained throughput. Use [Cloud Interconnect](https://cloud.google.com/network-connectivity/docs/interconnect) instead.

---

### VPN Types: Classic VPN vs. HA VPN

| Feature | Classic VPN | HA VPN |
|---|---|---|
| SLA | 99.9% | **99.99%** |
| External IPs | 1 (single interface) | **2** (2 interfaces) |
| Tunnels per gateway | Up to 8 | Up to 4 per interface (8 total) |
| Routing support | Static, dynamic, policy-based | **Dynamic (BGP) only** |
| Recommended | ❌ Legacy | ✅ **Use for all new deployments** |
| Use case | Simple/dev setups | Production, HA requirements |

> ⚠️ **Classic VPN** is in **legacy status**. Google recommends migrating to HA VPN. Classic VPN does **not** support active/active tunnels with BGP.

---

### Key Terminology

| Term | Description |
|---|---|
| **VPN Gateway** | GCP-side endpoint; holds one or more external IPs |
| **VPN Tunnel** | Encrypted IPsec link between GCP gateway and peer gateway |
| **Peer Gateway** | On-premises or third-party VPN device/gateway |
| **Cloud Router** | GCP-managed BGP router; required for dynamic routing |
| **BGP Session** | TCP session over a tunnel for exchanging routes dynamically |
| **IKE** | Internet Key Exchange — negotiates tunnel security parameters |
| **ESP** | Encapsulating Security Payload — encrypts tunnel data (protocol 50) |
| **Shared Secret (PSK)** | Pre-shared key used to authenticate IKE negotiation |
| **ASN** | Autonomous System Number — identifies a BGP router |
| **Link-local IPs** | `169.254.x.x/30` IPs used for BGP peering inside tunnels |

---

### IKE Versions & Cipher Suites

| | IKEv1 | IKEv2 |
|---|---|---|
| Support | ✅ Classic & HA VPN | ✅ Classic & HA VPN |
| Recommended | ❌ | ✅ **Preferred** |
| Dead Peer Detection | Supported | Supported (built-in) |
| MOBIKE support | ❌ | ✅ |

**Supported Phase 1 / Phase 2 Cipher Suites (IKEv2):**

```
Encryption : AES-128-CBC, AES-256-CBC, AES-128-GCM, AES-256-GCM
Integrity  : HMAC-SHA1-96, HMAC-SHA256-128, HMAC-SHA384-192, HMAC-SHA512-256
PRF        : PRF-HMAC-SHA1, PRF-HMAC-SHA256, PRF-HMAC-SHA384, PRF-HMAC-SHA512
DH Groups  : 2, 5, 14 (recommended), 15, 16, 18, 19, 20, 21, 24
```

> 💡 Use **AES-256-GCM** + **HMAC-SHA256** + **DH Group 14** or higher for strongest security.

> ⚠️ If your peer device only supports IKEv1, Cloud VPN will fall back — but this is less secure and not recommended.

---

## 2. Architecture & Topology

### How Cloud VPN Fits into VPC Networking

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GCP Project                                  │
│                                                                     │
│   ┌──────────────────────────────────────────────────────────────┐  │
│   │                      VPC Network                             │  │
│   │                                                              │  │
│   │   ┌─────────────┐    Routes     ┌────────────────────────┐  │  │
│   │   │  GCE / GKE  │◄─────────────►│     Cloud Router       │  │  │
│   │   │  Workloads  │               │  (BGP, ASN: 65001)     │  │  │
│   │   └─────────────┘               └──────────┬─────────────┘  │  │
│   │                                            │ BGP Session     │  │
│   │                                 ┌──────────▼─────────────┐  │  │
│   │                                 │   HA VPN Gateway       │  │  │
│   │                                 │  IF0: 34.x.x.x         │  │  │
│   │                                 │  IF1: 35.x.x.x         │  │  │
│   └─────────────────────────────────┴────────────────────────┘  │  │
└────────────────────────────────────┬────────────────────────────┘  │
                                     │ IPsec Tunnels (ESP over UDP)    │
                          ┌──────────▼──────────────┐
                          │    Public Internet       │
                          └──────────┬──────────────┘
                                     │
                          ┌──────────▼──────────────┐
                          │   On-Prem VPN Device     │
                          │   (Peer Gateway)         │
                          │   IP: 203.0.113.1        │
                          └─────────────────────────┘
                                     │
                          ┌──────────▼──────────────┐
                          │   On-Prem Network        │
                          │   192.168.0.0/16         │
                          └─────────────────────────┘
```

---

### HA VPN Architecture (99.99% SLA)

```
HA VPN Gateway
├── Interface 0 → External IP A
│   ├── Tunnel 0 → Peer Gateway Interface 0   [ACTIVE]
│   └── Tunnel 1 → Peer Gateway Interface 1   [ACTIVE]
└── Interface 1 → External IP B
    ├── Tunnel 2 → Peer Gateway Interface 0   [ACTIVE]
    └── Tunnel 3 → Peer Gateway Interface 1   [ACTIVE]
```

- **4 tunnels** across 2 GCP interfaces → 2 peer interfaces
- All tunnels **active simultaneously** (active/active)
- Requires **Cloud Router** with BGP on every tunnel
- Achieves 99.99% SLA when peer also has 2 interfaces

> 💡 If your peer only has **1 interface**, configure 2 tunnels (one per GCP interface) for 99.99% SLA on the GCP side.

---

### Classic VPN Architecture (99.9% SLA)

```
Classic VPN Gateway
└── Single Interface → External IP
    ├── Tunnel 1 → Peer Gateway   [ACTIVE]
    └── Tunnel 2 → Peer Gateway   [STANDBY] (policy/static only)
```

- Only **1 external IP** — single point of failure
- Supports static routes, policy-based, or dynamic (BGP)
- BGP supported but active/active NOT possible
- Max **99.9% SLA**

---

### Active/Active vs Active/Passive

| Mode | Description | Routing | Failover |
|---|---|---|---|
| **Active/Active** | All tunnels forward traffic simultaneously | BGP with ECMP | Automatic, sub-second |
| **Active/Passive** | One tunnel active, others on standby | Static or BGP | Manual or BGP reconvergence |

> 💡 HA VPN with BGP defaults to **active/active** (ECMP) — traffic is load-balanced across all healthy tunnels.

---

## 3. Routing Options

### Overview

| Routing Type | VPN Type | Description |
|---|---|---|
| **Dynamic (BGP)** | Classic & HA VPN | Cloud Router exchanges routes via BGP; auto-learns prefixes |
| **Static (route-based)** | Classic VPN only | Manually configured remote IP ranges; no Cloud Router needed |
| **Policy-based** | Classic VPN only | Traffic selected by source/dest IP ranges (not standard routing) |

> ⚠️ **HA VPN requires dynamic routing (BGP)**. Static and policy-based routing are not supported.

---

### Dynamic Routing with BGP (Recommended)

BGP routes are exchanged over the tunnel using **link-local addresses** (`169.254.x.x/30`).

**Key BGP concepts for Cloud VPN:**

| Parameter | Description | Example |
|---|---|---|
| **Cloud Router ASN** | GCP-side BGP identifier | `65001` (private range: 64512–65534) |
| **Peer ASN** | On-prem BGP identifier | `65002` |
| **BGP peer IP** | Link-local IP on peer side | `169.254.0.2` |
| **Cloud Router BGP IP** | Link-local IP on GCP side | `169.254.0.1` |
| **Advertised ranges** | Subnets to announce to peer | VPC subnets (auto or custom) |

**Routing modes:**

- `GLOBAL` routing: Cloud Router advertises all subnets across all regions
- `REGIONAL` routing: Cloud Router only advertises subnets in its region (default)

```bash
# Set routing mode to GLOBAL on VPC network
gcloud compute networks update MY_VPC \
  --bgp-routing-mode=GLOBAL
```

---

### Static (Route-Based) Routing — Classic VPN Only

```bash
# Add a static route pointing traffic to the VPN tunnel
gcloud compute routes create ROUTE_NAME \
  --network=MY_VPC \
  --destination-range=192.168.0.0/16 \       # Remote network
  --next-hop-vpn-tunnel=TUNNEL_NAME \
  --next-hop-vpn-tunnel-region=us-central1 \
  --priority=1000
```

---

### Policy-Based Routing — Classic VPN Only

Traffic is matched by both **local** and **remote** IP ranges (defined in the tunnel config). There are no routes added to the VPC routing table — traffic matching the selectors is sent into the tunnel.

> ⚠️ Policy-based routing is the **least flexible** option — avoid for new setups. Use BGP instead.

---

## 4. Quotas & Limits

| Resource | Default Limit | Notes |
|---|---|---|
| HA VPN gateways per project | 15 per region | Quota increase requestable |
| Classic VPN gateways per project | 15 per region | Legacy |
| Tunnels per HA VPN gateway | 8 (4 per interface) | |
| Tunnels per Classic VPN gateway | 8 | |
| **Bandwidth per tunnel** | **1.5–3 Gbps** | Depends on packet size |
| Recommended MTU | **1460 bytes** | Avoid fragmentation |
| Max MTU | 1500 bytes | Not recommended |
| BGP sessions per Cloud Router | 250 | Across all tunnels |
| Supported IP protocols | TCP, UDP, ICMP, ESP, GRE, SCTP | |
| IPv6 support | ✅ (HA VPN only, dual-stack) | Preview in some regions |

> 💡 **MTU Recommendation:** Set your on-prem VPN device and VM interfaces to **1460 bytes** to avoid packet fragmentation inside the IPsec tunnel.

```bash
# Set MTU on a GCE VM interface (Linux)
sudo ip link set eth0 mtu 1460
```

**Firewall rules required on GCP side:**

| Protocol | Port | Purpose |
|---|---|---|
| UDP | 500 | IKE negotiation |
| UDP | 4500 | NAT-T (IKE/ESP over NAT) |
| ESP (proto 50) | — | Encrypted tunnel data |

> ⚠️ GCP Cloud VPN **automatically** allows ESP/UDP 500/4500 on the VPN gateway. You may need to allow these on your **on-prem firewall**.

---

## 5. Setup & Configuration

### Prerequisites

- GCP project with Compute Engine API enabled
- A VPC network and subnet
- On-prem VPN device with public IP and IKE support
- `gcloud` CLI authenticated: `gcloud auth login`

---

### Step 1 — Create an HA VPN Gateway

```bash
# Create the HA VPN gateway (allocates 2 external IPs automatically)
gcloud compute vpn-gateways create HA_VPN_GW_NAME \
  --network=MY_VPC \                    # Target VPC network
  --region=us-central1 \               # GCP region
  --stack-type=IPV4_ONLY               # Use IPV4_IPV6 for dual-stack

# View the 2 external IPs assigned to interfaces 0 and 1
gcloud compute vpn-gateways describe HA_VPN_GW_NAME \
  --region=us-central1
```

---

### Step 2 — Create an External (Peer) VPN Gateway

```bash
# Register your on-prem VPN device in GCP (needed for HA VPN)
# Two-interface peer (for full 99.99% SLA)
gcloud compute external-vpn-gateways create PEER_GW_NAME \
  --redundancy-type=TWO_IPS_REDUNDANCY \
  --interfaces 0=203.0.113.1,1=203.0.113.2   # Peer's public IPs

# Single-interface peer
gcloud compute external-vpn-gateways create PEER_GW_NAME \
  --redundancy-type=SINGLE_IP_INTERNALLY_REDUNDANT \
  --interfaces 0=203.0.113.1
```

---

### Step 3 — Create a Cloud Router

```bash
# Cloud Router is required for BGP with HA VPN
gcloud compute routers create CLOUD_ROUTER_NAME \
  --network=MY_VPC \
  --region=us-central1 \
  --asn=65001                           # GCP-side BGP ASN (private range recommended)
```

---

### Step 4 — Create VPN Tunnels

```bash
# Tunnel 0: GCP interface 0 → Peer interface 0
gcloud compute vpn-tunnels create TUNNEL_0 \
  --region=us-central1 \
  --vpn-gateway=HA_VPN_GW_NAME \       # HA VPN gateway name
  --vpn-gateway-interface=0 \          # GCP gateway interface (0 or 1)
  --peer-external-gateway=PEER_GW_NAME \
  --peer-external-gateway-interface=0 \  # Peer interface index
  --shared-secret="YOUR_STRONG_PSK" \  # Pre-shared key (min 32 chars recommended)
  --ike-version=2 \                    # Use IKEv2
  --router=CLOUD_ROUTER_NAME

# Tunnel 1: GCP interface 0 → Peer interface 1
gcloud compute vpn-tunnels create TUNNEL_1 \
  --region=us-central1 \
  --vpn-gateway=HA_VPN_GW_NAME \
  --vpn-gateway-interface=0 \
  --peer-external-gateway=PEER_GW_NAME \
  --peer-external-gateway-interface=1 \
  --shared-secret="YOUR_STRONG_PSK" \
  --ike-version=2 \
  --router=CLOUD_ROUTER_NAME

# Tunnel 2: GCP interface 1 → Peer interface 0
gcloud compute vpn-tunnels create TUNNEL_2 \
  --region=us-central1 \
  --vpn-gateway=HA_VPN_GW_NAME \
  --vpn-gateway-interface=1 \
  --peer-external-gateway=PEER_GW_NAME \
  --peer-external-gateway-interface=0 \
  --shared-secret="YOUR_STRONG_PSK" \
  --ike-version=2 \
  --router=CLOUD_ROUTER_NAME

# Tunnel 3: GCP interface 1 → Peer interface 1
gcloud compute vpn-tunnels create TUNNEL_3 \
  --region=us-central1 \
  --vpn-gateway=HA_VPN_GW_NAME \
  --vpn-gateway-interface=1 \
  --peer-external-gateway=PEER_GW_NAME \
  --peer-external-gateway-interface=1 \
  --shared-secret="YOUR_STRONG_PSK" \
  --ike-version=2 \
  --router=CLOUD_ROUTER_NAME
```

---

### Step 5 — Configure BGP Sessions on Cloud Router

```bash
# Add BGP interface for Tunnel 0 on Cloud Router
gcloud compute routers add-interface CLOUD_ROUTER_NAME \
  --region=us-central1 \
  --interface-name=BGP_IF_0 \
  --vpn-tunnel=TUNNEL_0 \
  --ip-address=169.254.0.1 \           # GCP-side link-local BGP IP
  --mask-length=30                     # Always /30 for point-to-point

# Add BGP peer for Tunnel 0
gcloud compute routers add-bgp-peer CLOUD_ROUTER_NAME \
  --region=us-central1 \
  --peer-name=BGP_PEER_0 \
  --interface=BGP_IF_0 \
  --peer-ip-address=169.254.0.2 \      # Peer-side link-local BGP IP
  --peer-asn=65002                     # On-prem BGP ASN

# Repeat for TUNNEL_1, TUNNEL_2, TUNNEL_3 with unique 169.254.x.x/30 ranges
# e.g., 169.254.1.1/169.254.1.2, 169.254.2.1/169.254.2.2, 169.254.3.1/169.254.3.2
```

---

### Step 6 — Verify Tunnel & BGP Status

```bash
# List all VPN tunnels and their status
gcloud compute vpn-tunnels list --region=us-central1

# Describe a specific tunnel (check detailedStatus field)
gcloud compute vpn-tunnels describe TUNNEL_0 --region=us-central1

# Check BGP session status on Cloud Router
gcloud compute routers get-status CLOUD_ROUTER_NAME \
  --region=us-central1 \
  --format="json(result.bgpPeerStatus)"

# Expected BGP state: "Established"
# Expected tunnel detailedStatus: "Tunnel is up and running."
```

---

## 6. Terraform Example

> 💡 This is a minimal, complete HA VPN setup with 2 tunnels (single peer interface). Extend to 4 tunnels for full redundancy.

```hcl
# ── Variables ────────────────────────────────────────────────────────
variable "project_id"    { default = "my-gcp-project" }
variable "region"        { default = "us-central1" }
variable "peer_ip"       { default = "203.0.113.1" }   # On-prem public IP
variable "shared_secret" { sensitive = true }           # Pass via -var or tfvars

# ── Provider ─────────────────────────────────────────────────────────
provider "google" {
  project = var.project_id
  region  = var.region
}

# ── HA VPN Gateway ───────────────────────────────────────────────────
resource "google_compute_ha_vpn_gateway" "ha_vpn_gw" {
  name    = "ha-vpn-gw"
  network = "projects/${var.project_id}/global/networks/my-vpc"
  region  = var.region
}

# ── External (Peer) VPN Gateway ──────────────────────────────────────
resource "google_compute_external_vpn_gateway" "peer_gw" {
  name            = "on-prem-peer-gw"
  redundancy_type = "SINGLE_IP_INTERNALLY_REDUNDANT"

  interface {
    id         = 0
    ip_address = var.peer_ip
  }
}

# ── Cloud Router ─────────────────────────────────────────────────────
resource "google_compute_router" "cloud_router" {
  name    = "cloud-router-vpn"
  network = "my-vpc"
  region  = var.region

  bgp {
    asn               = 65001
    advertise_mode    = "DEFAULT"   # Advertise all VPC subnets
  }
}

# ── VPN Tunnels ──────────────────────────────────────────────────────
resource "google_compute_vpn_tunnel" "tunnel_0" {
  name                            = "ha-vpn-tunnel-0"
  region                          = var.region
  vpn_gateway                     = google_compute_ha_vpn_gateway.ha_vpn_gw.id
  vpn_gateway_interface           = 0                  # GCP interface 0
  peer_external_gateway           = google_compute_external_vpn_gateway.peer_gw.id
  peer_external_gateway_interface = 0
  shared_secret                   = var.shared_secret
  ike_version                     = 2
  router                          = google_compute_router.cloud_router.id
}

resource "google_compute_vpn_tunnel" "tunnel_1" {
  name                            = "ha-vpn-tunnel-1"
  region                          = var.region
  vpn_gateway                     = google_compute_ha_vpn_gateway.ha_vpn_gw.id
  vpn_gateway_interface           = 1                  # GCP interface 1
  peer_external_gateway           = google_compute_external_vpn_gateway.peer_gw.id
  peer_external_gateway_interface = 0
  shared_secret                   = var.shared_secret
  ike_version                     = 2
  router                          = google_compute_router.cloud_router.id
}

# ── Cloud Router Interfaces (BGP) ────────────────────────────────────
resource "google_compute_router_interface" "bgp_if_0" {
  name       = "bgp-if-0"
  router     = google_compute_router.cloud_router.name
  region     = var.region
  ip_range   = "169.254.0.1/30"                        # GCP BGP link-local IP
  vpn_tunnel = google_compute_vpn_tunnel.tunnel_0.name
}

resource "google_compute_router_interface" "bgp_if_1" {
  name       = "bgp-if-1"
  router     = google_compute_router.cloud_router.name
  region     = var.region
  ip_range   = "169.254.1.1/30"
  vpn_tunnel = google_compute_vpn_tunnel.tunnel_1.name
}

# ── BGP Peers ────────────────────────────────────────────────────────
resource "google_compute_router_peer" "bgp_peer_0" {
  name            = "bgp-peer-0"
  router          = google_compute_router.cloud_router.name
  region          = var.region
  interface       = google_compute_router_interface.bgp_if_0.name
  peer_ip_address = "169.254.0.2"                      # Peer BGP link-local IP
  peer_asn        = 65002                              # On-prem ASN
  advertised_route_priority = 100
}

resource "google_compute_router_peer" "bgp_peer_1" {
  name            = "bgp-peer-1"
  router          = google_compute_router.cloud_router.name
  region          = var.region
  interface       = google_compute_router_interface.bgp_if_1.name
  peer_ip_address = "169.254.1.2"
  peer_asn        = 65002
  advertised_route_priority = 100
}

# ── Outputs ──────────────────────────────────────────────────────────
output "ha_vpn_gateway_ip_0" {
  description = "HA VPN external IP for interface 0"
  value       = google_compute_ha_vpn_gateway.ha_vpn_gw.vpn_interfaces[0].ip_address
}

output "ha_vpn_gateway_ip_1" {
  description = "HA VPN external IP for interface 1"
  value       = google_compute_ha_vpn_gateway.ha_vpn_gw.vpn_interfaces[1].ip_address
}
```

```bash
# Deploy
terraform init
terraform plan -var="shared_secret=YourStrongPSKHere"
terraform apply -var="shared_secret=YourStrongPSKHere"
```

---

## 7. Monitoring & Troubleshooting

### Key Cloud Monitoring Metrics

| Metric | Description | Alert Threshold |
|---|---|---|
| `vpn.googleapis.com/tunnel/status` | Tunnel up/down (1=up, 0=down) | Alert if `0` |
| `vpn.googleapis.com/network/sent_bytes_count` | Bytes sent through tunnel | Baseline + anomaly |
| `vpn.googleapis.com/network/received_bytes_count` | Bytes received | Baseline + anomaly |
| `vpn.googleapis.com/network/dropped_sent_packets_count` | Dropped outbound packets | Alert if > 0 sustained |
| `vpn.googleapis.com/network/dropped_received_packets_count` | Dropped inbound packets | Alert if > 0 sustained |

> 💡 Create an **uptime check** or **alerting policy** in Cloud Monitoring on `tunnel/status == 0` to get paged on tunnel failures.

---

### Useful `gcloud` Status Commands

```bash
# Check all tunnel statuses
gcloud compute vpn-tunnels list \
  --region=us-central1 \
  --format="table(name,status,detailedStatus)"

# Check BGP peer sessions (look for "Established")
gcloud compute routers get-status CLOUD_ROUTER_NAME \
  --region=us-central1 \
  --format="json(result.bgpPeerStatus[].{name:name,status:status,ip:ipAddress,peerIp:peerIpAddress})"

# List learned BGP routes from peer
gcloud compute routers get-status CLOUD_ROUTER_NAME \
  --region=us-central1 \
  --format="json(result.bestRoutes)"

# Check advertised routes to peer
gcloud compute routers get-status CLOUD_ROUTER_NAME \
  --region=us-central1 \
  --format="json(result.bgpPeerStatus[].advertisedRoutes)"
```

---

### Common Failure Causes & Fixes

| Symptom | Likely Cause | Fix |
|---|---|---|
| Tunnel stuck in `Waiting for full config` | Missing BGP interface/peer on Cloud Router | Add router interface and BGP peer |
| Tunnel status `First Handshake` forever | IKE mismatch or firewall blocking UDP 500/4500 | Verify cipher suites match; allow UDP 500/4500 on peer firewall |
| Tunnel `Up` but no traffic flows | Missing firewall rules or routes | Check VPC firewall rules; verify BGP routes are learned |
| BGP session `Idle` | Wrong ASN, peer IP, or link-local IP mismatch | Verify ASN and `169.254.x.x` IPs match on both sides |
| BGP `Active` but not `Established` | TCP port 179 blocked between BGP IPs | Check on-prem firewall allows TCP 179 on link-local range |
| Packet loss / high latency | MTU mismatch causing fragmentation | Set MTU to 1460 on both ends |
| Tunnel flapping | DPD (Dead Peer Detection) failure | Check on-prem device DPD config; check internet path stability |
| `Allocating resources` status | Gateway still provisioning | Wait 2–5 minutes after creation |

---

### Enabling VPN Tunnel Logs (Cloud Logging)

```bash
# VPN tunnel logs are automatically sent to Cloud Logging under:
# logName: "projects/PROJECT_ID/logs/vpn-tunnel"

# Query logs in gcloud
gcloud logging read \
  'resource.type="vpn_gateway" AND severity>=WARNING' \
  --project=MY_PROJECT \
  --limit=50 \
  --format="table(timestamp,severity,jsonPayload.message)"
```

**In Cloud Logging UI — useful filter:**

```
resource.type="vpn_gateway"
resource.labels.gateway_id="HA_VPN_GW_NAME"
severity>=WARNING
```

> 💡 VPN tunnel events (up/down, IKE failures) are automatically logged. No additional enablement required.

---

## 8. Security Best Practices

### ✅ Pre-Shared Key (PSK) Guidelines

- Minimum **32 characters** — prefer **64+** characters
- Use a **cryptographically random** PSK:
  ```bash
  # Generate a strong PSK
  openssl rand -base64 48
  ```
- **Never hardcode** PSK in source code — use Secret Manager:
  ```bash
  # Store PSK in Secret Manager
  echo -n "YOUR_PSK" | gcloud secrets create vpn-psk \
    --data-file=- \
    --replication-policy=automatic

  # Reference in scripts
  PSK=$(gcloud secrets versions access latest --secret=vpn-psk)
  ```
- Use **different PSKs per tunnel** for blast-radius reduction
- **Rotate PSKs** periodically (update both ends simultaneously)

---

### ✅ Recommended IKE Configuration

```
IKE Version  : IKEv2 (always prefer over IKEv1)
Encryption   : AES-256-GCM
Integrity    : SHA-256 or SHA-384
DH Group     : 14 (2048-bit MODP) or higher (19, 20, 21)
PFS          : Enabled (Perfect Forward Secrecy)
DPD          : Enabled (Dead Peer Detection)
```

> ❌ Avoid: **DES, 3DES, MD5, SHA-1, DH Group 1/2/5**

---

### ✅ IAM Roles for VPN Management

| Role | Permissions | Use For |
|---|---|---|
| `roles/compute.networkAdmin` | Full VPN/network resource management | VPN administrators |
| `roles/compute.networkViewer` | Read-only access to network resources | Monitoring/auditing |
| `roles/compute.securityAdmin` | Manage firewall rules | Security team |
| `roles/viewer` | Project-wide read-only | Auditors |

```bash
# Grant VPN admin role to a service account
gcloud projects add-iam-policy-binding MY_PROJECT \
  --member="serviceAccount:vpn-sa@MY_PROJECT.iam.gserviceaccount.com" \
  --role="roles/compute.networkAdmin"
```

> 💡 Follow **least privilege** — grant `networkAdmin` only to automation SAs and specific engineers, not all project editors.

---

### ✅ Additional Security Recommendations

- ❌ **Do not use Classic VPN** for new deployments — no 99.99% SLA, legacy ciphers more likely on older devices
- ✅ Enable **VPC Service Controls** to restrict API access over VPN
- ✅ Use **Private Google Access** over VPN so on-prem hosts reach Google APIs without hitting the public internet
- ✅ Restrict on-prem advertised routes — only announce necessary prefixes via BGP (use route filtering)
- ✅ Use **Cloud Armor** in front of workloads that receive on-prem traffic for additional L7 protection
- ✅ Enable **audit logging** for `compute.googleapis.com` to track VPN config changes

---

## 9. Pricing Summary

> ⚠️ Pricing below is approximate as of early 2026. Always verify at [cloud.google.com/vpn/pricing](https://cloud.google.com/network-connectivity/docs/vpn/pricing).

| Component | Cost |
|---|---|
| **VPN tunnel** (HA or Classic) | ~**$0.05 / hour** per tunnel |
| **Egress to on-prem** | Standard internet egress rates apply (varies by region, typically $0.08–$0.12/GB) |
| **Ingress** | Free |
| **Cloud Router** | ~$0.01/hour per BGP session (when using dynamic routing) |
| **Free tier** | ❌ No free tier for Cloud VPN tunnels |

**Monthly cost estimate (HA VPN, 4 tunnels, 24/7):**

```
4 tunnels × $0.05/hr × 730 hrs = ~$146/month (tunnel cost only)
+ egress data transfer costs
```

> 💡 Tunnels are billed even when idle. Delete unused tunnels to avoid unnecessary charges.

---

## 10. Quick Reference Table

### Classic VPN vs. HA VPN

| Feature | Classic VPN | HA VPN |
|---|---|---|
| **SLA** | 99.9% | **99.99%** |
| **External IPs** | 1 | **2** |
| **Max tunnels** | 8 | 8 (4 per interface) |
| **Routing: Dynamic (BGP)** | ✅ | ✅ (required) |
| **Routing: Static** | ✅ | ❌ |
| **Routing: Policy-based** | ✅ | ❌ |
| **Active/Active tunnels** | ❌ | ✅ |
| **Cloud Router required** | Only for BGP | ✅ Always |
| **External gateway object** | ❌ | ✅ Required |
| **IPv6 support** | ❌ | ✅ (dual-stack) |
| **Recommended** | ❌ Legacy | ✅ **All new deployments** |
| **Typical use case** | Dev/test, migration | Production, HA workloads |

---

### IKEv1 vs. IKEv2 Quick Reference

| Feature | IKEv1 | IKEv2 |
|---|---|---|
| Supported by Cloud VPN | ✅ | ✅ |
| Recommended | ❌ | ✅ |
| MOBIKE | ❌ | ✅ |
| Built-in DPD | ❌ | ✅ |
| Exchange efficiency | Slower | Faster |
| EAP auth support | ❌ | ✅ |

---

### Tunnel Status Reference

| Status | Meaning | Action |
|---|---|---|
| `Tunnel is up and running.` | ✅ Healthy | None |
| `Waiting for full config` | ⏳ BGP not configured | Add Cloud Router BGP interface/peer |
| `First Handshake` | 🔄 IKE negotiating | Check firewall & cipher match |
| `Allocating resources` | ⏳ Provisioning | Wait |
| `Tunnel is down. No incoming packets.` | ❌ Peer unreachable | Check peer firewall, peer device status |
| `Tunnel is down due to local network issues.` | ❌ GCP-side issue | Check GCP status page |

---

### BGP Session Status Reference

| BGP Status | Meaning |
|---|---|
| `Established` | ✅ BGP working, routes exchanging |
| `Active` | 🔄 Attempting TCP connection to peer |
| `Idle` | ❌ Not trying — config error (wrong IP/ASN) |
| `Connect` | 🔄 TCP handshake in progress |
| `OpenSent` | 🔄 BGP OPEN message sent |

---

*Generated for GCP Cloud VPN | For updates see [Official Docs](https://cloud.google.com/network-connectivity/docs/vpn/concepts/overview)*
