# GCP Eventing & Async Services — Production Cheatsheet
### Eventarc · Cloud Tasks · Cloud Scheduler

---

## PART 1 — EVENTARC

Eventarc is GCP's fully managed eventing backbone that routes events from GCP services, custom application sources, and third-party SaaS providers to event consumers using the CloudEvents 1.0 specification. It requires no infrastructure management — the underlying transport is built on Pub/Sub and Cloud Audit Logs, abstracting the complexity of event routing, filtering, and delivery guarantees behind a single trigger API.

---

### 1.1 OVERVIEW & ARCHITECTURE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                          EVENT PROVIDERS                                    │
│                                                                             │
│  ┌─────────────────────┐  ┌──────────────────────┐  ┌───────────────────┐  │
│  │  GCP DIRECT SOURCES │  │  AUDIT LOG SOURCES   │  │  PUB/SUB TOPICS   │  │
│  │  Cloud Storage      │  │  ANY GCP API call    │  │  Custom messages  │  │
│  │  Artifact Registry  │  │  that writes to      │  │  Third-party data │  │
│  │  Cloud Firestore    │  │  Cloud Audit Logs    │  │                   │  │
│  │  Pub/Sub topics     │  │  (GCE, GKE, SQL...)  │  └───────────────────┘  │
│  └─────────────────────┘  └──────────────────────┘                         │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  CHANNEL-BASED (Third-party partner events via Channel Connection)   │  │
│  │  Datadog · GitHub · Zendesk · PagerDuty · Custom app events         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      EVENTARC ROUTING LAYER                                 │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  TRIGGER (per project, per region)                                   │  │
│  │  ┌─────────────────────┐    ┌─────────────────────────────────────┐ │  │
│  │  │  Event Filter(s)    │ →  │  Delivery (HTTP POST, CloudEvents)  │ │  │
│  │  │  type = X           │    │  with OIDC token auth               │ │  │
│  │  │  bucket = Y         │    └─────────────────────────────────────┘ │  │
│  │  │  serviceName = Z    │                                             │  │
│  │  └─────────────────────┘                                             │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  Retry: exponential backoff up to 24h  │  DLQ: Pub/Sub subscription DLQ   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼──────────────────────┐
                    │               │                      │
          ┌─────────▼──────┐ ┌──────▼──────┐  ┌──────────▼──────────┐
          │  Cloud Run     │ │  GKE        │  │  Google Cloud       │
          │  Service       │ │  Service    │  │  Workflows          │
          │  (HTTP POST)   │ │  (internal) │  │  (execution start)  │
          └────────────────┘ └─────────────┘  └─────────────────────┘
                    │
          ┌─────────▼──────┐
          │  Cloud         │
          │  Functions 2nd │
          │  gen           │
          └────────────────┘
```

#### Event Delivery Models

| Model | Transport | Latency | Ordering | Retry | Use Case |
|---|---|---|---|---|---|
| Direct (Google sources) | Internal GCP eventing | Sub-second | No | Yes, exponential backoff | Cloud Storage, Pub/Sub native events |
| Audit Log-based | Cloud Audit Logs → Eventarc | 1–3 minutes | No | Yes | Any GCP API call event (GCE create, GKE delete) |
| Pub/Sub-backed | Pub/Sub topic → trigger | Sub-second | Per-subscription | Yes, configurable | Custom sources, third-party integrations |
| Channel-based | Partner channel pipeline | Varies | No | Yes | Third-party SaaS events (Datadog, GitHub, etc.) |

> ⚠️ **NOTE:** Audit Log-based triggers have an inherent 1–3 minute latency due to Cloud Audit Log processing time. Do not use them for latency-sensitive workflows — use direct Google source triggers or Pub/Sub-backed triggers instead.

---

### 1.2 CORE CONCEPTS

Eventarc's data model centers on three resources: **Triggers** (what to listen for and where to send it), **Events** (the CloudEvents-formatted payloads), and **Channels** (pipelines for custom or partner events). Understanding these three is sufficient to design any Eventarc architecture.

#### CloudEvents 1.0 Attribute Reference

| Attribute | Required | Description | Example Value |
|---|---|---|---|
| `id` | ✅ | Unique identifier for the event | `1234567890` |
| `source` | ✅ | URI identifying the event source | `//storage.googleapis.com/my-bucket` |
| `specversion` | ✅ | CloudEvents specification version | `1.0` |
| `type` | ✅ | Event type string (reverse DNS notation) | `google.cloud.storage.object.v1.finalized` |
| `datacontenttype` | ✅ | Media type of the `data` attribute | `application/json` |
| `time` | Recommended | Event occurrence time (RFC 3339) | `2025-03-30T10:00:00Z` |
| `subject` | Optional | Subject within the event source | `objects/reports/q1-2025.csv` |

#### Key Definitions

- **Trigger**: Eventarc resource defining event filters and routing destination; scoped to a project and region; all filters are ANDed (every filter must match for the trigger to fire)
- **Event**: a CloudEvents 1.0 compliant JSON payload delivered as an HTTP POST to the destination
- **Channel**: a named, project-scoped pipeline for routing third-party or custom application events; requires a Channel Connection on the provider side
- **Event Provider**: any GCP service or third-party system that emits events consumable by Eventarc

---

### 1.3 EVENT TYPES & SOURCES REFERENCE

Eventarc supports two categories of events: **direct** (Google services emit natively) and **Audit Log-based** (any GCP API call that produces an audit log entry).

#### GCP Direct Event Sources

| Source Service | Event Type | Trigger Condition |
|---|---|---|
| Cloud Storage | `google.cloud.storage.object.v1.finalized` | Object created or overwritten |
| Cloud Storage | `google.cloud.storage.object.v1.deleted` | Object deleted |
| Cloud Storage | `google.cloud.storage.object.v1.archived` | Object archived (versioning on) |
| Cloud Storage | `google.cloud.storage.object.v1.metadataUpdated` | Object metadata changed |
| Pub/Sub | `google.cloud.pubsub.topic.v1.messagePublished` | Message published to topic |
| Artifact Registry | `google.cloud.artifactregistry.repository.v1.created` | Repository created |
| Artifact Registry | `google.cloud.artifactregistry.dockerimage.v1.updated` | Docker image pushed/updated |
| Cloud Firestore | `google.cloud.firestore.document.v1.created` | Document created |
| Cloud Firestore | `google.cloud.firestore.document.v1.updated` | Document updated |
| Firebase RTDB | `google.firebase.database.document.v1.created` | RTDB document created |
| BigQuery | `google.cloud.bigquery.v2.TableService.InsertTable` | Table created (Audit Log) |
| Cloud SQL | `google.cloud.cloudsql.v1.SqlService.Insert` | Instance created (Audit Log) |

#### Audit Log-Based Events — Universal GCP API Coverage

Any GCP API call that generates a Cloud Audit Log entry can trigger an Eventarc trigger using `google.cloud.audit.log.v1.written` with `serviceName` and `methodName` filters:

```text
Event Type: google.cloud.audit.log.v1.written
Filters:    serviceName = "compute.googleapis.com"
            methodName  = "v1.compute.instances.insert"
```

This enables event-driven reactions to **any GCP resource lifecycle event** — VM creation, GKE cluster deletion, IAM policy changes, Cloud SQL backups, and more — without any source-side code changes.

> 💡 **TIP:** To find the correct `serviceName` and `methodName` for an API call, make the call once, then check Cloud Logging: `resource.type="audited_resource"` and examine `protoPayload.serviceName` and `protoPayload.methodName` fields.

---

### 1.4 TRIGGERS — CLI REFERENCE

```bash
# ── Enable required APIs ───────────────────────────────────────────────────────
gcloud services enable eventarc.googleapis.com
gcloud services enable run.googleapis.com
gcloud services enable workflows.googleapis.com

# ── Trigger: Cloud Storage object finalized → Cloud Run ───────────────────────
gcloud eventarc triggers create storage-trigger \
  --location=us-central1 \
  --destination-run-service=my-processor \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=my-input-bucket" \
  --service-account=eventarc-sa@PROJECT.iam.gserviceaccount.com

# ── Trigger: Cloud Storage → Cloud Run with object prefix filter ───────────────
gcloud eventarc triggers create storage-prefix-trigger \
  --location=us-central1 \
  --destination-run-service=my-processor \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=my-input-bucket" \
  --event-filters-path-pattern="name=uploads/reports/*" \
  --service-account=eventarc-sa@PROJECT.iam.gserviceaccount.com

# ── Trigger: Pub/Sub message published → Cloud Run ────────────────────────────
gcloud eventarc triggers create pubsub-trigger \
  --location=us-central1 \
  --destination-run-service=my-processor \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.pubsub.topic.v1.messagePublished" \
  --transport-topic=projects/PROJECT/topics/my-topic \
  --service-account=eventarc-sa@PROJECT.iam.gserviceaccount.com

# ── Trigger: Audit Log (any GCE instance created) → Cloud Run ─────────────────
gcloud eventarc triggers create audit-log-trigger \
  --location=us-central1 \
  --destination-run-service=my-processor \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.audit.log.v1.written" \
  --event-filters="serviceName=compute.googleapis.com" \
  --event-filters="methodName=v1.compute.instances.insert" \
  --service-account=eventarc-sa@PROJECT.iam.gserviceaccount.com

# ── Trigger: Artifact Registry image push → Cloud Run ─────────────────────────
gcloud eventarc triggers create ar-trigger \
  --location=us-central1 \
  --destination-run-service=my-ci-handler \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.artifactregistry.dockerimage.v1.updated" \
  --service-account=eventarc-sa@PROJECT.iam.gserviceaccount.com

# ── Trigger: Cloud Storage → GKE service (Eventarc for GKE) ───────────────────
gcloud eventarc triggers create gke-trigger \
  --location=us-central1 \
  --destination-gke-cluster=my-cluster \
  --destination-gke-location=us-central1 \
  --destination-gke-namespace=my-namespace \
  --destination-gke-service=my-service \
  --destination-gke-path=/events \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=my-bucket" \
  --service-account=eventarc-sa@PROJECT.iam.gserviceaccount.com

# ── Trigger: Cloud Storage → Google Cloud Workflows ───────────────────────────
gcloud eventarc triggers create workflow-trigger \
  --location=us-central1 \
  --destination-workflow=my-workflow \
  --destination-workflow-location=us-central1 \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=my-bucket" \
  --service-account=eventarc-sa@PROJECT.iam.gserviceaccount.com

# ── Read / Inspect ─────────────────────────────────────────────────────────────
gcloud eventarc triggers list --location=us-central1
gcloud eventarc triggers list --location=us-central1 \
  --format="table(name, state, eventFilters, destination)"
gcloud eventarc triggers describe storage-trigger --location=us-central1

# ── Delete ─────────────────────────────────────────────────────────────────────
gcloud eventarc triggers delete storage-trigger --location=us-central1
```

