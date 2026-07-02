## Task 8: Advanced Querying and Query Profiling

**Objectives:**

- Build complex Elasticsearch queries that combine `bool`, `function_score`, boosts, filters, and scripts.
- Use the `profile` flag to inspect how Elasticsearch executes a query.
- Use Python to benchmark several query styles and compare their performance.
- Understand the query profiling breakdown and what each timing phase means.
- Configure search slow logs to detect expensive queries.
- Apply practical query optimization techniques.

### Query Profiling Breakdown

Elasticsearch can explain query execution when you add `"profile": true` to a search request.

Profiling is useful when a query is unexpectedly slow because it shows where time is spent. For example, it can reveal whether the query spends most of its time building scorers, advancing through matching documents, or calculating relevance scores.

```
  ┌────────────────────────────────────────────────────────┐
  │              QUERY PROFILING PHASES                    │
  │                                                        │
  │  ┌─────────────┐                                       │
  │  │  rewrite    │  Simplify or optimize the query tree  │
  │  └──────┬──────┘                                       │
  │         │                                              │
  │         ▼                                              │
  │  ┌───────────────┐                                     │
  │  │ create_weight │  Build internal query structures    │
  │  └──────┬────────┘   used for scoring                  │
  │         │                                              │
  │         ▼                                              │
  │  ┌──────────────┐                                      │
  │  │ build_scorer │  Create scorer iterators for         │
  │  └──────┬───────┘   each segment                       │
  │         │                                              │
  │         ▼                                              │
  │  ┌──────────────┐                                      │
  │  │ next_doc     │  Move to the next matching document  │
  │  └──────┬───────┘                                      │
  │         │                                              │
  │         ▼                                              │
  │  ┌──────────────┐                                      │
  │  │   score      │  Calculate relevance score for       │
  │  └──────┬───────┘   each matching document             │
  │         │                                              │
  │         ▼                                              │
  │  ┌──────────────┐                                      │
  │  │   advance    │  Skip directly to a target document  │
  │  └──────┬───────┘   ID, often used by conjunctions     │
  │         │                                              │
  │         ▼                                              │
  │     Results                                            │
  └────────────────────────────────────────────────────────┘
```

### Prerequisite

Before you begin, make sure both containers are running. If you created them previously, start them again:

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

### Task 0: Create and Fill the `products` Index

The queries in this task use a `products` index. Before running the search examples, create the index and insert sample product data.

Run the following commands in **Kibana Dev Tools**.

#### Delete the Existing Index, If Needed

Use this command if you want to start from a clean database.

```json
DELETE /products
```

If the index does not exist, Elasticsearch returns a `404` error. That is safe to ignore.

#### Create the Index Mapping

This mapping creates fields that support both full-text search and exact filtering:

- `name` and `description` are `text` fields, which are good for full-text search.
- `category`, `status`, and `tags` are `keyword` fields, which are good for exact filters, sorting, and aggregations.
- `price`, `rating`, `sales_count`, and `popularity` are numeric fields, which are useful for ranges and scoring functions.
- `in_stock` is a boolean field, which supports fast filtering.

```json
PUT /products
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "description": { "type": "text" },
      "category": { "type": "keyword" },
      "status": { "type": "keyword" },
      "price": { "type": "double" },
      "in_stock": { "type": "boolean" },
      "rating": { "type": "float" },
      "sales_count": { "type": "integer" },
      "popularity": { "type": "integer" },
      "tags": { "type": "keyword" },
      "created_at": { "type": "date" }
    }
  }
}
```

#### Insert Sample Product Documents

The following bulk request creates a realistic dataset with electronics, books, apparel, and discontinued products. It is designed so the later queries return meaningful results.

