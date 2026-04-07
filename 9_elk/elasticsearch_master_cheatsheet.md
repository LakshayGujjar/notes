# Elasticsearch Master Reference Cheatsheet
> Senior Engineer Reference — Elasticsearch 7.x / 8.x | Last updated: 2024

---

## Quick Reference Card — 20 Most Common Operations

```bash
# Cluster health
GET /_cluster/health?pretty

# List indices
GET /_cat/indices?v&s=index

# Create index with settings + mappings
PUT /my-index
{ "settings": { "number_of_shards": 1, "number_of_replicas": 1 },
  "mappings": { "properties": { "title": { "type": "text" }, "status": { "type": "keyword" } } } }

# Index a document
POST /my-index/_doc
{ "title": "Hello World", "status": "published", "@timestamp": "2024-01-01T00:00:00Z" }

# Get document by ID
GET /my-index/_doc/abc123

# Search (match all)
GET /my-index/_search
{ "query": { "match_all": {} }, "size": 10 }

# Full-text search
GET /my-index/_search
{ "query": { "match": { "title": "hello world" } } }

# Boolean search
GET /my-index/_search
{ "query": { "bool": { "must": [{ "match": { "title": "error" } }], "filter": [{ "term": { "status": "active" } }] } } }

# Aggregation
GET /my-index/_search
{ "size": 0, "aggs": { "by_status": { "terms": { "field": "status" } } } }

# Bulk index
POST /_bulk
{ "index": { "_index": "my-index" } }
{ "title": "Doc 1", "status": "active" }
{ "index": { "_index": "my-index" } }
{ "title": "Doc 2", "status": "inactive" }

# Update document
POST /my-index/_update/abc123
{ "doc": { "status": "archived" } }

# Delete document
DELETE /my-index/_doc/abc123

# Delete by query
POST /my-index/_delete_by_query
{ "query": { "term": { "status": "deleted" } } }

# Reindex
POST /_reindex
{ "source": { "index": "old-index" }, "dest": { "index": "new-index" } }

# Explain allocation issues
GET /_cluster/allocation/explain

# Index stats
GET /my-index/_stats

# Node stats
GET /_nodes/stats

# Cluster settings
PUT /_cluster/settings
{ "persistent": { "cluster.routing.allocation.enable": "all" } }

# Force merge (for read-only indices)
POST /my-index/_forcemerge?max_num_segments=1

# Refresh index
POST /my-index/_refresh
```

---

<!-- section: architecture -->
## 1. Core Concepts & Architecture

### What is Elasticsearch

Elasticsearch is a distributed, RESTful search and analytics engine built on Apache Lucene. Use cases:
- Full-text search (e-commerce, site search, enterprise search)
- Log/event data analytics (ELK/Elastic Stack)
- APM & observability
- Security analytics (SIEM)
- Vector/semantic search (kNN, ELSER)
- Geospatial search

### Cluster Topology

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Elasticsearch Cluster                        │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │  Master-    │  │  Master-    │  │  Master-    │  ← Quorum = 2   │
│  │  Eligible   │  │  Eligible   │  │  Eligible   │    (3 nodes)    │
│  │  + Data     │  │  + Data     │  │  + Data     │                 │
│  │  node-1     │  │  node-2     │  │  node-3     │                 │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                 │
│         │                │                │                         │
│  ┌──────┴──────┐  ┌──────┴──────┐         │                         │
│  │  Data-Only  │  │  Data-Only  │         │                         │
│  │  node-4     │  │  node-5     │         │                         │
│  └─────────────┘  └─────────────┘         │                         │
│                                           │                         │
│                                  ┌────────┴────────┐                │
│                                  │  Coordinating   │ ← Client-      │
│                                  │  Only node-6    │   facing       │
│                                  └─────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Terminology

| Term | Description |
|------|-------------|
| **Cluster** | One or more nodes sharing the same `cluster.name`, storing all data |
| **Node** | A single running instance of Elasticsearch |
| **Index** | Logical namespace (like a database) containing documents |
| **Shard** | A single Lucene index; unit of distribution and parallelism |
| **Primary Shard** | Original shard; accepts writes |
| **Replica Shard** | Copy of a primary; improves read throughput and HA |
| **Segment** | Immutable Lucene file within a shard; merged over time |
| **Document** | A JSON object stored in an index |
| **Mapping** | Schema definition of document fields and their types |
| **Alias** | Named pointer to one or more indices |

### Node Roles (8.x)

| Role | `node.roles` value | Responsibility |
|------|--------------------|----------------|
| Master-eligible | `master` | Cluster state management, index creation/deletion |
| Data | `data` | Stores shards, executes queries and aggregations |
| Data Content | `data_content` | Long-lived content data (non-time-series) |
| Data Hot | `data_hot` | Active write workload |
| Data Warm | `data_warm` | Less-frequently queried data |
| Data Cold | `data_cold` | Read-only historical data |
| Data Frozen | `data_frozen` | Searchable snapshot mounts |
| Ingest | `ingest` | Pre-indexing pipeline processing |
| Coordinating-only | _(no roles listed)_ | Routes requests, merges search results |
| ML | `ml` | Machine learning jobs |
| Transform | `transform` | Transform jobs |
| Remote cluster client | `remote_cluster_client` | Proxies CCS requests |

```yaml
# elasticsearch.yml — dedicated master node
node.roles: [ master ]

# dedicated data node
node.roles: [ data_hot, data_content ]

# coordinating only — omit all roles
node.roles: []
```

### Split-Brain Prevention

**Elasticsearch 7+**: Uses a Raft-based consensus; configure `cluster.initial_master_nodes` **only** on first bootstrap:
```yaml
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
discovery.seed_hosts: ["10.0.0.1:9300", "10.0.0.2:9300", "10.0.0.3:9300"]
```

> **Note:** Remove `cluster.initial_master_nodes` after the cluster forms. It is dangerous to leave it in after bootstrapping.

**Legacy 6.x**: `discovery.zen.minimum_master_nodes: 2` (for a 3-node cluster, always `floor(N/2) + 1`).

### Inverted Index Internals

```
Document 1: "quick brown fox"
Document 2: "quick lazy dog"

Inverted Index:
┌──────────┬───────────────────────────────────────┐
│  Term    │  Postings List (doc_id, positions)     │
├──────────┼───────────────────────────────────────┤
│ quick    │ [1: [0], 2: [0]]                       │
│ brown    │ [1: [1]]                               │
│ fox      │ [1: [2]]                               │
│ lazy     │ [2: [1]]                               │
│ dog      │ [2: [2]]                               │
└──────────┴───────────────────────────────────────┘
```

Each shard = one Lucene index. Lucene writes segments (immutable). Merges happen in background. `refresh` makes new docs searchable by creating a new in-memory segment.

---

<!-- section: mapping -->
## 2. Index & Mapping Management

### Create Index

```json
PUT /products-000001
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "5s",
    "index.codec": "best_compression"
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "id":          { "type": "keyword" },
      "name":        { "type": "text", "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } } },
      "description": { "type": "text", "analyzer": "english" },
      "price":       { "type": "scaled_float", "scaling_factor": 100 },
      "stock":       { "type": "integer" },
      "created_at":  { "type": "date", "format": "strict_date_optional_time||epoch_millis" },
      "tags":        { "type": "keyword" },
      "location":    { "type": "geo_point" },
      "metadata":    { "type": "flattened" },
      "attributes":  { "type": "nested",
                       "properties": {
                         "key":   { "type": "keyword" },
                         "value": { "type": "keyword" }
                       }
                     }
    }
  }
}
```

### Field Data Types Reference

| Type | Use Case | Notes |
|------|----------|-------|
| `text` | Full-text search | Analyzed, no doc_values by default |
| `keyword` | Exact match, aggregations, sorting | Not analyzed, doc_values enabled |
| `integer` / `long` / `short` / `byte` | Integer numbers | |
| `float` / `double` / `half_float` | Decimal numbers | |
| `scaled_float` | Price, percentage (fixed precision) | `scaling_factor` required |
| `boolean` | true/false | |
| `date` | Dates/timestamps | Stored as epoch milliseconds internally |
| `ip` | IPv4/IPv6 | Supports CIDR range queries |
| `geo_point` | Lat/lon coordinates | |
| `geo_shape` | Polygons, lines, shapes | |
| `object` | Nested JSON object | Flattened into dot-notation — no true isolation |
| `nested` | Array of objects with independent field correlation | Creates hidden documents |
| `flattened` | Arbitrary key-value objects | Single mapping, avoids mapping explosion |
| `wildcard` | Arbitrary text, unstructured log lines | Optimized for wildcard/regex queries |
| `dense_vector` | ML embeddings | `dims`, `index`, `similarity` params |
| `rank_features` | Sparse feature vectors for ranking | |
| `completion` | Suggesters / autocomplete | |
| `alias` | Field alias | Read-only pointer to another field |

### Dynamic Mapping Modes

