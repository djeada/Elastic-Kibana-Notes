## Task 13: SQL Compatibility with Elasticsearch

**Objectives:**

- Use the Elasticsearch SQL API to query Elasticsearch with familiar SQL syntax.
- Compare SQL statements with the Elasticsearch concepts they target.
- Practice filtering, sorting, grouping, aggregation, date functions, and pagination through SQL.
- Translate SQL statements into Elasticsearch Query DSL.
- Recognize important compatibility limits, especially the lack of traditional relational joins.
- Use Python to run Elasticsearch SQL queries in automation or reporting workflows.

### SQL Compatibility Workflow

Elasticsearch SQL lets you write SQL-like queries against Elasticsearch indices. Internally, Elasticsearch parses the SQL text, translates it into Query DSL and aggregations where possible, executes the request, and returns the result in a tabular format.

```
  SQL Client / Kibana Dev Tools / Python Script
             |
             v
  +-----------------------------+
  |   POST /_sql or /_sql/...   |
  |   Receive SQL text          |
  +-------------+---------------+
                |
                v
  +-----------------------------+
  |   SQL planner / translator  |
  |   Map SQL clauses to ES APIs|
  +-------------+---------------+
                |
                v
  +-----------------------------+
  |   Query DSL + aggregations  |
  |   Execute on target index   |
  +-------------+---------------+
                |
                v
  +-----------------------------+
  |  Tabular rows, JSON, CSV,   |
  |  text output, or cursor     |
  +-----------------------------+
```

> **Main idea:** Elasticsearch SQL is convenient for analysts and reporting tools, but it does not turn Elasticsearch into a relational database. It works best with denormalized, search-friendly documents.

### Prerequisite: Create Indices for SQL Queries

Use explicit mappings so SQL filtering, sorting, grouping, and date functions behave predictably.

#### 0a — Reset the lab indices

Run this first if you have already completed the lab and want a clean start.

```json
DELETE /products_sql
DELETE /suppliers_sql
```

If either index does not exist yet, Elasticsearch may return an `index_not_found_exception`. That is safe to ignore during setup.

#### 0b — Create the `products_sql` index

```json
PUT /products_sql
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "category":   { "type": "keyword" },
      "price":      { "type": "double" },
      "in_stock":   { "type": "boolean" },
      "stock_count":{ "type": "integer" },
      "created_at": { "type": "date" },
      "supplier":   { "type": "keyword" },
      "region":     { "type": "keyword" },
      "rating":     { "type": "double" }
    }
  }
}
```

**Why these field types matter:**

| Field | Mapping | Why it is useful for SQL |
|---|---|---|
| `name` | `text` + `keyword` subfield | Full-text search on `name`, exact sorting/grouping on `name.keyword` |
| `category`, `supplier`, `region` | `keyword` | Reliable `GROUP BY`, `ORDER BY`, and exact filters |
| `price`, `rating` | Numeric | Supports `AVG`, `SUM`, `MIN`, `MAX`, and numeric filters |
| `created_at` | `date` | Supports date functions like `YEAR()` and `MONTH()` |
| `in_stock` | `boolean` | Supports true/false filtering |

#### 0c — Load sample product data

This dataset is intentionally larger than the minimum required. It gives each query enough variety for filtering, sorting, grouping, date functions, and supplier-based examples.

