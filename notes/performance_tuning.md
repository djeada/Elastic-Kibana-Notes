## Performance Tuning

Performance tuning in Elasticsearch is the systematic practice of identifying and eliminating
bottlenecks across every layer of the stack — from hardware and JVM configuration through index
design and query optimization — to achieve the best possible balance between search latency,
indexing throughput, resource utilization, and cluster reliability. Unlike tuning a single-server
database where the knobs are relatively few, Elasticsearch is a distributed system with dozens of
interacting components: Lucene segments, JVM garbage collectors, OS page caches, network buffers,
thread pools, shard allocation heuristics, and caching layers all influence end-to-end performance.

A well-tuned cluster can handle millions of documents per second during ingestion while
simultaneously serving sub-second search queries across billions of documents. A poorly tuned
cluster — even on identical hardware — may struggle with timeouts, circuit breaker exceptions, and
cascading shard failures. The difference almost always comes down to understanding how data flows
through the system and where each layer can be optimized without destabilizing another.

This document provides a comprehensive guide to Elasticsearch performance tuning, covering
hardware selection, JVM configuration, indexing and search optimization, shard sizing, segment
management, caching, slow log analysis, monitoring, snapshots, index lifecycle management, and
common anti-patterns. Each section includes practical API examples, expected outputs, and
real-world guidance drawn from production deployments.


### Performance Factors

The following diagram illustrates the layers that influence Elasticsearch performance. Each layer
depends on the one below it, and a bottleneck at any level constrains the entire system:

```
  Performance Factor Stack
  -------------------------

  +-------------------------------------------------------------------+
  |                          RESULTS                                  |
  |              (Latency, Throughput, Availability)                   |
  +-------------------------------------------------------------------+
       ^                    ^                    ^
       |                    |                    |
  +----+------+   +---------+---------+   +------+------+
  |  Query    |   |   Index Design    |   |  Caching &  |
  |  Design   |   |                   |   |  Segments   |
  |           |   | Shard count,      |   |             |
  | Filters,  |   | mapping types,    |   | Node cache, |
  | pagination|   | refresh interval, |   | shard cache, |
  | _source   |   | bulk sizing       |   | force merge |
  +-----------+   +-------------------+   +-------------+
       ^                    ^                    ^
       |                    |                    |
       +----------+---------+--------------------+
                  |
  +---------------+---------------+
  |        JVM Settings           |
  |                               |
  |  Heap size, GC algorithm,     |
  |  memory lock, thread pools    |
  +---------------+---------------+
                  |
  +---------------+---------------+
  |          Hardware              |
  |                               |
  |  +-------+ +-------+ +-----+ |
  |  |  CPU  | |  RAM  | | Disk| |
  |  |       | |       | |     | |
  |  |Cores, | |Size,  | |SSD, | |
  |  |speed  | |speed  | |IOPS | |
  |  +-------+ +-------+ +-----+ |
  |                               |
  |  +-------+                    |
  |  |Network|                    |
  |  |       |                    |
  |  |Band-  |                    |
  |  |width  |                    |
  |  +-------+                    |
  +-------------------------------+
```

Tuning should proceed bottom-up: no amount of query optimization will compensate for insufficient
RAM, and the best hardware in the world cannot rescue a poorly designed mapping. Start with the
foundation and work upward.

---

### Hardware Considerations

**Disk: SSD vs HDD**

Disk I/O is the single most impactful hardware factor for Elasticsearch. Lucene reads and writes
segments on disk constantly — during indexing, merging, and search (when data is not in the OS
page cache). SSDs provide 10–100x the IOPS of spinning disks, which translates directly into
faster indexing, faster segment merges, and lower search latency.

For hot data nodes that receive active writes and serve real-time queries, NVMe SSDs are strongly
recommended. Warm and cold tiers that serve infrequent queries on historical data can use cheaper
SATA SSDs or even HDDs, as long as the latency trade-off is acceptable.

Avoid network-attached storage (NAS/SAN) for data nodes. The added network hop introduces
unpredictable latency spikes that destabilize shard allocation and can trigger master node
timeouts.

**Memory**

Elasticsearch uses memory in two distinct ways:

1. **JVM heap** — used for internal data structures, field data, shard-level caches, cluster
   state, and coordinating node operations (merging partial results).
2. **OS page cache** — the operating system caches frequently accessed Lucene segment files in
   remaining memory. This is often more important than heap for search performance.

The general rule is to give Elasticsearch **no more than 50% of available RAM** as JVM heap, and
leave the rest for the OS page cache. The heap should **never exceed 31 GB** because beyond that
threshold the JVM can no longer use compressed ordinary object pointers (compressed oops), which
wastes ~40% of the heap space. On a 64 GB machine, set the heap to 30–31 GB and leave 33–34 GB
for page cache.

```
  Memory Allocation on a 64 GB Node
  -----------------------------------

  +-------------------------------+-------------------------------+
  |        JVM Heap (30 GB)       |     OS Page Cache (34 GB)     |
  |                               |                               |
  |  Field data, node caches,     |  Lucene segment files,        |
  |  aggregation buckets,         |  transaction logs,            |
  |  cluster state, indexing      |  frequently accessed data     |
  |  buffers, query coordination  |  served directly from memory  |
  +-------------------------------+-------------------------------+
  0 GB                           30 GB                          64 GB
```

