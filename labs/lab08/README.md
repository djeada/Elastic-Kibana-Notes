## Task 8: Advanced Querying and Query Profiling

**Objectives:**
- Build complex queries combining multiple query types (function score, boosts, filters).
- Use the `profile` flag to analyze query performance.
- Leverage Python to iterate and refine queries based on performance data.

**Lab Steps:**

1. **Complex Query Construction (Kibana):**  
   Construct a query that uses function score, bool queries, and boosting:
   ```json
   GET /products/_search
   {
     "query": {
       "function_score": {
         "query": {
           "bool": {
             "must": [{ "match": { "name": "Elastic" } }],
             "filter": [{ "range": { "price": { "gte": 10, "lte": 50 } } }]
           }
         },
         "boost": 5,
         "functions": [
           { "random_score": {}, "weight": 2 }
         ],
         "score_mode": "sum",
         "boost_mode": "multiply"
       }
     }
   }
   ```
2. **Query Profiling:**  
   Enable the profiler:
   ```json
   GET /products/_search
   {
     "profile": true,
     "query": { ... }  // Use your existing complex query here.
   }
   ```
   Analyze the breakdown of query execution.
3. **Python Integration:**  
   Use Python to repeatedly run and adjust queries:
   ```python
   query = {
     "profile": True,
     "query": {
       "bool": {
         "must": [{"match": {"name": "Elastic"}}],
         "filter": [{"range": {"price": {"gte": 10, "lte": 50}}}]
       }
     }
   }
   prof_resp = requests.get(f"{es_url}/_search", json=query).json()
   print(json.dumps(prof_resp['profile'], indent=2))
   ```
4. **Reflection:**  
   Record which components of your query are most time‑consuming and propose adjustments to improve speed.

