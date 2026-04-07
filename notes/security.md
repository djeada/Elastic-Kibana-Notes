## Elasticsearch Security

Elasticsearch security encompasses the full set of mechanisms used to protect data at rest and
in transit, authenticate users and services, and enforce fine-grained access control across every
API call. In production environments, an unsecured cluster is an open door -- anyone who can reach
port 9200 can read, modify, or delete every index. The Elastic Stack's built-in security features
(formerly X-Pack Security) address this by layering TLS encryption, multi-realm authentication,
role-based access control, audit logging, and IP filtering into a cohesive security model. Starting
with Elasticsearch 8.0, security is enabled by default, eliminating the most common misconfiguration
that plagued earlier versions: running clusters with no authentication at all.

The following sections cover every layer of Elasticsearch security in depth, from transport-level
encryption through fine-grained document-level access control, providing the knowledge needed to
harden clusters for enterprise and compliance-sensitive workloads.

---

### Security Architecture

Every request to an Elasticsearch cluster passes through a layered security pipeline. Each layer
serves a distinct purpose, and all layers must be configured correctly for end-to-end protection.

```
  Security Request Pipeline
  --------------------------

  Client (curl / Kibana / App)
       |
       |  HTTPS (TLS 1.2+)
       v
  +-------------------------------+
  |  HTTP Interface (port 9200)   |
  |  TLS termination              |
  +------+------------------------+
         |
         v
  +-------------------------------+
  |  Authentication Layer         |
  |                               |
  |  Realms (checked in order):   |
  |   1. Native (built-in store)  |
  |   2. File-based               |
  |   3. LDAP / Active Directory  |
  |   4. SAML / OpenID Connect    |
  |   5. API Key / Service Token  |
  +------+------------------------+
         |
         v
  +-------------------------------+
  |  Authorization Layer          |
  |                               |
  |  Role-Based Access Control:   |
  |   - Cluster privileges        |
  |   - Index privileges          |
  |   - Field-level security      |
  |   - Document-level security   |
  +------+------------------------+
         |
         v
  +-------------------------------+
  |  Audit Logging                |
  |  (granted / denied events)    |
  +------+------------------------+
         |
         v
  +-------------------------------+
  |  Elasticsearch Engine         |
  |  (execute request)            |
  +-------------------------------+
```

The transport layer (port 9300) between nodes uses a separate TLS configuration, ensuring that
internal cluster communication is also encrypted and mutually authenticated. This prevents rogue
nodes from joining the cluster and protects replication traffic from eavesdropping.

---

### Security in Elasticsearch 8+

Elasticsearch 8.0 introduced a fundamental shift in security posture. In earlier versions (7.x
and below), security was available but disabled by default, meaning that many deployments ran
without any authentication or encryption. Version 8.0 changed this by enabling security
automatically during the first startup of a new cluster.

When Elasticsearch 8.0+ starts for the first time, it performs the following actions automatically:

- Generates a Certificate Authority (CA) and TLS certificates for both the transport and HTTP layers
- Enables authentication on all HTTP endpoints
- Creates the `elastic` superuser with a randomly generated password (printed to the terminal)
- Generates an enrollment token for Kibana so it can securely connect to the cluster
- Generates an enrollment token for additional Elasticsearch nodes to join the cluster

```
  First Startup Output (ES 8.x)
  --------------------------------

  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Elasticsearch security features have been automatically configured!

  Authentication is enabled and cluster connections are encrypted.

  Password for the elastic user:
    xK9mP2nQ8vL3wR4tYj6bZs5d

  HTTP CA certificate SHA-256 fingerprint:
    a1b2c3d4e5f6...

  Configure Kibana to use this cluster:
    bin/kibana-setup --enrollment-token <token>

  Configure other nodes to join this cluster:
    bin/elasticsearch --enrollment-token <token>
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

```
  Comparison: ES 7.x vs 8.x Security Defaults
  -----------------------------------------------

  +-----------------------------+-----------------+------------------+
  | Feature                     | ES 7.x Default  | ES 8.x Default   |
  +-----------------------------+-----------------+------------------+
  | Authentication              | Disabled        | Enabled          |
  +-----------------------------+-----------------+------------------+
  | TLS (HTTP layer)            | Disabled        | Enabled (auto)   |
  +-----------------------------+-----------------+------------------+
  | TLS (Transport layer)       | Disabled        | Enabled (auto)   |
  +-----------------------------+-----------------+------------------+
  | Built-in users              | Exist, no pwd   | elastic pwd set  |
  +-----------------------------+-----------------+------------------+
  | Enrollment tokens           | Not available   | Auto-generated   |
  +-----------------------------+-----------------+------------------+
  | Anonymous access            | Allowed         | Denied           |
  +-----------------------------+-----------------+------------------+