**CPU**

Elasticsearch benefits from multiple cores rather than high clock speed. Indexing, merging, and
search all execute in parallel across thread pools sized to the number of available processors.
For data nodes handling heavy aggregation workloads, 16–32 cores per node is typical. Master
nodes need fewer cores (4–8) but require consistently low latency.

**Network**

In a distributed system, every search fans out to multiple shards and every indexing request may
be forwarded to the correct primary shard on a different node. A 10 Gbps network is the minimum
recommendation for production clusters. Cross-datacenter replication (CCR) workloads require
careful bandwidth planning.

---

### JVM and Heap Configuration

Elasticsearch runs on the JVM, and incorrect JVM configuration is one of the most common sources
of instability in production clusters. The two critical settings are heap size and memory locking.

**Setting Heap Size**

Heap size is configured in `jvm.options` (or via `ES_JAVA_OPTS` environment variable). The
minimum and maximum should always be set to the same value to prevent the JVM from resizing the
heap at runtime, which causes long GC pauses:

```
  # /etc/elasticsearch/jvm.options
  -Xms30g
  -Xmx30g
```

Key rules:
- Set `-Xms` and `-Xmx` to the same value
- Never exceed 50% of physical RAM
- Never exceed 31 GB (compressed oops threshold)
- On machines with less than 8 GB RAM, use 50% of RAM

**Avoiding Swapping**

If the JVM heap is swapped to disk, GC pauses can stretch from milliseconds to minutes, causing
the node to appear dead to the rest of the cluster. There are three layers of protection:

1. **Disable swap entirely** at the OS level:
   ```
   sudo swapoff -a
   ```

2. **Lock JVM memory** using `bootstrap.memory_lock`:

```json
PUT /_cluster/settings
{
  "persistent": {
    "bootstrap.memory_lock": true
  }
}
```

3. **Verify memory lock** is active:

```json
GET /_nodes?filter_path=**.mlockall
```

```json
{
  "nodes": {
    "node_id_1": {
      "process": {
        "mlockall": true
      }
    }
  }
}
```

**Garbage Collection**

Elasticsearch ships with the G1GC garbage collector by default (since version 7.x). For most
workloads, the defaults are appropriate. Monitor GC behavior with:

```json
GET /_nodes/stats/jvm?filter_path=**.gc
```

```json
{
  "nodes": {
    "node_id_1": {
      "jvm": {
        "gc": {
          "collectors": {
            "young": {
              "collection_count": 15420,
              "collection_time_in_millis": 82340
            },
            "old": {
              "collection_count": 12,
              "collection_time_in_millis": 4230
            }
          }
        }
      }
    }
  }
}
```

If old GC collections are frequent (more than a few per hour) or long (more than a few seconds
each), the heap is likely too small or field data usage is too high.

---

### Indexing Performance

Indexing throughput — the number of documents per second that Elasticsearch can ingest — is often
the first bottleneck encountered as data volumes grow. The following settings and strategies can
increase indexing speed by 2–10x depending on the workload.

**Refresh Interval**

By default, Elasticsearch refreshes each index every 1 second, making newly indexed documents
visible to search. Each refresh creates a new Lucene segment, which consumes file handles and
triggers later segment merges. During heavy bulk ingestion, setting a longer refresh interval (or
disabling it entirely) dramatically reduces overhead:

```json
PUT /logs-2024.07/_settings
{
  "index": {
    "refresh_interval": "-1"
  }
}
```

After bulk ingestion completes, re-enable and manually refresh:

```json
PUT /logs-2024.07/_settings
{
  "index": {
    "refresh_interval": "1s"
  }
}
```

```json
POST /logs-2024.07/_refresh
```

**Bulk Request Sizing**

The `_bulk` API is the primary high-throughput ingestion path. Optimal batch size depends on
document size and cluster resources, but the following guidelines apply:

```
  Bulk Request Sizing Guidelines
  --------------------------------

  +------------------+-------------------+-------------------------------+
  | Document Size    | Recommended Batch | Explanation                   |
  +------------------+-------------------+-------------------------------+
  | < 1 KB           | 5,000 - 10,000    | Small docs; larger batches    |
  |                  | documents         | amortize HTTP overhead        |
  +------------------+-------------------+-------------------------------+
  | 1 KB - 10 KB     | 1,000 - 5,000     | Most common sweet spot for    |
  |                  | documents         | log and metric data           |
  +------------------+-------------------+-------------------------------+
  | 10 KB - 100 KB   | 500 - 1,000       | Larger docs fill network      |
  |                  | documents         | buffers faster                |
  +------------------+-------------------+-------------------------------+
  | > 100 KB         | 100 - 500         | Very large docs; avoid        |
  |                  | documents         | exceeding 100 MB per request  |
  +------------------+-------------------+-------------------------------+
```

The overall bulk request payload should stay between 5 MB and 15 MB. Going above 100 MB risks
overwhelming the node's network buffers and indexing thread pool.

**Translog Settings**

