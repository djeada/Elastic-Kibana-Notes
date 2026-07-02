## Task 10: Securing Your Elasticsearch Cluster

### Security Architecture

```
┌──────────┐      TLS/SSL       ┌────────────────┐     ┌────────────────┐     ┌───────────────┐
│  Client  │ ────────────────►  │ Authentication │ ──► │ Authorization  │ ──► │ Elasticsearch │
│ (curl /  │  encrypted channel │ Who are you?   │     │ What can you   │     │   Cluster     │
│  Python) │ ◄────────────────  │ (credentials / │     │ access? (roles │     │  (data layer) │
└──────────┘                    │  API key)      │     │  / privileges) │     └───────────────┘
                                └────────────────┘     └────────────────┘
```

**Objectives:**

- Understand the Elasticsearch 8+ security model.
- Verify active security with the `_security/_authenticate` API.
- Explore built-in roles and create custom roles with fine-grained privileges.
- Create users, assign roles, and test access-control enforcement.
- Load sample data so permissions can be tested against real indices.
- Configure TLS/SSL for the transport and HTTP layers.
- Use API keys as an alternative to basic authentication.
- Perform secure HTTPS requests from Python with certificate verification.
- Diagnose common security errors.


### Security in Elasticsearch 8+

Starting with Elasticsearch 8.0, security is **enabled by default**. A fresh secured cluster automatically:

1. Generates a Certificate Authority (CA) and node certificates.
2. Enables TLS on both the **transport layer** and the **HTTP layer**.
3. Creates the built-in `elastic` superuser.
4. Prints enrollment tokens for Kibana and additional nodes.

The security flow has two main checks:

| Security Step | Question Answered | Example |
|---------------|-------------------|---------|
| **Authentication** | Who is making the request? | `elastic`, `john`, `jane`, or an API key |
| **Authorization** | What is that identity allowed to do? | Read `products`, write `products`, manage users, monitor cluster health |

> **Important:** Authentication proves identity. Authorization decides permissions. A user can authenticate successfully and still receive `403 Forbidden` if their role does not allow the requested action.

> **Lab note:** Earlier Elasticsearch labs often disable security with `xpack.security.enabled=false`. This task requires a secured Elasticsearch 8.x cluster. If your existing container was started with security disabled, create a separate secured container for this task.

### Prerequisite: Start a Secured Local Cluster

Use a known training password so the examples are repeatable.

> Do **not** reuse this password in production. It is only for a local lab.

```bash
# Optional: remove old unsecured lab containers if they conflict on ports.
docker rm -f elasticsearch kibana 2>/dev/null || true

# Create a Docker network for Elasticsearch and Kibana.
docker network create elastic 2>/dev/null || true

# Start Elasticsearch with security enabled.
docker run -d \
  --name elasticsearch \
  --net elastic \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "ELASTIC_PASSWORD=Elastic_Training_123!" \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  docker.elastic.co/elasticsearch/elasticsearch:8.6.0
```

Copy the generated HTTP CA certificate from the container:

```bash
docker cp elasticsearch:/usr/share/elasticsearch/config/certs/http_ca.crt ./http_ca.crt
```

Verify Elasticsearch over HTTPS:

```bash
curl -s --cacert ./http_ca.crt \
  -u elastic:Elastic_Training_123! \
  "https://localhost:9200" | python3 -m json.tool
```

**Expected Output:**

```json
{
  "name": "...",
  "cluster_name": "docker-cluster",
  "version": {
    "number": "8.6.0"
  },
  "tagline": "You Know, for Search"
}
```

If you also want Kibana, create an enrollment token and start Kibana:

```bash
docker exec -it elasticsearch \
  bin/elasticsearch-create-enrollment-token -s kibana
```

Then run Kibana:

```bash
docker run -d \
  --name kibana \
  --net elastic \
  -p 5601:5601 \
  docker.elastic.co/kibana/kibana:8.6.0
```