```

For clusters upgrading from 7.x, security is not retroactively enabled -- administrators must
configure it manually. New installations on 8.x benefit from the secure-by-default approach,
which dramatically reduces the attack surface out of the box.

---

### Authentication

Authentication is the process of verifying the identity of a user or service making a request.
Elasticsearch supports multiple authentication realms, each representing a different identity
provider. Realms are checked in a configurable order; the first realm that successfully
authenticates the request wins.

#### Native Realm

The native realm stores user credentials directly inside a dedicated Elasticsearch system index
(`.security`). It is the default realm for most deployments and requires no external infrastructure.
Users are managed through the `_security/user` API, and passwords are stored as bcrypt hashes.

The native realm is suitable for small-to-medium teams or environments where an external identity
provider is not available. For larger organizations, it is common to delegate authentication to
LDAP or SAML while retaining a few native users for break-glass access.

#### File-Based Realm

The file-based realm reads user credentials from flat files on each Elasticsearch node:

- `config/users` -- stores username-to-password-hash mappings
- `config/users_roles` -- maps users to roles

Users are managed with the `elasticsearch-users` command-line tool:

```
  bin/elasticsearch-users useradd backup_admin -p s3cur3Pass -r backup_user
  bin/elasticsearch-users list
```

The file realm is checked before any other realm by default, making it ideal for emergency access
when the cluster is degraded and the `.security` index is unavailable.

#### LDAP and Active Directory

For enterprise environments, Elasticsearch can authenticate users against an LDAP directory or
Microsoft Active Directory. This eliminates the need to duplicate user accounts and allows
centralized credential management.

Configuration is placed in `elasticsearch.yml`:

```
  # LDAP realm configuration
  xpack.security.authc.realms.ldap.ldap1:
    order: 2
    url: "ldaps://ldap.example.com:636"
    bind_dn: "cn=elasticsearch,ou=services,dc=example,dc=com"
    user_search:
      base_dn: "ou=users,dc=example,dc=com"
      filter: "(uid={0})"
    group_search:
      base_dn: "ou=groups,dc=example,dc=com"
    ssl:
      certificate_authorities: ["config/certs/ldap-ca.pem"]
```

LDAP groups can be mapped to Elasticsearch roles using role mappings, allowing administrators to
control access centrally from the directory service without touching Elasticsearch configuration
for each new user.

#### SAML and OpenID Connect

SAML 2.0 and OpenID Connect (OIDC) realms enable browser-based single sign-on (SSO) for Kibana.
These realms redirect unauthenticated users to an external Identity Provider (IdP) such as Okta,
Azure AD, or Keycloak. After successful authentication at the IdP, the user is redirected back
to Kibana with an assertion or token that Elasticsearch validates.

SAML and OIDC are designed exclusively for Kibana access -- they are not suitable for direct API
authentication. For programmatic access in SSO environments, API keys or tokens should be used.

#### Checking Authentication

The `_security/_authenticate` API returns information about the currently authenticated user,
including their roles and the realm that authenticated them.

```json
GET /_security/_authenticate
```

```json
{
  "username": "elastic",
  "roles": [
    "superuser"
  ],
  "full_name": null,
  "email": null,
  "metadata": {},
  "enabled": true,
  "authentication_realm": {
    "name": "reserved",
    "type": "reserved"
  },
  "lookup_realm": {
    "name": "reserved",
    "type": "reserved"
  },
  "authentication_type": "realm"
}
```

This endpoint is particularly useful for debugging authentication issues -- it confirms which
realm authenticated the request and what roles were resolved for the user.

---

### Built-in Users and Roles

Elasticsearch ships with a set of reserved users and roles that provide baseline access for
internal components and administrative tasks. These users cannot be deleted, and their roles
cannot be modified.

```
  Built-in Users
  ---------------

  +----------------------+------------------------------------------------------+
  | User                 | Purpose                                              |
  +----------------------+------------------------------------------------------+
  | elastic              | Superuser with full access to all APIs and indices.  |
  |                      | Used for initial setup and break-glass access.       |
  +----------------------+------------------------------------------------------+
  | kibana_system        | Used internally by Kibana to communicate with        |
  |                      | Elasticsearch. Should not be used for Kibana login.  |
  +----------------------+------------------------------------------------------+
  | logstash_system      | Used by Logstash to store monitoring data in         |
  |                      | Elasticsearch.                                       |
  +----------------------+------------------------------------------------------+
  | beats_system         | Used by Beats agents (Filebeat, Metricbeat, etc.)    |
  |                      | to ship monitoring data.                             |
  +----------------------+------------------------------------------------------+
  | apm_system           | Used by APM Server to store trace and metric data.   |
  +----------------------+------------------------------------------------------+
  | remote_monitoring    | Used for cross-cluster monitoring when shipping       |
  | _user                | monitoring data to a dedicated monitoring cluster.    |
  +----------------------+------------------------------------------------------+