Every index operation is written to the transaction log (translog) before being applied to the
Lucene index. The translog provides crash recovery. By default, Elasticsearch fsyncs the translog
after every operation (`request` durability), which is safe but slow. For workloads that can
tolerate a small window of data loss, switching to `async` durability and increasing the flush
threshold can significantly improve performance:

```json
PUT /logs-2024.07/_settings
{
  "index": {
    "translog.durability": "async",
    "translog.sync_interval": "30s",
    "translog.flush_threshold_size": "1gb"
  }
}
```

With `async` durability, the translog is fsynced every `sync_interval` rather than on every
request. If the node crashes between fsyncs, up to `sync_interval` seconds of data may be lost.

**Auto-Generated IDs vs Client-Specified IDs**

When you index a document without specifying an `_id`, Elasticsearch generates a unique ID using
a time-based UUID scheme. This is faster than client-specified IDs because Elasticsearch can skip
the version lookup that checks whether the document already exists:

```json
// Faster: auto-generated ID
POST /logs-2024.07/_doc
{
  "message": "Connection established",
  "timestamp": "2024-07-15T10:30:00Z"
}

// Slower: client-specified ID (requires version lookup)
PUT /logs-2024.07/_doc/event-001
{
  "message": "Connection established",
  "timestamp": "2024-07-15T10:30:00Z"
}
```

Use auto-generated IDs for append-only workloads (logs, metrics, events). Use client-specified
IDs only when you need upsert/deduplication semantics.

**Indexing Buffer Size**

The indexing buffer holds newly indexed documents in memory before they are written to a segment
on disk. The default is 10% of the JVM heap, shared across all shards on the node. For
write-heavy nodes, increasing this can reduce the frequency of segment flushes:

```json
PUT /_cluster/settings
{
  "persistent": {
    "indices.memory.index_buffer_size": "20%"
  }
}
```

---

### Search Performance

Search latency is what users experience directly. Even small improvements — shaving 50ms off a
100ms query — can have a measurable impact on user engagement and conversion rates. The
following techniques address the most common search performance issues.

**Query Context vs Filter Context**

This is the single most impactful search optimization. Queries in **filter context** do not
calculate relevance scores, which means Elasticsearch can:
1. Skip the expensive TF-IDF / BM25 scoring step
2. Cache the resulting bitset for reuse across subsequent queries

```json
// Inefficient: scoring a term that does not need ranking
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "laptop" } },
        { "term": { "status": "active" } }
      ]
    }
  }
}

// Optimized: move non-scoring clauses to filter context
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "laptop" } }
      ],
      "filter": [
        { "term": { "status": "active" } }
      ]
    }
  }
}
```

**Avoiding Expensive Queries**

Certain query types are inherently expensive and should be avoided or constrained:

```
  Expensive Query Types
  ----------------------

  +------------------------+-----------------------------------------------+
  | Query Type             | Why It Is Expensive                           |
  +------------------------+-----------------------------------------------+
  | Leading wildcard       | query: "*earch" must scan every term in the   |
  | (wildcard, prefix)     | inverted index for every segment              |
  +------------------------+-----------------------------------------------+
  | Regular expressions    | Regex evaluation against all terms; cannot    |
  | (regexp)               | use the inverted index efficiently            |
  +------------------------+-----------------------------------------------+
  | Script queries         | Execute arbitrary code per document; no       |
  | (script_score, script) | index structure can accelerate them           |
  +------------------------+-----------------------------------------------+
  | Fuzzy with high edit   | Generates massive term expansions that        |
  | distance (fuzziness>2) | consume heap and CPU                          |
  +------------------------+-----------------------------------------------+
  | Nested queries on      | Each nested doc is a hidden Lucene document;  |
  | large arrays           | large arrays multiply query cost              |
  +------------------------+-----------------------------------------------+
```

To prevent expensive queries from destabilizing the cluster, use the circuit breaker:

```json
PUT /_cluster/settings
{
  "persistent": {
    "search.allow_expensive_queries": false
  }
}
```

**Preference Parameter for Cache Affinity**

By default, Elasticsearch rotates search requests across all shard copies (primary and replicas)
using adaptive replica selection. Setting a `preference` value ensures repeated identical queries
always hit the same shard copy, maximizing OS page cache and node query cache hit rates:

```json
GET /products/_search?preference=user_session_abc123
{
  "query": {
    "match": { "name": "laptop" }
  }
}
```

**Source Filtering**

Fetching the full `_source` field for every hit is wasteful when you only need a few fields.
Use `_source` filtering to reduce network bandwidth and serialization overhead:

```json
GET /products/_search
{
  "_source": ["name", "price", "sku"],
  "query": {
    "match": { "name": "laptop" }
  }
}
```

**Pagination Pitfalls**

Deep pagination is one of the most common performance traps. The `from + size` approach requires
every shard to produce `from + size` results, which the coordinating node then merges and
discards down to the requested page:

```
  Pagination Cost Comparison
  ---------------------------

  +------------------+-----------+--------------------------+----------------------------+
  | Method           | Max Depth | Memory per Shard         | Best Use Case              |
  +------------------+-----------+--------------------------+----------------------------+
  | from + size      | 10,000    | O(from + size) per shard | UI pagination, first few   |
  |                  | (default) |                          | pages only                 |
  +------------------+-----------+--------------------------+----------------------------+
  | search_after     | Unlimited | O(size) per shard        | Deep pagination, infinite  |
  |                  |           |                          | scroll UIs                 |
  +------------------+-----------+--------------------------+----------------------------+
  | scroll           | Unlimited | Holds search context     | Batch export, reindexing   |
  |                  |           | (resource intensive)     | (not for user-facing)      |
  +------------------+-----------+--------------------------+----------------------------+
  | Point in Time    | Unlimited | Holds PIT context        | Consistent deep pagination |
  | (PIT) + search   |           |                          | with search_after          |
  | _after           |           |                          |                            |
  +------------------+-----------+--------------------------+----------------------------+
```

Example using `search_after` for efficient deep pagination:

```json
GET /logs-2024.07/_search
{
  "size": 100,
  "sort": [
    { "timestamp": "desc" },
    { "_id": "asc" }
  ],
  "search_after": ["2024-07-15T10:30:00.000Z", "doc_id_500"]
}
```

**Profile API for Query Analysis**

When a query is slow, the Profile API reveals exactly where time is spent — at the level of
individual Lucene collectors, scorers, and rewrite steps:

```json
GET /products/_search
{
  "profile": true,
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "laptop" } }
      ],
      "filter": [
        { "range": { "price": { "gte": 500, "lte": 2000 } } }
      ]
    }
  }
}
```

```json
{
  "took": 12,
  "profile": {
    "shards": [
      {
        "id": "[node_1][products][0]",
        "searches": [
          {
            "query": [
              {
                "type": "BooleanQuery",
                "description": "+name:laptop #price:[500 TO 2000]",
                "time_in_nanos": 8423100,
                "breakdown": {
                  "score": 2145000,
                  "build_scorer": 1823000,
                  "match": 0,
                  "create_weight": 412000,
                  "advance": 1890000,
                  "next_doc": 2153100
                },
                "children": [
                  {
                    "type": "TermQuery",
                    "description": "name:laptop",
                    "time_in_nanos": 4210000
                  },
                  {
                    "type": "IndexOrDocValuesQuery",
                    "description": "price:[500 TO 2000]",
                    "time_in_nanos": 1520000
                  }
                ]
              }
            ],
            "collector": [
              {
                "name": "SimpleTopScoreDocCollector",
                "reason": "search_top_hits",
                "time_in_nanos": 3214000
              }
            ]
          }
        ]
      }
    ]
  }
}
```

The profile output reveals that scoring consumed the most time (2.1ms). If scoring is not needed,
wrapping the query in a `constant_score` filter would eliminate that cost.

---

### Shard Optimization

Shard count is the most consequential design decision for an Elasticsearch index. It is set at
index creation time and cannot be changed without reindexing (though the shrink and split APIs
offer workarounds). Getting it wrong causes either resource waste or performance degradation.

**Right-Sizing Shards**

The recommended shard size is **10–50 GB** per primary shard. This range balances several
competing concerns:

- Smaller shards recover faster during node failures
- Larger shards reduce per-shard overhead (each shard consumes heap, file handles, and a thread
  pool slot during search)
- Very large shards (>50 GB) make segment merges expensive and slow recovery

**Too Many Shards**

Every shard is a full Lucene index with its own data structures. Each shard consumes:
- ~10 MB of heap for metadata and caching structures
- File descriptors for each segment file
- A thread pool slot during search execution

A cluster with thousands of tiny shards (the "oversharding" problem) wastes heap, slows cluster
state propagation, and increases coordination overhead on the master node.

**Too Few Shards**

If an index has too few shards, you cannot fully utilize the data nodes in the cluster. Search
parallelism is limited by shard count: a 10-node cluster searching a single-shard index uses
only one node for the query phase. Write throughput is similarly bounded because indexing
operations are distributed across primary shards.

```
  Shard Sizing Guidelines
  -------------------------

  +------------------+------------------+---------------------------------------+
  | Data Volume      | Recommended      | Rationale                             |
  |                  | Primary Shards   |                                       |
  +------------------+------------------+---------------------------------------+
  | < 10 GB          | 1                | Single shard minimizes overhead;      |
  |                  |                  | data fits comfortably in one shard    |
  +------------------+------------------+---------------------------------------+
  | 10 - 50 GB       | 1 - 3            | One shard if queries are light;      |
  |                  |                  | more if write throughput is needed    |
  +------------------+------------------+---------------------------------------+
  | 50 - 200 GB      | 3 - 5            | Balances parallelism with overhead;  |
  |                  |                  | each shard is 10-50 GB               |
  +------------------+------------------+---------------------------------------+
  | 200 GB - 1 TB    | 5 - 20           | Scale shards with data nodes; aim    |
  |                  |                  | for 30-50 GB per shard               |
  +------------------+------------------+---------------------------------------+
  | > 1 TB           | 20+              | Use time-based indices with rollover |
  |                  |                  | to keep individual indices manageable |
  +------------------+------------------+---------------------------------------+
```

Use the `_cat/shards` API to monitor current shard sizes:

```json
GET /_cat/shards?v&h=index,shard,prirep,store,docs&s=store:desc
```

```
index          shard prirep store   docs
logs-2024.07   0     p      42.3gb  85234102
logs-2024.07   1     p      41.8gb  84921033
logs-2024.07   2     p      43.1gb  86012340
products       0     p      2.1gb   1503200
```

---

### Segment Management

Each Elasticsearch shard is a Lucene index, and each Lucene index is composed of **segments** —
immutable files that contain the inverted index, stored fields, doc values, and other data
structures. Understanding how segments work is essential for performance tuning.

**Segment Lifecycle**

When documents are indexed, they first accumulate in an in-memory buffer. When the buffer is
flushed (either by the refresh interval or a manual refresh), a new segment is written to disk.
Over time, many small segments accumulate. Lucene's merge policy periodically combines smaller
segments into larger ones, which reduces the number of files that must be searched and reclaims
space from deleted documents.

```
  Segment Merge Process
  ----------------------

  Before merge:
  +------+ +------+ +------+ +------+ +------+
  | Seg0 | | Seg1 | | Seg2 | | Seg3 | | Seg4 |
  | 2 MB | | 3 MB | | 1 MB | | 4 MB | | 2 MB |
  +------+ +------+ +------+ +------+ +------+

  Merge policy selects segments:
  +------+ +------+ +------+
  | Seg0 | | Seg1 | | Seg2 |   ---->   +----------+
  | 2 MB | | 3 MB | | 1 MB |           | Seg5     |
  +------+ +------+ +------+           | 6 MB     |
                                        +----------+

  After merge:
  +----------+ +------+ +------+
  |   Seg5   | | Seg3 | | Seg4 |
  |   6 MB   | | 4 MB | | 2 MB |
  +----------+ +------+ +------+
```

**Force Merge for Read-Only Indices**

For indices that are no longer receiving writes (e.g., time-based indices that have rolled over),
force merging down to a single segment eliminates merge overhead and optimizes search performance.
This operation is resource-intensive and should only be done during off-peak hours:

```json
POST /logs-2024.06/_forcemerge?max_num_segments=1
```

```json
{
  "_shards": {
    "total": 6,
    "successful": 6,
    "failed": 0
  }
}
```

After force merging, the index should be marked as read-only to prevent accidental writes that
would create new segments:

```json
PUT /logs-2024.06/_settings
{
  "index": {
    "blocks.write": true
  }
}
```

**Merge Policy Settings**

The default merge policy (`tiered`) works well for most workloads. For write-heavy indices,
adjusting the floor segment size and maximum merge-at-once count can reduce merge pressure:

```json
PUT /logs-2024.07/_settings
{
  "index": {
    "merge.policy.floor_segment": "5mb",
    "merge.policy.max_merge_at_once": 15,
    "merge.policy.max_merged_segment": "5gb"
  }
}
```

---

### Caching

Elasticsearch maintains several cache layers, each serving a different purpose. Understanding
which cache serves which role — and how to monitor them — is essential for diagnosing performance
issues.

```
  Caching Layers
  ---------------

  +-------------------------------------------------------------------+
  |                        OS Page Cache                               |
  |  Caches Lucene segment files at the filesystem level.              |
  |  Managed by the OS, not Elasticsearch. Leave sufficient RAM.       |
  +-------------------------------------------------------------------+

  +---------------------+  +---------------------+  +-----------------+
  |  Node Query Cache   |  | Shard Request Cache  |  | Field Data     |
  |                     |  |                      |  | Cache          |
  | Caches filter       |  | Caches full search   |  |                |
  | clause results as   |  | response (hits,      |  | In-memory data |
  | bitsets. Only for   |  | aggregations) for    |  | structure for  |
  | filter context.     |  | requests with        |  | text field     |
  |                     |  | size: 0 on read-     |  | fielddata      |
  | Default: 10% heap   |  | only indices.        |  | (sorting /     |
  |                     |  |                      |  | aggregations)  |
  | indices.queries     |  | index.requests       |  |                |
  | .cache.size         |  | .cache.enable        |  | Default: no    |
  |                     |  |                      |  | limit (danger) |
  +---------------------+  +---------------------+  +-----------------+
```

**Monitoring Cache Usage**

```json
GET /_nodes/stats/indices/query_cache,request_cache,fielddata
```

```json
{
  "nodes": {
    "node_id_1": {
      "indices": {
        "query_cache": {
          "memory_size_in_bytes": 52428800,
          "total_count": 234500,
          "hit_count": 189200,
          "miss_count": 45300,
          "cache_size": 1024,
          "evictions": 320
        },
        "request_cache": {
          "memory_size_in_bytes": 26214400,
          "evictions": 15,
          "hit_count": 8930,
          "miss_count": 2100
        },
        "fielddata": {
          "memory_size_in_bytes": 104857600,
          "evictions": 0
        }
      }
    }
  }
}
```

A high `evictions` count on the query cache suggests the cache is too small or query patterns are
too diverse. High fielddata memory usage (especially with evictions) is a red flag — fielddata on
text fields should be avoided in favor of `keyword` fields or doc values.

**Clearing Caches**

