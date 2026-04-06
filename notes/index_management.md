## Index Management

Index management in Elasticsearch encompasses the complete lifecycle of an index, from initial
creation and schema definition through active data ingestion, optimization for query performance,
archival of aging data, and eventual deletion. A well-managed index strategy is critical for
maintaining cluster health, controlling storage costs, and ensuring that search and analytics
workloads remain responsive as data volumes grow.

Unlike a traditional relational database where tables are relatively static structures,
Elasticsearch indices are dynamic resources that require ongoing attention. Settings like shard
count, replica configuration, and refresh intervals must be tuned for the specific workload.
Mappings must be designed to balance flexibility with type safety. As indices age, they transition
through operational phases -- from hot indices receiving active writes to cold indices serving
infrequent reads -- and tooling like Index Lifecycle Management (ILM) automates these transitions.

This document covers every major aspect of index management, beginning with creation and mapping,
progressing through templates and aliases, and culminating in lifecycle automation, rollover
strategies, and maintenance operations like shrink, split, and force merge.


### Index Lifecycle Overview

The following diagram illustrates the typical lifecycle of an Elasticsearch index, from initial
creation through eventual deletion:

```
  Index Lifecycle
  ---------------

  +----------+     +-----------+     +------------+     +----------+     +-----------+     +----------+
  |          |     |           |     |            |     |          |     |           |     |          |
  |  Create  +---->+ Configure +---->+ Index Data +---->+ Optimize +---->+  Archive  +---->+  Delete  |
  |          |     |           |     |            |     |          |     |           |     |          |
  +----------+     +-----------+     +------------+     +----------+     +-----------+     +----------+
       |                |                  |                  |                |                |
       |                |                  |                  |                |                |
   Define shards   Set mappings,     Bulk/single        Force merge,     Move to cold/     Remove index
   and replicas    analyzers,        document            shrink shards,   frozen tier,      when retention
                   aliases           indexing             adjust replicas  close index       period expires
```

Each phase involves specific APIs and operational considerations. In production environments,
these phases are typically automated using Index Lifecycle Management (ILM) policies, but
understanding each phase independently is essential for effective troubleshooting and custom
workflows.

---

### Creating an Index

An index is created with the `PUT /<index_name>` API. At creation time, you specify settings
that control the physical layout of the index and mappings that define the schema. The most
important settings are:

- **`number_of_shards`** -- the number of primary shards. This is fixed at creation time and
  cannot be changed without reindexing (though the shrink and split APIs offer workarounds).
- **`number_of_replicas`** -- the number of replica copies per primary shard. This can be
  changed dynamically at any time.
- **`refresh_interval`** -- how often the index is refreshed to make new documents searchable.
  The default is `1s`. Setting this to `30s` or `-1` (disable) during bulk ingestion can
  dramatically improve indexing throughput.

```json
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "5s"
  },
  "mappings": {
    "properties": {
      "name":        { "type": "text", "analyzer": "standard" },
      "sku":         { "type": "keyword" },
      "price":       { "type": "double" },
      "in_stock":    { "type": "boolean" },
      "category":    { "type": "keyword" },
      "description": { "type": "text" },
      "created_at":  { "type": "date", "format": "strict_date_optional_time||epoch_millis" },
      "location":    { "type": "geo_point" }
    }
  }
}
```

```json
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "products"
}
```

The `acknowledged` field confirms that the cluster state has been updated. The
`shards_acknowledged` field indicates that the required number of shard copies were started
before the timeout. If `shards_acknowledged` is false, the index was created but not all
shards are active yet -- this typically resolves itself as the cluster allocates shards.

---

### Dynamic vs Explicit Mappings

Elasticsearch supports two approaches to defining the schema of an index: dynamic mapping, where
field types are automatically inferred from the first document indexed, and explicit mapping,
where the developer defines every field type in advance.

