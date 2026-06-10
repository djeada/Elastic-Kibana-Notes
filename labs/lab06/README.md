# Task 6: End-to-End Data Ingestion with Python, Logstash, Filebeat, and Elasticsearch

## Objectives

By the end of this lab, you will be able to:

* Start a basic Elastic Stack lab using plain Docker containers.
* Generate sample product data with Python.
* Ingest CSV data into Elasticsearch using the Bulk API.
* Use an Elasticsearch ingest pipeline to enrich product documents at index time.
* Generate continuous Apache-style and JSON application logs at a controlled rate.
* Add Logstash only when Apache log parsing is needed.
* Ingest Apache access logs through Logstash.
* Add Filebeat only when JSON log shipping is needed.
* Ingest JSON application logs through Filebeat.
* Verify indexed data from the command line and in Kibana.
* Compare Python ingestion, Logstash, Filebeat, and Elasticsearch ingest pipelines.

## Data Ingestion Pipeline Overview

This lab is organized in stages. Each ingestion method is introduced only when it is needed.

```text
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │                          INGESTION ARCHITECTURE                               │
  │                                                                               │
  │  Product CSV ── Python Bulk API ── enrich-products pipeline ── products       │
  │                                                                               │
  │  access.log ── Logstash grok/date/mutate/geoip ────────────── weblogs-*       │
  │                                                                               │
  │  app JSON log ── Filebeat decode_json_fields ─ app-logs pipeline ─ filebeat-* │
  └───────────────────────────────────────────────────────────────────────────────┘
```

```text
  ┌──────────┐     ┌─────────────────┐     ┌──────────────────────┐
  │ Sources  │────▶│ Ingestion Layer │────▶│    Elasticsearch     │
  │          │     │                 │     │                      │
  │ CSV      │     │ Python Bulk API │     │ products             │
  │ Logs     │     │ Logstash        │     │ weblogs-logstash-*   │
  │ JSON     │     │ Filebeat        │     │ filebeat-app-*       │
  │ APIs     │     │ Ingest Pipeline │     │                      │
  └──────────┘     └─────────────────┘     └──────────────────────┘
```

## Prerequisites

In the prerequisites, only prepare the base lab environment.

You will do the following:

1. Create the project directory structure.
2. Create basic configuration files.
3. Create the Python environment.
4. Create a Docker network.
5. Start Elasticsearch.
6. Start Kibana.
7. Verify that Elasticsearch and Kibana are reachable.

### Project Structure at the Start

At the beginning of the lab, the project should look like this:

```text
ingestion-lab/
├── .env
├── requirements.txt
├── data/
│   └── logs/
│       └── app/
└── scripts/
    └── wait_for_elasticsearch.py
```

### Create the Project Folder

```bash
mkdir -p ingestion-lab/{scripts,data/logs/app}
cd ingestion-lab
```

### Create `.env`

The `.env` file stores simple version and port settings used by the Docker commands.

```bash
cat > .env <<'EOF'
ELASTIC_VERSION=8.6.0
ES_PORT=9200
KIBANA_PORT=5601
EOF
```

Load the variables into your current shell:

```bash
source .env
```

You need to run `source .env` again if you open a new terminal.

### Create `requirements.txt`

```bash
cat > requirements.txt <<'EOF'
requests>=2.31.0
python-dateutil>=2.9.0
EOF
```

### Create and Activate a Python Virtual Environment

On Linux or macOS:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

On Windows PowerShell:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

### Create a Docker Network

All lab containers use the same Docker network. This lets containers communicate using container names.

```bash
docker network inspect net >/dev/null 2>&1 || docker network create net
```

### Start Elasticsearch

If the Elasticsearch container does not exist yet, create it:

```bash
docker run -d \
  --name elasticsearch \
  --network net \
  -p ${ES_PORT}:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  -v esdata:/usr/share/elasticsearch/data \
  docker.elastic.co/elasticsearch/elasticsearch:${ELASTIC_VERSION}
```

If the container already exists from a previous run, start it instead:

```bash
docker start elasticsearch
```

### Start Kibana

If the Kibana container does not exist yet, create it:

```bash
docker run -d \
  --name kibana \
  --network net \
  -p ${KIBANA_PORT}:5601 \
  -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" \
  docker.elastic.co/kibana/kibana:${ELASTIC_VERSION}
```

If the container already exists from a previous run, start it instead:

```bash
docker start kibana
```

### Verify Elasticsearch and Kibana

Verify Elasticsearch from the command line:

```bash
curl http://localhost:9200
```

Open these URLs in your browser:

```text
Elasticsearch: http://localhost:9200
Kibana:        http://localhost:5601
```

Check running containers:

```bash
docker ps
```

At this point, you should only see these lab containers:

```text
elasticsearch
kibana
```

## Task 1: Sample Data Preparation

The original small CSV example is useful for explaining the file format, but an end-to-end ingestion lab should generate data automatically.

In this task, you will create a Python script that generates repeatable product data.

The output file will be:

```text
data/products.csv
```

### Manual CSV Example

A product CSV file has this structure:

```csv
sku,name,price,category,in_stock,description,rating
SKU-00001,Wireless Mouse 1,29.99,Electronics,true,Ergonomic wireless mouse with USB receiver,4.4
SKU-00002,Mechanical Keyboard 2,89.50,Electronics,true,RGB mechanical keyboard with tactile switches,4.7
SKU-00003,Standing Desk 3,349.00,Furniture,false,Adjustable height standing desk,4.2
SKU-00004,USB-C Hub 4,45.00,Electronics,true,7-port USB-C hub with HDMI output,4.1
SKU-00005,Office Chair 5,199.99,Furniture,true,Mesh back ergonomic office chair,4.6
```

Instead of manually writing the file, this lab generates it with Python.

### Create `scripts/prepare_products.py`

