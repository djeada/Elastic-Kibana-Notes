### 1. **How does segment merging reduce index fragmentation and improve performance?**

Segment merging is a background process that consolidates smaller segments into larger ones, removing deleted documents and applying compression, which reduces index fragmentation, improves query performance, and reclaims disk space; Elasticsearch handles merging automatically via built-in policies, but you can tune merge parameters for high-deletion workloads or trigger merges manually during off-peak hours.

### 2. **Why is Elasticsearch using so much memory?**

Elasticsearch consumes significant memory because the JVM heap is used for indexing, caching, and field data (especially during aggregations and sorting), the OS file system cache is heavily leveraged to speed up disk I/O, and additional memory goes to index/query caches and temporary buffers needed during segment merging.

### 3. **Why do replicas improve search performance and resiliency?**

Replicas are copies of primary shards that serve read queries to distribute the search load, provide fault tolerance so the cluster stays operational when nodes fail, and enable load balancing that lowers overall query latency.

### 4. **What is the effect of `refresh_interval` on indexing performance?**

The refresh_interval setting controls how often Elasticsearch makes recent changes searchable; a shorter interval provides fresher search results but reduces indexing throughput, while a longer interval (or disabling refresh with -1) boosts indexing speed at the cost of search latency, so it is common to increase the interval during bulk ingestion and restore it afterward.

### 5. **Why do deletions not immediately free disk space in Elasticsearch?**

Deleted documents are only marked with tombstones inside their segments and the actual disk space is not reclaimed until a segment merge consolidates those segments, removes the tombstoned documents, and writes a new compact segment, so aggressive merge policies or a manual force merge may be needed if rapid space recovery is required.

### 6. **Why can adding more documents shrink the index?**

Adding documents triggers segment merges that consolidate many small segments into fewer large ones, and during this process deleted or outdated documents are purged and compression is applied, which can make the resulting index smaller than it was before the new documents were added.

### 7. **Why is it beneficial to use the Bulk API for indexing operations?**

The Bulk API batches multiple index, update, and delete operations into a single HTTP request, which dramatically reduces per-request network overhead, increases throughput during large-scale ingestion or migration tasks, and allows the cluster to optimize internal write coordination across shards.

### 8. **What is the significance of shard allocation and balancing in an Elasticsearch cluster?**

Shard allocation distributes primary and replica shards evenly across nodes so that no single node is overloaded, which improves parallel query processing, increases fault tolerance against node failures, and ensures the cluster can scale horizontally by simply adding more nodes.

### 9. **What are the trade-offs of using nested objects in queries?**

Nested objects let you index arrays of objects and query each object independently while preserving relational context, but they add query complexity and performance overhead because Elasticsearch must perform internal joins; flattening data or using parent-child relationships may be preferable depending on query patterns and update frequency.

### 10. **How can caching improve search performance in Elasticsearch?**

Elasticsearch uses the node-level query cache for frequently executed filter clauses, the shard-level request cache for aggregation results, the field data cache for sorting and scripting, and the OS filesystem cache for segment file reads, all of which reduce repeated computation and disk I/O; monitoring cache hit rates and tuning cache sizes are essential for optimal performance.

### 11. **Why isn't my search for `*foo-bar*` matching `foo-bar`?**

The standard analyzer splits foo-bar into separate foo and bar tokens at the hyphen, so a wildcard query on the analyzed field will not find the original compound term; the fix is to query a keyword sub-field that stores the entire value as a single token, or to define a custom analyzer that does not split on hyphens.

### 12. **Why am I getting `IndexNotFoundException` errors?**

An IndexNotFoundException means the target index does not exist, usually because of a typo in the index name, a reference to an index that was never created, or an accidental deletion; verify the index name with the _cat/indices API, ensure index creation happens before any read or write operations, and add error handling that checks for index existence first.

### 13. **What is the difference between a query context and a filter context?**

In query context Elasticsearch calculates a relevance score for every matching document, whereas in filter context it only checks whether a document matches without scoring, making filters faster and cacheable; use query context for full-text search where ranking matters and filter context for exact-value conditions like status, date ranges, or boolean flags.

### 14. **What is an inverted index and why is it central to Elasticsearch?**

An inverted index maps every unique term produced by the analyzer to the list of document IDs containing that term along with positional data, enabling Elasticsearch to resolve full-text searches in near-constant time regardless of the total number of documents, which is the fundamental data structure behind Lucene and all Elasticsearch search operations.

### 15. **How does the BM25 scoring algorithm work?**

BM25 ranks documents by combining term frequency (how often the term appears in a document, with diminishing returns), inverse document frequency (rarer terms across the index score higher), and field length normalization (shorter fields receive a boost), replacing the older TF-IDF default and providing more accurate relevance scoring out of the box.

### 16. **What happens when an Elasticsearch node fails?**

When a node fails the master promotes replicas of the lost primary shards to primary status, then allocates new replicas on the remaining nodes to restore the configured replication factor; during this process the cluster health transitions from green to yellow (or red if primary shards are lost), and once all shards are reallocated health returns to green.

### 17. **Why should the JVM heap size not exceed 31 GB?**

Keeping the heap at or below approximately 31 GB allows the JVM to use compressed ordinary object pointers (compressed oops), which saves memory and improves garbage collection performance; exceeding this threshold disables compressed oops, effectively wasting several gigabytes and increasing GC pause times, so the remaining RAM should be left for the OS filesystem cache.

### 18. **What is an index alias and why is it useful?**

An index alias is a secondary name that points to one or more indices, letting you swap the underlying index transparently during reindexing or rollover operations, apply filtered or routed views for multi-tenant use cases, and simplify client configuration so applications never need to change the index name they query.

### 19. **What are ingest pipelines and how do they transform data?**

An ingest pipeline is an ordered sequence of processors (such as grok, date, geoip, rename, and script) that transform and enrich documents before they are indexed, running on ingest nodes so that source systems do not need to pre-process data; pipelines can be tested with the _simulate API and attached to indices via default_pipeline settings.

### 20. **How does Elasticsearch prevent split-brain scenarios?**

Elasticsearch requires a majority quorum of master-eligible nodes to elect a master, so with an odd number of master-eligible nodes (minimum three) no two partitions can independently form a quorum; this voting configuration ensures only one partition can elect a master, preventing conflicting writes and divergent cluster states.