```json
POST /products/_bulk?refresh=true
{ "index": { "_id": "1" } }
{ "name": "Elastic Widget", "description": "A compact scalable distributed widget for Elastic-powered dashboards.", "category": "Electronics", "status": "active", "price": 29.99, "in_stock": true, "rating": 4.7, "sales_count": 520, "popularity": 91, "tags": ["elastic", "widget", "dashboard"], "created_at": "2024-01-15" }
{ "index": { "_id": "2" } }
{ "name": "Elastic Search Appliance", "description": "A scalable distributed search appliance for logs, metrics, and application data.", "category": "Electronics", "status": "active", "price": 149.00, "in_stock": true, "rating": 4.9, "sales_count": 870, "popularity": 98, "tags": ["elastic", "search", "appliance"], "created_at": "2024-02-10" }
{ "index": { "_id": "3" } }
{ "name": "Elastic Dev Kit", "description": "Developer kit with sensors, cables, and a lightweight Elastic monitoring demo.", "category": "Electronics", "status": "active", "price": 79.50, "in_stock": true, "rating": 4.4, "sales_count": 330, "popularity": 84, "tags": ["elastic", "developer", "kit"], "created_at": "2024-03-22" }
{ "index": { "_id": "4" } }
{ "name": "Elastic Cloud Handbook", "description": "Practical book about scalable distributed search architecture and operations.", "category": "Books", "status": "active", "price": 39.95, "in_stock": true, "rating": 4.6, "sales_count": 410, "popularity": 76, "tags": ["elastic", "book", "cloud"], "created_at": "2024-04-02" }
{ "index": { "_id": "5" } }
{ "name": "Distributed Systems Poster", "description": "Large wall poster explaining nodes, shards, replicas, and scalable system design.", "category": "Office", "status": "active", "price": 12.99, "in_stock": true, "rating": 4.1, "sales_count": 140, "popularity": 55, "tags": ["distributed", "poster"], "created_at": "2024-04-18" }
{ "index": { "_id": "6" } }
{ "name": "Elastic Legacy Widget", "description": "Older Elastic widget model with limited compatibility.", "category": "Discontinued", "status": "archived", "price": 19.99, "in_stock": true, "rating": 3.2, "sales_count": 70, "popularity": 22, "tags": ["elastic", "legacy"], "created_at": "2023-11-05" }
{ "index": { "_id": "7" } }
{ "name": "Premium Elastic Console", "description": "High-end console for large observability deployments.", "category": "Electronics", "status": "active", "price": 499.00, "in_stock": true, "rating": 4.8, "sales_count": 180, "popularity": 88, "tags": ["elastic", "console", "premium"], "created_at": "2024-05-20" }
{ "index": { "_id": "8" } }
{ "name": "Elastic Cable Pack", "description": "Replacement cables for Elastic hardware demos.", "category": "Electronics", "status": "active", "price": 9.99, "in_stock": true, "rating": 4.0, "sales_count": 260, "popularity": 63, "tags": ["elastic", "cable"], "created_at": "2024-06-11" }
{ "index": { "_id": "9" } }
{ "name": "Search Analytics Notebook", "description": "Notebook for planning search relevance experiments and profiling results.", "category": "Office", "status": "active", "price": 14.50, "in_stock": false, "rating": 4.3, "sales_count": 95, "popularity": 48, "tags": ["search", "analytics", "notebook"], "created_at": "2024-06-28" }
{ "index": { "_id": "10" } }
{ "name": "Elastic Training Hoodie", "description": "Comfortable hoodie for Elastic training workshops.", "category": "Apparel", "status": "active", "price": 54.99, "in_stock": true, "rating": 4.5, "sales_count": 620, "popularity": 79, "tags": ["elastic", "training", "apparel"], "created_at": "2024-07-08" }
{ "index": { "_id": "11" } }
{ "name": "Scalable Storage Drive", "description": "Fast external drive designed for log archives and distributed backups.", "category": "Electronics", "status": "active", "price": 119.99, "in_stock": true, "rating": 4.2, "sales_count": 285, "popularity": 72, "tags": ["storage", "backup", "distributed"], "created_at": "2024-07-21" }
{ "index": { "_id": "12" } }
{ "name": "Query Profiling Workbook", "description": "Hands-on exercises for profiling search queries and interpreting slow logs.", "category": "Books", "status": "active", "price": 24.99, "in_stock": true, "rating": 4.6, "sales_count": 210, "popularity": 69, "tags": ["profiling", "queries", "workbook"], "created_at": "2024-08-01" }
```

#### Confirm the Data Was Inserted

```json
GET /products/_count
```

**Expected output:**