```

```
  Built-in Roles
  ---------------

  +----------------------+------------------------------------------------------+
  | Role                 | Privileges                                           |
  +----------------------+------------------------------------------------------+
  | superuser            | Full access to all cluster and index operations.     |
  |                      | Bypasses all security checks.                        |
  +----------------------+------------------------------------------------------+
  | kibana_admin         | Full access to all Kibana features. Does not grant   |
  |                      | direct Elasticsearch index access.                   |
  +----------------------+------------------------------------------------------+
  | kibana_system        | Grants the access Kibana needs internally to         |
  |                      | function (system indices, monitoring, etc.).          |
  +----------------------+------------------------------------------------------+
  | ingest_admin         | Manage ingest pipelines and index templates.         |
  +----------------------+------------------------------------------------------+
  | monitoring_user      | Read-only access to monitoring indices and the       |
  |                      | cluster health API.                                  |
  +----------------------+------------------------------------------------------+
  | editor               | Read and write access to all application indices     |
  |                      | and Kibana features.                                 |
  +----------------------+------------------------------------------------------+
  | viewer               | Read-only access to all application indices and      |
  |                      | Kibana features.                                     |
  +----------------------+------------------------------------------------------+
  | rollup_admin         | Manage rollup jobs and read rollup indices.          |
  +----------------------+------------------------------------------------------+
  | snapshot_user        | Create and restore snapshots. Required for backup    |
  |                      | operations.                                          |
  +----------------------+------------------------------------------------------+
