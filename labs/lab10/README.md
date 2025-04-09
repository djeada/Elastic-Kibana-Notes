## Task 10: Securing Your Elasticsearch Cluster

**Objectives:**
- Implement Elasticsearch security features (authentication, TLS).
- Configure roles, users, and permissions.
- Secure node‑to‑node communication.
- Validate security features with both Kibana and Python (using secure HTTPS requests).

**Lab Steps:**

1. **Enable Security:**  
   For Elasticsearch 8 (where security is enabled by default), check the security features via Kibana’s Security UI or using APIs:
   ```http
   GET /_security/_authenticate?pretty
   ```
2. **Create Users and Roles:**  
   Create a role that allows read access:
   ```http
   POST /_security/role/app_reader
   {
     "cluster": [],
     "indices": [
       {
         "names": [ "products", "articles" ],
         "privileges": [ "read" ]
       }
     ]
   }
   ```
   Create a user and assign the role:
   ```http
   POST /_security/user/john
   {
     "password" : "Passw0rd!",
     "roles" : [ "app_reader" ],
     "full_name" : "John Doe"
   }
   ```
3. **TLS/SSL Configuration:**  
   Use Elasticsearch’s built‑in tools (like `elasticsearch-certutil`) to generate certificates and configure nodes accordingly.
4. **Python Secure Request Example:**  
   Test secure endpoints with Python’s `requests` library by verifying certificates:
   ```python
   secure_resp = requests.get("https://localhost:9200/_cluster/health", verify="path/to/ca.crt")
   print(secure_resp.json())
   ```
5. **Reflection:**  
   Document your security configuration decisions and troubleshooting steps for connectivity issues.
