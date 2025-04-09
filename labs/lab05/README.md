## Task 5: Defining Mappings and Creating Custom Analyzers

**Objectives:**
- Create custom index mappings to control text analysis.
- Configure multi‑fields (for full‑text search plus keyword aggregations).
- Define and test both built‑in analyzers and a custom analyzer using Kibana’s _analyze API and Python.

**Lab Steps:**

1. **Custom Mapping via Kibana:**  
   Create an index `articles` with a custom analyzer:
   ```json
   PUT /articles
   {
     "settings": {
       "analysis": {
         "analyzer": {
           "custom_content": {
             "type": "custom",
             "tokenizer": "standard",
             "filter": ["lowercase", "stop", "porter_stem"]
           }
         }
       }
     },
     "mappings": {
       "properties": {
         "title": {
           "type": "text",
           "fields": {
             "raw": { "type": "keyword" }
           }
         },
         "content": {
           "type": "text",
           "analyzer": "custom_content"
         }
       }
     }
   }
   ```
2. **Testing the Analyzer Using Kibana:**  
   Test the analyzer:
   ```json
   GET /articles/_analyze
   {
     "analyzer": "custom_content",
     "text": "Elasticsearch is a highly scalable search engine."
   }
   ```
3. **Python Tester Script:**  
   Automate analyzer testing with Python:
   ```python
   analyze_payload = {
       "analyzer": "custom_content",
       "text": "Elasticsearch is a highly scalable search engine."
   }
   analysis_result = requests.get("http://localhost:9200/articles/_analyze", json=analyze_payload).json()
   print("Analyzed tokens:", analysis_result)
   ```
4. **Reflection:**  
   Summarize the tokenization process and how different analyzers affect search results.