```

Built-in user passwords are set during initial cluster startup (for the `elastic` user) or via
the `elasticsearch-reset-password` tool. System users like `kibana_system` should have their
passwords set and stored securely in the Kibana keystore, never in plain-text configuration files.

---

### Role-Based Access Control (RBAC)

Role-Based Access Control is the primary authorization mechanism in Elasticsearch. Every
authenticated user is assigned one or more roles, and each role defines a set of privileges
at the cluster and index levels. Elasticsearch evaluates the union of all assigned roles --
if any role grants a privilege, the action is permitted.

#### Cluster Privileges

Cluster privileges control access to cluster-wide operations such as managing snapshots,
viewing health status, or administering security settings. Common cluster privileges include
`monitor`, `manage`, `manage_security`, `manage_index_templates`, and `all`.

#### Index Privileges

Index privileges control access to specific indices or index patterns. Privileges include
`read`, `write`, `create_index`, `delete_index`, `manage`, and `all`. Index patterns support
wildcards, allowing a single role to cover entire families of indices (e.g., `logs-*`).

#### Creating a Custom Role

```json
PUT /_security/role/log_reader
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read", "view_index_metadata"],
      "field_security": {
        "grant": ["timestamp", "message", "level", "service"]
      },
      "query": {
        "term": {
          "environment": "production"
        }
      }
    }
  ]
}
```

```json
{
  "role": {
    "created": true
  }
}
```

This role demonstrates several important features:

- **Cluster privileges**: The `monitor` privilege allows the user to view cluster health and stats
  but not modify any cluster settings.
- **Index pattern**: `logs-*` matches all indices starting with `logs-`, covering time-based
  index patterns like `logs-2024.07.15`.
- **Field-level security**: The `field_security.grant` array restricts the user to seeing only
  the listed fields. All other fields are hidden from search results and aggregations.
- **Document-level security**: The `query` filter ensures the user can only access documents
  where `environment` is `production`. Documents in staging or development environments are
  completely invisible to this role.

#### Field-Level Security

Field-level security (FLS) restricts which fields a user can see within documents they have
access to. This is configured per index pattern in the role definition. Fields can be granted
(whitelist approach) or denied (blacklist approach):

```json
PUT /_security/role/pii_restricted
{
  "indices": [
    {
      "names": ["customers-*"],
      "privileges": ["read"],
      "field_security": {
        "grant": ["*"],
        "except": ["ssn", "credit_card", "date_of_birth"]
      }
    }
  ]
}
```

```json
{
  "role": {
    "created": true
  }
}
```

This pattern is common in compliance-sensitive environments where analysts need access to
customer data but must not see personally identifiable information (PII).

#### Document-Level Security

Document-level security (DLS) restricts which documents a user can see within the indices they
have access to. The restriction is expressed as an Elasticsearch query that acts as a filter --
only documents matching the query are visible.

```json
PUT /_security/role/team_alpha_data
{
  "indices": [
    {
      "names": ["projects-*"],
      "privileges": ["read", "write"],
      "query": {
        "bool": {
          "must": [
            { "term": { "team": "alpha" } },
            { "term": { "classification": "internal" } }
          ]
        }
      }
    }
  ]
}
```

```json
{
  "role": {
    "created": true
  }
}
```

---

### Creating Users

Users in the native realm are managed through the `_security/user` API. Each user is assigned
one or more roles that determine their access privileges. Users can also have metadata attached
for organizational purposes.

```json
PUT /_security/user/jdoe
{
  "password": "s3cur3-p4ssw0rd!",
  "roles": ["log_reader", "kibana_admin"],
  "full_name": "Jane Doe",
  "email": "jdoe@example.com",
  "metadata": {
    "department": "Engineering",
    "team": "Observability"
  }
}
```

```json
{
  "created": true
}
```

To update a user's password without changing other attributes:

```json
POST /_security/user/jdoe/_password
{
  "password": "n3w-s3cur3-p4ssw0rd!"
}
```

```json
{}
```

To disable a user without deleting them (preserving their configuration for later reactivation):

```json
PUT /_security/user/jdoe/_disable
```

```json
{}
```

To retrieve user details:

```json
GET /_security/user/jdoe
```

```json
{
  "jdoe": {
    "username": "jdoe",
    "roles": ["log_reader", "kibana_admin"],
    "full_name": "Jane Doe",
    "email": "jdoe@example.com",
    "metadata": {
      "department": "Engineering",
      "team": "Observability"
    },
    "enabled": true
  }
}
```

---

### API Key Authentication

API keys provide a secure alternative to basic authentication for programmatic access. They are
particularly useful for automated scripts, CI/CD pipelines, and service-to-service communication
where embedding user credentials is impractical or insecure. An API key can be scoped to a subset
of the creating user's privileges, and it can be set to expire after a configurable duration.

#### Creating an API Key

```json
POST /_security/api_key
{
  "name": "ingest-pipeline-key",
  "expiration": "30d",
  "role_descriptors": {
    "ingest_writer": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["logs-*", "metrics-*"],
          "privileges": ["write", "create_index"]
        }
      ]
    }
  },
  "metadata": {
    "application": "log-collector",
    "environment": "production"
  }
}
```

```json
{
  "id": "VuaCfGcBCdbkQm-e5aOx",
  "name": "ingest-pipeline-key",
  "expiration": 1721779200000,
  "api_key": "ui2lp2axTNmsyakw9tvNnw",
  "encoded": "VnVhQ2ZHY0JDZGJrUW0tZTVhT3g6dWkybHAyYXhUTm1zeWFrdzl0dk5udw=="
}
```

The `encoded` value is a Base64-encoded combination of the key ID and secret. It is used directly
in the `Authorization` header:

```
  # Using the API key in requests
  curl -H "Authorization: ApiKey VnVhQ2ZHY0JDZGJrUW0tZTVhT3g6dWkybHAyYXhUTm1zeWFrdzl0dk5udw==" \
       https://localhost:9200/logs-*/_search
