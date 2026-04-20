## Task 13: SQL Compatibility with Elasticsearch

**Objectives:**
- Use the Elasticsearch SQL API for familiar tabular queries.
- Compare SQL statements with the Elasticsearch concepts they target.
- Practice filtering, sorting, grouping, and aggregation through SQL.
- Translate SQL statements into Query DSL.
- Recognize important compatibility limits such as the lack of traditional joins.

---

### SQL Compatibility Workflow

```
  SQL Client / Kibana Dev Tools
             |
             v
  +-----------------------------+
  |   POST /_sql or /_sql/      |
  |   parse SQL text            |
  +-------------+---------------+
                |
                v
  +-----------------------------+
  |   SQL planner / translator  |
  |   maps clauses to ES APIs   |
  +-------------+---------------+
                |
                v
  +-----------------------------+
  |   Query DSL + aggregations  |
  |   executed on target index  |
  +-------------+---------------+
                |
                v
  +-----------------------------+
  |  Tabular rows or cursor     |
  |  returned to the client     |
  +-----------------------------+
```

---

### Prerequisite: Create an Index for SQL Queries

Use explicit mappings so SQL filtering, sorting, and grouping behave predictably.

```json
PUT /products_sql
{
  "mappings": {
    "properties": {
      "name":        { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "category":    { "type": "keyword" },
      "price":       { "type": "double" },
      "in_stock":    { "type": "boolean" },
      "created_at":  { "type": "date" },
      "supplier":    { "type": "keyword" }
    }
  }
}
```

Load sample data:

```json
POST /products_sql/_bulk
{"index": {"_id": "1"}}
{"name": "Elastic T-Shirt", "category": "clothing", "price": 19.99, "in_stock": true, "created_at": "2024-01-15", "supplier": "elastic-store"}
{"index": {"_id": "2"}}
{"name": "Kibana Mug", "category": "accessories", "price": 12.50, "in_stock": true, "created_at": "2024-02-01", "supplier": "dashboard-depot"}
{"index": {"_id": "3"}}
{"name": "Logstash Hoodie", "category": "clothing", "price": 45.00, "in_stock": true, "created_at": "2024-03-01", "supplier": "elastic-store"}
{"index": {"_id": "4"}}
{"name": "Beats Cap", "category": "accessories", "price": 15.00, "in_stock": false, "created_at": "2024-03-05", "supplier": "ops-wear"}
{"index": {"_id": "5"}}
{"name": "APM Sticker Pack", "category": "accessories", "price": 5.99, "in_stock": true, "created_at": "2024-03-10", "supplier": "dashboard-depot"}
{"index": {"_id": "6"}}
{"name": "Elastic Polo Shirt", "category": "clothing", "price": 34.99, "in_stock": true, "created_at": "2024-04-01", "supplier": "elastic-store"}
```

Refresh the index so documents become visible to SQL immediately:

```json
POST /products_sql/_refresh
```

---

### Step 1: Inspect the SQL Catalog

List the indices visible to the SQL engine:

```json
POST /_sql?format=txt
{
  "query": "SHOW TABLES"
}
```

Describe the columns in the lab index:

```json
POST /_sql?format=txt
{
  "query": "DESCRIBE products_sql"
}
```

Expected output shape:

```text
      column      |    type    | mapping
------------------+------------+---------
category          | VARCHAR    | keyword
created_at        | TIMESTAMP  | date
in_stock          | BOOLEAN    | boolean
name              | VARCHAR    | text
name.keyword      | VARCHAR    | keyword
price             | DOUBLE     | double
supplier          | VARCHAR    | keyword
```

---

### Step 2: Filter, Sort, and Limit Results

```json
POST /_sql?format=txt
{
  "query": """
    SELECT name, category, price
    FROM products_sql
    WHERE in_stock = true AND price >= 10
    ORDER BY price DESC
    LIMIT 3
  """
}
```

Expected output:

```text
       name         |  category   | price
--------------------+-------------+-------
Logstash Hoodie     | clothing    | 45.0
Elastic Polo Shirt  | clothing    | 34.99
Elastic T-Shirt     | clothing    | 19.99
```

Equivalent curl command:

```bash
curl -s -X POST "http://localhost:9200/_sql?format=txt" \
  -H "Content-Type: application/json" \
  -d '{"query":"SELECT name, category, price FROM products_sql WHERE in_stock = true AND price >= 10 ORDER BY price DESC LIMIT 3"}'
```

---

### Step 3: Aggregate with GROUP BY

```json
POST /_sql?format=txt
{
  "query": """
    SELECT category,
           COUNT(*) AS total_products,
           ROUND(AVG(price), 2) AS avg_price,
           MAX(price) AS max_price
    FROM products_sql
    WHERE in_stock = true
    GROUP BY category
    ORDER BY avg_price DESC
  """
}
```

Expected output:

```text
  category   | total_products | avg_price | max_price
-------------+----------------+-----------+----------
clothing     |              3 |     33.33 |     45.0
accessories  |              2 |      9.25 |     12.5
```

This is the SQL-facing equivalent of a `terms` aggregation with nested metric aggregations.

---

### Step 4: Use Date Functions

```json
POST /_sql?format=txt
{
  "query": """
    SELECT YEAR(created_at) AS sale_year,
           MONTH(created_at) AS sale_month,
           COUNT(*) AS documents_loaded
    FROM products_sql
    GROUP BY sale_year, sale_month
    ORDER BY sale_year, sale_month
  """
}
```

Expected output:

```text
sale_year | sale_month | documents_loaded
----------+------------+-----------------
     2024 |          1 |               1
     2024 |          2 |               1
     2024 |          3 |               3
     2024 |          4 |               1
```

---

### Step 5: Translate SQL into Query DSL

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

---

### Step 6: Observe an Important Compatibility Limit

Traditional relational joins are not supported:

```json
POST /_sql
{
  "query": """
    SELECT p.name, s.region
    FROM products_sql p
    JOIN suppliers s ON p.supplier = s.name
  """
}
```

Expected outcome: Elasticsearch returns an error explaining that joins are not supported by the SQL
engine.

> **Takeaway:** Favor denormalized documents and application-side joins when moving relational
> workloads into Elasticsearch.

---

### Optional Python Exercise

```python
import requests

query = {
    "query": """
        SELECT supplier, COUNT(*) AS total_products
        FROM products_sql
        WHERE in_stock = true
        GROUP BY supplier
        ORDER BY total_products DESC
    """
}

response = requests.post(
    "http://localhost:9200/_sql?format=json",
    json=query,
    timeout=30,
)
response.raise_for_status()
print(response.json())
```

This is a lightweight way to embed Elasticsearch SQL queries into automation or reporting scripts.