```json
POST /products_sql/_bulk
{"index": {"_id": "1"}}
{"name": "Elastic T-Shirt", "category": "clothing", "price": 19.99, "in_stock": true, "stock_count": 120, "created_at": "2024-01-15", "supplier": "elastic-store", "region": "north", "rating": 4.4}
{"index": {"_id": "2"}}
{"name": "Kibana Mug", "category": "accessories", "price": 12.50, "in_stock": true, "stock_count": 85, "created_at": "2024-02-01", "supplier": "dashboard-depot", "region": "west", "rating": 4.7}
{"index": {"_id": "3"}}
{"name": "Logstash Hoodie", "category": "clothing", "price": 45.00, "in_stock": true, "stock_count": 35, "created_at": "2024-03-01", "supplier": "elastic-store", "region": "north", "rating": 4.8}
{"index": {"_id": "4"}}
{"name": "Beats Cap", "category": "accessories", "price": 15.00, "in_stock": false, "stock_count": 0, "created_at": "2024-03-05", "supplier": "ops-wear", "region": "east", "rating": 4.1}
{"index": {"_id": "5"}}
{"name": "APM Sticker Pack", "category": "accessories", "price": 5.99, "in_stock": true, "stock_count": 300, "created_at": "2024-03-10", "supplier": "dashboard-depot", "region": "west", "rating": 4.2}
{"index": {"_id": "6"}}
{"name": "Elastic Polo Shirt", "category": "clothing", "price": 34.99, "in_stock": true, "stock_count": 50, "created_at": "2024-04-01", "supplier": "elastic-store", "region": "south", "rating": 4.5}
{"index": {"_id": "7"}}
{"name": "Observability Notebook", "category": "office", "price": 9.99, "in_stock": true, "stock_count": 160, "created_at": "2024-04-12", "supplier": "dashboard-depot", "region": "west", "rating": 4.0}
{"index": {"_id": "8"}}
{"name": "Search Relevance Book", "category": "books", "price": 39.99, "in_stock": true, "stock_count": 42, "created_at": "2024-05-02", "supplier": "learning-press", "region": "east", "rating": 4.9}
{"index": {"_id": "9"}}
{"name": "Elasticsearch Cookbook", "category": "books", "price": 49.99, "in_stock": false, "stock_count": 0, "created_at": "2024-05-18", "supplier": "learning-press", "region": "east", "rating": 4.6}
{"index": {"_id": "10"}}
{"name": "Cluster Admin Backpack", "category": "accessories", "price": 59.99, "in_stock": true, "stock_count": 20, "created_at": "2024-06-03", "supplier": "ops-wear", "region": "south", "rating": 4.3}
{"index": {"_id": "11"}}
{"name": "Index Lifecycle Poster", "category": "office", "price": 7.50, "in_stock": true, "stock_count": 200, "created_at": "2024-06-10", "supplier": "dashboard-depot", "region": "north", "rating": 3.9}
{"index": {"_id": "12"}}
{"name": "Vector Search Pin", "category": "accessories", "price": 8.25, "in_stock": true, "stock_count": 180, "created_at": "2024-06-20", "supplier": "elastic-store", "region": "north", "rating": 4.1}
```

#### 0d — Create and load a `suppliers_sql` index

This second index is useful for demonstrating why traditional SQL joins are not supported. It also gives you a realistic dataset to discuss denormalization.

```json
PUT /suppliers_sql
{
  "mappings": {
    "properties": {
      "name":    { "type": "keyword" },
      "region":  { "type": "keyword" },
      "tier":    { "type": "keyword" },
      "active":  { "type": "boolean" }
    }
  }
}

POST /suppliers_sql/_bulk
{"index": {"_id": "elastic-store"}}
{"name": "elastic-store", "region": "north", "tier": "gold", "active": true}
{"index": {"_id": "dashboard-depot"}}
{"name": "dashboard-depot", "region": "west", "tier": "silver", "active": true}
{"index": {"_id": "ops-wear"}}
{"name": "ops-wear", "region": "south", "tier": "bronze", "active": true}
{"index": {"_id": "learning-press"}}
{"name": "learning-press", "region": "east", "tier": "gold", "active": false}
```

#### 0e — Refresh and verify the data

Refresh both indices so the documents are immediately visible to SQL queries.

```json
POST /products_sql/_refresh
POST /suppliers_sql/_refresh
```

Check document counts:

```json
GET /products_sql/_count
GET /suppliers_sql/_count
```

**Expected output:**

```json
{ "count": 12 }
{ "count": 4 }
```

You can also preview a few documents with Query DSL before using SQL:

```json
GET /products_sql/_search
{
  "size": 3,
  "sort": [{ "price": "desc" }],
  "_source": ["name", "category", "price", "supplier", "created_at"]
}
```

### Step 1: Inspect the SQL Catalog

The SQL catalog shows which Elasticsearch indices are visible to the SQL engine and how their fields are interpreted as SQL columns.

List the indices visible to the SQL engine:

```json
POST /_sql?format=txt
{
  "query": "SHOW TABLES"
}
```

Describe the columns in the product lab index:

```json
POST /_sql?format=txt
{
  "query": "DESCRIBE products_sql"
}
```

Expected output shape:

```text
      column       |    type    | mapping
-------------------+------------+---------
category           | VARCHAR    | keyword
created_at         | TIMESTAMP  | date
in_stock           | BOOLEAN    | boolean
name               | VARCHAR    | text
name.keyword       | VARCHAR    | keyword
price              | DOUBLE     | double
rating             | DOUBLE     | double
region             | VARCHAR    | keyword
stock_count        | INTEGER    | integer
supplier           | VARCHAR    | keyword
```

> **Note:** The exact order of rows may vary. Focus on whether the fields and SQL types appear correctly.

### Step 2: Filter, Sort, and Limit Results