In rare cases (debugging, benchmarking), you may need to clear caches manually:

```json
POST /_cache/clear
```

```json
POST /products/_cache/clear?query=true&request=true&fielddata=true
```

---

### Slow Logs

Elasticsearch can log queries and indexing operations that exceed configurable time thresholds.
Slow logs are the first tool to reach for when users report sporadic latency spikes.

**Configuring Search Slow Logs**

```json
PUT /products/_settings
{
  "index": {
    "search.slowlog.threshold.query.warn": "10s",
    "search.slowlog.threshold.query.info": "5s",
    "search.slowlog.threshold.query.debug": "2s",
    "search.slowlog.threshold.query.trace": "500ms",
    "search.slowlog.threshold.fetch.warn": "1s",
    "search.slowlog.threshold.fetch.info": "800ms",
    "search.slowlog.threshold.fetch.debug": "500ms",
    "search.slowlog.threshold.fetch.trace": "200ms",
    "search.slowlog.level": "info"
  }
}
```

```json
{
  "acknowledged": true
}
```

**Configuring Indexing Slow Logs**

```json
PUT /products/_settings
{
  "index": {
    "indexing.slowlog.threshold.index.warn": "10s",
    "indexing.slowlog.threshold.index.info": "5s",
    "indexing.slowlog.threshold.index.debug": "2s",
    "indexing.slowlog.threshold.index.trace": "500ms",
    "indexing.slowlog.source": "1000"
  }
}
```

```json
{
  "acknowledged": true
}
```

The `indexing.slowlog.source` setting controls how many characters of the document source are
included in the slow log entry. Setting it to `1000` captures enough context for debugging
without filling log files with enormous documents.

Slow log entries appear in the Elasticsearch log directory as separate files
(`<index>_index_search_slowlog.log` and `<index>_index_indexing_slowlog.log`) and include the
full query body, shard ID, and execution time.

---

### Monitoring and Diagnostics

A production Elasticsearch cluster requires continuous monitoring. The following APIs form the
core diagnostic toolkit.

**Node Stats — Key Metrics**

The `_nodes/stats` API is the most comprehensive single endpoint for node health:

```json
GET /_nodes/stats?filter_path=**.jvm.mem,**.os.cpu,**.fs,**.thread_pool.search,**.thread_pool.write
```

```json
{
  "nodes": {
    "node_id_1": {
      "jvm": {
        "mem": {
          "heap_used_in_bytes": 21474836480,
          "heap_max_in_bytes": 32212254720,
          "heap_used_percent": 66
        }
      },
      "os": {
        "cpu": {
          "percent": 42
        }
      },
      "fs": {
        "total": {
          "total_in_bytes": 2000398934016,
          "available_in_bytes": 1200239360409
        }
      },
      "thread_pool": {
        "search": {
          "threads": 25,
          "queue": 0,
          "active": 3,
          "rejected": 0,
          "completed": 1892340
        },
        "write": {
          "threads": 16,
          "queue": 0,
          "active": 5,
          "rejected": 0,
          "completed": 5432100
        }
      }
    }
  }
}
```

Key indicators to watch:
- **heap_used_percent** > 75% sustained → risk of GC pressure
- **rejected** count on any thread pool → requests are being dropped
- **queue** size growing → node is falling behind

**Thread Pool Monitoring**

```json
GET /_cat/thread_pool?v&h=node_name,name,active,queue,rejected,completed&s=rejected:desc
```

```
node_name  name     active queue rejected completed
data-h1    search   3      0     0        1892340
data-h1    write    5      0     0        5432100
data-h2    search   2      0     0        1756200
data-h2    write    4      0     0        5210300
```

Non-zero `rejected` values on the `search` or `write` pools indicate the node is overloaded.

**Cluster Stats Overview**

```json
GET /_cluster/stats?filter_path=indices.count,indices.shards.total,indices.docs,nodes.count
```

```json
{
  "indices": {
    "count": 245,
    "shards": {
      "total": 1470
    },
    "docs": {
      "count": 12849302100
    }
  },
  "nodes": {
    "count": {
      "total": 15,
      "data": 10,
      "master": 3,
      "coordinating_only": 2
    }
  }
}
```

**Hot Threads API**

When nodes are unresponsive or CPU is pinned, the hot threads API captures stack traces of the
busiest threads:

```json
GET /_nodes/hot_threads?threads=5&timeout=5s
```

The output is plain text (not JSON), showing the top CPU-consuming threads with stack traces.
Look for patterns like repeated GC activity, long-running merge threads, or expensive aggregation
computations.

**Task Management API**

Long-running tasks (reindex, force merge, search) can be listed, inspected, and cancelled:

```json
GET /_tasks?detailed=true&actions=*search&group_by=parents
```

```json
{
  "tasks": {
    "node_id_1:12345": {
      "node": "node_id_1",
      "id": 12345,
      "type": "transport",
      "action": "indices:data/read/search",
      "description": "indices[products], search_type[QUERY_THEN_FETCH]",
      "start_time_in_millis": 1721035800000,
      "running_time_in_nanos": 45230000000,
      "cancellable": true
    }
  }
}
```

Cancel a long-running task:

```json
POST /_tasks/node_id_1:12345/_cancel
```

