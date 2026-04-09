## Aggregations

Aggregations are the primary analytics framework in Elasticsearch, enabling you to compute summaries, statistics, and groupings over your indexed data without extracting raw documents. Where a search query answers "which documents match?", an aggregation answers "what patterns exist across the matching documents?" This makes aggregations the backbone of dashboards, reports, and real-time analytics built on top of Elasticsearch.

Traditional relational databases serve a similar purpose with `GROUP BY` and aggregate functions like `SUM()` and `AVG()`. Elasticsearch aggregations go further by operating on distributed, denormalized data at scale -- computing results across millions of documents spread over many shards in near real-time. Because aggregations run within the same request as a search query, they can be scoped to any subset of documents defined by the Query DSL, allowing you to combine full-text search with analytic computation in a single round-trip.

Aggregations are composable: you can nest a metric aggregation inside a bucket aggregation, chain pipeline aggregations that consume the output of other aggregations, and build multi-level hierarchies that mirror the dimensional structure of your data. This composability is what makes the framework powerful enough to replace many traditional OLAP workloads.

### The Aggregation Pipeline

The following diagram illustrates how a search request with aggregations flows through
Elasticsearch, from query execution to the final aggregated response:

```
 Client Request (query + aggs)
       │
       ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │                       QUERY PHASE                                   │
 │  The query clause filters the full document set down to the         │
 │  matching documents. If no query is specified, a match_all is       │
 │  implied, and every document in the index participates.             │
 │                                                                     │
 │  Index Docs: [d1, d2, d3, d4, d5, d6, d7, d8, d9, d10]              │
 │  Query Match: [d1, d3, d4, d7, d9, d10]                             │
 └──────────────────────────────┬──────────────────────────────────────┘
                                │
                                ▼
 ┌───────────────────────────────────────────────────────────────────────┐
 │                    BUCKET FORMATION                                   │
 │  Bucket aggregations partition the matched documents into groups.     │
 │  Each bucket defines a criterion (term value, range, date interval).  │
 │                                                                       │
 │  terms agg on "status":                                               │
 │    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
 │    │ status=200   │  │ status=404   │  │ status=500   │               │
 │    │ [d1, d4, d9] │  │ [d3, d10]    │  │ [d7]         │               │
 │    │ doc_count: 3 │  │ doc_count: 2 │  │ doc_count: 1 │               │
 │    └──────┬───────┘  └──────┬───────┘  └──────┬───────┘               │
 │           │                 │                 │                       │
 └───────────┼─────────────────┼─────────────────┼───────────────────────┘
             │                 │                 │
             ▼                 ▼                 ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │                   METRIC COMPUTATION                                │
 │  Metric aggregations compute statistics within each bucket.         │
 │                                                                     │
 │  avg("response_time"):                                              │
 │    status=200 → avg: 120.5ms                                        │
 │    status=404 → avg: 45.2ms                                         │
 │    status=500 → avg: 980.0ms                                        │
 └──────────────────────────────┬──────────────────────────────────────┘
                                │
                                ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │                   PIPELINE AGGREGATIONS                             │
 │  Pipeline aggs consume output of other aggs (optional stage).       │
 │                                                                     │
 │  max_bucket("avg_response_time"):                                   │
 │    → bucket with highest avg: status=500 (980.0ms)                  │
 └──────────────────────────────┬──────────────────────────────────────┘
                                │
                                ▼
                         Client Response
                    (hits + aggregations JSON)
```

### Aggregation Types Overview

Elasticsearch organizes aggregations into four categories, each serving a distinct analytical
purpose. Understanding these categories is essential for constructing effective analytic queries.

| Category   | Purpose                                                        | Examples                                                    |
|------------|----------------------------------------------------------------|-------------------------------------------------------------|
| **Bucket** | Groups documents into buckets based on field values or ranges  | `terms`, `histogram`, `date_histogram`, `range`, `filters`  |
| **Metric** | Computes numeric statistics over a set of documents            | `avg`, `sum`, `min`, `max`, `stats`, `cardinality`          |
| **Pipeline** | Operates on the output of other aggregations, not on documents | `avg_bucket`, `derivative`, `cumulative_sum`, `max_bucket`  |
| **Matrix** | Computes statistics across multiple fields simultaneously      | `matrix_stats` (correlation, covariance, variance)          |

