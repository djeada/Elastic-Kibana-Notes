## SQL Compatibility

Elasticsearch exposes a SQL interface that lets you query indices with a familiar relational syntax
while still executing against Elasticsearch's distributed search engine. This is useful when you
want quick ad hoc analysis, integration with BI tooling, or a gentler learning path for users who
already know SQL but are new to the Query DSL.

The SQL layer does **not** turn Elasticsearch into a relational database. It translates supported
SQL statements into Elasticsearch requests and works best for filtering, projections, sorting, and
aggregations over denormalized documents. Understanding where the mapping between SQL concepts and
Elasticsearch concepts is strong -- and where it breaks down -- is essential.

### SQL vs Elasticsearch Concepts

| Relational SQL Concept | Elasticsearch Equivalent | Notes |
|------------------------|--------------------------|-------|
| Database               | Cluster / deployment     | One SQL request can target indices across the cluster. |
| Table                  | Index                    | An index stores JSON documents instead of rows. |
| Row                    | Document                 | Documents can contain nested JSON structures and arrays. |
| Column                 | Field                    | Field mappings determine types and query behavior. |
| Schema                 | Mapping                  | Mappings define field types, analyzers, and capabilities. |
| `SELECT ... WHERE ...` | SQL API or Query DSL     | SQL is translated internally to native search requests. |
| `GROUP BY`             | Aggregations             | Bucket + metric aggregations power grouped results. |
| `ORDER BY`             | Sort                     | Sorting requires sortable field types such as `keyword`, numeric, or date. |

### Why the Compatibility Layer Exists

The SQL API is designed for:

- users migrating from relational systems who need a familiar query syntax,
- analysts connecting BI tools through JDBC/ODBC drivers,
- quick data exploration without building verbose JSON Query DSL requests,
- translating SQL into Query DSL for learning and debugging.

It is **not** intended to replace the Query DSL in every scenario. Native Elasticsearch APIs remain
better for full-text relevance tuning, deeply nested queries, custom scoring, percolation, and
many advanced search features.

### Supported SQL Features

The Elasticsearch SQL API supports a practical subset of ANSI-style SQL:

- `SELECT`, `FROM`, `WHERE`, `ORDER BY`, `LIMIT`
- scalar functions such as `ROUND()`, `LENGTH()`, `YEAR()`, `MONTH()`
- aggregate functions such as `COUNT()`, `SUM()`, `AVG()`, `MIN()`, `MAX()`
- `GROUP BY` and `HAVING`
- catalog-style statements such as `SHOW TABLES`, `SHOW COLUMNS`, and `DESCRIBE`
- pagination through cursors for result sets that exceed a single page

Example query:

```json
POST /_sql?format=txt
{
  "query": """
    SELECT category, COUNT(*) AS total_products, ROUND(AVG(price), 2) AS avg_price
    FROM products_sql
    WHERE in_stock = true
    GROUP BY category
    HAVING COUNT(*) >= 1
    ORDER BY avg_price DESC
  """
}
```

Possible response:

```text
   category   | total_products | avg_price
--------------+----------------+----------
accessories   |              2 |     23.75
clothing      |              2 |     27.49
```

### Translating SQL to Query DSL

One of the most useful learning tools is the SQL Translate API. It reveals the Elasticsearch
request that the SQL engine generates.

```json
POST /_sql/translate
{
  "query": """
    SELECT name, price
    FROM products_sql
    WHERE category = 'clothing' AND price >= 20
    ORDER BY price DESC
    LIMIT 3
  """
}
```

Typical translated shape:

```json
{
  "size": 3,
  "_source": false,
  "fields": [
    { "field": "name" },
    { "field": "price" }
  ],
  "query": {
    "bool": {
      "filter": [
        { "term": { "category": "clothing" } },
        { "range": { "price": { "gte": 20 } } }
      ]
    }
  },
  "sort": [
    { "price": { "order": "desc" } }
  ]
}
```

This makes SQL a helpful bridge for users learning how relational clauses map to Elasticsearch
filters, ranges, sorting, and aggregations.

### Field Mapping Compatibility

SQL behavior depends heavily on index mappings.

| Field Type | SQL Experience | Recommendation |
|------------|----------------|----------------|
| `keyword`  | Best for equality filters, sorting, grouping | Use for IDs, categories, tags, and exact-match text |
| `text`     | Limited for SQL equality/grouping            | Prefer a `.keyword` multi-field when you need sorting or grouping |
| `integer`, `long`, `double` | Natural numeric behavior      | Good for filters, math, and aggregates |
| `date`     | Works well with date functions and ranges    | Store timestamps in a standard date format |
| `boolean`  | Straightforward true/false filters           | Useful in `WHERE` clauses |
| `nested`   | More limited than native nested queries      | Use Query DSL when nested semantics become complex |

The biggest practical rule is: **grouping and sorting work best on exact-value fields** such as
`keyword`, not analyzed `text` fields.

### Important Limitations

Elasticsearch SQL intentionally omits or constrains several relational features:

1. **No traditional joins.** Elasticsearch is built for denormalized documents, not runtime
   relational joins across large tables.
2. **Limited subquery patterns.** Some subqueries are rewritten internally, but support is narrower
   than in a full relational engine.
3. **No multi-statement transactions.** Elasticsearch is not an ACID relational database.
4. **Full-text relevance is richer in Query DSL.** SQL is convenient, but Query DSL is better for
   `match`, fuzziness, custom scoring, highlighting, and ranking control.
5. **Arrays and multi-valued fields can be surprising.** SQL expects scalar column values, while
   Elasticsearch documents may contain arrays.
6. **Schema flexibility cuts both ways.** Dynamic mappings make ingestion easy, but inconsistent
   field types across indices can break SQL queries.

Unsupported join example:

```json
POST /_sql
{
  "query": """
    SELECT o.order_id, c.customer_name
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
  """
}
```

Instead of joins, model related data together inside a document, use parent-child relationships
only when necessary, or perform the join in your application layer.

### Pagination and Result Windows

For larger result sets, the SQL API returns a `cursor` that you can use to fetch the next page:

```json
POST /_sql
{
  "query": "SELECT name, price FROM products_sql ORDER BY price",
  "fetch_size": 2
}
```

Follow-up request:

```json
POST /_sql
{
  "cursor": "sDXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAAEWYUpOYlF4..."
}
```

This is the SQL-layer equivalent of scrolling through a result set page by page.

### When to Use SQL vs Query DSL

Use **SQL** when you want:

- quick tabular exploration,
- compatibility with analysts and BI tools,
- familiar relational syntax for filtering and aggregation,
- easy translation into Query DSL for learning.

Use the **Query DSL** when you need:

- full-text search and relevance tuning,
- nested queries and advanced bool logic,
- highlighting, suggestions, and custom scoring,
- the broadest Elasticsearch feature coverage.

In practice, SQL compatibility is best viewed as an accessibility layer on top of Elasticsearch,
not as a replacement for the native APIs.
