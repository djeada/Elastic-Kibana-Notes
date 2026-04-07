## Basic Terminologies

Elasticsearch is a distributed, open-source search and analytics engine built on top of Apache Lucene. It is designed for horizontal scalability, real-time search, and multi-tenant capable full-text search across large volumes of data. Understanding the core terminology is essential for anyone working with Elasticsearch, whether you are building search-powered applications, analyzing log data, or managing observability pipelines through tools like Kibana.

The following sections cover each foundational concept in depth, beginning with the high-level architectural components and progressing toward the internal mechanisms that make Elasticsearch fast and reliable.


### Cluster

A cluster is a collection of one or more nodes (servers) that work together to store and manage your data, providing indexing and search capabilities across all nodes. The cluster coordinates data distribution and load balancing, ensuring fault tolerance and scalability. Every cluster is identified by a unique name, which defaults to "elasticsearch" if not explicitly configured. All nodes that share the same cluster name automatically discover each other and form a single cooperative unit.

A cluster continuously monitors the health of its constituent nodes and redistributes shards when nodes join or leave. Cluster health is reported as one of three states: green (all primary and replica shards are allocated), yellow (all primary shards are allocated but some replicas are not), or red (some primary shards are unallocated, meaning data loss is possible). Understanding cluster health is one of the first things an operator should check when troubleshooting.

```
Cluster Health States
---------------------

  GREEN            YELLOW              RED
  All shards       All primaries       Some primaries
  allocated        allocated,          unallocated
                   some replicas
                   missing

  [OK]             [WARN]              [CRITICAL]
```


### Node

A node is a single server instance within your cluster that stores a portion of your data and participates in indexing and searching. Each node is identified by a randomly assigned UUID at startup, though it can be given a human-readable name via configuration. Nodes communicate with each other using a transport protocol and can be configured to serve different roles depending on the operational requirements of your deployment.

When a node starts, it uses a discovery mechanism (seed hosts) to find and join an existing cluster. If no cluster is found and the node is configured to be eligible, it may bootstrap a new cluster. In production deployments, it is common to separate node responsibilities so that resource-intensive operations like indexing do not interfere with cluster coordination.

```
  Single Node Anatomy
  --------------------

  +---------------------------------------------+
  |  Node: "node-1"  (UUID: abc-123-def)        |
  |                                              |
  |  Roles: master, data, ingest                 |
  |                                              |
  |  +----------+  +----------+  +----------+   |
  |  | Shard 0  |  | Shard 1  |  | Shard 2  |   |
  |  | (primary)|  | (replica)|  | (primary)|   |
  |  +----------+  +----------+  +----------+   |
  |                                              |
  |  JVM Heap: 4 GB    Disk: 500 GB             |
  +---------------------------------------------+
```


### Node Roles

In a production Elasticsearch cluster, nodes are typically assigned specific roles to optimize resource usage and stability. A single node can hold multiple roles, but larger deployments benefit from dedicated role assignment.

Master-eligible node: A node that can be elected as the cluster master. The master node is responsible for lightweight cluster-wide operations such as creating or deleting indices, tracking which nodes are part of the cluster, and deciding which shards to allocate to which nodes. It is critical that master-eligible nodes are not overloaded with heavy data operations, as a slow master can destabilize the entire cluster. A cluster should have an odd number of master-eligible nodes (commonly three) to avoid split-brain scenarios.

Data node: A node that holds data shards and executes data-related operations such as indexing, searching, and aggregations. Data nodes are typically the most resource-intensive nodes in a cluster, requiring substantial disk, memory, and CPU. In larger deployments, data nodes may be further specialized into hot, warm, and cold tiers based on the age and access frequency of the data they hold.

Ingest node: A node that preprocesses documents before they are indexed. Ingest nodes execute ingest pipelines, which are sequences of processors that can transform, enrich, or filter documents. Common operations include parsing timestamps, converting data types, renaming fields, or running GeoIP lookups. Any node can serve as an ingest node by default, but dedicating nodes to this role avoids contention with search workloads.

Coordinating node: Every node in Elasticsearch acts as a coordinating node by default. When a search or bulk request arrives, the coordinating node routes the request to the appropriate data nodes, gathers partial results, and merges them into a final response. In high-throughput environments, dedicated coordinating-only nodes (nodes with no other roles) can be used to offload the scatter-gather overhead from data nodes.