Bucket aggregations produce sets of documents; metric aggregations produce numeric values. The
two are designed to be composed: a bucket aggregation partitions documents, and metric
aggregations compute values within each partition. Pipeline aggregations add a second pass that
derives new values from the output of those bucket-metric combinations.

### Metric Aggregations

Metric aggregations extract numeric values from document fields and compute statistics. They do
not create buckets -- they produce a single numeric result (single-value metrics) or a JSON
object with multiple statistics (multi-value metrics). Metric aggregations can run at the top
level of a request or nested inside a bucket aggregation.

#### Single-Value Metrics: avg, sum, min, max

The simplest metric aggregations each return a single number. Consider an index of e-commerce
orders where each document has a `price` field and a `quantity` field.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "average_price": {
      "avg": { "field": "price" }
    },
    "total_revenue": {
      "sum": { "field": "price" }
    },
    "cheapest_order": {
      "min": { "field": "price" }
    },
    "most_expensive_order": {
      "max": { "field": "price" }
    }
  }
}
```

```json
{
  "took": 12,
  "timed_out": false,
  "_shards": { "total": 5, "successful": 5, "failed": 0 },
  "hits": { "total": { "value": 4520, "relation": "eq" }, "hits": [] },
  "aggregations": {
    "average_price": { "value": 75.43 },
    "total_revenue": { "value": 340943.60 },
    "cheapest_order": { "value": 2.99 },
    "most_expensive_order": { "value": 1299.99 }
  }
}
```

Because `"size": 0` is set, no hits are returned in the response -- only the aggregation results.
This is a critical optimization discussed in detail later in this document.

#### Multi-Value Metrics: stats and extended_stats

The `stats` aggregation returns `count`, `min`, `max`, `avg`, and `sum` in a single request,
avoiding the overhead of running five separate aggregations. The `extended_stats` aggregation
adds `sum_of_squares`, `variance`, `variance_population`, `variance_sampling`, `std_deviation`,
and `std_deviation_bounds` for statistical analysis.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "price_statistics": {
      "stats": { "field": "price" }
    }
  }
}
```

```json
{
  "aggregations": {
    "price_statistics": {
      "count": 4520,
      "min": 2.99,
      "max": 1299.99,
      "avg": 75.43,
      "sum": 340943.60
    }
  }
}
```

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "price_extended": {
      "extended_stats": { "field": "price" }
    }
  }
}
```

```json
{
  "aggregations": {
    "price_extended": {
      "count": 4520,
      "min": 2.99,
      "max": 1299.99,
      "avg": 75.43,
      "sum": 340943.60,
      "sum_of_squares": 3.8921547E7,
      "variance": 8230.15,
      "variance_population": 8230.15,
      "variance_sampling": 8231.97,
      "std_deviation": 90.72,
      "std_deviation_bounds": {
        "upper": 256.87,
        "lower": -105.99,
        "upper_population": 256.87,
        "lower_population": -105.99,
        "upper_sampling": 256.89,
        "lower_sampling": -106.03
      }
    }
  }
}
```

The `std_deviation_bounds` are useful for anomaly detection: values outside the upper or lower
bound (typically two standard deviations from the mean) can be flagged as outliers.

#### value_count and cardinality

The `value_count` aggregation counts the number of values extracted from a field (similar to
SQL `COUNT(field)`), while `cardinality` estimates the number of distinct values using the
**HyperLogLog++** algorithm. The cardinality aggregation is approximate -- it trades perfect
accuracy for dramatically lower memory usage, making it feasible to count distinct values across
billions of documents. The `precision_threshold` parameter controls the trade-off: higher values
use more memory but improve accuracy for low-cardinality fields.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "total_orders": {
      "value_count": { "field": "order_id" }
    },
    "unique_customers": {
      "cardinality": {
        "field": "customer_id",
        "precision_threshold": 1000
      }
    }
  }
}
```

```json
{
  "aggregations": {
    "total_orders": { "value": 4520 },
    "unique_customers": { "value": 1842 }
  }
}
```

> **Note:** The default `precision_threshold` is 3000. Below this threshold, counts are nearly
> exact. Above it, the estimate has an error rate of about 1-6%. For most analytics use cases,
> this level of precision is more than sufficient.

### Bucket Aggregations

Bucket aggregations partition documents into groups -- called **buckets** -- based on field
values, ranges, or other criteria. Each bucket effectively acts as a filter, collecting the
documents that match its criterion. The power of bucket aggregations comes from their ability to
contain sub-aggregations: once documents are grouped into buckets, you can run metric or further
bucket aggregations within each bucket to drill down into your data.

