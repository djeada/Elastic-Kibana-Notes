## Task 5: Defining Mappings and Creating Custom Analyzers

### Overview

Mappings define how Elasticsearch stores and indexes fields. Analyzers control how
text is broken into searchable tokens. Together, they determine what you can search
for and how results are ranked.

```
        ┌──────────────── Analyzer Pipeline ────────────────┐
        │                                                    │
 Input  │  Char Filters ──▶ Tokenizer ──▶ Token Filters     │  Indexed
 Text ──│  (html_strip,     (standard,    (lowercase,        │──▶ Terms
        │   pattern_replace) whitespace)   stop, stemmer)    │
        └────────────────────────────────────────────────────┘

 Example: "The <b>Quick</b> Foxes"
   ──▶ char filter  ──▶ "The Quick Foxes"
   ──▶ tokenizer    ──▶ ["The", "Quick", "Foxes"]
   ──▶ token filters ──▶ ["quick", "fox"]
```

### Objectives

- Understand dynamic vs. explicit mappings.
- Compare built-in analyzers and observe how each tokenizes text differently.
- Configure multi-field mappings for full-text search **and** keyword aggregations.
- Build and test a custom analyzer with the `_analyze` API.
- Automate analyzer testing with Python.

---

### Step 1 — Dynamic vs. Explicit Mappings

Elasticsearch can guess field types (**dynamic mapping**) or you can declare them
up front (**explicit mapping**).

**Dynamic mapping — index a document with no predefined schema:**

```json
PUT /blog/_doc/1
{ "title": "Getting Started", "views": 42, "published": "2025-01-15" }
```

**Inspect the inferred mapping:**

```json
GET /blog/_mapping
```

Expected output:
```json
{
  "blog": { "mappings": { "properties": {
    "title":     { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
    "views":     { "type": "long" },
    "published": { "type": "date" }
  }}}
}
```

> Dynamic mapping added a `.keyword` sub-field and detected correct types, but you
> **cannot change a field type** after documents are indexed. Explicit mappings
> avoid surprises.

**Explicit mapping:**

```json
PUT /blog_v2
{
  "mappings": { "properties": {
    "title":     { "type": "text" },
    "views":     { "type": "integer" },
    "published": { "type": "date", "format": "yyyy-MM-dd" }
  }}
}
```

Clean up: `DELETE /blog`

---

### Step 2 — Comparing Built-in Analyzers

Run each command in **Kibana Dev Tools** and compare the tokens.

**Standard** (default — lowercases, removes punctuation):
```json
GET /_analyze
{ "analyzer": "standard", "text": "The Quick-Brown Fox's 2024 jumps!" }
```
Expected tokens: `["the", "quick", "brown", "fox's", "2024", "jumps"]`

**Simple** (splits on non-letters, lowercases — drops numbers):
```json
GET /_analyze
{ "analyzer": "simple", "text": "The Quick-Brown Fox's 2024 jumps!" }
```
Expected tokens: `["the", "quick", "brown", "fox", "s", "jumps"]`

**Whitespace** (splits only on whitespace — no lowercasing or punctuation removal):
```json
GET /_analyze
{ "analyzer": "whitespace", "text": "The Quick-Brown Fox's 2024 jumps!" }
```
Expected tokens: `["The", "Quick-Brown", "Fox's", "2024", "jumps!"]`

**Keyword** (no tokenization — entire input is one token):
```json
GET /_analyze
{ "analyzer": "keyword", "text": "The Quick-Brown Fox's 2024 jumps!" }
```
Expected tokens: `["The Quick-Brown Fox's 2024 jumps!"]`

> **Takeaway:** `standard` works well for general full-text search; `keyword` is
> best for exact-match filtering and aggregations.

---

### Step 3 — Multi-field Mappings

A single field can be indexed multiple ways using **multi-fields**. This lets you
run full-text queries on the root field and exact-match / aggregation queries on a
keyword sub-field.

```json
PUT /products
{
  "mappings": { "properties": {
    "name":     { "type": "text", "fields": { "raw": { "type": "keyword" } } },
    "category": { "type": "text", "fields": { "raw": { "type": "keyword" } } },
    "price":    { "type": "float" }
  }}
}
```

```json
PUT /products/_doc/1
{ "name": "Elasticsearch in Action", "category": "Books & Education", "price": 39.99 }
```

**Full-text search** on the analyzed `name` field:
```json
GET /products/_search
{ "query": { "match": { "name": "elasticsearch" } } }
```

**Aggregation** on the keyword sub-field:
```json
GET /products/_search
{ "size": 0, "aggs": { "categories": { "terms": { "field": "category.raw" } } } }
```

Expected output:
```json
{ "aggregations": { "categories": { "buckets": [
  { "key": "Books & Education", "doc_count": 1 }
]}}}
```

