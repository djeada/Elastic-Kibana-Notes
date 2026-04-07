## Task 1: Environment Setup, Cluster Basics, and Verification

**Objectives:**
- Install Elasticsearch and Kibana locally via Docker.
- Understand cluster fundamentals: nodes, shards, replicas.
- Verify installation via raw REST calls (Kibana), curl, and Python scripts.
- Learn to interpret cluster health and node information.

---

### Architecture Overview

```
+-----------------------------------------------------------+
|                    HOST MACHINE                            |
|                                                           |
|   +---------------------+    +----------------------+    |
|   |  Elasticsearch      |    |  Kibana              |    |
|   |  Container          |    |  Container           |    |
|   |                     |    |                      |    |
|   |  +---------------+  |    |  +----------------+  |    |
|   |  |  ES Node      |  |    |  |  Web UI        |  |    |
|   |  |  (master +    |  |<---|  |  Dev Tools     |  |    |
|   |  |   data)       |  |    |  |  Dashboards    |  |    |
|   |  +---------------+  |    |  +----------------+  |    |
|   |                     |    |                      |    |
|   |  Port: 9200 --------+----+---- Port: 5601      |    |
|   +---------------------+    +----------------------+    |
|                                                           |
|   Browser --> http://localhost:5601  (Kibana UI)          |
|   curl    --> http://localhost:9200  (ES REST API)        |
+-----------------------------------------------------------+
```

### Single-Node Cluster Architecture

```
+------------------------------------------+
|          Elasticsearch Cluster           |
|          Name: "docker-cluster"          |
|                                          |
|  +------------------------------------+  |
|  |  Node: single-node (master + data) |  |
|  |                                    |  |
|  |  Index: "products"                 |  |
|  |  +----------+  +----------+       |  |
|  |  | Primary  |  | Primary  |       |  |
|  |  | Shard 0  |  | Shard 1  |  ...  |  |
|  |  +----------+  +----------+       |  |
|  |                                    |  |
|  |  (Replicas = 0 in single-node)     |  |
|  +------------------------------------+  |
+------------------------------------------+
```

---

### Lab Steps

### Step 1: Installation Using Docker

Pull and start the Elasticsearch container:

```bash
docker run -d \
  --name elasticsearch \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  docker.elastic.co/elasticsearch/elasticsearch:8.6.0
```

**Environment Variables Explained:**

| Variable | Purpose |
|---|---|
| `discovery.type=single-node` | Runs ES as a single-node cluster (no multi-node discovery) |
| `xpack.security.enabled=false` | Disables security for local development (no HTTPS/auth) |
| `ES_JAVA_OPTS=-Xms512m -Xmx512m` | Sets JVM heap size (min and max) to 512 MB |

Wait approximately 30 seconds for Elasticsearch to start, then launch Kibana:

```bash
docker run -d \
  --name kibana \
  -p 5601:5601 \
  --link elasticsearch:elasticsearch \
  docker.elastic.co/kibana/kibana:8.6.0
```

Verify both containers are running:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**Expected Output:**
```
NAMES           STATUS          PORTS
kibana          Up 2 minutes    0.0.0.0:5601->5601/tcp
elasticsearch   Up 3 minutes    0.0.0.0:9200->9200/tcp, 9300/tcp
```

---

### Step 2: Cluster Verification via Kibana

Open your browser and navigate to `http://localhost:5601`. Go to **Management > Dev Tools**.

**Query the root endpoint:**

```http
GET /
```

**Equivalent curl command:**
```bash
curl -s http://localhost:9200/ | python3 -m json.tool
```

**Expected Output:**
```json
{
  "name": "abcdef123456",
  "cluster_name": "docker-cluster",
  "cluster_uuid": "Xy4Z...",
  "version": {
    "number": "8.6.0",
    "build_flavor": "default",
    "build_type": "docker",
    "build_hash": "...",
    "build_date": "2023-01-...",
    "build_snapshot": false,
    "lucene_version": "9.4.2",
    "minimum_wire_compatibility_version": "7.17.0",
    "minimum_index_compatibility_version": "7.0.0"
  },
  "tagline": "You Know, for Search"
}
```

---

### Step 3: Check Cluster Health

**In Kibana Dev Tools:**

```http
GET /_cluster/health
```

**Equivalent curl command:**
```bash
curl -s http://localhost:9200/_cluster/health?pretty
```

**Expected Output:**
```json
{
  "cluster_name": "docker-cluster",
  "status": "green",
  "timed_out": false,
  "number_of_nodes": 1,
  "number_of_data_nodes": 1,
  "active_primary_shards": 1,
  "active_shards": 1,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 0,
  "delayed_unassigned_shards": 0,
  "number_of_pending_tasks": 0,
  "number_of_in_flight_fetch": 0,
  "task_max_waiting_in_queue_millis": 0,
  "active_shards_percent_as_number": 100.0
}
```