**Dynamic mapping** is convenient for prototyping and exploratory work. When Elasticsearch
encounters a new field, it examines the JSON value and assigns a type based on heuristics. A
JSON string becomes a `text` field with a `keyword` sub-field, a JSON number becomes either
`long` or `float`, a JSON boolean becomes `boolean`, and a string that matches a date pattern
becomes `date`.

**Explicit mapping** is the recommended approach for production indices. It avoids surprises,
reduces storage waste, and ensures consistent query behavior. For instance, a product SKU like
`"12345"` would be dynamically mapped as both `text` (analyzed, searchable by tokens) and
`keyword` (exact match). If you only need exact-match lookups, explicitly mapping it as
`keyword` saves the overhead of maintaining the analyzed inverted index.

```
  Dynamic vs Explicit Mapping Comparison
  ----------------------------------------

  +-------------------+-------------------------------+----------------------------------+
  | Aspect            | Dynamic Mapping               | Explicit Mapping                 |
  +-------------------+-------------------------------+----------------------------------+
  | Definition        | ES auto-detects field types   | Developer defines all fields     |
  |                   | from first indexed document    | before indexing begins            |
  +-------------------+-------------------------------+----------------------------------+
  | Convenience       | High -- zero configuration     | Low -- requires upfront design    |
  +-------------------+-------------------------------+----------------------------------+
  | Type accuracy     | Often suboptimal              | Precise -- developer controls    |
  |                   | (e.g., "42" -> text+keyword)  | every field type                  |
  +-------------------+-------------------------------+----------------------------------+
  | Storage           | May create unnecessary         | Optimized -- only the fields     |
  | efficiency        | multi-fields and doc_values   | and structures you need           |
  +-------------------+-------------------------------+----------------------------------+
  | Production use    | Not recommended               | Strongly recommended              |
  +-------------------+-------------------------------+----------------------------------+
  | Schema evolution  | New fields added silently     | New fields rejected unless       |
  |                   |                               | dynamic is set to "strict"        |
  +-------------------+-------------------------------+----------------------------------+
```

**Common gotchas with dynamic mapping:**

1. **Strings become text + keyword.** Every string field gets both a `text` mapping (for
   full-text search) and a `.keyword` sub-field (for exact match, aggregations, and sorting).
   This doubles the indexing work and storage for fields that only need one type.

2. **Numeric strings become text.** A field like `"order_id": "98765"` is mapped as `text`,
   not `long`. If you later need to do range queries on this field, you must reindex.

3. **Mapping conflicts across indices.** If two indices have the same field name with
   different types (e.g., `status` as `keyword` in one and `integer` in another), cross-index
   searches on that field will fail.

You can control dynamic mapping behavior with the `dynamic` parameter:

- `"dynamic": true` -- new fields are added to the mapping automatically (default).
- `"dynamic": false` -- new fields are accepted in `_source` but not indexed or searchable.
- `"dynamic": "strict"` -- documents with unknown fields are rejected outright.

---

### Mapping Types

Elasticsearch provides a rich set of field types to accommodate diverse data models. Each type
controls how data is indexed, stored, and queried.

