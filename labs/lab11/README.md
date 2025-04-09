## Task 11: Dashboarding with Kibana and Python Data Visualization

**Objectives:**
- Visualize data in real‑time using Kibana’s dashboards.
- Create visualizations (bar charts, pie charts, maps) using raw data.
- Use Python (e.g., with Matplotlib or Plotly) to extract data via Elasticsearch queries and build custom visualizations.

**Lab Steps:**

1. **Kibana Discover and Visualize:**  
   - Use the “Discover” tab in Kibana to explore your indexed data.
   - Create visualizations by going to “Visualize Library” and build charts (e.g., a histogram of product prices).
2. **Dashboard Assembly:**  
   - Combine multiple visualizations into a comprehensive dashboard.
3. **Python Visualization:**  
   - Write a Python script that queries Elasticsearch for aggregated data (using the aggregation API).
   - Use a library such as Matplotlib:
     ```python
     import matplotlib.pyplot as plt
     import requests

     agg_query = {
       "size": 0,
       "aggs": {
         "by_name": {
           "terms": { "field": "name.keyword" }
         }
       }
     }
     resp = requests.get("http://localhost:9200/products/_search", json=agg_query).json()
     buckets = resp['aggregations']['by_name']['buckets']
     names = [bucket['key'] for bucket in buckets]
     counts = [bucket['doc_count'] for bucket in buckets]

     plt.bar(names, counts)
     plt.title("Product Count by Name")
     plt.xlabel("Product Name")
     plt.ylabel("Count")
     plt.xticks(rotation=45)
     plt.tight_layout()
     plt.show()
     ```
4. **Reflection:**  
   Compare and contrast how Kibana and Python visualizations offer flexibility and customization.
