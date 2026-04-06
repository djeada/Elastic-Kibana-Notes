## API Convention

The Elasticsearch API provides a RESTful interface for interacting with and managing Elasticsearch clusters, indices, and documents, making it easy to build, monitor, and analyze data workflows. Through simple HTTP methods, users can perform essential operations like creating, retrieving, updating, and deleting documents, as well as manage indices and check cluster health. Cluster-level endpoints allow for monitoring overall health (`/_cat/health`), listing nodes, and obtaining detailed stats (`/_cluster/stats`), while index-level endpoints support creating and deleting indices, managing settings, and handling aliases. Document-related endpoints provide options for indexing data, searching, updating, and deleting, while more advanced features like bulk operations, reindexing, and snapshots enable efficient data handling at scale. The API's flexibility and range of operations allow developers to efficiently manage and query Elasticsearch data, making it adaptable for various use cases from simple document storage to complex data analysis and search functionalities.

### Request and Response Flow

The following diagram illustrates how a typical API request travels from the client through the Elasticsearch cluster and back:

```
                          Elasticsearch Cluster
                   ┌─────────────────────────────────┐
                   │                                  │
  ┌────────┐       │  ┌─────────────┐                 │
  │        │  HTTP │  │ Coordinating│   ┌───────────┐ │
  │ Client ├──────►│  │    Node     ├──►│ Data Node │ │
  │  (curl │  REQ  │  │             │   │  (Shard)  │ │
  │  or    │       │  │  - Parse    │   │           │ │
  │  app)  │       │  │  - Route    │   │  - Read/  │ │
  │        │◄──────┤  │  - Merge    │◄──┤    Write  │ │
  │        │  HTTP │  │    results  │   │  - Store  │ │
  └────────┘  RES  │  └──────┬──────┘   └───────────┘ │
                   │         │                        │
                   │         │          ┌───────────┐ │
                   │         └─────────►│ Data Node │ │
                   │                    │  (Replica) │ │
                   │                    └───────────┘ │
                   └─────────────────────────────────┘

  1. Client sends HTTP request (GET, PUT, POST, DELETE)
  2. Coordinating node receives and parses the request
  3. Request is routed to the appropriate data node(s)
  4. Data nodes execute the operation on local shards
  5. Results are gathered back at the coordinating node
  6. Coordinating node merges results and sends HTTP response
```

### HTTP Methods Overview

Elasticsearch maps standard HTTP methods to CRUD operations:

GET retrieves data without modifying it. Use it to fetch documents by ID, run search queries, check cluster health, or retrieve index settings. GET requests are idempotent and safe to repeat.

PUT creates or replaces a resource at a known location. Use it to create an index, index a document with a specific ID, or update index settings. PUT replaces the entire document if one already exists at that ID.

POST submits data for processing or triggers actions. Use it to index a document without specifying an ID, perform searches with a request body, run bulk operations, or execute updates and deletes by query.

DELETE removes resources permanently. Use it to remove a document by ID, delete an entire index, or remove a snapshot. DELETE operations are irreversible.

### Document Lifecycle

A document in Elasticsearch goes through several stages from creation to removal. The following diagram shows the typical lifecycle:

```
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  CREATE   │     │  INDEX   │     │  REFRESH  │
  │           ├────►│          ├────►│           │
  │ PUT /idx  │     │ Analyze  │     │ Make      │
  │ /_doc/1   │     │ + Store  │     │ Searchable│
  └──────────┘     └──────────┘     └─────┬─────┘
                                          │
                                          ▼
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  DELETE   │     │  UPDATE  │     │  SEARCH  │
  │           │◄────┤          │◄────┤          │
  │ DELETE    │     │ POST /idx│     │ GET /idx │
  │ /idx/     │     │ /_update │     │ /_search │
  │  _doc/1   │     │          │     │          │
  └──────────┘     └──────────┘     └──────────┘

  CREATE  - Document is submitted via PUT or POST
  INDEX   - Elasticsearch analyzes fields and writes to a shard
  REFRESH - After the refresh interval (default 1s), the document
            becomes visible to search queries
  SEARCH  - The document can now be retrieved by queries
  UPDATE  - Partial or full updates re-index the document internally
  DELETE  - The document is marked for deletion and later purged
```

### API Endpoints

Below is a table of Elasticsearch API endpoints organized by category:

| Category  | Operation                      | HTTP Method | Endpoint                           | Description                                      |
|-----------|--------------------------------|-------------|------------------------------------|--------------------------------------------------|
| Cluster   | Check Cluster Health           | GET         | `/_cat/health?v`                  | Checks cluster health status                     |
| Cluster   | List Nodes in Cluster          | GET         | `/_cat/nodes?v`                   | Lists all cluster nodes                          |
| Cluster   | List All Indices               | GET         | `/_cat/indices?v`                 | Lists all indices and their details              |
| Cluster   | Check Cluster Stats            | GET         | `/_cluster/stats?pretty`          | Retrieves statistics on cluster status and nodes |
| Cluster   | Cluster State                  | GET         | `/_cluster/state?pretty`          | Provides detailed info about the cluster state   |
| Cluster   | Cluster Health                 | GET         | `/_cluster/health?pretty`         | Returns the health status of the cluster         |
| Cluster   | Allocate Replica Shards        | POST        | `/_cluster/reroute`               | Manually allocates replica shards                |
| Cluster   | Pending Tasks                  | GET         | `/_cluster/pending_tasks?pretty`  | Returns pending cluster-level tasks              |
| Index     | Create Index                   | PUT         | `/{index_name}?pretty`            | Creates a new index                              |
| Index     | Delete Index                   | DELETE      | `/{index_name}?pretty`            | Deletes a specified index                        |
| Index     | Refresh Index                  | POST        | `/{index_name}/_refresh?pretty`   | Refreshes an index to make changes searchable    |
| Index     | Get Index Mapping              | GET         | `/{index_name}/_mapping?pretty`   | Retrieves the mapping for a specific index       |
| Index     | Update Index Mapping           | PUT         | `/{index_name}/_mapping?pretty`   | Updates the mapping for an index                 |
| Index     | Get Index Settings             | GET         | `/{index_name}/_settings?pretty`  | Retrieves settings for an index                  |
| Index     | Update Index Settings          | PUT         | `/{index_name}/_settings?pretty`  | Updates settings for an index                    |
| Index     | Open Index                     | POST        | `/{index_name}/_open?pretty`      | Opens a previously closed index                  |
| Index     | Close Index                    | POST        | `/{index_name}/_close?pretty`     | Closes an index to save resources                |
| Index     | Analyze Text                   | GET         | `/{index_name}/_analyze`          | Analyzes text with the specified analyzer        |
| Document  | Index Document with ID         | PUT         | `/{index_name}/_doc/{id}?pretty`  | Adds a document with a specified ID              |
| Document  | Index Document without ID      | POST        | `/{index_name}/_doc?pretty`       | Adds a document with auto-generated ID           |
| Document  | Retrieve Document by ID        | GET         | `/{index_name}/_doc/{id}?pretty`  | Retrieves a document by ID                       |
| Document  | Delete Document by ID          | DELETE      | `/{index_name}/_doc/{id}?pretty`  | Deletes a document by ID                         |
| Document  | Update Document by ID          | POST        | `/{index_name}/_update/{id}?pretty` | Updates a document by ID with partial fields   |
| Document  | Delete By Query                | POST        | `/{index_name}/_delete_by_query?pretty` | Deletes documents matching a query          |
| Document  | Update By Query                | POST        | `/{index_name}/_update_by_query?pretty` | Updates documents matching a query          |
| Document  | Term Vectors                   | GET         | `/{index_name}/_termvectors/{id}?pretty` | Retrieves term vectors for a document      |
| Search    | Search Documents               | GET         | `/{index_name}/_search?pretty`    | Searches for documents based on query parameters |
| Search    | Multi-Search                   | POST        | `/_msearch?pretty`                | Executes multiple search queries in one request  |
| Search    | Validate Query                 | GET         | `/{index_name}/_validate/query?pretty` | Validates a query without executing it      |
| Search    | Explain Query                  | GET         | `/{index_name}/_doc/{id}/_explain?pretty` | Explains how a query matches a document   |
| Search    | Count Documents                | GET         | `/{index_name}/_count?pretty`     | Returns the number of documents matching a query |
| Bulk      | Bulk Operations                | POST        | `/_bulk?pretty`                   | Performs bulk index, update, delete operations    |
| Bulk      | Reindex                        | POST        | `/_reindex?pretty`                | Copies documents from one index to another       |
| Alias     | Get Index Alias                | GET         | `/_alias/{alias_name}?pretty`     | Retrieves information about an alias             |
| Alias     | Set Index Alias                | POST        | `/_aliases?pretty`                | Creates or updates aliases for an index          |
| Alias     | Delete Index Alias             | DELETE      | `/{index_name}/_alias/{alias_name}?pretty` | Deletes an alias for an index             |
| Snapshot  | Create Snapshot Repository     | PUT         | `/_snapshot/{repository}?pretty`  | Registers a snapshot repository                  |
| Snapshot  | Create Snapshot                | PUT         | `/_snapshot/{repository}/{snapshot}?wait_for_completion=true&pretty` | Creates a snapshot backup |
| Snapshot  | Restore Snapshot               | POST        | `/_snapshot/{repository}/{snapshot}/_restore?pretty` | Restores data from a snapshot       |
| Snapshot  | Delete Snapshot                | DELETE      | `/_snapshot/{repository}/{snapshot}?pretty` | Deletes a snapshot from a repository      |
| Snapshot  | List Snapshots                 | GET         | `/_snapshot/{repository}/_all?pretty` | Lists all snapshots in a repository          |

