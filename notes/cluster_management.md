## Cluster Management

Cluster management in Elasticsearch refers to the set of practices, APIs, and operational
strategies used to maintain a healthy, performant, and resilient multi-node search cluster.
Unlike managing a single database server, Elasticsearch clusters are distributed systems where
data is automatically partitioned into shards, replicated for fault tolerance, and spread across
a fleet of nodes that must coordinate to present a unified view of the data.

Effective cluster management spans a wide range of responsibilities: ensuring that the correct
number and type of nodes are available, monitoring resource consumption across the fleet,
configuring shard allocation policies to balance load and availability, handling rolling upgrades
without downtime, and diagnosing issues when shards become unassigned or nodes fall out of the
cluster. As data volumes and query loads grow, the operational complexity increases, making a
solid understanding of cluster internals essential for any Elasticsearch administrator.

This document covers the core aspects of cluster management, from architecture and node discovery
through health monitoring, shard allocation, scaling, and troubleshooting. Each section includes
the relevant APIs and practical examples drawn from real production scenarios.


### Cluster Architecture

An Elasticsearch cluster is composed of nodes, each of which serves one or more roles. In small
development environments, a single node may fill every role simultaneously. In production, roles
are separated across dedicated hardware to isolate failure domains and prevent resource contention
between coordination, data storage, and ingest workloads.

The following diagram shows a representative production cluster with dedicated master nodes, data
nodes in a hot-warm architecture, an ingest node, and a coordinating node:

```
  Production Cluster Architecture
  --------------------------------

  Clients / Kibana / Logstash
         |
         v
  +------------------------------------------------------+
  |              Coordinating Node (coord-1)              |
  |         Routes requests, merges partial results       |
  +---+------------------+-------------------+-------+----+
      |                  |                   |       |
      v                  v                   v       v
  +--------+   +--------+   +--------+   +------------+
  |Master 1|   |Master 2|   |Master 3|   | Ingest-1   |
  |(master)|   |(master)|   |(master)|   | (ingest)   |
  |        |   |        |   |        |   | Pipelines, |
  | Cluster|   | Cluster|   | Cluster|   | transforms |
  | state  |   | state  |   | state  |   +------------+
  +--------+   +--------+   +--------+
      |             |             |
      +------+------+------+-----+
             |             |
             v             v
  +-----------------------+-----------------------+
  |       HOT TIER        |      WARM TIER        |
  |                       |                       |
  | +-------+ +-------+  |  +-------+ +-------+  |
  | |Data-H1| |Data-H2|  |  |Data-W1| |Data-W2|  |
  | | NVMe  | | NVMe  |  |  |  HDD  | |  HDD  |  |
  | | 64 GB | | 64 GB |  |  | 32 GB | | 32 GB |  |
  | +-------+ +-------+  |  +-------+ +-------+  |
  +-----------------------+-----------------------+
```

**Master nodes** manage cluster state (index metadata, shard routing tables, node membership) and
should always be deployed as a quorum of three dedicated instances. They require minimal disk and
CPU but are latency-sensitive -- a slow master destabilizes the entire cluster.

**Data nodes** hold shards and execute indexing, search, and aggregation operations. They are the
most resource-intensive nodes and benefit from fast storage (NVMe for hot tiers) and large JVM
heaps.

**Ingest nodes** run ingest pipelines (parsing, enrichment, transformation) before documents are
indexed. Dedicating nodes to this role prevents pipeline-heavy workloads from starving search
queries of resources.

**Coordinating nodes** act as smart load balancers. They receive client requests, fan them out to
the relevant data nodes, collect partial results, and merge them into a final response. In
high-throughput deployments, dedicated coordinating nodes prevent scatter-gather overhead from
consuming data node resources.

---

### Node Discovery and Cluster Formation

When an Elasticsearch node starts, it must discover and join an existing cluster or, during
initial bootstrapping, form a new one. This process is governed by a combination of network
configuration and explicit seed-host settings.

**Seed hosts** are a list of addresses that a new node contacts to discover the current cluster
members. In `elasticsearch.yml`, this is configured as:

```
  discovery.seed_hosts:
    - 10.0.1.10:9300
    - 10.0.1.11:9300
    - 10.0.1.12:9300
```

The node sends a discovery request to each seed host. Any master-eligible node that responds
provides the full list of current cluster members, allowing the new node to join.

