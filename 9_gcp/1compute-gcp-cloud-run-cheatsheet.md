# GCP Cloud Run Cheatsheet

> **TL;DR:** Cloud Run is GCP's fully managed serverless platform for running stateless containers — auto-scaling to zero, paying only for what you use, with no infrastructure to manage.

---

## Table of Contents

1. [Overview & Core Concepts](#1-overview--core-concepts)
2. [Architecture & Execution Model](#2-architecture--execution-model)
3. [Container Requirements & Best Practices](#3-container-requirements--best-practices)
4. [Deploying Services](#4-deploying-services)
5. [Cloud Run Jobs](#5-cloud-run-jobs)
6. [Environment Configuration](#6-environment-configuration)
7. [Networking](#7-networking)
8. [IAM & Security](#8-iam--security)
9. [Observability](#9-observability)
10. [CI/CD Integration](#10-cicd-integration)
11. [gcloud CLI Quick Reference](#11-gcloud-cli-quick-reference)
12. [Terraform Snippet](#12-terraform-snippet)
13. [Common Patterns & Use Cases](#13-common-patterns--use-cases)
14. [Cost Model & Optimization](#14-cost-model--optimization)
15. [Troubleshooting Quick Reference](#15-troubleshooting-quick-reference)

---

## 1. Overview & Core Concepts

Cloud Run is a **fully managed serverless compute platform** that runs stateless containers on Google's infrastructure. It abstracts away all server management — you bring a container, Cloud Run handles scaling, networking, TLS, and availability.

### Key Terminology

| Term | Description |
|---|---|
| **Service** | A long-running container that responds to HTTP/gRPC requests, auto-scales based on traffic |
| **Job** | A container that runs to completion — for batch tasks, migrations, and pipelines |
| **Revision** | An immutable snapshot of a service's config + container image; created on every deploy |
| **Container Instance** | A running copy of your container; Cloud Run scales these up/down automatically |
| **Concurrency** | Number of simultaneous requests a single container instance can handle |
| **Cold Start** | Latency when Cloud Run must spin up a new instance to handle a request |
| **Scale to Zero** | Instances are terminated when there is no traffic, eliminating idle compute costs |
| **Min Instances** | Instances kept warm at all times to reduce cold starts (billed even when idle) |
| **Max Instances** | Upper cap on the number of container instances (prevents runaway scaling and cost) |
| **Traffic Splitting** | Routing a percentage of requests to different revisions (used for canary/blue-green) |
| **Ingress** | Controls which traffic sources can reach the service (public, internal, LB-only) |
| **HTTPS Endpoint** | Every Cloud Run service automatically gets a `*.run.app` TLS-terminated URL |

### Cloud Run vs Other GCP Compute Options

| Feature | Cloud Run | Cloud Functions | App Engine | GKE | GCE |
|---|---|---|---|---|---|
| **Unit of deploy** | Container | Function (code) | App (code) | Pod (container) | VM |
| **Scaling** | Auto (0 → N) | Auto (0 → N) | Auto (0 → N) | Manual / HPA | Manual / MIG |
| **Scale to zero** | ✅ | ✅ | ✅ (Standard) | ❌ | ❌ |
| **Concurrency** | Up to 1000/instance | 1/instance | Configurable | Configurable | N/A |
| **Max timeout** | 60 min (services) | 60 min (2nd gen) | 10 min | Unlimited | Unlimited |
| **Custom runtimes** | Any container | Limited | Yes (Flex) | Any container | Any OS |
| **Cold starts** | Yes (mitigable) | Yes | Yes | Rare | No |
| **Infra management** | None | None | None | Cluster management | Full |
| **Best for** | APIs, microservices | Event handlers | Web apps | Complex orchestration | Full control |

> **Note:** Choose Cloud Run when you have containerized workloads that need fast, scalable HTTP serving without managing infrastructure. Choose GKE when you need stateful workloads, multi-container pods, or advanced orchestration.

---

## 2. Architecture & Execution Model

### Request-Driven vs. Job-Based Execution

```
REQUEST-DRIVEN (Cloud Run Service)           JOB-BASED (Cloud Run Job)
─────────────────────────────────           ──────────────────────────
  HTTP Request                                Trigger (manual/scheduled)
       │                                             │
       ▼                                             ▼
  Cloud Run Service ──► Container           Cloud Run Job ──► N Tasks
  (auto-scales, always listening)           (runs to completion, exits 0)
       │                                             │
       ▼                                             ▼
  HTTP Response                             Success / Failure
```

### Container Lifecycle

```
Request arrives
     │
     ├─► Instance available? ──YES──► Route request ──► Send response
     │
     └─► NO: Cold start
              │
              ├── Pull image (if not cached)
              ├── Start container process
              ├── Wait for port to listen
              └── Route request ──► Send response

No traffic for ~15 min ──► Instance terminated (scale to zero)
```

### Concurrency Model

Cloud Run supports **per-instance concurrency** — a single instance can handle many requests at once:

```
Default concurrency: 80 requests/instance

Instance 1 ──► [req1, req2, ..., req80]   (80 simultaneous)
Instance 2 ──► [req81, ..., req160]        (auto-scaled)
```

> **Note:** Set concurrency to `1` for CPU-bound workloads (e.g., ML inference, FFmpeg transcoding) to prevent resource contention. Use higher concurrency for I/O-bound workloads (e.g., API proxies, database queries).

### CPU Allocation Modes

| Mode | CPU available | Billed | Best for |
|---|---|---|---|
| **Request-only** (default) | Only during request processing | Per request (CPU × time) | Sporadic, bursty traffic |
| **Always-on** | Throughout instance lifetime | Per instance uptime | Background threads, WebSockets, streaming |

```bash
# Set CPU to always-on
gcloud run deploy my-service \
  --cpu-throttling=false \
  --region=us-central1
```

### Timeout Behavior

| Resource | Default | Maximum |
|---|---|---|
| Cloud Run Service | 300s (5 min) | 3600s (60 min) |
| Cloud Run Job (per task) | 600s (10 min) | 86400s (24 hr) |
| Startup probe | 240s | 240s |

```bash
# Set service request timeout to 30 minutes
gcloud run deploy my-service \
  --timeout=1800 \
  --region=us-central1
```

---

## 3. Container Requirements & Best Practices

### Compatibility Checklist

| Requirement | Details |
|---|---|
| **Listen on `PORT` env var** | Cloud Run injects `PORT` (default `8080`); your app MUST bind to it |
| **Stateless** | No local file system state between requests; use GCS/Cloud SQL/Firestore |
| **Fast startup** | Target < 4 seconds; defer non-critical init to after port bind |
| **Handle `SIGTERM`** | Cloud Run sends `SIGTERM` before termination; drain in-flight requests |
| **Exit 0 on success** | For jobs, exit code 0 = success; non-zero = failure/retry |
| **Single process** | Prefer single foreground process; avoid background daemons |

### Minimal HTTP Server Example

```python
# main.py — a minimal Cloud Run-compatible Flask app
import os
import signal
import sys
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello from Cloud Run!", 200

@app.route("/healthz")
def health():
    return "ok", 200

def handle_sigterm(*args):
    """Graceful shutdown on SIGTERM from Cloud Run."""
    print("SIGTERM received — shutting down gracefully")
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_sigterm)

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    app.run(host="0.0.0.0", port=port)
```

### Multi-Stage Dockerfile

```dockerfile
# ── Stage 1: Build ─────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ── Stage 2: Runtime ───────────────────────────────────────
FROM python:3.12-slim AS runtime

# Non-root user for security
RUN useradd --no-create-home --shell /bin/false appuser

WORKDIR /app
COPY --from=builder /install /usr/local
COPY . .

# Cloud Run injects PORT; default to 8080
ENV PORT=8080

USER appuser

# Use exec form to receive signals properly
CMD ["python", "main.py"]
```

### Recommended Base Images

| Language | Recommended Base | Notes |
|---|---|---|
| Python | `python:3.12-slim` | Slim reduces attack surface and image size |
| Node.js | `node:20-alpine` | Alpine for smallest footprint |
| Go | `gcr.io/distroless/static` | Statically compiled binary, no shell |
| Java | `eclipse-temurin:21-jre-alpine` | JRE-only, not full JDK |
| Generic | `gcr.io/distroless/base` | No shell, minimal OS, Google-maintained |

### Image Tagging Strategy

```bash
# BAD: using :latest makes rollbacks impossible
docker push gcr.io/my-project/my-app:latest

# GOOD: semantic version + git SHA for traceability
IMAGE_TAG="v1.4.2-$(git rev-parse --short HEAD)"
docker build -t gcr.io/my-project/my-app:${IMAGE_TAG} .
docker push gcr.io/my-project/my-app:${IMAGE_TAG}
```

> **Note:** Never use `:latest` in production. Cloud Run caches images by digest — using a mutable tag means you can't guarantee which image version is running.

---

## 4. Deploying Services

### Deploy from a Container Image

```bash
# Basic deployment
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:v1.0.0 \
  --region=us-central1 \
  --platform=managed \
  --allow-unauthenticated    # Public service; omit for private
```

### Deploy from Source (Buildpacks — no Dockerfile needed)

```bash
# Cloud Run builds the image automatically using Google Cloud Buildpacks
gcloud run deploy my-service \
  --source=. \
  --region=us-central1 \
  --allow-unauthenticated
```

> **Note:** Source-based deploys require the Cloud Build API to be enabled and write access to Artifact Registry. The image is stored in `{REGION}-docker.pkg.dev/{PROJECT}/cloud-run-source-deploy/`.

### Traffic Splitting Between Revisions

```bash
# Send 90% to stable, 10% to canary revision
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-revisions=my-service-00010-abc=90,my-service-00011-xyz=10

# Promote canary to 100% after validation
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-latest
```

### Rollback to a Previous Revision

```bash
# List revisions
gcloud run revisions list --service=my-service --region=us-central1

# Roll back by sending 100% traffic to a specific revision
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-revisions=my-service-00009-prev=100
```

### Tag a Revision for Testing (Stable URL per revision)

```bash
# Assign a named URL tag to a revision
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --set-tags=canary=my-service-00011-xyz

# Access tagged revision at:
# https://canary---my-service-<hash>-uc.a.run.app
```

### Deploying with All Key Flags

```bash
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/repo/app:v2.1.0 \
  --region=us-central1 \
  --service-account=sa-cloudrun@my-project.iam.gserviceaccount.com \
  --memory=512Mi \
  --cpu=1 \
  --concurrency=80 \
  --min-instances=1 \
  --max-instances=20 \
  --timeout=300 \
  --set-env-vars=ENV=production,LOG_LEVEL=info \
  --set-secrets=DB_PASSWORD=db-password:latest \
  --ingress=internal-and-cloud-load-balancing \
  --no-allow-unauthenticated
```

---

## 5. Cloud Run Jobs

### Services vs. Jobs

| Attribute | Cloud Run Service | Cloud Run Job |
|---|---|---|
| **Trigger** | HTTP/gRPC request | Manual, Scheduler, Eventarc |
| **Runs until** | Request completes | Container exits with code 0 |
| **Scaling unit** | Per request | Per task (parallelism) |
| **Retries** | Per request | Per task (configurable) |
| **Use cases** | APIs, webhooks, streaming | Batch, ETL, DB migrations |

### Create and Run a Job

```bash
# Create a job
gcloud run jobs create my-etl-job \
  --image=us-central1-docker.pkg.dev/my-project/repo/etl:v1.0.0 \
  --region=us-central1 \
  --service-account=sa-jobs@my-project.iam.gserviceaccount.com \
  --memory=2Gi \
  --cpu=2 \
  --task-timeout=3600 \
  --max-retries=3

# Execute the job (fire and forget)
gcloud run jobs execute my-etl-job --region=us-central1

# Execute and wait for completion
gcloud run jobs execute my-etl-job \
  --region=us-central1 \
  --wait
```

### Parallelism & Task Indexing

```bash
# Run 10 parallel tasks (e.g., process 10 data shards simultaneously)
gcloud run jobs create my-parallel-job \
  --image=us-central1-docker.pkg.dev/my-project/repo/worker:v1 \
  --region=us-central1 \
  --tasks=10 \
  --parallelism=5    # Max 5 tasks running at once
```

Inside the container, each task receives its identity via environment variables:

```python
import os

# Which task shard am I? (0-indexed)
task_index = int(os.environ.get("CLOUD_RUN_TASK_INDEX", 0))
task_count = int(os.environ.get("CLOUD_RUN_TASK_COUNT", 1))

# Process only this task's slice of data
items = get_all_items()
shard = items[task_index::task_count]   # distribute items across tasks
process(shard)
```

### Schedule a Job with Cloud Scheduler

```bash
# Create a scheduler that runs the job every day at 2 AM UTC
gcloud scheduler jobs create http daily-etl \
  --schedule="0 2 * * *" \
  --uri="https://us-central1-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/my-project/jobs/my-etl-job:run" \
  --http-method=POST \
  --oauth-service-account-email=sa-scheduler@my-project.iam.gserviceaccount.com \
  --location=us-central1
```

---

## 6. Environment Configuration

### Environment Variables

```bash
# Set at deploy time
gcloud run deploy my-service \
  --set-env-vars=ENV=prod,PORT=8080,LOG_LEVEL=info \
  --region=us-central1

# Update a single variable without full redeploy
gcloud run services update my-service \
  --update-env-vars=LOG_LEVEL=debug \
  --region=us-central1
```

### Secrets via Secret Manager

```bash
# Create a secret
echo -n "super-secret-password" | \
  gcloud secrets create db-password --data-file=- --replication-policy=automatic

# Grant Cloud Run service account access
gcloud secrets add-iam-policy-binding db-password \
  --member=serviceAccount:sa-cloudrun@my-project.iam.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor

# Mount secret as an environment variable
gcloud run deploy my-service \
  --set-secrets=DB_PASSWORD=db-password:latest \
  --region=us-central1

# Mount secret as a file (for certificates, JSON keys, etc.)
gcloud run deploy my-service \
  --set-secrets=/secrets/db.json=db-config:latest \
  --region=us-central1
```

### Volume Mounts (In-Memory & Cloud Storage)

```bash
# Mount an in-memory volume (tmpfs) — ephemeral, fast, not persisted
gcloud run deploy my-service \
  --add-volume=name=tmpvol,type=in-memory,size-limit=256Mi \
  --add-volume-mount=volume=tmpvol,mount-path=/tmp/scratch \
  --region=us-central1

# Mount a GCS bucket as a volume (FUSE mount — read/write)
gcloud run deploy my-service \
  --add-volume=name=gcs-data,type=cloud-storage,bucket=my-bucket \
  --add-volume-mount=volume=gcs-data,mount-path=/data \
  --region=us-central1
```

### Compute Resources

```bash
gcloud run deploy my-service \
  --memory=1Gi \          # Options: 128Mi – 32Gi
  --cpu=2 \               # Options: 0.08 – 8 vCPUs (must be >= 4 for memory > 8Gi)
  --cpu-throttling \      # Throttle CPU when not handling requests (default)
  --region=us-central1
```

### Scaling Configuration

```bash
gcloud run deploy my-service \
  --min-instances=2 \     # Keep 2 instances warm (reduces cold starts)
  --max-instances=100 \   # Hard cap (prevents unbounded scaling)
  --concurrency=50 \      # Max simultaneous requests per instance
  --region=us-central1
```

### Health Probes

```yaml
# service.yaml — health probe configuration
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-service
spec:
  template:
    spec:
      containers:
        - image: us-central1-docker.pkg.dev/my-project/repo/app:v1
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 30
```

```bash
# Deploy from YAML
gcloud run services replace service.yaml --region=us-central1
```

> **Note:** Startup probes are essential for services with slow initialization (e.g., loading ML models). Without them, Cloud Run may send traffic before your app is ready, causing 503s.

---

## 7. Networking

### Ingress Controls

| Setting | Who can reach the service |
|---|---|
| `all` | Any client on the internet (public) |
| `internal` | Only VPC internal traffic and Cloud Run in same project |
| `internal-and-cloud-load-balancing` | VPC + Global Load Balancer (recommended for public APIs behind Cloud Armor) |

```bash
gcloud run deploy my-service \
  --ingress=internal-and-cloud-load-balancing \
  --region=us-central1
```

### VPC Egress (Outbound Traffic to Private Resources)

Two options for connecting Cloud Run to a VPC (e.g., Cloud SQL, Memorystore, internal services):

#### Option A: Direct VPC Egress (Recommended — lower latency, no extra resource)

```bash
gcloud run deploy my-service \
  --network=my-vpc \
  --subnet=my-subnet \
  --vpc-egress=private-ranges-only \   # Only private IPs go through VPC
  --region=us-central1
```

#### Option B: Serverless VPC Access Connector (Legacy)

```bash
# Create the connector
gcloud compute networks vpc-access connectors create my-connector \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.8.0.0/28

# Attach to service
gcloud run deploy my-service \
  --vpc-connector=my-connector \
  --vpc-egress=private-ranges-only \
  --region=us-central1
```

| | Direct VPC Egress | VPC Access Connector |
|---|---|---|
| **Additional resource** | None | Connector VMs (managed) |
| **Latency** | Lower | Slightly higher |
| **Max throughput** | Higher | Connector-limited |
| **Recommended** | ✅ Yes (new deployments) | Legacy only |

### Custom Domains

```bash
# Map a custom domain (requires domain verification in Search Console)
gcloud run domain-mappings create \
  --service=my-service \
  --domain=api.mycompany.com \
  --region=us-central1

# Get DNS records to configure at your registrar
gcloud run domain-mappings describe \
  --domain=api.mycompany.com \
  --region=us-central1
```

### Global Load Balancer + Cloud Armor

For WAF, DDoS protection, CDN, or multi-region routing, route traffic through a Global External Load Balancer:

```bash
# 1. Create a serverless NEG pointing to Cloud Run
gcloud compute network-endpoint-groups create my-run-neg \
  --region=us-central1 \
  --network-endpoint-type=serverless \
  --cloud-run-service=my-service

# 2. Create backend service
gcloud compute backend-services create my-backend \
  --global \
  --load-balancing-scheme=EXTERNAL_MANAGED

gcloud compute backend-services add-backend my-backend \
  --global \
  --network-endpoint-group=my-run-neg \
  --network-endpoint-group-region=us-central1

# 3. Attach Cloud Armor policy
gcloud compute backend-services update my-backend \
  --global \
  --security-policy=my-armor-policy
```

---

## 8. IAM & Security

### Making a Service Public or Private

```bash
# Allow anyone to invoke (public)
gcloud run services add-iam-policy-binding my-service \
  --member=allUsers \
  --role=roles/run.invoker \
  --region=us-central1

# Restrict to a specific service account (private)
gcloud run services add-iam-policy-binding my-service \
  --member=serviceAccount:caller-sa@my-project.iam.gserviceaccount.com \
  --role=roles/run.invoker \
  --region=us-central1
```

### Service-to-Service Authentication

A calling service must attach an **OIDC identity token** in the `Authorization` header:

```python
import google.auth.transport.requests
import google.oauth2.id_token
import requests

def call_private_service(url: str) -> dict:
    """Call a private Cloud Run service using the current identity."""
    auth_req = google.auth.transport.requests.Request()
    token = google.oauth2.id_token.fetch_id_token(auth_req, url)
    response = requests.get(
        url,
        headers={"Authorization": f"Bearer {token}"}
    )
    response.raise_for_status()
    return response.json()
```

> **Note:** Install `google-auth` and `requests` libraries. The calling Cloud Run service must have `roles/run.invoker` on the target service.

### End-User Authentication (Firebase / Identity Platform)

```python
from google.oauth2 import id_token as google_id_token
from google.auth.transport import requests as google_requests

def verify_firebase_token(bearer_token: str, expected_audience: str) -> dict:
    """Verify a Firebase ID token passed by a client."""
    token = bearer_token.replace("Bearer ", "")
    request = google_requests.Request()
    claims = google_id_token.verify_firebase_token(token, request, expected_audience)
    return claims   # Contains uid, email, etc.
```

### Key IAM Roles

| Role | Description |
|---|---|
| `roles/run.admin` | Full control over all Cloud Run resources |
| `roles/run.developer` | Deploy and manage services; cannot change IAM |
| `roles/run.invoker` | Call (invoke) a Cloud Run service endpoint |
| `roles/run.viewer` | Read-only view of services and revisions |

### Workload Identity Federation

Allows external identities (GitHub Actions, AWS, etc.) to access GCP without service account keys:

```bash
# Create a Workload Identity Pool for GitHub Actions
gcloud iam workload-identity-pools create github-pool \
  --location=global \
  --display-name="GitHub Actions Pool"

gcloud iam workload-identity-pools providers create-oidc github-provider \
  --workload-identity-pool=github-pool \
  --location=global \
  --issuer-uri=https://token.actions.githubusercontent.com \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository"

# Bind GitHub repo to service account
gcloud iam service-accounts add-iam-policy-binding deploy-sa@my-project.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUM/locations/global/workloadIdentityPools/github-pool/attribute.repository/my-org/my-repo"
```

### Binary Authorization

Enforces that only cryptographically verified container images can be deployed:

```bash
# Enable Binary Authorization on the project
gcloud services enable binaryauthorization.googleapis.com

# Deploy with Binary Authorization enforced
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/repo/app:v1.0.0 \
  --binary-authorization=default \
  --region=us-central1
```

### CMEK (Customer-Managed Encryption Keys)

```bash
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/repo/app:v1 \
  --key=projects/my-project/locations/us-central1/keyRings/my-ring/cryptoKeys/my-key \
  --region=us-central1
```

---

## 9. Observability

### Built-in Request Logging

Every HTTP request is automatically logged to Cloud Logging:

```
httpRequest.requestMethod: POST
httpRequest.requestUrl: /api/v1/users
httpRequest.status: 200
httpRequest.latency: 0.045s
httpRequest.responseSize: 1234
labels.instanceId: 00bf4bf02d...
```

### Structured Logging Best Practices

Cloud Run integrates with Cloud Logging when logs are written to stdout/stderr in **JSON format**:

```python
import json
import sys
import os

def log(severity: str, message: str, **kwargs):
    """Emit a structured log entry Cloud Logging can parse."""
    entry = {
        "severity": severity,          # DEBUG, INFO, WARNING, ERROR, CRITICAL
        "message": message,
        "component": "my-service",
        "traceId": os.environ.get("TRACE_ID", ""),
        **kwargs
    }
    print(json.dumps(entry), file=sys.stdout, flush=True)

# Usage
log("INFO", "Request processed", user_id="u123", duration_ms=42)
log("ERROR", "Database connection failed", error_code="CONN_TIMEOUT")
```

### Correlating Logs with Traces

```python
from flask import Flask, request
import os, json

app = Flask(__name__)

@app.before_request
def set_trace_header():
    """Extract Cloud Trace header to correlate logs with traces."""
    trace = request.headers.get("X-Cloud-Trace-Context", "")
    if trace:
        trace_id = trace.split("/")[0]
        project = os.environ.get("GOOGLE_CLOUD_PROJECT")
        # Attach to structured log entry for automatic trace correlation
        g.trace = f"projects/{project}/traces/{trace_id}"
```

### Useful Cloud Logging Filters

```bash
# All logs from a specific Cloud Run service
resource.type="cloud_run_revision"
resource.labels.service_name="my-service"

# Only errors
resource.type="cloud_run_revision"
resource.labels.service_name="my-service"
severity>=ERROR

# Cold starts (new instance initializations)
resource.type="cloud_run_revision"
labels."gce_instance_id"!="" AND textPayload:"Starting HTTP server"

# Requests taking over 5 seconds
resource.type="cloud_run_revision"
httpRequest.latency > "5s"

# Specific revision only
resource.labels.revision_name="my-service-00012-abc"
```

```bash
# Query logs from CLI
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="my-service" AND severity>=ERROR' \
  --limit=50 \
  --project=my-project
```

### OpenTelemetry Custom Metrics

```python
from opentelemetry import metrics
from opentelemetry.exporter.cloud_monitoring import CloudMonitoringMetricsExporter
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader

# Initialize the metrics pipeline
exporter = CloudMonitoringMetricsExporter()
reader = PeriodicExportingMetricReader(exporter, export_interval_millis=60000)
provider = MeterProvider(metric_readers=[reader])
metrics.set_meter_provider(provider)

meter = metrics.get_meter("my-service")
request_counter = meter.create_counter(
    "processed_requests",
    description="Total requests processed"
)
latency_histogram = meter.create_histogram(
    "request_latency_ms",
    description="Request processing latency in milliseconds"
)

# In your request handler
request_counter.add(1, {"endpoint": "/api/v1/users", "status": "200"})
latency_histogram.record(42, {"endpoint": "/api/v1/users"})
```

---

## 10. CI/CD Integration

### Cloud Build + Artifact Registry Pipeline

```yaml
# cloudbuild.yaml
steps:
  # Step 1: Run tests
  - name: python:3.12-slim
    entrypoint: pip
    args: [install, -r, requirements-test.txt]
  - name: python:3.12-slim
    entrypoint: pytest
    args: [tests/]

  # Step 2: Build container image
  - name: gcr.io/cloud-builders/docker
    args:
      - build
      - -t
      - us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA
      - -t
      - us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:latest
      - .

  # Step 3: Push to Artifact Registry
  - name: gcr.io/cloud-builders/docker
    args:
      - push
      - --all-tags
      - us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app

  # Step 4: Deploy to Cloud Run
  - name: gcr.io/google.com/cloudsdktool/cloud-sdk
    entrypoint: gcloud
    args:
      - run
      - deploy
      - my-service
      - --image=us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA
      - --region=us-central1
      - --platform=managed
      - --no-traffic        # Deploy without sending traffic (canary first)

  # Step 5: Gradual traffic shift (optional)
  - name: gcr.io/google.com/cloudsdktool/cloud-sdk
    entrypoint: gcloud
    args:
      - run
      - services
      - update-traffic
      - my-service
      - --to-latest
      - --region=us-central1

images:
  - us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA

options:
  logging: CLOUD_LOGGING_ONLY
```

```bash
# Create a Cloud Build trigger on push to main branch
gcloud builds triggers create github \
  --repo-name=my-repo \
  --repo-owner=my-org \
  --branch-pattern=^main$ \
  --build-config=cloudbuild.yaml \
  --region=us-central1
```

### Artifact Registry Setup

```bash
# Create a Docker repository
gcloud artifacts repositories create my-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Production container images"

# Configure Docker to authenticate with Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev

# Grant Cloud Build access to push images
gcloud artifacts repositories add-iam-policy-binding my-repo \
  --location=us-central1 \
  --member=serviceAccount:$(gcloud projects describe my-project --format='value(projectNumber)')@cloudbuild.gserviceaccount.com \
  --role=roles/artifactregistry.writer
```

### GitHub Actions Integration

```yaml
# .github/workflows/deploy.yml
name: Deploy to Cloud Run

on:
  push:
    branches: [main]

env:
  PROJECT_ID: my-project
  REGION: us-central1
  SERVICE: my-service
  IMAGE: us-central1-docker.pkg.dev/my-project/my-repo/my-app

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write    # Required for Workload Identity Federation

    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP (Keyless via WIF)
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/PROJECT_NUM/locations/global/workloadIdentityPools/github-pool/providers/github-provider
          service_account: deploy-sa@my-project.iam.gserviceaccount.com

      - name: Configure Docker
        run: gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev

      - name: Build & Push Image
        run: |
          docker build -t ${{ env.IMAGE }}:${{ github.sha }} .
          docker push ${{ env.IMAGE }}:${{ github.sha }}

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy ${{ env.SERVICE }} \
            --image=${{ env.IMAGE }}:${{ github.sha }} \
            --region=${{ env.REGION }} \
            --platform=managed
```

---

## 11. `gcloud` CLI Quick Reference

### Service Commands

| Command | Description |
|---|---|
| `gcloud run services list --region=REGION` | List all services in a region |
| `gcloud run services describe SERVICE --region=REGION` | Show full service config |
| `gcloud run services delete SERVICE --region=REGION` | Delete a service and all revisions |
| `gcloud run services update SERVICE --region=REGION [FLAGS]` | Update service config without redeploying image |
| `gcloud run services update-traffic SERVICE --to-latest` | Send 100% traffic to the latest revision |

```bash
# List all services across all regions
gcloud run services list --platform=managed

# Get the service URL
gcloud run services describe my-service \
  --region=us-central1 \
  --format='value(status.url)'
```

### Revision Commands

| Command | Description |
|---|---|
| `gcloud run revisions list --service=SERVICE` | List all revisions for a service |
| `gcloud run revisions describe REVISION` | Show config of a specific revision |
| `gcloud run revisions delete REVISION` | Delete a specific revision (must have 0% traffic) |

```bash
# Delete all revisions except the latest 3 (cleanup script)
gcloud run revisions list \
  --service=my-service \
  --region=us-central1 \
  --format='value(name)' \
  --sort-by='~creationTimestamp' | tail -n +4 | \
  xargs -I {} gcloud run revisions delete {} --region=us-central1 --quiet
```

### Traffic Commands

```bash
# Split traffic: 80% stable, 20% canary
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-revisions=my-service-v1=80,my-service-v2=20

# Tag a revision for direct URL access (without affecting traffic)
gcloud run services update-traffic my-service \
  --set-tags=staging=my-service-00015-xyz \
  --region=us-central1
```

### IAM Commands

```bash
# Grant invoker to a service account
gcloud run services add-iam-policy-binding my-service \
  --member=serviceAccount:sa@my-project.iam.gserviceaccount.com \
  --role=roles/run.invoker \
  --region=us-central1

# View current IAM policy
gcloud run services get-iam-policy my-service --region=us-central1
```

### Job Commands

```bash
# List all jobs
gcloud run jobs list --region=us-central1

# Describe a job
gcloud run jobs describe my-job --region=us-central1

# Update job image without re-creating
gcloud run jobs update my-job \
  --image=us-central1-docker.pkg.dev/my-project/repo/worker:v2 \
  --region=us-central1

# List executions of a job
gcloud run jobs executions list --job=my-job --region=us-central1

# Cancel a running execution
gcloud run jobs executions cancel EXECUTION_NAME --region=us-central1
```

### Useful Flags Reference

| Flag | Description |
|---|---|
| `--region` | GCP region (e.g., `us-central1`) |
| `--platform=managed` | Use fully managed Cloud Run (default) |
| `--allow-unauthenticated` | Make service publicly accessible |
| `--no-allow-unauthenticated` | Require authentication (private) |
| `--set-env-vars=K=V,K2=V2` | Set environment variables |
| `--set-secrets=ENV=secret:version` | Inject a Secret Manager secret |
| `--service-account=SA_EMAIL` | Assign a service account to the revision |
| `--cpu-throttling / --no-cpu-throttling` | Toggle CPU allocation mode |
| `--execution-environment=gen2` | Use 2nd gen execution environment (Linux kernel 5, faster) |
| `--no-traffic` | Deploy revision but send it 0% traffic |

---

## 12. Terraform Snippet

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

variable "image" {
  description = "Container image URI"
  type        = string
  default     = "us-central1-docker.pkg.dev/my-project/my-repo/my-app:v1.0.0"
}

# ── Secret Manager secret (pre-created) ────────────────────
data "google_secret_manager_secret_version" "db_password" {
  secret = "db-password"
}

# ── Service Account ────────────────────────────────────────
resource "google_service_account" "cloudrun_sa" {
  account_id   = "sa-cloudrun"
  display_name = "Cloud Run Service Account"
}

resource "google_secret_manager_secret_iam_member" "cloudrun_secret_access" {
  secret_id = "db-password"
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.cloudrun_sa.email}"
}

# ── Cloud Run Service ──────────────────────────────────────
resource "google_cloud_run_v2_service" "app" {
  name     = "my-terraform-service"
  location = "us-central1"
  ingress  = "INGRESS_TRAFFIC_INTERNAL_LOAD_BALANCER"

  template {
    service_account = google_service_account.cloudrun_sa.email

    scaling {
      min_instance_count = 1
      max_instance_count = 20
    }

    containers {
      image = var.image

      resources {
        limits = {
          cpu    = "1"
          memory = "512Mi"
        }
        cpu_idle = true   # Throttle CPU when not handling requests
      }

      # Environment variables (plain)
      env {
        name  = "ENV"
        value = "production"
      }
      env {
        name  = "LOG_LEVEL"
        value = "info"
      }

      # Secret injected as environment variable
      env {
        name = "DB_PASSWORD"
        value_source {
          secret_key_ref {
            secret  = "db-password"
            version = "latest"
          }
        }
      }

      # Health probe
      startup_probe {
        http_get {
          path = "/healthz"
          port = 8080
        }
        initial_delay_seconds = 5
        period_seconds        = 10
        failure_threshold     = 3
      }

      liveness_probe {
        http_get {
          path = "/healthz"
          port = 8080
        }
        period_seconds = 30
      }
    }

    # Direct VPC egress for accessing private resources
    vpc_access {
      network_interfaces {
        network    = "default"
        subnetwork = "default"
      }
      egress = "PRIVATE_RANGES_ONLY"
    }

    timeout = "300s"
    max_instance_request_concurrency = 80
  }

  traffic {
    type    = "TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST"
    percent = 100
  }

  depends_on = [google_secret_manager_secret_iam_member.cloudrun_secret_access]
}

# ── IAM: Allow specific SA to invoke the service ──────────
resource "google_cloud_run_v2_service_iam_member" "invoker" {
  name     = google_cloud_run_v2_service.app.name
  location = google_cloud_run_v2_service.app.location
  role     = "roles/run.invoker"
  member   = "serviceAccount:caller-sa@${var.project_id}.iam.gserviceaccount.com"
}

# ── Outputs ────────────────────────────────────────────────
output "service_url" {
  value       = google_cloud_run_v2_service.app.uri
  description = "The HTTPS URL of the deployed Cloud Run service"
}

output "service_account_email" {
  value = google_service_account.cloudrun_sa.email
}
```

```bash
terraform init
terraform plan  -var="project_id=my-project"
terraform apply -var="project_id=my-project"
```

---

## 13. Common Patterns & Use Cases

### REST API (Standard Pattern)

```
Client ──► Cloud Load Balancer ──► Cloud Run Service ──► Cloud SQL / Firestore
                 │
            Cloud Armor (WAF)
```

```bash
gcloud run deploy rest-api \
  --image=us-central1-docker.pkg.dev/my-project/repo/api:v1 \
  --ingress=internal-and-cloud-load-balancing \
  --no-allow-unauthenticated \
  --min-instances=2 \
  --concurrency=80 \
  --region=us-central1
```

### Event-Driven Processing with Eventarc

Trigger Cloud Run from GCS uploads, Pub/Sub messages, Audit Logs, and more:

```bash
# Trigger on GCS object finalization (file upload)
gcloud eventarc triggers create gcs-trigger \
  --location=us-central1 \
  --destination-run-service=my-processor \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=my-input-bucket" \
  --service-account=sa-eventarc@my-project.iam.gserviceaccount.com
```

```python
# Handle CloudEvents in your Cloud Run service
from flask import Flask, request
import json

app = Flask(__name__)

@app.route("/", methods=["POST"])
def handle_event():
    envelope = request.get_json()
    # CloudEvent attributes
    event_type = request.headers.get("ce-type")
    subject = request.headers.get("ce-subject")   # e.g., "objects/my-file.csv"
    
    if event_type == "google.cloud.storage.object.v1.finalized":
        bucket = envelope.get("bucket")
        name = envelope.get("name")
        process_file(bucket, name)
    
    return "ok", 200
```

### Scheduled Jobs with Cloud Scheduler

```
Cloud Scheduler (cron) ──► Cloud Run Job ──► Process data ──► BigQuery / GCS
```

See [Section 5](#5-cloud-run-jobs) for the Cloud Scheduler + Cloud Run Job setup.

### Sidecar Pattern (Multi-Container Services)

Run a proxy or log collector alongside your main container in the same instance:

```yaml
# service.yaml — multi-container with an Envoy sidecar proxy
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-multi-container-service
spec:
  template:
    metadata:
      annotations:
        run.googleapis.com/container-dependencies: '{"main-app":["envoy-proxy"]}'
    spec:
      containers:
        - name: main-app
          image: us-central1-docker.pkg.dev/my-project/repo/app:v1
          ports:
            - containerPort: 8080
        - name: envoy-proxy
          image: us-central1-docker.pkg.dev/my-project/repo/envoy:v1
          ports:
            - containerPort: 9000
```

### Fan-Out Pipeline (Pub/Sub)

```
HTTP Trigger ──► Cloud Run (orchestrator)
                     │
              Publish N messages
                     │
             Pub/Sub Topic ──► Cloud Run (worker) × N
                                    │
                              Process shard ──► Write to GCS/BQ
```

```python
from google.cloud import pubsub_v1
import json, os

publisher = pubsub_v1.PublisherClient()
topic = f"projects/{os.environ['PROJECT_ID']}/topics/work-queue"

def fan_out(items: list, batch_size: int = 100):
    """Distribute work across multiple Cloud Run worker instances."""
    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]
        message = json.dumps({"items": batch, "shard": i // batch_size})
        publisher.publish(topic, message.encode())
```

---

## 14. Cost Model & Optimization

### Billing Dimensions

| Dimension | Billed when | Unit |
|---|---|---|
| **CPU** | During request processing (request-only) OR always (always-on) | vCPU-seconds |
| **Memory** | During request processing OR always | GiB-seconds |
| **Requests** | Each HTTP request received | Per 1M requests |
| **Networking egress** | Data leaving the region | Per GiB |
| **Min instances** | Always (even with zero traffic) | vCPU-seconds + GiB-seconds |

### Free Tier (Per Month, Per Project)

| Resource | Free Amount |
|---|---|
| CPU | 180,000 vCPU-seconds |
| Memory | 360,000 GiB-seconds |
| Requests | 2,000,000 requests |
| Networking egress | 1 GiB (North America destinations) |

> **Note:** Free tier does NOT apply to min-instances — those are billed at full rate even at zero traffic.

### CPU Allocation Strategy

| Scenario | Best Mode | Rationale |
|---|---|---|
| REST API, webhooks | Request-only (default) | CPU only used during requests → cheaper |
| Background workers, streaming | Always-on | Needed for background goroutines/threads |
| WebSockets | Always-on | Connection persists between requests |
| ML inference (fast models) | Request-only | Inference completes within request |
| Long-running ML (>1s) | Always-on | Avoids CPU throttle mid-inference |

### Cost Optimization Tips

- **Use min-instances=0** for dev/staging services — zero idle cost.
- **Right-size memory** — the single biggest cost lever. Start with 256Mi and profile.
- **Increase concurrency** for I/O-bound workloads to serve more requests per instance.
- **Use Spot (preview)** for Cloud Run Jobs — significant discounts for interruptible batch workloads.
- **Set max-instances** to cap worst-case cost (e.g., during a traffic spike or bug).
- **Use `--cpu=0.08` (minimum)** for very light workloads — reduces cost dramatically vs. 1 full vCPU.
- **Co-locate** Cloud Run services in the same region as Cloud SQL / Memorystore to eliminate cross-region egress charges.
- **Enable request-based CPU** (default) unless your app genuinely needs always-on.

### Min Instances Cost Trade-off

```
                Cold Start Latency
                      ▲
                      │  ● min=0
                      │
                      │     ● min=1
                      │
                      │           ● min=2+
                      └──────────────────────► Monthly Cost
                      $0        $$$        $$$$
```

A single `min-instances=1` of an `e2-micro` equivalent typically costs ~$5–15/month and eliminates most cold starts for low-traffic services.

---

## 15. Troubleshooting Quick Reference

| Issue | Likely Cause | Fix |
|---|---|---|
| **Container fails to start (exits immediately)** | App crashes before binding to port, or wrong `PORT` | Check logs in Cloud Console; ensure app reads `PORT` env var: `port = int(os.environ.get("PORT", 8080))` |
| **503 Service Unavailable** | All instances are at max concurrency, or container unhealthy | Increase `--max-instances`, reduce `--concurrency`, or fix the health check endpoint |
| **Cold start latency is too high** | Image too large, slow initialization, scale-to-zero | Set `--min-instances=1`, reduce image size, defer non-critical init code |
| **Permission denied (403) on invoke** | Missing `roles/run.invoker` on the caller | Grant `roles/run.invoker` to the calling SA or `allUsers` for public services |
| **Cannot access Cloud SQL / Memorystore** | No VPC connectivity | Configure Direct VPC Egress or Serverless VPC Access Connector |
| **Memory limit exceeded (OOM killed)** | Container using more memory than allocated | Increase `--memory` limit; check for memory leaks; profile with Cloud Profiler |
| **Request timeout (deadline exceeded)** | Slow backend, long computation, or misconfigured timeout | Increase `--timeout`; offload long tasks to Cloud Tasks / Pub/Sub |
| **Secret not found / access denied** | SA missing `secretmanager.secretAccessor` or wrong secret name | Verify SA binding: `gcloud secrets add-iam-policy-binding` |
| **Deployment fails: image not found** | Wrong image path or Artifact Registry permissions | Verify image URI; grant `roles/artifactregistry.reader` to the Cloud Run SA |
| **Environment variable not available at startup** | Using Secret Manager volume before it's mounted | Ensure secret is injected as env var (not volume) for startup-time access |
| **VPC egress not working** | `--vpc-egress=all` not set, or connector in wrong region | Set `--vpc-egress=all-traffic` if all outbound must go through VPC; verify connector region matches service region |
| **Concurrency errors / data races** | Shared mutable state across concurrent requests | Make handlers stateless; use instance-level locking or per-request context |
| **Revision not receiving traffic after deploy** | Deployed with `--no-traffic` flag | Run `gcloud run services update-traffic --to-latest` |
| **Binary Authorization policy violation** | Image not attested / policy misconfigured | Check attestor config; use `gcloud container binauthz attestations list` to debug |
| **Logs not appearing in Cloud Logging** | Logs written to file instead of stdout/stderr | Always write to stdout/stderr; Cloud Run does NOT ship file-based logs automatically |
| **CPU throttled during background processing** | Default CPU allocation (request-only) throttles CPU between requests | Set `--no-cpu-throttling` (always-on CPU allocation) for background threads |

---

*Last updated: 2025 | Covers Cloud Run GA features as of this date.*
*Refer to [cloud.google.com/run/docs](https://cloud.google.com/run/docs) for the latest information.*
