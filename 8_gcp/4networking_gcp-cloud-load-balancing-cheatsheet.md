# GCP Cloud Load Balancing — Comprehensive Cheatsheet

> **TL;DR — Which Load Balancer to Use?**
> - **External HTTP(S) traffic (global, L7)** → External Application Load Balancer (classic or global)
> - **Internal HTTP(S) microservices (L7, inside VPC)** → Internal Application Load Balancer
> - **External TCP/UDP (non-HTTP, global)** → External Proxy Network LB (TCP/SSL Proxy) or External Passthrough Network LB
> - **Internal TCP/UDP (L4, inside VPC)** → Internal Passthrough Network LB
> - **Static content from GCS** → Backend Bucket behind an External HTTP(S) LB

---

## Table of Contents

1. [Overview](#1-overview)
2. [Core Concepts](#2-core-concepts)
3. [Load Balancer Types — Deep Dive](#3-load-balancer-types--deep-dive)
4. [Network Endpoint Groups (NEGs)](#4-network-endpoint-groups-negs)
5. [gcloud CLI Cheatsheet](#5-gcloud-cli-cheatsheet)
6. [Terraform Examples](#6-terraform-examples)
7. [SSL/TLS & Security](#7-ssltls--security)
8. [Observability & Troubleshooting](#8-observability--troubleshooting)
9. [Key Limits & Quotas](#9-key-limits--quotas)
10. [Gotchas & Best Practices](#10-gotchas--best-practices)

---

## 1. Overview

**GCP Cloud Load Balancing** is a fully distributed, software-defined, managed load balancing service. It requires no pre-warming, scales instantly, and is deeply integrated with GCP networking, security, and observability services.

**Key Benefits:**
- Single global anycast IP (for global LBs) — traffic routed to nearest healthy backend
- Native integration with Cloud Armor, IAP, Cloud CDN, and Certificate Manager
- Auto-scaling with zero warmup time
- Health-check-driven failover across regions and zones
- Support for VM instance groups, GKE pods (via NEGs), serverless (Cloud Run, App Engine, Cloud Functions), and on-prem endpoints

---

### Quick-Reference: Load Balancer Types

| Type | Layer | Scope | Protocols | External / Internal | Typical Use Case |
|---|---|---|---|---|---|
| **External Application LB** (HTTP(S)) | L7 | Global / Regional | HTTP, HTTPS, HTTP/2, gRPC | External | Public-facing web apps, APIs |
| **Internal Application LB** (HTTP(S)) | L7 | Regional / Cross-region | HTTP, HTTPS, HTTP/2, gRPC | Internal | Internal microservices, service mesh |
| **External Proxy Network LB** (TCP/SSL Proxy) | L4 | Global | TCP, SSL/TLS | External | Non-HTTP TCP apps needing global anycast |
| **External Passthrough Network LB** | L4 | Regional | TCP, UDP, ESP, ICMP, other IP | External | UDP workloads, legacy apps needing client IP preservation |
| **Internal Passthrough Network LB** | L4 | Regional | TCP, UDP, other IP | Internal | Internal services requiring preserved client IP, ILB for GKE |
| **Internal Proxy Network LB** (TCP) | L4 | Regional / Cross-region | TCP | Internal | Internal TCP proxy, used with Traffic Director |
| **Backend Bucket** (via App LB) | L7 | Global | HTTP, HTTPS | External | Serving static assets from GCS with CDN |

---

## 2. Core Concepts

### Architecture Overview (ASCII)

```
Client Request
      │
      ▼
┌─────────────────────────────────────────────────┐
│              FRONTEND (Forwarding Rule)          │
│  Global/Regional IP  │  Port  │  Protocol        │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│            TARGET PROXY                         │
│  (HTTP/HTTPS Target Proxy, TCP Proxy)           │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│               URL MAP (L7 only)                 │
│  Host Rules → Path Matchers → Route Actions     │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│            BACKEND SERVICE                      │
│  Health Checks │ Balancing Mode │ Session Affinity│
└──────────────────────┬──────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     Instance       Cloud Run     GCS Bucket
      Group          (NEG)       (Backend Bucket)
```

---

### Frontend Configuration

| Component | Description |
|---|---|
| **Forwarding Rule** | Defines the entry point: IP address, port range, protocol, and target |
| **Global IP** | Single anycast IP served by Google's PoPs worldwide (HTTP(S), TCP/SSL Proxy LBs) |
| **Regional IP** | IP bound to a single GCP region (Internal LBs, Regional External LBs) |
| **Port Range** | The port(s) the forwarding rule listens on (e.g., 80, 443, 8080-8090) |
| **Protocol** | TCP, UDP, HTTP, HTTPS — must match target proxy type |

```
Forwarding Rule → Target Proxy → URL Map → Backend Service
```

---

### Backend Configuration

| Component | Description |
|---|---|
| **Backend Service** | Logical configuration object: health check, balancing mode, session affinity, timeout |
| **Backend** | A single backend attached to a Backend Service — can be MIG, NEG, or Backend Bucket |
| **Instance Group (MIG/UIG)** | Managed or unmanaged group of VM instances |
| **NEG (Network Endpoint Group)** | Fine-grained endpoint group (VMs, containers, serverless, on-prem) |
| **Backend Bucket** | GCS bucket exposed as a backend — for static file serving |

**Balancing Modes (per backend):**

| Mode | Layer | Based On | Use Case |
|---|---|---|---|
| `UTILIZATION` | L7 | Backend CPU/memory utilization | MIGs with autoscaling |
| `RATE` | L7 | Requests per second (RPS) | Fine-grained traffic splitting |
| `CONNECTION` | L4 | Max concurrent connections | TCP/UDP LBs |

---

### URL Maps and Host/Path Rules (L7 only)

```
URL Map
├── Default Backend Service (catch-all)
└── Host Rules
    ├── Host: api.example.com
    │   └── Path Matcher
    │       ├── /v1/*   → backend-v1
    │       ├── /v2/*   → backend-v2
    │       └── /*      → backend-default
    └── Host: static.example.com
        └── Path Matcher
            └── /*      → gcs-backend-bucket
```

- **Host Rule**: matches `Host:` header — supports wildcards (`*.example.com`)
- **Path Matcher**: evaluates URL path — longest prefix or regex match wins
- **Route Action**: redirect, URL rewrite, traffic split, retry policy, timeout override
- **Traffic Splitting**: split by weight across multiple backends (canary deploys)

```yaml
# Example: 90/10 canary split
weightedBackendServices:
  - backendService: projects/.../backendServices/prod
    weight: 90
  - backendService: projects/.../backendServices/canary
    weight: 10
```

---

### Health Checks

| Type | Protocol | Key Config |
|---|---|---|
| HTTP | HTTP | Request path, host, expected response codes |
| HTTPS | HTTPS | Same as HTTP + TLS verification |
| TCP | TCP | Connect-only or send/receive probe data |
| gRPC | gRPC | Service name, gRPC health protocol |
| HTTP/2 | HTTP/2 | Path, host header |
| SSL | SSL/TLS | Connect + optional probe |

**Health Check Parameters:**

| Parameter | Default | Description |
|---|---|---|
| `check-interval-sec` | 10s | Time between probes |
| `timeout-sec` | 5s | Max wait per probe |
| `healthy-threshold` | 2 | Consecutive successes to mark healthy |
| `unhealthy-threshold` | 3 | Consecutive failures to mark unhealthy |

> ⚠️ **Firewall Rule Required**: GCP health check probers use IPs `35.191.0.0/16` and `130.211.0.0/22`. Always create an ingress allow rule for these ranges on your backends.

```bash
gcloud compute firewall-rules create allow-health-checks \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=35.191.0.0/16,130.211.0.0/22 \
  --target-tags=lb-backend \
  --rules=tcp:80,tcp:8080
```

---

### Session Affinity

| Mode | How It Works | Best For |
|---|---|---|
| `NONE` | Default, stateless round-robin | Stateless APIs |
| `CLIENT_IP` | Hashes client IP | Simple stateful apps |
| `CLIENT_IP_PORT_PROTO` | Hash of IP + port + protocol | Fine-grained L4 affinity |
| `GENERATED_COOKIE` | LB injects a cookie | Stateful web apps (L7 only) |
| `HEADER_FIELD` | Hash of a specific HTTP header | Custom affinity on header |
| `HTTP_COOKIE` | Existing cookie used as key | Existing session cookie |

> **Connection Draining**: Set `--connection-draining-timeout` (default 300s) on backend services to allow in-flight requests to complete before backend removal. Set to 0 to disable.

---

### Load Balancing Algorithms

| Algorithm | Available On | Description |
|---|---|---|
| **Round Robin** | All LBs | Default; distributes evenly across backends |
| **Least Requests** | External App LB (Envoy) | Routes to backend with fewest active requests |
| **Ring Hash** | External App LB (Envoy) | Consistent hashing — good for caches |
| **Maglev** | External App LB (Envoy) | Consistent hashing with better load spread |
| **Random** | External App LB (Envoy) | Randomly selects a backend |
| **Weighted Round Robin** | All LBs (via capacity scaler) | Weight by backend `capacityScaler` |

> **Locality LB Policy** (`localityLbPolicy`): Configured per backend service. Advanced policies (Least Requests, Ring Hash, Maglev, Random) require **Envoy-based** load balancers.

---

## 3. Load Balancer Types — Deep Dive

### External Application Load Balancer (HTTP(S))

**When to use:**
- Public-facing web apps, REST APIs, gRPC services
- Need content-based routing (host, path, headers)
- Want Cloud CDN, Cloud Armor, or IAP
- Global anycast with multi-region failover

**Modes:**

| Mode | Scope | Proxy Type |
|---|---|---|
| Global external (classic) | Global | Classic proxy |
| Global external | Global | Envoy proxy |
| Regional external | Single region | Envoy proxy |

**Supported Backends:** MIGs, Zonal NEGs, Serverless NEGs, Internet NEGs, Hybrid NEGs, Backend Buckets

**Key Limits:**
- Max URL map rules: 200 (default), 1000 (requestable)
- Max SSL certificates per target proxy: 15
- Max backend services per URL map: 200
- Timeout: 30s default (up to 86400s)

---

### Internal Application Load Balancer (HTTP(S))

**When to use:**
- Internal microservices communication within a VPC
- GKE-to-GKE or VM-to-VM HTTP routing
- Cross-region internal traffic (with cross-region mode)

**Modes:**

| Mode | Scope |
|---|---|
| Regional internal | Single region |
| Cross-region internal | Multiple regions |

**Supported Backends:** MIGs, Zonal NEGs, Serverless NEGs (Cloud Run)

**Key Limits:**
- Requires a **proxy-only subnet** (`--purpose=REGIONAL_MANAGED_PROXY`)
- Cannot be accessed from outside the VPC without Cloud VPN or Interconnect

```bash
# Create proxy-only subnet (required for internal app LB)
gcloud compute networks subnets create proxy-only-subnet \
  --purpose=REGIONAL_MANAGED_PROXY \
  --role=ACTIVE \
  --region=us-central1 \
  --network=my-vpc \
  --range=10.10.0.0/24
```

---

### External Proxy Network Load Balancer (TCP/SSL Proxy)

**When to use:**
- Non-HTTP TCP workloads needing global distribution
- Terminate TLS at the LB and forward plain TCP to backends
- Database proxying, MQTT, gaming TCP connections

**Protocols:** TCP (port 25, 43, 80, 110, 143, 195, 443, 465, 587, 700–900+)

**Supported Backends:** MIGs (global or regional), Zonal NEGs

**Key Limits:**
- Client IP is **not** preserved (traffic comes from Google's proxy IPs)
- Use PROXY Protocol on backends to recover original client IP
- Not suitable for UDP

---

### External Passthrough Network Load Balancer

**When to use:**
- UDP workloads (DNS, gaming, streaming)
- Need to preserve original client IP address
- Legacy TCP apps that cannot use a proxy
- ICMP, ESP (IPsec) traffic

**Protocols:** TCP, UDP, ESP, AH, SCTP, ICMP, ICMPv6

**Supported Backends:** MIGs (regional), Target Instances

**Key Limits:**
- Regional only — no global anycast
- No SSL offload
- No content-based routing
- Each backend pool is region-specific

---

### Internal Passthrough Network Load Balancer

**When to use:**
- Internal TCP/UDP services requiring client IP preservation
- L4 internal load balancing for GKE `LoadBalancer` services (default)
- HA VPN gateway, NVA (network virtual appliance) deployments

**Protocols:** TCP, UDP, ICMP, other IP

**Supported Backends:** MIGs (zonal or regional), Unmanaged Instance Groups

**Key Limits:**
- Regional — backends and clients must be in same region
- Supports **Global Access** option (allows cross-region clients within same VPC)

---

### Internal Proxy Network Load Balancer (TCP)

**When to use:**
- Internal TCP proxy where client IP preservation is not needed
- Used with Traffic Director / service mesh
- Cross-region internal TCP proxying

**Protocols:** TCP

**Supported Backends:** MIGs, Zonal NEGs

---

## 4. Network Endpoint Groups (NEGs)

NEGs are collections of endpoints (IP:port pairs) that can serve as backends for load balancers. More granular and flexible than Instance Groups.

### NEG Types Comparison

| NEG Type | Endpoints | Scope | Use Case |
|---|---|---|---|
| **Zonal NEG** | VM primary IPs or Pod IPs (GKE) | Zonal | GKE container-native LB, fine-grained VM routing |
| **Internet NEG** | External IP:port or FQDN:port | Global | Route to external/SaaS backends |
| **Serverless NEG** | Cloud Run, App Engine, Cloud Functions | Regional | Serverless backends behind HTTP(S) LB |
| **Hybrid NEG** | On-prem or other-cloud IPs via VPN/Interconnect | Regional | Hybrid cloud load balancing |
| **Private Service Connect NEG** | PSC endpoint | Regional | Access published services via PSC |

---

### NEGs vs. Instance Groups

| Feature | Instance Group (MIG/UIG) | Zonal NEG |
|---|---|---|
| Granularity | VM-level | Pod/container-level (IP:port) |
| GKE support | Node-level (not pod-aware) | Pod-level (container-native LB) |
| Port flexibility | Single named port per MIG | Per-endpoint port |
| Traffic splitting | Via `capacityScaler` | Via weighted backends |
| Serverless support | ❌ | ✅ (Serverless NEG) |
| On-prem endpoints | ❌ | ✅ (Hybrid NEG) |
| Auto-managed lifecycle | ✅ (MIG) | Manual or via GKE |

**Container-Native Load Balancing (GKE):**
```yaml
# GKE Service annotation to use NEG instead of node-level LB
apiVersion: v1
kind: Service
metadata:
  annotations:
    cloud.google.com/neg: '{"ingress": true}'  # auto-creates zonal NEG
```

---

### Creating NEGs

```bash
# Zonal NEG (for GKE or VM endpoints)
gcloud compute network-endpoint-groups create my-neg \
  --network-endpoint-type=GCE_VM_IP_PORT \
  --zone=us-central1-a \
  --network=my-vpc \
  --subnet=my-subnet

# Add endpoint to NEG
gcloud compute network-endpoint-groups update my-neg \
  --zone=us-central1-a \
  --add-endpoint='instance=my-vm,ip=10.0.0.5,port=8080'

# Serverless NEG (Cloud Run)
gcloud compute network-endpoint-groups create my-serverless-neg \
  --region=us-central1 \
  --network-endpoint-type=SERVERLESS \
  --cloud-run-service=my-cloud-run-service

# Internet NEG (external backend)
gcloud compute network-endpoint-groups create my-internet-neg \
  --network-endpoint-type=INTERNET_FQDN_PORT \
  --global

gcloud compute network-endpoint-groups update my-internet-neg \
  --global \
  --add-endpoint='fqdn=api.thirdparty.com,port=443'
```

---

## 5. gcloud CLI Cheatsheet

### Health Checks

```bash
# Create HTTP health check
gcloud compute health-checks create http my-http-hc \
  --port=80 \                        # Port to probe
  --request-path=/healthz \          # HTTP path to check
  --check-interval=10s \             # Probe interval
  --timeout=5s \                     # Max wait per probe
  --healthy-threshold=2 \            # Successes to mark healthy
  --unhealthy-threshold=3            # Failures to mark unhealthy

# Create HTTPS health check
gcloud compute health-checks create https my-https-hc \
  --port=443 \
  --request-path=/healthz

# Create TCP health check
gcloud compute health-checks create tcp my-tcp-hc \
  --port=8080

# Create gRPC health check
gcloud compute health-checks create grpc my-grpc-hc \
  --port=50051 \
  --grpc-service-name=my.service.HealthCheck

# List all health checks
gcloud compute health-checks list

# Describe a health check
gcloud compute health-checks describe my-http-hc
```

---

### Backend Services

```bash
# Create external HTTP(S) backend service
gcloud compute backend-services create my-backend-svc \
  --protocol=HTTP \                         # Protocol LB uses to talk to backends
  --port-name=http \                        # Named port on instance group
  --health-checks=my-http-hc \             # Attach health check
  --global \                                # Global scope (for external HTTP(S) LB)
  --timeout=30s \                           # Backend response timeout
  --connection-draining-timeout=300s \      # Draining window on removal
  --session-affinity=NONE                   # No session stickiness

# Create internal backend service (regional)
gcloud compute backend-services create my-internal-svc \
  --protocol=HTTP \
  --health-checks=my-http-hc \
  --region=us-central1 \                    # Regional scope for internal LB
  --load-balancing-scheme=INTERNAL_MANAGED  # Internal App LB scheme

# Add a MIG as a backend
gcloud compute backend-services add-backend my-backend-svc \
  --global \
  --instance-group=my-mig \                 # MIG name
  --instance-group-zone=us-central1-a \     # Zone of MIG
  --balancing-mode=UTILIZATION \            # Balance on CPU utilization
  --max-utilization=0.8 \                   # 80% CPU cap before overflow
  --capacity-scaler=1.0                     # 100% of declared capacity used

# Add a NEG as a backend
gcloud compute backend-services add-backend my-backend-svc \
  --global \
  --network-endpoint-group=my-neg \
  --network-endpoint-group-zone=us-central1-a \
  --balancing-mode=RATE \                   # Balance on RPS
  --max-rate-per-endpoint=100              # 100 RPS max per endpoint

# Update backend service (e.g., change timeout)
gcloud compute backend-services update my-backend-svc \
  --global \
  --timeout=60s

# Remove a backend
gcloud compute backend-services remove-backend my-backend-svc \
  --global \
  --instance-group=my-mig \
  --instance-group-zone=us-central1-a

# Delete backend service
gcloud compute backend-services delete my-backend-svc --global
```

---

### URL Maps, Target Proxies, Forwarding Rules

```bash
# ── URL MAP ──────────────────────────────────────────────
# Create URL map with default backend
gcloud compute url-maps create my-url-map \
  --default-service=my-backend-svc \        # Default backend if no rules match
  --global

# Add host and path rules
gcloud compute url-maps import my-url-map \
  --global \
  --source my-url-map.yaml                  # Import full YAML config

# Export URL map to YAML (for editing)
gcloud compute url-maps export my-url-map \
  --global \
  --destination my-url-map.yaml

# ── TARGET PROXY ─────────────────────────────────────────
# Create HTTP target proxy
gcloud compute target-http-proxies create my-http-proxy \
  --url-map=my-url-map \                    # Attach URL map
  --global

# Create HTTPS target proxy (with SSL cert)
gcloud compute target-https-proxies create my-https-proxy \
  --url-map=my-url-map \
  --ssl-certificates=my-ssl-cert \          # Attach SSL certificate
  --global

# Update HTTPS proxy to add another cert
gcloud compute target-https-proxies update my-https-proxy \
  --global \
  --ssl-certificates=cert1,cert2

# ── FORWARDING RULE ───────────────────────────────────────
# Create global HTTP forwarding rule
gcloud compute forwarding-rules create my-http-fw-rule \
  --address=my-global-ip \                  # Reserved static IP (or omit for ephemeral)
  --global \
  --target-http-proxy=my-http-proxy \       # Attach target proxy
  --ports=80                                # Listen on port 80

# Create global HTTPS forwarding rule
gcloud compute forwarding-rules create my-https-fw-rule \
  --address=my-global-ip \
  --global \
  --target-https-proxy=my-https-proxy \
  --ports=443

# Create internal forwarding rule (TCP ILB)
gcloud compute forwarding-rules create my-ilb-fw-rule \
  --region=us-central1 \
  --load-balancing-scheme=INTERNAL \        # Internal LB
  --backend-service=my-internal-backend \   # Attach backend service directly (no proxy)
  --ports=8080 \
  --network=my-vpc \
  --subnet=my-subnet

# List forwarding rules
gcloud compute forwarding-rules list
gcloud compute forwarding-rules describe my-http-fw-rule --global
```

---

### Listing and Describing LB Components

```bash
# List all LB-related resources
gcloud compute backend-services list
gcloud compute url-maps list
gcloud compute target-http-proxies list
gcloud compute target-https-proxies list
gcloud compute forwarding-rules list
gcloud compute health-checks list
gcloud compute ssl-certificates list
gcloud compute network-endpoint-groups list

# Get backend health status
gcloud compute backend-services get-health my-backend-svc \
  --global

# Describe full URL map config
gcloud compute url-maps describe my-url-map --global

# Validate URL map (dry run)
gcloud compute url-maps validate my-url-map \
  --global \
  --load-balancing-scheme=EXTERNAL_MANAGED
```

---

### Static IP Management

```bash
# Reserve global static IP (for external LB)
gcloud compute addresses create my-global-ip \
  --global \
  --ip-version=IPV4

# Reserve regional static IP
gcloud compute addresses create my-regional-ip \
  --region=us-central1

# Reserve internal IP
gcloud compute addresses create my-internal-ip \
  --region=us-central1 \
  --subnet=my-subnet \
  --addresses=10.0.0.50              # Optional: pin to specific IP

# List reserved addresses
gcloud compute addresses list
```

---

## 6. Terraform Examples

### External HTTP(S) Load Balancer

```hcl
# ── variables ────────────────────────────────────────────
variable "project_id" {}
variable "region"     { default = "us-central1" }

# ── static IP ────────────────────────────────────────────
resource "google_compute_global_address" "default" {
  name = "my-lb-ip"
}

# ── health check ─────────────────────────────────────────
resource "google_compute_health_check" "default" {
  name               = "my-http-hc"
  check_interval_sec = 10
  timeout_sec        = 5

  http_health_check {
    port         = 80
    request_path = "/healthz"
  }
}

# ── backend service ───────────────────────────────────────
resource "google_compute_backend_service" "default" {
  name                  = "my-backend-svc"
  protocol              = "HTTP"
  port_name             = "http"
  load_balancing_scheme = "EXTERNAL_MANAGED"   # Envoy-based global LB
  timeout_sec           = 30
  health_checks         = [google_compute_health_check.default.id]

  backend {
    group           = google_compute_instance_group_manager.default.instance_group
    balancing_mode  = "UTILIZATION"
    max_utilization = 0.8
    capacity_scaler = 1.0
  }

  log_config {
    enable      = true
    sample_rate = 1.0    # Log 100% of requests
  }
}

# ── url map ───────────────────────────────────────────────
resource "google_compute_url_map" "default" {
  name            = "my-url-map"
  default_service = google_compute_backend_service.default.id

  # Example: route /api/* to a different backend
  host_rule {
    hosts        = ["*"]
    path_matcher = "allpaths"
  }

  path_matcher {
    name            = "allpaths"
    default_service = google_compute_backend_service.default.id

    path_rule {
      paths   = ["/api/*"]
      service = google_compute_backend_service.api.id
    }
  }
}

# ── Google-managed SSL certificate ────────────────────────
resource "google_compute_managed_ssl_certificate" "default" {
  name = "my-ssl-cert"
  managed {
    domains = ["example.com", "www.example.com"]
  }
}

# ── HTTPS target proxy ────────────────────────────────────
resource "google_compute_target_https_proxy" "default" {
  name             = "my-https-proxy"
  url_map          = google_compute_url_map.default.id
  ssl_certificates = [google_compute_managed_ssl_certificate.default.id]
}

# ── HTTP target proxy (redirect to HTTPS) ─────────────────
resource "google_compute_url_map" "redirect" {
  name = "http-to-https-redirect"
  default_url_redirect {
    https_redirect         = true
    redirect_response_code = "MOVED_PERMANENTLY_DEFAULT"
    strip_query            = false
  }
}

resource "google_compute_target_http_proxy" "redirect" {
  name    = "http-redirect-proxy"
  url_map = google_compute_url_map.redirect.id
}

# ── forwarding rules ──────────────────────────────────────
resource "google_compute_global_forwarding_rule" "https" {
  name                  = "my-https-fwd-rule"
  ip_address            = google_compute_global_address.default.id
  ip_protocol           = "TCP"
  port_range            = "443"
  target                = google_compute_target_https_proxy.default.id
  load_balancing_scheme = "EXTERNAL_MANAGED"
}

resource "google_compute_global_forwarding_rule" "http" {
  name                  = "my-http-fwd-rule"
  ip_address            = google_compute_global_address.default.id
  ip_protocol           = "TCP"
  port_range            = "80"
  target                = google_compute_target_http_proxy.redirect.id
  load_balancing_scheme = "EXTERNAL_MANAGED"
}

# ── firewall: allow health check probers ──────────────────
resource "google_compute_firewall" "allow_hc" {
  name    = "allow-lb-health-checks"
  network = "default"

  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }

  source_ranges = ["35.191.0.0/16", "130.211.0.0/22"]
  target_tags   = ["lb-backend"]
}
```

---

### Internal TCP (Passthrough) Load Balancer

```hcl
# ── health check ─────────────────────────────────────────
resource "google_compute_health_check" "tcp_hc" {
  name               = "my-tcp-hc"
  check_interval_sec = 10
  timeout_sec        = 5

  tcp_health_check {
    port = 8080
  }
}

# ── backend service (internal passthrough) ────────────────
resource "google_compute_region_backend_service" "ilb" {
  name                  = "my-ilb-backend"
  region                = var.region
  protocol              = "TCP"
  load_balancing_scheme = "INTERNAL"          # Internal passthrough
  health_checks         = [google_compute_health_check.tcp_hc.id]

  backend {
    group          = google_compute_instance_group_manager.default.instance_group
    balancing_mode = "CONNECTION"
  }

  # Enable Global Access (allow cross-region clients in same VPC)
  connection_draining_timeout_sec = 300
}

# ── forwarding rule (internal) ────────────────────────────
resource "google_compute_forwarding_rule" "ilb" {
  name                  = "my-ilb-fwd-rule"
  region                = var.region
  load_balancing_scheme = "INTERNAL"
  backend_service       = google_compute_region_backend_service.ilb.id
  all_ports             = true              # Forward all ports (or specify ports = ["8080"])
  network               = "my-vpc"
  subnetwork            = "my-subnet"
  allow_global_access   = true              # Allow cross-region clients within VPC
}

# ── firewall: allow internal traffic to backends ──────────
resource "google_compute_firewall" "allow_ilb" {
  name    = "allow-ilb-traffic"
  network = "my-vpc"

  allow {
    protocol = "tcp"
    ports    = ["8080"]
  }

  source_ranges = ["10.0.0.0/8"]   # Internal RFC1918 ranges
  target_tags   = ["ilb-backend"]
}

# ── firewall: allow health checks ────────────────────────
resource "google_compute_firewall" "allow_hc_ilb" {
  name    = "allow-ilb-health-checks"
  network = "my-vpc"

  allow {
    protocol = "tcp"
    ports    = ["8080"]
  }

  source_ranges = ["35.191.0.0/16", "130.211.0.0/22"]
  target_tags   = ["ilb-backend"]
}
```

---

## 7. SSL/TLS & Security

### SSL Certificate Types

| Type | Managed By | Renewal | Use Case |
|---|---|---|---|
| **Google-managed** | Google (auto-renew) | Automatic | Recommended for most use cases |
| **Self-managed** | You | Manual (upload new cert) | Custom/internal CAs, wildcard certs |
| **Certificate Manager** (GA) | Certificate Manager service | Automatic | Enterprise-scale, wildcard, cross-region |

```bash
# Google-managed certificate
gcloud compute ssl-certificates create my-cert \
  --domains=example.com,www.example.com \
  --global

# Self-managed certificate
gcloud compute ssl-certificates create my-self-managed-cert \
  --certificate=cert.pem \
  --private-key=key.pem \
  --global

# Certificate Manager — create and attach
gcloud certificate-manager certificates create my-cm-cert \
  --domains=example.com \
  --dns-authorizations=my-dns-auth

gcloud certificate-manager maps create my-cert-map
gcloud certificate-manager maps entries create my-entry \
  --map=my-cert-map \
  --certificates=my-cm-cert \
  --hostname=example.com
```

> **Google-managed cert DNS validation**: GCP automatically provisions a Let's Encrypt cert after DNS points to the LB IP. Can take 10–60 minutes. Status: `gcloud compute ssl-certificates describe my-cert --global`

---

### SSL Policies

Control minimum TLS version and cipher suites:

```bash
# Create custom SSL policy
gcloud compute ssl-policies create my-ssl-policy \
  --min-tls-version=TLS_1_2 \         # Minimum TLS 1.2
  --profile=MODERN \                   # COMPATIBLE | MODERN | RESTRICTED | CUSTOM
  --global

# Attach to HTTPS proxy
gcloud compute target-https-proxies update my-https-proxy \
  --ssl-policy=my-ssl-policy \
  --global
```

| Profile | TLS Versions | Ciphers |
|---|---|---|
| `COMPATIBLE` | 1.0, 1.1, 1.2 | Broad compatibility |
| `MODERN` | 1.2, 1.3 | Modern secure ciphers |
| `RESTRICTED` | 1.2, 1.3 | FIPS-compliant subset |
| `CUSTOM` | Configurable | You choose cipher suites |

---

### Cloud Armor Integration

Cloud Armor is GCP's WAF/DDoS protection, attached to **backend services** of HTTP(S) LBs:

```bash
# Create security policy
gcloud compute security-policies create my-security-policy \
  --description="WAF policy"

# Add rule: block a country (e.g., block CN)
gcloud compute security-policies rules create 1000 \
  --security-policy=my-security-policy \
  --expression="origin.region_code == 'CN'" \
  --action=deny-403

# Add rule: allow specific IP range
gcloud compute security-policies rules create 500 \
  --security-policy=my-security-policy \
  --src-ip-ranges=203.0.113.0/24 \
  --action=allow

# Enable pre-configured WAF rules (OWASP Top 10)
gcloud compute security-policies rules create 900 \
  --security-policy=my-security-policy \
  --expression="evaluatePreconfiguredExpr('xss-v33-stable')" \
  --action=deny-403

# Attach to backend service
gcloud compute backend-services update my-backend-svc \
  --global \
  --security-policy=my-security-policy
```

**Key Cloud Armor Features:**
- DDoS protection (L3/L4 always on; L7 with Adaptive Protection)
- IP/CIDR allowlist/denylist
- Geo-based blocking
- Rate limiting per IP
- Pre-configured WAF rules (SQLi, XSS, LFI, RFI, etc.)
- Bot management (reCAPTCHA integration)

---

### Identity-Aware Proxy (IAP)

IAP enforces Google identity-based authentication at the LB level — no VPN required for internal app access:

```bash
# Enable IAP on a backend service
gcloud iap web enable \
  --resource-type=backend-services \
  --service=my-backend-svc

# Grant IAP access to a user
gcloud iap web add-iam-policy-binding \
  --resource-type=backend-services \
  --service=my-backend-svc \
  --member=user:alice@example.com \
  --role=roles/iap.httpsResourceAccessor

# Grant access to a Google Group
gcloud iap web add-iam-policy-binding \
  --resource-type=backend-services \
  --service=my-backend-svc \
  --member=group:devs@example.com \
  --role=roles/iap.httpsResourceAccessor
```

> ⚠️ **Firewall for IAP**: IAP-proxied requests come from `35.235.240.0/20`. Add this range to your firewall allow rule alongside health check IPs.

---

## 8. Observability & Troubleshooting

### Key Cloud Monitoring Metrics

| Metric | Description | Alert Threshold |
|---|---|---|
| `loadbalancing.googleapis.com/https/request_count` | Total requests by response code | Spike in 5xx |
| `loadbalancing.googleapis.com/https/backend_latencies` | P50/P95/P99 latency to backends | P99 > SLO |
| `loadbalancing.googleapis.com/https/total_latencies` | End-to-end client latency | P99 > SLO |
| `loadbalancing.googleapis.com/https/backend_request_count` | Requests per backend | Imbalance detection |
| `loadbalancing.googleapis.com/l3/internal/rtt_latencies` | Round-trip time (Internal LB) | Elevated RTT |
| `compute.googleapis.com/instance/network/received_bytes_count` | Backend network I/O | Saturation |

**Useful MQL Query (Cloud Monitoring):**
```
fetch https_lb_rule
| metric 'loadbalancing.googleapis.com/https/request_count'
| filter (resource.url_map_name == 'my-url-map')
| group_by 1m, [value_request_count_aggregate: aggregate(value.request_count)]
| every 1m
```

---

### Enabling Access Logging

```bash
# Enable logging on a backend service (100% sample rate)
gcloud compute backend-services update my-backend-svc \
  --global \
  --enable-logging \
  --logging-sample-rate=1.0       # 0.0 to 1.0 (fraction of requests to log)

# Disable logging
gcloud compute backend-services update my-backend-svc \
  --global \
  --no-enable-logging
```

**Log query in Cloud Logging:**
```
resource.type="http_load_balancer"
resource.labels.url_map_name="my-url-map"
httpRequest.status>=500
```

---

### Common HTTP Error Codes

| Code | Likely Cause | How to Fix |
|---|---|---|
| **502 Bad Gateway** | Backend returned invalid response or crashed | Check backend logs; verify backend port/protocol match; increase timeout |
| **503 Service Unavailable** | No healthy backends; backend overloaded | Check health checks; add capacity; verify firewall rules allow HC probes |
| **504 Gateway Timeout** | Backend too slow to respond within LB timeout | Increase `--timeout` on backend service; optimize backend; check for blocking I/O |
| **524** | Client connection timed out (Cloudflare) | Specific to Cloudflare in front of GCP LB |
| **400 Bad Request** | Malformed request, blocked by Cloud Armor | Check Cloud Armor policy; verify request headers |
| **403 Forbidden** | Cloud Armor deny rule triggered; IAP not authorized | Review Cloud Armor logs; check IAP IAM bindings |

---

### Troubleshooting Decision Tree

```
Request fails or returns error
│
├── Is the LB IP reachable at all?
│   ├── NO  → Check: Forwarding rule created? Static IP reserved? Firewall allows inbound?
│   └── YES → Continue...
│
├── Is error 502/503/504?
│   ├── 503 → Run: gcloud compute backend-services get-health my-backend --global
│   │          └── All UNHEALTHY? → Check:
│   │              ├── Health check path returns 200?
│   │              ├── Firewall allows HC probe IPs (35.191.0.0/16, 130.211.0.0/22)?
│   │              └── Backend port matches health check port?
│   ├── 502 → Check: Protocol match (HTTP vs HTTPS)? Backend app running?
│   │          Backend logs for stack traces?
│   └── 504 → Check: Backend response time vs LB timeout setting.
│              Increase: --timeout on backend service.
│
├── Is error 4xx (403/404)?
│   ├── 403 → Check: Cloud Armor rules? IAP policy? Target proxy SSL cert valid?
│   └── 404 → Check: URL map path rules. Export map and validate routing logic.
│
├── Latency high but no errors?
│   ├── Check backend latency metric vs total latency metric
│   │   ├── Backend latency HIGH → Optimize application; scale out backends
│   │   └── Backend latency OK, total HIGH → Check region routing (is traffic going far?)
│   └── Check: Session affinity causing hotspots on one backend?
│
└── Intermittent errors?
    ├── Check: Backend getting removed/replaced (rolling update)? → Increase connection draining timeout
    ├── Check: Health check flapping? → Tighten health check thresholds
    └── Check: Backends autoscaling down too aggressively?
```

---

## 9. Key Limits & Quotas

| Resource | Default Limit | Notes |
|---|---|---|
| **Global forwarding rules per project** | 75 | Can be raised |
| **Regional forwarding rules per region** | 75 | Can be raised |
| **Backend services per project** | 100 | Can be raised |
| **Backends per backend service** | 50 | MIGs or NEGs |
| **URL maps per project** | 20 | Can be raised |
| **URL map host rules per map** | 50 | |
| **URL map path rules per matcher** | 200 | Can be raised to 1000 |
| **SSL certificates per target HTTPS proxy** | 15 | |
| **SSL certificates per project** | 50 | Can be raised |
| **Target proxies per project** | 50 | |
| **Max request/response header size** | 64 KB / 256 KB | |
| **Connection draining timeout** | 0 – 3600s | 300s default |
| **Backend service timeout** | 1 – 86400s | 30s default |
| **Cloud Armor security policies** | 10 per project | Can be raised |
| **Cloud Armor rules per policy** | 200 | |
| **NEGs per project** | 700 | |
| **Endpoints per NEG** | 250 (zonal), 2000 (internet) | |
| **Health checks per project** | 1000 | |

> 📝 To request quota increases: **GCP Console → IAM & Admin → Quotas**

---

## 10. Gotchas & Best Practices

### Common Pitfalls

- **❌ Forgetting health check firewall rules** — The #1 cause of 503s. Always allow `35.191.0.0/16` and `130.211.0.0/22` to your backends on the health check port. Don't rely on default firewall rules.

- **❌ Global vs. Regional scope mismatch** — A global backend service cannot be attached to a regional URL map and vice versa. Keep scope consistent throughout the entire chain: forwarding rule → proxy → URL map → backend service.

- **❌ Named port mismatch** — Backend services using `--protocol=HTTP` with a MIG require the MIG to have a named port (`http:80`) matching `--port-name`. Unnamed ports cause connection failures.

- **❌ Using HTTP health check on HTTPS backend** — If your backend serves HTTPS, use an HTTPS health check. The LB will talk to backends using the backend service protocol, not the health check protocol by default.

- **❌ SSL cert not provisioning** — Google-managed certs require DNS to resolve to the LB's IP *before* provisioning starts. Creating the cert before the forwarding rule exists will cause it to stay in `PROVISIONING` state indefinitely.

- **❌ Internal LB missing proxy-only subnet** — Internal Application LBs (L7) require a dedicated proxy-only subnet (`--purpose=REGIONAL_MANAGED_PROXY`) in the region. Missing this causes LB creation to fail.

- **❌ Client IP not preserved with proxy LBs** — TCP Proxy, SSL Proxy, and HTTP(S) LBs terminate the connection. Use `X-Forwarded-For` header (L7) or PROXY Protocol (L4 proxy) to get original client IP. Internal Passthrough and External Passthrough preserve client IP natively.

- **❌ Cloud Armor not available on Internal LBs** — Cloud Armor only protects **External** Application LBs. Use VPC firewall rules and VPC Service Controls for internal traffic security.

- **❌ Timeout too short for streaming/websockets** — Default 30s timeout will kill long-lived connections. Set `--timeout=86400` (24 hours) for WebSocket backends.

- **❌ Draining timeout too short during rolling updates** — If backends are removed faster than in-flight requests complete, users see errors. Match `--connection-draining-timeout` to your slowest expected request duration.

---

### Best Practices

- **✅ Always use Managed Instance Groups** with autoscaling as backends — enables automatic scaling and health-based replacement.

- **✅ Use GKE container-native load balancing** (NEG-based) instead of node-level load balancing. Pod IPs are registered directly, eliminating the extra hop through `kube-proxy`.

- **✅ Enable access logging on critical backend services** with `--logging-sample-rate=1.0` during rollouts; reduce to 0.1 in steady state to control costs.

- **✅ Use separate backend services** for different microservices even if they share a URL map. This gives independent health checks, timeouts, and security policies.

- **✅ Reserve static IPs before creating LBs** — avoids IP changes on LB recreation and allows DNS pre-staging.

- **✅ Use Certificate Manager for wildcard certs** — Google-managed certs via `compute ssl-certificates` don't support wildcards. Certificate Manager does.

- **✅ Implement HTTP → HTTPS redirect** using a separate URL map with `default_url_redirect` instead of handling it in the application layer.

- **✅ Set Cloud Armor in preview mode first** — use `--action=allow` with `preview=true` on new rules to see what would be blocked before enforcing.

- **✅ Tag your backend VMs** consistently (e.g., `lb-backend`) and write firewall rules targeting those tags — avoids unintentional rules that apply to all VMs.

- **✅ Use `--global-access` on Internal LBs** when clients and backends span multiple regions within the same VPC — avoids creating duplicate LBs per region.

- **✅ Test health check paths** — ensure they return 200 without authentication, don't trigger DB queries on every probe, and respond quickly (<5s).

- **✅ Monitor backend health continuously** — set alerting on `loadbalancing.googleapis.com/https/backend_request_count` split by `response_code_class` to catch degradation early.

---

*Last updated: March 2026 | Based on GCP documentation and production patterns*