**Initial master node bootstrapping** is required only when forming a brand-new cluster for the
very first time. The `cluster.initial_master_nodes` setting tells the cluster which
master-eligible nodes should participate in the initial election:

```
  cluster.initial_master_nodes:
    - master-1
    - master-2
    - master-3
```

This setting must list node names (not addresses) and should be removed after the cluster has
formed to prevent accidental re-bootstrapping. On subsequent restarts, nodes use the persisted
cluster state to rejoin without needing this setting.

```
  Node Discovery Flow
  --------------------

  +----------+        +----------+        +----------+
  | New Node |        | Seed Host|        | Elected  |
  |          |        | (master- |        | Master   |
  |          |        |  eligible)|       |          |
  +----+-----+        +----+-----+        +----+-----+
       |                    |                   |
       |  1. Ping seed      |                   |
       |------------------->|                   |
       |                    |                   |
       |  2. Return cluster |                   |
       |     member list    |                   |
       |<-------------------|                   |
       |                    |                   |
       |  3. Join request   |                   |
       |--------------------------------------->|
       |                    |                   |
       |  4. Cluster state  |                   |
       |     update (new    |                   |
       |     node added)    |                   |
       |<---------------------------------------|
       |                    |                   |
```

**Master election** uses a quorum-based voting mechanism. A candidate needs votes from a majority
of the master-eligible nodes to be elected. In a three-node quorum, two votes are required. The
elected master then becomes responsible for publishing cluster state updates, allocating shards,
and managing node membership.

---

### Cluster Health

The `_cluster/health` API is the single most important monitoring endpoint. It provides an
at-a-glance summary of the cluster's operational status, including the number of nodes, shard
allocation progress, and any pending tasks.

```json
GET /_cluster/health
```

```json
{
  "cluster_name": "production-logging",
  "status": "green",
  "timed_out": false,
  "number_of_nodes": 9,
  "number_of_data_nodes": 4,
  "active_primary_shards": 247,
  "active_shards": 494,
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

The `status` field reports one of three values, each representing a progressively more serious
condition:

```
  Cluster Health States
  ----------------------

  +-------------------------------------------------------------------+
  | State  | Meaning                                                  |
  +--------+----------------------------------------------------------+
  | GREEN  | All primary and replica shards are allocated and active. |
  |        | The cluster is fully operational with full redundancy.   |
  +--------+----------------------------------------------------------+
  | YELLOW | All primary shards are allocated, but one or more        |
  |        | replica shards are unassigned. Data is complete but      |
  |        | redundancy is reduced. Common on single-node clusters    |
  |        | where replicas cannot be allocated to a different node.  |
  +--------+----------------------------------------------------------+
  | RED    | One or more primary shards are unassigned. Some data     |
  |        | is unavailable for search and indexing. This requires    |
  |        | immediate investigation.                                 |
  +--------+----------------------------------------------------------+