### Scenario 1: Checking Cluster Health

A system administrator wants to check the overall health of an Elasticsearch cluster to confirm that all nodes and shards are functioning correctly. This health check is crucial for preventing issues that could disrupt access to data or affect performance.

To perform this check, the administrator uses the following command:

```http
GET /_cat/health?v
```

**Expected Output**:

```plaintext
epoch      timestamp cluster      status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1666192345 15:25:45  my-cluster   green         3         3     12    6    0    0         0            0               0           100.0%
```

The `status` field indicates the cluster's health. A `green` status means all primary and replica shards are active and distributed across the nodes, showing a fully operational state. A `yellow` status would indicate that some replica shards are not allocated, although the primary data remains available, while a `red` status signals missing primary shards, which could lead to data unavailability.

### Scenario 2: Creating and Indexing a Document

A developer intends to set up a new "customer" index to manage customer records. The first step is to create the index, followed by adding a document representing a customer's details.

**Step I: Create Index**

To create the "customer" index, the developer runs:

```http
PUT /customer?pretty
```

**Expected Output**:

```json
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "customer"
}
```

The response confirms successful index creation with an acknowledgment, indicating that the index is now ready for document insertion.

**Step II: Index Document with ID**

The developer then adds a document with specific fields such as name, email, and age by executing:

```http
PUT /customer/_doc/1?pretty
{
  "name": "John Doe",
  "email": "johndoe@example.com",
  "age": 30
}
```

**Expected Output**:

```json
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "1",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  }
}
```

The `result` field shows "created," confirming that the document has been successfully added to the "customer" index.

### Scenario 3: Retrieving a Document by ID

The support team needs to retrieve information about a specific customer using their unique ID. This operation is helpful when quick access to a customer's details is required.

To retrieve the document, the team runs:

```http
GET /customer/_doc/1?pretty
```

**Expected Output**:

```json
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "1",
  "_version": 1,
  "found": true,
  "_source": {
    "name": "John Doe",
    "email": "johndoe@example.com",
    "age": 30
  }
}
```

The response includes the `found` field set to `true`, confirming the document's presence, and the `_source` field displays the stored information.

### Scenario 4: Updating a Document

The support team needs to update a customer's email address, ensuring their contact information remains current in the database.

To update the document, the team uses the following command:

```http
POST /customer/_doc/1/_update?pretty
{
  "doc": {
    "email": "newemail@example.com"
  }
}
```

**Expected Output**:

```json
{
  "_index": "customer",
  "_type": "_doc",
  "_id": "1",
  "_version": 2,
  "result": "updated",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  }
}
```

The response shows a `result` of "updated," verifying that the email address change was successful.

### Scenario 5: Searching Documents by a Query

A data analyst wants to find all customers over the age of 25. Querying by age allows the analyst to filter records that meet specific conditions.

The query is executed as follows:

```http
GET /customer/_search?pretty
{
  "query": {
    "range": {
      "age": {
        "gt": 25
      }
    }
  }
}
```

**Expected Output**:

```json
{
  "took": 5,
  "timed_out": false,
  "_shards": {
    "total": 2,
    "successful": 2,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"
    },
    "max_score": 1.0,
    "hits": [
      {
        "_index": "customer",
        "_type": "_doc",
        "_id": "1",
        "_score": 1.0,
        "_source": {
          "name": "John Doe",
          "email": "newemail@example.com",
          "age": 30
        }
      }
    ]
  }
}
```

In the response, the `hits` array lists all matching documents, with each document's details under `_source`.

### Scenario 6: Deleting an Index

The administrator decides to delete the `customer` index, intending to permanently remove all associated customer data. This operation is irreversible and clears the data from Elasticsearch.

To delete the index, the administrator executes:

```http
DELETE /customer?pretty
```

**Expected Output**:

```json
{
  "acknowledged": true
}
```

