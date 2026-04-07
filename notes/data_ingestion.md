## Data Ingestion

Data ingestion is the process of getting documents into Elasticsearch so they become searchable and available for analytics. Whether you are indexing a single JSON document through the REST API, streaming millions of log lines per second through Logstash, or shipping metrics from thousands of hosts with Beats, Elasticsearch offers a layered ingestion architecture that scales from developer laptops to planet-scale production clusters. Understanding each layer — direct API calls, ingest pipelines, Logstash, Beats, and the unified Elastic Agent — is essential for choosing the right tool for every workload.

The following sections walk through every major ingestion path in depth, starting with the lowest-level REST APIs and building up to fully managed fleet deployments.

---

### Ingestion Architecture Overview

Before diving into individual APIs, it helps to see how the major components relate to one another. Every document ultimately arrives at an Elasticsearch index, but the path it takes determines how much transformation, buffering, and reliability you get along the way.

```
  Ingestion Architecture
  ----------------------

  +-----------+    +-----------+    +-------------+    +-----------+
  |  App /    |    | Log Files |    | Metrics /   |    | Database  |
  |  Service  |    |           |    | Sysdata     |    | (JDBC)    |
  +-----------+    +-----------+    +-------------+    +-----------+
       |                |                 |                  |
       |                |                 |                  |
       ▼                ▼                 ▼                  |
  +---------+    +------------+    +-------------+           |
  | Direct  |    | Filebeat / |    | Metricbeat /|           |
  | REST API|    | Elastic    |    | Heartbeat / |           |
  | (Bulk / |    | Agent      |    | Packetbeat  |           |
  |  Index) |    +------------+    +-------------+           |
  +---------+         |                 |                    |
       |              ▼                 ▼                    ▼
       |         +----------------------------------------+ |
       |         |            Logstash                     | |
       |         |  (input → filter → output)             | |
       |         +----------------------------------------+ |
       |              |                                     |
       ▼              ▼                                     |
  +----------------------------------------------------------+
  |              Elasticsearch Cluster                        |
  |                                                          |
  |   +--------------------------------------------------+   |
  |   |            Ingest Pipeline                        |   |
  |   |  (set, rename, grok, date, geoip, script, ...)   |   |
  |   +--------------------------------------------------+   |
  |                        |                                  |
  |                        ▼                                  |
  |   +--------------------------------------------------+   |
  |   |           Target Index / Data Stream              |   |
  |   +--------------------------------------------------+   |
  +----------------------------------------------------------+
```

Key observations:

- **Direct REST API** is the lowest-level path — your application sends JSON directly.
- **Beats / Elastic Agent** are lightweight shippers that run on the source host.
- **Logstash** is a heavyweight pipeline for complex transformations, enrichments, and fan-out.
- **Ingest Pipelines** run inside the Elasticsearch cluster itself, transforming documents at index time with zero external infrastructure.

---

### Single Document Indexing

The simplest way to get a document into Elasticsearch is the Index API. You send a single JSON document, and Elasticsearch stores it in the target index.

#### Explicit ID with PUT

When you supply the document ID yourself, use `PUT`. If the document already exists with that ID, it is replaced (version incremented).

```json
PUT /products/_doc/1001
{
  "name": "Wireless Mouse",
  "sku": "WM-2024-BLK",
  "price": 29.99,
  "in_stock": true,
  "category": "peripherals",
  "description": "Ergonomic wireless mouse with USB-C receiver",
  "created_at": "2024-06-15T10:30:00Z"
}
```

Expected response:

```json
{
  "_index": "products",
  "_id": "1001",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 0,
  "_primary_term": 1
}
```

#### Auto-generated ID with POST

When you do not care about the ID, use `POST` without an ID segment. Elasticsearch generates a unique, URL-safe Base64 ID.

```json
POST /products/_doc
{
  "name": "USB-C Hub",
  "sku": "HUB-7P-SLV",
  "price": 49.99,
  "in_stock": true,
  "category": "peripherals",
  "description": "7-port USB-C hub with HDMI and Ethernet",
  "created_at": "2024-07-01T08:00:00Z"
}
```

Expected response:

```json
{
  "_index": "products",
  "_id": "a1B2c3D4e5F6g7H8",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 1,
  "_primary_term": 1
}
```

