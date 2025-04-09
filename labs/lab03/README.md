## Task 3: Basic Searching with Query DSL

**Objectives:**
- Construct search queries using Query DSL via Kibana.
- Learn the syntax of match, term, and bool queries.
- Understand the nuances of full‑text search versus exact matching.

**Lab Steps:**

1. **Match Query via Kibana:**  
   Execute the following in Dev Tools to search for products with “t‑shirt”:
   ```json
   GET /products/_search
   {
     "query": {
       "match": { "name": "t-shirt" }
     }
   }
   ```
2. **Boolean Query:**  
   Combine multiple conditions, for example:
   ```json
   GET /products/_search
   {
     "query": {
       "bool": {
         "must": [
           { "match": { "name": "Elastic" } },
           { "term": { "price": 17.99 } }
         ]
       }
     }
   }
   ```
3. **Python for Searching:**  
   Use Python to execute a search query:
   ```python
   query = {
     "query": {
       "match": {
         "name": "Elastic"
       }
     }
   }
   search_resp = requests.get(f"{es_url}/_search", json=query).json()
   print("Search results:", json.dumps(search_resp, indent=2))
   ```
4. **Analysis:**  
   Note the returned documents and score details. Write a brief explanation on how the Query DSL refines search results.