> ⚠️ **NOTE:** Most Eventarc trigger properties — including event filters, destination, and service account — are **immutable after creation**. To change any of these, delete the trigger and recreate it. Only the trigger's labels can be updated in place.

> ⚠️ **NOTE:** Cloud Storage triggers must be created in the **same region as the GCS bucket**. A trigger in `us-central1` will not receive events from a bucket in `us-east1`. Use a multi-region bucket if you need a single trigger to cover multiple regions.

---

### 1.5 IAM & SERVICE ACCOUNT REQUIREMENTS

Eventarc uses **two distinct service account identities** that serve different roles in the event delivery chain. Confusing them is the most common source of Eventarc IAM errors.

**Identity 1 — Trigger Service Account** (`--service-account` on the trigger): the identity that Eventarc impersonates when delivering events to the destination. Must have the invoker role for the target service.

**Identity 2 — Eventarc Service Agent** (`service-PROJECT_NUMBER@gcp-sa-eventarc.iam.gserviceaccount.com`): a Google-managed service account; needs `roles/eventarc.serviceAgent` and source-specific read roles. Managed automatically by GCP.

```bash
# ── Create a dedicated trigger service account ────────────────────────────────
gcloud iam service-accounts create eventarc-trigger-sa \
  --display-name="Eventarc Trigger Service Account"

# ── Grant Cloud Run invoker (for Cloud Run destinations) ─────────────────────
gcloud run services add-iam-policy-binding my-processor \
  --region=us-central1 \
  --member="serviceAccount:eventarc-trigger-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/run.invoker"

# ── Grant Workflows invoker (for Workflows destinations) ─────────────────────
gcloud projects add-iam-policy-binding PROJECT \
  --member="serviceAccount:eventarc-trigger-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/workflows.invoker"

# ── Grant Eventarc Event Receiver (required on ALL trigger service accounts) ──
gcloud projects add-iam-policy-binding PROJECT \
  --member="serviceAccount:eventarc-trigger-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/eventarc.eventReceiver"

# ── For Pub/Sub-backed triggers: grant Pub/Sub subscriber role ────────────────
gcloud projects add-iam-policy-binding PROJECT \
  --member="serviceAccount:eventarc-trigger-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/pubsub.subscriber"

# ── For Audit Log triggers: enable Data Access audit logs ────────────────────
# Export current policy, add auditConfigs for the target service, re-import:
gcloud projects get-iam-policy PROJECT --format=json > /tmp/policy.json
# Edit /tmp/policy.json to add:
# "auditConfigs": [{"service": "compute.googleapis.com",
#   "auditLogConfigs": [{"logType": "DATA_READ"}, {"logType": "DATA_WRITE"}]}]
gcloud projects set-iam-policy PROJECT /tmp/policy.json
```

#### Eventarc IAM Roles Reference

| Role | Principal | Purpose |
|---|---|---|
| `roles/eventarc.admin` | Developer | Create, manage, delete all Eventarc triggers and channels |
| `roles/eventarc.viewer` | Operator / auditor | Read-only view of triggers and channels |
| `roles/eventarc.eventReceiver` | Trigger SA | **Required** — receive and forward events |
| `roles/eventarc.serviceAgent` | Eventarc service agent | GCP-managed internal routing permissions |
| `roles/run.invoker` | Trigger SA | Invoke Cloud Run destination |
| `roles/workflows.invoker` | Trigger SA | Start Workflows execution as destination |
| `roles/iam.serviceAccountTokenCreator` | Eventarc service agent | Generate OIDC tokens for authenticated delivery |

---

### 1.6 RECEIVING & PROCESSING EVENTS IN CLOUD RUN

The event receiver is any HTTP service that accepts CloudEvents-formatted POST requests. Eventarc delivers events in the **binary content mode** by default — CloudEvents attributes arrive as HTTP headers prefixed with `ce-`, and the event data is the request body.

A valid Cloud Run receiver must: (1) listen on `$PORT`, (2) accept HTTP POST, (3) return 2xx to acknowledge, (4) return 4xx/5xx to trigger retry, and (5) process idempotently.

```python
import os
import json
from flask import Flask, request

app = Flask(__name__)

@app.route("/", methods=["POST"])
def handle_event():
    # ── Read CloudEvents attributes from HTTP headers (binary mode) ────────────
    ce_type    = request.headers.get("ce-type", "unknown")
    ce_source  = request.headers.get("ce-source", "unknown")
    ce_id      = request.headers.get("ce-id", "unknown")
    ce_subject = request.headers.get("ce-subject", "")

    # ── Parse the event data from the request body ────────────────────────────
    event_data = request.get_json(silent=True) or {}

    print(f"Event received: id={ce_id} type={ce_type} source={ce_source}")

    # ── Route to handler based on event type ──────────────────────────────────
    if ce_type == "google.cloud.storage.object.v1.finalized":
        bucket = event_data.get("bucket", "")
        name   = event_data.get("name", "")
        size   = event_data.get("size", 0)
        print(f"File uploaded: gs://{bucket}/{name} ({size} bytes)")
        process_new_file(bucket, name)

    elif ce_type == "google.cloud.audit.log.v1.written":
        payload = event_data.get("protoPayload", {})
        method  = payload.get("methodName", "")
        caller  = payload.get("authenticationInfo", {}).get("principalEmail", "")
        print(f"Audit event: {method} by {caller}")
        handle_audit_event(method, event_data)

    # Return 200 to acknowledge; Eventarc will NOT retry
    # Return 4xx / 5xx to signal failure; Eventarc WILL retry
    return ("", 204)

def process_new_file(bucket: str, name: str) -> None:
    """Idempotent file processing — safe to call multiple times."""
    # Check if already processed (use Firestore/Redis as idempotency store)
    pass

def handle_audit_event(method: str, data: dict) -> None:
    pass

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    app.run(host="0.0.0.0", port=port, debug=False)
```

**Idempotency is mandatory.** Eventarc guarantees **at-least-once delivery** — the same event can be delivered more than once due to retries, network issues, or infrastructure restarts. Use a durable idempotency key (the `ce-id` header value is unique per event occurrence) backed by Firestore, Redis, or Cloud SQL to detect and skip duplicate deliveries.

**Retry policy:** Eventarc uses exponential backoff, retrying for up to **24 hours** for direct Google source events. For Pub/Sub-backed triggers, retry duration is controlled by the Pub/Sub subscription's `ackDeadline` and message retention settings.

---

### 1.7 CUSTOM EVENTS & CHANNELS

Channels enable applications to publish their own domain events into Eventarc — transforming it from a GCP-events-only system into a full application eventing bus. Any service with the right IAM permissions can publish a CloudEvent to a channel, and any number of triggers can subscribe to those events.

```bash
# ── Create an Eventarc channel for custom application events ──────────────────
gcloud eventarc channels create my-app-channel \
  --location=us-central1

# ── Get the channel's full resource name (needed for publishing) ──────────────
CHANNEL_NAME=$(gcloud eventarc channels describe my-app-channel \
  --location=us-central1 \
  --format="value(name)")
echo "Channel: ${CHANNEL_NAME}"

# ── Create a trigger subscribed to a custom event type on the channel ─────────
gcloud eventarc triggers create order-completed-trigger \
  --location=us-central1 \
  --channel=my-app-channel \
  --event-filters="type=com.myapp.order.v1.completed" \
  --destination-run-service=order-fulfillment \
  --destination-run-region=us-central1 \
  --service-account=eventarc-sa@PROJECT.iam.gserviceaccount.com

# ── Grant permission to publish to the channel ────────────────────────────────
gcloud eventarc channels add-iam-policy-binding my-app-channel \
  --location=us-central1 \
  --member="serviceAccount:publisher-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/eventarc.eventPublisher"

# ── Publish a custom CloudEvent to the channel via REST API ───────────────────
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/cloudevents+json" \
  -d '{
    "specversion": "1.0",
    "type": "com.myapp.order.v1.completed",
    "source": "//myapp/orders-service",
    "id": "order-12345-completed-20250330",
    "time": "2025-03-30T10:00:00Z",
    "datacontenttype": "application/json",
    "data": {
      "orderId": "12345",
      "customerId": "cust-789",
      "total": 149.99,
      "currency": "USD",
      "items": [
        {"sku": "WIDGET-A", "qty": 2},
        {"sku": "GADGET-B", "qty": 1}
      ]
    }
  }' \
  "https://eventarc.googleapis.com/v1/${CHANNEL_NAME}:publishEvents"
```

> 💡 **TIP:** Use reverse DNS notation for custom event type names (e.g., `com.mycompany.service.v1.eventname`). This avoids collisions with GCP's own event types and clearly communicates ownership. Include a version in the type string so you can evolve the event schema without breaking existing subscribers.

---

### 1.8 EVENTARC ADVANCED TOPICS

#### Event Filtering

All `--event-filters` flags on a trigger are evaluated with AND logic — every filter must match for the trigger to fire. Two filter operators are supported:

- **Equality** (`=`): exact match — `bucket=my-bucket`, `serviceName=compute.googleapis.com`
- **Path pattern** (`--event-filters-path-pattern`): prefix/glob match on nested fields — `name=uploads/reports/*`

A trigger cannot be created with OR logic across filters. For OR-style routing (e.g., react to object finalized OR object deleted), create two separate triggers pointing to the same destination.

#### Dead Letter Queue Configuration

For Pub/Sub-backed Eventarc triggers, configure a DLQ on the underlying Pub/Sub subscription to capture events that exceed the maximum delivery attempts:

```bash
# ── Find the Pub/Sub subscription created by Eventarc for a trigger ──────────
SUBSCRIPTION=$(gcloud eventarc triggers describe my-trigger \
  --location=us-central1 \
  --format="value(transport.pubsub.subscription)")
echo "Subscription: ${SUBSCRIPTION}"

# ── Create a DLQ topic ────────────────────────────────────────────────────────
gcloud pubsub topics create eventarc-dlq

# ── Configure the subscription with DLQ and max retry attempts ───────────────
gcloud pubsub subscriptions modify-push-config "${SUBSCRIPTION}" \
  --dead-letter-topic=projects/PROJECT/topics/eventarc-dlq \
  --max-delivery-attempts=5

# ── Grant the Pub/Sub service account permission to publish to the DLQ ────────
gcloud pubsub topics add-iam-policy-binding eventarc-dlq \
  --member="serviceAccount:service-PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"

# ── Monitor DLQ for undeliverable events ─────────────────────────────────────
gcloud pubsub subscriptions create eventarc-dlq-monitor \
  --topic=eventarc-dlq
gcloud pubsub subscriptions pull eventarc-dlq-monitor --limit=10
```

#### Advanced Considerations

