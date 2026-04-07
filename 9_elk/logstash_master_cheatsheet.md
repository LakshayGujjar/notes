# Logstash Master Reference Cheatsheet
> Senior Engineer Reference — Logstash 7.x / 8.x | Elastic Stack | Last updated: 2024

---

## Quick Reference Card — Top 20 Snippets & Commands

```bash
# ── CLI ──────────────────────────────────────────────────────────────
logstash -f /etc/logstash/conf.d/          # Start with config directory
logstash -f pipeline.conf -t               # Test config syntax
logstash -e 'input{stdin{}} output{stdout{codec=>rubydebug}}'  # Inline test
logstash --config.reload.automatic         # Hot reload on change
logstash --log.level debug -f pipeline.conf # Debug logging
logstash-plugin install logstash-filter-translate  # Install plugin
logstash-plugin list --verbose             # List all plugins + versions
logstash-plugin update                     # Update all plugins
logstash-keystore add ES_PASSWORD          # Add secret to keystore

# ── Monitoring API ───────────────────────────────────────────────────
curl localhost:9600/_node/stats/pipelines  # Pipeline event counters
curl localhost:9600/_node/stats/jvm        # JVM heap + GC stats
curl localhost:9600/_node/hot_threads      # Identify slow plugins
curl localhost:9600/_node/plugins          # List installed plugins

# ── Pipeline Snippets ────────────────────────────────────────────────
# Parse Apache log
grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }

# Parse any JSON message field
json { source => "message" target => "parsed" }

# Rename + convert field
mutate { rename => { "clientip" => "[source][ip]" } convert => { "bytes" => "integer" } }

# Parse timestamp
date { match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"] target => "@timestamp" }

# Route to different ES index by field
output { if [type] == "nginx" { elasticsearch { index => "nginx-%{+YYYY.MM.dd}" } }
         else { elasticsearch { index => "logs-%{+YYYY.MM.dd}" } } }

# Drop unwanted events early
filter { if [http][request][method] == "OPTIONS" { drop {} } }

# Fingerprint for deduplication
fingerprint { source => ["host","message","@timestamp"] method => "SHA256" target => "[@metadata][fingerprint]" }
# then in output: document_id => "%{[@metadata][fingerprint]}"

# Add environment variable
mutate { add_field => { "environment" => "${ENV:production}" } }

# Geoip enrichment
geoip { source => "[source][ip]" target => "[source][geo]" }
```

---

<!-- section: architecture -->
## 1. Logstash Overview & Architecture

### What is Logstash

Logstash is a server-side, open-source data processing pipeline that ingests data from multiple sources, transforms it, and ships it to a "stash" (Elasticsearch, Kafka, S3, etc.). It is JVM-based, running JRuby on the JVM, and driven by a plugin-based architecture.

**Core responsibilities:**
- **Ingest** — collect data from files, Beats, Kafka, databases, HTTP, syslog, and 50+ other sources
- **Parse** — extract structure from unstructured text (grok, dissect, json, csv)
- **Enrich** — augment events with geo data, DNS lookups, user agent parsing, DB lookups
- **Transform** — normalize field names (ECS), convert types, aggregate correlated events
- **Route** — conditionally send events to different outputs based on content
- **Output** — index to Elasticsearch, produce to Kafka, write to S3, call webhooks

### Logstash vs Alternatives

| Feature | Logstash | Beats | Elastic Agent | Fluentd | Fluent Bit |
|---------|----------|-------|---------------|---------|------------|
| Language | JRuby/JVM | Go | Go | Ruby/C | C |
| RAM (idle) | 300–600 MB | 30–60 MB | 50–120 MB | 50–100 MB | < 50 MB |
| Complex transforms | ✅ Excellent | ❌ | Partial | ✅ Good | Limited |
| Plugin ecosystem | 200+ | Limited | Via integrations | 700+ | 100+ |
| JDBC input | ✅ | ❌ | ❌ | ✅ | ❌ |
| Persistent queue | ✅ | ✅ (Filebeat) | ✅ | ✅ | ✅ |
| Kubernetes-native | Partial | ✅ | ✅ | ✅ | ✅ |
| Central management | ✅ 💼 | Via Fleet | ✅ Fleet | ❌ | ❌ |
| **Best for** | Complex pipelines | Simple shipping | Unified agent | K8s, many outputs | Edge / IoT |

### Internal Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                            Logstash Process (JVM)                            │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                         Pipeline (pipeline.id: main)                    │ │
│  │                                                                         │ │
│  │  ┌──────────────┐        ┌──────────────┐        ┌───────────────────┐  │ │
│  │  │   INPUT(S)   │        │    QUEUE     │        │  FILTER WORKERS   │  │ │
│  │  │              │        │              │        │  (N threads)      │  │ │
│  │  │  beats :5044 │──push──► in-memory OR ├─batch──►  filter { }      │  │ │
│  │  │  file /logs  │        │  persistent  │        │  (grok,mutate,    │  │ │
│  │  │  kafka topic │        │  (disk)      │        │   date, geoip...) │  │ │
│  │  └──────────────┘        └──────────────┘        └────────┬──────────┘  │ │
│  │                                                           │              │ │
│  │                          ┌────────────────────────────────┘              │ │
│  │                          ▼                                               │ │
│  │                   ┌──────────────┐         ┌───────────────────────┐     │ │
│  │                   │  OUTPUT(S)   │         │  Dead Letter Queue    │     │ │
│  │                   │              │─reject──►  (disk, per pipeline) │     │ │
│  │                   │  elasticsearch│         └───────────────────────┘     │ │
│  │                   │  kafka       │                                       │ │
│  │                   │  s3, file... │                                       │ │
│  │                   └──────────────┘                                       │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  HTTP Monitoring API (localhost:9600)  |  Task Manager  |  Metrics   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Execution Model

| Parameter | Default | Controls |
|-----------|---------|---------|
| `pipeline.workers` | # of CPU cores | Parallel filter+output threads |
| `pipeline.batch.size` | 125 | Max events per worker batch |
| `pipeline.batch.delay` | 50ms | Wait time to fill a batch before processing |

**Event lifecycle:**
```
Source → Input plugin (codec decodes) → Event created
→ Added to Queue (in-memory or PQ)
→ Filter worker picks up batch
→ Filters applied sequentially per event
→ Output plugin sends batch to destination (codec encodes)
→ ACK returned to queue (PQ only)
```

**`@metadata`** — a special field that is never forwarded to outputs. Use it for pipeline-internal routing flags, computed document IDs, and debug data.

---

<!-- section: installation -->
## 2. Installation & Setup

### Package Install

```bash
# Debian/Ubuntu
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg
echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
sudo apt update && sudo apt install logstash

# RHEL/CentOS
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
cat << EOF | sudo tee /etc/yum.repos.d/logstash.repo
[logstash-8.x]
name=Elastic repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
EOF
sudo yum install logstash

# systemd
sudo systemctl enable logstash
sudo systemctl start logstash
sudo systemctl status logstash
sudo journalctl -u logstash -f    # Follow logs
```

### Docker

```bash
# Single container
docker run --rm -it \
  -v /etc/logstash/pipeline:/usr/share/logstash/pipeline:ro \
  -v /etc/logstash/logstash.yml:/usr/share/logstash/config/logstash.yml:ro \
  -p 5044:5044 -p 9600:9600 \
  docker.elastic.co/logstash/logstash:8.12.0
```

```yaml
# docker-compose.yml
version: "3.8"
services:
  logstash:
    image: docker.elastic.co/logstash/logstash:8.12.0
    volumes:
      - ./pipeline:/usr/share/logstash/pipeline:ro
      - ./config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - logstash-data:/usr/share/logstash/data
    environment:
      - LS_HEAP_SIZE=2g
      - ELASTICSEARCH_HOSTS=https://elasticsearch:9200
      - ELASTICSEARCH_PASSWORD=${ES_PASSWORD}
    ports:
      - "5044:5044"
      - "9600:9600"
    depends_on:
      - elasticsearch
volumes:
  logstash-data:
```

### Directory Structure

```
/etc/logstash/
├── logstash.yml              ← Main settings
├── pipelines.yml             ← Multi-pipeline definitions
├── jvm.options               ← JVM heap + GC settings
├── log4j2.properties         ← Log4j logging config
├── startup.options           ← SysV / startup options
└── conf.d/                   ← Pipeline config fragments (loaded alphabetically)
    ├── 01-input-beats.conf
    ├── 10-filter-apache.conf
    └── 99-output-elasticsearch.conf

/var/lib/logstash/
├── queue/                    ← Persistent queue storage
│   └── main/                 ← Per-pipeline queue
├── dead_letter_queue/        ← DLQ storage
│   └── main/
└── uuid                      ← Node UUID

/var/log/logstash/
└── logstash-plain.log

/usr/share/logstash/
├── bin/
│   ├── logstash              ← Main binary
│   ├── logstash-plugin       ← Plugin manager
│   └── logstash-keystore     ← Secrets keystore manager
└── vendor/                   ← Bundled plugins + gems
```

### Plugin Management

```bash
# Install a plugin
logstash-plugin install logstash-filter-translate
logstash-plugin install logstash-input-jdbc

# List installed plugins
logstash-plugin list
logstash-plugin list --verbose         # With versions
logstash-plugin list 'logstash-filter-*'

# Update a specific plugin
logstash-plugin update logstash-filter-grok

# Update all plugins
logstash-plugin update

# Remove plugin
logstash-plugin remove logstash-input-twitter

# Offline / air-gapped install
logstash-plugin pack                   # Create offline pack on internet-connected machine
# Copy logstash-offline-plugins-*.zip to air-gapped host
logstash-plugin install --local logstash-offline-plugins-8.12.0.zip

# Verify config
logstash -f /etc/logstash/conf.d/ -t
```

---

<!-- section: logstash-yml -->
## 3. logstash.yml — Full Settings Reference

