## Distributed Index Architecture

An Elasticsearch index is not stored on a single machine. It is split into smaller
units called **shards**, and those shards are distributed across the nodes that form
a cluster. This architecture is the foundation of Elasticsearch's ability to scale
horizontally, tolerate hardware failures, and execute searches in parallel.

```
  +-------------------+    +-------------------+    +-------------------+
  |      Node 1       |    |      Node 2       |    |      Node 3       |
  |   (data + master) |    |     (data only)    |    |     (data only)    |
  |                   |    |                   |    |                   |
  |  +-----+ +-----+ |    |  +-----+ +-----+ |    |  +-----+ +-----+ |
  |  | P0  | | R2  | |    |  | P1  | | R0  | |    |  | P2  | | R1  | |
  |  |docs | |copy | |    |  |docs | |copy | |    |  |docs | |copy | |
  |  |0-99 | |200+ | |    |  |100- | |0-99 | |    |  |200+ | |100- | |
  |  +-----+ +-----+ |    |  | 199 | |     | |    |  |     | | 199 | |
  |                   |    |  +-----+ +-----+ |    |  +-----+ +-----+ |
  |  +-----+         |    |                   |    |                   |
  |  | R1  |         |    |  +-----+         |    |  +-----+         |
  |  |copy |         |    |  | R2  |         |    |  | R0  |         |
  |  |100- |         |    |  |copy |         |    |  |copy |         |
  |  | 199 |         |    |  |200+ |         |    |  |0-99 |         |
  |  +-----+         |    |  +-----+         |    |  +-----+         |
  +-------------------+    +-------------------+    +-------------------+
          |                        |                        |
          +------------------------+------------------------+
                                   |
                    +--------------+--------------+
                    |  "my_index"  (3P + 3R = 6)  |
                    |  number_of_shards: 3         |
                    |  number_of_replicas: 1        |
                    +--------------+--------------+
```

**Key:** P0, P1, P2 = primary shards; R0, R1, R2 = replica shards. Each primary
shard holds a unique slice of the documents. Each replica is a full copy of its
corresponding primary, placed on a *different* node for fault tolerance.

---

### Index Shared Across Nodes

In a distributed Elasticsearch cluster, an **index** is divided into **shards** to
allow horizontal scaling and improve redundancy and performance. Each shard contains
part of the data in the index, and the shards are distributed across multiple
**nodes**.

When you create an index, you specify two key settings:

- **`number_of_shards`** -- how many primary shards the index will have (set once at
  creation time; cannot be changed without reindexing).
- **`number_of_replicas`** -- how many replica copies of each primary shard to
  maintain (can be changed at any time).

The total number of shard copies in the cluster is therefore
`number_of_shards * (1 + number_of_replicas)`. In the diagram above, with 3
primaries and 1 replica each, there are **6 shard copies** spread across 3 nodes.

---

### How an Index Is Split Into Shards

When a document is indexed, Elasticsearch uses a routing formula to determine which
primary shard will hold the document:

```
shard_number = hash(_routing) % number_of_primary_shards
```

By default, `_routing` equals the document `_id`. This ensures a deterministic,
even distribution of documents across shards.

```
             +-----------------------------+
             |   Incoming Document (id=42) |
             +-------------+---------------+
                           |
                    hash(42) % 3 = 0
                           |
                           v
    +----------+     +----------+     +----------+
    | Shard P0 |     | Shard P1 |     | Shard P2 |
    |  doc 42  |     |          |     |          |
    |  doc 3   |     |  doc 19  |     |  doc 8   |
    |  doc 57  |     |  doc 22  |     |  doc 11  |
    |  doc 78  |     |  doc 31  |     |  doc 14  |
    |  ...     |     |  ...     |     |  ...     |
    +----------+     +----------+     +----------+
         |                |                |
         v                v                v
      Node 1           Node 2           Node 3
```

