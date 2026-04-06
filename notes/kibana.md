## Kibana

Kibana is the official data visualization and exploration platform for Elasticsearch. As the primary
user interface of the Elastic Stack (formerly the ELK Stack), Kibana enables users to search, observe,
and protect their data through interactive dashboards, rich visualizations, and powerful querying tools.
It transforms raw Elasticsearch data into actionable insights without requiring users to write complex
API calls or manage low-level cluster operations directly. Whether an organization is monitoring
application performance, analyzing security events, or exploring business metrics, Kibana serves as the
centralized window into all data stored within Elasticsearch.

---

### Kibana Architecture

Kibana operates as a Node.js server application that acts as a proxy and rendering layer between the
user's browser and one or more Elasticsearch clusters. When a user interacts with a dashboard or
submits a query, the browser sends requests to the Kibana server, which translates those interactions
into Elasticsearch API calls, retrieves results, and renders them back to the client.

```
  Kibana High-Level Architecture
  ================================

  ┌────────────────────┐         ┌─────────────────────────────────┐
  │                    │  HTTP   │        Kibana Server            │
  │   Browser Client   ├────────►│  (Node.js, default port 5601)  │
  │                    │◄────────┤                                 │
  │  - Dashboards      │  JSON   │  ┌───────────────────────────┐ │
  │  - Visualizations  │         │  │  Saved Objects Store      │ │
  │  - Dev Tools       │         │  │  (.kibana system index)   │ │
  │  - Discover        │         │  │                           │ │
  │                    │         │  │  - Data Views             │ │
  └────────────────────┘         │  │  - Dashboards             │ │
                                 │  │  - Visualizations         │ │
                                 │  │  - Saved Searches         │ │
                                 │  │  - Canvas Workpads        │ │
                                 │  └───────────────────────────┘ │
                                 │                                 │
                                 │  ┌───────────────────────────┐ │
                                 │  │  Security Layer           │ │
                                 │  │  - Authentication         │ │
                                 │  │  - Spaces / RBAC          │ │
                                 │  │  - Encrypted settings     │ │
                                 │  └───────────────────────────┘ │
                                 └───────────┬─────────────────────┘
                                             │
                                             │  REST API (port 9200)
                                             ▼
                                 ┌─────────────────────────────────┐
                                 │     Elasticsearch Cluster       │
                                 │                                 │
                                 │  ┌──────────┐  ┌──────────┐    │
                                 │  │ Data Node│  │ Data Node│    │
                                 │  │ (shards) │  │ (shards) │    │
                                 │  └──────────┘  └──────────┘    │
                                 │                                 │
                                 │  .kibana index (saved objects)  │
                                 │  user indices (logs, metrics)   │
                                 └─────────────────────────────────┘
```

All Kibana configuration — dashboards, visualizations, saved searches — lives in a special
`.kibana` system index inside Elasticsearch itself. This means Kibana is essentially stateless: if
the Kibana server process is restarted or replaced, all user-created artifacts persist in the cluster.

---

### Core Features Overview

Kibana ships with a broad set of tools that serve different roles in the data lifecycle. The following
table summarizes the major features available in a standard Kibana deployment.

```
  +-------------------+------------------------------------------+-----------------------------+
  | Feature           | Description                              | Primary Use Case            |
  +-------------------+------------------------------------------+-----------------------------+
  | Discover          | Ad-hoc data exploration and search       | Investigating raw documents |
  | Visualize Library | Build individual charts and metrics       | Creating reusable visuals   |
  | Dashboard         | Combine visualizations into live panels  | Operational monitoring      |
  | Canvas            | Pixel-perfect, presentation-ready pages  | Executive reports           |
  | Maps              | Geospatial data on interactive maps      | Location-based analytics    |
  | Machine Learning  | Anomaly detection, forecasting           | Automated pattern finding   |
  | Dev Tools         | Interactive Elasticsearch console         | API exploration & debugging |
  | Stack Management  | Indices, ILM, users, roles, pipelines   | Cluster administration      |
  | Observability     | APM, Logs, Metrics, Uptime               | Application monitoring      |
  | Security          | SIEM, endpoint protection, detections    | Threat hunting & response   |
  +-------------------+------------------------------------------+-----------------------------+
```

---

### Data Views (Index Patterns)