```python
#!/usr/bin/env python3
"""Generate sample product data for the Python bulk ingestion part of the lab."""
import argparse
import csv
import random
from pathlib import Path


PRODUCT_SEEDS = [
    ("Wireless Mouse", "Electronics", "Ergonomic wireless mouse with USB receiver"),
    ("Mechanical Keyboard", "Electronics", "RGB mechanical keyboard with tactile switches"),
    ("Standing Desk", "Furniture", "Adjustable height standing desk"),
    ("USB-C Hub", "Electronics", "7-port USB-C hub with HDMI output"),
    ("Office Chair", "Furniture", "Mesh back ergonomic office chair"),
    ("Laptop Stand", "Accessories", "Foldable aluminum laptop stand"),
    ("Noise Cancelling Headphones", "Electronics", "Bluetooth headphones with active noise cancellation"),
    ("Desk Lamp", "Furniture", "LED desk lamp with dimmer"),
    ("Webcam", "Electronics", "1080p webcam with built-in microphone"),
    ("Notebook Pack", "Stationery", "Pack of lined notebooks"),
]


def generate_products(rows: int, output: Path, seed: int) -> None:
    random.seed(seed)
    output.parent.mkdir(parents=True, exist_ok=True)

    with output.open("w", newline="", encoding="utf-8") as file:
        writer = csv.DictWriter(
            file,
            fieldnames=[
                "sku",
                "name",
                "price",
                "category",
                "in_stock",
                "description",
                "rating",
            ],
        )

        writer.writeheader()

        for i in range(1, rows + 1):
            base_name, category, description = random.choice(PRODUCT_SEEDS)
            price = round(random.uniform(5, 600), 2)

            writer.writerow(
                {
                    "sku": f"SKU-{i:05d}",
                    "name": f"{base_name} {i}",
                    "price": price,
                    "category": category,
                    "in_stock": random.choice(["true", "true", "true", "false"]),
                    "description": description,
                    "rating": round(random.uniform(3.0, 5.0), 1),
                }
            )

    print(f"Wrote {rows} product rows to {output}")


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--rows", type=int, default=100, help="number of product rows to generate")
    parser.add_argument("--output", type=Path, default=Path("data/products.csv"))
    parser.add_argument("--seed", type=int, default=7)
    args = parser.parse_args()

    generate_products(args.rows, args.output, args.seed)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

### Run the Product Data Generator

```bash
python3 scripts/prepare_products.py --rows 100 --output data/products.csv
```

Expected output:

```text
Wrote 100 product rows to data/products.csv
```

Preview the generated file:

```bash
head data/products.csv
```

Expected structure:

```csv
sku,name,price,category,in_stock,description,rating
SKU-00001,Office Chair 1,569.15,Furniture,true,Mesh back ergonomic office chair,4.3
SKU-00002,Wireless Mouse 2,493.26,Electronics,true,Ergonomic wireless mouse with USB receiver,4.2
```

### Task 2: Data Ingestion via Python Bulk API

Indexing documents one at a time is slow for larger datasets.

In this task, you will use:

1. An Elasticsearch index with explicit mappings.
2. An Elasticsearch ingest pipeline.
3. The Bulk API from Python.

The Python script will read `data/products.csv`, convert values to correct types, send batches to Elasticsearch, and report the final document count.

### What the Product Pipeline Does

The product ingest pipeline enriches product documents at index time.

It will:

* Add `ingested_at`.
* Convert `category` to uppercase.
* Add `price_tier`.
* Add `available_label`.

Example:

```json
{
  "category": "Electronics",
  "price": 150,
  "in_stock": true
}
```

Becomes:

```json
{
  "category": "ELECTRONICS",
  "price": 150,
  "in_stock": true,
  "price_tier": "premium",
  "available_label": "available",
  "ingested_at": "2026-06-10T12:00:00.000Z"
}
```

### Create `scripts/setup_product_assets.py`

This script creates only the assets needed for product ingestion:

* `products` index
* `enrich-products` ingest pipeline

```python
#!/usr/bin/env python3
"""Create Elasticsearch assets for product ingestion."""
import argparse
import sys

import requests


def put_json(session: requests.Session, url: str, body: dict) -> None:
    response = session.put(url, json=body, timeout=20)
    if not response.ok:
        raise RuntimeError(f"PUT {url} failed: HTTP {response.status_code} {response.text}")

    print(f"OK PUT {url}")