- **Cross-project triggers**: triggers can route events to destinations in different projects; requires the trigger SA to have the invoker role in the destination project, and the Eventarc service agent to have token creator permissions cross-project
- **Eventarc for GKE**: requires installing the Eventarc GKE add-on operator in the target cluster; events are forwarded to internal Kubernetes Services via an in-cluster forwarding component; the destination must be reachable as a ClusterIP service
- **Regional alignment**: Eventarc triggers are regional resources; the trigger must be in the same region as the event source where applicable (GCS triggers must match bucket region; Pub/Sub-backed triggers can be multi-region)

---

### 1.9 LIMITS, QUOTAS & TROUBLESHOOTING

#### Key Limits

| Resource | Limit | Notes |
|---|---|---|
| Triggers per region per project | 1,000 | Soft limit; request increase via quota console |
| Event size (direct sources) | 512 KB | Events larger than 512 KB are truncated |
| Event size (Pub/Sub-backed) | 10 MB | Bound by Pub/Sub message size limit |
| Event filters per trigger | 10 | All ANDed; use multiple triggers for OR logic |
| Channels per project per region | 10 | Soft limit |
| Delivery retry window | 24h (direct), configurable (Pub/Sub) | Events dropped after window expires |
| Audit Log trigger latency | 1–3 minutes | Cloud Audit Log processing pipeline delay |
| Custom channel publish rate | 1,000 events/sec | Per channel; soft limit |
| Trigger creation propagation | ~1 minute | Trigger may not fire immediately after creation |

#### Troubleshooting Decision Tree

```text
Event not reaching destination?
│
├── Step 1: Verify trigger is ACTIVE (not FAILED or PROVISIONING)
│   gcloud eventarc triggers describe TRIGGER_NAME --location=REGION
│   Look for: state=ACTIVE
│   └── state=FAILED: inspect lastDeliveryAttempt.responseCode field
│
├── Step 2: Check service account permissions
│   ├── Trigger SA must have roles/eventarc.eventReceiver
│   ├── Trigger SA must have roles/run.invoker on the destination Cloud Run service
│   └── For Audit Log triggers: verify Data Access logs are ENABLED for the source service
│       gcloud projects get-iam-policy PROJECT --format=json | jq '.auditConfigs'
│
├── Step 3: Verify destination is healthy
│   gcloud run services describe DESTINATION_SERVICE --region=REGION
│   └── Latest revision must be serving 100% traffic (not in failed rollback)
│
├── Step 4: Confirm the source event actually occurred
│   For Cloud Storage: verify object was created in the EXACT bucket named in filter
│   For Audit Log: check Cloud Logging:
│     gcloud logging read 'protoPayload.serviceName="compute.googleapis.com"
│       AND protoPayload.methodName="v1.compute.instances.insert"' \
│       --freshness=1h
│
├── Step 5: Check for Pub/Sub DLQ messages (Pub/Sub-backed triggers only)
│   gcloud pubsub subscriptions pull EVENTARC_SUBSCRIPTION --limit=10 --auto-ack=false
│
└── For Audit Log triggers specifically:
    ├── Was the API call made in the SAME REGION as the trigger?
    ├── Are Data Access audit logs enabled for ADMIN_READ, DATA_READ, DATA_WRITE?
    └── Was the API call made by a service account? Some API calls don't generate
        DATA_WRITE logs when made by GCP-internal service agents
```


---

## PART 2 — CLOUD TASKS

Cloud Tasks is a fully managed, serverless asynchronous task queue service that decouples the components of an application by allowing producers to enqueue work items independently of when or how they are processed. Unlike event buses (Pub/Sub, Eventarc), Cloud Tasks is designed for **controlled execution** — you specify exactly where each task goes, at what rate, with what retry policy, and with what deduplication semantics.

---

### 2.1 OVERVIEW & ARCHITECTURE

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                           PRODUCERS                                          │
│  Cloud Run · GCE · Cloud Functions · App Engine · External HTTP client      │
│  (any service with roles/cloudtasks.enqueuer on the queue)                  │
└───────────────────────────────────────┬──────────────────────────────────────┘
                                        │ CreateTask (gRPC or HTTP)
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        CLOUD TASKS QUEUE                                     │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  Rate Limiting           Retry Policy           Scheduling           │   │
│  │  maxDispatchesPerSecond  maxAttempts             scheduleTime         │   │
│  │  maxConcurrentDispatches minBackoff / maxBackoff  (future delivery)   │   │
│  │  (global rate limiter)   maxDoublings                                 │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Task Storage (up to 30 days, up to 1M tasks per queue)             │    │
│  │  task.name (dedup key) · task.httpRequest · task.scheduleTime       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Dead Letter Queue (optional — Pub/Sub topic for failed tasks)      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────┬──────────────────────────────────────┘
                                        │ HTTP POST (OIDC/OAuth2 authenticated)
                                        │ X-CloudTasks-* headers injected
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                            HTTP TARGET WORKER                                │
│  Cloud Run · GKE service · App Engine · External HTTPS endpoint             │
│  Returns 2xx → task deleted  |  Returns 4xx/5xx → task retried              │
└──────────────────────────────────────────────────────────────────────────────┘
                                        │
                              Cloud Monitoring
                              queue/depth · task/attempt_count
                              task/dispatch_count · task/response_count
```

#### Cloud Tasks vs Pub/Sub — When to Use Which

| Aspect | Cloud Tasks | Pub/Sub |
|---|---|---|
| Primary model | Task queue — work item dispatched to one handler | Message bus — events broadcast to all subscribers |
| Consumers | Exactly one worker per task | Multiple concurrent subscriber groups |
| Delivery target | Specific HTTP endpoint (URL per task or per queue) | Any Pub/Sub subscriber (push or pull) |
| Rate limiting | ✅ Configurable per queue (`maxDispatchesPerSecond`) | ❌ No built-in rate limiting |
| Task deduplication | ✅ By task name (4-hour window) | ❌ No deduplication |
| Scheduled future delivery | ✅ Per-task `scheduleTime` (up to 30 days) | ❌ No per-message scheduling |
| Task visibility & inspection | ✅ List and describe individual tasks | ❌ Messages are ephemeral |
| Acknowledgement model | ✅ Worker returns 2xx (explicit completion) | ✅ Subscriber ACKs message |
| Strict ordering | ❌ No ordering guarantee | ❌ No (except per ordering key) |
| Fan-out | ❌ One handler per task | ✅ Multiple subscriptions per topic |
| Max message/task retention | 30 days | 7 days (standard), 31 days (BigTable storage) |

> 💡 **TIP:** Use Cloud Tasks when you need to **control how fast work is done** (rate limiting), **prevent duplicate work** (task name deduplication), **inspect queued work** (list/describe tasks), or **schedule work for the future** (per-task `scheduleTime`). Use Pub/Sub when you need **fan-out to multiple consumers** or **streaming/analytics pipelines**.

---

### 2.2 CORE CONCEPTS

| Concept | Definition |
|---|---|
| **Queue** | Named resource that buffers tasks and controls dispatch rate, concurrency, and retry behavior |
| **Task** | A unit of work: an HTTP request (method + URL + headers + body) plus optional name and schedule time |
| **Dispatch** | The act of Cloud Tasks sending the task's HTTP request to the target worker |
| **Attempt** | One delivery attempt for a task; a single task may have multiple attempts after failures |
| **scheduleTime** | Future UTC timestamp; task won't dispatch before this time even if queue has capacity |
| **Task name** | Optional unique identifier; enables deduplication — duplicate names return existing task |
| **dispatchDeadline** | Maximum time Cloud Tasks waits for the worker to respond; default 10 min, max 30 min |

---

### 2.3 QUEUE CONFIGURATION & MANAGEMENT

```bash
# ── Create a queue with rate limiting and retry policy ────────────────────────
gcloud tasks queues create my-processing-queue \
  --location=us-central1 \
  --max-dispatches-per-second=100 \      # global rate limit: max 100 tasks/sec
  --max-concurrent-dispatches=50 \       # max 50 tasks in-flight simultaneously
  --max-attempts=5 \                     # 1 initial + 4 retries maximum
  --min-backoff=10s \                    # wait at least 10s before first retry
  --max-backoff=300s \                   # wait at most 300s between retries
  --max-doublings=4 \                    # double backoff 4 times, then stay at max
  --task-ttl=7d \                        # expire tasks after 7 days if not processed
  --max-retry-duration=1h               # stop retrying after 1 hour total

# ── Create a queue with queue-level OIDC auth (all tasks use this SA) ─────────
gcloud tasks queues create authenticated-queue \
  --location=us-central1 \
  --max-dispatches-per-second=10 \
  --http-uri-override=https://my-worker.run.app \
  --http-header-override="X-Environment:production" \
  --http-oauth-token-service-account-email=tasks-sa@PROJECT.iam.gserviceaccount.com

# ── Update queue settings (takes effect immediately, in-flight tasks unaffected)
gcloud tasks queues update my-processing-queue \
  --location=us-central1 \
  --max-dispatches-per-second=200 \
  --max-concurrent-dispatches=100

# ── Pause (stop dispatching; tasks continue to accumulate in the queue) ────────
gcloud tasks queues pause my-processing-queue --location=us-central1

# ── Resume dispatch ────────────────────────────────────────────────────────────
gcloud tasks queues resume my-processing-queue --location=us-central1

# ── Purge all tasks (DESTRUCTIVE — cannot be undone) ─────────────────────────
gcloud tasks queues purge my-processing-queue --location=us-central1

# ── Inspect queue configuration and live stats ────────────────────────────────
gcloud tasks queues describe my-processing-queue --location=us-central1

# ── List all queues in a region ───────────────────────────────────────────────
gcloud tasks queues list --location=us-central1 \
  --format="table(name, state, rateLimits.maxDispatchesPerSecond, retryConfig.maxAttempts)"

# ── Delete a queue (must be empty or purged first) ────────────────────────────
gcloud tasks queues delete my-processing-queue --location=us-central1
```

#### Queue Configuration Parameters

| Parameter | CLI Flag | Default | Description |
|---|---|---|---|
| Max dispatches/sec | `--max-dispatches-per-second` | 500 | Global rate limit — tasks dispatched per second |
| Max concurrent | `--max-concurrent-dispatches` | 1,000 | Maximum tasks in-flight simultaneously |
| Max attempts | `--max-attempts` | unlimited | Total attempt limit (includes initial attempt) |
| Min backoff | `--min-backoff` | 100ms | Minimum wait between retry attempts |
| Max backoff | `--max-backoff` | 3,600s | Maximum wait between retry attempts |
| Max doublings | `--max-doublings` | 16 | Times backoff doubles before capping at max |
| Task TTL | `--task-ttl` | 30 days | Task expires if not processed within this window |
| Max retry duration | `--max-retry-duration` | 0 (unlimited) | Total wall-clock time budget for all retries |

> ⚠️ **NOTE:** `--max-attempts` counts the **first attempt** — setting `--max-attempts=5` means 1 initial attempt + 4 retries. Setting `--max-attempts=1` disables retries entirely, which is useful for idempotency-unsafe operations.

---

### 2.4 CREATING & MANAGING TASKS

```bash
# ── Create an HTTP task with JSON body and OIDC auth ─────────────────────────
gcloud tasks create-http-task \
  --queue=my-processing-queue \
  --location=us-central1 \
  --url=https://my-worker.run.app/process \
  --method=POST \
  --header="Content-Type:application/json" \
  --body-content='{"jobId": "job-123", "userId": "user-456", "action": "export"}' \
  --oidc-service-account-email=tasks-sa@PROJECT.iam.gserviceaccount.com \
  --oidc-token-audience=https://my-worker.run.app