```json
"dynamic": "true"    // default: auto-create fields
"dynamic": "false"   // ignore unknown fields (don't index, but store in _source)
"dynamic": "strict"  // reject documents with unmapped fields (throws exception)
"dynamic": "runtime" // map as runtime fields (compute at query time)
```

### Multi-Fields

```json
"name": {
  "type": "text",
  "analyzer": "english",
  "fields": {
    "keyword":   { "type": "keyword", "ignore_above": 256 },
    "autocomplete": { "type": "text", "analyzer": "edge_ngram_analyzer" }
  }
}
```
Query with: `"match": { "name": "foo" }` or `"term": { "name.keyword": "foo" }`

### Composable Index Templates (8.x preferred)

```json
// 1. Create component templates
PUT /_component_template/logs-settings
{
  "template": {
    "settings": { "number_of_shards": 1, "number_of_replicas": 1 }
  }
}

PUT /_component_template/logs-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message":    { "type": "text" },
        "level":      { "type": "keyword" }
      }
    }
  }
}

// 2. Create index template
PUT /_index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "priority": 100,
  "composed_of": ["logs-settings", "logs-mappings"],
  "data_stream": {},
  "template": {
    "settings": { "lifecycle.name": "logs-ilm-policy" }
  }
}
```

> **Note:** Composable templates (`_index_template`) take precedence over legacy templates (`_template`). Use priority to break ties.

### Dynamic Templates

```json
"dynamic_templates": [
  {
    "strings_as_keywords": {
      "match_mapping_type": "string",
      "match": "*_id",
      "mapping": { "type": "keyword" }
    }
  },
  {
    "long_as_integer": {
      "match_mapping_type": "long",
      "mapping": { "type": "integer" }
    }
  }
]
```

### Add Fields to Existing Mapping

```json
PUT /my-index/_mapping
{
  "properties": {
    "new_field": { "type": "keyword" }
  }
}
```

> **Note:** You cannot change an existing field's type. Use the Reindex API with a new index.

### Reindex API

```json
POST /_reindex
{
  "source": {
    "index": "old-index",
    "query": { "term": { "status": "active" } },
    "_source": ["field1", "field2"]
  },
  "dest": {
    "index": "new-index",
    "op_type": "create"
  },
  "script": {
    "source": "ctx._source.new_field = ctx._source.old_field; ctx._source.remove('old_field')"
  }
}

// Reindex from remote cluster
POST /_reindex
{
  "source": {
    "remote": { "host": "https://remote-cluster:9200", "username": "user", "password": "pass" },
    "index": "source-index"
  },
  "dest": { "index": "dest-index" }
}
```

---

<!-- section: analyzers -->
## 3. Analyzers, Tokenizers & Token Filters

### Analysis Pipeline

```
Input text → [Char Filters] → Tokenizer → [Token Filters] → Token stream → Index
```

### Test with _analyze API

```json
POST /_analyze
{
  "analyzer": "standard",
  "text": "The Quick Brown FOX jumps over lazy dogs!"
}

// Custom pipeline test
POST /_analyze
{
  "char_filter": ["html_strip"],
  "tokenizer": "standard",
  "filter": ["lowercase", "stop", "snowball"],
  "text": "<p>The Quick Brown FOX jumps over lazy dogs!</p>"
}

// Test against existing index analyzer
POST /my-index/_analyze
{
  "field": "description",
  "text": "Running tests quickly"
}
```

### Built-in Analyzers

| Analyzer | Behavior |
|----------|----------|
| `standard` | Unicode tokenization, lowercase, removes punctuation |
| `simple` | Splits on non-letters, lowercase |
| `whitespace` | Splits on whitespace only, no lowercasing |
| `stop` | Like `simple` + removes stop words |
| `keyword` | No tokenization — entire string as single token |
| `pattern` | Regex-based splitting (default: `\W+`) |
| `fingerprint` | Sorts, deduplicates tokens → single token fingerprint |
| `english` | Stemming, stop words, possessive filter |
| `{language}` | Language-specific analyzers (french, german, etc.) |

### Custom Analyzer

```json
PUT /my-index
{
  "settings": {
    "analysis": {
      "char_filter": {
        "html_cleaner": { "type": "html_strip" },
        "ampersand": { "type": "mapping", "mappings": ["& => and"] }
      },
      "tokenizer": {
        "my_edge_ngram": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 20,
          "token_chars": ["letter", "digit"]
        }
      },
      "filter": {
        "my_synonym": {
          "type": "synonym",
          "synonyms": ["cat,feline => cat", "dog,canine => dog"]
        },
        "my_stop": { "type": "stop", "stopwords": "_english_" }
      },
      "analyzer": {
        "autocomplete": {
          "type": "custom",
          "char_filter": ["html_cleaner"],
          "tokenizer": "my_edge_ngram",
          "filter": ["lowercase", "my_stop"]
        },
        "search_autocomplete": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "my_stop"]
        }
      }
    }
  }
}
```

> **Note:** For autocomplete, use `edge_ngram` at index time and `standard` at search time — different analyzers per field via `search_analyzer`.

---

<!-- section: search -->
## 4. Search API & Query DSL

### Search Request Anatomy

```json
GET /index1,index2/_search
{
  "from": 0,
  "size": 10,
  "timeout": "5s",
  "_source": ["field1", "field2"],
  "fields": ["@timestamp"],
  "sort": [{ "created_at": "desc" }, "_score"],
  "min_score": 0.5,
  "track_total_hits": true,
  "query": { ... },
  "aggs": { ... },
  "highlight": {
    "fields": { "description": { "fragment_size": 150 } }
  },
  "suggest": {
    "title-suggest": {
      "text": "quick brwon fox",
      "term": { "field": "title" }
    }
  }
}
```

### Query Context vs Filter Context

| | Query Context | Filter Context |
|--|---------------|----------------|
| Calculates score? | **Yes** | No |
| Cached? | No | **Yes** (node query cache) |
| Use for | Relevance ranking, full-text | Exact conditions, date ranges, status checks |
| DSL location | `must`, `should` | `filter`, `must_not` |

### Full-Text Queries

```json
// match — standard full-text
{ "match": { "title": { "query": "quick fox", "operator": "and", "fuzziness": "AUTO" } } }

// match_phrase — term order and proximity
{ "match_phrase": { "title": { "query": "quick brown fox", "slop": 1 } } }

// match_phrase_prefix — autocomplete (last term prefix)
{ "match_phrase_prefix": { "title": { "query": "quick bro", "max_expansions": 50 } } }

// multi_match — search across multiple fields
{ "multi_match": {
    "query": "quick fox",
    "fields": ["title^3", "description", "tags^2"],
    "type": "best_fields",
    "tie_breaker": 0.3
  }
}
// multi_match types: best_fields, most_fields, cross_fields, phrase, phrase_prefix, bool_prefix

// query_string — Lucene syntax support
{ "query_string": {
    "query": "title:(quick OR brown) AND status:active",
    "default_field": "description",
    "default_operator": "AND"
  }
}

// simple_query_string — safe user-facing query string (no exceptions thrown)
{ "simple_query_string": {
    "query": "\"quick brown\" +fox -lazy",
    "fields": ["title", "description"],
    "default_operator": "and"
  }
}
```

### Term-Level Queries

```json
// term — exact match
{ "term": { "status": { "value": "published", "boost": 2.0 } } }

// terms — one of many values
{ "terms": { "status": ["published", "featured"] } }

// terms lookup — values from another document
{ "terms": { "user_id": { "index": "users", "id": "user123", "path": "friend_ids" } } }

// range
{ "range": { "price": { "gte": 10, "lte": 100 } } }
{ "range": { "created_at": { "gte": "now-7d/d", "lt": "now/d", "time_zone": "+05:30" } } }

// exists — field is present and non-null
{ "exists": { "field": "updated_at" } }

// prefix
{ "prefix": { "username": { "value": "john" } } }

// wildcard (leading wildcard is very slow — avoid)
{ "wildcard": { "filename": { "value": "*.log", "case_insensitive": true } } }

// regexp
{ "regexp": { "username": { "value": "[a-z]{3,12}", "flags": "ALL" } } }

// fuzzy — edit distance
{ "fuzzy": { "title": { "value": "quikc", "fuzziness": "AUTO", "max_expansions": 50 } } }

// ids
{ "ids": { "values": ["id1", "id2", "id3"] } }
```

### Compound Queries

