## Task 6: Data Ingestion with Python, Logstash, and Beats

**Objectives:**

- Ingest data into Elasticsearch using both Python scripts and ingestion tools.
- Transform and enrich data during ingestion.
- Compare feeding data manually with API calls versus using an ingestion pipeline.
- Understand Elasticsearch ingest pipelines (ingest nodes).
- Configure Filebeat for lightweight log shipping.

### Data Ingestion Pipeline Overview

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    INGESTION ARCHITECTURE                       │
  │                                                                 │
  │  ┌──────────┐    ┌──────────────┐    ┌──────────────────────┐   │
  │  │  Sources │    │  Ingestion   │    │   Elasticsearch      │   │
  │  │          │───▶│   Layer      │───▶│                      │   │
  │  │ CSV/JSON │    │              │    │  ┌─────────────────┐ │   │
  │  │ Log files│    │ • Python     │    │  │ Ingest Pipeline │ │   │
  │  │ Databases│    │ • Logstash   │    │  │ (optional)      │ │   │
  │  │ APIs     │    │ • Filebeat   │    │  └───────┬─────────┘ │   │
  │  │ Kafka    │    │ • API direct │    │          │           │   │
  │  └──────────┘    └──────────────┘    │          ▼           │   │
  │                                      │  ┌────────────────┐  │   │
  │                                      │  │  Index / Shard │  │   │
  │                                      │  └────────────────┘  │   │
  │                                      └──────────────────────┘   │
  └─────────────────────────────────────────────────────────────────┘

  Data Flow Options:
  ─────────────────
  1. Python Script ──────────────────────────▶ Elasticsearch
  2. Logstash ──── filter/transform ─────────▶ Elasticsearch
  3. Filebeat ──── lightweight ship ──┬──────▶ Elasticsearch
                                      └──────▶ Logstash (optional)
  4. API + Ingest Pipeline ─── enrich ───────▶ Elasticsearch
```

### Prerequisite

Before you begin, make sure both containers are running. If you already created them previously, you can start them instead of creating new ones:

```bash
docker start elasticsearch
docker start kibana
```

If the containers do not exist yet, create and run **Elasticsearch**:

```bash
docker run -d \
  --name elasticsearch \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  docker.elastic.co/elasticsearch/elasticsearch:8.6.0
```

Then create and run **Kibana**:

```bash
docker run -d \
  --name kibana \
  -p 5601:5601 \
  --link elasticsearch:elasticsearch \
  docker.elastic.co/kibana/kibana:8.6.0
```

Verify that both services are accessible:

* **Elasticsearch:** `http://localhost:9200`
* **Kibana:** `http://localhost:5601`

You can confirm Elasticsearch is running by opening `http://localhost:9200` in your browser or by running:

```bash
curl http://localhost:9200
```

### Task 1: Sample Data Preparation

Create a sample CSV file `products.csv`:

```csv
name,price,category,in_stock,description
Wireless Mouse,29.99,Electronics,true,Ergonomic wireless mouse with USB receiver
Mechanical Keyboard,89.50,Electronics,true,RGB mechanical keyboard with Cherry MX switches
Standing Desk,349.00,Furniture,false,Adjustable height standing desk 48 inch
USB-C Hub,45.00,Electronics,true,7-port USB-C hub with HDMI output
Office Chair,199.99,Furniture,true,Mesh back ergonomic office chair
```

### Task 2: Data Ingestion via Python (Bulk API)

The single-document approach is slow for large datasets. Use the Bulk API instead:

