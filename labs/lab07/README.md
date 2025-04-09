## Task 7: Multi-Node Cluster Configuration and Monitoring

**Objectives:**
- Deploy and configure a multi‑node cluster for scalability.
- Configure shard allocation, replicas, and node roles.
- Use both raw API calls (Kibana) and Python scripts to monitor cluster health.

**Lab Steps:**

1. **Multi-Node Setup with Docker Compose:**  
   Create a `docker-compose.yml` file with at least three nodes:
   ```yaml
   version: '3'
   services:
     es1:
       image: docker.elastic.co/elasticsearch/elasticsearch:8.6.0
       environment:
         - node.name=es1
         - cluster.name=es_cluster
         - discovery.seed_hosts=es2,es3
         - cluster.initial_master_nodes=es1,es2,es3
       ports:
         - 9200:9200
     es2:
       image: docker.elastic.co/elasticsearch/elasticsearch:8.6.0
       environment:
         - node.name=es2
         - cluster.name=es_cluster
         - discovery.seed_hosts=es1,es3
         - cluster.initial_master_nodes=es1,es2,es3
     es3:
       image: docker.elastic.co/elasticsearch/elasticsearch:8.6.0
       environment:
         - node.name=es3
         - cluster.name=es_cluster
         - discovery.seed_hosts=es1,es2
         - cluster.initial_master_nodes=es1,es2,es3
   ```
   Bring up the cluster:
   ```bash
   docker-compose up -d
   ```
2. **Cluster Monitoring:**  
   Check cluster health in Kibana:
   ```http
   GET /_cluster/health?pretty
   ```
   Also, retrieve node stats using:
   ```http
   GET /_nodes/stats
   ```
3. **Python Monitoring Script:**  
   Write a Python script to poll cluster status periodically:
   ```python
   import time, requests, json

   while True:
       health = requests.get("http://localhost:9200/_cluster/health").json()
       print("Cluster status:", health['status'])
       time.sleep(10)
   ```
4. **Reflection:**  
   Discuss how shard allocation, nodes, and replica settings enhance reliability. Note any observed improvements.
