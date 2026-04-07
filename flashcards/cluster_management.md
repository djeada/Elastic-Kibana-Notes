### 1. **What do the cluster health statuses GREEN, YELLOW, and RED mean?**

GREEN means all primary and replica shards are allocated and the cluster is fully operational, YELLOW means all primary shards are allocated but one or more replica shards are not (data is available but redundancy is reduced), and RED means one or more primary shards are unassigned so some data is unavailable for search and indexing.

### 2. **How do you check cluster health?**

Use the GET _cluster/health API which returns the cluster name, status color, number of nodes, active shards, relocating shards, initializing shards, and unassigned shards; adding ?level=indices or ?level=shards provides per-index or per-shard detail for diagnosing specific problems.

### 3. **What is the role of the master node in a cluster?**

The master node is responsible for managing the cluster state which includes index metadata, shard allocation decisions, and node membership; it coordinates index creation and deletion, mapping changes, and shard rebalancing, but does not handle data indexing or search requests unless it is also assigned the data role.

### 4. **How does Elasticsearch elect a master node?**

Master-eligible nodes participate in an election using a quorum-based voting mechanism where a candidate must receive votes from a majority of the voting configuration to become the elected master, ensuring that only one master is active at any time and preventing split-brain in network partitions.

### 5. **What is the difference between transient and persistent cluster settings?**

Transient settings take effect immediately but are lost when the cluster restarts, while persistent settings survive full cluster restarts and are stored in the cluster state; both override the static settings in elasticsearch.yml, and transient settings take precedence over persistent ones when both are defined for the same key.

### 6. **What are shard allocation awareness and forced awareness?**

Allocation awareness uses custom node attributes (like rack or zone) to ensure that primary and replica shards are placed on nodes in different failure domains, and forced awareness prevents allocation of replicas if only one zone is available, which avoids concentrating all copies in a single failure domain.

### 7. **What are disk-based shard allocation watermarks?**

The low watermark (default 85% disk used) prevents new shards from being allocated to a node, the high watermark (default 90%) triggers shard relocation away from the node, and the flood-stage watermark (default 95%) makes all indices on the node read-only until disk space is freed, protecting the cluster from running out of disk.

### 8. **How do you perform a rolling restart of an Elasticsearch cluster?**

Disable shard allocation with cluster.routing.allocation.enable set to none, stop one node at a time, upgrade or maintain it, restart it, wait for the node to rejoin and re-enable allocation, then wait for the cluster to return to green before proceeding to the next node, which ensures continuous availability throughout the process.

### 9. **How do you drain a node for maintenance?**

Set the node's allocation exclude filter (e.g., cluster.routing.allocation.exclude._name) to the target node name so Elasticsearch migrates all shards off that node to other nodes in the cluster, then wait until the node holds zero shards before taking it offline for maintenance.

### 10. **What information does the `_cat/nodes` API provide?**

The _cat/nodes endpoint returns a human-readable table showing each node's IP address, heap usage, RAM usage, CPU load, role flags (master, data, ingest), name, and the number of shards it holds, making it a quick diagnostic tool for spotting imbalanced or resource-constrained nodes.

### 11. **What is cluster rebalancing?**

Cluster rebalancing is the automatic process by which the master node moves shards between data nodes to maintain an even distribution of shard count and disk usage, triggered whenever nodes are added, removed, or when shard counts become significantly uneven; the threshold sensitivity can be tuned with the cluster.routing.rebalance settings.

### 12. **What does the `_cluster/stats` API return?**

The _cluster/stats API returns aggregated statistics across all nodes including the total number of indices, documents, and shards, storage size, memory usage, JVM details, OS information, and plugin versions, providing a high-level overview of cluster capacity and resource consumption.

### 13. **How do you troubleshoot unassigned shards?**

Use the GET _cluster/allocation/explain API which returns the reason a specific shard is unassigned (such as insufficient disk space, allocation filters, or missing nodes), suggests which node could accept the shard, and explains any allocation decisions, making it the primary diagnostic tool for allocation problems.

### 14. **What is the voting configuration in Elasticsearch?**

The voting configuration is the set of master-eligible nodes whose votes are counted during master elections; Elasticsearch automatically maintains this set, adding nodes when they join and removing them when they leave, to ensure the quorum requirement always reflects the current set of available master-eligible nodes.

### 15. **Why should you use an odd number of master-eligible nodes?**

An odd number (typically three) of master-eligible nodes ensures that a clear majority quorum can be formed in any network partition, avoiding a tie where neither partition has enough votes to elect a master, which would render the entire cluster unable to process state changes or shard assignments.