```yaml
# ─── Node ──────────────────────────────────────────────────────────────────
node.name: logstash-prod-01

# ─── Path ──────────────────────────────────────────────────────────────────
path.data:    /var/lib/logstash
path.logs:    /var/log/logstash
path.config:  /etc/logstash/conf.d
path.plugins: /usr/share/logstash/plugins

# ─── Pipeline (global defaults; override per-pipeline in pipelines.yml) ────
pipeline.id:           main
pipeline.workers:      4              # Default: number of CPU cores
pipeline.batch.size:   125            # Events per batch per worker
pipeline.batch.delay:  50             # ms to wait for batch to fill
pipeline.unsafe_shutdown: false       # true = don't drain queue on SIGTERM
pipeline.ordered:      auto           # auto | true | false
pipeline.ecs_compatibility: v8        # disabled | v1 | v8

# ─── Queue ────────────────────────────────────────────────────────────────
queue.type:                  memory   # memory | persisted
queue.path:                  "${path.data}/queue"
queue.max_bytes:             1gb      # Max disk space per pipeline queue
queue.page_capacity:         64mb     # Size of each queue page file
queue.drain:                 false    # Drain queue before shutdown
queue.checkpoint.acks:       1024     # ACK checkpoint every N events
queue.checkpoint.writes:     1024     # Write checkpoint every N writes
queue.checkpoint.interval:   1000     # ms between checkpoint flushes

# ─── Dead Letter Queue ─────────────────────────────────────────────────────
dead_letter_queue.enable:            true
dead_letter_queue.max_bytes:         1gb
dead_letter_queue.storage_policy:    drop_newer   # drop_newer | drop_older
dead_letter_queue.flush_interval:    5000         # ms
dead_letter_queue.path:              "${path.data}/dead_letter_queue"

# ─── HTTP API (monitoring) ─────────────────────────────────────────────────
http.host:  "127.0.0.1"
http.port:  9600

# ─── Logging ───────────────────────────────────────────────────────────────
log.level:              info    # debug | info | warn | error | fatal
log.format:             plain   # plain | json
log.format.json.fix_duplicate_message_fields: true

# ─── Monitoring (ship metrics to Elasticsearch) ────────────────────────────
monitoring.enabled:                   false
monitoring.elasticsearch.hosts:       ["https://es-node:9200"]
monitoring.elasticsearch.username:    logstash_system
monitoring.elasticsearch.password:    "${LOGSTASH_SYSTEM_PASSWORD}"
monitoring.elasticsearch.ssl.certificate_authority: /etc/logstash/ca.crt
monitoring.collection.interval:       10s
monitoring.collection.pipeline.details.enabled: true

# ─── X-Pack Centralized Pipeline Management 💼 ─────────────────────────────
xpack.management.enabled:              false
xpack.management.pipeline.id:          ["main", "apache-pipeline"]
xpack.management.elasticsearch.hosts:  ["https://es-node:9200"]
xpack.management.elasticsearch.username: logstash_admin
xpack.management.elasticsearch.password: "${MGMT_PASSWORD}"
xpack.management.logstash.poll_interval: 5s
```

---

<!-- section: pipeline-syntax -->
## 4. Pipeline Configuration Syntax

### Basic Structure

```ruby
# /etc/logstash/conf.d/pipeline.conf
input {
  beats {
    port => 5044
    ssl  => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key         => "/etc/logstash/certs/logstash.key"
  }
}

filter {
  if [type] == "apache" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
    date {
      match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
      target => "@timestamp"
      remove_field => ["timestamp"]
    }
  }
}

output {
  elasticsearch {
    hosts    => ["https://es-node:9200"]
    user     => "logstash_writer"
    password => "${ES_PASSWORD}"
    index    => "apache-%{+YYYY.MM.dd}"
  }
}
```

### Field References

```ruby
# Top-level field
[message]
[type]
[@timestamp]
[@version]

# Nested field (dot notation equivalent)
[http][response][status_code]
[source][ip]
[kubernetes][pod][name]

# @metadata — never forwarded to outputs
[@metadata][pipeline]
[@metadata][target_index]
[@metadata][document_id]

# String interpolation in double-quoted strings
"%{[source][ip]}"
"%{[kubernetes][namespace]}-%{+YYYY.MM}"
"index-%{[@metadata][env]}-%{+YYYY.MM.dd}"
```

### Conditionals

```ruby
# Equality
if [status] == "200" { ... }
if [status] != "200" { ... }

# Numeric comparisons
if [http][response][bytes] > 1048576 { ... }
if [response_time_ms] >= 1000 { ... }

# Regex match (~ uses Ruby regex)
if [message] =~ /ERROR|FATAL/ { ... }
if [host][name] !~ /^prod-/ { ... }

# Array membership
if [tags] in ["_grokparsefailure", "_jsonparsefailure"] { ... }
if "debug" not in [tags] { ... }

# Field exists (non-nil, non-empty)
if [user][id] { ... }
if ![error][message] { ... }    # field does NOT exist

# Boolean logic
if [status] == "active" and [type] == "order" { ... }
if [level] == "error" or [level] == "fatal" { ... }
if !([status] == "200") { ... }

# Compound routing example
filter {
  if [agent][type] == "filebeat" {
    if [log][file][path] =~ /nginx/ {
      grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
      mutate { add_field => { "[event][dataset]" => "nginx.access" } }
    } else if [log][file][path] =~ /syslog/ {
      grok { match => { "message" => "%{SYSLOGLINE}" } }
      mutate { add_field => { "[event][dataset]" => "system.syslog" } }
    } else {
      mutate { add_tag => ["unrecognized_source"] }
    }
  } else if [tags] in ["_grokparsefailure"] {
    mutate { add_field => { "[@metadata][dlq_reason]" => "grok_failure" } }
  }
}
```

### Environment Variables & Keystore

```ruby
# Environment variable with default
input {
  jdbc {
    jdbc_connection_string => "jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/mydb"
    jdbc_user              => "${DB_USER}"
    jdbc_password          => "${DB_PASSWORD}"
  }
}

# Keystore secret (set via: logstash-keystore add ES_API_KEY)
output {
  elasticsearch {
    api_key => "${ES_API_KEY}"
  }
}
```

```bash
# Keystore management
logstash-keystore create
logstash-keystore add ES_PASSWORD
logstash-keystore add DB_PASSWORD
logstash-keystore list
logstash-keystore remove OLD_KEY
logstash-keystore --path.settings /etc/logstash passwd  # Change keystore password
```

### Multiple Config Files Loading Order

```
# Files in conf.d/ are loaded and concatenated alphabetically:
01-inputs.conf   → input { }
10-filters.conf  → filter { }
99-outputs.conf  → output { }

# This means all input{}, filter{}, output{} blocks from all files
# are merged into a single pipeline config in alphabetical order.
# Use numeric prefixes to control ordering.
```

> **Warning:** If you mix `input {}`, `filter {}`, and `output {}` blocks across files, the order within each section type is determined by filename order. Keep inputs in low-numbered files and outputs in high-numbered files.

---

<!-- section: inputs -->
## 5. Input Plugins

### beats — Receive from Beats/Elastic Agent

```ruby
input {
  beats {
    port => 5044
    host => "0.0.0.0"

    # TLS (recommended in production)
    ssl                        => true
    ssl_certificate            => "/etc/logstash/certs/logstash.crt"
    ssl_key                    => "/etc/logstash/certs/logstash.key"
    ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
    ssl_verify_mode            => "force_peer"   # none | peer | force_peer

    client_inactivity_timeout  => 60
    include_codec_tag          => true
    codec                      => plain
  }
}
```

> **Note:** For multiple Logstash nodes, configure Filebeat with multiple hosts and `loadbalance: true`. Each Logstash node should have a unique server certificate.

### file — Tail Log Files

```ruby
input {
  file {
    path           => ["/var/log/nginx/access.log", "/var/log/nginx/error.log"]
    start_position => "beginning"    # beginning | end
    sincedb_path   => "/var/lib/logstash/sincedb/.sincedb_nginx"
    sincedb_write_interval => 15     # seconds between sincedb writes
    stat_interval  => 1              # seconds between file stat checks
    mode           => "tail"         # tail | read (read = process whole file then close)
    file_completed_action  => "log"  # For mode:read — log | delete | do_nothing
    file_completed_log_path => "/var/log/logstash/completed_files.log"
    discover_interval => 15          # seconds between directory scans for new files
    ignore_older    => "24h"         # Ignore files not modified in this time
    close_older     => "1h"          # Close file handle after idle this long
    delimiter       => "\n"          # Line delimiter
    codec           => plain { charset => "UTF-8" }
    add_field       => { "[event][module]" => "nginx" }
    type            => "nginx_access"
  }
}
```

> **Note:** `sincedb` tracks file positions using inode numbers. If a log file is rotated (new inode), Logstash will re-read from the beginning of the new file. Set `start_position => "end"` on first run to avoid re-processing existing content.

### kafka — Consume from Kafka

```ruby
input {
  kafka {
    bootstrap_servers   => "kafka-1:9092,kafka-2:9092,kafka-3:9092"
    topics              => ["app-logs", "system-logs"]
    # OR topics_pattern => "logs-.*"
    group_id            => "logstash-consumer-group-1"
    consumer_threads    => 4          # Match or be multiple of Kafka partition count
    auto_offset_reset   => "latest"   # earliest | latest | none
    enable_auto_commit  => true
    auto_commit_interval_ms => "5000"
    max_poll_records    => 500
    session_timeout_ms  => "30000"
    fetch_max_bytes     => "52428800" # 50 MB
    codec               => json
    decorate_events     => extended   # true | false | extended — adds kafka.* metadata
    
    # SASL/SSL security
    security_protocol           => "SASL_SSL"
    sasl_mechanism              => "SCRAM-SHA-256"
    sasl_jaas_config            => "org.apache.kafka.common.security.scram.ScramLoginModule required username='logstash' password='${KAFKA_PASSWORD}';"
    ssl_truststore_location     => "/etc/logstash/certs/kafka-truststore.jks"
    ssl_truststore_password     => "${KAFKA_TRUSTSTORE_PASSWORD}"
    ssl_keystore_location       => "/etc/logstash/certs/kafka-keystore.jks"
    ssl_keystore_password       => "${KAFKA_KEYSTORE_PASSWORD}"
    ssl_endpoint_identification_algorithm => "https"  # "" to disable hostname verification
  }
}
```

### jdbc — Poll Relational Databases

```ruby
input {
  jdbc {
    jdbc_driver_library         => "/usr/share/logstash/drivers/postgresql-42.6.0.jar"
    jdbc_driver_class           => "org.postgresql.Driver"
    jdbc_connection_string      => "jdbc:postgresql://db-host:5432/mydb"
    jdbc_user                   => "${DB_USER}"
    jdbc_password               => "${DB_PASSWORD}"
    jdbc_validate_connection    => true
    jdbc_validation_timeout     => 3600

    # Incremental sync using tracking column
    use_column_value            => true
    tracking_column             => "updated_at"
    tracking_column_type        => "timestamp"   # numeric | timestamp
    last_run_metadata_path      => "/var/lib/logstash/.jdbc_last_run_orders"

    # Full sync with schedule
    statement                   => "SELECT * FROM orders WHERE updated_at > :sql_last_value ORDER BY updated_at ASC"
    schedule                    => "*/5 * * * *"   # cron: every 5 minutes
    
    jdbc_paging_enabled         => true
    jdbc_page_size              => 10000
    lowercase_column_names      => true
    record_last_run             => true
    clean_run                   => false          # true = ignore last_run on startup
    
    add_field => {
      "[event][dataset]" => "orders"
      "[event][module]"  => "myapp"
    }
  }
}
```

### http — HTTP Webhook Listener

