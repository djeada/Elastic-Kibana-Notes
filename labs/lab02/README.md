## Task 2: Index Creation, CRUD Operations, and Data Validation

**Objectives:**
- Create an index with custom settings and mappings.
- Perform document create, read, update, and delete (CRUD) operations.
- Learn bulk operations for efficient multi-document indexing.
- Understand `_source` filtering and optimistic concurrency control.
- Validate operations using both Kibana's Dev Tools, curl, and Python.

---

### CRUD Document Lifecycle

```
  +--------------------------------------------------------------+
  |                   DOCUMENT LIFECYCLE                         |
  |                                                              |
  |   CREATE            READ              UPDATE         DELETE  |
  |   (POST/PUT)        (GET)             (POST _update) (DELETE)|
  |                                                              |
  |   +------+       +------+          +------+      +------+   |
  |   | JSON |------>|Index |--------->|Merge |----->|Remove|   |
  |   | Body | index |Stored| retrieve |Fields| mark | From |   |
  |   +------+       +--+---+          +--+---+ del  +------+   |
  |                     |                  |                      |
  |              _id assigned        _version bumped              |
  |              _version = 1        _seq_no incremented          |
  |                                                              |
  |   +------------------------------------------------------+   |
  |   |          Inverted Index (Lucene Segments)            |   |
  |   |  term -> [doc1, doc3, ...]   (for full-text search)  |   |
  |   +------------------------------------------------------+   |
  +--------------------------------------------------------------+
```

---

### Lab Steps

### Step 1: Create an Index with Custom Settings

**In Kibana Dev Tools:**

```json
PUT /products
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "name":        { "type": "text" },
      "description": { "type": "text" },
      "price":       { "type": "float" },
      "category":    { "type": "keyword" },
      "in_stock":    { "type": "boolean" },
      "created_at":  { "type": "date" }
    }
  }
}
```

**Equivalent curl command:**
```bash
curl -s -X PUT "http://localhost:9200/products" \
  -H "Content-Type: application/json" \
  -d '{
    "settings": { "number_of_shards": 1, "number_of_replicas": 0 },
    "mappings": {
      "properties": {
        "name":        { "type": "text" },
        "description": { "type": "text" },
        "price":       { "type": "float" },
        "category":    { "type": "keyword" },
        "in_stock":    { "type": "boolean" },
        "created_at":  { "type": "date" }
      }
    }
  }' | python3 -m json.tool
```

**Expected Output:**
```json
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "products"
}
```

**Verify the index was created:**

```http
GET /products
```

**Expected Output (excerpt):**
```json
{
  "products": {
    "aliases": {},
    "mappings": {
      "properties": {
        "name":        { "type": "text" },
        "description": { "type": "text" },
        "price":       { "type": "float" },
        "category":    { "type": "keyword" },
        "in_stock":    { "type": "boolean" },
        "created_at":  { "type": "date" }
      }
    },
    "settings": {
      "index": {
        "number_of_shards": "1",
        "number_of_replicas": "0"
      }
    }
  }
}
```

---

### Step 2: Create (Index) Documents

**Create a document with an auto-generated ID:**

```json
POST /products/_doc
{
  "name": "Elastic T-Shirt",
  "description": "Comfortable cotton t-shirt with Elastic logo",
  "price": 19.99,
  "category": "clothing",
  "in_stock": true,
  "created_at": "2024-01-15"
}
```

**Expected Output:**
```json
{
  "_index": "products",
  "_id": "abc123XYZ",
  "_version": 1,
  "result": "created",
  "_shards": { "total": 1, "successful": 1, "failed": 0 },
  "_seq_no": 0,
  "_primary_term": 1
}
```

**Create a document with a specific ID:**

```json
PUT /products/_doc/1
{
  "name": "Kibana Mug",
  "description": "Ceramic mug featuring Kibana dashboard artwork",
  "price": 12.50,
  "category": "accessories",
  "in_stock": true,
  "created_at": "2024-02-01"
}
```

**Expected Output:**
```json
{
  "_index": "products",
  "_id": "1",
  "_version": 1,
  "result": "created",
  "_shards": { "total": 1, "successful": 1, "failed": 0 },
  "_seq_no": 1,
  "_primary_term": 1
}
```