> **Tip:** Use `PUT` with explicit IDs when your data has a natural key (e.g., database primary key, SKU). Use `POST` with auto-IDs for append-only workloads like logs or events where deduplication is handled upstream.

---

### Bulk API

Indexing documents one at a time is convenient for development but far too slow for production workloads. The Bulk API lets you send thousands of operations in a single HTTP request, dramatically reducing network round-trips and improving throughput.

#### NDJSON Format

The `_bulk` endpoint expects **Newline-Delimited JSON (NDJSON)** — alternating action lines and optional document body lines, each terminated by a newline character `\n`. There is **no enclosing JSON array**; each line is a self-contained JSON object.

```
  Bulk Request Structure
  ----------------------

  Line 1:  { "action": { "metadata" } }    ← action + routing info
  Line 2:  { "field": "value", ... }        ← document body (if needed)
  Line 3:  { "action": { "metadata" } }    ← next action
  Line 4:  { "field": "value", ... }        ← next body
  ...
  Final:   \n                                ← trailing newline required
```

#### Supported Actions

```
  Bulk Actions
  ------------

  +----------+--------------------------------------------------------------+
  | Action   | Description                                                  |
  +----------+--------------------------------------------------------------+
  | index    | Index a document (create or replace if ID exists)            |
  | create   | Index a document only if it does NOT already exist           |
  | update   | Partial update to an existing document                       |
  | delete   | Delete a document by ID (no body line follows)               |
  +----------+--------------------------------------------------------------+
```

#### Bulk Example

```json
POST /_bulk
{"index": {"_index": "products", "_id": "1002"}}
{"name": "Mechanical Keyboard", "sku": "KB-MEC-RGB", "price": 89.99, "in_stock": true, "category": "peripherals"}
{"index": {"_index": "products", "_id": "1003"}}
{"name": "Monitor Stand", "sku": "MS-ALU-001", "price": 39.99, "in_stock": false, "category": "accessories"}
{"create": {"_index": "products", "_id": "1004"}}
{"name": "Webcam HD", "sku": "WC-1080P", "price": 59.99, "in_stock": true, "category": "peripherals"}
{"update": {"_index": "products", "_id": "1001"}}
{"doc": {"price": 24.99, "in_stock": false}}
{"delete": {"_index": "products", "_id": "9999"}}
```

Expected response:

```json
{
  "took": 42,
  "errors": false,
  "items": [
    {
      "index": {
        "_index": "products",
        "_id": "1002",
        "_version": 1,
        "result": "created",
        "status": 201
      }
    },
    {
      "index": {
        "_index": "products",
        "_id": "1003",
        "_version": 1,
        "result": "created",
        "status": 201
      }
    },
    {
      "create": {
        "_index": "products",
        "_id": "1004",
        "_version": 1,
        "result": "created",
        "status": 201
      }
    },
    {
      "update": {
        "_index": "products",
        "_id": "1001",
        "_version": 2,
        "result": "updated",
        "status": 200
      }
    },
    {
      "delete": {
        "_index": "products",
        "_id": "9999",
        "_version": 1,
        "result": "not_found",
        "status": 404
      }
    }
  ]
}
```

> **Note:** The Bulk API never fails atomically. Each sub-action succeeds or fails independently. Always check `"errors": true` and inspect individual `items[].status` codes in production.

#### Performance Tips for Bulk Indexing

- **Batch size:** Start with 1,000–5,000 documents per request (5–15 MB payload). Benchmark and adjust — going too large wastes memory; going too small wastes round-trips.
- **Refresh control:** Append `?refresh=false` (the default) to avoid forcing a refresh after every bulk request. Batch your refreshes or rely on the default 1-second `refresh_interval`.
- **Parallelism:** Send bulk requests from multiple threads or processes. A good starting point is 2–4 concurrent bulk threads per data node.
- **Retry on 429:** When Elasticsearch returns `429 Too Many Requests`, back off and retry. The bulk thread pool is saturated.

---

### Ingest Pipelines

Ingest pipelines let you transform documents **inside the Elasticsearch cluster** before they are indexed. A pipeline is an ordered list of **processors**, each of which performs one transformation — parsing a timestamp, extracting fields from a log line with grok, enriching with GeoIP data, or running a Painless script.

This eliminates the need for an external transformation layer (like Logstash) for many common use cases, keeping your architecture simpler.

#### Creating a Pipeline

