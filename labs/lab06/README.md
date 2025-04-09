## Task 6: Data Ingestion with Python, Logstash, and Beats

**Objectives:**
- Ingest data into Elasticsearch using both Python scripts and ingestion tools.
- Transform and enrich data during ingestion.
- Compare feeding data manually with API calls versus using an ingestion pipeline.

**Lab Steps:**

1. **Data Ingestion via Python:**  
   Prepare a CSV file (or JSON data) and write a Python script to load data into Elasticsearch:
   ```python
   import csv
   import requests
   import json

   url = "http://localhost:9200/products/_doc/"
   with open("products.csv", "r") as csvfile:
       reader = csv.DictReader(csvfile)
       for row in reader:
           response = requests.post(url, json=row)
           print(response.json())
   ```
2. **Using Logstash (Alternative):**  
   Create a Logstash pipeline file `pipeline.conf` to read a log file, parse it, and output to Elasticsearch:
   ```conf
   input {
     file {
       path => "/path/to/sample.log"
       start_position => "beginning"
     }
   }
   filter {
     grok {
       match => { "message" => "%{COMMONAPACHELOG}" }
     }
   }
   output {
     elasticsearch {
       hosts => ["localhost:9200"]
       index => "weblogs"
     }
   }
   ```
   Run Logstash with:
   ```bash
   bin/logstash -f pipeline.conf
   ```
3. **Using Filebeat:**  
   Install Filebeat and configure it to ship log files to your Elasticsearch cluster.
4. **Reflection:**  
   Compare results from Python ingestion versus Logstash/Filebeat pipelines. Document scenarios when each method is preferable.