Because the number of primary shards is baked into this formula, it **cannot be
changed** after index creation without a full reindex. Choosing the right shard
count up front is therefore critical (see *Shard Sizing Best Practices* below).

---

### Primary vs Replica Shards

Every shard in an Elasticsearch index is either a **primary** or a **replica**.

| Aspect              | Primary Shard                          | Replica Shard                              |
|---------------------|----------------------------------------|--------------------------------------------|
| Purpose             | Holds the authoritative copy of data   | Holds a redundant copy for failover        |
| Writes              | Receives all index/delete operations   | Receives replicated ops from the primary   |
| Reads               | Can serve search requests              | Can also serve search requests             |
| Placement rule      | Spread across nodes by the allocator   | Never on the same node as its primary      |
| Promotion           | N/A                                    | Promoted to primary if the original fails  |
| Count change        | Fixed at index creation                | Can be increased or decreased at any time  |

```
  Primary/Replica Relationship
  ============================

     +--------+          replication          +--------+
     |  P0    |  -------- write ops --------> |  R0    |
     | Node 1 |                               | Node 2 |
     +--------+                               +--------+
                                                  |
                                          (also on Node 3
                                           if replicas = 2)
                                                  |
                                              +--------+
                                              |  R0'   |
                                              | Node 3 |
                                              +--------+
```

Replica shards serve two purposes: **high availability** (data survives node loss)
and **read throughput** (searches can be served by any copy of a shard).

---

### Shard Allocation and Rebalancing

The **master node** runs the **shard allocator**, which decides where each shard
copy lives. The allocator considers several factors:

1. **Allocation awareness** -- cluster-level settings such as
   `cluster.routing.allocation.awareness.attributes` let you ensure shards spread
   across racks or availability zones.
2. **Disk-based thresholds** -- Elasticsearch will not allocate shards to a node
   whose disk usage exceeds `cluster.routing.allocation.disk.watermark.low` (default
   85%). At `high` (90%) it starts relocating shards away, and at `flood_stage`
   (95%) it sets indices to read-only.
3. **Shard count balance** -- the allocator tries to keep roughly the same number of
   shards per node.
4. **Same-shard rule** -- a replica is never placed on the same node as its primary.
5. **Filter rules** -- you can include/exclude specific nodes for specific indices
   using `index.routing.allocation.include/exclude/require`.

**Rebalancing** happens automatically when the cluster topology changes (a node
joins, leaves, or a new index is created). The master computes the desired state and
issues shard move commands to data nodes.

```
  Before Rebalance (Node 4 joins)       After Rebalance
  ===================================   ==================================
  Node1: P0 R2 R1                       Node1: P0 R2
  Node2: P1 R0 R2                       Node2: P1 R0
  Node3: P2 R1 R0                       Node3: P2 R1
  Node4: (empty)                         Node4: R0 R1 R2
                                          ^ shards migrated to balance
```

You can observe allocation decisions using:
```
GET _cluster/allocation/explain
```

---

### Segment Lifecycle

Each Lucene shard is composed of **segments** -- immutable on-disk data structures.

#### Create, Search, Merge, Delete

```
  Time --------------------------------------------------------->

  t0  Write batch 1          t1  Write batch 2         t2  Write batch 3
       |                          |                         |
       v                          v                         v
  +---------+               +---------+               +---------+
  | Seg A   |               | Seg B   |               | Seg C   |
  | 500 docs|               | 500 docs|               | 500 docs|
  +---------+               +---------+               +---------+
       |                          |                         |
       |      t3  Merge triggered |                         |
       +----------+---------------+                         |
                  |                                         |
                  v                                         |
            +------------+                                  |
            | Seg AB     |                                  |
            | 1000 docs  |                                  |
            +------------+                                  |
                  |            t4  Merge triggered           |
                  +-------------------+---------------------+
                                      |
                                      v
                                +-----------+
                                | Seg ABC   |
                                | 1500 docs |
                                +-----------+

  (Seg A, Seg B, Seg C are deleted after their merged
   segment is fully committed.)
```