```json
PUT /_ingest/pipeline/web-logs-pipeline
{
  "description": "Parse web server access logs",
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": ["%{COMBINEDAPACHELOG}"]
      }
    },
    {
      "date": {
        "field": "timestamp",
        "formats": ["dd/MMM/yyyy:HH:mm:ss Z"],
        "target_field": "@timestamp"
      }
    },
    {
      "convert": {
        "field": "response",
        "type": "integer"
      }
    },
    {
      "convert": {
        "field": "bytes",
        "type": "long"
      }
    },
    {
      "geoip": {
        "field": "clientip",
        "target_field": "geo"
      }
    },
    {
      "user_agent": {
        "field": "agent",
        "target_field": "user_agent"
      }
    },
    {
      "remove": {
        "field": ["message", "agent", "timestamp"]
      }
    }
  ]
}
```

Expected response:

```json
{
  "acknowledged": true
}
```

#### Using a Pipeline at Index Time

Specify the pipeline with the `?pipeline` query parameter:

```json
PUT /web-logs/_doc/1?pipeline=web-logs-pipeline
{
  "message": "83.149.9.216 - - [17/Jun/2024:10:05:03 +0000] \"GET /index.html HTTP/1.1\" 200 12846 \"http://example.com\" \"Mozilla/5.0 (Windows NT 10.0; Win64; x64)\""
}
```

#### Common Ingest Processors

```
  Common Ingest Processors
  ------------------------

  +---------------+----------------------------------------------------------------+
  | Processor     | Description                                                    |
  +---------------+----------------------------------------------------------------+
  | set           | Sets a field to a static or template value                     |
  | rename        | Renames an existing field                                      |
  | remove        | Removes one or more fields from the document                   |
  | convert       | Converts a field value to a different type (int, long, float)  |
  | grok          | Parses unstructured text using regular expression patterns     |
  | date          | Parses date strings into a proper date field                   |
  | geoip         | Enriches documents with geographic data from IP addresses      |
  | user_agent    | Parses user-agent strings into structured fields               |
  | script        | Executes a Painless script for custom transformations          |
  | foreach       | Applies a processor to each element of an array field          |
  | dissect       | Simpler, faster alternative to grok for delimiter-based parse  |
  | lowercase     | Converts a string field to lowercase                           |
  | uppercase     | Converts a string field to uppercase                           |
  | trim          | Removes leading and trailing whitespace                        |
  | json          | Parses a JSON string field into a structured object            |
  +---------------+----------------------------------------------------------------+
```

---

### Simulating Pipelines

Before deploying a pipeline to production, you can test it with the Simulate API. This runs one or more sample documents through the pipeline and returns the transformed result without actually indexing anything.

```json
POST /_ingest/pipeline/web-logs-pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "83.149.9.216 - - [17/Jun/2024:10:05:03 +0000] \"GET /index.html HTTP/1.1\" 200 12846 \"http://example.com\" \"Mozilla/5.0 (Windows NT 10.0; Win64; x64)\""
      }
    }
  ]
}
```

Expected response:

```json
{
  "docs": [
    {
      "doc": {
        "_index": "_index",
        "_id": "_id",
        "_source": {
          "request": "/index.html",
          "geo": {
            "continent_name": "Europe",
            "country_iso_code": "DE",
            "country_name": "Germany",
            "location": { "lat": 51.2993, "lon": 9.491 }
          },
          "auth": "-",
          "ident": "-",
          "verb": "GET",
          "response": 200,
          "bytes": 12846,
          "clientip": "83.149.9.216",
          "httpversion": "1.1",
          "referrer": "\"http://example.com\"",
          "@timestamp": "2024-06-17T10:05:03.000Z",
          "user_agent": {
            "original": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
            "os": { "name": "Windows", "version": "10" },
            "name": "Other"
          }
        },
        "_ingest": {
          "timestamp": "2024-06-17T12:00:00.000Z"
        }
      }
    }
  ]
}
```

> **Tip:** You can also simulate an inline pipeline (without saving it first) by providing the `"pipeline"` definition directly in the simulate request body. This is useful for rapid prototyping.

---

### Logstash Overview

Logstash is a server-side data processing pipeline that ingests data from multiple sources simultaneously, transforms it, and sends it to one or more destinations (called "stashes"). Unlike ingest pipelines that run inside Elasticsearch, Logstash is a standalone JVM application with its own plugin ecosystem.

