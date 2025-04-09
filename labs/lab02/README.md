## Task 2: Index Creation, CRUD Operations, and Data Validation

**Objectives:**
- Create an index with custom settings and mappings.
- Perform document create, read, update, and delete (CRUD) operations.
- Validate operations using both Kibana’s Dev Tools and Python.

**Lab Steps:**

1. **Manual Index Creation via Kibana:**  
   In Dev Tools, create an index named `products` with custom settings and mappings:
   ```json
   PUT /products
   {
     "settings": {
       "number_of_shards": 1,
       "number_of_replicas": 0
     },
     "mappings": {
       "properties": {
         "name": { "type": "text" },
         "price": { "type": "float" }
       }
     }
   }
   ```
2. **Document CRUD Operations in Kibana:**  
   - **Create (Index) Document:**
     ```json
     POST /products/_doc
     {
       "name": "Elastic T-Shirt",
       "price": 19.99
     }
     ```
   - **Read Document:**  
     Retrieve the document using its ID:
     ```http
     GET /products/_doc/<document_id>
     ```
   - **Update Document:**  
     Update a field (for example, change the price):
     ```json
     POST /products/_doc/<document_id>/_update
     {
       "doc": { "price": 17.99 }
     }
     ```
   - **Delete Document:**  
     Remove the document:
     ```http
     DELETE /products/_doc/<document_id>
     ```
3. **CRUD Operations Using Python:**  
   Write a Python script to perform similar operations with the Elasticsearch REST API:
   ```python
   import requests
   import json

   es_url = "http://localhost:9200/products/_doc"
   
   # Create a document
   doc = {"name": "Elastic T-Shirt", "price": 19.99}
   create_resp = requests.post(es_url, json=doc).json()
   doc_id = create_resp.get('_id')
   print("Document created with ID:", doc_id)
   
   # Read the document
   read_resp = requests.get(f"{es_url}/{doc_id}").json()
   print("Document content:", json.dumps(read_resp, indent=2))
   
   # Update the document
   update_payload = { "doc": { "price": 17.99 } }
   update_resp = requests.post(f"{es_url}/{doc_id}/_update", json=update_payload).json()
   print("Update response:", json.dumps(update_resp, indent=2))
   
   # Delete the document
   delete_resp = requests.delete(f"{es_url}/{doc_id}").json()
   print("Delete response:", json.dumps(delete_resp, indent=2))
   ```
4. **Reflection:**  
   Write down differences observed between raw API calls (via Kibana) and using Python. Discuss the benefits of automation.
