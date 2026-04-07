### 1. **What is Kibana's role in the Elastic Stack?**

Kibana is the visualization and management layer of the Elastic Stack, providing a browser-based interface for searching, viewing, and interacting with data stored in Elasticsearch through features like Discover, dashboards, Lens visualizations, alerting, Dev Tools, and Stack Management.

### 2. **What is a data view (formerly index pattern) in Kibana?**

A data view tells Kibana which Elasticsearch indices to query by specifying an index name or wildcard pattern (e.g., logs-*) and optionally a timestamp field for time-based filtering; it must be created before you can explore data in Discover or build visualizations.

### 3. **How does KQL differ from Lucene query syntax?**

KQL (Kibana Query Language) provides a simplified syntax with field:value pairs, boolean operators (and, or, not), and wildcard support, with auto-complete in the Kibana search bar; Lucene syntax offers more advanced features like regex, fuzzy matching, and proximity searches but is less user-friendly for everyday filtering.

### 4. **What is the Discover feature used for?**

Discover lets you interactively explore data by running ad-hoc searches and filters against a data view, viewing individual documents in a table, inspecting field statistics, and adjusting the time range, making it the primary tool for log investigation and data exploration before building dashboards.

### 5. **What is Lens and why is it the recommended visualization editor?**

Lens is a drag-and-drop visualization editor that automatically suggests chart types based on the fields you select, supports bar, line, area, pie, metric, table, heatmap, and other chart types, and lets you switch between visualizations without losing configuration, making it the fastest way to build and iterate on visualizations.

### 6. **How do you create a dashboard in Kibana?**

Open the Dashboard app, click Create, add saved or new visualizations using the Add panel button, arrange and resize panels by dragging, apply dashboard-level filters and time ranges, and save the dashboard; you can also add controls (dropdown filters, sliders) and drilldowns that link panels to other dashboards or URLs.

### 7. **What are drilldowns in Kibana dashboards?**

Drilldowns are interactive actions attached to dashboard panels that let users click on a data point or region to navigate to another dashboard, a URL, or a Discover session with pre-applied filters, enabling guided exploration workflows within a single dashboard ecosystem.

### 8. **What is Canvas in Kibana?**

Canvas is a presentation-style workspace for creating pixel-perfect, live data displays using custom layouts, colors, images, and dynamic elements powered by Elasticsearch queries or SQL; it is designed for infographic-style reports and lobby screens rather than the standard grid-based dashboards.

### 9. **What are Spaces in Kibana?**

Spaces are isolated work areas within a single Kibana instance that let you organize saved objects (dashboards, visualizations, data views) by team, project, or environment, and control access to each space through role-based security so that different user groups see only the content relevant to them.

### 10. **What are saved objects and how do you manage them?**

Saved objects include dashboards, visualizations, data views, saved searches, and alerting rules that are stored in a hidden Kibana system index; you can export them as NDJSON files for backup, import them into another Kibana instance, and manage or delete them through Stack Management > Saved Objects.

### 11. **What is the Dev Tools Console?**

Dev Tools Console is a built-in REST client inside Kibana that lets you write and execute Elasticsearch API requests directly, with features like syntax highlighting, auto-complete, request history, and formatted JSON responses, making it the most convenient tool for ad-hoc cluster administration and query prototyping.

### 12. **How do you set up an alerting rule in Kibana?**

Create a rule in the Alerting section by choosing a rule type (e.g., threshold, log threshold, anomaly detection), defining the condition and schedule, and attaching one or more actions (connectors) that fire when the condition is met, such as sending an email, posting to Slack, or calling a webhook.

### 13. **What connectors are available for Kibana alerts?**

Kibana provides built-in connectors for email, Slack, PagerDuty, Microsoft Teams, webhook, IBM Resilient, Jira, ServiceNow, and server log, and supports custom webhook connectors for any HTTP-based integration; each connector is configured once and can be reused across multiple alerting rules.

### 14. **What are the essential `kibana.yml` configuration settings?**

Key settings include server.host and server.port for network binding, elasticsearch.hosts for pointing to the Elasticsearch cluster, elasticsearch.username/password or service token for authentication, server.basePath for reverse proxy setups, and xpack.encryptedSavedObjects.encryptionKey for encrypting sensitive saved object attributes.

### 15. **How does Kibana connect to a secured Elasticsearch cluster?**

Kibana authenticates to Elasticsearch using credentials (username/password), an API key, or a service token specified in kibana.yml, and communicates over HTTPS with TLS certificates configured via elasticsearch.ssl.certificateAuthorities; end users then authenticate to Kibana through the built-in login page or an external identity provider via SAML or OIDC.