```
  Field Types Reference
  ---------------------

  +---------------+-------------------------------------------------------------------+
  | Type          | Description and Use Case                                          |
  +---------------+-------------------------------------------------------------------+
  | text          | Analyzed string for full-text search. Passed through an analyzer  |
  |               | pipeline to produce tokens in the inverted index.                 |
  +---------------+-------------------------------------------------------------------+
  | keyword       | Non-analyzed string for exact match, sorting, and aggregations.   |
  |               | Stored as a single token. Ideal for IDs, tags, enum values.       |
  +---------------+-------------------------------------------------------------------+
  | integer       | 32-bit signed integer (-2^31 to 2^31-1).                          |
  +---------------+-------------------------------------------------------------------+
  | long          | 64-bit signed integer (-2^63 to 2^63-1). Default for JSON ints.   |
  +---------------+-------------------------------------------------------------------+
  | float         | 32-bit IEEE 754 floating point. Use for approximate decimals.     |
  +---------------+-------------------------------------------------------------------+
  | double        | 64-bit IEEE 754 floating point. Default for JSON floats.          |
  +---------------+-------------------------------------------------------------------+
  | boolean       | true or false. Also accepts "true"/"false" strings and 0/1.       |
  +---------------+-------------------------------------------------------------------+
  | date          | Date/time stored as epoch millis internally. Supports multiple    |
  |               | formats via the "format" parameter.                               |
  +---------------+-------------------------------------------------------------------+
  | object        | A JSON object. Inner fields are flattened into dot-notation keys  |
  |               | (e.g., address.city). Cannot maintain array-of-objects relations. |
  +---------------+-------------------------------------------------------------------+
  | nested        | Like object, but maintains the independence of each object in an  |
  |               | array. Required when array elements must be queried independently.|
  +---------------+-------------------------------------------------------------------+
  | geo_point     | Latitude/longitude pair. Supports geo-distance queries, bounding  |
  |               | box filters, and geo aggregations.                                |
  +---------------+-------------------------------------------------------------------+
  | ip            | IPv4 or IPv6 address. Supports CIDR range queries.                |
  +---------------+-------------------------------------------------------------------+
  | completion    | Special type for search-as-you-type autocomplete suggestions.     |
  |               | Uses an in-memory FST data structure for speed.                   |
  +---------------+-------------------------------------------------------------------+
```

**Choosing between object and nested:** Use `object` when you never need to query individual
elements of an array of objects independently. Use `nested` when each element has distinct
meaning -- for example, an array of `{ "name": "Alice", "role": "admin" }` objects where you
need to query "name is Alice AND role is admin" without false positives from cross-object
matches.

---

### Index Templates

Index templates allow you to predefine settings and mappings that are automatically applied to
new indices whose names match a specified pattern. This is essential for time-series data where
indices are created on a recurring schedule (e.g., `logs-2024.01.15`, `logs-2024.01.16`).
Without templates, you would need to manually configure each new index.

A template specifies an `index_patterns` list (glob-style patterns), a `priority` (higher
values win when multiple templates match), and the `settings` and `mappings` to apply.

```json
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "priority": 100,
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "refresh_interval": "10s"
    },
    "mappings": {
      "properties": {
        "timestamp":   { "type": "date" },
        "level":       { "type": "keyword" },
        "message":     { "type": "text" },
        "service":     { "type": "keyword" },
        "host":        { "type": "keyword" },
        "trace_id":    { "type": "keyword" },
        "duration_ms": { "type": "long" }
      }
    }
  }
}
```

```json
{
  "acknowledged": true
}
```

Now, any index created with a name matching `logs-*` will automatically inherit these settings
and mappings. For example, creating an index named `logs-2024.07.15` will apply this template
without any additional configuration.

---

### Component Templates

Introduced in Elasticsearch 7.8, component templates are reusable building blocks that can be
composed into index templates. Instead of duplicating common settings and mappings across many
index templates, you define them once as component templates and reference them by name.

This follows a principle of composition over duplication: a logging component template might
define timestamp and level fields, while a separate component defines shard and replica settings.
Different index templates can then mix and match these components.

```json
PUT /_component_template/base_settings
{
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "refresh_interval": "10s"
    }
  }
}
```

```json
{
  "acknowledged": true
}
```

```json
PUT /_component_template/logging_mappings
{
  "template": {
    "mappings": {
      "properties": {
        "timestamp":  { "type": "date" },
        "level":      { "type": "keyword" },
        "message":    { "type": "text" },
        "service":    { "type": "keyword" },
        "host":       { "type": "keyword" }
      }
    }
  }
}
```

```json
{
  "acknowledged": true
}
```

Now compose them into an index template:

```json
PUT /_index_template/logs_composed
{
  "index_patterns": ["logs-*"],
  "priority": 200,
  "composed_of": ["base_settings", "logging_mappings"],
  "template": {
    "mappings": {
      "properties": {
        "trace_id": { "type": "keyword" }
      }
    }
  }
}
```