```
  Node Role Summary
  -----------------

  +-------------------+----------------------------------------------+
  | Role              | Primary Responsibility                       |
  +-------------------+----------------------------------------------+
  | Master-eligible   | Cluster state management, shard allocation   |
  | Data              | Store shards, execute search and indexing     |
  | Ingest            | Preprocess documents via pipelines            |
  | Coordinating      | Route requests, merge results                |
  +-------------------+----------------------------------------------+

  Typical Production Layout (5 nodes):

  +------------+    +------------+    +------------+
  |  Master 1  |    |  Master 2  |    |  Master 3  |
  | (dedicated)|    | (dedicated)|    | (dedicated)|
  +------------+    +------------+    +------------+

  +--------------------+    +--------------------+
  |   Data Node 1      |    |   Data Node 2      |
  | (data + ingest)    |    | (data + ingest)    |
  +--------------------+    +--------------------+
```


### Index

An index is a logical namespace that maps to a collection of documents sharing similar characteristics. It is the primary unit of organization in Elasticsearch, analogous in some respects to a database table in a relational system, though the comparison is imperfect. In an e-commerce context, for example, one index might contain product listings while another contains customer reviews. In log management, a common pattern is to create time-based indices such as "logs-2024.01.15" to manage data lifecycle efficiently.

Each index has a set of configurable settings that control its behavior, including the number of primary shards, the number of replicas, refresh intervals, and analysis configurations. Once an index is created, the number of primary shards cannot be changed without reindexing, making capacity planning an important step. The number of replicas, however, can be adjusted dynamically.

Indices can also be grouped using aliases, which provide a layer of indirection. An alias can point to one or many indices, enabling use cases like zero-downtime reindexing, where a new index is built behind the scenes and the alias is atomically switched to point to the new index.

```
  Index Structure
  ---------------

  Index: "products"
  Settings: 3 primary shards, 1 replica

  +----------------------------------------------------+
  |                  Index: products                    |
  |                                                    |
  |  +-------------+  +-------------+  +-------------+ |
  |  | Shard 0 (P) |  | Shard 1 (P) |  | Shard 2 (P) | |
  |  +-------------+  +-------------+  +-------------+ |
  |  | Shard 0 (R) |  | Shard 1 (R) |  | Shard 2 (R) | |
  |  +-------------+  +-------------+  +-------------+ |
  |                                                    |
  |  P = Primary    R = Replica                        |
  +----------------------------------------------------+
```


### Document

A document is the basic unit of information that can be indexed and retrieved. Each document is a JSON object representing a real-world entity, such as a single product in an e-commerce index, a log entry from an application, or a user profile in a social platform. Documents contain fields, which are key-value pairs that describe the properties of the entity.

Every document within an index is assigned a unique identifier (the `_id` field). This identifier can be explicitly provided at index time or automatically generated by Elasticsearch. Documents also carry metadata fields such as `_index` (the index the document belongs to), `_source` (the original JSON body), and `_version` (a counter incremented on each update).

Documents in Elasticsearch are immutable. When you "update" a document, Elasticsearch internally marks the old version as deleted and indexes a new version. Deleted documents are eventually purged during segment merges, a background process managed by Lucene.

```
  Document Example
  ----------------

  {
    "_index": "products",
    "_id": "1001",
    "_version": 3,
    "_source": {
      "name": "Wireless Keyboard",
      "category": "electronics",
      "price": 49.99,
      "in_stock": true,
      "tags": ["wireless", "bluetooth", "keyboard"],
      "created_at": "2024-01-15T10:30:00Z"
    }
  }

  Document fields can be of various types:
    - text / keyword    (string data)
    - long / double     (numeric data)
    - boolean           (true / false)
    - date              (timestamps)
    - object / nested   (structured sub-documents)
    - geo_point         (latitude / longitude)
```


### Shard

A shard is a subdivided piece of an index. Each shard is a fully functional, self-contained Lucene index that can be hosted on any node in the cluster. Sharding is the fundamental mechanism that allows Elasticsearch to scale horizontally: by splitting an index into multiple shards, the data and computational load can be distributed across many nodes, enabling parallel processing of search and indexing operations.