Open Kibana at `http://localhost:5601`, paste the enrollment token, and log in as:

```text
Username: elastic
Password: Elastic_Training_123!
```

### Step 0: Create Sample Indices and Load Data

Before testing roles and permissions, create data that users can read or write. Run these commands as the `elastic` superuser.

#### 0a — Create the `products` Index

```http
PUT /products
{
  "mappings": {
    "properties": {
      "name":        { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "category":    { "type": "keyword" },
      "price":       { "type": "double" },
      "in_stock":    { "type": "boolean" },
      "rating":      { "type": "float" },
      "created_at":  { "type": "date" },
      "description": { "type": "text" }
    }
  }
}
```

**Expected Output:**

```json
{ "acknowledged": true, "shards_acknowledged": true, "index": "products" }
```

#### 0b — Create the `articles` Index

```http
PUT /articles
{
  "mappings": {
    "properties": {
      "title":       { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "author":      { "type": "keyword" },
      "topic":       { "type": "keyword" },
      "published":   { "type": "boolean" },
      "published_at": { "type": "date" },
      "body":        { "type": "text" }
    }
  }
}
```

**Expected Output:**

```json
{ "acknowledged": true, "shards_acknowledged": true, "index": "articles" }
```

#### 0c — Bulk-Load Product Documents

```http
POST /products/_bulk
{ "index": { "_id": "p1" } }
{ "name": "Elastic Widget", "category": "Electronics", "price": 29.99, "in_stock": true, "rating": 4.7, "created_at": "2026-01-10", "description": "Compact searchable device for demo applications" }
{ "index": { "_id": "p2" } }
{ "name": "Secure Sensor", "category": "Electronics", "price": 79.99, "in_stock": true, "rating": 4.4, "created_at": "2026-01-15", "description": "IoT sensor with encrypted telemetry" }
{ "index": { "_id": "p3" } }
{ "name": "Data Notebook", "category": "Office", "price": 12.50, "in_stock": true, "rating": 4.1, "created_at": "2026-02-01", "description": "Notebook for recording search experiments" }
{ "index": { "_id": "p4" } }
{ "name": "Legacy Cable", "category": "Discontinued", "price": 4.99, "in_stock": false, "rating": 2.3, "created_at": "2025-12-20", "description": "Older cable retained for permission testing" }
```

**Expected Output:**

```json
{
  "errors": false,
  "items": [
    { "index": { "_id": "p1", "status": 201 } },
    { "index": { "_id": "p2", "status": 201 } },
    { "index": { "_id": "p3", "status": 201 } },
    { "index": { "_id": "p4", "status": 201 } }
  ]
}
```

#### 0d — Bulk-Load Article Documents

```http
POST /articles/_bulk
{ "index": { "_id": "a1" } }
{ "title": "Securing Search Applications", "author": "training-team", "topic": "security", "published": true, "published_at": "2026-01-05", "body": "Use TLS, roles, and API keys to secure Elasticsearch applications." }
{ "index": { "_id": "a2" } }
{ "title": "Role-Based Access Control Basics", "author": "training-team", "topic": "security", "published": true, "published_at": "2026-01-18", "body": "RBAC maps users to roles and roles to cluster or index privileges." }
{ "index": { "_id": "a3" } }
{ "title": "Draft Migration Plan", "author": "platform-team", "topic": "operations", "published": false, "published_at": "2026-02-03", "body": "Internal draft used to test restricted write access." }
```

**Expected Output:**

```json
{
  "errors": false
}
```

#### 0e — Refresh and Confirm the Data

```http
POST /products,articles/_refresh

GET /products/_search
{
  "size": 3,
  "query": { "match_all": {} },
  "_source": ["name", "category", "price", "in_stock"]
}
```

**Expected Output (trimmed):**