```json
{
  "acknowledged": true
}
```

The final index template merges settings from both component templates and adds the `trace_id`
field from its own template body. When components and the index template define overlapping
keys, the index template's own values take precedence, followed by components listed later in
the `composed_of` array.

---

### Index Aliases

An alias is a secondary name that points to one or more indices. Aliases are a powerful
abstraction layer that decouples application code from the physical index name. This enables
several critical operational patterns:

- **Zero-downtime reindexing.** Build a new index behind the scenes, then atomically swap the
  alias to point to the new index. The application never experiences an outage.
- **Simplified querying across multiple indices.** An alias like `logs-current` can point to the
  last 7 days of daily log indices, providing a single query target.
- **Filtered aliases.** An alias can include a filter, so that queries against the alias are
  automatically restricted to a subset of documents.

**Creating an alias:**

```json
POST /_aliases
{
  "actions": [
    { "add": { "index": "products_v1", "alias": "products" } }
  ]
}
```

```json
{
  "acknowledged": true
}
```

**Querying through an alias** works identically to querying the index directly:

```json
GET /products/_search
{
  "query": {
    "match": { "name": "wireless keyboard" }
  }
}
```

**Swapping an alias** atomically from one index to another (zero-downtime reindexing):

```json
POST /_aliases
{
  "actions": [
    { "remove": { "index": "products_v1", "alias": "products" } },
    { "add":    { "index": "products_v2", "alias": "products" } }
  ]
}
```

```json
{
  "acknowledged": true
}
```

Because both actions execute in a single atomic operation, there is no window during which the
alias is undefined. Applications querying the `products` alias see the switch happen
instantaneously.

```
  Alias Swap (Zero-Downtime Reindexing)
  --------------------------------------

  Before:                              After:
  +---------------+                    +---------------+
  |  products_v1  | <-- "products"     |  products_v1  |   (no alias)
  +---------------+                    +---------------+
  +---------------+                    +---------------+
  |  products_v2  |   (no alias)       |  products_v2  | <-- "products"
  +---------------+                    +---------------+

  Application queries "products" alias and is unaware of the switch.
```

**Creating a filtered alias:**

```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "orders",
        "alias": "recent_orders",
        "filter": {
          "range": {
            "created_at": { "gte": "now-30d/d" }
          }
        }
      }
    }
  ]
}
```

```json
{
  "acknowledged": true
}
```

---

### Reindexing

When an index's mapping needs to change in ways that are incompatible with the existing schema
(e.g., changing a field from `text` to `keyword`, or splitting a single field into multiple
fields), the only option is to create a new index with the correct mapping and copy the data
from the old index. The `_reindex` API handles this server-side, avoiding the overhead of
pulling all documents to the client and re-indexing them.

```json
POST /_reindex
{
  "source": {
    "index": "products_v1"
  },
  "dest": {
    "index": "products_v2"
  }
}
```

```json
{
  "took": 4523,
  "timed_out": false,
  "total": 15000,
  "updated": 0,
  "created": 15000,
  "deleted": 0,
  "batches": 15,
  "version_conflicts": 0,
  "noops": 0,
  "retries": {
    "bulk": 0,
    "search": 0
  },
  "throttled_millis": 0,
  "requests_per_second": -1.0,
  "throttled_until_millis": 0,
  "failures": []
}
```

The response shows that 15,000 documents were created in the destination index. The `batches`
field indicates the data was transferred in 15 scroll/bulk batches (default batch size is 1000).

**Reindexing with a query** allows you to copy only a subset of documents:

```json
POST /_reindex
{
  "source": {
    "index": "products_v1",
    "query": {
      "term": { "category": "electronics" }
    }
  },
  "dest": {
    "index": "electronics_products"
  }
}
```

**Reindexing with throttling** prevents the operation from saturating cluster resources:

```json
POST /_reindex?requests_per_second=500
{
  "source": { "index": "products_v1" },
  "dest":   { "index": "products_v2" }
}
```

