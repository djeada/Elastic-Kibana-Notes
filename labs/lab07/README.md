## Task 7: Multi-Node Cluster Configuration and Monitoring

**Objectives:**
- Deploy and configure a multi‑node cluster for scalability.
- Configure shard allocation, replicas, and node roles.
- Use both raw API calls (Kibana) and Python scripts to monitor cluster health.
- Understand node roles and shard allocation awareness.
- Monitor cluster health using `_cat` APIs and custom scripts.

---

### Multi-Node Cluster Topology

```
  ┌───────────────────────────────────────────────────────────────┐
  │                    es_cluster (3 nodes)                        │
  │                                                               │
  │  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │
  │  │    es1       │   │    es2       │   │    es3       │        │
  │  │  (master,    │   │  (data,      │   │  (data,      │        │
  │  │   data)      │   │   ingest)    │   │   ingest)    │        │
  │  │             │   │             │   │             │        │
  │  │ ┌─────────┐ │   │ ┌─────────┐ │   │ ┌─────────┐ │        │
  │  │ │products │ │   │ │products │ │   │ │products │ │        │
  │  │ │ P0 (pri)│ │   │ │ P1 (pri)│ │   │ │ P2 (pri)│ │        │
  │  │ │ P2 (rep)│ │   │ │ P0 (rep)│ │   │ │ P1 (rep)│ │        │
  │  │ └─────────┘ │   │ └─────────┘ │   │ └─────────┘ │        │
  │  │             │   │             │   │             │        │
  │  │ Port: 9200  │   │ Port: 9201  │   │ Port: 9202  │        │
  │  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘        │
  │         │                 │                 │                │
  │         └────── discovery.seed_hosts ───────┘                │
  │                                                               │
  │  P0, P1, P2 = primary shards    (rep) = replica shard        │
  └───────────────────────────────────────────────────────────────┘
```

---

### Task 7.1: Multi-Node Setup with Docker Compose

Create a `docker-compose.yml` with resource limits and persistent volumes:

```yaml
version: '3.8'
services:
  es1:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.6.0
    container_name: es1
    environment:
      - node.name=es1
      - cluster.name=es_cluster
      - node.roles=master,data
      - discovery.seed_hosts=es2,es3
      - cluster.initial_master_nodes=es1,es2,es3
      - bootstrap.memory_lock=true
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es1_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - es_net
    deploy:
      resources:
        limits:
          memory: 1G
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  es2:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.6.0
    container_name: es2
    environment:
      - node.name=es2
      - cluster.name=es_cluster
      - node.roles=data,ingest
      - discovery.seed_hosts=es1,es3
      - cluster.initial_master_nodes=es1,es2,es3
      - bootstrap.memory_lock=true
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es2_data:/usr/share/elasticsearch/data
    ports:
      - "9201:9200"
    networks:
      - es_net
    deploy:
      resources:
        limits:
          memory: 1G

  es3:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.6.0
    container_name: es3
    environment:
      - node.name=es3
      - cluster.name=es_cluster
      - node.roles=data,ingest
      - discovery.seed_hosts=es1,es2
      - cluster.initial_master_nodes=es1,es2,es3
      - bootstrap.memory_lock=true
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es3_data:/usr/share/elasticsearch/data
    ports:
      - "9202:9200"
    networks:
      - es_net
    deploy:
      resources:
        limits:
          memory: 1G

volumes:
  es1_data:
    driver: local
  es2_data:
    driver: local
  es3_data:
    driver: local

networks:
  es_net:
    driver: bridge
```

Bring up the cluster:
```bash
docker-compose up -d
```

**Expected output:**
```
Creating network "lab07_es_net" with driver "bridge"
Creating volume "lab07_es1_data" with local driver
Creating volume "lab07_es2_data" with local driver
Creating volume "lab07_es3_data" with local driver
Creating es1 ... done
Creating es2 ... done
Creating es3 ... done
```