def delete_index_if_requested(
    session: requests.Session,
    es_url: str,
    index: str,
    reset: bool,
) -> None:
    if not reset:
        return

    response = session.delete(f"{es_url}/{index}", timeout=20)

    if response.status_code in (200, 404):
        print(f"Reset index {index}: HTTP {response.status_code}")
    else:
        raise RuntimeError(
            f"DELETE {index} failed: HTTP {response.status_code} {response.text}"
        )


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--es-url", default="http://localhost:9200")
    parser.add_argument("--products-index", default="products")
    parser.add_argument("--reset", action="store_true", help="delete and recreate the products index")
    args = parser.parse_args()

    session = requests.Session()

    try:
        health = session.get(args.es_url, timeout=10)
        health.raise_for_status()

        product_pipeline = {
            "description": "Normalize and enrich product documents during ingestion",
            "processors": [
                {
                    "set": {
                        "field": "ingested_at",
                        "value": "{{_ingest.timestamp}}",
                    }
                },
                {
                    "uppercase": {
                        "field": "category",
                        "ignore_missing": True,
                    }
                },
                {
                    "script": {
                        "lang": "painless",
                        "source": (
                            "ctx.price_tier = ctx.price >= 100 ? 'premium' : 'standard'; "
                            "ctx.available_label = ctx.in_stock == true ? "
                            "'available' : 'out_of_stock';"
                        ),
                    }
                },
            ],
        }

        put_json(
            session,
            f"{args.es_url}/_ingest/pipeline/enrich-products",
            product_pipeline,
        )

        delete_index_if_requested(
            session,
            args.es_url,
            args.products_index,
            args.reset,
        )

        products_mapping = {
            "settings": {
                "number_of_shards": 1,
                "number_of_replicas": 0,
            },
            "mappings": {
                "properties": {
                    "sku": {"type": "keyword"},
                    "name": {
                        "type": "text",
                        "fields": {
                            "raw": {"type": "keyword"},
                        },
                    },
                    "price": {"type": "float"},
                    "category": {"type": "keyword"},
                    "price_tier": {"type": "keyword"},
                    "in_stock": {"type": "boolean"},
                    "available_label": {"type": "keyword"},
                    "description": {"type": "text"},
                    "rating": {"type": "float"},
                    "source_file": {"type": "keyword"},
                    "loaded_at": {"type": "date"},
                    "ingested_at": {"type": "date"},
                }
            },
        }

        response = session.put(
            f"{args.es_url}/{args.products_index}",
            json=products_mapping,
            timeout=20,
        )

        if response.status_code == 400 and "resource_already_exists_exception" in response.text:
            print(f"Index {args.products_index} already exists; use --reset to recreate it")
        elif not response.ok:
            raise RuntimeError(
                f"Create products index failed: HTTP {response.status_code} {response.text}"
            )
        else:
            print(f"OK created index {args.products_index}")

    except Exception as exc:
        print(str(exc), file=sys.stderr)
        return 1

    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

### Run Product Asset Setup

```bash
python3 scripts/setup_product_assets.py --reset
```

Expected output:

```text
OK PUT http://localhost:9200/_ingest/pipeline/enrich-products
Reset index products: HTTP 404
OK created index products
```

If the index already existed, HTTP 200 may appear instead of HTTP 404 during reset.

### Create `scripts/ingest_products_bulk.py`

```python
#!/usr/bin/env python3
"""Load products.csv into Elasticsearch using the Bulk API."""
import argparse
import csv
import json
import sys
from datetime import datetime, timezone
from pathlib import Path

import requests


def to_bool(value: str) -> bool:
    return value.strip().lower() in {"true", "1", "yes", "y"}


def build_bulk_body(
    csv_path: Path,
    index_name: str,
    pipeline: str | None,
    batch_size: int,
):
    with csv_path.open("r", encoding="utf-8") as file:
        reader = csv.DictReader(file)
        batch = []
        line_no = 0

        for line_no, row in enumerate(reader, start=1):
            try:
                row["price"] = float(row["price"])
                row["rating"] = float(row.get("rating", 0) or 0)
                row["in_stock"] = to_bool(row["in_stock"])
                row["source_file"] = csv_path.name
                row["loaded_at"] = datetime.now(timezone.utc).isoformat()
            except (KeyError, ValueError) as exc:
                raise ValueError(f"Bad CSV row {line_no}: {row} ({exc})") from exc

            action = {
                "index": {
                    "_index": index_name,
                    "_id": row.get("sku") or line_no,
                }
            }

            if pipeline:
                action["index"]["pipeline"] = pipeline

            batch.append(json.dumps(action))
            batch.append(json.dumps(row))

            if line_no % batch_size == 0:
                yield "\n".join(batch) + "\n", line_no
                batch = []

        if batch:
            yield "\n".join(batch) + "\n", line_no


def send_bulk(session: requests.Session, es_url: str, body: str) -> tuple[int, list[str]]:
    response = session.post(
        f"{es_url}/_bulk",
        data=body,
        headers={"Content-Type": "application/x-ndjson"},
        timeout=60,
    )

    if not response.ok:
        raise RuntimeError(
            f"Bulk request failed: HTTP {response.status_code} {response.text}"
        )

    result = response.json()
    errors = []

    for item in result.get("items", []):
        action = item.get("index", {})
        if "error" in action:
            errors.append(
                f"doc={action.get('_id')} "
                f"status={action.get('status')} "
                f"reason={action['error'].get('reason')}"
            )

    return len(result.get("items", [])), errors


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--es-url", default="http://localhost:9200")
    parser.add_argument("--index", default="products")
    parser.add_argument("--csv", type=Path, default=Path("data/products.csv"))
    parser.add_argument("--pipeline", default="enrich-products")
    parser.add_argument("--batch-size", type=int, default=500)
    args = parser.parse_args()

    if not args.csv.exists():
        print(
            f"CSV not found: {args.csv}. Run scripts/prepare_products.py first.",
            file=sys.stderr,
        )
        return 1

    session = requests.Session()
    total = 0
    all_errors = []

    try:
        for body, upto in build_bulk_body(
            args.csv,
            args.index,
            args.pipeline,
            args.batch_size,
        ):
            indexed, errors = send_bulk(session, args.es_url, body)
            total += indexed
            all_errors.extend(errors)
            print(f"Sent batch ending at CSV row {upto}: {indexed} actions")

        if all_errors:
            print("Bulk completed with errors:", file=sys.stderr)
            for error in all_errors[:20]:
                print(f"  {error}", file=sys.stderr)
            return 1

        session.post(f"{args.es_url}/{args.index}/_refresh", timeout=20)
        count = session.get(
            f"{args.es_url}/{args.index}/_count",
            timeout=20,
        ).json()["count"]

        print(
            f"Successfully indexed {total} product documents. "
            f"Current {args.index} count: {count}"
        )

    except Exception as exc:
        print(str(exc), file=sys.stderr)
        return 1

    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

### Run the Complete Product Ingestion Flow

```bash
python3 scripts/prepare_products.py --rows 100 --output data/products.csv

python3 scripts/setup_product_assets.py --reset

python3 scripts/ingest_products_bulk.py \
  --csv data/products.csv \
  --index products \
  --pipeline enrich-products
