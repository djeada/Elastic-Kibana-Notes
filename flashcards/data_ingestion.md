### 1. **What is the difference between single document indexing and bulk indexing?**

Single document indexing sends one HTTP request per document which incurs significant per-request overhead, while bulk indexing batches hundreds or thousands of operations into a single NDJSON request, dramatically reducing network round-trips and allowing Elasticsearch to optimize internal write coordination, making bulk indexing the preferred method for any non-trivial data load.

### 2. **How does the Bulk API work?**

The Bulk API accepts a body of newline-delimited JSON where each operation has an action line (index, create, update, or delete with metadata like _index and _id) optionally followed by a document body line; Elasticsearch processes all operations in order, returns a per-operation status array, and does not roll back successful operations if some fail.

### 3. **What is the optimal bulk request size?**

The optimal bulk size depends on document size, cluster capacity, and network bandwidth, but a common starting point is 5–15 MB per request or 1,000–5,000 documents; too-small batches waste network overhead while too-large batches can cause memory pressure and long GC pauses, so you should benchmark and adjust iteratively.

### 4. **What is Logstash and what role does it play in data ingestion?**

Logstash is a server-side data processing pipeline that ingests data from multiple sources simultaneously through input plugins, transforms and enriches it using filter plugins (such as grok, mutate, date, and geoip), and outputs it to Elasticsearch or other destinations, acting as the central ETL component in the Elastic Stack.

### 5. **What are Beats and how do they differ from Logstash?**

Beats are lightweight, single-purpose data shippers installed on edge hosts that collect specific types of data (Filebeat for logs, Metricbeat for metrics, Packetbeat for network traffic, Heartbeat for uptime) and send it directly to Elasticsearch or through Logstash; they use fewer resources than Logstash and are designed for distributed collection at the source.

### 6. **What is an ingest pipeline?**

An ingest pipeline is a sequence of processors defined in Elasticsearch that transform documents before they are indexed, eliminating the need for external pre-processing; common processors include grok for parsing unstructured text, date for timestamp parsing, geoip for geographic enrichment, and rename/remove for field manipulation.

### 7. **What is the grok processor and when should you use it?**

The grok processor uses pattern matching based on regular expressions with named captures to extract structured fields from unstructured text like log lines; use it when incoming data is in a semi-structured string format and you need to parse it into discrete fields such as IP addresses, timestamps, and log levels.

### 8. **How does the dissect processor differ from grok?**

The dissect processor splits strings using simple delimiter-based tokenization without regular expressions, making it significantly faster than grok but less flexible; use dissect when the log format is consistent and can be described by fixed delimiters, and fall back to grok only when patterns vary or require regex.

### 9. **How do you test an ingest pipeline before deploying it?**

Use the _ingest/pipeline/_simulate API with a set of sample documents to see exactly how each processor transforms the data, including any errors or dropped fields, without actually indexing anything; this lets you iteratively refine processors and catch mapping issues before applying the pipeline to live data.

### 10. **What happens when a document fails processing in an ingest pipeline?**

If a processor fails on a document the entire document is rejected and an error is returned to the client unless an on_failure handler is defined on that processor; on_failure handlers can route the failed document to a dead-letter index, add error metadata, or apply alternative processing, ensuring the rest of the pipeline is not blocked.

### 11. **How do you attach an ingest pipeline to an index?**

Set the index.default_pipeline setting on the index so that every document indexed into it automatically goes through the specified pipeline, or specify the pipeline query parameter on individual index or bulk requests to override or supplement the default; a final_pipeline setting can also be used for mandatory last-step processing.

### 12. **What is the Newline Delimited JSON (NDJSON) format?**

NDJSON is a text format where each line is a valid JSON object separated by newline characters, used by the Bulk API and other streaming endpoints because it allows line-by-line parsing without loading the entire payload into memory, and every Bulk API request body must end with a trailing newline.

### 13. **What are common strategies for handling ingestion failures?**

Implement client-side retry logic with exponential backoff for transient errors (429, 503), use dead-letter queues or separate error indices for documents that repeatedly fail processing, monitor the Bulk API response items array for per-document errors, and set up ingest pipeline on_failure handlers to capture and tag problematic documents.

### 14. **How does Filebeat ensure at-least-once delivery?**

Filebeat tracks the read offset of each log file in a registry file on disk, so if Filebeat or the destination is temporarily unavailable it resumes from the last acknowledged position after restart; combined with acknowledgment from Elasticsearch or Logstash, this ensures every log line is delivered at least once.

### 15. **What is the `_reindex` API used for?**

The _reindex API copies documents from one index to another within the same cluster (or from a remote cluster), optionally applying an ingest pipeline or a script to transform documents during the copy, which is useful for changing mappings, splitting indices, or migrating data to a new index structure without downtime.