```python
import csv
import json
import requests
from datetime import datetime

ES_URL = "http://localhost:9200"
INDEX_NAME = "products"

# Step 1: Create index with explicit mapping
mapping = {
    "mappings": {
        "properties": {
            "name":        {"type": "text", "fields": {"raw": {"type": "keyword"}}},
            "price":       {"type": "float"},
            "category":    {"type": "keyword"},
            "in_stock":    {"type": "boolean"},
            "description": {"type": "text"},
            "ingested_at": {"type": "date"}
        }
    }
}

resp = requests.put(f"{ES_URL}/{INDEX_NAME}", json=mapping)
print(f"Create index: {resp.status_code} - {resp.json().get('acknowledged', resp.text)}")

# Step 2: Read CSV and build bulk payload
def build_bulk_body(csv_path, index_name):
    lines = []
    with open(csv_path, "r") as f:
        reader = csv.DictReader(f)
        for i, row in enumerate(reader):
            # Type conversions
            row["price"] = float(row["price"])
            row["in_stock"] = row["in_stock"].lower() == "true"
            row["ingested_at"] = datetime.utcnow().isoformat()

            action = json.dumps({"index": {"_index": index_name, "_id": i + 1}})
            doc = json.dumps(row)
            lines.append(action)
            lines.append(doc)
    return "\n".join(lines) + "\n"

# Step 3: Send bulk request
bulk_body = build_bulk_body("products.csv", INDEX_NAME)
headers = {"Content-Type": "application/x-ndjson"}

resp = requests.post(f"{ES_URL}/_bulk", data=bulk_body, headers=headers)
result = resp.json()

# Step 4: Check for errors
if result.get("errors"):
    for item in result["items"]:
        if "error" in item.get("index", {}):
            print(f"ERROR doc {item['index']['_id']}: {item['index']['error']['reason']}")
else:
    print(f"Successfully indexed {len(result['items'])} documents in {result['took']}ms")

# Step 5: Verify count
resp = requests.get(f"{ES_URL}/{INDEX_NAME}/_count")
print(f"Total documents: {resp.json()['count']}")
```

**Expected output:**

```
Create index: 200 - True
Successfully indexed 5 documents in 42ms
Total documents: 5
```

### Task 3: Using Logstash with Advanced Filters

Create a Logstash pipeline file `pipeline.conf` with multiple filters:

```conf
input {
  file {
    path => "/var/log/sample/access.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

filter {
  # Parse Apache/Nginx combined log format
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }

  # Parse the timestamp into a proper date
  date {
    match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
    target => "@timestamp"
    remove_field => ["timestamp"]
  }

  # Convert types
  mutate {
    convert => {
      "response" => "integer"
      "bytes"    => "integer"
    }
    remove_field => ["message", "host", "@version"]
    add_field => { "pipeline" => "logstash-apache" }
  }

  # Add geographic data from client IP
  geoip {
    source => "clientip"
    target => "geo"
  }

  # Tag slow responses (>1 second)
  if [bytes] > 1000000 {
    mutate {
      add_tag => ["large_response"]
    }
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "weblogs-%{+YYYY.MM.dd}"
  }

  # Debug: print to console
  stdout {
    codec => rubydebug
  }
}
```

Run Logstash:

```bash
bin/logstash -f pipeline.conf --config.reload.automatic
```

**Expected console output (stdout rubydebug):**

```ruby
{
       "clientip" => "192.168.1.50",
        "request" => "/index.html",
       "response" => 200,
          "bytes" => 2345,
           "verb" => "GET",
            "geo" => {
        "country_name" => "United States",
           "city_name" => "San Francisco",
            "location" => { "lat" => 37.7749, "lon" => -122.4194 }
    },
       "pipeline" => "logstash-apache",
     "@timestamp" => 2024-01-15T10:23:45.000Z
}
```

### Task 4: Filebeat Configuration

Create `filebeat.yml` for lightweight log shipping:

```yaml
# ===== Filebeat Inputs =====
filebeat.inputs:
  - type: filestream
    id: apache-access
    enabled: true
    paths:
      - /var/log/apache2/access.log
      - /var/log/httpd/access_log
    fields:
      log_type: apache_access
    fields_under_root: true

  - type: filestream
    id: app-logs
    enabled: true
    paths:
      - /var/log/myapp/*.log
    fields:
      log_type: application
    fields_under_root: true
    multiline:
      pattern: '^\d{4}-\d{2}-\d{2}'
      negate: true
      match: after

# ===== Processors =====
processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - drop_fields:
      fields: ["agent.ephemeral_id", "agent.hostname", "agent.id"]

# ===== Output: Elasticsearch =====
output.elasticsearch:
  hosts: ["localhost:9200"]
  index: "filebeat-%{+yyyy.MM.dd}"
  pipeline: "filebeat-ingest"

# ===== Logging =====
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat.log
  keepfiles: 7
```

Start Filebeat:

```bash
sudo filebeat -e -c filebeat.yml
```

**Expected output:**

```
INFO  [input]      log/input.go:152   Configured paths: [/var/log/apache2/access.log]
INFO  [publisher]  pipeline/output.go:105  Connection to backoff(elasticsearch(http://localhost:9200)) established
INFO  Harvester started for file: /var/log/apache2/access.log
```

### Task 5: Elasticsearch Ingest Pipelines

Elasticsearch can transform documents at index time without Logstash:

```json
PUT /_ingest/pipeline/enrich-products
{
  "description": "Enrich product documents at ingest time",
  "processors": [
    {
      "set": {
        "field": "ingested_at",
        "value": "{{_ingest.timestamp}}"
      }
    },
    {
      "uppercase": {
        "field": "category"
      }
    },
    {
      "script": {
        "lang": "painless",
        "source": "ctx.price_tier = ctx.price > 100 ? 'premium' : 'standard'"
      }
    }
  ]
}
```

Test the pipeline:

```json
POST /_ingest/pipeline/enrich-products/_simulate
{
  "docs": [
    { "_source": { "name": "Widget", "price": 150, "category": "electronics" } }
  ]
}
```

**Expected output:**

```json
{
  "docs": [
    {
      "doc": {
        "_source": {
          "name": "Widget",
          "price": 150,
          "category": "ELECTRONICS",
          "price_tier": "premium",
          "ingested_at": "2024-01-15T10:30:00.000Z"
        }
      }
    }
  ]
}
```

Use the pipeline when indexing:

```json
POST /products/_doc?pipeline=enrich-products
{
  "name": "Laptop Stand",
  "price": 45.00,
  "category": "accessories"
}
```

### Task 6: Comparing Ingestion Methods

| Feature             | Python Script       | Logstash             | Filebeat            | Ingest Pipeline     |
|---------------------|---------------------|----------------------|---------------------|---------------------|
| **Complexity**      | Low                 | Medium               | Low                 | Low                 |
| **Transformation**  | Full (code)         | Rich (filter plugins)| Limited (processors)| Moderate (processors)|
| **Throughput**      | Medium (bulk API)   | High                 | High                | High                |
| **Resource Usage**  | Low                 | High (JVM)           | Very Low (Go)       | None (ES internal)  |
| **Use Case**        | One-time migration  | Complex ETL          | Log shipping        | Simple enrichment   |
| **Buffering**       | Manual              | Built-in             | Built-in            | None                |
| **Backpressure**    | Manual              | Automatic            | Automatic           | Automatic           |

**When to use each:**

- **Python**: One-time data migration, custom business logic, integration with existing Python apps.
- **Logstash**: Complex parsing (grok), multiple inputs/outputs, enrichment with external lookups.
- **Filebeat**: Lightweight log collection from many servers, when you need low resource footprint.
- **Ingest Pipeline**: Simple field-level transforms, when you want to keep the pipeline inside Elasticsearch.

### Troubleshooting Tips

* When a *bulk API* request returns partial failures, checking the `items` array is helpful because each document has its own status and error details, while skipping that review can hide which documents failed; for example, one item may return `201 created` while another returns `400 mapper_parsing_exception`.
* If *Logstash* shows “connection refused,” verifying that Elasticsearch is running and confirming the `hosts` setting is correct is useful because the pipeline can reconnect once the endpoint is reachable, while omitting this check often leaves data stuck in the output stage; for example, `hosts => ["http://localhost:9200"]` will fail if Elasticsearch is stopped or listening on another port.
* When a *harvester* in Filebeat does not start, checking file permissions and confirming the configured paths exist is beneficial because Filebeat can begin reading files immediately once access is available, while missing permissions or invalid paths prevent collection entirely; for example, a log file owned by `root` may not be readable by the Filebeat service user.
* Before changing a grok pattern, using the *Grok* Debugger in Kibana is helpful because it shows whether the pattern matches sample input and which fields are extracted, while applying an untested pattern can produce repeated parse failures; for example, testing `%{IP:client}` against an Apache log line confirms whether the client IP is captured correctly.
* If an ingest pipeline reports “processor failed,” using the *_simulate* API is useful because it lets you test documents safely before applying changes to live traffic, while skipping simulation can make debugging harder after errors reach production; for example, `_ingest/pipeline/_simulate` can reveal which processor fails on a sample JSON payload.
* During *bulk ingestion*, increasing batch size, setting `"number_of_replicas": 0` temporarily, and raising `refresh_interval` is beneficial because indexing throughput usually improves when Elasticsearch performs fewer background operations, while leaving default settings can slow imports noticeably; for example, disabling replicas during a one-time migration often reduces indexing time and replicas can be restored afterward.

### Reflection

Compare results from Python ingestion versus Logstash/Filebeat pipelines:

1. Which method was easiest to set up for your use case?
2. When would you combine methods (e.g., Filebeat → Logstash → Elasticsearch)?
3. How does the ingest pipeline compare to Logstash for simple transformations?