#### terms Aggregation

The `terms` aggregation creates a bucket for each unique value of a field, returning the top
values ordered by document count. It is the most commonly used bucket aggregation, analogous to
`GROUP BY field` in SQL.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "popular_categories": {
      "terms": {
        "field": "category.keyword",
        "size": 5
      }
    }
  }
}
```

```json
{
  "aggregations": {
    "popular_categories": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 1230,
      "buckets": [
        { "key": "Electronics", "doc_count": 1450 },
        { "key": "Clothing", "doc_count": 980 },
        { "key": "Books", "doc_count": 420 },
        { "key": "Home & Garden", "doc_count": 280 },
        { "key": "Sports", "doc_count": 160 }
      ]
    }
  }
}
```

The `sum_other_doc_count` field indicates how many documents fell into buckets beyond the
requested `size`. The `doc_count_error_upper_bound` reflects the maximum potential error in
counts due to the distributed nature of the computation -- each shard returns its local top
terms, and the coordinating node merges them, which can miss terms that are globally frequent
but locally infrequent on individual shards.

#### histogram and date_histogram

The `histogram` aggregation creates fixed-width numeric buckets. The `date_histogram` does the
same for date fields, accepting calendar-aware intervals like `day`, `week`, `month`, and
`quarter`, as well as fixed intervals like `30m` or `1h`.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "price_distribution": {
      "histogram": {
        "field": "price",
        "interval": 50,
        "min_doc_count": 1
      }
    }
  }
}
```

```json
{
  "aggregations": {
    "price_distribution": {
      "buckets": [
        { "key": 0.0, "doc_count": 320 },
        { "key": 50.0, "doc_count": 1890 },
        { "key": 100.0, "doc_count": 1250 },
        { "key": 150.0, "doc_count": 640 },
        { "key": 200.0, "doc_count": 280 },
        { "key": 250.0, "doc_count": 140 }
      ]
    }
  }
}
```

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "orders_over_time": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "month",
        "format": "yyyy-MM"
      }
    }
  }
}
```

```json
{
  "aggregations": {
    "orders_over_time": {
      "buckets": [
        { "key_as_string": "2024-01", "key": 1704067200000, "doc_count": 820 },
        { "key_as_string": "2024-02", "key": 1706745600000, "doc_count": 760 },
        { "key_as_string": "2024-03", "key": 1709251200000, "doc_count": 910 },
        { "key_as_string": "2024-04", "key": 1711929600000, "doc_count": 1050 },
        { "key_as_string": "2024-05", "key": 1714521600000, "doc_count": 980 }
      ]
    }
  }
}
```

> **Tip:** Use `calendar_interval` for human-meaningful periods (month, quarter, year) and
> `fixed_interval` for exact durations (30m, 6h, 7d). The two parameters are mutually exclusive.

#### range and date_range

The `range` aggregation lets you define custom buckets with explicit boundaries, unlike
`histogram` which uses uniform widths. This is useful for business-defined tiers.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "price_tiers": {
      "range": {
        "field": "price",
        "ranges": [
          { "key": "budget", "to": 25 },
          { "key": "mid-range", "from": 25, "to": 100 },
          { "key": "premium", "from": 100, "to": 500 },
          { "key": "luxury", "from": 500 }
        ]
      }
    }
  }
}
```

```json
{
  "aggregations": {
    "price_tiers": {
      "buckets": [
        { "key": "budget", "to": 25.0, "doc_count": 210 },
        { "key": "mid-range", "from": 25.0, "to": 100.0, "doc_count": 2340 },
        { "key": "premium", "from": 100.0, "to": 500.0, "doc_count": 1780 },
        { "key": "luxury", "from": 500.0, "doc_count": 190 }
      ]
    }
  }
}
```