```ruby
input {
  http {
    port              => 8080
    host              => "0.0.0.0"
    user              => "webhookuser"
    password          => "${HTTP_INPUT_PASSWORD}"
    ssl               => true
    ssl_certificate   => "/etc/logstash/certs/logstash.crt"
    ssl_key           => "/etc/logstash/certs/logstash.key"
    codec             => json
    threads           => 4
    response_code     => 200
    response_headers  => { "Content-Type" => "application/json" }
    additional_codecs => { "application/json" => "json" }
  }
}
```

### s3 — Read from Amazon S3

```ruby
input {
  s3 {
    bucket               => "my-log-bucket"
    region               => "us-east-1"
    prefix               => "logs/production/"
    interval             => 60           # seconds between bucket polls
    sincedb_path         => "/var/lib/logstash/.s3_sincedb"
    delete               => false        # delete from S3 after reading
    backup_to_bucket     => "my-log-archive-bucket"
    backup_add_prefix    => "processed/"
    codec                => json_lines
    
    # Credentials (prefer IAM role over explicit keys)
    access_key_id        => "${AWS_ACCESS_KEY_ID}"
    secret_access_key    => "${AWS_SECRET_ACCESS_KEY}"
    # session_token      => "${AWS_SESSION_TOKEN}"   # For temp credentials
    
    additional_settings  => {
      "force_path_style"     => true
      "follow_redirects"     => false
    }
  }
}
```

### sqs — Amazon SQS

```ruby
input {
  sqs {
    queue          => "logstash-input-queue"
    region         => "us-east-1"
    threads        => 2
    batch_count    => 10              # Messages per SQS receive call (max 10)
    visibility_timeout => 30          # seconds
    wait_time_seconds  => 20          # Long polling (0-20)
    codec          => json
  }
}
```

### Other Key Inputs

```ruby
# TCP (syslog over TCP)
input {
  tcp {
    port  => 5000
    mode  => "server"
    codec => line
    ssl_enable => false
  }
}

# UDP (syslog)
input {
  udp {
    port        => 5140
    buffer_size => 65536
    codec       => plain
    workers     => 2
  }
}

# Syslog (RFC 3164 / RFC 5424)
input {
  syslog {
    port          => 5140
    protocol      => "udp"            # udp | tcp
    use_labels    => true
    timezone      => "Asia/Kolkata"
  }
}

# Elasticsearch (re-process existing data)
input {
  elasticsearch {
    hosts    => ["https://es-node:9200"]
    user     => "elastic"
    password => "${ES_PASSWORD}"
    index    => "old-index-*"
    query    => '{ "query": { "range": { "@timestamp": { "gte": "now-7d" } } } }'
    size     => 500
    scroll   => "5m"
  }
}

# Redis
input {
  redis {
    host       => "redis-host"
    port       => 6379
    password   => "${REDIS_PASSWORD}"
    key        => "logstash"
    data_type  => "list"              # list | channel | pattern_channel
    batch_count => 125
  }
}

# Heartbeat (testing / synthetic events)
input {
  heartbeat {
    interval => 5
    message  => "pipeline_alive"
    type     => "heartbeat"
  }
}

# Dead Letter Queue reprocessing
input {
  dead_letter_queue {
    path               => "/var/lib/logstash/dead_letter_queue"
    pipeline_id        => "main"
    commit_offsets     => true
    age_retention      => "7d"
    start_timestamp    => "2024-01-01T00:00:00"
  }
}
```

---

<!-- section: codecs -->
## 6. Codec Plugins

Codecs operate on the input/output level, encoding or decoding the raw byte stream.

```ruby
# json — single JSON object per message
input  { beats { port => 5044; codec => json } }
output { file  { path => "/tmp/out.json"; codec => json } }

# json_lines — one JSON per line (NDJSON / newline-delimited)
input  { file { path => "/logs/app.jsonl"; codec => json_lines } }
output { file { path => "/out/events.ndjson"; codec => json_lines } }

# plain — raw text, no parsing
input { tcp { port => 5000; codec => plain { charset => "UTF-8" } } }

# rubydebug — human-readable debug output (use ONLY in stdout for debugging)
output { stdout { codec => rubydebug { metadata => true } } }

# dots — print one dot per event (throughput indicator)
output { stdout { codec => dots } }

# line — single line with optional format string
output { file { path => "/out.log"; codec => line { format => "%{@timestamp} %{message}" } } }

# csv
input { file { path => "/data/events.csv";
  codec => csv {
    separator => ","
    columns   => ["timestamp", "host", "level", "message"]
    skip_header => true
  }
}}

# multiline — combine multi-line records into one event
input {
  file {
    path  => "/var/log/app/application.log"
    codec => multiline {
      pattern           => "^\d{4}-\d{2}-\d{2}"  # Line starts with date = new event
      negate            => true                    # Lines NOT matching pattern = continuation
      what              => "previous"              # Append to PREVIOUS event
      max_lines         => 500
      max_bytes         => "10mb"
      auto_flush_interval => 2                     # Flush incomplete event after N seconds
    }
  }
}

# Java stack trace multiline example
codec => multiline {
  pattern => "^[^\s]"        # Lines starting with non-whitespace = new event
  negate  => false
  what    => "previous"
}
```

> **Note:** Multiline in Logstash is stateful per-input-thread. If you have multiple Logstash workers reading the same file source, configure multiline in Filebeat instead (via `multiline` settings in the filebeat input) to avoid split-trace issues.

---

<!-- section: filters -->
## 7. Filter Plugins

### grok — Regex Field Extraction

The most powerful and most commonly used filter. Grok uses named regex patterns to extract fields from unstructured text.

```ruby
filter {
  grok {
    match => {
      "message" => [
        # Try pattern 1 first, fall back to pattern 2
        "%{COMBINEDAPACHELOG}",
        "%{COMMONAPACHELOG}",
        "(?<raw_message>.+)"    # catch-all fallback
      ]
    }
    overwrite             => ["message"]
    named_captures_only   => true      # Only add fields for named captures
    keep_empty_captures   => false     # Don't add fields for empty matches
    tag_on_failure        => ["_grokparsefailure"]
    tag_on_timeout        => ["_groktimeout"]
    timeout_scope         => "event"   # event | pattern
    timeout_millis        => 5000
    patterns_dir          => ["/etc/logstash/patterns"]
  }
}
```

**Custom pattern file** (`/etc/logstash/patterns/custom`):
```
# pattern_name  REGEX
SERVICE_NAME    [a-zA-Z0-9_\-]+
REQUEST_ID      [0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}
DURATION_MS     \d+(\.\d+)?
```

**Inline named capture:**
```ruby
match => { "message" => "User %{USERNAME:user_name} logged in from %{IP:source_ip} duration=%{NUMBER:duration_ms:float}ms" }
```

**Real-world examples:**
```ruby
# Nginx access log
match => { "message" => "%{IPORHOST:[source][ip]} - %{DATA:[user][name]} \[%{HTTPDATE:[event][created]}\] \"%{WORD:[http][request][method]} %{URIPATHPARAM:[url][path]} HTTP/%{NUMBER:[http][version]}\" %{NUMBER:[http][response][status_code]:int} %{NUMBER:[http][response][body][bytes]:int} \"%{DATA:[http][request][referrer]}\" \"%{DATA:[user_agent][original]}\"" }

# Syslog
match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }

# Java exception (first line only — use multiline codec for full trace)
match => { "message" => "%{JAVACLASS:[error][type]}: %{GREEDYDATA:[error][message]}" }
```

### dissect — Fast Ordered Field Extraction

```ruby
filter {
  dissect {
    mapping => {
      "message" => "%{[@timestamp]} %{+[@timestamp]} [%{[log][level]}] %{[service][name]} - %{[trace][id]} %{[log][logger]} %{?thread_name}: %{[message]}"
    }
    convert_datatype => {
      "[http][response][status_code]" => "integer"
      "[http][response][body][bytes]"  => "integer"
    }
    tag_on_failure => ["_dissectfailure"]
  }
}
```

**Dissect notation:**

| Notation | Meaning |
|----------|---------|
| `%{field}` | Capture into `field` |
| `%{+field}` | Append to `field` (with space separator) |
| `%{+field/2}` | Append with 2nd occurrence of delimiter |
| `%{?skip}` | Capture but discard (skip field) |
| `%{*key}` | Dynamic key reference |
| `%{&value}` | Value for preceding `%{*key}` |
| `%{}` | Skip, no name |

**Grok vs Dissect:**

| Aspect | Grok | Dissect |
|--------|------|---------|
| Format | Any regex | Fixed delimiter-separated |
| Speed | Slower (regex engine) | **Much faster** (string splitting) |
| Flexibility | Very high | Requires consistent format |
| Backtracking risk | Yes (use `timeout_millis`) | No |
| Best for | Variable/complex formats | Structured logs with consistent delimiters |

### mutate — Field Manipulation

```ruby
filter {
  mutate {
    # Add new fields
    add_field => {
      "[event][dataset]"   => "nginx.access"
      "[event][module]"    => "nginx"
      "[ecs][version]"     => "8.11.0"
      "[@metadata][index]" => "nginx-%{+YYYY.MM.dd}"
    }

    # Remove fields
    remove_field => ["message", "beat", "prospector", "input", "offset", "host_name"]

    # Rename fields (old => new)
    rename => {
      "clientip"   => "[source][ip]"
      "agent"      => "[user_agent][original]"
      "bytes"      => "[http][response][body][bytes]"
      "verb"       => "[http][request][method]"
      "response"   => "[http][response][status_code]"
    }

    # Replace field value
    replace => { "[event][kind]" => "event" }

    # Type conversion
    convert => {
      "[http][response][status_code]"    => "integer"
      "[http][response][body][bytes]"    => "integer"
      "[http][request][body][bytes]"     => "integer"
      "duration_ms"                      => "float"
      "is_active"                        => "boolean"
    }

    # Regex replacement on string value
    gsub => [
      "[url][path]", "\?.*$", "",           # Remove query string
      "user_agent", "[,]", "",              # Remove commas from UA string
      "message", "password=\S+", "password=***"  # Redact passwords
    ]

    # String operations
    uppercase  => ["[http][request][method]"]
    lowercase  => ["[log][level]", "[event][dataset]"]
    strip      => ["message"]                 # Trim whitespace

    # Split string into array
    split      => { "tags_string" => "," }

    # Join array into string
    join       => { "tags" => ", " }

    # Merge array or hash
    merge      => { "dest_field" => "source_field" }

    # Add/remove tags
    add_tag    => ["parsed", "nginx"]
    remove_tag => ["_grokparsefailure"]
  }
}
```

### date — Parse Timestamps

```ruby
filter {
  date {
    match    => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z", "ISO8601", "UNIX", "UNIX_MS"]
    target   => "@timestamp"
    timezone => "Asia/Kolkata"
    locale   => "en"
    tag_on_failure => ["_dateparsefailure"]
    remove_field   => ["timestamp"]
  }
}
```

**Common date format strings:**

