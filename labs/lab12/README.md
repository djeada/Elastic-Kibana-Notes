## Task 12: Performance Tuning, Snapshot Backup, and Recovery

Objectives

- Monitor cluster health and node performance with `_cat` and `_nodes` APIs.
- Tune index settings (`refresh_interval`, replicas, translog) for write/read performance.
- Optimize bulk indexing and force merge read-only indices.
- Set up snapshot repositories, create/list snapshots, and restore after data loss.
- Automate backups with SLM policies and Python monitoring scripts.
- Understand ILM for managing hot/warm/cold/delete lifecycle phases.

Backup and Recovery Workflow

```
┌──────────────┐     ┌─────────────┐     ┌──────────────┐
│ Live Cluster │────▶│  Snapshot   │────▶│  Repository  │
│  (indices)   │     │ (point-in-  │     │ (fs/S3/GCS)  │
│              │     │  time copy) │     │              │
└──────────────┘     └─────────────┘     └──────┬───────┘
                                                │ restore
┌──────────────┐     ┌─────────────┐            │
│  Recovered   │◀────│  Restore    │◀───────────┘
│   Cluster    │     │ (selective) │
└──────────────┘     └─────────────┘
```

Snapshots are Elasticsearch's built-in backup mechanism. A snapshot stores a point-in-time copy of selected indices in a registered repository. You can later restore all indices or only selected indices from that snapshot.

### Prerequisites and Lab Assumptions

This lab assumes you have Elasticsearch running locally and that you can run API requests from **Kibana Dev Tools** or with `curl`.

- Elasticsearch: `http://localhost:9200`
- Kibana: `http://localhost:5601`
- Python 3.x with `requests` installed for the monitoring script:

```bash
pip install requests
```

For the snapshot sections, Elasticsearch must be allowed to write to a filesystem repository. Add the following setting to `elasticsearch.yml` on every node and restart Elasticsearch:

```yaml
path.repo: ["/mount/backups"]
```

> **Docker note:** If Elasticsearch runs in Docker, mount a host directory into the container, for example `-v es_backups:/mount/backups`, so the snapshot repository path exists inside the container.

### 0. Preparing Lab Data

The performance, snapshot, SLM, and ILM examples below use three datasets:

| Index | Purpose |
|---|---|
| `products` | Product catalog for tuning, profiling, and restore tests |
| `articles` | Secondary index included in snapshots |
| `logs-2024.01` | Time-series style index for force merge and ILM examples |

Run this setup before the monitoring and snapshot sections so the examples return meaningful results.

#### 0a. Clean Up Any Previous Lab Indices

```http
DELETE /products
DELETE /articles
DELETE /logs-2024.01
```

If an index does not exist, Elasticsearch returns `index_not_found_exception`. That is safe to ignore during setup.

#### 0b. Create the `products` Index

```http
PUT /products
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "refresh_interval": "1s"
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "category": { "type": "keyword" },
      "brand": { "type": "keyword" },
      "price": { "type": "double" },
      "stock": { "type": "integer" },
      "rating": { "type": "double" },
      "in_stock": { "type": "boolean" },
      "created_at": { "type": "date" }
    }
  }
}
```

**Expected output:**

```json
{ "acknowledged": true, "shards_acknowledged": true, "index": "products" }
```

#### 0c. Create the `articles` Index

```http
PUT /articles
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "author": { "type": "keyword" },
      "topic": { "type": "keyword" },
      "status": { "type": "keyword" },
      "views": { "type": "integer" },
      "published_at": { "type": "date" }
    }
  }
}
```

#### 0d. Create the `logs-2024.01` Index

```http
PUT /logs-2024.01
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "service": { "type": "keyword" },
      "level": { "type": "keyword" },
      "message": { "type": "text" },
      "latency_ms": { "type": "integer" },
      "status_code": { "type": "integer" }
    }
  }
}
```

#### 0e. Bulk Index `products` Data