The `date_range` aggregation works similarly but accepts date math expressions:

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "order_recency": {
      "date_range": {
        "field": "order_date",
        "format": "yyyy-MM-dd",
        "ranges": [
          { "key": "last_7_days", "from": "now-7d/d", "to": "now/d" },
          { "key": "last_30_days", "from": "now-30d/d", "to": "now-7d/d" },
          { "key": "older", "to": "now-30d/d" }
        ]
      }
    }
  }
}
```

```json
{
  "aggregations": {
    "order_recency": {
      "buckets": [
        { "key": "last_7_days", "from_as_string": "2024-05-18", "to_as_string": "2024-05-25", "doc_count": 340 },
        { "key": "last_30_days", "from_as_string": "2024-04-25", "to_as_string": "2024-05-18", "doc_count": 1120 },
        { "key": "older", "to_as_string": "2024-04-25", "doc_count": 3060 }
      ]
    }
  }
}
```

#### filter and filters Aggregations

The `filter` aggregation narrows the document set for its sub-aggregations to those matching a
single query. The `filters` aggregation extends this to multiple named filters, each producing
its own bucket.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "expensive_orders": {
      "filter": {
        "range": { "price": { "gte": 500 } }
      },
      "aggs": {
        "avg_expensive_price": {
          "avg": { "field": "price" }
        }
      }
    }
  }
}
```

```json
{
  "aggregations": {
    "expensive_orders": {
      "doc_count": 190,
      "avg_expensive_price": { "value": 742.30 }
    }
  }
}
```

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "order_status_groups": {
      "filters": {
        "filters": {
          "completed": { "term": { "status.keyword": "completed" } },
          "pending": { "term": { "status.keyword": "pending" } },
          "cancelled": { "term": { "status.keyword": "cancelled" } }
        }
      },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }
      }
    }
  }
}
```

```json
{
  "aggregations": {
    "order_status_groups": {
      "buckets": {
        "completed": { "doc_count": 3200, "avg_price": { "value": 82.15 } },
        "pending": { "doc_count": 980, "avg_price": { "value": 65.40 } },
        "cancelled": { "doc_count": 340, "avg_price": { "value": 58.90 } }
      }
    }
  }
}
```

> **Tip:** The `filters` aggregation is particularly useful when building dashboard panels that
> show metrics for several pre-defined segments side by side.

#### missing Aggregation

The `missing` aggregation creates a single bucket containing all documents that lack a value
for the specified field. This is useful for data quality monitoring -- identifying records with
incomplete information.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "orders_without_category": {
      "missing": { "field": "category.keyword" }
    }
  }
}
```

```json
{
  "aggregations": {
    "orders_without_category": {
      "doc_count": 47
    }
  }
}
```

### Nested Aggregations

The real power of the aggregation framework emerges when you combine bucket and metric
aggregations through nesting. A bucket aggregation partitions documents into groups, and metric
aggregations within each bucket compute statistics scoped to that group. You can nest multiple
levels deep, building hierarchical analytics in a single request.

The following example computes the average order price and total revenue per product category:

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": {
        "field": "category.keyword",
        "size": 5
      },
      "aggs": {
        "avg_price": {
          "avg": { "field": "price" }
        },
        "total_revenue": {
          "sum": { "field": "price" }
        },
        "price_stats": {
          "stats": { "field": "price" }
        }
      }
    }
  }
}
```

```json
{
  "aggregations": {
    "by_category": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 1230,
      "buckets": [
        {
          "key": "Electronics",
          "doc_count": 1450,
          "avg_price": { "value": 124.50 },
          "total_revenue": { "value": 180525.00 },
          "price_stats": { "count": 1450, "min": 4.99, "max": 1299.99, "avg": 124.50, "sum": 180525.00 }
        },
        {
          "key": "Clothing",
          "doc_count": 980,
          "avg_price": { "value": 45.20 },
          "total_revenue": { "value": 44296.00 },
          "price_stats": { "count": 980, "min": 9.99, "max": 299.99, "avg": 45.20, "sum": 44296.00 }
        },
        {
          "key": "Books",
          "doc_count": 420,
          "avg_price": { "value": 18.75 },
          "total_revenue": { "value": 7875.00 },
          "price_stats": { "count": 420, "min": 2.99, "max": 89.99, "avg": 18.75, "sum": 7875.00 }
        }
      ]
    }
  }
}
```

You can also nest buckets within buckets. The following example groups orders by category and
then by month within each category:

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category.keyword", "size": 3 },
      "aggs": {
        "monthly_orders": {
          "date_histogram": {
            "field": "order_date",
            "calendar_interval": "month",
            "format": "yyyy-MM"
          },
          "aggs": {
            "monthly_revenue": { "sum": { "field": "price" } }
          }
        }
      }
    }
  }
}
```

This produces a two-level hierarchy: each category bucket contains monthly sub-buckets, and each
monthly sub-bucket contains the revenue sum for that category in that month.