```json
{
  "hits": {
    "total": { "value": 4, "relation": "eq" },
    "hits": [
      { "_id": "p1", "_source": { "name": "Elastic Widget", "category": "Electronics", "price": 29.99, "in_stock": true } },
      { "_id": "p2", "_source": { "name": "Secure Sensor", "category": "Electronics", "price": 79.99, "in_stock": true } },
      { "_id": "p3", "_source": { "name": "Data Notebook", "category": "Office", "price": 12.5, "in_stock": true } }
    ]
  }
}
```

### Step 1: Verify Authentication

The `_security/_authenticate` API shows which user or API key is currently authenticated.

In Kibana Dev Tools:

```http
GET /_security/_authenticate
```

With `curl`:

```bash
curl -s --cacert ./http_ca.crt \
  -u elastic:Elastic_Training_123! \
  "https://localhost:9200/_security/_authenticate" | python3 -m json.tool
```

**Expected Output:**

```json
{
  "username": "elastic",
  "roles": ["superuser"],
  "full_name": null,
  "email": null,
  "enabled": true,
  "authentication_realm": { "name": "reserved", "type": "reserved" },
  "lookup_realm": { "name": "reserved", "type": "reserved" },
  "authentication_type": "realm"
}
```

> **Important:** A `401 Unauthorized` response means authentication failed. Common causes are a wrong password, missing `-u user:password`, or using `http://` instead of `https://`.

### Step 2: Built-in Roles Overview

Elasticsearch includes reserved roles for common system and user tasks.

| Role | Purpose |
|------|---------|
| `superuser` | Full access to every cluster operation and index. Use sparingly. |
| `kibana_system` | Internal role for the Kibana server process. Do not assign to normal users. |
| `logstash_system` | Internal role for Logstash monitoring. |
| `beats_system` | Internal role for Beats monitoring. |
| `remote_monitoring_agent` | Writes monitoring data to a remote cluster. |
| `monitoring_user` | Read-only access to monitoring features. |
| `ingest_admin` | Manages ingest pipelines. |
| `viewer` | Read-only access to Kibana features. |
| `editor` | Read-write access to Kibana features. |

List all roles:

```http
GET /_security/role?pretty
```

**Expected Output (excerpt):**

```json
{
  "superuser": {
    "cluster": ["all"],
    "indices": [{ "names": ["*"], "privileges": ["all"] }],
    "metadata": { "_reserved": true }
  }
}
```

> **Tip:** Built-in roles are useful, but application access is usually safer with custom roles that follow the principle of least privilege.

### Step 3: Create Custom Roles

Custom roles define exactly which cluster and index actions are allowed.

#### 3a — Create a Read-Only Application Role

This role can monitor cluster health and read from `products` and `articles`, but it cannot write data.

```http
POST /_security/role/app_reader
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["products", "articles"],
      "privileges": ["read", "view_index_metadata"]
    }
  ]
}
```

**Expected Output:**

```json
{ "role": { "created": true } }
```

#### 3b — Create a Read-Write Application Role

This role can read and write `products`, but it does not have access to `articles`.

```http
POST /_security/role/app_writer
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["products"],
      "privileges": ["read", "write", "create_index", "view_index_metadata"]
    }
  ]
}
```

**Expected Output:**

```json
{ "role": { "created": true } }
```

#### 3c — Inspect the Custom Roles

```http
GET /_security/role/app_reader,app_writer
```

**Expected Output (trimmed):**

```json
{
  "app_reader": {
    "cluster": ["monitor"],
    "indices": [{ "names": ["products", "articles"], "privileges": ["read", "view_index_metadata"] }]
  },
  "app_writer": {
    "cluster": ["monitor"],
    "indices": [{ "names": ["products"], "privileges": ["read", "write", "create_index", "view_index_metadata"] }]
  }
}
```

### Step 4: Create Users and Assign Roles

A user can have one role or many roles. In this lab, `john` is read-only and `jane` can write to `products`.

#### 4a — Reader User

```http
POST /_security/user/john
{
  "password": "J0hn_S3cure!",
  "roles": ["app_reader"],
  "full_name": "John Doe",
  "email": "john.doe@example.com"
}
```