```json
// bool — the workhorse
{
  "bool": {
    "must":     [{ "match": { "title": "elasticsearch" } }],
    "should":   [{ "term": { "tags": "featured" } }],
    "must_not": [{ "term": { "status": "deleted" } }],
    "filter":   [{ "range": { "price": { "lte": 100 } } }],
    "minimum_should_match": 1,
    "boost": 1.5
  }
}

// boosting — demote matches
{
  "boosting": {
    "positive": { "match": { "description": "apple" } },
    "negative": { "term": { "tags": "electronics" } },
    "negative_boost": 0.2
  }
}

// constant_score — apply fixed score to filter
{
  "constant_score": {
    "filter": { "term": { "status": "published" } },
    "boost": 1.2
  }
}

// dis_max — max score from sub-queries
{
  "dis_max": {
    "queries": [
      { "match": { "title": "quick" } },
      { "match": { "body": "quick" } }
    ],
    "tie_breaker": 0.3
  }
}

// function_score
{
  "function_score": {
    "query": { "match": { "title": "elasticsearch" } },
    "functions": [
      { "gauss": { "date": { "origin": "now", "scale": "7d", "offset": "1d", "decay": 0.5 } } },
      { "filter": { "term": { "featured": true } }, "weight": 3 }
    ],
    "score_mode": "sum",
    "boost_mode": "multiply"
  }
}
```

### Joining Queries

```json
// nested query
{
  "nested": {
    "path": "attributes",
    "query": {
      "bool": {
        "must": [
          { "term": { "attributes.key": "color" } },
          { "term": { "attributes.value": "red" } }
        ]
      }
    },
    "score_mode": "avg",
    "inner_hits": { "size": 5 }
  }
}

// has_child — parent/child relationship (join field)
{ "has_child": { "type": "comment", "query": { "match": { "body": "urgent" } }, "score_mode": "max" } }

// has_parent
{ "has_parent": { "parent_type": "article", "query": { "term": { "category": "tech" } } } }
```

### kNN Query (8.x Approximate)

```json
GET /products/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [0.1, 0.2, 0.3, ...],
    "k": 10,
    "num_candidates": 100,
    "filter": { "term": { "status": "active" } }
  },
  "query": {
    "match": { "title": "wireless headphones" }
  },
  "rank": {
    "rrf": { "window_size": 100, "rank_constant": 60 }
  }
}
```

### Explain, Profile, Validate

```json
// Why did doc X score Y?
GET /my-index/_explain/doc-id
{ "query": { "match": { "title": "fox" } } }

// Profile query execution
GET /my-index/_search
{ "profile": true, "query": { "match": { "title": "fox" } } }

// Validate query (rewrite=true for expansion details)
GET /my-index/_validate/query?rewrite=true
{ "query": { "query_string": { "query": "title:quick~2" } } }
```

---

<!-- section: aggregations -->
## 5. Aggregations

### Aggregation Anatomy

```json
GET /orders/_search
{
  "size": 0,
  "query": { "range": { "date": { "gte": "now-30d" } } },
  "aggs": {
    "agg_name": {
      "agg_type": { /* params */ },
      "meta": { "description": "optional metadata" },
      "aggs": { /* sub-aggregations */ }
    }
  }
}
```

### Metric Aggregations

```json
// stats — all at once
{ "stats": { "field": "price" } }
// → { "count": 100, "min": 5, "max": 500, "avg": 87.3, "sum": 8730 }

// extended_stats — adds variance, std dev
{ "extended_stats": { "field": "price" } }

// cardinality (HyperLogLog — approximate)
{ "cardinality": { "field": "user_id", "precision_threshold": 40000 } }

// percentiles
{ "percentiles": { "field": "latency_ms", "percents": [50, 75, 95, 99, 99.9] } }

// top_hits — retrieve actual documents
{ "top_hits": { "size": 3, "sort": [{ "date": "desc" }], "_source": ["title", "date"] } }

// scripted_metric — custom aggregation
{
  "scripted_metric": {
    "init_script": "state.totals = []",
    "map_script": "state.totals.add(doc['price'].value * doc['qty'].value)",
    "combine_script": "double total = 0; for (t in state.totals) total += t; return total",
    "reduce_script": "double total = 0; for (t in states) total += t; return total"
  }
}
```

### Bucket Aggregations

```json
// terms
{ "terms": { "field": "category", "size": 20, "order": { "_count": "desc" }, "min_doc_count": 1 } }

// multi_terms (8.x)
{ "multi_terms": { "terms": [{ "field": "region" }, { "field": "category" }], "size": 10 } }

// range
{ "range": { "field": "price", "ranges": [{ "to": 50 }, { "from": 50, "to": 200 }, { "from": 200 }] } }

// date_histogram
{
  "date_histogram": {
    "field": "@timestamp",
    "calendar_interval": "1d",
    "time_zone": "Asia/Kolkata",
    "format": "yyyy-MM-dd",
    "min_doc_count": 0,
    "extended_bounds": { "min": "now-30d/d", "max": "now/d" }
  }
}

// histogram
{ "histogram": { "field": "price", "interval": 50, "min_doc_count": 1 } }

// filter / filters
{ "filter": { "term": { "status": "active" } } }
{
  "filters": {
    "filters": {
      "active": { "term": { "status": "active" } },
      "inactive": { "term": { "status": "inactive" } }
    }
  }
}

// nested + reverse_nested
{
  "aggs": {
    "nested_attrs": {
      "nested": { "path": "attributes" },
      "aggs": {
        "attr_keys": {
          "terms": { "field": "attributes.key" },
          "aggs": {
            "back_to_root": {
              "reverse_nested": {},
              "aggs": { "avg_price": { "avg": { "field": "price" } } }
            }
          }
        }
      }
    }
  }
}

// significant_terms — statistical outliers vs background
{ "significant_terms": { "field": "tags", "background_filter": { "term": { "category": "tech" } } } }

// rare_terms (8.x) — opposite of significant, low frequency
{ "rare_terms": { "field": "error_code", "max_doc_count": 5 } }
```

### Pipeline Aggregations

```json
// avg_bucket — avg of bucket metric values
{
  "sales_per_month": { "date_histogram": { "field": "date", "calendar_interval": "month" },
    "aggs": { "monthly_sales": { "sum": { "field": "amount" } } }
  },
  "avg_monthly": { "avg_bucket": { "buckets_path": "sales_per_month>monthly_sales" } }
}

// derivative — rate of change
{ "derivative": { "buckets_path": "monthly_sales" } }

// cumulative_sum
{ "cumulative_sum": { "buckets_path": "monthly_sales" } }

// moving_fn (more flexible than moving_avg)
{
  "moving_fn": {
    "buckets_path": "daily_sales",
    "window": 7,
    "script": "MovingFunctions.unweightedAvg(values)"
  }
}

// bucket_script — compute across sibling aggs
{
  "revenue": { "sum": { "field": "price" } },
  "cost":    { "sum": { "field": "cost" } },
  "margin":  {
    "bucket_script": {
      "buckets_path": { "rev": "revenue", "cost": "cost" },
      "script": "(params.rev - params.cost) / params.rev * 100"
    }
  }
}

// bucket_selector — filter buckets by metric condition
{
  "bucket_selector": {
    "buckets_path": { "sales": "monthly_sales" },
    "script": "params.sales > 1000"
  }
}

// bucket_sort — sort and paginate buckets
{
  "bucket_sort": {
    "sort": [{ "monthly_sales": { "order": "desc" } }],
    "from": 0,
    "size": 5
  }
}
```

---

<!-- section: ilm -->
## 6. Index Lifecycle Management (ILM)

### ILM Phase Overview

```
  Write                Read/Query            Occasional Query    Archive       Purge
    │                      │                       │               │              │
┌───▼──────┐  rollover  ┌──▼──────┐  rollover  ┌──▼──────┐  ┌───▼──────┐  ┌───▼──────┐
│   HOT    │ ──────────► │  WARM   │ ──────────► │  COLD   │  │ FROZEN  │  │  DELETE  │
│ max 50GB │            │ shrink  │            │read_only│  │ srchable │  │          │
│ max 30d  │            │ force   │            │ allocate│  │ snapshot │  │          │
└──────────┘            │ merge   │            └─────────┘  └──────────┘  └──────────┘
                        │ allocate│
                        └─────────┘
```

### ILM Policy

```json
PUT /_ilm/policy/logs-ilm-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "30d",
            "max_docs": 10000000
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "set_priority": { "priority": 50 },
          "readonly": {},
          "allocate": { "number_of_replicas": 0 },
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "set_priority": { "priority": 0 },
          "searchable_snapshot": { "snapshot_repository": "my-snapshot-repo" }
        }
      },
      "frozen": {
        "min_age": "90d",
        "actions": {
          "searchable_snapshot": { "snapshot_repository": "my-snapshot-repo" }
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

### Bootstrap Index + Alias Pattern

```json
// 1. Create first index with the rollover alias
PUT /logs-2024.01.01-000001
{
  "aliases": {
    "logs-write": { "is_write_index": true },
    "logs-read":  {}
  }
}

// 2. Apply ILM policy
PUT /logs-2024.01.01-000001/_settings
{ "index.lifecycle.name": "logs-ilm-policy", "index.lifecycle.rollover_alias": "logs-write" }

// 3. Write to alias; rollover triggers automatically
POST /logs-write/_doc
{ "@timestamp": "2024-01-01T00:00:00Z", "message": "test" }