### Pipeline Aggregations

Pipeline aggregations differ fundamentally from bucket and metric aggregations: instead of
operating on document fields, they consume the output of other aggregations. This enables
second-pass computations such as derivatives, cumulative sums, and cross-bucket comparisons.

Pipeline aggregations reference their input using a **buckets_path** parameter that specifies the
path to the metric they consume, using `>` to traverse nested aggregation levels.

#### avg_bucket and max_bucket

The `avg_bucket` sibling aggregation computes the average of a metric across all buckets
produced by a sibling bucket aggregation. The `max_bucket` aggregation identifies the bucket
with the highest value.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "monthly_orders": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "month",
        "format": "yyyy-MM"
      },
      "aggs": {
        "monthly_revenue": { "sum": { "field": "price" } }
      }
    },
    "avg_monthly_revenue": {
      "avg_bucket": {
        "buckets_path": "monthly_orders>monthly_revenue"
      }
    },
    "best_month": {
      "max_bucket": {
        "buckets_path": "monthly_orders>monthly_revenue"
      }
    }
  }
}
```

```json
{
  "aggregations": {
    "monthly_orders": {
      "buckets": [
        { "key_as_string": "2024-01", "key": 1704067200000, "doc_count": 820, "monthly_revenue": { "value": 61234.50 } },
        { "key_as_string": "2024-02", "key": 1706745600000, "doc_count": 760, "monthly_revenue": { "value": 54120.80 } },
        { "key_as_string": "2024-03", "key": 1709251200000, "doc_count": 910, "monthly_revenue": { "value": 72890.30 } },
        { "key_as_string": "2024-04", "key": 1711929600000, "doc_count": 1050, "monthly_revenue": { "value": 85400.00 } },
        { "key_as_string": "2024-05", "key": 1714521600000, "doc_count": 980, "monthly_revenue": { "value": 67298.00 } }
      ]
    },
    "avg_monthly_revenue": { "value": 68188.72 },
    "best_month": {
      "value": 85400.00,
      "keys": ["2024-04"]
    }
  }
}
```

#### derivative

The `derivative` pipeline aggregation computes the difference between consecutive bucket values,
useful for detecting trends and rates of change. It must be nested inside the parent histogram
or date_histogram bucket aggregation.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "monthly_orders": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "month",
        "format": "yyyy-MM"
      },
      "aggs": {
        "monthly_revenue": { "sum": { "field": "price" } },
        "revenue_change": {
          "derivative": {
            "buckets_path": "monthly_revenue"
          }
        }
      }
    }
  }
}
```

```json
{
  "aggregations": {
    "monthly_orders": {
      "buckets": [
        { "key_as_string": "2024-01", "doc_count": 820, "monthly_revenue": { "value": 61234.50 } },
        { "key_as_string": "2024-02", "doc_count": 760, "monthly_revenue": { "value": 54120.80 }, "revenue_change": { "value": -7113.70 } },
        { "key_as_string": "2024-03", "doc_count": 910, "monthly_revenue": { "value": 72890.30 }, "revenue_change": { "value": 18769.50 } },
        { "key_as_string": "2024-04", "doc_count": 1050, "monthly_revenue": { "value": 85400.00 }, "revenue_change": { "value": 12509.70 } },
        { "key_as_string": "2024-05", "doc_count": 980, "monthly_revenue": { "value": 67298.00 }, "revenue_change": { "value": -18102.00 } }
      ]
    }
  }
}
```

The first bucket has no derivative because there is no preceding bucket to compare against.
A negative derivative indicates a decline; a positive value indicates growth.

#### cumulative_sum

