## Distributed Index Architecture

```
  _________       _________      _________      _________
 | Node 1  |     | Node 2  |    | Node 3  |    | Node 4  |
 |         |     |         |    |         |    |         |
 |   [##]  |     |   [##]  |    |   [##]  |    |   [##]  |
 |   [##]  |     |         |    |   [##]  |    |         |
 |_________|     |_________|    |_________|    |_________|

        ^            ^               ^             ^
        |            |               |             |
  +------------------------------------------------------+
  |                  Shared Elasticsearch Index          |
  +------------------------------------------------------+
```

### Index Shared Across Nodes

In a distributed Elasticsearch cluster, an **index** is divided into **shards** to allow horizontal scaling and improve redundancy and performance. Each shard contains part of the data in the index, and the shards are distributed across multiple **nodes**. In the ASCII diagram, each box represents a node in the cluster, while each `[##]` symbol represents a shard that contains a subset of the index’s data. This structure enables Elasticsearch to manage larger datasets and handle increased search loads effectively by spreading storage and query handling across different nodes.

The nodes shown here (Node 1, Node 2, Node 3, and Node 4) each host one or more shards, illustrated as green blocks (`[##]`). This distribution lets Elasticsearch scale horizontally, distributing both data and search processes, which enhances both availability and query performance.

### Segments and Merging

Each shard is further composed of smaller data blocks known as **segments**. Segments are immutable data units within a shard that store indexed documents. As data is added or updated, Elasticsearch writes to new segments, and over time, smaller segments are merged into larger ones through **segment merging**. Merging is a background process that optimizes disk usage by reducing the total number of segments, which also improves search performance.

In the ASCII diagram, the multiple `[##]` blocks in some nodes represent individual segments within a shard. Over time, these smaller segments in Node 1, for example, may be merged into fewer, larger segments (as represented by the single `[##]` block in Node 2). This merging keeps the system efficient by minimizing the number of segments Elasticsearch has to search through.

### Horizontal Scaling and Performance

By distributing shards across multiple nodes, Elasticsearch not only expands data storage capabilities but also enhances **query execution efficiency**. This architecture provides high availability and fault tolerance. When a node fails, Elasticsearch can still access the data from other nodes, ensuring continuity and minimizing downtime. As nodes are added to the cluster, Elasticsearch can handle more data and queries, effectively scaling both **storage capacity** and **performance**.

In the scenario shown here, each node contributes to both storage and processing power, ensuring balanced performance across the cluster. Query requests are distributed across the nodes, and the cluster responds to the user’s request by retrieving data from multiple nodes in parallel, improving speed and resiliency.

### Replica Shards

To prevent data loss and increase fault tolerance, Elasticsearch allows the creation of **replica shards**, which are copies of the primary shards distributed on different nodes. These replicas ensure that if a primary shard becomes unavailable, the replica can fulfill requests, safeguarding the data against node failures. 

While not shown explicitly in this diagram, in a more detailed version, primary and replica shards could be represented across nodes to illustrate data redundancy and improved fault tolerance. For instance, if Node 1 hosts a primary shard, a replica of that shard might be stored on Node 2 or Node 3, ensuring that data remains accessible even if a single node goes down. This distribution of replicas across nodes provides redundancy and enhances the cluster's durability and stability.
