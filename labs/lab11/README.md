## Task 11: Dashboarding with Kibana and Python Data Visualization

Objectives:

- Index a sample dataset suitable for multi-dimensional visualization.
- Explore indexed data using Kibana Discover with filters, time ranges, and field selection.
- Build four visualization types in Kibana's Visualize Library: bar chart, pie chart, line chart, and data table.
- Assemble a Kibana dashboard combining multiple visualizations with shared filters and time controls.
- Write Python scripts that query Elasticsearch aggregations and render charts with Matplotlib.
- Produce at least two distinct Python chart types: a bar chart and a pie chart.
- Export and share Kibana dashboards for collaboration.
- Compare Kibana and Python visualization approaches on interactivity, customization, reproducibility, and sharing.

Visualization Pipeline Overview

```
┌──────────────────────┐
│  Elasticsearch Data  │
│  (Indexed Documents) │
└─────────┬────────────┘
          │
          ▼
┌──────────────────────┐
│  Aggregation Queries │
│  (Terms, Date Hist,  │
│   Metrics, Filters)  │
└────┬────────────┬────┘
     │            │
     ▼            ▼
┌───────────┐ ┌──────────────┐
│  Kibana   │ │   Python     │
│ Visualize │ │ (Matplotlib, │
│  Library  │ │  Plotly)     │
└────┬──────┘ └──────┬───────┘
     │               │
     ▼               ▼
┌─────────────┐ ┌──────────────┐
│ Dashboard   │ │  Plot / PNG  │
│(Interactive)│ │  (Static)    │
└─────────────┘ └──────────────┘
```

### Prerequisites

- Elasticsearch running on `http://localhost:9200`.
- Kibana running on `http://localhost:5601`.
- Python 3.x with `requests` and `matplotlib` installed:

```bash
pip install requests matplotlib
```

> **Note:** This lab assumes a local development Elasticsearch instance with security disabled, matching the earlier Docker-based labs. If your cluster uses HTTPS and authentication, update the Python scripts and `curl` commands to include credentials and certificate verification.

### Step 1 — Preparing Sample Data

Index a realistic e-commerce sales dataset with products, categories, prices, quantities, regions, stores, payment methods, order status, and timestamps.

The dataset includes enough variety to support useful dashboard filters and aggregations:

- `category.keyword` for terms aggregations.
- `product.keyword`, `region.keyword`, `store.keyword`, and `payment_method.keyword` for filters and breakdowns.
- `price`, `quantity`, and `revenue` for numeric metrics.
- `sale_date` for time-based charts.
- `status.keyword` for filtering completed, returned, pending, and cancelled orders.

#### 1a — Create the Index with an Explicit Mapping

Delete any previous copy of the index, then create a clean `store_sales` index.

```bash
curl -s -X DELETE "http://localhost:9200/store_sales?pretty"
```

```bash
curl -s -X PUT "http://localhost:9200/store_sales?pretty" \
  -H "Content-Type: application/json" -d '
{
  "mappings": {
    "properties": {
      "order_id":         { "type": "keyword" },
      "product":          { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "category":         { "type": "keyword" },
      "region":           { "type": "keyword" },
      "store":            { "type": "keyword" },
      "customer_segment": { "type": "keyword" },
      "payment_method":   { "type": "keyword" },
      "status":           { "type": "keyword" },
      "price":            { "type": "double" },
      "quantity":         { "type": "integer" },
      "revenue":          { "type": "double" },
      "sale_date":        { "type": "date" }
    }
  }
}'
```

**Expected output:**

```json
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "store_sales"
}
```

#### 1b — Bulk Index Sales Documents

The `_bulk` API requires one action line followed by one document line. Each document below represents one sales order.