**Lifecycle stages of a segment:**

1. **In-memory buffer** -- new documents are first written to an in-memory indexing
   buffer and the translog.
2. **Refresh (default every 1 s)** -- the buffer is written as a new segment that is
   *searchable* but not yet fsync'd to disk.
3. **Flush / Commit** -- the translog is replayed and segments are fsync'd. The
   translog is then truncated.
4. **Merge** -- background thread combines small segments into larger ones, purging
   deleted documents in the process.
5. **Delete** -- source segments are removed from disk once the merge completes.

Because segments are immutable, a "delete" operation merely marks a document as
deleted in a `.del` bitset. The document is physically removed only during the next
merge that includes that segment.

---

### Segments and Merging (Detailed)

Segment merging is governed by a **merge policy** (default: `tiered`). The tiered
merge policy works as follows:

- Segments smaller than `floor_segment` (default 2 MB) are always eligible.
- The policy finds the "least-cost" set of adjacent-size segments to merge.
- It will not merge segments larger than `max_merged_segment` (default 5 GB).
- The `max_merge_at_once` setting (default 10) limits how many segments are merged
  in a single pass.

```
  Tiered Merge Example
  ====================

  Level 0:  [2MB] [2MB] [2MB] [2MB] [2MB]    <-- 5 tiny segments
                    |  merge  |
                    v
  Level 1:         [10MB]            [2MB]    <-- one merged + one leftover
                              \     /
                               merge
                                |
  Level 2:                   [12MB]           <-- final segment
```

You can tune merge throttling with `indices.store.throttle.max_bytes_per_sec` to
limit I/O impact during heavy merges.

---

### The Write Path

When a client sends an index request, the document travels through several stages
before it is durable and searchable.

```
  Client
    |
    |  (1) Index request (PUT /my_index/_doc/42)
    v
  +---------------------+
  | Coordinating Node   |   (any node can coordinate)
  +---------------------+
    |
    |  (2) Route: hash(42) % 3 = 0  -->  primary shard P0
    v
  +---------------------+
  | Node 1 -- Primary P0|
  |                     |
  |  a. Validate doc    |
  |  b. Write to        |
  |     translog        |
  |  c. Index into      |
  |     in-memory buf   |
  +---------------------+
    |
    |  (3) Replicate in parallel to all in-sync replica copies
    |
    +---------------------------+---------------------------+
    |                           |                           |
    v                           v                           v
  +-----------+           +-----------+             (more replicas
  | Node 2    |           | Node 3    |              if configured)
  | Replica R0|           | Replica R0|
  | (copy 1)  |           | (copy 2)  |
  +-----------+           +-----------+
    |                           |
    |  (4) Each replica writes  |
    |      translog + buffer    |
    |                           |
    +---------------------------+
                |
                v
  (5) Once ALL in-sync replicas ACK --> coordinating node responds 200 OK
```

**Important write-path details:**

- The primary validates the document (mapping checks, field limits) before writing.
- The **translog** (write-ahead log) provides durability: even if the node crashes
  before a flush, the translog is replayed on recovery.
- By default, the translog is fsync'd after every request
  (`index.translog.durability: request`). Setting this to `async` improves
  throughput at the cost of potential data loss.
- A **refresh** (default every 1 second) makes the in-memory buffer searchable as a
  new segment. Until the refresh, newly indexed documents are *not* visible to
  searches (this is the "near real-time" aspect of ES).

---

### The Read Path

When a client sends a search request, Elasticsearch uses a **scatter-gather**
pattern across all shards of the target index.

