### 1. **What are the main HTTP methods used in the Elasticsearch REST API?**

Elasticsearch uses GET to retrieve documents and index metadata, PUT to create or replace indices, mappings, and documents with explicit IDs, POST to index documents with auto-generated IDs or execute searches and bulk requests, DELETE to remove documents or indices, and HEAD to check existence without returning a body.

### 2. **What is the general structure of an Elasticsearch REST endpoint?**

An Elasticsearch endpoint follows the pattern http://host:port/index/_endpoint where the index name identifies the target index, the endpoint specifies the operation (such as _search, _doc, _mapping, _settings, or _bulk), and optional query parameters like ?pretty, ?timeout, or ?refresh modify the request behavior.

### 3. **What is the difference between PUT and POST when indexing documents?**

PUT requires you to specify a document ID in the URL and creates or fully replaces the document at that ID, while POST to the _doc endpoint auto-generates a unique ID for the new document; use PUT when you need deterministic IDs and POST when you want Elasticsearch to assign them.

### 4. **What does the `?pretty` query parameter do?**

The ?pretty parameter tells Elasticsearch to format the JSON response with indentation and line breaks for human readability; it is useful during development and debugging but should be omitted in production clients because it adds overhead to response serialization.

### 5. **How does the `?refresh` parameter work?**

The refresh parameter controls when changes become visible to search: refresh=true forces an immediate refresh making the change searchable right away but at a performance cost, refresh=wait_for waits until the next scheduled refresh before returning, and omitting it (default) lets the normal refresh_interval handle visibility.

### 6. **What is the `?timeout` parameter used for?**

The timeout parameter sets the maximum time a request will wait for a response from each shard; if a shard does not respond within the specified duration the request returns partial results with a timed_out flag set to true, allowing clients to handle slow shards gracefully.

### 7. **What information does the `_source` field contain in a response?**

The _source field holds the original JSON document body exactly as it was indexed, and you can control which fields are returned using _source_includes and _source_excludes parameters or the _source field in the request body to reduce network transfer and parsing overhead.

### 8. **What are the common HTTP status codes returned by Elasticsearch?**

200 indicates success, 201 means a resource was created, 400 signals a malformed request, 404 means the index or document was not found, 409 indicates a version conflict, 429 means too many requests (circuit breaker or queue full), and 500 signals an internal server error; the response body always contains a detailed error object for non-2xx codes.

### 9. **How does routing work in the Elasticsearch API?**

By default Elasticsearch hashes the document ID to determine which shard stores the document; you can override this with a custom ?routing parameter so that related documents land on the same shard, which improves query performance when you always filter by the routing value and is required for parent-child joins.

### 10. **What is the Bulk API request format?**

The Bulk API uses newline-delimited JSON (NDJSON) where each operation consists of an action line (index, create, update, or delete with metadata) followed by an optional document body line, and the entire payload must end with a trailing newline; this format lets a single HTTP request carry thousands of operations.

### 11. **What does the `_cat` API provide?**

The _cat API returns cluster, node, index, shard, and allocation information in a compact, human-readable tabular format instead of JSON, with endpoints like _cat/health, _cat/indices, _cat/nodes, and _cat/shards; it accepts ?v for column headers and ?help to list available columns.

### 12. **How do you use multi-index API calls?**

You can target multiple indices in a single request by separating index names with commas (index1,index2), using wildcard patterns (logs-*), or specifying _all to target every index; this is useful for searching across time-based indices, applying bulk settings changes, or running cross-index aggregations.

### 13. **What is the `_bulk` error handling behavior?**

The Bulk API does not fail atomically; each sub-operation succeeds or fails independently, so the response contains an items array with a status and optional error for every operation, and clients must iterate through this array to detect and handle individual failures such as version conflicts or mapping errors.

### 14. **What are common request body conventions in the Elasticsearch API?**

Request bodies are JSON objects; search requests use a query key for the DSL, an aggs key for aggregations, and optional sort, from, size, and _source keys; index requests pass the document directly as the body; and update requests wrap partial documents inside a doc key or provide a script key for scripted updates.

### 15. **What does the `flat_settings` parameter do?**

The flat_settings parameter returns index or cluster settings as a flat key-value map with dot-delimited keys (e.g., index.number_of_replicas) instead of the default nested JSON structure, making it easier to scan settings programmatically or in scripts.