Clean up: `DELETE /products`

---

### Step 4 — Creating a Custom Analyzer

Create an index `articles` with a custom analyzer that lowercases text, removes
English stop words, and applies Porter stemming:

```json
PUT /articles
{
  "settings": {
    "analysis": {
      "analyzer": {
        "custom_content": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "stop", "porter_stem"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title":   { "type": "text", "fields": { "raw": { "type": "keyword" } } },
      "content": { "type": "text", "analyzer": "custom_content" }
    }
  }
}
```

Expected output:
```json
{ "acknowledged": true, "shards_acknowledged": true, "index": "articles" }
```

---

### Step 5 — Testing with the _analyze API

See exactly what tokens the custom analyzer produces:

```json
GET /articles/_analyze
{
  "analyzer": "custom_content",
  "text": "Elasticsearch is a highly scalable search engine."
}
```

Expected output:
```json
{ "tokens": [
  { "token": "elasticsearch", "position": 0 },
  { "token": "highli",        "position": 3 },
  { "token": "scalabl",       "position": 4 },
  { "token": "search",        "position": 5 },
  { "token": "engin",         "position": 6 }
]}
```

> The stop filter removed *"is"* and *"a"*. The Porter stemmer reduced *"highly"* →
> *"highli"*, *"scalable"* → *"scalabl"*, *"engine"* → *"engin"*. These stemmed
> forms let queries like "scale" or "engines" match this document.

---

### Step 6 — Common Mapping Mistakes

#### Mistake 1 — Changing a field type after indexing

```json
PUT /articles/_mapping
{ "properties": { "content": { "type": "keyword" } } }
```

Expected error:
```json
{ "error": { "type": "illegal_argument_exception",
  "reason": "mapper [content] cannot be changed from type [text] to [keyword]"
}, "status": 400 }
```

**Fix:** Create a new index with the correct mapping, then reindex:
```json
POST /_reindex
{ "source": { "index": "articles" }, "dest": { "index": "articles_v2" } }
```

#### Mistake 2 — Aggregating on an analyzed text field

```json
GET /articles/_search
{ "size": 0, "aggs": { "titles": { "terms": { "field": "title" } } } }
```

This errors or returns unexpected per-token buckets.
**Fix:** Use the keyword sub-field: `"field": "title.raw"`.

#### Mistake 3 — Custom analyzer "not found"

Using `GET /_analyze` (cluster level) with an index-level custom analyzer fails.
**Fix:** Always use `GET /<index>/_analyze` for custom analyzers.

---

### Step 7 — Python Script for Analyzer Testing

```python
import requests

ES_URL = "http://localhost:9200"

analyzers = ["standard", "simple", "whitespace", "custom_content"]
sample_text = "Elasticsearch is a highly scalable search engine."

for analyzer in analyzers:
    url = f"{ES_URL}/articles/_analyze" if analyzer == "custom_content" \
          else f"{ES_URL}/_analyze"
    payload = {"analyzer": analyzer, "text": sample_text}
    resp = requests.get(url, json=payload)
    if resp.status_code != 200:
        print(f"[{analyzer}] ERROR {resp.status_code}: {resp.text}")
        continue
    tokens = [t["token"] for t in resp.json().get("tokens", [])]
    print(f"[{analyzer}] {tokens}")
```

Expected output:
```
[standard]       ['elasticsearch', 'is', 'a', 'highly', 'scalable', 'search', 'engine']
[simple]         ['elasticsearch', 'is', 'a', 'highly', 'scalable', 'search', 'engine']
[whitespace]     ['Elasticsearch', 'is', 'a', 'highly', 'scalable', 'search', 'engine.']
[custom_content] ['elasticsearch', 'highli', 'scalabl', 'search', 'engin']
```

> `whitespace` preserves case and trailing punctuation; `custom_content` removes
> stop words and stems the remaining tokens.

---

### Troubleshooting Tips

| Symptom | Likely Cause | Solution |
|---|---|---|
| Search returns no results | Analyzer mismatch between index and query time | Verify both analyzers produce compatible tokens: test with `GET /<index>/_analyze` using each analyzer |
| Aggregation shows individual words | Aggregating on a `text` field | Use a `.keyword` sub-field |
| `mapper_parsing_exception` | Field value doesn't match mapping type | Check mapping with `GET /<index>/_mapping` |
| Cannot change field type | Fields are immutable once created | Reindex into a new index |
| Custom analyzer not found | Querying at cluster level | Use `GET /<index>/_analyze` |

---

### Clean Up

```json
DELETE /articles
DELETE /blog_v2
```

### Reflection

1. Why does stemming improve recall but may reduce precision?
2. When would you choose a `keyword` field over a `text` field?
3. What is the advantage of multi-field mappings over separate fields?