Wait ~30 seconds for the cluster to form, then verify:
```bash
curl -s http://localhost:9200/_cluster/health?pretty
```

---

### Task 7.2: Node Roles Configuration

Elasticsearch nodes can have different roles. Here's a summary:

| Role      | Purpose                                | Memory Needs |
|-----------|----------------------------------------|--------------|
| `master`  | Cluster state, index metadata          | Low          |
| `data`    | Store shards, execute queries          | High         |
| `ingest`  | Run ingest pipelines on documents      | Medium       |
| `ml`      | Machine learning jobs                  | High         |
| `transform` | Run transform jobs                  | Medium       |
| `coordinating` | Route requests (no role set)      | Medium       |

In production, dedicate roles:
```yaml
# Dedicated master node (no data storage)
- node.roles=master

# Dedicated data node (no master election)
- node.roles=data

# Coordinating-only node (set empty roles)
- node.roles=
```

---

### Task 7.3: Cluster Monitoring with Cat APIs

The `_cat` APIs return human-readable tabular output:

#### Cluster Health
```
GET /_cluster/health?pretty
```

**Expected output:**
```json
{
  "cluster_name": "es_cluster",
  "status": "green",
  "timed_out": false,
  "number_of_nodes": 3,
  "number_of_data_nodes": 3,
  "active_primary_shards": 3,
  "active_shards": 6,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 0,
  "delayed_unassigned_shards": 0,
  "number_of_pending_tasks": 0,
  "active_shards_percent_as_number": 100.0
}
```

> **Status meanings:** `green` = all shards assigned, `yellow` = primary shards ok but replicas unassigned, `red` = some primary shards unassigned.

#### List Nodes
```
GET /_cat/nodes?v&h=name,ip,role,master,heap.percent,disk.used_percent,cpu
```

**Expected output:**
```
name  ip          role    master  heap.percent  disk.used_percent  cpu
es1   172.18.0.2  dm      *       45            32                  5
es2   172.18.0.3  di      -       38            28                 12
es3   172.18.0.4  di      -       42            30                  8
```

> Roles: `d` = data, `m` = master, `i` = ingest. `*` = current elected master.

#### List Shards
```
GET /_cat/shards?v&h=index,shard,prirep,state,docs,store,node
```

**Expected output:**
```
index     shard  prirep  state    docs  store    node
products  0      p       STARTED  1000  1.2mb    es1
products  0      r       STARTED  1000  1.2mb    es2
products  1      p       STARTED  980   1.1mb    es2
products  1      r       STARTED  980   1.1mb    es3
products  2      p       STARTED  1020  1.3mb    es3
products  2      r       STARTED  1020  1.3mb    es1
```

#### Disk Allocation
```
GET /_cat/allocation?v
```

**Expected output:**
```
shards  disk.indices  disk.used  disk.avail  disk.total  disk.percent  host         ip           node
4       2.5mb         15.2gb     44.8gb      60gb        25            172.18.0.2   172.18.0.2   es1
4       2.3mb         14.8gb     45.2gb      60gb        24            172.18.0.3   172.18.0.3   es2
4       2.4mb         15.0gb     45.0gb      60gb        25            172.18.0.4   172.18.0.4   es3
```

---

### Task 7.4: Shard Allocation Awareness

Force Elasticsearch to distribute replicas across availability zones:

```json
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.awareness.attributes": "zone"
  }
}
```

Then tag each node (in `elasticsearch.yml` or docker environment):
```yaml
# es1 and es2 in zone-a
- node.attr.zone=zone-a

# es3 in zone-b
- node.attr.zone=zone-b
```

This ensures a primary and its replica are never on nodes in the same zone.

Create an index with awareness in mind:
```json
PUT /zone_aware_index
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1
  }
}
```

**Expected output:**
```json
{ "acknowledged": true, "shards_acknowledged": true, "index": "zone_aware_index" }
```