After reindexing, the standard workflow is to swap the alias (as described above) and then
delete the old index.

---

### Index Settings Updates

Elasticsearch index settings are divided into two categories: **static settings** that can only
be set at index creation time, and **dynamic settings** that can be changed on a live index.

```
  Static vs Dynamic Settings
  --------------------------

  +------------------------+-----------+----------------------------------------------+
  | Setting                | Type      | Notes                                        |
  +------------------------+-----------+----------------------------------------------+
  | number_of_shards       | Static    | Fixed at creation. Use shrink/split to change |
  +------------------------+-----------+----------------------------------------------+
  | codec                  | Static    | Compression algorithm (default, best_compress)|
  +------------------------+-----------+----------------------------------------------+
  | number_of_replicas     | Dynamic   | Can increase/decrease at any time             |
  +------------------------+-----------+----------------------------------------------+
  | refresh_interval       | Dynamic   | Controls how often data becomes searchable    |
  +------------------------+-----------+----------------------------------------------+
  | max_result_window      | Dynamic   | Max from + size for search (default: 10000)   |
  +------------------------+-----------+----------------------------------------------+
  | routing.allocation.*   | Dynamic   | Controls shard allocation to specific nodes   |
  +------------------------+-----------+----------------------------------------------+
```

**Changing replicas at runtime:**

```json
PUT /products/_settings
{
  "index": {
    "number_of_replicas": 2
  }
}
```

```json
{
  "acknowledged": true
}
```

**Disabling refresh during bulk ingestion** to maximize indexing throughput:

```json
PUT /products/_settings
{
  "index": {
    "refresh_interval": "-1"
  }
}
```

After bulk ingestion completes, restore the refresh interval and manually trigger a refresh:

```json
PUT /products/_settings
{
  "index": {
    "refresh_interval": "1s"
  }
}
```

```json
POST /products/_refresh
```

---

### Index Lifecycle Management (ILM)

Index Lifecycle Management automates the transitions an index goes through as it ages. ILM
divides an index's life into four phases, each with configurable actions:

```
  ILM Phases
  ----------

  +--------+       +---------+       +--------+       +----------+
  |        |       |         |       |        |       |          |
  |  Hot   +------>+  Warm   +------>+  Cold  +------>+  Delete  |
  |        |       |         |       |        |       |          |
  +---+----+       +----+----+       +---+----+       +-----+----+
      |                 |                |                   |
      |                 |                |                   |
  Active writes,    No new writes,   Infrequent reads,   Index is
  high query rate   still queried    reduced replicas,   permanently
  Rollover when     frequently.      possibly frozen.    removed after
  size/age/count    Force merge,     Move to cheaper     retention
  threshold met.    shrink shards.   hardware tier.      period expires.

  Timeline:  0-7 days        7-30 days        30-90 days        90+ days
             (example)       (example)        (example)         (example)
```

**Creating an ILM policy:**