| Format | Example Input | Pattern |
|--------|--------------|---------|
| Apache/Nginx | `01/Jan/2024:12:00:00 +0000` | `dd/MMM/yyyy:HH:mm:ss Z` |
| ISO 8601 | `2024-01-01T12:00:00.000Z` | `ISO8601` |
| Unix epoch (seconds) | `1704067200` | `UNIX` |
| Unix epoch (millis) | `1704067200000` | `UNIX_MS` |
| TAI64N | `@4000000065148cfc1a4e6598` | `TAI64N` |
| MySQL/Postgres | `2024-01-01 12:00:00` | `yyyy-MM-dd HH:mm:ss` |
| Syslog | `Jan  1 12:00:00` | `MMM dd HH:mm:ss`, `MMM  d HH:mm:ss` |

> **Warning:** Always specify `timezone` if the timestamp does not include timezone info. Logstash defaults to UTC. Wrong timezone = wrong @timestamp = wrong time bucket in ES.

### json — Parse JSON Fields

```ruby
filter {
  json {
    source          => "message"
    target          => "parsed"        # Omit to merge into root of event
    skip_on_invalid_json => true
    tag_on_failure  => ["_jsonparsefailure"]
    array_function  => "hash"          # hash | separate_field
  }

  # If target omitted, fields extracted to root level
  # After parsing, remove the original message string if redundant
  if "_jsonparsefailure" not in [tags] {
    mutate { remove_field => ["message"] }
  }
}
```

### kv — Key-Value Pair Parsing

```ruby
filter {
  kv {
    source           => "message"     # Field to parse (default: message)
    target           => "kv_data"    # Destination field for parsed pairs
    field_split      => " &"         # Characters that separate key=value pairs
    value_split      => "="          # Character separating key from value
    include_keys     => ["user", "action", "session_id"]
    exclude_keys     => ["password", "token"]
    allow_duplicate_values => false
    prefix           => "kv_"
    transform_key    => "lowercase"  # lowercase | upcase | capitalize
    transform_value  => "downcase"
    trim_key         => " "          # Trim chars from key
    trim_value       => "\""         # Trim quotes from value
    tag_on_failure   => ["_kvparsefailure"]
    tag_on_timeout   => ["_kvtimeout"]
  }
}
```

### geoip — IP Geolocation

```ruby
filter {
  geoip {
    source   => "[source][ip]"
    target   => "[source][geo]"
    database => "/usr/share/logstash/vendor/bundle/jruby/2.6.0/gems/logstash-filter-geoip-7.2.13-java/vendor/GeoLite2-City.mmdb"
    fields   => ["city_name", "country_code2", "country_name", "location", "latitude", "longitude", "region_name"]
    tag_on_failure => ["_geoip_lookup_failure"]
    ecs_compatibility => "v8"
  }
}
```

> **Note:** MaxMind requires periodic database updates. Use `logstash-filter-geoip`'s built-in automatic database update (enabled by default in 7.14+) or manually update with a cron job. The GeoLite2 database requires a MaxMind license key.

### useragent — Parse User Agent Strings

```ruby
filter {
  useragent {
    source => "[user_agent][original]"
    target => "[user_agent]"
    ecs_compatibility => "v8"
  }
  # Produces: user_agent.name, user_agent.version, user_agent.os.name, user_agent.device.name
}
```

### translate — Dictionary Lookup

```ruby
filter {
  translate {
    source      => "[http][response][status_code]"
    target      => "[http][response][status_text]"
    dictionary  => {
      "200" => "OK"
      "201" => "Created"
      "400" => "Bad Request"
      "401" => "Unauthorized"
      "403" => "Forbidden"
      "404" => "Not Found"
      "500" => "Internal Server Error"
    }
    fallback         => "Unknown"
    override         => true
    refresh_interval => 300     # Reload dictionary_path every N seconds (0 = never)
  }

  # OR use external file
  translate {
    source          => "[host][name]"
    target          => "[host][environment]"
    dictionary_path => "/etc/logstash/dicts/host_environments.yml"
    refresh_interval => 60
    fallback        => "unknown"
  }
}
```

**Dictionary file** (`host_environments.yml`):
```yaml
prod-web-01: production
prod-web-02: production
stg-web-01:  staging
dev-web-01:  development
```

### ruby — Custom Processing

```ruby
filter {
  ruby {
    # Inline code
    code => '
      begin
        ts = event.get("@timestamp").to_i
        event.set("[event][duration_seconds]", (event.get("end_time").to_i - ts).abs)
        
        # Manipulate arrays
        tags = event.get("tags") || []
        tags << "processed_by_ruby"
        event.set("tags", tags)
        
        # Conditional field removal
        if event.get("[http][response][status_code]").to_i < 400
          event.cancel   # Drop non-error events
        end
      rescue => e
        event.tag("_ruby_exception")
        event.set("[@metadata][ruby_error]", e.message)
      end
    '
  }

  # External Ruby script
  ruby {
    path          => "/etc/logstash/scripts/custom_transform.rb"
    script_params => {
      "environment" => "${ENVIRONMENT}"
      "threshold"   => 1000
    }
  }
}
```

**External Ruby script** (`custom_transform.rb`):
```ruby
def register(params)
  @environment = params["environment"]
  @threshold   = params["threshold"].to_i
  @logger.info("Custom transform initialized", env: @environment)
end

def filter(event)
  duration = event.get("[transaction][duration_ms]").to_i
  
  event.set("[transaction][is_slow]", duration > @threshold)
  event.set("[deployment][environment]", @environment)
  
  # Return array of events (can return multiple or empty to drop)
  [event]
end
```

### aggregate — Correlate Multi-Event Records

```ruby
# Use case: Correlate START and END log lines into a single transaction event
filter {
  if [message] =~ /^START request/ {
    grok { match => { "message" => "^START request id=%{UUID:request_id} path=%{URIPATH:[url][path]}" } }
    aggregate {
      task_id    => "%{request_id}"
      code       => "
        map['start_time'] = event.get('@timestamp').to_f
        map['path']       = event.get('[url][path]')
        map['request_id'] = event.get('request_id')
        event.cancel   # Don't forward the START event
      "
      map_action => "create"
    }
  }

  if [message] =~ /^END request/ {
    grok { match => { "message" => "^END request id=%{UUID:request_id} status=%{NUMBER:status_code:int}" } }
    aggregate {
      task_id    => "%{request_id}"
      code       => "
        end_time = event.get('@timestamp').to_f
        event.set('[transaction][duration_ms]', ((end_time - map['start_time']) * 1000).round(2))
        event.set('[url][path]',               map['path'])
        event.set('[transaction][id]',         map['request_id'])
        event.set('[http][response][status_code]', event.get('status_code'))
      "
      map_action         => "update"
      end_of_task        => true
      timeout            => 120    # Drop incomplete transactions after 120s
      timeout_code       => "event.set('[@metadata][aggregate_timeout]', true)"
      push_map_as_event_on_timeout => true
    }
  }
}
```

### fingerprint — Deterministic Document ID

```ruby
filter {
  fingerprint {
    source              => ["[source][ip]", "[url][path]", "@timestamp", "message"]
    target              => "[@metadata][fingerprint]"
    method              => "SHA256"      # SHA1 | SHA256 | SHA512 | MD5 | MURMUR3 | UUID
    key                 => "my-secret-hmac-key"   # For HMAC-based methods
    concatenate_sources => true
    base64encode        => true
  }
  # Then in output: document_id => "%{[@metadata][fingerprint]}"
}
```

### Other Essential Filters

```ruby
# clone — duplicate event and route copies separately
filter {
  clone {
    clones => ["security_copy", "metrics_copy"]
  }
  # Now have 3 events: original + "security_copy" type + "metrics_copy" type
  # Use type-based routing in output section
}

# split — one event with array → multiple events
filter {
  split {
    field       => "records"     # Must be an array field
    target      => "record"      # Each array element stored here
    remove_field => ["records"]
  }
}

# drop — discard events
filter {
  if [http][request][method] == "OPTIONS" { drop {} }
  if [log][level] == "DEBUG" and [@metadata][env] != "development" { drop {} }
  if [tags] in ["heartbeat"] { drop {} }
  # Probabilistic sampling — drop 90% of debug events
  if [level] == "debug" {
    drop { percentage => 90 }
  }
}

# de_dot — fix field names with dots (ES doesn't like dots in field names)
filter {
  de_dot {
    separator => "_"
    nested    => true
    fields    => ["kubernetes.pod.name", "kubernetes.namespace"]
  }
}

# cidr — classify IP ranges
filter {
  cidr {
    address => ["%{[source][ip]}"]
    network => ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"]
    add_tag => ["private_network"]
  }
}

# truncate — limit field length
filter {
  truncate {
    fields       => ["message", "[error][stack_trace]"]
    length_bytes => 32768    # 32 KB max
  }
}

# prune — whitelist fields
filter {
  prune {
    whitelist_names => ["@timestamp", "@version", "message", "host", "source", "http", "url", "user_agent", "event", "ecs"]
  }
}

# xml — parse XML field
filter {
  xml {
    source      => "xml_payload"
    target      => "xml_parsed"
    force_array => false
    store_xml   => false
    xpath       => [
      "//order/id/text()",     "[order][id]",
      "//order/amount/text()", "[order][amount]"
    ]
  }
}

# throttle — rate-limit alerts
filter {
  throttle {
    key          => "%{[source][ip]}"
    before_count => -1         # -1 = no limit before threshold
    after_count  => 10         # Tag after 10th event per key in period
    period       => 60         # seconds window
    add_tag      => ["throttled"]
  }
}
```

---

<!-- section: outputs -->
## 8. Output Plugins

### elasticsearch — Primary Output

```ruby
output {
  elasticsearch {
    hosts    => ["https://es-node-1:9200", "https://es-node-2:9200"]
    user     => "logstash_writer"
    password => "${ES_PASSWORD}"
    # OR: api_key => "${ES_API_KEY}"

    # TLS
    ssl                      => true
    cacert                   => "/etc/logstash/certs/ca.crt"
    ssl_certificate_verification => true

    # Index / Data Stream targeting
    index            => "logs-%{[event][dataset]}-%{+YYYY.MM.dd}"
    # OR for Data Streams:
    data_stream        => true
    data_stream_type   => "logs"
    data_stream_dataset => "%{[event][dataset]}"
    data_stream_namespace => "production"

    # Document settings
    document_id  => "%{[@metadata][fingerprint]}"   # Idempotent indexing
    action       => "index"                          # index | create | update | delete | upsert

    # For upsert (update or create)
    # action     => "update"
    # upsert     => '{ "status": "new" }'

    # Ingest pipeline
    pipeline     => "my-enrichment-pipeline"

    # ILM settings
    ilm_enabled          => true
    ilm_rollover_alias   => "logs-app"
    ilm_pattern          => "{now/d}-000001"
    ilm_policy           => "logs-ilm-policy"

    # Index template
    manage_template      => true
    template_name        => "logs-app"
    template             => "/etc/logstash/templates/logs-app-template.json"
    template_overwrite   => false
    template_api         => "auto"               # auto | legacy | composable

    # Bulk performance
    bulk_max_size        => 250                  # Events per ES bulk request
    timeout              => 60                   # HTTP request timeout (seconds)
    retry_max_interval   => 64                   # Max retry backoff (seconds)
    retry_initial_interval => 2                  # Initial retry backoff (seconds)
    
    # Pool / connection
    pool_max             => 1000                 # Max HTTP connections
    pool_max_per_route   => 100
    
    ecs_compatibility    => "v8"
  }
}
```

