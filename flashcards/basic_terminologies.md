### 1. **What is an Elasticsearch cluster?**

A cluster is a collection of one or more nodes that together hold all your data and provide federated indexing and search capabilities; nodes discover each other by sharing the same cluster.name setting, and the cluster collectively manages shard allocation, replication, and failover.

### 2. **What are the different node types in Elasticsearch?**

Master-eligible nodes participate in cluster state management and leader election, data nodes store shards and execute search and indexing operations, ingest nodes run pre-processing pipelines on documents before indexing, coordinating nodes route requests and merge results from data nodes, and machine-learning nodes run anomaly detection jobs; a single node can hold multiple roles simultaneously.

### 3. **What is a document in Elasticsearch?**

A document is the basic unit of information stored as a JSON object, uniquely identified by an _index and _id, containing an _source field with the original data; documents are schema-free by default but follow the mapping defined on the index once fields are established.

### 4. **What is an index in Elasticsearch?**

An index is a logical namespace that maps to one or more primary shards and their replicas, storing a collection of related documents with a shared mapping; it is roughly analogous to a database in relational systems and is the primary target for search and aggregation queries.

### 5. **What is a shard and why are shards important?**

A shard is an independent Lucene index that holds a subset of an index's documents; Elasticsearch splits every index into multiple shards so data can be distributed across nodes for parallel processing, and each shard can be replicated for fault tolerance and increased read throughput.

### 6. **What is the difference between a primary shard and a replica shard?**

A primary shard receives all write operations and is the authoritative copy of a subset of an index's data, while a replica shard is a copy of a primary that serves read requests and can be promoted to primary if the original fails; the number of primaries is fixed at index creation, but replicas can be adjusted dynamically.

### 7. **What is a mapping in Elasticsearch?**

A mapping defines the schema of an index by specifying each field's data type (text, keyword, integer, date, geo_point, etc.), its analyzer, and indexing options; once a field is mapped its type cannot be changed without reindexing, though new fields can be added dynamically.

### 8. **What is the difference between `text` and `keyword` field types?**

A text field is analyzed at index time into tokens for full-text search and cannot be used for exact matching or sorting, while a keyword field stores the original value as a single token enabling exact-match filtering, sorting, and aggregations; most string fields are mapped as multi-fields with both types by default.

### 9. **What is the inverted index?**

An inverted index is a data structure that maps each unique term to the set of documents (and positions within those documents) that contain it, enabling Elasticsearch to resolve search queries in near-constant time by looking up terms rather than scanning every document.

### 10. **What is term frequency (TF) and inverse document frequency (IDF)?**

Term frequency measures how often a term appears in a single document (higher TF indicates greater relevance to that term), while inverse document frequency measures how rare the term is across all documents in the index (rarer terms receive higher weight); together they form the basis of TF-IDF relevance scoring.

### 11. **How does the BM25 scoring algorithm improve on TF-IDF?**

BM25 applies a saturation function to term frequency so that repeated occurrences of a term yield diminishing returns, incorporates document length normalization to avoid unfairly boosting long documents, and uses tunable k1 and b parameters, making it a more accurate default relevance model than raw TF-IDF.

### 12. **What is a Lucene segment?**

A segment is an immutable, self-contained Lucene index file within a shard that contains its own inverted index, stored fields, and doc values; new documents are written to new segments and deleted documents are only marked as deleted until a merge combines segments and physically removes them.

### 13. **What is tokenization?**

Tokenization is the process of breaking a string of text into individual terms (tokens) using a tokenizer, such as splitting on whitespace or punctuation, so that each token can be stored in the inverted index and matched during searches.

### 14. **What are doc values and when are they used?**

Doc values are an on-disk columnar data structure built at index time that stores field values per document, enabling efficient sorting, aggregations, and scripting without loading data into the JVM heap; they are enabled by default for all non-text fields.

### 15. **What is the `_id` field and how is it assigned?**

The _id field is a unique identifier for each document within an index; it can be specified explicitly by the client during indexing or auto-generated by Elasticsearch as a URL-safe Base64-encoded UUID, and it is used internally for document routing, retrieval, and update operations.