The `cumulative_sum` aggregation computes a running total across the buckets of a parent
histogram, making it easy to produce year-to-date revenue or cumulative order count charts.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "monthly_orders": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "month",
        "format": "yyyy-MM"
      },
      "aggs": {
        "monthly_revenue": { "sum": { "field": "price" } },
        "cumulative_revenue": {
          "cumulative_sum": {
            "buckets_path": "monthly_revenue"
          }
        }
      }
    }
  }
}
```

```json
{
  "aggregations": {
    "monthly_orders": {
      "buckets": [
        { "key_as_string": "2024-01", "doc_count": 820, "monthly_revenue": { "value": 61234.50 }, "cumulative_revenue": { "value": 61234.50 } },
        { "key_as_string": "2024-02", "doc_count": 760, "monthly_revenue": { "value": 54120.80 }, "cumulative_revenue": { "value": 115355.30 } },
        { "key_as_string": "2024-03", "doc_count": 910, "monthly_revenue": { "value": 72890.30 }, "cumulative_revenue": { "value": 188245.60 } },
        { "key_as_string": "2024-04", "doc_count": 1050, "monthly_revenue": { "value": 85400.00 }, "cumulative_revenue": { "value": 273645.60 } },
        { "key_as_string": "2024-05", "doc_count": 980, "monthly_revenue": { "value": 67298.00 }, "cumulative_revenue": { "value": 340943.60 } }
      ]
    }
  }
}
```

### Significant Terms Aggregation

The `significant_terms` aggregation identifies terms that appear in the foreground set (matching
documents) with unusual frequency compared to the background set (all documents in the index).
Unlike the `terms` aggregation which simply returns the most frequent terms, `significant_terms`
surfaces terms that are **statistically interesting** -- they distinguish the foreground from the
background.

This is particularly valuable for root-cause analysis ("what terms are disproportionately
associated with errors?"), content recommendation ("users who bought X also frequently bought
Y"), and discovering hidden patterns in data.

```json
GET /logs/_search
{
  "size": 0,
  "query": {
    "term": { "level.keyword": "ERROR" }
  },
  "aggs": {
    "significant_error_terms": {
      "significant_terms": {
        "field": "message.keyword",
        "size": 5
      }
    }
  }
}
```

```json
{
  "aggregations": {
    "significant_error_terms": {
      "doc_count": 3420,
      "bg_count": 850000,
      "buckets": [
        { "key": "OutOfMemoryError", "doc_count": 890, "score": 0.452, "bg_count": 920 },
        { "key": "ConnectionTimeout", "doc_count": 720, "score": 0.381, "bg_count": 1100 },
        { "key": "DiskFull", "doc_count": 410, "score": 0.298, "bg_count": 430 },
        { "key": "NullPointerException", "doc_count": 650, "score": 0.187, "bg_count": 4200 },
        { "key": "StackOverflow", "doc_count": 280, "score": 0.142, "bg_count": 980 }
      ]
    }
  }
}
```

The `score` reflects statistical significance -- higher scores indicate terms that are more
disproportionately concentrated in the foreground. In the example above, "OutOfMemoryError"
appears in 890 of 3420 error documents (26%) but only 920 of 850000 total documents (0.1%),
making it highly significant. "NullPointerException" is more frequent in error documents but
also common overall, so its significance score is lower.

### Aggregations with Query Context

Aggregations and queries work together in the same request. The query filters documents first,
and aggregations run only on the matching documents. This is the standard behavior and the most
common pattern.

```json
GET /orders/_search
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        { "term": { "category.keyword": "Electronics" } },
        { "range": { "order_date": { "gte": "2024-01-01", "lt": "2024-06-01" } } }
      ]
    }
  },
  "aggs": {
    "monthly_electronics": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "month",
        "format": "yyyy-MM"
      },
      "aggs": {
        "revenue": { "sum": { "field": "price" } }
      }
    }
  }
}
```

```json
{
  "hits": { "total": { "value": 1450, "relation": "eq" }, "hits": [] },
  "aggregations": {
    "monthly_electronics": {
      "buckets": [
        { "key_as_string": "2024-01", "doc_count": 280, "revenue": { "value": 34200.00 } },
        { "key_as_string": "2024-02", "doc_count": 260, "revenue": { "value": 31800.00 } },
        { "key_as_string": "2024-03", "doc_count": 310, "revenue": { "value": 39500.00 } },
        { "key_as_string": "2024-04", "doc_count": 340, "revenue": { "value": 42100.00 } },
        { "key_as_string": "2024-05", "doc_count": 260, "revenue": { "value": 32925.00 } }
      ]
    }
  }
}
```

#### post_filter vs filter

A common confusion arises between `post_filter` and the `filter` aggregation. They serve
different purposes:

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │                      REQUEST STRUCTURE                               │
 │                                                                     │
 │  "query"       →  Filters documents for BOTH hits AND aggregations  │
 │  "aggs"        →  Runs on documents matched by "query"              │
 │  "post_filter" →  Filters hits ONLY, does NOT affect aggregations   │
 └─────────────────────────────────────────────────────────────────────┘
```