```json
{
  "count": 12
}
```

You can also preview the inserted documents:

```json
GET /products/_search
{
  "query": { "match_all": {} },
  "_source": ["name", "category", "price", "in_stock", "rating"],
  "sort": [{ "price": "asc" }],
  "size": 12
}
```

### Task 1: Complex Query Construction (Kibana)

This task demonstrates how different query types can be combined. The goal is to find relevant products while also applying business rules such as price limits, stock availability, and category restrictions.

#### Nested Bool Query with Multiple Clauses

A `bool` query combines several conditions:

- `must` clauses are required and contribute to scoring.
- `should` clauses are optional, but can improve the score.
- `filter` clauses are required but do **not** contribute to scoring.
- `must_not` clauses exclude matching documents.
- `minimum_should_match` controls how many `should` clauses must match.

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "Elastic" } }
      ],
      "should": [
        { "match": { "description": "scalable distributed" } },
        { "term":  { "category": { "value": "Electronics", "boost": 2.0 } } }
      ],
      "filter": [
        { "range": { "price": { "gte": 10, "lte": 200 } } },
        { "term":  { "in_stock": true } }
      ],
      "must_not": [
        { "term": { "category": "Discontinued" } }
      ],
      "minimum_should_match": 1
    }
  },
  "highlight": {
    "fields": {
      "name": {},
      "description": {}
    }
  }
}
```

**What this query does:**

- Finds products with `Elastic` in the `name`.
- Boosts products in the `Electronics` category.
- Also rewards descriptions that mention `scalable distributed`.
- Limits results to products priced between `10` and `200`.
- Requires products to be in stock.
- Excludes discontinued products.
- Returns highlights for matching text in `name` and `description`.

**Expected output:**

```json
{
  "took": 5,
  "hits": {
    "total": { "value": 4, "relation": "eq" },
    "max_score": 5.81,
    "hits": [
      {
        "_id": "2",
        "_score": 5.81,
        "_source": {
          "name": "Elastic Search Appliance",
          "price": 149.0,
          "category": "Electronics"
        },
        "highlight": {
          "name": ["<em>Elastic</em> Search Appliance"],
          "description": ["A <em>scalable</em> <em>distributed</em> search appliance for logs, metrics, and application data."]
        }
      }
    ]
  }
}
```

> Note: Scores and timing values can vary slightly between machines and Elasticsearch versions.

#### Function Score with Multiple Functions

A `function_score` query changes the score returned by a normal query. This is useful when relevance is not only about text matching. For example, you might want to boost products by category, price, rating, or popularity.

```json
GET /products/_search
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [
            { "match": { "name": "Elastic" } }
          ],
          "filter": [
            { "range": { "price": { "gte": 10, "lte": 200 } } }
          ]
        }
      },
      "functions": [
        {
          "filter": { "term": { "category": "Electronics" } },
          "weight": 3
        },
        {
          "field_value_factor": {
            "field": "price",
            "factor": 0.1,
            "modifier": "log1p",
            "missing": 1
          }
        },
        {
          "field_value_factor": {
            "field": "rating",
            "factor": 1.2,
            "modifier": "none",
            "missing": 1
          }
        },
        {
          "random_score": {
            "seed": 42,
            "field": "_seq_no"
          },
          "weight": 0.5
        },
        {
          "gauss": {
            "price": {
              "origin": 50,
              "scale": 20,
              "decay": 0.5
            }
          }
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply",
      "max_boost": 10
    }
  },
  "_source": ["name", "category", "price", "rating", "sales_count"]
}
```

**What this query does:**

- Starts with products that match `Elastic` in the name.
- Filters to products between `10` and `200`.
- Adds extra weight to electronics.
- Uses `price` and `rating` as scoring signals.
- Adds a small deterministic random score using seed `42`.
- Rewards products with prices close to `50`.
- Multiplies the final function score by the original text relevance score.

#### Script Score Query

A `script_score` query gives you full control over scoring logic. It is powerful, but usually slower than built-in scoring functions because Elasticsearch must execute a script for each matching document.

```json
GET /products/_search
{
  "query": {
    "script_score": {
      "query": {
        "match": { "name": "Elastic" }
      },
      "script": {
        "source": "_score * Math.log(2 + doc['price'].value) * (doc['in_stock'].value ? 1.5 : 0.5)"
      }
    }
  },
  "_source": ["name", "price", "in_stock", "category"]
}
```

**What this query does:**

- Starts with products that match `Elastic` in the name.
- Multiplies the original relevance score by a logarithmic price factor.
- Boosts products that are in stock.
- Reduces the score of products that are not in stock.

### Task 2: Query Profiling — Detailed Analysis

Query profiling helps you understand how Elasticsearch executes a search. Add `"profile": true` to your query to get a detailed timing breakdown.

Enable the profiler on the bool query:

```json
GET /products/_search
{
  "profile": true,
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "Elastic" } }
      ],
      "filter": [
        { "range": { "price": { "gte": 10, "lte": 50 } } }
      ]
    }
  }
}
```

**Expected profile output (simplified):**

```json
{
  "profile": {
    "shards": [
      {
        "id": "[node1][products][0]",
        "searches": [
          {
            "query": [
              {
                "type": "BooleanQuery",
                "description": "+name:elastic #price:[10.0 TO 50.0]",
                "time_in_nanos": 285430,
                "breakdown": {
                  "score":              95200,
                  "build_scorer_count": 2,
                  "build_scorer":       48300,
                  "create_weight":      32100,
                  "create_weight_count": 1,
                  "next_doc":           72800,
                  "next_doc_count":     15,
                  "advance":            0,
                  "advance_count":      0,
                  "match":              18500,
                  "match_count":        15,
                  "score_count":        12
                },
                "children": [
                  {
                    "type": "TermQuery",
                    "description": "name:elastic",
                    "time_in_nanos": 142000,
                    "breakdown": {
                      "score":         62000,
                      "build_scorer":  22000,
                      "create_weight": 18000,
                      "next_doc":      35000,
                      "advance":       0
                    }
                  },
                  {
                    "type": "IndexOrDocValuesQuery",
                    "description": "price:[10.0 TO 50.0]",
                    "time_in_nanos": 98000,
                    "breakdown": {
                      "score":         0,
                      "build_scorer":  26000,
                      "create_weight": 14000,
                      "next_doc":      38000,
                      "advance":       0
                    }
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

The exact numbers will be different on your machine. Focus on the relative cost of each phase rather than the specific nanosecond values.

---

### Understanding Profile Timing Fields

| Field            | Description                                                  |
|------------------|--------------------------------------------------------------|
| `create_weight`  | Time spent building internal query objects for each segment. |
| `build_scorer`   | Time spent creating iterators that walk through matches.     |
| `next_doc`       | Time spent moving from one matching document to the next.    |
| `advance`        | Time spent skipping directly to a specific document ID.      |
| `score`          | Time spent calculating relevance scores.                     |
| `match`          | Time spent verifying exact positions for phrase queries.     |
| `*_count`        | Number of times that operation was invoked.                  |

> **Tip:** If `score` dominates, move non-relevance conditions into `filter` context. Filters do not calculate relevance scores and can often be cached. If `next_doc` dominates, the query may be matching too many documents.

### Task 3: Search Slow Logs

Search slow logs help detect expensive queries in development and production. They record queries that exceed configured time thresholds.

Configure Elasticsearch to log slow queries:

```json
PUT /products/_settings
{
  "index.search.slowlog.threshold.query.warn":  "2s",
  "index.search.slowlog.threshold.query.info":  "1s",
  "index.search.slowlog.threshold.query.debug": "500ms",
  "index.search.slowlog.threshold.query.trace": "200ms",
  "index.search.slowlog.threshold.fetch.warn":  "1s",
  "index.search.slowlog.threshold.fetch.info":  "500ms",
  "index.search.slowlog.level": "info"
}
```

**Expected output:**

```json
{
  "acknowledged": true
}
```

Slow log entries appear in:

```text
<ES_HOME>/logs/<cluster_name>_index_search_slowlog.log
```

Example slow log entry:

```text
[2024-01-15T14:30:00,123][WARN][i.s.s.query] [es1] [products][0]
  took[2.3s], took_millis[2300], total_hits[15000],
  source[{"query":{"match_all":{}}}]
```

To test the slow log in a small local dataset, temporarily lower the threshold:

```json
PUT /products/_settings
{
  "index.search.slowlog.threshold.query.info": "1ms",
  "index.search.slowlog.level": "info"
}
```

Then run a query:

```json
GET /products/_search
{
  "query": { "match_all": {} },
  "size": 12
}
```

After testing, restore more realistic thresholds so the logs do not become noisy.

### Task 4: Query Optimization Techniques

The following techniques help reduce query cost and improve response time.

#### Use `filter` Context for Non-Scoring Clauses

Use `filter` when a condition is required but does not need to affect relevance scoring.

```json
// SLOWER: scoring a range unnecessarily
{
  "query": {
    "bool": {
      "must": [
        { "range": { "price": { "gte": 10 } } }
      ]
    }
  }
}
```

```json
// FASTER: filter context skips scoring
{
  "query": {
    "bool": {
      "filter": [
        { "range": { "price": { "gte": 10 } } }
      ]
    }
  }
}
```

#### Avoid Wildcards at the Start of a Pattern

Leading wildcards are expensive because Elasticsearch may need to scan many terms.

```json
// SLOW: leading wildcard scans many terms
{
  "query": {
    "wildcard": {
      "name.keyword": "*Elastic*"
    }
  }
}
```

```json
// FASTER: prefix-style query
{
  "query": {
    "match_phrase_prefix": {
      "name": "Elastic"
    }
  }
}
```

#### Limit Returned Fields with `_source`

Returning fewer fields reduces network transfer and response parsing time.

```json
GET /products/_search
{
  "query": { "match_all": {} },
  "_source": ["name", "price"],
  "size": 10
}
```

#### Use `terminate_after` for Existence Checks

If you only need to know whether at least one matching document exists, stop after the first match.

```json
GET /products/_search
{
  "query": {
    "term": { "category": "Electronics" }
  },
  "terminate_after": 1,
  "size": 0
}
```

#### Prefer `keyword` Fields for Exact Matches

Use `keyword` fields for exact values such as status, category, tags, and IDs.

```json
// SLOWER or less precise: analyzed text-style matching
{
  "query": {
    "match": { "status": "active" }
  }
}
```

```json
// FASTER and exact: keyword field
{
  "query": {
    "term": { "status": "active" }
  }
}
```

#### Use `size: 0` When You Only Need Counts or Aggregations

If you do not need actual documents, avoid returning hits.

```json
GET /products/_search
{
  "size": 0,
  "query": {
    "term": { "in_stock": true }
  },
  "aggs": {
    "products_by_category": {
      "terms": {
        "field": "category"
      }
    }
  }
}
```

### Task 5: Python Query Benchmarking Script

This script runs several queries multiple times and reports average, median, minimum, and maximum response times.

It uses the same `products` index created earlier.

```python
import requests
import time
from statistics import mean, median

ES_URL = "http://localhost:9200"
INDEX = "products"
ITERATIONS = 20

queries = {
    "simple_match": {
        "query": {
            "match": { "name": "Elastic" }
        }
    },
    "bool_with_filter": {
        "query": {
            "bool": {
                "must": [
                    { "match": { "name": "Elastic" } }
                ],
                "filter": [
                    { "range": { "price": { "gte": 10, "lte": 200 } } },
                    { "term": { "in_stock": True } }
                ]
            }
        }
    },
    "function_score": {
        "query": {
            "function_score": {
                "query": {
                    "match": { "name": "Elastic" }
                },
                "functions": [
                    {
                        "field_value_factor": {
                            "field": "price",
                            "modifier": "log1p",
                            "missing": 1
                        }
                    },
                    {
                        "field_value_factor": {
                            "field": "rating",
                            "factor": 1.2,
                            "missing": 1
                        }
                    }
                ],
                "score_mode": "sum",
                "boost_mode": "multiply"
            }
        }
    },
    "script_score": {
        "query": {
            "script_score": {
                "query": {
                    "match": { "name": "Elastic" }
                },
                "script": {
                    "source": "_score * Math.log(2 + doc['price'].value)"
                }
            }
        }
    }
}


def run_search(query_body):
    response = requests.post(
        f"{ES_URL}/{INDEX}/_search",
        json=query_body,
        timeout=10
    )
    response.raise_for_status()
    return response.json()


print(f"{'Query':<25} {'Avg (ms)':<12} {'Median (ms)':<12} {'Min (ms)':<10} {'Max (ms)':<10} {'Hits':<8}")
print("-" * 77)

for name, body in queries.items():
    times = []
    hit_count = 0

    for _ in range(ITERATIONS):
        start = time.perf_counter()
        result = run_search(body)
        elapsed_ms = (time.perf_counter() - start) * 1000

        times.append(elapsed_ms)
        hit_count = result.get("hits", {}).get("total", {}).get("value", 0)

    print(
        f"{name:<25} "
        f"{mean(times):<12.2f} "
        f"{median(times):<12.2f} "
        f"{min(times):<10.2f} "
        f"{max(times):<10.2f} "
        f"{hit_count:<8}"
    )

# Profile one query for deeper inspection.
print("\n--- Profiling function_score query ---")

profile_body = {
    **queries["function_score"],
    "profile": True
}

profile_result = run_search(profile_body)

for shard in profile_result.get("profile", {}).get("shards", []):
    for search in shard.get("searches", []):
        for query in search.get("query", []):
            total_ns = query.get("time_in_nanos", 0)
            breakdown = query.get("breakdown", {})

            print(f"\n  Query type: {query.get('type')}")
            print(f"  Total time: {total_ns / 1_000_000:.2f} ms")
            print("  Breakdown:")

            for key, value in sorted(breakdown.items()):
                if key.endswith("_count") or value <= 0:
                    continue

                pct = (value / total_ns) * 100 if total_ns else 0
                print(f"    {key:<20} {value / 1_000_000:>8.2f} ms  ({pct:>5.1f}%)")
```

**Expected output:**

```text
Query                     Avg (ms)     Median (ms)  Min (ms)   Max (ms)   Hits
-----------------------------------------------------------------------------
simple_match              3.25         2.80         1.90       8.50       7
bool_with_filter          3.10         2.65         1.85       7.20       5
function_score            4.80         4.20         2.50       12.30      7
script_score              6.15         5.40         3.10       15.80      7

--- Profiling function_score query ---

  Query type: FunctionScoreQuery
  Total time: 1.25 ms
  Breakdown:
    build_scorer             0.35 ms  ( 28.0%)
    create_weight            0.22 ms  ( 17.6%)
    next_doc                 0.38 ms  ( 30.4%)
    score                    0.28 ms  ( 22.4%)
```

The timing numbers are examples only. They depend on your machine, the current JVM state, cache warmth, and dataset size.

### Troubleshooting Tips

- **Profile output is too large:** Profile one index at a time. You can also use routing or a smaller query to reduce the number of shards and documents involved.
- **Script score errors:** Make sure every document has the fields used in the script. For optional fields, add a guard such as `doc['field'].size() > 0`.
- **High `score` time:** Move non-relevance clauses to `filter` context so Elasticsearch can skip scoring for those clauses.
- **High `next_doc` time:** The query may be matching too many documents. Add selective filters or narrow the search text.
- **`too_many_clauses` error:** The boolean query has too many terms. Use a `terms` query instead of many individual `term` clauses.
- **Slow aggregation queries:** Aggregate on `keyword`, numeric, boolean, or date fields. Avoid aggregating on `text` fields.
- **Unexpected zero results:** Confirm that the index contains data with `GET /products/_count`, then check field names and exact keyword values.
- **Slow first run but faster later runs:** Elasticsearch and the operating system cache data. Compare several iterations instead of relying on a single query.

### Reflection

Record which components of your query are most time-consuming and answer:

1. Which profiling phase dominates in your queries? What does that imply?
2. How much faster is `bool` with `filter` compared with using `must` for the range clause?
3. What is the performance trade-off between `function_score` and `script_score`?
4. Which query would you choose for a real product search feature, and why?
5. What changes would you make if the `score` phase dominated the profile output?
6. What changes would you make if `next_doc` dominated the profile output?
