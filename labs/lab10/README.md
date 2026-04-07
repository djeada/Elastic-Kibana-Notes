## Task 10: Securing Your Elasticsearch Cluster

### Security Architecture

```
┌──────────┐      TLS/SSL       ┌────────────────┐     ┌────────────────┐     ┌───────────────┐
│  Client   │ ────────────────► │ Authentication │ ──► │ Authorization  │ ──► │ Elasticsearch │
│ (curl /   │  encrypted channel │ Who are you?   │     │ What can you   │     │   Cluster     │
│  Python)  │ ◄──────────────── │ (credentials / │     │ access? (roles │     │  (data layer) │
└──────────┘                    │  API key)      │     │  / privileges) │     └───────────────┘
                                └────────────────┘     └────────────────┘
```

**Objectives:**

- Understand the Elasticsearch 8+ security model.
- Verify active security with the `_security/_authenticate` API.
- Explore built-in roles and create custom roles with fine-grained privileges.
- Create users, assign roles, and test access-control enforcement.
- Configure TLS/SSL for the transport and HTTP layers.
- Use API keys as an alternative to basic authentication.
- Perform secure HTTPS requests from Python with certificate verification.
- Diagnose common security errors.

---

### Security in Elasticsearch 8+

Starting with version 8.0, security is **enabled by default**. A fresh cluster automatically:

1. Generates a Certificate Authority (CA) and node certificates.
2. Enables TLS on the **transport** (node-to-node) and **HTTP** (client-to-node) layers.
3. Creates the `elastic` superuser with a random password.
4. Prints enrollment tokens for Kibana and additional nodes.

> **Tip:** In a fresh 8.x install no extra configuration is required. Upgrades from 7.x need manual enablement.

---

### Step 1: Verify Authentication

```http
GET /_security/_authenticate
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

> **Important:** A `401 Unauthorized` response means the credentials are wrong or
> `xpack.security.enabled` is not `true` in `elasticsearch.yml`.

---

### Step 2: Built-in Roles Overview

| Role | Purpose |
|------|---------|
| `superuser` | Full access to every cluster operation and index. |
| `kibana_system` | Internal role for the Kibana server process. |
| `logstash_system` | Internal role for Logstash monitoring. |
| `beats_system` | Internal role for Beats monitoring. |
| `remote_monitoring_agent` | Writes monitoring data to a remote cluster. |
| `monitoring_user` | Read-only access to monitoring indices. |
| `ingest_admin` | Manages ingest pipelines. |
| `viewer` | Read-only access to all Kibana features. |
| `editor` | Read-write access to all Kibana features. |

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

---

### Step 3: Create Custom Roles

Create a **read-only** role:

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

Create a **read-write** role:

```http
POST /_security/role/app_writer
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["products"],
      "privileges": ["read", "write", "create_index"]
    }
  ]
}
```

**Expected Output:**

```json
{ "role": { "created": true } }
```

---

### Step 4: Create Users and Assign Roles

**Reader user:**

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

**Writer user:**

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

Verify by authenticating as `john`:

```bash
curl -s -u john:J0hn_S3cure! --cacert config/certs/http_ca.crt \
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

---

### Step 5: Test Access Control

Attempt a **write** as the read-only user `john`:

```bash
curl -s -u john:J0hn_S3cure! --cacert config/certs/http_ca.crt \
  -X POST "https://localhost:9200/products/_doc" \
  -H "Content-Type: application/json" -d '{
    "name": "Unauthorized Widget", "price": 9.99
  }' | python3 -m json.tool
```

**Expected Output (403 Forbidden):**

```json
{
  "error": {
    "root_cause": [{
      "type": "security_exception",
      "reason": "action [indices:data/write/index] is unauthorized for user [john] with effective roles [app_reader]"
    }],
    "type": "security_exception",
    "reason": "action [indices:data/write/index] is unauthorized for user [john] with effective roles [app_reader]"
  },
  "status": 403
}
```

Same operation as `jane` (writer) succeeds:

```bash
curl -s -u jane:J4ne_S3cure! --cacert config/certs/http_ca.crt \
  -X POST "https://localhost:9200/products/_doc" \
  -H "Content-Type: application/json" -d '{
    "name": "Authorized Widget", "price": 19.99
  }' | python3 -m json.tool
```

**Expected Output (201 Created):**

```json
{
  "result": "created",
  "_index": "products",
  "_id": "a1B2c3D4e5",
  "_version": 1,
  "_seq_no": 0,
  "_primary_term": 1,
  "_shards": { "total": 2, "successful": 1, "failed": 0 }
}
```

---

### Step 6: TLS/SSL Configuration

| Layer | Setting Prefix | Purpose |
|-------|---------------|---------|
| Transport | `xpack.security.transport.ssl.*` | Node-to-node encryption |
| HTTP | `xpack.security.http.ssl.*` | Client-to-node encryption |