```

#### Retrieving API Key Information

```json
GET /_security/api_key?id=VuaCfGcBCdbkQm-e5aOx
```

```json
{
  "api_keys": [
    {
      "id": "VuaCfGcBCdbkQm-e5aOx",
      "name": "ingest-pipeline-key",
      "creation": 1719187200000,
      "expiration": 1721779200000,
      "invalidated": false,
      "username": "elastic",
      "realm": "reserved",
      "metadata": {
        "application": "log-collector",
        "environment": "production"
      }
    }
  ]
}
```

#### Invalidating an API Key

```json
DELETE /_security/api_key
{
  "ids": ["VuaCfGcBCdbkQm-e5aOx"]
}
```

```json
{
  "invalidated_api_keys": ["VuaCfGcBCdbkQm-e5aOx"],
  "previously_invalidated_api_keys": [],
  "error_count": 0
}
```

Invalidated keys are immediately rejected on subsequent requests. It is good practice to
invalidate keys when they are no longer needed, rather than waiting for them to expire.

---

### Service Tokens

Service tokens are a specialized authentication mechanism designed for service-to-service
communication within the Elastic Stack. Unlike API keys, service tokens are scoped to specific
Elastic services (Kibana, Fleet Server) and are stored in a file on the Elasticsearch node
rather than in the `.security` index.

Service tokens are created using the `elasticsearch-service-tokens` command-line tool:

```
  # Create a service token for Kibana
  bin/elasticsearch-service-tokens create elastic/kibana kibana-prod-instance-01

  SERVICE_TOKEN elastic/kibana/kibana-prod-instance-01 = AAEAAWVs...long-token-value...

  # List existing service tokens
  bin/elasticsearch-service-tokens list
```

The generated token is then configured in `kibana.yml`:

```
  # kibana.yml
  elasticsearch.serviceAccountToken: "AAEAAWVs...long-token-value..."
```

Service tokens offer several advantages over using the `kibana_system` user with a password:

- They do not require storing a password in the Kibana keystore
- They can be rotated independently on each Kibana instance
- They are stored in a local file, so they remain available even when the `.security` index is
  temporarily unavailable during cluster recovery

---

### TLS/SSL Configuration

Transport Layer Security (TLS) is the foundation of Elasticsearch network security. It provides
confidentiality (encryption), integrity (tamper detection), and authenticity (certificate-based
identity verification) for all network communication. Elasticsearch uses TLS on two distinct
layers, each requiring its own configuration.

#### Transport Layer (Node-to-Node)

The transport layer handles all internal cluster communication on port 9300, including shard
replication, cluster state updates, and node discovery. TLS on the transport layer is mandatory
in production and uses mutual TLS (mTLS), meaning both sides of every connection present and
verify certificates.

#### HTTP Layer (Client-to-Node)

The HTTP layer handles external client requests on port 9200. TLS on this layer encrypts traffic
between clients (Kibana, application code, curl) and Elasticsearch. While not strictly mandatory,
disabling HTTP TLS means credentials are transmitted in plaintext, making it effectively required
for any environment with authentication enabled.

#### Generating Certificates

Elasticsearch provides the `elasticsearch-certutil` tool to generate all required certificates:

```
  # Step 1: Generate a Certificate Authority (CA)
  bin/elasticsearch-certutil ca --pem --out config/certs/ca.zip

  # Step 2: Unzip the CA certificate and key
  unzip config/certs/ca.zip -d config/certs/

  # Step 3: Generate node certificates signed by the CA
  bin/elasticsearch-certutil cert \
    --ca-cert config/certs/ca/ca.crt \
    --ca-key config/certs/ca/ca.key \
    --pem \
    --out config/certs/certs.zip \
    --name node-1 \
    --dns node-1.example.com,localhost \
    --ip 192.168.1.10,127.0.0.1

  # Step 4: Unzip node certificates
  unzip config/certs/certs.zip -d config/certs/
```

For multi-node clusters, the `elasticsearch-certutil cert` command can generate certificates for
all nodes at once using an instances YAML file:

```
  # instances.yml
  instances:
    - name: "node-1"
      dns: ["node-1.example.com"]
      ip: ["192.168.1.10"]
    - name: "node-2"
      dns: ["node-2.example.com"]
      ip: ["192.168.1.11"]
    - name: "node-3"
      dns: ["node-3.example.com"]
      ip: ["192.168.1.12"]