A **data view** (historically called an **index pattern**) is the foundational abstraction that tells
Kibana which Elasticsearch indices to query. Without a data view, Kibana has no knowledge of your data.
When you create a data view, Kibana reads the field mappings from the matching indices and makes those
fields available for searching, filtering, and visualizing.

#### Creating a Data View

1. Navigate to **Stack Management → Data Views**.
2. Click **Create data view**.
3. Enter an index pattern string that matches one or more Elasticsearch indices.
4. Select a **time field** if the data is time-based (e.g., `@timestamp`).
5. Optionally assign the data view to a specific **Kibana Space**.

#### Pattern Matching

Data views use wildcard patterns to match indices:

```
  +------------------------------+------------------------------------------+
  | Pattern                      | Matches                                  |
  +------------------------------+------------------------------------------+
  | logs-*                       | logs-2024.01.01, logs-nginx, logs-app    |
  | filebeat-*                   | filebeat-7.17.0-2024.06.15               |
  | metrics-system.*             | metrics-system.cpu, metrics-system.mem   |
  | *                            | All indices (use with caution)           |
  +------------------------------+------------------------------------------+
```

#### Time Fields

When a time field is configured, Kibana enables time-based filtering across Discover, dashboards, and
visualizations. The time picker in the top-right corner of the Kibana interface controls which time
window is applied globally. Common time fields include `@timestamp`, `event.created`, and
`timestamp`. If no time field is selected, time-based features are disabled for that data view.

---

### Discover

Discover is the primary interface for ad-hoc data exploration. It presents documents in a tabular
format, allows free-text and structured queries, and supports field-level inspection. Analysts
typically start in Discover when investigating an incident or exploring an unfamiliar dataset.

#### Key Capabilities

- **Document table**: View raw documents with expandable JSON detail.
- **Field sidebar**: Browse all available fields, see top values, and toggle field visibility.
- **Time histogram**: A bar chart showing document distribution over time.
- **Saved searches**: Persist a query, selected fields, and sort order for reuse or embedding.

#### Time Range Selection

The global time picker supports absolute ranges, relative ranges (e.g., "Last 15 minutes"), and
quick selections. The time range applies to whatever time field is defined in the active data view.

#### Filtering Workflow

1. Enter a query in the search bar using **KQL** or **Lucene** syntax.
2. Click field values in the document table to add positive or negative filters.
3. Pin filters to persist them across Kibana applications.
4. Combine multiple filters with implicit AND logic.

#### Saving a Search

Saved searches capture the current query, selected columns, sort order, and time range. They can be
added to dashboards as panels, shared via short URLs, or used as the basis for alerting rules.

---

### KQL (Kibana Query Language)

KQL is the default query language in Kibana. It provides a simplified, user-friendly syntax for
filtering documents without requiring knowledge of the full Elasticsearch Query DSL. KQL queries
are translated by Kibana into Elasticsearch queries behind the scenes.

#### Basic Syntax

```
  field:value                    — Exact match on a keyword field
  field:"value with spaces"     — Phrase match
  message:error                 — Full-text match on analyzed fields
```

#### Wildcards

```
  status:4*                     — Matches 400, 401, 403, 404, etc.
  host.name:web-*               — Matches web-01, web-prod, web-staging
  message:time*out              — Matches "timeout", "timed out" tokens
```

#### Boolean Operators

```
  status:error AND host:web-01
  status:error OR status:warning
  NOT status:200
  (status:error OR status:warning) AND service:api
```

#### Comparison Operators

KQL supports range comparisons on numeric and date fields:

```
  response_time > 500
  response_time >= 200 AND response_time <= 1000
  @timestamp < "2024-01-01"
  bytes_sent > 1048576
```

#### Nested Field Queries

For documents with nested objects, KQL uses the `nested` syntax:

```
  items:{ product:shoes AND quantity > 2 }
```

This ensures that both conditions apply to the **same nested object**, not across different
nested entries.

#### Existence Checks

```
  error.message:*               — Documents where error.message exists
  NOT error.message:*           — Documents where error.message does not exist
```

#### KQL vs Lucene Syntax Comparison