#### Input → Filter → Output Architecture

```
  Logstash Pipeline Architecture
  ------------------------------

  +------------------+    +------------------+    +------------------+
  |     INPUTS       |    |     FILTERS      |    |     OUTPUTS      |
  |                  |    |                  |    |                  |
  |  file            |--->|  grok            |--->|  elasticsearch   |
  |  beats           |    |  date            |    |  stdout          |
  |  kafka           |    |  mutate          |    |  kafka           |
  |  jdbc            |    |  geoip           |    |  s3              |
  |  syslog          |    |  csv             |    |  file            |
  |  http            |    |  json            |    |  http            |
  |  s3              |    |  aggregate       |    |  redis           |
  +------------------+    +------------------+    +------------------+

  Each section runs in its own thread pool.
  Filters are applied in order, top to bottom.
  Outputs can fan out to multiple destinations.
```

#### Example: CSV to Elasticsearch

The following Logstash configuration reads a CSV file of sales records, parses columns, converts types, and sends the result to Elasticsearch:

```
input {
  file {
    path => "/var/data/sales/*.csv"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

filter {
  csv {
    columns => ["order_id", "customer", "product", "quantity", "unit_price", "order_date"]
    separator => ","
  }

  mutate {
    convert => {
      "quantity"   => "integer"
      "unit_price" => "float"
    }
    remove_field => ["message", "host", "path", "@version"]
  }

  date {
    match => ["order_date", "yyyy-MM-dd"]
    target => "@timestamp"
    remove_field => ["order_date"]
  }

  ruby {
    code => 'event.set("total_price", event.get("quantity") * event.get("unit_price"))'
  }
}

output {
  elasticsearch {
    hosts => ["https://localhost:9200"]
    index => "sales-%{+YYYY.MM}"
    user => "elastic"
    password => "${ES_PASSWORD}"
  }
}
```

#### Example: Syslog Parsing

```
input {
  syslog {
    port => 5514
  }
}

filter {
  grok {
    match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:hostname} %{DATA:program}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:log_message}" }
  }

  date {
    match => ["syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss"]
  }
}

output {
  elasticsearch {
    hosts => ["https://localhost:9200"]
    index => "syslog-%{+YYYY.MM.dd}"
  }
}
```

---

### Beats Family

Beats are lightweight, single-purpose data shippers that install as agents on your servers. Each Beat is optimized for a specific type of data and ships it either directly to Elasticsearch or through Logstash for additional processing.

```
  Beats Family Overview
  ---------------------

  +---------------+------------------------------------------------------------------+
  | Beat          | Purpose                                                          |
  +---------------+------------------------------------------------------------------+
  | Filebeat      | Ships log files, container logs, and other file-based data       |
  | Metricbeat    | Collects system and service metrics (CPU, memory, disk, etc.)    |
  | Packetbeat    | Captures network traffic and protocol-level data (HTTP, DNS)     |
  | Heartbeat     | Monitors uptime by pinging services (HTTP, TCP, ICMP)            |
  | Auditbeat     | Collects audit events from the Linux audit framework             |
  | Winlogbeat    | Ships Windows Event Log entries                                  |
  +---------------+------------------------------------------------------------------+
```

#### Basic Filebeat Configuration

Filebeat is the most commonly deployed Beat. The following `filebeat.yml` ships Nginx access logs to Elasticsearch with an ingest pipeline:

```
filebeat.inputs:
  - type: filestream
    id: nginx-access
    paths:
      - /var/log/nginx/access.log
    parsers:
      - ndjson:
          keys_under_root: true
          overwrite_keys: true

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - drop_fields:
      fields: ["agent.ephemeral_id", "ecs.version"]

output.elasticsearch:
  hosts: ["https://localhost:9200"]
  username: "elastic"
  password: "${ES_PASSWORD}"
  pipeline: "web-logs-pipeline"
  index: "nginx-access-%{+yyyy.MM.dd}"

setup.ilm.enabled: true
setup.ilm.rollover_alias: "nginx-access"

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat.log
  keepfiles: 7
```

> **Tip:** Filebeat maintains a registry of file positions so it survives restarts without re-sending data. This makes it safe for long-running deployments and host reboots.

---

### Elastic Agent and Fleet

Elastic Agent is a single, unified agent that replaces the need to install and manage individual Beats on each host. It bundles the functionality of Filebeat, Metricbeat, Heartbeat, Auditbeat, and more into one binary, managed centrally through **Fleet** in Kibana.

