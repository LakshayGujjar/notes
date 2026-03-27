# GCP Cloud CDN — Comprehensive Cheatsheet

> **TL;DR — Key Cloud CDN Decisions**
> - Cloud CDN only works **behind a Cloud Load Balancer** — it cannot be attached directly to origins.
> - Default cache mode is `CACHE_ALL_STATIC`; use `FORCE_CACHE_ALL` for APIs/dynamic content you explicitly want cached.
> - **Never use invalidation for static assets** — use versioned file names (e.g., `app.v2.js`) instead.
> - Responses with `Set-Cookie` or `Authorization` headers **will not be cached** by default.
> - Maximize cache hit ratio by customizing cache keys to strip irrelevant query params and headers.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Core Concepts](#2-core-concepts)
3. [Cache Key Customization](#3-cache-key-customization)
4. [Origin Configuration](#4-origin-configuration)
5. [gcloud CLI Cheatsheet](#5-gcloud-cli-cheatsheet)
6. [Terraform Examples](#6-terraform-examples)
7. [Signed URLs & Signed Cookies](#7-signed-urls--signed-cookies)
8. [Cache Invalidation](#8-cache-invalidation)
9. [Observability & Troubleshooting](#9-observability--troubleshooting)
10. [Key Limits & Quotas](#10-key-limits--quotas)
11. [Gotchas & Best Practices](#11-gotchas--best-practices)

---

## 1. Overview

**GCP Cloud CDN** (Content Delivery Network) is a globally distributed caching layer built on Google's network of edge Points of Presence (PoPs). It caches responses from your origins at the edge, reducing latency for end users and offloading traffic from your backends.

**How it works (high level):**
- Cloud CDN sits **between the internet and your origin**, attached to a Cloud Load Balancer backend service or backend bucket.
- When a user requests a cacheable resource, Cloud CDN checks if a fresh copy exists at the nearest edge PoP.
- On a **cache hit**: response is served directly from the edge — no origin contact.
- On a **cache miss**: request is forwarded to the origin, response is served to the user, and a copy is stored at the edge for future requests.

> ⚠️ **Architectural constraint**: Cloud CDN is **not a standalone service**. It must be enabled on a **Cloud Load Balancer backend service or backend bucket**. There is no way to use Cloud CDN without a Cloud LB in front.

---

### Cloud CDN vs. No CDN

| Dimension | Without CDN | With Cloud CDN |
|---|---|---|
| **Latency** | Round-trip to origin region | Served from nearest edge PoP (often <10ms) |
| **Origin load** | Every request hits origin | Only cache misses reach origin |
| **Availability** | Origin outage = full outage | Cached content survives short origin outages |
| **Bandwidth cost** | Full egress from origin | Reduced origin egress; CDN egress pricing |
| **Global performance** | Degrades with distance from origin | Consistent globally |
| **Cache control** | App-controlled headers only | App headers + CDN-level override (TTL, mode) |
| **Security** | WAF at LB only | WAF + signed URLs/cookies for private content |

---

### Google's Edge Network

- **130+ PoPs** worldwide across 200+ countries and territories
- Traffic routed via Google's private backbone — not public internet — between PoPs and origins
- Anycast IP means the nearest PoP answers every request automatically
- PoP caches are **multi-tier**: local edge cache + regional cache (origin shield)

---

## 2. Core Concepts

### Cache Hit vs. Cache Miss Flow

```
CACHE HIT:
                                        ┌─────────────┐
User ──── HTTPS GET /logo.png ────────► │  Edge PoP   │ ◄── Cached? YES
          ◄── 200 OK (Age: 3600) ─────  │  (Cache)    │
                                        └─────────────┘
                                        (Origin never contacted)

CACHE MISS:
                                        ┌─────────────┐     ┌─────────────┐
User ──── HTTPS GET /logo.png ────────► │  Edge PoP   │────►│  Cloud LB   │
          ◄── 200 OK (Age: 0) ────────  │  (No cache) │◄────│  + Origin   │
                                        └─────────────┘     └─────────────┘
                                        (Response stored in edge cache)
```

**Cache response headers to know:**
| Header | Meaning |
|---|---|
| `Age: N` | Response was served from cache; N = seconds since cached |
| `X-Cache-Lookup: HIT` | CDN found a cached response |
| `X-Cache-Lookup: MISS` | CDN had no cached copy |
| `Cache-Control: max-age=N` | Resource is fresh for N seconds |
| `Cache-Control: no-store` | CDN must not cache this response |
| `Cache-Control: no-cache` | CDN must revalidate before serving |

---

### Cache Keys

A **cache key** is the unique identifier used to look up a cached object. Two requests that produce the same cache key will share the same cached response.

**Default cache key components:**
```
Protocol (http/https) + Host + URL path + Query string
```

Example:
```
https://example.com/images/logo.png?v=1  →  unique cache key
https://example.com/images/logo.png?v=2  →  different cache key
http://example.com/images/logo.png?v=1   →  different cache key (protocol differs)
```

**Why cache keys matter:**
- Too broad → different users get each other's personalized content (correctness risk)
- Too narrow → unnecessary cache misses, low hit ratio (performance risk)
- Customizing keys (dropping irrelevant query params, adding relevant headers) is the #1 lever for improving hit ratio

---

### Cache Modes

| Mode | Behavior | When to Use |
|---|---|---|
| `CACHE_ALL_STATIC` | Caches static content types based on URL extension and `Content-Type`; respects `Cache-Control: no-store` / `private` | **Default.** Good for mixed apps with static + dynamic content |
| `USE_ORIGIN_HEADERS` | Only caches if origin sends explicit `Cache-Control: public, max-age=N`; no caching without header | When you want **origin to fully control** caching; dynamic APIs |
| `FORCE_CACHE_ALL` | Caches **all** responses regardless of `Cache-Control` headers (except `no-store`); applies `defaultTtl` when headers absent | Caching APIs or dynamic content you control; aggressive caching |

**Static content types cached by `CACHE_ALL_STATIC`:**
CSS, JS, HTML (with `Cache-Control`), fonts (woff, woff2, ttf), images (jpg, png, gif, webp, svg), video (mp4, webm), audio, PDF, ZIP.

```
CACHE_ALL_STATIC  ──►  Content-Type: text/css     → CACHED ✅
                  ──►  Content-Type: application/json → NOT cached ❌ (unless Cache-Control: public)
FORCE_CACHE_ALL   ──►  Content-Type: application/json → CACHED ✅
```

---

### TTL Settings

TTLs control how long responses are considered fresh at the CDN edge, independent of what the origin sends.

| TTL Setting | Description | Default |
|---|---|---|
| `defaultTtl` | Used when origin sends no `Cache-Control` or `Expires` | 3600s (1 hour) |
| `maxTtl` | Cap on how long a response is cached, even if origin says longer | 86400s (24 hours) |
| `clientTtl` | `max-age` sent to the client (browser) — can differ from edge TTL | Same as edge TTL |

**TTL interaction with `Cache-Control` headers:**

```
Origin Cache-Control: max-age=7200 (2 hours)
Cloud CDN maxTtl = 3600 (1 hour)
→ Edge caches for MIN(7200, 3600) = 3600s ✅

Origin Cache-Control: max-age=60 (1 min)
Cloud CDN defaultTtl = 3600
→ If mode is FORCE_CACHE_ALL: edge caches for 3600s (overrides origin)
→ If mode is USE_ORIGIN_HEADERS: edge caches for 60s (respects origin)

Origin sends no Cache-Control header
Cloud CDN defaultTtl = 3600
→ Edge caches for 3600s using defaultTtl
```

---

### Caching Layers

```
Browser Cache (client TTL)
        │
        ▼
  Edge PoP Cache (edge TTL / maxTtl)
        │  cache miss
        ▼
  Origin Shield (regional aggregation cache)  ← optional
        │  shield miss
        ▼
  Cloud Load Balancer
        │
        ▼
  Origin (MIG / GCS / NEG / Cloud Run)
```

| Layer | Controlled By | Scope |
|---|---|---|
| **Client (browser)** | `clientTtl` / `Cache-Control` sent to client | Per user device |
| **Edge PoP** | `defaultTtl`, `maxTtl`, cache mode | Nearest Google PoP |
| **Origin Shield** | Shield configuration (regional) | Entire region |
| **Origin** | Backend service / GCS bucket | Your infrastructure |

---

### Negative Caching

**Negative caching** caches error responses (4xx, 5xx) to prevent error storms from hammering the origin.

| Response Code | Default Cache Duration |
|---|---|
| 301, 308 | 10 minutes |
| 302, 307 | Not cached |
| 404 | 120 seconds |
| 405 | 60 seconds |
| 410 | 120 seconds |
| 5xx | Not cached by default |

```bash
# Enable negative caching with custom TTL per status code
gcloud compute backend-services update my-backend \
  --global \
  --negative-caching \
  --negative-caching-policy='code=404,ttl=30' \  # Cache 404s for 30s
  --negative-caching-policy='code=410,ttl=60'    # Cache 410s for 60s
```

> **Why it matters**: Without negative caching, every 404 for a missing asset hits your origin. With it, the CDN absorbs repeated requests for non-existent resources.

---

### Signed URLs and Signed Cookies (Overview)

Used to restrict access to cached content to **authorized users only**:

| Feature | Signed URLs | Signed Cookies |
|---|---|---|
| Scope | Single resource | Multiple resources / entire path prefix |
| How delivered | URL parameter (`?Expires=...&Signature=...`) | `Cloud-CDN-Cookie` HTTP cookie |
| Best for | Short-lived file downloads, one-off links | Video streaming, authenticated sections |
| Key management | Signing key on backend service | Same signing key |

> Full walkthrough in [Section 7](#7-signed-urls--signed-cookies).

---

### Cache Invalidation (Overview)

Invalidation **purges cached content** from all edge nodes before its TTL expires.

- Propagation time: typically **<1 minute** globally
- First **1,000 invalidations per day** are free; $0.005/request thereafter
- Supports exact paths (`/images/logo.png`) and wildcards (`/images/*`)

> Full walkthrough in [Section 8](#8-cache-invalidation).

---

## 3. Cache Key Customization

### Default vs. Custom Cache Key

```
Default key:    scheme + host + path + full query string
Custom key:     scheme + host + path + [selected params] + [selected headers] + [selected cookies]
```

### Excluding Query Parameters

Use when query params are tracking/analytics and don't affect content:

```bash
# Exclude UTM params (keeps ?color=blue but drops ?utm_source=google)
gcloud compute backend-services update my-backend \
  --global \
  --cache-key-query-string-whitelist=color,size,variant
  # Only 'color', 'size', 'variant' included in cache key; all others excluded

# Alternatively, exclude specific params (include everything EXCEPT listed)
gcloud compute backend-services update my-backend \
  --global \
  --cache-key-query-string-blacklist=utm_source,utm_medium,utm_campaign,fbclid
```

**Impact example:**
```
Without customization:
  /page?color=red&utm_source=google  →  unique cache key (cache miss)
  /page?color=red&utm_medium=email   →  different cache key (another cache miss)

With utm_* excluded from key:
  /page?color=red&utm_source=google  →  same key: /page?color=red  (cache hit on 2nd request)
  /page?color=red&utm_medium=email   →  same key: /page?color=red  ✅ Hit ratio improved
```

---

### Including HTTP Headers in Cache Key

Add request headers to the cache key when responses differ by header value:

```bash
# Include Accept-Language to serve locale-specific cached versions
gcloud compute backend-services update my-backend \
  --global \
  --cache-key-include-http-header=Accept-Language

# Include multiple headers
gcloud compute backend-services update my-backend \
  --global \
  --cache-key-include-http-header=Accept-Language,X-Country-Code
```

> ⚠️ Every unique header value creates a separate cache entry. Adding high-cardinality headers (e.g., `User-Agent`) will **destroy your hit ratio**. Only add headers with a small, bounded set of values.

---

### Including Cookies in Cache Key

```bash
# Include a specific cookie (e.g., 'user_plan' with values 'free' or 'premium')
gcloud compute backend-services update my-backend \
  --global \
  --cache-key-include-named-cookie=user_plan

# Include multiple cookies
gcloud compute backend-services update my-backend \
  --global \
  --cache-key-include-named-cookie=user_plan,region_pref
```

> Including cookies in the cache key means the CDN can serve **different cached responses** per cookie value — useful for A/B testing or plan-based content variation without bypassing the CDN entirely.

---

### Cache Key Policy — YAML Example

Export and edit the full backend service config:

```bash
# Export backend service to YAML
gcloud compute backend-services export my-backend \
  --global \
  --destination backend.yaml
```

```yaml
# backend.yaml — cache key policy section
cdnPolicy:
  cacheMode: CACHE_ALL_STATIC
  defaultTtl: 3600s
  maxTtl: 86400s
  clientTtl: 3600s
  negativeCaching: true
  cacheKeyPolicy:
    includeProtocol: false          # Treat http and https as same key
    includeHost: true               # Include Host header in key
    includeQueryString: true        # Include query string
    queryStringWhitelist:           # Only these query params in key
      - color
      - size
    includeHttpHeaders:             # Include these request headers in key
      - Accept-Language
    includeNamedCookies:            # Include these cookies in key
      - user_plan
```

```bash
# Apply updated config
gcloud compute backend-services import my-backend \
  --global \
  --source backend.yaml
```

---

### Cache Key Impact Summary

| Customization | Hit Ratio | Risk |
|---|---|---|
| Exclude tracking query params | ⬆️ Higher | Low — params don't affect content |
| Whitelist only relevant params | ⬆️ Higher | Medium — must audit all params |
| Add high-cardinality header | ⬇️ Lower | High — cache fragmentation |
| Add low-cardinality header (e.g., Accept-Language with 3 values) | Neutral / ⬆️ | Low — correct per-locale caching |
| Add session/auth cookie | ⬇️ Much lower | Very High — per-user cache entries |

---

## 4. Origin Configuration

### Supported Origin Types

| Origin Type | Backed By | Use Case |
|---|---|---|
| **Backend Service (MIG)** | Managed Instance Group of VMs | Dynamic apps, APIs behind CDN |
| **Backend Service (NEG — Zonal)** | GKE pods, VM IP:port | Container-native backends |
| **Backend Service (NEG — Serverless)** | Cloud Run, App Engine, Cloud Functions | Serverless origins |
| **Backend Service (NEG — Internet)** | External IP/FQDN | Custom / third-party origin |
| **Backend Bucket** | Google Cloud Storage bucket | Static asset hosting (images, JS, CSS, video) |

> Cloud CDN is enabled **per backend** — a single LB can have CDN-enabled and non-CDN backends simultaneously.

---

### Backend Buckets (GCS)

Backend Buckets are the simplest Cloud CDN setup — attach a GCS bucket as a CDN origin:

```bash
# Create a backend bucket
gcloud compute backend-buckets create my-cdn-bucket \
  --gcs-bucket-name=my-gcs-bucket \
  --enable-cdn \
  --cache-mode=CACHE_ALL_STATIC \
  --default-ttl=3600 \
  --max-ttl=86400
```

**GCS bucket permissions for CDN:**
- The bucket must be **publicly readable** OR CDN must have access via signed URLs.
- For public access: grant `allUsers` the `Storage Object Viewer` role.
- For private + signed URLs: bucket can remain private; CDN accesses via service account.

```bash
# Make bucket publicly readable (for open CDN serving)
gsutil iam ch allUsers:objectViewer gs://my-gcs-bucket

# Keep bucket private and use signed URLs for access control
# (No public IAM grant needed — handled by signing key on backend bucket)
```

---

### Origin Shield

**Origin Shield** is an optional additional caching layer — a **regional aggregation cache** that sits between edge PoPs and your origin. All PoPs in a region route cache misses through the shield before hitting the origin.

```
Without Shield:
  PoP (Tokyo) ──► Origin (us-central1)   (cache miss)
  PoP (Osaka) ──► Origin (us-central1)   (same miss, 2 origin hits)

With Shield (us-central1):
  PoP (Tokyo) ──► Shield ──► Origin      (1st miss: shield+origin hit)
  PoP (Osaka) ──► Shield ──► (HIT!)      (2nd miss: only shield hit, origin protected)
```

**When to use Origin Shield:**
- Origin is expensive to hit (compute-heavy or paid API)
- Content has high inter-PoP redundancy (same asset requested from many regions)
- Origin is geographically distant from most users
- You want to minimize inter-continental origin egress costs

**Enable Origin Shield:**
```bash
gcloud compute backend-services update my-backend \
  --global \
  --origin-shield-region=us-central1   # Choose region closest to your origin
```

**Supported shield regions:** `us-east1`, `us-central1`, `us-west1`, `southamerica-east1`, `europe-west1`, `europe-west4`, `asia-east1`, `asia-northeast1`, `asia-south1`, `australia-southeast1`.

---

### Custom Origins via Internet NEGs

Route CDN traffic to **external origins** (not in GCP) using Internet NEGs:

```bash
# Create an Internet NEG pointing to an external origin
gcloud compute network-endpoint-groups create my-external-neg \
  --network-endpoint-type=INTERNET_FQDN_PORT \
  --global

gcloud compute network-endpoint-groups update my-external-neg \
  --global \
  --add-endpoint='fqdn=origin.mycompany.com,port=443'

# Create backend service using the NEG
gcloud compute backend-services create my-custom-origin-svc \
  --global \
  --protocol=HTTPS \
  --enable-cdn

gcloud compute backend-services add-backend my-custom-origin-svc \
  --global \
  --network-endpoint-group=my-external-neg \
  --network-endpoint-group-zone=global
```

---

### Origin Failover

Configure a **primary + backup backend** for automatic failover on 5xx or connectivity failure:

```bash
# Set up failover policy on a backend service
gcloud compute backend-services update my-backend \
  --global \
  --failover-ratio=0.1   # Failover when >10% of primary traffic fails
```

For full failover config (primary/backup groups), use the REST API or Terraform — `gcloud` supports failover ratio but full primary/backup assignment requires the API.

---

## 5. gcloud CLI Cheatsheet

### Enable / Disable Cloud CDN

```bash
# Enable CDN on a backend SERVICE (e.g., backed by MIG or NEG)
gcloud compute backend-services update my-backend-svc \
  --global \
  --enable-cdn                          # Turn on Cloud CDN

# Disable CDN on a backend service
gcloud compute backend-services update my-backend-svc \
  --global \
  --no-enable-cdn                       # Turn off Cloud CDN

# Enable CDN on a BACKEND BUCKET (GCS origin)
gcloud compute backend-buckets update my-cdn-bucket \
  --enable-cdn                          # No --global needed for backend buckets

# Disable CDN on a backend bucket
gcloud compute backend-buckets update my-cdn-bucket \
  --no-enable-cdn
```

---

### Set Cache Mode, TTLs, and Cache Key Policy

```bash
# Set cache mode
gcloud compute backend-services update my-backend-svc \
  --global \
  --cache-mode=CACHE_ALL_STATIC         # or USE_ORIGIN_HEADERS, FORCE_CACHE_ALL

# Set TTL values
gcloud compute backend-services update my-backend-svc \
  --global \
  --default-ttl=3600 \                  # 1 hour — used when origin has no Cache-Control
  --max-ttl=86400 \                     # 24 hours — hard cap on edge TTL
  --client-ttl=600                      # 10 minutes — max-age sent to browser

# Set cache key: whitelist only specific query params
gcloud compute backend-services update my-backend-svc \
  --global \
  --cache-key-query-string-whitelist=color,size,format

# Set cache key: exclude specific query params
gcloud compute backend-services update my-backend-svc \
  --global \
  --cache-key-query-string-blacklist=utm_source,utm_medium,fbclid

# Set cache key: include a request header
gcloud compute backend-services update my-backend-svc \
  --global \
  --cache-key-include-http-header=Accept-Language

# Set cache key: include a named cookie
gcloud compute backend-services update my-backend-svc \
  --global \
  --cache-key-include-named-cookie=user_plan

# Enable negative caching with custom per-code TTL
gcloud compute backend-services update my-backend-svc \
  --global \
  --negative-caching \
  --negative-caching-policy='code=404,ttl=60' \
  --negative-caching-policy='code=410,ttl=300'
```

---

### Signed URL Keys

```bash
# Generate a random 256-bit signing key (base64url encoded)
python3 -c "import os,base64; print(base64.urlsafe_b64encode(os.urandom(32)).decode())" \
  > my-signing-key.txt

# Add signing key to a backend SERVICE
gcloud compute backend-services add-signed-url-key my-backend-svc \
  --global \
  --key-name=my-key-v1 \                # Key name referenced when signing URLs
  --key-file=my-signing-key.txt         # File with base64url-encoded key bytes

# Add signing key to a BACKEND BUCKET
gcloud compute backend-buckets add-signed-url-key my-cdn-bucket \
  --key-name=my-key-v1 \
  --key-file=my-signing-key.txt

# List signing keys on a backend service (names only — key bytes never returned)
gcloud compute backend-services describe my-backend-svc --global \
  --format="value(cdnPolicy.signedUrlKeyNames)"

# Delete (revoke) a signing key
gcloud compute backend-services delete-signed-url-key my-backend-svc \
  --global \
  --key-name=my-key-v1

gcloud compute backend-buckets delete-signed-url-key my-cdn-bucket \
  --key-name=my-key-v1
```

---

### Cache Invalidation

```bash
# Invalidate a single exact path
gcloud compute url-maps invalidate-cdn-cache my-url-map \
  --global \
  --path="/images/logo.png"

# Invalidate all files under a path prefix (wildcard)
gcloud compute url-maps invalidate-cdn-cache my-url-map \
  --global \
  --path="/images/*"

# Invalidate everything (use with extreme caution)
gcloud compute url-maps invalidate-cdn-cache my-url-map \
  --global \
  --path="/*"

# Invalidate with a specific host (for multi-host URL maps)
gcloud compute url-maps invalidate-cdn-cache my-url-map \
  --global \
  --host=static.example.com \
  --path="/css/*"

# Check invalidation status
gcloud compute operations list \
  --filter="operationType=invalidateCache" \
  --sort-by=~insertTime \
  --limit=10
```

---

### View CDN Configuration

```bash
# Show CDN config on a backend service
gcloud compute backend-services describe my-backend-svc \
  --global \
  --format="yaml(cdnPolicy)"

# Show CDN config on a backend bucket
gcloud compute backend-buckets describe my-cdn-bucket \
  --format="yaml(cdnPolicy)"

# List all backend services with CDN enabled
gcloud compute backend-services list \
  --global \
  --filter="cdnPolicy.cacheMode:*" \
  --format="table(name,cdnPolicy.cacheMode,cdnPolicy.defaultTtl)"

# Check CDN enabled/disabled status across all backends
gcloud compute backend-services list \
  --global \
  --format="table(name,enableCDN)"
```

---

## 6. Terraform Examples

### Cloud CDN on a Backend Service (MIG Origin)

```hcl
# ── health check ─────────────────────────────────────────
resource "google_compute_health_check" "default" {
  name = "cdn-backend-hc"
  http_health_check {
    port         = 80
    request_path = "/healthz"
  }
}

# ── backend service with CDN enabled ─────────────────────
resource "google_compute_backend_service" "cdn_backend" {
  name                  = "my-cdn-backend"
  protocol              = "HTTP"
  port_name             = "http"
  load_balancing_scheme = "EXTERNAL_MANAGED"
  timeout_sec           = 30
  health_checks         = [google_compute_health_check.default.id]

  backend {
    group           = google_compute_instance_group_manager.default.instance_group
    balancing_mode  = "UTILIZATION"
    max_utilization = 0.8
  }

  # ── CDN policy ─────────────────────────────────────────
  enable_cdn = true

  cdn_policy {
    cache_mode        = "CACHE_ALL_STATIC"   # Cache static content by default
    default_ttl       = 3600                 # 1 hour when no Cache-Control
    max_ttl           = 86400                # 24 hour hard cap
    client_ttl        = 3600                 # Browser cache TTL
    negative_caching  = true                 # Cache 404/410 responses

    negative_caching_policy {
      code = 404
      ttl  = 60
    }

    cache_key_policy {
      include_protocol       = false          # http and https share same cache key
      include_host           = true
      include_query_string   = true
      query_string_whitelist = ["color", "size", "format"]  # Only these params in key
      include_http_headers   = ["Accept-Language"]           # Vary by language
    }

    # Enable origin shield in us-central1
    serve_while_stale = 86400                # Serve stale for up to 24h while revalidating
  }

  log_config {
    enable      = true
    sample_rate = 1.0
  }
}

# ── signed URL key (stored in Secret Manager, referenced here) ──
resource "google_compute_backend_service_signed_url_key" "cdn_key" {
  name            = "cdn-signing-key-v1"
  key_value       = var.cdn_signing_key     # Base64url-encoded 32-byte key
  backend_service = google_compute_backend_service.cdn_backend.name
}
```

---

### Cloud CDN on a GCS Backend Bucket

```hcl
# ── GCS bucket ───────────────────────────────────────────
resource "google_storage_bucket" "cdn_assets" {
  name          = "my-cdn-static-assets"
  location      = "US"
  force_destroy = false

  uniform_bucket_level_access = true
}

# ── Make bucket publicly readable ────────────────────────
resource "google_storage_bucket_iam_member" "public_read" {
  bucket = google_storage_bucket.cdn_assets.name
  role   = "roles/storage.objectViewer"
  member = "allUsers"
}

# ── Backend bucket with CDN ───────────────────────────────
resource "google_compute_backend_bucket" "cdn_bucket" {
  name        = "my-cdn-backend-bucket"
  bucket_name = google_storage_bucket.cdn_assets.name
  enable_cdn  = true

  cdn_policy {
    cache_mode  = "CACHE_ALL_STATIC"
    default_ttl = 3600
    max_ttl     = 604800    # 7 days for static assets
    client_ttl  = 86400     # 24 hours browser cache

    negative_caching = true
    negative_caching_policy {
      code = 404
      ttl  = 30
    }

    cache_key_policy {
      include_http_headers   = []           # No extra headers in key
      query_string_whitelist = ["v"]        # Only include ?v= (version param)
    }
  }
}

# ── Attach to URL map ─────────────────────────────────────
resource "google_compute_url_map" "default" {
  name            = "my-url-map"
  default_service = google_compute_backend_service.cdn_backend.id

  host_rule {
    hosts        = ["static.example.com"]
    path_matcher = "static"
  }

  path_matcher {
    name            = "static"
    default_service = google_compute_backend_bucket.cdn_bucket.id
  }
}
```

---

### Signed URL Key via Terraform

```hcl
# ── Store key in Secret Manager ───────────────────────────
resource "google_secret_manager_secret" "cdn_key" {
  secret_id = "cdn-signing-key-v1"
  replication { automatic = true }
}

resource "google_secret_manager_secret_version" "cdn_key_v1" {
  secret      = google_secret_manager_secret.cdn_key.id
  secret_data = var.cdn_signing_key   # Pass via TF_VAR or -var flag; never hardcode
}

# ── Attach key to backend service ────────────────────────
resource "google_compute_backend_service_signed_url_key" "key" {
  name            = "signing-key-v1"
  key_value       = var.cdn_signing_key
  backend_service = google_compute_backend_service.cdn_backend.name
}

# ── Attach key to backend bucket ─────────────────────────
resource "google_compute_backend_bucket_signed_url_key" "bucket_key" {
  name           = "bucket-signing-key-v1"
  key_value      = var.cdn_signing_key
  backend_bucket = google_compute_backend_bucket.cdn_bucket.name
}
```

---

## 7. Signed URLs & Signed Cookies

### When to Use Which

| Scenario | Use |
|---|---|
| Sharing a single downloadable file (PDF, video, ZIP) | **Signed URL** |
| User accesses many protected resources (streaming site, gated section) | **Signed Cookie** |
| Time-limited access (expires in 15 minutes) | **Signed URL** |
| Authenticated dashboard with many assets | **Signed Cookie** |
| CDN-served content without login (public) | Neither — use unsigned CDN |

---

### Step 1: Generate and Register a Signing Key

```bash
# Generate a 256-bit (32-byte) cryptographically random key
python3 -c "
import os, base64
key = os.urandom(32)
print(base64.urlsafe_b64encode(key).decode())
" > cdn-signing-key.txt

cat cdn-signing-key.txt
# Example output: abc123XYZ_abcdefghijklmnopqrstuvwxyz01234567=

# Register the key with a backend service
gcloud compute backend-services add-signed-url-key my-backend \
  --global \
  --key-name=key-v1 \
  --key-file=cdn-signing-key.txt

# Register with a backend bucket
gcloud compute backend-buckets add-signed-url-key my-cdn-bucket \
  --key-name=key-v1 \
  --key-file=cdn-signing-key.txt
```

---

### Step 2: Generate a Signed URL (Python)

```python
import base64
import datetime
import hashlib
import hmac
import urllib.parse


def sign_url(
    url: str,
    key_name: str,
    key_value_base64: str,
    expiration_minutes: int = 60,
) -> str:
    """
    Generate a Cloud CDN signed URL.

    Args:
        url: The full URL to sign (e.g., https://cdn.example.com/video.mp4)
        key_name: The key name registered on the backend (e.g., 'key-v1')
        key_value_base64: The base64url-encoded signing key value
        expiration_minutes: How many minutes until the URL expires

    Returns:
        Signed URL string
    """
    # Decode the base64url key
    key_bytes = base64.urlsafe_b64decode(key_value_base64 + "==")  # pad if needed

    # Calculate expiration as Unix timestamp
    expiration = datetime.datetime.utcnow() + datetime.timedelta(minutes=expiration_minutes)
    expiration_ts = int(expiration.timestamp())

    # Append required query params
    parsed = urllib.parse.urlparse(url)
    params = urllib.parse.parse_qs(parsed.query, keep_blank_values=True)
    params["Expires"] = [str(expiration_ts)]
    params["KeyName"] = [key_name]

    # Rebuild URL without signature
    unsigned_url = urllib.parse.urlunparse(
        parsed._replace(query=urllib.parse.urlencode(params, doseq=True))
    )

    # Sign the URL using HMAC-SHA256
    signature = hmac.new(key_bytes, unsigned_url.encode("utf-8"), hashlib.sha256).digest()
    encoded_signature = base64.urlsafe_b64encode(signature).decode("utf-8").rstrip("=")

    # Append signature to URL
    return f"{unsigned_url}&Signature={encoded_signature}"


# ── Usage ─────────────────────────────────────────────────
if __name__ == "__main__":
    KEY_NAME = "key-v1"
    KEY_VALUE = "abc123XYZ_abcdefghijklmnopqrstuvwxyz01234567="  # from cdn-signing-key.txt

    signed = sign_url(
        url="https://cdn.example.com/protected/video.mp4",
        key_name=KEY_NAME,
        key_value_base64=KEY_VALUE,
        expiration_minutes=60,
    )
    print(signed)
    # https://cdn.example.com/protected/video.mp4?Expires=1700000000&KeyName=key-v1&Signature=...
```

---

### Step 3: Generate a Signed Cookie (Python)

```python
import base64
import datetime
import hashlib
import hmac
import urllib.parse


def sign_cookie(
    url_prefix: str,
    key_name: str,
    key_value_base64: str,
    expiration_minutes: int = 120,
) -> dict:
    """
    Generate a Cloud CDN signed cookie granting access to a URL prefix.

    Args:
        url_prefix: URL prefix to protect (e.g., https://cdn.example.com/protected/)
        key_name: Signing key name registered on backend
        key_value_base64: Base64url-encoded key bytes
        expiration_minutes: Cookie validity in minutes

    Returns:
        dict with cookie name and value to set in Set-Cookie header
    """
    key_bytes = base64.urlsafe_b64decode(key_value_base64 + "==")

    expiration = datetime.datetime.utcnow() + datetime.timedelta(minutes=expiration_minutes)
    expiration_ts = int(expiration.timestamp())

    # Build the cookie value string (URL-encoded prefix)
    encoded_prefix = base64.urlsafe_b64encode(url_prefix.encode()).decode().rstrip("=")
    policy_str = f"URLPrefix={encoded_prefix}:Expires={expiration_ts}:KeyName={key_name}"

    # Sign the policy string
    signature = hmac.new(key_bytes, policy_str.encode("utf-8"), hashlib.sha256).digest()
    encoded_signature = base64.urlsafe_b64encode(signature).decode().rstrip("=")

    cookie_value = f"{policy_str}:Signature={encoded_signature}"

    return {
        "name": "Cloud-CDN-Cookie",
        "value": cookie_value,
        "set_cookie_header": (
            f"Cloud-CDN-Cookie={cookie_value}; "
            f"Path=/protected/; "
            f"HttpOnly; Secure; SameSite=Strict; "
            f"Max-Age={expiration_minutes * 60}"
        ),
    }


# ── Usage (in a Flask/FastAPI response) ──────────────────
if __name__ == "__main__":
    cookie = sign_cookie(
        url_prefix="https://cdn.example.com/protected/",
        key_name="key-v1",
        key_value_base64="abc123XYZ_abcdefghijklmnopqrstuvwxyz01234567=",
        expiration_minutes=120,
    )
    print(cookie["set_cookie_header"])
    # Cloud-CDN-Cookie=URLPrefix=...:Expires=...:KeyName=key-v1:Signature=...; Path=/protected/; ...
```

---

### Key Rotation Best Practice

```
Step 1: Add NEW key (key-v2) to backend — both key-v1 and key-v2 are valid simultaneously
Step 2: Update signing service to generate URLs/cookies using key-v2
Step 3: Wait for all key-v1 signed URLs/cookies to expire (or revoke early if security incident)
Step 4: Remove old key (key-v1) from backend

Never delete the active key before the new key is deployed — this will cause 403 errors for
all users holding valid signed URLs/cookies issued with the old key.
```

```bash
# Add new key (step 1)
gcloud compute backend-services add-signed-url-key my-backend \
  --global \
  --key-name=key-v2 \
  --key-file=new-signing-key.txt

# Delete old key (step 4 — only after v1 URLs have expired)
gcloud compute backend-services delete-signed-url-key my-backend \
  --global \
  --key-name=key-v1
```

---

## 8. Cache Invalidation

### How Invalidation Works

```
gcloud url-maps invalidate-cdn-cache ...
          │
          ▼
  GCP Control Plane ──► Broadcasts invalidation to ALL edge PoPs globally
                         (propagation: typically <1 minute, guaranteed <5 minutes)

  Next request to any PoP for that path:
          │
          ▼
  Edge PoP: cached entry marked invalid
          │
          ▼
  Origin fetched → new response cached at edge
```

- Invalidation does **not** delete the object — it marks the edge entry as stale, so the next request triggers a fresh origin fetch.
- **Wildcard invalidation** (`/images/*`) affects all paths under the prefix across all edge PoPs.
- Invalidation does **not** affect client (browser) caches — users may still see stale content based on their local `Cache-Control` / `clientTtl`.

---

### Path-Based vs. Wildcard Invalidation

```bash
# Exact path — only invalidates this one URL
gcloud compute url-maps invalidate-cdn-cache my-url-map \
  --global \
  --path="/images/hero.jpg"

# Prefix wildcard — invalidates all objects under /images/
gcloud compute url-maps invalidate-cdn-cache my-url-map \
  --global \
  --path="/images/*"

# Root wildcard — invalidates entire CDN (use only in emergencies)
gcloud compute url-maps invalidate-cdn-cache my-url-map \
  --global \
  --path="/*"

# Scoped to a specific host
gcloud compute url-maps invalidate-cdn-cache my-url-map \
  --global \
  --host=cdn.example.com \
  --path="/js/*"
```

---

### Cost Implications

| Invalidations per Day | Cost |
|---|---|
| First 1,000 | **Free** |
| Beyond 1,000 | **$0.005 per invalidation** |

> A single wildcard invalidation (`/images/*`) counts as **1 invalidation** regardless of how many objects it affects. However, each unique path you pass to the API counts separately.

---

### When NOT to Use Invalidation

| Scenario | Better Approach |
|---|---|
| Deploying new JS/CSS build | Use versioned filenames: `app.abc123.js` (content hash) |
| Updating product images | Append `?v=2` to image URLs in HTML |
| Full site redeploy | Use a deploy-time generated prefix: `/v42/assets/...` |
| Frequent content updates | Set a short `defaultTtl` (e.g., 60s) rather than invalidating |
| A/B test variant swap | Use separate CDN paths or use Traffic Director |

> **Why versioned URLs > invalidation**: Versioned URLs create a new cache key automatically — no propagation delay, no cost, no stale window, and old versions stay cached for users still on old HTML.

---

## 9. Observability & Troubleshooting

### Cloud Monitoring Metrics

| Metric | Description | How to Use |
|---|---|---|
| `loadbalancing.googleapis.com/https/request_count` split by `cache_result` | Cache HIT vs MISS count | Track hit ratio over time |
| `loadbalancing.googleapis.com/https/total_latencies` | End-to-end latency (p50/p95/p99) | Compare HIT vs MISS latency |
| `loadbalancing.googleapis.com/https/backend_latencies` | Origin response time | Detect slow origin |
| `loadbalancing.googleapis.com/https/response_bytes_count` split by `cache_result` | Bytes served from cache vs origin | Calculate byte hit ratio |
| `loadbalancing.googleapis.com/https/request_bytes_count` | Inbound request bytes | Baseline traffic |

**Key derived metrics to alert on:**

```
Cache Hit Ratio (by request) = HIT requests / Total requests
Byte Hit Ratio               = Bytes from cache / Total bytes served
Origin Offload               = 1 - (MISS requests / Total requests)
```

**MQL Query — Cache hit ratio over time:**
```
fetch https_lb_rule
| metric 'loadbalancing.googleapis.com/https/request_count'
| filter resource.url_map_name == 'my-url-map'
| group_by 5m, [cache_result: label_value(metric.cache_result),
               count: aggregate(value.request_count)]
| every 5m
```

---

### Enabling CDN Access Logs

Cloud CDN logging is tied to the **LB access log** — enable it on the backend service:

```bash
# Enable logging on the backend service (100% sample rate for CDN debugging)
gcloud compute backend-services update my-backend-svc \
  --global \
  --enable-logging \
  --logging-sample-rate=1.0   # Log all requests during debugging; lower in production
```

**CDN-specific log fields:**
| Field | Description |
|---|---|
| `httpRequest.cacheHit` | `true` if served from cache |
| `httpRequest.cacheLookup` | `true` if cache was consulted |
| `httpRequest.cacheValidatedWithOriginServer` | `true` if origin revalidated a stale entry |
| `httpRequest.cacheFillBytes` | Bytes written to cache on a miss |
| `jsonPayload.statusDetails` | CDN decision string (e.g., `response_cache_miss`) |

**Sample Cloud Logging queries:**

```
# All cache misses for a specific path prefix
resource.type="http_load_balancer"
resource.labels.url_map_name="my-url-map"
httpRequest.cacheLookup=true
httpRequest.cacheHit=false
httpRequest.requestUrl=~"/images/.*"

# All 403s from signed URL validation failures
resource.type="http_load_balancer"
httpRequest.status=403
jsonPayload.statusDetails="request_blocked_by_cloud_cdn_security_policy"

# Requests that bypassed cache (no lookup at all)
resource.type="http_load_balancer"
httpRequest.cacheLookup=false

# High-latency origin fetches (>1 second)
resource.type="http_load_balancer"
httpRequest.cacheHit=false
httpRequest.latency>"1s"
```

---

### Troubleshooting Decision Tree

```
Request not being cached / low hit ratio?
│
├── Check: Is CDN enabled on the backend?
│   └── gcloud compute backend-services describe ... --format="value(enableCDN)"
│   └── NO → Enable CDN: --enable-cdn
│
├── Check: Does response have Cache-Control: no-store or private?
│   └── YES → Origin is opting out. Fix: Change app headers OR use FORCE_CACHE_ALL mode.
│
├── Check: Does response have Set-Cookie header?
│   └── YES → CDN skips caching. Fix: Remove Set-Cookie from cacheable responses.
│
├── Check: Does request have Authorization header?
│   └── YES → CDN bypasses cache. Fix: Use Signed URLs/Cookies instead of Authorization headers.
│
├── Check: Is cache mode correct?
│   └── CACHE_ALL_STATIC + Content-Type: application/json → Not cached by default.
│   └── Fix: Use FORCE_CACHE_ALL, or add Cache-Control: public, max-age=N from origin.
│
└── Check: Is cache key too granular?
    └── Too many unique query params → each param combo = new cache entry = low hit ratio
    └── Fix: Whitelist only essential query params in cacheKeyPolicy.

─────────────────────────────────────────────────────────────

Signed URL returning 403?
│
├── Check: Is the key-name in the URL matching a registered key?
│   └── gcloud compute backend-services describe ... → cdnPolicy.signedUrlKeyNames
│
├── Check: Has the URL expired?
│   └── Decode Expires= timestamp and compare to current UTC time
│
├── Check: Is the URL being modified (extra params added after signing)?
│   └── Any change to the URL after signing invalidates the signature
│
└── Check: Is the key file correct (same bytes registered on backend)?
    └── Re-sign with exact key bytes from cdn-signing-key.txt

─────────────────────────────────────────────────────────────

Serving stale content after deploy?
│
├── Is clientTtl high? → Browser is caching stale version
│   └── Cannot invalidate browser caches. Fix: Use versioned URLs going forward.
│
├── Invalidation issued but still stale at edge?
│   └── Check propagation: typically <1 min. Check gcloud operations list.
│
└── Using FORCE_CACHE_ALL with high maxTtl?
    └── Fix: Issue wildcard invalidation for the affected path, then reduce TTL or version URLs.
```

---

## 10. Key Limits & Quotas

| Resource | Default Limit | Notes |
|---|---|---|
| **Max cacheable object size** | 10 GB | Per individual response body |
| **Max cache entry size** | 10 GB | Same as object size |
| **Min cacheable object size** | 0 bytes | Empty responses can be cached |
| **Max signed URL key value size** | 32 bytes (256 bits) | Must be exactly 32 bytes |
| **Max signed URL keys per backend** | 3 | Supports up to 3 simultaneous keys (for rotation) |
| **Free cache invalidations per day** | 1,000 | Per project across all URL maps |
| **Invalidation propagation time** | <5 minutes | Typically <1 minute |
| **Max cache key header count** | 5 headers | Per `includeHttpHeaders` list |
| **Max cache key cookie count** | 5 cookies | Per `includeNamedCookies` list |
| **Supported query string params in whitelist** | No documented hard limit | Practical limit: keep small |
| **Cache TTL range (defaultTtl)** | 0 – 31,622,400s (1 year) | |
| **Cache TTL range (maxTtl)** | 0 – 31,622,400s (1 year) | Must be ≥ defaultTtl |
| **Client TTL range** | 0 – 31,622,400s | Must be ≤ maxTtl |
| **Origin Shield regions** | 10 supported regions | One per backend service |
| **Bypass cache on request** | Not supported natively | Use `Cache-Control: no-cache` to revalidate |
| **Signed cookie prefix length** | URL prefix, any length | Must be a valid URL prefix |
| **Negative caching TTL max** | 1800s (30 minutes) | Per status code |

---

## 11. Gotchas & Best Practices

### Common Pitfalls

- **❌ `Set-Cookie` in response prevents caching** — If your origin sets *any* cookie on a response, Cloud CDN will not cache it, even for static assets. Serve assets from a separate cookie-free domain (e.g., `static.example.com`) or strip cookies using a custom response header in Nginx/your app server.

- **❌ `Authorization` request header bypasses CDN cache** — Any request carrying an `Authorization` header is passed directly to the origin, cache is not consulted. Use Signed URLs or Signed Cookies for authenticated CDN access instead.

- **❌ `Vary` header fragments cache badly** — A `Vary: User-Agent` from the origin creates a unique cache entry per browser/device, effectively destroying your hit ratio. Avoid `Vary` on CDN-served responses, or whitelist only predictable values (`Vary: Accept-Encoding` is generally safe).

- **❌ Not adding health check firewall rules** — If your backend VMs don't allow traffic from GCP health check probers (`35.191.0.0/16`, `130.211.0.0/22`), backends go unhealthy and CDN has no origin to pull from on a miss — resulting in 502s.

- **❌ GCS bucket not publicly readable** — For a public CDN setup with a backend bucket, `allUsers` must have `Storage Object Viewer`. Without it, every cache miss returns a 403 from GCS.

- **❌ Serving HTML through `FORCE_CACHE_ALL` without versioned assets** — If you cache HTML pages and they reference JS/CSS by fixed filename, updating those files without invalidating HTML will serve stale HTML pointing to new assets — causing mismatch errors.

- **❌ Invalidating before deploy completes** — If you invalidate CDN cache before new backend code/files are live, the CDN will immediately re-cache the *old* version. Always deploy first, then invalidate.

- **❌ Using high-cardinality headers in cache key** — Adding `User-Agent`, `X-Request-ID`, or `Cookie` (full) to cache key means every request is a cache miss. Only add headers with a small finite set of values.

- **❌ Deleting signing key while signed URLs are still valid** — Active signed URLs / cookies will return 403 immediately. Always add new key first, wait for old URLs to expire, then remove old key.

---

### Best Practices

- **✅ Use versioned asset filenames** (`bundle.a1b2c3.js`) generated at build time (e.g., Webpack content hash). This eliminates the need for invalidation entirely for static assets.

- **✅ Set `FORCE_CACHE_ALL` for JSON APIs you control** — combined with a short `defaultTtl` (e.g., 60s) this lets you serve API responses from cache without modifying origin headers.

- **✅ Use a dedicated static domain** (`static.example.com`) without cookies — maximizes cacheability of static assets by eliminating the `Set-Cookie` bypass risk from your main domain.

- **✅ Enable Origin Shield** when your origin is in a single region and traffic is global. A single shield in your origin's region drastically reduces origin QPS on cache misses.

- **✅ Set `clientTtl` shorter than edge TTL** — e.g., `defaultTtl=3600`, `clientTtl=300`. This lets you invalidate at the edge and have users pick up fresh content within 5 minutes, rather than being stuck with a 1-hour browser-cached version.

- **✅ Cache warm after deploy** — After a deploy and invalidation, pre-warm the cache by fetching key URLs with a script or a synthetic monitoring tool. This prevents the origin from being hammered by real users during the cold-cache window.

- **✅ Monitor byte hit ratio, not just request hit ratio** — A high request hit ratio with low byte hit ratio means small files are cached but large files (videos, downloads) are not. Byte hit ratio is what determines actual origin offload and cost savings.

- **✅ Use Secret Manager for signing keys** — Never hardcode signing key bytes in source code or Terraform state files. Store in Secret Manager and reference at runtime.

- **✅ Prefer `queryStringWhitelist` over `queryStringBlacklist`** — Whitelisting is safer: new query params added by tracking scripts are automatically excluded. Blacklisting requires you to keep the list up to date.

- **✅ Test cache behavior with `curl -v`** — Check `Age:`, `X-Cache-Lookup:`, and `Cache-Control:` headers in responses to verify what the CDN is actually doing:
  ```bash
  curl -sI https://cdn.example.com/images/logo.png \
    | grep -E "(Age|Cache-Control|X-Cache)"
  ```

- **✅ Set `negativeCaching: true` with short TTLs** — Cache 404s for 30–60 seconds to protect the origin from error storms (e.g., crawlers hitting bad URLs), but keep TTLs short so legitimate new resources become available quickly.

- **✅ Log at 100% sample rate during initial CDN setup**, then reduce to 10% (`--logging-sample-rate=0.1`) once stable to reduce Cloud Logging costs.

---

*Last updated: March 2026 | Based on GCP Cloud CDN documentation and production patterns*