---

### Task 7.5: Python Monitoring Script

A comprehensive monitoring script that outputs formatted cluster stats:

```python
import time
import json
import requests
from datetime import datetime

ES_URL = "http://localhost:9200"

def get_json(path):
    return requests.get(f"{ES_URL}{path}").json()

def print_separator():
    print("=" * 65)

def monitor_cluster():
    while True:
        try:
            now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            health = get_json("/_cluster/health")
            stats = get_json("/_nodes/stats/os,jvm,fs")

            print_separator()
            print(f"  CLUSTER MONITOR — {now}")
            print_separator()

            # Cluster Health
            status = health["status"].upper()
            print(f"  Cluster:  {health['cluster_name']}")
            print(f"  Status:   {status}")
            print(f"  Nodes:    {health['number_of_nodes']} total, "
                  f"{health['number_of_data_nodes']} data")
            print(f"  Shards:   {health['active_primary_shards']} primary, "
                  f"{health['active_shards']} total")
            print(f"  Unassigned: {health['unassigned_shards']}")

            # Per-Node Stats
            print(f"\n  {'Node':<10} {'Heap%':<8} {'CPU%':<8} {'Disk Used':<12} {'Disk Free':<12}")
            print(f"  {'-'*10} {'-'*8} {'-'*8} {'-'*12} {'-'*12}")

            for node_id, node in stats["nodes"].items():
                name = node["name"]
                heap_pct = node["jvm"]["mem"]["heap_used_percent"]
                cpu_pct = node["os"]["cpu"]["percent"]
                disk_total = node["fs"]["total"]["total_in_bytes"]
                disk_free = node["fs"]["total"]["free_in_bytes"]
                disk_used = disk_total - disk_free

                print(f"  {name:<10} {heap_pct:<8} {cpu_pct:<8} "
                      f"{disk_used // (1024**2):<12} MB "
                      f"{disk_free // (1024**2):<12} MB")

            print()

        except requests.ConnectionError:
            print(f"[{datetime.now()}] Connection to Elasticsearch failed!")

        time.sleep(10)

if __name__ == "__main__":
    monitor_cluster()
```

**Expected output:**
```
=================================================================
  CLUSTER MONITOR — 2024-01-15 14:30:00
=================================================================
  Cluster:  es_cluster
  Status:   GREEN
  Nodes:    3 total, 3 data
  Shards:   3 primary, 6 total
  Unassigned: 0

  Node       Heap%    CPU%     Disk Used    Disk Free
  ---------- -------- -------- ------------ ------------
  es1        45       5        15565        MB 45035        MB
  es2        38       12       15155        MB 45445        MB
  es3        42       8        15360        MB 45240        MB
```

---

### Troubleshooting Tips

- **Cluster status YELLOW**: Replicas cannot be assigned — usually because you have only 1 node. Add more nodes or set `number_of_replicas: 0`.
- **Cluster status RED**: A primary shard is unassigned. Check `GET /_cluster/allocation/explain` for details.
- **Node won't join cluster**: Verify `cluster.name` is identical on all nodes and `discovery.seed_hosts` is correct.
- **Out of memory**: Increase `ES_JAVA_OPTS` heap size (never exceed 50% of available RAM or 31GB).
- **Disk watermark reached**: Elasticsearch stops allocating shards when disk usage exceeds 85%. Free disk space or add nodes.
- **Split brain**: Ensure `cluster.initial_master_nodes` is set correctly and you have an odd number of master-eligible nodes.

```bash
# Check why a shard is unassigned
GET /_cluster/allocation/explain?pretty
```

---

### Reflection

Discuss how shard allocation, nodes, and replica settings enhance reliability:
1. What happens if `es2` goes down? Does the cluster remain green?
2. How does shard allocation awareness prevent data loss in a zone failure?
3. What is the trade-off between more replicas and indexing performance?