// Manual rollover
POST /logs-write/_rollover
{ "conditions": { "max_docs": 1 } }
```

### Data Streams (Modern Alternative)

```json
// Requires composable index template with data_stream: {}
PUT /_index_template/logs-ds-template
{
  "index_patterns": ["logs-app-*"],
  "data_stream": {},
  "priority": 200,
  "template": {
    "settings": { "index.lifecycle.name": "logs-ilm-policy" },
    "mappings": { "properties": { "@timestamp": { "type": "date" } } }
  }
}

// Data stream auto-creates on first document
POST /logs-app-prod/_doc
{ "@timestamp": "2024-01-01T12:00:00Z", "message": "app started" }

// List data streams
GET /_data_stream/logs-app-*

// Rollover data stream
POST /logs-app-prod/_rollover
```

### ILM Management APIs

```bash
GET /_ilm/policy                          # List all policies
GET /_ilm/policy/my-policy               # Get specific policy
DELETE /_ilm/policy/my-policy            # Delete policy
GET /my-index/_ilm/explain               # ILM status for index
POST /my-index/_ilm/retry                # Retry failed step
POST /_ilm/stop                          # Stop ILM (maintenance)
POST /_ilm/start                         # Start ILM
GET /_ilm/status                         # ILM running status
POST /my-index/_ilm/move_to_step         # Manually advance step
```

---

<!-- section: settings -->
## 7. Cluster & Index Settings

### Cluster Settings

```json
// Persistent (survives restart) vs Transient (lost on restart — avoid in 8.x)
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "all",
    "cluster.routing.rebalance.enable": "all",
    "cluster.routing.allocation.disk.threshold_enabled": true,
    "cluster.routing.allocation.disk.watermark.low": "85%",
    "cluster.routing.allocation.disk.watermark.high": "90%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "95%",
    "indices.recovery.max_bytes_per_sec": "200mb",
    "cluster.max_shards_per_node": 1000,
    "search.max_buckets": 65536
  }
}

// Temporarily disable shard allocation (for rolling restarts)
PUT /_cluster/settings
{ "persistent": { "cluster.routing.allocation.enable": "primaries" } }

// Re-enable after node restart
PUT /_cluster/settings
{ "persistent": { "cluster.routing.allocation.enable": null } }
```

### Index Settings

```json
// Dynamic settings (changeable on live index)
PUT /my-index/_settings
{
  "number_of_replicas": 2,
  "refresh_interval": "30s",
  "index.max_result_window": 50000,
  "index.routing.allocation.require.zone": "us-east-1a",
  "index.routing.allocation.exclude.box_type": "slow",
  "index.blocks.write": false,
  "index.blocks.read_only": false,
  "index.blocks.read_only_allow_delete": false
}

// Static settings (set at creation time only)
// number_of_shards, index.codec, index.routing_partition_size
```

### Shard Allocation Awareness (Multi-AZ)

```yaml
# elasticsearch.yml on each node — set the attribute
node.attr.zone: us-east-1a  # or 1b, 1c

# Cluster-level setting
cluster.routing.allocation.awareness.attributes: zone
cluster.routing.allocation.awareness.force.zone.values: us-east-1a,us-east-1b,us-east-1c
```

### Custom Routing

```json
// Index with routing value (sends to specific shard)
PUT /my-index/_doc/1?routing=user-123
{ "user": "user-123", "title": "my doc" }

// Search with routing — only hits relevant shard
GET /my-index/_search?routing=user-123
{ "query": { "term": { "user": "user-123" } } }
```

---

<!-- section: performance -->
## 8. Performance Tuning

### Shard Sizing Guidelines

| Rule | Recommendation |
|------|----------------|
| Target shard size | 20–50 GB (up to 65 GB max) |
| Shards per node | Heap GB × 20 (guideline) |
| Total cluster shards | < 20 per GB of heap across all nodes |
| Shard count per index | Match query concurrency; avoid over-sharding |

### Bulk Indexing Optimization

```json
// 1. Set replicas=0 before bulk load
PUT /my-index/_settings
{ "number_of_replicas": 0, "refresh_interval": "-1" }

// 2. Bulk API (5-15 MB per request is sweet spot)
POST /_bulk
{ "index": { "_index": "my-index" } }
{ "field": "value" }
...

// 3. Re-enable after bulk load
PUT /my-index/_settings
{ "number_of_replicas": 1, "refresh_interval": "1s" }
POST /my-index/_refresh
```

### Query Performance

```json
// Prefer filter over query context for non-scoring conditions
// BAD:
{ "query": { "bool": { "must": [{ "term": { "status": "active" } }] } } }
// GOOD:
{ "query": { "bool": { "filter": [{ "term": { "status": "active" } }] } } }

// Avoid leading wildcards — full scan
// BAD:
{ "wildcard": { "title": "*elasticsearch*" } }
// BETTER (if you need substring search):
// Use an ngram analyzer at index time

// Use keyword fields for exact match + aggregations
// Never aggregate on a text field (requires fielddata=true — heap expensive)
```

### Pagination Strategies

| Method | Use case | Max depth | Notes |
|--------|----------|-----------|-------|
| `from` / `size` | Random access, UI pagination | 10,000 (max_result_window) | Deep pagination is expensive |
| `search_after` | Sequential scrolling, exports | Unlimited | Requires consistent sort + PIT |
| `scroll` | Bulk export (legacy) | Unlimited | ⚠️ Deprecated for deep pagination — use PIT instead |
| Point-in-Time + `search_after` | Modern deep pagination | Unlimited | Preferred approach |

```json
// search_after with PIT
POST /my-index/_pit?keep_alive=1m
// → { "id": "abc123..." }

GET /_search
{
  "size": 100,
  "pit": { "id": "abc123...", "keep_alive": "1m" },
  "sort": [{ "@timestamp": "desc" }, { "_id": "asc" }],
  "search_after": ["2024-01-15T12:00:00Z", "doc-id-999"]
}

// Close PIT when done
DELETE /_pit
{ "id": "abc123..." }
```

### JVM & OS Tuning

```yaml
# jvm.options
-Xms16g
-Xmx16g   # Always equal; never exceed 50% RAM or ~31GB (compressed oops limit)

# /etc/sysctl.conf
vm.max_map_count=262144    # Required for Lucene mmapped files
vm.swappiness=1            # Minimize swapping

# /etc/security/limits.conf
elasticsearch  -  nofile  65535
elasticsearch  -  nproc   4096
```

```bash
# Disable swap entirely
sudo swapoff -a
# Or lock memory in elasticsearch.yml:
bootstrap.memory_lock: true
```

### Profile API Deep Dive

```json
GET /my-index/_search
{
  "profile": true,
  "query": {
    "bool": {
      "must": [{ "match": { "title": "elasticsearch" } }],
      "filter": [{ "term": { "status": "active" } }]
    }
  }
}
// Response includes: query breakdown (create_weight, build_scorer, score, match),
// collector timing, and aggregation profiling
```

---

<!-- section: scroll-pit -->
## 9. Scroll, Search After & PIT

### Scroll API (⚠️ Deprecated for deep pagination)

```json
// Initial request — returns scroll_id
GET /my-index/_search?scroll=2m
{ "size": 100, "sort": ["_doc"], "query": { "match_all": {} } }

// Subsequent pages
POST /_search/scroll
{ "scroll": "2m", "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gBAAAA..." }

// Clear scroll when done
DELETE /_search/scroll
{ "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gBAAAA..." }
```

### Point-in-Time (Recommended)

```json
// 1. Open PIT
POST /my-index/_pit?keep_alive=5m
// Returns: { "id": "46ToAwMDaWR5BXV1a..." }

// 2. First page
GET /_search
{
  "size": 1000,
  "pit": { "id": "46ToAwMDaWR5BXV1a...", "keep_alive": "5m" },
  "sort": [{ "@timestamp": "asc" }, { "_id": "asc" }],
  "query": { "match_all": {} }
}

// 3. Subsequent pages — use last hit's sort values
GET /_search
{
  "size": 1000,
  "pit": { "id": "46ToAwMDaWR5BXV1a...", "keep_alive": "5m" },
  "sort": [{ "@timestamp": "asc" }, { "_id": "asc" }],
  "search_after": ["2024-01-15T10:00:00.000Z", "last-doc-id"]
}

// 4. Close PIT
DELETE /_pit
{ "id": "46ToAwMDaWR5BXV1a..." }
```

> **Note:** Include `_id` as a tiebreaker in sort to ensure deterministic ordering across pages.

---

<!-- section: ingest -->
## 10. Ingest Pipelines

### Pipeline CRUD

```json
// Create pipeline
PUT /_ingest/pipeline/my-pipeline
{
  "description": "Parse Apache access logs",
  "processors": [
    { "grok": {
        "field": "message",
        "patterns": ["%{COMBINEDAPACHELOG}"],
        "on_failure": [{ "set": { "field": "parse_error", "value": "grok_failed" } }]
      }
    },
    { "date": { "field": "timestamp", "formats": ["dd/MMM/yyyy:HH:mm:ss Z"], "target_field": "@timestamp" } },
    { "remove": { "field": ["timestamp", "message"] } },
    { "geoip": { "field": "clientip", "target_field": "geo" } },
    { "user_agent": { "field": "agent" } },
    { "convert": { "field": "bytes", "type": "integer", "ignore_missing": true } },
    { "set": { "field": "environment", "value": "production" } }
  ],
  "on_failure": [
    { "set": { "field": "_index", "value": "failed-{{ _index }}" } }
  ]
}