### kafka — Produce to Kafka

```ruby
output {
  kafka {
    bootstrap_servers  => "kafka-1:9092,kafka-2:9092,kafka-3:9092"
    topic_id           => "processed-logs"
    codec              => json
    message_key        => "%{[source][ip]}"    # Partition by source IP
    compression_type   => "snappy"             # none | gzip | snappy | lz4 | zstd
    batch_size         => 16384
    linger_ms          => 5
    acks               => "all"                # 0 | 1 | all
    retries            => 3
    
    # SASL/SSL
    security_protocol           => "SASL_SSL"
    sasl_mechanism              => "SCRAM-SHA-256"
    sasl_jaas_config            => "org.apache.kafka.common.security.scram.ScramLoginModule required username='logstash' password='${KAFKA_PASSWORD}';"
    ssl_truststore_location     => "/etc/logstash/certs/kafka-truststore.jks"
    ssl_truststore_password     => "${KAFKA_TRUSTSTORE_PASSWORD}"
  }
}
```

### s3 — Write to Amazon S3

```ruby
output {
  s3 {
    bucket           => "my-log-archive"
    region           => "ap-south-1"
    prefix           => "year=%{+YYYY}/month=%{+MM}/day=%{+dd}/hour=%{+HH}/"
    codec            => json_lines
    encoding         => "gzip"
    rotation_strategy => "time"       # size_and_time | size | time
    time_file        => 15            # Rotate every 15 minutes
    size_file        => 10485760      # Rotate at 10 MB
    canned_acl       => "private"
    server_side_encryption => "AES256"
    temporary_directory    => "/tmp/logstash-s3"
    upload_multipart_threshold => 104857600  # 100 MB
  }
}
```

### http — Webhook / REST API

```ruby
output {
  http {
    url             => "https://api.example.com/v1/events"
    http_method     => "post"
    format          => "json_batch"       # json | json_batch | form | message
    batch_size      => 50
    headers         => {
      "Authorization" => "Bearer ${API_TOKEN}"
      "Content-Type"  => "application/json"
    }
    ssl_verification_mode => "full"
    keepalive       => true
    retry_failed    => true
    retryable_codes => [429, 500, 502, 503, 504]
    socket_timeout  => 10
    request_timeout => 60
  }
}
```

### Other Outputs

```ruby
# File output (log rotation)
output {
  file {
    path           => "/var/log/logstash-output/%{[event][dataset]}-%{+YYYY-MM-dd}.log"
    codec          => json_lines
    flush_interval => 2
    gzip           => true
    dir_mode       => 0750
    file_mode      => 0640
  }
}

# StatsD (metrics forwarding)
output {
  statsd {
    host      => "statsd.monitoring.svc"
    port      => 8125
    namespace => "logstash.pipeline"
    gauge     => { "events_in" => "%{[pipeline][events][in]}" }
    increment => ["events.%{[event][dataset]}"]
    timing    => { "duration_ms" => "%{[transaction][duration_ms]}" }
    count     => { "bytes" => "%{[http][response][body][bytes]}" }
  }
}

# stdout (debug only)
output {
  stdout { codec => rubydebug { metadata => true } }
}

# Null (benchmarking / testing)
output { null {} }

# Conditional multi-output
output {
  if [event][module] == "nginx" {
    elasticsearch { index => "nginx-%{+YYYY.MM.dd}" hosts => ["https://es:9200"] }
  } else if "_grokparsefailure" in [tags] {
    elasticsearch { index => "failed-parsing-%{+YYYY.MM.dd}" hosts => ["https://es:9200"] }
    file { path => "/var/log/logstash/parse_failures.log" codec => json_lines }
  } else {
    elasticsearch { index => "generic-logs-%{+YYYY.MM.dd}" hosts => ["https://es:9200"] }
  }
}
```

---

<!-- section: pq -->
## 9. Persistent Queue (PQ)

```
In-Memory Queue (default):
  ┌──────┐   push   ┌───────────────────────────────┐   batch   ┌────────┐
  │Input │ ───────► │  Ring buffer (in JVM heap)    │ ────────► │Filter/ │
  └──────┘          └───────────────────────────────┘           │Output  │
                    ⚠️ Data lost on crash/restart                └────────┘
                    ✅ Lower latency, less disk I/O

Persistent Queue (persisted):
  ┌──────┐   push   ┌───────────────────────────────┐   batch   ┌────────┐
  │Input │ ───────► │  Disk pages + checkpoints     │ ────────► │Filter/ │
  └──────┘   ACK◄── └───────────────────────────────┘  ACK◄──  │Output  │
                    ✅ Survives crash/restart                    └────────┘
                    ✅ Absorbs burst traffic
                    ⚠️ Higher latency, disk I/O
```

| Aspect | In-Memory | Persistent |
|--------|-----------|------------|
| Durability | None (data lost on crash) | Yes (WAL-based) |
| Performance | Higher throughput | ~10-20% overhead |
| Burst absorption | Limited by heap | Limited by disk |
| Disk usage | None | Configurable (default: 1 GB) |
| Use case | Latency-sensitive, data can be re-read | Production logs, cannot afford data loss |

```yaml
# logstash.yml — enable PQ
queue.type:           persisted
queue.max_bytes:      4gb         # Size limit per pipeline
queue.page_capacity:  64mb        # Individual page file size
queue.drain:          true        # Process remaining events before shutdown
queue.checkpoint.acks:   1024
queue.checkpoint.writes: 1024
queue.checkpoint.interval: 1000
```

**Monitor queue depth:**
```bash
curl -s localhost:9600/_node/stats/pipelines | jq '.pipelines.main.queue'
# → { type, events_count, queue_size_in_bytes, max_queue_size_in_bytes, ... }
```

**Disk sizing:** `queue.max_bytes` × number of pipelines × safety factor (1.5x). For a 4-pipeline deployment with 1 GB each: `4 × 1 GB × 1.5 = 6 GB` reserved for PQ.

> **Warning:** PQ pages are only deleted after events are ACKed by the output. If your output is perpetually failing (e.g., ES down), the queue will fill to `queue.max_bytes` and backpressure will propagate to inputs (blocking).

---

<!-- section: dlq -->
## 10. Dead Letter Queue (DLQ)

### When DLQ Is Triggered

| Scenario | Goes to DLQ? |
|----------|-------------|
| ES output rejects event (400 mapping conflict) | ✅ Yes |
| ES bulk request fails after max retries | ✅ Yes |
| Filter plugin raises exception | ❌ No (event tagged, continues) |
| Input plugin error | ❌ No |
| `drop {}` filter | ❌ No (event discarded) |
| ES returns 429 (too many requests) | ❌ No (retried indefinitely) |

```yaml
# logstash.yml
dead_letter_queue.enable: true
dead_letter_queue.max_bytes: 2gb
dead_letter_queue.storage_policy: drop_newer   # drop_newer | drop_older
dead_letter_queue.flush_interval: 5000
```

### DLQ Event Metadata

```ruby
# Fields added to events in the DLQ:
[@metadata][dead_letter_queue][original_data]   # Original serialized event
[@metadata][dead_letter_queue][reason]           # Why it was rejected
[@metadata][dead_letter_queue][entry_time]       # When it entered the DLQ
[@metadata][dead_letter_queue][plugin_id]        # Which plugin rejected it
[@metadata][dead_letter_queue][plugin_type]      # output | codec
```

### DLQ Reprocessing Pipeline

```ruby
# /etc/logstash/conf.d/dlq-reprocessor.conf
input {
  dead_letter_queue {
    path            => "/var/lib/logstash/dead_letter_queue"
    pipeline_id     => "main"
    commit_offsets  => true
    age_retention   => "7d"
  }
}

filter {
  # Inspect the rejection reason
  if [@metadata][dead_letter_queue][reason] =~ /mapper_parsing_exception/ {
    # Fix type mismatch — convert the problematic field
    mutate {
      convert => { "[http][response][status_code]" => "string" }
    }
  }
  
  if [@metadata][dead_letter_queue][reason] =~ /illegal_argument_exception.*text/ {
    # Remove field causing mapping conflict
    mutate {
      remove_field => ["problematic_field"]
    }
  }

  # Prevent re-routing to DLQ in a loop
  mutate {
    add_field => { "[@metadata][from_dlq]" => "true" }
  }
}

output {
  elasticsearch {
    hosts        => ["https://es:9200"]
    user         => "logstash_writer"
    password     => "${ES_PASSWORD}"
    index        => "dlq-reprocessed-%{+YYYY.MM.dd}"
    action       => "index"
  }
}
```

> **Tip:** Run the DLQ reprocessor as a separate pipeline in `pipelines.yml`. Use a distinct pipeline ID so it has its own DLQ — prevent infinite loops by checking `[@metadata][from_dlq]` in your output conditional.

---

<!-- section: multiple-pipelines -->
## 11. Multiple Pipelines

### pipelines.yml

```yaml
# /etc/logstash/pipelines.yml
- pipeline.id: beats-ingestion
  path.config: /etc/logstash/pipelines/beats-ingestion/
  pipeline.workers: 4
  pipeline.batch.size: 250
  queue.type: persisted
  queue.max_bytes: 2gb
  dead_letter_queue.enable: true

- pipeline.id: kafka-high-throughput
  path.config: /etc/logstash/pipelines/kafka-pipeline.conf
  pipeline.workers: 8
  pipeline.batch.size: 500
  queue.type: persisted
  queue.max_bytes: 4gb

- pipeline.id: dlq-reprocessor
  path.config: /etc/logstash/pipelines/dlq-reprocessor.conf
  pipeline.workers: 1
  pipeline.batch.size: 50
  queue.type: memory

- pipeline.id: jdbc-sync
  path.config: /etc/logstash/pipelines/jdbc-sync.conf
  pipeline.workers: 1
  pipeline.batch.size: 100
  queue.type: memory
```

### Pipeline-to-Pipeline (P2P)

```
Distributor Pipeline
   beats input
       │
       ├──► [type==nginx]     ──── pipeline output ───► Nginx Pipeline ──► ES nginx-*
       ├──► [type==syslog]    ──── pipeline output ───► Syslog Pipeline ──► ES syslog-*
       └──► [type==app_logs]  ──── pipeline output ───► App Pipeline ───► ES app-*
```

