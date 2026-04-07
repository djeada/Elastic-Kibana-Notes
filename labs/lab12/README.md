## Task 12: Performance Tuning, Snapshot Backup, and Recovery

### Backup and Recovery Workflow

```
┌──────────────┐     ┌────────────┐     ┌──────────────┐
│ Live Cluster │────▶│  Snapshot   │────▶│  Repository  │
│  (indices)   │     │ (point-in-  │     │ (fs/S3/GCS)  │
│              │     │  time copy) │     │              │
└──────────────┘     └────────────┘     └──────┬───────┘
                                               │ restore
┌──────────────┐     ┌────────────┐            │
│  Recovered   │◀────│  Restore   │◀───────────┘
│   Cluster    │     │ (selective) │
└──────────────┘     └────────────┘
```

### Objectives

- Monitor cluster health and node performance with `_cat` and `_nodes` APIs.
- Tune index settings (`refresh_interval`, replicas, translog) for write/read performance.
- Optimize bulk indexing and force merge read-only indices.
- Set up snapshot repositories, create/list snapshots, and restore after data loss.
- Automate backups with SLM policies and Python monitoring scripts.
- Understand ILM for managing hot/warm/cold/delete lifecycle phases.

---

### 1. Performance Monitoring

**Cluster health:**
```bash
GET /_cat/health?v
```
**Expected output:**
```
epoch      timestamp cluster     status node.total node.data shards pri relo init unassign
1717012345 14:32:25  my-cluster  green           3         3     20  10    0    0        0
```

**Index sizes (sorted):**
```bash
GET /_cat/indices?v&s=store.size:desc
```
**Expected output:**
```
health status index      pri rep docs.count store.size pri.store.size
green  open   logs-2024    1   1     150000     45.2mb         22.6mb
green  open   products     1   1       5000     12.3mb          6.1mb
```

**Node-level statistics:**
```bash
GET /_nodes/stats/jvm,os,indices?human
```
**Expected output (abbreviated):**
```json
{
  "nodes": { "node-1-id": {
    "name": "es-node-1",
    "os": { "cpu": { "percent": 23 }, "mem": { "used_percent": 70 } },
    "jvm": {
      "mem": { "heap_used": "4.1gb", "heap_max": "8gb", "heap_used_percent": 51 },
      "gc": { "collectors": {
        "young": { "collection_count": 1240, "collection_time": "12.4s" },
        "old":   { "collection_count": 3,    "collection_time": "1.8s" }
      }}
    },
    "indices": {
      "docs": { "count": 158200 },
      "search": { "query_total": 8430, "query_time_in_millis": 32100 },
      "indexing": { "index_total": 158200, "index_time_in_millis": 45000 }
    }
  }}
}
```

> **Key metrics:** heap usage %, old-gen GC count, search/indexing latency.

---

### 2. Index Settings Tuning

#### 2a. `refresh_interval` Adjustment

Increasing refresh interval reduces overhead during heavy writes (default is `1s`):
```bash
PUT /products/_settings
{ "index": { "refresh_interval": "30s" } }
```
**Expected output:**
```json
{ "acknowledged": true }
```
Verify:
```bash
GET /products/_settings/index.refresh_interval
```
**Expected output:**
```json
{ "products": { "settings": { "index": { "refresh_interval": "30s" } } } }
```

> Set to `"-1"` to disable refresh entirely during bulk loads; restore afterward.

#### 2b. `number_of_replicas` Adjustment

Set replicas to 0 during bulk loads, then restore afterward:
```bash
PUT /products/_settings
{ "index": { "number_of_replicas": 0 } }
```
**Expected output:**
```json
{ "acknowledged": true }
```

#### 2c. Translog Settings

