# 🔄 GCP Pub/Sub — Comprehensive Cheatsheet

> **Audience:** Backend engineers, data engineers, platform engineers, and architects building event-driven systems on GCP.
> **Last updated:** March 2026 | Covers Standard Pub/Sub, Pub/Sub Lite, schemas, exactly-once delivery, and all integration patterns.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Topics](#2-topics)
3. [Subscriptions](#3-subscriptions)
4. [Publishing Messages](#4-publishing-messages)
5. [Subscribing & Message Processing](#5-subscribing--message-processing)
6. [Message Ordering](#6-message-ordering)
7. [Schemas](#7-schemas)
8. [Dead-Letter Topics (DLQ)](#8-dead-letter-topics-dlq)
9. [Exactly-Once Delivery](#9-exactly-once-delivery)
10. [Pub/Sub Lite](#10-pubsub-lite)
11. [IAM, Security & Encryption](#11-iam-security--encryption)
12. [Monitoring, Observability & Alerting](#12-monitoring-observability--alerting)
13. [Integration Patterns & Common Use Cases](#13-integration-patterns--common-use-cases)
14. [gcloud CLI & API Quick Reference](#14-gcloud-cli--api-quick-reference)
15. [Pricing Summary](#15-pricing-summary)
16. [Quick Reference & Comparison Tables](#16-quick-reference--comparison-tables)

---

## 1. Overview & Key Concepts

### What is Pub/Sub?

**Google Cloud Pub/Sub** is a fully managed, globally distributed, durable asynchronous messaging service. It decouples message producers (publishers) from consumers (subscribers), enabling scalable event-driven architectures with strong delivery guarantees.

**Core value proposition:**

| Pillar | Description |
|---|---|
| **Global** | Single endpoint routes messages across all GCP regions |
| **Durable** | Messages replicated across multiple zones; retained up to 31 days |
| **Scalable** | Handles millions of messages/second with zero provisioning |
| **Managed** | No servers, brokers, or partitions to manage |
| **Flexible delivery** | Push, pull, BigQuery, and Cloud Storage subscription types |

---

### Core Messaging Model

```
Publishers                    Pub/Sub Service               Subscribers
    │                               │                            │
    │  publish(message)             │                            │
    ├──────────────────────► TOPIC  │                            │
    │                          │    │                            │
    │                          ├────► Subscription A (pull) ────► Service A
    │                          │    │                            │
    │                          ├────► Subscription B (push) ────► Service B
    │                          │    │
    │                          └────► Subscription C (BigQuery) ► BQ Table
```

| Term | Description |
|---|---|
| **Topic** | Named channel publishers send messages to |
| **Subscription** | Named resource attached to a topic; receives a copy of every message |
| **Message** | Unit of data: `data` (bytes) + `attributes` (key-value) + metadata |
| **Publisher** | Any client that sends messages to a topic |
| **Subscriber** | Any client that receives messages from a subscription |
| **Ack** | Confirmation that a message was successfully processed |

> 💡 Each subscription gets an **independent copy** of every message published to its topic. Fan-out is free — add subscriptions without changing the publisher.

---

### Push vs. Pull Delivery

| Dimension | Pull | Push |
|---|---|---|
| **Initiator** | Subscriber polls Pub/Sub | Pub/Sub calls subscriber's HTTP endpoint |
| **Latency** | Low (streaming pull) to seconds (sync pull) | Very low (triggered immediately) |
| **Scaling** | Subscriber controls pace | Pub/Sub controls delivery rate |
| **Auth** | GCP IAM on subscriber | OIDC token validated by endpoint |
| **Protocol** | gRPC / HTTP | HTTPS only |
| **Best for** | Batch processing, worker pools, back-pressure | Cloud Run, Cloud Functions, webhooks |
| **Infrastructure** | Subscriber must be running | ✅ Endpoint must be HTTPS-accessible |

---

### At-Least-Once vs. Exactly-Once Delivery

| Model | How It Works | Trade-off |
|---|---|---|
| **At-least-once** (default) | Message delivered until acked; duplicates possible | Higher throughput, simpler |
| **Exactly-once** | Ack IDs with sequence numbers prevent redelivery | Slightly higher latency; regional only |

> ⚠️ Even with exactly-once delivery enabled at the Pub/Sub layer, **your consumer should still be idempotent** — network failures between Pub/Sub and your code can cause you to process a message without successfully acking it.

---

### Message Ordering

- Ordering requires an **ordering key** on published messages and `enable_message_ordering=True` on the subscription
- Messages with the **same ordering key** in the **same region** are delivered in publish order
- Different ordering keys are delivered independently (parallel)
- If a message fails delivery for a key, that key is **paused** until explicitly resumed

---

### Message Retention

| Setting | Default | Maximum |
|---|---|---|
| **Topic retention** | 0 (disabled — messages not retained at topic level) | 31 days |
| **Subscription retention** | 7 days | 31 days |
| **Retained after ack** | ❌ (disabled) | 31 days (enable `retainAckedMessages`) |

> 💡 Enable `retainAckedMessages` + `messageRetentionDuration` on a subscription to support **message replay** via `seek`.

---

### Pub/Sub vs. Alternatives

| Service | Model | Ordering | Delivery | Best For |
|---|---|---|---|---|
| **Pub/Sub** | Topic/sub, fan-out | Per key | At-least-once / exactly-once | General async messaging, event fan-out |
| **Kafka** | Topic/partition, log | Per partition | At-least-once | High-throughput log streaming, replay |
| **Cloud Tasks** | Queue, task | FIFO optional | At-least-once, rate-limited | Delayed tasks, rate limiting, retries |
| **Eventarc** | Event routing | N/A | At-least-once | Routing GCP events to Cloud Run / functions |

---

### Pub/Sub Lite vs. Standard Pub/Sub

| Feature | Standard Pub/Sub | Pub/Sub Lite |
|---|---|---|
| **Provisioning** | Auto-scaled | Manual (capacity units) |
| **Routing** | Global | Zonal or Regional |
| **Cost** | Per message data | Per provisioned capacity |
| **Ordering** | Per key | Per partition |
| **Management** | Zero | Partition management needed |
| **Best for** | Variable traffic, fan-out | High-volume, cost-predictable |

---

### Key Limits & Quotas

| Limit | Value |
|---|---|
| Max message size | **10 MB** |
| Max attributes per message | 100 |
| Max attribute key/value size | 256 bytes each |
| Max topics per project | 10,000 |
| Max subscriptions per topic | 10,000 |
| Max ack deadline | 600 seconds (10 min) |
| Max message retention | 31 days |
| Max throughput (default) | ~1 GB/s per region (quota-adjustable) |

---

## 2. Topics

### Topic Configuration

```bash
# Create a topic with retention
gcloud pubsub topics create my-topic \
  --message-retention-duration=7d \           # Retain messages at topic level
  --message-storage-policy-allowed-regions=us-central1,us-east1  # Regional enforcement

# Create a topic with CMEK
gcloud pubsub topics create my-encrypted-topic \
  --topic-encryption-key=projects/proj/locations/us/keyRings/kr/cryptoKeys/key

# List topics
gcloud pubsub topics list --format="table(name)"

# Describe a topic
gcloud pubsub topics describe my-topic

# Update retention duration
gcloud pubsub topics update my-topic \
  --message-retention-duration=14d

# Delete a topic
gcloud pubsub topics delete my-topic

# Publish a test message via CLI
gcloud pubsub topics publish my-topic \
  --message='{"event":"test","user_id":"u123"}' \
  --attribute=source=cli,env=dev
```

---

### Topic with Dead-Letter Configuration

```bash
# First create the dead-letter topic
gcloud pubsub topics create my-topic-dlq

# Create subscription with DLQ (covered in detail in Section 8)
gcloud pubsub subscriptions create my-sub \
  --topic=my-topic \
  --dead-letter-topic=my-topic-dlq \
  --max-delivery-attempts=5
```

---

### Terraform Example

```yaml
# terraform/pubsub.tf
resource "google_pubsub_topic" "main" {
  name = "my-topic"

  message_retention_duration = "604800s"   # 7 days

  message_storage_policy {
    allowed_persistence_regions = ["us-central1", "us-east1"]
  }

  schema_settings {
    schema   = google_pubsub_schema.main.id
    encoding = "JSON"
  }

  kms_key_name = google_kms_crypto_key.pubsub_key.id
}

resource "google_pubsub_schema" "main" {
  name       = "my-schema"
  type       = "AVRO"
  definition = file("schemas/event.avsc")
}
```

---

## 3. Subscriptions

### Subscription Types

| Type | Description | Best For |
|---|---|---|
| **Pull** | Subscriber polls Pub/Sub for messages | Worker pools, batch consumers, back-pressure |
| **Push** | Pub/Sub POSTs messages to HTTPS endpoint | Cloud Run, Cloud Functions, webhooks |
| **BigQuery** | Messages written directly to a BQ table | Analytics pipelines, no code required |
| **Cloud Storage** | Messages batched into GCS files | Archival, log storage, data lake ingestion |

---

### Pull Subscription

```bash
# Create pull subscription
gcloud pubsub subscriptions create my-pull-sub \
  --topic=my-topic \
  --ack-deadline=60 \                          # 60-second ack deadline
  --message-retention-duration=7d \
  --expiration-period=never                    # Don't auto-delete

# Pull messages (blocking)
gcloud pubsub subscriptions pull my-pull-sub \
  --limit=10 \
  --auto-ack                                   # Ack all pulled messages
```

---

### Push Subscription

```bash
# Create push subscription with OIDC auth
gcloud pubsub subscriptions create my-push-sub \
  --topic=my-topic \
  --push-endpoint=https://my-service.run.app/pubsub \
  --push-auth-service-account=sa@proj.iam.gserviceaccount.com \
  --push-auth-token-audience=https://my-service.run.app \
  --ack-deadline=30
```

> 💡 Pub/Sub adds an `Authorization: Bearer <OIDC_TOKEN>` header to every push request. Your endpoint **must** validate this token. Return HTTP `200`–`299` to ack; any other status causes redelivery.

---

### BigQuery Subscription

```bash
# Create BigQuery subscription (writes messages directly to BQ)
gcloud pubsub subscriptions create my-bq-sub \
  --topic=my-topic \
  --bigquery-table=my-project:my_dataset.my_table \
  --write-metadata \                           # Include messageId, publishTime etc.
  --use-topic-schema \                         # Use topic's Pub/Sub schema for BQ schema
  --drop-unknown-fields                        # Ignore extra fields not in BQ schema
```

**BigQuery table schema for `--write-metadata`:**

```sql
-- Auto-created columns when --write-metadata is set
subscription_name  STRING
message_id         STRING
publish_time       TIMESTAMP
data               BYTES (or STRING if encoding is JSON)
attributes         STRING (JSON)
```

---

### Cloud Storage Subscription

```bash
# Create Cloud Storage subscription
gcloud pubsub subscriptions create my-gcs-sub \
  --topic=my-topic \
  --cloud-storage-bucket=my-archive-bucket \
  --cloud-storage-filename-prefix=pubsub/events/ \
  --cloud-storage-filename-suffix=.json \
  --cloud-storage-max-duration=5m \           # Flush file every 5 minutes
  --cloud-storage-max-bytes=100000000 \       # Flush at 100 MB
  --cloud-storage-output-format=text          # or avro
```

---

### Subscription Configuration Options

| Option | CLI Flag | Default | Description |
|---|---|---|---|
| Ack deadline | `--ack-deadline` | 10s | Window to process before redelivery (10–600s) |
| Retain acked | `--retain-acked-messages` | false | Keep messages after ack for replay |
| Retention duration | `--message-retention-duration` | 7d | How long unacked messages are kept |
| Exactly-once | `--enable-exactly-once-delivery` | false | Prevent duplicate delivery |
| Ordering | `--enable-message-ordering` | false | Deliver per-key in order |
| Expiration | `--expiration-period` | 31d | Auto-delete if inactive |
| Filter | `--message-filter` | none | Server-side attribute filtering |

---

### Subscription Filtering

Server-side filtering using **Common Expression Language (CEL)**:

```bash
# Create subscription with attribute filter
gcloud pubsub subscriptions create my-filtered-sub \
  --topic=my-topic \
  --message-filter='attributes.env = "prod" AND attributes.type = "order"'

# Filter operators
# = (equals)        attributes.key = "value"
# != (not equals)   attributes.key != "value"
# : (has prefix)    attributes.key : "prefix"
# AND / OR          combine conditions
# NOT               negate condition
# hasPrefix()       hasPrefix(attributes.region, "us-")

# Examples
'attributes.priority = "high"'
'attributes.source = "web" OR attributes.source = "mobile"'
'NOT attributes.test = "true"'
'hasPrefix(attributes.user_id, "premium_")'
```

> ⚠️ Filtered-out messages are **still billed** as delivered but are not charged for ack. Filtering reduces subscriber workload but not ingestion cost.

---

### Full Subscription Create Reference

```bash
gcloud pubsub subscriptions create MY_SUB \
  --topic=MY_TOPIC \
  --topic-project=OTHER_PROJECT \             # If topic is in a different project
  --ack-deadline=60 \
  --push-endpoint=https://endpoint/path \     # Omit for pull
  --push-auth-service-account=SA_EMAIL \
  --dead-letter-topic=MY_DLQ_TOPIC \
  --max-delivery-attempts=10 \
  --message-retention-duration=3d \
  --retain-acked-messages \
  --enable-exactly-once-delivery \
  --enable-message-ordering \
  --message-filter='attributes.env = "prod"' \
  --expiration-period=never \
  --min-retry-delay=10s \                     # Push only: min backoff
  --max-retry-delay=600s                      # Push only: max backoff
```

---

## 4. Publishing Messages

### Message Structure

```json
{
  "data": "eyJ1c2VyX2lkIjoidTEyMyIsImFtb3VudCI6OTkuOTl9",  // base64-encoded bytes
  "attributes": {
    "source": "checkout-service",
    "env": "prod",
    "version": "2"
  },
  "messageId": "13264421353124",        // Set by Pub/Sub server
  "publishTime": "2026-03-16T10:00:00Z", // Set by Pub/Sub server
  "orderingKey": "user_u123"            // Optional; for ordered delivery
}
```

---

### Python Publisher Examples

```python
from google.cloud import pubsub_v1
from google.api_core import retry
from concurrent import futures
import json

PROJECT   = "my-project"
TOPIC_ID  = "my-topic"

# ── Basic publish ─────────────────────────────────────────────────────
publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path(PROJECT, TOPIC_ID)

data = json.dumps({"user_id": "u123", "amount": 99.99}).encode("utf-8")
future = publisher.publish(topic_path, data=data)
message_id = future.result()           # Blocks until published
print(f"Published message ID: {message_id}")

# ── Publish with attributes ───────────────────────────────────────────
future = publisher.publish(
    topic_path,
    data=data,
    source="checkout-service",         # Attributes as keyword args
    env="prod",
    version="2"
)
future.result()

# ── Publish with ordering key ─────────────────────────────────────────
# Publisher must be configured with enable_message_ordering=True
publisher_ordered = pubsub_v1.PublisherClient(
    publisher_options=pubsub_v1.types.PublisherOptions(
        enable_message_ordering=True
    )
)

for i in range(10):
    future = publisher_ordered.publish(
        topic_path,
        data=f"message-{i}".encode(),
        ordering_key="user_u123"       # All messages for same key are ordered
    )
    future.result()

# ── Async batch publish with callbacks ───────────────────────────────
from google.cloud.pubsub_v1.types import BatchSettings

publisher_batch = pubsub_v1.PublisherClient(
    batch_settings=pubsub_v1.types.BatchSettings(
        max_messages=1000,             # Batch up to 1000 messages
        max_bytes=1024 * 1024 * 5,    # Or up to 5 MB
        max_latency=0.1,              # Or max 100ms wait
    )
)

publish_futures = []

def on_publish_done(future: futures.Future) -> None:
    try:
        msg_id = future.result(timeout=5)
        print(f"Published: {msg_id}")
    except Exception as e:
        print(f"Publish failed: {e}")

for i in range(100):
    future = publisher_batch.publish(
        topic_path,
        data=f"batch-msg-{i}".encode()
    )
    future.add_done_callback(on_publish_done)
    publish_futures.append(future)

# Wait for all publishes to complete
futures.wait(publish_futures, return_when=futures.ALL_COMPLETED)

# ── Publisher with custom retry ───────────────────────────────────────
from google.api_core import retry as api_retry

custom_retry = api_retry.Retry(
    initial=0.1,       # Initial backoff in seconds
    maximum=60.0,      # Max backoff
    multiplier=1.3,    # Backoff multiplier
    deadline=300.0,    # Total retry deadline (5 min)
)

publisher_retry = pubsub_v1.PublisherClient()
future = publisher_retry.publish(topic_path, data=data, retry=custom_retry)
future.result()

# ── Publish with GZIP compression ────────────────────────────────────
publisher_compressed = pubsub_v1.PublisherClient(
    publisher_options=pubsub_v1.types.PublisherOptions(
        enable_message_compression=True,
        message_compression_threshold=240,  # Compress if payload > 240 bytes
    )
)
```

---

### Java Publisher Example

```java
import com.google.cloud.pubsub.v1.Publisher;
import com.google.pubsub.v1.PubsubMessage;
import com.google.pubsub.v1.TopicName;
import com.google.protobuf.ByteString;
import io.grpc.netty.shaded.io.netty.util.concurrent.Future;
import org.threeten.bp.Duration;

TopicName topicName = TopicName.of("my-project", "my-topic");

// Configure batch settings
BatchingSettings batchingSettings = BatchingSettings.newBuilder()
    .setElementCountThreshold(1000L)
    .setRequestByteThreshold(1024 * 1024L)  // 1 MB
    .setDelayThreshold(Duration.ofMillis(100))
    .build();

Publisher publisher = Publisher.newBuilder(topicName)
    .setBatchingSettings(batchingSettings)
    .setEnableMessageOrdering(true)          // For ordered publishing
    .build();

try {
    ByteString data = ByteString.copyFromUtf8("{\"user_id\":\"u123\"}");
    PubsubMessage message = PubsubMessage.newBuilder()
        .setData(data)
        .putAttributes("source", "checkout")
        .putAttributes("env", "prod")
        .setOrderingKey("user_u123")         // For ordered delivery
        .build();

    ApiFuture<String> future = publisher.publish(message);
    // Add callback
    ApiFutures.addCallback(future, new ApiFutureCallback<String>() {
        public void onSuccess(String messageId) {
            System.out.println("Published: " + messageId);
        }
        public void onFailure(Throwable t) {
            System.err.println("Publish failed: " + t.getMessage());
        }
    }, MoreExecutors.directExecutor());
} finally {
    publisher.shutdown();
    publisher.awaitTermination(1, TimeUnit.MINUTES);
}
```

---

### gcloud Publish Examples

```bash
# Publish a message
gcloud pubsub topics publish my-topic \
  --message='Hello, World!'

# Publish with attributes
gcloud pubsub topics publish my-topic \
  --message='{"event":"purchase"}' \
  --attribute=source=web,env=prod,user_id=u123

# Publish with ordering key
gcloud pubsub topics publish my-topic \
  --message='order-1' \
  --ordering-key=user_u123

# Publish from file
gcloud pubsub topics publish my-topic \
  --message="$(cat payload.json)"
```

---

## 5. Subscribing & Message Processing

### Python: Synchronous Pull

```python
from google.cloud import pubsub_v1

subscriber = pubsub_v1.SubscriberClient()
sub_path = subscriber.subscription_path("my-project", "my-pull-sub")

# Pull up to 10 messages (blocking if no messages)
response = subscriber.pull(
    request={"subscription": sub_path, "max_messages": 10},
    timeout=30
)

ack_ids = []
for msg in response.received_messages:
    payload = msg.message.data.decode("utf-8")
    attrs   = dict(msg.message.attributes)
    print(f"Received: {payload}, attrs={attrs}")
    ack_ids.append(msg.ack_id)

# Ack all at once
if ack_ids:
    subscriber.acknowledge(
        request={"subscription": sub_path, "ack_ids": ack_ids}
    )
```

---

### Python: Streaming Pull (Recommended for Production)

```python
from google.cloud import pubsub_v1
from concurrent.futures import TimeoutError
import time

subscriber = pubsub_v1.SubscriberClient()
sub_path   = subscriber.subscription_path("my-project", "my-pull-sub")

def callback(message: pubsub_v1.types.PubsubMessage) -> None:
    """Called in a background thread for each message."""
    try:
        payload = message.data.decode("utf-8")
        print(f"Processing: {payload}")

        # Long-running processing...
        process_message(payload)

        message.ack()                          # ✅ Ack on success
    except Exception as e:
        print(f"Processing failed: {e}")
        message.nack()                         # ❌ Nack — redelivery requested

# Flow control: limit concurrent messages in memory
flow_control = pubsub_v1.types.FlowControl(
    max_outstanding_messages=100,              # Max messages held in memory
    max_outstanding_bytes=50 * 1024 * 1024,   # Max bytes in memory (50 MB)
)

streaming_pull = subscriber.subscribe(
    sub_path,
    callback=callback,
    flow_control=flow_control,
    await_callbacks_on_shutdown=True,
)

print("Listening for messages...")
try:
    streaming_pull.result(timeout=None)        # Block forever
except KeyboardInterrupt:
    streaming_pull.cancel()
    streaming_pull.result()                    # Wait for clean shutdown
```

---

### Python: Push Handler (FastAPI)

```python
# push_handler.py — Cloud Run / App Engine push endpoint
from fastapi import FastAPI, Request, HTTPException
from google.auth.transport import requests as grequests
from google.oauth2 import id_token
import base64, json

app = FastAPI()

PUBSUB_AUDIENCE = "https://my-service.run.app"

@app.post("/pubsub")
async def receive_pubsub_message(request: Request):
    # 1. Validate OIDC token from Authorization header
    auth_header = request.headers.get("Authorization", "")
    if not auth_header.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing token")

    token = auth_header.split("Bearer ")[1]
    try:
        id_token.verify_oauth2_token(
            token, grequests.Request(), audience=PUBSUB_AUDIENCE
        )
    except ValueError as e:
        raise HTTPException(status_code=403, detail=f"Invalid token: {e}")

    # 2. Parse the Pub/Sub envelope
    body = await request.json()
    pubsub_msg = body.get("message", {})

    data_b64  = pubsub_msg.get("data", "")
    data      = base64.b64decode(data_b64).decode("utf-8") if data_b64 else ""
    attrs     = pubsub_msg.get("attributes", {})
    msg_id    = pubsub_msg.get("messageId")

    print(f"Received [{msg_id}]: {data}, attrs={attrs}")

    try:
        process(json.loads(data))
        return {"status": "ok"}          # HTTP 200 = ack
    except Exception as e:
        # Return non-2xx to trigger redelivery (nack)
        raise HTTPException(status_code=500, detail=str(e))
```

---

### Ack/Nack Semantics

| Action | Method | Effect |
|---|---|---|
| **Ack** | `message.ack()` | Message removed from subscription; not redelivered |
| **Nack** | `message.nack()` | Message immediately eligible for redelivery |
| **Ignore** | (no action) | Message redelivered after `ackDeadlineSeconds` |
| **Extend deadline** | `message.modify_ack_deadline(seconds)` | Reset deadline for long-running processing |

```python
# Extend ack deadline during long processing (streaming pull)
def long_running_callback(message):
    # Extend deadline every 30s for up to 5 minutes of processing
    import threading
    def extend():
        while not done:
            message.modify_ack_deadline(60)  # Reset to 60 more seconds
            threading.Event().wait(30)
    done = False
    extender = threading.Thread(target=extend, daemon=True)
    extender.start()

    try:
        slow_process(message.data)
        done = True
        message.ack()
    except Exception:
        done = True
        message.nack()
```

---

### Java: Async Subscriber

```java
import com.google.cloud.pubsub.v1.AckReplyConsumer;
import com.google.cloud.pubsub.v1.MessageReceiver;
import com.google.cloud.pubsub.v1.Subscriber;
import com.google.pubsub.v1.ProjectSubscriptionName;
import com.google.pubsub.v1.PubsubMessage;

ProjectSubscriptionName subName =
    ProjectSubscriptionName.of("my-project", "my-pull-sub");

MessageReceiver receiver = (PubsubMessage message, AckReplyConsumer consumer) -> {
    try {
        String data = message.getData().toStringUtf8();
        System.out.println("Processing: " + data);
        process(data);
        consumer.ack();                    // ✅ Ack on success
    } catch (Exception e) {
        consumer.nack();                   // ❌ Nack on failure
    }
};

Subscriber subscriber = Subscriber.newBuilder(subName, receiver)
    .setFlowControlSettings(
        FlowControlSettings.newBuilder()
            .setMaxOutstandingElementCount(100L)
            .setMaxOutstandingRequestBytes(50 * 1024 * 1024L)
            .build()
    )
    .build();

subscriber.startAsync().awaitRunning();
System.out.println("Subscriber running...");
// subscriber.stopAsync().awaitTerminated();   // Graceful shutdown
```

---

## 6. Message Ordering

### How Ordering Works

```
Publisher (ordering_key="user_123")
    │
    ├── msg-1 (ordering_key="user_123") ──► same server ──► delivered 1st
    ├── msg-2 (ordering_key="user_123") ──► same server ──► delivered 2nd
    └── msg-3 (ordering_key="user_123") ──► same server ──► delivered 3rd

Publisher (ordering_key="user_456")
    │
    └── msg-A (ordering_key="user_456") ──► different server (parallel)
```

> 💡 Messages with **different ordering keys** are processed independently and in parallel — ordering does not reduce overall throughput if you have many keys.

---

### Publishing with Ordering Keys

```python
from google.cloud import pubsub_v1

# MUST enable ordering on publisher client
publisher = pubsub_v1.PublisherClient(
    publisher_options=pubsub_v1.types.PublisherOptions(
        enable_message_ordering=True
    )
)
topic_path = publisher.topic_path("my-project", "my-topic")

# All messages with same key are delivered in publish order
for i, event in enumerate(user_events):
    future = publisher.publish(
        topic_path,
        data=json.dumps(event).encode(),
        ordering_key=f"user_{event['user_id']}"    # Key = user ID
    )
    future.result()
```

---

### Subscribing in Order

```python
# Subscription MUST have enable_message_ordering enabled
# gcloud pubsub subscriptions create my-ordered-sub \
#   --topic=my-topic --enable-message-ordering

subscriber = pubsub_v1.SubscriberClient()
sub_path   = subscriber.subscription_path("my-project", "my-ordered-sub")

sequence = {}   # Track per-key sequence

def ordered_callback(message):
    key = message.ordering_key
    data = json.loads(message.data.decode())

    expected_seq = sequence.get(key, 0)
    actual_seq   = data.get("seq", 0)

    if actual_seq == expected_seq:
        process(data)
        sequence[key] = expected_seq + 1
        message.ack()
    else:
        print(f"Unexpected sequence {actual_seq} for key {key}")
        message.nack()

subscriber.subscribe(sub_path, callback=ordered_callback)
```

---

### Resuming a Paused Ordering Key

```python
# If delivery fails for an ordering key, that key is paused.
# The publisher must explicitly resume it.

from google.api_core.exceptions import ServiceUnavailable

publisher = pubsub_v1.PublisherClient(
    publisher_options=pubsub_v1.types.PublisherOptions(enable_message_ordering=True)
)

def publish_with_resume(topic_path, data, ordering_key):
    try:
        future = publisher.publish(topic_path, data=data, ordering_key=ordering_key)
        future.result()
    except Exception as e:
        print(f"Publish failed for key {ordering_key}: {e}")
        # Resume the paused ordering key before retrying
        publisher.resume_publish(topic_path, ordering_key)
        # Retry once
        publisher.publish(topic_path, data=data, ordering_key=ordering_key).result()
```

> ⚠️ When an ordering key is paused due to a publish error, **all subsequent publishes to that key are immediately rejected** until `resume_publish()` is called. Failing to call this leaves the key permanently paused.

---

## 7. Schemas

### Creating and Attaching Schemas

```bash
# Create an Avro schema
cat > event.avsc << 'EOF'
{
  "type": "record",
  "name": "Event",
  "fields": [
    {"name": "user_id",  "type": "string"},
    {"name": "amount",   "type": "double"},
    {"name": "event_ts", "type": "long", "logicalType": "timestamp-millis"}
  ]
}
EOF

gcloud pubsub schemas create my-avro-schema \
  --type=AVRO \
  --definition-file=event.avsc

# Create a Protobuf schema
cat > event.proto << 'EOF'
syntax = "proto3";
message Event {
  string user_id  = 1;
  double amount   = 2;
  int64  event_ts = 3;
}
EOF

gcloud pubsub schemas create my-proto-schema \
  --type=PROTOCOL_BUFFER \
  --definition-file=event.proto

# Attach schema to topic
gcloud pubsub topics create my-schema-topic \
  --schema=my-avro-schema \
  --message-encoding=JSON         # or BINARY

# Validate a message against a schema (dry-run)
gcloud pubsub schemas validate-message \
  --message-encoding=JSON \
  --schema-name=my-avro-schema \
  --message='{"user_id":"u1","amount":99.99,"event_ts":1710590000000}'

# List schemas
gcloud pubsub schemas list

# Describe schema
gcloud pubsub schemas describe my-avro-schema

# Delete schema
gcloud pubsub schemas delete my-avro-schema
```

---

### Python: Publishing Avro-Encoded Messages

```python
import io
import json
import avro.schema
import avro.io
from google.cloud import pubsub_v1

AVRO_SCHEMA_STR = """
{
  "type": "record",
  "name": "Event",
  "fields": [
    {"name": "user_id",  "type": "string"},
    {"name": "amount",   "type": "double"},
    {"name": "event_ts", "type": "long"}
  ]
}
"""

schema  = avro.schema.parse(AVRO_SCHEMA_STR)
publisher  = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path("my-project", "my-schema-topic")

def publish_avro(record: dict) -> str:
    """Serialize record as Avro binary and publish."""
    writer = avro.io.DatumWriter(schema)
    buf    = io.BytesIO()
    encoder = avro.io.BinaryEncoder(buf)
    writer.write(record, encoder)
    data = buf.getvalue()

    future = publisher.publish(topic_path, data=data)
    return future.result()

publish_avro({"user_id": "u123", "amount": 99.99, "event_ts": 1710590000000})
```

---

### Python: Publishing Protobuf-Encoded Messages

```python
# event_pb2 generated from: protoc --python_out=. event.proto
import event_pb2
from google.cloud import pubsub_v1

publisher  = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path("my-project", "my-proto-topic")

event = event_pb2.Event(
    user_id  = "u123",
    amount   = 99.99,
    event_ts = 1710590000000
)

future = publisher.publish(topic_path, data=event.SerializeToString())
print(f"Published proto message: {future.result()}")
```

---

### Schema Evolution

| Change | Backward Compatible | Forward Compatible | Safe to Deploy |
|---|---|---|---|
| Add optional field with default | ✅ | ✅ | ✅ Yes |
| Remove field | ✅ (readers ignore) | ❌ (writers missing) | ⚠️ Careful |
| Rename field | ❌ | ❌ | ❌ Breaking |
| Change field type | ❌ | ❌ | ❌ Breaking |
| Add required field (no default) | ❌ | ✅ | ❌ Breaking |

> 💡 Use schema **revision IDs** to track evolution. Pub/Sub maintains a history of all revisions for a schema.

---

## 8. Dead-Letter Topics (DLQ)

### What DLQ Is

A **dead-letter topic** receives messages that Pub/Sub cannot deliver after `--max-delivery-attempts` retries. This prevents poison messages from blocking the subscription backlog indefinitely.

---

### DLQ Setup

```bash
# 1. Create source topic and DLQ topic
gcloud pubsub topics create my-topic
gcloud pubsub topics create my-topic-dlq

# 2. Get the Pub/Sub service account for the project
PROJECT_NUMBER=$(gcloud projects describe my-project --format="value(projectNumber)")
PUBSUB_SA="service-${PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com"

# 3. Grant Pub/Sub SA permission to publish to DLQ topic
gcloud pubsub topics add-iam-policy-binding my-topic-dlq \
  --member="serviceAccount:${PUBSUB_SA}" \
  --role="roles/pubsub.publisher"

# 4. Grant Pub/Sub SA permission to subscribe from source
gcloud pubsub subscriptions add-iam-policy-binding my-sub \
  --member="serviceAccount:${PUBSUB_SA}" \
  --role="roles/pubsub.subscriber"

# 5. Create subscription with DLQ
gcloud pubsub subscriptions create my-sub \
  --topic=my-topic \
  --dead-letter-topic=projects/my-project/topics/my-topic-dlq \
  --max-delivery-attempts=5 \    # 5–100; message goes to DLQ after 5 failures
  --ack-deadline=30

# 6. Create a subscription on the DLQ topic for inspection
gcloud pubsub subscriptions create my-dlq-sub \
  --topic=my-topic-dlq
```

---

### DLQ Message Metadata Attributes

When a message is forwarded to the DLQ, Pub/Sub adds these attributes:

| Attribute | Description |
|---|---|
| `CloudPubSubDeadLetterSourceDeliveryCount` | Number of delivery attempts |
| `CloudPubSubDeadLetterSourceSubscriptionName` | Original subscription full name |
| `CloudPubSubDeadLetterSourceTopicPublishTime` | Original publish timestamp |
| `CloudPubSubDeadLetterSourceMessageId` | Original message ID |

---

### Python: Consuming DLQ Messages

```python
from google.cloud import pubsub_v1
import json

subscriber = pubsub_v1.SubscriberClient()
dlq_path   = subscriber.subscription_path("my-project", "my-dlq-sub")

def handle_dlq_message(message):
    # Inspect dead-letter metadata
    delivery_count = message.attributes.get(
        "CloudPubSubDeadLetterSourceDeliveryCount", "?")
    original_sub   = message.attributes.get(
        "CloudPubSubDeadLetterSourceSubscriptionName", "?")
    original_time  = message.attributes.get(
        "CloudPubSubDeadLetterSourceTopicPublishTime", "?")

    payload = message.data.decode("utf-8")

    print(f"Dead letter message: attempts={delivery_count}, "
          f"sub={original_sub}, published={original_time}")
    print(f"Payload: {payload}")

    # Options:
    # 1. Log and ack (discard with audit trail)
    # 2. Fix the data and republish to the original topic
    # 3. Store in BigQuery for analysis

    # Fix and republish
    try:
        data = json.loads(payload)
        data["fixed"] = True
        publisher = pubsub_v1.PublisherClient()
        publisher.publish(
            publisher.topic_path("my-project", "my-topic"),
            data=json.dumps(data).encode()
        ).result()
        message.ack()
    except Exception as e:
        print(f"Could not reprocess: {e}")
        message.ack()   # Ack to avoid infinite DLQ loop

subscriber.subscribe(dlq_path, callback=handle_dlq_message).result()
```

---

## 9. Exactly-Once Delivery

### How It Works

Standard Pub/Sub provides **at-least-once** delivery — a message may be delivered more than once if the ack is not received before the deadline. With `enable-exactly-once-delivery`:

1. Each message has a unique **ack ID with sequence numbers**
2. When your code calls `ack()`, Pub/Sub verifies the ack ID is still valid
3. If the ack fails transiently, the client library returns `temporary_failure` — **message will not be redelivered** until you retry the ack
4. Permanent failures are surfaced as `permanent_failure`

> ⚠️ Exactly-once delivery applies **only at the Pub/Sub delivery layer**, not end-to-end. If your code processes a message but the `ack()` call times out, Pub/Sub will not redeliver it — but your processing code already ran. **Always make consumers idempotent.**

---

### Enabling and Handling Exactly-Once

```bash
# Create subscription with exactly-once delivery
gcloud pubsub subscriptions create my-eod-sub \
  --topic=my-topic \
  --enable-exactly-once-delivery
```

```python
from google.cloud import pubsub_v1
from google.cloud.pubsub_v1.open_telemetry.subscribe_opentelemetry import SubscribeOpenTelemetry
from google.cloud.pubsub_v1 import types

subscriber = pubsub_v1.SubscriberClient()
sub_path   = subscriber.subscription_path("my-project", "my-eod-sub")

def callback(message: pubsub_v1.types.PubsubMessage) -> None:
    print(f"Processing: {message.data.decode()}")

    try:
        process_message(message.data)

        # With exactly-once, ack() returns an AcknowledgementResult
        ack_result = message.ack_with_response()

        # Handle ack result
        if ack_result == pubsub_v1.types.AcknowledgementStatus.SUCCESS:
            print("Message acked successfully")

        elif ack_result == pubsub_v1.types.AcknowledgementStatus.TEMPORARY_FAILURE:
            # Message will NOT be redelivered — retry the ack
            print("Temporary ack failure — retrying")
            # The client library automatically retries temporary failures

        elif ack_result == pubsub_v1.types.AcknowledgementStatus.INVALID_ACK_ID:
            print("Ack ID expired — message may have been redelivered")

        elif ack_result == pubsub_v1.types.AcknowledgementStatus.PERMISSION_DENIED:
            print("Permission denied to ack")

    except Exception as e:
        print(f"Processing failed: {e}")
        message.nack()

streaming_pull = subscriber.subscribe(sub_path, callback=callback)
streaming_pull.result()
```

---

### Exactly-Once Limitations

| Limitation | Detail |
|---|---|
| **Regional only** | Guarantees apply within a single GCP region |
| **Not cross-region** | Cross-region delivery may still have duplicates |
| **Not end-to-end** | Guarantees delivery layer only, not your processing |
| **Higher latency** | Slightly higher p99 latency than at-least-once |
| **Idempotency still needed** | Network failures between Pub/Sub and your code |

---

## 10. Pub/Sub Lite

### What is Pub/Sub Lite?

Pub/Sub Lite is a **partition-based, zonal/regional** messaging service with **manual capacity provisioning** and significantly lower cost for high-volume workloads. Think: Kafka-like economics with managed GCP infrastructure.

---

### gcloud Pub/Sub Lite Examples

```bash
# Create a Lite topic (zonal)
gcloud pubsub lite-topics create my-lite-topic \
  --location=us-central1-a \               # Zonal location
  --partitions=4 \                         # Number of partitions
  --per-partition-publish-mib=4 \          # 4 MiB/s publish throughput per partition
  --per-partition-subscribe-mib=8 \        # 8 MiB/s subscribe throughput per partition
  --per-partition-storage-gib=30           # 30 GiB storage per partition

# Create a Lite subscription
gcloud pubsub lite-subscriptions create my-lite-sub \
  --location=us-central1-a \
  --topic=my-lite-topic \
  --delivery-requirement=DELIVER_IMMEDIATELY   # or DELIVER_AFTER_STORED

# List Lite topics
gcloud pubsub lite-topics list --location=us-central1-a

# Describe Lite topic
gcloud pubsub lite-topics describe my-lite-topic --location=us-central1-a

# Delete
gcloud pubsub lite-topics delete my-lite-topic --location=us-central1-a
```

---

### Python: Pub/Sub Lite Publish and Subscribe

```python
# pip install google-cloud-pubsublite

from google.cloud.pubsublite.cloudpubsub import PublisherClient, SubscriberClient
from google.cloud.pubsublite.types import (
    CloudRegion, CloudZone, TopicPath, SubscriptionPath, MessageMetadata
)
from google.pubsub_v1 import PubsubMessage

# ── Publish ───────────────────────────────────────────────────────────
location  = CloudZone(CloudRegion("us-central1"), "a")
topic_path = TopicPath("my-project", location, "my-lite-topic")

with PublisherClient() as publisher:
    for i in range(10):
        msg = PubsubMessage(data=f"message-{i}".encode())
        future = publisher.publish(topic_path, msg)
        metadata: MessageMetadata = future.result()
        print(f"Published: partition={metadata.partition.value}, "
              f"offset={metadata.cursor.offset}")

# ── Subscribe ─────────────────────────────────────────────────────────
sub_path = SubscriptionPath("my-project", location, "my-lite-sub")

def lite_callback(message: PubsubMessage) -> None:
    print(f"Received: {message.data.decode()}")
    message.ack()

with SubscriberClient() as subscriber:
    future = subscriber.subscribe(sub_path, callback=lite_callback)
    try:
        future.result(timeout=60)
    except TimeoutError:
        future.cancel()
```

---

## 11. IAM, Security & Encryption

### IAM Roles

| Role | Scope | Grants |
|---|---|---|
| `roles/pubsub.admin` | Project | Full control: topics, subscriptions, schemas |
| `roles/pubsub.editor` | Project | Create/delete topics and subscriptions; publish and subscribe |
| `roles/pubsub.publisher` | Topic | Publish messages to the topic only |
| `roles/pubsub.subscriber` | Subscription | Subscribe and consume messages only |
| `roles/pubsub.viewer` | Project | View topic and subscription metadata (no data) |

> 💡 Grant `pubsub.publisher` at the **topic level** (not project) and `pubsub.subscriber` at the **subscription level** — this is the principle of least privilege.

---

### Per-Resource IAM Bindings

```bash
# Grant publisher access to a specific topic (not whole project)
gcloud pubsub topics add-iam-policy-binding my-topic \
  --member="serviceAccount:publisher-sa@proj.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"

# Grant subscriber access to a specific subscription
gcloud pubsub subscriptions add-iam-policy-binding my-sub \
  --member="serviceAccount:subscriber-sa@proj.iam.gserviceaccount.com" \
  --role="roles/pubsub.subscriber"

# Grant cross-project publishing (publisher in project A, topic in project B)
gcloud pubsub topics add-iam-policy-binding my-topic \
  --project=project-b \
  --member="serviceAccount:publisher-sa@project-a.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"

# View current IAM policy
gcloud pubsub topics get-iam-policy my-topic
gcloud pubsub subscriptions get-iam-policy my-sub
```

---

### Service Account Least-Privilege Setup

```bash
# Publisher service account (only can publish to specific topic)
gcloud iam service-accounts create publisher-sa \
  --display-name="Pub/Sub Publisher"

gcloud pubsub topics add-iam-policy-binding my-topic \
  --member="serviceAccount:publisher-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"

# Subscriber service account (only can read from specific subscription)
gcloud iam service-accounts create subscriber-sa \
  --display-name="Pub/Sub Subscriber"

gcloud pubsub subscriptions add-iam-policy-binding my-sub \
  --member="serviceAccount:subscriber-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/pubsub.subscriber"
```

---

### Push Subscription OIDC Token Validation

```python
# Validate the OIDC token that Pub/Sub sends with every push delivery
from google.auth.transport import requests
from google.oauth2 import id_token

def validate_pubsub_token(bearer_token: str, audience: str) -> dict:
    """
    Validate OIDC token from Pub/Sub push subscription.
    audience = the push endpoint URL (e.g., https://my-service.run.app/pubsub)
    """
    try:
        claim = id_token.verify_oauth2_token(
            bearer_token,
            requests.Request(),
            audience=audience
        )
        # claim["email"] will be the SA email configured on the push sub
        # claim["email_verified"] must be True
        assert claim.get("email_verified"), "Email not verified"
        return claim
    except Exception as e:
        raise ValueError(f"Invalid Pub/Sub OIDC token: {e}")
```

---

### CMEK with Cloud KMS

```bash
# Create a KMS key for Pub/Sub
gcloud kms keyrings create pubsub-keyring --location=us-central1

gcloud kms keys create pubsub-key \
  --location=us-central1 \
  --keyring=pubsub-keyring \
  --purpose=encryption

# Grant Pub/Sub service account access to key
PROJECT_NUMBER=$(gcloud projects describe my-project --format="value(projectNumber)")
gcloud kms keys add-iam-policy-binding pubsub-key \
  --location=us-central1 \
  --keyring=pubsub-keyring \
  --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Create CMEK-encrypted topic
gcloud pubsub topics create my-encrypted-topic \
  --topic-encryption-key=projects/my-project/locations/us-central1/keyRings/pubsub-keyring/cryptoKeys/pubsub-key
```

---

### Enabling Audit Logging

```bash
# Enable Data Access audit logs (ADMIN_READ, DATA_READ, DATA_WRITE)
# Via gcloud IAM policy:
cat > audit-policy.yaml << 'EOF'
auditConfigs:
- service: pubsub.googleapis.com
  auditLogConfigs:
  - logType: ADMIN_READ
  - logType: DATA_READ    # Logs subscribe/pull operations
  - logType: DATA_WRITE   # Logs publish operations
EOF

gcloud projects set-iam-policy my-project audit-policy.yaml
```

---

## 12. Monitoring, Observability & Alerting

### Key Metrics

| Metric | Full Name | Alert When |
|---|---|---|
| 📊 **Backlog size** | `subscription/num_undelivered_messages` | > threshold for SLA |
| ⏱️ **Oldest unacked age** | `subscription/oldest_unacked_message_age` | > processing SLA |
| 🔄 **In-flight messages** | `subscription/num_outstanding_messages` | Spikes indicate slow consumers |
| 📤 **Publish throughput** | `topic/send_message_operation_count` | Drops indicate publisher issues |
| 📥 **Pull throughput** | `subscription/pull_message_operation_count` | Drops indicate subscriber issues |
| ☠️ **DLQ forwards** | `subscription/dead_letter_message_count` | > 0 needs investigation |
| ❌ **Publish errors** | `topic/send_message_operation_count` (filter errors) | Any sustained errors |

---

### Creating Alerting Policies

```bash
# Alert when subscription backlog exceeds 10,000 messages
gcloud alpha monitoring policies create \
  --display-name="Pub/Sub High Backlog" \
  --condition-filter='
    metric.type="pubsub.googleapis.com/subscription/num_undelivered_messages"
    resource.labels.subscription_id="my-sub"' \
  --condition-threshold-value=10000 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=300s \
  --notification-channels=CHANNEL_ID \
  --project=my-project

# Alert when oldest unacked message age exceeds 5 minutes
gcloud alpha monitoring policies create \
  --display-name="Pub/Sub Stale Messages" \
  --condition-filter='
    metric.type="pubsub.googleapis.com/subscription/oldest_unacked_message_age"
    resource.labels.subscription_id="my-sub"' \
  --condition-threshold-value=300 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=120s \
  --notification-channels=CHANNEL_ID
```

---

### Cloud Logging Queries

```bash
# Query admin activity logs (topic/subscription creation)
gcloud logging read \
  'resource.type="pubsub_topic"
   protoPayload.methodName="google.pubsub.v1.Publisher.CreateTopic"' \
  --project=my-project --limit=50

# Query publish errors
gcloud logging read \
  'resource.type="pubsub_topic"
   severity=ERROR' \
  --project=my-project --limit=100

# Query failed push deliveries
gcloud logging read \
  'resource.type="pubsub_subscription"
   jsonPayload.@type="type.googleapis.com/google.pubsub.v1.PushConfig"
   severity>=WARNING' \
  --project=my-project --limit=50
```

---

### Debugging Common Issues

| Symptom | Likely Cause | Investigation |
|---|---|---|
| **Growing backlog** | Consumer too slow or down | Check `oldest_unacked_message_age`; scale consumers |
| **Stuck backlog** (not growing/shrinking) | Consumer nack-looping | Check `num_outstanding_messages`; add DLQ |
| **Push endpoint errors** | App returning non-2xx | Check app logs; verify OIDC token validation |
| **Duplicate messages** | Consumer crashes before ack | Enable exactly-once or make consumer idempotent |
| **No messages received** | Wrong subscription / filter | `gcloud subscriptions pull` to verify messages exist |
| **DLQ growing** | Poison messages | Inspect DLQ messages; fix consumer or input data |
| **High publish latency** | Batching too large | Reduce `max_latency` in batch settings |

---

### Diagnostic Commands

```bash
# Manually pull to verify messages are present
gcloud pubsub subscriptions pull my-sub --limit=5 --format=json

# Describe subscription to check backlog and config
gcloud pubsub subscriptions describe my-sub \
  --format="json(pushConfig,ackDeadlineSeconds,messageRetentionDuration)"

# Check dead-letter configuration
gcloud pubsub subscriptions describe my-sub \
  --format="json(deadLetterPolicy)"

# Seek to a timestamp (replay messages)
gcloud pubsub subscriptions seek my-sub \
  --time=2026-03-15T00:00:00Z

# Create a snapshot and seek to it
gcloud pubsub snapshots create my-snapshot --subscription=my-sub
gcloud pubsub subscriptions seek my-sub --snapshot=my-snapshot
```

---

## 13. Integration Patterns & Common Use Cases

### Fan-Out Pattern

```
                    ┌─────────────────────────────────────────┐
Publisher           │              TOPIC: orders              │
  │ publish ───────►│                                         │
                    └──┬───────────────┬────────────────┬────┘
                       │               │                │
              ┌────────▼──────┐ ┌──────▼───────┐ ┌─────▼──────────┐
              │ Sub: inventory│ │ Sub: billing │ │ Sub: analytics │
              │  (pull)       │ │  (push)      │ │  (BigQuery)    │
              └───────────────┘ └──────────────┘ └────────────────┘
                     │                 │                  │
              Inventory Svc     Billing Svc          BQ Table
```

```bash
# One topic, three subscriptions — each service gets its own copy
gcloud pubsub subscriptions create inventory-sub --topic=orders
gcloud pubsub subscriptions create billing-sub   --topic=orders \
  --push-endpoint=https://billing.run.app/orders
gcloud pubsub subscriptions create analytics-sub --topic=orders \
  --bigquery-table=proj:ds.orders_raw
```

---

### Event-Driven Microservices

```
Service A                     Service B                   Service C
  │                              │                            │
  │ publish(user.created)        │                            │
  ├──────────────────► TOPIC ────► Sub (push) ────► Cloud Run │
  │                              │                (sends email)
  │                              └──────────────► Cloud Function
  │                                             (provision account)
```

---

### CDC Pipeline (Datastream → Pub/Sub → BigQuery)

```
MySQL/PostgreSQL
      │  binlog
      ▼
 Datastream ──────────────► Pub/Sub Topic (CDC events)
                                    │
                          ┌─────────┴──────────┐
                          │                    │
                   BigQuery Sub         Dataflow Job
                   (raw CDC)          (merge/upsert)
                          │                    │
                     BQ Raw Table       BQ Final Table
                                       (SCD Type 2)
```

---

### Replay Pattern (Seek)

```bash
# 1. Ensure subscription retains acked messages
gcloud pubsub subscriptions modify-config my-sub \
  --retain-acked-messages \
  --message-retention-duration=7d

# 2a. Replay from a specific timestamp
gcloud pubsub subscriptions seek my-sub \
  --time=2026-03-10T00:00:00Z

# 2b. Create snapshot (point-in-time marker) before a deploy
gcloud pubsub snapshots create pre-deploy-snapshot \
  --subscription=my-sub

# 2c. Rollback: replay from snapshot after a bad deploy
gcloud pubsub subscriptions seek my-sub \
  --snapshot=pre-deploy-snapshot
```

---

### Cloud Scheduler → Pub/Sub (Scheduled Publishing)

```bash
# Create a Cloud Scheduler job that publishes to Pub/Sub every hour
gcloud scheduler jobs create pubsub hourly-trigger \
  --schedule="0 * * * *" \
  --topic=my-topic \
  --message-body='{"trigger":"hourly_batch","ts":"$(date -u +%Y-%m-%dT%H:%M:%SZ)"}' \
  --attributes=source=scheduler,job=hourly-batch \
  --time-zone=UTC \
  --location=us-central1
```

---

### Pub/Sub → Cloud Functions Trigger

```bash
# Deploy a Cloud Function triggered by Pub/Sub
gcloud functions deploy process-pubsub-message \
  --gen2 \
  --runtime=python311 \
  --trigger-topic=my-topic \
  --entry-point=handle_message \
  --region=us-central1
```

```python
# main.py — Cloud Function entry point
import base64, json
import functions_framework

@functions_framework.cloud_event
def handle_message(cloud_event):
    """Triggered by Pub/Sub message via Eventarc."""
    data = base64.b64decode(cloud_event.data["message"]["data"]).decode()
    attrs = cloud_event.data["message"].get("attributes", {})
    print(f"Received: {data}, attrs={attrs}")
    process(json.loads(data))
    # Returning without exception = ack; raising exception = nack (retry)
```

---

### Pub/Sub as Dataflow Source/Sink

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions
import json

options = PipelineOptions(streaming=True, runner='DataflowRunner',
                          project='my-project', region='us-central1',
                          temp_location='gs://bucket/tmp')

with beam.Pipeline(options=options) as p:
    messages = (
        p
        | 'ReadPubSub' >> beam.io.ReadFromPubSub(
            subscription='projects/my-project/subscriptions/my-sub',
            timestamp_attribute='event_ts',
            with_attributes=True
        )
        | 'ParseJSON' >> beam.Map(
            lambda m: json.loads(m.data.decode()))
        | 'Transform' >> beam.Map(lambda r: {**r, 'processed': True})
        | 'WritePubSub' >> beam.io.WriteToPubSub(
            topic='projects/my-project/topics/output-topic'
        )
    )
```

---

## 14. gcloud CLI & API Quick Reference

### Topics Commands

```bash
gcloud pubsub topics create TOPIC                   # Create topic
gcloud pubsub topics list                           # List all topics
gcloud pubsub topics describe TOPIC                 # Full topic details
gcloud pubsub topics delete TOPIC                   # Delete topic
gcloud pubsub topics update TOPIC \
  --message-retention-duration=14d                  # Update retention
gcloud pubsub topics publish TOPIC \
  --message=DATA --attribute=k=v                    # Publish message
gcloud pubsub topics get-iam-policy TOPIC           # View IAM
gcloud pubsub topics add-iam-policy-binding TOPIC \ # Add IAM binding
  --member=... --role=...
```

---

### Subscriptions Commands

```bash
gcloud pubsub subscriptions create SUB --topic=T    # Create subscription
gcloud pubsub subscriptions list                    # List subscriptions
gcloud pubsub subscriptions describe SUB            # Full subscription details
gcloud pubsub subscriptions delete SUB              # Delete subscription
gcloud pubsub subscriptions pull SUB \
  --limit=10 --auto-ack                             # Pull and ack messages
gcloud pubsub subscriptions pull SUB \
  --limit=1 --format=json                           # Pull as JSON
gcloud pubsub subscriptions ack SUB \
  --ack-ids=ACK_ID_1,ACK_ID_2                       # Manually ack
gcloud pubsub subscriptions modify-ack-deadline SUB \
  --ack-ids=ACK_ID --ack-deadline=60                # Extend deadline
gcloud pubsub subscriptions seek SUB \
  --time=2026-03-15T00:00:00Z                       # Replay from time
gcloud pubsub subscriptions seek SUB \
  --snapshot=SNAPSHOT                               # Replay from snapshot
gcloud pubsub subscriptions modify-push-config SUB \
  --push-endpoint=https://new-endpoint/path         # Update push endpoint
```

---

### Schemas & Snapshots

```bash
# Schemas
gcloud pubsub schemas create SCHEMA --type=AVRO --definition-file=f.avsc
gcloud pubsub schemas list
gcloud pubsub schemas describe SCHEMA
gcloud pubsub schemas validate-message --schema-name=SCHEMA \
  --message-encoding=JSON --message='{"key":"val"}'
gcloud pubsub schemas delete SCHEMA

# Snapshots
gcloud pubsub snapshots create SNAPSHOT --subscription=SUB
gcloud pubsub snapshots list
gcloud pubsub snapshots describe SNAPSHOT
gcloud pubsub snapshots delete SNAPSHOT
```

---

### REST API Quick Reference

```bash
# Publish messages
curl -X POST \
  "https://pubsub.googleapis.com/v1/projects/MY_PROJECT/topics/MY_TOPIC:publish" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {
        "data": "'$(echo -n '{"user_id":"u1"}' | base64)'",
        "attributes": {"source": "api", "env": "prod"}
      }
    ]
  }'

# Pull messages
curl -X POST \
  "https://pubsub.googleapis.com/v1/projects/MY_PROJECT/subscriptions/MY_SUB:pull" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"maxMessages": 10}'

# Acknowledge messages
curl -X POST \
  "https://pubsub.googleapis.com/v1/projects/MY_PROJECT/subscriptions/MY_SUB:acknowledge" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"ackIds": ["ACK_ID_1", "ACK_ID_2"]}'
```

---

### Python Client Library Quick Reference

```python
from google.cloud import pubsub_v1

# ── Publisher ─────────────────────────────────────────────────────────
publisher  = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path("project", "topic")

publisher.publish(topic_path, data=b"payload", attr="value").result()

# Create topic
publisher.create_topic(request={"name": topic_path})

# Delete topic
publisher.delete_topic(request={"topic": topic_path})

# List topics
for topic in publisher.list_topics(request={"project": "projects/project"}):
    print(topic.name)

# ── Subscriber ────────────────────────────────────────────────────────
subscriber = pubsub_v1.SubscriberClient()
sub_path   = subscriber.subscription_path("project", "subscription")

# Streaming pull
future = subscriber.subscribe(sub_path, callback=my_callback)
future.result()    # Block; or future.cancel() to stop

# Synchronous pull
response = subscriber.pull(
    request={"subscription": sub_path, "max_messages": 10}
)
for msg in response.received_messages:
    print(msg.message.data)
    subscriber.acknowledge(request={"subscription": sub_path,
                                    "ack_ids": [msg.ack_id]})

# Create subscription
subscriber.create_subscription(
    request={"name": sub_path, "topic": topic_path, "ack_deadline_seconds": 60}
)

# Delete subscription
subscriber.delete_subscription(request={"subscription": sub_path})
```

---

### Output Formatting

```bash
# JSON output
gcloud pubsub topics list --format=json

# Custom table
gcloud pubsub subscriptions list \
  --format="table(name,topic,pushConfig.pushEndpoint,ackDeadlineSeconds)"

# Filter by topic
gcloud pubsub subscriptions list \
  --filter="topic:my-topic" \
  --format="table(name,pushConfig.pushEndpoint)"

# Extract just subscription names
gcloud pubsub subscriptions list --format="value(name)"
```

---

## 15. Pricing Summary

> ⚠️ Prices are approximate as of early 2026. Verify at [cloud.google.com/pubsub/pricing](https://cloud.google.com/pubsub/pricing).

### Standard Pub/Sub Pricing

| Operation | Price | Notes |
|---|---|---|
| **Message delivery** | $0.040 / GiB | Publish + deliver counted separately |
| **Message storage** | $0.27 / GiB-month | Only for `retainAckedMessages` or topic retention |
| **Seek operations** | $0.40 / GiB | Data replayed via seek |
| **Snapshot storage** | $0.27 / GiB-month | Per snapshot |
| **Free tier** | **First 10 GiB/month** | Per project per month |

> 💡 Each message is counted **twice** — once for publish and once for each subscription delivery. A topic with 3 subscriptions delivering 1 GiB total = 4 GiB billed (1 publish + 3 deliveries).

---

### Pub/Sub Lite Pricing

| Resource | Price | Notes |
|---|---|---|
| **Throughput capacity** | $0.0035 / MiB/s-hour | Per MiB/s provisioned |
| **Storage capacity** | $0.0095 / GiB-month | Per GiB provisioned |

---

### 💰 Cost Optimization Tips

| Strategy | Savings | How |
|---|---|---|
| **Message batching** | 50–90% | Fewer API calls; batch up to 1000 msgs |
| **GZIP compression** | 60–80% | Compress large JSON/text payloads |
| **Right-size retention** | Proportional | Don't retain messages longer than needed |
| **Filter subscriptions** | Moderate | Reduce delivery count with server-side filters |
| **Pub/Sub Lite** | 60–80% | For high-volume, predictable workloads |
| **Disable `retainAckedMessages`** | Significant | Only enable if replay is needed |

---

### Monthly Cost Estimate

```
Low-volume (10 GB/month ingested, 3 subscriptions):
  Publish:   10 GB × $0.040/GiB     = $0.40
  Deliver:   30 GB × $0.040/GiB     = $1.20
  Free tier: -10 GiB                = -$0.40
  Total:                           ~ $1.20/month

High-volume (1 TB/month ingested, 2 subscriptions):
  Publish:   1,000 GiB × $0.040    = $40.00
  Deliver:   2,000 GiB × $0.040    = $80.00
  Storage:   50 GiB × $0.27        = $13.50  (retained msgs)
  Total:                           ~ $133.50/month

  Same with Pub/Sub Lite (100 MiB/s, 500 GiB storage):
  Throughput: 100 × $0.0035 × 730h = $255.50/month   (capacity-based)
  Storage:    500 × $0.0095        = $4.75/month
  Total:                           ~ $260/month
  → Standard Pub/Sub cheaper at this volume if traffic is bursty
```

---

## 16. Quick Reference & Comparison Tables

### Pub/Sub vs. Kafka vs. Cloud Tasks vs. Eventarc

| Feature | Pub/Sub | Kafka | Cloud Tasks | Eventarc |
|---|---|---|---|---|
| **Model** | Topic/sub | Topic/partition | Queue/task | Event routing |
| **Management** | Fully managed | Self-managed or Confluent | Fully managed | Fully managed |
| **Ordering** | Per key | Per partition | FIFO optional | N/A |
| **Delivery** | At-least / exactly-once | At-least-once | At-least-once | At-least-once |
| **Replay** | Via seek (up to 31 days) | ✅ Configurable retention | ❌ | ❌ |
| **Fan-out** | ✅ Native | ✅ Consumer groups | ❌ | ✅ |
| **Rate limiting** | ❌ | ❌ | ✅ Native | ❌ |
| **Delay scheduling** | ❌ | ❌ | ✅ | ❌ |
| **GCP event routing** | ❌ | ❌ | ❌ | ✅ Native |
| **Best for** | Async messaging, fan-out | High-throughput log streaming | Delayed/rate-limited tasks | GCP service event triggers |

---

### Subscription Type Comparison

| Feature | Pull | Push | BigQuery | Cloud Storage |
|---|---|---|---|---|
| **Consumer** | Your code | HTTP endpoint | BQ service | GCS service |
| **Latency** | Low (streaming) | Very low | Seconds | Minutes (batched) |
| **Back-pressure** | ✅ Consumer controls | ❌ Pub/Sub controls | N/A | N/A |
| **Auth** | IAM | OIDC token | IAM | IAM |
| **Infrastructure** | Worker must run | HTTPS endpoint | None | GCS bucket |
| **Best for** | Worker pools, batch | Cloud Run, Functions | Direct analytics | Archival, data lake |
| **Custom logic** | ✅ Full control | ✅ In endpoint | ❌ | ❌ |

---

### Key Subscription Configuration Reference

| Config | Default | Range | Notes |
|---|---|---|---|
| `ackDeadlineSeconds` | 10s | 10–600s | Reset with `modify_ack_deadline` |
| `messageRetentionDuration` | 7d | 10min–31d | Per-subscription override |
| `retainAckedMessages` | false | — | Required for replay |
| `enableExactlyOnceDelivery` | false | — | Higher latency; regional only |
| `enableMessageOrdering` | false | — | Requires ordering key on publish |
| `expirationPolicy.ttl` | 31d | 1d–never | Auto-delete if inactive |
| `maxDeliveryAttempts` (DLQ) | N/A | 5–100 | Requires dead-letter topic |

---

### Pub/Sub Standard vs. Pub/Sub Lite

| Feature | Standard Pub/Sub | Pub/Sub Lite |
|---|---|---|
| **Provisioning** | Auto-scaled | Manual (capacity units) |
| **Routing** | Global | Zonal or Regional |
| **Ordering** | Per ordering key | Per partition |
| **Fan-out** | ✅ Unlimited subscriptions | ✅ Multiple subscriptions |
| **Replay** | Seek (up to 31d) | Seek (per configured retention) |
| **Cost model** | Per-message data | Per provisioned capacity |
| **Schema support** | ✅ Avro / Proto | ❌ |
| **DLQ support** | ✅ | ❌ |
| **Exactly-once** | ✅ | ❌ |
| **Best for** | Variable traffic, fan-out | High-volume, cost-predictable |

---

### Message Attribute vs. Ordering Key vs. Schema

| Feature | Attributes | Ordering Key | Schema |
|---|---|---|---|
| **Purpose** | Metadata / routing / filtering | Guarantee delivery order per key | Enforce message format |
| **Stored in** | Message header | Message header | Topic configuration |
| **Max size** | 256 bytes per key/value | Not specified | Up to 20 MB definition |
| **Use for filtering** | ✅ Server-side filter | ❌ | ❌ |
| **Affects routing** | With filter subscriptions | ✅ Routes to same server | ❌ |
| **Validated at** | Client | Client | Publish time (rejected if invalid) |
| **Typical use** | env, source, type, version | user_id, session_id, entity_id | Avro/Proto data contracts |

---

### Common Anti-Patterns

| ❌ Anti-Pattern | 🔴 Problem | ✅ Fix |
|---|---|---|
| Not setting a DLQ | Poison messages block backlog forever | Always configure DLQ with `--max-delivery-attempts=5` |
| Acking before processing | Message lost if processing fails | Always ack **after** successful processing |
| Nacking without DLQ | Infinite redelivery loop | Add DLQ or add circuit-breaker logic |
| Consumer not idempotent | Duplicate processing on redelivery | Design consumers to handle duplicate messages |
| Using push without OIDC validation | Unauthorized message injection | Always validate OIDC token in push endpoint |
| Retaining acked messages unnecessarily | High storage cost | Only enable if replay is actually needed |
| Single global ordering key | All messages on 1 server = no parallelism | Use fine-grained keys (user_id, entity_id) |
| Subscribing at project level | Over-permissive access | Bind `pubsub.subscriber` at subscription level |
| Long ack deadline with no extension | Redelivery spikes on slow processing | Use `modify_ack_deadline` or streaming pull |
| No backlog alert | Silent consumer failure | Alert on `oldest_unacked_message_age` |

---

*Generated for GCP Pub/Sub | Official Docs: [cloud.google.com/pubsub/docs](https://cloud.google.com/pubsub/docs)*