**Expected Output:**

```json
{ "created": true }
```

#### 4b — Writer User

```http
POST /_security/user/jane
{
  "password": "J4ne_S3cure!",
  "roles": ["app_writer"],
  "full_name": "Jane Smith",
  "email": "jane.smith@example.com"
}
```

**Expected Output:**

```json
{ "created": true }
```

#### 4c — Verify Reader Authentication

```bash
curl -s -u john:J0hn_S3cure! --cacert ./http_ca.crt \
  "https://localhost:9200/_security/_authenticate" | python3 -m json.tool
```

**Expected Output:**

```json
{
  "username": "john",
  "roles": ["app_reader"],
  "full_name": "John Doe",
  "enabled": true,
  "authentication_realm": { "name": "native", "type": "native" },
  "authentication_type": "realm"
}
```

### Step 5: Test Access Control

This step proves that authentication and authorization are separate. Both `john` and `jane` can log in, but their roles allow different actions.

#### 5a — Read `products` as the Read-Only User

```bash
curl -s -u john:J0hn_S3cure! --cacert ./http_ca.crt \
  "https://localhost:9200/products/_search?size=2" | python3 -m json.tool
```

**Expected Output (200 OK, trimmed):**

```json
{
  "hits": {
    "total": { "value": 4, "relation": "eq" },
    "hits": [
      { "_id": "p1", "_source": { "name": "Elastic Widget", "category": "Electronics" } }
    ]
  }
}
```

#### 5b — Attempt a Write as the Read-Only User

This request should fail because `john` has `read`, not `write`.

```bash
curl -s -u john:J0hn_S3cure! --cacert ./http_ca.crt \
  -X POST "https://localhost:9200/products/_doc" \
  -H "Content-Type: application/json" -d '{
    "name": "Unauthorized Widget",
    "category": "Electronics",
    "price": 9.99,
    "in_stock": true
  }' | python3 -m json.tool
```

**Expected Output (403 Forbidden):**

```json
{
  "error": {
    "type": "security_exception",
    "reason": "action [indices:data/write/index] is unauthorized for user [john] with effective roles [app_reader]"
  },
  "status": 403
}
```

#### 5c — Write to `products` as the Writer User

This request succeeds because `jane` has `write` privileges on the `products` index.

```bash
curl -s -u jane:J4ne_S3cure! --cacert ./http_ca.crt \
  -X POST "https://localhost:9200/products/_doc" \
  -H "Content-Type: application/json" -d '{
    "name": "Authorized Widget",
    "category": "Electronics",
    "price": 19.99,
    "in_stock": true,
    "rating": 4.2,
    "created_at": "2026-02-10",
    "description": "Document created by a user with write privileges"
  }' | python3 -m json.tool
```

**Expected Output (201 Created):**

```json
{
  "result": "created",
  "_index": "products",
  "_version": 1,
  "_shards": { "total": 2, "successful": 1, "failed": 0 }
}
```

#### 5d — Attempt to Read `articles` as the Writer User

This request should fail because `jane` only has access to `products`.

```bash
curl -s -u jane:J4ne_S3cure! --cacert ./http_ca.crt \
  "https://localhost:9200/articles/_search?size=1" | python3 -m json.tool
```

**Expected Output (403 Forbidden):**

```json
{
  "error": {
    "type": "security_exception",
    "reason": "action [indices:data/read/search] is unauthorized for user [jane] with effective roles [app_writer]"
  },
  "status": 403
}
```

> **Observation:** `app_writer` is not more powerful everywhere. It is only more powerful on `products`. This is the principle of least privilege in practice.

### Step 6: TLS/SSL Configuration

TLS protects traffic from being read or modified in transit.

| Layer | Setting Prefix | Purpose |
|-------|----------------|---------|
| Transport | `xpack.security.transport.ssl.*` | Encrypts node-to-node traffic inside the cluster. |
| HTTP | `xpack.security.http.ssl.*` | Encrypts client-to-node traffic from curl, Kibana, Python, and applications. |

