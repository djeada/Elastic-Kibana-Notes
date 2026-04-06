## Task 4: Data Aggregations for Analytics

**Objectives:**
- Use metric, bucket, and pipeline aggregations to analyze indexed data.
- Understand the aggregation pipeline: buckets → metrics → pipelines.
- Build nested aggregations for multi-dimensional analysis.
- Visualize aggregated data via raw queries, curl, and Python.

---

### Aggregation Pipeline Architecture

```
  Search Request (size: 0 → no hits, only aggregations)
          │
          ▼
┌──────────────────────────────────────────────────────────┐
│                  AGGREGATION PIPELINE                    │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  1. BUCKET AGGREGATIONS                            │  │
│  │     Split documents into groups (buckets)           │  │
│  │                                                    │  │
│  │     terms ──► group by field value                 │  │
│  │     range ──► group by numeric/date ranges         │  │
│  │     histogram ──► fixed-interval numeric buckets   │  │
│  │     date_histogram ──► fixed-interval date buckets │  │
│  └──────────────────┬─────────────────────────────────┘  │
│                     │ each bucket                        │
│                     ▼                                    │
│  ┌────────────────────────────────────────────────────┐  │
│  │  2. METRIC AGGREGATIONS                            │  │
│  │     Compute statistics within each bucket           │  │
│  │                                                    │  │
│  │     avg ──────► average value                      │  │
│  │     sum ──────► total sum                          │  │
│  │     min / max ► minimum / maximum                  │  │
│  │     stats ────► count, min, max, avg, sum          │  │
│  │     extended_stats ──► stats + variance, std_dev   │  │
│  │     cardinality ─────► approximate distinct count  │  │
│  └──────────────────┬─────────────────────────────────┘  │
│                     │ computed values                    │
│                     ▼                                    │
│  ┌────────────────────────────────────────────────────┐  │
│  │  3. PIPELINE AGGREGATIONS                          │  │
│  │     Compute over other aggregation results          │  │
│  │                                                    │  │
│  │     avg_bucket ──► average across buckets          │  │
│  │     max_bucket ──► bucket with highest value       │  │
│  │     cumulative_sum ──► running total               │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

### Prerequisite: Seed Sample Data

Load diverse products so aggregations return meaningful results:

```json
POST /products/_bulk
{"index": {"_id": "1"}}
{"name": "Elastic T-Shirt", "price": 19.99, "category": "clothing", "in_stock": true, "created_at": "2024-01-15"}
{"index": {"_id": "2"}}
{"name": "Kibana Mug", "price": 12.50, "category": "accessories", "in_stock": true, "created_at": "2024-02-01"}
{"index": {"_id": "3"}}
{"name": "Logstash Hoodie", "price": 45.00, "category": "clothing", "in_stock": true, "created_at": "2024-03-01"}
{"index": {"_id": "4"}}
{"name": "Beats Cap", "price": 15.00, "category": "accessories", "in_stock": true, "created_at": "2024-03-05"}
{"index": {"_id": "5"}}
{"name": "APM Sticker Pack", "price": 5.99, "category": "accessories", "in_stock": false, "created_at": "2024-03-10"}
{"index": {"_id": "6"}}
{"name": "Elastic Polo Shirt", "price": 34.99, "category": "clothing", "in_stock": true, "created_at": "2024-04-01"}
{"index": {"_id": "7"}}
{"name": "Fleet Backpack", "price": 59.99, "category": "accessories", "in_stock": true, "created_at": "2024-05-15"}
{"index": {"_id": "8"}}
{"name": "Security Jacket", "price": 89.99, "category": "clothing", "in_stock": false, "created_at": "2024-06-01"}
```

---

### Lab Steps

### Step 1: Metric Aggregation — Average Price

**In Kibana Dev Tools:**

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "average_price": { "avg": { "field": "price" } }
  }
}
```