---

### Step 3: Read Documents

**Retrieve a document by ID:**

```http
GET /products/_doc/1
```

**Expected Output:**
```json
{
  "_index": "products",
  "_id": "1",
  "_version": 1,
  "_seq_no": 1,
  "_primary_term": 1,
  "found": true,
  "_source": {
    "name": "Kibana Mug",
    "description": "Ceramic mug featuring Kibana dashboard artwork",
    "price": 12.50,
    "category": "accessories",
    "in_stock": true,
    "created_at": "2024-02-01"
  }
}
```

### _source Filtering

Retrieve only specific fields to reduce response size:

**Include only selected fields:**

```http
GET /products/_doc/1?_source_includes=name,price
```

**Expected Output:**
```json
{
  "_index": "products",
  "_id": "1",
  "found": true,
  "_source": {
    "name": "Kibana Mug",
    "price": 12.50
  }
}
```

**Exclude specific fields:**

```http
GET /products/_doc/1?_source_excludes=description,created_at
```

**Expected Output:**
```json
{
  "_index": "products",
  "_id": "1",
  "found": true,
  "_source": {
    "name": "Kibana Mug",
    "price": 12.50,
    "category": "accessories",
    "in_stock": true
  }
}
```

---

### Step 4: Update Documents

**Partial update -- change the price:**

```json
POST /products/_update/1
{
  "doc": { "price": 10.99 }
}
```

**Expected Output:**
```json
{
  "_index": "products",
  "_id": "1",
  "_version": 2,
  "result": "updated",
  "_shards": { "total": 1, "successful": 1, "failed": 0 },
  "_seq_no": 2,
  "_primary_term": 1
}
```

**Verify the update:**

```http
GET /products/_doc/1?_source_includes=name,price
```

**Expected Output:**
```json
{
  "_index": "products",
  "_id": "1",
  "found": true,
  "_source": {
    "name": "Kibana Mug",
    "price": 10.99
  }
}
```

**Update with a script (increment price by 2):**

```json
POST /products/_update/1
{
  "script": {
    "source": "ctx._source.price += params.amount",
    "params": { "amount": 2 }
  }
}
```

**Expected Output:**
```json
{
  "_index": "products",
  "_id": "1",
  "_version": 3,
  "result": "updated"
}
```

---

### Step 5: Delete Documents

```http
DELETE /products/_doc/1
```

**Expected Output:**
```json
{
  "_index": "products",
  "_id": "1",
  "_version": 4,
  "result": "deleted",
  "_shards": { "total": 1, "successful": 1, "failed": 0 },
  "_seq_no": 4,
  "_primary_term": 1
}
```

**Verify deletion:**

```http
GET /products/_doc/1
```

**Expected Output:**
```json
{
  "_index": "products",
  "_id": "1",
  "found": false
}
```

---

### Step 6: Bulk Operations with _bulk

The `_bulk` API lets you perform multiple index, update, and delete operations in a single request for much better performance.

```json
POST /products/_bulk
{"index": {"_id": "10"}}
{"name": "Logstash Hoodie", "description": "Warm hoodie with Logstash logo", "price": 45.00, "category": "clothing", "in_stock": true, "created_at": "2024-03-01"}
{"index": {"_id": "11"}}
{"name": "Beats Cap", "description": "Adjustable cap with Beats branding", "price": 15.00, "category": "accessories", "in_stock": true, "created_at": "2024-03-05"}
{"index": {"_id": "12"}}
{"name": "APM Sticker Pack", "description": "Set of 10 APM stickers", "price": 5.99, "category": "accessories", "in_stock": false, "created_at": "2024-03-10"}
```

**Expected Output:**
```json
{
  "took": 30,
  "errors": false,
  "items": [
    { "index": { "_index": "products", "_id": "10", "_version": 1, "result": "created", "status": 201 } },
    { "index": { "_index": "products", "_id": "11", "_version": 1, "result": "created", "status": 201 } },
    { "index": { "_index": "products", "_id": "12", "_version": 1, "result": "created", "status": 201 } }
  ]
}
```

**Verify all documents exist:**