Switch to async translog flush for bulk indexing throughput:
```bash
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

> **Warning:** Async translog risks data loss on crash. Revert to `"request"` after bulk operations.

---

### 3. Query Performance — Profile API (Recap from Lab 08)

```bash
GET /products/_search
{
  "profile": true,
  "query": { "match": { "name": "laptop" } }
}
```
**Expected output (key section):**
```json
{
  "profile": { "shards": [{ "searches": [{ "query": [{
    "type": "TermQuery", "description": "name:laptop",
    "time_in_nanos": 254300,
    "breakdown": { "score": 48200, "create_weight": 95400, "advance": 18200 }
  }]}]}]}
}
```

> Use profiling to identify whether bottlenecks are in scoring, matching, or aggregation.

---

### 4. Bulk Indexing Optimization

**Step 1 — Disable refresh and replicas:**
```bash
PUT /products/_settings
{ "index": { "refresh_interval": "-1", "number_of_replicas": 0 } }
```
**Expected output:** `{ "acknowledged": true }`

**Step 2 — Send bulk requests (target 5–15 MB each):**
```bash
POST /_bulk
{"index": {"_index": "products"}}
{"name": "Wireless Mouse", "price": 25.99, "category": "accessories"}
{"index": {"_index": "products"}}
{"name": "USB-C Hub", "price": 39.99, "category": "accessories"}
```
**Expected output:**
```json
{
  "took": 42, "errors": false,
  "items": [
    { "index": { "_index": "products", "_id": "abc123", "status": 201 } },
    { "index": { "_index": "products", "_id": "def456", "status": 201 } }
  ]
}
```

**Step 3 — Re-enable refresh and replicas:**
```bash
PUT /products/_settings
{ "index": { "refresh_interval": "1s", "number_of_replicas": 1 } }
```
**Expected output:** `{ "acknowledged": true }`

| Tip | Reason |
|-----|--------|
| Disable refresh (`"-1"`) | Avoids new segment every second |
| Set replicas to 0 | Prevents duplicate writes to replica shards |
| Async translog | Reduces fsync overhead |
| 5–15 MB bulk requests | Balances throughput and memory pressure |

---

### 5. Force Merge for Read-Only Indices

```bash
POST /logs-2024.01/_forcemerge?max_num_segments=1
```
**Expected output:**
```json
{ "_shards": { "total": 2, "successful": 2, "failed": 0 } }
```

> **Important:** Never force merge indices still receiving writes.

---

### 6. Snapshot Repository Setup

> **Prerequisite:** Add `path.repo: ["/mount/backups"]` to `elasticsearch.yml` on every node.

```bash
PUT /_snapshot/my_backup
{
  "type": "fs",
  "settings": { "location": "/mount/backups/my_backup", "compress": true }
}
```
**Expected output:**
```json
{ "acknowledged": true }
```
Verify:
```bash
POST /_snapshot/my_backup/_verify
```
**Expected output:**
```json
{ "nodes": { "node-1-id": { "name": "es-node-1" }, "node-2-id": { "name": "es-node-2" } } }
```

---

### 7. Taking a Snapshot

```bash
PUT /_snapshot/my_backup/snapshot_2024_06_01?wait_for_completion=true
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
    "snapshot": "snapshot_2024_06_01",
    "indices": ["products", "articles"],
    "state": "SUCCESS",
    "start_time": "2024-06-01T10:00:00.000Z",
    "end_time": "2024-06-01T10:00:03.450Z",
    "duration_in_millis": 3450,
    "shards": { "total": 4, "failed": 0, "successful": 4 }
  }
}
```

---

### 8. Listing Snapshots

```bash
GET /_snapshot/my_backup/_all
```
**Expected output:**
```json
{
  "snapshots": [{
    "snapshot": "snapshot_2024_06_01",
    "indices": ["products", "articles"],
    "state": "SUCCESS",
    "start_time": "2024-06-01T10:00:00.000Z",
    "duration_in_millis": 3450
  }]
}
```

---

### 9. Simulating Data Loss and Restoring from Snapshot

**Delete an index:**
```bash
DELETE /products
```
**Expected output:** `{ "acknowledged": true }`

Confirm deletion:
```bash
GET /_cat/indices/products?v
```
**Expected output:** *(empty — no matching indices)*

**Restore from snapshot:**
```bash
POST /_snapshot/my_backup/snapshot_2024_06_01/_restore
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
```bash
GET /_cat/indices/products?v
```
**Expected output:**
```
health status index    pri rep docs.count store.size pri.store.size
green  open   products   1   1       5000     12.3mb          6.1mb
```

> If the target index already exists, close it first or use `rename_pattern`/`rename_replacement`.

---

### 10. Snapshot Lifecycle Management (SLM)

Create a policy for nightly snapshots with 30-day retention:
```bash
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
  "retention": { "expire_after": "30d", "min_count": 5, "max_count": 50 }
}
```
**Expected output:** `{ "acknowledged": true }`