```http
POST /products/_bulk?refresh=true
{ "index": { "_id": "p001" } }
{ "name": "Laptop Pro 14", "category": "electronics", "brand": "Northstar", "price": 1299.99, "stock": 12, "rating": 4.8, "in_stock": true, "created_at": "2024-01-04T10:00:00Z" }
{ "index": { "_id": "p002" } }
{ "name": "Laptop Air 13", "category": "electronics", "brand": "Northstar", "price": 999.99, "stock": 8, "rating": 4.6, "in_stock": true, "created_at": "2024-01-05T11:30:00Z" }
{ "index": { "_id": "p003" } }
{ "name": "Wireless Mouse", "category": "accessories", "brand": "ClickCo", "price": 25.99, "stock": 150, "rating": 4.3, "in_stock": true, "created_at": "2024-01-06T09:45:00Z" }
{ "index": { "_id": "p004" } }
{ "name": "USB-C Hub", "category": "accessories", "brand": "PortWorks", "price": 39.99, "stock": 75, "rating": 4.1, "in_stock": true, "created_at": "2024-01-07T14:10:00Z" }
{ "index": { "_id": "p005" } }
{ "name": "Mechanical Keyboard", "category": "accessories", "brand": "KeyForge", "price": 119.99, "stock": 32, "rating": 4.7, "in_stock": true, "created_at": "2024-01-08T16:20:00Z" }
{ "index": { "_id": "p006" } }
{ "name": "Office Chair", "category": "furniture", "brand": "ErgoHome", "price": 249.00, "stock": 18, "rating": 4.4, "in_stock": true, "created_at": "2024-01-09T08:15:00Z" }
{ "index": { "_id": "p007" } }
{ "name": "Standing Desk", "category": "furniture", "brand": "ErgoHome", "price": 499.00, "stock": 6, "rating": 4.5, "in_stock": true, "created_at": "2024-01-10T13:40:00Z" }
{ "index": { "_id": "p008" } }
{ "name": "Noise Cancelling Headphones", "category": "electronics", "brand": "SoundMax", "price": 199.99, "stock": 0, "rating": 4.2, "in_stock": false, "created_at": "2024-01-11T17:05:00Z" }
{ "index": { "_id": "p009" } }
{ "name": "Portable Monitor", "category": "electronics", "brand": "ViewLite", "price": 229.99, "stock": 21, "rating": 4.0, "in_stock": true, "created_at": "2024-01-12T10:25:00Z" }
{ "index": { "_id": "p010" } }
{ "name": "Webcam HD", "category": "accessories", "brand": "ViewLite", "price": 79.99, "stock": 54, "rating": 3.9, "in_stock": true, "created_at": "2024-01-13T12:00:00Z" }
```

#### 0f. Bulk Index `articles` Data

```http
POST /articles/_bulk?refresh=true
{ "index": { "_id": "a001" } }
{ "title": "Elasticsearch Snapshot Strategy", "author": "Mina Patel", "topic": "backup", "status": "published", "views": 1840, "published_at": "2024-01-03T09:00:00Z" }
{ "index": { "_id": "a002" } }
{ "title": "How Refresh Intervals Affect Indexing", "author": "Jon Keller", "topic": "performance", "status": "published", "views": 2420, "published_at": "2024-01-06T10:30:00Z" }
{ "index": { "_id": "a003" } }
{ "title": "Force Merge Best Practices", "author": "Mina Patel", "topic": "performance", "status": "draft", "views": 680, "published_at": "2024-01-09T15:45:00Z" }
{ "index": { "_id": "a004" } }
{ "title": "ILM for Log Retention", "author": "Sam Rivera", "topic": "lifecycle", "status": "published", "views": 3100, "published_at": "2024-01-12T08:20:00Z" }
{ "index": { "_id": "a005" } }
{ "title": "Monitoring JVM Pressure", "author": "Ava Chen", "topic": "monitoring", "status": "published", "views": 1950, "published_at": "2024-01-15T12:15:00Z" }
```

#### 0g. Bulk Index `logs-2024.01` Data