```bash
curl -s -X POST "http://localhost:9200/store_sales/_bulk?pretty" \
  -H "Content-Type: application/x-ndjson" -d '
{"index":{"_id":"S-1001"}}
{"order_id":"S-1001","product":"Laptop Pro 14","category":"Electronics","region":"North","store":"Berlin Mitte","customer_segment":"Business","payment_method":"Credit Card","status":"completed","price":1299.99,"quantity":2,"revenue":2599.98,"sale_date":"2025-01-03T10:15:00"}
{"index":{"_id":"S-1002"}}
{"order_id":"S-1002","product":"Noise Cancelling Headphones","category":"Electronics","region":"North","store":"Berlin Mitte","customer_segment":"Consumer","payment_method":"PayPal","status":"completed","price":199.99,"quantity":5,"revenue":999.95,"sale_date":"2025-01-04T14:40:00"}
{"index":{"_id":"S-1003"}}
{"order_id":"S-1003","product":"USB-C Dock","category":"Electronics","region":"West","store":"Cologne Central","customer_segment":"Business","payment_method":"Invoice","status":"completed","price":149.99,"quantity":4,"revenue":599.96,"sale_date":"2025-01-05T09:30:00"}
{"index":{"_id":"S-1004"}}
{"order_id":"S-1004","product":"Wireless Mouse","category":"Electronics","region":"South","store":"Munich City","customer_segment":"Consumer","payment_method":"Debit Card","status":"returned","price":39.99,"quantity":3,"revenue":119.97,"sale_date":"2025-01-06T16:10:00"}
{"index":{"_id":"S-1005"}}
{"order_id":"S-1005","product":"Desk Chair","category":"Furniture","region":"South","store":"Munich City","customer_segment":"Business","payment_method":"Invoice","status":"completed","price":249.00,"quantity":6,"revenue":1494.00,"sale_date":"2025-01-08T11:20:00"}
{"index":{"_id":"S-1006"}}
{"order_id":"S-1006","product":"Bookshelf","category":"Furniture","region":"East","store":"Leipzig West","customer_segment":"Consumer","payment_method":"Credit Card","status":"completed","price":129.50,"quantity":4,"revenue":518.00,"sale_date":"2025-01-09T13:15:00"}
{"index":{"_id":"S-1007"}}
{"order_id":"S-1007","product":"Standing Desk","category":"Furniture","region":"North","store":"Hamburg Harbor","customer_segment":"Business","payment_method":"Invoice","status":"completed","price":499.00,"quantity":3,"revenue":1497.00,"sale_date":"2025-01-10T15:50:00"}
{"index":{"_id":"S-1008"}}
{"order_id":"S-1008","product":"Desk Lamp","category":"Furniture","region":"West","store":"Cologne Central","customer_segment":"Consumer","payment_method":"PayPal","status":"pending","price":44.99,"quantity":7,"revenue":314.93,"sale_date":"2025-01-11T18:05:00"}
{"index":{"_id":"S-1009"}}
{"order_id":"S-1009","product":"Running Shoes","category":"Apparel","region":"West","store":"Cologne Central","customer_segment":"Consumer","payment_method":"Credit Card","status":"completed","price":89.99,"quantity":12,"revenue":1079.88,"sale_date":"2025-01-13T12:25:00"}
{"index":{"_id":"S-1010"}}
{"order_id":"S-1010","product":"Winter Jacket","category":"Apparel","region":"North","store":"Hamburg Harbor","customer_segment":"Consumer","payment_method":"Debit Card","status":"completed","price":149.99,"quantity":8,"revenue":1199.92,"sale_date":"2025-01-14T09:45:00"}
{"index":{"_id":"S-1011"}}
{"order_id":"S-1011","product":"Cotton T-Shirt","category":"Apparel","region":"East","store":"Leipzig West","customer_segment":"Consumer","payment_method":"PayPal","status":"completed","price":19.99,"quantity":25,"revenue":499.75,"sale_date":"2025-01-15T17:30:00"}
{"index":{"_id":"S-1012"}}
{"order_id":"S-1012","product":"Backpack","category":"Apparel","region":"South","store":"Munich City","customer_segment":"Student","payment_method":"Debit Card","status":"cancelled","price":69.99,"quantity":6,"revenue":419.94,"sale_date":"2025-01-16T10:05:00"}
{"index":{"_id":"S-1013"}}
{"order_id":"S-1013","product":"Blender","category":"Kitchen","region":"South","store":"Munich City","customer_segment":"Consumer","payment_method":"Credit Card","status":"completed","price":45.00,"quantity":10,"revenue":450.00,"sale_date":"2025-01-18T08:30:00"}
{"index":{"_id":"S-1014"}}
{"order_id":"S-1014","product":"Coffee Maker","category":"Kitchen","region":"West","store":"Cologne Central","customer_segment":"Consumer","payment_method":"PayPal","status":"completed","price":79.99,"quantity":9,"revenue":719.91,"sale_date":"2025-01-19T11:10:00"}
{"index":{"_id":"S-1015"}}
{"order_id":"S-1015","product":"Air Fryer","category":"Kitchen","region":"East","store":"Leipzig West","customer_segment":"Consumer","payment_method":"Credit Card","status":"completed","price":119.99,"quantity":5,"revenue":599.95,"sale_date":"2025-01-20T14:35:00"}
{"index":{"_id":"S-1016"}}
{"order_id":"S-1016","product":"Chef Knife Set","category":"Kitchen","region":"North","store":"Berlin Mitte","customer_segment":"Business","payment_method":"Invoice","status":"returned","price":89.99,"quantity":4,"revenue":359.96,"sale_date":"2025-01-21T16:55:00"}
{"index":{"_id":"S-1017"}}
{"order_id":"S-1017","product":"Novel Collection","category":"Books","region":"East","store":"Leipzig West","customer_segment":"Consumer","payment_method":"Debit Card","status":"completed","price":24.99,"quantity":15,"revenue":374.85,"sale_date":"2025-01-23T12:00:00"}
{"index":{"_id":"S-1018"}}
{"order_id":"S-1018","product":"Data Science Textbook","category":"Books","region":"South","store":"Munich City","customer_segment":"Student","payment_method":"Credit Card","status":"completed","price":79.99,"quantity":7,"revenue":559.93,"sale_date":"2025-01-24T15:30:00"}
{"index":{"_id":"S-1019"}}
{"order_id":"S-1019","product":"Cookbook","category":"Books","region":"West","store":"Cologne Central","customer_segment":"Consumer","payment_method":"PayPal","status":"completed","price":29.99,"quantity":11,"revenue":329.89,"sale_date":"2025-01-25T10:25:00"}
{"index":{"_id":"S-1020"}}
{"order_id":"S-1020","product":"Travel Guide","category":"Books","region":"North","store":"Hamburg Harbor","customer_segment":"Consumer","payment_method":"Debit Card","status":"pending","price":18.99,"quantity":10,"revenue":189.90,"sale_date":"2025-01-26T13:20:00"}
{"index":{"_id":"S-1021"}}
{"order_id":"S-1021","product":"Smart Watch","category":"Electronics","region":"East","store":"Leipzig West","customer_segment":"Consumer","payment_method":"Credit Card","status":"completed","price":249.99,"quantity":6,"revenue":1499.94,"sale_date":"2025-01-27T09:10:00"}
{"index":{"_id":"S-1022"}}
{"order_id":"S-1022","product":"Gaming Monitor","category":"Electronics","region":"South","store":"Munich City","customer_segment":"Business","payment_method":"Invoice","status":"completed","price":349.99,"quantity":4,"revenue":1399.96,"sale_date":"2025-01-28T16:45:00"}
{"index":{"_id":"S-1023"}}
{"order_id":"S-1023","product":"Sofa","category":"Furniture","region":"North","store":"Hamburg Harbor","customer_segment":"Consumer","payment_method":"Credit Card","status":"completed","price":899.00,"quantity":1,"revenue":899.00,"sale_date":"2025-01-29T10:50:00"}
{"index":{"_id":"S-1024"}}
{"order_id":"S-1024","product":"Dining Table","category":"Furniture","region":"East","store":"Leipzig West","customer_segment":"Consumer","payment_method":"PayPal","status":"completed","price":399.00,"quantity":2,"revenue":798.00,"sale_date":"2025-01-30T18:25:00"}
'
```

