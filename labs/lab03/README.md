## Task 3: Basic Searching with Query DSL

**Objectives:**
- Construct search queries using Query DSL via Kibana and curl.
- Learn the syntax of match, term, range, match_phrase, wildcard, and bool queries.
- Understand full-text search versus exact matching.
- Understand relevance scoring (`_score`) and how Elasticsearch ranks results.
- Combine queries with bool (must, should, must_not, filter).

---

### Query Execution Flow

```
                         Search Request
                              |
                              v
+-----------------------------------------------------------+
|                    QUERY PHASE                            |
|                                                           |
|  Coordinating Node receives the request                   |
|         |                                                 |
|         +---> Shard 0: score & rank local matches         |
|         +---> Shard 1: score & rank local matches         |
|         +---> Shard N: score & rank local matches         |
|                    |                                      |
|         Merge sorted results (top N doc IDs + scores)     |
+--------------------+--------------------------------------+
                     |
                     v
+-----------------------------------------------------------+
|                    FETCH PHASE                            |
|                                                           |
|  Coordinating Node requests full _source for top N docs   |
|         |                                                 |
|         +---> Shard holding doc A  ---> return _source    |
|         +---> Shard holding doc B  ---> return _source    |
|         +---> Shard holding doc C  ---> return _source    |
|                    |                                      |
|         Assemble final response and return to client      |
+-----------------------------------------------------------+
```

---

### Prerequisite: Seed Sample Data

Before searching, load sample documents so queries return meaningful results:

```json
POST /products/_bulk
{"index": {"_id": "1"}}
{"name": "Elastic T-Shirt", "description": "Comfortable cotton t-shirt with Elastic logo", "price": 19.99, "category": "clothing", "in_stock": true, "created_at": "2024-01-15"}
{"index": {"_id": "2"}}
{"name": "Kibana Mug", "description": "Ceramic mug featuring Kibana dashboard artwork", "price": 12.50, "category": "accessories", "in_stock": true, "created_at": "2024-02-01"}
{"index": {"_id": "3"}}
{"name": "Logstash Hoodie", "description": "Warm hoodie with Logstash logo on front", "price": 45.00, "category": "clothing", "in_stock": true, "created_at": "2024-03-01"}
{"index": {"_id": "4"}}
{"name": "Beats Cap", "description": "Adjustable cap with Beats branding", "price": 15.00, "category": "accessories", "in_stock": true, "created_at": "2024-03-05"}
{"index": {"_id": "5"}}
{"name": "APM Sticker Pack", "description": "Set of 10 APM monitoring stickers", "price": 5.99, "category": "accessories", "in_stock": false, "created_at": "2024-03-10"}
{"index": {"_id": "6"}}
{"name": "Elastic Polo Shirt", "description": "Professional polo shirt with embroidered Elastic logo", "price": 34.99, "category": "clothing", "in_stock": true, "created_at": "2024-04-01"}
```

---

### Lab Steps

### Step 1: Match Query (Full-Text Search)

The `match` query analyzes the input text and searches the inverted index.

**In Kibana Dev Tools:**

```json
GET /products/_search
{
  "query": {
    "match": { "name": "t-shirt" }
  }
}
```

**Equivalent curl command:**
```bash
curl -s -X GET "http://localhost:9200/products/_search" \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":{"name":"t-shirt"}}}' | python3 -m json.tool
```

**Expected Output:**
```json
{
  "took": 5,
  "timed_out": false,
  "_shards": { "total": 1, "successful": 1, "skipped": 0, "failed": 0 },
  "hits": {
    "total": { "value": 1, "relation": "eq" },
    "max_score": 1.6931472,
    "hits": [
      {
        "_index": "products",
        "_id": "1",
        "_score": 1.6931472,
        "_source": {
          "name": "Elastic T-Shirt",
          "price": 19.99,
          "category": "clothing"
        }
      }
    ]
  }
}
```

---

### Step 2: Term Query (Exact Match)

The `term` query does **not** analyze the input -- it searches for the exact value. Use `keyword` fields for exact matching.

```json
GET /products/_search
{
  "query": {
    "term": { "category": "clothing" }
  }
}
```

**Expected Output:**
```json
{
  "hits": {
    "total": { "value": 3, "relation": "eq" },
    "hits": [
      { "_id": "1", "_source": { "name": "Elastic T-Shirt", "category": "clothing" } },
      { "_id": "3", "_source": { "name": "Logstash Hoodie", "category": "clothing" } },
      { "_id": "6", "_source": { "name": "Elastic Polo Shirt", "category": "clothing" } }
    ]
  }
}
```

