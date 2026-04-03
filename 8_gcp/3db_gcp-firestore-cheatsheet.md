# 🔥 GCP Firestore — Comprehensive Cheatsheet

> **Audience:** Developers & Cloud Engineers | **Last updated:** 2025
> A single-reference document covering theory, data modeling, queries, security rules, SDK, and best practices.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Data Model & Structure](#2-data-model--structure)
3. [Database Modes & Multi-Database](#3-database-modes--multi-database)
4. [Reading Data](#4-reading-data)
5. [Writing Data](#5-writing-data)
6. [Indexes](#6-indexes)
7. [Security Rules](#7-security-rules)
8. [Transactions & Batched Writes](#8-transactions--batched-writes)
9. [Real-Time Listeners & Offline Support](#9-real-time-listeners--offline-support)
10. [Firestore Emulator & Local Development](#10-firestore-emulator--local-development)
11. [IAM & Security](#11-iam--security)
12. [Import, Export & Backup](#12-import-export--backup)
13. [Client Library — Python](#13-client-library--python)
14. [gcloud CLI Quick Reference](#14-gcloud-cli-quick-reference)
15. [Pricing Model Summary](#15-pricing-model-summary)
16. [Common Errors & Troubleshooting](#16-common-errors--troubleshooting)

---

## 1. Overview & Key Concepts

**Cloud Firestore** is Google's serverless, fully managed NoSQL document database with real-time sync, offline support, ACID transactions, and automatic multi-region replication. It is the next generation of Cloud Datastore and the recommended document database for new GCP applications.

### Firestore vs Other GCP Databases

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         GCP Database Landscape                                  │
├──────────────┬──────────────┬─────────────┬────────────┬────────────────────────┤
│  Firestore   │  Datastore   │  Realtime DB│  Bigtable  │      Spanner           │
├──────────────┼──────────────┼─────────────┼────────────┼────────────────────────┤
│ Document/NoSQL│ Document/NoSQL│ JSON tree  │ Wide-column│ Relational/NewSQL      │
│ Serverless   │ Serverless   │ Serverless  │ Managed    │ Managed                │
│ Real-time    │ No real-time │ Real-time   │ No real-time│ No real-time          │
│ ACID txn     │ ACID txn     │ No ACID     │ Row-level  │ Full ACID + ext.consist│
│ Rich queries │ Limited      │ Limited     │ Key-range  │ Full SQL               │
│ Auto-index   │ Auto-index   │ Manual      │ Manual     │ Manual DDL             │
│ Mobile/web   │ Server-side  │ Mobile/web  │ Analytics  │ Enterprise OLTP        │
│ $–$$         │ $–$$         │ $–$$        │ $$–$$$     │ $$$$–$$$$$             │
└──────────────┴──────────────┴─────────────┴────────────┴────────────────────────┘
```

### Core Terminology

| Term | Description |
|---|---|
| **Database** | Top-level Firestore resource within a project. Multiple databases per project supported. |
| **Collection** | A container of documents. Must contain at least one document to exist. |
| **Document** | A JSON-like record with fields. Max 1 MiB. Has a unique ID within its collection. |
| **Field** | A key-value pair within a document. Values can be any supported type. |
| **Subcollection** | A collection nested inside a document. Enables hierarchical data. |
| **Document ID** | Unique string identifier within a collection. Auto-generated or custom. |
| **Collection Group** | All collections across the database sharing the same collection ID. Queryable together. |
| **Single-Field Index** | Auto-created for every field. Enables simple queries on that field. |
| **Composite Index** | Manually created index on multiple fields. Required for compound queries with ordering. |
| **Transaction** | Atomic read-then-write operation with automatic optimistic concurrency retry. |
| **Batched Write** | Atomic set of up to 500 write operations (no reads). Cheaper than transactions. |
| **Listener** | Real-time subscription (`onSnapshot`) that pushes updates when data changes. |
| **Offline Mode** | Client SDK caches data locally; writes buffer and sync when connection restores. |
| **Native Mode** | Full Firestore feature set: real-time, rich queries, mobile SDKs, collection groups. |
| **Datastore Mode** | Firestore backend with Datastore API; no real-time, no mobile SDKs. |
| **PITR** | Point-In-Time Recovery — restore database to any second within the last 7 days. |

---

## 2. Data Model & Structure

### Document-Collection Hierarchy

```
Firestore Database
├── users/                              ← Collection
│   ├── alice-uid/                      ← Document (ID: alice-uid)
│   │   ├── name: "Alice Smith"         ← Field
│   │   ├── email: "alice@example.com"
│   │   ├── createdAt: Timestamp
│   │   └── orders/                     ← Subcollection
│   │       ├── order-001/              ← Document in subcollection
│   │       │   ├── amount: 49.99
│   │       │   └── status: "shipped"
│   │       └── order-002/
│   └── bob-uid/
│       └── name: "Bob Jones"
└── products/                           ← Top-level Collection
    ├── prod-abc/
    │   ├── title: "Laptop"
    │   ├── tags: ["electronics","sale"] ← Array field
    │   └── specs: {ram: 16, storage: 512} ← Map field
    └── prod-xyz/
```

### Field Value Types

| Type | GoogleSQL Equivalent | Example | Notes |
|---|---|---|---|
| `String` | VARCHAR | `"Hello"` | Max 1 MiB per field |
| `Number` | INT64 / FLOAT64 | `42`, `3.14` | 64-bit integers and doubles |
| `Boolean` | BOOL | `true`, `false` | |
| `Null` | NULL | `null` | Represents absence of value |
| `Timestamp` | TIMESTAMP | `2025-01-15T10:00:00Z` | Nanosecond precision; UTC |
| `GeoPoint` | N/A | `{lat: 37.4, lng: -122.0}` | WGS84 latitude/longitude |
| `Bytes` | BYTES | `<binary data>` | Max 1 MiB per field |
| `Reference` | N/A | `/users/alice-uid` | Pointer to another document |
| `Array` | ARRAY | `[1, "a", true]` | Ordered; can contain any type except Array |
| `Map` | STRUCT | `{key: value}` | Nested key-value pairs |

### Document Size Limits

| Resource | Limit |
|---|---|
| Max document size | 1 MiB (1,048,576 bytes) |
| Max field name length | 1,500 bytes |
| Max depth of nested maps | 20 levels |
| Max writes to one document | ~1 per second (sustained) |
| Max collection group query results | 300 composite index entries per document |

### Document ID Strategies

```json
// Auto-generated ID (recommended for most cases)
// Firestore generates a random, collision-resistant 20-char ID
// e.g., "K7mJhN2xQpL8vR3tY1wZ"
{
  "id": "K7mJhN2xQpL8vR3tY1wZ",
  "name": "Alice",
  "email": "alice@example.com"
}

// Custom ID (when you have a natural key)
// Use when: user UID from Auth, order number, product SKU
{
  "id": "user-alice-001",
  "name": "Alice",
  "email": "alice@example.com"
}
```

> ⚠️ **Warning:** Avoid sequential custom IDs (e.g., `order-001`, `order-002`). Firestore distributes writes across servers based on document path — sequential IDs create **write hotspots**. Use auto-IDs or hash-prefixed IDs for high-write collections.

### Flat vs Nested Data Models

| Approach | Structure | Pros | Cons |
|---|---|---|---|
| **Flat (denormalized)** | All data in one document | Fast single-doc reads; simple queries | Data duplication; sync complexity |
| **Subcollections** | Child data in subcollections | Scalable; no doc size limit | Requires separate queries for children |
| **Root collections** | All entities at root level | Flexible; queryable independently | More complex relationship management |
| **References** | Document stores a reference field | Normalized; consistent | Requires multiple reads to resolve |

```json
// ✅ Denormalized — embed frequently co-read data
{
  "orderId": "ord-001",
  "customerName": "Alice Smith",    // duplicated from users collection
  "customerEmail": "alice@example.com",
  "items": [
    {"productTitle": "Laptop", "quantity": 1, "price": 999.99}
  ],
  "total": 999.99
}

// ✅ Subcollection — for unbounded child data
// users/alice-uid/orders/ord-001  (orders grow independently)
{
  "amount": 999.99,
  "status": "shipped",
  "createdAt": "2025-01-15T10:00:00Z"
}
```

---

## 3. Database Modes & Multi-Database

### Native Mode vs Datastore Mode

| Feature | Native Mode | Datastore Mode |
|---|---|---|
| **Real-time listeners** | ✅ | ✗ |
| **Mobile / Web SDKs** | ✅ | ✗ (server-only) |
| **Offline support** | ✅ | ✗ |
| **Collection group queries** | ✅ | ✗ |
| **Rich query operators** | ✅ | Limited |
| **Multi-database per project** | ✅ | ✗ |
| **API compatibility** | Firestore API | Datastore API (legacy) |
| **Migration from Datastore** | One-way upgrade available | N/A |
| **Best for** | New apps, mobile, web | Legacy Datastore migrations |

> **Tip:** Always choose **Native Mode** for new applications. Datastore Mode exists solely for migrating existing Cloud Datastore applications without code changes.

### Database Locations

| Type | Examples | Replication | Use Case |
|---|---|---|---|
| **Single region** | `us-central1`, `europe-west1` | 3 zones (same region) | Lowest cost; lowest latency for regional apps |
| **Multi-region (US)** | `nam5` (US) | Multiple US regions | 99.999% availability; US-only data residency |
| **Multi-region (EU)** | `eur3` (EU) | Multiple EU regions | 99.999% availability; EU data residency |

> ⚠️ **Warning:** Database location is **permanent** — it cannot be changed after creation. Choose your region carefully based on user geography, data residency requirements, and cost.

### Multi-Database per Project

```bash
# Create the default database (Native mode, US multi-region)
gcloud firestore databases create \
  --location=nam5 \
  --type=firestore-native

# Create a named database (e.g., for staging environment)
gcloud firestore databases create \
  --database=staging \
  --location=us-central1 \
  --type=firestore-native

# Create a database per tenant
gcloud firestore databases create \
  --database=tenant-acme \
  --location=europe-west1 \
  --type=firestore-native

# List all databases in the project
gcloud firestore databases list

# Describe a specific database
gcloud firestore databases describe --database=staging

# Delete a database (irreversible — deletes all data)
gcloud firestore databases delete --database=staging
```

### Connecting to a Named Database (Python)

```python
from google.cloud import firestore

# Connect to default database
db = firestore.Client(project="my-project")

# Connect to a named database
db = firestore.Client(project="my-project", database="staging")
```

---

## 4. Reading Data

### Get a Single Document

```python
# Python
doc_ref = db.collection("users").document("alice-uid")
doc = doc_ref.get()
if doc.exists:
    print(doc.to_dict())
else:
    print("Document not found")
```

```javascript
// JavaScript / Node.js
const docRef = db.collection("users").doc("alice-uid");
const doc = await docRef.get();
if (doc.exists) {
  console.log(doc.data());
} else {
  console.log("Document not found");
}
```

### Get Multiple Documents (Batch Get)

```python
# Python — fetch multiple docs in one RPC
refs = [
    db.collection("users").document("alice-uid"),
    db.collection("users").document("bob-uid"),
]
docs = db.get_all(refs)
for doc in docs:
    print(doc.id, doc.to_dict())
```

### Simple Queries

```python
# Equality filter
query = db.collection("products").where("category", "==", "electronics")

# Inequality filter (requires index if combined with orderBy)
query = db.collection("orders").where("amount", ">", 100)

# Range filter
query = db.collection("orders") \
    .where("amount", ">=", 50) \
    .where("amount", "<=", 500)

# Array contains (single value)
query = db.collection("products").where("tags", "array_contains", "sale")

# Array contains any (OR across values)
query = db.collection("products").where("tags", "array_contains_any", ["sale", "new"])

# in filter (equality OR — up to 30 values)
query = db.collection("orders").where("status", "in", ["shipped", "delivered"])

# not-in filter (up to 10 values)
query = db.collection("orders").where("status", "not_in", ["cancelled", "refunded"])

# Execute and iterate
for doc in query.stream():
    print(doc.id, doc.to_dict())
```

### Compound Queries with Ordering and Limiting

```python
# Compound query — requires composite index
query = (
    db.collection("orders")
    .where("status", "==", "shipped")
    .where("amount", ">", 100)
    .order_by("amount", direction=firestore.Query.DESCENDING)
    .limit(20)
)

results = query.stream()
```

### Cursor-Based Pagination

```python
# Page 1 — get first 10 results and save the last document
page1_query = db.collection("products").order_by("title").limit(10)
page1_docs = list(page1_query.stream())
last_doc = page1_docs[-1]

# Page 2 — start AFTER the last document of page 1
page2_query = (
    db.collection("products")
    .order_by("title")
    .start_after(last_doc)   # exclusive start
    .limit(10)
)

# startAt — inclusive start
# endAt — inclusive end
# endBefore — exclusive end
```

```javascript
// JavaScript pagination
const page1 = await db.collection("products").orderBy("title").limit(10).get();
const lastDoc = page1.docs[page1.docs.length - 1];

const page2 = await db.collection("products")
  .orderBy("title")
  .startAfter(lastDoc)
  .limit(10)
  .get();
```

### Collection Group Queries

```python
# Query ALL "orders" subcollections across ALL parent documents
# Requires a collection group index on the queried field
orders_query = (
    db.collection_group("orders")
    .where("status", "==", "shipped")
    .order_by("createdAt", direction=firestore.Query.DESCENDING)
    .limit(50)
)

for doc in orders_query.stream():
    print(doc.reference.path, doc.to_dict())
```

> **Tip:** Collection group queries require a **composite index** with the `__name__` field and the queried fields. Firestore will prompt you with the exact index creation link when the query fails due to a missing index.

---

## 5. Writing Data

### Creating Documents

```python
# set() — creates or overwrites the document
db.collection("users").document("alice-uid").set({
    "name": "Alice Smith",
    "email": "alice@example.com",
    "createdAt": firestore.SERVER_TIMESTAMP,
})

# set() with merge=True — creates or updates (keeps existing fields)
db.collection("users").document("alice-uid").set(
    {"email": "newalice@example.com"},
    merge=True
)

# add() — creates document with auto-generated ID
new_ref = db.collection("products").add({
    "title": "Wireless Mouse",
    "price": 29.99,
    "tags": ["electronics", "accessories"],
})
print(f"Created: {new_ref[1].id}")  # new_ref = (update_time, DocumentReference)
```

```javascript
// JavaScript
await db.collection("users").doc("alice-uid").set({
  name: "Alice Smith",
  email: "alice@example.com",
  createdAt: firebase.firestore.FieldValue.serverTimestamp(),
});

// Auto-ID
const newRef = await db.collection("products").add({ title: "Mouse", price: 29.99 });
console.log("Created:", newRef.id);
```

### Updating Documents

```python
from google.cloud.firestore_v1 import ArrayUnion, ArrayRemove, Increment

# update() — updates specific fields only (document must exist)
db.collection("users").document("alice-uid").update({
    "email": "updated@example.com",
    "lastLogin": firestore.SERVER_TIMESTAMP,
})

# Dot notation for nested fields (avoids overwriting whole map)
db.collection("products").document("prod-abc").update({
    "specs.ram": 32,           # updates only specs.ram; preserves specs.storage
    "specs.storage": 1024,
})

# Field transforms
db.collection("products").document("prod-abc").update({
    "viewCount": Increment(1),             # atomic counter increment
    "tags": ArrayUnion(["featured"]),      # add to array (no duplicates)
    "tags": ArrayRemove(["sale"]),         # remove from array
})
```

```javascript
// JavaScript field transforms
const { FieldValue } = require("firebase-admin/firestore");

await db.collection("products").doc("prod-abc").update({
  viewCount: FieldValue.increment(1),
  tags: FieldValue.arrayUnion("featured"),
  "specs.ram": 32,                        // dot notation for nested field
});
```

### Deleting Documents and Fields

```python
from google.cloud.firestore_v1 import DELETE_FIELD

# Delete a specific field
db.collection("users").document("alice-uid").update({
    "tempToken": DELETE_FIELD,
})

# Delete an entire document
db.collection("users").document("alice-uid").delete()
```

```javascript
// JavaScript
await db.collection("users").doc("alice-uid").update({
  tempToken: FieldValue.delete(),
});

await db.collection("users").doc("alice-uid").delete();
```

### Writing to Subcollections

```python
# Write to a subcollection
order_ref = (
    db.collection("users")
    .document("alice-uid")
    .collection("orders")
    .document()               # auto-ID
)
order_ref.set({
    "amount": 99.99,
    "status": "pending",
    "createdAt": firestore.SERVER_TIMESTAMP,
})

# Shorthand path notation
db.document("users/alice-uid/orders/ord-001").set({"amount": 49.99})
```

---

## 6. Indexes

### Index Types

| Type | Auto-created? | When Required | Management |
|---|---|---|---|
| **Single-field (ascending)** | ✅ Yes | Simple equality / range queries | Automatic; can be exempted |
| **Single-field (descending)** | ✅ Yes | Descending order queries | Automatic |
| **Single-field (array)** | ✅ Yes | `array_contains` queries | Automatic |
| **Composite** | ✗ No | Compound queries (multiple where + orderBy) | Manual via Console or gcloud |
| **Collection group** | ✗ No | Collection group queries | Manual |

> **Tip:** When a query requires a missing composite index, the Firestore SDK error message includes a **direct URL** to create the exact index needed. Click the URL or copy it into the Console.

### Creating a Composite Index (CLI)

```bash
# Create a composite index via gcloud
gcloud firestore indexes composite create \
  --collection-group=orders \
  --field-config=field-path=status,order=ASCENDING \
  --field-config=field-path=amount,order=DESCENDING \
  --database="(default)"

# List composite indexes
gcloud firestore indexes composite list --database="(default)"

# Delete a composite index
gcloud firestore indexes composite delete INDEX_ID --database="(default)"
```

### firestore.indexes.json (Firebase CLI format)

```json
{
  "indexes": [
    {
      "collectionGroup": "orders",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "status",    "order": "ASCENDING"  },
        { "fieldPath": "amount",    "order": "DESCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "orders",
      "queryScope": "COLLECTION_GROUP",
      "fields": [
        { "fieldPath": "customerId", "order": "ASCENDING" },
        { "fieldPath": "createdAt",  "order": "DESCENDING" }
      ]
    }
  ],
  "fieldOverrides": [
    {
      "collectionGroup": "logs",
      "fieldPath": "rawPayload",
      "indexes": []
    }
  ]
}
```

```bash
# Deploy indexes from firestore.indexes.json
firebase deploy --only firestore:indexes
```

### Index Exemptions (Disable Auto-Indexing)

```bash
# Disable auto-index on a large text field (e.g., article body)
gcloud firestore indexes fields update \
  --collection-group=articles \
  --field=body \
  --disable-indexes \
  --database="(default)"
```

### Index Limits

| Resource | Limit |
|---|---|
| Composite indexes per database | 200 |
| Single-field index exemptions | 200 |
| Fields in one composite index | 100 |
| Index entries per document | 40,000 |
| Max index entry size | 7,500 bytes |

---

## 7. Security Rules

### Rules Syntax Fundamentals

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // ── Public read, no write ────────────────────────────────────────
    match /publicContent/{document} {
      allow read: if true;
      allow write: if false;
    }

    // ── Authenticated users only ─────────────────────────────────────
    match /posts/{postId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null
                    && request.resource.data.authorId == request.auth.uid;
      allow update, delete: if request.auth != null
                            && resource.data.authorId == request.auth.uid;
    }

    // ── Owner-only access ────────────────────────────────────────────
    match /users/{userId} {
      allow read, write: if request.auth != null
                         && request.auth.uid == userId;
    }

    // ── Admin role (custom claim) ────────────────────────────────────
    match /adminData/{document} {
      allow read, write: if request.auth != null
                         && request.auth.token.admin == true;
    }
  }
}
```

### Request & Resource Objects

| Object | Property | Description |
|---|---|---|
| `request.auth` | `.uid` | Authenticated user's UID |
| `request.auth` | `.token` | JWT claims (email, custom claims) |
| `request.auth` | `.token.email_verified` | Whether email is verified |
| `request.resource` | `.data` | Incoming document data (on write) |
| `request.resource` | `.data.keys()` | Set of incoming field names |
| `resource` | `.data` | Existing document data (before write) |
| `resource` | `.id` | Document ID |
| `request.time` | — | Server timestamp of the request |
| `request.method` | — | `create`, `update`, `delete`, `get`, `list` |

### Data Validation Rules

```javascript
match /orders/{orderId} {
  allow create: if request.auth != null
    // Required fields present
    && request.resource.data.keys().hasAll(["amount", "status", "customerId"])
    // Type validation
    && request.resource.data.amount is number
    && request.resource.data.amount > 0
    // Enum validation
    && request.resource.data.status in ["pending", "confirmed"]
    // Owner matches auth
    && request.resource.data.customerId == request.auth.uid
    // No extra fields
    && request.resource.data.keys().hasOnly(
         ["amount", "status", "customerId", "createdAt", "items"]);
}
```

### Wildcards and Recursive Wildcards

```javascript
// Single-segment wildcard — matches one collection segment
match /users/{userId}/orders/{orderId} {
  allow read: if request.auth.uid == userId;
}

// Recursive wildcard — matches ANY path under users/{userId}
// including the document itself and all subcollections
match /users/{userId}/{document=**} {
  allow read, write: if request.auth.uid == userId;
}
```

### Subcollection Rules

```javascript
// Rules for subcollections must be explicitly declared
match /users/{userId} {
  allow read: if request.auth.uid == userId;

  // Subcollection rules inside parent match block
  match /orders/{orderId} {
    allow read: if request.auth.uid == userId;
    allow create: if request.auth.uid == userId
                  && request.resource.data.amount > 0;
  }

  // Nested subcollection
  match /orders/{orderId}/items/{itemId} {
    allow read: if request.auth.uid == userId;
  }
}
```

### Common Helper Functions

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Reusable helper functions
    function isAuthenticated() {
      return request.auth != null;
    }

    function isOwner(userId) {
      return isAuthenticated() && request.auth.uid == userId;
    }

    function isAdmin() {
      return isAuthenticated() && request.auth.token.get("role", "") == "admin";
    }

    function isValidOrder() {
      let data = request.resource.data;
      return data.amount is number
          && data.amount > 0
          && data.status in ["pending", "confirmed"];
    }

    match /orders/{orderId} {
      allow read:   if isAuthenticated();
      allow create: if isOwner(request.resource.data.customerId)
                    && isValidOrder();
      allow update: if isAdmin() || isOwner(resource.data.customerId);
      allow delete: if isAdmin();
    }
  }
}
```

---

## 8. Transactions & Batched Writes

### Transactions vs Batched Writes

| Property | Transaction | Batched Write |
|---|---|---|
| **Reads allowed** | ✅ (read-then-write) | ✗ |
| **Atomic** | ✅ | ✅ |
| **Auto-retry on contention** | ✅ | ✗ |
| **Max operations** | 500 reads + 500 writes | 500 writes |
| **Max data size** | 10 MiB | 10 MiB |
| **Performance** | Slower (lock + retry) | Faster (no locks) |
| **Use when** | You need to read before writing | You know all values upfront |

### Transactions (Python)

```python
from google.cloud import firestore

@firestore.transactional
def transfer_credits(transaction, from_ref, to_ref, amount):
    from_snap = from_ref.get(transaction=transaction)
    to_snap   = to_ref.get(transaction=transaction)

    if from_snap.get("credits") < amount:
        raise ValueError("Insufficient credits")

    transaction.update(from_ref, {"credits": firestore.Increment(-amount)})
    transaction.update(to_ref,   {"credits": firestore.Increment(amount)})

# Execute (auto-retries on ABORTED up to 5 times)
db = firestore.Client()
transaction = db.transaction()
from_ref = db.collection("wallets").document("alice")
to_ref   = db.collection("wallets").document("bob")
transfer_credits(transaction, from_ref, to_ref, 50)
```

### Transactions (JavaScript)

```javascript
await db.runTransaction(async (transaction) => {
  const fromRef = db.collection("wallets").doc("alice");
  const toRef   = db.collection("wallets").doc("bob");

  const fromDoc = await transaction.get(fromRef);
  const toDoc   = await transaction.get(toRef);

  if (fromDoc.data().credits < 50) {
    throw new Error("Insufficient credits");
  }

  transaction.update(fromRef, { credits: FieldValue.increment(-50) });
  transaction.update(toRef,   { credits: FieldValue.increment( 50) });
});
```

> ⚠️ **Warning:** All **reads in a transaction must come before writes**. Performing a read after a write in the same transaction throws an error. Structure your transaction as: read all → validate → write all.

### Batched Writes

```python
# Python — batch up to 500 operations atomically (no reads)
batch = db.batch()

# Mix of set, update, delete in one atomic commit
batch.set(
    db.collection("products").document("prod-1"),
    {"title": "Widget", "price": 9.99}
)
batch.update(
    db.collection("products").document("prod-2"),
    {"price": 14.99, "updatedAt": firestore.SERVER_TIMESTAMP}
)
batch.delete(db.collection("products").document("prod-old"))

# Commit all 3 operations atomically
batch.commit()
```

```javascript
// JavaScript
const batch = db.batch();
batch.set(db.collection("products").doc("prod-1"), { title: "Widget", price: 9.99 });
batch.update(db.collection("products").doc("prod-2"), { price: 14.99 });
batch.delete(db.collection("products").doc("prod-old"));
await batch.commit();
```

### Distributed Counter (Sharded Counter Pattern)

> The **1 write/second per document** sustained limit applies to hot documents. Overcome it with sharded counters:

```python
import random

NUM_SHARDS = 10

def increment_counter(db, counter_id: str, amount: int = 1):
    """Write to a random shard to distribute load."""
    shard_id = random.randint(0, NUM_SHARDS - 1)
    shard_ref = (
        db.collection("counters")
        .document(counter_id)
        .collection("shards")
        .document(str(shard_id))
    )
    shard_ref.set(
        {"count": firestore.Increment(amount)},
        merge=True
    )

def get_counter(db, counter_id: str) -> int:
    """Sum all shards to get total count."""
    shards = (
        db.collection("counters")
        .document(counter_id)
        .collection("shards")
        .stream()
    )
    return sum(shard.get("count", 0) for shard in shards)
```

---

## 9. Real-Time Listeners & Offline Support

### Document Listener (onSnapshot)

```javascript
// JavaScript — listen to a single document
const unsubscribe = db.collection("users").doc("alice-uid")
  .onSnapshot(
    (doc) => {
      if (doc.exists) {
        console.log("Current data:", doc.data());
      } else {
        console.log("Document deleted");
      }
    },
    (error) => {
      console.error("Listener error:", error);
    }
  );

// Detach listener when no longer needed
unsubscribe();
```

### Collection Listener

```javascript
// Listen to a collection with a query
const unsubscribe = db.collection("orders")
  .where("status", "==", "pending")
  .orderBy("createdAt", "desc")
  .onSnapshot((querySnapshot) => {
    querySnapshot.docChanges().forEach((change) => {
      if (change.type === "added")    console.log("New order:", change.doc.data());
      if (change.type === "modified") console.log("Updated:",  change.doc.data());
      if (change.type === "removed")  console.log("Removed:",  change.doc.id);
    });
  });
```

### DocumentSnapshot vs QuerySnapshot

| Property | DocumentSnapshot | QuerySnapshot |
|---|---|---|
| `.exists` | ✅ Boolean | ✗ (use `.empty`) |
| `.data()` | Returns document fields | ✗ |
| `.docs` | ✗ | Array of DocumentSnapshots |
| `.empty` | ✗ | ✅ Boolean |
| `.size` | ✗ | Number of documents |
| `.docChanges()` | ✗ | Array of `{type, doc}` changes |

### Collection Group Listener

```javascript
// Listen to ALL "orders" subcollections across all users
const unsubscribe = db.collectionGroup("orders")
  .where("status", "==", "shipped")
  .onSnapshot((snapshot) => {
    snapshot.forEach((doc) => {
      console.log(doc.ref.path, doc.data());
    });
  });
```

### Offline Persistence

```javascript
// Web SDK — enable offline persistence (disabled by default in web)
import { initializeApp } from "firebase/app";
import { getFirestore, enableIndexedDbPersistence } from "firebase/firestore";

const app = initializeApp(firebaseConfig);
const db  = getFirestore(app);

enableIndexedDbPersistence(db).catch((err) => {
  if (err.code === "failed-precondition") {
    console.warn("Persistence failed: multiple tabs open");
  } else if (err.code === "unimplemented") {
    console.warn("Persistence not supported in this browser");
  }
});
```

```swift
// iOS — offline persistence is enabled by default
// To configure cache size:
let settings = FirestoreSettings()
settings.cacheSettings = PersistentCacheSettings(sizeBytes: NSNumber(value: 100 * 1024 * 1024))
db.settings = settings
```

> **Tip:** In mobile SDKs (iOS, Android), offline persistence is **on by default** with an unlimited cache size. In the Web SDK, you must explicitly enable it via `enableIndexedDbPersistence()`. Offline writes are buffered locally and replayed when connectivity is restored.

---

## 10. Firestore Emulator & Local Development

### Installation & Setup

```bash
# Install Firebase CLI
npm install -g firebase-tools

# Login and initialise project
firebase login
firebase init firestore   # creates firestore.rules and firestore.indexes.json

# Start all emulators (Firestore + Auth + others)
firebase emulators:start

# Start Firestore emulator only
firebase emulators:start --only firestore

# Start with a specific project
firebase emulators:start --project=my-project

# Emulator UI available at: http://localhost:4000
# Firestore emulator runs at: localhost:8080
```

### Connect Client SDKs to the Emulator

```python
# Python — connect to local emulator
import os
os.environ["FIRESTORE_EMULATOR_HOST"] = "localhost:8080"

from google.cloud import firestore
db = firestore.Client(project="demo-project")  # project name doesn't need to be real
```

```javascript
// JavaScript — connect to emulator
import { getFirestore, connectFirestoreEmulator } from "firebase/firestore";

const db = getFirestore(app);
if (process.env.NODE_ENV === "development") {
  connectFirestoreEmulator(db, "localhost", 8080);
}
```

### Export & Import Emulator Data

```bash
# Export current emulator state to a directory
firebase emulators:export ./emulator-data

# Start emulator with pre-loaded data (for reproducible tests)
firebase emulators:start --import=./emulator-data

# Start, run tests, and auto-export on shutdown
firebase emulators:start --import=./emulator-data --export-on-exit=./emulator-data
```

### Seed Data Script

```python
# seed.py — populate emulator with test data
import os
os.environ["FIRESTORE_EMULATOR_HOST"] = "localhost:8080"

from google.cloud import firestore
db = firestore.Client(project="demo-project")

users = [
    {"name": "Alice", "email": "alice@test.com", "role": "admin"},
    {"name": "Bob",   "email": "bob@test.com",   "role": "user"},
]
for user in users:
    db.collection("users").add(user)

print("Seeding complete.")
```

### Emulator in CI/CD (GitHub Actions)

```yaml
# .github/workflows/test.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20" }

      - run: npm install -g firebase-tools

      - name: Run tests with Firestore emulator
        run: |
          firebase emulators:exec \
            --only firestore \
            --project=demo-test \
            "pytest tests/"
```

### Emulator vs Production Differences

| Behaviour | Emulator | Production |
|---|---|---|
| Security Rules enforced | ✅ (if configured) | ✅ |
| Index enforcement | ✗ (queries work without indexes) | ✅ (queries fail without indexes) |
| Document limits | Relaxed | Strict (1 MiB / doc) |
| Write rate limits | None | ~1 write/sec/doc sustained |
| Real-time listeners | ✅ | ✅ |
| Persistence | In-memory (lost on stop) | Durable |
| IAM | ✗ (not enforced) | ✅ |

> ⚠️ **Warning:** The emulator does **not enforce composite index requirements**. Always test queries against a real Firestore instance (dev project) before deploying to production to catch missing index errors.

---

## 11. IAM & Security

### IAM Roles Reference

| Role | Access Level | Use Case |
|---|---|---|
| `roles/owner` | Full project access | Project owner — avoid for apps |
| `roles/editor` | Full Firestore + project edit | Broad; avoid for service accounts |
| `roles/datastore.owner` | Full Firestore control | DBA / admin automation |
| `roles/datastore.user` | Read + write all data | Backend application service accounts |
| `roles/datastore.viewer` | Read-only all data | Reporting, analytics service accounts |
| `roles/datastore.importExportAdmin` | Import/export only | Data pipeline service accounts |
| `roles/datastore.indexAdmin` | Manage indexes | CI/CD deployment service accounts |

```bash
# Grant datastore.user to an application service account
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-app@my-project.iam.gserviceaccount.com" \
  --role="roles/datastore.user"

# Grant import/export admin for data pipelines
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:pipeline@my-project.iam.gserviceaccount.com" \
  --role="roles/datastore.importExportAdmin"
```

### IAM vs Security Rules

| Aspect | IAM | Security Rules |
|---|---|---|
| **Controls** | Server-to-server (Admin SDK, gcloud) | Client-to-Firestore (web, mobile SDK) |
| **Bypasses Rules?** | Yes — Admin SDK ignores Security Rules | No — always applied to client requests |
| **Auth mechanism** | GCP service account / user | Firebase Authentication token |
| **Granularity** | Project or database level | Collection / document / field level |
| **Best for** | Backend services, migrations, data pipelines | Mobile apps, web apps, direct client access |

### Service Account Authentication (Server-Side)

```python
# Explicit service account key (not recommended for GCP-hosted workloads)
from google.oauth2 import service_account
from google.cloud import firestore

creds = service_account.Credentials.from_service_account_file(
    "/path/to/sa-key.json",
    scopes=["https://www.googleapis.com/auth/datastore"],
)
db = firestore.Client(project="my-project", credentials=creds)

# Preferred: Application Default Credentials (ADC) on GCE/Cloud Run/GKE
db = firestore.Client(project="my-project")  # uses ADC automatically
```

### CMEK (Customer-Managed Encryption Keys)

```bash
# Create a KMS key for Firestore
gcloud kms keyrings create firestore-keyring --location=us-central1
gcloud kms keys create firestore-key \
  --location=us-central1 \
  --keyring=firestore-keyring \
  --purpose=encryption

# Create a Firestore database with CMEK
gcloud firestore databases create \
  --database=secure-db \
  --location=us-central1 \
  --kms-key-name=projects/my-project/locations/us-central1/keyRings/firestore-keyring/cryptoKeys/firestore-key
```

### Audit Logging

```bash
# Enable Data Access audit logs for Firestore
# In IAM > Audit Logs: enable DATA_READ and DATA_WRITE for
# Cloud Datastore API (datastore.googleapis.com)

# Query audit logs
gcloud logging read \
  'resource.type="datastore_database" AND
   protoPayload.serviceName="datastore.googleapis.com"' \
  --limit=20 \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.methodName)"
```

---

## 12. Import, Export & Backup

### Managed Export to Cloud Storage

```bash
# Export entire database
gcloud firestore export gs://my-backup-bucket/exports/$(date +%Y%m%d) \
  --database="(default)"

# Export specific collections only
gcloud firestore export gs://my-backup-bucket/exports/partial \
  --collection-ids=users,orders,products \
  --database="(default)"

# Export a named database
gcloud firestore export gs://my-backup-bucket/exports/staging \
  --database=staging

# Check export operation status
gcloud firestore operations list --database="(default)"
gcloud firestore operations describe OPERATION_ID
```

### Import from Cloud Storage

```bash
# Import a full export back into Firestore
gcloud firestore import gs://my-backup-bucket/exports/20250115 \
  --database="(default)"

# Import specific collections only
gcloud firestore import gs://my-backup-bucket/exports/20250115 \
  --collection-ids=users,orders \
  --database="(default)"
```

> ⚠️ **Warning:** Importing **merges** data with existing documents — it does not clear the database first. Documents with the same ID are overwritten; other documents are left untouched. For a clean restore, delete the database and recreate it before importing.

### Scheduled Exports (Cloud Scheduler)

```bash
# Create a service account for the scheduled export
gcloud iam service-accounts create firestore-backup-sa

gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:firestore-backup-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/datastore.importExportAdmin"

gcloud storage buckets add-iam-policy-binding gs://my-backup-bucket \
  --member="serviceAccount:firestore-backup-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.admin"

# Create a Cloud Scheduler job to trigger daily exports at 2 AM
gcloud scheduler jobs create http daily-firestore-backup \
  --schedule="0 2 * * *" \
  --uri="https://firestore.googleapis.com/v1/projects/my-project/databases/(default):exportDocuments" \
  --message-body='{"outputUriPrefix": "gs://my-backup-bucket/exports"}' \
  --oauth-service-account-email=firestore-backup-sa@my-project.iam.gserviceaccount.com \
  --location=us-central1
```

### Point-In-Time Recovery (PITR)

```bash
# Enable PITR on a database (retains data for up to 7 days)
gcloud firestore databases update \
  --database="(default)" \
  --enable-pitr

# Restore database to a specific point in time (creates a NEW database)
gcloud firestore databases restore \
  --source-database="(default)" \
  --destination-database=restored-db \
  --pitr-time=2025-01-15T10:00:00Z

# Disable PITR (reduces storage costs)
gcloud firestore databases update \
  --database="(default)" \
  --no-enable-pitr
```

### Database Backups

```bash
# Create an on-demand backup
gcloud firestore backups create \
  --database="(default)" \
  --location=us-central1

# Create a backup schedule (daily, keep 7 days)
gcloud firestore backup-schedules create \
  --database="(default)" \
  --recurrence=daily \
  --retention=7d

# List backups
gcloud firestore backups list --location=us-central1

# Restore from a backup to a new database
gcloud firestore databases restore \
  --source-backup=projects/my-project/locations/us-central1/backups/BACKUP_ID \
  --destination-database=restored-from-backup
```

---

## 13. Client Library — Python

### Installation & Client Hierarchy

```bash
pip install google-cloud-firestore
```

```
google.cloud.firestore.Client          # Top-level client (project + database)
    └── CollectionReference            # db.collection("users")
        └── DocumentReference          # db.collection("users").document("alice")
            ├── DocumentSnapshot       # result of .get()
            └── CollectionReference    # subcollection: .collection("orders")
```

### Full CRUD + Advanced Operations

```python
from google.cloud import firestore
from google.cloud.firestore_v1 import (
    ArrayUnion, ArrayRemove, Increment,
    DELETE_FIELD, SERVER_TIMESTAMP
)
import datetime

# ── Initialise client ────────────────────────────────────────────────
db = firestore.Client(project="my-project")                  # default database
# db = firestore.Client(project="my-project", database="staging")  # named DB

# ── CREATE a document ────────────────────────────────────────────────
db.collection("users").document("alice-uid").set({
    "name": "Alice Smith",
    "email": "alice@example.com",
    "role": "user",
    "tags": ["vip", "early-adopter"],
    "createdAt": SERVER_TIMESTAMP,
})

# Auto-generated ID
_, new_ref = db.collection("products").add({
    "title": "Widget",
    "price": 9.99,
    "stock": 100,
})
print(f"Created product: {new_ref.id}")

# ── READ a document ──────────────────────────────────────────────────
doc = db.collection("users").document("alice-uid").get()
if doc.exists:
    data = doc.to_dict()
    print(f"{doc.id}: {data['name']}")

# ── UPDATE fields ────────────────────────────────────────────────────
db.collection("users").document("alice-uid").update({
    "email": "alice-new@example.com",
    "lastSeen": SERVER_TIMESTAMP,
    "loginCount": Increment(1),
    "tags": ArrayUnion(["premium"]),
})

# Remove a field
db.collection("users").document("alice-uid").update({
    "tempToken": DELETE_FIELD,
})

# ── DELETE a document ────────────────────────────────────────────────
db.collection("users").document("old-uid").delete()

# ── BATCH WRITE ──────────────────────────────────────────────────────
batch = db.batch()
for i in range(5):
    ref = db.collection("items").document(f"item-{i}")
    batch.set(ref, {"index": i, "value": i * 10})
batch.commit()

# ── TRANSACTION (read-then-write) ────────────────────────────────────
@firestore.transactional
def increment_stock(transaction, product_ref, qty):
    snap = product_ref.get(transaction=transaction)
    new_stock = snap.get("stock") + qty
    transaction.update(product_ref, {"stock": new_stock})

txn = db.transaction()
product_ref = db.collection("products").document(new_ref.id)
increment_stock(txn, product_ref, 50)

# ── COMPOUND QUERY WITH PAGINATION ──────────────────────────────────
page1 = list(
    db.collection("products")
    .where("price", ">", 5.0)
    .order_by("price")
    .limit(10)
    .stream()
)

# Next page using cursor
if page1:
    page2 = (
        db.collection("products")
        .where("price", ">", 5.0)
        .order_by("price")
        .start_after(page1[-1])
        .limit(10)
        .stream()
    )
    for doc in page2:
        print(doc.id, doc.to_dict())

# ── SUBCOLLECTION ACCESS ─────────────────────────────────────────────
order_ref = (
    db.collection("users")
    .document("alice-uid")
    .collection("orders")
    .document()
)
order_ref.set({
    "amount": 49.99,
    "status": "pending",
    "createdAt": SERVER_TIMESTAMP,
})

# ── COLLECTION GROUP QUERY ────────────────────────────────────────────
for doc in (
    db.collection_group("orders")
    .where("status", "==", "pending")
    .order_by("createdAt", direction=firestore.Query.DESCENDING)
    .limit(20)
    .stream()
):
    print(doc.reference.path, doc.get("amount"))

# ── REAL-TIME LISTENER (synchronous watch — for scripts) ─────────────
def on_snapshot(doc_snapshot, changes, read_time):
    for doc in doc_snapshot:
        print(f"Received: {doc.id} → {doc.to_dict()}")

watch = db.collection("orders").where("status", "==", "pending").on_snapshot(on_snapshot)

# Detach listener after 30 seconds
import time; time.sleep(30)
watch.unsubscribe()
```

---

## 14. gcloud CLI Quick Reference

### Database Management

```bash
# CREATE database
gcloud firestore databases create \
  --database=DB_NAME \
  --location=LOCATION \
  --type=firestore-native

# LIST databases
gcloud firestore databases list

# DESCRIBE database
gcloud firestore databases describe --database=DB_NAME

# UPDATE database (enable PITR, delete protection)
gcloud firestore databases update --database=DB_NAME --enable-pitr
gcloud firestore databases update --database=DB_NAME --delete-protection

# DELETE database
gcloud firestore databases delete --database=DB_NAME
```

### Export & Import

```bash
# EXPORT all collections
gcloud firestore export GCS_URI --database=DB_NAME

# EXPORT specific collections
gcloud firestore export GCS_URI \
  --collection-ids=users,orders \
  --database=DB_NAME

# IMPORT
gcloud firestore import GCS_URI --database=DB_NAME

# IMPORT specific collections
gcloud firestore import GCS_URI \
  --collection-ids=users \
  --database=DB_NAME
```

### Backup & PITR

```bash
# CREATE on-demand backup
gcloud firestore backups create --database=DB_NAME --location=LOCATION

# LIST backups
gcloud firestore backups list --location=LOCATION

# DESCRIBE backup
gcloud firestore backups describe BACKUP_ID --location=LOCATION

# DELETE backup
gcloud firestore backups delete BACKUP_ID --location=LOCATION

# RESTORE from backup
gcloud firestore databases restore \
  --source-backup=BACKUP_RESOURCE_NAME \
  --destination-database=NEW_DB_NAME

# RESTORE from PITR
gcloud firestore databases restore \
  --source-database=DB_NAME \
  --destination-database=NEW_DB_NAME \
  --pitr-time=TIMESTAMP

# CREATE backup schedule
gcloud firestore backup-schedules create \
  --database=DB_NAME \
  --recurrence=daily \
  --retention=7d

# LIST backup schedules
gcloud firestore backup-schedules list --database=DB_NAME

# DELETE backup schedule
gcloud firestore backup-schedules delete SCHEDULE_ID --database=DB_NAME
```

### Index Management

```bash
# LIST composite indexes
gcloud firestore indexes composite list --database=DB_NAME

# CREATE composite index
gcloud firestore indexes composite create \
  --collection-group=COLLECTION \
  --field-config=field-path=FIELD,order=ASCENDING \
  --field-config=field-path=FIELD2,order=DESCENDING \
  --database=DB_NAME

# DELETE composite index
gcloud firestore indexes composite delete INDEX_ID --database=DB_NAME

# LIST field index configurations (single-field overrides)
gcloud firestore indexes fields list --database=DB_NAME

# UPDATE field index (disable auto-index on a field)
gcloud firestore indexes fields update \
  --collection-group=COLLECTION \
  --field=FIELD_NAME \
  --disable-indexes \
  --database=DB_NAME
```

### Operations

```bash
# LIST long-running operations
gcloud firestore operations list --database=DB_NAME

# DESCRIBE an operation
gcloud firestore operations describe OPERATION_ID

# CANCEL an operation
gcloud firestore operations cancel OPERATION_ID
```

---

## 15. Pricing Model Summary

> Prices are approximate US multi-region rates as of 2025. Single-region is ~20% cheaper. Always verify at [firebase.google.com/pricing](https://firebase.google.com/pricing) and [cloud.google.com/firestore/pricing](https://cloud.google.com/firestore/pricing).

### Spark Plan (Free Tier — per day)

| Resource | Free Allowance |
|---|---|
| Document reads | 50,000 / day |
| Document writes | 20,000 / day |
| Document deletes | 20,000 / day |
| Stored data | 1 GiB total |
| Network egress | 10 GiB / month |

### Blaze Plan (Pay-As-You-Go)

| Resource | Price |
|---|---|
| Document reads | $0.06 / 100,000 reads |
| Document writes | $0.18 / 100,000 writes |
| Document deletes | $0.02 / 100,000 deletes |
| Storage (multi-region) | $0.18 / GiB / month |
| Storage (single-region) | $0.10 / GiB / month |
| Network egress (to internet) | $0.12 / GiB (first 1 GiB/month free) |
| PITR storage overhead | ~$0.10 / GiB / month (for retained versions) |
| Backup storage | $0.10 / GiB / month |

### Cost Impact of Operations

| Operation | Reads Charged |
|---|---|
| `get()` one document (exists) | 1 read |
| `get()` one document (not exists) | 1 read |
| `collection.stream()` returning N docs | N reads |
| `onSnapshot` initial load (N docs) | N reads |
| `onSnapshot` incremental update (M changed) | M reads |
| Transaction read | 1 read per document |
| Import from GCS | 0 reads charged |

> ⚠️ **Warning:** Real-time listeners charge **1 read per document on initial load**, plus 1 read per changed document on each update. A listener on a 10,000-document collection charges 10,000 reads every time it reconnects. Scope listeners to small, focused queries.

### Cost Optimisation Strategies

| Strategy | Savings |
|---|---|
| Use `select()` to fetch specific fields only | Reduces document payload; same read count, lower bandwidth |
| Denormalize to avoid extra reads | Embed frequently co-read data in one document |
| Use `onSnapshot` with small queries | Pay once per change, not per poll |
| Batch writes instead of individual writes | Same write count; fewer round-trips |
| Use `limit()` on all queries | Avoids unbounded scans charging thousands of reads |
| Cache document IDs client-side to avoid re-reads | `getAll()` for batch gets vs N individual gets |
| Disable auto-indexes on unused large-text fields | Reduces write costs (fewer index entries per write) |
| Set TTL / delete old documents via export+import | Reduces storage cost for time-series data |
| Use PITR only when required | Each day of retention multiplies stored data cost |
| Use single-region for non-critical databases | ~44% cheaper storage vs multi-region |

---

## 16. Common Errors & Troubleshooting

| Error / Issue | Cause | Fix |
|---|---|---|
| `PERMISSION_DENIED` (client SDK) | Security Rules blocking the request | Check Rules Playground in Firebase Console; verify `request.auth` is populated; check match path |
| `PERMISSION_DENIED` (server SDK) | Service account lacks IAM role | Grant `roles/datastore.user` to the service account; verify ADC credentials are correct |
| `NOT_FOUND: database not found` | Wrong database name; database not created | Run `gcloud firestore databases list`; check `--database` parameter spelling |
| `NOT_FOUND: document not found` | Document path typo or document was deleted | Verify path; check if document exists in Console; use `doc.exists` before accessing data |
| `RESOURCE_EXHAUSTED: quota exceeded` | Spark plan daily quota hit | Upgrade to Blaze plan; implement exponential backoff; reduce read/write rate |
| `RESOURCE_EXHAUSTED: write rate too high` | > 1 sustained write/sec to same document | Use sharded counters; redesign schema to distribute writes across documents |
| `FAILED_PRECONDITION: index required` | Missing composite index for compound query | Click the index creation URL in the error message; or add to `firestore.indexes.json` |
| `FAILED_PRECONDITION: PITR not enabled` | Attempted PITR restore without enabling it first | Run `gcloud firestore databases update --enable-pitr`; wait 7 days for full window |
| Transaction retry exhaustion | High contention on same documents; transaction too slow | Reduce transaction scope; use blind writes (batch) where reads are not needed; avoid user input inside transactions |
| Listener not receiving updates | Offline / network issue; Security Rules preventing reads | Check network; verify Rules allow read for the authenticated user; check for errors in `onSnapshot` error callback |
| Slow queries / `DEADLINE_EXCEEDED` | Missing composite index; large unbounded scan; fetching too many documents | Add composite index; add `.limit()`; use cursor-based pagination; use `select()` for fewer fields |
| Large document reads causing latency | Document has large arrays or deeply nested maps | Split data into subcollections; store large blobs in Cloud Storage; use `select()` |
| `Argument 'data' is not a valid Firestore value` | Passing `undefined`, class instances, or circular refs | Use plain JS objects / Python dicts; handle `undefined` → `null` conversions explicitly |
| Security Rules not updating | Cached rules in emulator or client | Restart emulator; wait up to 1 minute for production rules propagation |
| `indexes composite create` fails | Identical index already exists or conflicting fields | Check `gcloud firestore indexes composite list`; delete conflicting index first |
| Collection group query returns no results | Missing collection group index for the queried fields | Create a collection group scope index in `firestore.indexes.json` or via Console |

### Diagnostic Commands

```bash
# Check long-running operations (export, import, index builds)
gcloud firestore operations list --database="(default)"

# View Security Rules evaluation in logs
gcloud logging read \
  'resource.type="firestore.googleapis.com/Database" AND
   protoPayload.methodName="google.firestore.v1.Firestore.RunQuery"' \
  --limit=10

# Check index status
gcloud firestore indexes composite list --database="(default)" \
  --format="table(name,state,queryScope)"

# Verify which database a client connects to
python3 -c "
from google.cloud import firestore
db = firestore.Client(project='my-project')
print('Database:', db._database)
print('Project:',  db.project)
"
```

---

## Quick Reference Card

```
Hierarchy:           Project → Database → Collection → Document → Subcollection
Document size limit: 1 MiB
Write rate per doc:  ~1/sec sustained (use sharded counters for higher rates)
Auto-ID format:      20-char random string (e.g., "K7mJhN2xQpL8vR3tY1wZ")
Index auto-created:  Yes — for every field (single-field ascending/descending)
Composite index:     Manual — required for compound where + orderBy
Transaction retries: Automatic (up to 5 retries on ABORTED)
Batch write limit:   500 operations per batch
PITR window:         Up to 7 days (must be enabled)
Backup retention:    Up to 90 days (configurable)
Offline persistence: Default ON (mobile) | Manual enable (web)
Free tier reads:     50,000 / day (Spark plan)
Free tier writes:    20,000 / day (Spark plan)
Free storage:        1 GiB total (Spark plan)
Admin SDK vs client: Admin SDK bypasses Security Rules; client SDK respects them
Emulator index:      Not enforced — test queries in real Firestore too
```

---

*Reference: [Firestore Docs](https://cloud.google.com/firestore/docs) | [Security Rules](https://firebase.google.com/docs/firestore/security/get-started) | [Python SDK](https://googleapis.dev/python/firestore/latest/) | [Pricing](https://cloud.google.com/firestore/pricing)*