# ── Create a named task (enables deduplication) ───────────────────────────────
# If task "process-order-789" already exists, returns the existing task (no duplicate)
gcloud tasks create-http-task process-order-789 \
  --queue=my-processing-queue \
  --location=us-central1 \
  --url=https://my-worker.run.app/orders \
  --method=POST \
  --body-content='{"orderId": "order-789", "action": "fulfill"}'

# ── Create a task scheduled for future dispatch ───────────────────────────────
# Task enters queue immediately but won't be dispatched until scheduleTime
gcloud tasks create-http-task \
  --queue=my-processing-queue \
  --location=us-central1 \
  --url=https://my-worker.run.app/reminders \
  --method=POST \
  --body-content='{"userId": "user-123", "type": "24h-reminder"}' \
  --schedule-time="2025-04-01T09:00:00Z"

# ── List tasks in a queue ─────────────────────────────────────────────────────
gcloud tasks list \
  --queue=my-processing-queue \
  --location=us-central1 \
  --format="table(name, scheduleTime, status.attemptDispatchCount)"

# ── Describe a specific task ──────────────────────────────────────────────────
gcloud tasks describe TASK_ID \
  --queue=my-processing-queue \
  --location=us-central1

# ── Force a task to dispatch immediately (bypasses scheduleTime) ───────────────
gcloud tasks run TASK_ID \
  --queue=my-processing-queue \
  --location=us-central1

# ── Cancel a task before it's dispatched ────────────────────────────────────
gcloud tasks delete TASK_ID \
  --queue=my-processing-queue \
  --location=us-central1
```

#### Programmatic Task Creation (Python SDK)

The Python SDK is the primary way to enqueue tasks from application code running in Cloud Run, GCE, or Cloud Functions:

```python
from google.cloud import tasks_v2
from google.protobuf import timestamp_pb2
import datetime
import json

def create_http_task(
    project: str,
    location: str,
    queue: str,
    url: str,
    payload: dict,
    task_name: str = None,
    schedule_seconds_from_now: int = 0,
    service_account_email: str = None,
    dispatch_deadline_seconds: int = 600,
) -> tasks_v2.Task:
    """
    Creates an HTTP task in a Cloud Tasks queue.

    Args:
        task_name: If set, used as deduplication key. Named tasks must be unique
                   per queue; duplicate names return the existing task.
        schedule_seconds_from_now: If > 0, task won't dispatch before this delay.
        service_account_email: SA for OIDC token auth. Audience = url.
        dispatch_deadline_seconds: Max time (sec) to wait for worker response.
    """
    client = tasks_v2.CloudTasksClient()
    parent = client.queue_path(project, location, queue)

    # Build the HTTP request object
    http_request = tasks_v2.HttpRequest(
        http_method=tasks_v2.HttpMethod.POST,
        url=url,
        headers={"Content-Type": "application/json"},
        body=json.dumps(payload).encode("utf-8"),
    )

    # Add OIDC authentication if a service account is provided
    if service_account_email:
        http_request.oidc_token = tasks_v2.OidcToken(
            service_account_email=service_account_email,
            audience=url,  # MUST exactly match the URL (no trailing slash)
        )

    task = tasks_v2.Task(
        http_request=http_request,
        dispatch_deadline=datetime.timedelta(seconds=dispatch_deadline_seconds),
    )

    # Set unique task name for deduplication
    if task_name:
        task.name = client.task_path(project, location, queue, task_name)

    # Schedule for future dispatch
    if schedule_seconds_from_now > 0:
        execute_time = (
            datetime.datetime.utcnow()
            + datetime.timedelta(seconds=schedule_seconds_from_now)
        )
        ts = timestamp_pb2.Timestamp()
        ts.FromDatetime(execute_time)
        task.schedule_time = ts

    response = client.create_task(request={"parent": parent, "task": task})
    print(f"Created task: {response.name}")
    return response


# Usage: enqueue a task immediately (dispatch as soon as queue has capacity)
create_http_task(
    project="my-project",
    location="us-central1",
    queue="my-processing-queue",
    url="https://my-worker.run.app/process",
    payload={"orderId": "order-123", "action": "process"},
    task_name="process-order-123",            # deduplication key
    service_account_email="tasks-sa@my-project.iam.gserviceaccount.com",
)

# Usage: schedule a reminder task for 24 hours from now
create_http_task(
    project="my-project",
    location="us-central1",
    queue="reminder-queue",
    url="https://notifications.run.app/send",
    payload={"userId": "user-456", "template": "trial-expiring"},
    task_name=f"trial-reminder-user-456",
    schedule_seconds_from_now=86400,          # 24 hours
    service_account_email="tasks-sa@my-project.iam.gserviceaccount.com",
)
```

---

### 2.5 AUTHENTICATION & IAM

Cloud Tasks supports two authentication modes for HTTP task delivery. Choose based on the target service type.

| Mode | CLI Flag | Target Accepts | Use When |
|---|---|---|---|
| OIDC token | `--oidc-service-account-email` | Google OIDC JWTs | Cloud Run, GKE, any JWT-validating service |
| OAuth2 token | `--oauth-service-account-email` | Google OAuth2 tokens | Cloud Functions 1st gen, App Engine, Google APIs |
| No auth | (omit both) | Any request | Public endpoints, internal services on same VPC |

> ⚠️ **NOTE:** The `--oidc-token-audience` must **exactly match** the URL the task is dispatching to, including scheme and host but excluding query strings. For Cloud Run, the audience should be the service's base URL (e.g., `https://my-worker-abc123-uc.a.run.app`), not the path.

```bash
# ── Create SA for Cloud Tasks dispatch ────────────────────────────────────────
gcloud iam service-accounts create cloud-tasks-dispatcher \
  --display-name="Cloud Tasks Dispatcher SA"

# ── Allow Cloud Tasks service agent to create tokens for the dispatcher SA ────
# (required for OIDC authentication to work)
gcloud iam service-accounts add-iam-policy-binding \
  cloud-tasks-dispatcher@PROJECT.iam.gserviceaccount.com \
  --member="serviceAccount:service-PROJECT_NUMBER@gcp-sa-cloudtasks.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountTokenCreator"

# ── Grant dispatcher SA permission to invoke the Cloud Run worker ─────────────
gcloud run services add-iam-policy-binding my-worker \
  --region=us-central1 \
  --member="serviceAccount:cloud-tasks-dispatcher@PROJECT.iam.gserviceaccount.com" \
  --role="roles/run.invoker"

# ── Grant the producer application permission to enqueue tasks ────────────────
gcloud projects add-iam-policy-binding PROJECT \
  --member="serviceAccount:app-service-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/cloudtasks.enqueuer"
```

#### Cloud Tasks IAM Roles

| Role | Purpose |
|---|---|
| `roles/cloudtasks.admin` | Full queue and task lifecycle management |
| `roles/cloudtasks.enqueuer` | Create tasks in existing queues (for producer apps) |
| `roles/cloudtasks.viewer` | Read-only view of queues and tasks |
| `roles/cloudtasks.queueAdmin` | Create, update, pause, resume, purge queues |
| `roles/run.invoker` | Invoke the Cloud Run worker (grant to the dispatcher SA) |

---

### 2.6 WORKER IMPLEMENTATION PATTERNS

A Cloud Tasks worker is any HTTP service that follows the acknowledgement contract: return 2xx to signal success (task deleted), or return 4xx/5xx to signal failure (task retried per queue policy). Cloud Tasks injects `X-CloudTasks-*` headers into every dispatched request that workers can use for logging, idempotency, and retry-aware logic.

#### X-CloudTasks-* Headers Reference

| Header | Description | Example Value |
|---|---|---|
| `X-CloudTasks-QueueName` | Short name of the queue | `my-processing-queue` |
| `X-CloudTasks-TaskName` | Short name (ID) of the task | `task-abc123def456` |
| `X-CloudTasks-TaskRetryCount` | Number of retries (0 on first attempt) | `2` |
| `X-CloudTasks-TaskExecutionCount` | Number of times task was attempted and received a non-500 response | `0` |
| `X-CloudTasks-TaskETA` | ETA in Unix epoch microseconds | `1712131200000000` |
| `X-CloudTasks-TaskPreviousResponse` | HTTP status code of previous failed attempt | `503` |
| `X-CloudTasks-TaskRetryReason` | Why this retry is happening | `NON_2XX_RESPONSE` |

```python
import os
import json
from flask import Flask, request

app = Flask(__name__)

@app.route("/process", methods=["POST"])
def process_task():
    # ── Read Cloud Tasks metadata headers ──────────────────────────────────────
    task_name       = request.headers.get("X-CloudTasks-TaskName", "unknown")
    retry_count     = int(request.headers.get("X-CloudTasks-TaskRetryCount", "0"))
    queue_name      = request.headers.get("X-CloudTasks-QueueName", "unknown")
    prev_response   = request.headers.get("X-CloudTasks-TaskPreviousResponse", "none")

    print(
        f"Task: {task_name} | Queue: {queue_name} | "
        f"Attempt: {retry_count + 1} | PrevResponse: {prev_response}"
    )

    # ── Idempotency check (use Firestore/Redis in production) ──────────────────
    if is_already_processed(task_name):
        print(f"Task {task_name} already successfully processed — skipping")
        return ("", 204)   # 2xx = success; Cloud Tasks deletes the task

    # ── Parse task payload ─────────────────────────────────────────────────────
    payload  = request.get_json(silent=True) or {}
    order_id = payload.get("orderId")
    action   = payload.get("action", "process")

    if not order_id:
        # Missing required field — permanent failure; return 4xx to STOP retrying
        print(f"ERROR: Missing orderId in payload — abandoning task")
        return ("Missing required field: orderId", 400)

    try:
        execute_business_logic(order_id, action)
        mark_as_processed(task_name)          # record in idempotency store
        return ("", 204)                       # success

    except TemporaryServiceError as e:
        # Transient error (downstream timeout, etc.) — return 5xx to RETRY
        print(f"Transient error processing {order_id}: {e}")
        return (f"Temporary error: {e}", 503)

    except InvalidDataError as e:
        # Permanent error (bad data, business rule violation) — 4xx to STOP
        print(f"Permanent error for {order_id}: {e}")
        return (f"Invalid task data: {e}", 422)

def is_already_processed(task_name: str) -> bool:
    """Check idempotency store. In production: query Firestore, Redis, or Cloud SQL."""
    return False  # placeholder

def mark_as_processed(task_name: str) -> None:
    """Record successful completion in idempotency store."""
    pass

def execute_business_logic(order_id: str, action: str) -> None:
    print(f"Processing order {order_id} with action={action}")

class TemporaryServiceError(Exception): pass
class InvalidDataError(Exception): pass

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    app.run(host="0.0.0.0", port=port)
```