The number of primary shards is defined when the index is created and determines the maximum parallelism for that index. Choosing the right shard count involves balancing several factors: too few shards can create hotspots on individual nodes, while too many shards introduce overhead in cluster state management and reduce the efficiency of each individual shard. A common guideline is to aim for shard sizes between 10 GB and 50 GB for most workloads.

When a document is indexed, Elasticsearch determines which shard it belongs to using the formula:

```
  shard_number = hash(_routing) % number_of_primary_shards
```

By default, `_routing` is the document's `_id`, but custom routing can be configured to co-locate related documents on the same shard, which is useful for optimizing join-like queries.

```
  Shard Distribution Across Nodes
  --------------------------------

  Node A               Node B               Node C
  +-----------+        +-----------+        +-----------+
  | Shard 0 P |        | Shard 1 P |        | Shard 2 P |
  | Shard 1 R |        | Shard 2 R |        | Shard 0 R |
  +-----------+        +-----------+        +-----------+

  P = Primary shard    R = Replica shard

  If Node A fails:
  - Shard 0 R on Node C is promoted to primary
  - Shard 1 P on Node B continues serving
  - Cluster reallocates a new replica for Shard 0
```


### Replica Shards

Replica shards are additional copies of primary shards within an index. Each replica is an exact duplicate of its corresponding primary shard and serves two purposes: increasing data redundancy for fault tolerance and improving read throughput by allowing search requests to be served from either the primary or any of its replicas.

In the event of a node failure, replica shards ensure that no data is lost and queries can continue without interruption. The cluster automatically promotes a replica to primary status when the original primary becomes unavailable, then attempts to allocate new replicas on the remaining healthy nodes to restore the desired replication factor.

By default, Elasticsearch creates one replica per primary shard, but this is configurable. Write-heavy workloads may temporarily reduce the replica count to improve indexing speed, while read-heavy workloads may increase it to distribute search load across more nodes.

Replica shards are never allocated on the same node as their corresponding primary shard. This guarantees that a single node failure cannot cause the loss of both the primary and its replica.

```
  Replication Diagram
  -------------------

  Write path (indexing):

    Client
      |
      v
    Coordinating Node
      |
      +----------> Primary Shard (Node A)
                      |
                      +----> Replica Shard (Node B)  [sync]
                      +----> Replica Shard (Node C)  [sync]
                      |
                   Response sent after all
                   in-sync replicas acknowledge

  Read path (searching):

    Client
      |
      v
    Coordinating Node
      |
      +----> Primary OR Replica  (load balanced)
```


### Mapping

A mapping defines the schema of an index: which fields exist, what data type each field has, and how each field should be indexed and stored. Mappings are roughly analogous to table schemas in relational databases, though Elasticsearch supports a much wider variety of field types and indexing strategies.

Elasticsearch supports two modes of mapping: dynamic and explicit. With dynamic mapping, Elasticsearch automatically infers the field type when it encounters a new field in an incoming document. For example, a string that looks like a date will be mapped as a date field. While convenient for exploration, dynamic mapping can produce suboptimal results in production, such as mapping a numeric identifier as a `long` when `keyword` would be more appropriate.

Explicit mapping gives you full control. You define each field's type and options before indexing data. This is the recommended approach for production indices, as it ensures consistent behavior and optimal storage.

Important mapping properties include:

- `type`: The data type of the field (text, keyword, long, double, date, boolean, etc.)
- `analyzer`: Which analyzer to use for full-text fields
- `index`: Whether the field should be searchable (default: true)
- `doc_values`: Whether column-oriented storage is enabled for sorting and aggregations
- `store`: Whether the original field value is stored separately from `_source`

```
  Mapping Example
  ---------------

  PUT /products
  {
    "mappings": {
      "properties": {
        "name":        { "type": "text", "analyzer": "standard" },
        "sku":         { "type": "keyword" },
        "price":       { "type": "double" },
        "in_stock":    { "type": "boolean" },
        "description": { "type": "text", "analyzer": "english" },
        "created_at":  { "type": "date" },
        "location":    { "type": "geo_point" }
      }
    }
  }

  Key rule: Once a field is mapped, its type cannot be changed
  without reindexing. New fields can be added at any time.
```