Refresh the index so the documents are immediately searchable:

```bash
curl -s -X POST "http://localhost:9200/store_sales/_refresh?pretty"
```

#### 1c — Verify the Data

Check the document count:

```bash
curl -s "http://localhost:9200/store_sales/_count?pretty"
```

**Expected output:**

```json
{
  "count": 24,
  "_shards": { "total": 1, "successful": 1, "skipped": 0, "failed": 0 }
}
```

Preview a few documents:

```bash
curl -s "http://localhost:9200/store_sales/_search?size=3&pretty" \
  -H "Content-Type: application/json" -d '
{
  "sort": [{ "sale_date": "asc" }],
  "_source": ["order_id", "product", "category", "region", "quantity", "revenue", "sale_date"]
}'
```

Run a quick revenue aggregation to confirm the data supports dashboard metrics:

```bash
curl -s "http://localhost:9200/store_sales/_search?pretty" \
  -H "Content-Type: application/json" -d '
{
  "size": 0,
  "aggs": {
    "revenue_by_category": {
      "terms": { "field": "category", "size": 10 },
      "aggs": {
        "total_revenue": { "sum": { "field": "revenue" } },
        "units_sold":    { "sum": { "field": "quantity" } }
      }
    }
  }
}'
```

### Step 2 — Kibana Discover: Exploring Data