> 💡 **TIP:** Use `X-CloudTasks-TaskName` as your idempotency key — it is stable across retries for the same task and globally unique within a queue. Store processed task names in Firestore (using the task name as the document ID) for a scalable, serverless idempotency store.

---

### 2.7 ADVANCED PATTERNS

#### Task Deduplication

Named tasks provide at-most-once-per-4-hours deduplication. If a task with name `process-order-123` is created while another task with the same name is queued or in-flight, the `CreateTask` call returns the existing task without creating a duplicate. After a task **completes** (2xx returned or max attempts exceeded), the same name can be reused after a **4-hour cooldown window**.

#### Fan-Out Pattern

```text
Trigger (one task) → Fan-out worker → Creates N child tasks in Cloud Tasks queue
                                                   ↓ (each child task)
                                         Item worker (processes one record)
```

Used for: parallel batch processing, splitting large CSV/GCS files into per-row tasks, distributing work across multiple Cloud Run instances with rate control.

#### Task Chaining (Pipeline)

```text
Worker A → processes task → before returning 2xx, creates Task B in next queue
Worker B → processes task → before returning 2xx, creates Task C in next queue
Worker C → final step → returns 2xx
```

Each step creates the next step atomically before acknowledging. If Worker B crashes after creating Task C but before returning 2xx, Task B is retried — resulting in a potential duplicate Task C creation. Use named tasks for each step to handle this safely.

#### Rate-Limited Import / Throttling Pattern

Cloud Tasks acts as a rate-limiting proxy in front of a downstream API or database. Set `maxDispatchesPerSecond` on the queue to the API's rate limit, and all tasks will be dispatched at that ceiling regardless of how fast producers enqueue them.

```bash
# Example: throttle Stripe API calls to 100/sec
gcloud tasks queues create stripe-api-queue \
  --location=us-central1 \
  --max-dispatches-per-second=100 \
  --max-concurrent-dispatches=200 \
  --max-attempts=3
```

#### Delay / Debounce Pattern

```python
from google.cloud import tasks_v2

def debounce_update(user_id: str, delay_seconds: int = 30):
    """
    Schedule a user profile update task, debounced by delay_seconds.
    If called again before dispatch, resets the timer.
    """
    client = tasks_v2.CloudTasksClient()
    parent = client.queue_path("my-project", "us-central1", "update-queue")
    task_name = f"update-user-{user_id}"
    full_task_name = client.task_path("my-project", "us-central1", "update-queue", task_name)

    # Delete existing pending task (if any) — resets the debounce timer
    try:
        client.delete_task(request={"name": full_task_name})
        print(f"Reset debounce timer for {user_id}")
    except Exception:
        pass  # Task didn't exist — that's fine

    # Create new task scheduled delay_seconds from now
    create_http_task(
        project="my-project",
        location="us-central1",
        queue="update-queue",
        url="https://my-worker.run.app/update-user",
        payload={"userId": user_id},
        task_name=task_name,
        schedule_seconds_from_now=delay_seconds,
    )
```

---

### 2.8 LIMITS, QUOTAS & TROUBLESHOOTING

#### Key Limits

| Resource | Limit | Notes |
|---|---|---|
| Queues per project | 1,000 | Soft limit; deletable queues can be recreated after 1 week |
| Tasks per queue | ~1,000,000 | Per-queue storage limit (approximate) |
| Task size | 1 MB | Request headers + body combined |
| Task name length | 500 characters | URL-safe characters only (`[a-zA-Z0-9_-]`) |
| Task retention | 30 days (default) | Configurable with `--task-ttl` |
| Max dispatch rate | 500/sec per queue | Hard max; configurable below this |
| scheduleTime maximum | 30 days from now | Cannot schedule beyond 30-day horizon |
| Deduplication window | 4 hours post-completion | Same task name cannot be reused during window |
| dispatchDeadline max | 30 minutes | Worker must respond within this time |

#### Troubleshooting Decision Tree

```text
Tasks not being dispatched or completing?
│
├── Is the queue PAUSED?
│   gcloud tasks queues describe QUEUE --location=REGION
│   state=PAUSED → gcloud tasks queues resume QUEUE --location=REGION
│
├── Rate limit throttling?
│   → maxDispatchesPerSecond may be set too low for your throughput
│   → Check Cloud Monitoring: cloudtasks.googleapis.com/queue/depth
│   → Increase: gcloud tasks queues update QUEUE --max-dispatches-per-second=N
│
├── Worker returning 5xx (retries in progress)?
│   → Tasks are being retried per backoff policy — this is expected behavior
│   → Inspect worker logs: gcloud logging read 'resource.type="cloud_run_revision"'
│   → Check task retry status: gcloud tasks describe TASK_ID --queue=QUEUE
│   → If all retries exhausted: task moves to DLQ (if configured) or is dropped
│
├── 403 / OIDC auth failure?
│   → Dispatcher SA lacks roles/run.invoker on target Cloud Run service
│   → OIDC audience doesn't exactly match the task URL (check for trailing slashes)
│   → Verify: gcloud run services get-iam-policy WORKER --region=REGION
│   → Fix: gcloud run services add-iam-policy-binding WORKER \
│           --member="serviceAccount:DISPATCHER_SA" --role="roles/run.invoker"
│
├── Task created but shows scheduleTime in the future?
│   → Expected — task is pending scheduled delivery
│   → Force immediate dispatch: gcloud tasks run TASK_ID --queue=QUEUE
│
└── Duplicate tasks being processed?
    → Worker is not idempotent — use X-CloudTasks-TaskName as idempotency key
    → Use a named task (task_name in CreateTask) if producer creates duplicate calls
    → Deduplication only applies within 4-hour window; check task age
```


---

## PART 3 — CLOUD SCHEDULER

Cloud Scheduler is GCP's fully managed enterprise cron service that triggers jobs on a defined recurring schedule using standard unix-cron syntax. It handles WHEN work starts — the actual work, retry logic, and rate control belong to the target service. For anything beyond simple HTTP calls, the recommended pattern is Cloud Scheduler → Cloud Tasks → Worker, which adds queue depth, per-task retry, and rate limiting that Scheduler alone cannot provide.

---

### 3.1 OVERVIEW & ARCHITECTURE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CLOUD SCHEDULER JOB                                  │
│                                                                             │
│  schedule: "0 2 * * *"         timezone: "America/New_York"                │
│  target: HTTP POST             attempt-deadline: 10m                        │
│  retry: max 3 attempts         backoff: 5s–60s exponential                 │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │ Fires at scheduled time
                    ┌────────────────┼────────────────────┐
                    │                │                    │
          ┌─────────▼──────┐ ┌───────▼───────┐ ┌─────────▼──────────┐
          │  HTTP TARGET   │ │  PUB/SUB      │ │  APP ENGINE HTTP   │
          │                │ │  TARGET       │ │  TARGET (legacy)   │
          │  Cloud Run     │ │               │ │                    │
          │  GKE service   │ │  Publish      │ │  /cron/my-handler  │
          │  External HTTPS│ │  message to   │ │                    │
          │  (OIDC/OAuth2) │ │  topic        │ │                    │
          └────────────────┘ └───────────────┘ └────────────────────┘
                    │
                    │  ◄── For production workloads, prefer:
                    ▼
          ┌──────────────────────────────────────────────────┐
          │  RECOMMENDED PATTERN                             │
          │                                                  │
          │  Cloud Scheduler                                 │
          │       ↓ (HTTP POST to fan-out Cloud Run)         │
          │  Cloud Run fan-out handler                       │
          │       ↓ (creates N tasks in Cloud Tasks)         │
          │  Cloud Tasks queue (rate-limited)                │
          │       ↓ (dispatches to workers at controlled rate)│
          │  Cloud Run worker                                │
          └──────────────────────────────────────────────────┘

  Cloud Monitoring:
  cloudscheduler.googleapis.com/job/attempt_count
  cloudscheduler.googleapis.com/job/completed_job_count
  cloudscheduler.googleapis.com/job/failed_job_count
  cloudscheduler.googleapis.com/job/last_completed_time
```

> 💡 **TIP:** Cloud Scheduler's retry policy is per-job-execution, not per-item. If a job fires once and the target fails, Scheduler retries the entire trigger (up to 5 times). For item-level retries, use Cloud Tasks as the intermediate layer — Scheduler enqueues one task per batch item, and Cloud Tasks handles per-item retry with backoff.

---

### 3.2 JOB CONFIGURATION — FULL CLI REFERENCE

```bash
# ── Enable required API ────────────────────────────────────────────────────────
gcloud services enable cloudscheduler.googleapis.com

# ── Create an HTTP target job (Cloud Run, external HTTPS) ─────────────────────
gcloud scheduler jobs create http my-daily-report \
  --location=us-central1 \
  --schedule="0 9 * * 1-5" \           # weekdays at 09:00 in specified timezone
  --uri=https://my-worker.run.app/reports/generate \
  --http-method=POST \
  --headers="Content-Type=application/json,X-Triggered-By=scheduler" \
  --message-body='{"reportType": "daily", "format": "pdf", "env": "prod"}' \
  --oidc-service-account-email=scheduler-sa@PROJECT.iam.gserviceaccount.com \
  --oidc-token-audience=https://my-worker.run.app \
  --time-zone="America/New_York" \
  --attempt-deadline=10m \             # time allowed for worker to respond
  --max-retry-attempts=3 \             # retry up to 3 times on failure
  --min-backoff=5s \                   # wait 5s before first retry
  --max-backoff=60s \                  # wait at most 60s between retries
  --max-doublings=3 \                  # double backoff 3 times
  --description="Weekday morning PDF report generation"

# ── Create a Pub/Sub target job ────────────────────────────────────────────────
gcloud scheduler jobs create pubsub heartbeat-job \
  --location=us-central1 \
  --schedule="*/15 * * * *" \          # every 15 minutes
  --topic=projects/PROJECT/topics/system-heartbeat \
  --message-body='{"event": "heartbeat", "source": "scheduler"}' \
  --attributes="source=scheduler,environment=prod,version=1" \
  --time-zone="UTC" \
  --description="System heartbeat every 15 minutes"