```

Expected output:

```text
Wrote 100 product rows to data/products.csv
OK PUT http://localhost:9200/_ingest/pipeline/enrich-products
OK created index products
Sent batch ending at CSV row 100: 100 actions
Successfully indexed 100 product documents. Current products count: 100
```

### Verify the Products Index

```bash
curl "http://localhost:9200/products/_search?pretty&size=3"
```

Look for enriched fields:

```json
{
  "sku": "SKU-00001",
  "name": "Office Chair 1",
  "price": 199.99,
  "category": "FURNITURE",
  "in_stock": true,
  "price_tier": "premium",
  "available_label": "available",
  "ingested_at": "2026-06-10T12:00:00.000Z"
}
```

Check only the count:

```bash
curl "http://localhost:9200/products/_count?pretty"
```

## Task 3: Using Logstash with Advanced Filters

Logstash is useful when logs need richer parsing, transformations, conditional logic, and enrichment before indexing.

In this task, Logstash will read Apache combined access logs from:

```text
data/logs/access.log
```

Inside the Logstash container, that file will appear as:

```text
/usr/share/logstash/sample/access.log
```

Logstash will parse the logs and write documents to:

```text
weblogs-logstash-YYYY.MM.dd
```

### Create the Logstash Folder

```bash
mkdir -p logstash/pipeline
```

The project structure now expands to:

```text
ingestion-lab/
├── logstash/
│   └── pipeline/
│       └── pipeline.conf
```

### Create `logstash/pipeline/pipeline.conf`

This pipeline:

* Reads Apache access logs.
* Parses each line with `grok`.
* Converts response code and byte fields to numbers.
* Adds a custom `pipeline` field.
* Adds tags for client errors, server errors, and large responses.
* Adds GeoIP data when possible.
* Writes to `weblogs-logstash-*`.

```conf
input {
  file {
    path => "/usr/share/logstash/sample/access.log"
    mode => "tail"
    start_position => "beginning"
    sincedb_path => "/usr/share/logstash/data/sincedb_access"
  }
}

filter {
  grok {
    ecs_compatibility => disabled
    match => { "message" => "%{COMBINEDAPACHELOG}" }
    tag_on_failure => ["_grokparsefailure_apache"]
  }

  date {
    match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
    target => "@timestamp"
    remove_field => ["timestamp"]
  }

  mutate {
    convert => {
      "response" => "integer"
      "bytes" => "integer"
    }
    add_field => {
      "pipeline" => "logstash-apache"
    }
    remove_field => ["message", "@version"]
  }

  if [response] >= 500 {
    mutate {
      add_tag => ["server_error"]
    }
  } else if [response] >= 400 {
    mutate {
      add_tag => ["client_error"]
    }
  }

  if [bytes] > 1000000 {
    mutate {
      add_tag => ["large_response"]
    }
  }

  geoip {
    ecs_compatibility => disabled
    source => "clientip"
    target => "geo"
    tag_on_failure => ["_geoip_lookup_failure"]
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "weblogs-logstash-%{+YYYY.MM.dd}"
  }

  stdout {
    codec => rubydebug
  }
}
```

### Create `scripts/setup_logstash_assets.py`

This script creates the Elasticsearch index template for Logstash output.

```python
#!/usr/bin/env python3
"""Create Elasticsearch assets needed for Logstash Apache logs."""
import argparse
import sys

import requests


def put_json(session: requests.Session, url: str, body: dict) -> None:
    response = session.put(url, json=body, timeout=20)
    if not response.ok:
        raise RuntimeError(f"PUT {url} failed: HTTP {response.status_code} {response.text}")

    print(f"OK PUT {url}")


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--es-url", default="http://localhost:9200")
    args = parser.parse_args()

    session = requests.Session()

    try:
        health = session.get(args.es_url, timeout=10)
        health.raise_for_status()

        weblogs_template = {
            "index_patterns": ["weblogs-logstash-*"],
            "template": {
                "mappings": {
                    "properties": {
                        "clientip": {"type": "ip"},
                        "ident": {"type": "keyword"},
                        "auth": {"type": "keyword"},
                        "verb": {"type": "keyword"},
                        "request": {"type": "keyword"},
                        "httpversion": {"type": "keyword"},
                        "response": {"type": "integer"},
                        "bytes": {"type": "long"},
                        "referrer": {"type": "keyword"},
                        "agent": {"type": "text"},
                        "pipeline": {"type": "keyword"},
                        "geo": {
                            "properties": {
                                "location": {"type": "geo_point"},
                                "country_name": {"type": "keyword"},
                                "city_name": {"type": "keyword"},
                            }
                        },
                    }
                }
            },
        }

        put_json(
            session,
            f"{args.es_url}/_index_template/weblogs-logstash-template",
            weblogs_template,
        )

    except Exception as exc:
        print(str(exc), file=sys.stderr)
        return 1

    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

Run it:

```bash
python3 scripts/setup_logstash_assets.py
```

Expected output:

```text
OK PUT http://localhost:9200/_index_template/weblogs-logstash-template
```

### Start the Logstash Container

Make sure the environment variables are loaded:

```bash
source .env
```

Start Logstash:

```bash
docker run -d \
  --name logstash \
  --network net \
  -e "LS_JAVA_OPTS=-Xms256m -Xmx256m" \
  -v "$PWD/logstash/pipeline:/usr/share/logstash/pipeline:ro" \
  -v "$PWD/data/logs:/usr/share/logstash/sample:ro" \
  -v logstashdata:/usr/share/logstash/data \
  docker.elastic.co/logstash/logstash:${ELASTIC_VERSION}
```

Watch Logstash startup logs:

```bash
docker logs -f logstash
```

If you edit the pipeline later, restart Logstash:

```bash
docker restart logstash
```

### Create the Shared Log Generator: `scripts/generate_logs.py`