```
  +-----------------------------+-------------------------------+-------------------------------+
  | Feature                     | KQL                           | Lucene                        |
  +-----------------------------+-------------------------------+-------------------------------+
  | Default operator            | OR (for multi-term text)      | OR                            |
  | Boolean operators           | AND, OR, NOT                  | AND, OR, NOT, +, -            |
  | Wildcards                   | * only                        | *, ?                          |
  | Range queries               | >, >=, <, <=                  | [min TO max], {min TO max}    |
  | Nested queries              | field:{ sub:val }             | Not natively supported        |
  | Regex support               | Not supported                 | /regex/                       |
  | Fuzzy matching              | Not supported                 | term~N                        |
  | Proximity search            | Not supported                 | "term1 term2"~N               |
  | Scripted fields             | Supported                     | Supported                     |
  +-----------------------------+-------------------------------+-------------------------------+
```

> **Tip:** KQL is recommended for most users because it handles nested fields correctly and offers
> autocomplete. Switch to Lucene only when you need regex or fuzzy matching.

---

### Visualizations

Kibana offers multiple visualization editors, each designed for different levels of complexity and
flexibility. Visualizations are the building blocks that get assembled into dashboards.

#### Visualization Types

```
  +-------------------+------------------------------+--------------------------------------------+
  | Type              | Editor                       | Best For                                   |
  +-------------------+------------------------------+--------------------------------------------+
  | Bar chart         | Lens, Aggregation-based      | Comparing categories or time buckets       |
  | Line chart        | Lens, TSVB                   | Trends over time                           |
  | Area chart        | Lens, TSVB                   | Stacked trends, volume over time           |
  | Pie / Donut       | Lens, Aggregation-based      | Proportional breakdowns                    |
  | Metric            | Lens, Aggregation-based      | Single KPI values                          |
  | Gauge             | Lens, Aggregation-based      | Progress toward a threshold                |
  | Data table        | Lens, Aggregation-based      | Tabular aggregated data                    |
  | Tag cloud         | Aggregation-based            | Term frequency visualization               |
  | Heatmap           | Lens                         | Two-dimensional density                    |
  | TSVB              | TSVB (dedicated)             | Advanced time-series with math             |
  | Lens              | Lens (drag-and-drop)         | Quick, intuitive chart creation            |
  | Timelion          | Timelion (legacy)            | Time-series expressions (deprecated)       |
  +-------------------+------------------------------+--------------------------------------------+
```

#### Lens

**Lens** is the recommended visualization editor in modern Kibana. It provides a drag-and-drop
interface that automatically suggests chart types based on the fields you select. Key characteristics:

- **Smart suggestions**: Lens analyzes your data and recommends appropriate chart types.
- **Drag-and-drop**: Drag fields from the sidebar onto the axes to build a visualization.
- **Formula support**: Write mathematical expressions like `average(response_time) / 1000`.
- **Multi-layer**: Combine multiple metrics on a single chart with independent configurations.
- **Reference lines**: Add static threshold lines for SLA or performance targets.

```
  Lens Editor Layout
  ====================

  ┌────────────────────────────────────────────────────────┐
  │  ┌──────────┐   ┌──────────────────────────────────┐  │
  │  │          │   │                                  │  │
  │  │  Field   │   │       Chart Preview Area         │  │
  │  │  List    │   │                                  │  │
  │  │          │   │   (updates in real time as you   │  │
  │  │  drag ──►│   │    drop fields onto axes)        │  │
  │  │  fields  │   │                                  │  │
  │  │  here    │   │                                  │  │
  │  │          │   └──────────────────────────────────┘  │
  │  │          │   ┌──────────────────────────────────┐  │
  │  │          │   │  Horizontal axis: @timestamp     │  │
  │  │          │   │  Vertical axis:   avg(bytes)     │  │
  │  │          │   │  Break down by:  host.name       │  │
  │  └──────────┘   └──────────────────────────────────┘  │
  └────────────────────────────────────────────────────────┘
```

#### TSVB (Time Series Visual Builder)

TSVB is a dedicated time-series visualization tool that supports advanced mathematical operations,
annotations, and multi-index queries on a single panel. It is particularly useful when:

- You need to compute derivatives, cumulative sums, or moving averages.
- You want to overlay annotations (e.g., deployment markers) on a time-series chart.
- You need to query multiple indices in a single visualization.
- You require conditional formatting based on metric thresholds.

TSVB supports panel types including **Time Series**, **Metric**, **Top N**, **Gauge**, **Markdown**,
and **Table**.