### Understanding Cluster Health Response Fields

| Field | Description |
|---|---|
| `status` | **green** = all shards assigned; **yellow** = primary shards OK but replicas unassigned; **red** = some primary shards unassigned |
| `number_of_nodes` | Total nodes in the cluster (1 for single-node) |
| `number_of_data_nodes` | Nodes that hold data |
| `active_primary_shards` | Number of active primary shard copies |
| `active_shards` | Total active shards (primaries + replicas) |
| `unassigned_shards` | Shards waiting to be assigned to a node |
| `active_shards_percent_as_number` | Percentage of active shards -- 100.0 means fully healthy |

---

### Step 4: Check Node Information

**In Kibana Dev Tools:**

```http
GET /_cat/nodes?v
```

**Equivalent curl command:**
```bash
curl -s http://localhost:9200/_cat/nodes?v
```

**Expected Output:**
```
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role  master name
172.17.0.2           45          78   3    0.12    0.08     0.05 cdfhilmrstw *      abcdef123456
```

**Check version details:**

```http
GET /_cat/nodes?v&h=name,version,jdk,disk.total,disk.used_percent
```

**Expected Output:**
```
name           version jdk       disk.total disk.used_percent
abcdef123456   8.6.0   19.0.1    50gb       42.15
```

---

### Step 5: Verification via Python Script

Install the requests library if needed:

```bash
pip install requests
```

Use the following Python script to query cluster health:

```python
import requests
import json

ES_URL = "http://localhost:9200"

# 1. Cluster info
info = requests.get(f"{ES_URL}/").json()
print("=== Cluster Info ===")
print(f"Cluster Name : {info['cluster_name']}")
print(f"ES Version   : {info['version']['number']}")
print(f"Lucene Ver.  : {info['version']['lucene_version']}")
print()

# 2. Cluster health
health = requests.get(f"{ES_URL}/_cluster/health").json()
print("=== Cluster Health ===")
print(f"Status       : {health['status']}")
print(f"Nodes        : {health['number_of_nodes']}")
print(f"Data Nodes   : {health['number_of_data_nodes']}")
print(f"Active Shards: {health['active_shards']}")
print()

# 3. Node stats
nodes = requests.get(f"{ES_URL}/_nodes/stats/jvm").json()
for node_id, node in nodes['nodes'].items():
    print(f"=== Node: {node['name']} ===")
    jvm = node['jvm']['mem']
    print(f"Heap Used    : {jvm['heap_used_in_bytes'] // (1024*1024)} MB")
    print(f"Heap Max     : {jvm['heap_max_in_bytes'] // (1024*1024)} MB")
```

**Expected Output:**
```
=== Cluster Info ===
Cluster Name : docker-cluster
ES Version   : 8.6.0
Lucene Ver.  : 9.4.2

=== Cluster Health ===
Status       : green
Nodes        : 1
Data Nodes   : 1
Active Shards: 1

=== Node: abcdef123456 ===
Heap Used    : 198 MB
Heap Max     : 512 MB
```

---

### Step 6: Concept Discussion

In your lab journal, describe the following:

- **Node**: A single running instance of Elasticsearch. Each node stores data, participates in indexing, and handles search requests.
- **Shard**: A subdivision of an index. Each shard is a self-contained Lucene index. Primary shards hold the original data; replica shards are copies for fault tolerance.
- **Replica**: A copy of a primary shard on a different node. Replicas provide high availability and can serve read requests to improve throughput.

> **Why distributed architecture?** Spreading data across multiple nodes and shards allows Elasticsearch to handle datasets far larger than a single machine, parallelize search and indexing, and survive node failures without data loss.

---

### Troubleshooting Tips

| Problem | Cause | Solution |
|---|---|---|
| `port is already allocated` | Another process uses port 9200 or 5601 | Stop the conflicting process: `lsof -i :9200` then `kill <PID>`, or change the host port: `-p 9201:9200` |
| Container exits immediately | Insufficient memory for JVM | Increase Docker memory (Settings > Resources > 4 GB+) or reduce heap: `-e "ES_JAVA_OPTS=-Xms256m -Xmx256m"` |
| `max virtual memory areas vm.max_map_count [65530] is too low` | Linux kernel limit too low | Run on the host: `sudo sysctl -w vm.max_map_count=262144` |
| Kibana shows "Kibana server is not ready yet" | Elasticsearch has not fully started | Wait 60-90 seconds; check ES logs: `docker logs elasticsearch` |
| `connection refused` on curl | Container not running or still starting | Verify with `docker ps`; check logs with `docker logs elasticsearch` |
| Cannot connect with security enabled | xpack security requires HTTPS + token | Add `-e "xpack.security.enabled=false"` for local dev, or follow the ES security setup guide |

### Cleanup

When you are done, remove the containers:

```bash
docker stop elasticsearch kibana
docker rm elasticsearch kibana
```