```json
PUT /_ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "7d",
            "max_docs": 100000000
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0
          },
          "set_priority": {
            "priority": 0
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

```json
{
  "acknowledged": true
}
```

**Attaching an ILM policy to an index template:**

```json
PUT /_index_template/logs_ilm_template
{
  "index_patterns": ["logs-*"],
  "priority": 200,
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs_policy",
      "index.lifecycle.rollover_alias": "logs-write"
    }
  }
}
```

```json
{
  "acknowledged": true
}
```

To bootstrap the first index and alias for rollover:

```json
PUT /logs-000001
{
  "aliases": {
    "logs-write": {
      "is_write_index": true
    }
  }
}
```

```json
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "logs-000001"
}
```

Now all writes go to the `logs-write` alias, and ILM will automatically roll over to
`logs-000002` when the configured thresholds are met, then transition older indices through
warm, cold, and delete phases.

---

### Rollover

The rollover API creates a new index when the current write index meets specified conditions.
Rollover is typically used with aliases and ILM, but can also be invoked manually. The
conditions can be based on:

- **`max_age`** -- the maximum time since the index was created.
- **`max_primary_shard_size`** -- the maximum size of the largest primary shard.
- **`max_docs`** -- the maximum number of documents in the index.
- **`max_size`** -- the maximum total size of all primary shards.

```json
POST /logs-write/_rollover
{
  "conditions": {
    "max_age": "7d",
    "max_primary_shard_size": "50gb",
    "max_docs": 100000000
  }
}
```

```json
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "old_index": "logs-000001",
  "new_index": "logs-000002",
  "rolled_over": true,
  "dry_run": false,
  "conditions": {
    "[max_age: 7d]": true,
    "[max_primary_shard_size: 50gb]": false,
    "[max_docs: 100000000]": false
  }
}
```

The response shows that the rollover occurred because the `max_age` condition was met (the
index was older than 7 days). Rollover succeeds if **any** of the conditions evaluate to true.
The new index `logs-000002` now becomes the write index for the `logs-write` alias, and the
old index `logs-000001` is no longer the write target.

**Dry run** to check conditions without actually rolling over:

```json
POST /logs-write/_rollover?dry_run
{
  "conditions": {
    "max_age": "7d",
    "max_primary_shard_size": "50gb"
  }
}
```

---

### Shrink and Split

The **shrink API** reduces the number of primary shards in an index. This is useful in the warm
phase of ILM to consolidate shards and reduce overhead. The target shard count must be a factor
of the original shard count (e.g., 6 shards can shrink to 3, 2, or 1).

Before shrinking, the index must be made read-only and all shards must be relocated to a single
node:

```json
PUT /logs-000001/_settings
{
  "settings": {
    "index.routing.allocation.require._name": "shrink_node",
    "index.blocks.write": true
  }
}
```

```json
{
  "acknowledged": true
}
```

Then perform the shrink:

```json
POST /logs-000001/_shrink/logs-000001-shrunk
{
  "settings": {
    "index.number_of_shards": 1,
    "index.number_of_replicas": 1,
    "index.routing.allocation.require._name": null,
    "index.blocks.write": null
  }
}
```

```json
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "logs-000001-shrunk"
}
```

The **split API** increases the number of primary shards. The target shard count must be a
multiple of the original (e.g., 2 shards can split to 4, 6, 8, etc.). The index must also be
made read-only before splitting.

```json
PUT /products/_settings
{
  "settings": {
    "index.blocks.write": true
  }
}
```

```json
POST /products/_split/products-split
{
  "settings": {
    "index.number_of_shards": 6
  }
}
```

```json
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "products-split"
}
```

Both shrink and split create a new index -- the original remains unchanged until you delete it
or swap aliases.

---

### Force Merge

The force merge API triggers a Lucene-level segment merge, reducing the number of segments in
each shard to improve search performance and reclaim disk space from deleted documents. This is
most beneficial for read-only indices that will no longer receive writes.

**Warning:** Force merging a write-active index can cause large segments to be produced that
will never be eligible for automatic merging, leading to degraded performance over time. Only
force merge indices that are no longer receiving writes.

```json
POST /logs-000001/_forcemerge?max_num_segments=1
```

```json
{
  "_shards": {
    "total": 4,
    "successful": 4,
    "failed": 0
  }
}
```

The `max_num_segments=1` parameter tells Elasticsearch to merge all segments in each shard down
to a single segment. This is the most aggressive optimization and yields the best read
performance, but the merge operation itself can be resource-intensive.

Force merge across multiple indices:

```json
POST /logs-2024.01.*/_forcemerge?max_num_segments=1
```

---

### Close and Open

Closing an index removes it from the cluster's active resource pool. A closed index consumes
almost no cluster resources (no memory for field data, no shard allocation overhead) but retains
its data on disk. This is useful for indices that are rarely accessed but must be kept for
compliance or historical analysis.

**Closing an index:**

```json
POST /logs-000001/_close
```

```json
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "indices": {
    "logs-000001": {
      "closed": true
    }
  }
}
```

A closed index cannot be read from or written to. Attempting to search a closed index returns
an error.

**Opening an index:**

```json
POST /logs-000001/_open
```

```json
{
  "acknowledged": true,
  "shards_acknowledged": true
}
```

Opening an index restores it to full operational status. The cluster reallocates its shards and
makes it available for reads and writes. The open operation may take time depending on the size
of the index and the number of shards to recover.

```
  Close / Open Workflow
  ---------------------

  +-------------------+       +--------------------+       +-------------------+
  |   Active Index    |       |   Closed Index     |       |   Active Index    |
  |                   |       |                    |       |                   |
  |  - Searchable     +------>+  - Not searchable  +------>+  - Searchable     |
  |  - Writable       | CLOSE |  - Not writable    | OPEN  |  - Writable       |
  |  - Uses memory    |       |  - Minimal resources|      |  - Uses memory    |
  |  - Shard alloc    |       |  - Data on disk    |       |  - Shard alloc    |
  +-------------------+       +--------------------+       +-------------------+