Use `WHERE` to filter rows, `ORDER BY` to sort them, and `LIMIT` to restrict the number of returned results.

```json
POST /_sql?format=txt
{
  "query": """
    SELECT name, category, price
    FROM products_sql
    WHERE in_stock = true AND price >= 10
    ORDER BY price DESC
    LIMIT 5
  """
}
```

Expected output:

```text
          name           |  category   | price
-------------------------+-------------+-------
Cluster Admin Backpack   | accessories | 59.99
Logstash Hoodie          | clothing    | 45.0
Search Relevance Book    | books       | 39.99
Elastic Polo Shirt       | clothing    | 34.99
Elastic T-Shirt          | clothing    | 19.99
```

Equivalent curl command:

```bash
curl -s -X POST "http://localhost:9200/_sql?format=txt" \
  -H "Content-Type: application/json" \
  -d '{"query":"SELECT name, category, price FROM products_sql WHERE in_stock = true AND price >= 10 ORDER BY price DESC LIMIT 5"}'
```

**What this SQL means in Elasticsearch terms:**

| SQL clause | Elasticsearch equivalent |
|---|---|
| `WHERE in_stock = true` | Boolean filter on a `boolean` field |
| `WHERE price >= 10` | Range filter on a numeric field |
| `ORDER BY price DESC` | Sort on a numeric doc values field |
| `LIMIT 5` | Search request `size: 5` |

### Step 3: Aggregate with `GROUP BY`

SQL aggregations are translated into Elasticsearch bucket and metric aggregations.

```json
POST /_sql?format=txt
{
  "query": """
    SELECT category,
           COUNT(*) AS total_products,
           ROUND(AVG(price), 2) AS avg_price,
           MAX(price) AS max_price,
           SUM(stock_count) AS total_stock
    FROM products_sql
    WHERE in_stock = true
    GROUP BY category
    ORDER BY avg_price DESC
  """
}
```

Expected output:

```text
  category   | total_products | avg_price | max_price | total_stock
-------------+----------------+-----------+-----------+------------
books        |              1 |     39.99 |     39.99 |          42
clothing     |              3 |     33.33 |      45.0 |         205
accessories  |              4 |     21.68 |     59.99 |         585
office       |              2 |      8.75 |      9.99 |         360
```

This is the SQL-facing equivalent of a `terms` aggregation on `category` with nested metric aggregations for count, average, maximum, and sum.

### Step 4: Use Date Functions

Date functions make it easier to group documents by calendar period without manually writing a date histogram.

```json
POST /_sql?format=txt
{
  "query": """
    SELECT YEAR(created_at) AS product_year,
           MONTH(created_at) AS product_month,
           COUNT(*) AS documents_loaded
    FROM products_sql
    GROUP BY product_year, product_month
    ORDER BY product_year, product_month
  """
}
```

Expected output:

```text
product_year | product_month | documents_loaded
-------------+---------------+-----------------
        2024 |             1 |               1
        2024 |             2 |               1
        2024 |             3 |               3
        2024 |             4 |               2
        2024 |             5 |               2
        2024 |             6 |               3
```

You can also filter by date range:

```json
POST /_sql?format=txt
{
  "query": """
    SELECT name, created_at, price
    FROM products_sql
    WHERE created_at >= '2024-04-01' AND created_at < '2024-07-01'
    ORDER BY created_at ASC
  """
}
```

### Step 5: Translate SQL into Query DSL

Use the SQL translate API to see how Elasticsearch converts SQL into a Query DSL search request.

```json
POST /_sql/translate
{
  "query": """
    SELECT name, price
    FROM products_sql
    WHERE supplier = 'elastic-store' AND price >= 20
    ORDER BY price DESC
    LIMIT 2
  """
}
```

Expected translated shape:

```json
{
  "size": 2,
  "query": {
    "bool": {
      "filter": [
        { "term": { "supplier": "elastic-store" } },
        { "range": { "price": { "gte": 20 } } }
      ]
    }
  },
  "sort": [
    { "price": { "order": "desc" } }
  ]
}
```

Use this response to connect SQL clauses to their Query DSL equivalents.

| SQL | Query DSL concept |
|---|---|
| `SELECT name, price` | Source/field selection |
| `FROM products_sql` | Target index |
| `WHERE supplier = ...` | `term` filter |
| `WHERE price >= ...` | `range` filter |
| `ORDER BY price DESC` | Sort clause |
| `LIMIT 2` | Search request size |

### Step 6: Observe an Important Compatibility Limit

Traditional relational joins are not supported in Elasticsearch SQL. The following query looks like ordinary SQL, but it is not a supported Elasticsearch SQL pattern:

```json
POST /_sql
{
  "query": """
    SELECT p.name, s.region
    FROM products_sql p
    JOIN suppliers_sql s ON p.supplier = s.name
  """
}
```

Expected outcome: Elasticsearch returns an error explaining that joins are not supported by the SQL engine.

> **Takeaway:** Favor denormalized documents when moving relational workloads into Elasticsearch. If product search results need supplier region or supplier tier, store those fields directly in each product document or perform the join in the application before indexing.

A denormalized product document could look like this:

```json
POST /products_sql/_doc/13
{
  "name": "Denormalized Demo Item",
  "category": "demo",
  "price": 22.00,
  "in_stock": true,
  "stock_count": 10,
  "created_at": "2024-07-01",
  "supplier": "elastic-store",
  "supplier_region": "north",
  "supplier_tier": "gold",
  "region": "north",
  "rating": 4.0
}
```

### Step 7: Paginate SQL Results with a Cursor

For larger result sets, use a cursor instead of requesting all rows at once. The cursor lets Elasticsearch return one page of rows at a time.

```json
POST /_sql?format=json
{
  "query": """
    SELECT name, category, price
    FROM products_sql
    ORDER BY name.keyword ASC
  """,
  "fetch_size": 5
}
```

Expected output shape:

```json
{
  "columns": [
    { "name": "name", "type": "text" },
    { "name": "category", "type": "keyword" },
    { "name": "price", "type": "double" }
  ],
  "rows": [
    ["APM Sticker Pack", "accessories", 5.99],
    ["Beats Cap", "accessories", 15.0]
  ],
  "cursor": "..."
}
```

Request the next page using the returned `cursor` value:

```json
POST /_sql?format=json
{
  "cursor": "PASTE_CURSOR_VALUE_HERE"
}
```

When you are finished with a cursor before consuming all pages, clear it:

```json
POST /_sql/close
{
  "cursor": "PASTE_CURSOR_VALUE_HERE"
}
```

### Optional Python Exercise

This Python example runs an Elasticsearch SQL aggregation and prints the returned rows.

```python
import requests

ES_URL = "http://localhost:9200"

query = {
    "query": """
        SELECT supplier,
               COUNT(*) AS total_products,
               ROUND(AVG(price), 2) AS avg_price
        FROM products_sql
        WHERE in_stock = true
        GROUP BY supplier
        ORDER BY total_products DESC
    """
}

response = requests.post(
    f"{ES_URL}/_sql?format=json",
    json=query,
    timeout=30,
)
response.raise_for_status()

result = response.json()
columns = [col["name"] for col in result["columns"]]
rows = result["rows"]

print(" | ".join(columns))
print("-" * 60)
for row in rows:
    print(" | ".join(str(value) for value in row))
```

Expected output shape:

```text
supplier | total_products | avg_price
------------------------------------------------------------
elastic-store | 4 | 27.06
dashboard-depot | 4 | 8.99
ops-wear | 1 | 59.99
learning-press | 1 | 39.99
```

### Troubleshooting Tips

| Problem | Likely Cause | Solution |
|---|---|---|
| `Unknown index [products_sql]` | The setup index was not created | Run the prerequisite setup queries first |
| Empty SQL results | Documents were bulk indexed but not refreshed | Run `POST /products_sql/_refresh` |
| Cannot group by a field | Field is mapped as `text` only | Group by a `keyword`, numeric, date, or boolean field |
| Sort behaves unexpectedly on text | Sorting on analyzed text is not appropriate | Sort on `name.keyword` instead of `name` |
| Join query fails | Traditional SQL joins are unsupported | Denormalize data or join before indexing |
| SQL output is hard to read | Default JSON format is verbose | Use `?format=txt`, `?format=json`, or `?format=csv` |
| Python raises `KeyError` | Elasticsearch returned an error response | Print `response.text` before reading `result["rows"]` |
| Large query times out | Requesting too many rows at once | Use `fetch_size` and SQL cursors |

### Cleanup

Remove the lab indices when you are finished:

```json
DELETE /products_sql
DELETE /suppliers_sql
```

Expected output for each index:

```json
{ "acknowledged": true }
```

### Reflection

1. Which SQL clauses translated most directly into Query DSL concepts?
2. Why are `keyword` fields better than `text` fields for grouping and sorting?
3. How does `GROUP BY` in SQL relate to a `terms` aggregation in Elasticsearch?
4. Why are relational joins a poor fit for Elasticsearch's document-oriented model?
5. When would you choose Elasticsearch SQL instead of writing Query DSL directly?
6. How could denormalization make the supplier lookup use case easier to query?
