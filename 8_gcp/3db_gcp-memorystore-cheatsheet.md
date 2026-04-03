# ⚡ GCP Memorystore — Comprehensive Cheatsheet

> **Audience:** Developers & Cloud Engineers | **Last updated:** 2025
> A single-reference document covering Redis & Memcached theory, data structures, patterns, security, and best practices.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Memorystore for Redis — Architecture & Tiers](#2-memorystore-for-redis--architecture--tiers)
3. [Memorystore for Memcached — Architecture](#3-memorystore-for-memcached--architecture)
4. [Networking & Connectivity](#4-networking--connectivity)
5. [Creating & Managing Instances](#5-creating--managing-instances)
6. [Redis Data Structures & Commands](#6-redis-data-structures--commands)
7. [Connecting & Client Libraries](#7-connecting--client-libraries)
8. [Eviction Policies & Memory Management](#8-eviction-policies--memory-management)
9. [Persistence & Durability](#9-persistence--durability)
10. [Redis Cluster Mode (Memorystore Cluster)](#10-redis-cluster-mode-memorystore-cluster)
11. [Security — Auth, TLS & IAM](#11-security--auth-tls--iam)
12. [High Availability & Failover](#12-high-availability--failover)
13. [Scaling & Performance Tuning](#13-scaling--performance-tuning)
14. [Common Patterns & Use Cases](#14-common-patterns--use-cases)
15. [Monitoring & Observability](#15-monitoring--observability)
16. [gcloud CLI Quick Reference](#16-gcloud-cli-quick-reference)
17. [Pricing Model Summary](#17-pricing-model-summary)
18. [Common Errors & Troubleshooting](#18-common-errors--troubleshooting)

---

## 1. Overview & Key Concepts

**Cloud Memorystore** is Google's fully managed in-memory data store service, offering two OSS-compatible engines: **Redis** and **Memcached**. It eliminates the operational burden of deploying, patching, and scaling caches while staying API-compatible with open-source clients.

### Memorystore vs Alternatives

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                       In-Memory Store Landscape                                  │
├──────────────────┬──────────────┬──────────────┬────────────────┬───────────────┤
│  Memorystore     │  Self-managed│  ElastiCache │  MemoryDB/AWS  │  Redis Cloud  │
│  (GCP)           │  Redis (VM)  │  (AWS)       │  (AWS)         │               │
├──────────────────┼──────────────┼──────────────┼────────────────┼───────────────┤
│ Fully managed    │ Self-managed │ Managed      │ Managed        │ Managed       │
│ VPC-native       │ VM-based     │ VPC          │ VPC            │ Cloud-hosted  │
│ Auto failover    │ Manual       │ Auto failover│ Auto failover  │ Auto failover │
│ OSS-compatible   │ OSS          │ OSS-compat.  │ Redis-compat.  │ OSS-compat.   │
│ GCP IAM          │ OS-level     │ AWS IAM      │ AWS IAM        │ ACL-based     │
│ No patching      │ Manual patch │ No patching  │ No patching    │ No patching   │
│ GCP VPC only     │ Any network  │ AWS VPC only │ AWS VPC only   │ Multi-cloud   │
└──────────────────┴──────────────┴──────────────┴────────────────┴───────────────┘
```

### Core Terminology

| Term | Engine | Description |
|---|---|---|
| **Instance** | Both | Top-level Memorystore resource. One Redis or Memcached deployment. |
| **Tier** | Redis | Basic (no HA) or Standard (HA with replica). |
| **Node** | Memcached | A single Memcached process with dedicated CPU and memory. |
| **Shard** | Redis Cluster | A partition of the keyspace; each shard has a primary + replica. |
| **Replica** | Redis | A read-only copy of a primary for HA and read scaling. |
| **Eviction Policy** | Redis | Controls which keys are removed when `maxmemory` is reached. |
| **Persistence** | Redis | RDB snapshots and/or AOF logs written to durable storage. |
| **Pub/Sub** | Redis | Publish-Subscribe messaging pattern via PUBLISH/SUBSCRIBE. |
| **Keyspace Notifications** | Redis | Server-side events emitted when keys are modified/expired. |
| **Cluster Mode** | Redis | Sharded Redis where keyspace is split across multiple primaries. |
| **Hash Slot** | Redis Cluster | One of 16,384 partitions; each key maps to one slot via CRC16. |
| **App Profile** | Redis | Not applicable (Bigtable concept — use auth token for Redis). |
| **AUTH Token** | Redis | Password-based in-transit authentication (Redis AUTH command). |
| **Connection Pool** | Both | Pre-established set of connections reused across requests. |
| **Auto-discovery** | Memcached | Endpoint that returns the current list of Memcached nodes. |
| **PSA** | Both | Private Service Access — VPC peering method for Memorystore. |

---

## 2. Memorystore for Redis — Architecture & Tiers

### Tier Comparison

| Feature | Basic | Standard | Cluster |
|---|---|---|---|
| **High Availability** | ✗ | ✅ Primary + replica | ✅ Primary + replica per shard |
| **Automatic failover** | ✗ | ✅ ~30 seconds | ✅ Per-shard automatic |
| **Read replicas** | ✗ | Up to 5 | 1 replica per shard |
| **Max memory** | 300 GB | 300 GB | Up to 1+ TB (multi-shard) |
| **Persistence (RDB)** | ✗ | ✅ | ✅ |
| **Cross-zone replication** | ✗ | ✅ Different zones | ✅ |
| **Horizontal scaling** | ✗ | ✗ | ✅ Add/remove shards |
| **Use case** | Dev/test, ephemeral cache | Production HA cache | Very large datasets, high throughput |
| **Relative cost** | $ | $$ | $$$ |

### Redis Version Support

| Version | Status | Notable Features |
|---|---|---|
| Redis 6.x | Supported | ACL users, LMPOP, improved TLS |
| Redis 7.x | Supported (recommended) | Redis Functions, LMPOP, SINTERCARD, sharded pub/sub |

### Memory Size Options

- **Minimum:** 1 GB (Basic/Standard), 1 GB per shard (Cluster)
- **Maximum:** 300 GB (Basic/Standard single instance), scales horizontally with Cluster
- **Standard tier sizes:** 1, 2, 4, 5, 6, 8, 10, 12, 16, 20, 24, 32, 36, 48, 64, 80, 100, 120, 160, 200, 250, 300 GB

### Connection Limits

| Memory Size | Max Connections |
|---|---|
| 1–5 GB | 1,000 |
| 6–10 GB | 4,000 |
| 11–50 GB | 8,000 |
| 51–100 GB | 16,000 |
| > 100 GB | 32,000 |

> **Tip:** Connection limits are per-instance for Basic/Standard. For Cluster, limits apply per shard node. If you regularly hit connection limits, implement connection pooling aggressively or use Cluster.

---

## 3. Memorystore for Memcached — Architecture

### Instance Configuration

| Parameter | Range | Default |
|---|---|---|
| Node count | 1–20 nodes | 1 |
| Node CPU | 1–32 vCPUs | 1 |
| Node memory | 1–256 GB | 1 GB |
| Max total memory | 5 TB (20 nodes × 256 GB) | — |
| Memcached version | 1.6 | 1.6 |

### How Memcached Distributes Keys

Memcached uses **consistent hashing** across nodes — the client library computes which node owns each key:

```
Client (with Memcached library)
    │
    ├── hash("session:abc") → Node 1  (192.168.1.10:11211)
    ├── hash("user:xyz")    → Node 2  (192.168.1.11:11211)
    └── hash("product:001") → Node 3  (192.168.1.12:11211)

Auto-discovery endpoint: 10.0.0.5:11211
  → Returns list of all current node IPs
  → Client library uses this to build the hash ring
```

> ⚠️ **Warning:** Memcached has **no built-in replication** — if a node fails, all keys on that node are lost. Memcached is appropriate only for pure caches where cache misses are acceptable. Never store primary data in Memcached.

### Redis vs Memcached for Memorystore

| Feature | Memorystore Redis | Memorystore Memcached |
|---|---|---|
| **Data structures** | Strings, Lists, Sets, Hashes, Sorted Sets, Streams, Bitmaps, HLL | Strings only |
| **Persistence** | ✅ RDB + AOF (Standard) | ✗ |
| **Replication / HA** | ✅ Standard tier | ✗ (no replication) |
| **Pub/Sub** | ✅ | ✗ |
| **Lua scripting** | ✅ | ✗ |
| **Transactions** | ✅ MULTI/EXEC | ✗ |
| **Cluster / sharding** | ✅ Cluster tier | ✅ Client-side hashing |
| **Multi-threading** | Single-threaded I/O (Redis 6: I/O threads) | Multi-threaded |
| **Max memory per instance** | 300 GB (Standard); unlimited with Cluster | 5 TB |
| **ACL / Auth** | ✅ | ✗ |
| **Keyspace notifications** | ✅ | ✗ |
| **Choose when** | Rich data types, HA, persistence, pub/sub | Simple key-value cache, very high concurrency, multi-threaded workloads |

---

## 4. Networking & Connectivity

### VPC-Native Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GCP Project VPC                              │
│                                                                     │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────────────────┐  │
│  │  GKE Cluster│    │  Cloud Run   │    │  Compute Engine VM    │  │
│  │  (same VPC) │    │  (needs      │    │  (same VPC subnet)    │  │
│  └──────┬──────┘    │  Serverless  │    └───────────┬───────────┘  │
│         │           │  VPC Access) │                │              │
│         │           └──────┬───────┘                │              │
│         │                  │                         │              │
│         └──────────────────┼─────────────────────────┘              │
│                            │ VPC internal traffic only               │
│                  ┌─────────▼──────────┐                             │
│                  │  Memorystore Redis │                             │
│                  │  10.0.0.5:6379     │  ← No public IP             │
│                  │  (PSA peered)      │                             │
│                  └────────────────────┘                             │
│                                                                     │
│  ❌ No direct internet access to Memorystore                        │
│  ✅ Access via Cloud VPN or Interconnect from on-premises           │
└─────────────────────────────────────────────────────────────────────┘
```

### Connection Methods by Compute Platform

| Platform | Connection Method | Extra Config Required |
|---|---|---|
| **Compute Engine** | Direct VPC IP | Same VPC or peered VPC |
| **GKE** | Direct VPC IP | Same VPC; use instance IP as env var |
| **Cloud Run** | Serverless VPC Access connector | Connector in same VPC/region as Memorystore |
| **App Engine Standard** | Serverless VPC Access connector | Same as Cloud Run |
| **App Engine Flexible** | Direct VPC | VPC-native enabled |
| **Cloud Functions** | Serverless VPC Access connector | Same as Cloud Run |
| **On-premises** | Cloud VPN or Interconnect | Authorised network must include on-prem CIDR |

### Private Service Access (PSA) vs Direct Peering

| Method | How it Works | When Used |
|---|---|---|
| **Private Service Access (PSA)** | Allocates a reserved IP range in your VPC; Memorystore peers into it | Default for new instances (`PRIVATE_SERVICE_ACCESS`) |
| **Direct Peering (legacy)** | Memorystore VPC peers directly with customer VPC | Older instances; being deprecated |

```bash
# Set up PSA before creating Memorystore (if not already done)
gcloud compute addresses create google-managed-services-default \
  --global \
  --purpose=VPC_PEERING \
  --prefix-length=16 \
  --network=default

gcloud services vpc-peerings connect \
  --service=servicenetworking.googleapis.com \
  --ranges=google-managed-services-default \
  --network=default \
  --project=my-project
```

### Serverless VPC Access (Cloud Run / Cloud Functions)

```bash
# Create a Serverless VPC Access connector
gcloud compute networks vpc-access connectors create redis-connector \
  --network=default \
  --region=us-central1 \
  --range=10.8.0.0/28

# Deploy Cloud Run service with VPC connector
gcloud run deploy my-service \
  --image=gcr.io/my-project/my-app \
  --vpc-connector=redis-connector \
  --region=us-central1
```

### Firewall Rules

```bash
# Memorystore uses VPC internal IPs — typically no firewall rule needed
# If using Shared VPC or custom firewall policies, allow:
gcloud compute firewall-rules create allow-memorystore-redis \
  --network=default \
  --allow=tcp:6379 \
  --source-ranges=10.0.0.0/8 \
  --description="Allow Redis access from internal VPC"

# Memcached port
gcloud compute firewall-rules create allow-memorystore-memcached \
  --network=default \
  --allow=tcp:11211 \
  --source-ranges=10.0.0.0/8
```

---

## 5. Creating & Managing Instances

### Create Redis Instance

```bash
# ── Basic tier (dev/test, no HA) ─────────────────────────────────────
gcloud redis instances create my-redis-basic \
  --size=5 \
  --region=us-central1 \
  --tier=BASIC \
  --redis-version=redis_7_0 \
  --network=projects/my-project/global/networks/default

# ── Standard tier (HA, production) ──────────────────────────────────
gcloud redis instances create my-redis-prod \
  --size=10 \
  --region=us-central1 \
  --tier=STANDARD_HA \
  --redis-version=redis_7_0 \
  --replica-count=1 \
  --auth-enabled \
  --transit-encryption-mode=SERVER_AUTHENTICATION \
  --maintenance-window-day=SUNDAY \
  --maintenance-window-hour=2 \
  --connect-mode=PRIVATE_SERVICE_ACCESS \
  --network=projects/my-project/global/networks/default

# ── Standard tier with read replicas ────────────────────────────────
gcloud redis instances create my-redis-read-replicas \
  --size=20 \
  --region=us-central1 \
  --tier=STANDARD_HA \
  --redis-version=redis_7_0 \
  --replica-count=3 \
  --read-replicas-mode=READ_REPLICAS_ENABLED \
  --auth-enabled \
  --network=projects/my-project/global/networks/default
```

### Create Memcached Instance

```bash
# Create a Memcached instance (3 nodes, 2 vCPU, 4 GB each)
gcloud memcache instances create my-memcache \
  --node-count=3 \
  --node-cpu=2 \
  --node-memory=4096 \
  --region=us-central1 \
  --network=projects/my-project/global/networks/default
```

### Update & Scale

```bash
# Scale up Redis memory (online for Standard tier)
gcloud redis instances update my-redis-prod \
  --size=20 \
  --region=us-central1

# Enable AUTH on existing instance
gcloud redis instances update my-redis-prod \
  --auth-enabled \
  --region=us-central1

# Disable transit encryption
gcloud redis instances update my-redis-prod \
  --transit-encryption-mode=DISABLED \
  --region=us-central1

# Update maintenance window
gcloud redis instances update my-redis-prod \
  --maintenance-window-day=SATURDAY \
  --maintenance-window-hour=3 \
  --region=us-central1

# Scale Memcached (add nodes)
gcloud memcache instances update my-memcache \
  --node-count=5 \
  --region=us-central1
```

### Get Auth Token & Instance Info

```bash
# Get AUTH token for a Redis instance
gcloud redis instances get-auth-string my-redis-prod \
  --region=us-central1

# Describe Redis instance (get IP, port, version, state)
gcloud redis instances describe my-redis-prod \
  --region=us-central1

# Get Memcached discovery endpoint
gcloud memcache instances describe my-memcache \
  --region=us-central1 \
  --format="value(discoveryEndpoint)"

# List all Redis instances
gcloud redis instances list --region=us-central1

# List all Memcached instances
gcloud memcache instances list --region=us-central1
```

### Manual Failover & Deletion

```bash
# Trigger manual failover (Standard tier — for DR testing)
gcloud redis instances failover my-redis-prod \
  --region=us-central1 \
  --data-protection-mode=LIMITED_DATA_LOSS

# Delete Redis instance
gcloud redis instances delete my-redis-prod --region=us-central1

# Delete Memcached instance
gcloud memcache instances delete my-memcache --region=us-central1
```

---

## 6. Redis Data Structures & Commands

### Data Structure → Use Case Mapping

| Data Structure | Best Use Cases |
|---|---|
| **String** | Cache HTML fragments, JSON blobs, counters, feature flags, session tokens |
| **List** | Job queues, activity feeds, recent items (fixed-length), FIFO/LIFO buffers |
| **Set** | Unique visitors, tags, ACL membership, friends/followers, deduplication |
| **Sorted Set** | Leaderboards, rate limiting, priority queues, time-series index |
| **Hash** | User profiles, session data, product attributes, feature configs |
| **Bitmap** | Daily active users (DAU), feature flags per user ID, attendance tracking |
| **HyperLogLog** | Approximate unique count (< 1% error, fixed 12 KB memory) |
| **Stream** | Event log, message bus, audit trail, IoT sensor feed |

### Strings

```redis
SET    user:123:name "Alice"
GET    user:123:name
SETEX  session:abc  3600  "payload"    # set with TTL (seconds)
SETNX  lock:resource "1"               # set only if Not eXists
INCR   counter:pageviews               # atomic increment
INCRBY counter:pageviews 10
MSET   k1 v1 k2 v2 k3 v3              # multi-set
MGET   k1 k2 k3                        # multi-get
```

### Lists

```redis
LPUSH  queue:jobs "job1" "job2"        # push to HEAD
RPUSH  queue:jobs "job3"               # push to TAIL
LPOP   queue:jobs                      # pop from HEAD (FIFO with RPUSH)
RPOP   queue:jobs                      # pop from TAIL
BRPOP  queue:jobs 30                   # blocking pop (30s timeout) — job queue
LRANGE queue:jobs 0 9                  # get first 10 items (0-indexed)
LLEN   queue:jobs                      # length
LTRIM  recent:items 0 99               # keep only first 100 items
```

### Sets

```redis
SADD     tags:product:001 "electronics" "sale" "new"
SMEMBERS tags:product:001              # all members
SISMEMBER tags:product:001 "sale"      # membership test → 0 or 1
SCARD    tags:product:001              # cardinality (count)
SINTER   tags:product:001 tags:product:002   # intersection
SUNION   tags:product:001 tags:product:002   # union
SDIFF    tags:product:001 tags:product:002   # difference
```

### Sorted Sets

```redis
ZADD   leaderboard 1500 "alice"
ZADD   leaderboard 2100 "bob"
ZADD   leaderboard 1800 "carol"
ZRANGE leaderboard 0 -1 WITHSCORES     # ascending (lowest first)
ZREVRANGE leaderboard 0 2 WITHSCORES   # top 3 (highest first)
ZRANGEBYSCORE leaderboard 1000 2000    # by score range
ZRANK  leaderboard "alice"             # rank (0-based, ascending)
ZREVRANK leaderboard "alice"           # rank (0-based, descending)
ZINCRBY leaderboard 500 "alice"        # add to score
ZREM   leaderboard "alice"             # remove member
ZREMRANGEBYSCORE leaderboard -inf 999  # remove by score range
```

### Hashes

```redis
HSET   user:123 name "Alice" email "alice@example.com" age 30
HGET   user:123 name
HGETALL user:123                       # all fields + values
HMSET  user:123 city "NYC" country "US"
HDEL   user:123 age
HINCRBY user:123 loginCount 1          # atomic field increment
HEXISTS user:123 email                 # field existence → 0 or 1
HKEYS  user:123                        # all field names
HLEN   user:123                        # number of fields
```

### Key Expiry

```redis
EXPIRE  session:abc 3600               # expire in 3600 seconds
EXPIREAT session:abc 1706918400        # expire at Unix timestamp
TTL     session:abc                    # seconds remaining (-1 = no expiry, -2 = gone)
PTTL    session:abc                    # milliseconds remaining
PERSIST session:abc                    # remove expiry (make permanent)
```

### Safe Key Scanning (SCAN vs KEYS)

```redis
# ❌ NEVER use in production — blocks entire Redis server
KEYS user:*

# ✅ SCAN — cursor-based, non-blocking, iterate in batches
SCAN 0 MATCH user:* COUNT 100          # returns (next_cursor, [keys])
# Repeat with returned cursor until cursor = 0 (full scan complete)
```

### Bitmaps & HyperLogLog

```redis
# Bitmap — track daily active users (bit per user ID)
SETBIT  dau:20250115  12345  1         # user 12345 was active on Jan 15
GETBIT  dau:20250115  12345
BITCOUNT dau:20250115                  # total active users today

# HyperLogLog — approximate unique count
PFADD   uniq:pageviews "user-a" "user-b" "user-c"
PFCOUNT uniq:pageviews                 # approximate unique visitors
PFMERGE uniq:week uniq:day1 uniq:day2  # merge multiple HLLs
```

### Streams

```redis
# Append event to stream (auto-generated ID)
XADD  events:orders * orderId ord-001 amount 49.99 status "placed"

# Read new messages from stream (consumer group pattern)
XGROUP CREATE events:orders order-processors $ MKSTREAM
XREADGROUP GROUP order-processors worker-1 COUNT 10 STREAMS events:orders >
XACK  events:orders order-processors <message-id>

# Read range of messages
XRANGE events:orders - + COUNT 100    # oldest → newest
XLEN   events:orders                  # total messages in stream
```

---

## 7. Connecting & Client Libraries

### Python — redis-py

```python
import redis
import os

# ── Basic connection (no TLS, no auth) ───────────────────────────────
r = redis.Redis(
    host=os.environ["REDIS_HOST"],   # e.g., "10.0.0.5"
    port=6379,
    db=0,
    decode_responses=True,           # return str instead of bytes
)

# ── Connection with AUTH token ────────────────────────────────────────
r = redis.Redis(
    host=os.environ["REDIS_HOST"],
    port=6379,
    password=os.environ["REDIS_AUTH_TOKEN"],
    decode_responses=True,
)

# ── TLS connection (SERVER_AUTHENTICATION mode) ───────────────────────
import ssl

r = redis.Redis(
    host=os.environ["REDIS_HOST"],
    port=6379,
    password=os.environ["REDIS_AUTH_TOKEN"],
    ssl=True,
    ssl_cert_reqs=ssl.CERT_REQUIRED,
    ssl_ca_certs="/path/to/server-ca.pem",  # download from GCP Console
    decode_responses=True,
)

# ── Connection pool (production — share across app) ───────────────────
pool = redis.ConnectionPool(
    host=os.environ["REDIS_HOST"],
    port=6379,
    password=os.environ["REDIS_AUTH_TOKEN"],
    max_connections=20,              # size to: max_threads * 2
    decode_responses=True,
    socket_timeout=2,                # seconds — fail fast
    socket_connect_timeout=2,
    retry_on_timeout=True,
)
r = redis.Redis(connection_pool=pool)

# ── Error handling with retry ─────────────────────────────────────────
from redis.exceptions import RedisError, ConnectionError
import time

def redis_get_with_retry(r, key: str, retries: int = 3):
    for attempt in range(retries):
        try:
            return r.get(key)
        except ConnectionError as e:
            if attempt == retries - 1:
                raise
            time.sleep(0.1 * (2 ** attempt))  # exponential backoff
    return None
```

### Node.js — ioredis

```javascript
const Redis = require("ioredis");

// Basic connection with auth
const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: 6379,
  password: process.env.REDIS_AUTH_TOKEN,
  tls: { rejectUnauthorized: false },  // for TLS; set to true with CA cert
  retryStrategy: (times) => Math.min(times * 100, 2000),  // retry backoff
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
});

// Connection pool is built into ioredis (lazyConnect + keepAlive)
redis.on("error", (err) => console.error("Redis error:", err));
redis.on("connect", () => console.log("Redis connected"));

// Usage
await redis.set("key", "value", "EX", 3600);
const val = await redis.get("key");
```

### Java — Lettuce (recommended for Spring Boot)

```java
// Lettuce — async, non-blocking, thread-safe (one client for whole app)
RedisClient redisClient = RedisClient.create(
    RedisURI.Builder.redis(System.getenv("REDIS_HOST"), 6379)
        .withPassword(System.getenv("REDIS_AUTH_TOKEN").toCharArray())
        .withSsl(true)
        .build()
);

StatefulRedisConnection<String, String> connection = redisClient.connect();
RedisCommands<String, String> commands = connection.sync();

commands.set("key", "value");
String value = commands.get("key");

// Always reuse the connection — it is thread-safe
```

### Go — go-redis

```go
import "github.com/redis/go-redis/v9"

rdb := redis.NewClient(&redis.Options{
    Addr:         os.Getenv("REDIS_HOST") + ":6379",
    Password:     os.Getenv("REDIS_AUTH_TOKEN"),
    DB:           0,
    PoolSize:     20,
    DialTimeout:  2 * time.Second,
    ReadTimeout:  2 * time.Second,
    WriteTimeout: 2 * time.Second,
})

ctx := context.Background()
if err := rdb.Set(ctx, "key", "value", time.Hour).Err(); err != nil {
    log.Fatal(err)
}
val, err := rdb.Get(ctx, "key").Result()
```

### Connection Management Best Practices

| Practice | Guideline |
|---|---|
| **Pool size** | `max_connections = max_threads × 2` (never create a pool per request) |
| **Singleton client** | Create ONE client / pool per process — reuse across all requests |
| **Socket timeout** | Set `socket_timeout=2s` — fail fast rather than hang |
| **Retry on timeout** | Enable `retry_on_timeout=True` for transient network blips |
| **Exponential backoff** | Retry with backoff on connection failures during failover |
| **Decode responses** | Use `decode_responses=True` (Python) to avoid manual `.decode()` |
| **Health check** | Call `PING` on startup to validate connectivity before serving traffic |

---

## 8. Eviction Policies & Memory Management

### Eviction Policies

| Policy | Evicts From | Algorithm | Use Case |
|---|---|---|---|
| `noeviction` | Nothing — returns error on write | None | When data must never be lost; requires monitoring |
| `allkeys-lru` | All keys | Least Recently Used | General-purpose cache — default for most caches |
| `volatile-lru` | Keys with TTL only | LRU among TTL keys | Mixed persistent + cache data; protect non-expiring keys |
| `allkeys-lfu` | All keys | Least Frequently Used | Skewed workloads with hot keys |
| `volatile-lfu` | Keys with TTL only | LFU among TTL keys | TTL-based data with frequency skew |
| `allkeys-random` | All keys | Random | Uniform access patterns (rare in practice) |
| `volatile-random` | Keys with TTL only | Random | Simple TTL-based cache with random eviction acceptable |
| `volatile-ttl` | Keys with TTL only | Shortest remaining TTL first | Expire soonest-to-die keys first |

> **Tip:** For a **pure cache** (all data is re-fetchable), use `allkeys-lru`. For **mixed workloads** (some keys must persist), use `volatile-lru` and always set TTLs on cache entries. Avoid `noeviction` for caches — it causes write errors instead of graceful eviction.

### Setting Eviction Policy

```bash
# Set eviction policy on a Memorystore Redis instance
gcloud redis instances update my-redis-prod \
  --maxmemory-policy=allkeys-lru \
  --region=us-central1
```

### Memory Monitoring Commands

```redis
# Total memory info
INFO memory

# Memory usage of a single key (in bytes)
MEMORY USAGE user:123

# Memory usage of a single key with nested sampling
MEMORY USAGE user:123 SAMPLES 5

# Memory doctor (fragmentation and usage advice)
MEMORY DOCTOR

# Check fragmentation ratio (>1.5 = significant fragmentation)
INFO memory | grep mem_fragmentation_ratio
```

### Key Expiry vs Eviction

| Mechanism | Trigger | Control | Data Loss? |
|---|---|---|---|
| **Key Expiry** | TTL reached | `EXPIRE`/`SETEX` per key | No (intentional expiry) |
| **Eviction** | `maxmemory` threshold reached | `maxmemory-policy` global | Yes (LRU/LFU removes data) |

### Memory-Efficient Design

```python
# ❌ BAD: One key per field (high overhead per key ~50-100 bytes metadata)
r.set("user:123:name",   "Alice")
r.set("user:123:email",  "alice@example.com")
r.set("user:123:city",   "NYC")

# ✅ GOOD: Hash groups fields (1 key metadata overhead, many fields)
r.hset("user:123", mapping={
    "name": "Alice",
    "email": "alice@example.com",
    "city": "NYC",
})

# ✅ GOOD: Serialise complex objects with MessagePack (smaller than JSON)
import msgpack
data = {"id": 123, "scores": [1, 2, 3], "active": True}
r.set("obj:123", msgpack.packb(data))
obj = msgpack.unpackb(r.get("obj:123"))
```

---

## 9. Persistence & Durability

### RDB vs AOF Trade-offs

| Aspect | RDB (Snapshot) | AOF (Append-Only File) |
|---|---|---|
| **Mechanism** | Point-in-time snapshot written to disk | Every write appended to log file |
| **Recovery point** | Up to snapshot interval (minutes) | Up to last second (`appendfsync everysec`) |
| **Recovery speed** | Fast (single file load) | Slower (replay all commands) |
| **File size** | Compact binary | Larger; grows with history |
| **Performance impact** | Fork cost at snapshot time | Slight write overhead |
| **Data loss on crash** | Up to snapshot interval | Up to 1 second (with `everysec`) |
| **Memorystore support** | ✅ Automated on Standard tier | ✗ Not available (Memorystore manages internally) |

### How Memorystore Manages Persistence

- **Basic tier:** No persistence — data is lost on restart or failure
- **Standard tier:** Automated RDB snapshots managed by Google; used for failover recovery
- **Cluster tier:** Automated RDB per shard

> ⚠️ **Warning:** Memorystore does **not expose AOF** to customers. If you need sub-second recovery guarantees, use Memorystore as a cache with a durable primary store (Cloud SQL, Spanner, Firestore) as the source of truth.

### Import & Export RDB Snapshots

```bash
# Export RDB snapshot to Cloud Storage
gcloud redis instances export my-redis-prod \
  --destination-file=gs://my-bucket/backups/redis-$(date +%Y%m%d).rdb \
  --region=us-central1

# Import RDB snapshot from Cloud Storage
gcloud redis instances import my-redis-prod \
  --source-file=gs://my-bucket/backups/redis-20250115.rdb \
  --region=us-central1

# Check operation status
gcloud redis operations list --region=us-central1
gcloud redis operations describe OPERATION_ID --region=us-central1
```

> ⚠️ **Warning:** Importing an RDB file **overwrites all existing data** in the instance. The instance must be in a READY state. Export does not block reads/writes but snapshot timing is best-effort.

### Persistence Use Cases

| Use Case | Persistence Needed? | Recommendation |
|---|---|---|
| HTML/API response cache | No | Basic tier; TTL on all keys |
| Session store (re-loginable) | Optional | Standard tier; TTL on sessions |
| Rate limiter counters | No | Basic tier with `volatile-lru` |
| Leaderboard (re-computable) | No | Basic tier |
| Job queue (jobs are critical) | Yes | Standard tier + export RDB to GCS regularly |
| Feature flags store | Optional | Standard tier |
| Primary data store | ⚠️ Avoid | Use Firestore/Spanner; Redis as cache only |

---

## 10. Redis Cluster Mode (Memorystore Cluster)

### Cluster Architecture

```
Memorystore for Redis Cluster
┌──────────────────────────────────────────────────────────────────┐
│                    16,384 Hash Slots                             │
│                                                                  │
│  Shard 0          Shard 1          Shard 2                       │
│  slots 0–5460     slots 5461–10922 slots 10923–16383             │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐               │
│  │ Primary A  │   │ Primary B  │   │ Primary C  │               │
│  │ + Replica  │   │ + Replica  │   │ + Replica  │               │
│  └────────────┘   └────────────┘   └────────────┘               │
│                                                                  │
│  Key routing: CRC16(key) % 16384 → slot → shard → primary       │
└──────────────────────────────────────────────────────────────────┘
```

### Creating a Memorystore Cluster

```bash
# Create a Memorystore for Redis Cluster (3 shards, 1 replica per shard)
gcloud redis clusters create my-redis-cluster \
  --shard-count=3 \
  --replica-count=1 \
  --region=us-central1 \
  --network=projects/my-project/global/networks/default

# Describe cluster (get discovery endpoint)
gcloud redis clusters describe my-redis-cluster --region=us-central1

# Update shard count (horizontal scale-out)
gcloud redis clusters update my-redis-cluster \
  --shard-count=6 \
  --region=us-central1

# Delete cluster
gcloud redis clusters delete my-redis-cluster --region=us-central1

# List clusters
gcloud redis clusters list --region=us-central1
```

### Python Cluster Client

```python
from redis.cluster import RedisCluster, ClusterNode

# Cluster-aware client — automatically discovers all nodes
startup_nodes = [
    ClusterNode(host="10.0.0.5", port=6379),  # any node as entry point
]

rc = RedisCluster(
    startup_nodes=startup_nodes,
    decode_responses=True,
    password=os.environ.get("REDIS_AUTH_TOKEN"),
    skip_full_coverage_check=True,  # required for Memorystore Cluster
)

# Standard commands work transparently
rc.set("user:123", "Alice")
rc.get("user:123")

# Multi-key operations require keys in same hash slot
# Use hash tags {} to force key co-location
rc.set("{user:123}:name",  "Alice")
rc.set("{user:123}:email", "alice@example.com")
rc.mget("{user:123}:name", "{user:123}:email")  # ✅ same slot
```

### Cross-Slot Limitations

| Operation | Single-Key | Multi-Key (same slot) | Multi-Key (different slots) |
|---|---|---|---|
| `GET`/`SET` | ✅ | ✅ | N/A |
| `MGET`/`MSET` | ✅ | ✅ | ❌ CROSSSLOT error |
| `MULTI`/`EXEC` transaction | ✅ | ✅ | ❌ |
| Lua scripts (`EVAL`) | ✅ | ✅ (same slot) | ❌ |
| `SUNION`/`SINTER` | ✅ | ✅ | ❌ |

> **Tip:** Use **hash tags** `{tag}` in key names to force co-location on the same slot: `{user:123}:sessions` and `{user:123}:profile` will always be on the same shard, enabling MGET and transactions.

---

## 11. Security — Auth, TLS & IAM

### Redis AUTH Token

```bash
# Create instance with AUTH enabled
gcloud redis instances create my-redis-prod \
  --auth-enabled \
  --region=us-central1 \
  --size=5 \
  --tier=STANDARD_HA

# Get the AUTH token (stored securely by Memorystore)
AUTH_TOKEN=$(gcloud redis instances get-auth-string my-redis-prod \
  --region=us-central1 \
  --format="value(authString)")

# Rotate AUTH token (generates a new one; old token stops working)
gcloud redis instances update my-redis-prod \
  --region=us-central1 \
  --update-labels=rotate-auth=true   # triggers rotation; update via API
```

### TLS In-Transit Encryption

```bash
# Create instance with TLS (SERVER_AUTHENTICATION mode)
gcloud redis instances create my-redis-tls \
  --transit-encryption-mode=SERVER_AUTHENTICATION \
  --auth-enabled \
  --region=us-central1 \
  --size=5 \
  --tier=STANDARD_HA

# Download server CA certificate
gcloud redis instances describe my-redis-tls \
  --region=us-central1 \
  --format="value(serverCaCerts[0].cert)" > server-ca.pem
```

```python
import redis, ssl, os

# Connect with TLS + AUTH
r = redis.Redis(
    host=os.environ["REDIS_HOST"],
    port=6378,                          # TLS port is 6378 (not 6379)
    password=os.environ["REDIS_AUTH_TOKEN"],
    ssl=True,
    ssl_ca_certs="server-ca.pem",
    ssl_cert_reqs=ssl.CERT_REQUIRED,
    decode_responses=True,
)
r.ping()
```

> ⚠️ **Warning:** When TLS is enabled, Memorystore Redis uses port **6378** instead of the standard 6379. Update all client configurations and firewall rules accordingly.

### IAM Roles

| Role | Resource Level | Permissions |
|---|---|---|
| `roles/redis.admin` | Project | Full control: create, update, delete, read instances |
| `roles/redis.editor` | Project | Create/update instances; no delete |
| `roles/redis.viewer` | Project | View instance metadata only; no data access |
| `roles/memcache.admin` | Project | Full control of Memcached instances |
| `roles/memcache.editor` | Project | Create/update Memcached; no delete |
| `roles/memcache.viewer` | Project | View Memcached metadata |

```bash
# Grant redis.editor to a service account
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-app@my-project.iam.gserviceaccount.com" \
  --role="roles/redis.editor"
```

> **Note:** IAM controls **management plane** access (creating/modifying instances via API/CLI). It does **not** control data plane access (Redis commands). Use Redis AUTH tokens and ACLs for data-plane access control.

### Redis ACL Users (Redis 6+)

```redis
# Create an ACL user with limited permissions
ACL SETUSER app-user on >strongpassword ~cache:* +GET +SET +DEL +EXPIRE

# Create a read-only user
ACL SETUSER readonly-user on >readpass ~* +@read

# List all ACL users
ACL LIST

# Check current user's permissions
ACL WHOAMI
ACL CAT                # list all command categories
```

### Audit Logging

```bash
# Enable Data Access audit logs for Memorystore
# In IAM > Audit Logs: enable ADMIN_READ and DATA_ACCESS for
# Redis (redis.googleapis.com) and Memcache (memcache.googleapis.com)

gcloud logging read \
  'resource.type="redis_instance" AND
   protoPayload.serviceName="redis.googleapis.com"' \
  --limit=20 \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.methodName)"
```

---

## 12. High Availability & Failover

### Standard Tier HA Architecture

```
Standard Tier Instance — Two zones, automatic failover

  Zone: us-central1-a          Zone: us-central1-b
  ┌──────────────────┐         ┌──────────────────┐
  │   PRIMARY NODE   │ ──────► │  REPLICA NODE    │
  │   (read/write)   │  async  │  (read-only)     │
  │   10.0.0.5:6379  │  repl.  │  10.0.0.5:6379   │ ← same VIP
  └──────────────────┘         └──────────────────┘
          │                            │
          └─── Memorystore VIP ────────┘
               Clients connect to VIP; auto-redirected after failover
```

### Failover Timeline

```
T+0s:   Primary node becomes unhealthy (crash, zone failure)
T+0-5s: Memorystore detects failure (health checks)
T+5-30s: Replica promoted to primary; VIP updated
T+30s:  Clients reconnect — existing connections see ECONNRESET
T+30s+: Service restored; new replica being provisioned
```

### Python Retry-on-Failover Pattern

```python
import redis
import time
from redis.exceptions import ConnectionError, TimeoutError, RedisError

pool = redis.ConnectionPool(
    host=os.environ["REDIS_HOST"],
    port=6379,
    password=os.environ.get("REDIS_AUTH_TOKEN"),
    max_connections=20,
    socket_timeout=2,
    socket_connect_timeout=2,
    retry_on_timeout=True,
    decode_responses=True,
)
r = redis.Redis(connection_pool=pool)

def redis_op_with_retry(func, *args, retries=5, **kwargs):
    """Retry Redis operations during failover (~30s window)."""
    for attempt in range(retries):
        try:
            return func(*args, **kwargs)
        except (ConnectionError, TimeoutError) as e:
            if attempt == retries - 1:
                raise
            wait = min(0.5 * (2 ** attempt), 10)  # cap at 10s
            print(f"Redis error (attempt {attempt+1}): {e}. Retrying in {wait}s")
            time.sleep(wait)
        except RedisError:
            raise  # don't retry logical errors (WRONGTYPE, NOAUTH, etc.)

# Usage
value = redis_op_with_retry(r.get, "my-key")
redis_op_with_retry(r.set, "my-key", "value", ex=3600)
```

### Read Replica Endpoints

```python
# Standard tier with read replicas — use separate read endpoint
# Primary endpoint: handles reads AND writes
# Read endpoint: load-balanced across all replicas

import redis

write_client = redis.Redis(host=PRIMARY_HOST,   port=6379, ...)
read_client  = redis.Redis(host=READ_HOST,       port=6379, ...)
# READ_HOST is the read endpoint from instance description

# Use read client for non-critical reads
result = read_client.get("popular-key")

# Always use write client for reads that need consistency after a write
write_client.set("key", "value")
value = write_client.get("key")   # ✅ read-your-writes
```

---

## 13. Scaling & Performance Tuning

### Vertical vs Horizontal Scaling

| Type | Method | Downtime? | Limit |
|---|---|---|---|
| **Vertical (Standard tier)** | `gcloud redis instances update --size=N` | None (online resize) | 300 GB |
| **Vertical (Basic tier)** | `gcloud redis instances update --size=N` | Brief downtime | 300 GB |
| **Horizontal (Cluster)** | `gcloud redis clusters update --shard-count=N` | None (resharding) | Unlimited |
| **Memcached nodes** | `gcloud memcache instances update --node-count=N` | None | 20 nodes |

### Pipeline for Throughput

```python
# ❌ SLOW: N round trips for N commands (network latency multiplied)
for i in range(1000):
    r.set(f"key:{i}", f"value:{i}", ex=3600)

# ✅ FAST: All 1000 commands sent in ONE round trip
with r.pipeline(transaction=False) as pipe:
    for i in range(1000):
        pipe.set(f"key:{i}", f"value:{i}", ex=3600)
    pipe.execute()   # single round-trip

# Pipeline with read + write mix
with r.pipeline(transaction=False) as pipe:
    pipe.get("key:1")
    pipe.get("key:2")
    pipe.incr("counter")
    results = pipe.execute()   # [value1, value2, new_counter]
```

> **Tip:** Use `transaction=False` for pure pipelining (no MULTI/EXEC overhead). Use `transaction=True` only when you need atomic all-or-nothing execution.

### Slow Log Analysis

```redis
# Configure slow log (log commands taking > 10ms = 10000 microseconds)
CONFIG SET slowlog-log-slower-than 10000
CONFIG SET slowlog-max-len 128

# View slow log entries
SLOWLOG GET 10          # last 10 slow commands
SLOWLOG LEN             # total entries in slow log
SLOWLOG RESET           # clear slow log
```

### SCAN vs KEYS

```python
# ✅ Safe SCAN — iterate keys in batches, non-blocking
def scan_keys(r, pattern: str, count: int = 100):
    cursor = 0
    while True:
        cursor, keys = r.scan(cursor, match=pattern, count=count)
        for key in keys:
            yield key
        if cursor == 0:
            break

for key in scan_keys(r, "user:*"):
    print(key)
```

### Connection Pool Sizing Formula

```
pool_size = max_concurrent_requests_per_process × average_redis_ops_per_request

Example:
  - 4 worker threads, each handling 10 req/s
  - Each request does 3 Redis operations
  - pool_size = 4 × 3 = 12  →  set max_connections = 15–20 (add buffer)
```

### redis-benchmark

```bash
# Benchmark from a GCE VM in the same VPC
# Install redis-tools
sudo apt-get install redis-tools

# 100k requests, 50 parallel connections, 1KB payload
redis-benchmark \
  -h REDIS_IP \
  -p 6379 \
  -a AUTH_TOKEN \
  -n 100000 \
  -c 50 \
  -d 1024 \
  -t get,set,incr,lpush,rpop,sadd,hset
```

---

## 14. Common Patterns & Use Cases

### Cache-Aside (Lazy Loading)

```python
import json

def get_product(r, db, product_id: str) -> dict:
    """Fetch from cache; populate from DB on miss."""
    cache_key = f"product:{product_id}"

    # 1. Check cache
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # 2. Cache miss — fetch from primary store
    product = db.query("SELECT * FROM products WHERE id = %s", product_id)
    if product:
        # 3. Store in cache with TTL
        r.set(cache_key, json.dumps(product), ex=300)  # 5 min TTL

    return product
```

### Session Storage

```python
import secrets, json

def create_session(r, user_id: str, ttl: int = 3600) -> str:
    token = secrets.token_hex(32)
    r.setex(f"session:{token}", ttl, json.dumps({"userId": user_id}))
    return token

def get_session(r, token: str) -> dict | None:
    data = r.get(f"session:{token}")
    return json.loads(data) if data else None

def extend_session(r, token: str, ttl: int = 3600):
    r.expire(f"session:{token}", ttl)

def delete_session(r, token: str):
    r.delete(f"session:{token}")
```

### Rate Limiting (Sliding Window)

```python
import time

def is_rate_limited(r, user_id: str, limit: int = 100, window: int = 60) -> bool:
    """Allow `limit` requests per `window` seconds per user."""
    key  = f"ratelimit:{user_id}"
    now  = time.time()
    window_start = now - window

    pipe = r.pipeline(transaction=True)
    pipe.zremrangebyscore(key, "-inf", window_start)  # remove old entries
    pipe.zadd(key, {str(now): now})                   # add current request
    pipe.zcard(key)                                   # count in window
    pipe.expire(key, window)                          # reset TTL
    _, _, request_count, _ = pipe.execute()

    return request_count > limit
```

### Distributed Lock (SET NX PX)

```python
import uuid, time

def acquire_lock(r, resource: str, ttl_ms: int = 5000) -> str | None:
    """Returns lock token if acquired, None if lock held by someone else."""
    token = str(uuid.uuid4())
    acquired = r.set(
        f"lock:{resource}",
        token,
        nx=True,    # SET only if Not eXists
        px=ttl_ms,  # auto-expire in ms (prevents deadlock)
    )
    return token if acquired else None

def release_lock(r, resource: str, token: str) -> bool:
    """Release lock only if we own it (Lua for atomicity)."""
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    return bool(r.eval(script, 1, f"lock:{resource}", token))

# Usage
token = acquire_lock(r, "billing:user:123")
if token:
    try:
        pass  # critical section
    finally:
        release_lock(r, "billing:user:123", token)
```

### Leaderboard (Sorted Set)

```python
def update_score(r, leaderboard: str, user_id: str, score: float):
    r.zadd(leaderboard, {user_id: score})

def get_top_n(r, leaderboard: str, n: int = 10) -> list:
    return r.zrevrange(leaderboard, 0, n - 1, withscores=True)

def get_rank(r, leaderboard: str, user_id: str) -> int | None:
    rank = r.zrevrank(leaderboard, user_id)
    return rank + 1 if rank is not None else None  # 1-based rank
```

### Pub/Sub Messaging

```python
import threading

def publisher(r, channel: str):
    r.publish(channel, '{"event": "order_placed", "orderId": "ord-001"}')

def subscriber(r, channel: str):
    pubsub = r.pubsub()
    pubsub.subscribe(channel)
    for message in pubsub.listen():
        if message["type"] == "message":
            print(f"Received: {message['data']}")

# Run subscriber in background thread
t = threading.Thread(target=subscriber, args=(r, "orders"))
t.daemon = True
t.start()

publisher(r, "orders")
```

### Job Queue (List)

```python
QUEUE_KEY = "jobs:email"

def enqueue(r, job: dict):
    r.lpush(QUEUE_KEY, json.dumps(job))   # push to front

def worker(r):
    """Blocking pop — waits for jobs, processes one at a time."""
    while True:
        _, job_data = r.brpop(QUEUE_KEY, timeout=30)  # pop from tail (FIFO)
        if job_data:
            job = json.loads(job_data)
            process_job(job)
```

### Feature Flags (Hash)

```python
def get_flag(r, flag_name: str) -> bool:
    return r.hget("feature_flags", flag_name) == "true"

def set_flag(r, flag_name: str, enabled: bool):
    r.hset("feature_flags", flag_name, "true" if enabled else "false")

def get_all_flags(r) -> dict:
    return {k: v == "true" for k, v in r.hgetall("feature_flags").items()}
```

---

## 15. Monitoring & Observability

### Key Metrics Reference

| Metric | Alert Threshold | Description |
|---|---|---|
| `redis.googleapis.com/stats/memory/usage_ratio` | > 0.80 | Fraction of maxmemory used; above 80% risks OOM |
| `redis.googleapis.com/stats/cache_hit_ratio` | < 0.80 | Fraction of reads served from cache; low = schema or TTL issue |
| `redis.googleapis.com/stats/connected_clients` | > 80% of limit | Active client connections; pool leak if unexpectedly high |
| `redis.googleapis.com/stats/commands_processed` | — | Total Redis commands per second; use for capacity planning |
| `redis.googleapis.com/stats/keyspace_hits` | — | Cache hits per second |
| `redis.googleapis.com/stats/keyspace_misses` | — | Cache misses per second |
| `redis.googleapis.com/replication/master_repl_offset` | — | Replication offset; compare primary vs replica to detect lag |
| `redis.googleapis.com/stats/evicted_keys` | > 0 | Keys being evicted; increase memory or tighten TTLs |
| `redis.googleapis.com/stats/blocked_clients` | > 0 | Clients waiting on BRPOP/BLPOP; normal for job queues |

### Redis INFO Command

```redis
# Full server info
INFO

# Specific sections
INFO server       # version, uptime, OS
INFO clients      # connected_clients, blocked_clients
INFO memory       # used_memory, maxmemory, fragmentation_ratio
INFO stats        # keyspace_hits, misses, evicted_keys, commands_processed
INFO replication  # role, connected_slaves, replication offset
INFO keyspace     # per-db key count and TTL stats
INFO persistence  # rdb_last_save_time, aof_enabled
```

### Slow Log Configuration

```bash
# Configure slow log threshold (microseconds; 0 = log everything)
gcloud redis instances update my-redis-prod \
  --region=us-central1 \
  --update-redis-config=slowlog-log-slower-than=10000

# Read slow log via redis-cli
redis-cli -h REDIS_IP -p 6379 -a AUTH_TOKEN SLOWLOG GET 25
```

> ⚠️ **Warning:** The `MONITOR` command logs every Redis command in real-time. It **doubles CPU load** and should never be used in production for more than a few seconds of debugging.

### Creating Monitoring Alerts (Cloud Monitoring)

```bash
# Create an alert for memory usage > 80%
gcloud alpha monitoring policies create \
  --display-name="Memorystore Redis High Memory" \
  --condition-display-name="Memory usage > 80%" \
  --condition-filter='resource.type="redis_instance"
    metric.type="redis.googleapis.com/stats/memory/usage_ratio"' \
  --condition-threshold-value=0.80 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-duration=300s \
  --notification-channels=CHANNEL_ID
```

---

## 16. gcloud CLI Quick Reference

### Redis Instance Operations

```bash
# CREATE
gcloud redis instances create NAME --size=N --region=REGION \
  --tier=STANDARD_HA --redis-version=redis_7_0 --auth-enabled

# LIST
gcloud redis instances list --region=REGION

# DESCRIBE (get IP, port, auth status, version)
gcloud redis instances describe NAME --region=REGION

# UPDATE memory size
gcloud redis instances update NAME --size=N --region=REGION

# UPDATE Redis config parameter
gcloud redis instances update NAME --region=REGION \
  --update-redis-config=maxmemory-policy=allkeys-lru

# GET AUTH token
gcloud redis instances get-auth-string NAME --region=REGION

# FAILOVER (manual, Standard tier only)
gcloud redis instances failover NAME \
  --region=REGION \
  --data-protection-mode=LIMITED_DATA_LOSS

# EXPORT RDB to GCS
gcloud redis instances export NAME \
  --destination-file=gs://BUCKET/PATH.rdb --region=REGION

# IMPORT RDB from GCS
gcloud redis instances import NAME \
  --source-file=gs://BUCKET/PATH.rdb --region=REGION

# DELETE
gcloud redis instances delete NAME --region=REGION
```

### Redis Cluster Operations

```bash
# CREATE cluster
gcloud redis clusters create NAME \
  --shard-count=3 --replica-count=1 \
  --region=REGION \
  --network=projects/PROJECT/global/networks/NETWORK

# LIST clusters
gcloud redis clusters list --region=REGION

# DESCRIBE cluster
gcloud redis clusters describe NAME --region=REGION

# UPDATE shard count (scale out)
gcloud redis clusters update NAME --shard-count=N --region=REGION

# GET cluster credentials
gcloud redis clusters describe NAME --region=REGION \
  --format="value(discoveryEndpoints)"

# DELETE cluster
gcloud redis clusters delete NAME --region=REGION
```

### Memcached Instance Operations

```bash
# CREATE
gcloud memcache instances create NAME \
  --node-count=N --node-cpu=C --node-memory=MiB \
  --region=REGION

# LIST
gcloud memcache instances list --region=REGION

# DESCRIBE (get discovery endpoint)
gcloud memcache instances describe NAME --region=REGION

# UPDATE node count
gcloud memcache instances update NAME \
  --node-count=N --region=REGION

# UPDATE Memcached parameters
gcloud memcache instances update NAME --region=REGION \
  --parameters=protocol=ascii,max_item_size=1m

# DELETE
gcloud memcache instances delete NAME --region=REGION
```

### Operations & Maintenance

```bash
# List long-running operations (import, export, failover)
gcloud redis operations list --region=REGION

# Describe an operation
gcloud redis operations describe OPERATION_ID --region=REGION

# List maintenance schedules
gcloud redis instances describe NAME --region=REGION \
  --format="value(maintenanceSchedule)"
```

---

## 17. Pricing Model Summary

> Prices are approximate US regional rates as of 2025. Multi-region and other regions are higher. Always verify at [cloud.google.com/memorystore/docs/redis/pricing](https://cloud.google.com/memorystore/docs/redis/pricing).

### Memorystore for Redis — Basic & Standard Tier

| Memory (GB) | Basic ($/GB-hr) | Standard ($/GB-hr) |
|---|---|---|
| 1–5 GB | $0.049 | $0.098 |
| 6–100 GB | $0.049 | $0.098 |
| 101–300 GB | $0.049 | $0.098 |

> Both tiers charge per **GB provisioned per hour** (not per GB used).

### Memorystore for Redis Cluster

| Resource | Price |
|---|---|
| Per shard-hour (1 GB per shard minimum) | ~$0.098 / GB-hr across all shard nodes |
| Billed per primary + replica node per shard | Each shard = 2 node-hours when replica=1 |

### Memorystore for Memcached

| vCPU / Memory | Price per Node-Hour |
|---|---|
| 1 vCPU / 1 GB | $0.064 |
| 2 vCPU / 4 GB | $0.142 |
| 4 vCPU / 8 GB | $0.268 |
| 8 vCPU / 32 GB | $0.952 |

### No Free Tier

Memorystore has **no free tier** — charges begin from the first GB-hour.

### Cost Comparison

| Configuration | Memory | Hourly Cost | Monthly (~730 hr) |
|---|---|---|---|
| Redis Basic, 5 GB | 5 GB | $0.245 | ~$179 |
| Redis Standard HA, 5 GB | 5 GB | $0.490 | ~$358 |
| Redis Standard HA, 50 GB | 50 GB | $4.90 | ~$3,577 |
| Redis Cluster, 3 shards × 5 GB | 30 GB effective | $2.94 | ~$2,146 |
| Memcached, 3 nodes × 4 GB | 12 GB total | $0.426 | ~$311 |

### Cost Optimisation Strategies

| Strategy | Savings |
|---|---|
| Right-size instance (monitor memory < 60%) | Avoid over-provisioning; downsize if usage is consistently low |
| Use Basic tier for dev/test and non-critical caches | ~50% cheaper than Standard |
| Set aggressive TTLs + `allkeys-lru` eviction | Reclaim memory automatically; defer scaling |
| Use Memcached for simple key-value cache (no HA needed) | Cheaper than Redis Standard for pure cache |
| Use Cluster only when data > 300 GB | Per-shard overhead — use Standard for smaller datasets |
| Enable autoscaling for Cluster (preview) | Scale shards down during off-peak hours |
| Store large values in GCS; keep only references in Redis | Avoids large memory footprint for blob data |

---

## 18. Common Errors & Troubleshooting

| Error / Issue | Cause | Fix |
|---|---|---|
| `NOAUTH Authentication required` | AUTH not provided; wrong token; token rotated | Get fresh token: `gcloud redis instances get-auth-string`; ensure `password=` set in client |
| `OOM command not allowed when used memory > maxmemory` | Maxmemory reached; `noeviction` policy active | Switch eviction to `allkeys-lru`; increase instance memory; add TTLs to cache keys |
| `MOVED <slot> <host>:<port>` | Using non-cluster client with Cluster instance | Use `RedisCluster` client (`redis-py` cluster mode, `ioredis` cluster) |
| `WRONGTYPE Operation against a key holding the wrong kind of value` | Calling a List command on a String key (or vice versa) | Check existing key type with `TYPE keyname`; delete stale key before recreating |
| Connection timeout / `Connection refused` | Wrong IP or port; VPC not peered; firewall blocking | Verify instance IP with `gcloud redis instances describe`; check PSA setup; add Serverless VPC connector for Cloud Run |
| `CLUSTERDOWN The cluster is down` | Shard majority lost; auto-recovery in progress | Wait for automatic replica promotion (~30s); check cluster status in Console; check zone health |
| High memory fragmentation (`mem_fragmentation_ratio > 1.5`) | Many small keys with TTL; frequent key deletions; size mismatches | Restart instance during low-traffic window; upgrade Redis version; defragment with `MEMORY PURGE` |
| `LOADING Redis is loading the dataset in memory` | Instance restarted and is loading RDB snapshot | Wait for load to complete; reduce RDB frequency to speed up restarts |
| Slow commands under load | Missing pipeline; using `KEYS *`; large `LRANGE` on long list; `SMEMBERS` on large set | Use `pipeline()`; replace `KEYS *` with `SCAN`; use `LRANGE` with small range; use `SSCAN` for large sets |
| Client connection pool exhaustion | Pool too small; connections not released; pool created per-request | Use singleton pool; set `max_connections`; check for connection leaks; use context manager |
| Cache hit ratio < 60% | TTL too short; keys evicted too aggressively; cache not warm | Increase TTL; review eviction policy; pre-warm cache after deployment |
| `ERR max number of clients reached` | Too many concurrent connections; pool misconfigured | Reduce pool size across app instances; use Cluster to increase total connection limit; enable connection multiplexing |
| `CROSSSLOT Keys in request don't hash to the same slot` | Multi-key command to different hash slots in Cluster mode | Use hash tags `{tag}` to co-locate related keys; use Lua scripts for atomic multi-key ops within same slot |
| Replication lag after failover | New replica provisioning; heavy write load | Monitor `replication/master_repl_offset` metric; temporarily reduce write rate; wait for replica sync |
| TLS connection fails (`SSL: CERTIFICATE_VERIFY_FAILED`) | CA certificate not downloaded or path incorrect; using port 6379 instead of 6378 | Download CA cert via `gcloud redis instances describe`; use port **6378** for TLS |

### Diagnostic Commands

```bash
# Check instance state and current settings
gcloud redis instances describe my-redis-prod --region=us-central1

# Verify connectivity from a GCE VM
redis-cli -h REDIS_IP -p 6379 -a AUTH_TOKEN PING

# Check memory and eviction stats
redis-cli -h REDIS_IP -p 6379 -a AUTH_TOKEN INFO memory
redis-cli -h REDIS_IP -p 6379 -a AUTH_TOKEN INFO stats

# View slow commands
redis-cli -h REDIS_IP -p 6379 -a AUTH_TOKEN SLOWLOG GET 25

# Count total keys per database
redis-cli -h REDIS_IP -p 6379 -a AUTH_TOKEN INFO keyspace

# Check client connection count
redis-cli -h REDIS_IP -p 6379 -a AUTH_TOKEN CLIENT LIST | wc -l

# Cluster health check
redis-cli -h REDIS_IP -p 6379 -a AUTH_TOKEN CLUSTER INFO
redis-cli -h REDIS_IP -p 6379 -a AUTH_TOKEN CLUSTER NODES
```

---

## Quick Reference Card

```
Engines:              Redis (rich data structures, HA, persistence) | Memcached (simple cache, multi-threaded)
Redis Tiers:          Basic (no HA) | Standard (HA, failover) | Cluster (sharded, horizontal scale)
Default ports:        Redis: 6379 | Redis TLS: 6378 | Memcached: 11211
Network:              VPC-native only — no public IP; Cloud Run needs Serverless VPC Access connector
Max memory:           Redis Standard: 300 GB | Redis Cluster: unlimited (per-shard × shard count)
Hash slots:           16,384 (Redis Cluster) — CRC16(key) % 16384 → slot → shard
Connection pool:      ONE pool per process; pool_size = max_threads × 2
Failover time:        Standard tier ~30s automatic; design clients with retry + exponential backoff
TLS port:             6378 (not 6379) when transit encryption is enabled
Eviction default:     noeviction — always set maxmemory-policy=allkeys-lru for caches
Persistence:          Standard tier: automated RDB; no AOF exposed to customers
Pipeline:             Always use pipeline(transaction=False) for bulk reads/writes
KEYS * warning:       Never use in production — use SCAN with cursor instead
Hash tags:            {tag} in key name forces same hash slot in Cluster mode
No free tier:         Charges start from first GB-hour
Singleton client:     Create ONE redis.Redis / RedisCluster per process — never per request
AUTH token:           Data-plane auth; separate from IAM (which controls management plane only)
```

---

*Reference: [Memorystore Redis Docs](https://cloud.google.com/memorystore/docs/redis) | [Memorystore Memcached Docs](https://cloud.google.com/memorystore/docs/memcached) | [Redis Commands](https://redis.io/commands/) | [redis-py SDK](https://redis-py.readthedocs.io/) | [Pricing](https://cloud.google.com/memorystore/docs/redis/pricing)*