```

---

### Deleting an Index

Deleting an index permanently removes all of its data, mappings, and settings. This operation
is irreversible -- there is no recycle bin or undo. In production environments, index deletion
is typically handled automatically by ILM policies, but manual deletion is available for
cleanup and maintenance.

```json
DELETE /logs-000001
```

```json
{
  "acknowledged": true
}
```

**Deleting multiple indices** using a wildcard pattern:

```json
DELETE /logs-2024.01.*
```

To prevent accidental deletion of all indices (via `DELETE /*` or `DELETE /_all`), Elasticsearch
provides the `action.destructive_requires_name` cluster setting, which should be set to `true`
in production:

```json
PUT /_cluster/settings
{
  "persistent": {
    "action.destructive_requires_name": true
  }
}
```

---

### Command Reference

| Operation | REST Verb & Endpoint | Key Parameters |
|-----------|----------------------|----------------|
| Create index | `PUT /<index>` | `settings`, `mappings` |
| Delete index | `DELETE /<index>` | Supports wildcards |
| Get index settings | `GET /<index>/_settings` | `flat_settings`, `include_defaults` |
| Update index settings | `PUT /<index>/_settings` | `index.number_of_replicas`, `index.refresh_interval` |
| Get mapping | `GET /<index>/_mapping` | |
| Update mapping | `PUT /<index>/_mapping` | `properties` (add-only, cannot change existing) |
| Create index template | `PUT /_index_template/<name>` | `index_patterns`, `priority`, `template`, `composed_of` |
| Create component template | `PUT /_component_template/<name>` | `template` (settings, mappings, or aliases) |
| Add/remove alias | `POST /_aliases` | `actions: [add/remove]`, `index`, `alias`, `filter` |
| Get aliases | `GET /<index>/_alias` | |
| Reindex | `POST /_reindex` | `source.index`, `dest.index`, `source.query` |
| Refresh | `POST /<index>/_refresh` | |
| Force merge | `POST /<index>/_forcemerge` | `max_num_segments` |
| Rollover | `POST /<alias>/_rollover` | `conditions.max_age`, `conditions.max_docs`, `conditions.max_primary_shard_size` |
| Shrink index | `POST /<index>/_shrink/<target>` | `settings.index.number_of_shards` |
| Split index | `POST /<index>/_split/<target>` | `settings.index.number_of_shards` |
| Close index | `POST /<index>/_close` | |
| Open index | `POST /<index>/_open` | |
| Create ILM policy | `PUT /_ilm/policy/<name>` | `policy.phases` (hot, warm, cold, delete) |
| Get ILM policy | `GET /_ilm/policy/<name>` | |
| Explain ILM status | `GET /<index>/_ilm/explain` | Shows current phase, action, and step |
| Cat indices | `GET /_cat/indices?v` | `health`, `status`, `index`, `pri`, `rep`, `docs.count` |
| Index exists | `HEAD /<index>` | Returns 200 if exists, 404 if not |
