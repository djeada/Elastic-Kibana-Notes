## Task 11: Dashboarding with Kibana and Python Data Visualization

### Visualization Pipeline Overview

```
┌─────────────────────┐
│  Elasticsearch Data  │
│  (Indexed Documents) │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Aggregation Queries │
│  (Terms, Date Hist,  │
│   Metrics, Filters)  │
└────┬────────────┬────┘
     │            │
     ▼            ▼
┌──────────┐ ┌──────────────┐
│  Kibana   │ │   Python     │
│ Visualize │ │ (Matplotlib, │
│  Library  │ │  Plotly)     │
└────┬──────┘ └──────┬───────┘
     │               │
     ▼               ▼
┌──────────┐ ┌──────────────┐
│ Dashboard │ │  Plot / PNG  │
│ (Interactive)│ │  (Static)    │
└──────────┘ └──────────────┘
```

---

### Objectives

- Index a sample dataset suitable for multi-dimensional visualization.
- Explore indexed data using Kibana Discover with filters, time ranges, and field selection.
- Build four visualization types in Kibana's Visualize Library: bar chart, pie chart, line chart, and data table.
- Assemble a Kibana dashboard combining multiple visualizations with shared filters and time controls.
- Write Python scripts that query Elasticsearch aggregations and render charts with Matplotlib.
- Produce at least two distinct Python chart types (bar chart and pie chart).
- Export and share Kibana dashboards for collaboration.
- Compare Kibana and Python visualization approaches on interactivity, customization, and sharing.

---

### Prerequisites

- Elasticsearch running on `http://localhost:9200`.
- Kibana running on `http://localhost:5601`.
- Python 3.x with `requests` and `matplotlib` installed:
  ```bash
  pip install requests matplotlib
  ```

---

### Step 1 — Preparing Sample Data

Index a realistic e-commerce dataset with product categories, prices, quantities, and timestamps.
This gives us enough variety for meaningful aggregations.

```bash
curl -s -X POST "http://localhost:9200/store_sales/_bulk?pretty" \
  -H "Content-Type: application/json" -d '
{"index":{}}
{"product":"Laptop","category":"Electronics","price":999.99,"quantity":2,"sale_date":"2025-01-05T10:30:00"}
{"index":{}}
{"product":"Headphones","category":"Electronics","price":59.99,"quantity":10,"sale_date":"2025-01-07T14:00:00"}
{"index":{}}
{"product":"Desk Chair","category":"Furniture","price":249.00,"quantity":4,"sale_date":"2025-01-10T09:15:00"}
{"index":{}}
{"product":"Bookshelf","category":"Furniture","price":129.50,"quantity":3,"sale_date":"2025-01-12T11:45:00"}
{"index":{}}
{"product":"Running Shoes","category":"Apparel","price":89.99,"quantity":8,"sale_date":"2025-01-15T16:20:00"}
{"index":{}}
{"product":"Winter Jacket","category":"Apparel","price":149.99,"quantity":5,"sale_date":"2025-01-18T13:00:00"}
{"index":{}}
{"product":"Blender","category":"Kitchen","price":45.00,"quantity":6,"sale_date":"2025-01-20T08:30:00"}
{"index":{}}
{"product":"Coffee Maker","category":"Kitchen","price":79.99,"quantity":7,"sale_date":"2025-01-22T10:00:00"}
{"index":{}}
{"product":"Novel","category":"Books","price":14.99,"quantity":15,"sale_date":"2025-01-25T12:00:00"}
{"index":{}}
{"product":"Textbook","category":"Books","price":59.99,"quantity":9,"sale_date":"2025-01-28T15:30:00"}
'
```

Verify the data is indexed:

```bash
curl -s "http://localhost:9200/store_sales/_count?pretty"
```

Expected output:

```json
{
  "count": 10,
  "_shards": { "total": 1, "successful": 1, "skipped": 0, "failed": 0 }
}
```