**Equivalent curl command:**
```bash
curl -s -X GET "http://localhost:9200/products/_search" \
  -H "Content-Type: application/json" \
  -d '{"size":0,"aggs":{"average_price":{"avg":{"field":"price"}}}}' \
  | python3 -m json.tool
```

**Expected Output:**
```json
{
  "took": 3,
  "hits": { "total": { "value": 8, "relation": "eq" }, "hits": [] },
  "aggregations": {
    "average_price": {
      "value": 35.43125
    }
  }
}
```

> **Note:** `"size": 0` tells Elasticsearch to skip returning document hits — only aggregation results are returned.

---

### Step 2: Stats and Extended Stats

**Stats (five-number summary):**

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_stats": { "stats": { "field": "price" } }
  }
}
```

**Expected Output:**
```json
{
  "aggregations": {
    "price_stats": {
      "count": 8,
      "min": 5.99,
      "max": 89.99,
      "avg": 35.43125,
      "sum": 283.45
    }
  }
}
```

**Extended stats (adds variance, std deviation, bounds):**

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_extended": { "extended_stats": { "field": "price" } }
  }
}
```

**Expected Output:**
```json
{
  "aggregations": {
    "price_extended": {
      "count": 8,
      "min": 5.99,
      "max": 89.99,
      "avg": 35.43125,
      "sum": 283.45,
      "sum_of_squares": 14291.1309,
      "variance": 530.878,
      "variance_population": 530.878,
      "variance_sampling": 606.718,
      "std_deviation": 23.04,
      "std_deviation_population": 23.04,
      "std_deviation_sampling": 24.63,
      "std_deviation_bounds": {
        "upper": 81.51,
        "lower": -10.65,
        "upper_population": 81.51,
        "lower_population": -10.65,
        "upper_sampling": 84.69,
        "lower_sampling": -13.83
      }
    }
  }
}
```

---

### Step 3: Cardinality (Approximate Distinct Count)

How many distinct categories exist?

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "unique_categories": {
      "cardinality": { "field": "category" }
    }
  }
}
```

**Expected Output:**
```json
{
  "aggregations": {
    "unique_categories": {
      "value": 2
    }
  }
}
```

---

### Step 4: Bucket Aggregation — Terms

Group documents by category:

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category" }
    }
  }
}
```

**Expected Output:**
```json
{
  "aggregations": {
    "by_category": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        { "key": "accessories", "doc_count": 4 },
        { "key": "clothing",    "doc_count": 4 }
      ]
    }
  }
}
```

---

### Step 5: Histogram Aggregation

Group products into price buckets with a fixed interval of $20:

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_histogram": {
      "histogram": {
        "field": "price",
        "interval": 20
      }
    }
  }
}
```

**Expected Output:**
```json
{
  "aggregations": {
    "price_histogram": {
      "buckets": [
        { "key": 0.0,  "doc_count": 3 },
        { "key": 20.0, "doc_count": 1 },
        { "key": 40.0, "doc_count": 2 },
        { "key": 60.0, "doc_count": 1 },
        { "key": 80.0, "doc_count": 1 }
      ]
    }
  }
}
```

---

### Step 6: Date Histogram Aggregation

Group documents by month:

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "products_per_month": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "month",
        "format": "yyyy-MM"
      }
    }
  }
}
```

**Expected Output:**
```json
{
  "aggregations": {
    "products_per_month": {
      "buckets": [
        { "key_as_string": "2024-01", "key": 1704067200000, "doc_count": 1 },
        { "key_as_string": "2024-02", "key": 1706745600000, "doc_count": 1 },
        { "key_as_string": "2024-03", "key": 1709251200000, "doc_count": 3 },
        { "key_as_string": "2024-04", "key": 1711929600000, "doc_count": 1 },
        { "key_as_string": "2024-05", "key": 1714521600000, "doc_count": 1 },
        { "key_as_string": "2024-06", "key": 1717200000000, "doc_count": 1 }
      ]
    }
  }
}
```

---