# ── Create an App Engine HTTP target job (legacy) ─────────────────────────────
gcloud scheduler jobs create app-engine midnight-cleanup \
  --location=us-central1 \
  --schedule="0 0 * * *" \
  --relative-url="/cron/cleanup?daysOld=30" \
  --http-method=GET \
  --time-zone="UTC"

# ── Update an existing job (can update schedule, URI, body, retry, timezone) ───
gcloud scheduler jobs update http my-daily-report \
  --location=us-central1 \
  --schedule="0 8 * * 1-5" \
  --time-zone="Europe/London" \
  --max-retry-attempts=5

# ── Manually trigger a job immediately (ignores schedule — for testing) ────────
gcloud scheduler jobs run my-daily-report --location=us-central1

# ── Pause a job (stops scheduled firing; manual trigger still works) ──────────
gcloud scheduler jobs pause my-daily-report --location=us-central1

# ── Resume a paused job ────────────────────────────────────────────────────────
gcloud scheduler jobs resume my-daily-report --location=us-central1

# ── Inspect a job (shows state, last attempt time, next scheduled time) ────────
gcloud scheduler jobs describe my-daily-report --location=us-central1

# ── List all jobs in a region ─────────────────────────────────────────────────
gcloud scheduler jobs list --location=us-central1 \
  --format="table(name, state, schedule, timeZone, lastAttemptTime)"

# ── Delete a job ───────────────────────────────────────────────────────────────
gcloud scheduler jobs delete my-daily-report --location=us-central1
```

---

### 3.3 CRON EXPRESSION REFERENCE

Cloud Scheduler uses standard unix-cron format: five space-separated fields evaluated in the specified timezone. The minimum interval is 1 minute — sub-minute scheduling is not supported.

#### Cron Field Reference

| Field | Values | Special Characters |
|---|---|---|
| Minute | 0–59 | `*` (any), `,` (list), `-` (range), `/` (step) |
| Hour | 0–23 | `*`, `,`, `-`, `/` |
| Day of Month | 1–31 | `*`, `,`, `-`, `/`, `?` (ignored when DOW set) |
| Month | 1–12 or `JAN`–`DEC` | `*`, `,`, `-`, `/` |
| Day of Week | 0–6 or `SUN`–`SAT` (0 = Sunday) | `*`, `,`, `-`, `/`, `?` |

#### Common Cron Expressions

| Expression | Description | Notes |
|---|---|---|
| `* * * * *` | Every minute | Maximum frequency; use with caution |
| `*/5 * * * *` | Every 5 minutes | Step syntax — every N minutes |
| `*/15 * * * *` | Every 15 minutes | Common for heartbeats and polling |
| `0 * * * *` | Every hour on the hour | Hourly jobs |
| `0 9 * * *` | Daily at 09:00 | Always specify `--time-zone` for local time |
| `0 9 * * 1-5` | Weekdays (Mon–Fri) at 09:00 | 1=Monday, 5=Friday |
| `0 9 * * MON-FRI` | Weekdays at 09:00 | Named days equivalent |
| `0 0 1 * *` | First of every month at midnight | Monthly jobs |
| `0 0 1 1 *` | January 1st at midnight | Annual jobs |
| `0 */6 * * *` | Every 6 hours | 00:00, 06:00, 12:00, 18:00 |
| `30 8 * * MON` | Every Monday at 08:30 | Weekly reports |
| `0 0 * * 0` | Every Sunday at midnight | Weekly cleanup |
| `0 2 * * *` | Daily at 02:00 (off-peak) | Batch processing during low traffic |
| `0 8-18 * * 1-5` | Every hour, 08:00–18:00, weekdays | Business hours monitoring |

#### Timezone Handling

Cloud Scheduler uses IANA timezone database names. The `--time-zone` flag accepts any valid IANA name:

```bash
# Common IANA timezone examples:
# UTC, America/New_York, America/Chicago, America/Los_Angeles
# Europe/London, Europe/Paris, Europe/Berlin
# Asia/Tokyo, Asia/Kolkata, Asia/Shanghai, Asia/Singapore
# Australia/Sydney, Pacific/Auckland

# DST behavior: Cloud Scheduler automatically adjusts for daylight saving time.
# A job scheduled for "0 9 * * *" in "America/New_York" fires at 09:00 Eastern
# whether that's UTC-4 (EDT) or UTC-5 (EST) — Scheduler handles the offset change.

# ⚠️ Exception: jobs scheduled at 02:00–03:00 in DST-observing zones may
# fire twice (spring) or be skipped (fall) due to clock changes.
gcloud scheduler jobs create http dst-safe-job \
  --schedule="0 10 * * *" \   # schedule away from 02:00–03:00 DST window
  --time-zone="America/New_York" \
  --uri=https://my-worker.run.app/daily
```

---

### 3.4 AUTHENTICATION & IAM

Cloud Scheduler uses the same OIDC/OAuth2 token mechanisms as Cloud Tasks for HTTP targets. The scheduler's service account must have the invoker role on the target service.

```bash
# ── Create a dedicated service account for Scheduler ─────────────────────────
gcloud iam service-accounts create cloud-scheduler-sa \
  --display-name="Cloud Scheduler Service Account"

# ── Grant Cloud Run invoker role (for HTTP targets pointing to Cloud Run) ─────
gcloud run services add-iam-policy-binding my-worker \
  --region=us-central1 \
  --member="serviceAccount:cloud-scheduler-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/run.invoker"

# ── Grant Pub/Sub publisher role (for Pub/Sub target jobs) ───────────────────
gcloud pubsub topics add-iam-policy-binding my-topic \
  --member="serviceAccount:cloud-scheduler-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"

# ── Grant permission to manage Scheduler jobs (for developers) ────────────────
gcloud projects add-iam-policy-binding PROJECT \
  --member="user:developer@example.com" \
  --role="roles/cloudscheduler.admin"

# ── Grant permission to manually trigger jobs from CI/CD ─────────────────────
gcloud projects add-iam-policy-binding PROJECT \
  --member="serviceAccount:cicd-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/cloudscheduler.jobRunner"
```

#### Cloud Scheduler IAM Roles

| Role | Purpose |
|---|---|
| `roles/cloudscheduler.admin` | Create, update, delete, pause, resume, and manually run jobs |
| `roles/cloudscheduler.viewer` | Read-only view of jobs and execution history |
| `roles/cloudscheduler.jobRunner` | Manually trigger jobs (`run` command) — for CI/CD pipelines |
| `roles/run.invoker` | Grant to scheduler SA for Cloud Run HTTP targets |
| `roles/pubsub.publisher` | Grant to scheduler SA for Pub/Sub topic targets |

---

### 3.5 RECOMMENDED PATTERN — SCHEDULER → TASKS → WORKER

Direct Scheduler → Cloud Run is appropriate for simple, low-volume triggers (send one HTTP request, get one 200 OK). For any production workload involving multiple items, variable processing time, or rate-limited downstream calls, the three-tier pattern provides critical advantages.

#### Why Direct Scheduler → Cloud Run Falls Short

| Limitation | Direct Scheduler → Cloud Run | Scheduler → Tasks → Worker |
|---|---|---|
| Retry granularity | Retries the entire job trigger | Retries individual failed items |
| Rate limiting | None | Per-queue `maxDispatchesPerSecond` |
| Concurrency control | None | Per-queue `maxConcurrentDispatches` |
| Task inspection | Cannot see what's queued | List/describe individual tasks |
| Backpressure | Cloud Run scales unboundedly | Queue buffers and paces work |
| Max execution time | Limited by `attempt-deadline` (30 min) | Cloud Run max 60 min per request |

#### Implementation

```bash
# Step 1: Create the Cloud Tasks queue with controlled throughput
gcloud tasks queues create batch-processing-queue \
  --location=us-central1 \
  --max-dispatches-per-second=20 \      # process 20 items/second max
  --max-concurrent-dispatches=10 \      # max 10 items in-flight
  --max-attempts=5 \
  --min-backoff=30s \
  --max-backoff=600s

# Step 2: Deploy the fan-out Cloud Run service
# (This service receives the Scheduler trigger and creates N Cloud Tasks)
# Pseudocode for the fan-out handler:
#   1. Read list of items from GCS/BigQuery/Firestore
#   2. For each item: create one task in batch-processing-queue
#   3. Return 200 immediately (enqueuing is fast; processing is async)

# Step 3: Create the Scheduler job pointing to the fan-out service
gcloud scheduler jobs create http nightly-batch-trigger \
  --location=us-central1 \
  --schedule="0 2 * * *" \             # 2:00 AM UTC nightly
  --uri=https://batch-fanout.run.app/start-batch \
  --http-method=POST \
  --message-body='{"batchDate": "today", "maxItems": 10000}' \
  --oidc-service-account-email=scheduler-sa@PROJECT.iam.gserviceaccount.com \
  --oidc-token-audience=https://batch-fanout.run.app \
  --time-zone="UTC" \
  --attempt-deadline=5m \              # fan-out handler just enqueues; should be fast
  --max-retry-attempts=2 \
  --description="Nightly batch processing via Cloud Tasks"
```

---

### 3.6 MONITORING JOB EXECUTION

```bash
# ── Describe a job — shows last run status, next scheduled time ───────────────
gcloud scheduler jobs describe my-daily-report \
  --location=us-central1
# Key fields:
#   state: ENABLED | PAUSED | DISABLED
#   status.code: 0=success, non-zero=last failure code
#   lastAttemptTime: RFC3339 timestamp of last trigger
#   scheduleTime: next scheduled fire time

# ── Get execution history via Cloud Logging ───────────────────────────────────
gcloud logging read \
  'resource.type="cloud_scheduler_job"
   AND resource.labels.job_id="my-daily-report"
   AND resource.labels.location="us-central1"' \
  --project=PROJECT \
  --freshness=48h \
  --format="table(timestamp, jsonPayload.status, jsonPayload.description)"

# ── Filter for failed executions only ────────────────────────────────────────
gcloud logging read \
  'resource.type="cloud_scheduler_job"
   AND resource.labels.job_id="my-daily-report"
   AND jsonPayload.status!="SUCCESS"' \
  --project=PROJECT \
  --freshness=7d \
  --format=json | jq '.[] | {time: .timestamp, status: .jsonPayload.status, desc: .jsonPayload.description}'

# ── View all scheduler job logs across all jobs ───────────────────────────────
gcloud logging read \
  'resource.type="cloud_scheduler_job"' \
  --project=PROJECT \
  --freshness=24h \
  --format="table(timestamp, resource.labels.job_id, jsonPayload.status)"