---

### Snapshot and Backup

Snapshots are the only supported method for backing up an Elasticsearch cluster. They capture
index data, cluster state, and metadata, and store them in an external repository (S3, GCS,
Azure Blob, HDFS, or a shared filesystem).

**Registering a Repository**

```json
PUT /_snapshot/my_s3_backup
{
  "type": "s3",
  "settings": {
    "bucket": "elasticsearch-backups",
    "region": "us-east-1",
    "base_path": "production/snapshots",
    "compress": true,
    "server_side_encryption": true
  }
}
```

```json
{
  "acknowledged": true
}
```

**Taking a Snapshot**

```json
PUT /_snapshot/my_s3_backup/snapshot_2024_07_15?wait_for_completion=true
{
  "indices": "logs-2024.07,products",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

```json
{
  "snapshot": {
    "snapshot": "snapshot_2024_07_15",
    "uuid": "abc123def456",
    "version_id": 8100099,
    "version": "8.10.0",
    "indices": ["logs-2024.07", "products"],
    "state": "SUCCESS",
    "start_time": "2024-07-15T02:00:00.000Z",
    "end_time": "2024-07-15T02:12:34.567Z",
    "duration_in_millis": 754567,
    "shards": {
      "total": 9,
      "failed": 0,
      "successful": 9
    }
  }
}
```

**Restoring from a Snapshot**

```json
POST /_snapshot/my_s3_backup/snapshot_2024_07_15/_restore
{
  "indices": "products",
  "ignore_unavailable": true,
  "include_global_state": false,
  "rename_pattern": "(.+)",
  "rename_replacement": "restored_$1"
}
```

```json
{
  "accepted": true
}
```

**Snapshot Lifecycle Management (SLM)**

SLM automates snapshot creation and retention. Define a policy that takes daily snapshots and
retains them for 30 days:

```json
PUT /_slm/policy/nightly_backup
{
  "schedule": "0 30 2 * * ?",
  "name": "<nightly-snap-{now/d}>",
  "repository": "my_s3_backup",
  "config": {
    "indices": ["*"],
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

```json
{
  "acknowledged": true
}
```

Execute the policy immediately (for testing):

```json
POST /_slm/policy/nightly_backup/_execute
```

---

### Index Lifecycle Management (ILM)

ILM automates the transition of indices through a series of phases — hot, warm, cold, frozen, and
delete — each optimized for a different access pattern and cost profile. This is essential for
time-series data (logs, metrics, traces) where recent data is queried frequently and older data
is accessed rarely.

```
  ILM Data Tiers
  ----------------

  +-----------+     +-----------+     +-----------+     +-----------+     +-----------+
  |           |     |           |     |           |     |           |     |           |
  |    HOT    +---->+   WARM    +---->+   COLD    +---->+  FROZEN   +---->+  DELETE   |
  |           |     |           |     |           |     |           |     |           |
  +-----------+     +-----------+     +-----------+     +-----------+     +-----------+
       |                 |                 |                 |                 |
       |                 |                 |                 |                 |
  Active writes,    No new writes,    Infrequent reads, Rare reads,      Index removed
  real-time search  force merge,      reduce replicas,  searchable       permanently
  NVMe SSDs,        shrink shards,    move to cheaper   snapshots,
  high replica      read-only         storage (HDD)     minimal
  count                                                 resources

  Typical Timeline:
  [0-7 days]       [7-30 days]       [30-90 days]      [90-365 days]    [365+ days]
```

**Creating an ILM Policy**

```json
PUT /_ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "7d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "allocate": {
            "require": {
              "data": "warm"
            }
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {
            "require": {
              "data": "cold"
            },
            "number_of_replicas": 0
          },
          "set_priority": {
            "priority": 0
          }
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

```json
{
  "acknowledged": true
}
```

**Applying the Policy to an Index Template**

```json
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs_policy",
      "index.lifecycle.rollover_alias": "logs-write"
    }
  }
}
```

**Checking ILM Status**

```json
GET /logs-2024.07/_ilm/explain
```

```json
{
  "indices": {
    "logs-2024.07": {
      "index": "logs-2024.07",
      "managed": true,
      "policy": "logs_policy",
      "lifecycle_date_millis": 1720915200000,
      "age": "5.2d",
      "phase": "hot",
      "phase_time_millis": 1720915200000,
      "action": "rollover",
      "action_time_millis": 1720915200000,
      "step": "check-rollover-ready",
      "step_time_millis": 1721035800000
    }
  }
}
```

---

### Common Anti-Patterns

The following table summarizes frequent mistakes in Elasticsearch deployments and their
consequences:

```
  Common Anti-Patterns
  ----------------------

  +---------------------------------------+---------------------------------------------+
  | Anti-Pattern                          | Why It Hurts                                |
  +---------------------------------------+---------------------------------------------+
  | Heap size > 31 GB                     | Loses compressed oops; wastes ~40% of heap  |
  |                                       | space compared to staying under 31 GB       |
  +---------------------------------------+---------------------------------------------+
  | Heap size = 100% of RAM               | Starves OS page cache; search performance   |
  |                                       | collapses because Lucene segments are not   |
  |                                       | cached in memory                            |
  +---------------------------------------+---------------------------------------------+
  | Thousands of tiny shards              | Each shard consumes heap and file handles;  |
  | (oversharding)                        | master node overwhelmed by cluster state    |
  +---------------------------------------+---------------------------------------------+
  | Single shard for large index          | Cannot parallelize search across nodes;     |
  |                                       | single node becomes bottleneck              |
  +---------------------------------------+---------------------------------------------+
  | Using from+size for deep pagination   | Each shard must produce from+size results;  |
  |                                       | memory and CPU cost grows linearly          |
  +---------------------------------------+---------------------------------------------+
  | Scoring clauses in filter context     | Wastes CPU on relevance scoring that is     |
  | positions                             | never used; prevents caching                |
  +---------------------------------------+---------------------------------------------+
  | Leading wildcard queries in           | Scans every term in the inverted index;     |
  | production                            | cannot be optimized                         |
  +---------------------------------------+---------------------------------------------+
  | Mapping explosion (too many fields)   | Cluster state grows unbounded; master node  |
  |                                       | OOM; index recovery slows dramatically      |
  +---------------------------------------+---------------------------------------------+
  | Not using bulk API for ingestion      | Single-document indexing has 10-100x more   |
  |                                       | HTTP overhead than bulk requests            |
  +---------------------------------------+---------------------------------------------+
  | Disabling replicas permanently        | Zero fault tolerance; any node failure      |
  |                                       | causes data loss and red cluster health     |
  +---------------------------------------+---------------------------------------------+
  | Running on swap-enabled systems       | GC pauses extend from milliseconds to       |
  |                                       | minutes; node appears dead to cluster       |
  +---------------------------------------+---------------------------------------------+
  | Fielddata on text fields              | Unbounded heap usage; causes OOM or         |
  |                                       | constant evictions and re-computation       |
  +---------------------------------------+---------------------------------------------+
  | Not setting refresh_interval during   | Creates thousands of tiny segments;         |
  | bulk ingestion                        | wastes I/O on constant merges               |
  +---------------------------------------+---------------------------------------------+
  | Force merge on active write indices   | New segments appear immediately after the   |
  |                                       | merge; wasted I/O and CPU                   |
  +---------------------------------------+---------------------------------------------+