### Step 7: Nested Aggregations (Bucket Inside Bucket)

Get average price per category:

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category" },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } },
        "price_range": { "stats": { "field": "price" } }
      }
    }
  }
}
```

**Expected Output:**
```json
{
  "aggregations": {
    "by_category": {
      "buckets": [
        {
          "key": "accessories",
          "doc_count": 4,
          "avg_price": { "value": 23.37 },
          "price_range": { "count": 4, "min": 5.99, "max": 59.99, "avg": 23.37, "sum": 93.48 }
        },
        {
          "key": "clothing",
          "doc_count": 4,
          "avg_price": { "value": 47.49 },
          "price_range": { "count": 4, "min": 19.99, "max": 89.99, "avg": 47.49, "sum": 189.97 }
        }
      ]
    }
  }
}
```

**Multi-level nesting — products per month, with category breakdown inside each month:**

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_month": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "month",
        "format": "yyyy-MM"
      },
      "aggs": {
        "by_category": {
          "terms": { "field": "category" },
          "aggs": {
            "avg_price": { "avg": { "field": "price" } }
          }
        }
      }
    }
  }
}
```

**Expected Output (excerpt):**
```json
{
  "aggregations": {
    "by_month": {
      "buckets": [
        {
          "key_as_string": "2024-01",
          "doc_count": 1,
          "by_category": {
            "buckets": [
              { "key": "clothing", "doc_count": 1, "avg_price": { "value": 19.99 } }
            ]
          }
        },
        {
          "key_as_string": "2024-03",
          "doc_count": 3,
          "by_category": {
            "buckets": [
              { "key": "accessories", "doc_count": 2, "avg_price": { "value": 10.50 } },
              { "key": "clothing",    "doc_count": 1, "avg_price": { "value": 45.00 } }
            ]
          }
        }
      ]
    }
  }
}
```

---

### Step 8: Pipeline Aggregations

Pipeline aggregations compute values over the results of other aggregations.

**Cumulative sum of products over time:**

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "products_over_time": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "month",
        "format": "yyyy-MM"
      },
      "aggs": {
        "monthly_count": { "value_count": { "field": "price" } },
        "cumulative_total": {
          "cumulative_sum": { "buckets_path": "monthly_count" }
        }
      }
    }
  }
}
```

**Expected Output:**
```json
{
  "aggregations": {
    "products_over_time": {
      "buckets": [
        { "key_as_string": "2024-01", "doc_count": 1, "monthly_count": { "value": 1 }, "cumulative_total": { "value": 1.0 } },
        { "key_as_string": "2024-02", "doc_count": 1, "monthly_count": { "value": 1 }, "cumulative_total": { "value": 2.0 } },
        { "key_as_string": "2024-03", "doc_count": 3, "monthly_count": { "value": 3 }, "cumulative_total": { "value": 5.0 } },
        { "key_as_string": "2024-04", "doc_count": 1, "monthly_count": { "value": 1 }, "cumulative_total": { "value": 6.0 } },
        { "key_as_string": "2024-05", "doc_count": 1, "monthly_count": { "value": 1 }, "cumulative_total": { "value": 7.0 } },
        { "key_as_string": "2024-06", "doc_count": 1, "monthly_count": { "value": 1 }, "cumulative_total": { "value": 8.0 } }
      ]
    }
  }
}
```

**Find the category with the highest average price (max_bucket):**

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category" },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }
      }
    },
    "most_expensive_category": {
      "max_bucket": {
        "buckets_path": "by_category>avg_price"
      }
    }
  }
}
```

**Expected Output:**
```json
{
  "aggregations": {
    "by_category": {
      "buckets": [
        { "key": "accessories", "doc_count": 4, "avg_price": { "value": 23.37 } },
        { "key": "clothing",    "doc_count": 4, "avg_price": { "value": 47.49 } }
      ]
    },
    "most_expensive_category": {
      "value": 47.49,
      "keys": ["clothing"]
    }
  }
}
```