```http
POST /logs-2024.01/_bulk?refresh=true
{ "index": { "_id": "l001" } }
{ "@timestamp": "2024-01-01T00:05:00Z", "service": "catalog", "level": "INFO", "message": "Product lookup completed", "latency_ms": 42, "status_code": 200 }
{ "index": { "_id": "l002" } }
{ "@timestamp": "2024-01-01T01:15:00Z", "service": "checkout", "level": "WARN", "message": "Payment gateway retry", "latency_ms": 620, "status_code": 202 }
{ "index": { "_id": "l003" } }
{ "@timestamp": "2024-01-01T02:30:00Z", "service": "search", "level": "INFO", "message": "Search query executed", "latency_ms": 85, "status_code": 200 }
{ "index": { "_id": "l004" } }
{ "@timestamp": "2024-01-02T04:45:00Z", "service": "checkout", "level": "ERROR", "message": "Payment authorization failed", "latency_ms": 1400, "status_code": 502 }
{ "index": { "_id": "l005" } }
{ "@timestamp": "2024-01-02T06:10:00Z", "service": "catalog", "level": "INFO", "message": "Inventory sync completed", "latency_ms": 110, "status_code": 200 }
{ "index": { "_id": "l006" } }
{ "@timestamp": "2024-01-03T08:25:00Z", "service": "search", "level": "WARN", "message": "Slow terms aggregation detected", "latency_ms": 980, "status_code": 200 }
{ "index": { "_id": "l007" } }
{ "@timestamp": "2024-01-04T09:40:00Z", "service": "orders", "level": "INFO", "message": "Order created", "latency_ms": 73, "status_code": 201 }
{ "index": { "_id": "l008" } }
{ "@timestamp": "2024-01-05T11:05:00Z", "service": "orders", "level": "ERROR", "message": "Order fulfillment timeout", "latency_ms": 2100, "status_code": 504 }
```

#### 0h. Verify the Dataset

```http
GET /_cat/indices/products,articles,logs-2024.01?v&s=index
```

**Expected output:**

```text
health status index        pri rep docs.count store.size pri.store.size
green  open   articles       1   0          5       ...           ...
green  open   logs-2024.01   1   0          8       ...           ...
green  open   products       1   0         10       ...           ...
```

Check a simple product aggregation:

```http
GET /products/_search
{
  "size": 0,
  "aggs": {
    "products_by_category": {
      "terms": { "field": "category", "size": 10 }
    }
  }
}
```

**Expected output (abbreviated):**

```json
{
  "aggregations": {
    "products_by_category": {
      "buckets": [
        { "key": "accessories", "doc_count": 4 },
        { "key": "electronics", "doc_count": 4 },
        { "key": "furniture", "doc_count": 2 }
      ]
    }
  }
}
```

### 1. Performance Monitoring

Monitoring gives you a baseline before you tune anything. Start with quick `_cat` APIs, then use `_nodes/stats` for deeper diagnostics.

**Cluster health:**

```http
GET /_cat/health?v
```

**Expected output:**

```text
epoch      timestamp cluster     status node.total node.data shards pri relo init unassign
1717012345 14:32:25  my-cluster  green           1         1      3   3    0    0        0
```

`green` means all primary and replica shards are assigned. `yellow` is common in a single-node lab if replicas are configured, because replicas cannot be assigned to the same node that holds the primary shard.

**Index sizes, sorted by storage size:**

```http
GET /_cat/indices?v&s=store.size:desc
```

**Expected output:**

```text
health status index        pri rep docs.count store.size pri.store.size
green  open   products       1   0         10     12.3kb        12.3kb
green  open   logs-2024.01   1   0          8      9.1kb         9.1kb
green  open   articles       1   0          5      7.4kb         7.4kb
```

**Node-level statistics:**

```http
GET /_nodes/stats/jvm,os,indices?human
```

**Expected output (abbreviated):**

```json
{
  "nodes": {
    "node-1-id": {
      "name": "es-node-1",
      "os": {
        "cpu": { "percent": 23 },
        "mem": { "used_percent": 70 }
      },
      "jvm": {
        "mem": {
          "heap_used": "412mb",
          "heap_max": "512mb",
          "heap_used_percent": 80
        },
        "gc": {
          "collectors": {
            "young": { "collection_count": 1240, "collection_time": "12.4s" },
            "old":   { "collection_count": 3,    "collection_time": "1.8s" }
          }
        }
      },
      "indices": {
        "docs": { "count": 23 },
        "search": {
          "query_total": 8430,
          "query_time_in_millis": 32100
        },
        "indexing": {
          "index_total": 23,
          "index_time_in_millis": 45000
        }
      }
    }
  }
}
```