```

```
  bin/elasticsearch-certutil cert \
    --ca-cert config/certs/ca/ca.crt \
    --ca-key config/certs/ca/ca.key \
    --pem \
    --in instances.yml \
    --out config/certs/all-nodes.zip
```

#### elasticsearch.yml TLS Configuration

```
  # ======================== Transport Layer TLS ========================
  xpack.security.transport.ssl.enabled: true
  xpack.security.transport.ssl.verification_mode: full
  xpack.security.transport.ssl.key: certs/node-1/node-1.key
  xpack.security.transport.ssl.certificate: certs/node-1/node-1.crt
  xpack.security.transport.ssl.certificate_authorities: ["certs/ca/ca.crt"]

  # ======================== HTTP Layer TLS =============================
  xpack.security.http.ssl.enabled: true
  xpack.security.http.ssl.key: certs/node-1/node-1.key
  xpack.security.http.ssl.certificate: certs/node-1/node-1.crt
  xpack.security.http.ssl.certificate_authorities: ["certs/ca/ca.crt"]
```

The `verification_mode` setting controls how strictly certificates are validated:

```
  TLS Verification Modes
  -----------------------

  +---------------+------------------------------------------------------+
  | Mode          | Behavior                                             |
  +---------------+------------------------------------------------------+
  | full          | Verifies the certificate chain AND the hostname.     |
  |               | Recommended for production.                          |
  +---------------+------------------------------------------------------+
  | certificate   | Verifies the certificate chain but not the hostname. |
  |               | Useful during initial setup or testing.              |
  +---------------+------------------------------------------------------+
  | none          | Disables all certificate verification. NEVER use     |
  |               | in production -- defeats the purpose of TLS.         |
  +---------------+------------------------------------------------------+
```

---

### Audit Logging

Audit logging records security-relevant events such as authentication successes and failures,
access grants and denials, and changes to security configuration. Audit logs are critical for
compliance (SOC 2, HIPAA, PCI-DSS) and for investigating security incidents.

Audit logging is enabled in `elasticsearch.yml`:

```
  # Enable audit logging
  xpack.security.audit.enabled: true

  # Control which events are logged
  xpack.security.audit.logfile.events.include:
    - "access_denied"
    - "access_granted"
    - "anonymous_access_denied"
    - "authentication_failed"
    - "authentication_success"
    - "connection_denied"
    - "run_as_denied"
    - "run_as_granted"
    - "tampered_request"

  # Exclude noisy system events (optional)
  xpack.security.audit.logfile.events.exclude:
    - "access_granted"

  # Filter by user to reduce volume
  xpack.security.audit.logfile.events.ignore_filters:
    system_filter:
      users: ["kibana_system", "beats_system"]
```

Audit events are written to a dedicated log file (`<ES_HOME>/logs/<cluster_name>_audit.json`).
Each event is a JSON document containing the timestamp, event type, user, source IP, request
details, and the outcome. These logs can be shipped to a monitoring cluster using Filebeat for
centralized analysis and alerting.

---

### IP Filtering

IP filtering restricts which network addresses can connect to the Elasticsearch cluster. It
operates at the network layer, rejecting connections before authentication is attempted. This
provides defense-in-depth by limiting the attack surface even if credentials are compromised.

IP filtering is configured in `elasticsearch.yml`:

```
  # Allow only specific subnets and deny everything else
  xpack.security.transport.filter.allow: ["192.168.1.0/24", "10.0.0.0/8"]
  xpack.security.transport.filter.deny: ["_all"]

  # HTTP layer filtering (client connections)
  xpack.security.http.filter.allow: ["192.168.1.0/24", "10.0.0.0/8"]
  xpack.security.http.filter.deny: ["_all"]
