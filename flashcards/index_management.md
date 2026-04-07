### 1. **How do you create an index in Elasticsearch?**

Send a PUT request to the index name endpoint with an optional body specifying settings (number_of_shards, number_of_replicas, refresh_interval) and mappings (field types and analyzers); if no body is provided Elasticsearch creates the index with default settings and allows dynamic mapping to infer field types from the first indexed documents.

### 2. **What is the difference between dynamic mapping and explicit mapping?**

Dynamic mapping lets Elasticsearch automatically detect and assign field types when new fields appear in indexed documents, which is convenient but can produce suboptimal types (e.g., mapping a numeric string as text); explicit mapping requires you to define every field's type and analyzer up front, giving full control over how data is indexed and queried.

### 3. **What is an index template and when should you use one?**

An index template is a predefined set of settings, mappings, and aliases that are automatically applied to new indices whose names match a specified pattern, which is essential for time-series data where new indices are created regularly (e.g., logs-2024-01) and you want consistent configuration without manual setup.

### 4. **What is an index alias?**

An index alias is a virtual name that points to one or more concrete indices, allowing transparent index swapping during reindexing, zero-downtime rollover of time-based indices, filtered views for multi-tenant access, and simplified application configuration where the alias name never changes even as underlying indices rotate.

### 5. **What are the main field data types in Elasticsearch?**

Core types include text (analyzed full-text), keyword (exact-value strings), long/integer/short/byte/double/float (numerics), date (timestamps), boolean, ip, geo_point and geo_shape (geographic), nested and object (structured sub-documents), and binary (Base64-encoded blobs); choosing the right type affects search capabilities, storage size, and query performance.

### 6. **What is multi-field mapping?**

Multi-field mapping uses the fields parameter to index the same source field in multiple ways under different sub-field names, such as mapping a string as both text (for full-text search) and keyword (for sorting and aggregations), so that a single field supports different query types without duplicating data in the source document.

### 7. **What does the `copy_to` parameter do?**

The copy_to parameter copies the value of one or more fields into a designated target field at index time, allowing you to create a combined field that can be searched without running expensive multi-field queries; the target field exists only in the index and does not appear in the stored _source document.

### 8. **What is a normalizer and when is it used?**

A normalizer is like an analyzer for keyword fields that applies character filters and token filters (such as lowercase) to the entire value as a single token without tokenization, ensuring that keyword searches, aggregations, and sorting treat values like "Foo" and "foo" as identical.

### 9. **What is index lifecycle management (ILM)?**

ILM is a policy-driven framework that automates the progression of indices through phases—hot (active writes and searches), warm (read-only, possibly fewer replicas), cold (infrequent access, maybe frozen), and delete (removal)—based on age, size, or document count thresholds, reducing operational overhead for time-series and log data.

### 10. **What is index rollover?**

Rollover automatically creates a new index when the current write index meets specified conditions (max age, max docs, or max size), aliases the new index as the active write target, and marks the old index as read-only, which keeps individual index sizes manageable and works hand-in-hand with ILM policies.

### 11. **How do you close and reopen an index?**

Closing an index releases its shard resources (memory and file handles) while preserving the data on disk, making it invisible to search and indexing operations; reopening the index reallocates its shards and makes it fully operational again, which is useful for temporarily freeing resources on infrequently accessed indices.

### 12. **What is the `_reindex` API used for in index management?**

The _reindex API copies documents from a source index to a destination index, optionally transforming them with a script or ingest pipeline, which is the standard way to change mappings, adjust shard counts, or apply new analyzers since existing field mappings cannot be altered in place.

### 13. **What is the shrink API?**

The shrink API reduces the number of primary shards in an index by creating a new index with fewer shards and hard-linking the existing segments, which is useful when an index was originally over-sharded; the source index must be read-only and all shards must be relocated to a single node before the shrink can proceed.

### 14. **What are index settings categories?**

Index settings are divided into static settings (like number_of_shards) that can only be set at creation time, and dynamic settings (like number_of_replicas, refresh_interval, and routing allocation filters) that can be changed on a live index using the PUT _settings API without requiring a reindex.

### 15. **How do you delete an index safely?**

Send a DELETE request to the index name; to prevent accidental deletion of all indices, set action.destructive_requires_name to true in the cluster settings so that wildcard or _all deletions are forbidden, and always use index aliases in applications so the application layer is decoupled from the physical index name.