> **Key metrics:** Watch heap usage percentage, old-generation garbage collection count, search latency, indexing latency, rejected thread-pool tasks, and unassigned shards.

### 2. Index Settings Tuning

Index settings affect how quickly Elasticsearch makes documents searchable and how much durability or redundancy it provides. For bulk loading, you can temporarily reduce overhead. After loading, restore safer defaults.

#### 2a. `refresh_interval` Adjustment

Elasticsearch refreshes an index so newly indexed documents become searchable. The default refresh interval is commonly `1s`. During heavy writes, a longer refresh interval reduces segment creation overhead.

```http
PUT /products/_settings
{
  "index": {
    "refresh_interval": "30s"
  }
}
```

**Expected output:**

```json
{ "acknowledged": true }
```

Verify the setting:

```http
GET /products/_settings/index.refresh_interval
```

**Expected output:**

```json
{
  "products": {
    "settings": {
      "index": {
        "refresh_interval": "30s"
      }
    }
  }
}
```

> Set `refresh_interval` to `"-1"` to disable refresh entirely during large bulk loads. Always restore it afterward, otherwise new data will not become searchable until a manual or automatic refresh occurs.

#### 2b. `number_of_replicas` Adjustment

Replicas improve availability and search capacity, but every replica also receives the same writes. During a large one-time bulk load, temporarily setting replicas to `0` can improve indexing throughput.

```http
PUT /products/_settings
{
  "index": {
    "number_of_replicas": 0
  }
}
```

**Expected output:**

```json
{ "acknowledged": true }
```

After the bulk load, restore the intended replica count:

```http
PUT /products/_settings
{
  "index": {
    "number_of_replicas": 1
  }
}
```

> In a single-node lab, setting replicas to `1` will usually make cluster health `yellow`, because Elasticsearch cannot place a replica shard on the same node as its primary shard.

#### 2c. Translog Settings

The translog protects recently indexed operations before they are fully committed to Lucene segments. The safest default durability mode is `request`, where Elasticsearch fsyncs after each request. During controlled bulk loads, `async` can improve throughput at the cost of possible data loss if the node crashes.

```http
PUT /products/_settings
{
  "index": {
    "translog.durability": "async",
    "translog.sync_interval": "30s",
    "translog.flush_threshold_size": "1gb"
  }
}
```

**Expected output:**

```json
{ "acknowledged": true }
```

Return to safer durability after the bulk load:

```http
PUT /products/_settings
{
  "index": {
    "translog.durability": "request"
  }
}
```

> **Warning:** Async translog can lose the most recent acknowledged writes if the node crashes before the next sync. Use it only when the source data can be replayed.

### 3. Query Performance — Profile API (Recap from Lab 08)

The Profile API explains how Elasticsearch executes a query. It is useful for identifying whether time is spent on query rewriting, building scorers, walking documents, or calculating scores.

```http
GET /products/_search
{
  "profile": true,
  "query": {
    "match": {
      "name": "laptop"
    }
  }
}
```

**Expected output (key section):**

```json
{
  "profile": {
    "shards": [
      {
        "searches": [
          {
            "query": [
              {
                "type": "TermQuery",
                "description": "name:laptop",
                "time_in_nanos": 254300,
                "breakdown": {
                  "score": 48200,
                  "create_weight": 95400,
                  "advance": 18200
                }
              }
            ]
          }
        ]
      }
    ]
  }
}
```

> Use profiling to identify whether bottlenecks are in scoring, matching, filtering, or aggregation. Do not leave `profile: true` enabled in production queries because it adds overhead and can produce large responses.

### 4. Bulk Indexing Optimization

Bulk indexing is fastest when Elasticsearch can spend less time refreshing, replicating, and syncing each request. The usual pattern is: temporarily tune for writes, bulk index, then restore normal settings.

**Step 1 — Disable refresh and replicas:**

```http
PUT /products/_settings
{
  "index": {
    "refresh_interval": "-1",
    "number_of_replicas": 0
  }
}
```

