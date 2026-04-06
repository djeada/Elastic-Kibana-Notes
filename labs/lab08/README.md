## Task 8: Advanced Querying and Query Profiling

**Objectives:**
- Build complex queries combining multiple query types (function score, boosts, filters).
- Use the `profile` flag to analyze query performance.
- Leverage Python to iterate and refine queries based on performance data.
- Understand the query profiling breakdown and each timing phase.
- Configure search slow logs to catch performance issues.
- Apply query optimization techniques.

---

### Query Profiling Breakdown

When `"profile": true` is enabled, Elasticsearch reports timing for each query phase:

```
  ┌────────────────────────────────────────────────────────┐
  │              QUERY PROFILING PHASES                     │
  │                                                        │
  │  ┌─────────────┐                                       │
  │  │  rewrite     │  Simplify/optimize the query tree    │
  │  └──────┬──────┘                                       │
  │         │                                              │
  │         ▼                                              │
  │  ┌──────────────┐                                      │
  │  │ create_weight │  Build internal data structures     │
  │  └──────┬───────┘   for scoring                        │
  │         │                                              │
  │         ▼                                              │
  │  ┌──────────────┐                                      │
  │  │ build_scorer  │  Create scorer iterators for        │
  │  └──────┬───────┘   each segment                       │
  │         │                                              │
  │         ▼                                              │
  │  ┌──────────────┐                                      │
  │  │ next_doc      │  Advance to next matching document  │
  │  └──────┬───────┘                                      │
  │         │                                              │
  │         ▼                                              │
  │  ┌──────────────┐                                      │
  │  │   score       │  Calculate relevance score for      │
  │  └──────┬───────┘   each matching document             │
  │         │                                              │
  │         ▼                                              │
  │  ┌──────────────┐                                      │
  │  │   advance     │  Skip to a target document ID       │
  │  └──────┬───────┘   (used by conjunctions)             │
  │         │                                              │
  │         ▼                                              │
  │     Results                                            │
  └────────────────────────────────────────────────────────┘
```

---

### Task 8.1: Complex Query Construction (Kibana)

#### Nested Bool Query with Multiple Clauses

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
    "fields": { "name": {}, "description": {} }
  }
}
```

**Expected output:**
```json
{
  "took": 5,
  "hits": {
    "total": { "value": 2, "relation": "eq" },
    "max_score": 3.42,
    "hits": [
      {
        "_id": "1",
        "_score": 3.42,
        "_source": {
          "name": "Elastic Widget",
          "price": 29.99,
          "category": "Electronics"
        },
        "highlight": {
          "name": ["<em>Elastic</em> Widget"]
        }
      }
    ]
  }
}
```

#### Function Score with Multiple Functions

```json
GET /products/_search
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [{ "match": { "name": "Elastic" } }],
          "filter": [{ "range": { "price": { "gte": 10, "lte": 200 } } }]
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
          "random_score": { "seed": 42, "field": "_seq_no" },
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
  }
}
```

#### Script Score Query

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
  }
}
```

---

### Task 8.2: Query Profiling — Detailed Analysis

Enable the profiler on the bool query:

```json
GET /products/_search
{
  "profile": true,
  "query": {
    "bool": {
      "must": [{ "match": { "name": "Elastic" } }],
      "filter": [{ "range": { "price": { "gte": 10, "lte": 50 } } }]
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

### Understanding Profile Timing Fields

| Field            | Description                                                  |
|------------------|--------------------------------------------------------------|
| `create_weight`  | Time to build query internals (once per query per segment)   |
| `build_scorer`   | Time to create the iterator that walks matching docs         |
| `next_doc`       | Time advancing to the next matching document                 |
| `advance`        | Time skipping to a specific document ID                      |
| `score`          | Time computing the relevance score of matches                |
| `match`          | Time for phrase/proximity queries to verify exact positions  |
| `*_count`        | How many times each operation was invoked                    |

> **Tip:** If `score` dominates, consider using `filter` context (no scoring) for non-relevance clauses. If `next_doc` dominates, the query is matching too many documents.

---

### Task 8.3: Search Slow Logs

Configure Elasticsearch to log slow queries. This helps catch performance problems in production:

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
{ "acknowledged": true }
```

Slow log entries appear in `<ES_HOME>/logs/<cluster_name>_index_search_slowlog.log`:
```
[2024-01-15T14:30:00,123][WARN][i.s.s.query] [es1] [products][0]
  took[2.3s], took_millis[2300], total_hits[15000],
  source[{"query":{"match_all":{}}}]
```

---

### Task 8.4: Query Optimization Techniques

#### 1. Use `filter` Context for Non-Scoring Clauses

Filters are cached and skip scoring — much faster:

```json
// SLOW: scoring a range unnecessarily
{ "must": [{ "range": { "price": { "gte": 10 } } }] }

// FAST: use filter context
{ "filter": [{ "range": { "price": { "gte": 10 } } }] }
```

#### 2. Avoid Wildcards at the Start of a Pattern