> **Important:** `term` queries on `text` fields often fail because ES stores analyzed (lowercased, tokenized) terms. Always use `keyword` type fields for exact matching.

---

### Step 3: Range Query

Find products within a price range:

```json
GET /products/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 10,
        "lte": 30
      }
    }
  }
}
```

**Expected Output:**
```json
{
  "hits": {
    "total": { "value": 3, "relation": "eq" },
    "hits": [
      { "_id": "1", "_source": { "name": "Elastic T-Shirt", "price": 19.99 } },
      { "_id": "2", "_source": { "name": "Kibana Mug", "price": 12.50 } },
      { "_id": "4", "_source": { "name": "Beats Cap", "price": 15.00 } }
    ]
  }
}
```

**Date range query:**

```json
GET /products/_search
{
  "query": {
    "range": {
      "created_at": {
        "gte": "2024-03-01",
        "lte": "2024-12-31"
      }
    }
  }
}
```

**Expected Output:**
```json
{
  "hits": {
    "total": { "value": 4, "relation": "eq" },
    "hits": [
      { "_id": "3", "_source": { "name": "Logstash Hoodie", "created_at": "2024-03-01" } },
      { "_id": "4", "_source": { "name": "Beats Cap", "created_at": "2024-03-05" } },
      { "_id": "5", "_source": { "name": "APM Sticker Pack", "created_at": "2024-03-10" } },
      { "_id": "6", "_source": { "name": "Elastic Polo Shirt", "created_at": "2024-04-01" } }
    ]
  }
}
```

---

### Step 4: Match Phrase Query

Matches an exact sequence of words (order matters):

```json
GET /products/_search
{
  "query": {
    "match_phrase": {
      "description": "Elastic logo"
    }
  }
}
```

**Expected Output:**
```json
{
  "hits": {
    "total": { "value": 1, "relation": "eq" },
    "hits": [
      {
        "_id": "1",
        "_score": 1.89,
        "_source": {
          "name": "Elastic T-Shirt",
          "description": "Comfortable cotton t-shirt with Elastic logo"
        }
      }
    ]
  }
}
```

> **Note:** `match_phrase` requires terms to appear in the exact order and adjacent to each other, unlike `match` which finds terms anywhere in the field.

---

### Step 5: Wildcard Query

Pattern-based matching using `*` (any characters) and `?` (single character):

```json
GET /products/_search
{
  "query": {
    "wildcard": {
      "name": {
        "value": "elastic*"
      }
    }
  }
}
```

**Expected Output:**
```json
{
  "hits": {
    "total": { "value": 2, "relation": "eq" },
    "hits": [
      { "_id": "1", "_source": { "name": "Elastic T-Shirt" } },
      { "_id": "6", "_source": { "name": "Elastic Polo Shirt" } }
    ]
  }
}
```

> **Caution:** Wildcard queries with leading wildcards (`*shirt`) are expensive. Avoid them on large datasets.

---

### Step 6: Boolean Query (Combining Conditions)

The `bool` query combines multiple clauses:

| Clause | Behavior |
|---|---|
| `must` | Document **must** match (contributes to score) |
| `should` | Document **should** match (boosts score if matched) |
| `must_not` | Document **must not** match (excludes, no scoring) |
| `filter` | Document **must** match (no scoring, cached for speed) |

**Example: Find in-stock clothing priced under $40:**

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "Elastic" } }
      ],
      "filter": [
        { "term": { "category": "clothing" } },
        { "range": { "price": { "lt": 40 } } }
      ],
      "must_not": [
        { "term": { "in_stock": false } }
      ]
    }
  }
}
```

**Expected Output:**
```json
{
  "hits": {
    "total": { "value": 2, "relation": "eq" },
    "hits": [
      {
        "_id": "1",
        "_score": 0.69,
        "_source": { "name": "Elastic T-Shirt", "price": 19.99, "category": "clothing", "in_stock": true }
      },
      {
        "_id": "6",
        "_score": 0.52,
        "_source": { "name": "Elastic Polo Shirt", "price": 34.99, "category": "clothing", "in_stock": true }
      }
    ]
  }
}
```

**Example with `should` (boosting):**

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "category": "clothing" } }
      ],
      "should": [
        { "match": { "description": "logo" } },
        { "range": { "price": { "lte": 20 } } }
      ]
    }
  }
}
```