Elasticsearch 8.x can auto-configure TLS for a fresh cluster. The manual steps below are useful when you need to generate and manage your own certificates.

#### 6a — Generate Certificates

Run these commands from the Elasticsearch installation directory:

```bash
# Create a Certificate Authority.
bin/elasticsearch-certutil ca \
  --out config/certs/elastic-stack-ca.p12 \
  --pass ""

# Create a node certificate signed by that CA.
bin/elasticsearch-certutil cert \
  --ca config/certs/elastic-stack-ca.p12 \
  --ca-pass "" \
  --out config/certs/elastic-certificates.p12 \
  --pass ""
```

**Expected Output:**

```text
CA certificate and key have been generated successfully.
```

#### 6b — Configure `elasticsearch.yml`

```yaml
xpack.security.enabled: true

# Transport layer: node-to-node encryption.
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12

# HTTP layer: client-to-node encryption.
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.http.ssl.truststore.path: certs/elastic-certificates.p12
```

Restart the node and verify HTTPS access:

```bash
sudo systemctl restart elasticsearch

curl -s --cacert config/certs/http_ca.crt \
  -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200" | python3 -m json.tool
```

**Expected Output:**

```json
{
  "name": "node-1",
  "cluster_name": "my-cluster",
  "version": { "number": "8.x.x" },
  "tagline": "You Know, for Search"
}
```

> **Production note:** Use certificates with proper hostnames, protect private keys, rotate certificates, and avoid disabling certificate verification.

### Step 7: API Key Authentication

API keys are useful for services, scheduled jobs, CI/CD pipelines, and applications that should not store a user's password.

Create an API key that can only read the `products` index:

```http
POST /_security/api_key
{
  "name": "my-app-key",
  "expiration": "30d",
  "role_descriptors": {
    "app_reader_scope": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["products"],
          "privileges": ["read", "view_index_metadata"]
        }
      ]
    }
  }
}
```

**Expected Output:**

```json
{
  "id": "VuaCfGcBCdbkQm-e5aOx",
  "name": "my-app-key",
  "expiration": 1751328000000,
  "api_key": "ui2lp2axTNmsyakw9tvNnw",
  "encoded": "VnVhQ2ZHY0JDZGJrUW0tZTVhT3g6dWkybHAyYXhUTm1zeWFrdzl0dk5udw=="
}
```

> **Important:** Copy the `encoded` value immediately. It is the value you send in the `Authorization: ApiKey ...` header.

Use the API key to read `products`:

```bash
curl -s \
  -H "Authorization: ApiKey <encoded-api-key>" \
  --cacert ./http_ca.crt \
  "https://localhost:9200/products/_search?size=1" | python3 -m json.tool
```

**Expected Output:**

```json
{
  "took": 3,
  "timed_out": false,
  "hits": {
    "total": { "value": 5, "relation": "eq" },
    "hits": [
      {
        "_index": "products",
        "_id": "p1",
        "_score": 1.0,
        "_source": {
          "name": "Elastic Widget",
          "category": "Electronics",
          "price": 29.99
        }
      }
    ]
  }
}
```

Try the same API key against `articles`:

```bash
curl -s \
  -H "Authorization: ApiKey <encoded-api-key>" \
  --cacert ./http_ca.crt \
  "https://localhost:9200/articles/_search?size=1" | python3 -m json.tool
```

**Expected Output (403 Forbidden):**

```json
{
  "error": {
    "type": "security_exception",
    "reason": "action [indices:data/read/search] is unauthorized for API key"
  },
  "status": 403
}
```

### Step 8: Python Secure Request Examples

Python clients should verify the Elasticsearch CA certificate instead of disabling TLS verification.

#### Basic Auth with TLS