### Analyzer

An analyzer is a component that processes text during indexing and search time. When a text field is indexed, the analyzer breaks the raw text into individual tokens (terms) that are stored in the inverted index. The same or a different analyzer is applied at search time to ensure the query terms match the indexed terms.

An analyzer consists of three building blocks, applied in order:

1. Character filters: Transform the raw text before tokenization. Examples include stripping HTML tags, replacing characters, or normalizing Unicode.

2. Tokenizer: Splits the character-filtered text into individual tokens. The standard tokenizer, for instance, splits on whitespace and punctuation. Other tokenizers include `whitespace` (split on whitespace only), `keyword` (treat entire input as one token), and `ngram` (generate substrings of specified length).

3. Token filters: Post-process the tokens produced by the tokenizer. Common operations include lowercasing, removing stop words (e.g., "the", "is", "at"), stemming (reducing words to their root form), and synonym expansion.

```
  Analyzer Pipeline
  -----------------

  Input text: "<p>The Quick Brown Foxes jumped!</p>"

  Step 1 - Character Filter (html_strip):
    "The Quick Brown Foxes jumped!"

  Step 2 - Tokenizer (standard):
    ["The", "Quick", "Brown", "Foxes", "jumped"]

  Step 3 - Token Filters:
    lowercase:     ["the", "quick", "brown", "foxes", "jumped"]
    stop words:    ["quick", "brown", "foxes", "jumped"]
    stemmer:       ["quick", "brown", "fox", "jump"]

  Final tokens stored in inverted index:
    ["quick", "brown", "fox", "jump"]
```

Built-in analyzers include `standard` (general-purpose, language-agnostic), `simple` (splits on non-letter characters and lowercases), `whitespace` (splits on whitespace only), and language-specific analyzers like `english`, `french`, and `german` that apply appropriate stemming and stop-word rules.


### Inverted Index

The inverted index is the core data structure that powers full-text search in Elasticsearch (and in Lucene, the library it is built on). Rather than storing documents and then scanning each one for matching terms, an inverted index maps every unique term to the list of documents that contain it. This inversion makes search extremely fast, even across millions of documents.

When a document is indexed, its text fields are passed through an analyzer that produces a list of terms. Each term is recorded in the inverted index along with metadata such as the document ID, the position of the term within the document, and the term frequency. This metadata enables features like phrase queries, proximity searches, and relevance scoring.

```
  Inverted Index Example
  ----------------------

  Documents:
    Doc 1: "Elasticsearch is fast"
    Doc 2: "Elasticsearch is scalable"
    Doc 3: "Lucene is fast and scalable"

  After analysis (lowercase + standard tokenizer):

  Term            | Document IDs  | Positions
  ----------------+---------------+-----------------
  "elasticsearch" | Doc 1, Doc 2  | Doc1:[0], Doc2:[0]
  "is"            | Doc 1, Doc 2, | Doc1:[1], Doc2:[1],
                  |   Doc 3       |   Doc3:[1]
  "fast"          | Doc 1, Doc 3  | Doc1:[2], Doc3:[2]
  "scalable"      | Doc 2, Doc 3  | Doc2:[2], Doc3:[4]
  "lucene"        | Doc 3         | Doc3:[0]
  "and"           | Doc 3         | Doc3:[3]

  Search for "fast":
    -> Look up "fast" in the index
    -> Return Doc 1 and Doc 3
    -> No need to scan any other documents
```

Each shard maintains its own inverted index. Within a shard, the inverted index is further divided into immutable segments. When new documents are indexed, they are written to a new segment. Periodically, segments are merged in the background to reclaim space from deleted documents and improve search performance.


### Relevance Score

When Elasticsearch executes a search query, each matching document receives a relevance score (stored in the `_score` field) that represents how well the document matches the query. Results are returned sorted by this score in descending order by default.

Elasticsearch uses the BM25 algorithm (Best Matching 25) as its default scoring function. BM25 considers three main factors:

1. Term frequency (TF): How often the search term appears in the document. More occurrences indicate higher relevance, but with diminishing returns.