// Test with simulate
POST /_ingest/pipeline/my-pipeline/_simulate
{
  "docs": [
    { "_source": { "message": "192.168.1.1 - - [01/Jan/2024:00:00:00 +0000] \"GET /index.html HTTP/1.1\" 200 1234 \"-\" \"Mozilla/5.0\"" } }
  ]
}

// Apply at index time
POST /my-index/_doc?pipeline=my-pipeline
{ "message": "192.168.1.1 - - [01/Jan/2024:00:00:00 +0000] \"GET /index.html HTTP/1.1\" 200 1234 \"-\" \"Mozilla/5.0\"" }
```

### Key Processors Reference

| Processor | Purpose |
|-----------|---------|
| `set` | Set field to value |
| `rename` | Rename field |
| `remove` | Remove field(s) |
| `append` | Append to array |
| `convert` | Type conversion (integer, float, boolean, string, ip, auto) |
| `date` | Parse date string to @timestamp |
| `date_index_name` | Route to date-based index |
| `grok` | Regex pattern extraction (Logstash-compatible) |
| `dissect` | Fast ordered field extraction (no regex) |
| `json` | Parse JSON string into object |
| `kv` | Parse key=value pairs |
| `csv` | Parse CSV line |
| `split` | Split string to array |
| `join` | Join array to string |
| `gsub` | Regex replace |
| `trim` / `lowercase` / `uppercase` | String manipulation |
| `script` | Painless script |
| `foreach` | Iterate array |
| `pipeline` | Call another pipeline |
| `drop` | Drop document (silently discard) |
| `fail` | Fail with custom message |
| `fingerprint` | Generate hash (SHA-256, SHA-512, MD5, etc.) |
| `geoip` | IP → geo lookup (requires geoip database) |
| `user_agent` | Parse user-agent string |
| `enrich` | Lookup from enrich policy index |
| `dot_expander` | Expand `a.b.c` to nested object |
| `redact` | 8.7+ Redact PII via grok patterns |

### Default + Final Pipelines

```json
// Apply pipeline to all documents in an index by default
PUT /my-index/_settings
{ "default_pipeline": "my-pipeline" }

// Final pipeline (always runs last, even with per-request override)
PUT /my-index/_settings
{ "final_pipeline": "add-metadata-pipeline" }
```

---

<!-- section: security -->
## 11. Security (X-Pack) 💼 Licensed

### Enable Security (8.x — on by default)

```yaml
# elasticsearch.yml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: http.p12
```

```bash
# Generate CA and node certificates
./bin/elasticsearch-certutil ca --out elastic-ca.p12 --pass ""
./bin/elasticsearch-certutil cert --ca elastic-ca.p12 --out elastic-certificates.p12 --pass ""

# HTTP layer certificates (interactive wizard)
./bin/elasticsearch-certutil http

# Set built-in passwords
./bin/elasticsearch-setup-passwords interactive

# Reset password (8.x)
./bin/elasticsearch-reset-password -u elastic
```

### Roles and Privileges

```json
// Create custom role
PUT /_security/role/app-reader
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["app-logs-*"],
      "privileges": ["read", "view_index_metadata"],
      "field_security": { "grant": ["@timestamp", "level", "message", "service"] },
      "query": "{ \"term\": { \"environment\": \"production\" } }"
    }
  ]
}

// Create user
POST /_security/user/john
{
  "password": "s3cur3p@ss!",
  "roles": ["app-reader"],
  "full_name": "John Doe",
  "email": "john@example.com"
}

// Create API key
POST /_security/api_key
{
  "name": "my-app-key",
  "expiration": "30d",
  "role_descriptors": {
    "app-role": {
      "cluster": ["monitor"],
      "index": [{ "names": ["app-*"], "privileges": ["read"] }]
    }
  }
}

// Invalidate API key
DELETE /_security/api_key
{ "name": "my-app-key" }

// Use API key in request
curl -H "Authorization: ApiKey base64(id:api_key)" https://localhost:9200/
```

### Built-in Users

| User | Purpose |
|------|---------|
| `elastic` | Superuser |
| `kibana_system` | Kibana internal communication |
| `logstash_system` | Logstash monitoring |
| `beats_system` | Beats monitoring |
| `apm_system` | APM server monitoring |
| `remote_monitoring_user` | Metricbeat monitoring collection |

---

<!-- section: snapshots -->
## 12. Snapshot & Restore

### Repository Setup

```json
// File system repository
PUT /_snapshot/my-fs-repo
{
  "type": "fs",
  "settings": { "location": "/mnt/backups/elasticsearch", "compress": true }
}

// S3 repository (requires repository-s3 plugin)
PUT /_snapshot/my-s3-repo
{
  "type": "s3",
  "settings": {
    "bucket": "my-es-snapshots",
    "region": "us-east-1",
    "base_path": "snapshots/prod",
    "server_side_encryption": true
  }
}

// GCS repository
PUT /_snapshot/my-gcs-repo
{
  "type": "gcs",
  "settings": { "bucket": "my-es-snapshots", "base_path": "prod" }
}
```

### Snapshot CRUD

```json
// Take snapshot
PUT /_snapshot/my-s3-repo/snapshot-2024-01-01
{
  "indices": "logs-*,app-*",
  "ignore_unavailable": true,
  "include_global_state": false,
  "metadata": { "taken_by": "ops-team", "taken_because": "daily backup" }
}

// Check snapshot status
GET /_snapshot/my-s3-repo/snapshot-2024-01-01

// List snapshots
GET /_snapshot/my-s3-repo/_all?verbose=false

// Delete snapshot
DELETE /_snapshot/my-s3-repo/snapshot-2024-01-01

// Restore
POST /_snapshot/my-s3-repo/snapshot-2024-01-01/_restore
{
  "indices": "logs-2024.01.01-*",
  "ignore_unavailable": true,
  "include_global_state": false,
  "rename_pattern": "logs-(.+)",
  "rename_replacement": "restored-logs-$1",
  "index_settings": { "number_of_replicas": 0 }
}
```

### Snapshot Lifecycle Management (SLM) 💼 Licensed

```json
PUT /_slm/policy/daily-snapshots
{
  "schedule": "0 30 1 * * ?",
  "name": "<daily-snap-{now/d}>",
  "repository": "my-s3-repo",
  "config": {
    "indices": "*",
    "include_global_state": true,
    "ignore_unavailable": true
  },
  "retention": { "expire_after": "30d", "min_count": 5, "max_count": 50 }
}

// Execute immediately
POST /_slm/policy/daily-snapshots/_execute

// SLM status
GET /_slm/stats
GET /_slm/policy/daily-snapshots
```

### Searchable Snapshots 💼 Licensed

```json
// Mount snapshot as searchable index (fully mounted — faster, uses local cache)
POST /_snapshot/my-s3-repo/snapshot-2024-01-01/_mount?wait_for_completion=true
{
  "index": "logs-2024.01.01-000001",
  "renamed_index": "logs-2024.01.01-000001-mounted",
  "index_settings": { "number_of_replicas": 0 },
  "storage": "full_copy"   // or "shared_cache" for partially mounted (frozen tier)
}
```

---

<!-- section: cross-cluster -->
## 13. Cross-Cluster Operations

### Cross-Cluster Search (CCS)

```yaml
# elasticsearch.yml (static config)
cluster.remote.my-remote:
  seeds: ["remote-node-1:9300", "remote-node-2:9300"]
  skip_unavailable: true
```

```json
// Dynamic config
PUT /_cluster/settings
{
  "persistent": {
    "cluster.remote.my-remote.seeds": ["remote-node-1:9300"],
    "cluster.remote.my-remote.skip_unavailable": true
  }
}

// Search across clusters — use :: syntax
GET /local-index,my-remote:remote-index/_search
{ "query": { "match": { "title": "elasticsearch" } } }

// Cross-cluster aggregation
GET /my-remote:logs-*,logs-*/_search
{ "size": 0, "aggs": { "total_errors": { "filter": { "term": { "level": "error" } } } } }
```

### Cross-Cluster Replication (CCR) 💼 Licensed

```json
// On the follower cluster: replicate a specific index
PUT /follower-index/_ccr/follow
{
  "remote_cluster": "leader-cluster",
  "leader_index": "leader-index",
  "settings": { "number_of_replicas": 0 },
  "max_read_request_operation_count": 5120,
  "max_outstanding_read_requests": 12
}