```
  Client
    |
    |  (1) Search request (GET /my_index/_search?q=...)
    v
  +---------------------+
  | Coordinating Node   |
  +---------------------+
    |
    |  (2) Scatter: send query to one copy of each shard
    |      (primary OR replica -- chosen by adaptive replica selection)
    |
    +------------+-----------+-----------+
    |            |           |           |
    v            v           v           |
  +------+   +------+   +------+        |
  |Shard0|   |Shard1|   |Shard2|        |
  |(P0 or|   |(P1 or|   |(P2 or|        |
  | R0)  |   | R1)  |   | R2)  |        |
  +------+   +------+   +------+        |
    |            |           |           |
    |  local     |  local    |  local    |
    |  search    |  search   |  search   |
    |            |           |           |
    v            v           v           |
  +------+   +------+   +------+        |
  |top N |   |top N |   |top N |        |
  |doc ID|   |doc ID|   |doc ID|        |
  |+score|   |+score|   |+score|        |
  +------+   +------+   +------+        |
    |            |           |           |
    +------------+-----------+           |
                 |                       |
                 v                       |
    (3) Gather: coordinating node merges |
        partial results, picks global   |
        top N doc IDs                   |
                 |                       |
    (4) Fetch phase: retrieve full       |
        documents from relevant shards   |
                 |                       |
                 v                       |
    (5) Return final results to client   |
         <-------------------------------+
```

**Two-phase execution:**

| Phase   | What happens                                                    |
|---------|-----------------------------------------------------------------|
| Query   | Each shard executes the query locally, returning only doc IDs   |
|         | and scores (lightweight).                                       |
| Fetch   | The coordinating node asks the relevant shards for the full     |
|         | document bodies of the top-N results.                           |

**Adaptive replica selection (ARS):** Instead of round-robin, ES routes shard
requests to the copy with the lowest estimated response time, based on queue size,
service time, and response time statistics.

---

### Node Failure and Failover

When a node leaves the cluster (crash, network partition, or maintenance), the
master node detects the loss and promotes replicas to primaries as needed.

```
  ==================== BEFORE FAILURE ====================

  Node 1              Node 2              Node 3
  +--------+          +--------+          +--------+
  | P0  R2 |          | P1  R0 |          | P2  R1 |
  +--------+          +--------+          +--------+

  Cluster health: GREEN  (all primaries + replicas assigned)

  ==================== NODE 2 FAILS ======================

  Node 1              Node 2              Node 3
  +--------+          +--------+          +--------+
  | P0  R2 |          |  XXXX  |          | P2  R1 |
  +--------+          +--------+          +--------+
                       (offline)

  Lost: P1, R0
  Master promotes R1 on Node 3 --> new P1  (was a replica, now primary)

  ==================== AFTER PROMOTION ===================

  Node 1              Node 3
  +--------+          +--------+
  | P0  R2 |          | P2  P1 |
  +--------+          +--------+

  Cluster health: YELLOW  (all primaries assigned, but some replicas missing)
  Missing: R0 (copy of P0), R1 (copy of P1), R2 (copy of P2)

  ==================== RECOVERY (Node 2 rejoins or Node 4 added) ====

  Node 1              Node 3              Node 4
  +--------+          +--------+          +--------+
  | P0  R2 |          | P2  P1 |          | R0  R1 |
  +--------+          +--------+          +--------+

  Cluster health: GREEN  (all primaries + replicas assigned again)
```

**Health indicators:**

| Colour | Meaning                                                       |
|--------|---------------------------------------------------------------|
| GREEN  | All primary and replica shards are assigned.                  |
| YELLOW | All primaries assigned, but at least one replica is missing.  |
| RED    | At least one primary shard is unassigned -- data loss risk.   |

---

### Split Brain Problem

A **split brain** occurs when a cluster partitions into two or more sub-clusters,
each electing its own master and accepting writes independently. When the partition
heals, conflicting writes cannot be reconciled and data is lost.