---

### Step 2 — Kibana Discover: Exploring Data

Kibana Discover lets you explore raw documents before building visualizations.

1. Open **Kibana** at `http://localhost:5601`.
2. Navigate to **Discover** from the left sidebar menu.
3. Create a **Data View** (formerly Index Pattern):
   - Go to **Stack Management → Data Views**.
   - Click **Create data view**, enter `store_sales` as the name and index pattern.
   - Select `sale_date` as the time field, then save.
4. Return to **Discover** and select the `store_sales` data view.

**Filtering and field selection:**

| Action | How |
|---|---|
| Set time range | Click the time picker (top right) → set to **Jan 1 2025 – Jan 31 2025** |
| Add a filter | Click **+ Add filter** → field: `category`, operator: `is`, value: `Electronics` |
| Select visible fields | In the field list (left panel), click **+** next to `product`, `price`, `quantity` |
| Search with KQL | Type `price > 100` in the search bar and press Enter |

> **Tip:** Kibana Query Language (KQL) supports `AND`, `OR`, and field-level filters.
> Example: `category: "Electronics" AND price > 50`

---

### Step 3 — Kibana Visualize Library

Navigate to **Visualize Library** from the left sidebar (or **Dashboard → Create visualization**).

#### 3a — Bar Chart: Product Count by Category

This chart answers: *How many products exist in each category?*

1. Click **Create visualization** → select **Lens** (recommended).
2. Drag the `category.keyword` field to the canvas.
3. Kibana auto-generates a bar chart showing document counts per category.
4. Configure the axes:
   - **Horizontal axis:** `category.keyword` (Terms aggregation, top 10).
   - **Vertical axis:** Count of records.
5. Click the chart-type dropdown and confirm **Bar vertical stacked** is selected.
6. Set the title: `Product Count by Category`.
7. Click **Save and return** (or **Save to library** to reuse later).

#### 3b — Pie Chart: Revenue Distribution by Category

This chart answers: *What share of total revenue does each category represent?*

1. Click **Create visualization** → select **Lens**.
2. Switch chart type to **Pie**.
3. Configure:
   - **Slice by:** `category.keyword` (Terms aggregation).
   - **Size by:** Sum of `price` (or Sum of a calculated `revenue` if available).
4. Set the title: `Revenue Distribution by Category`.
5. Save the visualization.

#### 3c — Line Chart: Sales Over Time

This chart answers: *How do sales trend across the month?*

1. Click **Create visualization** → select **Lens**.
2. Switch chart type to **Line**.
3. Configure:
   - **Horizontal axis:** `sale_date` (Date histogram, interval: **3 days**).
   - **Vertical axis:** Sum of `quantity`.
4. Optionally add a **breakdown** by `category.keyword` to see per-category trends.
5. Set the title: `Sales Quantity Over Time`.
6. Save the visualization.

#### 3d — Data Table: Detailed Product View

This table provides a structured numeric summary.

1. Click **Create visualization** → select **Lens**.
2. Switch chart type to **Table**.
3. Configure columns:
   - **Row:** `product.keyword` (Terms aggregation).
   - **Metric 1:** Average of `price`.
   - **Metric 2:** Sum of `quantity`.
4. Set the title: `Product Summary Table`.
5. Save the visualization.

---

### Step 4 — Dashboard Assembly

A dashboard combines multiple visualizations into a single interactive view.

1. Navigate to **Dashboard** from the left sidebar → click **Create dashboard**.
2. Click **Add from library** and select all four saved visualizations:
   - Product Count by Category (bar)
   - Revenue Distribution by Category (pie)
   - Sales Quantity Over Time (line)
   - Product Summary Table (table)
3. **Arrange panels** by dragging and resizing to a logical layout:
   - Top row: bar chart (left), pie chart (right).
   - Middle row: line chart (full width).
   - Bottom row: data table (full width).