```ruby
# distributor.conf — receives all beats traffic, routes to sub-pipelines
input {
  beats { port => 5044 ssl => true ssl_certificate => "..." ssl_key => "..." }
}

filter {
  # Tag by source to route downstream
  if [event][dataset] =~ /^nginx/ {
    mutate { add_field => { "[@metadata][pipeline]" => "nginx-pipeline" } }
  } else if [event][dataset] =~ /^system/ {
    mutate { add_field => { "[@metadata][pipeline]" => "syslog-pipeline" } }
  } else {
    mutate { add_field => { "[@metadata][pipeline]" => "app-pipeline" } }
  }
}

output {
  pipeline {
    send_to        => ["%{[@metadata][pipeline]}"]
    ensure_delivery => true   # Block until downstream pipeline accepts event
  }
}
```

```ruby
# nginx-pipeline.conf — downstream pipeline for nginx logs
input {
  pipeline { address => "nginx-pipeline" }
}

filter {
  grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
  date { match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"] target => "@timestamp" }
  geoip { source => "[source][ip]" target => "[source][geo]" }
  useragent { source => "[user_agent][original]" target => "[user_agent]" }
  mutate {
    convert => {
      "[http][response][status_code]" => "integer"
      "[http][response][body][bytes]"  => "integer"
    }
    remove_field => ["timestamp", "message"]
  }
}

output {
  elasticsearch {
    hosts => ["https://es:9200"]
    data_stream => true
    data_stream_type      => "logs"
    data_stream_dataset   => "nginx.access"
    data_stream_namespace => "production"
  }
}
```

> **Note:** P2P uses an in-memory channel. If the downstream pipeline is slow, backpressure propagates to the distributor. Size `pipeline.batch.size` appropriately and monitor with `/_node/stats/pipelines`.

---

<!-- section: performance -->
## 13. Performance Tuning

### Tuning Methodology

```
1. Establish baseline: curl localhost:9600/_node/stats/pipelines
   → Note events.in, events.out, events.duration_in_millis per second

2. Identify bottleneck:
   → Input limited:  queue.events_count growing, filter workers idle
   → Filter limited: pipeline.duration_in_millis high, workers always busy
   → Output limited: output.duration_in_millis high, queue.events_count growing

3. Tune the bottleneck (see below)

4. Re-measure after each change
```

### pipeline.workers Tuning

```yaml
# Start: pipeline.workers = number of CPU cores
# If filter workers are CPU-bound: increase workers (up to 2× CPU cores)
# If output is the bottleneck: adding workers won't help — tune output instead
# If I/O bound (geoip, dns, http filter): increase workers significantly (4-8×)
pipeline.workers: 8
```

### pipeline.batch.size Tuning

```yaml
# Larger batch = more efficient output (fewer ES bulk requests)
# Larger batch = higher latency per event (waits longer to fill batch)
# Start at 125, increase to 500-1000 for high-throughput pipelines
pipeline.batch.size: 500
pipeline.batch.delay: 50   # ms — reduce for lower latency, increase for higher throughput
```

### Elasticsearch Output Optimization

```ruby
output {
  elasticsearch {
    # Increase bulk request size (default 125 is often too small)
    bulk_max_size => 500

    # Use multiple output workers by duplicating the output block
    # (Logstash does NOT parallelize a single output plugin block)
  }
}

# For very high throughput: use multiple ES output blocks
# Each block gets its own connection pool
output {
  elasticsearch { hosts => ["https://es-1:9200"] bulk_max_size => 500 }
  elasticsearch { hosts => ["https://es-2:9200"] bulk_max_size => 500 }
}
```

### Grok Optimization

```ruby
# BAD — DATA is greedy, backtracks heavily
grok { match => { "message" => "%{DATA:user} did %{DATA:action} at %{DATA:time}" } }

# GOOD — NOTSPACE doesn't backtrack
grok { match => { "message" => "%{NOTSPACE:user} did %{NOTSPACE:action} at %{GREEDYDATA:time}" } }

# BAD — no anchor
grok { match => { "message" => "ERROR: %{GREEDYDATA:error_msg}" } }

# GOOD — anchored at start
grok { match => { "message" => "^ERROR: %{GREEDYDATA:error_msg}" } }

# Always set timeout to prevent runaway regex
grok {
  match          => { "message" => "%{COMBINEDAPACHELOG}" }
  timeout_millis => 5000
  tag_on_timeout => ["_groktimeout"]
}

# Order patterns by frequency — most common patterns first
grok {
  match => { "message" => [
    "%{MOST_COMMON_PATTERN}",     # 80% of events match this
    "%{SECOND_PATTERN}",          # 15% of events
    "%{RARE_FALLBACK_PATTERN}"    # 5% of events
  ]}
}
```

### JVM Tuning

```bash
# /etc/logstash/jvm.options
-Xms4g          # Initial heap — set equal to Xmx
-Xmx4g          # Max heap — 4-8 GB typical; never > 50% of system RAM

# GC settings (G1GC is default and recommended)
-XX:+UseG1GC
-XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=30
-XX:MaxGCPauseMillis=200

# Environment variable alternative
export LS_HEAP_SIZE="4g"
```

### OS-Level Tuning

```bash
# /etc/security/limits.conf
logstash  -  nofile  65535
logstash  -  nproc   4096

# /etc/sysctl.conf
vm.swappiness     = 1       # Minimize swap
net.core.rmem_max = 26214400  # Increase UDP receive buffer (for high-volume UDP/syslog)
net.core.wmem_max = 26214400

# Apply sysctl without reboot
sysctl -p

# Check file descriptors in use
curl -s localhost:9600/_node/stats/process | jq '.process.open_file_descriptors'
```

### Benchmarking

```ruby
# Baseline: measure max throughput with no processing
input  { generator { count => 1000000 message => "benchmark test event" } }
filter { }
output { null { } }
# Run: logstash -f benchmark.conf
# Watch: curl localhost:9600/_node/stats/pipelines | jq '.pipelines.main.events'
```

---

<!-- section: monitoring -->
## 14. Logstash Monitoring & Observability

### Node Stats API Reference

```bash
# All stats
curl -s localhost:9600/_node/stats | jq .

# JVM heap and GC
curl -s localhost:9600/_node/stats/jvm | jq '.jvm.mem, .jvm.gc'

# Pipeline event counters (most useful for throughput monitoring)
curl -s localhost:9600/_node/stats/pipelines | jq '.pipelines.main.events'
# → { "in": 1234567, "filtered": 1234567, "out": 1234560, "duration_in_millis": 98765,
#     "queue_push_duration_in_millis": 1234 }

# Per-plugin duration (find slow filters)
curl -s localhost:9600/_node/stats/pipelines | jq '.pipelines.main.plugins.filters'

# Queue stats
curl -s localhost:9600/_node/stats/pipelines | jq '.pipelines.main.queue'

# Process stats (CPU, FDs)
curl -s localhost:9600/_node/stats/process

# Hot threads (identify CPU hogs)
curl -s localhost:9600/_node/hot_threads

# Node info (version, config)
curl -s localhost:9600/_node | jq '.version, .pipeline'

# Installed plugins
curl -s localhost:9600/_node/plugins | jq '.plugins[] | {name, version}'
```

### Key Metrics to Alert On

| Metric | Path | Alert Threshold |
|--------|------|-----------------|
| JVM Heap % | `jvm.mem.heap_used_percent` | > 75% sustained |
| GC duration | `jvm.gc.collectors.old.collection_time_in_millis` | Rate increase > 20% |
| Pipeline events/sec | `pipelines.X.events.out` per second | Drop > 20% from baseline |
| Filter duration | `pipelines.X.plugins.filters[n].events.duration_in_millis` / `events.out` | > 100ms avg |
| Queue depth | `pipelines.X.queue.events_count` | Sustained growth |
| DLQ size | `pipelines.X.dead_letter_queue.queue_size_in_bytes` | Any growth (means events failing) |
| CPU % | `process.cpu.percent` | > 80% sustained |
| Open FDs | `process.open_file_descriptors` | > 80% of `max_file_descriptors` |

### Enable Metricbeat Monitoring (Recommended over internal)

```yaml
# metricbeat.yml
metricbeat.modules:
  - module: logstash
    metricsets: ["node", "node_stats"]
    period: 10s
    hosts: ["http://localhost:9600"]
    xpack.enabled: true

output.elasticsearch:
  hosts: ["https://es-node:9200"]
  username: "beats_system"
  password: "${BEATS_SYSTEM_PASSWORD}"
```

---

<!-- section: security -->
## 15. Security Configuration

### Beats Input with Mutual TLS

```bash
# Generate Logstash server certificate using ES certutil
./bin/elasticsearch-certutil cert \
  --ca /etc/elasticsearch/certs/elastic-ca.p12 \
  --name logstash-server \
  --out /etc/logstash/certs/logstash.p12 \
  --pass ""

# Convert to PEM for Logstash (Logstash requires PEM, not PKCS12)
openssl pkcs12 -in /etc/logstash/certs/logstash.p12 -nocerts -nodes -out /etc/logstash/certs/logstash.key
openssl pkcs12 -in /etc/logstash/certs/logstash.p12 -clcerts -nokeys -out /etc/logstash/certs/logstash.crt
openssl pkcs12 -in /etc/logstash/certs/logstash.p12 -cacerts -nokeys -out /etc/logstash/certs/ca.crt
```

```ruby
# Beats input with mutual TLS
input {
  beats {
    port                         => 5044
    ssl                          => true
    ssl_certificate              => "/etc/logstash/certs/logstash.crt"
    ssl_key                      => "/etc/logstash/certs/logstash.key"
    ssl_certificate_authorities  => ["/etc/logstash/certs/ca.crt"]
    ssl_verify_mode              => "force_peer"   # Require client cert
    ssl_supported_protocols      => ["TLSv1.2", "TLSv1.3"]
    ssl_cipher_suites            => [
      "TLS_AES_256_GCM_SHA384",
      "TLS_AES_128_GCM_SHA256",
      "TLS_CHACHA20_POLY1305_SHA256"
    ]
  }
}
```

### Elasticsearch Output with TLS + API Key

```ruby
output {
  elasticsearch {
    hosts                     => ["https://es-node:9200"]
    api_key                   => "${ES_API_KEY}"     # id:api_key format, base64-encoded
    ssl                       => true
    ssl_certificate_verification => true
    cacert                    => "/etc/logstash/certs/ca.crt"
  }
}
```