Test the policy manually:
```bash
POST /_slm/policy/nightly-backup/_execute
```
**Expected output:**
```json
{ "snapshot_name": "nightly-snap-2024.06.01-abc123" }
```

---

### 11. Python Script for Automated Backup Monitoring

```python
import requests, json
from datetime import datetime

ES_HOST = "http://localhost:9200"
REPO    = "my_backup"

def check_snapshots():
    """Fetch all snapshots and report status summary."""
    data = requests.get(f"{ES_HOST}/_snapshot/{REPO}/_all").json()
    snapshots = data.get("snapshots", [])
    print(f"[{datetime.now().isoformat()}] Snapshots in '{REPO}': {len(snapshots)}")

    if not snapshots:
        print("  WARNING: No snapshots found!")
        return

    latest = snapshots[-1]
    duration_s = latest["duration_in_millis"] / 1000
    print(f"  Latest   : {latest['snapshot']}")
    print(f"  State    : {latest['state']}")
    print(f"  Duration : {duration_s:.1f}s")
    print(f"  Indices  : {', '.join(latest['indices'])}")

    if latest["state"] != "SUCCESS":
        print(f"  ALERT: Snapshot '{latest['snapshot']}' state is '{latest['state']}'!")

    states = {}
    for s in snapshots:
        states[s["state"]] = states.get(s["state"], 0) + 1
    print(f"  Summary  : {json.dumps(states)}")

if __name__ == "__main__":
    check_snapshots()
```
**Expected output:**
```
[2024-06-01T14:30:00.123456] Snapshots in 'my_backup': 12
  Latest   : nightly-snap-2024.06.01-abc123
  State    : SUCCESS
  Duration : 3.5s
  Indices  : products, articles
  Summary  : {"SUCCESS": 11, "PARTIAL": 1}
```

---

### 12. Index Lifecycle Management (ILM) Overview

ILM moves indices through phases automatically based on age or size:

```
 HOT  ──▶  WARM  ──▶  COLD  ──▶  DELETE
 (active    (shrink,   (frozen,    (purge
  writes)    merge)     archive)    >90d)
```

```bash
PUT /_ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot":  { "min_age": "0ms",  "actions": {
        "rollover": { "max_primary_shard_size": "50gb", "max_age": "1d" } }},
      "warm": { "min_age": "7d",   "actions": {
        "shrink": { "number_of_shards": 1 }, "forcemerge": { "max_num_segments": 1 } }},
      "cold": { "min_age": "30d",  "actions": {
        "searchable_snapshot": { "snapshot_repository": "my_backup" } }},
      "delete": { "min_age": "90d", "actions": { "delete": {} }}
    }
  }
}
```
**Expected output:** `{ "acknowledged": true }`

---

### 13. Troubleshooting Tips

#### Snapshot Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| `repository_missing_exception` | `path.repo` not set | Add to `elasticsearch.yml`, restart nodes |
| State is `PARTIAL` | Shards unavailable | Check `_cluster/health`, ensure shards started |
| State is `FAILED` | Disk full / permissions | Verify disk space and repo directory permissions |
| `concurrent_snapshot_execution` | Another snapshot running | Wait or cancel with `DELETE /_snapshot/repo/snap` |

#### Slow Queries

| Symptom | Cause | Fix |
|---------|-------|-----|
| High `query_time_in_millis` | Unoptimized query | Profile API → add keyword fields, refine query |
| Slow aggregations | High-cardinality terms | Use `composite` aggregation with pagination |
| Latency spikes | Too many segments | Force merge read-only indices |

#### Memory Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Frequent old-gen GC | Heap too small | Increase heap (max 50% RAM, ≤ 31 GB) |
| `circuit_breaking_exception` | Request too large | Reduce bulk batch size |
| High OS memory | Large indices, cache pressure | Add nodes to distribute shard load |

---

### Summary

| Topic | Key Takeaway |
|-------|-------------|
| Monitoring | `_cat` for quick checks, `_nodes/stats` for deep diagnostics |
| Tuning | Adjust `refresh_interval`, replicas, translog per workload |
| Profiling | Profile API pinpoints query bottlenecks |
| Bulk optimization | Disable refresh + replicas during large ingests |
| Force merge | Reduce segments on read-only indices |
| Snapshots | Repository setup → create → list → restore |
| SLM | Scheduled policies with retention rules |
| ILM | Automate hot → warm → cold → delete lifecycle |