```
  Normal Cluster                    Split-Brain Scenario
  ================                  =====================

  +------+------+------+           +------+------+  |  +------+
  |Node1 |Node2 |Node3 |           |Node1 |Node2 |  |  |Node3 |
  |master|      |      |           |master|      |  |  |master|
  +------+------+------+           +------+------+  |  +------+
                                    sub-cluster A   NET  sub-cluster B
                                                   SPLIT
        Writes go to one            Writes -> A     |  Writes -> B
        master only                 (conflict!)     |  (conflict!)
```

**How Elasticsearch prevents split brain (ES 7+):**

In Elasticsearch 7.0 and later, the old `discovery.zen.minimum_master_nodes` setting
was replaced by an automatic **voting configuration** mechanism:

- The cluster automatically tracks which nodes are eligible to vote in master
  elections.
- A master is only elected if it receives votes from a **quorum** (majority) of the
  voting configuration: `quorum = (voting_nodes / 2) + 1`.
- In a 3-master-eligible-node cluster, quorum = 2. A single isolated node **cannot**
  elect itself master.

**Best practices to avoid split brain:**

- Use an **odd number** of master-eligible nodes (3 or 5 is typical).
- In production, dedicate master-eligible nodes (set `node.data: false`) so they are
  not burdened by heavy indexing or search loads.
- Use `cluster.initial_master_nodes` only during the very first bootstrap of a new
  cluster; remove it afterward.

---

### Horizontal Scaling and Performance

By distributing shards across multiple nodes, Elasticsearch not only expands data
storage capabilities but also enhances **query execution efficiency**. This
architecture provides high availability and fault tolerance. When a node fails,
Elasticsearch can still access data from other nodes, ensuring continuity and
minimizing downtime.

```
  Scaling from 3 to 5 nodes (index: 5 primaries, 1 replica)
  ==========================================================

  3 Nodes:                              5 Nodes (after rebalance):
  +-------+  +-------+  +-------+      +-----+ +-----+ +-----+ +-----+ +-----+
  |P0 P1  |  |P2 P3  |  |P4 R0  |      | P0  | | P1  | | P2  | | P3  | | P4  |
  |R3 R4  |  |R0 R1  |  |R2 R3  |      | R3  | | R4  | | R0  | | R1  | | R2  |
  +-------+  +-------+  +-------+      +-----+ +-----+ +-----+ +-----+ +-----+
  ~3-4 shards/node                       2 shards/node  <-- less load per node
```

As nodes are added, the allocator automatically **rebalances** shards so each node
carries a fair share. More nodes means:

- **More disk capacity** -- the index data is spread over more disks.
- **More CPU / RAM** -- searches are parallelized across more hardware.
- **Better fault tolerance** -- losing one of five nodes is less impactful than
  losing one of three.

---

### Shard Sizing Best Practices

Choosing the right number and size of shards is one of the most important capacity
planning decisions in Elasticsearch.

| Guideline                          | Recommendation                              |
|------------------------------------|---------------------------------------------|
| Target shard size                  | 10 GB - 50 GB per shard (sweet spot ~30 GB) |
| Maximum shard size                 | Do not exceed 50-65 GB                      |
| Shards per node                    | Keep below ~20 shards per GB of heap         |
| Heap per node                      | Do not exceed 50% of RAM (max 31 GB)        |
| Shards per index                   | Start with 1 shard per index unless data     |
|                                    | exceeds 50 GB or you need write parallelism  |
| Replicas                           | At least 1 in production for fault tolerance |
| Over-sharding                      | Avoid: too many small shards waste memory    |
|                                    | (each shard has fixed overhead ~5-10 MB)     |

**Rule of thumb:** total data size / 30 GB = approximate number of primary shards.

**Example:** 150 GB of data --> 5 primary shards x 30 GB each.

**Time-series indices (e.g., logs):**

- Use **index lifecycle management (ILM)** to rollover indices when they hit a size
  or age threshold.
- A common pattern: 1 primary shard per rollover index, rollover at 30 GB or 1 day.
- Use **data tiers** (hot --> warm --> cold --> frozen) to move older indices to
  cheaper hardware.