```bash
# Create Logstash writer role in Elasticsearch
curl -X PUT https://es:9200/_security/role/logstash_writer \
  -H "Content-Type: application/json" -u elastic:${ELASTIC_PASSWORD} \
  -d '{
    "cluster": ["monitor", "manage_index_templates", "manage_ilm", "manage_pipeline"],
    "indices": [{
      "names": ["logs-*", "metrics-*", "traces-*", ".ds-*"],
      "privileges": ["write", "create_index", "manage", "auto_configure"]
    }]
  }'

# Create API key for Logstash
curl -X POST https://es:9200/_security/api_key \
  -H "Content-Type: application/json" -u elastic:${ELASTIC_PASSWORD} \
  -d '{ "name": "logstash-writer-key", "role_descriptors": { "logstash_writer": { "cluster": ["monitor"], "index": [{ "names": ["logs-*"], "privileges": ["write","create_index"] }] } } }'
```

---

<!-- section: pipeline-patterns -->
## 18. Common Pipeline Patterns & Recipes

### Pattern 1: Production Nginx Access Log Pipeline (ECS)

```ruby
# /etc/logstash/pipelines/nginx-pipeline.conf
input {
  beats {
    port => 5044
    ssl  => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key         => "/etc/logstash/certs/logstash.key"
    ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
    ssl_verify_mode => "force_peer"
  }
}

filter {
  # Drop health check and monitoring requests early
  if [url][path] in ["/health", "/ping", "/metrics"] {
    drop {}
  }

  grok {
    match => { "message" => [
      # Combined Log Format
      "%{IPORHOST:[source][ip]} - %{DATA:[user][name]} \[%{HTTPDATE:[event][created]}\] \"%{WORD:[http][request][method]} %{URIPATHPARAM:[url][original]} HTTP/%{NUMBER:[http][version]}\" %{NUMBER:[http][response][status_code]:int} (?:%{NUMBER:[http][response][body][bytes]:int}|-) \"%{DATA:[http][request][referrer]}\" \"%{DATA:[user_agent][original]}\"",
      # Error log fallback
      "%{DATESTAMP:[event][created]} \[%{LOGLEVEL:[log][level]}\] %{GREEDYDATA:[error][message]}"
    ]}
    tag_on_failure => ["_grokparsefailure"]
    timeout_millis => 5000
  }

  if "_grokparsefailure" not in [tags] {
    date {
      match    => ["[event][created]", "dd/MMM/yyyy:HH:mm:ss Z"]
      target   => "@timestamp"
      remove_field => ["[event][created]"]
    }

    # Parse query string from URL
    if [url][original] =~ /\?/ {
      ruby {
        code => '
          url = event.get("[url][original]")
          if url && url.include?("?")
            parts = url.split("?", 2)
            event.set("[url][path]", parts[0])
            event.set("[url][query]", parts[1])
          else
            event.set("[url][path]", url)
          end
        '
      }
    }

    geoip {
      source            => "[source][ip]"
      target            => "[source][geo]"
      tag_on_failure    => ["_geoip_failure"]
      ecs_compatibility => "v8"
    }

    useragent {
      source            => "[user_agent][original]"
      target            => "[user_agent]"
      ecs_compatibility => "v8"
    }

    # Classify response
    if [http][response][status_code] >= 500 {
      mutate { add_field => { "[event][outcome]" => "failure" } }
    } else if [http][response][status_code] >= 400 {
      mutate { add_field => { "[event][outcome]" => "failure" } }
    } else {
      mutate { add_field => { "[event][outcome]" => "success" } }
    }

    mutate {
      add_field => {
        "[event][dataset]"  => "nginx.access"
        "[event][module]"   => "nginx"
        "[event][kind]"     => "event"
        "[event][category]" => "web"
        "[event][type]"     => "access"
        "[ecs][version]"    => "8.11.0"
      }
      remove_field => ["message", "agent", "input", "log", "prospector", "beat"]
    }
  }
}

output {
  if "_grokparsefailure" in [tags] {
    elasticsearch {
      hosts    => ["https://es:9200"]
      api_key  => "${ES_API_KEY}"
      ssl      => true
      cacert   => "/etc/logstash/certs/ca.crt"
      index    => "parse-failures-%{+YYYY.MM.dd}"
    }
  } else {
    elasticsearch {
      hosts            => ["https://es:9200"]
      api_key          => "${ES_API_KEY}"
      ssl              => true
      cacert           => "/etc/logstash/certs/ca.crt"
      data_stream      => true
      data_stream_type      => "logs"
      data_stream_dataset   => "nginx.access"
      data_stream_namespace => "production"
      ilm_enabled      => true
      ilm_policy       => "logs-90day-policy"
      bulk_max_size    => 500
    }
  }
}
```

### Pattern 2: JDBC Incremental Sync (PostgreSQL → Elasticsearch)

```ruby
input {
  jdbc {
    jdbc_driver_library         => "/usr/share/logstash/drivers/postgresql-42.6.0.jar"
    jdbc_driver_class           => "org.postgresql.Driver"
    jdbc_connection_string      => "jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}"
    jdbc_user                   => "${DB_USER}"
    jdbc_password               => "${DB_PASSWORD}"
    jdbc_validate_connection    => true
    use_column_value            => true
    tracking_column             => "updated_at"
    tracking_column_type        => "timestamp"
    last_run_metadata_path      => "/var/lib/logstash/.jdbc_orders_last_run"
    record_last_run             => true
    statement                   => "
      SELECT
        o.id,
        o.order_number,
        o.status,
        o.total_amount,
        o.customer_id,
        c.email   AS customer_email,
        c.country AS customer_country,
        o.created_at,
        o.updated_at
      FROM orders o
      JOIN customers c ON c.id = o.customer_id
      WHERE o.updated_at > :sql_last_value
      ORDER BY o.updated_at ASC
    "
    schedule                    => "*/2 * * * *"   # Every 2 minutes
    jdbc_paging_enabled         => true
    jdbc_page_size              => 5000
    lowercase_column_names      => true
    type                        => "order"
  }
}

filter {
  date {
    match        => ["updated_at", "ISO8601"]
    target       => "@timestamp"
    remove_field => ["updated_at"]
  }

  mutate {
    rename => {
      "id"               => "[order][id]"
      "order_number"     => "[order][number]"
      "status"           => "[order][status]"
      "total_amount"     => "[order][amount]"
      "customer_id"      => "[customer][id]"
      "customer_email"   => "[customer][email]"
      "customer_country" => "[customer][country]"
    }
    convert => { "[order][amount]" => "float" }
    add_field => {
      "[event][dataset]" => "orders"
      "[@metadata][doc_id]" => "%{[order][id]}"
    }
    remove_field => ["@version", "type"]
  }
}

output {
  elasticsearch {
    hosts       => ["https://es:9200"]
    api_key     => "${ES_API_KEY}"
    ssl         => true
    cacert      => "/etc/logstash/certs/ca.crt"
    index       => "orders"
    document_id => "%{[@metadata][doc_id]}"
    action      => "index"   # Upsert by document ID
    bulk_max_size => 1000
  }
}
```

### Pattern 3: Kafka High-Throughput Consumer Pipeline

```ruby
input {
  kafka {
    bootstrap_servers  => "${KAFKA_BROKERS}"
    topics             => ["app-events"]
    group_id           => "logstash-app-events-v1"
    consumer_threads   => 6        # Match Kafka partition count
    auto_offset_reset  => "latest"
    codec              => json
    decorate_events    => extended
    fetch_max_bytes    => "52428800"
    max_poll_records   => 1000
    security_protocol  => "SASL_SSL"
    sasl_mechanism     => "SCRAM-SHA-256"
    sasl_jaas_config   => "org.apache.kafka.common.security.scram.ScramLoginModule required username='${KAFKA_USER}' password='${KAFKA_PASSWORD}';"
    ssl_truststore_location => "/etc/logstash/certs/kafka.truststore.jks"
    ssl_truststore_password => "${KAFKA_TRUSTSTORE_PASS}"
  }
}

filter {
  # Minimal processing for hot path
  date {
    match  => ["event_time", "UNIX_MS"]
    target => "@timestamp"
    remove_field => ["event_time"]
  }

  fingerprint {
    source              => ["[@metadata][kafka][topic]", "[@metadata][kafka][partition]", "[@metadata][kafka][offset]"]
    target              => "[@metadata][fingerprint]"
    method              => "MURMUR3"
    concatenate_sources => true
  }

  mutate {
    add_field => {
      "[@metadata][index]" => "events-%{[app_name]}-%{+YYYY.MM.dd}"
    }
    remove_field => ["@version"]
  }
}

output {
  elasticsearch {
    hosts         => ["https://es-1:9200", "https://es-2:9200", "https://es-3:9200"]
    api_key       => "${ES_API_KEY}"
    ssl           => true
    cacert        => "/etc/logstash/certs/ca.crt"
    index         => "%{[@metadata][index]}"
    document_id   => "%{[@metadata][fingerprint]}"
    bulk_max_size => 1000
    timeout       => 120
  }
}
```

### Pattern 4: DLQ Reprocessing Pipeline

```ruby
input {
  dead_letter_queue {
    path            => "/var/lib/logstash/dead_letter_queue"
    pipeline_id     => "main"
    commit_offsets  => true
    age_retention   => "30d"
  }
}

filter {
  # Log the DLQ reason for analysis
  mutate {
    add_field => {
      "[@metadata][dlq_reason]"     => "%{[@metadata][dead_letter_queue][reason]}"
      "[@metadata][dlq_entry_time]" => "%{[@metadata][dead_letter_queue][entry_time]}"
    }
  }

  # Fix: type mismatch on status_code field
  if [@metadata][dlq_reason] =~ /mapper_parsing_exception/ and [@metadata][dlq_reason] =~ /status_code/ {
    mutate {
      convert => { "[http][response][status_code]" => "integer" }
    }
  }

  # Fix: keyword too long (ES 32766 byte limit)
  if [@metadata][dlq_reason] =~ /max_bytes_length_exceeded_exception/ {
    truncate {
      fields       => ["message", "error.stack_trace"]
      length_bytes => 30000
    }
  }

  # Fix: remove unmapped object field that's causing conflicts
  if [@metadata][dlq_reason] =~ /object mapping for \[metadata\]/ {
    mutate { remove_field => ["metadata"] }
  }

  # Prevent re-routing to DLQ  
  mutate {
    add_field    => { "[@metadata][from_dlq]" => "true" }
    add_tag      => ["from_dlq"]
  }
}

output {
  if "_reprocessing_failed" not in [tags] {
    elasticsearch {
      hosts       => ["https://es:9200"]
      api_key     => "${ES_API_KEY}"
      ssl         => true
      cacert      => "/etc/logstash/certs/ca.crt"
      index       => "dlq-reprocessed-%{+YYYY.MM.dd}"
    }
  } else {
    file {
      path  => "/var/log/logstash/unrecoverable_dlq.log"
      codec => json_lines
    }
  }
}
```

### Pattern 5: Pipeline-to-Pipeline Security Fan-Out