// Auto-follow pattern
PUT /_ccr/auto_follow/my-auto-follow
{
  "remote_cluster": "leader-cluster",
  "leader_index_patterns": ["logs-*"],
  "follow_index_pattern": "{{leader_index}}",
  "settings": { "number_of_replicas": 0 }
}

// Pause/resume replication
POST /follower-index/_ccr/pause_follow
POST /follower-index/_ccr/resume_follow

// CCR stats
GET /_ccr/stats
GET /follower-index/_ccr/info
GET /follower-index/_ccr/stats
```

---

<!-- section: vector-search -->
## 14. Vector Search & kNN

### dense_vector Mapping

```json
PUT /products
{
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "embedding": {
        "type": "dense_vector",
        "dims": 384,
        "index": true,
        "similarity": "cosine",
        "index_options": {
          "type": "hnsw",
          "m": 16,
          "ef_construction": 100
        }
      }
    }
  }
}
```

| `similarity` option | Use for |
|---------------------|---------|
| `cosine` | Direction-based similarity (most common for NLP) |
| `dot_product` | When vectors are normalized; faster than cosine |
| `l2_norm` (euclidean) | Distance-based, image/numeric data |

### Exact kNN (Script Score)

```json
GET /products/_search
{
  "query": {
    "script_score": {
      "query": { "bool": { "filter": [{ "term": { "status": "active" } }] } },
      "script": {
        "source": "cosineSimilarity(params.query_vector, 'embedding') + 1.0",
        "params": { "query_vector": [0.1, 0.2, 0.3, ...] }
      }
    }
  }
}
```

### Approximate kNN (8.x)

```json
GET /products/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [0.1, 0.2, 0.3, ...],
    "k": 10,
    "num_candidates": 100,
    "filter": [{ "term": { "category": "electronics" } }],
    "boost": 0.8
  },
  "query": {
    "match": { "title": "wireless headphones" }
  },
  "size": 10
}
```

### ELSER (Sparse Vector / Semantic Search) 💼 Licensed

```json
// 1. Deploy ELSER model
PUT /_ml/trained_models/.elser_model_2
{ "input": { "field_names": ["text_field"] } }

POST /_ml/trained_models/.elser_model_2/deployment/_start
{ "number_of_allocations": 1, "threads_per_allocation": 1 }

// 2. Ingest pipeline with ELSER inference
PUT /_ingest/pipeline/elser-pipeline
{
  "processors": [{
    "inference": {
      "model_id": ".elser_model_2",
      "input_output": [{ "input_field": "body", "output_field": "body_sparse_embedding" }]
    }
  }]
}

// 3. Mapping
PUT /semantic-search
{
  "mappings": {
    "properties": {
      "body": { "type": "text" },
      "body_sparse_embedding": { "type": "sparse_vector" }
    }
  }
}

// 4. Query
GET /semantic-search/_search
{
  "query": {
    "sparse_vector": {
      "field": "body_sparse_embedding",
      "inference_id": ".elser_model_2",
      "query": "what is the capital of France"
    }
  }
}
```

---

<!-- section: rest-api -->
## 16. REST API Quick Reference

### Document CRUD

```bash
# Index (with explicit ID — creates or replaces)
PUT  /index/_doc/id                       { "field": "value" }

# Index (auto-generate ID)
POST /index/_doc                          { "field": "value" }

# Index (create-only — fails if exists)
PUT  /index/_create/id                    { "field": "value" }
POST /index/_doc?op_type=create           { "field": "value" }

# Get document
GET  /index/_doc/id
GET  /index/_doc/id?_source=field1,field2

# Get source only
GET  /index/_source/id

# Check if document exists
HEAD /index/_doc/id

# Update (partial — merge)
POST /index/_update/id                    { "doc": { "field": "new_value" } }

# Update (scripted)
POST /index/_update/id
{ "script": { "source": "ctx._source.counter += params.count", "params": { "count": 1 } } }

# Update (upsert — create if not exists)
POST /index/_update/id
{ "doc": { "field": "value" }, "upsert": { "field": "value", "counter": 0 } }

# Delete
DELETE /index/_doc/id

# Bulk
POST /_bulk
{ "index": { "_index": "idx", "_id": "1" } }
{ "field": "value" }
{ "create": { "_index": "idx", "_id": "2" } }
{ "field": "value" }
{ "update": { "_index": "idx", "_id": "1" } }
{ "doc": { "field": "updated" } }
{ "delete": { "_index": "idx", "_id": "1" } }

# Multi-get
GET /_mget
{ "docs": [{ "_index": "idx", "_id": "1" }, { "_index": "idx", "_id": "2" }] }

# Multi-search
POST /_msearch
{ "index": "index1" }
{ "query": { "match_all": {} }, "size": 1 }
{ "index": "index2" }
{ "query": { "term": { "status": "active" } }, "size": 5 }
```

### Index Management

```bash
# Create / delete index
PUT  /my-index    { "settings": {}, "mappings": {} }
DELETE /my-index

# Open / close index (closed = no read/write, still tracked)
POST /my-index/_close
POST /my-index/_open

# Freeze / unfreeze (⚠️ Deprecated in 8.x — use cold/frozen tiers)
POST /my-index/_freeze
POST /my-index/_unfreeze

# Block index
PUT /my-index/_block/write
PUT /my-index/_block/read

# Index aliases
POST /_aliases
{ "actions": [
    { "add":    { "index": "my-index-v2", "alias": "my-index", "is_write_index": true } },
    { "remove": { "index": "my-index-v1", "alias": "my-index" } }
  ]
}
```

### Common Query Parameters

| Parameter | Description |
|-----------|-------------|
| `?pretty` | Pretty-print JSON response |
| `?filter_path=hits.hits._source` | Filter response fields |
| `?human` | Human-readable values (e.g. "1.5gb" instead of bytes) |
| `?error_trace` | Include full stack trace on errors |
| `?timeout=5s` | Per-shard timeout |
| `?routing=value` | Custom routing |
| `?preference=_local` | Prefer local shards for read |
| `?pipeline=name` | Apply ingest pipeline |
| `?wait_for_active_shards=all` | Wait for shard availability |
| `?refresh=true` | Refresh after operation (use sparingly) |
| `?refresh=wait_for` | Wait for refresh (preferred over `true`) |

---

<!-- section: monitoring -->
## 17. Monitoring & Observability

### Cluster Health

```bash
GET /_cluster/health?pretty
GET /_cluster/health/my-index?wait_for_status=green&timeout=30s
```

| Color | Meaning |
|-------|---------|
| 🟢 Green | All primary + replica shards assigned |
| 🟡 Yellow | All primary shards assigned; some replicas unassigned |
| 🔴 Red | Some primary shards unassigned; data unavailable |

### Key Monitoring Endpoints

```bash
# Cluster state & stats
GET /_cluster/state
GET /_cluster/stats?human

# Nodes
GET /_nodes
GET /_nodes/stats?human
GET /_nodes/stats/jvm,os,process,thread_pool?human
GET /_nodes/hot_threads

# Index stats
GET /my-index/_stats
GET /_stats                              # All indices

# Pending cluster tasks
GET /_cluster/pending_tasks

# Task management
GET /_tasks
GET /_tasks?actions=*search*&detailed=true
POST /_tasks/task-id/_cancel
```

### Cat APIs (Human-Readable)

```bash
GET /_cat/indices?v&s=index&h=index,docs.count,store.size,health
GET /_cat/nodes?v&h=name,heap.percent,cpu,load_1m,node.role
GET /_cat/shards?v&s=index&h=index,shard,prirep,state,docs,store,node
GET /_cat/allocation?v
GET /_cat/health?v
GET /_cat/master?v
GET /_cat/recovery?v&active_only=true
GET /_cat/segments?v
GET /_cat/pending_tasks?v
GET /_cat/thread_pool?v&h=name,active,queue,rejected

# Filter cat output
GET /_cat/indices?v&s=store.size:desc&h=index,docs.count,store.size&bytes=gb
```

### Slow Logs Configuration

```json
PUT /my-index/_settings
{
  "index.search.slowlog.threshold.query.warn":   "5s",
  "index.search.slowlog.threshold.query.info":   "2s",
  "index.search.slowlog.threshold.query.debug":  "1s",
  "index.search.slowlog.threshold.query.trace":  "500ms",
  "index.search.slowlog.threshold.fetch.warn":   "1s",
  "index.search.slowlog.threshold.fetch.info":   "800ms",
  "index.indexing.slowlog.threshold.index.warn": "10s",
  "index.indexing.slowlog.threshold.index.info": "5s",
  "index.search.slowlog.level": "info",
  "index.search.slowlog.source": 1000
}
```

---

<!-- section: ilm-data-streams -->
## 18. Kibana & Elastic Stack Integration

### Elastic Stack Architecture

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Filebeat   │    │  Metricbeat  │    │  Elastic APM │
│  (log files) │    │  (metrics)   │    │    (traces)  │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │  (direct or via Logstash/Fleet)
                    ┌──────▼──────────────────────────┐
                    │          Elasticsearch           │
                    │   (indices + ILM + data streams) │
                    └──────────────────────────────────┘
                                    │
                    ┌───────────────▼────────────────┐
                    │              Kibana             │
                    │  Discover | Lens | Dashboards  │
                    │  APM | SIEM | Fleet | ML        │
                    └────────────────────────────────┘
```