```
  Elastic Agent Architecture
  --------------------------

  +--------------------+       +--------------------+
  |   Host A           |       |   Host B           |
  |                    |       |                    |
  |  +---------------+ |       |  +---------------+ |
  |  | Elastic Agent | |       |  | Elastic Agent | |
  |  |               | |       |  |               | |
  |  | - Logs        | |       |  | - Logs        | |
  |  | - Metrics     | |       |  | - Metrics     | |
  |  | - Security    | |       |  | - Security    | |
  |  +-------+-------+ |       |  +-------+-------+ |
  +----------|----------+       +----------|----------+
             |                             |
             ▼                             ▼
  +------------------------------------------------------+
  |                   Fleet Server                        |
  |  (Central policy management and agent enrollment)     |
  +------------------------------------------------------+
             |
             ▼
  +------------------------------------------------------+
  |              Elasticsearch Cluster                    |
  +------------------------------------------------------+
             |
             ▼
  +------------------------------------------------------+
  |                    Kibana                             |
  |  (Fleet UI, Integrations, Agent Policies)             |
  +------------------------------------------------------+
```

Key benefits of Elastic Agent over individual Beats:

- **Single binary:** Install one agent instead of multiple Beats per host.
- **Central management:** Update configurations and policies for thousands of agents from the Fleet UI in Kibana — no SSH required.
- **Integrations:** Pre-built integrations for hundreds of data sources (AWS, Kubernetes, Nginx, PostgreSQL, etc.) that auto-configure inputs, ingest pipelines, dashboards, and index templates.
- **Agent policies:** Group agents by role (e.g., "Web Servers", "Database Hosts") and apply different collection policies to each group.

---

### Ingestion Method Comparison

Choosing the right ingestion method depends on your transformation needs, operational complexity budget, and throughput requirements.

```
  Ingestion Methods Comparison
  ----------------------------

  +------------------+---------------+---------------------------+------------+-------------------+
  | Method           | Complexity    | Use Case                  | Throughput | Transformation    |
  +------------------+---------------+---------------------------+------------+-------------------+
  | Direct REST API  | Low           | Application indexing,     | Medium     | None (client-side |
  |                  |               | one-off imports           |            | only)             |
  +------------------+---------------+---------------------------+------------+-------------------+
  | Ingest Pipelines | Low–Medium    | Lightweight enrichment    | High       | Moderate (built-  |
  |                  |               | at index time (grok,      |            | in processors)    |
  |                  |               | geoip, date parsing)      |            |                   |
  +------------------+---------------+---------------------------+------------+-------------------+
  | Logstash         | Medium–High   | Complex ETL, fan-out      | High       | High (400+        |
  |                  |               | to multiple outputs,      |            | plugins, Ruby     |
  |                  |               | JDBC/Kafka integration    |            | code, aggregates) |
  +------------------+---------------+---------------------------+------------+-------------------+
  | Beats            | Low           | Shipping logs, metrics,   | High       | Low (lightweight  |
  |                  |               | network data from hosts   |            | processors only)  |
  +------------------+---------------+---------------------------+------------+-------------------+
  | Elastic Agent    | Low           | Unified data collection,  | High       | Low–Moderate      |
  |                  |               | fleet-managed hosts,      |            | (integrations +   |
  |                  |               | security use cases        |            | ingest pipelines) |
  +------------------+---------------+---------------------------+------------+-------------------+
```

---

### Update API

Elasticsearch documents are immutable. An "update" internally fetches the current document, applies your changes, re-indexes the new version, and marks the old version for deletion. The Update API provides a convenient way to perform partial updates without sending the full document.

#### Partial Update with `doc`

```json
POST /products/_update/1001
{
  "doc": {
    "price": 22.99,
    "in_stock": true,
    "tags": ["wireless", "ergonomic", "sale"]
  }
}
```

Expected response:

```json
{
  "_index": "products",
  "_id": "1001",
  "_version": 3,
  "result": "updated",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 5,
  "_primary_term": 1
}
```

#### Scripted Update with Painless

For updates that depend on the current value of a field, use a Painless script:

```json
POST /products/_update/1001
{
  "script": {
    "source": "ctx._source.price *= (1 - params.discount); ctx._source.tags.add('discounted')",
    "params": {
      "discount": 0.10
    }
  }
}
```