---

### Step 9: Python Aggregation and Visualization

```python
import requests
import json

ES_URL = "http://localhost:9200"
SEARCH_URL = f"{ES_URL}/products/_search"

# --- Nested aggregation: avg price per category ---
query = {
    "size": 0,
    "aggs": {
        "by_category": {
            "terms": {"field": "category"},
            "aggs": {
                "avg_price": {"avg": {"field": "price"}}
            }
        }
    }
}
resp = requests.get(SEARCH_URL, json=query).json()

print("=== Average Price by Category ===")
print(f"{'Category':<15} {'Count':>5} {'Avg Price':>10}")
print("-" * 32)
for bucket in resp["aggregations"]["by_category"]["buckets"]:
    name = bucket["key"]
    count = bucket["doc_count"]
    avg = bucket["avg_price"]["value"]
    print(f"{name:<15} {count:>5} {avg:>10.2f}")
print()

# --- Monthly product count with ASCII bar chart ---
query = {
    "size": 0,
    "aggs": {
        "by_month": {
            "date_histogram": {
                "field": "created_at",
                "calendar_interval": "month",
                "format": "yyyy-MM"
            }
        }
    }
}
resp = requests.get(SEARCH_URL, json=query).json()

print("=== Products Added per Month ===")
for bucket in resp["aggregations"]["by_month"]["buckets"]:
    month = bucket["key_as_string"]
    count = bucket["doc_count"]
    bar = "█" * (count * 4)
    print(f"  {month} | {bar} ({count})")
print()

# --- Price histogram ---
query = {
    "size": 0,
    "aggs": {
        "price_ranges": {
            "histogram": {"field": "price", "interval": 20}
        }
    }
}
resp = requests.get(SEARCH_URL, json=query).json()

print("=== Price Distribution ===")
for bucket in resp["aggregations"]["price_ranges"]["buckets"]:
    lo = bucket["key"]
    count = bucket["doc_count"]
    bar = "▓" * (count * 3)
    print(f"  ${lo:5.0f}-${lo+20:<5.0f} | {bar} ({count})")
```

**Expected Output:**
```
=== Average Price by Category ===
Category          Count  Avg Price
--------------------------------
accessories           4      23.37
clothing              4      47.49

=== Products Added per Month ===
  2024-01 | ████ (1)
  2024-02 | ████ (1)
  2024-03 | ████████████ (3)
  2024-04 | ████ (1)
  2024-05 | ████ (1)
  2024-06 | ████ (1)

=== Price Distribution ===
  $    0-$20    | ▓▓▓▓▓▓▓▓▓ (3)
  $   20-$40    | ▓▓▓ (1)
  $   40-$60    | ▓▓▓▓▓▓ (2)
  $   60-$80    | ▓▓▓ (1)
  $   80-$100   | ▓▓▓ (1)
```

---

### Troubleshooting Tips

| Problem | Cause | Solution |
|---|---|---|
| `"type": "illegal_argument_exception"` on text field aggregation | Cannot aggregate on analyzed `text` fields | Use `fieldname.keyword` or map the field as `keyword` |
| Empty buckets missing from histogram | Buckets with 0 docs are omitted by default | Add `"min_doc_count": 0` to the histogram config |
| `"buckets_path"` error in pipeline agg | Incorrect path reference | Use `>` to separate nesting levels: `"parent_agg>child_agg"` |
| Cardinality returns approximate values | Uses HyperLogLog++ algorithm | This is by design; set `"precision_threshold": 40000` for higher accuracy |
| Very slow aggregation | Too many unique terms or large dataset | Add a `query` clause to filter documents before aggregating |

---

### Reflection

Document what metrics you obtained and discuss:
1. How bucket aggregations split data into groups vs. how metric aggregations compute summaries.
2. When pipeline aggregations are necessary (operating on aggregation results, not raw documents).
3. How these aggregations could support real-time analytics dashboards in Kibana.