```http
GET /products/_count
```

**Expected Output:**
```json
{
  "count": 3,
  "_shards": { "total": 1, "successful": 1, "failed": 0 }
}
```

---

### Step 7: Optimistic Concurrency Control

Elasticsearch uses `_seq_no` and `_primary_term` to prevent conflicting writes. This is essential in multi-client environments.

**First, read the current document to get its sequence number:**

```http
GET /products/_doc/10
```

Note the `_seq_no` and `_primary_term` values from the response.

**Then, update using those values for concurrency safety:**

```json
POST /products/_update/10?if_seq_no=5&if_primary_term=1
{
  "doc": { "price": 42.00 }
}
```

**Expected Output (success):**
```json
{
  "_index": "products",
  "_id": "10",
  "_version": 2,
  "result": "updated",
  "_seq_no": 6,
  "_primary_term": 1
}
```

**If another client already modified the document (conflict):**
```json
{
  "error": {
    "type": "version_conflict_engine_exception",
    "reason": "[10]: version conflict, required seqNo [5], primary term [1]. current document has seqNo [6] and primary term [1]"
  },
  "status": 409
}
```

> **Tip:** On a 409 conflict, re-read the document to get the latest `_seq_no`, then retry your update.

---

### Step 8: CRUD Operations Using Python

```python
import requests
import json

ES_URL = "http://localhost:9200"
INDEX = "products"
DOC_URL = f"{ES_URL}/{INDEX}/_doc"

headers = {"Content-Type": "application/json"}

# --- CREATE ---
doc = {
    "name": "Elastic T-Shirt",
    "description": "Comfortable cotton t-shirt with Elastic logo",
    "price": 19.99,
    "category": "clothing",
    "in_stock": True,
    "created_at": "2024-01-15"
}
resp = requests.post(DOC_URL, json=doc).json()
doc_id = resp["_id"]
print(f"Created document ID: {doc_id}")
print(f"  result : {resp['result']}")
print(f"  version: {resp['_version']}")
print()

# --- READ ---
resp = requests.get(f"{DOC_URL}/{doc_id}").json()
print(f"Read document {doc_id}:")
print(f"  name : {resp['_source']['name']}")
print(f"  price: {resp['_source']['price']}")
print()

# --- UPDATE ---
update = {"doc": {"price": 17.99}}
resp = requests.post(f"{ES_URL}/{INDEX}/_update/{doc_id}", json=update).json()
print(f"Updated document {doc_id}:")
print(f"  result : {resp['result']}")
print(f"  version: {resp['_version']}")
print()

# --- VERIFY UPDATE ---
resp = requests.get(f"{DOC_URL}/{doc_id}").json()
print(f"Verified price after update: {resp['_source']['price']}")
print()

# --- DELETE ---
resp = requests.delete(f"{DOC_URL}/{doc_id}").json()
print(f"Deleted document {doc_id}:")
print(f"  result: {resp['result']}")
```

**Expected Output:**
```
Created document ID: kX3b8Y0BqZ...
  result : created
  version: 1

Read document kX3b8Y0BqZ...:
  name : Elastic T-Shirt
  price: 19.99

Updated document kX3b8Y0BqZ...:
  result : updated
  version: 2

Verified price after update: 17.99

Deleted document kX3b8Y0BqZ...:
  result: deleted
```

---

### Troubleshooting Tips

| Problem | Cause | Solution |
|---|---|---|
| `index_not_found_exception` | Index does not exist | Create the index first with `PUT /products` |
| `document_missing_exception` on update | Document ID does not exist | Verify the ID with `GET /products/_doc/<id>` |
| `mapper_parsing_exception` | Field value does not match mapping type | Check your mapping with `GET /products/_mapping` and fix the data |
| `version_conflict_engine_exception` | Concurrency conflict on `if_seq_no` | Re-read the document, get the latest `_seq_no`, and retry |
| `_bulk` reports `"errors": true` | One or more operations failed | Inspect individual items in the `items` array for the error details |

---

### Reflection

Write down differences observed between raw API calls (via Kibana) and using Python. Discuss the benefits of automation, especially for bulk operations and repeatable test workflows.
