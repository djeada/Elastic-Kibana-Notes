## Task 9: Modeling Relational Data with Nested Objects and Parent-Child Relationships

### Objectives

By the end of this task, you should be able to:

- Explain why Elasticsearch flattens arrays of objects by default and why that can produce incorrect matches.
- Use the `nested` field type to keep each object in an array independent during search.
- Use `inner_hits` to show the exact nested object or objects that matched a query.
- Model one-to-many relationships with the `join` field type.
- Write `has_parent` and `has_child` queries to move between related parent and child documents.
- Compare when to use regular objects, nested objects, and parent-child relationships.
- Query nested data programmatically with Python.

### How Elasticsearch Stores Relational Data

Elasticsearch is **not** a relational database. It does not support traditional SQL joins across tables or indices.

Instead, Elasticsearch gives you a few modeling options inside a single index. The right choice depends on how your data is queried, how often child records change, and whether matching must preserve object boundaries.

```
┌────────────────────────────────────────────────────────────────┐
│  FLAT OBJECT (default)                                         │
│  One Lucene document. Arrays are flattened into multi-value    │
│  fields:                                                       │
│    reviews.user  → ["alice", "bob"]                            │
│    reviews.stars → [5, 2]                                      │
│                                                                │
│  ⚠ Cross-object matches are possible.                          │
│    Example: alice + 2 stars can match even if alice gave 5.    │
├────────────────────────────────────────────────────────────────┤
│  NESTED OBJECT                                                 │
│  Each nested object becomes a hidden Lucene document linked    │
│  to the root document:                                         │
│    doc 0 (nested): user=alice, stars=5                         │
│    doc 1 (nested): user=bob,   stars=2                         │
│    doc 2 (root):   name="Wireless Mouse"                       │
│                                                                │
│  ✔ Cross-object matching is prevented.                         │
├────────────────────────────────────────────────────────────────┤
│  PARENT-CHILD (join field)                                     │
│  Parent and child records are separate Elasticsearch documents │
│  connected by a join field:                                    │
│    Doc 1   (parent/author): name="George Orwell"               │
│    Doc 101 (child/book):    title="1984", parent="1"           │
│                                                                │
│  ✔ Children can be updated without reindexing the parent.      │
│  ⚠ Parent and children must live on the same shard.            │
│    This means child documents need routing.                    │
└────────────────────────────────────────────────────────────────┘
```

### Prerequisite

Before you begin, make sure both containers are running. If you already created them previously, you can start them instead of creating new ones:

```bash
docker start elasticsearch
docker start kibana
```

If the containers do not exist yet, create and run **Elasticsearch**:

```bash
docker run -d \
  --name elasticsearch \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  docker.elastic.co/elasticsearch/elasticsearch:8.6.0
```

Then create and run **Kibana**:

```bash
docker run -d \
  --name kibana \
  -p 5601:5601 \
  --link elasticsearch:elasticsearch \
  docker.elastic.co/kibana/kibana:8.6.0
```

Verify that both services are accessible:

* **Elasticsearch:** `http://localhost:9200`
* **Kibana:** `http://localhost:5601`

You can confirm Elasticsearch is running by opening `http://localhost:9200` in your browser or by running:

```bash
curl http://localhost:9200
```

### Why Relational Data Modeling Matters

Choosing the wrong modeling strategy can lead to three common problems:

1. **Incorrect results** — regular `object` fields can match values from different objects in the same array.
2. **Expensive updates** — nested objects preserve correct matching, but updating one nested item requires reindexing the whole parent document.
3. **Higher memory and query cost** — parent-child relationships allow independent child updates, but queries are usually slower because Elasticsearch must maintain join metadata such as global ordinals.

Use this rule of thumb:

- Use **object** when the fields do not need to stay tied to the same object.
- Use **nested** when each object in an array must be searched independently.
- Use **parent-child** when child records change often and must be updated separately from the parent.

### Lab Steps

#### Step 0 — Reset the Lab Data

Run this first if you have completed the lab before. It removes old versions of the indices so the mappings and sample data below can be recreated cleanly.