#### Creating a Visualization Step-by-Step

1. Navigate to **Visualize Library** → **Create visualization**.
2. Select the editor: **Lens** (recommended), **TSVB**, or **Aggregation based**.
3. Choose or confirm the **data view** that contains the target data.
4. Configure axes:
   - Drag a date field (e.g., `@timestamp`) to the horizontal axis.
   - Drag a numeric field to the vertical axis and select an aggregation (`avg`, `sum`, `count`).
   - Optionally add a **break down by** field for grouping (e.g., `service.name`).
5. Adjust chart type, colors, labels, and legend placement.
6. Click **Save** to add the visualization to the library for reuse across dashboards.

---

### Dashboards

Dashboards are collections of visualizations, saved searches, and controls arranged in a grid layout.
They provide a unified view of operational, business, or security data that updates in real time or on
a configurable refresh interval.

#### Building a Dashboard

1. Navigate to **Dashboard** → **Create dashboard**.
2. Click **Add from library** to insert existing visualizations or saved searches.
3. Click **Create visualization** to build a new panel directly within the dashboard.
4. Resize and rearrange panels by dragging edges and handles.
5. Add **filter controls** (dropdowns, sliders) to allow interactive filtering.

#### Filters and Drilldowns

- **Global filters**: Applied via the query bar or filter pills; affect all panels.
- **Panel-level filters**: Configured per visualization to scope a single panel to specific data.
- **Drilldowns**: Clicking a chart element can navigate to another dashboard or external URL with
  context passed as URL parameters. Drilldowns are configured under the panel's **Create drilldown**
  option.

#### Sharing and Embedding

Dashboards can be shared through several mechanisms:

```
  +-------------------+--------------------------------------------------+
  | Method            | Description                                      |
  +-------------------+--------------------------------------------------+
  | Short URL         | Generate a shortened link to the dashboard state |
  | Snapshot          | Save a read-only snapshot with fixed time range  |
  | Embed code        | iframe HTML for embedding in external pages      |
  | PDF / PNG export  | Export a static report (requires reporting plugin)|
  | CSV export        | Export underlying panel data as CSV              |
  +-------------------+--------------------------------------------------+
```

#### Auto-Refresh

Dashboards support automatic refresh intervals (e.g., every 5 seconds, 30 seconds, 1 minute). This
is configured via the time picker's **Refresh every** dropdown. Auto-refresh re-queries Elasticsearch
on each interval, making dashboards suitable for live monitoring on wall-mounted screens.

---

### Dev Tools Console

The Dev Tools Console is an interactive interface for executing raw Elasticsearch REST API calls
directly from Kibana. It eliminates the need for external HTTP clients like `curl` when exploring
or debugging cluster operations.

#### Features

- **Autocomplete**: Context-aware suggestions for endpoints, query parameters, and JSON body fields.
- **Syntax highlighting**: Color-coded JSON and HTTP methods.
- **Multi-request execution**: Write multiple requests separated by blank lines and execute them in
  sequence.
- **History**: Access previously executed requests via the history panel.
- **Copy as cURL**: Export any request as a `curl` command.

#### Example Usage

```json
GET _cluster/health

GET _cat/indices?v&s=docs.count:desc

POST logs-nginx-*/_search
{
  "size": 5,
  "query": {
    "bool": {
      "must": [
        { "match": { "status": 500 } }
      ],
      "filter": [
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "sort": [
    { "@timestamp": { "order": "desc" } }
  ]
}
```

```json
PUT _ingest/pipeline/add-geo
{
  "description": "Add geo data from IP",
  "processors": [
    {
      "geoip": {
        "field": "client.ip"
      }
    }
  ]
}
```

#### Keyboard Shortcuts

```
  +-----------------------------+-------------------------------------------+
  | Shortcut                    | Action                                    |
  +-----------------------------+-------------------------------------------+
  | Ctrl+Enter (Cmd+Enter)     | Execute current request                   |
  | Ctrl+Up / Ctrl+Down        | Navigate between requests                 |
  | Ctrl+/ (Cmd+/)             | Toggle comment on current line            |
  | Ctrl+Space                 | Trigger autocomplete                      |
  | Ctrl+I                     | Auto-indent current request               |
  +-----------------------------+-------------------------------------------+
```

