### 1. **How is an Elasticsearch index distributed across nodes?**

An index is divided into primary shards, each stored on a different data node, and each primary shard has one or more replica shards placed on other nodes; the master node manages shard allocation to ensure even distribution, fault tolerance, and parallel query execution across the cluster.

### 2. **What is the shard routing formula?**

Elasticsearch determines which shard holds a document using the formula shard_number = hash(_routing) % number_of_primary_shards, where _routing defaults to the document's _id; this is why the number of primary shards cannot be changed after index creation, as doing so would change the routing and make existing documents unlocatable.

### 3. **How many total shards exist with 3 primaries and 1 replica?**

With number_of_shards set to 3 and number_of_replicas set to 1, the index has 3 primary shards plus 3 replica shards for a total of 6 shard copies; each replica is a full copy of its corresponding primary and is always placed on a different node than its primary.

### 4. **What is shard allocation and who controls it?**

Shard allocation is the process of assigning shards to data nodes, controlled by the master node's allocator which considers disk usage watermarks, node attributes, allocation awareness settings, forced awareness, and custom allocation filters to determine the optimal placement of each shard.

### 5. **What is the write path in Elasticsearch?**

A write request arrives at the coordinating node which routes it to the correct primary shard using the routing formula, the primary shard writes the operation to the translog and in-memory buffer, then replicates the operation to all replica shards in parallel; the request only returns success after all active replicas acknowledge the write.

### 6. **What is the translog and what role does it play?**

The translog (transaction log) is an append-only file that records every write operation before it is committed to a Lucene segment, ensuring durability in case of a crash; during recovery Elasticsearch replays uncommitted translog entries to restore the shard to its last consistent state.

### 7. **What happens during a refresh?**

A refresh writes the contents of the in-memory indexing buffer to a new Lucene segment on the filesystem cache (but not yet fsynced to disk), making recently indexed documents searchable; by default this occurs every second, and each refresh creates a new small segment that will later be merged.

### 8. **How does the read path (scatter-gather) work?**

The coordinating node broadcasts the search request to one copy of every shard (primary or replica) in the index, each shard executes the query locally and returns its top results, then the coordinating node merges and re-sorts all shard results to produce the final global result set.

### 9. **What is adaptive replica selection?**

Adaptive replica selection routes search requests to the shard copy that is most likely to respond fastest, considering factors like queue size, response time history, and node load, rather than using simple round-robin; this reduces tail latency and improves overall search throughput in heterogeneous clusters.

### 10. **What happens when a node fails?**

The master detects the failed node via the fault detection mechanism, removes it from the cluster, promotes the replica shards of any lost primary shards to primary status, and then allocates new replicas on the remaining nodes to restore the configured replication factor; cluster health transitions to yellow during recovery and returns to green once all shards are allocated.

### 11. **What is the cluster health color model?**

Green means all primary and replica shards are assigned, yellow means all primaries are assigned but some replicas are not (often because there are not enough nodes), and red means one or more primary shards are unassigned so some data is unavailable and the cluster cannot serve complete results.

### 12. **What is split brain and how is it prevented?**

Split brain occurs when a network partition causes two groups of nodes to independently elect their own master, leading to divergent cluster states and data corruption; Elasticsearch prevents this by requiring a majority quorum of master-eligible nodes for leader election so that at most one partition can have enough votes.

### 13. **What are the recommended shard sizing guidelines?**

Keep shards between 10 and 50 GB, aim for fewer than 20 shards per GB of heap memory on each node, and avoid creating more primary shards than necessary for your data volume; under-sharding limits parallelism while over-sharding wastes resources on per-shard overhead.

### 14. **What is a Lucene segment's lifecycle within a shard?**

Documents are first written to an in-memory buffer and the translog, then during a refresh they become a new immutable segment visible to searches, and over time background merges combine smaller segments into larger ones, physically removing deleted documents and compressing data in the process.

### 15. **How does horizontal scaling work in Elasticsearch?**

Adding data nodes to the cluster triggers automatic shard rebalancing by the master, which moves shards to the new nodes to distribute the load more evenly; this increases total storage capacity, memory, CPU resources, and search parallelism without downtime, though the number of primary shards for existing indices remains fixed.