```ruby
# ─── distributor.conf ────────────────────────────────────────────────────
input {
  beats { port => 5044; ssl => true; ssl_certificate => "..." }
}
filter {
  if [event][category] in ["authentication", "network", "process", "file"] {
    mutate { add_field => { "[@metadata][target_pipeline]" => "security-pipeline" } }
  } else if [event][module] in ["nginx", "apache"] {
    mutate { add_field => { "[@metadata][target_pipeline]" => "web-pipeline" } }
  } else {
    mutate { add_field => { "[@metadata][target_pipeline]" => "generic-pipeline" } }
  }
}
output {
  pipeline { send_to => ["%{[@metadata][target_pipeline]}"] ensure_delivery => true }
}

# ─── security-pipeline.conf ──────────────────────────────────────────────
input { pipeline { address => "security-pipeline" } }
filter {
  geoip { source => "[source][ip]" target => "[source][geo]" }
  # ECS normalization for security events
  mutate { add_field => { "[event][kind]" => "event" } }
}
output {
  elasticsearch {
    hosts => ["https://es:9200"] api_key => "${ES_API_KEY}"
    data_stream => true
    data_stream_type => "logs" data_stream_dataset => "%{[event][module]}" data_stream_namespace => "security"
  }
}
```

---

<!-- section: troubleshooting -->
## 20. Troubleshooting Guide

### DLQ Events Flowchart

```
Events going to DLQ?
│
├── Check DLQ file size: ls -lh /var/lib/logstash/dead_letter_queue/main/
├── Check Logstash logs: grep "dead_letter" /var/log/logstash/logstash-plain.log
├── Check DLQ reason via reprocessing pipeline (@metadata][dead_letter_queue][reason])
│
├── "mapper_parsing_exception"
│   └── Fix: convert field type, remove field, or update ES mapping
│
├── "max_bytes_length_exceeded_exception"
│   └── Fix: truncate filter before output
│
├── "illegal_argument_exception: text fields are not optimised for operations"
│   └── Fix: use .keyword subfield or change mapping to keyword
│
└── "document_missing_exception" (on update action)
    └── Fix: use action => "upsert" or ensure document exists first
```

### High Pipeline Latency Flowchart

```
High events.duration_in_millis / low events.out/sec?
│
├── curl localhost:9600/_node/hot_threads
│   └── If grok appears: check for backtracking → add ^ anchor, use NOTSPACE
│   └── If ruby filter: optimize code or switch to native plugin
│   └── If output (elasticsearch): check ES cluster health, increase bulk_max_size
│
├── curl localhost:9600/_node/stats/pipelines | jq '.pipelines.main.plugins.filters'
│   └── Sort by duration_in_millis / events.out → find slowest filter
│
├── Is it the geoip filter?
│   └── Normal for first run (database load). Use cache hit rate to assess.
│
├── Is it the dns filter?
│   └── DNS is synchronous and blocking. Increase workers or disable.
│
└── Is it the http/elasticsearch enrichment filter?
    └── These are network calls — cache results, reduce lookup frequency
```

### Common Errors Reference

| Error | Cause | Fix |
|-------|-------|-----|
| `_grokparsefailure` tag | Pattern didn't match | Test in Kibana Grok Debugger; use fallback catch-all pattern |
| `_dateparsefailure` tag | Date format mismatch | Verify format string; check for timezone issues |
| `_jsonparsefailure` tag | Field is not valid JSON | Use `skip_on_invalid_json => true`; check source field |
| ES 400 + DLQ | Mapping conflict | Run DLQ reprocessor; fix field types |
| ES 429 | ES overloaded | Reduce `bulk_max_size`; check ES heap |
| `Pipeline aborted due to error` | Plugin exception on startup | Check config syntax; verify plugin installed; check TLS certs |
| `SIGTERM received` → forced shutdown | `pipeline.unsafe_shutdown: false` + queue drain timeout | Either increase drain timeout or accept potential data loss |
| File input not tailing | sincedb has old inode | Delete sincedb file to reset, or set `start_position => "beginning"` |
| Kafka consumer lag | Too few consumer_threads | Increase to match Kafka partition count |
| `Could not find file...` | JARs missing for JDBC | Install correct JDBC driver JAR |

---

<!-- section: cli -->
## 21. Logstash CLI Reference

```bash
# ── Startup ───────────────────────────────────────────────────────────
logstash -f /etc/logstash/conf.d/            # Load config directory
logstash -f /etc/logstash/pipeline.conf      # Single config file
logstash -e 'input{stdin{}} output{stdout{codec=>rubydebug}}'  # Inline config
logstash -t                                  # Test config and exit (0 = ok)
logstash -f pipeline.conf -t                 # Test specific file

# ── Runtime options ──────────────────────────────────────────────────
logstash --config.reload.automatic           # Enable hot reload
logstash --config.reload.interval 3s         # Check config every 3s
logstash --log.level debug                   # Verbose logging
logstash --log.level warn                    # Less noise
logstash --path.settings /etc/logstash       # Custom settings path
logstash --path.data /var/lib/logstash       # Custom data path
logstash --path.config /etc/logstash/conf.d  # Custom config path
logstash --pipeline.workers 8               # Override workers
logstash --pipeline.batch.size 500          # Override batch size

# ── Plugin management ─────────────────────────────────────────────────
logstash-plugin list                         # List all plugins
logstash-plugin list --verbose               # With versions
logstash-plugin list 'logstash-filter-*'    # Filter by name pattern
logstash-plugin install logstash-filter-translate
logstash-plugin install --version 3.0.0 logstash-filter-grok
logstash-plugin update                       # Update all plugins
logstash-plugin update logstash-filter-grok  # Update specific plugin
logstash-plugin remove logstash-filter-sleep
logstash-plugin pack                         # Create offline pack
logstash-plugin install --local ./logstash-offline-plugins-8.12.0.zip

# ── Keystore ──────────────────────────────────────────────────────────
logstash-keystore create
logstash-keystore add ES_PASSWORD
logstash-keystore add DB_PASSWORD
logstash-keystore list
logstash-keystore remove OLD_KEY
logstash-keystore --path.settings /etc/logstash passwd   # Change master password
```

---

## Appendix A: Grok Pattern Quick Reference

| Pattern Name | Regex | Example Match |
|-------------|-------|---------------|
| `WORD` | `\b\w+\b` | `hello` |
| `NOTSPACE` | `\S+` | `any-non-space` |
| `DATA` | `.*?` | anything (non-greedy) |
| `GREEDYDATA` | `.*` | anything (greedy) |
| `INT` | `(?:[+-]?(?:[0-9]+))` | `42`, `-7` |
| `NUMBER` | `(?:%{BASE10NUM})` | `3.14`, `100` |
| `POSINT` | `\b(?:[1-9][0-9]*)\b` | `1`, `999` |
| `IP` | IPv4 regex | `192.168.1.1` |
| `IPORHOST` | IP or hostname | `10.0.0.1` or `myhost` |
| `HOSTNAME` | Hostname regex | `myhost.example.com` |
| `URIPATH` | `(?:/[^\s?#]*)*` | `/api/v1/users` |
| `URIPATHPARAM` | Path with optional query | `/path?key=val` |
| `HTTPDATE` | Apache date format | `01/Jan/2024:12:00:00 +0000` |
| `COMMONAPACHELOG` | Common Log Format | Full CLF line |
| `COMBINEDAPACHELOG` | Combined Log Format | CLF + referrer + UA |
| `LOGLEVEL` | Log level keyword | `INFO`, `ERROR`, `WARN` |
| `TIMESTAMP_ISO8601` | ISO 8601 datetime | `2024-01-01T12:00:00.000Z` |
| `SYSLOGTIMESTAMP` | Syslog date | `Jan  1 12:00:00` |
| `SYSLOGHOST` | Syslog host | hostname or IP |
| `SYSLOGPROG` | `syslog_program\[pid\]` | `sshd[1234]` |
| `SYSLOGLINE` | Full syslog line | Complete syslog entry |
| `JAVACLASS` | Java class name | `com.example.MyClass` |
| `JAVAFILE` | Java file:line | `MyClass.java:42` |
| `JAVASTACKTRACEPART` | Java stack frame | `at com.example.Main.main(Main.java:10)` |
| `JAVALOGMESSAGE` | Java log message | Log4j/Logback line |
| `WINPATH` | Windows path | `C:\Users\foo\bar.txt` |
| `UNIXPATH` | Unix path | `/var/log/app.log` |
| `MAC` | MAC address | `00:1A:2B:3C:4D:5E` |
| `UUID` | UUID | `550e8400-e29b-41d4-a716-446655440000` |
| `BASE16NUM` | Hex number | `0x1f4`, `DEADBEEF` |

---

## Appendix B: logstash.yml Production Template

```yaml
# ── Identity ──────────────────────────────────────────────────────────
node.name: "${HOSTNAME}"
path.data: /var/lib/logstash
path.logs: /var/log/logstash

# ── Pipeline Defaults ─────────────────────────────────────────────────
pipeline.workers:    4          # Override per pipeline in pipelines.yml
pipeline.batch.size: 250
pipeline.batch.delay: 50
pipeline.ecs_compatibility: v8

# ── Persistent Queue ──────────────────────────────────────────────────
queue.type:          persisted
queue.max_bytes:     2gb
queue.page_capacity: 64mb
queue.drain:         true
queue.checkpoint.acks:   1024
queue.checkpoint.writes: 1024

# ── DLQ ──────────────────────────────────────────────────────────────
dead_letter_queue.enable:          true
dead_letter_queue.max_bytes:       1gb
dead_letter_queue.storage_policy:  drop_newer
dead_letter_queue.flush_interval:  5000

# ── HTTP API ──────────────────────────────────────────────────────────
http.host: "127.0.0.1"
http.port: 9600

# ── Logging ───────────────────────────────────────────────────────────
log.level:  warn
log.format: json
log.format.json.fix_duplicate_message_fields: true

# ── Monitoring via Metricbeat (recommended) ──────────────────────────
# (Leave monitoring.enabled: false and use Metricbeat instead)
monitoring.enabled: false
```

---

## Appendix C: jvm.options Tuning Reference

```bash
# /etc/logstash/jvm.options

# ── Heap ─────────────────────────────────────────────────────────────
# Always set Xms == Xmx to prevent heap resizing
# Recommended: 4-8 GB; never exceed 50% of system RAM
-Xms4g
-Xmx4g

# ── GC (G1GC — recommended) ──────────────────────────────────────────
-XX:+UseG1GC
-XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=30
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/lib/logstash/heap_dump.hprof

# ── JVM diagnostics ──────────────────────────────────────────────────
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/var/log/logstash/gc.log
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=5
-XX:GCLogFileSize=20m

# ── JRuby ─────────────────────────────────────────────────────────────
-Djruby.compile.invokedynamic=true
-Djruby.jit.threshold=10

# ── File encoding ─────────────────────────────────────────────────────
-Dfile.encoding=UTF-8
-Djava.io.tmpdir=${HOME}
```

---

*End of Logstash Master Reference Cheatsheet*
*Covers Logstash 7.x / 8.x — verify version-specific behavior at elastic.co/docs/logstash*
