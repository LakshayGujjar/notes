# GCP Cloud Functions Cheatsheet

> **TL;DR:** Cloud Functions is GCP's fully managed serverless Functions-as-a-Service (FaaS) platform — write a single-purpose function, wire it to a trigger, and Google handles everything else; 2nd gen (built on Cloud Run) adds concurrency, longer timeouts, and Eventarc-powered triggers.

---

## Table of Contents

1. [Overview & Core Concepts](#1-overview--core-concepts)
2. [1st Gen vs. 2nd Gen Deep Dive](#2-1st-gen-vs-2nd-gen-deep-dive)
3. [Trigger Types](#3-trigger-types)
4. [Function Structure & Programming Model](#4-function-structure--programming-model)
5. [Runtimes & Language Support](#5-runtimes--language-support)
6. [Deploying Functions](#6-deploying-functions)
7. [Environment Configuration](#7-environment-configuration)
8. [Networking](#8-networking)
9. [IAM & Security](#9-iam--security)
10. [Eventarc Integration (2nd Gen)](#10-eventarc-integration-2nd-gen)
11. [Concurrency & Scaling](#11-concurrency--scaling)
12. [Observability](#12-observability)
13. [Local Development & Testing](#13-local-development--testing)
14. [gcloud CLI Quick Reference](#14-gcloud-cli-quick-reference)
15. [Terraform Snippet](#15-terraform-snippet)
16. [Common Patterns & Use Cases](#16-common-patterns--use-cases)
17. [Troubleshooting Quick Reference](#17-troubleshooting-quick-reference)

---

## 1. Overview & Core Concepts

Cloud Functions is GCP's **Functions-as-a-Service (FaaS)** platform. You write a single-purpose function in a supported language, configure a trigger, and deploy — Google manages all infrastructure, OS patches, scaling, load balancing, and availability.

### Key Terminology

| Term | Description |
|---|---|
| **Function** | A single unit of deployable, event-driven code with one entry point |
| **Trigger** | The event source that invokes a function (HTTP, Pub/Sub, GCS, Firestore, etc.) |
| **Event** | A message describing something that happened (e.g., file uploaded, message published) |
| **Runtime** | The language execution environment (e.g., `python312`, `nodejs20`, `go122`) |
| **Invocation** | A single execution of a function in response to a trigger |
| **Cold Start** | Latency when a new instance must be initialized before handling a request |
| **Instance** | A single isolated execution environment running one or more invocations |
| **Concurrency** | Number of simultaneous requests a single instance handles (1 in 1st gen; up to 1000 in 2nd gen) |
| **Execution Environment** | The sandbox (OS, runtime, dependencies) in which the function runs |
| **Min Instances** | Instances kept warm to eliminate cold starts (billed even at idle) |
| **Max Instances** | Hard ceiling on the number of instances (prevents runaway cost/scaling) |
| **CloudEvents** | An open standard for event metadata format; used natively in 2nd gen triggers |
| **Eventarc** | GCP's unified eventing platform that routes events to Cloud Functions 2nd gen |
| **Functions Framework** | An open-source library for running Cloud Functions locally |

### Cloud Functions vs. Other GCP Compute Options

| Feature | Cloud Functions | Cloud Run | App Engine | GKE | GCE |
|---|---|---|---|---|---|
| **Deploy unit** | Function (code) | Container | App / Service | Pod | VM |
| **Scale to zero** | ✅ | ✅ | ✅ (Standard) | ❌ | ❌ |
| **Cold starts** | Yes | Yes | Yes | Rare | No |
| **Max concurrency/instance** | 1 (1st gen) / 1000 (2nd gen) | 1000 | Configurable | Configurable | N/A |
| **Max timeout** | 9 min (1st) / 60 min (2nd) | 60 min | 10 min (Std) | Unlimited | Unlimited |
| **Custom runtimes** | ❌ (2nd gen: limited) | ✅ | ✅ (Flex) | ✅ | ✅ |
| **Infra management** | None | None | None | Cluster ops | Full |
| **Event-driven triggers** | ✅ Native (many sources) | Via Eventarc | Via Scheduler | Via adapters | Manual |
| **Pricing** | Per invocation + duration | Per request + duration | Per instance-hour | Per node | Per VM |
| **Best for** | Single-purpose event handlers | Containerized APIs | Full web apps | Complex orchestration | Full VM control |

> **Note:** Cloud Functions 2nd gen is essentially a Cloud Run service with an Eventarc trigger attached. If you need containers, custom runtimes, or advanced routing, use Cloud Run directly. Use Cloud Functions when you want the simplest possible FaaS experience with native GCP event source integrations.

---

## 2. 1st Gen vs. 2nd Gen Deep Dive

### Side-by-Side Comparison

| Attribute | 1st Gen | 2nd Gen |
|---|---|---|
| **Underlying platform** | Proprietary sandbox | Cloud Run (fully managed) |
| **Eventing system** | Background triggers (legacy) | Eventarc (CloudEvents standard) |
| **Max timeout** | 9 minutes | 60 minutes |
| **Concurrency per instance** | 1 (one request at a time) | Up to 1,000 simultaneous requests |
| **Min instances** | ✅ Supported | ✅ Supported |
| **Max instances** | 3,000 | 3,000 (per region) |
| **Max memory** | 8 GB | 32 GB |
| **Max vCPUs** | 4 | 8 |
| **CPU allocation** | Request-only (throttled) | Request-only or always-on |
| **Traffic splitting** | ❌ Not supported | ✅ Supported (revisions) |
| **VPC egress options** | Serverless VPC Access Connector only | Connector + Direct VPC Egress |
| **Ingress controls** | Allow-all / internal / internal+GCLB | Allow-all / internal / internal+GCLB |
| **Build system** | Cloud Build (implicit) | Cloud Build → Artifact Registry |
| **Binary Authorization** | ❌ | ✅ Supported |
| **Supported runtimes** | Python, Node.js, Go, Java, Ruby, PHP, .NET | Python, Node.js, Go, Java, Ruby, PHP, .NET |
| **Pricing model** | Per invocation + CPU/mem duration | Per invocation + CPU/mem duration (Cloud Run pricing) |
| **Free tier** | 2M invocations/month | 2M invocations/month |
| **Event signature** | Background function `(event, context)` | CloudEvents `(cloud_event)` |
| **CMEK support** | ❌ | ✅ |

### When to Choose 1st Gen

- You have existing 1st gen functions and no urgent need to migrate.
- Your function is strictly single-threaded and request-isolated.
- You rely on 1st gen-specific background trigger SDKs.

### When to Choose 2nd Gen (Recommended for all new functions)

- You need concurrency to reduce cold starts and cost.
- You need timeouts longer than 9 minutes.
- You want traffic splitting for safe rollouts.
- You need Direct VPC Egress (no connector overhead).
- You want the full Eventarc event source ecosystem.
- You need memory above 8 GB or CPU above 4 vCPUs.

> **Note:** Google recommends **2nd gen for all new Cloud Functions deployments**. 1st gen is in maintenance mode and will not receive new features. Use the `--gen2` flag when deploying.

---

## 3. Trigger Types

### HTTP Trigger

Invoked synchronously by any HTTP client. Returns a response directly.

```python
# main.py
import functions_framework

@functions_framework.http
def hello_http(request):
    name = request.args.get("name", "World")
    return f"Hello, {name}!", 200
```

```bash
gcloud functions deploy hello-http \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=hello_http \
  --trigger-http \
  --allow-unauthenticated
```

### Pub/Sub Trigger

Invoked when a message is published to a Pub/Sub topic.

```python
# main.py
import base64
import functions_framework
from cloudevents.http import CloudEvent

@functions_framework.cloud_event
def handle_pubsub(cloud_event: CloudEvent):
    # Pub/Sub message data is base64-encoded in the CloudEvent
    data = base64.b64decode(cloud_event.data["message"]["data"]).decode()
    print(f"Received message: {data}")
```

```bash
gcloud functions deploy handle-pubsub \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=handle_pubsub \
  --trigger-topic=my-topic
```

### Cloud Storage Trigger

Invoked on object lifecycle events within a GCS bucket.

```python
# main.py
import functions_framework
from cloudevents.http import CloudEvent

@functions_framework.cloud_event
def handle_gcs(cloud_event: CloudEvent):
    data = cloud_event.data
    bucket = data["bucket"]
    name   = data["name"]
    event  = cloud_event["type"]   # e.g. google.cloud.storage.object.v1.finalized
    print(f"Event: {event} | gs://{bucket}/{name}")
```

```bash
# Trigger on object finalization (upload complete)
gcloud functions deploy handle-gcs \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=handle_gcs \
  --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
  --trigger-event-filters="bucket=my-bucket"
```

| GCS Event Type | Trigger Condition |
|---|---|
| `google.cloud.storage.object.v1.finalized` | Object created or overwritten (upload complete) |
| `google.cloud.storage.object.v1.deleted` | Object permanently deleted |
| `google.cloud.storage.object.v1.archived` | Live version archived (versioning enabled) |
| `google.cloud.storage.object.v1.metadataUpdated` | Object metadata updated |

### Firestore Trigger (via Eventarc)

Invoked on document write events in Firestore.

```python
# main.py
import functions_framework
from cloudevents.http import CloudEvent

@functions_framework.cloud_event
def handle_firestore(cloud_event: CloudEvent):
    # Event data contains old and new document values
    data = cloud_event.data
    print(f"Document path: {cloud_event['subject']}")
    print(f"Old value: {data.get('oldValue')}")
    print(f"New value: {data.get('value')}")
```

```bash
gcloud functions deploy handle-firestore \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=handle_firestore \
  --trigger-event-filters="type=google.cloud.firestore.document.v1.written" \
  --trigger-event-filters="database=(default)" \
  --trigger-event-filters-path-pattern="document=users/{userId}"
```

### Cloud Scheduler (HTTP) Trigger

Cloud Scheduler calls an HTTP function on a cron schedule.

```python
# main.py
import functions_framework

@functions_framework.http
def scheduled_job(request):
    # Verify the request comes from Cloud Scheduler
    if not request.headers.get("X-CloudScheduler-JobName"):
        return "Forbidden", 403
    print("Running scheduled job...")
    # ... job logic ...
    return "Done", 200
```

```bash
# 1. Deploy the HTTP function (private — no --allow-unauthenticated)
gcloud functions deploy scheduled-job \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=scheduled_job \
  --trigger-http

# 2. Create the Cloud Scheduler job targeting the function
gcloud scheduler jobs create http daily-job \
  --schedule="0 9 * * *" \
  --uri=$(gcloud functions describe scheduled-job --gen2 --region=us-central1 --format='value(serviceConfig.uri)') \
  --oidc-service-account-email=sa-scheduler@my-project.iam.gserviceaccount.com \
  --location=us-central1
```

### Eventarc — Cloud Audit Log Trigger

Fires on any GCP API call captured in Cloud Audit Logs.

```python
# main.py — triggered when a GCS bucket is created
import functions_framework
from cloudevents.http import CloudEvent

@functions_framework.cloud_event
def handle_audit_log(cloud_event: CloudEvent):
    protoPayload = cloud_event.data.get("protoPayload", {})
    method = protoPayload.get("methodName")
    resource = protoPayload.get("resourceName")
    principal = protoPayload.get("authenticationInfo", {}).get("principalEmail")
    print(f"Audit: {principal} called {method} on {resource}")
```

```bash
gcloud functions deploy handle-audit-log \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=handle_audit_log \
  --trigger-event-filters="type=google.cloud.audit.log.v1.written" \
  --trigger-event-filters="serviceName=storage.googleapis.com" \
  --trigger-event-filters="methodName=storage.buckets.create"
```

> **Note:** Audit Log triggers require Data Access Audit Logs to be enabled for the target service. Admin Activity logs are always on, but Data Read/Write logs must be explicitly enabled in IAM → Audit Logs.

---

## 4. Function Structure & Programming Model

### HTTP Function Signature

```python
import functions_framework
from flask import Request, Response
import json

@functions_framework.http
def my_http_function(request: Request) -> Response:
    # Parse request
    if request.method == "OPTIONS":
        # Handle CORS preflight
        headers = {
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Methods": "GET, POST",
            "Access-Control-Allow-Headers": "Content-Type",
        }
        return ("", 204, headers)

    body = request.get_json(silent=True) or {}
    name = body.get("name", "World")

    # Return structured response
    return Response(
        json.dumps({"message": f"Hello, {name}!"}),
        status=200,
        mimetype="application/json",
        headers={"Access-Control-Allow-Origin": "*"},
    )
```

### CloudEvents Function Signature (2nd Gen Event Triggers)

```python
import functions_framework
from cloudevents.http import CloudEvent

@functions_framework.cloud_event
def my_event_function(cloud_event: CloudEvent) -> None:
    # CloudEvent attributes
    event_id   = cloud_event["id"]
    event_type = cloud_event["type"]
    source     = cloud_event["source"]
    subject    = cloud_event.get("subject")
    time       = cloud_event["time"]

    # Event payload (structure depends on event type)
    data = cloud_event.data

    print(f"[{event_type}] id={event_id} subject={subject}")
    # Event functions must NOT return a value — return None implicitly
    # Non-zero exit or exception = failure → retry (if configured)
```

### Background Function Signature (1st Gen — Legacy)

```python
# 1st gen signature — still works but not recommended for new functions
def my_background_function(event: dict, context) -> None:
    """
    event   — dict containing the event data
    context — google.cloud.functions.Context object with metadata:
              context.event_id, context.event_type, context.timestamp,
              context.resource
    """
    print(f"Event ID: {context.event_id}")
    print(f"Data: {event}")
```

### Error Handling & Idempotency

```python
import functions_framework
from cloudevents.http import CloudEvent
import logging

logger = logging.getLogger(__name__)

@functions_framework.cloud_event
def idempotent_handler(cloud_event: CloudEvent) -> None:
    event_id = cloud_event["id"]

    try:
        # Check if already processed (use Firestore/Redis as deduplication store)
        if already_processed(event_id):
            logger.info(f"Duplicate event {event_id} — skipping")
            return   # Return normally so Eventarc does NOT retry

        process_event(cloud_event.data)
        mark_processed(event_id)

    except ValueError as e:
        # Non-retryable error — log and return normally to prevent retry
        logger.error(f"Invalid data in event {event_id}: {e}")
        return

    except Exception as e:
        # Retryable error — raise exception to signal Eventarc to retry
        logger.error(f"Transient error processing {event_id}: {e}")
        raise   # Eventarc will retry with exponential backoff
```

> **Note:** For event-triggered functions, **raising an exception** signals the eventing system to **retry**. Returning normally (even with an error logged) signals **success and no retry**. Design accordingly to avoid duplicate processing.

### CloudEvents Data Structure

```python
# Standard CloudEvent envelope fields
cloud_event["id"]          # Unique event identifier
cloud_event["source"]      # URI of the event origin
cloud_event["type"]        # Event type string
cloud_event["time"]        # RFC 3339 timestamp
cloud_event["subject"]     # Optional: resource within the source
cloud_event.data           # Event-specific payload dict
```

---

## 5. Runtimes & Language Support

### Supported Runtimes

| Language | 1st Gen Runtimes | 2nd Gen Runtimes | Dep. File |
|---|---|---|---|
| **Python** | 3.7, 3.8, 3.9, 3.10, 3.11 | 3.9, 3.10, 3.11, 3.12 | `requirements.txt` |
| **Node.js** | 10, 12, 14, 16, 18 | 18, 20, 22 | `package.json` |
| **Go** | 1.11, 1.13, 1.16, 1.18, 1.19, 1.20 | 1.21, 1.22 | `go.mod` |
| **Java** | 11, 17 | 11, 17, 21 | `pom.xml` / `build.gradle` |
| **Ruby** | 2.6, 2.7, 3.0 | 3.2, 3.3 | `Gemfile` |
| **PHP** | 7.4, 8.1 | 8.1, 8.2, 8.3 | `composer.json` |
| **.NET** | 3.1, 6.0 | 6.0, 8.0 | `*.csproj` |

> **Note:** Runtimes without active GCP support are in maintenance mode (security patches only). Always target the latest stable runtime for new functions. Check [cloud.google.com/functions/docs/concepts/execution-environment](https://cloud.google.com/functions/docs/concepts/execution-environment) for current EOL dates.

### Deployment Package Structure by Language

#### Python

```
my-function/
├── main.py              # Entry point (function defined here)
├── requirements.txt     # pip dependencies
└── helper.py            # Additional modules (co-located)
```

```
# requirements.txt
functions-framework==3.*
google-cloud-storage==2.*
```

#### Node.js

```
my-function/
├── index.js             # Entry point (exports the function)
├── package.json         # npm dependencies + "main" field
└── package-lock.json
```

```javascript
// index.js
const functions = require('@google-cloud/functions-framework');

functions.http('helloHttp', (req, res) => {
  res.send(`Hello, ${req.query.name || 'World'}!`);
});
```

#### Go

```
my-function/
├── function.go          # Package must be 'p' (or declared in go.mod)
├── go.mod
└── go.sum
```

```go
// function.go
package p

import (
    "fmt"
    "net/http"

    "github.com/GoogleCloudPlatform/functions-framework-go/funcframework"
)

func init() {
    funcframework.RegisterHTTPFunctionContext(HelloHTTP)
}

func HelloHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "Hello, World!")
}
```

#### Java (Maven)

```
my-function/
├── pom.xml
└── src/main/java/com/example/
    └── HelloFunction.java
```

```
# pom.xml dependency snippet
<dependency>
  <groupId>com.google.cloud.functions</groupId>
  <artifactId>functions-framework-api</artifactId>
  <version>1.1.0</version>
</dependency>
```

---

## 6. Deploying Functions

### Full Deployment Command (2nd Gen — All Key Flags)

```bash
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \                                        # Local source directory
  --entry-point=my_handler \                          # Function name in source
  --trigger-http \                                    # or --trigger-topic, --trigger-event-filters
  --allow-unauthenticated \                           # Public HTTP; omit for private
  --memory=512MB \                                    # 128MB–32GB
  --cpu=1 \                                           # vCPUs (2nd gen only)
  --timeout=60s \                                     # Max request duration
  --min-instances=1 \                                 # Keep warm (avoids cold starts)
  --max-instances=100 \                               # Hard scaling cap
  --concurrency=80 \                                  # Requests per instance (2nd gen)
  --service-account=sa-fn@my-project.iam.gserviceaccount.com \
  --set-env-vars=ENV=prod,LOG_LEVEL=info \
  --set-secrets=DB_PASSWORD=db-password:latest \
  --vpc-connector=my-connector \
  --vpc-egress=private-ranges-only \
  --ingress-settings=internal-and-gclb \
  --run-service-account=sa-fn@my-project.iam.gserviceaccount.com \
  --labels=team=backend,app=my-api
```

### Deploy from Cloud Storage (zip)

```bash
# Package and upload source
zip -r function.zip main.py requirements.txt
gsutil cp function.zip gs://my-bucket/functions/function.zip

# Deploy from GCS zip
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=gs://my-bucket/functions/function.zip \
  --entry-point=my_handler \
  --trigger-http
```

### Deploy from Cloud Source Repositories

```bash
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=https://source.developers.google.com/projects/my-project/repos/my-repo/moveable-aliases/main/paths/functions/my-function \
  --entry-point=my_handler \
  --trigger-http
```

### Update a Function (Partial Update)

```bash
# Update only the environment variables — no source redeploy
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --update-env-vars=LOG_LEVEL=debug

# Update only the memory limit
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --memory=1GB
```

### Traffic Splitting (2nd Gen Only)

```bash
# Deploy a new version without sending it traffic
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=my_handler \
  --trigger-http \
  --no-allow-unauthenticated

# The underlying Cloud Run service can then be traffic-split
gcloud run services update-traffic my-function \
  --region=us-central1 \
  --to-revisions=my-function-00002-abc=10,my-function-00001-xyz=90
```

### Delete a Function

```bash
gcloud functions delete my-function \
  --gen2 \
  --region=us-central1 \
  --quiet
```

### Build Process Under the Hood

```
gcloud functions deploy
        │
        ▼
  1. Source uploaded to Cloud Storage (staging bucket)
        │
        ▼
  2. Cloud Build triggered automatically
     - Installs dependencies (pip / npm / go mod)
     - Builds container image
        │
        ▼
  3. Image pushed to Artifact Registry
     (us-central1-docker.pkg.dev/PROJECT/gcf-artifacts/)
        │
        ▼
  4. Cloud Run service created/updated (2nd gen)
  4. Sandbox environment provisioned  (1st gen)
        │
        ▼
  5. Eventarc trigger attached (2nd gen event functions)
```

> **Note:** You can view Cloud Build logs for each deployment in the Cloud Console under Cloud Build → History, or with `gcloud builds list --filter="tags=gcf-deploy"`.

---

## 7. Environment Configuration

### Environment Variables

```bash
# Set at deploy time
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --set-env-vars=ENV=prod,DB_HOST=10.0.0.1,LOG_LEVEL=info

# Update without full redeploy
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --update-env-vars=LOG_LEVEL=debug

# Remove specific variables
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --remove-env-vars=OLD_VAR
```

```python
# Access in code
import os
env       = os.environ.get("ENV", "development")
log_level = os.environ.get("LOG_LEVEL", "info")
```

### Secrets via Secret Manager

```bash
# Inject secret as an environment variable
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --set-secrets=DB_PASSWORD=db-password:latest

# Inject secret as a mounted file
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --set-secrets=/secrets/db.json=db-config:latest
```

```python
# Access env-var secret in code (same as a regular env var)
import os
db_password = os.environ["DB_PASSWORD"]

# Access file-mounted secret
with open("/secrets/db.json") as f:
    import json
    db_config = json.load(f)
```

### Memory & CPU Configuration

| Memory | Default CPU (2nd gen) | Notes |
|---|---|---|
| 128 MB – 512 MB | 0.333 vCPU | Low-compute, short tasks |
| 512 MB – 2 GB | 1 vCPU | Standard functions |
| 2 GB – 8 GB | 2 vCPU | CPU-intensive tasks |
| 8 GB – 16 GB | 4 vCPU | ML inference, heavy processing |
| 16 GB – 32 GB | 8 vCPU | Maximum — large batch tasks |

```bash
# Explicit CPU override (2nd gen only — decouples from memory)
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --memory=512MB \
  --cpu=2          # 2 vCPUs even with 512MB memory
```

### Scaling & Concurrency

```bash
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --min-instances=2 \      # Keep 2 warm; billed even at zero traffic
  --max-instances=50 \     # Never exceed 50 instances
  --concurrency=100        # Each instance handles up to 100 simultaneous requests
```

### Timeout Configuration

```bash
# 1st gen: max 540s (9 minutes)
gcloud functions deploy my-function \
  --region=us-central1 \
  --timeout=540s

# 2nd gen: max 3600s (60 minutes)
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --timeout=3600s
```

### Service Account Assignment

```bash
# Assign a custom, least-privilege service account
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --service-account=sa-my-fn@my-project.iam.gserviceaccount.com
```

---

## 8. Networking

### Ingress Controls

| Setting | Who can invoke |
|---|---|
| `all` (default) | Any client from the internet |
| `internal-only` | Only VPC internal traffic and Cloud Functions/Run in same project |
| `internal-and-gclb` | Internal + Google Cloud Load Balancer (recommended for public APIs behind Cloud Armor) |

```bash
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --ingress-settings=internal-and-gclb
```

### VPC Egress — Serverless VPC Access Connector

Required for 1st gen; also available in 2nd gen. Allows functions to reach private VPC resources.

```bash
# Create the connector
gcloud compute networks vpc-access connectors create my-connector \
  --network=default \
  --region=us-central1 \
  --range=10.8.0.0/28

# Attach to function
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --vpc-connector=my-connector \
  --vpc-egress=private-ranges-only    # or 'all-traffic'
```

### VPC Egress — Direct VPC Egress (2nd Gen Only — Recommended)

Lower latency, no connector resource to manage:

```bash
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --network=my-vpc \
  --subnet=my-subnet \
  --vpc-egress=private-ranges-only
```

| | VPC Access Connector | Direct VPC Egress |
|---|---|---|
| **Gen support** | 1st + 2nd gen | 2nd gen only |
| **Extra resource** | Connector VMs | None |
| **Latency** | Slightly higher | Lower |
| **Recommended** | 1st gen / legacy | ✅ 2nd gen new deployments |

### Cloud SQL Connection

```python
# Connect to Cloud SQL via Unix socket (no VPC connector needed — uses Cloud SQL API)
import sqlalchemy
import os

def create_pool():
    return sqlalchemy.create_engine(
        sqlalchemy.engine.url.URL.create(
            drivername="postgresql+pg8000",
            username=os.environ["DB_USER"],
            password=os.environ["DB_PASS"],
            database=os.environ["DB_NAME"],
            query={"unix_sock": f"/cloudsql/{os.environ['INSTANCE_CONNECTION_NAME']}/.s.PGSQL.5432"}
        ),
        pool_size=2,          # Keep small — functions don't have persistent connections
        max_overflow=1,
        pool_timeout=10,
        pool_recycle=1800,
    )

# Initialize pool at module level (reused across warm invocations)
_pool = None

def get_pool():
    global _pool
    if _pool is None:
        _pool = create_pool()
    return _pool
```

```bash
# Reference the Cloud SQL instance at deploy time
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --set-env-vars=INSTANCE_CONNECTION_NAME=my-project:us-central1:my-db \
  --set-secrets=DB_PASS=db-password:latest
```

### Memorystore (Redis) Connection

Requires VPC connector or Direct VPC Egress since Memorystore has no public IP:

```python
import redis
import os

# Initialize client at module level for connection reuse
redis_client = redis.Redis(
    host=os.environ["REDIS_HOST"],
    port=int(os.environ.get("REDIS_PORT", 6379)),
    decode_responses=True,
)
```

---

## 9. IAM & Security

### Making a Function Public or Private

```bash
# Public (allow any unauthenticated caller)
gcloud functions add-invoker-policy-binding my-function \
  --gen2 \
  --region=us-central1 \
  --member=allUsers

# Private (restrict to a specific service account)
gcloud functions add-invoker-policy-binding my-function \
  --gen2 \
  --region=us-central1 \
  --member=serviceAccount:caller-sa@my-project.iam.gserviceaccount.com

# Remove public access
gcloud functions remove-invoker-policy-binding my-function \
  --gen2 \
  --region=us-central1 \
  --member=allUsers
```

### Key IAM Roles

| Role | Description |
|---|---|
| `roles/cloudfunctions.admin` | Full control over all Cloud Functions resources |
| `roles/cloudfunctions.developer` | Create, update, delete functions; cannot change IAM |
| `roles/cloudfunctions.invoker` | Invoke HTTP functions (1st gen) |
| `roles/cloudfunctions.viewer` | Read-only view of functions |
| `roles/run.invoker` | Invoke 2nd gen functions (they're Cloud Run services underneath) |

> **Note:** For 2nd gen functions, the invoker role is `roles/run.invoker` (granted on the underlying Cloud Run service), not `roles/cloudfunctions.invoker`. The `gcloud functions add-invoker-policy-binding` command handles this automatically.

### Service-to-Service Authentication (OIDC)

```python
# Calling a private Cloud Function from another function/service
import google.auth.transport.requests
import google.oauth2.id_token
import requests

def call_private_function(function_url: str, payload: dict) -> dict:
    """Call a private Cloud Function with an OIDC identity token."""
    auth_req = google.auth.transport.requests.Request()
    # The audience must be the exact function URL
    token = google.oauth2.id_token.fetch_id_token(auth_req, function_url)

    response = requests.post(
        function_url,
        json=payload,
        headers={"Authorization": f"Bearer {token}"},
    )
    response.raise_for_status()
    return response.json()
```

### Workload Identity Federation

Allows GitHub Actions (or other external identities) to deploy functions without service account keys:

```bash
# Configure GitHub Actions to deploy Cloud Functions keylessly
gcloud iam workload-identity-pools create github-pool --location=global
gcloud iam workload-identity-pools providers create-oidc github-provider \
  --workload-identity-pool=github-pool \
  --location=global \
  --issuer-uri=https://token.actions.githubusercontent.com \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository"

gcloud iam service-accounts add-iam-policy-binding deploy-sa@my-project.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUM/locations/global/workloadIdentityPools/github-pool/attribute.repository/my-org/my-repo"
```

### Security Best Practices

- Never use `allUsers` invoker on event-triggered functions — they are invoked by GCP internally, not HTTP clients.
- Always assign a **dedicated, least-privilege service account** per function.
- Store all credentials in **Secret Manager** — never in environment variables or source code.
- Set `--ingress-settings=internal-and-gclb` for HTTP functions that must be public, then front with Cloud Armor.
- Enable **Binary Authorization** for 2nd gen functions in regulated environments.
- Audit function invocations via Cloud Audit Logs (`cloudaudit.googleapis.com/activity`).

---

## 10. Eventarc Integration (2nd Gen)

### What Eventarc Is

Eventarc is GCP's **unified eventing platform** that routes events from 100+ GCP services to Cloud Functions (2nd gen), Cloud Run, and GKE. It replaces the legacy background trigger system using the **CloudEvents 1.0** open standard.

```
Event Source (GCS, Pub/Sub, Audit Log, etc.)
        │
        ▼
   Eventarc Trigger
   (filters events, handles delivery + retry)
        │
        ▼
   Cloud Function 2nd Gen (CloudEvent handler)
```

### Supported Event Sources

| Source | Event Types | Notes |
|---|---|---|
| **Cloud Audit Logs** | Any GCP API operation | Requires audit logging enabled |
| **Pub/Sub** | Message published | Direct; no audit log needed |
| **Cloud Storage** | Object lifecycle events | Via Pub/Sub notification internally |
| **Firestore** | Document create/update/delete | Via audit log |
| **Firebase RTDB** | Data write events | Via audit log |
| **Firebase Auth** | User created/deleted | Via audit log |
| **Third-party (Channels)** | Custom partner events | Via Eventarc Channels |

### Creating an Eventarc Trigger Directly

```bash
# Create a standalone Eventarc trigger (separate from function deploy)
gcloud eventarc triggers create my-trigger \
  --location=us-central1 \
  --destination-run-service=my-function \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=my-input-bucket" \
  --service-account=sa-eventarc@my-project.iam.gserviceaccount.com

# List all triggers
gcloud eventarc triggers list --location=us-central1

# Describe a trigger
gcloud eventarc triggers describe my-trigger --location=us-central1
```

### CloudEvents Handler — Full Example

```python
# main.py — 2nd gen CloudEvents handler with full attribute access
import functions_framework
from cloudevents.http import CloudEvent
import logging, json

logger = logging.getLogger(__name__)

@functions_framework.cloud_event
def handle_cloud_event(cloud_event: CloudEvent) -> None:
    # Standard CloudEvent attributes
    logger.info(json.dumps({
        "severity": "INFO",
        "event_id":   cloud_event["id"],
        "event_type": cloud_event["type"],
        "source":     cloud_event["source"],
        "subject":    cloud_event.get("subject"),
        "time":       str(cloud_event["time"]),
    }))

    # Route by event type
    event_type = cloud_event["type"]

    if event_type == "google.cloud.storage.object.v1.finalized":
        handle_gcs_upload(cloud_event.data)
    elif event_type.startswith("google.cloud.pubsub"):
        handle_pubsub_message(cloud_event.data)
    else:
        logger.warning(f"Unhandled event type: {event_type}")

def handle_gcs_upload(data: dict) -> None:
    bucket = data["bucket"]
    name   = data["name"]
    size   = data.get("size", "unknown")
    logger.info(f"New file: gs://{bucket}/{name} ({size} bytes)")
```

### Eventarc Retry Behavior

| Scenario | Behavior |
|---|---|
| Function returns 2xx | Event acknowledged — no retry |
| Function returns non-2xx | Event retried with exponential backoff |
| Function raises exception | Event retried with exponential backoff |
| Max retry duration exceeded | Event sent to dead-letter topic (if configured) |
| Default retry window | 24 hours |

```bash
# Attach a dead-letter Pub/Sub topic for undeliverable events
gcloud eventarc triggers update my-trigger \
  --location=us-central1 \
  --dead-letter-topic=projects/my-project/topics/dead-letter
```

---

## 11. Concurrency & Scaling

### 1st Gen Scaling Model

```
Request 1 ──► Instance A  (1 request at a time)
Request 2 ──► Instance B  (new instance spun up)
Request 3 ──► Instance C  (another new instance)

→ N simultaneous requests = N instances
→ High concurrency = many cold starts
```

### 2nd Gen Scaling Model

```
Request 1  ┐
Request 2  ├──► Instance A  (up to 1000 concurrent)
Request 80 ┘

Request 81 ──► Instance B  (new instance only when Instance A is full)

→ N simultaneous requests ≈ N / concurrency instances
→ Much fewer cold starts, dramatically lower cost at scale
```

### Cold Start Anatomy

```
Cold Start Timeline:
  t=0ms    Trigger fires
  t=50ms   Instance allocated
  t=300ms  Container image pulled (cached after first run)
  t=800ms  Runtime initialized (Python interpreter started)
  t=1200ms Dependencies imported (your requirements.txt loaded)
  t=1300ms Module-level code executed (DB pool created, etc.)
  t=1350ms Handler invoked ← actual function code runs
```

### Minimizing Cold Starts

```python
# PATTERN: Initialize expensive resources at module level
# These are reused across warm invocations on the same instance

import functions_framework
from google.cloud import bigquery, storage
import os

# Initialized ONCE per instance (not per invocation)
bq_client  = bigquery.Client()
gcs_client = storage.Client()
TABLE_ID   = os.environ["BQ_TABLE"]

@functions_framework.http
def ingest_data(request):
    # bq_client is already warm — no initialization overhead
    rows = [{"data": request.get_json()}]
    bq_client.insert_rows_json(TABLE_ID, rows)
    return "ok", 200
```

```bash
# Keep instances warm to avoid cold starts entirely
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --min-instances=2 \
  --concurrency=80
```

### CPU Boost During Cold Start (2nd Gen)

```bash
# Grant extra CPU during initialization, throttle back for requests
# Enabled automatically when --cpu >= 1 and concurrency >= 1
gcloud functions deploy my-function \
  --gen2 \
  --region=us-central1 \
  --cpu=1 \
  --concurrency=10   # CPU boost active during cold start
```

### Stateless Design Best Practices

- **Never store per-request state in global variables.** Multiple concurrent requests share the same instance.
- **Use connection pools, not single connections.** Pools handle concurrent requests sharing one socket.
- **Write outputs to external storage** (GCS, Firestore, BigQuery) — local disk is ephemeral and not shared.
- **Make functions idempotent** — the same event may be delivered more than once.
- **Avoid background goroutines/threads** that outlive the request in 1st gen.

---

## 12. Observability

### Built-in Request Logging

Every invocation automatically produces a log entry in Cloud Logging with:
- Function name, region, execution ID
- HTTP method, URL, response code, latency
- Memory usage, instance ID

### Structured Logging (JSON to stdout)

```python
import functions_framework
import json, sys, os, logging

def structured_log(severity: str, message: str, **kwargs) -> None:
    """Emit a JSON log entry that Cloud Logging parses natively."""
    entry = {
        "severity": severity,
        "message": message,
        "function": os.environ.get("K_SERVICE", "local"),
        "revision": os.environ.get("K_REVISION", "local"),
        **kwargs,
    }
    print(json.dumps(entry), flush=True)

@functions_framework.http
def my_function(request):
    structured_log("INFO", "Request received",
                   method=request.method,
                   path=request.path,
                   user_agent=request.headers.get("User-Agent"))
    try:
        result = process(request)
        structured_log("INFO", "Request completed", result_size=len(result))
        return result, 200
    except Exception as e:
        structured_log("ERROR", "Request failed", error=str(e))
        raise
```

### Cloud Trace Integration

```python
# pip install opentelemetry-exporter-gcp-trace opentelemetry-sdk
from opentelemetry import trace
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Initialize once at module level
provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(CloudTraceSpanExporter()))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)

import functions_framework

@functions_framework.http
def traced_function(request):
    with tracer.start_as_current_span("process-request") as span:
        span.set_attribute("http.method", request.method)
        result = do_work()
        span.set_attribute("result.size", len(result))
    return result, 200
```

### Error Reporting

Exceptions are automatically captured when using structured logging or the Error Reporting library:

```python
# pip install google-cloud-error-reporting
from google.cloud import error_reporting

err_client = error_reporting.Client()

@functions_framework.http
def my_function(request):
    try:
        return risky_operation(), 200
    except Exception:
        err_client.report_exception()   # Sends full stack trace to Error Reporting
        return "Internal Server Error", 500
```

### Useful Cloud Logging Filters

```bash
# All logs from a specific function
resource.type="cloud_function"
resource.labels.function_name="my-function"

# 2nd gen function logs
resource.type="cloud_run_revision"
resource.labels.service_name="my-function"

# Errors only
resource.type="cloud_function"
resource.labels.function_name="my-function"
severity>=ERROR

# Cold start events (new instance initialized)
resource.type="cloud_function"
textPayload:"worker_started"

# Slow invocations (2nd gen — over 5 seconds)
resource.type="cloud_run_revision"
resource.labels.service_name="my-function"
httpRequest.latency>"5s"

# Eventarc delivery failures
resource.type="audited_resource"
protoPayload.serviceName="eventarc.googleapis.com"
severity=ERROR
```

```bash
# Stream function logs in real time
gcloud functions logs read my-function \
  --gen2 \
  --region=us-central1 \
  --limit=50
```

---

## 13. Local Development & Testing

### Functions Framework (Local Runner)

```bash
# Install the Functions Framework
pip install functions-framework

# Run an HTTP function locally
functions-framework --target=my_handler --port=8080 --debug

# Run a CloudEvent function locally
functions-framework --target=my_event_handler \
  --signature-type=cloudevent \
  --port=8080
```

### Send Test Events Locally

```bash
# Test HTTP function
curl -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{"name": "Test"}'

# Test CloudEvent function (simulate GCS event)
curl -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -H "ce-id: test-001" \
  -H "ce-type: google.cloud.storage.object.v1.finalized" \
  -H "ce-source: //storage.googleapis.com/projects/_/buckets/my-bucket" \
  -H "ce-specversion: 1.0" \
  -d '{"bucket":"my-bucket","name":"file.csv","size":"1024"}'

# Test Pub/Sub CloudEvent locally
curl -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -H "ce-type: google.cloud.pubsub.topic.v1.messagePublished" \
  -H "ce-source: //pubsub.googleapis.com/projects/my-project/topics/my-topic" \
  -H "ce-specversion: 1.0" \
  -d '{"message":{"data":"SGVsbG8gV29ybGQ=","messageId":"123"}}'
```

### Unit Testing (Python)

```python
# test_main.py
import unittest
from unittest.mock import MagicMock, patch
from cloudevents.http import CloudEvent
import main   # your function module

class TestHTTPFunction(unittest.TestCase):
    def test_hello_returns_200(self):
        request = MagicMock()
        request.method = "GET"
        request.args = {"name": "Alice"}

        response, status = main.hello_http(request)

        self.assertEqual(status, 200)
        self.assertIn("Alice", response)

class TestCloudEventFunction(unittest.TestCase):
    def test_handles_gcs_event(self):
        attributes = {
            "type": "google.cloud.storage.object.v1.finalized",
            "source": "//storage.googleapis.com/projects/_/buckets/test-bucket",
            "id": "evt-001",
        }
        data = {"bucket": "test-bucket", "name": "test-file.csv"}
        cloud_event = CloudEvent(attributes, data)

        # Should not raise
        main.handle_gcs(cloud_event)

if __name__ == "__main__":
    unittest.main()
```

### Integration Testing Against Deployed Function

```bash
# Call an authenticated 2nd gen function using gcloud token
FUNCTION_URL=$(gcloud functions describe my-function \
  --gen2 --region=us-central1 --format='value(serviceConfig.uri)')

TOKEN=$(gcloud auth print-identity-token)

curl -X POST "$FUNCTION_URL" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"test": true}'
```

### Using Pub/Sub Emulator for Local Testing

```bash
# Start the emulator
gcloud beta emulators pubsub start --project=my-project

# Configure environment
$(gcloud beta emulators pubsub env-init)

# Create topic and subscription
gcloud pubsub topics create my-topic
gcloud pubsub subscriptions create my-sub --topic=my-topic

# Publish a test message
gcloud pubsub topics publish my-topic --message='{"key":"value"}'
```

---

## 14. `gcloud` CLI Quick Reference

### Deploy & Manage

| Command | Description |
|---|---|
| `gcloud functions deploy NAME --gen2 ...` | Deploy or update a 2nd gen function |
| `gcloud functions deploy NAME ...` | Deploy or update a 1st gen function |
| `gcloud functions delete NAME --gen2 --region=R` | Delete a function and all its resources |
| `gcloud functions describe NAME --gen2 --region=R` | Show full function configuration |
| `gcloud functions list --gen2` | List all 2nd gen functions |
| `gcloud functions list` | List all 1st gen functions |

```bash
# Get the deployed function's URL
gcloud functions describe my-function \
  --gen2 \
  --region=us-central1 \
  --format='value(serviceConfig.uri)'
```

### Invoke & Test

| Command | Description |
|---|---|
| `gcloud functions call NAME --data='{...}'` | Invoke a 1st gen function with JSON data |
| `gcloud run services proxy NAME --region=R` | Proxy a 2nd gen function locally for testing |

```bash
# Call a 1st gen function
gcloud functions call my-function \
  --region=us-central1 \
  --data='{"name":"Alice"}'

# Call a deployed 2nd gen HTTP function with auth token
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  $(gcloud functions describe my-function --gen2 --region=us-central1 --format='value(serviceConfig.uri)')
```

### Logs

| Command | Description |
|---|---|
| `gcloud functions logs read NAME --gen2` | Read recent logs for a 2nd gen function |
| `gcloud functions logs read NAME` | Read recent logs for a 1st gen function |
| `gcloud functions logs read NAME --limit=N` | Limit number of log entries returned |

```bash
# Stream logs (tail -f equivalent)
gcloud beta functions logs read my-function \
  --gen2 \
  --region=us-central1 \
  --limit=100 \
  --format="table(level,name,execution_id,time_utc,log)"
```

### IAM

```bash
# View current IAM policy
gcloud functions get-iam-policy my-function \
  --gen2 --region=us-central1

# Grant invoker to a service account
gcloud functions add-invoker-policy-binding my-function \
  --gen2 \
  --region=us-central1 \
  --member=serviceAccount:sa@my-project.iam.gserviceaccount.com

# Make a function public
gcloud functions add-invoker-policy-binding my-function \
  --gen2 \
  --region=us-central1 \
  --member=allUsers

# Remove public access
gcloud functions remove-invoker-policy-binding my-function \
  --gen2 \
  --region=us-central1 \
  --member=allUsers
```

### Eventarc Triggers

```bash
# List Eventarc triggers in a region
gcloud eventarc triggers list --location=us-central1

# Describe a trigger
gcloud eventarc triggers describe my-trigger --location=us-central1

# Delete a trigger
gcloud eventarc triggers delete my-trigger --location=us-central1
```

---

## 15. Terraform Snippet

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

# ── Source code bucket ─────────────────────────────────────
resource "google_storage_bucket" "source_bucket" {
  name                        = "${var.project_id}-fn-source"
  location                    = "US"
  uniform_bucket_level_access = true
  force_destroy               = true
}

resource "google_storage_bucket_object" "function_source" {
  name   = "function-source-${filemd5("${path.module}/function.zip")}.zip"
  bucket = google_storage_bucket.source_bucket.name
  source = "${path.module}/function.zip"   # Pre-built zip of your function code
}

# ── Service Account ────────────────────────────────────────
resource "google_service_account" "function_sa" {
  account_id   = "sa-cloud-function"
  display_name = "Cloud Function Service Account"
}

# ── IAM: SA can access BigQuery ───────────────────────────
resource "google_project_iam_member" "function_bq" {
  project = var.project_id
  role    = "roles/bigquery.dataEditor"
  member  = "serviceAccount:${google_service_account.function_sa.email}"
}

# ── IAM: SA can access Secret Manager ────────────────────
resource "google_project_iam_member" "function_secret" {
  project = var.project_id
  role    = "roles/secretmanager.secretAccessor"
  member  = "serviceAccount:${google_service_account.function_sa.email}"
}

# ── Serverless VPC Access Connector ───────────────────────
resource "google_vpc_access_connector" "connector" {
  name          = "fn-vpc-connector"
  region        = "us-central1"
  network       = "default"
  ip_cidr_range = "10.8.0.0/28"
}

# ── Pub/Sub Topic (event source) ──────────────────────────
resource "google_pubsub_topic" "trigger_topic" {
  name = "fn-trigger-topic"
}

# ── Cloud Function 2nd Gen ────────────────────────────────
resource "google_cloudfunctions2_function" "function" {
  name     = "my-terraform-function"
  location = "us-central1"

  build_config {
    runtime     = "python312"
    entry_point = "handle_pubsub"

    source {
      storage_source {
        bucket = google_storage_bucket.source_bucket.name
        object = google_storage_bucket_object.function_source.name
      }
    }
  }

  service_config {
    max_instance_count               = 50
    min_instance_count               = 1
    available_memory                 = "512M"
    timeout_seconds                  = 60
    max_instance_request_concurrency = 80

    service_account_email = google_service_account.function_sa.email

    # Environment variables (plain)
    environment_variables = {
      ENV      = "production"
      LOG_LEVEL = "info"
      PROJECT  = var.project_id
    }

    # Secret injected as environment variable
    secret_environment_variables {
      key        = "DB_PASSWORD"
      project_id = var.project_id
      secret     = "db-password"
      version    = "latest"
    }

    # VPC connector for private resource access
    vpc_connector                  = google_vpc_access_connector.connector.id
    vpc_connector_egress_settings  = "PRIVATE_RANGES_ONLY"
    ingress_settings               = "ALLOW_INTERNAL_ONLY"
  }

  # Eventarc trigger on Pub/Sub topic
  event_trigger {
    trigger_region        = "us-central1"
    event_type            = "google.cloud.pubsub.topic.v1.messagePublished"
    pubsub_topic          = google_pubsub_topic.trigger_topic.id
    service_account_email = google_service_account.function_sa.email
    retry_policy          = "RETRY_POLICY_RETRY"
  }

  depends_on = [
    google_project_iam_member.function_bq,
    google_project_iam_member.function_secret,
    google_vpc_access_connector.connector,
  ]
}

# ── IAM: Allow specific SA to invoke the function ─────────
resource "google_cloudfunctions2_function_iam_member" "invoker" {
  project        = google_cloudfunctions2_function.function.project
  location       = google_cloudfunctions2_function.function.location
  cloud_function = google_cloudfunctions2_function.function.name
  role           = "roles/cloudfunctions.invoker"
  member         = "serviceAccount:caller-sa@${var.project_id}.iam.gserviceaccount.com"
}

# ── Outputs ────────────────────────────────────────────────
output "function_uri" {
  value       = google_cloudfunctions2_function.function.service_config[0].uri
  description = "The HTTPS URI of the deployed Cloud Function"
}

output "function_name" {
  value = google_cloudfunctions2_function.function.name
}

output "trigger_topic" {
  value = google_pubsub_topic.trigger_topic.name
}
```

```bash
terraform init
terraform plan  -var="project_id=my-project"
terraform apply -var="project_id=my-project"
```

---

## 16. Common Patterns & Use Cases

### Image Processing Pipeline (GCS → Process → Write Back)

```python
# main.py — resize images on upload
import functions_framework
from cloudevents.http import CloudEvent
from google.cloud import storage
from PIL import Image
import io

gcs = storage.Client()
OUTPUT_BUCKET = "my-processed-images"

@functions_framework.cloud_event
def resize_image(cloud_event: CloudEvent) -> None:
    data   = cloud_event.data
    bucket = data["bucket"]
    name   = data["name"]

    if not name.lower().endswith((".jpg", ".jpeg", ".png")):
        return   # Skip non-images

    # Download original
    blob = gcs.bucket(bucket).blob(name)
    content = blob.download_as_bytes()

    # Resize
    img = Image.open(io.BytesIO(content))
    img.thumbnail((800, 800))

    # Upload to output bucket
    out = io.BytesIO()
    img.save(out, format=img.format or "JPEG")
    gcs.bucket(OUTPUT_BUCKET).blob(f"resized/{name}").upload_from_string(
        out.getvalue(), content_type=blob.content_type
    )
```

### Real-Time Data Ingestion (Pub/Sub → BigQuery)

```python
# main.py — stream Pub/Sub messages into BigQuery
import base64, json, os
import functions_framework
from cloudevents.http import CloudEvent
from google.cloud import bigquery

bq    = bigquery.Client()
TABLE = os.environ["BQ_TABLE"]   # project.dataset.table

@functions_framework.cloud_event
def pubsub_to_bq(cloud_event: CloudEvent) -> None:
    raw  = base64.b64decode(cloud_event.data["message"]["data"])
    row  = json.loads(raw)
    errors = bq.insert_rows_json(TABLE, [row])
    if errors:
        raise RuntimeError(f"BigQuery insert errors: {errors}")
```

### Scheduled Data Export (Cloud Scheduler → HTTP Function)

```python
# main.py — export daily report to GCS
import functions_framework
from google.cloud import bigquery, storage
import datetime, json, os

bq  = bigquery.Client()
gcs = storage.Client()

@functions_framework.http
def export_report(request):
    if not request.headers.get("X-CloudScheduler-JobName"):
        return "Forbidden", 403

    date  = datetime.date.today().isoformat()
    query = f"SELECT * FROM `my_dataset.events` WHERE DATE(ts) = '{date}'"
    rows  = [dict(row) for row in bq.query(query).result()]

    blob = gcs.bucket(os.environ["OUTPUT_BUCKET"]).blob(f"reports/{date}.json")
    blob.upload_from_string(json.dumps(rows), content_type="application/json")

    return f"Exported {len(rows)} rows for {date}", 200
```

### Webhook Handler (HTTP → Pub/Sub Fan-Out)

```python
# main.py — validate webhook and publish to Pub/Sub
import functions_framework, hashlib, hmac, json, os
from flask import Request
from google.cloud import pubsub_v1

publisher  = pubsub_v1.PublisherClient()
TOPIC_PATH = publisher.topic_path(os.environ["PROJECT_ID"], os.environ["TOPIC"])
SECRET     = os.environ["WEBHOOK_SECRET"].encode()

@functions_framework.http
def webhook_handler(request: Request):
    # Verify HMAC signature
    sig = request.headers.get("X-Hub-Signature-256", "").removeprefix("sha256=")
    expected = hmac.new(SECRET, request.data, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(sig, expected):
        return "Unauthorized", 401

    payload = request.get_json()
    publisher.publish(TOPIC_PATH, json.dumps(payload).encode(),
                      event_type=payload.get("action", "unknown"))
    return "Accepted", 202
```

### Database Change Listener (Firestore Trigger)

```python
# main.py — react to Firestore document changes
import functions_framework
from cloudevents.http import CloudEvent
from google.cloud import firestore
import json

db = firestore.Client()

@functions_framework.cloud_event
def on_user_updated(cloud_event: CloudEvent) -> None:
    # Extract document path from subject
    # subject format: "documents/users/USER_ID"
    doc_path = cloud_event["subject"].replace("documents/", "")

    old_val = cloud_event.data.get("oldValue", {}).get("fields", {})
    new_val = cloud_event.data.get("value", {}).get("fields", {})

    old_email = old_val.get("email", {}).get("stringValue")
    new_email = new_val.get("email", {}).get("stringValue")

    if old_email and old_email != new_email:
        # Email changed — trigger verification workflow
        db.document(doc_path).update({"emailVerified": False})
        send_verification_email(new_email)
```

### Slack Notification Bot (HTTP Webhook)

```python
# main.py — send Slack alert on GCS upload
import functions_framework, os, requests
from cloudevents.http import CloudEvent

SLACK_WEBHOOK = os.environ["SLACK_WEBHOOK_URL"]

@functions_framework.cloud_event
def notify_slack(cloud_event: CloudEvent) -> None:
    data   = cloud_event.data
    name   = data["name"]
    bucket = data["bucket"]
    size   = int(data.get("size", 0))

    requests.post(SLACK_WEBHOOK, json={
        "text": f":file_folder: New file uploaded!\n"
                f"*File:* `{name}`\n"
                f"*Bucket:* `{bucket}`\n"
                f"*Size:* {size / 1024:.1f} KB"
    }, timeout=10)
```

---

## 17. Troubleshooting Quick Reference

| Issue | Likely Cause | Fix |
|---|---|---|
| **Cold start timeout — request fails before handler runs** | Heavy module-level initialization; large dependency tree | Move non-critical init inside handler; use lazy imports; set `--min-instances=1` |
| **Function not triggered after deploy** | Eventarc trigger not created; wrong event filter; IAM missing on trigger SA | Check `gcloud eventarc triggers list`; verify filter values; grant `roles/eventarc.eventReceiver` to trigger SA |
| **Permission denied (403) on HTTP invoke** | Missing `roles/run.invoker` (2nd gen) or `roles/cloudfunctions.invoker` (1st gen) on caller | Run `gcloud functions add-invoker-policy-binding`; for public: add `allUsers` |
| **Memory limit exceeded (OOM killed)** | Function allocating more RAM than configured | Increase `--memory`; profile with Cloud Profiler; fix memory leaks |
| **VPC resource unreachable** | No VPC connector or Direct VPC Egress configured; wrong egress setting | Add `--vpc-connector` or `--network/--subnet`; set `--vpc-egress=all-traffic` if needed |
| **Cloud SQL connection refused** | Missing Cloud SQL client IAM role; wrong socket path; connection pool exhausted | Grant `roles/cloudsql.client`; verify `INSTANCE_CONNECTION_NAME`; reduce `pool_size` |
| **Deployment fails: build error** | Missing dependency in `requirements.txt`; syntax error; unsupported package requiring C headers | Check Cloud Build logs; add system packages via `apt.txt` (not available in Cloud Functions — use Cloud Run) |
| **Eventarc trigger not firing** | Data Access Audit Logs not enabled for service; event filter mismatch | Enable audit logs in IAM → Audit Logs; double-check `methodName` and `serviceName` filter values |
| **Duplicate event processing** | Eventarc delivers at-least-once; your function is not idempotent | Implement deduplication using Firestore/Memorystore keyed on `cloud_event["id"]` |
| **Function times out** | Task exceeds `--timeout`; downstream service is slow | Increase timeout (max 9 min 1st gen, 60 min 2nd gen); add async offloading via Pub/Sub |
| **Function scales to max instances unexpectedly** | Low concurrency setting; traffic spike; expensive handler | Increase `--concurrency` (2nd gen); optimize handler latency; check for downstream bottleneck |
| **Secret not accessible at runtime** | Function SA missing `roles/secretmanager.secretAccessor`; secret name typo | Grant SA access to secret; verify secret ID matches exactly (case-sensitive) |
| **`gcloud functions call` returns empty response** | Function returns `None` instead of a string/tuple | Ensure HTTP function explicitly returns `(body, status_code)` or a Response object |
| **Logs not showing in Cloud Logging** | Logs written to stderr without JSON structure; buffered output | Write JSON to stdout; call `flush=True`; avoid print buffering |
| **2nd gen function shows as Cloud Run in Console** | Expected — 2nd gen functions are Cloud Run services | This is normal; manage via `gcloud functions` CLI or Cloud Functions console section |
| **Function crashes on first request after deploy** | Module-level code raises exception (missing env var, bad config) | Wrap module-level init in try/except; validate env vars at startup; check logs for traceback |
| **Billing higher than expected** | `--min-instances` set too high; `--concurrency=1` causing many instances | Reduce `min-instances`; increase concurrency for I/O-bound functions; review Cloud Billing dashboard |

---

*Last updated: 2025 | Covers Cloud Functions GA features (1st gen and 2nd gen) as of this date.*
*Refer to [cloud.google.com/functions/docs](https://cloud.google.com/functions/docs) for the latest information.*