---

### Replica Shards (Expanded)

To prevent data loss and increase fault tolerance, Elasticsearch maintains **replica
shards** -- full copies of primary shards placed on different nodes. Replicas serve
two critical functions:

1. **Failover** -- if a primary is lost, a replica is promoted instantly.
2. **Search throughput** -- read requests can be served by any shard copy, so more
   replicas means more search parallelism.

```
  Replica Configurations
  ======================

  replicas=0            replicas=1               replicas=2
  (no redundancy)       (standard)               (high availability)

  Node1: P0 P1 P2      Node1: P0  R1  R2        Node1: P0  R1  R2
                        Node2: P1  R2  R0        Node2: P1  R2  R0
                        Node3: P2  R0  R1        Node3: P2  R0  R1
                                                 Node4: R0' R1' R2'
  Survives: 0 node      Survives: 1 node          Survives: 2 nodes
  failures              failure                   failures
```

You can dynamically adjust the replica count without downtime:

```json
PUT /my_index/_settings
{
  "index": {
    "number_of_replicas": 2
  }
}
```

---

### Command Reference: Shard and Index Management

| Command                                          | Description                                      |
|--------------------------------------------------|--------------------------------------------------|
| `GET _cat/indices?v`                             | List all indices with health, doc count, size     |
| `GET _cat/shards?v`                              | List all shards, their node, state, size          |
| `GET _cat/shards/my_index?v`                     | Shards for a specific index                       |
| `GET _cat/allocation?v`                          | Disk usage and shard count per node               |
| `GET _cat/nodes?v`                               | Node list with roles, heap, CPU                   |
| `GET _cluster/health`                            | Cluster status (green/yellow/red)                 |
| `GET _cluster/health/my_index`                   | Health for a specific index                       |
| `GET _cluster/allocation/explain`                | Why a shard is unassigned or placed where it is   |
| `GET _cluster/settings?include_defaults=true`    | All cluster-level settings                        |
| `PUT /my_index`                                  | Create an index (with settings/mappings in body)  |
| `DELETE /my_index`                               | Delete an index                                   |
| `POST /my_index/_close`                          | Close an index (releases resources)               |
| `POST /my_index/_open`                           | Open a previously closed index                    |
| `PUT /my_index/_settings`                        | Update dynamic index settings (e.g., replicas)    |
| `POST /my_index/_forcemerge?max_num_segments=1`  | Force merge segments (use off-peak)               |
| `POST /my_index/_refresh`                        | Make recent writes searchable immediately          |
| `POST /my_index/_flush`                          | Flush translog to Lucene commit                   |
| `POST /_cluster/reroute`                         | Manually move, allocate, or cancel shard actions  |
| `PUT _cluster/settings`                          | Set transient/persistent cluster-level settings   |
| `GET _cat/recovery?v`                            | Monitor ongoing shard recoveries                  |
| `POST /my_index/_split/my_index_split`           | Split an index into more primary shards           |
| `POST /my_index/_shrink/my_index_shrink`         | Shrink an index into fewer primary shards         |
| `POST _reindex`                                  | Copy documents from one index to another          |

**Useful `_cat` parameters:** `?v` (headers), `?help` (column descriptions),
`?s=column:desc` (sort), `?format=json` (JSON output).

---

### Summary

Elasticsearch's distributed index architecture rests on a few core ideas:

1. **Sharding** splits data so it can live across many machines.
2. **Replication** copies each shard so hardware failures do not lose data.
3. **Segments** provide immutable, search-optimized storage within each shard.
4. **Coordinated routing** ensures writes land on the correct primary and reads
   scatter across all available copies.
5. **Automatic allocation and rebalancing** keep the cluster healthy as nodes come
   and go.

Understanding these mechanisms is essential for capacity planning, performance
tuning, and troubleshooting in any production Elasticsearch deployment.