This script can generate:

* Apache access logs for Logstash.
* JSON application logs for Filebeat.
* Both log types if needed.

The default paths are:

```text
data/logs/access.log
data/logs/app/application.log
```

```python
#!/usr/bin/env python3
"""Generate Apache access logs and JSON application logs.

Use --mode apache for Logstash.
Use --mode app for Filebeat.
Use --mode both to generate both log types.
"""
import argparse
import json
import random
import signal
import sys
import time
from datetime import datetime, timezone
from pathlib import Path


RUNNING = True

IP_POOL = [
    "8.8.8.8",
    "1.1.1.1",
    "20.42.65.84",
    "34.117.59.81",
    "52.95.110.1",
    "93.184.216.34",
    "142.250.180.14",
    "185.199.108.153",
    "151.101.1.69",
]

PATHS = [
    "/",
    "/index.html",
    "/products",
    "/products/42",
    "/cart",
    "/checkout",
    "/api/search",
    "/api/products",
    "/favicon.ico",
]

METHODS = ["GET", "GET", "GET", "POST", "PUT"]

STATUSES = [
    200,
    200,
    200,
    201,
    204,
    301,
    302,
    400,
    401,
    403,
    404,
    500,
    502,
    503,
]

USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 13_6)",
    "Mozilla/5.0 (X11; Linux x86_64)",
    "curl/8.0.1",
    "Python-requests/2.31.0",
]

SERVICES = ["catalog", "checkout", "search", "identity"]
LEVELS = ["INFO", "INFO", "INFO", "WARN", "ERROR"]


def handle_stop(signum, frame):  # noqa: ARG001
    global RUNNING
    RUNNING = False


def apache_timestamp() -> str:
    return datetime.now(timezone.utc).strftime("%d/%b/%Y:%H:%M:%S +0000")


def write_apache_line(path: Path) -> None:
    client_ip = random.choice(IP_POOL)
    method = random.choice(METHODS)
    request_path = random.choice(PATHS)
    status = random.choice(STATUSES)
    bytes_sent = random.randint(100, 2_500_000 if status < 500 else 50_000)
    referrer = random.choice(["-", "https://example.com/", "https://search.example.test/"])
    user_agent = random.choice(USER_AGENTS)

    line = (
        f'{client_ip} - - [{apache_timestamp()}] '
        f'"{method} {request_path} HTTP/1.1" '
        f'{status} {bytes_sent} "{referrer}" "{user_agent}"\n'
    )

    path.parent.mkdir(parents=True, exist_ok=True)

    with path.open("a", encoding="utf-8") as file:
        file.write(line)


def write_app_line(path: Path) -> None:
    status = random.choice(STATUSES)

    event = {
        "event_time": datetime.now(timezone.utc).isoformat(),
        "service": random.choice(SERVICES),
        "environment": "lab",
        "level": "ERROR" if status >= 500 else random.choice(LEVELS),
        "status": status,
        "latency_ms": random.randint(5, 3000),
        "message_text": random.choice(
            [
                "request completed",
                "cache lookup",
                "database query",
                "payment authorization",
            ]
        ),
        "trace_id": f"trace-{random.randint(100000, 999999)}",
    }

    path.parent.mkdir(parents=True, exist_ok=True)

    with path.open("a", encoding="utf-8") as file:
        file.write(json.dumps(event) + "\n")


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--mode",
        choices=["apache", "app", "both"],
        default="both",
        help="which log type to generate",
    )
    parser.add_argument(
        "--rate",
        type=int,
        default=10,
        help="events per minute",
    )
    parser.add_argument(
        "--duration-minutes",
        type=float,
        default=0,
        help="0 means run until Ctrl+C",
    )
    parser.add_argument(
        "--access-log",
        type=Path,
        default=Path("data/logs/access.log"),
    )
    parser.add_argument(
        "--app-log",
        type=Path,
        default=Path("data/logs/app/application.log"),
    )
    parser.add_argument("--seed", type=int, default=None)
    args = parser.parse_args()

    if args.rate <= 0:
        print("--rate must be positive", file=sys.stderr)
        return 1

    if args.seed is not None:
        random.seed(args.seed)

    signal.signal(signal.SIGINT, handle_stop)
    signal.signal(signal.SIGTERM, handle_stop)

    interval = 60.0 / args.rate
    deadline = time.time() + (args.duration_minutes * 60) if args.duration_minutes else None
    written = 0

    print(
        f"Generating mode={args.mode} at {args.rate} events per minute. "
        "Press Ctrl+C to stop."
    )

    while RUNNING:
        if deadline and time.time() >= deadline:
            break

        if args.mode in {"apache", "both"}:
            write_apache_line(args.access_log)

        if args.mode in {"app", "both"}:
            write_app_line(args.app_log)

        written += 1
        print(f"wrote event cycle #{written}", flush=True)
        time.sleep(interval)

    print(f"Stopped after writing {written} event cycles.")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

### Generate Apache Logs for Logstash

For a short classroom demo:

```bash
python3 scripts/generate_logs.py \
  --mode apache \
  --rate 10 \
  --duration-minutes 3