Kibana Discover lets you inspect raw documents before building visualizations. This is where you confirm that the data exists, the time field works, and your fields are mapped correctly.

1. Open **Kibana** at `http://localhost:5601`.
2. Navigate to **Discover** from the left sidebar menu.
3. Create a **Data View**:
   - Go to **Stack Management → Data Views**.
   - Click **Create data view**.
   - Enter `store_sales` as the name and index pattern.
   - Select `sale_date` as the time field.
   - Save the data view.
4. Return to **Discover** and select the `store_sales` data view.

**Filtering and field selection:**

| Action | How |
|---|---|
| Set time range | Click the time picker in the top-right corner → set it to **Jan 1 2025 – Jan 31 2025** |
| Add a category filter | Click **+ Add filter** → field: `category`, operator: `is`, value: `Electronics` |
| Add a status filter | Click **+ Add filter** → field: `status`, operator: `is`, value: `completed` |
| Select visible fields | In the left field list, click **+** next to `order_id`, `product`, `category`, `region`, `quantity`, and `revenue` |
| Search with KQL | Type `revenue > 1000` in the search bar and press Enter |

> **Tip:** Kibana Query Language (KQL) supports `AND`, `OR`, and field-level filters.
> Example: `category: "Electronics" AND status: "completed" AND revenue > 1000`

### Step 3 — Kibana Visualize Library

Navigate to **Visualize Library** from the left sidebar, or go to **Dashboard → Create visualization**.

Kibana Lens is recommended because it lets you drag fields onto the canvas and automatically suggests chart types.

#### 3a — Bar Chart: Revenue by Category

This chart answers: *Which product categories generate the most revenue?*

1. Click **Create visualization** → select **Lens**.
2. Select the `store_sales` data view.
3. Drag the `category` field to the horizontal axis.
4. Drag the `revenue` field to the vertical axis.
5. Configure the axes:
   - **Horizontal axis:** `category` using a **Top values** / **Terms** aggregation.
   - **Vertical axis:** **Sum** of `revenue`.
6. Click the chart-type dropdown and select **Bar vertical stacked**.
7. Set the title: `Revenue by Category`.
8. Click **Save and return** or **Save to library**.

#### 3b — Pie Chart: Revenue Distribution by Category

This chart answers: *What percentage of total revenue comes from each category?*

1. Click **Create visualization** → select **Lens**.
2. Switch the chart type to **Pie**.
3. Configure:
   - **Slice by:** `category` using a **Top values** / **Terms** aggregation.
   - **Size by:** **Sum** of `revenue`.
4. Set the title: `Revenue Distribution by Category`.
5. Save the visualization.

> **Why use `revenue` instead of `price`?**
> `price` is the unit price. `revenue` is `price × quantity`, so it represents the real sales value of each order.

#### 3c — Line Chart: Sales Over Time

This chart answers: *How do sales trend across the month?*

1. Click **Create visualization** → select **Lens**.
2. Switch the chart type to **Line**.
3. Configure:
   - **Horizontal axis:** `sale_date` using a **Date histogram**.
   - **Interval:** `3 days` or `1 week`.
   - **Vertical axis:** **Sum** of `revenue` or **Sum** of `quantity`.
4. Optional: add a **breakdown** by `category` to compare category trends.
5. Set the title: `Sales Revenue Over Time`.
6. Save the visualization.

#### 3d — Data Table: Detailed Product Summary

This table provides a structured numeric summary by product.

1. Click **Create visualization** → select **Lens**.
2. Switch the chart type to **Table**.
3. Configure columns:
   - **Rows:** `product.keyword` or `product` using **Top values**.
   - **Metric 1:** **Sum** of `quantity`.
   - **Metric 2:** **Sum** of `revenue`.
   - **Metric 3:** **Average** of `price`.
4. Sort the table by **Sum of revenue** in descending order.
5. Set the title: `Product Summary Table`.
6. Save the visualization.

### Step 4 — Dashboard Assembly

A dashboard combines multiple visualizations into one interactive view.

1. Navigate to **Dashboard** from the left sidebar → click **Create dashboard**.
2. Click **Add from library** and select all four saved visualizations:
   - Revenue by Category
   - Revenue Distribution by Category
   - Sales Revenue Over Time
   - Product Summary Table
