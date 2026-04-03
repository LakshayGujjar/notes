# GCP Cloud Armor — Comprehensive Cheatsheet

> **TL;DR — 5 Most Important Cloud Armor Decisions**
> - Always **run new rules in preview mode first** — enforce only after validating in logs that no legitimate traffic is blocked.
> - **Never delete the default rule** (priority `2147483647`) — it is the catch-all; without it, unmatched traffic is undefined.
> - Cloud Armor **only protects external Cloud Load Balancers** — Internal LBs, direct VM IPs, and Cloud Run without LB are unprotected.
> - Use **sensitivity levels + exclusions** to tune WAF false positives — never disable an entire rule set just because one rule fires incorrectly.
> - **Edge security policies** are evaluated before backend policies and protect CDN cache hits too — use them for GCS buckets and geo-blocking at the lowest cost.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Core Concepts](#2-core-concepts)
3. [Security Policy Types — Deep Dive](#3-security-policy-types--deep-dive)
4. [Rule Configuration — Deep Dive](#4-rule-configuration--deep-dive)
5. [gcloud CLI Cheatsheet](#5-gcloud-cli-cheatsheet)
6. [Terraform Examples](#6-terraform-examples)
7. [CEL Expression Reference](#7-cel-expression-reference)
8. [Adaptive Protection & DDoS](#8-adaptive-protection--ddos)
9. [Rate Limiting — Deep Dive](#9-rate-limiting--deep-dive)
10. [Observability & Troubleshooting](#10-observability--troubleshooting)
11. [Key Limits & Quotas](#11-key-limits--quotas)
12. [Gotchas & Best Practices](#12-gotchas--best-practices)

---

## 1. Overview

**GCP Cloud Armor** is Google Cloud's managed application security service providing WAF (Web Application Firewall), DDoS protection, bot management, and adaptive threat detection. It integrates exclusively with **Cloud Load Balancing** and sits inline in the request path, evaluating every request before it reaches your backends.

```
Internet Traffic
      │
      ▼
┌─────────────────────────────────────────────────────┐
│           Cloud Armor Security Policy               │  ← WAF, DDoS, Rate Limiting,
│   Rule 100: Block CN, RU (geo-block)                │    Bot Management evaluated HERE
│   Rule 200: Allow office IPs                        │    before any origin contact
│   Rule 300: Pre-configured SQLi WAF rule            │
│   Rule 2147483647: Default deny                     │
└──────────────────────┬──────────────────────────────┘
                       │  Allowed traffic only
                       ▼
         Cloud Load Balancer (External only)
                       │
                       ▼
              Backend Service / Bucket
                       │
                       ▼
            Origin (MIG, GKE, Cloud Run, GCS)
```

**Key Benefits:**
- **Always-on L3/L4 DDoS protection** — built into Google's network infrastructure, no config needed
- **L7 WAF** — OWASP ModSecurity CRS pre-configured rules with tunable sensitivity
- **Adaptive Protection** — ML-based anomaly detection and auto-deploy for L7 DDoS
- **Global enforcement** — policies enforced at Google's edge PoPs, not at origin
- **Zero-latency deployment** — rule changes propagate globally in seconds
- **Preview mode** — test rules without blocking traffic

---

### Cloud Armor Tiers

| Feature | Standard | Managed Protection Plus |
|---|---|---|
| **Price** | Pay-per-policy + per-rule + per-request | ~$3,000/month per LB resource |
| **L3/L4 DDoS protection** | ✅ Always-on | ✅ Always-on |
| **L7 WAF rules** | ✅ | ✅ |
| **IP/Geo-based rules** | ✅ | ✅ |
| **Rate limiting** | ✅ | ✅ |
| **Bot management** | ✅ | ✅ |
| **Adaptive Protection (alerts)** | ✅ | ✅ |
| **Adaptive Protection (auto-deploy)** | ❌ | ✅ |
| **Named IP lists (Threat Intel)** | ✅ | ✅ |
| **DDoS response support (Google SRE)** | ❌ | ✅ |
| **Advanced network DDoS protection** | ❌ | ✅ |
| **Pre-configured WAF rule updates** | Manual | Automatic |

---

## 2. Core Concepts

### Security Policies

A **security policy** is a named set of ordered rules that Cloud Armor evaluates against each HTTP(S) request hitting an attached backend service or backend bucket.

- One policy can be attached to **multiple backend services**
- One backend service can have **only one** security policy attached at a time
- Policies are **global** resources (not regional)
- Policy changes take effect within seconds and apply globally

```
Security Policy: "prod-waf-policy"
├── Rule 100  →  allow  │  ip in ['203.0.113.0/24']
├── Rule 200  →  deny   │  origin.region_code in ['CN', 'RU', 'KP']
├── Rule 300  →  deny   │  evaluatePreconfiguredExpr('sqli-v33-stable')
├── Rule 400  →  throttle│  rate limit 100 req/min per IP
└── Rule 2147483647  →  allow  │  * (default: allow all unmatched)
```

---

### Rule Structure

Each rule has four components:

| Component | Description | Example |
|---|---|---|
| **Priority** | Integer 0–2147483646; lower = evaluated first | `100` |
| **Description** | Optional human-readable label | `"Block SQLi attacks"` |
| **Match condition** | What to match (IP, CEL expr, pre-config WAF) | `evaluatePreconfiguredExpr('sqli-v33-stable')` |
| **Action** | What to do when rule matches | `deny(403)`, `allow`, `redirect`, `throttle` |

**Actions available:**

| Action | Description | Use Case |
|---|---|---|
| `allow` | Pass request to backend | Allowlisting trusted IPs |
| `deny(STATUS)` | Return HTTP error (403, 404, 429, 502) | Blocking attackers |
| `redirect` | Redirect to reCAPTCHA or external URL | Bot challenge |
| `throttle` | Rate-limit requests; allow conforming traffic | API protection |
| `rate_based_ban` | Rate-limit then temporarily ban the source | Abuse prevention |

---

### Rule Evaluation Order

```
Request arrives
      │
      ▼
Rule 100 matches? → YES → Apply action (allow/deny/redirect/rate-limit) → DONE
      │ NO
      ▼
Rule 200 matches? → YES → Apply action → DONE
      │ NO
      ▼
Rule 300 matches? → YES → Apply action → DONE
      │ NO
      ▼
...
      │ NO match on any rule
      ▼
Rule 2147483647 (DEFAULT RULE) → Always matches (*) → Apply default action
```

**Key evaluation rules:**
- Rules are evaluated **strictly in priority order** — lowest number first
- First matching rule wins — no further rules evaluated after a match
- The **default rule** (priority `2147483647`) must always exist and always matches everything (`*`)
- Typical default action: `allow` (allowlist model) or `deny(403)` (denylist model with explicit allows)

---

### Match Conditions

Three types of match conditions:

**1. IP/CIDR match:**
```
srcIpRanges: ["203.0.113.0/24", "198.51.100.5/32"]
```

**2. CEL expression match:**
```cel
origin.region_code == 'CN' && request.path.matches('/admin/.*')
```

**3. Pre-configured WAF expression:**
```cel
evaluatePreconfiguredExpr('sqli-v33-stable')
evaluatePreconfiguredExpr('xss-v33-stable', {'sensitivity': 1})
```

---

### CEL (Common Expression Language)

CEL is the expression language used for Cloud Armor rule conditions. It is evaluated per-request with access to request attributes.

**Key attributes:**

| Attribute | Type | Description | Example |
|---|---|---|---|
| `request.headers` | map | HTTP request headers (lowercase keys) | `request.headers['user-agent']` |
| `request.path` | string | URI path | `/api/v1/users` |
| `request.query` | string | Raw query string | `page=1&sort=asc` |
| `request.method` | string | HTTP method | `POST` |
| `request.scheme` | string | `http` or `https` | `https` |
| `origin.ip` | string | Client IP address | `203.0.113.5` |
| `origin.region_code` | string | ISO 3166-1 alpha-2 country code | `CN`, `US`, `RU` |
| `origin.asn` | int | Client AS number | `15169` |

> Full CEL cookbook in [Section 7](#7-cel-expression-reference).

---

### Pre-Configured WAF Rules

Cloud Armor provides managed WAF rule sets based on the **OWASP ModSecurity Core Rule Set (CRS) v3.3**. Each rule set has multiple sensitivity levels:

| Sensitivity | Behavior |
|---|---|
| `0` | All rules in the set disabled (used for targeted exclusions) |
| `1` | Low sensitivity — fewest false positives, catches obvious attacks |
| `2` | Medium sensitivity (default for most rules) |
| `3` | High sensitivity — more detections, more potential false positives |
| `4` | Highest sensitivity — maximum detection, highest false positive risk |

```cel
// Use default sensitivity (determined per rule set)
evaluatePreconfiguredExpr('sqli-v33-stable')

// Override to sensitivity 1 (low)
evaluatePreconfiguredExpr('sqli-v33-stable', {'sensitivity': 1})
```

---

### Adaptive Protection

**Adaptive Protection** uses ML to model normal traffic baselines for each backend service and detect anomalous L7 traffic patterns indicative of DDoS or scraping attacks.

- Requires 1+ hour of baseline traffic to build a model
- Generates **alerts** with suggested rule signatures when anomalies are detected
- In **Managed Protection Plus**: can **auto-deploy** suggested rules automatically
- Alert confidence score: 0.0–1.0 (higher = more certain it's an attack)

---

### Rate Limiting

Rate limiting controls request frequency from a source, independent of WAF rule matches.

**Two actions:**
- **`throttle`**: Allow requests up to the limit; respond with 429 to excess requests
- **`rate_based_ban`**: Allow up to the limit; ban the source entirely for a duration when exceeded

**Rate limit keys** determine what is counted per-limit:

| Key | Description |
|---|---|
| `ALL` | Single global counter (all traffic combined) |
| `IP` | Per source IP |
| `XFF-IP` | Per X-Forwarded-For IP (trusted proxy scenario) |
| `HTTP-HEADER` | Per unique value of a specific header |
| `HTTP-COOKIE` | Per unique value of a specific cookie |
| `HTTP-PATH` | Per unique URI path |
| `REGION-CODE` | Per source country |
| `USER-IP` | Per IP from XFF or direct (Cloud Armor selects) |

---

### Bot Management

Cloud Armor integrates with **reCAPTCHA Enterprise** for bot detection:

- **Token validation**: Attach reCAPTCHA Enterprise site key; Cloud Armor validates the token in each request
- **Bot score**: Requests scored 0.0 (bot) to 1.0 (human); set threshold in rules
- **Actions**: Allow high-score (human) traffic; redirect low-score (bot) traffic to reCAPTCHA challenge

---

### Threat Intelligence (Named IP Lists)

Google maintains curated IP lists updated automatically:

| Named List | Description |
|---|---|
| `sourceiplist-tor-exit-nodes` | Tor network exit nodes |
| `sourceiplist-known-malicious-ips` | Google-curated malicious IPs |
| `sourceiplist-search-crawlers` | Legitimate search engine crawlers |
| `sourceiplist-fastly` | Fastly CDN egress IPs |
| `sourceiplist-cloudflare` | Cloudflare CDN egress IPs |

```cel
// Block Tor exit nodes
evaluateThreatIntelligence('sourceiplist-tor-exit-nodes')

// Allow only Cloudflare IPs (if behind Cloudflare)
!evaluateThreatIntelligence('sourceiplist-cloudflare')
```

---

### Edge vs. Backend Security Policies

| Feature | Backend Security Policy | Edge Security Policy |
|---|---|---|
| **Attaches to** | Backend service | Backend service OR backend bucket |
| **Evaluated at** | Load balancer (after edge) | CDN edge (before cache lookup) |
| **Protects CDN cache hits** | ❌ No | ✅ Yes |
| **Supported actions** | allow, deny, redirect, throttle, rate_based_ban | allow, deny |
| **WAF rules** | ✅ Yes | ❌ No |
| **Rate limiting** | ✅ Yes | ❌ No |
| **Bot management** | ✅ Yes | ❌ No |
| **Geo-blocking** | ✅ Yes | ✅ Yes |
| **IP blocking** | ✅ Yes | ✅ Yes |
| **GCS bucket support** | ❌ No | ✅ Yes |

**Evaluation order when both exist:**
```
Request → Edge Policy → (if allowed) → Backend Policy → (if allowed) → Origin
```

---

### Preview Mode

**Preview mode** evaluates a rule and **logs** what would have happened — without taking any action on the traffic.

- Set per-rule with `--preview` flag
- Preview hits appear in Cloud Logging with `previewSecurityPolicy` field
- Use to validate rules before enforcement
- A policy can have a mix of preview and enforced rules

---

## 3. Security Policy Types — Deep Dive

### Backend Security Policy

- **Definition**: Standard WAF and access control policy for HTTP(S) traffic
- **Attaches to**: External Application Load Balancer backend services
- **Supported actions**: `allow`, `deny`, `redirect`, `throttle`, `rate_based_ban`
- **Layer**: L7 (application layer — sees full HTTP request)
- **Key limitations**:
  - Cannot attach to backend buckets (GCS) — use edge policy
  - Cannot protect Internal LBs
  - Does not protect CDN cache hits (edge policy needed for that)
- **When to choose**: Default choice for all web app and API protection; required for WAF, rate limiting, bot management

---

### Edge Security Policy

- **Definition**: Lightweight IP/geo policy enforced at CDN edge PoPs, before cache lookup
- **Attaches to**: Backend services and **backend buckets** (GCS)
- **Supported actions**: `allow`, `deny` only
- **Layer**: L3/L4 + IP/Geo (no deep HTTP inspection)
- **Key limitations**:
  - No WAF, rate limiting, or bot management
  - Only IP CIDR and CEL-based IP/geo rules
  - Cannot inspect HTTP headers, path, or body
- **When to choose**: Geo-blocking at lowest cost; protecting GCS buckets; blocking CDN cache hit requests from banned IPs/regions

---

### Network Edge Security Policy

- **Definition**: L3/L4 DDoS and IP protection for **passthrough load balancers** (External Passthrough Network LB, Protocol Forwarding, VMs with public IPs)
- **Attaches to**: Security policy of type `CLOUD_ARMOR_NETWORK` assigned to LB
- **Supported actions**: `allow`, `deny`
- **Layer**: L3/L4
- **Key limitations**:
  - Cannot inspect HTTP content
  - Only available with **Managed Protection Plus**
  - Not applicable to proxy-based (L7) LBs
- **When to choose**: Protecting UDP workloads, gaming servers, VoIP — anything that uses passthrough LBs where L7 inspection isn't possible

---

## 4. Rule Configuration — Deep Dive

### IP/CIDR Allowlist and Denylist Rules

```bash
# Allowlist — allow specific office CIDRs, deny everything else
# Rule 100: Allow office IPs
gcloud compute security-policies rules create 100 \
  --security-policy=my-policy \
  --src-ip-ranges="203.0.113.0/24,198.51.100.0/28" \
  --action=allow \
  --description="Allow corporate office IPs"

# Rule 2147483647: Default deny
gcloud compute security-policies rules update 2147483647 \
  --security-policy=my-policy \
  --action=deny-403 \
  --description="Default deny all"
```

```bash
# Denylist — block specific known-bad IPs
gcloud compute security-policies rules create 50 \
  --security-policy=my-policy \
  --src-ip-ranges="192.0.2.100/32,192.0.2.200/32" \
  --action=deny-403 \
  --description="Block known attacker IPs"
```

---

### Geo-Blocking Rules

```bash
# Block specific countries using origin.region_code
gcloud compute security-policies rules create 200 \
  --security-policy=my-policy \
  --expression="origin.region_code == 'CN' || origin.region_code == 'RU' || origin.region_code == 'KP'" \
  --action=deny-403 \
  --description="Geo-block CN, RU, KP"

# Allow ONLY US traffic (deny all other countries)
gcloud compute security-policies rules create 200 \
  --security-policy=my-policy \
  --expression="origin.region_code != 'US'" \
  --action=deny-403 \
  --description="Allow US only"

# Geo-block combined with path restriction
gcloud compute security-policies rules create 210 \
  --security-policy=my-policy \
  --expression="origin.region_code in ['CN', 'RU'] && request.path.startsWith('/checkout')" \
  --action=deny-403 \
  --description="Block CN/RU from checkout"
```

---

### HTTP Header Inspection Rules

```bash
# Block requests with no User-Agent (automated scanners)
gcloud compute security-policies rules create 300 \
  --security-policy=my-policy \
  --expression="!has(request.headers, 'user-agent')" \
  --action=deny-403 \
  --description="Block requests without User-Agent"

# Block specific bad User-Agents (scanners, scrapers)
gcloud compute security-policies rules create 310 \
  --security-policy=my-policy \
  --expression="request.headers['user-agent'].lower().contains('sqlmap') || request.headers['user-agent'].lower().contains('nikto') || request.headers['user-agent'].lower().contains('masscan')" \
  --action=deny-403 \
  --description="Block known scanner User-Agents"

# Require a specific custom header (internal API key header)
gcloud compute security-policies rules create 150 \
  --security-policy=my-policy \
  --expression="!has(request.headers, 'x-api-gateway-key')" \
  --action=deny-403 \
  --description="Require x-api-gateway-key header"

# Block Referer-based hotlinking
gcloud compute security-policies rules create 320 \
  --security-policy=my-policy \
  --expression="has(request.headers, 'referer') && !request.headers['referer'].contains('example.com')" \
  --action=deny-403 \
  --description="Block hotlinking from other domains"
```

---

### URI Path and Query String Matching

```bash
# Block access to admin paths from non-office IPs
gcloud compute security-policies rules create 250 \
  --security-policy=my-policy \
  --expression="request.path.startsWith('/admin') && !inIpRange(origin.ip, '203.0.113.0/24')" \
  --action=deny-403 \
  --description="Restrict /admin to office IPs"

# Block specific file extensions (shell, config, backup files)
gcloud compute security-policies rules create 260 \
  --security-policy=my-policy \
  --expression="request.path.matches('.*\\.(php|sh|env|bak|sql|config)$')" \
  --action=deny-403 \
  --description="Block sensitive file extension access"

# Block suspicious query parameters
gcloud compute security-policies rules create 270 \
  --security-policy=my-policy \
  --expression="request.query.contains('UNION SELECT') || request.query.contains('<script>') || request.query.contains('../..')" \
  --action=deny-403 \
  --description="Block obvious injection in query string"

# Block PUT/DELETE methods on public endpoints
gcloud compute security-policies rules create 280 \
  --security-policy=my-policy \
  --expression="request.method == 'DELETE' || request.method == 'PUT'" \
  --action=deny-405 \
  --description="Block destructive HTTP methods"
```

---

### Pre-Configured WAF Rule Sets

| Rule Set Name | Category | Default Sensitivity | Rule IDs |
|---|---|---|---|
| `sqli-v33-stable` | SQL Injection | 2 | 942100–942999 |
| `xss-v33-stable` | Cross-Site Scripting | 2 | 941100–941999 |
| `lfi-v33-stable` | Local File Inclusion | 2 | 930100–930999 |
| `rfi-v33-stable` | Remote File Inclusion | 2 | 931100–931999 |
| `rce-v33-stable` | Remote Code Execution | 2 | 932100–932999 |
| `scannerdetection-v33-stable` | Scanner Detection | 2 | 913100–913999 |
| `protocolattack-v33-stable` | Protocol Attack | 2 | 921100–921999 |
| `php-v33-stable` | PHP Injection | 2 | 933100–933999 |
| `sessionfixation-v33-stable` | Session Fixation | 2 | 943100–943999 |
| `java-v33-stable` | Java Attack | 2 | 944100–944999 |
| `nodejs-v33-stable` | Node.js Attack | 2 | 934100–934999 |
| `methodenforcement-v33-stable` | Method Enforcement | 2 | 911100–911999 |
| `cve-canary` | CVE-specific rules | N/A | Various |

```bash
# Apply SQLi WAF rule with default sensitivity
gcloud compute security-policies rules create 1000 \
  --security-policy=my-policy \
  --expression="evaluatePreconfiguredExpr('sqli-v33-stable')" \
  --action=deny-403 \
  --description="Block SQL injection attacks"

# Apply XSS rule with sensitivity 1 (fewer false positives)
gcloud compute security-policies rules create 1010 \
  --security-policy=my-policy \
  --expression="evaluatePreconfiguredExpr('xss-v33-stable', {'sensitivity': 1})" \
  --action=deny-403 \
  --description="Block XSS attacks (low sensitivity)"

# Apply multiple WAF rules combined
gcloud compute security-policies rules create 1020 \
  --security-policy=my-policy \
  --expression="evaluatePreconfiguredExpr('lfi-v33-stable') || evaluatePreconfiguredExpr('rfi-v33-stable') || evaluatePreconfiguredExpr('rce-v33-stable')" \
  --action=deny-403 \
  --description="Block LFI, RFI, RCE attacks"
```

---

### WAF Rule Exclusions

Exclude specific request fields from WAF evaluation to reduce false positives:

```bash
# Exclude a specific request header from WAF evaluation
# (e.g., a JWT token that triggers SQLi rule)
gcloud compute security-policies rules create 1000 \
  --security-policy=my-policy \
  --expression="evaluatePreconfiguredExpr('sqli-v33-stable', {'opt_out_rule_ids': ['owasp-crs-id942440', 'owasp-crs-id942450']})" \
  --action=deny-403 \
  --description="SQLi with specific rule IDs excluded"
```

**Exclusion via JSON config (for full field exclusions):**

```bash
# Export policy, edit exclusions, reimport
gcloud compute security-policies export my-policy \
  --file-name=policy.json \
  --file-format=json
```

```json
{
  "rules": [{
    "priority": 1000,
    "match": {
      "expr": {
        "expression": "evaluatePreconfiguredExpr('sqli-v33-stable')"
      }
    },
    "action": "deny(403)",
    "preconfiguredWafConfig": {
      "exclusions": [{
        "targetRuleSet": "sqli-v33-stable",
        "requestHeadersToExclude": [
          {"val": "Authorization", "op": "EQUALS"},
          {"val": "X-Custom-Token", "op": "EQUALS"}
        ],
        "requestQueryParamsToExclude": [
          {"val": "token", "op": "EQUALS"}
        ]
      }]
    }
  }]
}
```

---

### Redirect Actions

```bash
# Redirect to reCAPTCHA Enterprise challenge
gcloud compute security-policies rules create 500 \
  --security-policy=my-policy \
  --expression="origin.region_code in ['CN', 'RU']" \
  --action=redirect \
  --redirect-type=GOOGLE_RECAPTCHA \
  --description="Challenge suspicious geo traffic with reCAPTCHA"

# Redirect to an external URL (e.g., custom block page)
gcloud compute security-policies rules create 510 \
  --security-policy=my-policy \
  --src-ip-ranges="192.0.2.0/24" \
  --action=redirect \
  --redirect-type=EXTERNAL_302 \
  --redirect-target="https://example.com/blocked" \
  --description="Redirect blocked IPs to custom page"
```

---

### Named IP Lists (Threat Intelligence)

```bash
# Block Tor exit nodes
gcloud compute security-policies rules create 50 \
  --security-policy=my-policy \
  --expression="evaluateThreatIntelligence('sourceiplist-tor-exit-nodes')" \
  --action=deny-403 \
  --description="Block Tor exit nodes"

# Block known malicious IPs
gcloud compute security-policies rules create 55 \
  --security-policy=my-policy \
  --expression="evaluateThreatIntelligence('sourceiplist-known-malicious-ips')" \
  --action=deny-403 \
  --description="Block known malicious IPs"

# Allow only search engine crawlers to /sitemap.xml
gcloud compute security-policies rules create 60 \
  --security-policy=my-policy \
  --expression="request.path == '/sitemap.xml' && !evaluateThreatIntelligence('sourceiplist-search-crawlers')" \
  --action=deny-403 \
  --description="Restrict sitemap to verified crawlers only"
```

---

## 5. gcloud CLI Cheatsheet

### Policy Management

```bash
# ── CREATE POLICIES ───────────────────────────────────────

# Create a basic backend security policy
gcloud compute security-policies create my-waf-policy \
  --description="Production WAF policy" \
  --type=CLOUD_ARMOR                    # Default; omit for backend policy

# Create an edge security policy
gcloud compute security-policies create my-edge-policy \
  --description="Edge geo-block policy" \
  --type=CLOUD_ARMOR_EDGE               # Edge policy type

# Create a network edge security policy (Managed Protection Plus only)
gcloud compute security-policies create my-network-policy \
  --description="Network DDoS policy" \
  --type=CLOUD_ARMOR_NETWORK

# ── LIST & DESCRIBE ───────────────────────────────────────

# List all security policies
gcloud compute security-policies list

# Describe a policy (shows all rules)
gcloud compute security-policies describe my-waf-policy

# List only rules in a policy
gcloud compute security-policies rules list \
  --security-policy=my-waf-policy

# Describe a specific rule
gcloud compute security-policies rules describe 1000 \
  --security-policy=my-waf-policy

# ── DELETE ────────────────────────────────────────────────

# Delete a security policy (must detach from all backends first)
gcloud compute security-policies delete my-waf-policy
```

---

### Adding and Managing Rules

```bash
# ── ADD RULES ─────────────────────────────────────────────

# Add allow rule for IP range
gcloud compute security-policies rules create 100 \
  --security-policy=my-waf-policy \
  --src-ip-ranges="203.0.113.0/24" \    # CIDR range(s) comma-separated
  --action=allow \
  --description="Allow corporate office"

# Add deny rule with CEL expression
gcloud compute security-policies rules create 200 \
  --security-policy=my-waf-policy \
  --expression="origin.region_code in ['CN', 'RU', 'KP', 'IR']" \
  --action=deny-403 \                   # deny-403, deny-404, deny-429, deny-502
  --description="Block high-risk geos"

# Add pre-configured WAF rule
gcloud compute security-policies rules create 1000 \
  --security-policy=my-waf-policy \
  --expression="evaluatePreconfiguredExpr('sqli-v33-stable')" \
  --action=deny-403 \
  --description="SQLi WAF protection"

# Add rule in PREVIEW mode (logs only, no enforcement)
gcloud compute security-policies rules create 1010 \
  --security-policy=my-waf-policy \
  --expression="evaluatePreconfiguredExpr('xss-v33-stable')" \
  --action=deny-403 \
  --preview \                           # Enable preview mode
  --description="XSS WAF (preview only)"

# ── UPDATE RULES ──────────────────────────────────────────

# Update rule action
gcloud compute security-policies rules update 1010 \
  --security-policy=my-waf-policy \
  --action=deny-403                     # Remove --preview to enforce

# Promote preview rule to enforcement (remove --preview)
gcloud compute security-policies rules update 1010 \
  --security-policy=my-waf-policy \
  --no-preview                          # Disable preview mode = enforce

# Update rule description
gcloud compute security-policies rules update 200 \
  --security-policy=my-waf-policy \
  --description="Block CN, RU, KP, IR - updated geo list"

# ── DELETE RULES ──────────────────────────────────────────

# Delete a specific rule (never delete 2147483647!)
gcloud compute security-policies rules delete 1010 \
  --security-policy=my-waf-policy

# ── UPDATE DEFAULT RULE ───────────────────────────────────

# Change default rule to deny-all (zero-trust model)
gcloud compute security-policies rules update 2147483647 \
  --security-policy=my-waf-policy \
  --action=deny-403 \
  --description="Default deny all"
```

---

### Attaching / Detaching from Backend Services

```bash
# Attach backend security policy to a backend service
gcloud compute backend-services update my-backend-svc \
  --global \
  --security-policy=my-waf-policy       # Attach policy by name

# Attach edge security policy to a backend service
gcloud compute backend-services update my-backend-svc \
  --global \
  --edge-security-policy=my-edge-policy

# Attach edge security policy to a BACKEND BUCKET (GCS)
gcloud compute backend-buckets update my-cdn-bucket \
  --edge-security-policy=my-edge-policy

# Detach security policy from a backend service
gcloud compute backend-services update my-backend-svc \
  --global \
  --security-policy=""                  # Empty string detaches

# Verify policy attachment
gcloud compute backend-services describe my-backend-svc \
  --global \
  --format="value(securityPolicy,edgeSecurityPolicy)"
```

---

### Rate Limiting Rules

```bash
# Throttle rule: 100 requests/min per IP; return 429 for excess
gcloud compute security-policies rules create 400 \
  --security-policy=my-waf-policy \
  --expression="true" \                 # Match all traffic (or use specific expr)
  --action=throttle \
  --rate-limit-threshold-count=100 \    # Max requests
  --rate-limit-threshold-interval-sec=60 \  # Per interval (seconds)
  --conform-action=allow \              # Action for requests within limit
  --exceed-action=deny-429 \            # Action for excess requests
  --enforce-on-key=IP \                 # Count per source IP
  --description="Global per-IP rate limit"

# Rate-based ban: ban IP for 300s after exceeding 50 req/30s
gcloud compute security-policies rules create 410 \
  --security-policy=my-waf-policy \
  --expression="request.path.startsWith('/login')" \
  --action=rate_based_ban \
  --rate-limit-threshold-count=50 \
  --rate-limit-threshold-interval-sec=30 \
  --ban-duration-sec=300 \              # Ban for 5 minutes
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP \
  --description="Ban IPs brute-forcing /login"

# Rate limit by HTTP header value
gcloud compute security-policies rules create 420 \
  --security-policy=my-waf-policy \
  --expression="true" \
  --action=throttle \
  --rate-limit-threshold-count=1000 \
  --rate-limit-threshold-interval-sec=60 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=HTTP-HEADER \
  --enforce-on-key-name="X-API-Key" \   # Count per API key value
  --description="Per-API-key rate limit"
```

---

### Adaptive Protection

```bash
# Enable Adaptive Protection on a policy
gcloud compute security-policies update my-waf-policy \
  --enable-layer7-ddos-defense          # Enables Adaptive Protection

# Enable Adaptive Protection with auto-deploy (Managed Protection Plus only)
gcloud compute security-policies update my-waf-policy \
  --enable-layer7-ddos-defense \
  --layer7-ddos-defense-rule-visibility=STANDARD  # STANDARD or PREMIUM

# Configure auto-deploy thresholds
gcloud compute security-policies update my-waf-policy \
  --enable-layer7-ddos-defense \
  --auto-deploy-load-threshold=0.7 \        # Deploy when load > 70% above baseline
  --auto-deploy-confidence-threshold=0.8 \  # Deploy when confidence > 80%
  --auto-deploy-expiration-sec=7200         # Auto-expire deployed rules after 2 hours

# Check Adaptive Protection alerts
gcloud compute security-policies get-adaptive-protection-log my-waf-policy
```

---

### Export / Import

```bash
# Export policy to JSON
gcloud compute security-policies export my-waf-policy \
  --file-name=my-waf-policy.json \
  --file-format=json

# Export policy to YAML
gcloud compute security-policies export my-waf-policy \
  --file-name=my-waf-policy.yaml \
  --file-format=yaml

# Import/update policy from file
gcloud compute security-policies import my-waf-policy \
  --file-name=my-waf-policy.json \
  --file-format=json
```

---

## 6. Terraform Examples

### Baseline Policy: IP Allowlist + Geo-Block + Default Deny

```hcl
resource "google_compute_security_policy" "baseline" {
  name        = "baseline-security-policy"
  description = "IP allowlist, geo-block, default deny"
  type        = "CLOUD_ARMOR"

  # ── Rule 100: Allow corporate office IPs ─────────────────
  rule {
    priority    = 100
    description = "Allow corporate office CIDRs"
    action      = "allow"
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["203.0.113.0/24", "198.51.100.0/28"]
      }
    }
  }

  # ── Rule 200: Block high-risk countries ──────────────────
  rule {
    priority    = 200
    description = "Geo-block CN, RU, KP, IR"
    action      = "deny(403)"
    match {
      expr {
        expression = "origin.region_code in ['CN', 'RU', 'KP', 'IR']"
      }
    }
  }

  # ── Rule 300: Block Tor exit nodes ────────────────────────
  rule {
    priority    = 300
    description = "Block Tor exit nodes"
    action      = "deny(403)"
    match {
      expr {
        expression = "evaluateThreatIntelligence('sourceiplist-tor-exit-nodes')"
      }
    }
  }

  # ── Rule 2147483647: Default deny ─────────────────────────
  rule {
    priority    = 2147483647
    description = "Default deny all"
    action      = "deny(403)"
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
  }
}
```

---

### WAF Policy with OWASP Rules and Sensitivity Tuning

```hcl
resource "google_compute_security_policy" "waf" {
  name        = "owasp-waf-policy"
  description = "OWASP CRS WAF protection"
  type        = "CLOUD_ARMOR"

  # ── Rule 1000: SQLi protection (sensitivity 1) ────────────
  rule {
    priority    = 1000
    description = "Block SQL injection (low sensitivity)"
    action      = "deny(403)"
    preview     = false
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('sqli-v33-stable', {'sensitivity': 1})"
      }
    }
  }

  # ── Rule 1010: XSS protection (sensitivity 2, preview) ────
  rule {
    priority    = 1010
    description = "Block XSS attacks (preview mode)"
    action      = "deny(403)"
    preview     = true                   # Preview only — monitor before enforcing
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('xss-v33-stable')"
      }
    }
  }

  # ── Rule 1020: LFI + RFI + RCE ───────────────────────────
  rule {
    priority    = 1020
    description = "Block LFI, RFI, RCE"
    action      = "deny(403)"
    match {
      expr {
        expression = <<-EOT
          evaluatePreconfiguredExpr('lfi-v33-stable') ||
          evaluatePreconfiguredExpr('rfi-v33-stable') ||
          evaluatePreconfiguredExpr('rce-v33-stable')
        EOT
      }
    }
  }

  # ── Rule 1030: Scanner detection ─────────────────────────
  rule {
    priority    = 1030
    description = "Block scanner tools"
    action      = "deny(403)"
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('scannerdetection-v33-stable')"
      }
    }
  }

  # ── Rule 1040: PHP injection ──────────────────────────────
  rule {
    priority    = 1040
    description = "Block PHP injection"
    action      = "deny(403)"
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('php-v33-stable', {'sensitivity': 1})"
      }
    }
  }

  # ── Default: Allow ────────────────────────────────────────
  rule {
    priority    = 2147483647
    description = "Default allow"
    action      = "allow"
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
  }

  adaptive_protection_config {
    layer_7_ddos_defense_config {
      enable = true
    }
  }
}
```

---

### Rate Limiting Policy

```hcl
resource "google_compute_security_policy" "rate_limit" {
  name        = "rate-limiting-policy"
  description = "Per-IP rate limiting and brute-force protection"

  # ── Rule 400: Throttle all traffic — 1000 req/min per IP ──
  rule {
    priority    = 400
    description = "Per-IP rate limit: 1000 req/min"
    action      = "throttle"
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
    rate_limit_options {
      conform_action = "allow"
      exceed_action  = "deny(429)"
      enforce_on_key = "IP"
      rate_limit_threshold {
        count        = 1000
        interval_sec = 60
      }
    }
  }

  # ── Rule 410: Ban IPs brute-forcing /login ────────────────
  rule {
    priority    = 410
    description = "Rate-based ban on /login brute force"
    action      = "rate_based_ban"
    match {
      expr {
        expression = "request.path == '/login' || request.path == '/api/auth'"
      }
    }
    rate_limit_options {
      conform_action  = "allow"
      exceed_action   = "deny(429)"
      enforce_on_key  = "IP"
      ban_duration_sec = 600            # Ban for 10 minutes
      rate_limit_threshold {
        count        = 30
        interval_sec = 60
      }
    }
  }

  # ── Rule 420: Per-API-key throttle ────────────────────────
  rule {
    priority    = 420
    description = "Per-API-key rate limit via header"
    action      = "throttle"
    match {
      expr {
        expression = "has(request.headers, 'x-api-key')"
      }
    }
    rate_limit_options {
      conform_action     = "allow"
      exceed_action      = "deny(429)"
      enforce_on_key     = "HTTP-HEADER"
      enforce_on_key_name = "x-api-key"
      rate_limit_threshold {
        count        = 500
        interval_sec = 60
      }
    }
  }

  rule {
    priority = 2147483647
    action   = "allow"
    match {
      versioned_expr = "SRC_IPS_V1"
      config { src_ip_ranges = ["*"] }
    }
  }
}
```

---

### Bot Management Policy with reCAPTCHA Redirect

```hcl
resource "google_compute_security_policy" "bot_management" {
  name        = "bot-management-policy"
  description = "reCAPTCHA Enterprise bot management"

  # ── Rule 100: Allow verified humans (high bot score) ──────
  rule {
    priority    = 100
    description = "Allow reCAPTCHA verified humans"
    action      = "allow"
    match {
      expr {
        expression = "token.recaptcha_session.score >= 0.8"
      }
    }
  }

  # ── Rule 200: Redirect bots to reCAPTCHA challenge ────────
  rule {
    priority    = 200
    description = "Challenge low-confidence requests"
    action      = "redirect"
    redirect_options {
      type = "GOOGLE_RECAPTCHA"         # Redirect to reCAPTCHA Enterprise
    }
    match {
      expr {
        expression = "origin.region_code in ['CN', 'RU'] || !has(request.headers, 'user-agent')"
      }
    }
  }

  # ── Rule 300: Block known bots (very low score) ───────────
  rule {
    priority    = 300
    description = "Block confirmed bots"
    action      = "deny(403)"
    match {
      expr {
        expression = "token.recaptcha_session.score < 0.2"
      }
    }
  }

  rule {
    priority = 2147483647
    action   = "allow"
    match {
      versioned_expr = "SRC_IPS_V1"
      config { src_ip_ranges = ["*"] }
    }
  }
}

# ── Attach backend policy to backend service ──────────────
resource "google_compute_backend_service" "protected" {
  name            = "my-protected-backend"
  security_policy = google_compute_security_policy.bot_management.id
  # ... other backend service config
}
```

---

## 7. CEL Expression Reference

### Available Attributes

| Attribute | Type | Example Value | Notes |
|---|---|---|---|
| `request.path` | string | `/api/v1/users` | URL path only, no query string |
| `request.query` | string | `page=1&sort=name` | Raw query string |
| `request.headers['name']` | string | `Mozilla/5.0...` | Header names are lowercase |
| `request.method` | string | `GET`, `POST`, `DELETE` | Uppercase HTTP method |
| `request.scheme` | string | `https` | `http` or `https` |
| `origin.ip` | string | `203.0.113.5` | Client IP (or XFF IP if trusted) |
| `origin.region_code` | string | `US`, `CN`, `RU` | ISO 3166-1 alpha-2 |
| `origin.asn` | int | `15169` | Autonomous System Number |
| `token.recaptcha_session.score` | float | `0.9` | reCAPTCHA session token score |
| `token.recaptcha_action.score` | float | `0.7` | reCAPTCHA action token score |

---

### CEL Functions

| Function | Signature | Description | Example |
|---|---|---|---|
| `inIpRange()` | `inIpRange(ip, cidr)` | Check if IP is in CIDR range | `inIpRange(origin.ip, '10.0.0.0/8')` |
| `has()` | `has(map, key)` | Check if header/field exists | `has(request.headers, 'x-auth-token')` |
| `matches()` | `str.matches(regex)` | Regex match on string | `request.path.matches('.*/admin/.*')` |
| `contains()` | `str.contains(substr)` | Substring match | `request.headers['user-agent'].contains('bot')` |
| `startsWith()` | `str.startsWith(prefix)` | Prefix match | `request.path.startsWith('/api/')` |
| `endsWith()` | `str.endsWith(suffix)` | Suffix match | `request.path.endsWith('.php')` |
| `lower()` | `str.lower()` | Lowercase string | `request.headers['user-agent'].lower()` |
| `evaluatePreconfiguredExpr()` | `(name, opts?)` | Run WAF rule set | `evaluatePreconfiguredExpr('sqli-v33-stable')` |
| `evaluateThreatIntelligence()` | `(list_name)` | Check against threat list | `evaluateThreatIntelligence('sourceiplist-tor-exit-nodes')` |

---

### CEL Expression Cookbook

```cel
// 1. Block specific country
origin.region_code == 'CN'

// 2. Block multiple countries (list syntax)
origin.region_code in ['CN', 'RU', 'KP', 'IR', 'SY']

// 3. Allow only one country
origin.region_code != 'US'

// 4. Block IP range
inIpRange(origin.ip, '192.168.0.0/16')

// 5. Allow only specific ASN (e.g., corporate ISP)
origin.asn != 12345

// 6. Block requests without User-Agent (bots/scripts)
!has(request.headers, 'user-agent')

// 7. Block scanner User-Agents (case-insensitive)
request.headers['user-agent'].lower().contains('sqlmap') ||
request.headers['user-agent'].lower().contains('nikto') ||
request.headers['user-agent'].lower().contains('nmap') ||
request.headers['user-agent'].lower().contains('masscan')

// 8. Restrict admin paths to office IP range
request.path.startsWith('/admin') &&
!inIpRange(origin.ip, '203.0.113.0/24')

// 9. Block sensitive file extensions
request.path.matches('.*\\.(php|asp|aspx|cgi|sh|bash|env|bak|sql|git|svn)$')

// 10. Block OPTIONS method (CORS preflight abuse)
request.method == 'OPTIONS' &&
!has(request.headers, 'origin')

// 11. Block requests with suspicious query strings
request.query.lower().contains('union select') ||
request.query.lower().contains('drop table') ||
request.query.lower().contains('exec(') ||
request.query.lower().contains('<script')

// 12. Enforce HTTPS-only (block HTTP)
request.scheme == 'http'

// 13. Block missing Referer on POST (CSRF-like protection)
request.method == 'POST' &&
!has(request.headers, 'referer')

// 14. Rate-limit specific API path (use in throttle rule)
request.path.startsWith('/api/v1/auth') ||
request.path.startsWith('/api/v1/login')

// 15. Combined: geo-block AND path restriction
origin.region_code in ['CN', 'RU'] &&
(request.path.startsWith('/checkout') ||
 request.path.startsWith('/account'))

// 16. Allow Googlebot, block all other non-browser agents
!evaluateThreatIntelligence('sourceiplist-search-crawlers') &&
request.headers['user-agent'].lower().contains('bot')

// 17. Block direct-to-IP access (missing Host header for your domain)
!request.headers['host'].contains('example.com')

// 18. WAF SQLi with sensitivity override
evaluatePreconfiguredExpr('sqli-v33-stable', {'sensitivity': 1})

// 19. Combined WAF + geo exclusion (WAF for all except US)
evaluatePreconfiguredExpr('sqli-v33-stable') &&
origin.region_code != 'US'

// 20. Block known malicious IPs + Tor combined
evaluateThreatIntelligence('sourceiplist-known-malicious-ips') ||
evaluateThreatIntelligence('sourceiplist-tor-exit-nodes')
```

---

## 8. Adaptive Protection & DDoS

### How Adaptive Protection Works

```
Phase 1 — Baseline (first 1+ hour of traffic):
  Cloud Armor ML model learns normal request patterns:
  - Requests/second per backend
  - Geographic distribution
  - User-Agent distribution
  - URL path distribution

Phase 2 — Detection (ongoing):
  Model continuously compares live traffic to baseline
  Anomaly score calculated per time window (1–5 min)
  If anomaly score exceeds threshold → alert generated

Phase 3 — Alert:
  Alert contains:
  - Attack signature (CEL expression matching attack traffic)
  - Confidence score (0.0–1.0)
  - Estimated impact (% of traffic that is attack)
  - Suggested rule to block attack

Phase 4 — Response (manual or auto-deploy):
  Manual: Engineer reviews alert, creates rule from signature
  Auto-deploy: Cloud Armor automatically creates + enforces rule
              (Managed Protection Plus only)
```

---

### Reading an Adaptive Protection Alert

```json
{
  "alertId": "6e6be4a2-...",
  "policy": "projects/my-project/global/securityPolicies/prod-policy",
  "backendService": "my-backend",
  "confidence": 0.92,
  "attackVolume": {
    "ppsLongtermBaseline": 1000,
    "bpsLongtermBaseline": 5000000,
    "ppsShortterm": 50000,
    "bpsShortterm": 250000000
  },
  "suggestedRule": {
    "action": "deny(403)",
    "priority": 11,
    "match": {
      "expr": {
        "expression": "origin.region_code == 'CN' && request.headers['user-agent'].contains('python-requests')"
      }
    }
  },
  "requestStatistics": {
    "allowedRequests": 1000,
    "deniedRequests": 49000,
    "attackedRequests": 49000
  }
}
```

**Key fields to review:**
- `confidence` > 0.7: High confidence — safe to act on
- `attackVolume.ppsShortterm` vs `ppsLongtermBaseline`: Ratio shows attack multiplier
- `suggestedRule.match.expr.expression`: CEL expression to copy into a deny rule

---

### Auto-Deploy Configuration

```bash
# Enable auto-deploy with thresholds
gcloud compute security-policies update prod-policy \
  --enable-layer7-ddos-defense \
  --auto-deploy-load-threshold=0.7 \        # Trigger when attack load > 70% above baseline
  --auto-deploy-confidence-threshold=0.85 \ # Only auto-deploy if confidence > 85%
  --auto-deploy-expiration-sec=3600         # Auto-expire the deployed rule after 1 hour
```

**Auto-deploy risks and mitigations:**

| Risk | Mitigation |
|---|---|
| Blocking legitimate traffic | Set `confidence_threshold` > 0.85; monitor logs |
| Rule doesn't expire | Set `expiration_sec` to 1–4 hours; review and extend if needed |
| Overlapping with manual rules | Use priority gaps; auto-deploy uses low priority numbers |
| Alert fatigue | Set `load_threshold` high enough to ignore background noise |

---

### Standard DDoS vs. Managed Protection Plus

| Protection Type | Standard | Managed Protection Plus |
|---|---|---|
| **L3/L4 volumetric DDoS** | ✅ Always-on, automatic | ✅ Always-on |
| **L7 DDoS (HTTP flood)** | Manual rules only | ✅ Adaptive Protection with auto-deploy |
| **Network DDoS (passthrough LB)** | ❌ | ✅ Advanced network DDoS protection |
| **Google SRE DDoS response** | ❌ | ✅ 24/7 support during attacks |
| **Attack analytics** | Basic metrics | Detailed telemetry |

---

### DDoS Attack Response Playbook

```
Step 1 — DETECT (T+0 minutes)
  Symptoms: Error rate spike, latency spike, origin CPU 100%
  Check:
    gcloud compute security-policies get-adaptive-protection-log prod-policy
    Cloud Monitoring: loadbalancing.googleapis.com/https/request_count by response code

Step 2 — CHARACTERIZE (T+5 minutes)
  Identify attack pattern from Adaptive Protection alert or logs:
    Cloud Logging query:
      resource.type="http_load_balancer"
      httpRequest.status >= 200
      severity=DEFAULT
    Aggregate by: origin country, User-Agent, IP, URL path
    Look for: high volume from single country, identical User-Agents

Step 3 — CONTAIN (T+10 minutes)
  Deploy emergency block rule at high priority:
    gcloud compute security-policies rules create 10 \
      --security-policy=prod-policy \
      --expression="origin.region_code == 'CN'" \  # Adjust to attack source
      --action=deny-403 \
      --description="Emergency DDoS block - $(date)"

Step 4 — REFINE (T+30 minutes)
  Replace broad block with precise rule from Adaptive Protection suggestion
  Add preview rule first if time allows; otherwise enforce directly
  Monitor: request count returns to baseline

Step 5 — RECOVER (T+60 minutes)
  Verify origin health (CPU, error rate back to normal)
  Review blocked traffic for false positives in logs
  Adjust rule if legitimate traffic caught in block

Step 6 — POST-MORTEM
  Document attack signature and timeline
  Create permanent rules for patterns likely to recur
  Review rate limiting rules — tighten if needed
  Consider upgrading to Managed Protection Plus if attack recurs
```

---

## 9. Rate Limiting — Deep Dive

### Throttle vs. Rate-Based Ban

| Feature | `throttle` | `rate_based_ban` |
|---|---|---|
| **Behavior at limit** | Return error (429) for excess requests | Return error, then **ban source** for duration |
| **Source stays allowed** | ✅ Yes (just throttled) | ❌ No — banned for `ban_duration_sec` |
| **Ban duration** | N/A | 1–86400 seconds |
| **Use case** | API rate limiting, general traffic shaping | Brute-force protection, abuse prevention |
| **Recovery** | Automatic per interval | Must wait for ban to expire |

---

### Rate Limit Keys

```bash
# ALL — single counter for entire backend (rare; useful for emergency flood control)
--enforce-on-key=ALL

# IP — most common; count per source IP
--enforce-on-key=IP

# XFF-IP — trust X-Forwarded-For header for IP (use when behind trusted proxy)
--enforce-on-key=XFF-IP

# HTTP-HEADER — count per unique value of a specific header
--enforce-on-key=HTTP-HEADER \
--enforce-on-key-name="X-API-Key"      # Header name

# HTTP-COOKIE — count per unique value of a specific cookie
--enforce-on-key=HTTP-COOKIE \
--enforce-on-key-name="session_id"     # Cookie name

# HTTP-PATH — count per unique URL path (useful for detecting path enumeration)
--enforce-on-key=HTTP-PATH

# REGION-CODE — count per source country
--enforce-on-key=REGION-CODE

# USER-IP — Cloud Armor determines best IP from XFF or direct IP
--enforce-on-key=USER-IP
```

---

### Conform / Exceed / Enforce Actions

```
Request arrives
      │
      ▼
Count for key (IP/header/etc.) within interval window
      │
      ├── Count ≤ threshold → CONFORM action (typically: allow)
      │
      └── Count > threshold → EXCEED action (typically: deny-429)
                                │
                                └── [rate_based_ban only]
                                    Total count > ban threshold?
                                    → YES → Apply BAN for ban_duration_sec
                                    → NO  → Just apply exceed action
```

---

### Real-World Rate Limiting Examples

```bash
# ── Example 1: Login endpoint brute-force protection ─────

# Ban IP for 10 minutes after 20 failed-looking login attempts in 60s
gcloud compute security-policies rules create 410 \
  --security-policy=prod-policy \
  --expression="request.path == '/api/login' && request.method == 'POST'" \
  --action=rate_based_ban \
  --rate-limit-threshold-count=20 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=600 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP \
  --description="Login brute-force ban"

# ── Example 2: Per-region scraping protection ────────────

# Throttle all traffic from a region known for scraping
gcloud compute security-policies rules create 420 \
  --security-policy=prod-policy \
  --expression="origin.region_code == 'CN'" \
  --action=throttle \
  --rate-limit-threshold-count=500 \
  --rate-limit-threshold-interval-sec=60 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP \
  --description="Per-IP throttle for CN region"

# ── Example 3: API key rate limiting ─────────────────────

# Allow 10,000 req/hour per API key
gcloud compute security-policies rules create 430 \
  --security-policy=prod-policy \
  --expression="request.path.startsWith('/api/') && has(request.headers, 'x-api-key')" \
  --action=throttle \
  --rate-limit-threshold-count=10000 \
  --rate-limit-threshold-interval-sec=3600 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=HTTP-HEADER \
  --enforce-on-key-name="x-api-key" \
  --description="Per-API-key hourly limit"

# ── Example 4: Search endpoint bot throttle ──────────────

# Throttle /search — 10 req/10s per IP
gcloud compute security-policies rules create 440 \
  --security-policy=prod-policy \
  --expression="request.path.startsWith('/search')" \
  --action=throttle \
  --rate-limit-threshold-count=10 \
  --rate-limit-threshold-interval-sec=10 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP \
  --description="Search endpoint throttle"
```

---

## 10. Observability & Troubleshooting

### Cloud Monitoring Metrics

| Metric | Description | Alert On |
|---|---|---|
| `networksecurity.googleapis.com/https/request_count` split by `blocked` | Requests allowed vs. blocked | Spike in blocked requests |
| `networksecurity.googleapis.com/https/rule_request_count` | Requests matched per rule | Rule triggering unexpectedly |
| `networksecurity.googleapis.com/adaptive_protection/alert_count` | Adaptive Protection alerts | Any alert > 0 |
| `networksecurity.googleapis.com/https/rate_limit_request_count` | Rate limit events per rule | Sustained rate limit hits |
| `loadbalancing.googleapis.com/https/request_count` by `response_code` | Overall LB request count | Spike in 403/429 |

---

### Cloud Logging — Security Policy Fields

| Log Field | Description | Values |
|---|---|---|
| `jsonPayload.enforcedSecurityPolicy.name` | Name of enforced policy | `"prod-waf-policy"` |
| `jsonPayload.enforcedSecurityPolicy.priority` | Priority of matched rule | `1000` |
| `jsonPayload.enforcedSecurityPolicy.outcome` | What happened | `"DENY"`, `"ACCEPT"` |
| `jsonPayload.enforcedSecurityPolicy.configuredAction` | Configured action of rule | `"DENY"`, `"ALLOW"`, `"REDIRECT"` |
| `jsonPayload.previewSecurityPolicy.name` | Policy name for preview rule | `"prod-waf-policy"` |
| `jsonPayload.previewSecurityPolicy.priority` | Priority of preview rule match | `1010` |
| `jsonPayload.previewSecurityPolicy.outcome` | What would have happened | `"DENY"` |
| `jsonPayload.statusDetails` | Human-readable reason | `"body_denied_by_cloud_armor"` |
| `httpRequest.remoteIp` | Client IP address | `"203.0.113.5"` |
| `httpRequest.requestUrl` | Full request URL | `"https://example.com/api/..."` |
| `httpRequest.status` | HTTP response code | `403`, `200`, `429` |

**`statusDetails` values:**

| Value | Meaning |
|---|---|
| `body_denied_by_cloud_armor` | Request denied by enforced rule |
| `body_denied_by_cloud_armor_preview` | Would be denied (preview mode) |
| `ip_range_denied_by_cloud_armor` | IP/CIDR block rule matched |
| `request_rate_limited_by_cloud_armor` | Rate limit rule triggered |
| `redirected_by_cloud_armor` | Redirect action triggered |
| `response_cache_miss` | CDN cache miss (not Cloud Armor — informational) |

---

### Sample Cloud Logging Queries

```
# ── All requests blocked by Cloud Armor ──────────────────
resource.type="http_load_balancer"
jsonPayload.enforcedSecurityPolicy.outcome="DENY"

# ── Preview mode rule hits (what WOULD be blocked) ────────
resource.type="http_load_balancer"
jsonPayload.previewSecurityPolicy.priority=1010

# ── Specific WAF rule hits (rule 1000 = SQLi) ─────────────
resource.type="http_load_balancer"
jsonPayload.enforcedSecurityPolicy.priority=1000
jsonPayload.enforcedSecurityPolicy.outcome="DENY"

# ── Rate limit events ─────────────────────────────────────
resource.type="http_load_balancer"
jsonPayload.statusDetails="request_rate_limited_by_cloud_armor"

# ── Blocks from a specific country ────────────────────────
resource.type="http_load_balancer"
jsonPayload.enforcedSecurityPolicy.outcome="DENY"
httpRequest.remoteIp=~"^(CN|RU|KP)"  # Note: use geoip in expression, not IP pattern

# ── False positive hunting — blocked 2xx-expecting requests
resource.type="http_load_balancer"
jsonPayload.enforcedSecurityPolicy.outcome="DENY"
jsonPayload.enforcedSecurityPolicy.priority=1000  # Replace with your WAF rule priority
httpRequest.status=403

# ── All policy hits for a specific backend ────────────────
resource.type="http_load_balancer"
resource.labels.backend_service_name="my-backend-svc"
jsonPayload.enforcedSecurityPolicy.name="prod-waf-policy"

# ── Top blocked IPs (use in Log Analytics / BigQuery export)
resource.type="http_load_balancer"
jsonPayload.enforcedSecurityPolicy.outcome="DENY"
| group by httpRequest.remoteIp
| order by count desc
| limit 20
```

---

### Troubleshooting Decision Tree

```
Legitimate traffic being blocked (false positives)?
│
├── Which rule is blocking? Check priority in logs:
│   jsonPayload.enforcedSecurityPolicy.priority = ?
│
├── Is it a WAF pre-configured rule (1000–1999)?
│   ├── Set the rule to --preview and observe for 24h
│   ├── Identify which specific WAF sub-rule triggered (check statusDetails)
│   ├── Try lower sensitivity: {'sensitivity': 1}
│   └── Add exclusion for specific header/param causing false positive
│
├── Is it a rate limiting rule (400–499)?
│   ├── Review rate limit threshold — too low?
│   ├── Check if key is too broad (ALL or REGION-CODE catching real users)
│   └── Increase threshold or narrow match condition
│
└── Is it a geo-block or IP rule?
    ├── Check if VPN / proxy is masking real client location
    ├── Verify rule expression with: dig +short myip.opendns.com
    └── Add specific allowlist rule at lower priority for the affected IP

──────────────────────────────────────────────────────────

Traffic NOT being blocked when it should be?
│
├── Check: Is security policy attached to the correct backend?
│   gcloud compute backend-services describe ... --format="value(securityPolicy)"
│
├── Check: Is the rule in PREVIEW mode?
│   gcloud compute security-policies rules describe PRIORITY \
│     --security-policy=... --format="value(preview)"
│   → true = preview; remove --preview to enforce
│
├── Check: Is rule priority correct? (lower = higher priority)
│   A higher-priority ALLOW rule may be matching first
│   gcloud compute security-policies rules list --security-policy=...
│
├── Check: Is it an Internal LB? (Cloud Armor only covers External LBs)
│   gcloud compute forwarding-rules describe ... | grep loadBalancingScheme
│   → INTERNAL = NOT protected by Cloud Armor
│
└── Check: Is the match condition correct?
    Test CEL expression manually against sample log entries
    Use preview mode on the rule and send test traffic

──────────────────────────────────────────────────────────

geo-block not working (traffic still coming from blocked country)?
│
├── Check: Is the client using a VPN or proxy?
│   Cloud Armor evaluates the connecting IP, not XFF by default
│   If behind CDN proxy (Cloudflare, etc.), Cloud Armor sees CDN IP
│
├── Check: Is the rule using CEL or IP match?
│   IP/CIDR rules don't do geo lookup — use CEL with origin.region_code
│
├── Check: Is an earlier allow rule matching first?
│   A rule at priority 100 (allow all US) won't exclude country blocks at 200
│   Verify no allow rule before geo-block rule catches traffic first
│
└── Is the blocking rule in preview mode?
    grep previewSecurityPolicy in logs for the rule priority

──────────────────────────────────────────────────────────

Adaptive Protection alert — is it real?
│
├── confidence > 0.85 + attack volume >> baseline → Likely real → Act on it
├── confidence 0.6–0.85 → Review logs first → Deploy in preview mode
├── confidence < 0.6 → Likely noise → Monitor, don't auto-deploy
└── Check attack signature against actual logs to verify pattern
    before deploying suggested rule
```

---

## 11. Key Limits & Quotas

| Resource | Default Limit | Notes |
|---|---|---|
| **Security policies per project** | 200 | Edge + backend combined |
| **Rules per security policy** | 200 | Excluding default rule |
| **IP ranges per IP match rule** | 10 | Comma-separated in `src_ip_ranges` |
| **Rate limit threshold (max count)** | 10,000,000 | Per interval |
| **Rate limit interval options** | 10s, 30s, 60s, 120s, 180s, 240s, 300s | Fixed interval options only |
| **Ban duration (max)** | 86,400 seconds (24 hours) | `--ban-duration-sec` |
| **Adaptive Protection baseline period** | 1 hour minimum | Required before alerts fire |
| **Auto-deploy expiration (max)** | 86,400 seconds | `--auto-deploy-expiration-sec` |
| **Named IP list size** | Up to millions of IPs | Google-managed; not user-configurable |
| **CEL expression length** | 1,000 characters | Per rule expression |
| **WAF exclusions per rule** | 50 | Per `preconfiguredWafConfig.exclusions` |
| **Preview rules per policy** | No separate limit | Counts toward 200 rule limit |
| **Backends per security policy** | No documented hard limit | One policy per backend service |
| **Edge security policies per project** | Included in 200 policy limit | |
| **Network edge policies (Plus only)** | Included in 200 policy limit | |
| **reCAPTCHA Enterprise site keys** | 1 per policy | For bot management |
| **Transaction/import size** | No published limit | Export/import via JSON |

> 📝 Raise quotas: **GCP Console → IAM & Admin → Quotas → Filter "Cloud Armor"**

---

## 12. Gotchas & Best Practices

### Common Pitfalls

- **❌ Enforcing new WAF rules without preview mode first** — Pre-configured WAF rules (especially SQLi, XSS at sensitivity 2+) are notorious for false positives on legitimate API traffic, search queries with special characters, and rich-text inputs. Always deploy with `--preview` for at least 24–48 hours. Review logs for your actual traffic patterns before enforcing.

- **❌ Deleting the default rule (priority 2147483647)** — Cloud Armor requires a default rule. Deleting it causes API errors and leaves rule evaluation behavior undefined. The default rule must always exist. Change its action between `allow` (permissive default) and `deny(403)` (zero-trust default) based on your model.

- **❌ Using sequential priorities (1, 2, 3...)** — Leaves no room to insert rules between existing ones later. Use gaps: 100, 200, 300, 500, 1000. This allows inserting emergency rules (e.g., priority 150) without renumbering.

- **❌ Attaching Cloud Armor to Internal Load Balancers** — Cloud Armor only protects **external** Cloud Load Balancers. Attaching a security policy to an Internal LB's backend service will fail. Use VPC firewall rules for internal traffic protection.

- **❌ Trusting X-Forwarded-For blindly** — When using `XFF-IP` as rate limit key or in `inIpRange()`, verify that your architecture actually uses a trusted proxy that sets XFF correctly. Otherwise, attackers can spoof XFF headers to bypass IP-based rules. Cloud Armor appends the real connecting IP to XFF but clients can set arbitrary values in earlier XFF headers.

- **❌ Disabling entire WAF rule sets for one false positive** — If rule `sqli-v33-stable` is flagging a specific form field, don't disable the entire rule set. Use `opt_out_rule_ids` to exclude specific sub-rules, or add a field exclusion for the specific query parameter or header causing the false positive.

- **❌ Not accounting for CDN cache hits in security policy design** — A backend security policy does NOT inspect cached CDN responses. An attacker can still receive a 200 response with stolen content if it's cached. Use an **edge security policy** for full enforcement including cache hits.

- **❌ Named IP list staleness** — Google-curated threat intelligence lists (Tor, malicious IPs) are updated asynchronously. There can be a lag of minutes to hours between a new IP being added to a threat feed and being reflected in Cloud Armor. Do not rely solely on named lists for real-time blocking during an active attack.

- **❌ Creating WAF + rate limit rules without testing order** — A rate limit rule at priority 400 that matches all traffic (`true`) can block legitimate users before the WAF at priority 1000 ever runs — or vice versa. Map out your rule evaluation flow on paper before deployment.

- **❌ Geo-blocking without VPN exception** — If your corporate users access services via a VPN with an exit node in a blocked country, they will be blocked. Always add an explicit allowlist rule for your VPN's exit IP ranges at a lower priority number than the geo-block rule.

---

### Best Practices

- **✅ Use a layered rule structure** — Organize rules in a consistent priority band pattern:
  - 1–99: Emergency/override rules
  - 100–199: Allowlist (trusted IPs, CDN IPs)
  - 200–299: Denylist (known bad IPs, geo-blocks, threat intel)
  - 300–399: Request characteristic blocks (User-Agent, method, header checks)
  - 400–499: Rate limiting rules
  - 1000–1999: Pre-configured WAF rules
  - 2000–2999: Custom CEL rules
  - 2147483647: Default rule

- **✅ Start all WAF rules in preview mode** — Never enforce a new WAF rule directly in production. Deploy with `--preview`, let it run for at least 24–48 hours, review logs for false positives, tune sensitivity/exclusions, then promote to enforcement.

- **✅ Log at INFO level and export to BigQuery** — Route Cloud Armor logs to BigQuery using a log sink for long-term analysis, dashboarding, and alert tuning. Logs are essential for identifying false positive patterns over time.

- **✅ Use the edge policy for GCS backend buckets** — If you host static assets (images, JS, CSS) in GCS with a backend bucket, you MUST use an **edge security policy** (not backend policy) to geo-block or IP-block access. Backend policies cannot attach to backend buckets.

- **✅ Tighten the default rule over time** — Start with `default: allow` and build explicit deny rules. Once you have good rule coverage and understand your traffic patterns, switch `default: deny(403)` and add explicit allow rules for known-good sources. The zero-trust model is stronger.

- **✅ Combine Cloud Armor with Cloud CDN for cost efficiency** — Edge policies are cheaper to evaluate than backend policies (no WAF processing cost). Use edge policies for cheap geo-blocking and IP filtering; reserve backend policies for WAF and rate limiting which require deeper inspection.

- **✅ Set up Adaptive Protection alerts before you need them** — Enable Adaptive Protection (`--enable-layer7-ddos-defense`) on all production policies even if you don't use auto-deploy. The ML model needs baseline data. If you enable it during an attack, it won't have enough history to be effective.

- **✅ Test with synthetic traffic before go-live** — Use tools like `curl` or `k6` with specific patterns (SQL strings in params, scanner User-Agents, blocked country VPNs) to validate rules trigger correctly in preview mode before enforcement.

- **✅ Optimize cost with rule consolidation** — Cloud Armor charges per policy ($5/month), per rule ($1/month), and per million requests ($0.75M). Consolidate similar conditions with `||` operators to reduce rule count. Use edge policies (lower cost) for simple IP/geo checks rather than backend policies.

- **✅ Document every rule with `--description`** — Include the date, ticket number, and reason for each rule. Rules without descriptions become unmanageable after a few months. Use descriptions like: `"Block SQLi - JIRA-1234 - 2026-03-15 - reviewed by @alice"`.

- **✅ Review and rotate emergency rules** — Emergency rules created during incidents (DDoS, attacks) should be time-limited. Set a calendar reminder to review and remove or formalize them within 7 days. Stale emergency rules accumulate and create unintended blocks.

- **✅ Use Terraform for all rule management** — Manual gcloud changes to production security policies are risky and unaudited. Manage policies as code with Terraform, use PR reviews for rule changes, and maintain a clear audit trail of who changed what and when.

---

*Last updated: March 2026 | Based on GCP Cloud Armor documentation and production security patterns*
