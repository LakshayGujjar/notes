# GCP App Engine Cheatsheet

> **TL;DR:** App Engine is GCP's fully managed PaaS — deploy code (not containers) and Google handles infrastructure, scaling, and operations; choose Standard for fast scale-to-zero serverless, or Flexible for custom runtimes and full container control.

---

## Table of Contents

1. [Overview & Core Concepts](#1-overview--core-concepts)
2. [Standard vs. Flexible Environment Deep Dive](#2-standard-vs-flexible-environment-deep-dive)
3. [Application Structure & Configuration](#3-application-structure--configuration)
4. [Runtimes & Language Support](#4-runtimes--language-support)
5. [Deploying Applications](#5-deploying-applications)
6. [Scaling Configuration](#6-scaling-configuration)
7. [Services & Versions](#7-services--versions)
8. [Networking](#8-networking)
9. [Datastore, Storage & Integrations](#9-datastore-storage--integrations)
10. [IAM & Security](#10-iam--security)
11. [Task Queues & Cron Jobs](#11-task-queues--cron-jobs)
12. [Observability](#12-observability)
13. [gcloud CLI Quick Reference](#13-gcloud-cli-quick-reference)
14. [Terraform Snippet](#14-terraform-snippet)
15. [Migration & Modernization Paths](#15-migration--modernization-paths)
16. [Troubleshooting Quick Reference](#16-troubleshooting-quick-reference)

---

## 1. Overview & Core Concepts

App Engine is GCP's **Platform-as-a-Service (PaaS)** offering. You deploy application code (or containers in Flexible), and Google fully manages the underlying servers, OS patching, load balancing, and auto-scaling. Each GCP project can have **exactly one** App Engine application, but that application can run many services and versions simultaneously.

### Key Terminology

| Term | Description |
|---|---|
| **Application** | The top-level App Engine resource — one per GCP project, bound to a region |
| **Service** | A logical component of an app (analogous to a microservice); the default service is named `default` |
| **Version** | An immutable, deployable snapshot of a service's code + config; multiple versions can coexist |
| **Instance** | A single running copy of a version; App Engine manages instance count automatically |
| **Runtime** | The language/execution environment (e.g., `python312`, `nodejs20`, `go122`) |
| **Sandbox** | The isolated execution environment in Standard that restricts filesystem and network access |
| **Scaling** | The policy controlling how many instances to run — automatic, basic, or manual |
| **Warm-up Request** | An HTTP request to `/_ah/warmup` sent to new instances before live traffic hits them |
| **Resident Instance** | A permanently running instance (never shut down); used with manual scaling |
| **Dynamic Instance** | An instance started and stopped by App Engine's autoscaler |
| **Handler** | A URL pattern → script/static mapping defined in `app.yaml` (Standard only) |
| **Dispatch Rules** | Cross-service URL routing rules defined in `dispatch.yaml` |
| **Bundled Services** | Legacy App Engine APIs (Datastore, Memcache, Task Queues, Blobstore) built into the Standard sandbox |

### App Engine vs. Other GCP Compute Options

| Feature | App Engine Standard | App Engine Flexible | Cloud Run | GKE | GCE |
|---|---|---|---|---|---|
| **Abstraction level** | PaaS (code) | PaaS (container) | Serverless (container) | CaaS (container) | IaaS (VM) |
| **Deploy unit** | Source code | Container / source | Container | Pod | VM image |
| **Scale to zero** | ✅ | ❌ (min 1) | ✅ | ❌ | ❌ |
| **Cold starts** | Yes (mitigable) | Slow (1–3 min) | Yes (fast) | Rare | No |
| **Custom runtimes** | ❌ | ✅ (Dockerfile) | ✅ | ✅ | ✅ |
| **SSH access** | ❌ | ✅ | ❌ | ✅ | ✅ |
| **Local disk** | Read-only `/tmp` (32MB) | Read/write (ephemeral) | Read-only `/tmp` (32MB) | Configurable | Full |
| **Max request timeout** | 10 min (background: 24h) | 60 min | 60 min | Unlimited | Unlimited |
| **Pricing model** | Per instance-hour (F-class) | Per vCPU/memory/disk | Per request (CPU+mem) | Per node | Per VM |
| **Infra management** | None | Minimal | None | Cluster ops | Full |
| **Best for** | Web apps, quick deploys | Polyglot, long requests | Microservices, APIs | Complex orchestration | Full VM control |

> **Note:** App Engine Standard is the right choice when you want zero-ops PaaS with fast auto-scaling and you're using a supported runtime. Choose Flexible only when you need capabilities unavailable in Standard (custom runtimes, background threads, writable disk, or longer request timeouts).

---

## 2. Standard vs. Flexible Environment Deep Dive

### Side-by-Side Comparison

| Attribute | Standard Environment | Flexible Environment |
|---|---|---|
| **Supported runtimes** | Python, Java, Node.js, Go, PHP, Ruby (specific versions) | Any language via Dockerfile; also managed runtimes |
| **Runtime model** | Proprietary sandbox (gVisor) | Docker container on GCE VM |
| **Scale to zero** | ✅ Yes — scales to 0 instances | ❌ No — minimum 1 instance always running |
| **Cold start speed** | Fast: milliseconds to ~2 seconds | Slow: 1–3 minutes (new VM + container pull) |
| **Startup time** | Near-instant | Minutes |
| **Local disk write** | ❌ No (read-only `/tmp`, 32 MB max) | ✅ Yes (ephemeral disk, configurable size) |
| **Background threads** | ❌ Limited (Standard Gen 1) / ✅ Gen 2 | ✅ Full support |
| **SSH into instance** | ❌ Not available | ✅ Available via `gcloud app instances ssh` |
| **Network access** | ✅ Full outbound (Gen 2) | ✅ Full outbound |
| **WebSockets** | ❌ (Gen 1) / ✅ (Gen 2) | ✅ Full support |
| **Max request timeout** | 10 minutes (60 min for task handlers) | 60 minutes |
| **Pricing unit** | F1–F4 / B1–B4 instance classes (per hour) | Per vCPU, memory, and persistent disk per minute |
| **Minimum instances** | 0 (scales to zero) | 1 (always at least one instance) |
| **Free tier** | ✅ 28 instance-hours/day | ❌ No free tier |
| **Bundled services** | ✅ Available (Datastore, Memcache, etc.) | ⚠️ Partial (via App Engine SDK wrapper) |
| **Custom Dockerfile** | ❌ Not supported | ✅ Full Dockerfile support |
| **OS-level access** | ❌ Sandboxed | ✅ Root in container, SSH to VM |
| **GPU support** | ❌ | ❌ (use GKE or Cloud Run) |
| **Health checks** | Basic liveness via `/_ah/health` | Configurable liveness + readiness probes |

### When to Choose Standard

- You use Python, Java, Node.js, Go, PHP, or Ruby (supported versions).
- Traffic is spiky or unpredictable and you need scale-to-zero.
- You want the lowest cost for low-traffic apps (free tier).
- You need fast cold starts.
- You want zero infrastructure management.

### When to Choose Flexible

- You need a custom runtime (e.g., Rust, C++, or a specific library requiring native binaries).
- Your app requires writable disk access or background threads.
- You need request timeouts longer than 10 minutes.
- You're running a containerized app and want PaaS-style management.
- You need SSH access for debugging.

> **Note:** For most new projects, **Cloud Run** is a better choice than App Engine Flexible — it offers the same container flexibility with faster scaling, lower minimum cost, and a more modern feature set.

---

## 3. Application Structure & Configuration

### Project Layout (Standard)

```
my-app/
├── app.yaml            # Required: App Engine service configuration
├── main.py             # Application entrypoint
├── requirements.txt    # Python dependencies
├── static/             # Static assets
│   └── style.css
├── templates/          # HTML templates
├── cron.yaml           # Optional: Cron job definitions
├── dispatch.yaml       # Optional: Cross-service URL routing
└── queue.yaml          # Optional: Task queue configuration
```

### Annotated `app.yaml` — Standard Environment (Python)

```yaml
# ── Runtime ────────────────────────────────────────────────
runtime: python312          # Required: language runtime identifier
# entrypoint is optional; defaults to gunicorn for Python
entrypoint: gunicorn -b :$PORT main:app

# ── Environment ────────────────────────────────────────────
env: standard               # standard (default) or flex

# ── Environment Variables ──────────────────────────────────
env_variables:
  ENV: production
  LOG_LEVEL: info
  # Never put secrets here — use Secret Manager instead

# ── Instance Class (CPU/memory tier) ──────────────────────
# F1=256MB, F2=512MB, F4=1GB, F4_1G=2GB (Standard)
# B1–B4 for manual/basic scaling
instance_class: F2

# ── Scaling ────────────────────────────────────────────────
automatic_scaling:
  min_instances: 0
  max_instances: 10
  target_cpu_utilization: 0.65
  max_concurrent_requests: 80

# ── URL Handlers (Standard only) ──────────────────────────
handlers:
  # Serve static files directly (bypass app server)
  - url: /static
    static_dir: static
    expiration: "1d"
    http_headers:
      Cache-Control: public, max-age=86400

  # Require Google login for all other routes
  - url: /admin/.*
    script: auto
    login: admin

  # All other routes go to the app
  - url: /.*
    script: auto

# ── Files to exclude from deployment ─────────────────────
skip_files:
  - ^(.*/)?#.*#$
  - ^(.*/)?.*~$
  - ^(.*/)?.*\.pyc$
  - ^(.*/)?\..*$
  - ^tests/.*$
  - ^venv/.*$

# ── VPC Access ─────────────────────────────────────────────
vpc_access_connector:
  name: projects/my-project/locations/us-central1/connectors/my-connector
  egress_setting: private-ranges-only

# ── Inbound Services ───────────────────────────────────────
inbound_services:
  - warmup   # Enable /_ah/warmup requests for pre-warming instances
```

### Annotated `app.yaml` — Flexible Environment (Custom Runtime)

```yaml
# ── Runtime ────────────────────────────────────────────────
runtime: custom             # Use 'custom' to provide your own Dockerfile
env: flex                   # Required: enables Flexible environment

# ── Resources ──────────────────────────────────────────────
resources:
  cpu: 2
  memory_gb: 2.0
  disk_size_gb: 20          # Writable persistent disk (Flexible only)

# ── Scaling ────────────────────────────────────────────────
automatic_scaling:
  min_num_instances: 1      # Note: min must be >= 1 for Flexible
  max_num_instances: 10
  cool_down_period_sec: 180
  cpu_utilization:
    target_utilization: 0.65

# ── Environment Variables ──────────────────────────────────
env_variables:
  ENV: production
  DB_HOST: 127.0.0.1        # Cloud SQL via proxy runs on localhost

# ── Health Check ───────────────────────────────────────────
health_check:
  enable_health_check: true
  check_interval_sec: 5
  timeout_sec: 4
  unhealthy_threshold: 2
  healthy_threshold: 2
  restart_threshold: 60
  path: /healthz

liveness_check:
  path: /healthz
  check_interval_sec: 30
  timeout_sec: 4
  failure_threshold: 3
  success_threshold: 1

readiness_check:
  path: /ready
  check_interval_sec: 5
  timeout_sec: 4
  failure_threshold: 2
  success_threshold: 2
  app_start_timeout_sec: 300

# ── Network ────────────────────────────────────────────────
network:
  forwarded_ports:
    - 8080/tcp

# ── Beta Settings (Cloud SQL, etc.) ───────────────────────
beta_settings:
  cloud_sql_instances: my-project:us-central1:my-db
```

> **Note:** The `handlers` section is only valid for Standard environment. Flexible environment handles routing entirely within the container.

---

## 4. Runtimes & Language Support

### Standard Environment Runtimes

| Language | Supported Versions | Generation | Notes |
|---|---|---|---|
| **Python** | 3.8, 3.9, 3.10, 3.11, 3.12 | Gen 2 | 3.7 and below = Gen 1 (legacy) |
| **Java** | 11, 17, 21 | Gen 2 | Java 8 = Gen 1 (legacy) |
| **Node.js** | 18, 20, 22 | Gen 2 | Node 10/12 = Gen 1 (legacy) |
| **Go** | 1.21, 1.22 | Gen 2 | Go 1.11/1.12 = Gen 1 (legacy) |
| **PHP** | 7.4, 8.1, 8.2, 8.3 | Gen 2 | PHP 5.5/7.2 = Gen 1 (legacy) |
| **Ruby** | 3.2, 3.3 | Gen 2 | No Gen 1 Ruby |

### Generation 1 vs Generation 2 (Standard)

| Feature | Gen 1 (Legacy) | Gen 2 (Current) |
|---|---|---|
| Sandbox | Proprietary (restricted) | gVisor (Linux-compatible) |
| Full network access | ❌ | ✅ |
| Background threads | ❌ | ✅ |
| Writable `/tmp` | 32 MB | 512 MB+ |
| Any pip package | ❌ (restricted) | ✅ |
| Bundled Services | ✅ (native) | ✅ (via `appengine-sdk`) |
| WebSockets | ❌ | ✅ |

> **Note:** All new development should target **Gen 2** runtimes. Gen 1 runtimes have been in maintenance-only mode and will eventually reach end-of-life.

### Flexible Environment — Custom Runtime (Dockerfile)

```yaml
# app.yaml
runtime: custom
env: flex
```

```dockerfile
# Dockerfile — placed in the same directory as app.yaml
FROM python:3.12-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# App Engine Flexible injects PORT (default 8080)
ENV PORT=8080

# Use exec form to handle SIGTERM gracefully
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "2", "main:app"]
```

### Runtime Naming Convention

```
# Standard
runtime: python312   # python + major + minor (no dot)
runtime: nodejs20
runtime: go122
runtime: java21
runtime: php83
runtime: ruby33

# Flexible (managed runtimes)
runtime: python
runtime: nodejs
runtime: go
runtime: java
# Plus env: flex and a runtime_config block for version pinning:
runtime_config:
  operating_system: ubuntu22
  runtime_version: "3.12"
```

---

## 5. Deploying Applications

### Basic Deployment

```bash
# Deploy the default service using app.yaml in the current directory
gcloud app deploy

# Deploy a specific app.yaml (useful for multiple services)
gcloud app deploy services/api/app.yaml

# Deploy without promoting (new version gets 0% traffic)
gcloud app deploy --no-promote

# Deploy a specific version name
gcloud app deploy --version=v20240101

# Deploy to a specific project
gcloud app deploy --project=my-project
```

### Promoting a Version (Traffic Migration)

```bash
# Migrate 100% of traffic to a specific version
gcloud app services set-traffic default \
  --splits=v20240101=1 \
  --migrate

# Immediate switch (no gradual migration)
gcloud app services set-traffic default \
  --splits=v20240101=1
```

### Traffic Splitting (Canary / A/B)

```bash
# Split traffic: 90% stable, 10% canary
gcloud app services set-traffic default \
  --splits=v20240101=0.9,v20240115=0.1

# Split by IP address (sticky sessions per client IP)
gcloud app services set-traffic default \
  --splits=v1=0.5,v2=0.5 \
  --split-by=ip

# Split by cookie (sticky sessions per browser)
gcloud app services set-traffic default \
  --splits=v1=0.5,v2=0.5 \
  --split-by=cookie

# Split randomly (uniform, no stickiness)
gcloud app services set-traffic default \
  --splits=v1=0.5,v2=0.5 \
  --split-by=random
```

### Rollback

```bash
# List versions to find the previous stable one
gcloud app versions list --service=default

# Migrate all traffic back to the previous version
gcloud app services set-traffic default \
  --splits=v20231201=1
```

### Deploying Multiple Services

```bash
# Deploy several services in one command
gcloud app deploy \
  services/frontend/app.yaml \
  services/api/app.yaml \
  services/worker/app.yaml

# Deploy only the cron, dispatch, and queue configs
gcloud app deploy cron.yaml dispatch.yaml queue.yaml
```

### `cron.yaml` — Scheduled Tasks

```yaml
# cron.yaml
cron:
  - description: "Daily data export at 2 AM UTC"
    url: /tasks/export-data
    schedule: every day 02:00
    timezone: UTC
    retry_parameters:
      job_retry_limit: 3

  - description: "Hourly cache refresh"
    url: /tasks/refresh-cache
    schedule: every 1 hours
    target: worker          # Route to the 'worker' service

  - description: "Every weekday morning"
    url: /tasks/morning-report
    schedule: every monday,tuesday,wednesday,thursday,friday 08:00
    timezone: America/New_York
```

### `dispatch.yaml` — URL Routing Across Services

```yaml
# dispatch.yaml — route requests to specific services by URL
dispatch:
  # API requests go to the 'api' service
  - url: "*/api/*"
    service: api

  # Admin routes go to the 'admin' service
  - url: "*/admin/*"
    service: admin

  # Static asset CDN service
  - url: "static.myapp.com/*"
    service: static-server

  # Everything else goes to default
  - url: "*/*"
    service: default
```

```bash
# Deploy dispatch rules
gcloud app deploy dispatch.yaml
```

### `queue.yaml` — Task Queue Configuration

```yaml
# queue.yaml
queue:
  - name: default
    rate: 5/s
    retry_parameters:
      task_retry_limit: 3
      task_age_limit: 86400s  # 24 hours

  - name: high-priority
    rate: 100/s
    max_concurrent_requests: 20

  - name: batch-processing
    rate: 1/s
    retry_parameters:
      task_retry_limit: 5
      min_backoff_seconds: 10
      max_backoff_seconds: 300
      max_doublings: 4
```

---

## 6. Scaling Configuration

### Automatic Scaling (Default — Recommended)

App Engine manages instance count based on load metrics.

```yaml
# app.yaml — automatic_scaling
automatic_scaling:
  # Minimum instances kept alive (0 = scale to zero, costs nothing at rest)
  min_instances: 0

  # Hard cap on instance count
  max_instances: 20

  # Scale up when CPU exceeds this (0.0–1.0)
  target_cpu_utilization: 0.65

  # Scale up when throughput exceeds this
  target_throughput_utilization: 0.65

  # Max simultaneous requests per instance
  max_concurrent_requests: 80

  # Spin up new instances before pending requests wait this long
  max_pending_latency: automatic   # or e.g. "30ms"

  # Minimum latency before scaling down
  min_pending_latency: 30ms

  # Keep this many instances idle and ready
  min_idle_instances: 1
  max_idle_instances: automatic
```

### Basic Scaling

Creates instances on demand; instances shut down when idle. Best for intermittent or offline work.

```yaml
# app.yaml — basic_scaling
basic_scaling:
  # Shut down instance after this idle period
  idle_timeout: 10m

  # Hard cap
  max_instances: 5
```

### Manual Scaling

A fixed number of permanent (resident) instances. No scale-up/down — instances run 24/7.

```yaml
# app.yaml — manual_scaling
manual_scaling:
  instances: 3
```

```bash
# Change the instance count at runtime (no redeploy needed)
gcloud app versions update VERSION \
  --service=default \
  --instances=5
```

### Scaling Type Comparison

| Attribute | Automatic | Basic | Manual |
|---|---|---|---|
| **Scales to zero** | ✅ (min_instances=0) | ✅ (on idle_timeout) | ❌ |
| **Scales dynamically** | ✅ | ✅ | ❌ (fixed count) |
| **Instance type** | Dynamic | Dynamic | Resident |
| **Warm-up requests** | ✅ Supported | ❌ | ❌ |
| **Best for** | Web apps, APIs | Background jobs | Constant workloads |
| **Cold starts** | Yes | Yes | No |

### Warm-Up Requests

Enable in `app.yaml` to pre-initialize instances before live traffic:

```yaml
# app.yaml
inbound_services:
  - warmup
```

```python
# main.py — handle the warmup request
from flask import Flask
app = Flask(__name__)

@app.route('/_ah/warmup')
def warmup():
    # Pre-load models, open DB connections, warm caches here
    return 'Warmed up', 200
```

> **Note:** Warm-up requests are only sent when `min_instances: 0` and a new instance is about to receive traffic. Set `min_instances: 1` or higher to eliminate cold starts entirely (at added cost).

---

## 7. Services & Versions

### Multi-Service Architecture

```
my-app/
├── frontend/
│   └── app.yaml     # service: default (or service: frontend)
├── api/
│   └── app.yaml     # service: api
├── worker/
│   └── app.yaml     # service: worker
├── dispatch.yaml    # Routes URLs to the right service
└── cron.yaml        # Can target specific services
```

Each `app.yaml` declares its service name:

```yaml
# api/app.yaml
service: api
runtime: python312
```

### Inter-Service Communication

```python
# Call another App Engine service internally
import requests
import google.auth
import google.auth.transport.requests

def call_api_service(path: str) -> dict:
    """Call the 'api' service with an authenticated internal request."""
    # Internal URL pattern: https://{service}-dot-{project}.appspot.com
    url = f"https://api-dot-my-project.appspot.com{path}"

    # Use ID token for service-to-service auth
    creds, project = google.auth.default()
    auth_req = google.auth.transport.requests.Request()
    creds.refresh(auth_req)

    response = requests.get(
        url,
        headers={"Authorization": f"Bearer {creds.token}"}
    )
    return response.json()
```

> **Note:** Internal URLs follow the pattern `https://{version}-dot-{service}-dot-{project}.appspot.com`. Omitting the version routes to the traffic-serving version.

### Version Management

```bash
# List all versions of a service
gcloud app versions list --service=default

# Describe a specific version
gcloud app versions describe v20240101 --service=default

# Stop a version (keeps it but stops instances — billing stops)
gcloud app versions stop v20240101 --service=default

# Start a previously stopped version
gcloud app versions start v20240101 --service=default

# Delete a version (irreversible — cannot have traffic)
gcloud app versions delete v20240101 --service=default
```

### Traffic Migration vs. Splitting

| Feature | Traffic Migration | Traffic Splitting |
|---|---|---|
| **Purpose** | Move 100% traffic to a new version | Divide traffic between versions |
| **Gradual rollout** | ✅ `--migrate` flag (gradual) | ✅ Percentage-based |
| **Rollback** | Re-migrate to old version | Adjust percentages |
| **Use case** | Standard deploys | A/B testing, canary deploys |

```bash
# Blue/Green: deploy without traffic, validate, then cut over
gcloud app deploy --no-promote --version=green
# ... validate green version at https://green-dot-default-dot-project.appspot.com
gcloud app services set-traffic default --splits=green=1
```

---

## 8. Networking

### Custom Domains & SSL

```bash
# Verify domain ownership first (via Search Console or DNS TXT record)
gcloud domains verify mycompany.com

# Map a custom domain to your App Engine app
gcloud app domain-mappings create www.mycompany.com \
  --certificate-management=automatic   # Google-managed SSL cert

# List domain mappings
gcloud app domain-mappings list

# Delete a domain mapping
gcloud app domain-mappings delete www.mycompany.com
```

### Serverless VPC Access Connector

Required to reach private VPC resources (Cloud SQL private IP, Memorystore, internal services):

```bash
# Create a connector
gcloud compute networks vpc-access connectors create my-connector \
  --network=default \
  --region=us-central1 \
  --range=10.8.0.0/28

# Reference it in app.yaml
```

```yaml
# app.yaml
vpc_access_connector:
  name: projects/my-project/locations/us-central1/connectors/my-connector
  egress_setting: private-ranges-only  # or 'all-traffic' to route all egress via VPC
```

### Ingress Controls

```yaml
# app.yaml — restrict who can send requests to this service
network:
  # Options: all | internal | internal-and-cloud-load-balancing
  ingress: internal-and-cloud-load-balancing
```

```bash
# Update ingress setting without redeploying
gcloud app services update default \
  --ingress=internal-and-cloud-load-balancing
```

### App Engine Firewall Rules

```bash
# Allow only traffic from a specific IP range
gcloud app firewall-rules create 100 \
  --action=allow \
  --source-range=203.0.113.0/24 \
  --description="Office IP range"

# Deny all other traffic (set as the default rule)
gcloud app firewall-rules update default \
  --action=deny

# List all firewall rules (ordered by priority)
gcloud app firewall-rules list
```

### Identity-Aware Proxy (IAP)

IAP enforces Google identity authentication in front of App Engine — no application code changes needed:

```bash
# Enable IAP on App Engine
gcloud iap web enable --resource-type=app-engine --service=default

# Grant a user access through IAP
gcloud iap web add-iam-policy-binding \
  --resource-type=app-engine \
  --member=user:developer@mycompany.com \
  --role=roles/iap.httpsResourceAccessor
```

```python
# Verify IAP JWT in your app (defense-in-depth)
from google.auth.transport import requests as google_requests
from google.oauth2 import id_token

def verify_iap_jwt(iap_jwt: str, expected_audience: str) -> dict:
    """Validate the IAP-signed JWT header."""
    certs_url = "https://www.gstatic.com/iap/verify/public_key"
    return id_token.verify_token(
        iap_jwt,
        google_requests.Request(),
        audience=expected_audience,
        certs_url=certs_url,
    )
```

### Cloud Armor Integration

Route through a Global Load Balancer to apply Cloud Armor WAF policies:

```bash
# Create a serverless NEG for App Engine
gcloud compute network-endpoint-groups create app-engine-neg \
  --region=us-central1 \
  --network-endpoint-type=serverless \
  --app-engine-service=default

# Create backend service and attach Cloud Armor policy
gcloud compute backend-services create app-engine-backend \
  --global \
  --security-policy=my-armor-policy

gcloud compute backend-services add-backend app-engine-backend \
  --global \
  --network-endpoint-group=app-engine-neg \
  --network-endpoint-group-region=us-central1
```

---

## 9. Datastore, Storage & Integrations

### Recommended Data Backends

| Use Case | Recommended Service | Notes |
|---|---|---|
| Structured NoSQL | Firestore (Native mode) | Default for new apps; real-time, mobile-ready |
| Legacy Datastore apps | Firestore (Datastore mode) | 100% Datastore-compatible API |
| Relational DB | Cloud SQL (PostgreSQL/MySQL) | Use Unix socket + connection pooling |
| In-memory cache | Memorystore (Redis) | Requires VPC connector from Standard |
| Object/file storage | Cloud Storage | Via GCS client library |
| Full-text search | Firestore + Algolia / Vertex AI | No native search engine |
| Analytics / OLAP | BigQuery | Stream inserts from App Engine |

### Cloud SQL Connection (Standard)

```python
# main.py — Cloud SQL via Unix domain socket (recommended for Standard)
import sqlalchemy
import os

def create_connection_pool() -> sqlalchemy.engine.Engine:
    db_user = os.environ["DB_USER"]
    db_pass = os.environ["DB_PASS"]
    db_name = os.environ["DB_NAME"]
    # Cloud SQL instance connection name: project:region:instance
    instance_connection_name = os.environ["INSTANCE_CONNECTION_NAME"]

    pool = sqlalchemy.create_engine(
        sqlalchemy.engine.url.URL.create(
            drivername="postgresql+pg8000",
            username=db_user,
            password=db_pass,
            database=db_name,
            # App Engine uses a Unix socket to connect to Cloud SQL
            query={"unix_sock": f"/cloudsql/{instance_connection_name}/.s.PGSQL.5432"}
        ),
        pool_size=5,
        max_overflow=2,
        pool_timeout=30,
        pool_recycle=1800,
    )
    return pool

# Initialize once at module level (shared across requests in the same instance)
db_pool = create_connection_pool()
```

```yaml
# app.yaml — reference the Cloud SQL instance
beta_settings:
  cloud_sql_instances: my-project:us-central1:my-postgres-db
```

### Cloud Storage Integration

```python
from google.cloud import storage

def upload_file(bucket_name: str, source_path: str, dest_blob: str) -> str:
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(dest_blob)
    blob.upload_from_filename(source_path)
    return f"gs://{bucket_name}/{dest_blob}"
```

### Firestore Integration

```python
from google.cloud import firestore

db = firestore.Client()

def save_user(user_id: str, data: dict) -> None:
    db.collection("users").document(user_id).set(data)

def get_user(user_id: str) -> dict | None:
    doc = db.collection("users").document(user_id).get()
    return doc.to_dict() if doc.exists else None
```

### Bundled Services vs. Standalone (Legacy Migration)

| Bundled Service | Modern Equivalent | Migration Effort |
|---|---|---|
| `ndb` / Datastore API | Firestore (Datastore mode) via `google-cloud-datastore` | Low — API is compatible |
| Memcache | Memorystore (Redis) via `redis-py` | Medium — requires VPC connector |
| Task Queues (push) | Cloud Tasks | Medium — similar API, different client |
| Task Queues (pull) | Pub/Sub | Medium — different paradigm |
| Blobstore | Cloud Storage | Low — drop-in replacement |
| Mail API | SendGrid / Mailgun / SMTP relay | Low |
| Images API | Cloud Storage + Imgix / Cloud CDN | Medium |
| Search API | Firestore + Vertex AI Search | High |

> **Note:** Bundled Services work only on Gen 1 runtimes natively. Gen 2 runtimes can use them via the `appengine-python-standard` SDK package, but this is considered legacy — prefer standalone GCP services for all new code.

---

## 10. IAM & Security

### Default Service Account

Every App Engine app gets a default service account:
`my-project@appspot.gserviceaccount.com`

> **Note:** The default service account has `Editor` role on the project — this is overly permissive. Always create a dedicated, least-privilege service account for production services.

### Key IAM Roles

| Role | Description |
|---|---|
| `roles/appengine.appAdmin` | Full control over App Engine resources |
| `roles/appengine.appCreator` | Create the App Engine application |
| `roles/appengine.deployer` | Deploy new versions (cannot modify traffic or delete) |
| `roles/appengine.serviceAdmin` | Manage versions and traffic (cannot deploy new code) |
| `roles/appengine.viewer` | Read-only access |
| `roles/iap.httpsResourceAccessor` | Access an IAP-protected resource |

### Dedicated Service Account per Service

```bash
# Create a service account for the API service
gcloud iam service-accounts create sa-api \
  --display-name="App Engine API Service"

# Grant only what it needs (e.g., Firestore read/write + GCS read)
gcloud projects add-iam-policy-binding my-project \
  --member=serviceAccount:sa-api@my-project.iam.gserviceaccount.com \
  --role=roles/datastore.user

gcloud projects add-iam-policy-binding my-project \
  --member=serviceAccount:sa-api@my-project.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer
```

```yaml
# app.yaml — assign the service account
service_account: sa-api@my-project.iam.gserviceaccount.com
```

### Secret Manager Integration

```bash
# Create a secret
echo -n "my-db-password" | gcloud secrets create db-password --data-file=-

# Grant the App Engine service account access
gcloud secrets add-iam-policy-binding db-password \
  --member=serviceAccount:sa-api@my-project.iam.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor
```

```python
# Access the secret at runtime — never hardcode credentials
from google.cloud import secretmanager

def get_secret(project_id: str, secret_id: str) -> str:
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("utf-8")

DB_PASSWORD = get_secret("my-project", "db-password")
```

### Security Headers (Add in your framework)

```python
# Add security headers in Flask
from flask import Flask
app = Flask(__name__)

@app.after_request
def set_security_headers(response):
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["Content-Security-Policy"] = "default-src 'self'"
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    return response
```

---

## 11. Task Queues & Cron Jobs

### Push Queues vs. Pull Queues

| Feature | Push Queue | Pull Queue |
|---|---|---|
| **Execution model** | App Engine delivers tasks to a URL handler | Workers explicitly lease and execute tasks |
| **Who runs the task** | App Engine runtime | Your worker code (leases tasks) |
| **Equivalent (modern)** | Cloud Tasks | Pub/Sub |
| **Rate control** | Yes (`rate`) | Yes (controlled by workers) |
| **Retry** | Automatic | Worker-managed |
| **Best for** | URL-based task handlers | Custom workers, cross-platform |

### Modern: Cloud Tasks (Push Queue Replacement)

```python
# Enqueue a task using Cloud Tasks (modern replacement)
from google.cloud import tasks_v2
from google.protobuf import timestamp_pb2
import json, datetime

def enqueue_task(project: str, location: str, queue: str,
                 handler_url: str, payload: dict) -> None:
    client = tasks_v2.CloudTasksClient()
    parent = client.queue_path(project, location, queue)

    task = {
        "app_engine_http_request": {
            "http_method": tasks_v2.HttpMethod.POST,
            "relative_uri": handler_url,
            "body": json.dumps(payload).encode(),
            "headers": {"Content-Type": "application/json"},
        }
    }

    client.create_task(request={"parent": parent, "task": task})
```

```python
# Task handler in your App Engine service
from flask import Flask, request

app = Flask(__name__)

@app.route("/tasks/process-item", methods=["POST"])
def process_item():
    # Cloud Tasks sends the task payload as the request body
    payload = request.get_json()
    item_id = payload["item_id"]
    # ... process item ...
    return "ok", 200   # 200 = success; 2xx marks task complete
```

### Legacy: `queue.yaml` Push Queue Handler

```python
# Legacy push queue task handler (still works in Gen 2)
from flask import Flask, request

app = Flask(__name__)

@app.route("/tasks/send-email", methods=["POST"])
def send_email_task():
    # Verify the request comes from App Engine Task Queue
    if not request.headers.get("X-AppEngine-QueueName"):
        return "Forbidden", 403

    user_id = request.form.get("user_id")
    # ... send email logic ...
    return "", 200
```

### `cron.yaml` — Cron Job Reference

```yaml
# cron.yaml — all schedules are in UTC unless timezone is specified
cron:
  - description: "Nightly cleanup"
    url: /cron/cleanup
    schedule: every night 00:00
    retry_parameters:
      job_retry_limit: 2
      min_backoff_seconds: 30

  - description: "Every 5 minutes health sync"
    url: /cron/health-sync
    schedule: every 5 minutes

  - description: "First Monday of the month"
    url: /cron/monthly-report
    schedule: 1 of month 09:00
    timezone: America/New_York
```

```python
# Cron job handler — verify it's from App Engine
from flask import Flask, request

app = Flask(__name__)

@app.route("/cron/cleanup", methods=["GET"])
def cleanup():
    # App Engine sets this header for all cron requests
    if not request.headers.get("X-Appengine-Cron"):
        return "Forbidden", 403
    # ... cleanup logic ...
    return "Done", 200
```

```bash
# Deploy cron config
gcloud app deploy cron.yaml

# List configured cron jobs
gcloud app describe

# Manually trigger a cron job (for testing)
curl -H "X-Appengine-Cron: true" https://my-project.appspot.com/cron/cleanup
```

---

## 12. Observability

### Built-in Request Logging

Every HTTP request is automatically logged to Cloud Logging with:
- Request method, URL, status code, latency
- Instance ID, version, service name
- User agent, referrer, IP

### Structured Logging

```python
# Emit structured JSON logs that Cloud Logging can parse and index
import json, sys, os

def log(severity: str, message: str, **kwargs) -> None:
    entry = {
        "severity": severity,            # DEBUG, INFO, WARNING, ERROR, CRITICAL
        "message": message,
        "service": os.environ.get("GAE_SERVICE", "default"),
        "version": os.environ.get("GAE_VERSION", "unknown"),
        **kwargs
    }
    print(json.dumps(entry), file=sys.stdout, flush=True)

# Usage
log("INFO", "User logged in", user_id="u-123", ip="203.0.113.5")
log("ERROR", "Payment failed", order_id="ord-456", error_code="DECLINED")
```

### Cloud Trace Auto-Instrumentation

```python
# Install: pip install google-cloud-trace opentelemetry-exporter-gcp-trace
from opentelemetry import trace
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(CloudTraceSpanExporter()))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

# Instrument a code section
with tracer.start_as_current_span("process-order") as span:
    span.set_attribute("order.id", order_id)
    span.set_attribute("order.amount", amount)
    result = process_order(order_id)
```

### Cloud Profiler

```python
# Continuous CPU/memory profiling with near-zero overhead
import googlecloudprofiler

def initialize_profiler():
    googlecloudprofiler.start(
        service=os.environ.get("GAE_SERVICE", "default"),
        service_version=os.environ.get("GAE_VERSION", "dev"),
        # Profiling types: CPU, WALL, HEAP, THREADS
        verbose=0,
    )
```

### Error Reporting

Exceptions are automatically captured if you use the Error Reporting library:

```python
# pip install google-cloud-error-reporting
from google.cloud import error_reporting

client = error_reporting.Client()

try:
    risky_operation()
except Exception:
    client.report_exception()   # Sends full traceback to Error Reporting
    raise
```

### Useful Cloud Logging Filters

```bash
# All logs from a specific App Engine service
resource.type="gae_app"
resource.labels.module_id="api"

# Only error logs from a specific version
resource.type="gae_app"
resource.labels.module_id="default"
resource.labels.version_id="v20240101"
severity>=ERROR

# Slow requests (over 2 seconds)
resource.type="gae_app"
httpRequest.latency>"2s"

# A specific request trace ID
resource.type="gae_app"
labels."appengine.googleapis.com/trace_id"="abc123def456"

# Cron job executions
resource.type="gae_app"
httpRequest.requestUrl:"/cron/"
protoPayload.requestId!=""
```

```bash
# Stream logs from CLI
gcloud app logs tail --service=default
gcloud app logs read --service=api --version=v20240101 --limit=100
```

---

## 13. `gcloud` CLI Quick Reference

### Application Commands

| Command | Description |
|---|---|
| `gcloud app create --region=REGION` | Create the App Engine app for the project (one-time) |
| `gcloud app describe` | Show app details (region, default hostname, cron) |
| `gcloud app open-console` | Open App Engine dashboard in Cloud Console |
| `gcloud app browse` | Open the app's default URL in the browser |

```bash
# Initialize App Engine for the first time
gcloud app create --project=my-project --region=us-central
```

### Deploy Commands

| Command | Description |
|---|---|
| `gcloud app deploy` | Deploy using `app.yaml` in the current directory |
| `gcloud app deploy --no-promote` | Deploy without routing traffic to the new version |
| `gcloud app deploy --version=NAME` | Deploy with a specific version identifier |
| `gcloud app deploy cron.yaml` | Deploy only the cron configuration |
| `gcloud app deploy dispatch.yaml` | Deploy only the dispatch routing rules |
| `gcloud app deploy queue.yaml` | Deploy only the task queue configuration |

### Version Commands

| Command | Description |
|---|---|
| `gcloud app versions list` | List all versions across all services |
| `gcloud app versions list --service=SVC` | List versions for a specific service |
| `gcloud app versions describe VER --service=SVC` | Describe a version |
| `gcloud app versions stop VER --service=SVC` | Stop a version (pause instances, stop billing) |
| `gcloud app versions start VER --service=SVC` | Start a stopped version |
| `gcloud app versions delete VER --service=SVC` | Delete a version permanently |

```bash
# Delete all stopped/non-serving versions older than 30 days
gcloud app versions list \
  --filter="traffic_split=0 AND create_time<$(date -d '30 days ago' +%Y-%m-%d)" \
  --format="value(id,service)" | \
  while read ver svc; do
    gcloud app versions delete "$ver" --service="$svc" --quiet
  done
```

### Traffic Commands

```bash
# Send 100% traffic to a specific version
gcloud app services set-traffic default --splits=v20240101=1

# Gradual migration to new version
gcloud app services set-traffic default \
  --splits=v20240115=1 --migrate

# Canary: 10% to new, 90% to stable
gcloud app services set-traffic default \
  --splits=stable=0.9,canary=0.1 --split-by=cookie
```

### Service Commands

```bash
# List all services
gcloud app services list

# Delete an entire service (all versions)
gcloud app services delete worker --quiet

# Update ingress setting
gcloud app services update default --ingress=internal
```

### Log Commands

```bash
# Stream logs in real time
gcloud app logs tail

# Read recent logs from a service
gcloud app logs read --service=api --limit=200

# Read logs filtered by severity
gcloud app logs read --service=default --level=error
```

### Instance Commands

```bash
# List running instances
gcloud app instances list

# SSH into a Flexible environment instance
gcloud app instances ssh INSTANCE_ID \
  --service=default \
  --version=v20240101

# Enable debug mode on a Flexible instance (allows SSH)
gcloud app instances enable-debug INSTANCE_ID \
  --service=default \
  --version=v20240101
```

---

## 14. Terraform Snippet

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = "us-central1"
}

variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

# ── App Engine Application (one per project) ───────────────
resource "google_app_engine_application" "app" {
  project     = var.project_id
  location_id = "us-central1"

  # Use Firestore in Datastore mode (default for new projects)
  database_type = "CLOUD_FIRESTORE"
}

# ── Serverless VPC Access Connector ───────────────────────
resource "google_vpc_access_connector" "connector" {
  name          = "app-engine-connector"
  region        = "us-central1"
  network       = "default"
  ip_cidr_range = "10.8.0.0/28"
  depends_on    = [google_app_engine_application.app]
}

# ── Service Account ────────────────────────────────────────
resource "google_service_account" "app_sa" {
  account_id   = "sa-app-engine"
  display_name = "App Engine Service Account"
}

resource "google_project_iam_member" "app_sa_firestore" {
  project = var.project_id
  role    = "roles/datastore.user"
  member  = "serviceAccount:${google_service_account.app_sa.email}"
}

resource "google_project_iam_member" "app_sa_secret" {
  project = var.project_id
  role    = "roles/secretmanager.secretAccessor"
  member  = "serviceAccount:${google_service_account.app_sa.email}"
}

# ── App Engine Standard Version ────────────────────────────
resource "google_app_engine_standard_app_version" "default_v1" {
  service    = "default"
  version_id = "v1"
  runtime    = "python312"

  entrypoint {
    shell = "gunicorn -b :$PORT main:app"
  }

  deployment {
    files {
      name       = "main.py"
      source_url = "https://storage.googleapis.com/${google_storage_bucket.app_bucket.name}/main.py"
    }
    files {
      name       = "requirements.txt"
      source_url = "https://storage.googleapis.com/${google_storage_bucket.app_bucket.name}/requirements.txt"
    }
  }

  env_variables = {
    ENV       = "production"
    LOG_LEVEL = "info"
    PROJECT   = var.project_id
  }

  automatic_scaling {
    min_idle_instances = 0
    max_idle_instances = 3
    standard_scheduler_settings {
      target_cpu_utilization        = 0.65
      target_throughput_utilization = 0.75
      min_instances                 = 1
      max_instances                 = 20
    }
  }

  vpc_access_connector {
    name = google_vpc_access_connector.connector.id
  }

  service_account = google_service_account.app_sa.email

  # Deploy but do not automatically route traffic
  noop_on_destroy       = true
  delete_service_on_destroy = false
}

# ── GCS Bucket for app source files ───────────────────────
resource "google_storage_bucket" "app_bucket" {
  name                        = "${var.project_id}-app-source"
  location                    = "US"
  uniform_bucket_level_access = true
  force_destroy               = true
}

# ── IAP: Protect the App Engine service ───────────────────
resource "google_iap_web_app_engine_service_iam_binding" "iap_binding" {
  project = var.project_id
  app_id  = google_app_engine_application.app.app_id
  service = google_app_engine_standard_app_version.default_v1.service
  role    = "roles/iap.httpsResourceAccessor"
  members = [
    "user:developer@mycompany.com",
    "group:engineering@mycompany.com",
  ]
}

# ── Cloud Scheduler: Trigger cron via HTTP ────────────────
resource "google_cloud_scheduler_job" "nightly_cleanup" {
  name             = "nightly-cleanup"
  description      = "Trigger the nightly cleanup task"
  schedule         = "0 2 * * *"   # 2 AM UTC daily
  time_zone        = "UTC"
  attempt_deadline = "320s"
  region           = "us-central1"

  app_engine_http_target {
    http_method  = "POST"
    relative_uri = "/tasks/cleanup"
    app_engine_routing {
      service = "default"
    }
  }

  retry_config {
    retry_count          = 3
    min_backoff_duration = "30s"
    max_backoff_duration = "3600s"
    max_doublings        = 4
  }

  depends_on = [google_app_engine_application.app]
}

# ── Outputs ────────────────────────────────────────────────
output "app_default_hostname" {
  value       = google_app_engine_application.app.default_hostname
  description = "The default hostname of the App Engine application"
}

output "service_account_email" {
  value = google_service_account.app_sa.email
}
```

```bash
terraform init
terraform plan  -var="project_id=my-project"
terraform apply -var="project_id=my-project"
```

> **Note:** The `google_app_engine_application` resource creates the App Engine app for the project. This is a **one-time, irreversible** action — the region cannot be changed after creation.

---

## 15. Migration & Modernization Paths

### Bundled Services → Standalone GCP Services

| Bundled Service | Modern Equivalent | Key Differences |
|---|---|---|
| **Datastore (ndb)** | Firestore (Datastore mode) | Same data model; use `google-cloud-datastore` or `google-cloud-firestore` |
| **Memcache** | Memorystore (Redis or Memcached) | Requires Serverless VPC Access connector |
| **Task Queues (push)** | Cloud Tasks | New client library; queues configured via API instead of `queue.yaml` |
| **Task Queues (pull)** | Pub/Sub | Different paradigm; subscriber model |
| **Blobstore** | Cloud Storage | Simple drop-in; use `google-cloud-storage` |
| **Mail API** | SendGrid / Postmark / SMTP relay | No direct GCP equivalent |
| **Images API** | Cloud Storage + Cloud CDN | Manual resize logic; or use Thumbor |
| **Search API** | Vertex AI Search / Firestore queries | Significant rework required |
| **Users API** | Firebase Authentication / IAP / Identity Platform | Use OAuth 2.0 / OIDC directly |
| **Channel API** | Firebase Realtime Database / Pub/Sub | Websocket-based replacement |

### App Engine Standard → Cloud Run Migration

```
App Engine Standard               Cloud Run
──────────────────               ─────────
app.yaml                   →     service.yaml / gcloud flags
runtime: python312         →     FROM python:3.12-slim (Dockerfile)
handlers:                  →     Application routing (Flask/FastAPI routes)
inbound_services: warmup   →     --min-instances=1 (or startup probe)
automatic_scaling          →     --min-instances / --max-instances
instance_class: F2         →     --memory=512Mi --cpu=1
env_variables              →     --set-env-vars / --set-secrets
vpc_access_connector       →     --network / --subnet (Direct VPC Egress)
cron.yaml                  →     Cloud Scheduler → Cloud Run HTTP trigger
queue.yaml                 →     Cloud Tasks → Cloud Run HTTP handler
dispatch.yaml              →     Cloud Load Balancer path rules
```

**Migration Steps:**

1. **Containerize**: Add a `Dockerfile` (multi-stage, non-root user, `PORT` env var).
2. **Remove bundled service dependencies**: Replace `ndb`, `memcache`, legacy task queue imports.
3. **Update Cloud SQL connection**: Switch from Unix socket (still works) or use Cloud SQL Auth Proxy sidecar.
4. **Migrate cron jobs**: Replace `cron.yaml` with Cloud Scheduler jobs targeting Cloud Run URLs.
5. **Migrate task queues**: Replace App Engine Task Queue client with Cloud Tasks client.
6. **Push image**: Build and push to Artifact Registry.
7. **Deploy to Cloud Run**: `gcloud run deploy`.
8. **Validate** with a traffic split before full cutover.

### Migration Decision Matrix

| Condition | Recommendation |
|---|---|
| Using Gen 1 runtime (Python 2.7, Java 8) | **Migrate** — EOL, security risk |
| Using bundled services heavily (ndb, Memcache) | **Migrate gradually** — replace service by service |
| Standard, no bundled services, happy with PaaS | **Stay on App Engine** — no migration needed |
| Need containers, custom runtimes, full control | **Migrate to Cloud Run** |
| Need WebSockets, bidirectional streaming | **Migrate to Cloud Run** (Gen 2 supports WS) |
| Need stateful workloads, sidecar containers | **Migrate to GKE** |
| Simple scheduled jobs | **Keep on App Engine** or move to Cloud Run + Scheduler |
| Long-running background processing | **Cloud Run Jobs** or **Pub/Sub + Cloud Run** |
| Multi-region with global load balancing | **Cloud Run + Global LB** or **GKE multi-region** |
| Very high request rates, extreme scale | **Cloud Run** (faster scale-out) |

> **Note:** There is no automated migration tool from App Engine to Cloud Run. The primary effort is containerization and replacing bundled services. Start with stateless services that have no bundled service dependencies — these are the easiest to migrate.

---

## 16. Troubleshooting Quick Reference

| Issue | Likely Cause | Fix |
|---|---|---|
| **Deployment fails: `app.yaml` validation error** | Invalid YAML syntax, unknown field, or wrong value type | Run `gcloud app deploy --verbosity=debug`; check for tabs vs. spaces in YAML |
| **New version not receiving traffic** | Deployed with `--no-promote` flag | Run `gcloud app services set-traffic default --splits=NEW_VERSION=1` |
| **503 errors on traffic spike** | Cold starts overwhelming the instance pool | Set `min_instances >= 1`, enable `warmup` in `inbound_services`, or set `min_idle_instances` |
| **Cold start latency too high** | Scale-to-zero with heavy initialization | Set `min_instances: 1`; move heavy init (DB connections, model loads) outside request handlers |
| **Memory limit exceeded (OOM)** | App using more RAM than instance class allows | Upgrade `instance_class` (e.g., F2 → F4); check for memory leaks with Cloud Profiler |
| **Cloud SQL connection refused** | Missing `cloud_sql_instances` in `beta_settings`, wrong socket path, or IAM missing `cloudsql.client` | Add `beta_settings.cloud_sql_instances`; grant `roles/cloudsql.client` to the service account |
| **Cron job not triggering** | `cron.yaml` not deployed, wrong URL, or URL returns non-2xx | Deploy `cron.yaml` separately; test URL manually with `X-Appengine-Cron: true` header |
| **Task queue tasks not executing** | Handler returning non-2xx, rate limits, or queue paused | Check task handler returns 200; verify queue is not paused in Cloud Console; check `X-AppEngine-QueueName` header validation |
| **VPC resource unreachable** | Missing VPC connector or wrong `egress_setting` | Create Serverless VPC Access connector; add `vpc_access_connector` to `app.yaml` |
| **`app.yaml`: handlers not working** | `handlers` section used in Flexible environment | Remove `handlers` — only valid in Standard; use application-level routing |
| **Static files not served** | Wrong path in `static_dir` or `static_files` handler | Check path is relative to `app.yaml`; verify `skip_files` doesn't exclude static assets |
| **IAP blocking legitimate requests** | Missing IAP role grant, or cookie not propagated | Grant `roles/iap.httpsResourceAccessor`; check JWT audience in app verification code |
| **`gcloud app deploy` hangs** | Large number of files being uploaded | Add comprehensive `skip_files` rules to exclude `node_modules`, `venv`, `.git`, build artifacts |
| **Flexible env: unhealthy instance cycling** | Health check path returning non-200, or startup too slow | Fix `/healthz` endpoint; increase `app_start_timeout_sec` in `readiness_check` |
| **Secret not accessible at runtime** | Service account lacks `secretmanager.secretAccessor` | Run `gcloud secrets add-iam-policy-binding`; verify secret name matches exactly |
| **Region mismatch error (App Engine + other services)** | App Engine region locked; other resource in different region | App Engine region is immutable; create other resources in the same region |
| **`ERROR_UPLOAD_FAILED`: source too large** | Deployment bundle exceeds 500 MB limit | Exclude large binaries/assets via `skip_files`; store large files in GCS |

---

*Last updated: 2025 | Covers App Engine GA features as of this date.*
*Refer to [cloud.google.com/appengine/docs](https://cloud.google.com/appengine/docs) for the latest information.*
