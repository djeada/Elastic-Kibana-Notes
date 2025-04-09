## Task 1: Environment Setup, Cluster Basics, and Verification

**Objectives:**
- Install Elasticsearch and Kibana (locally via Docker).
- Understand cluster fundamentals: nodes, shards, replicas.
- Verify installation via raw REST calls (Kibana) and Python scripts.

**Lab Steps:**

1. **Installation Using Docker:**  
   Run the following commands in your terminal:
   ```bash
   docker run -d --name elasticsearch -p 9200:9200 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:8.6.0
   docker run -d --name kibana -p 5601:5601 --link elasticsearch:elasticsearch docker.elastic.co/kibana/kibana:8.6.0
   ```
2. **Cluster Verification via Kibana:**  
   Open Kibana’s Dev Tools and execute:
   ```http
   GET /
   ```
   This displays cluster information (name, version, cluster UUID).
3. **Verification via Python Script:**  
   Use a simple Python script (with the `requests` library) to query cluster health:
   ```python
   import requests
   response = requests.get("http://localhost:9200/_cluster/health?pretty")
   print(response.json())
   ```
4. **Concept Discussion:**  
   In your lab journal, describe the roles of nodes, shards, and replicas, and why distributed architecture is beneficial.