4. **Add dashboard-level controls:**
   - Click **Controls** → add a dropdown for `category.keyword` so users can filter all panels at once.
   - Set the time picker to the desired range (Jan 2025).
5. Click **Save** and name it `Store Sales Dashboard`.

> **Key feature:** Clicking a bar segment or pie slice automatically filters all other panels
> on the dashboard. This cross-filtering makes Kibana dashboards interactive by default.

---

### Step 5 — Python Visualization with Matplotlib: Bar Chart

Use the Elasticsearch aggregation API to retrieve bucket data, then plot with Matplotlib.

#### Aggregation Query

```python
import requests
import matplotlib.pyplot as plt

ES_URL = "http://localhost:9200"

# Aggregate product count by category
agg_query = {
    "size": 0,
    "aggs": {
        "by_category": {
            "terms": {
                "field": "category.keyword",
                "size": 10
            }
        }
    }
}

response = requests.get(f"{ES_URL}/store_sales/_search", json=agg_query)
result = response.json()

buckets = result["aggregations"]["by_category"]["buckets"]
categories = [b["key"] for b in buckets]
counts = [b["doc_count"] for b in buckets]

print("Category Counts:")
for cat, cnt in zip(categories, counts):
    print(f"  {cat}: {cnt}")
```

**Expected output from the aggregation:**

```
Category Counts:
  Electronics: 2
  Furniture: 2
  Apparel: 2
  Kitchen: 2
  Books: 2
```

#### Plotting the Bar Chart

```python
fig, ax = plt.subplots(figsize=(8, 5))

bars = ax.bar(categories, counts, color=["#4e79a7", "#f28e2b", "#e15759", "#76b7b2", "#59a14f"])
ax.set_title("Product Count by Category", fontsize=14, fontweight="bold")
ax.set_xlabel("Category", fontsize=12)
ax.set_ylabel("Number of Products", fontsize=12)
ax.set_ylim(0, max(counts) + 1)

# Add value labels on top of each bar
for bar, count in zip(bars, counts):
    ax.text(bar.get_x() + bar.get_width() / 2, bar.get_height() + 0.1,
            str(count), ha="center", va="bottom", fontsize=11)

plt.xticks(rotation=30, ha="right")
plt.tight_layout()
plt.savefig("category_bar_chart.png", dpi=150)
plt.show()
```

---

### Step 6 — Python Visualization with Matplotlib: Pie Chart

A second chart type shows revenue distribution, mirroring the Kibana pie chart.

#### Aggregation Query for Revenue

```python
revenue_query = {
    "size": 0,
    "aggs": {
        "revenue_by_category": {
            "terms": {
                "field": "category.keyword",
                "size": 10
            },
            "aggs": {
                "total_revenue": {
                    "sum": { "field": "price" }
                }
            }
        }
    }
}

response = requests.get(f"{ES_URL}/store_sales/_search", json=revenue_query)
result = response.json()

buckets = result["aggregations"]["revenue_by_category"]["buckets"]
categories = [b["key"] for b in buckets]
revenues = [b["total_revenue"]["value"] for b in buckets]

print("Revenue by Category:")
for cat, rev in zip(categories, revenues):
    print(f"  {cat}: ${rev:,.2f}")
```

**Expected output from the aggregation:**

```
Revenue by Category:
  Electronics: $1,059.98
  Furniture: $378.50
  Apparel: $239.98
  Kitchen: $124.99
  Books: $74.98
```

#### Plotting the Pie Chart

```python
fig, ax = plt.subplots(figsize=(7, 7))

colors = ["#4e79a7", "#f28e2b", "#e15759", "#76b7b2", "#59a14f"]
explode = [0.05] * len(categories)

wedges, texts, autotexts = ax.pie(
    revenues,
    labels=categories,
    autopct="%1.1f%%",
    startangle=140,
    colors=colors,
    explode=explode,
    textprops={"fontsize": 11}
)

ax.set_title("Revenue Distribution by Category", fontsize=14, fontweight="bold")
plt.tight_layout()
plt.savefig("revenue_pie_chart.png", dpi=150)
plt.show()
```