```

#### Cloud Monitoring Metrics for Cloud Scheduler

| Metric | Description | Recommended Alert |
|---|---|---|
| `cloudscheduler.googleapis.com/job/attempt_count` | Total execution attempts | Informational baseline |
| `cloudscheduler.googleapis.com/job/completed_job_count` | Successful completions | Alert if drops to 0 for 2+ consecutive cycles |
| `cloudscheduler.googleapis.com/job/failed_job_count` | Failed executions | Alert if > 0 for critical jobs |
| `cloudscheduler.googleapis.com/job/last_completed_time` | Timestamp of last success | Alert if stale beyond 2× expected interval |

> 💡 **TIP:** Create a Cloud Monitoring alert on `last_completed_time` that fires if a job hasn't succeeded in 2× its expected interval. This catches jobs that run but consistently fail, jobs that stop firing (paused accidentally), and jobs where the cron expression is misconfigured.

---

### 3.7 LIMITS, QUOTAS & TROUBLESHOOTING

#### Key Limits

| Resource | Limit | Notes |
|---|---|---|
| Jobs per project per region | 500 | Soft limit; request increase via Cloud Console |
| Minimum schedule frequency | Every 1 minute | `* * * * *` — no sub-minute scheduling |
| Maximum job body size | 100 KB | For both HTTP and Pub/Sub targets |
| Attempt deadline | 30 minutes max | Worker must respond within this window |
| Max retry attempts | 5 per job execution | Per-schedule-fire; not total retries ever |
| Job description length | 499 characters | |
| Missed fires recovery | At-most-once | Missed executions are NOT backfilled |
| Timezone support | Full IANA database | All standard IANA timezone names supported |

> ⚠️ **NOTE:** Cloud Scheduler guarantees **at-most-once** execution per schedule slot. If a scheduled fire is missed (e.g., due to a GCP outage), it is **not** retried in the next slot. For workloads that cannot miss a run, implement a "catch-up" mechanism in your worker that checks for and processes any missed windows.

#### Troubleshooting Decision Tree

```text
Cloud Scheduler job not behaving as expected?
│
├── Job state is PAUSED?
│   gcloud scheduler jobs describe JOB --location=REGION | grep state
│   Fix: gcloud scheduler jobs resume JOB --location=REGION
│
├── Job fires but target returns an error?
│   gcloud logging read 'resource.type="cloud_scheduler_job"
│     AND resource.labels.job_id="JOB"' --freshness=1d
│   │
│   ├── HTTP 403 (Forbidden)
│   │   → Scheduler SA lacks roles/run.invoker on target Cloud Run service
│   │   → Fix: gcloud run services add-iam-policy-binding TARGET \
│   │           --member="serviceAccount:SCHEDULER_SA" --role="roles/run.invoker"
│   │
│   ├── HTTP 404 (Not Found)
│   │   → URI is wrong or Cloud Run service was deleted/renamed
│   │   → Verify: gcloud run services list --region=REGION
│   │   → Update URI: gcloud scheduler jobs update http JOB --uri=CORRECT_URL
│   │
│   └── HTTP 5xx (Server Error)
│       → Target service is failing — check target service logs separately
│       → Scheduler will retry up to --max-retry-attempts times per execution
│
├── Job doesn't fire at expected local time?
│   ├── Timezone mismatch:
│   │   gcloud scheduler jobs describe JOB | grep timeZone
│   │   → Fix: gcloud scheduler jobs update http JOB --time-zone="CORRECT_TZ"
│   │
│   ├── Job created AFTER the cron slot in the current cycle?
│   │   → First fire will be at the NEXT matching cron slot, not retroactively
│   │
│   └── DST transition (02:00–03:00 window)?
│       → Schedule jobs outside 02:00–03:00 in DST-observing timezones
│       → Or use UTC to avoid DST issues entirely
│
├── OIDC token rejected by Cloud Run (401/403 on token validation)?
│   → --oidc-token-audience must EXACTLY match the Cloud Run service URL
│   → No trailing slash: https://my-service.run.app (not https://my-service.run.app/)
│   → Verify audience: gcloud scheduler jobs describe JOB | grep oidcToken
│
└── Job was manually triggered (gcloud scheduler jobs run) but seems to have no effect?
    → Check that the target service URL is reachable and returning 2xx
    → Manual trigger fires synchronously but still respects attempt-deadline
    → Check Cloud Logging for the specific execution: filter by timestamp
```


---

## PART 4 — CROSS-SERVICE PATTERNS & DECISION GUIDE

Eventarc, Cloud Tasks, and Cloud Scheduler solve different problems in the event-driven and asynchronous space — but they compose naturally. Most production architectures use all three together: Scheduler for the WHEN, Eventarc for reactive triggers, and Cloud Tasks for controlled execution of the WHAT.

---

### 4.1 CHOOSING THE RIGHT SERVICE

#### Master Decision Matrix

| Requirement | Eventarc | Cloud Tasks | Cloud Scheduler |
|---|---|---|---|
| React to a GCP event (file upload, DB write) | ✅ **Best** | ❌ | ❌ |
| React to any GCP API call | ✅ (Audit Log trigger) | ❌ | ❌ |
| Fan-out to multiple consumers on same event | ✅ (Pub/Sub-backed trigger) | ❌ | ❌ |
| Run a job at a recurring schedule (cron) | ❌ | ❌ | ✅ **Best** |
| Rate-limit async work items | ❌ | ✅ **Best** | ❌ |
| Deduplicate work items | ❌ | ✅ (task name) | ❌ |
| Schedule a single task for future dispatch | ❌ | ✅ (`scheduleTime`) | ❌ (recurring only) |
| Retry individual failed work items with backoff | ❌ | ✅ **Best** | ⚠️ (limited, per-job) |
| Control worker concurrency | ❌ | ✅ | ❌ |
| Emit/receive custom domain events | ✅ (Channels) | ❌ | ❌ |
| Inspect queued/pending work | ❌ | ✅ | ❌ |
| At-least-once guaranteed delivery | ✅ | ✅ | ✅ |
| At-most-once per schedule slot | ❌ | ❌ | ✅ |
| Approximate deduplication (per-item) | ❌ | ✅ (task name, 4h window) | ❌ |

---

### 4.2 COMMON COMBINED ARCHITECTURE PATTERNS

#### Pattern 1 — Event-Driven Rate-Limited Pipeline

```text
GCS File Upload
      │
      ▼ (sub-second latency)
Eventarc Trigger (Storage finalized)
      │
      ▼ HTTP POST (CloudEvent)
Cloud Run — Fan-out Handler
      │ (creates 1 task per file record)
      ▼
Cloud Tasks Queue (maxDispatchesPerSecond=50)
      │ (dispatches at controlled rate)
      ▼
Cloud Run — Worker (processes individual records)
```

**Use case**: User uploads a CSV with 10,000 rows. Eventarc fires immediately on upload. The fan-out handler creates 10,000 tasks. Cloud Tasks dispatches them at 50/sec, preventing overwhelming the downstream database. Each task is independently retriable.

#### Pattern 2 — Scheduled Batch with Fan-Out

```text
Cloud Scheduler (cron: "0 2 * * *", UTC)
      │ HTTP POST
      ▼
Cloud Run — Batch Orchestrator
      │ (reads list of items from BigQuery/GCS/Firestore)
      │ (creates N tasks in Cloud Tasks)
      ▼
Cloud Tasks Queue (rate-limited, retriable)
      │
      ▼
Cloud Run — Item Processor (processes one item per task)
      │
      ▼
BigQuery / Cloud SQL / Pub/Sub (output)
```

**Use case**: Nightly reconciliation job, hourly data export, daily report generation for N customers. Scheduler fires once; Cloud Tasks fans out and handles per-item retry.

#### Pattern 3 — Event → Orchestrated Multi-Step Workflow

```text
Any GCP Event (Storage, Audit Log, Pub/Sub)
      │
      ▼ Eventarc Trigger
Google Cloud Workflows (orchestration)
      │
      ├── Step 1: Cloud Run (validate & transform data)
      │
      ├── Step 2: Cloud Tasks (async write to rate-limited API)
      │            └── Cloud Run worker
      │
      ├── Step 3: BigQuery (load transformed data)
      │
      └── Step 4: Pub/Sub (notify downstream consumers)
```

**Use case**: Complex data pipeline triggered by a source event; each step must complete before the next begins; Cloud Workflows provides orchestration with retries and error handling at the workflow level.

#### Pattern 4 — Debounced Task Execution

```text
User activity (multiple rapid events)
      │
      ▼ (each event calls debounce function)
Cloud Tasks — Named Task (scheduleTime=now+30s)
      │
      ├── Event fires again within 30s?
      │   → Delete existing task → Create new task (timer reset)
      │
      └── 30s of silence?
          ↓
     Task dispatched to worker (processes only the final state)
```

**Use case**: User edits a document rapidly; debounce ensures the save/sync operation fires once after they stop typing, not on every keystroke. Named tasks + `scheduleTime` implement this without any additional infrastructure.

#### Pattern 5 — Scheduled Heartbeat with Alert

```text
Cloud Scheduler (every 5 minutes)
      │ Pub/Sub target
      ▼
Pub/Sub topic "system-heartbeat"
      │
      ├── Cloud Function subscriber (updates last-seen timestamp in Firestore)
      │
      └── Cloud Monitoring alert:
          If no message received in 10 minutes → alert on-call
```

**Use case**: Liveness monitoring. If Scheduler itself fails or the pipeline breaks, the heartbeat stops and the alert fires. Simpler and cheaper than a dedicated uptime-check service.

---

### 4.3 IAM ROLES CONSOLIDATED REFERENCE

| Service | Role | Recommended Principal | Purpose |
|---|---|---|---|
| Eventarc | `roles/eventarc.admin` | Developer, Platform SA | Create, manage, delete triggers and channels |
| Eventarc | `roles/eventarc.viewer` | Operator, SRE | Read-only view of triggers and channels |
| Eventarc | `roles/eventarc.eventReceiver` | Trigger SA | **Required** — receive and forward events |
| Eventarc | `roles/eventarc.serviceAgent` | GCP service agent | GCP-managed internal routing (auto-assigned) |
| Eventarc | `roles/eventarc.eventPublisher` | Publisher app SA | Publish to Eventarc channels |
| Cloud Tasks | `roles/cloudtasks.admin` | Developer, Platform SA | Full queue and task lifecycle management |
| Cloud Tasks | `roles/cloudtasks.enqueuer` | Producer app SA | Create tasks in queues |
| Cloud Tasks | `roles/cloudtasks.viewer` | Operator, SRE | Read-only view of queues and tasks |
| Cloud Tasks | `roles/cloudtasks.queueAdmin` | Platform SA | Manage queues (create, pause, purge) |
| Cloud Scheduler | `roles/cloudscheduler.admin` | Developer | Create, update, delete, manually trigger jobs |
| Cloud Scheduler | `roles/cloudscheduler.viewer` | Operator, SRE | Read-only view of job configs and history |
| Cloud Scheduler | `roles/cloudscheduler.jobRunner` | CI/CD SA | Manually trigger jobs (`run` command only) |
| **Shared** | `roles/run.invoker` | Trigger SA / Dispatcher SA / Scheduler SA | Invoke Cloud Run target (grant on service) |
| **Shared** | `roles/pubsub.publisher` | Scheduler SA, Eventarc agent | Publish to Pub/Sub topics |
| **Shared** | `roles/pubsub.subscriber` | Trigger SA | Subscribe to Pub/Sub topics (Eventarc-backed) |
| **Shared** | `roles/iam.serviceAccountTokenCreator` | Service agents | Generate OIDC/OAuth2 tokens for delivery |
| **Shared** | `roles/workflows.invoker` | Trigger SA | Start Workflows executions (Eventarc → Workflows) |

---

### 4.4 QUICK REFERENCE COMMAND CHEATSHEET

```bash
# ════════════════════════════════════════════════════════════════════════════════
# EVENTARC
# ════════════════════════════════════════════════════════════════════════════════