```

---

### Command Reference

| Operation | REST Verb & Endpoint | Key Parameters |
|-----------|----------------------|----------------|
| Refresh index | `POST /<index>/_refresh` | Makes recently indexed documents searchable |
| Force merge | `POST /<index>/_forcemerge` | `max_num_segments` |
| Update settings | `PUT /<index>/_settings` | `refresh_interval`, `translog.*`, `merge.policy.*` |
| Cluster settings | `PUT /_cluster/settings` | `persistent`, `transient` |
| Node stats | `GET /_nodes/stats` | Filter by metric: `jvm`, `fs`, `thread_pool`, `indices` |
| Cluster stats | `GET /_cluster/stats` | Aggregate indices, nodes, and resource overview |
| Cat thread pool | `GET /_cat/thread_pool?v` | `h` (name, active, rejected, queue, completed) |
| Cat shards | `GET /_cat/shards?v` | `h`, `s`, filter by index |
| Hot threads | `GET /_nodes/hot_threads` | `threads`, `timeout`, `type` (cpu, wait, block) |
| Task list | `GET /_tasks` | `detailed`, `actions`, `group_by` |
| Cancel task | `POST /_tasks/<task_id>/_cancel` | Cancels a running task |
| Profile query | `GET /<index>/_search` (body) | `"profile": true` in request body |
| Slow log config | `PUT /<index>/_settings` | `search.slowlog.threshold.*`, `indexing.slowlog.threshold.*` |
| Clear cache | `POST /_cache/clear` | `query`, `request`, `fielddata` |
| Cache stats | `GET /_nodes/stats/indices` | `query_cache`, `request_cache`, `fielddata` |
| Snapshot repo | `PUT /_snapshot/<repo>` | `type`, `settings` (bucket, region, base_path) |
| Take snapshot | `PUT /_snapshot/<repo>/<snap>` | `indices`, `wait_for_completion`, `include_global_state` |
| Restore snapshot | `POST /_snapshot/<repo>/<snap>/_restore` | `indices`, `rename_pattern`, `rename_replacement` |
| SLM policy | `PUT /_slm/policy/<name>` | `schedule`, `repository`, `config`, `retention` |
| Execute SLM | `POST /_slm/policy/<name>/_execute` | Triggers immediate snapshot |
| Create ILM policy | `PUT /_ilm/policy/<name>` | `policy.phases` (hot, warm, cold, delete) |
| Explain ILM | `GET /<index>/_ilm/explain` | Shows current phase, action, step |
| Memory lock check | `GET /_nodes?filter_path=**.mlockall` | Verify bootstrap.memory_lock |
| Allocation explain | `GET /_cluster/allocation/explain` | `index`, `shard`, `primary` |
| Disable expensive queries | `PUT /_cluster/settings` | `search.allow_expensive_queries: false` |
| Index buffer size | `PUT /_cluster/settings` | `indices.memory.index_buffer_size` |