```json
DELETE /products?ignore_unavailable=true
DELETE /products_nested?ignore_unavailable=true
DELETE /library?ignore_unavailable=true
```

**Expected output:**

```json
{ "acknowledged": true }
```

> You may see one response for each `DELETE` request. If an index did not exist, the request is still safe because `ignore_unavailable=true` is enabled.

#### Step 1 — Object Type (Default): The Flattening Problem

A regular `object` field is the default way Elasticsearch stores objects. This is simple and fast, but arrays of objects are flattened into separate multi-value fields.

Create a `products` index that uses the default object behavior for `reviews`:

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "category": { "type": "keyword" },
      "price": { "type": "double" },
      "reviews": {
        "properties": {
          "user": { "type": "keyword" },
          "stars": { "type": "integer" },
          "verified": { "type": "boolean" },
          "comment": { "type": "text" }
        }
      }
    }
  }
}
```

Fill the index with sample products and reviews:

```json
POST /products/_bulk
{ "index": { "_id": "1" } }
{ "name": "Wireless Mouse", "category": "Accessories", "price": 29.99, "reviews": [ { "user": "alice", "stars": 5, "verified": true, "comment": "Excellent build quality" }, { "user": "bob", "stars": 2, "verified": true, "comment": "Stopped working after a week" } ] }
{ "index": { "_id": "2" } }
{ "name": "Mechanical Keyboard", "category": "Accessories", "price": 89.99, "reviews": [ { "user": "alice", "stars": 4, "verified": true, "comment": "Great switches and solid frame" }, { "user": "carol", "stars": 5, "verified": false, "comment": "Very satisfying typing feel" } ] }
{ "index": { "_id": "3" } }
{ "name": "USB-C Hub", "category": "Adapters", "price": 39.99, "reviews": [ { "user": "dan", "stars": 2, "verified": true, "comment": "Gets hot under load" }, { "user": "erin", "stars": 4, "verified": true, "comment": "Useful ports for travel" } ] }

POST /products/_refresh
```

Now query for a product where **alice gave 2 stars**:

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "reviews.user": "alice" } },
        { "term": { "reviews.stars": 2 } }
      ]
    }
  }
}
```

**Expected output** — incorrect cross-object match:

```json
{
  "hits": {
    "total": { "value": 1, "relation": "eq" },
    "hits": [
      {
        "_id": "1",
        "_source": {
          "name": "Wireless Mouse",
          "reviews": [
            { "user": "alice", "stars": 5, "verified": true, "comment": "Excellent build quality" },
            { "user": "bob", "stars": 2, "verified": true, "comment": "Stopped working after a week" }
          ]
        }
      }
    ]
  }
}
```

This result is wrong from a business perspective. Alice gave the `Wireless Mouse` **5 stars**, not 2. The product matched because Elasticsearch flattened the reviews into independent arrays:

```text
reviews.user  = ["alice", "bob"]
reviews.stars = [5, 2]
```

The query found `alice` somewhere in `reviews.user` and `2` somewhere in `reviews.stars`, but it did not know those values belonged to different review objects.

#### Step 2 — Nested Type: Solving the Flattening Problem

Use the `nested` type when each object in an array must keep its own identity during search.

Create a second index where `reviews` is mapped as `nested`:

```json
PUT /products_nested
{
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "category": { "type": "keyword" },
      "price": { "type": "double" },
      "reviews": {
        "type": "nested",
        "properties": {
          "user": { "type": "keyword" },
          "stars": { "type": "integer" },
          "verified": { "type": "boolean" },
          "comment": { "type": "text" }
        }
      }
    }
  }
}
```

Fill the nested index with the same sample data:

```json
POST /products_nested/_bulk
{ "index": { "_id": "1" } }
{ "name": "Wireless Mouse", "category": "Accessories", "price": 29.99, "reviews": [ { "user": "alice", "stars": 5, "verified": true, "comment": "Excellent build quality" }, { "user": "bob", "stars": 2, "verified": true, "comment": "Stopped working after a week" } ] }
{ "index": { "_id": "2" } }
{ "name": "Mechanical Keyboard", "category": "Accessories", "price": 89.99, "reviews": [ { "user": "alice", "stars": 4, "verified": true, "comment": "Great switches and solid frame" }, { "user": "carol", "stars": 5, "verified": false, "comment": "Very satisfying typing feel" } ] }
{ "index": { "_id": "3" } }
{ "name": "USB-C Hub", "category": "Adapters", "price": 39.99, "reviews": [ { "user": "dan", "stars": 2, "verified": true, "comment": "Gets hot under load" }, { "user": "erin", "stars": 4, "verified": true, "comment": "Useful ports for travel" } ] }

POST /products_nested/_refresh
```

Run the same logical query: **alice gave 2 stars**. This time the query is wrapped in a `nested` query so both conditions must match inside the same review object.

```json
GET /products_nested/_search
{
  "query": {
    "nested": {
      "path": "reviews",
      "query": {
        "bool": {
          "must": [
            { "term": { "reviews.user": "alice" } },
            { "term": { "reviews.stars": 2 } }
          ]
        }
      }
    }
  }
}
```

**Expected output:**

```json
{
  "hits": {
    "total": { "value": 0, "relation": "eq" },
    "hits": []
  }
}
```

This is now correct. Alice has reviews in the dataset, and 2-star reviews exist in the dataset, but no single review object has both `user = alice` and `stars = 2`.

Now run a query that should match: **alice gave 5 stars**.

```json
GET /products_nested/_search
{
  "query": {
    "nested": {
      "path": "reviews",
      "query": {
        "bool": {
          "must": [
            { "term": { "reviews.user": "alice" } },
            { "term": { "reviews.stars": 5 } }
          ]
        }
      }
    }
  }
}
```

**Expected output:**

```json
{
  "hits": {
    "total": { "value": 1, "relation": "eq" },
    "hits": [
      {
        "_id": "1",
        "_source": {
          "name": "Wireless Mouse",
          "category": "Accessories",
          "price": 29.99
        }
      }
    ]
  }
}
```

#### Step 3 — `inner_hits`: Retrieving Matching Nested Objects

A nested query returns the parent document by default. That can be confusing because the parent document may contain matching and non-matching nested objects.

Add `inner_hits` when you want Elasticsearch to show the exact nested review objects that matched.

Find products that have at least one review with **4 or more stars**:

```json
GET /products_nested/_search
{
  "query": {
    "nested": {
      "path": "reviews",
      "query": {
        "range": {
          "reviews.stars": { "gte": 4 }
        }
      },
      "inner_hits": {
        "size": 5,
        "_source": ["reviews.user", "reviews.stars", "reviews.comment", "reviews.verified"]
      }
    }
  }
}
```

**Expected output (trimmed):**

```json
{
  "hits": {
    "total": { "value": 3, "relation": "eq" },
    "hits": [
      {
        "_id": "1",
        "_source": { "name": "Wireless Mouse" },
        "inner_hits": {
          "reviews": {
            "hits": {
              "hits": [
                {
                  "_nested": { "field": "reviews", "offset": 0 },
                  "_source": {
                    "user": "alice",
                    "stars": 5,
                    "verified": true,
                    "comment": "Excellent build quality"
                  }
                }
              ]
            }
          }
        }
      }
    ]
  }
}
```

The parent product is returned in the main `hits` array. The matching nested review is returned inside `inner_hits.reviews.hits.hits`.

For `_id = 1`, only Alice's review appears because Bob's 2-star review does not satisfy `reviews.stars >= 4`.

#### Step 4 — Parent-Child Relationships with the Join Field

Use parent-child when related records need to be updated independently. A common example is an author with many books, or a product with many events, where child documents change more often than the parent.

Important constraints:

- Parent and child documents must be in the **same index**.
- Child documents must be routed to the same shard as their parent.
- Parent-child queries are usually slower than nested queries.

Create a `library` index with a `join` field:

```json
PUT /library
{
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "country": { "type": "keyword" },
      "title": { "type": "text" },
      "genre": { "type": "keyword" },
      "published_year": { "type": "integer" },
      "rating": { "type": "float" },
      "my_join_field": {
        "type": "join",
        "relations": {
          "author": "book"
        }
      }
    }
  }
}
```