2. Inverse document frequency (IDF): How rare or common the term is across all documents. Rare terms contribute more to the score than common ones. A search for "elasticsearch" in an index where only 5 out of 10,000 documents contain the term will score those 5 documents much higher than a search for "the" which appears in nearly every document.

3. Field length normalization: Shorter fields receive a higher score than longer fields for the same term match. A match in a title field (few words) is considered more relevant than the same match in a body field (many words).

```
  BM25 Scoring Intuition
  ----------------------

  Query: "fast search"

  Doc A (title: "Fast search engine")
    - "fast":   TF=1, IDF=high, field_len=short  -> high score
    - "search": TF=1, IDF=medium, field_len=short -> medium score
    - Combined: HIGH

  Doc B (body: "...this is a fast way to search through millions...")
    - "fast":   TF=1, IDF=high, field_len=long    -> moderate score
    - "search": TF=1, IDF=medium, field_len=long   -> lower score
    - Combined: MODERATE

  Doc C (body: "...search search search fast fast fast...")
    - "fast":   TF=3, IDF=high, field_len=long    -> moderate (diminishing TF)
    - "search": TF=3, IDF=medium, field_len=long   -> moderate
    - Combined: MODERATE-HIGH

  Final ranking: Doc A > Doc C > Doc B
```

Elasticsearch also supports function score queries, boosting, and custom scoring scripts for cases where BM25 alone does not capture the desired ranking behavior. For example, you might boost newer documents or factor in a popularity metric.


### How It All Fits Together

The following diagram shows the hierarchical relationship between the major Elasticsearch concepts, from the cluster level down to individual terms in the inverted index.

```
+=========================================================================+
|                                                                         |
|   CLUSTER: "my-application"             Health: GREEN                   |
|                                                                         |
|   +------------------------------+    +------------------------------+  |
|   |        NODE 1                |    |        NODE 2                |  |
|   |   Role: master, data         |    |   Role: data, ingest         |  |
|   |                              |    |                              |  |
|   | +----------+ +----------+   |    | +----------+ +----------+   |  |
|   | |Index:    | |Index:    |   |    | |Index:    | |Index:    |   |  |
|   | |products  | |products  |   |    | |products  | |products  |   |  |
|   | |Shard 0(P)| |Shard 1(R)|   |    | |Shard 1(P)| |Shard 0(R)|   |  |
|   | +----+-----+ +----------+   |    | +----+-----+ +----------+   |  |
|   |      |                       |    |      |                       |  |
|   +------|------------------------+    +------|------------------------+  |
|          |                                    |                          |
|          v                                    v                          |
|   +-------------+                     +-------------+                   |
|   | Documents:  |                     | Documents:  |                   |
|   |  doc_1001   |                     |  doc_1002   |                   |
|   |  doc_1003   |                     |  doc_1004   |                   |
|   |  doc_1005   |                     |  doc_1006   |                   |
|   +------+------+                     +------+------+                   |
|          |                                    |                          |
|          v                                    v                          |
|   +--------------+                    +--------------+                  |
|   |  Inverted    |                    |  Inverted    |                  |
|   |  Index       |                    |  Index       |                  |
|   |              |                    |              |                  |
|   | "keyboard"   |                    | "mouse"      |                  |
|   |   -> 1001    |                    |   -> 1002    |                  |
|   |   -> 1005    |                    |   -> 1004    |                  |
|   | "wireless"   |                    | "bluetooth"  |                  |
|   |   -> 1001    |                    |   -> 1002    |                  |
|   |   -> 1003    |                    |   -> 1006    |                  |
|   +--------------+                    +--------------+                  |
|                                                                         |
+=========================================================================+

Data Flow:

  1. Client sends index request to coordinating node
  2. Document is routed to the correct primary shard:
       shard = hash(doc._id) % num_primary_shards
  3. Primary shard indexes the document:
       a. Analyzer processes text fields into tokens
       b. Tokens are added to the inverted index
       c. Original document is stored in _source
  4. Primary shard forwards the operation to replica shards
  5. Once all in-sync replicas acknowledge, success is returned

  Search follows the reverse path:
  1. Query arrives at coordinating node
  2. Query is broadcast to one copy of each shard (primary or replica)
  3. Each shard searches its local inverted index
  4. Partial results are returned to the coordinating node
  5. Coordinating node merges, sorts by _score, and returns final results
```