Expected response:

```json
{
  "_index": "products",
  "_id": "1001",
  "_version": 4,
  "result": "updated",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 6,
  "_primary_term": 1
}
```

#### Upsert

If the document may not exist yet, combine `script` with `upsert` to supply a fallback document:

```json
POST /products/_update/2001
{
  "script": {
    "source": "ctx._source.quantity += params.qty",
    "params": { "qty": 5 }
  },
  "upsert": {
    "name": "New Widget",
    "quantity": 5,
    "price": 9.99
  }
}
```

> **Tip:** When `result` is `"noop"`, it means the `doc` you provided is identical to the existing document. No write occurred, saving disk I/O. You can control this with the `"detect_noop": false` option.

---

### Update by Query

The Update by Query API applies changes to every document matching a query across one or more indices. It works like a `SELECT ... WHERE` combined with an `UPDATE` — Elasticsearch scrolls through matching documents and applies a script to each.

```json
POST /products/_update_by_query
{
  "query": {
    "term": {
      "category": "peripherals"
    }
  },
  "script": {
    "source": "ctx._source.price = Math.round(ctx._source.price * (1 - params.discount) * 100) / 100.0",
    "params": {
      "discount": 0.15
    }
  }
}
```

Expected response:

```json
{
  "took": 156,
  "timed_out": false,
  "total": 4,
  "updated": 4,
  "deleted": 0,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "retries": {
    "bulk": 0,
    "search": 0
  },
  "failures": []
}
```

> **Note:** Update-by-query takes a snapshot of the index at the start. If documents are modified concurrently, you may see `version_conflicts`. Add `"conflicts": "proceed"` to skip conflicting documents instead of aborting.

---

### Delete by Query

The Delete by Query API removes all documents matching a given query. This is the Elasticsearch equivalent of `DELETE FROM table WHERE condition`.

```json
POST /products/_delete_by_query
{
  "query": {
    "bool": {
      "must": [
        { "term": { "in_stock": false } },
        { "range": { "created_at": { "lt": "2024-01-01" } } }
      ]
    }
  }
}
```

Expected response:

```json
{
  "took": 87,
  "timed_out": false,
  "total": 12,
  "deleted": 12,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "retries": {
    "bulk": 0,
    "search": 0
  },
  "failures": []
}
```

> **Tip:** For large deletions, add `?wait_for_completion=false` to run the operation asynchronously. Elasticsearch returns a task ID you can poll with `GET /_tasks/<task_id>`.

---

### Reindex API

The Reindex API copies documents from one index to another. This is essential when you need to change mappings, apply new ingest pipelines, migrate to a different shard layout, or move data between clusters.

#### Same-Cluster Reindex

```json
POST /_reindex
{
  "source": {
    "index": "products",
    "query": {
      "term": { "category": "peripherals" }
    }
  },
  "dest": {
    "index": "products-v2"
  }
}
```

Expected response:

```json
{
  "took": 312,
  "timed_out": false,
  "total": 4,
  "updated": 0,
  "created": 4,
  "deleted": 0,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "retries": {
    "bulk": 0,
    "search": 0
  },
  "failures": []
}
```

#### Cross-Cluster Reindex

To pull data from a remote cluster, configure `reindex.remote.whitelist` in `elasticsearch.yml` and use the `remote` source:

```json
POST /_reindex
{
  "source": {
    "remote": {
      "host": "https://old-cluster:9200",
      "username": "elastic",
      "password": "secret"
    },
    "index": "legacy-products"
  },
  "dest": {
    "index": "products-v2"
  }
}
```

#### Reindex with Transformation

You can combine reindex with a script to reshape data during migration:

```json
POST /_reindex
{
  "source": {
    "index": "products"
  },
  "dest": {
    "index": "products-v2"
  },
  "script": {
    "source": "ctx._source.price_cents = (int)(ctx._source.price * 100); ctx._source.remove('price')"
  }
}
```

> **Tip:** For very large indices, add `"slices": "auto"` to parallelize the reindex across multiple threads. Monitor progress with `GET /_tasks?actions=*reindex`.

---

### Performance Tips for Data Ingestion

When indexing large volumes of data, several settings and techniques can dramatically improve throughput:

#### 1. Disable Refresh During Bulk Loading