Index parent documents first. In this example, the parents are authors:

```json
POST /library/_bulk
{ "index": { "_id": "1" } }
{ "name": "George Orwell", "country": "UK", "my_join_field": "author" }
{ "index": { "_id": "2" } }
{ "name": "Aldous Huxley", "country": "UK", "my_join_field": "author" }
{ "index": { "_id": "3" } }
{ "name": "Ursula K. Le Guin", "country": "US", "my_join_field": "author" }
```

Index child documents next. In this example, the children are books.

The `routing` value must be the parent document ID. For example, books by George Orwell use `routing=1` because George Orwell's author document has `_id = 1`.

```json
POST /library/_bulk?routing=1
{ "index": { "_id": "101" } }
{ "title": "1984", "genre": "dystopian", "published_year": 1949, "rating": 4.8, "my_join_field": { "name": "book", "parent": "1" } }
{ "index": { "_id": "102" } }
{ "title": "Animal Farm", "genre": "satire", "published_year": 1945, "rating": 4.5, "my_join_field": { "name": "book", "parent": "1" } }

POST /library/_bulk?routing=2
{ "index": { "_id": "201" } }
{ "title": "Brave New World", "genre": "dystopian", "published_year": 1932, "rating": 4.6, "my_join_field": { "name": "book", "parent": "2" } }

POST /library/_bulk?routing=3
{ "index": { "_id": "301" } }
{ "title": "The Left Hand of Darkness", "genre": "science_fiction", "published_year": 1969, "rating": 4.7, "my_join_field": { "name": "book", "parent": "3" } }
{ "index": { "_id": "302" } }
{ "title": "The Dispossessed", "genre": "science_fiction", "published_year": 1974, "rating": 4.6, "my_join_field": { "name": "book", "parent": "3" } }

POST /library/_refresh
```

#### Step 5 — `has_child` Query: Find Parents by Their Children

Use `has_child` when you want to find parent documents based on child document criteria.

Find authors who wrote at least one dystopian book:

```json
GET /library/_search
{
  "query": {
    "has_child": {
      "type": "book",
      "query": {
        "term": { "genre": "dystopian" }
      }
    }
  }
}
```

**Expected output:**

```json
{
  "hits": {
    "total": { "value": 2, "relation": "eq" },
    "hits": [
      { "_id": "1", "_source": { "name": "George Orwell", "country": "UK" } },
      { "_id": "2", "_source": { "name": "Aldous Huxley", "country": "UK" } }
    ]
  }
}
```

Find authors with at least two science fiction books:

```json
GET /library/_search
{
  "query": {
    "has_child": {
      "type": "book",
      "query": {
        "term": { "genre": "science_fiction" }
      },
      "min_children": 2
    }
  }
}
```

**Expected output:**

```json
{
  "hits": {
    "total": { "value": 1, "relation": "eq" },
    "hits": [
      { "_id": "3", "_source": { "name": "Ursula K. Le Guin", "country": "US" } }
    ]
  }
}
```

#### Step 6 — `has_parent` Query: Find Children by Their Parent

Use `has_parent` when you want to find child documents based on parent document criteria.

Find all books by George Orwell:

```json
GET /library/_search
{
  "query": {
    "has_parent": {
      "parent_type": "author",
      "query": {
        "match": { "name": "George Orwell" }
      }
    }
  }
}
```

**Expected output:**

```json
{
  "hits": {
    "total": { "value": 2, "relation": "eq" },
    "hits": [
      { "_id": "101", "_source": { "title": "1984", "genre": "dystopian", "published_year": 1949 } },
      { "_id": "102", "_source": { "title": "Animal Farm", "genre": "satire", "published_year": 1945 } }
    ]
  }
}
```

Find all books whose parent author is from the UK:

```json
GET /library/_search
{
  "query": {
    "has_parent": {
      "parent_type": "author",
      "query": {
        "term": { "country": "UK" }
      }
    }
  }
}
```

**Expected output:**