3. **Arrange panels** by dragging and resizing:
   - Top row: bar chart on the left, pie chart on the right.
   - Middle row: line chart full width.
   - Bottom row: data table full width.
4. **Add dashboard-level controls:**
   - Click **Controls**.
   - Add a dropdown for `category`.
   - Add a dropdown for `region`.
   - Add a dropdown for `status`.
   - These controls filter all panels at once.
5. Set the time picker to **Jan 2025**.
6. Click **Save** and name it `Store Sales Dashboard`.

> **Key feature:** Clicking a bar segment or pie slice can filter the rest of the dashboard. This cross-filtering makes Kibana useful for interactive analysis because one action updates every connected panel.

### Step 5 — Python Visualization with Matplotlib: Bar Chart

Use the Elasticsearch aggregation API to retrieve bucket data, then plot it with Matplotlib.

#### Aggregation Query

```python
import requests
import matplotlib.pyplot as plt

ES_URL = "http://localhost:9200"
INDEX = "store_sales"

# Aggregate total revenue by category.
agg_query = {
    "size": 0,
    "query": {
        "term": { "status": "completed" }
    },
    "aggs": {
        "by_category": {
            "terms": {
                "field": "category",
                "size": 10,
                "order": { "total_revenue": "desc" }
            },
            "aggs": {
                "total_revenue": { "sum": { "field": "revenue" } },
                "units_sold":    { "sum": { "field": "quantity" } }
            }
        }
    }
}

response = requests.get(f"{ES_URL}/{INDEX}/_search", json=agg_query, timeout=10)
result = response.json()

if "error" in result:
    raise RuntimeError(result["error"])

buckets = result["aggregations"]["by_category"]["buckets"]
categories = [bucket["key"] for bucket in buckets]
revenues = [bucket["total_revenue"]["value"] for bucket in buckets]
units = [bucket["units_sold"]["value"] for bucket in buckets]

print("Completed Revenue by Category:")
for category, revenue, unit_count in zip(categories, revenues, units):
    print(f"  {category:<12} revenue=${revenue:,.2f} units={unit_count:.0f}")
```

**Expected output from the aggregation:**

```text
Completed Revenue by Category:
  Electronics  revenue=$7,099.79 units=21
  Furniture    revenue=$5,206.00 units=16
  Apparel      revenue=$2,779.55 units=45
  Kitchen      revenue=$1,769.86 units=24
  Books        revenue=$1,264.67 units=33
```

#### Plotting the Bar Chart

```python
fig, ax = plt.subplots(figsize=(9, 5))

bars = ax.bar(categories, revenues)
ax.set_title("Completed Revenue by Category", fontsize=14, fontweight="bold")
ax.set_xlabel("Category", fontsize=12)
ax.set_ylabel("Revenue ($)", fontsize=12)

# Add value labels on top of each bar.
for bar, revenue in zip(bars, revenues):
    ax.text(
        bar.get_x() + bar.get_width() / 2,
        bar.get_height(),
        f"${revenue:,.0f}",
        ha="center",
        va="bottom",
        fontsize=10
    )

plt.xticks(rotation=30, ha="right")
plt.tight_layout()
plt.savefig("category_revenue_bar_chart.png", dpi=150)
plt.show()
```

### Step 6 — Python Visualization with Matplotlib: Pie Chart

A second chart type shows revenue distribution, mirroring the Kibana pie chart.

#### Aggregation Query for Revenue

```python
revenue_query = {
    "size": 0,
    "query": {
        "term": { "status": "completed" }
    },
    "aggs": {
        "revenue_by_category": {
            "terms": {
                "field": "category",
                "size": 10,
                "order": { "total_revenue": "desc" }
            },
            "aggs": {
                "total_revenue": {
                    "sum": { "field": "revenue" }
                }
            }
        }
    }
}

response = requests.get(f"{ES_URL}/{INDEX}/_search", json=revenue_query, timeout=10)
result = response.json()

if "error" in result:
    raise RuntimeError(result["error"])

buckets = result["aggregations"]["revenue_by_category"]["buckets"]
categories = [bucket["key"] for bucket in buckets]
revenues = [bucket["total_revenue"]["value"] for bucket in buckets]

total_revenue = sum(revenues)

print("Revenue Distribution by Category:")
for category, revenue in zip(categories, revenues):
    percent = (revenue / total_revenue) * 100 if total_revenue else 0
    print(f"  {category:<12} ${revenue:,.2f} ({percent:.1f}%)")
```