```

IP filters can also be updated dynamically using the cluster settings API:

```json
PUT /_cluster/settings
{
  "persistent": {
    "xpack.security.transport.filter.allow": ["192.168.1.0/24"],
    "xpack.security.transport.filter.deny": "_all"
  }
}
```

```json
{
  "acknowledged": true,
  "persistent": {
    "xpack": {
      "security": {
        "transport": {
          "filter": {
            "allow": "192.168.1.0/24",
            "deny": "_all"
          }
        }
      }
    }
  },
  "transient": {}
}
```

---

### Encrypted Communication

Understanding the TLS handshake process helps with debugging connection issues between nodes
and clients.

```
  TLS Handshake Flow (Client -> Elasticsearch)
  -----------------------------------------------

  Client                                    Elasticsearch
    |                                            |
    |  1. ClientHello                            |
    |    (supported TLS versions, ciphers)       |
    |------------------------------------------->|
    |                                            |
    |  2. ServerHello                            |
    |    (chosen TLS version, cipher)            |
    |<-------------------------------------------|
    |                                            |
    |  3. Server Certificate                     |
    |    (X.509 cert chain)                      |
    |<-------------------------------------------|
    |                                            |
    |  4. Client verifies certificate            |
    |    - Check CA signature chain              |
    |    - Check hostname matches SAN/CN         |
    |    - Check expiration date                 |
    |                                            |
    |  5. Key Exchange                           |
    |    (ECDHE parameters)                      |
    |------------------------------------------->|
    |                                            |
    |  6. Finished (encrypted)                   |
    |<=========================================>|
    |                                            |
    |  7. Application Data (HTTP/JSON)           |
    |    encrypted with session keys             |
    |<=========================================>|
    |                                            |
```

```
  Mutual TLS (Transport Layer, Node-to-Node)
  ---------------------------------------------

  Node A                                     Node B
    |                                            |
    |  1. ClientHello                            |
    |------------------------------------------->|
    |                                            |
    |  2. ServerHello + Node B Certificate       |
    |<-------------------------------------------|
    |                                            |
    |  3. CertificateRequest (mutual TLS)        |
    |<-------------------------------------------|
    |                                            |
    |  4. Node A Certificate                     |
    |------------------------------------------->|
    |                                            |
    |  5. Both nodes verify each other's certs   |
    |    - Same CA must have signed both          |
    |    - Prevents rogue nodes from joining      |
    |                                            |
    |  6. Encrypted channel established          |
    |<=========================================>|
    |                                            |
```

Mutual TLS on the transport layer is the primary mechanism that prevents unauthorized nodes
from joining the cluster. Both nodes must present certificates signed by the same trusted CA.

---

### Common Security Configurations

Different deployment scenarios require different levels of security configuration. The following
table summarizes three common profiles.

```
  Security Configuration Profiles
  ----------------------------------

  +---------------------------+----------------+------------------+---------------------+
  | Feature                   | Minimal (Dev)  | Standard (Prod)  | Enterprise (Compl.) |
  +---------------------------+----------------+------------------+---------------------+
  | Authentication            | Native realm   | Native + LDAP    | SAML/OIDC + Native  |
  +---------------------------+----------------+------------------+---------------------+
  | TLS (Transport)           | Auto-generated | Custom CA certs  | Custom CA + mTLS    |
  +---------------------------+----------------+------------------+---------------------+
  | TLS (HTTP)                | Auto-generated | Custom CA certs  | Custom CA + client  |
  |                           |                |                  | cert auth           |
  +---------------------------+----------------+------------------+---------------------+
  | Authorization             | Built-in roles | Custom RBAC      | RBAC + FLS + DLS    |
  +---------------------------+----------------+------------------+---------------------+
  | Audit logging             | Disabled       | Enabled (errors)  | Full audit trail    |
  +---------------------------+----------------+------------------+---------------------+
  | IP filtering              | Disabled       | Subnet restrict  | Strict allowlist    |
  +---------------------------+----------------+------------------+---------------------+
  | API key management        | Manual         | Expiring keys    | Scoped + monitored  |
  +---------------------------+----------------+------------------+---------------------+
  | Monitoring                | Basic          | Stack Monitoring | SIEM integration    |
  +---------------------------+----------------+------------------+---------------------+