**Expected Output:** All clothing documents are returned, but those matching `should` clauses are ranked higher (higher `_score`).

---

### Step 7: Understanding Relevance Scoring (_score)

Each document receives a `_score` based on BM25. Use `explain` to see how a score is calculated:

```json
GET /products/_search
{
  "explain": true,
  "query": {
    "match": { "name": "Elastic" }
  }
}
```

**Expected Output (excerpt for one hit):**
```json
{
  "_id": "1",
  "_score": 0.6931472,
  "_explanation": {
    "value": 0.6931472,
    "description": "weight(name:elastic in 0) [PerFieldSimilarity]",
    "details": [
      { "value": 0.6931472, "description": "idf, computed as log(1 + (N - n + 0.5) / (n + 0.5))" },
      { "value": 1.0, "description": "tf, computed as freq / (freq + k1 * (1 - b + b * dl / avgdl))" }
    ]
  }
}
```

**Key scoring concepts:**

| Factor | Meaning |
|---|---|
| **TF** (Term Frequency) | How often the term appears in the field -- more occurrences = higher score |
| **IDF** (Inverse Document Frequency) | How rare the term is across all documents -- rarer terms score higher |
| **Field Length** | Shorter fields score higher because matches are more significant |

> **Tip:** Use `filter` clauses instead of `must` when you do not need scoring (e.g., exact category matches). Filters are faster because ES can cache them.

---

### Step 8: Python Search Examples

```python
import requests
import json

ES_URL = "http://localhost:9200"
SEARCH_URL = f"{ES_URL}/products/_search"
headers = {"Content-Type": "application/json"}

# --- Match query ---
query = {
    "query": { "match": { "name": "Elastic" } }
}
resp = requests.get(SEARCH_URL, json=query).json()
print("=== Match Query: 'Elastic' ===")
print(f"Total hits: {resp['hits']['total']['value']}")
for hit in resp['hits']['hits']:
    print(f"  [{hit['_score']:.4f}] {hit['_source']['name']} - ${hit['_source']['price']}")
print()

# --- Bool query ---
query = {
    "query": {
        "bool": {
            "must": [{"term": {"category": "clothing"}}],
            "filter": [{"range": {"price": {"lte": 25}}}]
        }
    }
}
resp = requests.get(SEARCH_URL, json=query).json()
print("=== Bool Query: clothing under $25 ===")
print(f"Total hits: {resp['hits']['total']['value']}")
for hit in resp['hits']['hits']:
    print(f"  {hit['_source']['name']} - ${hit['_source']['price']}")
print()

# --- Range query ---
query = {
    "query": {
        "range": {
            "created_at": { "gte": "2024-03-01", "lte": "2024-12-31" }
        }
    }
}
resp = requests.get(SEARCH_URL, json=query).json()
print("=== Range Query: created after 2024-03-01 ===")
print(f"Total hits: {resp['hits']['total']['value']}")
for hit in resp['hits']['hits']:
    print(f"  {hit['_source']['name']} ({hit['_source']['created_at']})")
```

**Expected Output:**
```
=== Match Query: 'Elastic' ===
Total hits: 2
  [0.6931] Elastic T-Shirt - $19.99
  [0.5234] Elastic Polo Shirt - $34.99

=== Bool Query: clothing under $25 ===
Total hits: 1
  Elastic T-Shirt - $19.99

=== Range Query: created after 2024-03-01 ===
Total hits: 4
  Logstash Hoodie (2024-03-01)
  Beats Cap (2024-03-05)
  APM Sticker Pack (2024-03-10)
  Elastic Polo Shirt (2024-04-01)
```

---

### Troubleshooting Tips

| Problem | Cause | Solution |
|---|---|---|
| `term` query returns 0 hits on a `text` field | `text` fields are analyzed (lowercased/tokenized) | Use `fieldname.keyword` or switch to a `match` query |
| Unexpected results from `match` | Default operator is OR (any term matches) | Add `"operator": "and"` to require all terms |
| `_score` is always 0.0 | Using only `filter` clauses in bool | Move relevant clauses to `must` or `should` |
| `search_phase_execution_exception` | Querying a field with wrong type (e.g., term on object) | Check mapping with `GET /products/_mapping` |

---

### Analysis

Note the returned documents and score details. Write a brief explanation covering:
1. How `match` differs from `term` (analyzed vs. exact).
2. When to use `filter` vs. `must` in a bool query.
3. How BM25 scoring determines document ranking.
