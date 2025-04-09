## Task 4: Data Aggregations for Analytics

**Objectives:**
- Use metric and bucket aggregations to analyze indexed data.
- Visualize aggregated data (counts, averages) via raw queries and Python.

**Lab Steps:**

1. **Metric Aggregation via Kibana:**  
   Find the average product price:
   ```json
   GET /products/_search
   {
     "size": 0,
     "aggs": {
       "average_price": { "avg": { "field": "price" } }
     }
   }
   ```
2. **Bucket Aggregation:**  
   Group documents by `name`:
   ```json
   GET /products/_search
   {
     "size": 0,
     "aggs": {
       "name_groups": {
         "terms": { "field": "name.keyword" }
       }
     }
   }
   ```
3. **Python Aggregation Query:**  
   Use Python to retrieve aggregation results:
   ```python
   agg_query = {
     "size": 0,
     "aggs": {
       "price_stats": { "avg": { "field": "price" } }
     }
   }
   agg_resp = requests.get(f"{es_url}/_search", json=agg_query).json()
   print("Aggregation response:", json.dumps(agg_resp['aggregations'], indent=2))
   ```
4. **Reflection:**  
   Document what metrics you obtained and discuss how such aggregations can support real‑time analytics.