```

The **Minimal** profile is suitable for local development and testing. Security is enabled
(as of ES 8.x) but uses auto-generated certificates and the native realm only.

The **Standard** profile adds custom CA-signed certificates, LDAP integration, audit logging
for security events, and IP filtering. This is appropriate for most production deployments.

The **Enterprise** profile adds SAML/OIDC for SSO, field-level and document-level security,
full audit trails for compliance, and SIEM integration for security event correlation.

---

### Troubleshooting

Security misconfigurations are among the most common causes of Elasticsearch cluster issues.
The following section covers frequently encountered errors and their resolutions.

```
  Common Security Errors and Solutions
  ---------------------------------------

  +------------------------------------+-------------------------------------------+
  | Error                              | Solution                                  |
  +------------------------------------+-------------------------------------------+
  | "unable to find valid              | The client does not trust the ES CA.      |
  |  certification path"               | Add the CA cert to the client truststore  |
  |                                    | or use --cacert with curl.                |
  +------------------------------------+-------------------------------------------+
  | "certificate does not match        | The hostname used in the URL does not     |
  |  any of the subject alternative    | match the SAN entries in the cert.        |
  |  names"                            | Regenerate the cert with the correct DNS  |
  |                                    | names or use the correct hostname.        |
  +------------------------------------+-------------------------------------------+
  | "security_exception: missing       | The request has no credentials. Add       |
  |  authentication credentials"       | -u user:password or an Authorization      |
  |                                    | header to the request.                    |
  +------------------------------------+-------------------------------------------+
  | "security_exception: action        | The authenticated user lacks the required |
  |  [indices:data/read/search] is     | index privilege. Add the 'read' privilege |
  |  unauthorized for user [jdoe]"     | for the target index to the user's role.  |
  +------------------------------------+-------------------------------------------+
  | "failed to establish trust with    | The transport TLS certs are not signed by |
  |  remote cluster"                   | the same CA. Use the same CA for all      |
  |                                    | nodes or add the remote CA to the trust   |
  |                                    | store.                                    |
  +------------------------------------+-------------------------------------------+
  | "token expired"                    | The OAuth/SAML token has expired.         |
  |                                    | Re-authenticate through the IdP or        |
  |                                    | refresh the token.                        |
  +------------------------------------+-------------------------------------------+
  | "api_key is invalidated"           | The API key was explicitly invalidated.   |
  |                                    | Create a new API key.                     |
  +------------------------------------+-------------------------------------------+
```

Additional debugging techniques:

- **Check certificate details**: `openssl s_client -connect localhost:9200` reveals the presented
  certificate chain and any verification errors.
- **Verify CA trust**: `curl --cacert config/certs/ca/ca.crt https://localhost:9200` tests whether
  the CA certificate is correct.
- **Inspect audit logs**: Check `logs/<cluster_name>_audit.json` for `authentication_failed` events
  with details about which realm was tried and why it failed.
- **Reset passwords**: Use `bin/elasticsearch-reset-password -u elastic` to reset a built-in user's
  password when locked out.
- **Check role assignments**: `GET /_security/user/jdoe` confirms which roles are assigned and
  whether the user is enabled.

---

### Command Reference

| Operation | REST Verb & Endpoint | Key Parameters |
|-----------|----------------------|----------------|
| Authenticate | `GET /_security/_authenticate` | Returns current user, roles, and realm |
| Create user | `PUT /_security/user/<username>` | `password`, `roles`, `full_name`, `email`, `metadata` |
| Get user | `GET /_security/user/<username>` | Returns user details and role assignments |
| Delete user | `DELETE /_security/user/<username>` | Permanently removes the user |
| Change password | `POST /_security/user/<username>/_password` | `password` |
| Enable user | `PUT /_security/user/<username>/_enable` | Re-enables a disabled user |
| Disable user | `PUT /_security/user/<username>/_disable` | Disables without deleting |
| Create role | `PUT /_security/role/<role_name>` | `cluster`, `indices`, `applications`, `run_as` |
| Get role | `GET /_security/role/<role_name>` | Returns role definition |
| Delete role | `DELETE /_security/role/<role_name>` | Permanently removes the role |
| Create role mapping | `PUT /_security/role_mapping/<name>` | `roles`, `rules`, `enabled` |
| Create API key | `POST /_security/api_key` | `name`, `expiration`, `role_descriptors`, `metadata` |
| Get API key | `GET /_security/api_key` | `id`, `name`, `username`, `realm_name` |
| Invalidate API key | `DELETE /_security/api_key` | `ids`, `name`, `username`, `realm_name` |
| Create service token | `POST /_security/service/<service>/credential/token/<name>` | Service-scoped token |
| Get privileges | `GET /_security/privilege` | Lists all application privileges |
| Clear cache | `POST /_security/realm/<realm>/_clear_cache` | Clears authentication cache for realm |
| SSL certificates | `GET /_ssl/certificates` | Lists all loaded TLS certificates and expiration |
| IP filter (dynamic) | `PUT /_cluster/settings` | `xpack.security.http.filter.allow/deny` |