```json
// SLOW: leading wildcard scans all terms
{ "wildcard": { "name": "*elastic*" } }

// FAST: prefix or match query
{ "match_phrase_prefix": { "name": "elastic" } }
```

#### 3. Limit Returned Fields with `_source`

```json
{
  "query": { "match_all": {} },
  "_source": ["name", "price"],
  "size": 10
}
```

#### 4. Use `terminate_after` for Existence Checks

```json
{
  "query": { "term": { "category": "Electronics" } },
  "terminate_after": 1,
  "size": 0
}
```

#### 5. Prefer `keyword` Over `text` for Exact Matches

```json
// SLOW: analyzed text field
{ "match": { "status": "published" } }

// FAST: keyword field (no analysis)
{ "term": { "status": "published" } }
```

---

### Task 8.5: Python Query Benchmarking Script

```python
import requests
import json
import time
from statistics import mean, median

ES_URL = "http://localhost:9200"
INDEX = "products"

queries = {
    "simple_match": {
        "query": { "match": { "name": "Elastic" } }
    },
    "bool_with_filter": {
        "query": {
            "bool": {
                "must":   [{"match": {"name": "Elastic"}}],
                "filter": [{"range": {"price": {"gte": 10, "lte": 200}}}]
            }
        }
    },
    "function_score": {
        "query": {
            "function_score": {
                "query": {"match": {"name": "Elastic"}},
                "functions": [
                    {"field_value_factor": {"field": "price", "modifier": "log1p"}}
                ]
            }
        }
    },
    "script_score": {
        "query": {
            "script_score": {
                "query": {"match": {"name": "Elastic"}},
                "script": {
                    "source": "_score * Math.log(2 + doc['price'].value)"
                }
            }
        }
    }
}

ITERATIONS = 20

print(f"{'Query':<25} {'Avg (ms)':<12} {'Median (ms)':<12} {'Min (ms)':<10} {'Max (ms)':<10} {'Hits':<8}")
print("-" * 77)

for name, body in queries.items():
    times = []
    hit_count = 0

    for i in range(ITERATIONS):
        start = time.time()
        resp = requests.get(f"{ES_URL}/{INDEX}/_search", json=body).json()
        elapsed_ms = (time.time() - start) * 1000

        times.append(elapsed_ms)
        hit_count = resp.get("hits", {}).get("total", {}).get("value", 0)

    print(f"{name:<25} {mean(times):<12.2f} {median(times):<12.2f} "
          f"{min(times):<10.2f} {max(times):<10.2f} {hit_count:<8}")

# Profile the slowest query
print("\n--- Profiling function_score query ---")
profile_body = queries["function_score"].copy()
profile_body["profile"] = True

resp = requests.get(f"{ES_URL}/{INDEX}/_search", json=profile_body).json()

for shard in resp.get("profile", {}).get("shards", []):
    for search in shard.get("searches", []):
        for query in search.get("query", []):
            total_ns = query["time_in_nanos"]
            breakdown = query["breakdown"]
            print(f"\n  Query type: {query['type']}")
            print(f"  Total time: {total_ns / 1_000_000:.2f} ms")
            print(f"  Breakdown:")
            for key, val in sorted(breakdown.items()):
                if not key.endswith("_count") and val > 0:
                    pct = (val / total_ns) * 100
                    print(f"    {key:<20} {val / 1_000_000:>8.2f} ms  ({pct:>5.1f}%)")
```

**Expected output:**
```
Query                     Avg (ms)     Median (ms)  Min (ms)   Max (ms)   Hits
-----------------------------------------------------------------------------
simple_match              3.25         2.80         1.90       8.50       5
bool_with_filter          3.10         2.65         1.85       7.20       3
function_score            4.80         4.20         2.50       12.30      5
script_score              6.15         5.40         3.10       15.80      5

--- Profiling function_score query ---

  Query type: FunctionScoreQuery
  Total time: 1.25 ms
  Breakdown:
    build_scorer             0.35 ms  ( 28.0%)
    create_weight            0.22 ms  ( 17.6%)
    next_doc                 0.38 ms  ( 30.4%)
    score                    0.28 ms  ( 22.4%)
```

---

### Troubleshooting Tips

- **Profile output too large**: Profile only targets one index at a time; use `routing` to limit shards.
- **Script score errors**: Ensure the field exists in all docs; use `doc['field'].size() > 0` guard.
- **High `score` time**: Move non-relevance clauses to `filter` context; they'll be cached.
- **"too_many_clauses" error**: Boolean query has too many terms — use `terms` query instead of many `term` clauses.
- **Slow aggregation queries**: Ensure aggregated fields use `keyword` or numeric types, not `text`.

---

### Reflection

Record which components of your query are most time-consuming and answer:
1. Which profiling phase dominates in your queries? What does that imply?
2. How much faster is `bool` with `filter` vs `must` for the range clause?
3. What is the performance trade-off between `function_score` and `script_score`?

