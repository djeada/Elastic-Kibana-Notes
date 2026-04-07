### 1. **What is the recommended JVM heap size for Elasticsearch?**

Set the JVM heap to no more than 50% of available physical RAM and never exceed approximately 31 GB, because staying at or below 31 GB lets the JVM use compressed ordinary object pointers (compressed oops) which saves memory and reduces garbage collection overhead; the remaining RAM should be left for the OS filesystem cache which Elasticsearch relies on heavily for segment reads.

### 2. **Why are SSDs strongly preferred over HDDs for Elasticsearch?**

Elasticsearch performs many random I/O operations during indexing, merging, and searching, and SSDs deliver orders-of-magnitude faster random read and write latency compared to spinning disks, which directly translates to faster query responses, shorter segment merge times, and higher indexing throughput.

### 3. **What is the optimal shard size?**

Shards should generally be between 10 GB and 50 GB, because shards that are too small create excessive per-shard overhead (memory, file descriptors, thread scheduling) while shards that are too large increase recovery time after node failures and limit the ability to distribute work evenly across nodes.

### 4. **What problems does oversharding cause?**

Oversharding (too many small shards) wastes memory on per-shard metadata and Lucene data structures, increases cluster state size, slows down master node operations, and produces more fan-out during search, all of which degrade performance; consolidate small shards using the shrink API or by reducing the shard count in index templates.

### 5. **How does `refresh_interval` affect performance?**

Each refresh creates a new Lucene segment that consumes memory and file handles, so the default one-second interval is expensive during heavy ingestion; increasing refresh_interval to 30s or setting it to -1 during bulk loads significantly reduces overhead and improves indexing throughput, and you can restore the default once ingestion completes.

### 6. **What is the translog and what are its durability options?**

The translog (transaction log) records every write operation before it is committed to a Lucene segment, providing crash recovery; with index.translog.durability set to request (default) the translog is fsynced after every operation for maximum safety, while async mode fsyncs at a configurable interval (sync_interval) trading durability for higher write speed.

### 7. **Why are auto-generated document IDs faster than client-specified IDs?**

When a client provides an ID, Elasticsearch must first check whether a document with that ID already exists (a version lookup) before indexing, whereas auto-generated IDs skip this existence check entirely because they are guaranteed to be unique, saving one Lucene lookup per document.

### 8. **What is the difference between query context and filter context for performance?**

Filter context does not calculate relevance scores and its results are cached by the node query cache, making repeated filter-based queries extremely fast; query context computes a score for every matching document which is more CPU-intensive and not cached, so moving exact-value conditions into filter clauses inside a bool query is a key optimization.

### 9. **What are the three main caching layers in Elasticsearch?**

The OS page cache caches segment files in RAM for fast disk reads, the node-level query cache stores the results of frequently used filter clauses per segment, and the shard-level request cache stores the results of entire search or aggregation requests on read-only shards; all three work together to minimize repeated computation and I/O.

### 10. **What is a force merge and when should it be used?**

Force merge reduces the number of Lucene segments in a shard to a specified count (typically one), which improves search performance and frees disk space from deleted documents; it should only be run on read-only indices (such as those that have been rolled over) because the operation is I/O-intensive and blocks further segment merges.

### 11. **How do you configure and analyze slow logs?**

Set index.search.slowlog.threshold.query.warn and index.indexing.slowlog.threshold.index.warn (and similar levels) to time thresholds, and Elasticsearch will log any query or indexing operation that exceeds those thresholds; analyze slow log entries to identify expensive queries, suboptimal mappings, or resource-constrained nodes.

### 12. **What metrics should you monitor for performance tuning?**

Monitor JVM heap usage and GC times, CPU load, disk I/O wait, search and indexing latency, segment count and merge activity, thread pool queue sizes and rejections, cache hit rates, and cluster-level shard balance; the _nodes/stats and _cluster/stats APIs expose all of these metrics.

### 13. **What is memory locking and why is it important?**

Memory locking (bootstrap.memory_lock: true) prevents the OS from swapping the JVM heap to disk, which would cause severe latency spikes and unpredictable GC pauses; it ensures that the heap memory Elasticsearch needs is always resident in physical RAM, and the process must be allowed to lock memory via OS-level settings (e.g., ulimit or systemd).

### 14. **What are common performance anti-patterns?**

Common anti-patterns include mapping all string fields as text when keyword is needed, running wildcard queries with a leading asterisk, using deeply nested aggregations, querying across thousands of shards, not leveraging filter context for non-scoring clauses, allocating too much or too little heap, and indexing with a one-second refresh interval during bulk loads.

### 15. **How does the `preference` parameter improve search caching?**

Setting the preference parameter to a session ID or user ID routes repeated identical searches to the same shard copies, improving the node query cache and OS page cache hit rates because the same data segments are accessed consistently rather than being randomly distributed across different replicas.