### Command Reference

The following table lists commonly used Elasticsearch REST API commands for inspecting and managing each concept discussed above. All commands assume a default Elasticsearch instance running on localhost:9200.

```
  Cluster Commands
  ---------------------------------------------------------------
  GET /_cluster/health              Cluster health status
  GET /_cluster/state               Full cluster state
  GET /_cluster/stats               Cluster-wide statistics
  GET /_cluster/settings            Cluster settings

  Node Commands
  ---------------------------------------------------------------
  GET /_cat/nodes?v                 List all nodes with details
  GET /_nodes                       Detailed node information
  GET /_nodes/stats                 Node-level statistics
  GET /_nodes/hot_threads           Identify busy threads

  Index Commands
  ---------------------------------------------------------------
  GET /_cat/indices?v               List all indices
  PUT /my-index                     Create an index
  DELETE /my-index                  Delete an index
  GET /my-index/_settings           View index settings
  GET /my-index/_mapping            View index mapping
  POST /my-index/_open              Open a closed index
  POST /my-index/_close             Close an index

  Document Commands
  ---------------------------------------------------------------
  POST /my-index/_doc               Index a document (auto ID)
  PUT /my-index/_doc/1              Index a document (explicit ID)
  GET /my-index/_doc/1              Retrieve a document by ID
  DELETE /my-index/_doc/1           Delete a document by ID
  POST /my-index/_update/1          Partial update a document
  POST /my-index/_search            Search documents

  Shard Commands
  ---------------------------------------------------------------
  GET /_cat/shards?v                List all shards
  GET /_cat/shards/my-index?v       Shards for a specific index
  GET /_cluster/allocation/explain  Explain shard allocation
  POST /_cluster/reroute            Manually move shards

  Mapping and Analysis Commands
  ---------------------------------------------------------------
  GET /my-index/_mapping            View current mapping
  PUT /my-index/_mapping            Update mapping (add fields)
  POST /_analyze                    Test an analyzer
  GET /my-index/_analyze            Test analyzer on an index

  Monitoring Commands
  ---------------------------------------------------------------
  GET /_cat/health?v                Quick cluster health
  GET /_cat/recovery?v              Shard recovery status
  GET /_cat/thread_pool?v           Thread pool statistics
  GET /_cat/pending_tasks?v         Pending cluster tasks
```

Example usage with curl:

```
  # Check cluster health
  curl -X GET "localhost:9200/_cluster/health?pretty"

  # List all indices
  curl -X GET "localhost:9200/_cat/indices?v"

  # Create an index with custom settings
  curl -X PUT "localhost:9200/products" -H 'Content-Type: application/json' -d'
  {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "name": { "type": "text" },
        "price": { "type": "double" }
      }
    }
  }'

  # Index a document
  curl -X POST "localhost:9200/products/_doc" -H 'Content-Type: application/json' -d'
  {
    "name": "Wireless Keyboard",
    "price": 49.99
  }'

  # Search for documents
  curl -X POST "localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
  {
    "query": {
      "match": { "name": "wireless" }
    }
  }'

  # Test an analyzer
  curl -X POST "localhost:9200/_analyze" -H 'Content-Type: application/json' -d'
  {
    "analyzer": "standard",
    "text": "The Quick Brown Foxes jumped!"
  }'
```


### Summary

The concepts covered in this document form the foundation of Elasticsearch's architecture:

- A cluster is the top-level organizational unit, composed of nodes.
- Nodes serve specialized roles (master, data, ingest, coordinating) and host shards.
- An index is a logical collection of documents, divided into shards for distribution.
- Documents are JSON objects representing individual records, stored within shards.
- Shards are self-contained Lucene indices; primary shards handle writes, replica shards provide redundancy and read scaling.
- Mappings define the schema and control how fields are indexed and stored.
- Analyzers process text into tokens for storage in the inverted index.
- The inverted index maps terms to documents, enabling fast full-text search.
- Relevance scoring (BM25) ranks results by how well they match the query.

Together, these components allow Elasticsearch to ingest, store, and search large volumes of data with low latency and high availability.
