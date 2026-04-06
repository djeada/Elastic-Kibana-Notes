## Task 9: Modeling Relational Data with Nested Objects and Parent-Child Relationships

### Objectives

- Understand why Elasticsearch flattens object arrays and the problems this causes.
- Use the `nested` type to preserve the independence of objects within an array.
- Use `inner_hits` to retrieve the specific nested objects that matched a query.
- Model parent-child relationships with the `join` field type.
- Write `has_parent` and `has_child` queries to navigate between related documents.
- Compare the tradeoffs of object, nested, and parent-child approaches.
- Query nested data programmatically with Python.

---

### How Elasticsearch Stores Relational Data

Elasticsearch is **not** a relational database — it has no JOIN across indices.
Instead it offers three strategies for relationships inside a single index:

```
┌────────────────────────────────────────────────────────────────┐
│  FLAT OBJECT (default)                                         │
│  One Lucene doc. Arrays flattened into multi-value fields:     │
│    reviews.user → ["alice","bob"]  reviews.stars → [5, 2]     │
│  ⚠ Cross-object matches possible (alice + 2 stars matches!)   │
├────────────────────────────────────────────────────────────────┤
│  NESTED OBJECT                                                 │
│  Each nested object → hidden separate Lucene doc:              │
│    doc 0 (nested): user=alice, stars=5                         │
│    doc 1 (nested): user=bob,   stars=2                         │
│    doc 2 (root):   title="Wireless Mouse"                      │
│  ✔ Cross-object matching prevented                             │
├────────────────────────────────────────────────────────────────┤
│  PARENT-CHILD (join field)                                     │
│  Fully independent ES documents linked by a join field:        │
│    Doc 1 (parent/author): name="George Orwell"                 │
│    Doc 2 (child/book):    title="1984", parent="1"             │
│  ✔ Children updated without reindexing the parent              │
│  ⚠ Must live on the same shard (routing required)              │
└────────────────────────────────────────────────────────────────┘
```

---

### Why Relational Data Modeling Matters

Choosing the wrong strategy leads to **incorrect results** (flat objects match
across array boundaries), **expensive reindexing** (nested requires reindexing
the parent on child update), or **wasted memory** (parent-child uses an
in-memory global ordinals map). Understanding each approach helps you pick the
right one.

---

### Lab Steps

#### Step 1 — Object Type (Default): The Flattening Problem

Create an index using the default `object` mapping and index a document:

```json
PUT /products
{ "mappings": { "properties": {
    "name": { "type": "text" },
    "reviews": { "properties": {
      "user":  { "type": "keyword" },
      "stars": { "type": "integer" }
    }}
}}}

POST /products/_doc/1
{ "name": "Wireless Mouse",
  "reviews": [
    { "user": "alice", "stars": 5 },
    { "user": "bob",   "stars": 2 }
  ]}
```

Query for alice with 2 stars — **should not match** but does:

```json
GET /products/_search
{ "query": { "bool": { "must": [
    { "term": { "reviews.user": "alice" } },
    { "term": { "reviews.stars": 2 } }
]}}}
```

**Expected output** — incorrect cross-object match:

```json
{ "hits": { "total": { "value": 1 }, "hits": [
    { "_id": "1", "_source": { "name": "Wireless Mouse", "reviews": [
        { "user": "alice", "stars": 5 }, { "user": "bob", "stars": 2 }
    ]}}
]}}
```

> Alice gave 5 stars, not 2. Elasticsearch stored `reviews.user: ["alice","bob"]`
> and `reviews.stars: [5,2]` as independent arrays, so the query matched across
> object boundaries.

---

#### Step 2 — Nested Type: Solving the Flattening Problem

```json
PUT /products_nested
{ "mappings": { "properties": {
    "name": { "type": "text" },
    "reviews": {
      "type": "nested",
      "properties": {
        "user":  { "type": "keyword" },
        "stars": { "type": "integer" },
        "comment": { "type": "text" }
      }
    }
}}}

POST /products_nested/_doc/1
{ "name": "Wireless Mouse",
  "reviews": [
    { "user": "alice", "stars": 5, "comment": "Excellent build quality" },
    { "user": "bob",   "stars": 2, "comment": "Stopped working after a week" }
  ]}
```

Same cross-object query — now correctly returns **zero** hits:

```json
GET /products_nested/_search
{ "query": { "nested": { "path": "reviews", "query": { "bool": { "must": [
    { "term": { "reviews.user": "alice" } },
    { "term": { "reviews.stars": 2 } }
]}}}}}
```

**Expected output:**

```json
{ "hits": { "total": { "value": 0 }, "hits": [] } }
```

Correct query — alice with 5 stars:

```json
GET /products_nested/_search
{ "query": { "nested": { "path": "reviews", "query": { "bool": { "must": [
    { "term": { "reviews.user": "alice" } },
    { "term": { "reviews.stars": 5 } }
]}}}}}
```

**Expected output:**

```json
{ "hits": { "total": { "value": 1 }, "hits": [
    { "_id": "1", "_source": { "name": "Wireless Mouse", "reviews": ["..."] } }
]}}
```

---

#### Step 3 — inner_hits: Retrieving Matching Nested Objects

By default a nested query returns the full parent. Add `inner_hits` to see
**which** nested objects matched:

```json
GET /products_nested/_search
{ "query": { "nested": {
    "path": "reviews",
    "query": { "range": { "reviews.stars": { "gte": 4 } } },
    "inner_hits": {}
}}}
```

**Expected output (trimmed):**