**Expected output:**

```json
{ "acknowledged": true }
```

**Step 2 — Send bulk requests:**

```http
POST /_bulk
{ "index": { "_index": "products", "_id": "p011" } }
{ "name": "Bluetooth Speaker", "price": 89.99, "category": "electronics", "brand": "SoundMax", "stock": 44, "rating": 4.2, "in_stock": true, "created_at": "2024-01-20T09:00:00Z" }
{ "index": { "_index": "products", "_id": "p012" } }
{ "name": "Laptop Stand", "price": 34.99, "category": "accessories", "brand": "ErgoHome", "stock": 61, "rating": 4.0, "in_stock": true, "created_at": "2024-01-20T09:05:00Z" }
```

**Expected output:**

```json
{
  "took": 42,
  "errors": false,
  "items": [
    { "index": { "_index": "products", "_id": "p011", "status": 201 } },
    { "index": { "_index": "products", "_id": "p012", "status": 201 } }
  ]
}
```

**Step 3 — Re-enable refresh and replicas:**

```http
PUT /products/_settings
{
  "index": {
    "refresh_interval": "1s",
    "number_of_replicas": 0,
    "translog.durability": "request"
  }
}
```

**Expected output:**

```json
{ "acknowledged": true }
```

**Step 4 — Refresh manually so the new documents are searchable immediately:**

```http
POST /products/_refresh
```

| Tip | Reason |
|-----|--------|
| Disable refresh (`"-1"`) | Avoids frequent segment creation during large writes |
| Set replicas to `0` | Prevents duplicate write work during initial loading |
| Use async translog only when replay is possible | Reduces fsync overhead but weakens durability |
| Use 5–15 MB bulk requests | Balances throughput and memory pressure |
| Watch for bulk errors | A bulk response can be HTTP 200 while individual items failed |

### 5. Force Merge for Read-Only Indices

Force merge reduces the number of Lucene segments. It is useful for old, read-only indices such as monthly log archives. It is not appropriate for indices that are still receiving writes.

```http
POST /logs-2024.01/_forcemerge?max_num_segments=1
```

**Expected output:**

```json
{
  "_shards": {
    "total": 1,
    "successful": 1,
    "failed": 0
  }
}
```

> **Important:** Never force merge indices that are still actively receiving writes. Force merge can be expensive and may create very large segments.

### 6. Snapshot Repository Setup

A snapshot repository tells Elasticsearch where backups should be stored. This example uses a filesystem repository named `my_backup`.

> **Prerequisite:** Add `path.repo: ["/mount/backups"]` to `elasticsearch.yml` on every node and restart Elasticsearch before registering the repository.

```http
PUT /_snapshot/my_backup
{
  "type": "fs",
  "settings": {
    "location": "/mount/backups/my_backup",
    "compress": true
  }
}
```

**Expected output:**

```json
{ "acknowledged": true }
```

Verify that all nodes can access the repository:

```http
POST /_snapshot/my_backup/_verify
```

**Expected output:**

```json
{
  "nodes": {
    "node-1-id": {
      "name": "es-node-1"
    }
  }
}
```

> If repository verification fails, check that the directory exists, has the correct permissions, and is listed in `path.repo` on every node.

### 7. Taking a Snapshot

Create a snapshot of the `products` and `articles` indices. This snapshot excludes global cluster state so the restore is focused on data only.