An `acknowledged` value of `true` confirms that the `customer` index has been successfully deleted.

### Scenario 7: Bulk Indexing Multiple Documents

A developer needs to add multiple customer records simultaneously. Bulk indexing is useful for efficiently inserting large volumes of data in a single request.

The developer runs:

```http
POST /_bulk?pretty
{ "index": { "_index": "customer", "_id": "2" } }
{ "name": "Jane Smith", "email": "jane.smith@example.com", "age": 28 }
{ "index": { "_index": "customer", "_id": "3" } }
{ "name": "Alice Johnson", "email": "alice.j@example.com", "age": 35 }
```

**Expected Output**:

```json
{
  "took": 30,
  "errors": false,
  "items": [
    { "index": { "_index": "customer", "_id": "2", "result": "created" } },
    { "index": { "_index": "customer", "_id": "3", "result": "created" } }
  ]
}
```

Each entry in the `items` array provides confirmation that the corresponding document was successfully created, with the `result` field indicating "created."

### Scenario 8: Reindexing Data

An operations engineer needs to migrate data from an old index to a new one. This is common when changing mappings, renaming an index, or consolidating data. The reindex API copies documents from a source index to a destination index without requiring the client to pull and re-insert every document.

First the engineer creates the destination index, then runs the reindex operation:

```http
PUT /customer_v2?pretty
```

```http
POST /_reindex?pretty
{
  "source": {
    "index": "customer"
  },
  "dest": {
    "index": "customer_v2"
  }
}
```

**Expected Output**:

```json
{
  "took": 52,
  "timed_out": false,
  "total": 3,
  "updated": 0,
  "created": 3,
  "deleted": 0,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "failures": []
}
```

The `total` field shows how many documents were processed, and `created` confirms how many were written to the new index. An empty `failures` array means the operation completed without errors.

### Scenario 9: Using Aliases

A developer wants to use an alias so that application code always references a stable name like `current_customers`, even when the underlying index changes during reindexing or migrations. Aliases act as pointers that can be redirected without modifying application configuration.

To create an alias pointing to an existing index:

```http
POST /_aliases?pretty
{
  "actions": [
    { "add": { "index": "customer_v2", "alias": "current_customers" } }
  ]
}
```

**Expected Output**:

```json
{
  "acknowledged": true
}
```

The alias is now active. Queries against `current_customers` will be routed to `customer_v2`. To verify the alias:

```http
GET /_alias/current_customers?pretty
```

**Expected Output**:

```json
{
  "customer_v2": {
    "aliases": {
      "current_customers": {}
    }
  }
}
```

When migrating to a new index later, both actions (remove old, add new) can be performed atomically in a single request, ensuring zero downtime:

```http
POST /_aliases?pretty
{
  "actions": [
    { "remove": { "index": "customer_v2", "alias": "current_customers" } },
    { "add": { "index": "customer_v3", "alias": "current_customers" } }
  ]
}
```

### Scenario 10: Snapshot and Restore

A database administrator wants to back up the `customer` index before a risky schema migration. Snapshots provide point-in-time backups that can be restored if something goes wrong.

First, register a snapshot repository:

```http
PUT /_snapshot/my_backup?pretty
{
  "type": "fs",
  "settings": { "location": "/mnt/backups/elasticsearch" }
}
```

Next, create a snapshot:

```http
PUT /_snapshot/my_backup/snapshot_1?wait_for_completion=true&pretty
{
  "indices": "customer",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

**Expected Output**:

```json
{
  "snapshot": {
    "snapshot": "snapshot_1",
    "version": "7.17.0",
    "indices": ["customer"],
    "state": "SUCCESS",
    "shards": { "total": 2, "failed": 0, "successful": 2 }
  }
}
```

A `state` of "SUCCESS" confirms the backup completed. To restore the snapshot later (the index must be closed or deleted first):

```http
POST /_snapshot/my_backup/snapshot_1/_restore?pretty
{
  "indices": "customer",
  "ignore_unavailable": true
}
```

### Scenario 11: Multi-Search

A dashboard application needs to fetch results from multiple queries in a single network round trip. The multi-search API allows sending several independent search requests in one HTTP call, reducing latency compared to issuing them individually.

```http
POST /_msearch?pretty
{"index": "customer"}
{"query": {"match": {"name": "John"}}}
{"index": "customer"}
{"query": {"range": {"age": {"gte": 30}}}}
```

**Expected Output**:

```json
{
  "took": 8,
  "responses": [
    {
      "took": 3,
      "timed_out": false,
      "hits": {
        "total": { "value": 1, "relation": "eq" },
        "hits": [
          { "_index": "customer", "_id": "1", "_source": { "name": "John Doe", "age": 30 } }
        ]
      },
      "status": 200
    },
    {
      "took": 4,
      "timed_out": false,
      "hits": {
        "total": { "value": 2, "relation": "eq" },
        "hits": [
          { "_index": "customer", "_id": "1", "_source": { "name": "John Doe", "age": 30 } },
          { "_index": "customer", "_id": "3", "_source": { "name": "Alice Johnson", "age": 35 } }
        ]
      },
      "status": 200
    }
  ]
}
```

Each entry in the `responses` array corresponds to one of the search requests, in order. The `status` field indicates whether that individual query succeeded.

### Scenario 12: Delete by Query

An administrator needs to remove all customer records where the age is below 18 to comply with a data retention policy. Rather than deleting documents one by one, the delete by query API removes all documents matching a search query in a single operation.

```http
POST /customer/_delete_by_query?pretty
{
  "query": {
    "range": {
      "age": {
        "lt": 18
      }
    }
  }
}
```

**Expected Output**:

```json
{
  "took": 35,
  "timed_out": false,
  "total": 2,
  "deleted": 2,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "failures": []
}
```

The `total` field indicates how many documents matched and `deleted` confirms how many were removed. The empty `failures` array means the operation completed cleanly.

### Common HTTP Response Codes

Elasticsearch uses standard HTTP status codes to indicate the result of an API request. Understanding these codes helps with debugging and error handling.

| Code | Status                 | Meaning                                                                 |
|------|------------------------|-------------------------------------------------------------------------|
| 200  | OK                     | The request succeeded. Used for successful GET, POST, and PUT requests. |
| 201  | Created                | A new resource was created. Returned when a document is indexed.        |
| 400  | Bad Request            | The request body or parameters are malformed or invalid.                |
| 401  | Unauthorized           | Authentication credentials are missing or invalid.                      |
| 403  | Forbidden              | The authenticated user lacks permission for the requested operation.    |
| 404  | Not Found              | The requested resource does not exist (index, document, or endpoint).   |
| 405  | Method Not Allowed     | The HTTP method is not supported for the given endpoint.                |
| 408  | Request Timeout        | The request took longer than the server was willing to wait.            |
| 409  | Conflict               | A version conflict occurred during an update or index operation.        |
| 429  | Too Many Requests      | The cluster is rejecting requests due to high load or queue pressure.   |
| 500  | Internal Server Error  | An unexpected error occurred on the server side.                        |
| 503  | Service Unavailable    | The cluster is not ready to handle requests, often during startup.      |

When a request fails, Elasticsearch returns a JSON body with error details:

```json
{
  "error": {
    "root_cause": [
      { "type": "index_not_found_exception", "reason": "no such index [missing_index]" }
    ],
    "type": "index_not_found_exception",
    "reason": "no such index [missing_index]"
  },
  "status": 404
}
```

The `error.type` field identifies the exception class, and `error.reason` provides a human-readable explanation.

### Query String Parameters

Elasticsearch supports several query string parameters that control response formatting and behavior. These can be appended to any endpoint.

| Parameter              | Purpose                                                    | Example                                        |
|------------------------|------------------------------------------------------------|-------------------------------------------------|
| `?pretty`              | Formats JSON with indentation for readability              | `GET /customer/_search?pretty`                 |
| `?v`                   | Adds column headers to `_cat` API responses                | `GET /_cat/indices?v`                          |
| `?format=json`         | Returns `_cat` output as JSON instead of plain text        | `GET /_cat/health?format=json`                 |
| `?h=`                  | Selects specific columns in `_cat` output                  | `GET /_cat/indices?v&h=index,docs.count`       |
| `?filter_path=`        | Limits JSON response to specified fields only              | `GET /customer/_search?filter_path=hits.hits`  |
| `?timeout=`            | Sets maximum wait time before returning partial results    | `GET /customer/_search?timeout=5s`             |
| `?wait_for_completion` | Blocks until a long-running operation finishes             | `PUT /_snapshot/repo/snap?wait_for_completion=true` |
| `?refresh=true`        | Forces an immediate refresh after a write operation        | `PUT /customer/_doc/1?refresh=true`            |

The `?pretty` parameter is useful during development but should be omitted in production to reduce response size. The `?filter_path` parameter is particularly valuable in automated scripts, where reducing payload size improves performance. For long-running operations like reindex or snapshot, `?wait_for_completion=false` returns a task ID immediately, which can then be polled with `GET /_tasks/{task_id}` for progress updates.