By default, Elasticsearch refreshes every index every second, making new documents searchable. During a large import, this wastes I/O. Disable it temporarily:

```json
PUT /products/_settings
{
  "index": {
    "refresh_interval": "-1"
  }
}
```

After the import completes, restore the default and force a refresh:

```json
PUT /products/_settings
{
  "index": {
    "refresh_interval": "1s"
  }
}
```

```json
POST /products/_refresh
```

#### 2. Increase Translog Flush Threshold

The translog provides durability. Increasing its flush threshold reduces disk syncs during heavy indexing:

```json
PUT /products/_settings
{
  "index": {
    "translog.flush_threshold_size": "1gb",
    "translog.durability": "async",
    "translog.sync_interval": "30s"
  }
}
```

> **Warning:** Setting `translog.durability` to `"async"` risks losing up to `sync_interval` seconds of data on hard crash. Only use during initial bulk loads, then revert.

#### 3. Reduce Replica Count During Import

Writing to replicas doubles the indexing work. Set replicas to zero during bulk load and restore afterward:

```json
PUT /products/_settings
{
  "index": {
    "number_of_replicas": 0
  }
}
```

#### 4. Use Routing for Co-located Data

If your queries always filter by a specific field (e.g., `tenant_id`), use custom routing to place related documents on the same shard:

```json
PUT /products/_doc/1001?routing=tenant_42
{
  "name": "Wireless Mouse",
  "tenant_id": "tenant_42",
  "price": 29.99
}
```

#### 5. General Guidelines

```
  Bulk Indexing Performance Checklist
  -----------------------------------

  +-----+----------------------------------------------+-----------------------------+
  | #   | Optimization                                 | Expected Impact             |
  +-----+----------------------------------------------+-----------------------------+
  | 1   | Disable refresh (refresh_interval: -1)       | 20-30% throughput increase  |
  | 2   | Set replicas to 0 during load                | ~50% throughput increase    |
  | 3   | Increase translog flush threshold             | 10-15% throughput increase  |
  | 4   | Optimal bulk size (5-15 MB per request)       | Maximize network efficiency |
  | 5   | Use multiple bulk threads (2-4 per node)      | Linear scaling with threads |
  | 6   | Use auto-generated IDs when possible          | 15-20% faster (no lookup)   |
  | 7   | Disable swapping (bootstrap.memory_lock)      | Prevents JVM heap paging    |
  | 8   | Use SSD storage                               | 5-10x I/O improvement       |
  +-----+----------------------------------------------+-----------------------------+
```

---

### Command Reference

| Operation | REST Verb & Endpoint | Key Parameters |
|-----------|----------------------|----------------|
| Index document (explicit ID) | `PUT /<index>/_doc/<id>` | `?pipeline`, `?routing`, `?refresh` |
| Index document (auto ID) | `POST /<index>/_doc` | `?pipeline`, `?routing`, `?refresh` |
| Bulk operations | `POST /_bulk` | NDJSON body, `?pipeline`, `?refresh` |
| Partial update | `POST /<index>/_update/<id>` | `doc`, `script`, `upsert`, `detect_noop` |
| Update by query | `POST /<index>/_update_by_query` | `query`, `script`, `conflicts`, `slices` |
| Delete document | `DELETE /<index>/_doc/<id>` | `?routing`, `?refresh` |
| Delete by query | `POST /<index>/_delete_by_query` | `query`, `conflicts`, `wait_for_completion` |
| Reindex | `POST /_reindex` | `source.index`, `dest.index`, `script`, `slices` |
| Cross-cluster reindex | `POST /_reindex` | `source.remote.host`, `source.index`, `dest.index` |
| Create ingest pipeline | `PUT /_ingest/pipeline/<id>` | `description`, `processors` |
| Get ingest pipeline | `GET /_ingest/pipeline/<id>` | |
| Delete ingest pipeline | `DELETE /_ingest/pipeline/<id>` | |
| Simulate pipeline | `POST /_ingest/pipeline/<id>/_simulate` | `docs`, `pipeline` (inline) |
| Refresh index | `POST /<index>/_refresh` | |
| Update index settings | `PUT /<index>/_settings` | `refresh_interval`, `number_of_replicas`, `translog.*` |
| Get task status | `GET /_tasks/<task_id>` | Used for async update/delete/reindex |
| Cancel task | `POST /_tasks/<task_id>/_cancel` | |
