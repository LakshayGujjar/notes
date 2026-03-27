# ☁️ GCP Virtual Private Cloud (VPC) — Comprehensive Cheatsheet

> **Audience:** Beginner to Advanced Cloud Engineers
> **Last Updated:** 2025 | **GCP Product:** Virtual Private Cloud

---

## Table of Contents

1. [Overview & Core Concepts](#1-overview--core-concepts)
2. [VPC Network Architecture](#2-vpc-network-architecture)
3. [Firewall Rules](#3-firewall-rules)
4. [Routes & Routing](#4-routes--routing)
5. [VPC Peering & Shared VPC](#5-vpc-peering--shared-vpc)
6. [Hybrid Connectivity](#6-hybrid-connectivity)
7. [Private Google Access & Private Service Connect](#7-private-google-access--private-service-connect)
8. [Cloud NAT](#8-cloud-nat)
9. [VPC Flow Logs](#9-vpc-flow-logs)
10. [Network Security Best Practices](#10-network-security-best-practices)
11. [Key gcloud Commands Cheatsheet](#11-key-gcloud-commands-cheatsheet)
12. [Limits & Quotas](#12-limits--quotas)

---

## 1. Overview & Core Concepts

### What is GCP VPC?

A **Virtual Private Cloud (VPC)** in GCP is a **globally scoped, software-defined network** that provides connectivity for your Compute Engine VMs, GKE clusters, App Engine flexible environments, and other GCP services. Unlike traditional on-prem networks, a GCP VPC is:

- **Global by default** — a single VPC spans all GCP regions without requiring VPC peering or gateways between regions
- **Software-defined** — no physical routers or switches; managed entirely via API/console/gcloud
- **Project-scoped** — each VPC belongs to one GCP project (unless using Shared VPC)
- **Not tied to an IP range at the VPC level** — IP ranges are defined at the **subnet** level

---

### GCP VPC vs Traditional Networking

| Feature | Traditional Network | GCP VPC |
|---|---|---|
| Scope | Single datacenter / region | **Global** (spans all regions) |
| Routing between regions | Requires WAN/MPLS links | **Automatic** via Google's backbone |
| IP management | VLAN/subnet per physical segment | Subnets per region, VPC is logical |
| Firewall | Hardware appliance or host-based | **Distributed, stateful** firewall rules |
| Scalability | Manual provisioning | Elastic, API-driven |
| Multi-tenancy isolation | Physical separation or VLANs | IAM + VPC isolation |

---

### Auto Mode vs Custom Mode

| | **Auto Mode VPC** | **Custom Mode VPC** |
|---|---|---|
| Subnet creation | One subnet per region created automatically | You define all subnets manually |
| IP ranges | Fixed `/20` ranges from `10.128.0.0/9` | Fully flexible — any valid RFC 1918 range |
| Best for | Quick prototypes / dev environments | Production, enterprise, multi-team setups |
| Can convert? | ✅ Auto → Custom (one-way, irreversible) | ❌ Cannot convert to auto |
| New region support | New subnets added automatically | Must add manually |

> ⚠️ **Gotcha:** Auto mode uses pre-defined IP ranges. If these overlap with your on-prem or peered networks, you'll hit routing conflicts. Always use **custom mode** for production.

> 💡 **Tip:** You can convert an auto-mode VPC to custom mode via:
> ```bash
> gcloud compute networks update MY_VPC --switch-to-custom-subnet-mode
> ```

---

### Default VPC

Every new GCP project comes with a **default VPC** (auto mode) with:
- Pre-created subnets in every region
- Default firewall rules (`allow-internal`, `allow-ssh`, `allow-rdp`, `allow-icmp`)

> ⚠️ **Gotcha:** The default VPC is convenient but insecure for production. Delete it or restrict its firewall rules before deploying sensitive workloads.

---

## 2. VPC Network Architecture

### ASCII Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         GCP PROJECT                             │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     VPC Network (Global)                  │  │
│  │                                                           │  │
│  │  ┌──────────────────────┐  ┌──────────────────────────┐  │  │
│  │  │   Region: us-east1   │  │   Region: europe-west1   │  │  │
│  │  │                      │  │                          │  │  │
│  │  │  ┌────────────────┐  │  │  ┌────────────────────┐  │  │  │
│  │  │  │ Subnet A       │  │  │  │ Subnet C           │  │  │  │
│  │  │  │ 10.0.1.0/24    │  │  │  │ 10.0.3.0/24        │  │  │  │
│  │  │  │  [VM] [VM]     │  │  │  │  [VM]              │  │  │  │
│  │  │  └────────────────┘  │  │  └────────────────────┘  │  │  │
│  │  │  ┌────────────────┐  │  │                          │  │  │
│  │  │  │ Subnet B       │  │  └──────────────────────────┘  │  │
│  │  │  │ 10.0.2.0/24    │  │                               │  │
│  │  │  │  [GKE Cluster] │  │  ┌──────────────────────────┐  │  │
│  │  │  └────────────────┘  │  │   Region: asia-east1     │  │  │
│  │  └──────────────────────┘  │  ┌────────────────────┐  │  │  │
│  │                            │  │ Subnet D           │  │  │  │
│  │                            │  │ 10.0.4.0/24        │  │  │  │
│  │                            │  └────────────────────┘  │  │  │
│  │                            └──────────────────────────┘  │  │
│  │                                                           │  │
│  │         Internal traffic routed via Google backbone       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  [Cloud Router]  [Cloud NAT]  [VPN Gateway]  [Interconnect]    │
└─────────────────────────────────────────────────────────────────┘
```

---

### Subnets

- Subnets are **regional** resources — they exist within a specific region
- Each subnet has one **primary IP range** (used by VMs' primary interfaces)
- Each subnet can have multiple **secondary IP ranges** (used by GKE Pods and Services)
- Subnets within the same VPC can communicate across regions without extra config

```bash
# Create a custom subnet
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.1.0/24

# Create subnet with secondary ranges (for GKE)
gcloud compute networks subnets create gke-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.2.0/24 \
  --secondary-range pods=10.1.0.0/16,services=10.2.0.0/20
```

> 💡 **Tip:** Secondary IP ranges are required for GKE alias IP-based clusters. Plan your IP space to avoid exhaustion — GKE pods can consume large CIDR blocks.

---

### CIDR Notation Quick Reference

| CIDR | Subnet Mask | # of Usable IPs | Common Use |
|---|---|---|---|
| `/8` | 255.0.0.0 | ~16.7M | Large enterprise |
| `/16` | 255.255.0.0 | ~65K | VPC-level planning |
| `/20` | 255.255.240.0 | 4,094 | Auto mode default |
| `/24` | 255.255.255.0 | 254 | Standard subnet |
| `/28` | 255.255.255.240 | 14 | Small subnet / PSC |
| `/29` | 255.255.255.248 | 6 | Minimum for VPC peering |

> ⚠️ **Gotcha:** GCP reserves **4 IPs per subnet**: network address, default gateway, second-to-last, and broadcast. A `/29` gives only 3 usable IPs for VMs.

---

### IP Address Types

| Type | Description | Scope |
|---|---|---|
| **Internal (Private)** | RFC 1918 address from subnet range; assigned to VM NIC | Within VPC |
| **External (Ephemeral)** | Publicly routable IP; released when VM stops | Global |
| **External (Static)** | Reserved public IP; persists across VM restarts | Regional or Global |
| **Alias IP** | Secondary IPs on a VM's NIC from primary or secondary range | Within VPC |
| **Private Google Access** | Allows VMs without external IPs to reach Google APIs | Via internal routing |
| **Internal Load Balancer IP** | Frontend IP of an ILB, routable within VPC | Regional |

```bash
# Reserve a static external IP
gcloud compute addresses create my-static-ip \
  --region=us-central1

# Reserve a global static IP (for HTTPS Load Balancer)
gcloud compute addresses create my-global-ip \
  --global

# List all reserved IPs
gcloud compute addresses list
```

---

## 3. Firewall Rules

### Core Concepts

GCP firewall rules are **stateful** (connection tracking), **distributed** (enforced at the VM level), and **VPC-scoped**.

Every VPC has two **implied rules** (lowest priority = 65535) that cannot be deleted:
- `deny-all-ingress` — blocks all inbound traffic
- `allow-all-egress` — permits all outbound traffic

---

### Rule Components

| Component | Description | Example Values |
|---|---|---|
| **Direction** | `INGRESS` or `EGRESS` | `INGRESS` |
| **Priority** | 0 (highest) – 65535 (lowest) | `1000` (default) |
| **Action** | `ALLOW` or `DENY` | `ALLOW` |
| **Target** | VMs the rule applies to | All instances, tag, service account |
| **Source/Dest Filter** | Traffic origin (ingress) or destination (egress) | IP range, tag, service account |
| **Protocols/Ports** | Layer 4 protocol + port | `tcp:80,443`, `icmp`, `all` |
| **Enforcement** | `ENABLED` or `DISABLED` | `ENABLED` |

---

### Ingress vs Egress

```
INGRESS rule: controls traffic coming INTO your VM
  Source filter options:  IP ranges  |  source tags  |  source service accounts

EGRESS rule: controls traffic going OUT of your VM
  Dest filter options:   IP ranges  |  dest tags  |  dest service accounts
```

---

### gcloud Firewall Rule Examples

```bash
# Allow SSH from a specific IP range
gcloud compute firewall-rules create allow-ssh \
  --network=my-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=203.0.113.0/24 \
  --target-tags=ssh-access

# Allow HTTP/HTTPS from anywhere to web servers
gcloud compute firewall-rules create allow-web \
  --network=my-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server

# Allow internal traffic between tagged VMs
gcloud compute firewall-rules create allow-internal \
  --network=my-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=all \
  --source-tags=backend \
  --target-tags=database

# Block egress to a specific IP range
gcloud compute firewall-rules create deny-egress-bad-ip \
  --network=my-vpc \
  --direction=EGRESS \
  --priority=500 \
  --action=DENY \
  --rules=all \
  --destination-ranges=198.51.100.0/24

# Disable a rule without deleting it
gcloud compute firewall-rules update allow-ssh \
  --disabled

# List all firewall rules sorted by priority
gcloud compute firewall-rules list \
  --sort-by=priority \
  --format="table(name,direction,priority,action,sourceRanges,targetTags)"

# Delete a firewall rule
gcloud compute firewall-rules delete allow-ssh
```

---

### Firewall Rule Priority Logic

```
Priority 0    ──────── Highest priority (evaluated first)
Priority 500
Priority 1000 ──────── Default
...
Priority 65534
Priority 65535 ─────── Lowest (implied rules live here)

First matching rule WINS — lower number = higher priority.
```

> ⚠️ **Gotcha:** Firewall rules are **not evaluated top-to-bottom** like traditional ACLs — the **lowest priority number wins**, not the first rule in a list. A `DENY` at priority 500 beats an `ALLOW` at priority 1000 for the same traffic.

> 💡 **Tip:** Use **service accounts** as firewall targets instead of network tags. Tags are manually assigned and easy to misconfigure; service accounts are cryptographically bound to VM identity.

---

### Hierarchical Firewall Policies

Applied at the **Organization** or **Folder** level — enforced before VPC-level rules.

| Feature | VPC Firewall Rules | Hierarchical Firewall Policy |
|---|---|---|
| Scope | Single VPC | Org / Folder / Project |
| Inheritance | No | Yes — cascades down |
| Actions | ALLOW / DENY | ALLOW / DENY / **GOTO_NEXT** |
| Management | Per-VPC | Centralized |
| Use case | Application-level control | Org-wide baselines |

```bash
# Create an org-level firewall policy
gcloud compute firewall-policies create \
  --organization=ORG_ID \
  --short-name=org-baseline-policy \
  --description="Organization baseline firewall rules"

# Add a rule to the policy
gcloud compute firewall-policies rules create 1000 \
  --firewall-policy=org-baseline-policy \
  --organization=ORG_ID \
  --direction=INGRESS \
  --action=allow \
  --layer4-configs=tcp:22 \
  --src-ip-ranges=10.0.0.0/8 \
  --description="Allow SSH from internal only"

# Associate policy with org
gcloud compute firewall-policies associations create \
  --firewall-policy=org-baseline-policy \
  --organization=ORG_ID
```

---

## 4. Routes & Routing

### Types of Routes

| Route Type | Created By | Purpose |
|---|---|---|
| **Subnet routes** | Auto-created per subnet | Route traffic within a subnet's CIDR range |
| **Default route** | Auto-created | Send traffic to the internet (`0.0.0.0/0`) via default internet gateway |
| **Custom static routes** | You | Direct traffic to specific CIDRs via a next-hop |
| **Dynamic routes** | Cloud Router (BGP) | Learned from on-prem or peer networks |
| **Peering routes** | VPC Peering | Routes imported/exported between peered VPCs |

---

### Next-Hop Options for Static Routes

| Next-Hop Type | Flag | Use Case |
|---|---|---|
| Default internet gateway | `--next-hop-gateway=default-internet-gateway` | Internet egress |
| VM instance | `--next-hop-instance=VM_NAME` | Route through a VM (e.g., NVA/firewall) |
| Internal IP | `--next-hop-address=IP` | Route through a specific IP |
| VPN tunnel | `--next-hop-vpn-tunnel=TUNNEL` | Route to on-prem via VPN |
| ILB (forwarding rule) | `--next-hop-ilb=RULE` | Route to an ILB (for NVA HA) |

```bash
# Create a VPC network
gcloud compute networks create my-vpc \
  --subnet-mode=custom

# Add a custom static route
gcloud compute routes create to-onprem \
  --network=my-vpc \
  --destination-range=192.168.1.0/24 \
  --next-hop-vpn-tunnel=my-vpn-tunnel \
  --next-hop-vpn-tunnel-region=us-central1 \
  --priority=1000

# Route specific tag traffic through an NVA VM
gcloud compute routes create nva-route \
  --network=my-vpc \
  --destination-range=10.5.0.0/16 \
  --next-hop-instance=nva-vm \
  --next-hop-instance-zone=us-central1-a \
  --tags=route-through-nva \
  --priority=900

# List all routes in a VPC
gcloud compute routes list \
  --filter="network=my-vpc"
```

---

### Dynamic Routing with Cloud Router

**Cloud Router** is a fully managed, BGP-based routing service that enables dynamic route exchange with on-prem or other clouds.

```
Cloud Router  ──── BGP Session ────  On-prem Router
     │                                      │
     └── Advertises VPC subnets             └── Advertises on-prem CIDRs
     └── Learns on-prem routes              └── Learns GCP routes
```

```bash
# Create a Cloud Router
gcloud compute routers create my-router \
  --network=my-vpc \
  --region=us-central1 \
  --asn=65001

# Add BGP interface to router (for VPN/Interconnect)
gcloud compute routers add-interface my-router \
  --interface-name=bgp-interface-1 \
  --vpn-tunnel=my-vpn-tunnel \
  --region=us-central1

# Add BGP peer
gcloud compute routers add-bgp-peer my-router \
  --peer-name=onprem-peer \
  --peer-asn=65002 \
  --interface=bgp-interface-1 \
  --region=us-central1

# View router status and learned routes
gcloud compute routers get-status my-router \
  --region=us-central1
```

---

### Routing Modes

| Mode | Behavior | Use Case |
|---|---|---|
| **Regional** (default) | Dynamic routes learned by Cloud Router apply only to the **same region** | Simple single-region hybrid setups |
| **Global** | Dynamic routes learned anywhere are propagated to **all regions** in the VPC | Multi-region hybrid; all regions need on-prem access |

```bash
# Set VPC to global dynamic routing mode
gcloud compute networks update my-vpc \
  --bgp-routing-mode=global

# Revert to regional
gcloud compute networks update my-vpc \
  --bgp-routing-mode=regional
```

> ⚠️ **Gotcha:** Global routing mode means a Cloud Router in `us-central1` will advertise routes to VMs in `europe-west1`. This can cause unexpected traffic patterns or expose routes you didn't intend to share cross-region.

---

## 5. VPC Peering & Shared VPC

### VPC Peering

Connects two VPC networks (same or different projects, same or different orgs) so their internal IPs can communicate **privately**, without traffic traversing the internet.

**How it works:**
- Both sides must create a peering connection — it's **not transitive** and requires explicit configuration on both ends
- Routes are exchanged between the two VPCs
- RFC 1918 IPs must **not overlap** between peered VPCs

```
VPC-A  ◄──── Peering ────►  VPC-B   ✅ Direct communication
VPC-A  ◄──── Peering ────►  VPC-C   ✅ Direct communication
VPC-B  ✗ cannot reach VPC-C via VPC-A (no transitivity)
```

```bash
# Step 1: From Project A — peer to Project B's VPC
gcloud compute networks peerings create peer-a-to-b \
  --network=vpc-a \
  --peer-project=project-b \
  --peer-network=vpc-b \
  --export-custom-routes \
  --import-custom-routes

# Step 2: From Project B — peer to Project A's VPC (must mirror)
gcloud compute networks peerings create peer-b-to-a \
  --network=vpc-b \
  --peer-project=project-a \
  --peer-network=vpc-a \
  --export-custom-routes \
  --import-custom-routes

# List peerings
gcloud compute networks peerings list --network=vpc-a

# Delete a peering
gcloud compute networks peerings delete peer-a-to-b \
  --network=vpc-a
```

---

### VPC Peering Limitations

| Limitation | Detail |
|---|---|
| **No transitivity** | VPC-A ↔ VPC-B ↔ VPC-C does NOT mean VPC-A ↔ VPC-C |
| **No overlapping CIDRs** | Peered VPCs cannot share IP ranges |
| **No tag/SA sharing** | Firewall source tags/SAs don't work across peerings |
| **Subnet route import limit** | Max 25 peerings per VPC, 15,000 imported subnets |
| **No transitive peering** | Cannot chain peerings for hub-spoke routing |

> 💡 **Tip:** For hub-and-spoke topologies with transitive routing, use **Network Connectivity Center** or route through an NVA (Network Virtual Appliance) rather than trying to chain peerings.

---

### Shared VPC

**Shared VPC** allows one **host project** to share its VPC networks with multiple **service projects**. Workloads in service projects can use subnets from the host VPC.

```
┌─────────────────────────────────────────┐
│          Host Project                   │
│   ┌─────────────────────────────────┐   │
│   │      Shared VPC Network         │   │
│   │  Subnet-1: 10.0.1.0/24         │   │
│   │  Subnet-2: 10.0.2.0/24         │   │
│   └───────────┬─────────────────────┘   │
└───────────────│─────────────────────────┘
                │ Subnet shared to ↓
    ┌───────────┼───────────┐
    ▼           ▼           ▼
[Service      [Service    [Service
 Project A]    Project B]  Project C]
 (Team: Web)  (Team: API) (Team: Data)
```

```bash
# Enable host project (requires Shared VPC Admin role)
gcloud compute shared-vpc enable HOST_PROJECT_ID

# Attach a service project to host
gcloud compute shared-vpc associated-projects add SERVICE_PROJECT_ID \
  --host-project=HOST_PROJECT_ID

# List associated service projects
gcloud compute shared-vpc associated-projects list \
  --host-project=HOST_PROJECT_ID

# Detach a service project
gcloud compute shared-vpc associated-projects remove SERVICE_PROJECT_ID \
  --host-project=HOST_PROJECT_ID
```

---

### VPC Peering vs Shared VPC

| | **VPC Peering** | **Shared VPC** |
|---|---|---|
| **Use case** | Connect independent VPCs | Centralized networking for multiple teams |
| **IP management** | Each VPC manages its own | Host project owns all IPs |
| **Transitive routing** | ❌ No | ✅ All service projects share one VPC |
| **Firewall control** | Each VPC independently | Centralized in host project |
| **Best for** | Cross-org / third-party integrations | Enterprise multi-team environments |
| **Admin overhead** | Higher (both sides must configure) | Lower (centralized host) |

---

## 6. Hybrid Connectivity

### Overview Comparison

| Option | Bandwidth | Latency | SLA | Setup Time | Cost |
|---|---|---|---|---|---|
| **Cloud VPN (Classic)** | Up to 3 Gbps/tunnel | Variable | 99.9% | Hours | Low |
| **Cloud VPN (HA)** | Up to 3 Gbps/tunnel | Variable | **99.99%** | Hours | Low-Med |
| **Partner Interconnect** | 50 Mbps–10 Gbps | Low | 99.9–99.99% | Days–Weeks | Medium |
| **Dedicated Interconnect** | 10 Gbps or 100 Gbps | Very Low | **99.99%** | Weeks | High |

---

### Cloud VPN

Encrypted IPsec tunnels over the public internet connecting your VPC to on-prem or another cloud.

**Classic VPN** — Single tunnel, single external IP, 99.9% SLA.
**HA VPN** — Two tunnels, two external IPs, **99.99% SLA** (requires BGP via Cloud Router).

```bash
# ---- HA VPN Setup ----

# 1. Create HA VPN gateway (GCP side)
gcloud compute vpn-gateways create ha-vpn-gw \
  --network=my-vpc \
  --region=us-central1

# 2. Create external (peer) VPN gateway
gcloud compute external-vpn-gateways create onprem-gw \
  --interfaces 0=203.0.113.1,1=203.0.113.2

# 3. Create VPN tunnels (one per interface)
gcloud compute vpn-tunnels create tunnel-1 \
  --vpn-gateway=ha-vpn-gw \
  --vpn-gateway-region=us-central1 \
  --peer-external-gateway=onprem-gw \
  --peer-external-gateway-interface=0 \
  --shared-secret=MY_SECRET \
  --ike-version=2 \
  --interface=0 \
  --router=my-router \
  --region=us-central1

gcloud compute vpn-tunnels create tunnel-2 \
  --vpn-gateway=ha-vpn-gw \
  --vpn-gateway-region=us-central1 \
  --peer-external-gateway=onprem-gw \
  --peer-external-gateway-interface=1 \
  --shared-secret=MY_SECRET \
  --ike-version=2 \
  --interface=1 \
  --router=my-router \
  --region=us-central1

# 4. List tunnel status
gcloud compute vpn-tunnels list \
  --filter="region=us-central1"
```

> ⚠️ **Gotcha:** Classic VPN only supports **policy-based** or **route-based** VPN and does NOT support BGP. For dynamic routing and 99.99% SLA, always use **HA VPN** with Cloud Router.

---

### Cloud Interconnect

Dedicated physical connection (or via a partner) between on-prem and Google's network — traffic bypasses the public internet.

```bash
# Request a Dedicated Interconnect (generates a Letter of Authorization)
gcloud compute interconnects create my-interconnect \
  --location=us-east4-zone1-1 \
  --link-type=LINK_TYPE_ETHERNET_10G_LR \
  --requested-link-count=2 \
  --admin-enabled

# Create a VLAN attachment (connects Interconnect to Cloud Router)
gcloud compute interconnects attachments dedicated create my-attachment \
  --interconnect=my-interconnect \
  --router=my-router \
  --region=us-east4 \
  --vlan=100 \
  --bandwidth=BPS_10G

# List interconnect attachments
gcloud compute interconnects attachments list
```

> 💡 **Tip:** For **Partner Interconnect**, work with a [supported service provider](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/service-providers). You create a VLAN attachment first, get a pairing key, and give it to your provider.

---

### Cloud Router Summary

| Feature | Detail |
|---|---|
| Protocol | BGP (eBGP or iBGP) |
| ASN range | 64512–65534 (private) or public ASN |
| Route advertisement | VPC subnets + custom IP ranges |
| Route learning | On-prem prefixes via BGP |
| Works with | HA VPN, Classic VPN, Interconnect |
| Scoped to | Region |

---

## 7. Private Google Access & Private Service Connect

### Private Google Access (PGA)

Allows VMs **without external IPs** to reach Google APIs and services (e.g., Cloud Storage, BigQuery) using internal IPs routed over Google's network.

```bash
# Enable PGA on a subnet
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-private-ip-google-access

# Verify PGA status
gcloud compute networks subnets describe my-subnet \
  --region=us-central1 \
  --format="get(privateIpGoogleAccess)"
```

> ⚠️ **Gotcha:** PGA does not apply to `googleapis.com` calls from on-prem via VPN/Interconnect. For that, you need **Private Google Access for on-premises hosts** with specific DNS and route configurations.

---

### Private Google Access — On-Premises

To use Google APIs privately from on-prem via VPN/Interconnect:
1. Create a route for `199.36.153.4/30` (restricted.googleapis.com) via VPN tunnel
2. Configure on-prem DNS to resolve `*.googleapis.com` to `199.36.153.4/30`

```bash
# Add route for restricted Google APIs
gcloud compute routes create restricted-google-apis \
  --network=my-vpc \
  --destination-range=199.36.153.4/30 \
  --next-hop-gateway=default-internet-gateway \
  --priority=100
```

---

### Private Service Connect (PSC)

Enables **private, IP-address-based access** to:
- Google APIs (`googleapis.com`)
- Managed services (Cloud SQL, Memorystore, etc.)
- Third-party services in other VPCs

```
Consumer VPC          Google Network / Producer VPC
    │                         │
 [PSC Endpoint]  ──────►  [Service Attachment]
 (private IP in             (published service)
  your subnet)
```

```bash
# Create a PSC endpoint for Google APIs
gcloud compute addresses create psc-endpoint-ip \
  --global \
  --purpose=PRIVATE_SERVICE_CONNECT \
  --addresses=10.0.10.100 \
  --network=my-vpc

gcloud compute forwarding-rules create psc-google-apis \
  --global \
  --network=my-vpc \
  --address=psc-endpoint-ip \
  --target-google-apis-bundle=all-apis

# Create PSC endpoint for a producer service
gcloud compute forwarding-rules create psc-to-producer \
  --region=us-central1 \
  --network=my-vpc \
  --address=psc-consumer-ip \
  --target-service-attachment=projects/PRODUCER_PROJECT/regions/us-central1/serviceAttachments/my-service
```

---

### Serverless VPC Access

Allows **Cloud Run**, **Cloud Functions**, and **App Engine** (standard) to access resources in your VPC using private IPs.

```bash
# Create a Serverless VPC Access connector
gcloud compute networks vpc-access connectors create my-connector \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.8.0.0/28

# Deploy Cloud Run with VPC connector
gcloud run deploy my-service \
  --image=gcr.io/my-project/my-image \
  --vpc-connector=my-connector \
  --vpc-egress=all-traffic \
  --region=us-central1
```

> 💡 **Tip:** For new deployments, prefer **Direct VPC Egress** over Serverless VPC Access connectors — it's simpler, lower latency, and doesn't require managing connector VMs.

---

## 8. Cloud NAT

### What is Cloud NAT?

**Cloud NAT** provides outbound internet access for VM instances that have **no external IP addresses**, without exposing them to inbound internet connections.

```
[VM: no external IP]  ──►  [Cloud NAT Gateway]  ──►  Internet
                                    │
                            [NAT IP Pool]
                         (static or ephemeral IPs)
```

**Key characteristics:**
- **Software-defined** — no NAT proxy VMs; runs on Google's network
- **Per-region, per-network** — one NAT gateway per region/VPC
- **Port allocation** — each VM gets a share of NAT ports (default: 64 ports per VM)
- **Stateful** — only allows **outbound-initiated** connections (inbound blocked)

---

### When to Use Cloud NAT

| Use Case | Use Cloud NAT? |
|---|---|
| VMs need internet access but no external IP | ✅ Yes |
| Download OS patches, packages from internet | ✅ Yes |
| Call external APIs from private VMs | ✅ Yes |
| Inbound connections from internet to VMs | ❌ No — use Load Balancer |
| GKE nodes pulling container images | ✅ Yes (if nodes have no external IP) |

---

### Cloud NAT Configuration Options

| Option | Description |
|---|---|
| **NAT IP allocation** | Manual (reserved static IPs) or Auto (ephemeral) |
| **Subnetwork scope** | All subnets in region, or specific subnets |
| **Min ports per VM** | Default 64; increase for apps with many connections |
| **Max ports per VM** | Default 65536 |
| **Enable logging** | Log all, errors only, or disable |
| **Endpoint types** | VM, GKE node/pod, serverless |

```bash
# Step 1: Create a Cloud Router (required for NAT)
gcloud compute routers create nat-router \
  --network=my-vpc \
  --region=us-central1

# Step 2: Create a Cloud NAT gateway
gcloud compute routers nats create my-nat \
  --router=nat-router \
  --region=us-central1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges \
  --enable-logging

# Use specific static IPs instead of auto-allocated
gcloud compute addresses create nat-ip-1 \
  --region=us-central1

gcloud compute routers nats create my-nat-static \
  --router=nat-router \
  --region=us-central1 \
  --nat-external-ip-pool=nat-ip-1 \
  --nat-all-subnet-ip-ranges

# Update NAT to increase ports per VM (for high-connection workloads)
gcloud compute routers nats update my-nat \
  --router=nat-router \
  --region=us-central1 \
  --min-ports-per-vm=128

# View NAT config
gcloud compute routers nats describe my-nat \
  --router=nat-router \
  --region=us-central1
```

> ⚠️ **Gotcha:** If your VMs exhaust NAT port allocations, new connections will fail with `RESOURCE_EXHAUSTION` errors in NAT logs. Monitor `nat/allocated_ports` and increase `--min-ports-per-vm` proactively.

> 💡 **Tip:** Enable Cloud NAT logging to Cloud Logging for outbound connection auditing:
> ```bash
> gcloud compute routers nats update my-nat \
>   --router=nat-router \
>   --region=us-central1 \
>   --log-filter=ERRORS_ONLY
> ```

---

## 9. VPC Flow Logs

### What Do Flow Logs Capture?

VPC Flow Logs record **network flow samples** (TCP/UDP metadata — not payload) for traffic to/from VM NICs. Each log record includes:

| Field | Description |
|---|---|
| `src_ip`, `dest_ip` | Source and destination IP addresses |
| `src_port`, `dest_port` | Source and destination ports |
| `protocol` | TCP, UDP, ICMP, etc. |
| `bytes_sent`, `packets_sent` | Volume metrics |
| `start_time`, `end_time` | Flow time window |
| `src_instance`, `dest_instance` | VM metadata (if within GCP) |
| `src_vpc`, `dest_vpc` | VPC metadata |
| `connection.reporter` | `SRC` or `DEST` |

> ⚠️ **Gotcha:** Flow logs are **sampled** — not every packet is recorded. Default sampling is 50% for most VM types. This is useful for analysis but not suitable for complete traffic auditing.

---

### Enabling VPC Flow Logs

```bash
# Enable flow logs on a subnet (default sampling = 0.5 = 50%)
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-flow-logs

# Enable with custom sampling rate (0.1 = 10%, 1.0 = 100%)
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-flow-logs \
  --logging-flow-sampling=0.5 \
  --logging-aggregation-interval=INTERVAL_5_SEC \
  --logging-metadata=INCLUDE_ALL_METADATA

# Disable flow logs
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --no-enable-flow-logs

# Check flow log status
gcloud compute networks subnets describe my-subnet \
  --region=us-central1 \
  --format="get(logConfig)"
```

---

### Aggregation Intervals

| Interval | Use Case |
|---|---|
| `INTERVAL_5_SEC` | High-fidelity analysis (more data, higher cost) |
| `INTERVAL_30_SEC` | Default — balanced |
| `INTERVAL_1_MIN` | Cost-optimized |
| `INTERVAL_5_MIN` | Long-term trend analysis |
| `INTERVAL_15_MIN` | Minimal cost |

---

### Querying Flow Logs in BigQuery

Flow logs are exported to Cloud Logging and can be routed to **BigQuery** via a log sink.

```bash
# Create a BigQuery log sink for flow logs
gcloud logging sinks create vpc-flow-sink \
  bigquery.googleapis.com/projects/MY_PROJECT/datasets/vpc_logs \
  --log-filter='resource.type="gce_subnetwork" AND log_id("compute.googleapis.com/vpc_flows")'
```

```sql
-- Top 10 source IPs by bytes sent in the last 24 hours
SELECT
  jsonPayload.connection.src_ip AS source_ip,
  SUM(CAST(jsonPayload.bytes_sent AS INT64)) AS total_bytes,
  SUM(CAST(jsonPayload.packets_sent AS INT64)) AS total_packets,
  COUNT(*) AS flow_count
FROM
  `my_project.vpc_logs.compute_googleapis_com_vpc_flows_*`
WHERE
  _TABLE_SUFFIX = FORMAT_DATE('%Y%m%d', CURRENT_DATE())
GROUP BY source_ip
ORDER BY total_bytes DESC
LIMIT 10;

-- Find all flows between two specific subnets
SELECT
  jsonPayload.connection.src_ip,
  jsonPayload.connection.dest_ip,
  jsonPayload.connection.dest_port,
  jsonPayload.connection.protocol,
  jsonPayload.bytes_sent,
  timestamp
FROM
  `my_project.vpc_logs.compute_googleapis_com_vpc_flows_*`
WHERE
  _TABLE_SUFFIX BETWEEN '20250101' AND '20250131'
  AND jsonPayload.src_vpc.subnetwork_name = 'subnet-a'
  AND jsonPayload.dest_vpc.subnetwork_name = 'subnet-b'
ORDER BY timestamp DESC
LIMIT 1000;

-- Detect potential port scanning (many dest ports from one src IP)
SELECT
  jsonPayload.connection.src_ip AS attacker_ip,
  COUNT(DISTINCT jsonPayload.connection.dest_port) AS unique_ports_scanned,
  COUNT(*) AS total_flows
FROM
  `my_project.vpc_logs.compute_googleapis_com_vpc_flows_*`
WHERE
  _TABLE_SUFFIX = FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  AND jsonPayload.connection.protocol = '6' -- TCP
GROUP BY attacker_ip
HAVING unique_ports_scanned > 50
ORDER BY unique_ports_scanned DESC;
```

---

## 10. Network Security Best Practices

### Firewall Hygiene

- **✅ Deny by default** — rely on the implied `deny-all-ingress` rule; only explicitly allow what's needed
- **✅ Use service accounts** as firewall targets rather than network tags for stronger identity binding
- **✅ Scope source ranges** — never use `0.0.0.0/0` for management ports (SSH/RDP/etc.) unless behind IAP
- **✅ Set specific ports** — avoid `--rules=all`; specify exact protocols and ports
- **✅ Audit regularly** — review rules quarterly; remove stale rules for decommissioned services

```bash
# Use Identity-Aware Proxy (IAP) for SSH instead of opening port 22
gcloud compute firewall-rules create allow-iap-ssh \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-service-accounts=my-vm-sa@project.iam.gserviceaccount.com
```

> 💡 **Tip:** `35.235.240.0/20` is Google's IAP CIDR range. Using IAP for SSH eliminates the need for public IPs or open SSH ports to the internet entirely.

---

### Network Segmentation

- **Separate environments** — use distinct VPCs (or at minimum, subnets with tight firewall rules) for dev/staging/prod
- **Micro-segmentation** — use service account-based firewall rules to enforce least-privilege between tiers (web → app → db)
- **Isolate sensitive workloads** — PCI, PHI, or regulated workloads should be in dedicated subnets or VPCs with strict egress

```
[Internet] → [Load Balancer] → [web-tier SA]
                                     │ (fw: allow tcp:8080 from web-tier → app-tier)
                               [app-tier SA]
                                     │ (fw: allow tcp:5432 from app-tier → db-tier only)
                               [db-tier SA]
```

---

### Private Infrastructure

- **✅ No external IPs on internal VMs** — use Cloud NAT for outbound, load balancers for inbound
- **✅ Enable Private Google Access** — so internal VMs can reach Google APIs without external IPs
- **✅ Use Private Service Connect** for accessing Google managed services privately
- **✅ Restrict metadata server access** — use `--metadata-from-file` and OS Login; avoid storing secrets in metadata

---

### Cloud Armor Basics

**Cloud Armor** is GCP's WAF and DDoS protection service, attached to **Global External HTTP(S) Load Balancers**.

```bash
# Create a Cloud Armor security policy
gcloud compute security-policies create my-armor-policy \
  --description="WAF policy for web app"

# Block traffic from a specific country (e.g., block all except US)
gcloud compute security-policies rules create 1000 \
  --security-policy=my-armor-policy \
  --expression="origin.region_code != 'US'" \
  --action=deny-403

# Enable OWASP Top 10 preconfigured WAF rules
gcloud compute security-policies rules create 2000 \
  --security-policy=my-armor-policy \
  --expression="evaluatePreconfiguredExpr('xss-v33-stable')" \
  --action=deny-403

# Attach policy to a backend service
gcloud compute backend-services update my-backend \
  --security-policy=my-armor-policy \
  --global
```

---

### Monitoring & Alerting

- Enable **VPC Flow Logs** on all production subnets
- Enable **Firewall Rules Logging** for critical DENY rules to detect blocked attacks
- Set up **Cloud Monitoring** alerts on:
  - Firewall deny rates spiking
  - NAT port exhaustion (`nat/allocated_ports` near limit)
  - Unexpected egress traffic volumes

```bash
# Enable logging on a specific firewall rule
gcloud compute firewall-rules update deny-all-ingress \
  --enable-logging \
  --logging-metadata=INCLUDE_ALL_METADATA
```

---

## 11. Key `gcloud` Commands Cheatsheet

### Networks

| Command | Description |
|---|---|
| `gcloud compute networks create NAME --subnet-mode=custom` | Create a custom mode VPC |
| `gcloud compute networks list` | List all VPCs in the project |
| `gcloud compute networks describe NAME` | Show VPC details |
| `gcloud compute networks update NAME --bgp-routing-mode=global` | Set global dynamic routing |
| `gcloud compute networks delete NAME` | Delete a VPC |

### Subnets

| Command | Description |
|---|---|
| `gcloud compute networks subnets create NAME --network=VPC --region=R --range=CIDR` | Create a subnet |
| `gcloud compute networks subnets list --filter="region=us-central1"` | List subnets in a region |
| `gcloud compute networks subnets describe NAME --region=R` | Show subnet details |
| `gcloud compute networks subnets update NAME --region=R --range=NEW_CIDR` | Expand subnet range |
| `gcloud compute networks subnets update NAME --region=R --enable-private-ip-google-access` | Enable PGA |
| `gcloud compute networks subnets update NAME --region=R --enable-flow-logs` | Enable Flow Logs |
| `gcloud compute networks subnets delete NAME --region=R` | Delete a subnet |

### Firewall Rules

| Command | Description |
|---|---|
| `gcloud compute firewall-rules create NAME --network=VPC --rules=tcp:80 --action=ALLOW` | Create allow rule |
| `gcloud compute firewall-rules list --filter="network=my-vpc"` | List rules for a VPC |
| `gcloud compute firewall-rules describe NAME` | Show rule details |
| `gcloud compute firewall-rules update NAME --priority=500` | Update rule priority |
| `gcloud compute firewall-rules update NAME --disabled` | Disable (not delete) rule |
| `gcloud compute firewall-rules delete NAME` | Delete a firewall rule |

### Routes

| Command | Description |
|---|---|
| `gcloud compute routes create NAME --network=VPC --destination-range=CIDR --next-hop-...` | Create static route |
| `gcloud compute routes list --filter="network=my-vpc"` | List all routes |
| `gcloud compute routes describe NAME` | Show route details |
| `gcloud compute routes delete NAME` | Delete a route |

### Cloud Routers & VPN

| Command | Description |
|---|---|
| `gcloud compute routers create NAME --network=VPC --region=R --asn=ASN` | Create Cloud Router |
| `gcloud compute routers list` | List all routers |
| `gcloud compute routers get-status NAME --region=R` | Show BGP sessions and learned routes |
| `gcloud compute vpn-gateways create NAME --network=VPC --region=R` | Create HA VPN gateway |
| `gcloud compute vpn-tunnels create NAME --vpn-gateway=GW --ike-version=2 ...` | Create VPN tunnel |
| `gcloud compute vpn-tunnels list` | List all VPN tunnels |
| `gcloud compute vpn-tunnels describe NAME --region=R` | Show tunnel details & status |

### NAT

| Command | Description |
|---|---|
| `gcloud compute routers nats create NAME --router=R --region=REG --auto-allocate-nat-external-ips --nat-all-subnet-ip-ranges` | Create NAT gateway |
| `gcloud compute routers nats list --router=R --region=REG` | List NAT configs |
| `gcloud compute routers nats describe NAME --router=R --region=REG` | Show NAT details |
| `gcloud compute routers nats update NAME --router=R --region=REG --min-ports-per-vm=128` | Update NAT config |
| `gcloud compute routers nats delete NAME --router=R --region=REG` | Delete NAT |

---

## 12. Limits & Quotas

> 💡 **Tip:** Most of these limits can be increased by submitting a quota increase request in the GCP Console. Some limits are **hard limits** that cannot be increased.

### Network & Subnet Limits

| Resource | Default Limit | Hard Limit? |
|---|---|---|
| VPC networks per project | 15 | No (quota increase available) |
| Subnets per VPC | 256 | No |
| Secondary IP ranges per subnet | 30 | Yes |
| IP addresses per subnet | Up to `/8` (depends on CIDR) | N/A |
| Alias IP ranges per VM NIC | 10 | Yes |
| NICs per VM instance | 8 | Yes |

### Firewall & Routes

| Resource | Default Limit | Hard Limit? |
|---|---|---|
| Firewall rules per VPC | 300 | No |
| Routes per VPC | 250 | No |
| Firewall policy rules per policy | 256 | No |
| Hierarchical firewall policies per org | 10 | No |
| Hierarchical policy rules per org | 256 | No |

### Peering & Connectivity

| Resource | Default Limit | Hard Limit? |
|---|---|---|
| VPC peering connections per VPC | 25 | No |
| Imported subnet routes per VPC | 15,000 | No |
| Shared VPC service projects per host | 100 | No |
| HA VPN tunnels per gateway | 8 | Yes |
| Cloud Routers per region per VPC | 3 | No |
| Dedicated Interconnect connections per region | 8 | No |

### NAT & Flow Logs

| Resource | Default Limit | Hard Limit? |
|---|---|---|
| Cloud NAT gateways per region per VPC | 3 | No |
| NAT IP addresses per NAT gateway | 65,000 ports × IPs ÷ VMs | No |
| Max ports per VM (NAT) | 65,536 | Yes |
| Flow log record size (max) | 500 bytes metadata | N/A |

---

### Checking & Requesting Quota Increases

```bash
# Check current quotas for networking
gcloud compute project-info describe \
  --format="table(quotas.metric, quotas.limit, quotas.usage)" \
  | grep -i "network\|firewall\|route\|subnet"

# Check regional quotas
gcloud compute regions describe us-central1 \
  --format="table(quotas.metric, quotas.limit, quotas.usage)"
```

> 💡 **Tip:** Request quota increases at **IAM & Admin → Quotas** in the GCP Console, or use the `gcloud alpha services quota` command. Plan for 2–5 business days for approval on large increases.

---

*© GCP VPC Cheatsheet — For the latest limits and feature updates, always refer to the [official GCP VPC documentation](https://cloud.google.com/vpc/docs).*