```

For continuous log generation:

```bash
python3 scripts/generate_logs.py --mode apache --rate 10
```

Expected generator output:

```text
Generating mode=apache at 10 events per minute. Press Ctrl+C to stop.
wrote event cycle #1
wrote event cycle #2
wrote event cycle #3
```

Example Apache line:

```text
8.8.8.8 - - [10/Jun/2026:12:05:12 +0000] "GET /products HTTP/1.1" 200 1843 "https://example.com/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 13_6)"
```

### Watch Logstash Parse Logs

```bash
docker logs -f logstash
```

You should see parsed events printed by the `rubydebug` output.

### Verify Indexed Logstash Documents

```bash
curl "http://localhost:9200/weblogs-logstash-*/_search?pretty&size=3"
```

Example parsed output fields:

```json
{
  "clientip": "8.8.8.8",
  "verb": "GET",
  "request": "/products",
  "response": 200,
  "bytes": 1843,
  "pipeline": "logstash-apache",
  "tags": []
}
```

Check the document count:

```bash
curl "http://localhost:9200/weblogs-logstash-*/_count?pretty"
```

## Task 4: Filebeat Configuration

Filebeat is a lightweight log shipper.

In this lab, Filebeat reads JSON application logs and sends them directly to Elasticsearch.

The generator writes JSON logs to:

```text
data/logs/app/application.log
```

Inside the Filebeat container, the mounted path is:

```text
/logs/app/*.log
```

Filebeat will write documents to:

```text
filebeat-app-YYYY.MM.dd
```

The documents will also pass through an Elasticsearch ingest pipeline called:

```text
app-logs-pipeline
```

### Create the Filebeat Folder

```bash
mkdir -p filebeat
```

The project structure now expands to:

```text
ingestion-lab/
├── filebeat/
│   └── filebeat.yml
```

### Create `filebeat/filebeat.yml`

This Filebeat configuration:

* Reads JSON application logs.
* Decodes JSON from the `message` field into top-level fields.
* Adds host metadata.
* Drops noisy metadata fields.
* Sends logs to `filebeat-app-YYYY.MM.dd`.
* Uses the `app-logs-pipeline` ingest pipeline.

```yaml
filebeat.inputs:
  - type: filestream
    id: app-json-logs
    enabled: true
    paths:
      - /logs/app/*.log
    fields:
      log_type: application
      pipeline: filebeat
    fields_under_root: true

processors:
  - decode_json_fields:
      fields: ["message"]
      target: ""
      overwrite_keys: true
      add_error_key: true

  - add_host_metadata: ~

  - drop_fields:
      fields:
        - agent.ephemeral_id
        - agent.id
        - ecs.version
        - input.type
      ignore_missing: true

setup.ilm.enabled: false
setup.template.enabled: true
setup.template.name: "filebeat-app"
setup.template.pattern: "filebeat-app-*"

output.elasticsearch:
  hosts: ["http://elasticsearch:9200"]
  index: "filebeat-app-%{+yyyy.MM.dd}"
  pipeline: "app-logs-pipeline"

logging.level: info
logging.to_stderr: true
```

### Create `scripts/setup_filebeat_assets.py`

This script creates:

* `app-logs-pipeline`
* `filebeat-app-*` index template

```python
#!/usr/bin/env python3
"""Create Elasticsearch assets needed for Filebeat JSON application logs."""
import argparse
import sys

import requests


def put_json(session: requests.Session, url: str, body: dict) -> None:
    response = session.put(url, json=body, timeout=20)
    if not response.ok:
        raise RuntimeError(f"PUT {url} failed: HTTP {response.status_code} {response.text}")

    print(f"OK PUT {url}")


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--es-url", default="http://localhost:9200")
    args = parser.parse_args()

    session = requests.Session()

    try:
        health = session.get(args.es_url, timeout=10)
        health.raise_for_status()

        app_logs_pipeline = {
            "description": "Enrich application JSON logs shipped by Filebeat",
            "processors": [
                {
                    "set": {
                        "field": "ingested_at",
                        "value": "{{_ingest.timestamp}}",
                    }
                },
                {
                    "script": {
                        "lang": "painless",
                        "source": (
                            "if (ctx.containsKey('status') && ctx.status >= 500) { "
                            "ctx.error_class = 'server_error'; "
                            "} else if (ctx.containsKey('status') && ctx.status >= 400) { "
                            "ctx.error_class = 'client_error'; "
                            "} else { "
                            "ctx.error_class = 'none'; "
                            "}"
                        ),
                    }
                },
            ],
        }

        put_json(
            session,
            f"{args.es_url}/_ingest/pipeline/app-logs-pipeline",
            app_logs_pipeline,
        )

        app_logs_template = {
            "index_patterns": ["filebeat-app-*"],
            "template": {
                "mappings": {
                    "properties": {
                        "service": {"type": "keyword"},
                        "environment": {"type": "keyword"},
                        "level": {"type": "keyword"},
                        "status": {"type": "integer"},
                        "latency_ms": {"type": "integer"},
                        "event_time": {"type": "date"},
                        "message_text": {"type": "text"},
                        "trace_id": {"type": "keyword"},
                        "error_class": {"type": "keyword"},
                        "ingested_at": {"type": "date"},
                    }
                }
            },
        }

        put_json(
            session,
            f"{args.es_url}/_index_template/filebeat-app-template",
            app_logs_template,
        )

    except Exception as exc:
        print(str(exc), file=sys.stderr)
        return 1

    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

Run it:

```bash
python3 scripts/setup_filebeat_assets.py
```

Expected output:

```text
OK PUT http://localhost:9200/_ingest/pipeline/app-logs-pipeline
OK PUT http://localhost:9200/_index_template/filebeat-app-template
```

### Start the Filebeat Container

Make sure variables are loaded:

```bash
source .env
```

Start Filebeat:

```bash
docker run -d \
  --name filebeat \
  --network net \
  --user root \
  -v "$PWD/filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro" \
  -v "$PWD/data/logs:/logs:ro" \
  -v filebeatdata:/usr/share/filebeat/data \
  docker.elastic.co/beats/filebeat:${ELASTIC_VERSION} \
  filebeat -e --strict.perms=false
```

Watch Filebeat:

```bash
docker logs -f filebeat
```

If you edit the Filebeat configuration later, restart Filebeat:

```bash
docker restart filebeat
```

### Generate JSON Application Logs

```bash
python3 scripts/generate_logs.py \
  --mode app \
  --rate 10 \
  --duration-minutes 3
```

Expected generator output:

```text
Generating mode=app at 10 events per minute. Press Ctrl+C to stop.
wrote event cycle #1
wrote event cycle #2
wrote event cycle #3
```

Example generated JSON log:

```json
{
  "event_time": "2026-06-10T12:06:30.000000+00:00",
  "service": "checkout",
  "environment": "lab",
  "level": "ERROR",
  "status": 503,
  "latency_ms": 1884,
  "message_text": "payment authorization",
  "trace_id": "trace-514223"
}
```

### Verify Filebeat Documents

```bash
curl "http://localhost:9200/filebeat-app-*/_search?pretty&size=3"
```

Example enriched document field added by the Elasticsearch ingest pipeline:

```json
{
  "service": "checkout",
  "environment": "lab",
  "level": "ERROR",
  "status": 503,
  "latency_ms": 1884,
  "message_text": "payment authorization",
  "trace_id": "trace-514223",
  "error_class": "server_error",
  "ingested_at": "2026-06-10T12:06:31.120Z"
}
```

Check the count:

```bash
curl "http://localhost:9200/filebeat-app-*/_count?pretty"
```

## Task 5: Elasticsearch Ingest Pipelines

Elasticsearch ingest pipelines transform documents at index time.

This lab uses two pipelines:

| Pipeline            | Created In | Used By         | Purpose                                                                              |
| - | - |  |  |
| `enrich-products`   | Task 2     | Python Bulk API | Uppercase category, add ingest timestamp, classify price tier, classify availability |
| `app-logs-pipeline` | Task 4     | Filebeat        | Add ingest timestamp and classify HTTP status into `error_class`                     |


### Product Ingest Pipeline

The product pipeline is created by `scripts/setup_product_assets.py`.

It uses this body:

```json
{
  "description": "Normalize and enrich product documents during ingestion",
  "processors": [
    {
      "set": {
        "field": "ingested_at",
        "value": "{{_ingest.timestamp}}"
      }
    },
    {
      "uppercase": {
        "field": "category",
        "ignore_missing": true
      }
    },
    {
      "script": {
        "lang": "painless",
        "source": "ctx.price_tier = ctx.price >= 100 ? 'premium' : 'standard'; ctx.available_label = ctx.in_stock == true ? 'available' : 'out_of_stock';"
      }
    }
  ]
}
```

### Create or Update the Product Pipeline Manually

```bash
curl -X PUT "http://localhost:9200/_ingest/pipeline/enrich-products" \
  -H "Content-Type: application/json" \
  -d @- <<'JSON'
{
  "description": "Normalize and enrich product documents during ingestion",
  "processors": [
    {
      "set": {
        "field": "ingested_at",
        "value": "{{_ingest.timestamp}}"
      }
    },
    {
      "uppercase": {
        "field": "category",
        "ignore_missing": true
      }
    },
    {
      "script": {
        "lang": "painless",
        "source": "ctx.price_tier = ctx.price >= 100 ? 'premium' : 'standard'; ctx.available_label = ctx.in_stock == true ? 'available' : 'out_of_stock';"
      }
    }
  ]
}
JSON
```

### Test the Product Pipeline

```bash
curl -X POST "http://localhost:9200/_ingest/pipeline/enrich-products/_simulate?pretty" \
  -H "Content-Type: application/json" \
  -d @- <<'JSON'
{
  "docs": [
    {
      "_source": {
        "name": "Widget",
        "price": 150,
        "category": "electronics",
        "in_stock": true
      }
    }
  ]
}
JSON
```

Expected important fields:

```json
{
  "category": "ELECTRONICS",
  "price_tier": "premium",
  "available_label": "available",
  "ingested_at": "2026-06-10T12:00:00.000Z"
}
```

### Application Log Ingest Pipeline

The application log pipeline classifies status codes.

```json
{
  "description": "Enrich application JSON logs shipped by Filebeat",
  "processors": [
    {
      "set": {
        "field": "ingested_at",
        "value": "{{_ingest.timestamp}}"
      }
    },
    {
      "script": {
        "lang": "painless",
        "source": "if (ctx.containsKey('status') && ctx.status >= 500) { ctx.error_class = 'server_error'; } else if (ctx.containsKey('status') && ctx.status >= 400) { ctx.error_class = 'client_error'; } else { ctx.error_class = 'none'; }"
      }
    }
  ]
}
```

### Test the Application Log Pipeline

```bash
curl -X POST "http://localhost:9200/_ingest/pipeline/app-logs-pipeline/_simulate?pretty" \
  -H "Content-Type: application/json" \
  -d @- <<'JSON'
{
  "docs": [
    {
      "_source": {
        "service": "checkout",
        "status": 503
      }
    },
    {
      "_source": {
        "service": "catalog",
        "status": 404
      }
    },
    {
      "_source": {
        "service": "search",
        "status": 200
      }
    }
  ]
}
JSON
```

Expected classifications:

```text
503 -> server_error
404 -> client_error
200 -> none
```

## Task 6: Comparing and Verifying Ingestion Methods

At this point, all three ingestion paths should be active:

1. Python Bulk API sends CSV product data to `products`.
2. Logstash parses Apache access logs into `weblogs-logstash-*`.
3. Filebeat ships JSON application logs into `filebeat-app-*`.

### Create `scripts/check_results.py`

This script prints counts and sample documents from each index pattern.

```python
#!/usr/bin/env python3
"""Print document counts and sample documents from the lab indices."""
import argparse