```

You can also check health at the index level by appending the index name:

```json
GET /_cluster/health/logs-2024.07?level=shards
```

```json
{
  "cluster_name": "production-logging",
  "status": "green",
  "number_of_nodes": 9,
  "number_of_data_nodes": 4,
  "active_primary_shards": 5,
  "active_shards": 10,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 0,
  "indices": {
    "logs-2024.07": {
      "status": "green",
      "number_of_shards": 5,
      "number_of_replicas": 1,
      "active_primary_shards": 5,
      "active_shards": 10,
      "relocating_shards": 0,
      "initializing_shards": 0,
      "unassigned_shards": 0
    }
  }
}
```

The `level` parameter accepts `cluster` (default), `indices`, or `shards`, with each level
providing progressively more granular detail.

---

### Node Info and Stats

Understanding the resource consumption and operational characteristics of each node is essential
for capacity planning and troubleshooting. Elasticsearch provides two complementary APIs for
this purpose.

**Node info** returns static configuration details -- roles, installed plugins, OS information,
JVM settings, and transport/HTTP addresses:

```json
GET /_nodes/info
```

To retrieve info for specific nodes or specific categories:

```json
GET /_nodes/data-h1,data-h2/jvm,os
```

**Node stats** returns dynamic runtime metrics -- heap usage, garbage collection frequency,
indexing throughput, search latency, disk I/O, and thread pool activity:

```json
GET /_nodes/stats
```

```json
{
  "cluster_name": "production-logging",
  "nodes": {
    "xB2r...kQ4": {
      "name": "data-h1",
      "roles": ["data_hot", "data_content"],
      "os": {
        "cpu": {
          "percent": 42,
          "load_average": { "1m": 2.15, "5m": 1.87, "15m": 1.62 }
        },
        "mem": {
          "total_in_bytes": 68719476736,
          "used_in_bytes": 58522451968,
          "free_in_bytes": 10197024768,
          "used_percent": 85
        }
      },
      "jvm": {
        "mem": {
          "heap_used_in_bytes": 21474836480,
          "heap_max_in_bytes": 32212254720,
          "heap_used_percent": 66
        },
        "gc": {
          "collectors": {
            "old": { "collection_count": 3, "collection_time_in_millis": 1250 },
            "young": { "collection_count": 14502, "collection_time_in_millis": 98200 }
          }
        }
      },
      "fs": {
        "total": {
          "total_in_bytes": 1073741824000,
          "free_in_bytes": 429496729600,
          "available_in_bytes": 429496729600
        }
      },
      "thread_pool": {
        "search": { "threads": 13, "queue": 0, "active": 3, "rejected": 0 },
        "write":  { "threads": 8,  "queue": 0, "active": 1, "rejected": 0 },
        "get":    { "threads": 8,  "queue": 0, "active": 0, "rejected": 0 }
      }
    }
  }
}
```

Key metrics to monitor:

```
  Critical Node Metrics
  ----------------------

  +------------------------+---------------------------------------------------+
  | Metric                 | What to Watch                                     |
  +------------------------+---------------------------------------------------+
  | jvm.mem.heap_used_%    | Sustained usage above 75% triggers GC pressure.   |
  |                        | Above 85% risks OOM circuit breakers.             |
  +------------------------+---------------------------------------------------+
  | os.cpu.percent         | Sustained usage above 80% indicates the node      |
  |                        | needs more CPU or fewer shards.                   |
  +------------------------+---------------------------------------------------+
  | fs.total.available     | When disk usage crosses the high watermark (90%), |
  |                        | ES stops allocating shards to that node.          |
  +------------------------+---------------------------------------------------+
  | thread_pool.*.rejected | Non-zero rejected counts mean the node cannot     |
  |                        | keep up. Scale out or reduce query load.          |
  +------------------------+---------------------------------------------------+
  | gc.old.collection_time | Frequent long old-gen GC pauses indicate heap     |
  |                        | pressure. Consider reducing heap or adding nodes. |
  +------------------------+---------------------------------------------------+
```

---

### Shard Allocation

Shard allocation is the process by which Elasticsearch decides which node should host each
primary and replica shard. The allocation engine runs on the elected master node and considers
a variety of factors when making placement decisions.

**Disk-based allocation** ensures that nodes do not run out of storage. Three watermark thresholds
control this behavior:

- **Low watermark** (`cluster.routing.allocation.disk.watermark.low`, default `85%`) -- ES will
  not allocate new shards to nodes that have crossed this threshold.
- **High watermark** (`cluster.routing.allocation.disk.watermark.high`, default `90%`) -- ES
  will attempt to relocate shards away from nodes that have crossed this threshold.
- **Flood stage** (`cluster.routing.allocation.disk.watermark.flood_stage`, default `95%`) --
  ES enforces a read-only index block on every index that has a shard on a node exceeding this
  threshold.

```json
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low": "80%",
    "cluster.routing.allocation.disk.watermark.high": "88%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "93%"
  }
}
```

```json
{
  "acknowledged": true,
  "persistent": {
    "cluster": {
      "routing": {
        "allocation": {
          "disk": {
            "watermark": {
              "low": "80%",
              "high": "88%",
              "flood_stage": "93%"
            }
          }
        }
      }
    }
  },
  "transient": {}
}
```

**Allocation awareness** allows you to distribute shards across failure domains such as
availability zones or server racks. By tagging each node with a custom attribute and configuring
awareness, Elasticsearch ensures that a primary shard and its replica are never placed in the
same zone:

```
  # elasticsearch.yml on each node
  node.attr.zone: us-east-1a          # or us-east-1b, us-east-1c
  cluster.routing.allocation.awareness.attributes: zone
```

```
  Allocation Awareness (2 zones)
  -------------------------------

  Zone A                          Zone B
  +---------------------------+   +---------------------------+
  | data-1                    |   | data-2                    |
  |  [P0] [P2] [R1] [R3]     |   |  [P1] [P3] [R0] [R2]     |
  +---------------------------+   +---------------------------+

  Primary 0 is in Zone A, its replica (R0) is in Zone B.
  If Zone A fails, all data remains available via replicas in Zone B.