**Expected output from the aggregation:**

```text
Revenue Distribution by Category:
  Electronics  $7,099.79 (39.2%)
  Furniture    $5,206.00 (28.7%)
  Apparel      $2,779.55 (15.3%)
  Kitchen      $1,769.86 (9.8%)
  Books        $1,264.67 (7.0%)
```

#### Plotting the Pie Chart

```python
fig, ax = plt.subplots(figsize=(7, 7))

ax.pie(
    revenues,
    labels=categories,
    autopct="%1.1f%%",
    startangle=140
)

ax.set_title("Revenue Distribution by Category", fontsize=14, fontweight="bold")
plt.tight_layout()
plt.savefig("revenue_pie_chart.png", dpi=150)
plt.show()
```

### Step 7 — Exporting and Sharing Dashboards

Kibana provides several methods to share dashboards with collaborators.

| Method | How | Best For |
|---|---|---|
| **Share link** | Dashboard → **Share** → **Get link** → copy permalink | Quick sharing inside the same Kibana environment |
| **Embed iframe** | Dashboard → **Share** → **Embed code** | Embedding in internal applications or portals |
| **PDF / PNG export** | Dashboard → **Share** → **PDF Reports** or **PNG Reports** | Offline distribution and presentations |
| **Export saved objects** | Stack Management → **Saved Objects** → select dashboard → **Export** | Migrating dashboards between Kibana instances |
| **Import saved objects** | Stack Management → **Saved Objects** → **Import** → upload `.ndjson` | Restoring or sharing dashboard definitions |

> **Note:** PDF and PNG reporting availability depends on your Elastic license and Kibana configuration.

### Kibana vs Python Visualization — Comparison

| Criteria | Kibana | Python (Matplotlib) |
|---|---|---|
| **Interactivity** | Built-in click-to-filter, zoom, hover tooltips, dashboard controls | Static by default; interactive if using libraries such as Plotly |
| **Setup effort** | Low — browser-based, mostly point-and-click | Moderate — requires scripts and Python packages |
| **Real-time data** | Supports dashboard auto-refresh | Must re-run the script or build a refresh loop |
| **Customization** | Good for standard dashboards, but constrained by available chart settings | Full control over layout, labels, annotations, and export format |
| **Sharing** | Links, embeds, saved objects, and reports | PNG/SVG/PDF files, notebooks, scripts, or generated reports |
| **Reproducibility** | Saved objects can be exported, but manual edits are common | Script is version-controllable and repeatable |
| **Learning curve** | Low for basic charts; moderate for Lens, TSVB, and Vega | Moderate; requires Python and visualization knowledge |
| **Best use case** | Live operational dashboards and team monitoring | Custom analysis, automated reports, and publication-quality figures |

### Troubleshooting Tips

| Problem | Likely Cause | Solution |
|---|---|---|
| No data in Discover | Time range does not cover document timestamps | Set the time picker to **Jan 1 2025 – Jan 31 2025** |
| Field not available in Visualize | Data view has stale field metadata | Refresh the `store_sales` data view in Stack Management |
| Aggregation returns empty buckets | Index name mismatch, zero documents, or active filters | Verify with `GET /store_sales/_count` and clear filters |
| Python script returns `KeyError` | Elasticsearch returned an error object instead of aggregations | Print `result`; check index name, field names, and mapping |
| Pie chart shows an `Other` slice | Too many unique terms or default term limit is small | Increase the **Top values** size or disable grouping if appropriate |
| Kibana shows a field as non-aggregatable | Field is mapped as `text` only | Use a `keyword`, numeric, or date field for aggregations |
| Matplotlib chart looks cramped | Labels overlap due to long category names | Use `plt.tight_layout()` and rotate x-axis labels |
| Dashboard filters do not affect all panels | Panels use different data views or time fields | Ensure all visualizations use the same `store_sales` data view |
| Revenue looks too low | Chart is summing `price` instead of `revenue` | Use **Sum of `revenue`**, not **Sum of `price`** |

### Cleanup

Remove the sample index when you are finished with the lab:

```bash
curl -s -X DELETE "http://localhost:9200/store_sales?pretty"
```

**Expected output:**

```json
{
  "acknowledged": true
}
```