The `post_filter` is applied after aggregations are computed but before hits are returned. This
is essential for **faceted navigation**: you want the aggregation counts to reflect the broad
query so users can see all available filter options, but you want the hit list to reflect the
user's currently selected filters.

```json
GET /orders/_search
{
  "query": {
    "term": { "category.keyword": "Electronics" }
  },
  "aggs": {
    "all_brands": {
      "terms": { "field": "brand.keyword", "size": 10 }
    }
  },
  "post_filter": {
    "term": { "brand.keyword": "Samsung" }
  }
}
```

In this example, the `all_brands` aggregation counts documents across all electronics brands
(so the user sees all brand options in the sidebar), but the returned hits are restricted to
Samsung products only. Without `post_filter`, filtering by Samsung in the query would cause the
brand aggregation to show only Samsung, hiding other options from the user.

### The size: 0 Optimization

When your request only needs aggregation results and not individual document hits, setting
`"size": 0` is a critical performance optimization. It tells Elasticsearch to skip the fetch
phase entirely -- no `_source` fields are loaded, no highlights are computed, and no documents
are serialized into the response.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "total_revenue": { "sum": { "field": "price" } }
  }
}
```

The performance impact is significant:

```
 ┌────────────────────────────────────────────────────────────────┐
 │                   WITH size: 0                                 │
 │                                                                │
 │  1. Query Phase   →  Match documents, build aggregation data   │
 │  2. (Fetch Phase  →  SKIPPED -- no document retrieval)         │
 │  3. Response      →  Aggregation results only                  │
 │                                                                │
 │  Benefits:                                                     │
 │  - No _source field deserialization                            │
 │  - No network transfer of document bodies                     │
 │  - Reduced heap pressure on coordinating node                  │
 │  - Response payload is dramatically smaller                    │
 └────────────────────────────────────────────────────────────────┘

 ┌────────────────────────────────────────────────────────────────┐
 │                   WITH default size: 10                        │
 │                                                                │
 │  1. Query Phase   →  Match documents, build aggregation data   │
 │  2. Fetch Phase   →  Load _source for top 10 documents        │
 │  3. Response      →  Aggregation results + 10 hit documents    │
 │                                                                │
 │  Overhead:                                                     │
 │  - Additional shard round-trip for fetch phase                │
 │  - _source deserialization and network transfer               │
 │  - Larger response payload                                    │
 └────────────────────────────────────────────────────────────────┘
```

> **Tip:** Always set `"size": 0` in aggregation-only requests. This is one of the simplest and
> most impactful performance improvements for analytics workloads.

### Performance Considerations

Aggregations can be resource-intensive, especially on high-cardinality fields or deeply nested
bucket hierarchies. Understanding the tuning parameters and their trade-offs is essential for
running aggregations at scale.

#### Memory Usage and the collect_mode Parameter

By default, Elasticsearch uses **depth-first** collection for nested aggregations: it builds
the entire sub-aggregation tree for each bucket before moving to the next. For deeply nested
hierarchies with many top-level buckets, this can consume substantial heap memory.

Setting `"collect_mode": "breadth_first"` changes the strategy to process all buckets at one
level before descending. After the top-level buckets are collected and pruned to the requested
`size`, only the surviving buckets are expanded with sub-aggregations. This can dramatically
reduce memory usage when the top-level aggregation has high cardinality but you only want a
small number of results.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": {
        "field": "category.keyword",
        "size": 5,
        "collect_mode": "breadth_first"
      },
      "aggs": {
        "by_brand": {
          "terms": { "field": "brand.keyword", "size": 5 },
          "aggs": {
            "avg_price": { "avg": { "field": "price" } }
          }
        }
      }
    }
  }
}
```

#### shard_size Parameter

The `shard_size` parameter controls how many candidate buckets each shard returns to the
coordinating node during a `terms` aggregation. The coordinating node then merges these
candidates into the final top-`size` result. A higher `shard_size` improves accuracy at the cost
of increased network and memory overhead.

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "popular_products": {
      "terms": {
        "field": "product_id.keyword",
        "size": 10,
        "shard_size": 50
      }
    }
  }
}
```

The default `shard_size` is `size * 1.5 + 10`. Increasing it is recommended when:

- The field has very high cardinality (millions of unique values).
- You need precise counts and cannot tolerate the approximation error.
- `doc_count_error_upper_bound` in the response is unacceptably large.

#### execution_hint Parameter

The `execution_hint` tells Elasticsearch which internal data structure to use for building
term buckets. The two options are:

| Hint          | Strategy                                           | Best For                            |
|---------------|----------------------------------------------------|-------------------------------------|
| `map`         | Builds an in-memory map per shard                  | Low-cardinality fields (< 1000)     |
| `global_ordinals` | Uses the global ordinals data structure        | High-cardinality fields (default)   |

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_status": {
      "terms": {
        "field": "status.keyword",
        "execution_hint": "map"
      }
    }
  }
}
```