```

**Forced awareness** goes further by refusing to allocate replicas if the required zone is
unavailable, preventing all copies from landing in a single zone:

```
  cluster.routing.allocation.awareness.force.zone.values: us-east-1a,us-east-1b
```

Other allocation settings include:

- `cluster.routing.allocation.enable` -- controls whether allocation is enabled. Values:
  `all` (default), `primaries`, `new_primaries`, `none`.
- `cluster.routing.allocation.cluster_concurrent_rebalance` -- limits the number of concurrent
  shard movements during rebalancing (default `2`).
- `cluster.routing.allocation.node_concurrent_recoveries` -- limits the number of concurrent
  shard recoveries per node (default `2`).

---

### _cat APIs

The `_cat` APIs provide human-readable, tabular output for quick operational checks. They are the
command-line operator's best friend and support a `v` parameter for column headers.

**_cat/health** -- compact cluster health in a single line:

```json
GET /_cat/health?v
```

```
epoch      timestamp cluster            status node.total node.data shards  pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1720180200 12:30:00  production-logging green           9         4    494  247    0    0        0             0                  -                100.0%
```

**_cat/nodes** -- one line per node with key resource metrics:

```json
GET /_cat/nodes?v&h=name,role,heap.percent,disk.used_percent,cpu,load_1m,master
```

```
name      role   heap.percent disk.used_percent cpu load_1m master
master-1  m                12                 8   2    0.15 *
master-2  m                10                 7   1    0.10 -
master-3  m                11                 9   1    0.12 -
data-h1   dh               66                60  42    2.15 -
data-h2   dh               58                55  38    1.90 -
data-w1   dw               45                72  15    0.80 -
data-w2   dw               42                68  12    0.65 -
ingest-1  i                30                12   8    0.45 -
coord-1   -                25                 5   6    0.35 -
```

The `master` column shows `*` for the currently elected master and `-` for other nodes. The
`role` column uses shorthand characters: `m` (master), `d` (data), `h` (data_hot), `w`
(data_warm), `i` (ingest), `-` (coordinating only).

**_cat/indices** -- one line per index with document counts, storage, and health:

```json
GET /_cat/indices?v&h=health,status,index,pri,rep,docs.count,store.size&s=store.size:desc
```

```
health status index              pri rep docs.count store.size
green  open   logs-2024.07         5   1   84200150     42.1gb
green  open   logs-2024.06         5   1   78430000     38.7gb
green  open   metrics-2024.07      3   1   25000000     12.3gb
green  open   products             3   1      45000    128.5mb
yellow open   temp-imports         1   1       5200      2.1mb
```

An index showing `yellow` health means at least one replica is unassigned. The `s` parameter
sorts output by the specified column.

**_cat/shards** -- shows every shard, its state, node placement, and size:

```json
GET /_cat/shards/logs-2024.07?v&h=index,shard,prirep,state,docs,store,node
```

```
index          shard prirep state   docs     store node
logs-2024.07   0     p      STARTED 16840030 8.4gb data-h1
logs-2024.07   0     r      STARTED 16840030 8.4gb data-h2
logs-2024.07   1     p      STARTED 16840030 8.4gb data-h2
logs-2024.07   1     r      STARTED 16840030 8.4gb data-h1
logs-2024.07   2     p      STARTED 16840030 8.4gb data-h1
logs-2024.07   2     r      STARTED 16840030 8.4gb data-h2
logs-2024.07   3     p      STARTED 16840030 8.4gb data-h2
logs-2024.07   3     r      STARTED 16840030 8.4gb data-h1
logs-2024.07   4     p      STARTED 16840030 8.4gb data-h1
logs-2024.07   4     r      STARTED 16840030 8.4gb data-h2
```

**_cat/allocation** -- disk usage and shard count per node:

```json
GET /_cat/allocation?v
```

```
shards disk.indices disk.used disk.avail disk.total disk.percent host       ip         node
  124       21.3gb    25.1gb     74.9gb    100.0gb           25 10.0.1.20  10.0.1.20  data-h1
  124       19.8gb    23.6gb     76.4gb    100.0gb           23 10.0.1.21  10.0.1.21  data-h2
  123       14.2gb    17.9gb     82.1gb    100.0gb           17 10.0.1.30  10.0.1.30  data-w1
  123       13.7gb    17.1gb     82.9gb    100.0gb           17 10.0.1.31  10.0.1.31  data-w2