---

### Saved Objects

Saved objects are the serialized representations of all Kibana artifacts — dashboards,
visualizations, data views, saved searches, Canvas workpads, alerting rules, and more. They are
stored in the `.kibana` system index within Elasticsearch.

#### Common Saved Object Types

```
  +------------------------+---------------------------------------------------+
  | Object Type            | Description                                       |
  +------------------------+---------------------------------------------------+
  | index-pattern          | Data view definition with field mappings           |
  | dashboard              | Dashboard layout, panel references, filters        |
  | visualization          | Individual chart configuration                     |
  | search                 | Saved Discover search with query and columns       |
  | lens                   | Lens visualization configuration                   |
  | map                    | Maps application saved state                       |
  | canvas-workpad         | Canvas presentation workpad                        |
  | alert                  | Alerting rule definition                           |
  +------------------------+---------------------------------------------------+
```

#### Export and Import

Saved objects can be exported as `.ndjson` (newline-delimited JSON) files and imported into other
Kibana instances. This is the primary mechanism for promoting artifacts across environments
(e.g., development → staging → production).

**Export**:
1. Navigate to **Stack Management → Saved Objects**.
2. Select objects or choose **Export all**.
3. Enable **Include related objects** to capture dependencies (e.g., data views referenced by
   dashboards).

**Import**:
1. Navigate to **Stack Management → Saved Objects → Import**.
2. Upload the `.ndjson` file.
3. Choose conflict resolution: **Overwrite**, **Skip**, or **Create new copies**.

#### API-Based Management

Saved objects can also be managed programmatically via the Kibana Saved Objects API:

```json
GET /api/saved_objects/_find?type=dashboard&per_page=20

POST /api/saved_objects/_export
{
  "type": ["dashboard", "visualization"],
  "includeReferencesDeep": true
}

POST /api/saved_objects/_import?overwrite=true
```

---

### Spaces

Spaces provide multi-tenant isolation within a single Kibana instance. Each space functions as its
own virtual Kibana environment with separate saved objects, feature visibility settings, and
role-based access controls. Spaces do not create separate Elasticsearch indices — they segment
the Kibana experience only.

#### Key Concepts

- **Default space**: Every Kibana installation includes a default space. It cannot be deleted.
- **Feature visibility**: Each space can selectively enable or disable Kibana features (e.g., hide
  Machine Learning from a space intended for business analysts).
- **Object isolation**: Dashboards and visualizations in one space are not visible in another unless
  explicitly copied.
- **URL namespacing**: Spaces are reflected in the URL path: `/s/marketing/app/dashboards`.

#### Creating a Space

1. Navigate to **Stack Management → Spaces**.
2. Click **Create a space**.
3. Provide a name, URL identifier, and optional description.
4. Select which features are visible within the space.

#### Role-Based Access per Space

Roles can be assigned space-level privileges, enabling fine-grained access control:

```json
PUT /_security/role/marketing_viewer
{
  "elasticsearch": {
    "indices": [
      {
        "names": ["marketing-*"],
        "privileges": ["read"]
      }
    ]
  },
  "kibana": [
    {
      "base": ["read"],
      "spaces": ["marketing"]
    }
  ]
}
```

This role grants read-only access to the `marketing` space and read access to `marketing-*` indices.

---

### Alerting and Actions

Kibana's alerting framework enables users to define rules that run on a schedule, evaluate conditions
against Elasticsearch data, and trigger actions when those conditions are met. This is the built-in
mechanism for proactive monitoring without requiring external tools.

#### Core Concepts