# Enable APIs
gcloud services enable eventarc.googleapis.com run.googleapis.com workflows.googleapis.com

# ── Triggers ──────────────────────────────────────────────────────────────────

# Create: Storage finalized → Cloud Run
gcloud eventarc triggers create TRIGGER_NAME \
  --location=REGION \
  --destination-run-service=SERVICE_NAME \
  --destination-run-region=REGION \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=BUCKET_NAME" \
  --service-account=SA@PROJECT.iam.gserviceaccount.com

# Create: Pub/Sub → Cloud Run
gcloud eventarc triggers create TRIGGER_NAME \
  --location=REGION \
  --destination-run-service=SERVICE_NAME \
  --destination-run-region=REGION \
  --event-filters="type=google.cloud.pubsub.topic.v1.messagePublished" \
  --transport-topic=projects/PROJECT/topics/TOPIC \
  --service-account=SA@PROJECT.iam.gserviceaccount.com

# Create: Audit Log → Cloud Run
gcloud eventarc triggers create TRIGGER_NAME \
  --location=REGION \
  --destination-run-service=SERVICE_NAME \
  --destination-run-region=REGION \
  --event-filters="type=google.cloud.audit.log.v1.written" \
  --event-filters="serviceName=compute.googleapis.com" \
  --event-filters="methodName=v1.compute.instances.insert" \
  --service-account=SA@PROJECT.iam.gserviceaccount.com

# Create: → GKE service
gcloud eventarc triggers create TRIGGER_NAME \
  --location=REGION \
  --destination-gke-cluster=CLUSTER \
  --destination-gke-location=REGION \
  --destination-gke-namespace=NAMESPACE \
  --destination-gke-service=SERVICE \
  --destination-gke-path=/events \
  --event-filters="type=EVENT_TYPE" \
  --service-account=SA@PROJECT.iam.gserviceaccount.com

# Create: → Cloud Workflows
gcloud eventarc triggers create TRIGGER_NAME \
  --location=REGION \
  --destination-workflow=WORKFLOW_NAME \
  --destination-workflow-location=REGION \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=BUCKET" \
  --service-account=SA@PROJECT.iam.gserviceaccount.com

# List / Describe / Delete
gcloud eventarc triggers list --location=REGION
gcloud eventarc triggers describe TRIGGER --location=REGION
gcloud eventarc triggers delete TRIGGER --location=REGION

# ── Channels ──────────────────────────────────────────────────────────────────
gcloud eventarc channels create CHANNEL --location=REGION
gcloud eventarc channels describe CHANNEL --location=REGION
gcloud eventarc channels list --location=REGION
gcloud eventarc channels delete CHANNEL --location=REGION

# Add IAM publisher binding on channel
gcloud eventarc channels add-iam-policy-binding CHANNEL \
  --location=REGION \
  --member="serviceAccount:SA@PROJECT.iam.gserviceaccount.com" \
  --role="roles/eventarc.eventPublisher"

# Publish custom event to channel
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/cloudevents+json" \
  -d '{"specversion":"1.0","type":"com.myapp.event.v1","source":"//myapp","id":"evt-001","datacontenttype":"application/json","data":{}}' \
  "https://eventarc.googleapis.com/v1/projects/PROJECT/locations/REGION/channels/CHANNEL:publishEvents"

# ── IAM ───────────────────────────────────────────────────────────────────────
gcloud iam service-accounts create eventarc-trigger-sa
gcloud projects add-iam-policy-binding PROJECT \
  --member="serviceAccount:eventarc-trigger-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/eventarc.eventReceiver"
gcloud run services add-iam-policy-binding SERVICE \
  --region=REGION \
  --member="serviceAccount:eventarc-trigger-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/run.invoker"


# ════════════════════════════════════════════════════════════════════════════════
# CLOUD TASKS
# ════════════════════════════════════════════════════════════════════════════════

gcloud services enable cloudtasks.googleapis.com

# ── Queue management ──────────────────────────────────────────────────────────
gcloud tasks queues create QUEUE \
  --location=REGION \
  --max-dispatches-per-second=100 \
  --max-concurrent-dispatches=50 \
  --max-attempts=5 \
  --min-backoff=10s \
  --max-backoff=300s \
  --task-ttl=7d

gcloud tasks queues update QUEUE --location=REGION \
  --max-dispatches-per-second=200
gcloud tasks queues describe QUEUE --location=REGION
gcloud tasks queues list --location=REGION
gcloud tasks queues pause QUEUE --location=REGION
gcloud tasks queues resume QUEUE --location=REGION
gcloud tasks queues purge QUEUE --location=REGION       # DESTRUCTIVE
gcloud tasks queues delete QUEUE --location=REGION

# ── Task operations ───────────────────────────────────────────────────────────
# Create HTTP task (anonymous)
gcloud tasks create-http-task \
  --queue=QUEUE --location=REGION \
  --url=https://WORKER_URL/path \
  --method=POST \
  --body-content='{"key": "value"}'

# Create HTTP task with OIDC auth
gcloud tasks create-http-task \
  --queue=QUEUE --location=REGION \
  --url=https://WORKER_URL/path \
  --method=POST \
  --body-content='{"key": "value"}' \
  --oidc-service-account-email=SA@PROJECT.iam.gserviceaccount.com \
  --oidc-token-audience=https://WORKER_URL

# Create named task (deduplication)
gcloud tasks create-http-task UNIQUE_TASK_ID \
  --queue=QUEUE --location=REGION \
  --url=https://WORKER_URL/path \
  --body-content='{"orderId": "123"}'

# Create scheduled future task
gcloud tasks create-http-task \
  --queue=QUEUE --location=REGION \
  --url=https://WORKER_URL/path \
  --body-content='{"type": "reminder"}' \
  --schedule-time="2025-12-01T09:00:00Z"

gcloud tasks list --queue=QUEUE --location=REGION
gcloud tasks describe TASK_ID --queue=QUEUE --location=REGION
gcloud tasks run TASK_ID --queue=QUEUE --location=REGION     # force immediate dispatch
gcloud tasks delete TASK_ID --queue=QUEUE --location=REGION  # cancel task

# ── IAM ───────────────────────────────────────────────────────────────────────
gcloud iam service-accounts create cloud-tasks-dispatcher
gcloud iam service-accounts add-iam-policy-binding \
  cloud-tasks-dispatcher@PROJECT.iam.gserviceaccount.com \
  --member="serviceAccount:service-PROJECT_NUMBER@gcp-sa-cloudtasks.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountTokenCreator"
gcloud run services add-iam-policy-binding WORKER \
  --region=REGION \
  --member="serviceAccount:cloud-tasks-dispatcher@PROJECT.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
gcloud projects add-iam-policy-binding PROJECT \
  --member="serviceAccount:APP_SA@PROJECT.iam.gserviceaccount.com" \
  --role="roles/cloudtasks.enqueuer"


# ════════════════════════════════════════════════════════════════════════════════
# CLOUD SCHEDULER
# ════════════════════════════════════════════════════════════════════════════════

gcloud services enable cloudscheduler.googleapis.com

# ── Job management ────────────────────────────────────────────────────────────
# Create HTTP job with OIDC auth
gcloud scheduler jobs create http JOB_NAME \
  --location=REGION \
  --schedule="0 9 * * 1-5" \
  --uri=https://SERVICE_URL/endpoint \
  --http-method=POST \
  --headers="Content-Type=application/json" \
  --message-body='{"key": "value"}' \
  --oidc-service-account-email=SA@PROJECT.iam.gserviceaccount.com \
  --oidc-token-audience=https://SERVICE_URL \
  --time-zone="America/New_York" \
  --attempt-deadline=10m \
  --max-retry-attempts=3

# Create Pub/Sub job
gcloud scheduler jobs create pubsub JOB_NAME \
  --location=REGION \
  --schedule="*/15 * * * *" \
  --topic=projects/PROJECT/topics/TOPIC \
  --message-body='{"trigger": "scheduled"}' \
  --attributes="source=scheduler" \
  --time-zone="UTC"

# Update job
gcloud scheduler jobs update http JOB_NAME \
  --location=REGION \
  --schedule="0 8 * * *" \
  --time-zone="Europe/London"

# Operations
gcloud scheduler jobs run JOB_NAME --location=REGION        # manual trigger
gcloud scheduler jobs pause JOB_NAME --location=REGION
gcloud scheduler jobs resume JOB_NAME --location=REGION
gcloud scheduler jobs describe JOB_NAME --location=REGION
gcloud scheduler jobs list --location=REGION \
  --format="table(name, state, schedule, timeZone, lastAttemptTime)"
gcloud scheduler jobs delete JOB_NAME --location=REGION

# ── Execution history via Cloud Logging ───────────────────────────────────────
# All executions for a job
gcloud logging read \
  'resource.type="cloud_scheduler_job"
   AND resource.labels.job_id="JOB_NAME"
   AND resource.labels.location="REGION"' \
  --project=PROJECT --freshness=48h \
  --format="table(timestamp, jsonPayload.status)"

# Failed executions only
gcloud logging read \
  'resource.type="cloud_scheduler_job"
   AND resource.labels.job_id="JOB_NAME"
   AND jsonPayload.status!="SUCCESS"' \
  --project=PROJECT --freshness=7d

# ── IAM ───────────────────────────────────────────────────────────────────────
gcloud iam service-accounts create cloud-scheduler-sa
gcloud run services add-iam-policy-binding SERVICE \
  --region=REGION \
  --member="serviceAccount:cloud-scheduler-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
gcloud pubsub topics add-iam-policy-binding TOPIC \
  --member="serviceAccount:cloud-scheduler-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"
gcloud projects add-iam-policy-binding PROJECT \
  --member="user:DEVELOPER@example.com" \
  --role="roles/cloudscheduler.admin"
gcloud projects add-iam-policy-binding PROJECT \
  --member="serviceAccount:CICD_SA@PROJECT.iam.gserviceaccount.com" \
  --role="roles/cloudscheduler.jobRunner"
```