```json
{
  "hits": {
    "total": { "value": 3, "relation": "eq" },
    "hits": [
      { "_id": "101", "_source": { "title": "1984", "genre": "dystopian" } },
      { "_id": "102", "_source": { "title": "Animal Farm", "genre": "satire" } },
      { "_id": "201", "_source": { "title": "Brave New World", "genre": "dystopian" } }
    ]
  }
}
```

### Comparison Table: Object vs Nested vs Parent-Child

| Criteria | Object (default) | Nested | Parent-Child (Join) |
|---|---|---|---|
| **Storage** | One Lucene document | Root document plus hidden nested Lucene documents | Separate Elasticsearch documents |
| **Query accuracy** | Can produce cross-object matches ⚠ | Accurate for arrays of objects ✔ | Accurate across parent and child documents ✔ |
| **Query speed** | Fastest | Usually fast, but slower than simple objects | Usually slowest because joins are more expensive |
| **Update cost** | Reindex the whole document | Reindex the whole parent document | Update child independently ✔ |
| **Memory cost** | Low | Low to moderate | Higher because of join metadata/global ordinals |
| **Best for** | Simple metadata or objects not queried together | Small to medium arrays where object boundaries matter | Large or frequently updated child collections |
| **Avoid when** | You need accurate matching inside arrays of objects | Nested children change very frequently | You can denormalize or use nested instead |

### Python Script: Querying Nested Data

This script queries the `products_nested` index and prints only the nested reviews that matched the query.

```python
"""Query nested reviews in the products_nested index."""
from elasticsearch import Elasticsearch

es = Elasticsearch("http://localhost:9200")

resp = es.search(
    index="products_nested",
    body={
        "query": {
            "nested": {
                "path": "reviews",
                "query": {
                    "bool": {
                        "must": [
                            {"term": {"reviews.user": "alice"}},
                            {"range": {"reviews.stars": {"gte": 4}}}
                        ]
                    }
                },
                "inner_hits": {
                    "size": 5,
                    "_source": [
                        "reviews.user",
                        "reviews.stars",
                        "reviews.comment",
                        "reviews.verified"
                    ]
                }
            }
        }
    }
)

print(f"Total products matched: {resp['hits']['total']['value']}")

for hit in resp["hits"]["hits"]:
    product = hit["_source"]["name"]
    print(f"\nProduct: {product}")

    matching_reviews = hit["inner_hits"]["reviews"]["hits"]["hits"]
    for inner in matching_reviews:
        review = inner["_source"]
        print(
            f"  ★ {review['stars']} by {review['user']} "
            f"— {review['comment']}"
        )
```

**Expected output:**

```text
Total products matched: 2

Product: Wireless Mouse
  ★ 5 by alice — Excellent build quality

Product: Mechanical Keyboard
  ★ 4 by alice — Great switches and solid frame
```

### Troubleshooting Tips

| Symptom | Cause | Fix |
|---|---|---|
| Cross-object matches | Field mapped as `object`, not `nested` | Change the mapping to `"type": "nested"` and reindex |
| `query_shard_exception` on `has_child` | Child document is not routed to the parent's shard | Add `?routing=<parent_id>` when indexing child documents |
| `inner_hits` is empty | `inner_hits` is outside the `nested` query block | Move `"inner_hits": {}` inside the nested query |
| Parent-child query returns no children | Child document has the wrong parent ID or routing value | Check both `my_join_field.parent` and `?routing=` |
| Slow parent-child queries | Join metadata/global ordinals add overhead | Prefer denormalization or nested fields when independent child updates are not required |
| `mapper_parsing_exception` | Wrong join field format in child document | Use `{ "name": "book", "parent": "<parent_id>" }` |

### Cleanup

Run this when you are finished with the lab:

```json
DELETE /products
DELETE /products_nested
DELETE /library
```

**Expected output for each index:**

```json
{ "acknowledged": true }
```

### Reflection

Answer the following questions after completing the lab:

1. Why does the default `object` type produce incorrect matches on arrays of objects?
2. How does the `nested` type prevent cross-object matches?
3. What does `inner_hits` show that the main search hits do not show?
4. What is the performance tradeoff of `nested` compared with parent-child?
5. In what scenario would you prefer parent-child over nested?
6. Why is routing required when indexing child documents in a join-field relationship?
