# GCP Cloud DNS — Comprehensive Cheatsheet

> **TL;DR — 5 Most Important Cloud DNS Decisions**
> - **Public zone** for internet-facing domains; **Private zone** for internal VPC resolution — never mix them up.
> - Always **delegate NS records at your registrar** to Cloud DNS nameservers before expecting resolution to work.
> - Set **short TTLs (60–300s) before migrations**; restore long TTLs (3600s+) once stable — TTLs are your blast radius control.
> - **CNAME cannot coexist at the zone apex** (`example.com`) — use A/AAAA records there; CNAME only on subdomains.
> - For hybrid DNS: use **forwarding zones** (Cloud → on-prem) and **inbound DNS policies** (on-prem → Cloud); never route all DNS through a single chokepoint.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Core Concepts](#2-core-concepts)
3. [Zone Types — Deep Dive](#3-zone-types--deep-dive)
4. [DNS Record Management](#4-dns-record-management)
5. [gcloud CLI Cheatsheet](#5-gcloud-cli-cheatsheet)
6. [Terraform Examples](#6-terraform-examples)
7. [DNSSEC](#7-dnssec)
8. [Private DNS & Hybrid Architectures](#8-private-dns--hybrid-architectures)
9. [Observability & Troubleshooting](#9-observability--troubleshooting)
10. [Key Limits & Quotas](#10-key-limits--quotas)
11. [Gotchas & Best Practices](#11-gotchas--best-practices)

---

## 1. Overview

**GCP Cloud DNS** is a scalable, reliable, fully managed authoritative DNS service running on Google's global infrastructure. It supports both public DNS (internet-facing) and private DNS (VPC-internal) with a **100% uptime SLA** backed by anycast routing across Google's worldwide PoPs.

**How it works (high level):**
- You create **managed zones** mapped to domain names (e.g., `example.com.`)
- You populate zones with **resource record sets** (A, CNAME, MX, TXT, etc.)
- Cloud DNS serves authoritative answers from **anycast nameservers** (`ns-cloud-*.googledomains.com`)
- For private zones, resolution is available only to **bound VPC networks**

**Key Benefits:**
- **100% availability SLA** — no single point of failure; globally distributed
- **Anycast routing** — queries answered at the nearest Google PoP, typically <10ms
- **No infrastructure to manage** — no BIND configs, no zone transfers to maintain
- **Atomic record updates** — changes are transactional and propagate globally in seconds
- **DNSSEC support** — managed signing with automatic key rotation
- **Hybrid DNS** — forwarding, peering, and inbound policies for on-prem integration

---

### Cloud DNS vs. Self-Managed DNS

| Dimension | Cloud DNS | Self-Managed (BIND/NSD) |
|---|---|---|
| **Availability SLA** | 100% | Depends on your HA setup |
| **Query latency** | <10ms (anycast global PoPs) | Depends on topology |
| **Scalability** | Unlimited (auto-scaled) | Manual capacity planning |
| **DNSSEC** | Managed, auto key rotation | Manual key management |
| **Propagation time** | Seconds (atomic global push) | Zone transfer dependent |
| **DDoS resilience** | Built-in (Google-scale) | Requires separate mitigation |
| **Management overhead** | Low — API/CLI/Terraform | High — config files, patches |
| **Cost model** | Per-zone + per-query | Infrastructure + ops cost |
| **Private DNS** | Native VPC integration | Requires split-view BIND |
| **Hybrid support** | Forwarding + peering built-in | Manual conditional forwarding |

---

## 2. Core Concepts

### DNS Resolution Flow

```
Client (VM, browser, app)
        │
        │  1. Query: what is api.example.com?
        ▼
┌───────────────────┐
│  Recursive        │  2. Check local cache → miss
│  Resolver         │  3. Ask root (.) → .com NS → example.com NS
│  (169.254.169.254 │
│   in GCP VMs)     │
└───────────────────┘
        │
        │  4. Query Cloud DNS authoritative nameservers
        ▼
┌───────────────────────────────────────┐
│  Cloud DNS Authoritative Nameservers  │
│  ns-cloud-a1.googledomains.com        │
│  ns-cloud-b1.googledomains.com        │
│  ns-cloud-c1.googledomains.com        │
│  ns-cloud-d1.googledomains.com        │
└───────────────────────────────────────┘
        │
        │  5. Answer: api.example.com → 34.120.1.5 (TTL 300)
        ▼
Client receives answer → caches for TTL duration
```

For **private zones**, GCP VMs use the metadata server (`169.254.169.254`) as their recursive resolver, which consults Cloud DNS private zones before forwarding to public DNS.

---

### Public vs. Private Zones

| Feature | Public Zone | Private Zone |
|---|---|---|
| **Visibility** | Entire internet | Only bound VPC networks |
| **Purpose** | Internet DNS (website, email, APIs) | Internal service discovery |
| **Resolver** | Any recursive resolver worldwide | GCP metadata server (VMs in bound VPC) |
| **DNSSEC** | Supported | Not supported |
| **Forwarding** | Not applicable | Supported (to on-prem) |
| **Peering** | Not applicable | Supported (across VPCs) |
| **Example use** | `api.example.com → 34.x.x.x` | `db.internal → 10.0.0.5` |

---

### Managed Zones

A **managed zone** is the container for DNS records for a specific domain suffix.

- Each managed zone maps to exactly **one DNS name** (e.g., `example.com.` — note the trailing dot)
- A project can have **multiple managed zones** (one per domain or subdomain)
- Cloud DNS assigns **4 nameservers** per managed zone (from the `ns-cloud-*.googledomains.com` pool)
- The zone is **authoritative** — it's the source of truth for records within that domain

```
Managed Zone: "example-com-zone"
  DNS Name:   example.com.
  Type:       Public
  NS servers: ns-cloud-a1.googledomains.com.
              ns-cloud-b1.googledomains.com.
              ns-cloud-c1.googledomains.com.
              ns-cloud-d1.googledomains.com.

  Records:
    example.com.       A      34.120.1.5
    www.example.com.   CNAME  example.com.
    mail.example.com.  MX     10 smtp.example.com.
    example.com.       TXT    "v=spf1 include:_spf.google.com ~all"
```

---

### DNS Record Types

| Type | Full Name | Purpose | Example Value |
|---|---|---|---|
| **A** | Address | IPv4 address for a hostname | `34.120.1.5` |
| **AAAA** | IPv6 Address | IPv6 address for a hostname | `2001:db8::1` |
| **CNAME** | Canonical Name | Alias pointing to another hostname | `example.com.` |
| **MX** | Mail Exchange | Mail server for a domain | `10 smtp.example.com.` |
| **TXT** | Text | Arbitrary text; SPF, DKIM, domain verification | `"v=spf1 include:_spf.google.com ~all"` |
| **NS** | Name Server | Authoritative nameservers for a zone | `ns-cloud-a1.googledomains.com.` |
| **SOA** | Start of Authority | Zone metadata (serial, refresh, retry, expire) | `ns-cloud-a1... hostmaster... 1 21600 3600 259200 300` |
| **PTR** | Pointer | Reverse DNS — IP → hostname | `api.example.com.` |
| **SRV** | Service | Service location (host, port, priority) | `10 20 443 api.example.com.` |
| **CAA** | Certification Authority Authorization | Which CAs may issue TLS certs | `0 issue "letsencrypt.org"` |
| **DNSKEY** | DNS Key | DNSSEC public key (auto-managed) | `256 3 8 AwEAAb...` |
| **DS** | Delegation Signer | DNSSEC chain-of-trust at parent zone | `12345 8 2 ABC123...` |
| **IPSECKEY** | IPsec Key | IPsec public key for a host | `10 1 2 192.0.2.1 AQNRU...` |
| **NAPTR** | Naming Authority Pointer | Regex-based rewriting (VoIP/ENUM) | `100 10 "u" "E2U+sip" "!^.*$!sip:info@example.com!" .` |
| **SPF** | Sender Policy Framework | (Legacy) Same as TXT SPF record | `"v=spf1 mx ~all"` |
| **SSHFP** | SSH Fingerprint | SSH host key fingerprint for DANE | `1 1 AABBCCDDEEFF...` |
| **TLSA** | TLS Authentication | DANE — binds TLS cert to DNS | `3 1 1 AABBCC...` |

---

### TTL (Time-To-Live)

- **TTL** is the number of seconds resolvers and clients may cache a DNS answer.
- Set per resource record set — not globally per zone.
- Lower TTL = faster propagation of changes, but more queries to Cloud DNS (higher cost).
- Higher TTL = fewer queries, lower cost, but changes take longer to propagate globally.

**TTL Strategy Guide:**

| Scenario | Recommended TTL |
|---|---|
| Stable production record (rarely changes) | 3600–86400s (1–24 hours) |
| Normal operational records | 300–3600s (5 min – 1 hour) |
| Pre-migration (48h before cutover) | 60–300s (1–5 minutes) |
| Active migration / failover | 30–60s |
| Post-migration (restore after 24h) | 3600s+ |
| Internal service discovery (private zone) | 30–300s |

> **Propagation math**: After lowering TTL to 60s and waiting 1 TTL cycle (60s), ALL cached copies of the old TTL have expired. You can now safely change the record and have 60s or less before all resolvers see the new value.

---

### DNSSEC (Overview)

**DNSSEC** (DNS Security Extensions) adds cryptographic signatures to DNS records, allowing resolvers to verify that responses are authentic and unmodified.

- **When to enable**: For public zones where record integrity is critical (financial, healthcare, government).
- **Cloud DNS handles**: Key generation, signing, key rotation — automatically.
- **You must**: Register the **DS record** with your domain registrar to complete the chain of trust.
- **Does not**: Encrypt DNS queries (use DNS-over-HTTPS/TLS for that).

> Full DNSSEC walkthrough in [Section 7](#7-dnssec).

---

### DNS Peering

**DNS peering** allows a VPC network to resolve private zone records from another VPC's managed zones — without the consumer VPC owning those zones.

```
VPC A (producer) — owns private zone "internal.example.com."
VPC B (consumer) — peered to VPC A's DNS
  → VM in VPC B can resolve "db.internal.example.com." via VPC A's zone
```

- **Direction**: Peering is one-directional (consumer queries producer's zones)
- **Use case**: Shared VPC, hub-and-spoke, cross-project service resolution
- **Not the same as VPC Network Peering** — DNS peering is a separate Cloud DNS concept

---

### Forwarding Zones

A **forwarding zone** delegates DNS resolution for a specific domain to external resolvers (on-premises DNS servers or third-party resolvers) instead of using Cloud DNS records.

```
VM in GCP queries: server.corp.internal.
                        │
                        ▼
         Cloud DNS: forwarding zone "corp.internal."
                        │
                        ▼
          On-prem resolver: 192.168.1.53  ←── answers from on-prem DNS
```

- **Outbound forwarding**: GCP → on-prem resolver
- **Inbound forwarding**: on-prem → Cloud DNS (via forwarding IP on GCP)
- Forwarding targets can be internal (RFC1918) or external IPs

---

### Split-Horizon DNS

**Split-horizon** (split-view) serves **different DNS answers** for the same domain name depending on whether the client is internal (VPC) or external (internet).

```
External client queries api.example.com → 34.120.1.5   (public IP, via Public zone)
Internal VM  queries api.example.com → 10.0.0.10        (private IP, via Private zone)
```

**How to implement in Cloud DNS:**
- Create a **public zone** for `example.com.` with external IP records
- Create a **private zone** for `example.com.` bound to the VPC with internal IP records
- Private zone takes precedence for VMs in bound VPCs — no overlap conflict

---

### DNS Policies

**DNS Server Policy** — applied to a VPC network; controls:
- Inbound DNS forwarding (creates a forwarding target IP in the VPC)
- Alternative nameservers (override default resolver for the VPC)
- Logging (enable query logging per VPC)

**Response Policy (RPZ-like)** — a managed zone type that overrides DNS responses:
- Block domains (return NXDOMAIN)
- Override records (return custom IP instead of real answer)
- Used for security (block malware C2 domains), compliance, or internal overrides

---

## 3. Zone Types — Deep Dive

### Public Zone

- **Definition**: Authoritative zone visible to the entire internet
- **Scope**: Global (anycast, no VPC binding)
- **Primary use case**: Hosting DNS for internet-facing domains (`example.com`, `api.myapp.io`)
- **Key config**: Requires NS delegation at domain registrar; supports DNSSEC
- **Choose when**: Serving websites, APIs, email (MX), or any publicly accessible service
- **Not for**: Internal IPs or private service discovery (use Private zone)

```bash
gcloud dns managed-zones create example-com \
  --dns-name="example.com." \
  --description="Public zone for example.com" \
  --visibility=public
```

---

### Private Zone

- **Definition**: Zone visible only to explicitly bound VPC networks
- **Scope**: Per-VPC (must specify one or more VPC network bindings)
- **Primary use case**: Internal service discovery (`db.internal`, `api.svc.cluster.local`)
- **Key config**: VPC network binding; supports forwarding and peering
- **Choose when**: Resolving internal hostnames, microservices, GKE internal DNS
- **Supports**: Overriding public domains for internal routing (split-horizon)

```bash
gcloud dns managed-zones create internal-zone \
  --dns-name="internal.example.com." \
  --description="Private zone for internal services" \
  --visibility=private \
  --networks=my-vpc
```

---

### Forwarding Zone

- **Definition**: Private zone that delegates resolution to specified external DNS servers
- **Scope**: VPC-specific (private visibility)
- **Primary use case**: Resolving on-prem domain names (`corp.internal.`, `ad.company.com.`) from GCP
- **Key config**: One or more forwarding targets (IP addresses of on-prem DNS servers)
- **Choose when**: Hybrid connectivity (Cloud VPN / Interconnect) and on-prem DNS exists
- **Routing**: Uses private networking path (not public internet) for RFC1918 targets

```bash
gcloud dns managed-zones create corp-forwarding \
  --dns-name="corp.internal." \
  --visibility=private \
  --networks=my-vpc \
  --forwarding-targets=192.168.1.53,192.168.1.54
```

---

### Peering Zone

- **Definition**: Private zone that resolves records from another VPC's managed zones
- **Scope**: VPC-specific; consumer VPC reads producer VPC's zones
- **Primary use case**: Hub-and-spoke; shared services VPC hosting internal DNS
- **Key config**: Target network (the VPC whose zones to peer with)
- **Choose when**: Multiple VPCs need to resolve records owned by one central VPC
- **One-directional**: Consumer can query producer; producer cannot automatically query consumer

```bash
gcloud dns managed-zones create peered-zone \
  --dns-name="shared.internal." \
  --visibility=private \
  --networks=spoke-vpc \
  --target-network=hub-vpc
```

---

### Reverse Lookup Zone (Manual)

- **Definition**: Maps IP addresses to hostnames (PTR records); manually managed
- **Scope**: Public or private
- **DNS name format**: Reverse octets + `.in-addr.arpa.` (IPv4) or `.ip6.arpa.` (IPv6)
- **Primary use case**: Email server rDNS, security auditing, network diagnostics
- **Choose when**: You control the IP range and need custom PTR records

```text
Zone DNS name for 10.0.0.0/24: 0.0.10.in-addr.arpa.
PTR record: 5.0.0.10.in-addr.arpa. → api.example.com.
```

---

### Managed Reverse Lookup Zone

- **Definition**: Cloud DNS automatically creates PTR records for GCP resources in a VPC
- **Scope**: Private, VPC-specific
- **Primary use case**: Automatic rDNS for GCP VM internal IPs
- **Choose when**: You want reverse DNS without manually maintaining PTR records
- **Limitation**: Only covers GCP-assigned internal IPs, not manually assigned ranges

```bash
gcloud dns managed-zones create reverse-zone \
  --dns-name="0.0.10.in-addr.arpa." \
  --visibility=private \
  --networks=my-vpc \
  --managed-reverse-lookup
```

---

### Response Policy Zone (RPZ)

- **Definition**: A special zone that overrides DNS responses matching defined rules
- **Scope**: Private (VPC-bound)
- **Primary use case**: Block malicious domains, override external records for internal routing
- **Behavior options**: `NXDOMAIN` (block), `NODATA`, or custom records (redirect)
- **Choose when**: DNS-layer security filtering, compliance domain blocking

---

## 4. DNS Record Management

### Cloud DNS Data Model

```
Project
└── Managed Zone (one per domain suffix)
    └── Resource Record Sets (RRset) — one per name + type combination
        └── Multiple RData values (e.g., multiple A records for one hostname)
```

- A **resource record set (RRset)** is the atomic unit: `name + type + TTL + [values]`
- You cannot have two RRsets with the same name and type — updating means **replacing** the entire RRset
- All values in an RRset share the same TTL

---

### Creating and Updating Records

**Simple record add (via transaction):**

```bash
# Start a transaction on the zone
gcloud dns record-sets transaction start --zone=example-com

# Add an A record
gcloud dns record-sets transaction add "34.120.1.5" \
  --name="api.example.com." \
  --ttl=300 \
  --type=A \
  --zone=example-com

# Execute (commit) the transaction atomically
gcloud dns record-sets transaction execute --zone=example-com
```

**Update an existing record (replace the RRset):**

```bash
gcloud dns record-sets transaction start --zone=example-com

# Remove the old value
gcloud dns record-sets transaction remove "34.120.1.5" \
  --name="api.example.com." \
  --ttl=300 \
  --type=A \
  --zone=example-com

# Add the new value
gcloud dns record-sets transaction add "34.120.2.10" \
  --name="api.example.com." \
  --ttl=300 \
  --type=A \
  --zone=example-com

gcloud dns record-sets transaction execute --zone=example-com
```

---

### Batch Changes via Transaction Files

Transactions are stored as local YAML files before execution — inspect before committing:

```bash
# Start transaction (creates transaction.yaml in current directory)
gcloud dns record-sets transaction start --zone=example-com

# Add multiple records
gcloud dns record-sets transaction add "34.120.1.5" \
  --name="www.example.com." --ttl=300 --type=A --zone=example-com

gcloud dns record-sets transaction add "34.120.1.5" \
  --name="api.example.com." --ttl=300 --type=A --zone=example-com

gcloud dns record-sets transaction add \
  "10 smtp.example.com." \
  --name="example.com." --ttl=3600 --type=MX --zone=example-com

# Inspect the pending changes before committing
cat transaction.yaml

# Execute all changes atomically
gcloud dns record-sets transaction execute --zone=example-com

# OR abort if something looks wrong
gcloud dns record-sets transaction abort --zone=example-com
```

**Example `transaction.yaml`:**
```yaml
---
additions:
- kind: dns#resourceRecordSet
  name: www.example.com.
  rrdatas:
  - 34.120.1.5
  ttl: 300
  type: A
- kind: dns#resourceRecordSet
  name: api.example.com.
  rrdatas:
  - 34.120.1.5
  ttl: 300
  type: A
deletions: []
```

---

### Import and Export (BIND Zone File Format)

```bash
# Export zone to BIND format
gcloud dns record-sets export zone-export.txt \
  --zone=example-com \
  --zone-file-format          # Output as BIND zone file

# Import from BIND zone file
gcloud dns record-sets import zone-import.txt \
  --zone=example-com \
  --zone-file-format \
  --delete-all-existing       # Replace all records (use with caution)

# Import without --delete-all-existing = merge (add new, keep existing)
gcloud dns record-sets import additions.txt \
  --zone=example-com \
  --zone-file-format
```

**Example BIND zone file:**
```text
$ORIGIN example.com.
$TTL 300
@         IN  A     34.120.1.5
www       IN  CNAME example.com.
mail      IN  A     34.120.1.10
@         IN  MX    10 mail.example.com.
@         IN  TXT   "v=spf1 include:_spf.google.com ~all"
```

---

### Wildcard Records

Wildcard records match any subdomain not explicitly defined:

```bash
# Add wildcard A record (matches *.example.com)
gcloud dns record-sets transaction start --zone=example-com
gcloud dns record-sets transaction add "34.120.1.5" \
  --name="*.example.com." \
  --ttl=300 \
  --type=A \
  --zone=example-com
gcloud dns record-sets transaction execute --zone=example-com
```

**Wildcard matching rules:**
- `*.example.com.` matches `foo.example.com.` but NOT `example.com.` or `a.b.example.com.`
- Explicit records take precedence over wildcards
- Wildcards work only one label deep (not multi-level)

---

### Alias Records / Zone Apex CNAME Problem

Cloud DNS does **not** support ALIAS or ANAME record types (unlike Route 53 or NS1). The zone apex (`example.com.` itself) cannot be a CNAME per DNS spec.

**Workarounds:**
- Use an **A record** with your load balancer's static IP at the apex
- Use a **CNAME at a subdomain** (`www.example.com.` → `example.com.`)
- Use **Cloud Load Balancer** with a reserved global IP → point A record to that IP

```bash
# Zone apex — must use A record, not CNAME
gcloud dns record-sets transaction add "34.120.1.5" \
  --name="example.com." --ttl=300 --type=A --zone=example-com

# Subdomain — CNAME is fine
gcloud dns record-sets transaction add "example.com." \
  --name="www.example.com." --ttl=300 --type=CNAME --zone=example-com
```

---

## 5. gcloud CLI Cheatsheet

### Zone Management

```bash
# ── CREATE ZONES ──────────────────────────────────────────

# Create public zone
gcloud dns managed-zones create example-com \
  --dns-name="example.com." \            # Trailing dot required
  --description="Public zone for example.com" \
  --visibility=public                    # Visible to all internet resolvers

# Create private zone (bound to a VPC)
gcloud dns managed-zones create internal-zone \
  --dns-name="internal.example.com." \
  --visibility=private \
  --networks=my-vpc \                    # Comma-separated VPC names
  --description="Internal service discovery"

# Create forwarding zone (private, to on-prem resolver)
gcloud dns managed-zones create corp-fwd \
  --dns-name="corp.internal." \
  --visibility=private \
  --networks=my-vpc \
  --forwarding-targets=192.168.1.53 \    # On-prem DNS server IP
  --description="Forward corp.internal to on-prem DNS"

# Create peering zone (consumer queries producer VPC's zones)
gcloud dns managed-zones create peered-zone \
  --dns-name="shared.internal." \
  --visibility=private \
  --networks=spoke-vpc \                 # Consumer VPC
  --target-network=hub-vpc \            # Producer VPC owning the zone
  --description="Peer to hub VPC DNS"

# ── LIST & DESCRIBE ───────────────────────────────────────

# List all managed zones in project
gcloud dns managed-zones list

# Describe a specific zone (shows NS servers, config, DNSSEC state)
gcloud dns managed-zones describe example-com

# ── DELETE ZONES ──────────────────────────────────────────

# Delete a managed zone (zone must be empty — delete records first)
gcloud dns managed-zones delete example-com
```

---

### Record Set Management

```bash
# ── ADD RECORDS (via transaction) ─────────────────────────

gcloud dns record-sets transaction start --zone=example-com

# A record
gcloud dns record-sets transaction add "34.120.1.5" \
  --name="api.example.com." \
  --type=A --ttl=300 --zone=example-com

# AAAA record
gcloud dns record-sets transaction add "2001:db8::1" \
  --name="api.example.com." \
  --type=AAAA --ttl=300 --zone=example-com

# CNAME record
gcloud dns record-sets transaction add "api.example.com." \
  --name="gateway.example.com." \
  --type=CNAME --ttl=300 --zone=example-com

# MX record (priority + server)
gcloud dns record-sets transaction add "10 smtp.example.com." \
  --name="example.com." \
  --type=MX --ttl=3600 --zone=example-com

# Multiple MX records (space-separated)
gcloud dns record-sets transaction add \
  "10 smtp1.example.com." "20 smtp2.example.com." \
  --name="example.com." \
  --type=MX --ttl=3600 --zone=example-com

# TXT record (wrap in double quotes, escape inner quotes)
gcloud dns record-sets transaction add \
  '"v=spf1 include:_spf.google.com ~all"' \
  --name="example.com." \
  --type=TXT --ttl=3600 --zone=example-com

# SRV record (priority weight port target)
gcloud dns record-sets transaction add "10 20 443 api.example.com." \
  --name="_https._tcp.example.com." \
  --type=SRV --ttl=300 --zone=example-com

# CAA record
gcloud dns record-sets transaction add '0 issue "letsencrypt.org"' \
  --name="example.com." \
  --type=CAA --ttl=3600 --zone=example-com

# PTR record (reverse DNS)
gcloud dns record-sets transaction add "api.example.com." \
  --name="5.1.120.34.in-addr.arpa." \
  --type=PTR --ttl=300 --zone=reverse-zone

gcloud dns record-sets transaction execute --zone=example-com

# ── UPDATE RECORDS ────────────────────────────────────────

# One-step update using record-sets update (replaces entire RRset)
gcloud dns record-sets update "api.example.com." \
  --name="www.example.com." \
  --type=CNAME \
  --ttl=300 \
  --zone=example-com

# ── DELETE RECORDS ────────────────────────────────────────

gcloud dns record-sets transaction start --zone=example-com
gcloud dns record-sets transaction remove "34.120.1.5" \
  --name="api.example.com." \
  --type=A --ttl=300 --zone=example-com
gcloud dns record-sets transaction execute --zone=example-com

# Direct delete (no transaction needed for single delete)
gcloud dns record-sets delete api.example.com. \
  --type=A \
  --zone=example-com

# ── LIST RECORDS ──────────────────────────────────────────

# List all record sets in a zone
gcloud dns record-sets list --zone=example-com

# Filter by record type
gcloud dns record-sets list --zone=example-com \
  --filter="type=MX" \
  --format="table(name,type,ttl,rrdatas)"

# List with pagination (large zones)
gcloud dns record-sets list --zone=example-com \
  --page-size=100
```

---

### DNSSEC Commands

```bash
# Enable DNSSEC on a public zone
gcloud dns managed-zones update example-com \
  --dnssec-state=on                      # Activates key generation and signing

# Check DNSSEC status and retrieve DS record
gcloud dns managed-zones describe example-com \
  --format="yaml(dnssecConfig,nameServers)"

# Get DS record to register at your domain registrar
gcloud dns dns-keys list --zone=example-com \
  --filter="type=keySigning" \           # Only KSK (used for DS record at registrar)
  --format="table(id,algorithm,keyLength,type,dsRecord)"

# Describe a specific DNSKEY
gcloud dns dns-keys describe KEY_ID --zone=example-com

# Disable DNSSEC (WARNING: causes validation failures if DS still registered)
gcloud dns managed-zones update example-com \
  --dnssec-state=off
```

---

### DNS Policies (Inbound Forwarding)

```bash
# Create a DNS policy enabling inbound DNS forwarding for a VPC
gcloud dns policies create inbound-policy \
  --networks=my-vpc \                    # Apply to this VPC
  --enable-inbound-forwarding \          # Creates a forwarding IP in the VPC
  --description="Allow on-prem to query Cloud DNS private zones"

# Create policy with alternative nameservers (override default resolver)
gcloud dns policies create alt-ns-policy \
  --networks=my-vpc \
  --alternative-name-servers=8.8.8.8,1.1.1.1 \  # Use public DNS for external queries
  --enable-logging \                     # Enable query logging for this VPC
  --description="Custom resolver + logging policy"

# List DNS policies
gcloud dns policies list

# Delete DNS policy
gcloud dns policies delete inbound-policy
```

---

### Import/Export & Utility

```bash
# Export zone to BIND format
gcloud dns record-sets export my-zone-export.txt \
  --zone=example-com \
  --zone-file-format

# Import zone from BIND file
gcloud dns record-sets import my-zone.txt \
  --zone=example-com \
  --zone-file-format

# Check NS servers assigned to your zone (needed for registrar delegation)
gcloud dns managed-zones describe example-com \
  --format="value(nameServers)"

# Verify a record is resolving correctly (from Cloud Shell or a GCP VM)
dig api.example.com @ns-cloud-a1.googledomains.com

# Get DNSSEC DS record in registrar-ready format
gcloud dns dns-keys list \
  --zone=example-com \
  --filter="type=keySigning" \
  --format="value(dsRecord)"
```

---

## 6. Terraform Examples

### Public Zone with A, CNAME, MX, TXT Records

```hcl
# ── Public managed zone ────────────────────────────────────
resource "google_dns_managed_zone" "public" {
  name        = "example-com-zone"
  dns_name    = "example.com."           # Trailing dot required
  description = "Public zone for example.com"
  visibility  = "public"

  # Optional: enable Cloud Logging for queries
  # (requires DNS policy — see logging section)
}

# ── A record (zone apex) ──────────────────────────────────
resource "google_dns_record_set" "apex_a" {
  name         = google_dns_managed_zone.public.dns_name  # "example.com."
  type         = "A"
  ttl          = 300
  managed_zone = google_dns_managed_zone.public.name
  rrdatas      = ["34.120.1.5"]
}

# ── A record (subdomain) ──────────────────────────────────
resource "google_dns_record_set" "api_a" {
  name         = "api.${google_dns_managed_zone.public.dns_name}"
  type         = "A"
  ttl          = 300
  managed_zone = google_dns_managed_zone.public.name
  rrdatas      = ["34.120.2.10"]
}

# ── CNAME record ──────────────────────────────────────────
resource "google_dns_record_set" "www_cname" {
  name         = "www.${google_dns_managed_zone.public.dns_name}"
  type         = "CNAME"
  ttl          = 300
  managed_zone = google_dns_managed_zone.public.name
  rrdatas      = [google_dns_managed_zone.public.dns_name]  # → example.com.
}

# ── MX records ────────────────────────────────────────────
resource "google_dns_record_set" "mx" {
  name         = google_dns_managed_zone.public.dns_name
  type         = "MX"
  ttl          = 3600
  managed_zone = google_dns_managed_zone.public.name
  rrdatas = [
    "1 aspmx.l.google.com.",
    "5 alt1.aspmx.l.google.com.",
    "5 alt2.aspmx.l.google.com.",
  ]
}

# ── TXT records (SPF + domain verification) ───────────────
resource "google_dns_record_set" "txt" {
  name         = google_dns_managed_zone.public.dns_name
  type         = "TXT"
  ttl          = 3600
  managed_zone = google_dns_managed_zone.public.name
  rrdatas = [
    "\"v=spf1 include:_spf.google.com ~all\"",
    "\"google-site-verification=abc123XYZ\"",
  ]
}

# ── CAA record ────────────────────────────────────────────
resource "google_dns_record_set" "caa" {
  name         = google_dns_managed_zone.public.dns_name
  type         = "CAA"
  ttl          = 3600
  managed_zone = google_dns_managed_zone.public.name
  rrdatas      = ["0 issue \"letsencrypt.org\"", "0 issue \"pki.goog\""]
}
```

---

### Private Zone Bound to a VPC

```hcl
# ── VPC (referenced by private zone) ─────────────────────
data "google_compute_network" "vpc" {
  name = "my-vpc"
}

# ── Private managed zone ──────────────────────────────────
resource "google_dns_managed_zone" "private" {
  name        = "internal-zone"
  dns_name    = "internal.example.com."
  description = "Private zone for internal services"
  visibility  = "private"

  private_visibility_config {
    networks {
      network_url = data.google_compute_network.vpc.id
    }
  }
}

# ── Internal A records ────────────────────────────────────
resource "google_dns_record_set" "db_internal" {
  name         = "db.${google_dns_managed_zone.private.dns_name}"
  type         = "A"
  ttl          = 60
  managed_zone = google_dns_managed_zone.private.name
  rrdatas      = ["10.0.0.5"]
}

resource "google_dns_record_set" "cache_internal" {
  name         = "cache.${google_dns_managed_zone.private.dns_name}"
  type         = "A"
  ttl          = 60
  managed_zone = google_dns_managed_zone.private.name
  rrdatas      = ["10.0.0.10", "10.0.0.11"]   # Multiple IPs = round-robin
}
```

---

### Forwarding Zone to On-Prem Resolver

```hcl
resource "google_dns_managed_zone" "forwarding" {
  name        = "corp-forwarding-zone"
  dns_name    = "corp.internal."
  description = "Forward corp.internal queries to on-prem DNS"
  visibility  = "private"

  private_visibility_config {
    networks {
      network_url = data.google_compute_network.vpc.id
    }
  }

  forwarding_config {
    target_name_servers {
      ipv4_address    = "192.168.1.53"     # Primary on-prem resolver
      forwarding_path = "private"           # Use private peering path (VPN/Interconnect)
    }
    target_name_servers {
      ipv4_address    = "192.168.1.54"     # Secondary on-prem resolver
      forwarding_path = "private"
    }
  }
}
```

---

### DNS Peering Between Two VPCs

```hcl
data "google_compute_network" "hub_vpc" { name = "hub-vpc" }
data "google_compute_network" "spoke_vpc" { name = "spoke-vpc" }

# ── Producer: private zone in hub VPC ─────────────────────
resource "google_dns_managed_zone" "hub_private" {
  name       = "shared-services-zone"
  dns_name   = "shared.internal."
  visibility = "private"

  private_visibility_config {
    networks {
      network_url = data.google_compute_network.hub_vpc.id
    }
  }
}

# ── Consumer: peering zone in spoke VPC ───────────────────
resource "google_dns_managed_zone" "spoke_peering" {
  name       = "spoke-to-hub-peering"
  dns_name   = "shared.internal."
  visibility = "private"

  private_visibility_config {
    networks {
      network_url = data.google_compute_network.spoke_vpc.id
    }
  }

  peering_config {
    target_network {
      network_url = data.google_compute_network.hub_vpc.id   # Query hub VPC's zone
    }
  }
}
```

---

### DNSSEC on a Public Zone

```hcl
resource "google_dns_managed_zone" "dnssec_zone" {
  name        = "secure-example-com"
  dns_name    = "secure.example.com."
  description = "DNSSEC-enabled zone"
  visibility  = "public"

  dnssec_config {
    state         = "on"                   # Enable DNSSEC
    non_existence = "nsec3"                # NSEC3 for zone walking prevention
    default_key_specs {
      algorithm  = "rsasha256"
      key_length = 2048
      key_type   = "keySigning"            # KSK — signs DNSKEY RRset
    }
    default_key_specs {
      algorithm  = "rsasha256"
      key_length = 1024
      key_type   = "zoneSigning"           # ZSK — signs all other RRsets
    }
  }
}

# After apply, retrieve DS record for registrar:
# gcloud dns dns-keys list --zone=secure-example-com --filter="type=keySigning"
```

---

## 7. DNSSEC

### How DNSSEC Works

```
Chain of Trust:
. (root zone) — DNSKEY + RRSIG (signed by ICANN root KSK)
    │
    │  DS record at root proves .com KSK is trusted
    ▼
.com zone — DNSKEY + RRSIG (signed by Verisign .com KSK)
    │
    │  DS record at .com proves example.com KSK is trusted
    ▼
example.com zone (Cloud DNS) — DNSKEY (KSK + ZSK) + RRSIG on all records
    │
    │  ZSK signs: A, MX, TXT, CNAME, etc.
    │  KSK signs: DNSKEY RRset only
    ▼
Resolver validates: signature on record → ZSK → KSK → DS at parent → chain to root
```

**Key types:**
| Key | Signs | Rotation | Size |
|---|---|---|---|
| **KSK (Key-Signing Key)** | DNSKEY RRset only | Less frequent (yearly) | 2048-bit RSA |
| **ZSK (Zone-Signing Key)** | All other RRsets | More frequent (monthly) | 1024-bit RSA |

Cloud DNS manages both keys automatically — rotation happens without downtime.

---

### Enabling DNSSEC Step by Step

```bash
# Step 1: Enable DNSSEC on the zone
gcloud dns managed-zones update example-com \
  --dnssec-state=on

# Step 2: Wait for key generation (usually <1 minute)
gcloud dns managed-zones describe example-com \
  --format="yaml(dnssecConfig)"
# Look for: state: on, nonExistence: nsec3

# Step 3: Retrieve the DS record
gcloud dns dns-keys list \
  --zone=example-com \
  --filter="type=keySigning" \
  --format="value(dsRecord)"

# Output example:
# 12345 8 2 AABBCCDDEEFF001122334455...
# Format: <key-tag> <algorithm> <digest-type> <digest>

# Step 4: Register DS record at your domain registrar
# (GoDaddy, Namecheap, Google Domains, etc.)
# Field mapping:
#   Key Tag:      12345
#   Algorithm:    8  (RSA/SHA-256)
#   Digest Type:  2  (SHA-256)
#   Digest:       AABBCCDDEEFF001122334455...

# Step 5: Verify DNSSEC is working
dig example.com +dnssec @8.8.8.8
# Look for 'ad' flag in response (Authenticated Data)
# and RRSIG records in the answer section
```

---

### DNSSEC Validation

When a DNSSEC-validating resolver receives a response:

```
1. Fetch RRset (e.g., A record for api.example.com.)
2. Fetch RRSIG for that RRset
3. Fetch DNSKEY (ZSK) from example.com. zone
4. Verify RRSIG using ZSK public key → ✅ record is authentic
5. Verify ZSK is trusted: fetch DNSKEY (KSK), check it matches DS record at .com
6. Verify DS record at .com is signed by .com ZSK → chain continues to root
7. If chain validates: return answer with 'ad' flag
8. If chain breaks: return SERVFAIL
```

---

### DNSSEC Gotchas

- **❌ Disabling DNSSEC without removing DS record** → all DNSSEC-validating resolvers return SERVFAIL for your domain; your site goes down for many users
- **❌ Registrar doesn't support DS record entry** → DNSSEC chain cannot be established; choose a registrar that supports DNSSEC (Google Domains, Cloudflare, GoDaddy, etc.)
- **❌ Enabling DNSSEC on a private zone** → not supported; DNSSEC is for public zones only
- **❌ NS delegation mismatch** → if your registrar's NS records don't match Cloud DNS-assigned NS servers, DNSSEC validation will fail
- **⚠️ Algorithm 8 (RSA/SHA-256)** is the default and most widely supported; avoid legacy algorithms (1, 3, 5)

---

## 8. Private DNS & Hybrid Architectures

### Private Zone + VPC Binding

```bash
# Step 1: Create private zone
gcloud dns managed-zones create internal-zone \
  --dns-name="svc.internal." \
  --visibility=private \
  --networks=my-vpc

# Step 2: Add records
gcloud dns record-sets transaction start --zone=internal-zone
gcloud dns record-sets transaction add "10.0.0.5" \
  --name="db.svc.internal." --type=A --ttl=60 --zone=internal-zone
gcloud dns record-sets transaction execute --zone=internal-zone

# Step 3: Verify from a VM in the bound VPC
# SSH into any VM in my-vpc and run:
# nslookup db.svc.internal
# dig db.svc.internal @169.254.169.254
```

---

### DNS Forwarding to On-Premises (Outbound)

```
GCP VM → 169.254.169.254 (metadata resolver) → Cloud DNS
         → matches forwarding zone "corp.internal."
         → forwards to 192.168.1.53 (on-prem DNS via VPN/Interconnect)
         → on-prem DNS answers
```

**Requirements:**
- Cloud VPN or Cloud Interconnect must exist between GCP and on-prem
- On-prem DNS server IP must be reachable from GCP VPC
- Use `forwarding-path=private` to route via VPN/Interconnect (not internet)

```bash
gcloud dns managed-zones create onprem-fwd \
  --dns-name="corp.internal." \
  --visibility=private \
  --networks=my-vpc \
  --forwarding-targets=192.168.1.53 \
  --server-logging-config=off
```

---

### Inbound DNS Forwarding (On-Prem → Cloud)

Allows **on-premises systems** to resolve Cloud DNS private zones by querying a GCP-hosted forwarding IP.

```
On-prem server → queries 10.10.0.4 (inbound forwarding IP in GCP VPC)
               → Cloud DNS resolves private zone records
               → returns answer to on-prem
```

```bash
# Step 1: Create a DNS policy with inbound forwarding enabled
gcloud dns policies create inbound-fw-policy \
  --networks=my-vpc \
  --enable-inbound-forwarding

# Step 2: Find the assigned inbound forwarding IPs
gcloud dns policies describe inbound-fw-policy

# Look for: inboundForwardingAddress — one IP per region with VMs
# Example: 10.10.0.4 in us-central1

# Step 3: Configure on-prem DNS server to forward *.svc.internal to 10.10.0.4
# (BIND example):
# zone "svc.internal" {
#     type forward;
#     forwarders { 10.10.0.4; };
# };

# Step 4: Verify from on-prem
# dig db.svc.internal @10.10.0.4
```

---

### DNS Peering for Hub-and-Spoke

```
Architecture:
  Hub VPC: owns private zone "shared.internal."  →  db.shared.internal = 10.1.0.5
  Spoke-A VPC: peering zone → Hub VPC
  Spoke-B VPC: peering zone → Hub VPC

  VM in Spoke-A queries db.shared.internal
    → Spoke-A's peering zone → Hub VPC's private zone → 10.1.0.5 ✅
```

```bash
# Create peering zone in each spoke VPC
for SPOKE in spoke-a-vpc spoke-b-vpc spoke-c-vpc; do
  gcloud dns managed-zones create "peering-to-hub-${SPOKE//-/}" \
    --dns-name="shared.internal." \
    --visibility=private \
    --networks="$SPOKE" \
    --target-network=hub-vpc
done
```

> **Note**: VPC Network Peering and DNS Peering are independent — configuring VPC peering does NOT automatically enable DNS peering between the networks.

---

### Cloud VPN / Interconnect + DNS Integration

**Pattern 1: Bidirectional hybrid DNS**

```
GCP → On-prem:   Forwarding zone "corp.internal." → 192.168.1.53
On-prem → GCP:   Inbound policy on GCP + configure on-prem to forward "svc.internal." → GCP inbound IP
```

**Pattern 2: GCP as authoritative for everything**

```
All DNS (including on-prem hostnames) managed in Cloud DNS
On-prem DNS servers forward all queries to GCP inbound IP
Works well when migrating on-prem DNS to Cloud DNS
```

**Pattern 3: On-prem as authoritative, Cloud forwards to it**

```
On-prem DNS owns all zones
Cloud DNS forwarding zones → on-prem for all private names
GCP VMs use Cloud DNS → Cloud DNS forwards to on-prem
```

---

### Response Policy Zones (RPZ)

RPZ allows **overriding DNS responses** for security or compliance:

```bash
# Create a response policy
gcloud dns response-policies create security-policy \
  --networks=my-vpc \
  --description="Block malware domains, override CDN to internal"

# Add a rule: block a malicious domain (return NXDOMAIN)
gcloud dns response-policies rules create block-malware \
  --response-policy=security-policy \
  --dns-name="malware.example.com." \
  --local-data=name="malware.example.com.",type="A",ttl=5,rrdatas=""  # NXDOMAIN behavior

# Add a rule: override a public domain to internal IP
gcloud dns response-policies rules create internal-override \
  --response-policy=security-policy \
  --dns-name="cdn.thirdparty.com." \
  --local-data=name="cdn.thirdparty.com.",type="A",ttl=60,rrdatas="10.0.0.99"

# List response policy rules
gcloud dns response-policies rules list --response-policy=security-policy
```

---

## 9. Observability & Troubleshooting

### Enabling Query Logging

Cloud DNS query logs are enabled via **DNS policies**:

```bash
# Create or update a DNS policy with logging enabled
gcloud dns policies create logging-policy \
  --networks=my-vpc \
  --enable-logging \
  --description="Enable DNS query logging for my-vpc"

# Update existing policy to enable logging
gcloud dns policies update existing-policy \
  --enable-logging

# Logs appear in Cloud Logging under:
# resource.type = "dns_query"
```

**DNS Log Fields:**

| Field | Description |
|---|---|
| `resource.labels.target_name` | Zone name queried |
| `resource.labels.location` | GCP region of the resolver |
| `jsonPayload.queryName` | DNS name queried (e.g., `api.example.com.`) |
| `jsonPayload.queryType` | Record type requested (A, AAAA, MX, etc.) |
| `jsonPayload.responseCode` | NOERROR, NXDOMAIN, SERVFAIL, etc. |
| `jsonPayload.rdata` | Response data returned |
| `jsonPayload.sourceIP` | Client IP that made the query |
| `jsonPayload.vmInstanceId` | GCE instance ID (if from a VM) |
| `jsonPayload.vmInstanceName` | GCE instance name |
| `jsonPayload.protocol` | UDP or TCP |
| `timestamp` | Query timestamp |

---

### Cloud Monitoring Metrics

| Metric | Description | Alert On |
|---|---|---|
| `dns.googleapis.com/query/request_count` | Total DNS queries per zone | Sudden spike or drop |
| `dns.googleapis.com/query/response_count` | Responses by response code | NXDOMAIN / SERVFAIL rate spike |
| `dns.googleapis.com/query/latency` | Query latency distribution | P99 > 100ms |
| `dns.googleapis.com/response_policy/request_count` | RPZ policy hits | Blocked domain queries |

**MQL Query — NXDOMAIN rate:**
```
fetch dns_query
| metric 'dns.googleapis.com/query/response_count'
| filter metric.response_code == 'NXDOMAIN'
| group_by 5m, [value: aggregate(value.response_count)]
| every 5m
```

---

### Sample Cloud Logging Queries

```
# All DNS queries from a specific VM
resource.type="dns_query"
jsonPayload.vmInstanceName="my-vm-instance"

# All NXDOMAIN responses in the last hour
resource.type="dns_query"
jsonPayload.responseCode="NXDOMAIN"
timestamp >= "2026-03-15T00:00:00Z"

# All queries for a specific domain
resource.type="dns_query"
jsonPayload.queryName="api.example.com."

# Queries that hit a response policy (RPZ)
resource.type="dns_query"
jsonPayload.responseCode="NOERROR"
resource.labels.target_name="security-policy"

# High-volume query sources (who is hammering DNS?)
resource.type="dns_query"
| group by jsonPayload.sourceIP

# All SERVFAIL responses (potential DNSSEC or config issue)
resource.type="dns_query"
jsonPayload.responseCode="SERVFAIL"
```

---

### Troubleshooting Tools

```bash
# ── dig ──────────────────────────────────────────────────

# Query A record via Cloud DNS nameserver directly
dig api.example.com @ns-cloud-a1.googledomains.com

# Query with DNSSEC validation flags
dig api.example.com +dnssec +multiline @8.8.8.8

# Check NS delegation
dig NS example.com @a.root-servers.net

# Check SOA record (contains serial number, useful for propagation check)
dig SOA example.com @ns-cloud-a1.googledomains.com

# Reverse DNS lookup
dig -x 34.120.1.5

# Check DS record at parent zone
dig DS example.com @a.gtld-servers.net

# Full DNSSEC chain trace
dig api.example.com +dnssec +trace

# ── nslookup ─────────────────────────────────────────────

nslookup api.example.com
nslookup -type=MX example.com 8.8.8.8          # Query specific resolver
nslookup -type=TXT example.com                  # Check TXT records

# ── gcloud ───────────────────────────────────────────────

# Check zone config
gcloud dns managed-zones describe example-com

# Verify record exists
gcloud dns record-sets list --zone=example-com \
  --filter="name=api.example.com."

# Get inbound forwarding IPs for a policy
gcloud dns policies describe my-policy \
  --format="yaml(networks,inboundForwardingAddress)"

# Check DNSSEC key status
gcloud dns dns-keys list --zone=example-com
```

---

### Troubleshooting Decision Tree

```
DNS resolution failing (NXDOMAIN, timeout, wrong IP)?
│
├── Is it a PUBLIC zone?
│   │
│   ├── Records exist in Cloud DNS (gcloud dns record-sets list)?
│   │   └── NO → Add missing records via transaction
│   │   └── YES → Continue...
│   │
│   ├── NS records at registrar match Cloud DNS nameservers?
│   │   └── dig NS example.com @a.root-servers.net
│   │   └── NO → Update NS at registrar → wait for TTL (up to 48h)
│   │   └── YES → Continue...
│   │
│   ├── Getting NXDOMAIN from Cloud DNS NS directly?
│   │   └── dig api.example.com @ns-cloud-a1.googledomains.com
│   │   └── NXDOMAIN → Record missing or wrong zone (check zone DNS name)
│   │
│   └── DNSSEC enabled? Getting SERVFAIL?
│       └── dig example.com +dnssec @8.8.8.8
│       └── SERVFAIL → Check DS record at registrar matches Cloud DNS KSK
│           └── gcloud dns dns-keys list --zone=... --filter="type=keySigning"
│
├── Is it a PRIVATE zone?
│   │
│   ├── Querying from a VM in the bound VPC?
│   │   └── gcloud dns managed-zones describe ... → check networks list
│   │   └── NO → Add VPC to private visibility config
│   │
│   ├── Record exists in private zone?
│   │   └── gcloud dns record-sets list --zone=internal-zone
│   │   └── NO → Add record
│   │
│   ├── Is there a forwarding zone for the same domain?
│   │   └── Forwarding zone takes precedence over private zone for same domain
│   │   └── Check: gcloud dns managed-zones list | grep <domain>
│   │
│   └── Peering zone not resolving?
│       └── Verify target-network has the zone and VMs can reach it
│       └── Peering is one-directional — check direction is correct
│
└── Is the record correct but propagation seems slow?
    ├── Check TTL: old TTL may still be cached
    │   └── dig api.example.com → look at TTL in response; wait for it to count to 0
    ├── Test from multiple locations or use: https://dnschecker.org
    └── If TTL was long (86400s), caches may hold old value for up to 24h
        └── Prevention: always lower TTL 48h before planned changes
```

---

## 10. Key Limits & Quotas

| Resource | Default Limit | Notes |
|---|---|---|
| **Managed zones per project** | 1,000 | Can be raised |
| **Resource record sets per zone** | 10,000 | Can be raised |
| **Records (RRdata) per record set** | 100 | Per RRset (e.g., 100 A records for one hostname) |
| **Total RRdata length per record set** | 65,535 bytes | Includes all values |
| **DNS policies per project** | 100 | |
| **Networks per DNS policy** | 50 | VPCs per policy |
| **Forwarding targets per forwarding zone** | 20 | IP addresses |
| **Response policy rules per policy** | 1,000 | |
| **Response policies per project** | 50 | |
| **DNS keys (DNSKEY) per zone** | 2 KSK + 2 ZSK | 4 total (supports key rotation overlap) |
| **Transaction size limit** | 100 resource record sets | Per transaction commit |
| **Zone file import size** | 100 MB | Per import operation |
| **Label length per zone** | 63 characters | Per DNS label component |
| **Total DNS name length** | 255 characters | Full FQDN |
| **SOA record** | 1 per zone | Auto-created; not deletable |
| **NS record at apex** | 1 per zone | Auto-created; not directly editable |
| **Private zone VPC bindings** | 10,000 per zone | Networks bound to one zone |
| **Peering zones per network** | 500 | Peering zones pointing to same producer |
| **Query logging** | Per-VPC via DNS policy | Requires logging policy; billed as Cloud Logging |

> 📝 Raise quotas: **GCP Console → IAM & Admin → Quotas → Filter "Cloud DNS"**

---

## 11. Gotchas & Best Practices

### Common Pitfalls

- **❌ Not delegating NS records at the registrar** — Cloud DNS only becomes authoritative for your domain when the registrar's NS records point to the Cloud DNS nameservers (`ns-cloud-*.googledomains.com`). This is the #1 mistake when setting up Cloud DNS for the first time. Always verify with `dig NS yourdomain.com @a.root-servers.net`.

- **❌ CNAME at the zone apex** — DNS spec prohibits CNAME at the zone root (`example.com.`) because it conflicts with SOA and NS records. Use an A record pointing to a static IP (e.g., Cloud Load Balancer global IP) at the apex. Only use CNAME on subdomains (`www.example.com.`).

- **❌ Forgetting trailing dots in DNS names** — Cloud DNS FQDNs require a trailing dot (`example.com.`). Missing it in `gcloud` commands or Terraform causes silent failures or unexpected behavior.

- **❌ Disabling DNSSEC without first removing the DS record at the registrar** — This severs the chain of trust and causes SERVFAIL for all DNSSEC-validating resolvers (which includes Google's 8.8.8.8). Your domain will appear down for ~30% of internet users. Always remove DS record first, wait for the parent zone TTL, then disable DNSSEC.

- **❌ Not lowering TTLs before migration** — If your current record TTL is 86400s (24h) and you change the record without lowering it first, users may see the old IP for up to 24 hours. Lower TTL to 60–300s at least 2× the current TTL period before the change.

- **❌ VPC peering ≠ DNS peering** — Configuring VPC Network Peering does not automatically allow cross-VPC DNS resolution. DNS Peering must be configured separately as a managed zone with `peering_config`.

- **❌ Private zone override not working** — If a private zone for `example.com.` is created but the VM still resolves to the public IP, check that the VPC is in the zone's `private_visibility_config.networks` list. Binding is not automatic.

- **❌ Forwarding zone to on-prem using `forwarding-path=default`** — The default path uses public internet routing, which may fail for RFC1918 on-prem IPs. Always use `forwarding-path=private` for on-prem targets reachable via VPN or Interconnect.

- **❌ TXT record quoting** — TXT record values must be quoted strings. In `gcloud`, wrap the entire value in single quotes with the string value in double quotes: `'"v=spf1 mx ~all"'`. Terraform uses `\"` escaping inside HCL strings.

- **❌ Updating a record without using a transaction** — Using `record-sets update` replaces the entire RRset atomically. If you have multiple IPs in an A record and only want to change one, use `transaction start` → `remove` old → `add` new → `execute`. Otherwise you'll drop other IPs.

---

### Best Practices

- **✅ Use atomic transactions for multi-record changes** — Wrap all related record additions and deletions in a single transaction. This ensures all changes are applied together, preventing partial states that cause resolution errors.

- **✅ TTL strategy: lower before, restore after** — 48h before any planned record change, lower TTL to 300s. Make the change. After 24h of stable operation, restore TTL to 3600s or higher. This minimizes both risk and cost.

- **✅ Use TXT records for domain verification and SPF** — Google Workspace, GitHub, AWS SES, and many other services require TXT records for domain verification. Keep a canonical list in your IaC (Terraform) — don't manage them manually.

- **✅ Manage DNS with Terraform or gcloud exports** — DNS records are configuration — treat them as code. Use Terraform for all zones and records, keep them in version control. Use `gcloud dns record-sets export` for disaster recovery backups.

- **✅ Split-horizon for hybrid apps** — Create matching public and private zones for the same domain. Public zone holds external IPs; private zone holds internal IPs for the same hostnames. VMs in bound VPCs automatically use the private zone, avoiding hairpin routing through the load balancer.

- **✅ Scope private zone visibility carefully** — Don't bind a private zone to every VPC by default. Each additional VPC binding increases the blast radius if a record is misconfigured. Only bind to VPCs that actually need resolution of that zone.

- **✅ Use CAA records to restrict certificate issuance** — Add a CAA record to limit which Certificate Authorities can issue TLS certs for your domain. Prevents unauthorized cert issuance even if someone compromises your CA account. Minimum: `0 issue "pki.goog"` for Google-managed certs.

- **✅ Cost optimization** — Cloud DNS charges ~$0.20/month per zone (first 25 zones, then $0.10) plus $0.40/million queries (first billion/month). Reduce query volume by: raising TTLs on stable records, using private zones for internal traffic (cheaper path), and consolidating zones where possible.

- **✅ GKE and Cloud DNS** — GKE clusters use `kube-dns` or `CoreDNS` for `cluster.local` resolution internally. For external service discovery (services exposed outside the cluster), use **ExternalDNS** controller to automatically sync Kubernetes Service/Ingress resources to Cloud DNS records.

- **✅ SOA record tuning** — Cloud DNS auto-manages the SOA record, but the default refresh/retry/expire values matter for secondary resolvers. For public zones with DNSSEC, the default SOA TTL (300s) is fine. Don't modify SOA unless you have a specific reason (e.g., secondary DNS configuration).

- **✅ Enable query logging during migrations** — Turn on DNS policy logging when migrating domains or making major record changes. Log `NXDOMAIN` and `SERVFAIL` responses to catch resolution failures within minutes, before users report them.

- **✅ Use `gcloud dns record-sets list` to audit before changes** — Before any `transaction execute`, run `list` to see the current state. This prevents accidental deletion of existing records when doing partial updates.

---

*Last updated: March 2026 | Based on GCP Cloud DNS documentation and production patterns*