---

### Step 7 — Exporting and Sharing Dashboards

Kibana provides several methods to share dashboards with collaborators.

| Method | How | Best For |
|---|---|---|
| **Share link** | Dashboard → **Share** → **Get link** → copy permalink | Quick sharing within the team |
| **Embed iframe** | Dashboard → **Share** → **Embed code** | Embedding in internal web apps |
| **PDF / PNG export** | Dashboard → **Share** → **PDF Reports** (requires Reporting plugin) | Offline distribution, presentations |
| **Export saved objects** | Stack Management → **Saved Objects** → select dashboard → **Export** | Migrating dashboards between Kibana instances |
| **Import saved objects** | Stack Management → **Saved Objects** → **Import** → upload `.ndjson` | Restoring or sharing dashboard definitions |

> **Note:** PDF/PNG reporting requires an active Kibana Reporting subscription or the
> basic reporting feature (availability depends on your license tier).

---

### Kibana vs Python Visualization — Comparison

| Criteria | Kibana | Python (Matplotlib) |
|---|---|---|
| **Interactivity** | Built-in: click-to-filter, zoom, hover tooltips | Static by default; interactive with Plotly |
| **Setup effort** | Minimal — point-and-click in browser | Requires scripting, library installation |
| **Real-time data** | Auto-refreshes on configurable intervals | Must re-run script or build refresh loop |
| **Customization** | Moderate — constrained to available chart types and options | Full control over every visual element |
| **Sharing** | Links, embeds, PDF export built in | Export to PNG/SVG; share as files or in notebooks |
| **Reproducibility** | Dashboard JSON export; manual recreation otherwise | Script is version-controllable and fully reproducible |
| **Learning curve** | Low for basic charts; moderate for TSVB / Vega | Moderate; requires Python and library knowledge |
| **Best use case** | Live operational dashboards, team monitoring | Custom reports, publication-quality figures, data science |

---

### Troubleshooting Tips

| Problem | Likely Cause | Solution |
|---|---|---|
| No data in Discover | Time range does not cover document timestamps | Adjust time picker to match `sale_date` values |
| Field not available in Visualize | Field is of type `text` instead of `keyword` | Use `category.keyword` (the multi-field mapping) |
| Aggregation returns empty buckets | Index name mismatch or zero documents | Verify with `GET /store_sales/_count` |
| Python script returns `KeyError` | Elasticsearch response contains an error object | Print `result` to inspect; check index name and field names |
| Pie chart shows "Other" slice | Too many unique terms; default `size` is small | Increase `"size"` in the terms aggregation |
| Kibana "field not indexed" warning | Field is not mapped or mapping was not refreshed | Refresh the Data View in Stack Management |
| Matplotlib chart looks cramped | Labels overlap due to long category names | Add `plt.tight_layout()` and rotate x-tick labels |
| Dashboard filters not affecting all panels | Panels use different data views or time fields | Ensure all visualizations use the same `store_sales` data view |

---

### Cleanup

Remove the sample index when you are finished with the lab:

```bash
curl -s -X DELETE "http://localhost:9200/store_sales?pretty"
```

---

### Summary

In this lab you:

1. Indexed a multi-field e-commerce dataset into Elasticsearch.
2. Explored raw documents in Kibana Discover using KQL, filters, and field selection.
3. Created four Kibana visualizations — bar, pie, line, and data table — in the Visualize Library.
4. Assembled an interactive dashboard with cross-filtering and time controls.
5. Wrote Python scripts that query Elasticsearch aggregations and produce bar and pie charts with Matplotlib.
6. Compared Kibana and Python visualization workflows across interactivity, customization, and sharing.

These two approaches are complementary: use Kibana for live, interactive monitoring and Python for
reproducible, publication-ready analysis.
