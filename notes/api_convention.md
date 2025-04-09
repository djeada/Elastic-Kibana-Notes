## API Convention

The Elasticsearch API provides a RESTful interface for interacting with and managing Elasticsearch clusters, indices, and documents, making it easy to build, monitor, and analyze data workflows. Through simple HTTP methods, users can perform essential operations like creating, retrieving, updating, and deleting documents, as well as manage indices and check cluster health. Cluster-level endpoints allow for monitoring overall health (`/_cat/health`), listing nodes, and obtaining detailed stats (`/_cluster/stats`), while index-level endpoints support creating and deleting indices, managing settings, and handling aliases. Document-related endpoints provide options for indexing data, searching, updating, and deleting, while more advanced features like bulk operations, reindexing, and snapshots enable efficient data handling at scale. The API's flexibility and range of operations allow developers to efficiently manage and query Elasticsearch data, making it adaptable for various use cases from simple document storage to complex data analysis and search functionalities.

Below is a table with Elasticsearch API endpoints:

| Operation                      | HTTP Method | Endpoint                           | Description                                      |
|--------------------------------|-------------|------------------------------------|--------------------------------------------------|
| Check Cluster Health           | GET         | `/_cat/health?v`                  | Checks cluster health status                     |
| List Nodes in Cluster          | GET         | `/_cat/nodes?v`                   | Lists all cluster nodes                          |
| List All Indices               | GET         | `/_cat/indices?v`                 | Lists all indices and their details              |
| Create Index                   | PUT         | `/{index_name}?pretty`            | Creates a new index                              |
| Delete Index                   | DELETE      | `/{index_name}?pretty`            | Deletes a specified index                        |
| Index Document with ID         | PUT         | `/{index_name}/{document_id}?pretty` | Adds a document with a specified ID              |
| Index Document without ID      | POST        | `/{index_name}?pretty`            | Adds a document, letting Elasticsearch assign ID |
| Retrieve Document by ID        | GET         | `/{index_name}/{document_id}?pretty` | Retrieves a document by ID                       |
| Delete Document by ID          | DELETE      | `/{index_name}/{document_id}?pretty` | Deletes a document by ID                         |
| Update Document by ID          | POST        | `/{index_name}/{document_id}/_update?pretty` | Updates a document by ID, providing updated fields |
| Search Documents               | GET         | `/{index_name}/_search?pretty`    | Searches for documents in an index based on query parameters |
| Multi-Search                   | POST        | `/_msearch?pretty`                | Executes multiple search queries within the same request |
| Bulk Operations                | POST        | `/_bulk?pretty`                   | Performs bulk operations (indexing, updating, deleting multiple documents) |
| Refresh Index                  | POST        | `/{index_name}/_refresh?pretty`   | Refreshes an index to make recent changes searchable |
| Get Index Mapping              | GET         | `/{index_name}/_mapping?pretty`   | Retrieves the mapping for a specific index       |
| Update Index Mapping           | PUT         | `/{index_name}/_mapping?pretty`   | Updates the mapping for an index                 |
| Get Index Settings             | GET         | `/{index_name}/_settings?pretty`  | Retrieves settings for an index                  |
| Update Index Settings          | PUT         | `/{index_name}/_settings?pretty`  | Updates settings for an index                    |
| Reindex                        | POST        | `/_reindex?pretty`                | Reindexes data from one index to another         |
| Check Cluster Stats            | GET         | `/_cluster/stats?pretty`          | Retrieves comprehensive statistics on cluster status and nodes |
| Cluster State                  | GET         | `/_cluster/state?pretty`          | Provides detailed information about the cluster’s current state |
| Cluster Health                 | GET         | `/_cluster/health?pretty`         | Returns the health status of the cluster         |
| Allocate Replica Shards        | POST        | `/_cluster/reroute`               | Manually allocates replica shards                |
| Create Snapshot                | PUT         | `/_snapshot/{repository}/{snapshot}?wait_for_completion=true&pretty` | Creates a snapshot of an index or cluster backup |
| Restore Snapshot               | POST        | `/_snapshot/{repository}/{snapshot}/_restore?pretty` | Restores a snapshot                              |
| Delete Snapshot                | DELETE      | `/_snapshot/{repository}/{snapshot}?pretty` | Deletes a snapshot from a repository             |
| Analyze Text                   | GET         | `/{index_name}/_analyze`          | Analyzes text according to the specified analyzer |
| Get Index Alias                | GET         | `/_alias/{alias_name}?pretty`     | Retrieves information about an alias             |
| Set Index Alias                | POST        | `/_aliases?pretty`                | Creates or updates aliases for an index          |
| Delete Index Alias             | DELETE      | `/{index_name}/_alias/{alias_name}?pretty` | Deletes an alias for an index                    |
| Validate Query                 | GET         | `/{index_name}/_validate/query?pretty` | Validates a query without executing it           |
| Explain Query                  | GET         | `/{index_name}/{document_id}/_explain?pretty` | Provides an explanation of how a query matches a document |
| Term Vectors                   | GET         | `/{index_name}/{document_id}/_termvectors?pretty` | Retrieves term vectors for a document            |
| Delete By Query                | POST        | `/{index_name}/_delete_by_query?pretty` | Deletes documents based on a search query        |
| Update By Query                | POST        | `/{index_name}/_update_by_query?pretty` | Updates documents based on a query               |

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