```http
PUT /_snapshot/my_backup/snapshot_lab_01?wait_for_completion=true
{
  "indices": "products,articles",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

**Expected output:**

```json
{
  "snapshot": {
    "snapshot": "snapshot_lab_01",
    "indices": ["products", "articles"],
    "state": "SUCCESS",
    "start_time": "2024-06-01T10:00:00.000Z",
    "end_time": "2024-06-01T10:00:03.450Z",
    "duration_in_millis": 3450,
    "shards": {
      "total": 2,
      "failed": 0,
      "successful": 2
    }
  }
}
```

> Snapshot names must be unique within a repository. If `snapshot_lab_01` already exists, use a new name such as `snapshot_lab_02`.

### 8. Listing Snapshots

List all snapshots in the repository:

```http
GET /_snapshot/my_backup/_all
```

**Expected output:**

```json
{
  "snapshots": [
    {
      "snapshot": "snapshot_lab_01",
      "indices": ["products", "articles"],
      "state": "SUCCESS",
      "start_time": "2024-06-01T10:00:00.000Z",
      "duration_in_millis": 3450
    }
  ]
}
```

You can also inspect one snapshot by name:

```http
GET /_snapshot/my_backup/snapshot_lab_01
```

### 9. Simulating Data Loss and Restoring from Snapshot

This section intentionally deletes the `products` index and restores it from the snapshot.

**Check the document count before deletion:**

```http
GET /products/_count
```

**Expected output:**

```json
{ "count": 12 }
```

**Delete the index:**

```http
DELETE /products
```

**Expected output:**

```json
{ "acknowledged": true }
```

Confirm deletion:

```http
GET /_cat/indices/products?v
```

**Expected output:**

```text
# empty response because no matching index exists
```

**Restore from snapshot:**

```http
POST /_snapshot/my_backup/snapshot_lab_01/_restore
{
  "indices": "products",
  "include_global_state": false
}
```

**Expected output:**

```json
{ "accepted": true }
```

**Verify restore:**

```http
GET /_cat/indices/products?v
```

**Expected output:**

```text
health status index    pri rep docs.count store.size pri.store.size
green  open   products   1   0         12       ...           ...
```

Confirm the restored document count:

```http
GET /products/_count
```

**Expected output:**

```json
{ "count": 12 }
```

> If the target index already exists, close it before restoring, delete it first, or restore with `rename_pattern` and `rename_replacement` to avoid overwriting an active index.

### 10. Snapshot Lifecycle Management (SLM)

Snapshot Lifecycle Management automates snapshot creation and retention. This example creates a nightly policy that snapshots application and log indices.

```http
PUT /_slm/policy/nightly-backup
{
  "schedule": "0 30 2 * * ?",
  "name": "<nightly-snap-{now/d}>",
  "repository": "my_backup",
  "config": {
    "indices": ["products", "articles", "logs-*"],
    "ignore_unavailable": true,
    "include_global_state": false
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}
```

**Expected output:**

```json
{ "acknowledged": true }
```

Test the policy manually:

```http
POST /_slm/policy/nightly-backup/_execute
```

**Expected output:**

```json
{ "snapshot_name": "nightly-snap-2024.06.01-abc123" }
```

Check policy execution history:

```http
GET /_slm/policy/nightly-backup?human
```

### 11. Python Script for Automated Backup Monitoring

This script checks the repository, reports the latest snapshot, and prints a simple status summary.

```python
import json
from datetime import datetime

import requests

ES_HOST = "http://localhost:9200"
REPO = "my_backup"


def check_snapshots():
    """Fetch all snapshots and report a status summary."""
    response = requests.get(f"{ES_HOST}/_snapshot/{REPO}/_all", timeout=15)
    data = response.json()

    if response.status_code >= 400:
        print("Snapshot API returned an error:")
        print(json.dumps(data, indent=2))
        return

    snapshots = data.get("snapshots", [])
    print(f"[{datetime.now().isoformat()}] Snapshots in '{REPO}': {len(snapshots)}")

    if not snapshots:
        print("  WARNING: No snapshots found!")
        return

    # Sort by start time so the latest snapshot is shown even if the API order changes.
    snapshots.sort(key=lambda item: item.get("start_time", ""))
    latest = snapshots[-1]

    duration_ms = latest.get("duration_in_millis", 0)
    duration_s = duration_ms / 1000

    print(f"  Latest   : {latest.get('snapshot')}")
    print(f"  State    : {latest.get('state')}")
    print(f"  Duration : {duration_s:.1f}s")
    print(f"  Indices  : {', '.join(latest.get('indices', []))}")

    if latest.get("state") != "SUCCESS":
        print(f"  ALERT: Snapshot '{latest.get('snapshot')}' state is '{latest.get('state')}'!")

    states = {}
    for snapshot in snapshots:
        state = snapshot.get("state", "UNKNOWN")
        states[state] = states.get(state, 0) + 1

    print(f"  Summary  : {json.dumps(states)}")


if __name__ == "__main__":
    check_snapshots()
```

**Expected output:**

```text
[2024-06-01T14:30:00.123456] Snapshots in 'my_backup': 2
  Latest   : nightly-snap-2024.06.01-abc123
  State    : SUCCESS
  Duration : 3.5s
  Indices  : products, articles, logs-2024.01
  Summary  : {"SUCCESS": 2}
```

### 12. Index Lifecycle Management (ILM) Overview

ILM automatically moves time-series indices through lifecycle phases based on age, size, or rollover conditions.

```
 HOT  ──▶  WARM  ──▶  COLD  ──▶  DELETE
(active    (shrink,   (archive,   (purge
 writes)    merge)     snapshot)   old data)
```

| Phase | Typical Purpose |
|---|---|
| Hot | Actively written and frequently searched |
| Warm | Read-only or rarely updated data optimized for search cost |
| Cold | Infrequently searched data, often backed by snapshots |
| Delete | Data removed after retention period |

Create an ILM policy for log indices:

```http
PUT /_ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "searchable_snapshot": {
            "snapshot_repository": "my_backup"
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

**Expected output:**

```json
{ "acknowledged": true }
```

Attach the policy to a log index template:

```http
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "logs_policy"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "service": { "type": "keyword" },
        "level": { "type": "keyword" },
        "message": { "type": "text" },
        "latency_ms": { "type": "integer" },
        "status_code": { "type": "integer" }
      }
    }
  }
}
```

Check lifecycle status for an index:

```http
GET /logs-2024.01/_ilm/explain
```

> For a production rollover setup, use a write alias such as `logs-write` and create the first index with `is_write_index: true`.

### 13. Troubleshooting Tips

#### Snapshot Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| `repository_missing_exception` | Repository was not registered | Create it with `PUT /_snapshot/<repo>` |
| `repository_exception` | `path.repo` not configured | Add the path to `elasticsearch.yml` and restart all nodes |
| State is `PARTIAL` | Some shards were unavailable | Check `_cluster/health` and restore shard allocation |
| State is `FAILED` | Disk full or permission problem | Verify repository disk space and directory permissions |
| `concurrent_snapshot_execution` | Another snapshot is running | Wait for completion or cancel the conflicting snapshot |
| Restore fails because index exists | Target index is already open | Close/delete the target index or restore with a renamed index |

#### Slow Queries

| Symptom | Cause | Fix |
|---------|-------|-----|
| High `query_time_in_millis` | Unoptimized query | Use Profile API, filters, keyword fields, and narrower queries |
| Slow aggregations | High-cardinality terms | Use `composite` aggregation with pagination |
| Latency spikes | Too many small segments | Force merge read-only indices only |
| Slow searches after bulk load | Refresh/merge pressure | Restore refresh settings and allow merges to complete |

#### Memory Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Frequent old-gen GC | Heap pressure | Increase heap carefully or reduce shard/query pressure |
| `circuit_breaking_exception` | Request too large | Reduce bulk batch size or aggregation cardinality |
| High OS memory usage | File-system cache pressure | Add nodes, reduce shard count, or improve storage capacity |
| Rejected write/search tasks | Thread pools saturated | Lower concurrency, increase nodes, or tune workload patterns |

### Cleanup

Remove the lab indices, SLM policy, ILM policy, and snapshot repository when you are finished.

```http
DELETE /products
DELETE /articles
DELETE /logs-2024.01
DELETE /_slm/policy/nightly-backup
DELETE /_ilm/policy/logs_policy
DELETE /_index_template/logs_template
DELETE /_snapshot/my_backup
```

> Deleting the snapshot repository unregisters it from Elasticsearch. It does not necessarily delete snapshot files from the filesystem location.

### Reflection

1. Which monitoring API gives you the fastest high-level view of cluster health?
2. Why can disabling refresh and replicas improve bulk indexing performance?
3. What risk does `translog.durability: async` introduce?
4. Why should force merge be used only on read-only indices?
5. What is the difference between taking a snapshot and restoring an index?
6. How do SLM and ILM work together in a production backup and retention strategy?