```

This view is essential for identifying imbalanced nodes. If one node holds significantly more
shards or disk than others, the cluster may benefit from manual rebalancing or adjustment of
allocation settings.

---

### Scaling Horizontally

Elasticsearch is designed to scale by adding nodes. When a new data node joins the cluster, the
master node automatically begins rebalancing shards to include the new node, distributing the
storage and query load more evenly.

**Adding a new node** requires minimal configuration. The new node needs the same `cluster.name`,
network access to the seed hosts, and appropriate role configuration:

```
  # elasticsearch.yml on the new node
  cluster.name: production-logging
  node.name: data-h3
  node.roles: [data_hot, data_content]
  discovery.seed_hosts: ["10.0.1.10:9300", "10.0.1.11:9300", "10.0.1.12:9300"]
```

Once the node starts and joins the cluster, shard rebalancing begins automatically. You can
monitor progress with:

```json
GET /_cat/recovery?v&active_only=true&h=index,shard,source_node,target_node,bytes_percent
```

```
index          shard source_node target_node bytes_percent
logs-2024.07   2     data-h1     data-h3     45.2%
logs-2024.07   4     data-h2     data-h3     32.8%
metrics-2024   1     data-h1     data-h3     67.1%
```

**Recommended production node roles** for a medium-sized cluster:

```
  Scaling Strategy
  -----------------

  +---------------------------------------------------------------+
  | Node Count | Recommended Layout                                |
  +------------+--------------------------------------------------+
  |     3      | All nodes: master + data + ingest (dev/small)    |
  +------------+--------------------------------------------------+
  |     5-7    | 3 dedicated masters, 2-4 data nodes              |
  +------------+--------------------------------------------------+
  |    8-15    | 3 dedicated masters, 4-10 data (hot/warm),       |
  |            | 1-2 dedicated ingest nodes                       |
  +------------+--------------------------------------------------+
  |    15+     | 3 dedicated masters, hot/warm/cold data tiers,   |
  |            | dedicated ingest, dedicated coordinating nodes   |
  +------------+--------------------------------------------------+
```

When scaling out, it is important to consider the shard-to-node ratio. A common guideline is to
keep each shard between 10 GB and 50 GB in size, and to avoid exceeding 20 shards per GB of JVM
heap on a data node. Oversharding -- having too many small shards -- wastes resources because
each shard consumes memory for its segment metadata regardless of size.

---

### Rolling Restarts

A rolling restart is the standard procedure for upgrading Elasticsearch, applying configuration
changes, or performing JVM maintenance without incurring downtime. The key principle is to take
one node offline at a time while ensuring shard redundancy is maintained.

**Step-by-step procedure:**

**Step 1 -- Disable shard allocation.** This prevents the cluster from immediately trying to
replicate shards away from the node you are about to stop, which would cause unnecessary I/O:

```json
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "primaries"
  }
}
```

```json
{
  "acknowledged": true,
  "persistent": {
    "cluster": {
      "routing": {
        "allocation": {
          "enable": "primaries"
        }
      }
    }
  },
  "transient": {}
}
```

**Step 2 -- Perform a synced flush (pre-8.x) or flush.** This ensures that the transaction log
is empty and shard recovery will be fast when the node comes back:

```json
POST /_flush
```

```json
{
  "_shards": {
    "total": 494,
    "successful": 494,
    "failed": 0
  }
}
```

**Step 3 -- Stop the node.** Use your system's service manager to shut down Elasticsearch on the
target node:

```
  sudo systemctl stop elasticsearch          # systemd
  sudo /etc/init.d/elasticsearch stop        # SysV init
```

**Step 4 -- Perform maintenance.** Apply the upgrade, configuration change, or JVM update.

**Step 5 -- Start the node and wait for it to rejoin the cluster:**

```
  sudo systemctl start elasticsearch