### Logstash → Elasticsearch

```ruby
# logstash.conf
output {
  elasticsearch {
    hosts  => ["https://es-node:9200"]
    index  => "logs-%{+YYYY.MM.dd}"
    user   => "logstash_system"
    password => "${LOGSTASH_PASSWORD}"
    ssl    => true
    cacert => "/etc/logstash/ca.crt"
    ilm_enabled => true
    ilm_rollover_alias => "logs-write"
    ilm_policy => "logs-ilm-policy"
    pipeline => "my-ingest-pipeline"
  }
}
```

### Filebeat Direct to Elasticsearch

```yaml
# filebeat.yml
output.elasticsearch:
  hosts: ["https://es-node:9200"]
  username: "beats_system"
  password: "${BEATS_PASSWORD}"
  ssl.certificate_authorities: ["/etc/beats/ca.crt"]
  index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"
  pipeline: "filebeat-pipeline"

setup.ilm.enabled: true
setup.ilm.rollover_alias: "filebeat"
setup.ilm.policy_name: "filebeat-policy"
```

---

<!-- section: cli -->
## 19. Elasticsearch CLI & Dev Tools

### elasticsearch-keystore

```bash
# Add secret (e.g. S3 credentials)
./bin/elasticsearch-keystore add s3.client.default.access_key
./bin/elasticsearch-keystore add s3.client.default.secret_key

# List keys
./bin/elasticsearch-keystore list

# Remove key
./bin/elasticsearch-keystore remove s3.client.default.secret_key

# Reload keystore secrets without restart
POST /_nodes/reload_secure_settings
{ "secure_settings_password": "optional-keystore-pass" }
```

### elasticsearch-certutil

```bash
# Generate CA
./bin/elasticsearch-certutil ca --out elastic-ca.p12 --pass ""

# Generate node cert signed by CA
./bin/elasticsearch-certutil cert --ca elastic-ca.p12 --out elastic-certificates.p12 --pass ""

# HTTP certificate wizard (for Kibana/browser trust)
./bin/elasticsearch-certutil http

# Convert to PEM
openssl pkcs12 -in elastic-certificates.p12 -nocerts -nodes -out node-key.pem
openssl pkcs12 -in elastic-certificates.p12 -clcerts -nokeys -out node-cert.pem
```

### Common curl Patterns

```bash
# Basic auth
curl -u elastic:password https://localhost:9200/_cluster/health?pretty

# With CA cert
curl --cacert /etc/elasticsearch/ca.crt -u elastic:password https://localhost:9200/_cat/nodes?v

# POST JSON
curl -u elastic:password -X POST https://localhost:9200/my-index/_search \
  -H "Content-Type: application/json" \
  -d '{ "query": { "match_all": {} } }'

# API key auth
curl -H "Authorization: ApiKey <base64-encoded-id:api_key>" https://localhost:9200/

# Check version
curl -s https://localhost:9200 | jq '.version.number'
```

---

<!-- section: patterns -->
## 20. Common Patterns & Best Practices

### Write Alias + Read Alias Pattern

```json
// Create index
PUT /products-v2
{ "mappings": { "properties": { "name": { "type": "text" }, "price": { "type": "float" } } } }

// Atomic alias swap (zero-downtime re-index)
POST /_aliases
{
  "actions": [
    { "add":    { "index": "products-v2", "alias": "products-write", "is_write_index": true } },
    { "add":    { "index": "products-v2", "alias": "products-read" } },
    { "remove": { "index": "products-v1", "alias": "products-write" } },
    { "remove": { "index": "products-v1", "alias": "products-read" } }
  ]
}
```

### Search Templates

```json
// Store template
PUT /_scripts/product-search-template
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "bool": {
          "must": [{ "match": { "name": "{{query}}" } }],
          "filter": [
            {{#category}}{ "term": { "category": "{{category}}" } },{{/category}}
            { "range": { "price": { "lte": {{max_price}} } } }
          ]
        }
      },
      "size": "{{size}}{{^size}}10{{/size}}"
    }
  }
}

// Use template
GET /products/_search/template
{
  "id": "product-search-template",
  "params": { "query": "headphones", "category": "electronics", "max_price": 200, "size": 5 }
}
```

### Avoiding Mapping Explosion

```json
// BAD — dynamic: true on object with arbitrary keys
{
  "user_properties": { "type": "object" }
}

// GOOD — use flattened for arbitrary key-value
{
  "user_properties": { "type": "flattened" }
}

// Or disable dynamic for sub-objects
{
  "metadata": { "type": "object", "dynamic": false }
}
```

### Async Search

```json
// Submit async search
POST /my-index/_async_search?wait_for_completion_timeout=1s&keep_on_completion=true&keep_alive=1d
{ "query": { "match_all": {} }, "aggs": { "by_category": { "terms": { "field": "category" } } } }
// Returns: { "id": "async-search-id", "is_partial": true, "is_running": true }

// Poll for results
GET /_async_search/async-search-id

// Delete when done
DELETE /_async_search/async-search-id
```

### Index Naming Conventions

```
# Time-series pattern (with ILM)
logs-{app}-{environment}-{date}
logs-nginx-production-2024.01.01-000001   ← ILM managed

# Application data
{product}-{environment}
products-production

# Data stream naming
logs-{type}-{dataset}
logs-nginx.access-default
```

---

<!-- section: troubleshooting -->
## 21. Troubleshooting Decision Trees

### Cluster RED/YELLOW Troubleshooting

```
Cluster is RED or YELLOW?
│
├── GET /_cluster/health → which indices affected?
├── GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason → find UNASSIGNED shards
│
├── GET /_cluster/allocation/explain → WHY is the shard unassigned?
│   │
│   ├── "no valid shard copy" → data loss? Try: POST /_cluster/reroute?retry_failed=true
│   ├── "disk watermark exceeded" → free disk space or raise watermark threshold
│   ├── "node left cluster" → wait for node to return or increase replicas
│   ├── "max retries exceeded" → POST /_cluster/reroute?retry_failed=true
│   └── "allocation decider" → check allocation rules, node tags, zone awareness
│
├── If single-node cluster → replicas can never be assigned (need 2+ nodes or replicas=0)
└── If RED (primary unassigned) → data is unavailable; treat as incident
```

### High Heap Usage Investigation

```bash
# Check heap usage
GET /_nodes/stats/jvm

# Common culprits:
# 1. fielddata cache (text fields with aggregations)
GET /_cat/fielddata?v                              # fielddata usage
PUT /my-index/_settings { "index.fielddata.cache.size": "20%" }   # limit it
POST /my-index/_cache/clear?fielddata=true         # clear it

# 2. Large aggregations (too many buckets)
# 3. Segment memory (too many small shards)
GET /_cat/nodes?v&h=name,segments.memory          # segment memory

# Force GC (last resort — don't use regularly)
POST /_nodes/hot_threads
```

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `circuit_breaking_exception` | Query would exceed memory limit | Optimize query, increase circuit breaker limits, or add memory |
| `es_rejected_execution_exception` | Thread pool queue full | Scale nodes, optimize queries, check bulk queue |
| `index_closed_exception` | Index is closed | `POST /index/_open` |
| `cluster_block_exception` | Index is read-only (usually disk full) | Free disk space, then clear `index.blocks.read_only_allow_delete` |
| `illegal_argument_exception: Text fields are not optimised for operations` | Aggregating on text field | Use `.keyword` sub-field |
| `mapper_parsing_exception` | Document doesn't match mapping | Check field types, `ignore_malformed` |
| `index_not_found_exception` | Index doesn't exist | Create index or use wildcard pattern |
| `search_phase_execution_exception` | Query execution error on shard | Check slow logs, Explain API |
| `max_result_window` exceeded | `from + size > max_result_window` | Use `search_after` + PIT, or increase `max_result_window` |

### Slow Query Troubleshooting

```
Queries are slow?
│
├── Check slow logs → identify slow queries
│
├── Run Profile API → identify expensive query phases
│   ├── High "create_weight" → rewrite/expansion heavy (wildcard/regexp/fuzzy)
│   ├── High "build_scorer" → many matching documents to score
│   └── High "advance" / "next_doc" → inefficient filter
│
├── Is it a filter that should be cached?
│   └── Use filter context (bool.filter) instead of query context (bool.must)
│
├── Is it a deep aggregation?
│   ├── Reduce terms agg size
│   ├── Use sampler aggregation
│   └── Consider approximate cardinality
│
└── Is the index too large / shards too many?
    └── Review shard sizing, consider ILM/rollover
```