import requests


def count(session: requests.Session, es_url: str, pattern: str) -> int:
    response = session.get(f"{es_url}/{pattern}/_count", timeout=20)

    if response.status_code == 404:
        return 0

    response.raise_for_status()
    return response.json()["count"]


def sample(session: requests.Session, es_url: str, pattern: str) -> None:
    response = session.get(
        f"{es_url}/{pattern}/_search",
        json={
            "size": 2,
            "query": {
                "match_all": {}
            },
        },
        timeout=20,
    )

    if response.status_code == 404:
        print(f"  No index matched {pattern}")
        return

    response.raise_for_status()
    hits = response.json()["hits"]["hits"]

    for hit in hits:
        print(f"  {hit['_index']} -> {hit['_source']}")


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--es-url", default="http://localhost:9200")
    args = parser.parse_args()

    session = requests.Session()

    patterns = [
        "products",
        "weblogs-logstash-*",
        "filebeat-app-*",
    ]

    for pattern in patterns:
        try:
            print(f"\n{pattern}: {count(session, args.es_url, pattern)} documents")
            sample(session, args.es_url, pattern)
        except Exception as exc:
            print(f"\nCould not query {pattern}: {exc}")

    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

### Run the Check Script

```bash
python3 scripts/check_results.py
```

Expected style of output:

```text
products: 100 documents
  products -> {'sku': 'SKU-00001', 'category': 'FURNITURE', 'price_tier': 'premium', ...}

weblogs-logstash-*: 30 documents
  weblogs-logstash-2026.06.10 -> {'clientip': '8.8.8.8', 'response': 200, ...}

filebeat-app-*: 30 documents
  filebeat-app-2026.06.10 -> {'service': 'checkout', 'status': 503, 'error_class': 'server_error', ...}
```

### Query Counts Manually

```bash
curl "http://localhost:9200/products/_count?pretty"
curl "http://localhost:9200/weblogs-logstash-*/_count?pretty"
curl "http://localhost:9200/filebeat-app-*/_count?pretty"
```

### Compare Ingestion Methods

| Feature              | Python Script                     | Logstash                              | Filebeat                         | Ingest Pipeline                |
| -------------------- | --------------------------------- | ------------------------------------- | -------------------------------- | ------------------------------ |
| Complexity           | Low to Medium                     | Medium                                | Low                              | Low                            |
| Main Purpose         | Custom ingestion and migration    | Parsing and ETL                       | Lightweight log shipping         | Index-time enrichment          |
| Transformation Power | Full Python logic                 | Rich plugin ecosystem                 | Limited processors               | Moderate processors            |
| Resource Usage       | Low                               | Higher (JVM-based)                    | Very low                         | Runs inside Elasticsearch      |
| Throughput           | Good with Bulk API                | High                                  | High                             | High, but uses ingest node CPU |
| Buffering            | Must be implemented by the client | Built-in                              | Built-in                         | None by itself                 |
| Best Use Case        | CSV, API, or database migration   | Complex logs and multi-step pipelines | Shipping logs from many machines | Simple field enrichment        |

### When to Use Each Method

Use Python when:

* You need custom business logic.
* You are doing a one-time migration.
* You need to read data from CSV files, APIs, or databases.
* You need strong control over validation, retries, batching, and error handling.

Use Logstash when:

* Logs require heavy parsing.
* You need grok patterns.
* You need conditional transformations.
* You need enrichment before indexing.
* You need multiple inputs or outputs.
* You want a centralized processing layer.

Use Filebeat when:

* Logs are already structured.
* You need a lightweight shipper.
* You want to collect logs from many machines.
* You only need simple processors before Elasticsearch.

Use Elasticsearch ingest pipelines when:

* Transformations are simple.
* Enrichment should happen at index time.
* You want consistent transformations regardless of the sending client.
* You do not need a separate processing service.

Combine methods when needed.

For example:

```text
Filebeat -> Logstash -> Elasticsearch
```

This design is common when you want lightweight log collection on many machines but centralized parsing and enrichment in Logstash.

## Kibana Exploration

Open Kibana:

```text
http://localhost:5601
```

Create data views for:

```text
products
weblogs-logstash-*
filebeat-app-*
```

Useful fields to inspect:

| Index Pattern        | Fields                                                                     |
| -- | -- |
| `products`           | `category`, `price`, `price_tier`, `available_label`, `rating`             |
| `weblogs-logstash-*` | `clientip`, `verb`, `request`, `response`, `bytes`, `tags`, `geo.location` |
| `filebeat-app-*`     | `service`, `level`, `status`, `latency_ms`, `error_class`, `trace_id`      |

Example Kibana questions:

* How many products are `premium` versus `standard`?
* Which HTTP status codes appear most often in `weblogs-logstash-*`?
* Which services produce the most `server_error` events?
* What is the average `latency_ms` by service?
* Which web requests produce the largest responses?

## Troubleshooting Tips

- If Elasticsearch is not reachable, run `docker ps` and `docker logs task6-elasticsearch`.
- If Python ingestion fails, check the Bulk API response. A bulk request can partially succeed and still contain per-document errors.
- If the `products` index already exists with the wrong mapping, rerun `python3 scripts/setup_elastic_assets.py --reset`.
- If Logstash does not index documents, confirm that `data/logs/access.log` exists and that new lines are being appended.
- If Filebeat does not index documents, confirm that `data/logs/app/application.log` exists and restart Filebeat with `docker compose restart filebeat`.
- If a grok parse fails, copy one generated Apache line into Kibana Grok Debugger and test the `%{COMBINEDAPACHELOG}` pattern.
- If an ingest pipeline fails, use the `_simulate` API before sending live data.
- During large one-time imports, increase batch size, temporarily set replicas to `0`, and restore replica settings after loading.

## Reflection Questions

1. Which ingestion method was easiest to set up: Python, Logstash, Filebeat, or ingest pipelines?
2. Which method gave the most control over transformations?
3. Which method is best for continuously generated logs?
4. Why might a production design use `Filebeat -> Logstash -> Elasticsearch` instead of sending logs directly to Elasticsearch?
5. What transformations belong in Python code, and what transformations belong in Elasticsearch ingest pipelines?