```

Monitor the node's return:

```json
GET /_cat/nodes?v&h=name,role,master
```

**Step 6 -- Re-enable shard allocation:**

```json
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "all"
  }
}
```

```json
{
  "acknowledged": true,
  "persistent": {
    "cluster": {
      "routing": {
        "allocation": {
          "enable": "all"
        }
      }
    }
  },
  "transient": {}
}
```

**Step 7 -- Wait for the cluster to return to green health before proceeding to the next node:**

```json
GET /_cluster/health?wait_for_status=green&timeout=5m
```

Repeat steps 1 through 7 for each node in the cluster. Always restart data nodes before master
nodes when upgrading, and save the elected master for last.

---

### Cluster Settings

Elasticsearch supports two scopes for dynamic cluster-level settings: **transient** and
**persistent**. Both can be updated at runtime without restarting nodes, but they differ in
durability.

- **Transient settings** survive a full cluster restart but are cleared if the cluster is fully
  shut down and re-bootstrapped. They are useful for temporary operational overrides.
- **Persistent settings** are stored in the cluster state and survive full restarts. They are
  the correct choice for permanent configuration changes.

The order of precedence is: transient settings > persistent settings > `elasticsearch.yml`
settings > default values.

```json
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.cluster_concurrent_rebalance": 4,
    "action.destructive_requires_name": true
  },
  "transient": {
    "logger.org.elasticsearch.discovery": "DEBUG"
  }
}
```

```json
{
  "acknowledged": true,
  "persistent": {
    "cluster": {
      "routing": {
        "allocation": {
          "cluster_concurrent_rebalance": "4"
        }
      }
    },
    "action": {
      "destructive_requires_name": "true"
    }
  },
  "transient": {
    "logger": {
      "org.elasticsearch.discovery": "DEBUG"
    }
  }
}
```

To reset a setting back to its default, set it to `null`:

```json
PUT /_cluster/settings
{
  "transient": {
    "logger.org.elasticsearch.discovery": null
  }
}
```

To view all current cluster settings:

```json
GET /_cluster/settings?include_defaults=true&flat_settings=true
```

---

### Split Brain Prevention

A split brain occurs when a cluster partitions into two or more independent sub-clusters, each
believing it is the authoritative cluster. Both halves continue accepting writes independently,
leading to divergent data that is extremely difficult to reconcile. Preventing split brain is one
of the most critical aspects of cluster configuration.

```
  Split Brain Scenario
  ---------------------

  Normal:                        Split Brain:
  +--------+                     +--------+     +--------+
  |Master 1|  <-- all nodes -->  |Master 1|     |Master 2|
  |Master 2|      connected      | Node A |     | Node B |
  |Master 3|                     | (thinks|     | (thinks|
  | Node A |                     |  it is |     |  it is |
  | Node B |                     |  THE   |     |  THE   |
  +--------+                     | cluster|     | cluster|
                                 +--------+     +--------+
                                   Writes          Writes
                                   diverge!        diverge!
```

**Pre-7.x: `discovery.zen.minimum_master_nodes`**

In Elasticsearch versions before 7.0, operators had to manually configure the minimum number of
master-eligible nodes required to form a quorum. The formula was:

```
  minimum_master_nodes = (number_of_master_eligible_nodes / 2) + 1
```

For a cluster with 3 master-eligible nodes, this would be `2`. If a network partition left a
minority of master-eligible nodes on one side, that side could not elect a master and would stop
accepting writes -- preventing the split brain.

```json
PUT /_cluster/settings
{
  "persistent": {
    "discovery.zen.minimum_master_nodes": 2
  }
}
```

The danger of this approach was that operators could forget to update the setting when adding or
removing master-eligible nodes, accidentally making split brain possible.

**7.x and later: Automatic voting configuration**

Starting with Elasticsearch 7.0, the voting configuration is managed automatically. The cluster
maintains a set of voting nodes and adjusts the quorum as master-eligible nodes are added or
removed. This eliminates the need for manual `minimum_master_nodes` configuration and
significantly reduces the risk of misconfiguration.

The voting configuration can be inspected via the cluster state:

```json
GET /_cluster/state/metadata?filter_path=metadata.cluster_coordination
```

If a master-eligible node is permanently removed, it should be explicitly excluded from the
voting configuration to prevent the cluster from waiting for a node that will never return:

```json
POST /_cluster/voting_config_exclusions?node_names=master-old
```

---

### Monitoring

Continuous monitoring is essential for maintaining cluster health and catching problems before
they escalate. Elasticsearch provides several layers of monitoring, from lightweight API calls
to full-featured dashboards in Kibana.

**Cluster stats** provide an aggregate view of resource usage across the entire cluster:

```json
GET /_cluster/stats
```

```json
{
  "cluster_name": "production-logging",
  "cluster_uuid": "abc123...",
  "status": "green",
  "indices": {
    "count": 42,
    "shards": {
      "total": 494,
      "primaries": 247
    },
    "docs": {
      "count": 187650000,
      "deleted": 1240000
    },
    "store": {
      "size_in_bytes": 102005473280
    }
  },
  "nodes": {
    "count": {
      "total": 9,
      "master": 3,
      "data": 4,
      "ingest": 1,
      "coordinating_only": 1
    },
    "os": {
      "available_processors": 72,
      "mem": {
        "total_in_bytes": 412316860416,
        "used_in_bytes": 350069331353
      }
    },
    "jvm": {
      "mem": {
        "heap_used_in_bytes": 85899345920,
        "heap_max_in_bytes": 137438953472
      }
    },
    "fs": {
      "total_in_bytes": 4294967296000,
      "available_in_bytes": 2576980377600
    }
  }
}
```

**Kibana Stack Monitoring** provides a purpose-built dashboard for visualizing cluster metrics
over time. It tracks node performance, index rates, search latency, JVM memory pressure, and
circuit breaker trips. To enable it:

```json
PUT /_cluster/settings
{
  "persistent": {
    "xpack.monitoring.collection.enabled": true,
    "xpack.monitoring.collection.interval": "10s"
  }
}
```

For production deployments, monitoring data should be shipped to a dedicated monitoring cluster
rather than stored on the production cluster itself, to avoid the monitoring overhead from
affecting production performance.

**Key monitoring thresholds:**

```
  Monitoring Alert Thresholds
  ----------------------------

  +------------------------------+------------------+------------------+
  | Metric                       | Warning          | Critical         |
  +------------------------------+------------------+------------------+
  | Cluster health               | yellow           | red              |
  | JVM heap usage               | > 75%            | > 85%            |
  | Disk usage per node          | > 80%            | > 90%            |
  | CPU usage per node           | > 75% sustained  | > 90% sustained  |
  | Search latency (p95)         | > 500ms          | > 2000ms         |
  | Indexing rejected threads    | > 0              | > 100/min        |
  | Pending cluster tasks        | > 10             | > 50             |
  | Old GC collection time       | > 5s/min         | > 15s/min        |
  +------------------------------+------------------+------------------+
```

---

### Troubleshooting

Cluster issues most commonly manifest as unassigned shards, degraded performance, or nodes
leaving the cluster unexpectedly. The following sections cover the most frequent problems and
their resolutions.

**Identifying unassigned shards:**

```json
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason&s=state
```

```
index          shard prirep state      unassigned.reason
logs-2024.07   3     r      UNASSIGNED NODE_LEFT
logs-2024.07   4     r      UNASSIGNED ALLOCATION_FAILED
products       0     r      UNASSIGNED INDEX_CREATED
```

Common `unassigned.reason` values:

```
  +---------------------+------------------------------------------------------+
  | Reason              | Description                                          |
  +---------------------+------------------------------------------------------+
  | INDEX_CREATED       | Shard was created with the index but never assigned. |
  |                     | Often occurs on single-node clusters with replicas.  |
  +---------------------+------------------------------------------------------+
  | NODE_LEFT           | The node hosting the shard left the cluster.         |
  +---------------------+------------------------------------------------------+
  | ALLOCATION_FAILED   | Shard allocation was attempted but failed (e.g.,     |
  |                     | disk full, incompatible version).                    |
  +---------------------+------------------------------------------------------+
  | CLUSTER_RECOVERED   | Shards are being recovered after a full cluster      |
  |                     | restart.                                             |
  +---------------------+------------------------------------------------------+
  | REROUTE_CANCELLED   | Manual reroute was cancelled.                        |
  +---------------------+------------------------------------------------------+
  | REINITIALIZED       | Shard recovery was restarted.                        |
  +---------------------+------------------------------------------------------+
```

**Allocation explain API** is the most powerful diagnostic tool for understanding why a shard
is unassigned:

```json
GET /_cluster/allocation/explain
{
  "index": "logs-2024.07",
  "shard": 3,
  "primary": false
}
```

```json
{
  "index": "logs-2024.07",
  "shard": 3,
  "primary": false,
  "current_state": "unassigned",
  "unassigned_info": {
    "reason": "NODE_LEFT",
    "at": "2024-07-15T12:30:00.000Z",
    "last_allocation_status": "no_attempt"
  },
  "can_allocate": "no",
  "allocate_explanation": "cannot allocate because allocation is not permitted to any of the nodes",
  "node_allocation_decisions": [
    {
      "node_id": "xB2r...kQ4",
      "node_name": "data-h1",
      "node_decision": "no",
      "deciders": [
        {
          "decider": "same_shard",
          "decision": "NO",
          "explanation": "a copy of this shard is already allocated to this node"
        }
      ]
    }
  ]
}
```

In the example above, the replica cannot be allocated because the only available data node
already hosts the primary copy of the same shard. The fix is to add another data node or
reduce the replica count.

**Common fixes for unassigned shards:**

1. **Add more data nodes** if the cluster lacks capacity to host all replicas.

2. **Reduce replica count** if the cluster intentionally has fewer nodes than replicas require:

```json
PUT /logs-2024.07/_settings
{
  "index": {
    "number_of_replicas": 0
  }
}
```

3. **Re-enable allocation** if it was previously disabled during maintenance:

```json
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "all"
  }
}
```

4. **Clear the read-only block** if an index was locked due to flood-stage disk usage:

```json
PUT /logs-2024.07/_settings
{
  "index.blocks.read_only_allow_delete": null
}
```

5. **Retry failed allocations** to force the cluster to reattempt shard assignment:

```json
POST /_cluster/reroute?retry_failed=true
```

```json
{
  "acknowledged": true,
  "state": {
    "cluster_uuid": "abc123...",
    "routing_nodes": {}
  }
}
```

6. **Manually allocate an unassigned primary** as a last resort. This can cause data loss if
the shard data on disk is corrupt or unavailable:

```json
POST /_cluster/reroute
{
  "commands": [
    {
      "allocate_stale_primary": {
        "index": "logs-2024.07",
        "shard": 3,
        "node": "data-h1",
        "accept_data_loss": true
      }
    }
  ]
}
```

---

### Command Reference

| Operation | REST Verb & Endpoint | Key Parameters |
|-----------|----------------------|----------------|
| Cluster health | `GET /_cluster/health` | `level` (cluster, indices, shards), `wait_for_status`, `timeout` |
| Cluster stats | `GET /_cluster/stats` | Aggregate view of indices, nodes, and resource usage |
| Cluster state | `GET /_cluster/state` | `filter_path`, `metric` (nodes, routing_table, metadata) |
| Cluster settings (get) | `GET /_cluster/settings` | `include_defaults`, `flat_settings` |
| Cluster settings (put) | `PUT /_cluster/settings` | `persistent`, `transient` |
| Node info | `GET /_nodes/info` | Filter by node ID or category (`jvm`, `os`, `plugins`) |
| Node stats | `GET /_nodes/stats` | Filter by node ID or metric (`jvm`, `fs`, `thread_pool`) |
| Node hot threads | `GET /_nodes/hot_threads` | `threads`, `timeout`, `type` (cpu, wait, block) |
| Allocation explain | `GET /_cluster/allocation/explain` | `index`, `shard`, `primary` |
| Reroute | `POST /_cluster/reroute` | `retry_failed`, `commands` (allocate, move, cancel) |
| Voting exclusions | `POST /_cluster/voting_config_exclusions` | `node_names`, `node_ids` |
| Cat health | `GET /_cat/health?v` | Single-line cluster health |
| Cat nodes | `GET /_cat/nodes?v` | `h` (column selection), `s` (sort) |
| Cat indices | `GET /_cat/indices?v` | `h`, `s`, `health`, `expand_wildcards` |
| Cat shards | `GET /_cat/shards?v` | `h`, `s`, filter by index name |
| Cat allocation | `GET /_cat/allocation?v` | Disk and shard count per node |
| Cat recovery | `GET /_cat/recovery?v` | `active_only`, shard recovery progress |
| Cat pending tasks | `GET /_cat/pending_tasks?v` | Pending cluster state updates |
| Cat thread pool | `GET /_cat/thread_pool?v` | `h` (name, active, rejected, queue, completed) |
| Flush | `POST /_flush` | Flush all indices before node shutdown |
| Disable allocation | `PUT /_cluster/settings` | `cluster.routing.allocation.enable: primaries` |
| Enable allocation | `PUT /_cluster/settings` | `cluster.routing.allocation.enable: all` |
| Clear read-only block | `PUT /<index>/_settings` | `index.blocks.read_only_allow_delete: null` |