For fields with very few unique values (e.g., status codes, boolean flags), `map` avoids the
overhead of building global ordinals. For high-cardinality fields, the default
`global_ordinals` is almost always faster because it uses a compact integer mapping rather than
repeated string comparisons.

#### General Performance Guidelines

- **Avoid deeply nested aggregation trees.** Each nesting level multiplies the number of
  buckets. A terms aggregation with `size: 100` nested three levels deep can produce up to
  one million buckets.
- **Use `filter` aggregations instead of top-level queries** when you need different subsets
  for different aggregations in the same request.
- **Prefer `keyword` fields for terms aggregations.** Running terms aggregations on analyzed
  `text` fields produces unexpected results based on individual tokens rather than full values.
- **Monitor with the `_nodes/stats` API** to track aggregation memory pressure, especially the
  `request_breaker` circuit breaker that protects against out-of-memory conditions.
- **Use `min_doc_count`** to eliminate empty buckets in histogram aggregations, reducing response
  size and processing overhead.

---

### Command Reference

| Aggregation Type | REST Verb & Endpoint | Key Body Parameters |
|------------------|----------------------|---------------------|
| Average metric | `GET /<index>/_search` | `aggs.<name>.avg.field` |
| Sum metric | `GET /<index>/_search` | `aggs.<name>.sum.field` |
| Min metric | `GET /<index>/_search` | `aggs.<name>.min.field` |
| Max metric | `GET /<index>/_search` | `aggs.<name>.max.field` |
| Stats metric | `GET /<index>/_search` | `aggs.<name>.stats.field` |
| Extended stats metric | `GET /<index>/_search` | `aggs.<name>.extended_stats.field` |
| Value count metric | `GET /<index>/_search` | `aggs.<name>.value_count.field` |
| Cardinality metric | `GET /<index>/_search` | `aggs.<name>.cardinality.field`, `precision_threshold` |
| Terms bucket | `GET /<index>/_search` | `aggs.<name>.terms.field`, `size`, `shard_size` |
| Histogram bucket | `GET /<index>/_search` | `aggs.<name>.histogram.field`, `interval` |
| Date histogram bucket | `GET /<index>/_search` | `aggs.<name>.date_histogram.field`, `calendar_interval` / `fixed_interval` |
| Range bucket | `GET /<index>/_search` | `aggs.<name>.range.field`, `ranges[{from,to}]` |
| Date range bucket | `GET /<index>/_search` | `aggs.<name>.date_range.field`, `ranges`, `format` |
| Filter bucket | `GET /<index>/_search` | `aggs.<name>.filter.{query}` |
| Filters bucket | `GET /<index>/_search` | `aggs.<name>.filters.filters.{name: query}` |
| Missing bucket | `GET /<index>/_search` | `aggs.<name>.missing.field` |
| Significant terms | `GET /<index>/_search` | `aggs.<name>.significant_terms.field`, `size` |
| Avg bucket pipeline | `GET /<index>/_search` | `aggs.<name>.avg_bucket.buckets_path` |
| Max bucket pipeline | `GET /<index>/_search` | `aggs.<name>.max_bucket.buckets_path` |
| Derivative pipeline | `GET /<index>/_search` | `aggs.<name>.derivative.buckets_path` |
| Cumulative sum pipeline | `GET /<index>/_search` | `aggs.<name>.cumulative_sum.buckets_path` |
| Nested sub-aggs | `GET /<index>/_search` | `aggs.<name>.{bucket}.aggs.<sub_name>.{metric}` |
| Query + aggregation | `GET /<index>/_search` | `query.{dsl}`, `aggs.<name>.{type}` |
| Aggregation-only (no hits) | `GET /<index>/_search` | `size: 0`, `aggs.<name>.{type}` |
| Post-filter (faceted nav) | `GET /<index>/_search` | `post_filter.{query}`, `aggs.<name>.{type}` |