- **Rule**: A scheduled job that evaluates a condition (e.g., "average CPU > 90% over the last 5
  minutes").
- **Rule type**: The kind of check being performed (index threshold, ES query, metric threshold,
  anomaly detection, etc.).
- **Connector**: An integration that delivers notifications (email, Slack, PagerDuty, webhook, etc.).
- **Action**: A specific notification sent via a connector when the rule fires.

#### Supported Connectors

```
  +-------------------+-----------------------------------------------------------+
  | Connector         | Description                                               |
  +-------------------+-----------------------------------------------------------+
  | Email             | Send notifications via SMTP                               |
  | Slack             | Post messages to a Slack channel                          |
  | PagerDuty         | Create PagerDuty incidents                                |
  | Webhook           | Send HTTP POST to any endpoint                            |
  | Microsoft Teams   | Post messages to a Teams channel                          |
  | Jira              | Create Jira issues                                        |
  | ServiceNow        | Create ServiceNow incidents                               |
  | Index             | Write alert data to an Elasticsearch index                |
  | Server log        | Write to the Kibana server log                            |
  +-------------------+-----------------------------------------------------------+
```

#### Creating a Rule

1. Navigate to **Stack Management → Rules** (or **Observability → Alerts**).
2. Click **Create rule**.
3. Select a rule type (e.g., **Index threshold**).
4. Define the condition:

```json
{
  "index": "metrics-system.cpu-*",
  "timeField": "@timestamp",
  "aggType": "avg",
  "aggField": "system.cpu.total.pct",
  "groupBy": "top",
  "termField": "host.name",
  "termSize": 10,
  "threshold": [0.9],
  "thresholdComparator": ">",
  "timeWindowSize": 5,
  "timeWindowUnit": "m"
}
```

5. Add one or more actions (e.g., send a Slack message when the threshold is exceeded).
6. Set the **check interval** (how often the rule runs).

---

### Canvas

Canvas is a presentation tool that lets users create pixel-perfect, infographic-style reports powered
by live data from Elasticsearch. Unlike standard dashboards, Canvas offers complete control over
layout, styling, fonts, and background images — making it ideal for executive presentations, lobby
displays, and branded reports.

#### Key Features

- **Workpads**: The Canvas equivalent of a slide deck; each workpad contains one or more pages.
- **Elements**: Individual components on a page — charts, images, text, shapes, and custom metrics.
- **Expression language**: Every element is backed by a Timelion-like expression language that defines
  data retrieval and rendering logic.
- **Custom CSS**: Elements support inline CSS for precise styling.
- **Auto-play**: Workpads can cycle through pages automatically on a timer, perfect for wall screens.

#### Example Expression

```
filters
| essql query="SELECT host.name, avg(system.cpu.total.pct) AS cpu FROM \"metrics-*\" GROUP BY host.name"
| pointseries x="host.name" y="cpu"
| plot defaultStyle={seriesStyle bars=0.75}
| render
```

---

### Maps

The Maps application provides interactive geospatial visualization for location data stored in
Elasticsearch. It supports multiple layer types and natively understands Elasticsearch's `geo_point`
and `geo_shape` field types.

#### Layer Types

```
  +-------------------+-----------------------------------------------------------+
  | Layer Type        | Description                                               |
  +-------------------+-----------------------------------------------------------+
  | Documents         | Individual geo_point or geo_shape documents on the map    |
  | Clusters          | Aggregated geo_point clusters using geotile grid          |
  | Heatmap           | Density heatmap from geo_point aggregation                |
  | Choropleth        | Join ES data with geographic boundaries (region shading)  |
  | Tile (EMS)        | Base map tiles from Elastic Maps Service                  |
  | WMS               | External Web Map Service layers                           |
  +-------------------+-----------------------------------------------------------+
```

#### Geo Queries

Maps integrates with Elasticsearch geo queries to enable spatial filtering:

- **Bounding box filter**: Draw a rectangle on the map to filter documents within that region.
- **Distance filter**: Specify a point and radius to find documents within a geographic distance.
- **Spatial joins**: Combine geographic boundaries with document data for choropleth visualizations.

Maps supports tooltips, field-based styling (e.g., color by severity), and time-based animation to
visualize how location data changes over time.

---

### Stack Management

Stack Management is the centralized administration hub within Kibana. It consolidates cluster
management tasks that would otherwise require direct Elasticsearch API calls.

#### Index Management

Browse all indices, view health status, shard count, document count, and storage size. Perform
operations like closing, deleting, freezing, or force-merging indices directly from the UI.

#### Index Lifecycle Management (ILM)

Create and attach ILM policies that automate the progression of indices through lifecycle phases:

```
  ILM Phase Progression
  =======================

  ┌───────┐     ┌──────┐     ┌──────┐     ┌────────┐
  │  Hot  │────►│ Warm │────►│ Cold │────►│ Delete │
  │       │     │      │     │      │     │        │
  │ write │     │ read │     │ rare │     │ remove │
  │ heavy │     │ only │     │ read │     │ data   │
  └───────┘     └──────┘     └──────┘     └────────┘
      │
      └──── Optional: Frozen phase between Cold and Delete
```

ILM policies are defined in Stack Management and can be attached to index templates so that new
indices automatically follow the lifecycle policy.

#### Snapshot and Restore

Configure snapshot repositories (S3, GCS, Azure Blob, shared filesystem) and manage backup schedules.
Create Snapshot Lifecycle Management (SLM) policies to automate periodic snapshots.

#### Users and Roles

Create and manage users, assign roles, and configure role mappings for external identity providers
(LDAP, Active Directory, SAML, OIDC). The role editor allows defining:

- **Elasticsearch privileges**: Cluster-level and index-level permissions.
- **Kibana privileges**: Space-specific access with granularity down to individual features.

#### Ingest Pipelines

Create and test ingest pipelines from the UI. Each pipeline consists of a chain of processors:

```json
PUT _ingest/pipeline/web-logs
{
  "description": "Process web server logs",
  "processors": [
    { "grok": { "field": "message", "patterns": ["%{COMBINEDAPACHELOG}"] } },
    { "date": { "field": "timestamp", "formats": ["dd/MMM/yyyy:HH:mm:ss Z"] } },
    { "geoip": { "field": "clientip" } },
    { "user_agent": { "field": "agent" } },
    { "remove": { "field": ["message", "agent", "timestamp"] } }
  ]
}
```

The **Simulate** feature allows testing a pipeline against sample documents before deploying it.

---

### Configuration

Kibana is configured through the `kibana.yml` file, typically located at `/etc/kibana/kibana.yml`
or within the Kibana installation directory. Below are the most important settings.

#### Essential Settings

```
  +----------------------------------------------+--------------------------------------------------+
  | Setting                                      | Description                                      |
  +----------------------------------------------+--------------------------------------------------+
  | server.port                                  | Port for the Kibana server (default: 5601)       |
  | server.host                                  | Bind address (default: localhost)                |
  | server.basePath                              | URL prefix for reverse proxy setups              |
  | server.publicBaseUrl                         | Full public URL (required for sharing/reporting) |
  | elasticsearch.hosts                          | Array of ES node URLs to connect to              |
  | elasticsearch.username                       | User for authenticating to Elasticsearch         |
  | elasticsearch.password                       | Password for the Kibana system user              |
  | elasticsearch.ssl.certificateAuthorities     | Path to CA cert for TLS connections              |
  | elasticsearch.ssl.verificationMode           | TLS verification: full, certificate, or none     |
  | xpack.encryptedSavedObjects.encryptionKey   | 32+ char key for encrypting saved objects        |
  | xpack.security.encryptionKey                 | 32+ char key for session encryption              |
  | xpack.reporting.encryptionKey                | 32+ char key for reporting encryption            |
  | logging.root.level                           | Log level: all, debug, info, warn, error, fatal  |
  | server.maxPayload                            | Max request payload size (default: 1048576)      |
  +----------------------------------------------+--------------------------------------------------+
```

#### Connecting to a Secured Elasticsearch Cluster

When Elasticsearch has security enabled (the default since 8.x), Kibana requires credentials and
TLS configuration:

```yaml
# kibana.yml — Secured Elasticsearch Connection

server.host: "0.0.0.0"
server.port: 5601
server.publicBaseUrl: "https://kibana.example.com"

elasticsearch.hosts: ["https://es-node-1:9200", "https://es-node-2:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "${KIBANA_ES_PASSWORD}"

elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/ca.crt"]
elasticsearch.ssl.verificationMode: full

server.ssl.enabled: true
server.ssl.certificate: "/etc/kibana/certs/kibana.crt"
server.ssl.key: "/etc/kibana/certs/kibana.key"

xpack.encryptedSavedObjects.encryptionKey: "a]random]32+]character]string]here!"
xpack.security.encryptionKey: "another-32-char-random-key-value!"
xpack.reporting.encryptionKey: "yet-another-32-char-encryption-key"
```

> **Note:** In Elastic 8.x and later, security is enabled by default. The `kibana_system` built-in
> user is the recommended service account for Kibana's connection to Elasticsearch. Avoid using the
> `elastic` superuser for this purpose in production.

---

### Feature Comparison Reference

The following table summarizes when to use each major Kibana feature, helping users choose the
right tool for their analysis needs.

```
  +-------------------+-----------------------------------+-------------------------------------------+
  | Feature           | What It Does                      | When To Use It                            |
  +-------------------+-----------------------------------+-------------------------------------------+
  | Discover          | Search and browse raw documents   | Ad-hoc investigation, log tailing,        |
  |                   |                                   | exploring unfamiliar data                 |
  +-------------------+-----------------------------------+-------------------------------------------+
  | Lens              | Drag-and-drop visualization       | Quick chart creation, most common         |
  |                   | editor with smart suggestions     | visualization needs                       |
  +-------------------+-----------------------------------+-------------------------------------------+
  | TSVB              | Advanced time-series builder      | Complex math on time-series, annotations, |
  |                   | with multi-index support          | derivative and moving average metrics     |
  +-------------------+-----------------------------------+-------------------------------------------+
  | Dashboard         | Grid of combined visualizations   | Operational monitoring, status screens,   |
  |                   | with shared filters               | team-facing analytics                     |
  +-------------------+-----------------------------------+-------------------------------------------+
  | Canvas            | Pixel-perfect presentations       | Executive reports, branded displays,      |
  |                   | with live data                    | lobby screens, infographics               |
  +-------------------+-----------------------------------+-------------------------------------------+
  | Maps              | Geospatial data on interactive    | Fleet tracking, regional analytics,       |
  |                   | maps with multiple layers         | store location analysis                   |
  +-------------------+-----------------------------------+-------------------------------------------+
  | Dev Tools         | Interactive ES REST console       | API exploration, debugging queries,       |
  |                   | with autocomplete                 | cluster administration                    |
  +-------------------+-----------------------------------+-------------------------------------------+
  | Machine Learning  | Anomaly detection, forecasting,   | Automated root-cause analysis, capacity   |
  |                   | outlier detection                 | planning, proactive alerting              |
  +-------------------+-----------------------------------+-------------------------------------------+
  | Alerting          | Scheduled condition checks with   | Proactive monitoring, SLA enforcement,    |
  |                   | automated notifications           | on-call incident routing                  |
  +-------------------+-----------------------------------+-------------------------------------------+
  | Spaces            | Multi-tenant environment          | Team separation, feature scoping,         |
  |                   | isolation within Kibana           | environment-based access control          |
  +-------------------+-----------------------------------+-------------------------------------------+
  | Stack Management  | Centralized admin for indices,    | Cluster administration, user management,  |
  |                   | ILM, users, roles, pipelines      | lifecycle policy configuration            |
  +-------------------+-----------------------------------+-------------------------------------------+
  | Saved Objects     | Export/import Kibana artifacts    | Promoting dashboards across environments, |
  |                   | as .ndjson files                  | backup and disaster recovery              |
  +-------------------+-----------------------------------+-------------------------------------------+
```

---

### Summary

Kibana is the visualization and management layer of the Elastic Stack, providing a rich set of tools
for every stage of the data analysis lifecycle:

- **Data Views** connect Kibana to Elasticsearch indices using wildcard patterns and time fields.
- **Discover** enables ad-hoc exploration of raw documents using KQL or Lucene queries.
- **KQL** provides a simplified query language with boolean operators, wildcards, and nested field
  support.
- **Visualizations** (Lens, TSVB, aggregation-based) transform aggregated data into charts, metrics,
  and tables.
- **Dashboards** combine multiple visualizations into unified monitoring views with shared filters
  and drilldowns.
- **Dev Tools** offers a built-in REST console for direct Elasticsearch API interaction.
- **Saved Objects** are the serialized form of all Kibana artifacts, exportable as `.ndjson` for
  cross-environment promotion.
- **Spaces** provide multi-tenant isolation with per-space feature visibility and role-based access.
- **Alerting** enables proactive monitoring through scheduled rules and connector-based notifications.
- **Canvas** delivers pixel-perfect, presentation-ready reports with live data.
- **Maps** visualizes geospatial data with support for documents, clusters, heatmaps, and
  choropleth layers.
- **Stack Management** centralizes index management, ILM policies, user and role administration,
  snapshot configuration, and ingest pipeline editing.
- **Configuration** is managed through `kibana.yml`, with critical settings for Elasticsearch
  connectivity, TLS, and encryption keys.
