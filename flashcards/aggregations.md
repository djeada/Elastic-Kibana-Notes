### 1. **What are the three main categories of aggregations in Elasticsearch?**

Elasticsearch aggregations fall into bucket aggregations that group documents into buckets based on field values or ranges, metric aggregations that compute numeric summaries (sum, avg, min, max, cardinality) over the documents in each bucket, and pipeline aggregations that take the output of other aggregations as input to perform secondary calculations like derivatives or cumulative sums.

### 2. **How does a terms aggregation work?**

A terms aggregation groups documents into buckets where each bucket corresponds to a unique value of the specified field, returning the top N buckets ordered by document count by default; it works best on keyword fields and uses an approximation algorithm that may not be perfectly accurate when data is spread across many shards.

### 3. **What is the difference between a metric aggregation and a bucket aggregation?**

A bucket aggregation partitions documents into groups (buckets) based on criteria such as field values, ranges, or date intervals, while a metric aggregation computes a numeric statistic (like average, sum, or percentile) over all documents or within each bucket; buckets define groupings, metrics produce numbers.

### 4. **How do sub-aggregations (nested aggregations) work?**

Sub-aggregations are aggregations defined inside a parent bucket aggregation so that the child aggregation runs independently on each bucket's subset of documents, allowing you to compute per-bucket statistics like the average price per product category within a single query.

### 5. **What is the cardinality aggregation used for?**

The cardinality aggregation estimates the number of distinct values for a given field using the HyperLogLog++ algorithm, which is memory-efficient and fast but approximate, making it suitable for counting unique users, IPs, or session IDs at scale where exact uniqueness is not required.

### 6. **When should you use a date histogram aggregation?**

Use a date histogram aggregation when you need to group time-series data into uniform time intervals such as minutes, hours, days, or months for trend analysis, dashboards, or time-based charts; the fixed_interval or calendar_interval parameter controls the bucket width.

### 7. **What is a range aggregation?**

A range aggregation lets you define custom numeric or date ranges and groups documents into buckets based on which range a field value falls into, which is useful for creating price bands, age brackets, or any scenario where you need user-defined intervals rather than equal-width buckets.

### 8. **How does the filter aggregation differ from a query-level filter?**

A filter aggregation applies a filter criterion to create a single bucket containing only the matching documents within the aggregation context, letting you compute metrics on a subset of results without affecting the main query; a query-level filter restricts the entire result set before any aggregation runs.

### 9. **What are pipeline aggregations and when are they useful?**

Pipeline aggregations operate on the output of other aggregations rather than directly on documents, enabling calculations like moving averages, cumulative sums, derivatives, and bucket-level sorting or selection; they are useful for time-series analytics where you need to smooth data or compare sequential buckets.

### 10. **What is the extended_stats aggregation?**

The extended_stats aggregation computes all the statistics of the stats aggregation (count, min, max, avg, sum) plus additional metrics including sum_of_squares, variance, std_deviation, and configurable standard deviation bounds, giving a comprehensive statistical summary in a single request.

### 11. **How does the significant_terms aggregation find unusual terms?**

The significant_terms aggregation compares term frequencies in the foreground set (matching documents) against the background set (all documents in the index) and surfaces terms whose frequency is statistically higher than expected, making it useful for anomaly detection, root-cause analysis, or discovering what is unique about a particular subset of data.

### 12. **What is a composite aggregation and why is it used for pagination?**

A composite aggregation creates multi-field buckets from combinations of sources and supports pagination through an after_key cursor, allowing you to iterate over all buckets in manageable pages rather than loading every bucket into memory at once, which is critical for high-cardinality aggregations.

### 13. **What is the percentiles aggregation?**

The percentiles aggregation estimates the values below which a given percentage of data points fall, using the TDigest algorithm by default, so requesting the 50th, 95th, and 99th percentiles of response times shows the median, tail latency, and near-worst-case latency in a single aggregation.

### 14. **How does the geo_distance aggregation work?**

The geo_distance aggregation defines concentric distance rings around a central geo_point and groups documents into buckets based on which ring their location falls into, which is useful for proximity searches like finding how many stores are within 5 km, 10 km, and 50 km of a given location.

### 15. **What are the best practices for running aggregations efficiently?**

Use keyword or numeric fields rather than analyzed text fields, leverage filter context to reduce the document set before aggregating, set an appropriate shard_size on terms aggregations for accuracy, avoid deep nesting of many sub-aggregations in a single request, and use composite aggregations with pagination instead of unbounded terms aggregations on high-cardinality fields.