```json
{ "hits": { "hits": [
    { "_id": "1",
      "inner_hits": { "reviews": { "hits": { "hits": [
        { "_nested": { "field": "reviews", "offset": 0 },
          "_source": { "user": "alice", "stars": 5,
                       "comment": "Excellent build quality" } }
      ]}}}
    }
]}}
```

> Only alice's review (offset 0) appears — bob's 2-star review does not
> satisfy `gte: 4`.

---

#### Step 4 — Parent-Child Relationships with the Join Field

Use parent-child when children are updated frequently and you want to avoid
reindexing the parent.

```json
PUT /library
{ "mappings": { "properties": {
    "name":  { "type": "text" },
    "title": { "type": "text" },
    "genre": { "type": "keyword" },
    "my_join_field": {
      "type": "join",
      "relations": { "author": "book" }
    }
}}}
```

Index parents (authors) and children (books) — note `routing`:

```json
POST /library/_doc/1
{ "name": "George Orwell", "my_join_field": "author" }

POST /library/_doc/2
{ "name": "Aldous Huxley", "my_join_field": "author" }

POST /library/_doc/101?routing=1
{ "title": "1984", "genre": "dystopian",
  "my_join_field": { "name": "book", "parent": "1" } }

POST /library/_doc/102?routing=1
{ "title": "Animal Farm", "genre": "satire",
  "my_join_field": { "name": "book", "parent": "1" } }

POST /library/_doc/201?routing=2
{ "title": "Brave New World", "genre": "dystopian",
  "my_join_field": { "name": "book", "parent": "2" } }
```

---

#### Step 5 — has_child Query (Find Parents by Their Children)

Find authors who wrote a dystopian book:

```json
GET /library/_search
{ "query": { "has_child": {
    "type": "book",
    "query": { "term": { "genre": "dystopian" } }
}}}
```

**Expected output:**

```json
{ "hits": { "total": { "value": 2 }, "hits": [
    { "_id": "1", "_source": { "name": "George Orwell" } },
    { "_id": "2", "_source": { "name": "Aldous Huxley" } }
]}}
```

---

#### Step 6 — has_parent Query (Find Children by Their Parent)

Find all books by George Orwell:

```json
GET /library/_search
{ "query": { "has_parent": {
    "parent_type": "author",
    "query": { "match": { "name": "George Orwell" } }
}}}
```

**Expected output:**

```json
{ "hits": { "total": { "value": 2 }, "hits": [
    { "_id": "101", "_source": { "title": "1984", "genre": "dystopian" } },
    { "_id": "102", "_source": { "title": "Animal Farm", "genre": "satire" } }
]}}
```

---

### Comparison Table: Object vs Nested vs Parent-Child

| Criteria           | Object (default)       | Nested                       | Parent-Child (Join)          |
|--------------------|------------------------|------------------------------|------------------------------|
| **Storage**        | Single Lucene doc      | Hidden Lucene docs per item  | Separate ES documents        |
| **Query accuracy** | Cross-object matches ⚠ | Accurate ✔                   | Accurate ✔                   |
| **Query speed**    | Fastest                | Fast (block-nested join)     | Slower (global ordinals)     |
| **Update cost**    | Reindex entire doc     | Reindex entire doc           | Update child independently ✔ |
| **Memory**         | Low                    | Low                          | Higher (ordinals in heap)    |
| **Best for**       | Rarely queried objects | Small, rarely updated arrays | Frequently updated children  |

---

### Python Script: Querying Nested Data

```python
"""Query nested reviews in the products_nested index."""
from elasticsearch import Elasticsearch

es = Elasticsearch("http://localhost:9200")

resp = es.search(index="products_nested", body={
    "query": { "nested": {
        "path": "reviews",
        "query": { "bool": { "must": [
            {"term": {"reviews.user": "alice"}},
            {"range": {"reviews.stars": {"gte": 4}}}
        ]}},
        "inner_hits": {"size": 5}
    }}
})

print(f"Total hits: {resp['hits']['total']['value']}")
for hit in resp["hits"]["hits"]:
    print(f"\nProduct: {hit['_source']['name']}")
    for inner in hit["inner_hits"]["reviews"]["hits"]["hits"]:
        src = inner["_source"]
        print(f"  ★ {src['stars']} by {src['user']} — \"{src['comment']}\"")
```

**Expected output:**

```
Total hits: 1

Product: Wireless Mouse
  ★ 5 by alice — "Excellent build quality"
```

---

### Troubleshooting Tips

| Symptom | Cause | Fix |
|---------|-------|-----|
| Cross-object matches | Field mapped as `object` not `nested` | Change mapping to `"type": "nested"` and reindex |
| `query_shard_exception` on `has_child` | Child not routed to parent's shard | Add `?routing=<parent_id>` when indexing |
| `inner_hits` empty | `inner_hits` placed outside `nested` block | Move `"inner_hits": {}` inside the nested query |
| Slow parent-child queries | Global ordinals rebuilt each refresh | Set `"eager_global_ordinals": true` on join field |
| `mapper_parsing_exception` | Wrong join field format in child doc | Use `{ "name": "<child>", "parent": "<id>" }` |

---

### Cleanup

```json
DELETE /products
DELETE /products_nested
DELETE /library
```

**Expected output (each):** `{ "acknowledged": true }`

---

### Reflection

1. Why does the default `object` type produce incorrect matches on arrays?
2. What is the performance tradeoff of `nested` vs `parent-child`?
3. In what scenario would you prefer `parent-child` over `nested`?
4. How does `inner_hits` help when debugging nested queries?