#### 6a — Generate Certificates

```bash
# Create a CA
bin/elasticsearch-certutil ca --out config/certs/elastic-stack-ca.p12 --pass ""

# Create a node certificate signed by that CA
bin/elasticsearch-certutil cert \
  --ca config/certs/elastic-stack-ca.p12 --ca-pass "" \
  --out config/certs/elastic-certificates.p12 --pass ""
```

**Expected Output:**

```
CA certificate and key have been generated successfully.
```

#### 6b — Configure `elasticsearch.yml`

```yaml
xpack.security.enabled: true

# Transport layer (node-to-node)
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12

# HTTP layer (client-to-node)
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.http.ssl.truststore.path: certs/elastic-certificates.p12
```

Restart the node and verify:

```bash
sudo systemctl restart elasticsearch
curl -s --cacert config/certs/http_ca.crt -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200" | python3 -m json.tool
```

**Expected Output:**

```json
{
  "name": "node-1",
  "cluster_name": "my-cluster",
  "version": { "number": "8.17.0" },
  "tagline": "You Know, for Search"
}
```

---

### Step 7: API Key Authentication

API keys are ideal for services and CI/CD pipelines where long-lived credentials are impractical.

```http
POST /_security/api_key
{
  "name": "my-app-key",
  "expiration": "30d",
  "role_descriptors": {
    "app_reader_scope": {
      "cluster": ["monitor"],
      "indices": [{ "names": ["products"], "privileges": ["read"] }]
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

Use the `encoded` value in the `Authorization` header:

```bash
curl -s -H "Authorization: ApiKey VnVhQ2ZHY0JDZGJrUW0tZTVhT3g6dWkybHAyYXhUTm1zeWFrdzl0dk5udw==" \
  --cacert config/certs/http_ca.crt \
  "https://localhost:9200/products/_search?size=1" | python3 -m json.tool
```

**Expected Output:**

```json
{
  "took": 3,
  "timed_out": false,
  "hits": {
    "total": { "value": 1, "relation": "eq" },
    "hits": [{
      "_index": "products",
      "_id": "a1B2c3D4e5",
      "_score": 1.0,
      "_source": { "name": "Authorized Widget", "price": 19.99 }
    }]
  }
}
```

---

### Step 8: Python Secure Request Examples

**Basic Auth with TLS:**

```python
import requests

CA_CERT = "config/certs/http_ca.crt"
ES_URL  = "https://localhost:9200"

resp = requests.get(
    f"{ES_URL}/_cluster/health",
    auth=("elastic", "your_elastic_password"),
    verify=CA_CERT
)
print(resp.json())
```

**Expected Output:**

```json
{
  "cluster_name": "my-cluster",
  "status": "green",
  "number_of_nodes": 1,
  "active_primary_shards": 5,
  "unassigned_shards": 0
}
```

**API Key Auth with TLS:**

```python
import requests

CA_CERT = "config/certs/http_ca.crt"
ES_URL  = "https://localhost:9200"
API_KEY = "VnVhQ2ZHY0JDZGJrUW0tZTVhT3g6dWkybHAyYXhUTm1zeWFrdzl0dk5udw=="

resp = requests.get(
    f"{ES_URL}/products/_search",
    headers={"Authorization": f"ApiKey {API_KEY}"},
    verify=CA_CERT,
    params={"size": 1}
)
print(resp.json())
```

**Expected Output:**

```json
{
  "took": 2,
  "hits": {
    "total": { "value": 1, "relation": "eq" },
    "hits": [{
      "_index": "products",
      "_id": "a1B2c3D4e5",
      "_source": { "name": "Authorized Widget", "price": 19.99 }
    }]
  }
}
```

---

### Troubleshooting Tips

| Problem | Likely Cause | Solution |
|---------|-------------|----------|
| `unable to authenticate user` | Wrong username or password | Reset with `bin/elasticsearch-reset-password -u <user>` |
| `certificate_unknown` / `PKIX path building failed` | Client does not trust the CA | Pass the CA cert via `--cacert` or `verify=` |
| `received plaintext http traffic on an https channel` | Using `http://` instead of `https://` | Change URL scheme to `https://` |
| `Connection refused` on port 9200 | Node not running or wrong `network.host` | Check `elasticsearch.yml` and restart |
| `security_exception ... is unauthorized` | User lacks the required privilege | Review the role's `indices` / `cluster` privileges |
| `api_key ... is expired` | API key past its expiration | Create a new key with a longer `expiration` |

---

### Reflection

1. Why is TLS required on **both** the transport and HTTP layers in production?
2. How does the principle of least privilege apply to the `app_reader` vs `app_writer` roles?
3. When would you prefer API key authentication over basic username/password auth?
4. What additional hardening steps would you apply to a multi-node cluster exposed to the internet?