---

<!-- section: migration -->
## 22. Version Compatibility & Migration

### Compatibility Matrix

| ES Version | Kibana | Beats | Logstash | Java |
|------------|--------|-------|----------|------|
| 8.x | Must match minor | 7.17+ or 8.x | 7.17+ or 8.x | JDK 17+ (bundled) |
| 7.17 | 7.17 | 7.17 | 7.17 | JDK 11+ |
| 7.x | Same minor version | 6.8+ | 6.8+ | JDK 8/11 |

### Rolling Upgrade (7.x → 8.x)

```
Prerequisites:
1. Upgrade to 7.17.x first (latest 7.x)
2. Run GET /_migration/deprecations — fix all deprecations
3. Review breaking changes at elastic.co/guide/en/elasticsearch/reference/8.0/breaking-changes-8.0.html

Steps per node (rolling):
1. Disable shard allocation
   PUT /_cluster/settings { "persistent": { "cluster.routing.allocation.enable": "primaries" } }
2. Do a synced flush (optional in 7.6+)
   POST /_flush/synced
3. Stop Elasticsearch
4. Upgrade the package
5. Start Elasticsearch
6. Wait for node to join cluster (GET /_cat/nodes)
7. Re-enable shard allocation
   PUT /_cluster/settings { "persistent": { "cluster.routing.allocation.enable": null } }
8. Wait for cluster to go green (GET /_cluster/health?wait_for_status=green)
9. Repeat for next node
```

### Notable Breaking Changes

| Version | Change |
|---------|--------|
| **7.0** | Removal of `_type` (mapping types) — all docs in single `_doc` type |
| **7.0** | `include_type_name` parameter for transition period |
| **8.0** | Security enabled by default (`xpack.security.enabled: true`) |
| **8.0** | `_type` field completely removed |
| **8.0** | Frozen indices changed to use searchable snapshots (old freeze deprecated) |
| **8.0** | Removed `discovery.zen.minimum_master_nodes` |
| **8.1** | `knn` top-level parameter for approximate kNN |
| **8.8** | Semantic search with `semantic_text` field (tech preview) |

### Deprecation API

```bash
GET /_migration/deprecations                   # Cluster-wide deprecations
GET /my-index/_migration/deprecations          # Index-specific deprecations
GET /_nodes/deprecations                       # Node-specific deprecations
```

---

<!-- section: elasticsearch-vs -->
## 23. Elasticsearch vs OpenSearch vs Solr

| Feature | Elasticsearch (8.x) | OpenSearch (2.x) | Apache Solr (9.x) |
|---------|---------------------|------------------|-------------------|
| License | Elastic License 2.0 / SSPL | Apache 2.0 | Apache 2.0 |
| Parent | Elastic | AWS (fork from ES 7.10) | Apache Foundation |
| Managed cloud | Elastic Cloud | AWS OpenSearch Service | — |
| Security | X-Pack (bundled free in 8.x) | OpenSearch Security plugin | Solr Auth / Ranger |
| ML / Anomaly detection | X-Pack ML 💼 | k-NN, ML commons | Limited |
| Vector search | kNN, ELSER | k-NN plugin | Dense vector (9.x) |
| Schema-less | Yes (dynamic mapping) | Yes | No (requires schema.xml) |
| Query language | Query DSL (JSON) | Query DSL (JSON, mostly compatible) | Lucene syntax, JSON facets |
| Aggregations | Full pipeline aggs | Similar to ES 7.10 | Facets (less powerful) |
| ILM / Rollup | ILM, Data Streams, Rollup 💼 | ISM, Rollup | No native ILM |
| CCR / CCS | CCR 💼, CCS free | CCR, CCS | SolrCloud replication |
| API compatibility | — | ~Compatible with ES 7.10 | Incompatible |
| Notable gaps (OpenSearch vs ES 8.x) | — | No EQL, No ELSER, No Elastic Agent, security on by default requires config | — |

---

<!-- section: runtime-fields -->
## 24. Runtime Fields

```json
// Define at index level
PUT /my-index/_mapping
{
  "runtime": {
    "day_of_week": {
      "type": "keyword",
      "script": {
        "source": "emit(doc['@timestamp'].value.dayOfWeekEnum.getDisplayName(TextStyle.FULL, Locale.ROOT))"
      }
    },
    "price_eur": {
      "type": "double",
      "script": "emit(doc['price_usd'].value * 0.92)"
    }
  }
}

// Define at query time (ad-hoc, not persisted)
GET /my-index/_search
{
  "runtime_mappings": {
    "total_revenue": {
      "type": "double",
      "script": "emit(doc['price'].value * doc['quantity'].value)"
    }
  },
  "aggs": {
    "revenue_stats": { "stats": { "field": "total_revenue" } }
  }
}
```

> **Note:** Runtime fields compute values at query time — flexible but slower than indexed fields. Use for prototyping or rarely-queried derived values.

---

<!-- section: painless -->
## 25. Painless Scripting

```json
// Script contexts: update, search (score), aggregation, ingest

// Inline script example — score booster
{
  "script_score": {
    "query": { "match_all": {} },
    "script": {
      "source": """
        double score = _score;
        if (doc['featured'].value == true) score *= 2;
        if (doc.containsKey('views') && !doc['views'].empty) {
          score += Math.log(doc['views'].value + 1);
        }
        return score;
      """
    }
  }
}

// Stored script
PUT /_scripts/boost-featured
{ "script": { "lang": "painless", "source": "doc['featured'].value ? _score * params.boost : _score" } }

// Use stored script
{ "script_score": { "query": { "match_all": {} }, "script": { "id": "boost-featured", "params": { "boost": 3.0 } } } }

// Update with script
POST /products/_update/1
{
  "script": {
    "source": """
      if (ctx._source.tags == null) ctx._source.tags = new ArrayList();
      if (!ctx._source.tags.contains(params.tag)) ctx._source.tags.add(params.tag);
    """,
    "params": { "tag": "new-tag" }
  }
}
```

---

<!-- section: enrich -->
## 26. Enrich Processor

```json
// 1. Create enrich source index
PUT /products
{ "mappings": { "properties": { "sku": { "type": "keyword" }, "category": { "type": "keyword" }, "price": { "type": "float" } } } }

// 2. Create enrich policy
PUT /_enrich/policy/product-policy
{
  "match": {
    "indices": "products",
    "match_field": "sku",
    "enrich_fields": ["category", "price", "brand"]
  }
}

// 3. Execute policy (builds enrich index)
POST /_enrich/policy/product-policy/_execute

// 4. Use enrich processor in pipeline
PUT /_ingest/pipeline/enrich-orders
{
  "processors": [{
    "enrich": {
      "policy_name": "product-policy",
      "field": "order_sku",
      "target_field": "product",
      "max_matches": 1
    }
  }]
}
```

---

<!-- section: rollup -->
## 27. Rollup & Transform

### Transform (Replaces Rollup for most use cases)

```json
// Continuous transform — incrementally pivot data
PUT /_transform/hourly-product-stats
{
  "source": {
    "index": "orders-*",
    "query": { "range": { "@timestamp": { "gte": "now-7d" } } }
  },
  "pivot": {
    "group_by": {
      "product_id": { "terms": { "field": "product_id" } },
      "hour":       { "date_histogram": { "field": "@timestamp", "fixed_interval": "1h" } }
    },
    "aggregations": {
      "total_revenue": { "sum":         { "field": "amount" } },
      "order_count":   { "value_count": { "field": "_id" } },
      "avg_amount":    { "avg":         { "field": "amount" } }
    }
  },
  "dest": { "index": "hourly-product-stats" },
  "sync": {
    "time": { "field": "@timestamp", "delay": "60s" }
  }
}

// Start transform
POST /_transform/hourly-product-stats/_start

// Transform stats
GET /_transform/hourly-product-stats/_stats
```

---

## Appendix: elasticsearch.yml Key Settings Reference

```yaml
# ────── Cluster ──────
cluster.name: my-production-cluster
node.name: node-1
node.roles: [master, data, ingest]

# ────── Paths ──────
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

# ────── Network ──────
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300
http.max_content_length: 100mb

# ────── Discovery ──────
discovery.seed_hosts: ["es-node-1", "es-node-2", "es-node-3"]
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]

# ────── Security (8.x) ──────
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/transport.p12
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http.p12

# ────── Memory ──────
bootstrap.memory_lock: true

# ────── Thread pools ──────
thread_pool.write.queue_size: 1000
thread_pool.search.queue_size: 1000

# ────── Logging ──────
logger.org.elasticsearch: INFO
logger.org.elasticsearch.discovery: DEBUG

# ────── Monitoring ──────
xpack.monitoring.collection.enabled: false  # Use Metricbeat instead
```

---

*End of Elasticsearch Master Reference Cheatsheet*
*Covers Elasticsearch 7.x / 8.x — verify version-specific behavior at elastic.co/docs*