```python
import requests

CA_CERT = "./http_ca.crt"
ES_URL = "https://localhost:9200"

resp = requests.get(
    f"{ES_URL}/_cluster/health",
    auth=("elastic", "Elastic_Training_123!"),
    verify=CA_CERT,
    timeout=10,
)
resp.raise_for_status()

print(resp.json())
```

**Expected Output:**

```json
{
  "cluster_name": "docker-cluster",
  "status": "green",
  "number_of_nodes": 1,
  "active_primary_shards": 2,
  "unassigned_shards": 0
}
```

#### API Key Auth with TLS

```python
import requests

CA_CERT = "./http_ca.crt"
ES_URL = "https://localhost:9200"
API_KEY = "<encoded-api-key>"

resp = requests.get(
    f"{ES_URL}/products/_search",
    headers={"Authorization": f"ApiKey {API_KEY}"},
    verify=CA_CERT,
    params={"size": 1},
    timeout=10,
)
resp.raise_for_status()

print(resp.json())
```

**Expected Output:**

```json
{
  "took": 2,
  "hits": {
    "total": { "value": 5, "relation": "eq" },
    "hits": [
      {
        "_index": "products",
        "_source": {
          "name": "Elastic Widget",
          "price": 29.99
        }
      }
    ]
  }
}
```

#### Handling Authorization Errors in Python

```python
import requests

CA_CERT = "./http_ca.crt"
ES_URL = "https://localhost:9200"

resp = requests.post(
    f"{ES_URL}/products/_doc",
    auth=("john", "J0hn_S3cure!"),
    verify=CA_CERT,
    json={"name": "Python Unauthorized Widget", "price": 11.99},
    timeout=10,
)

if resp.status_code == 403:
    print("Authenticated, but not authorized to write this document.")
    print(resp.json()["error"]["reason"])
else:
    resp.raise_for_status()
    print(resp.json())
```

**Expected Output:**

```text
Authenticated, but not authorized to write this document.
action [indices:data/write/index] is unauthorized for user [john] with effective roles [app_reader]
```

### Troubleshooting Tips

| Problem | Likely Cause | Solution |
|---------|--------------|----------|
| `unable to authenticate user` | Wrong username or password | Reset the password with `bin/elasticsearch-reset-password -u <user>` or verify your lab password. |
| `certificate_unknown` / `PKIX path building failed` | Client does not trust the CA | Pass the CA certificate with `--cacert` in curl or `verify=` in Python. |
| `received plaintext http traffic on an https channel` | You used `http://` against a secured HTTPS endpoint | Change the URL to `https://localhost:9200`. |
| `Connection refused` on port 9200 | Elasticsearch is not running or the port is not exposed | Check `docker ps`, container logs, and port mappings. |
| `security_exception ... is unauthorized` | User authenticated but lacks the required privilege | Review the role's `cluster` and `indices` privileges. |
| `api_key ... is expired` | API key passed its expiration date | Create a new key with an appropriate `expiration`. |
| `index_not_found_exception` | Sample data was not loaded | Run Step 0 again and confirm with `GET /products/_count`. |
| `403` for `jane` on `articles` | Expected behavior | `app_writer` only grants access to `products`, not `articles`. |

### Cleanup

Run these commands when you finish the lab.

#### Delete Sample Data and Security Objects

```http
DELETE /products
DELETE /articles

DELETE /_security/user/john
DELETE /_security/user/jane

DELETE /_security/role/app_reader
DELETE /_security/role/app_writer
```

**Expected Output (each):**

```json
{ "acknowledged": true }
```

#### Stop Containers

```bash
docker rm -f kibana elasticsearch
```

### Reflection

1. Why is TLS required on **both** the transport and HTTP layers in production?
2. How does the principle of least privilege apply to the `app_reader` vs `app_writer` roles?
3. Why can a user authenticate successfully but still receive `403 Forbidden`?
4. When would you prefer API key authentication over basic username/password authentication?
5. What additional hardening steps would you apply to a multi-node cluster exposed to the internet?
