## Search Queries

### Introduction to the Query DSL

Elasticsearch provides a powerful **Query DSL** (Domain Specific Language) built on JSON that lets you define complex search requests. Every search is sent as a JSON body to the `_search` endpoint of an index. The DSL is split into two major categories:

- **Leaf queries** — look for a particular value in a specific field (e.g., `match`, `term`, `range`). These are the building blocks of search.
- **Compound queries** — wrap other queries to combine them with boolean logic or to alter their behavior (e.g., `bool`, `dis_max`, `constant_score`).

Queries operate in one of two **contexts**:

| Context | Purpose | Affects `_score`? |
|---------|---------|-------------------|
| **Query context** | "How well does this document match?" | Yes |
| **Filter context** | "Does this document match yes/no?" | No (faster, cacheable) |

Understanding the distinction between these contexts is critical. Placing clauses in filter context whenever you do not need relevance scoring dramatically improves performance because Elasticsearch can cache filter results and skip the expensive scoring step.

### Query Execution Pipeline

The following diagram illustrates how a search request flows through Elasticsearch from the moment the coordinating node receives it to the final result set returned to the client.

```
 Client Request
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      COORDINATING NODE                              │
│  Receives the REST request, parses JSON body, validates syntax      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │   Shard 0    │   │   Shard 1    │   │   Shard 2    │
   │              │   │              │   │              │
   │ ┌──────────┐ │   │ ┌──────────┐ │   │ ┌──────────┐ │
   │ │  QUERY   │ │   │ │  QUERY   │ │   │ │  QUERY   │ │
   │ │  PHASE   │ │   │ │  PHASE   │ │   │ │  PHASE   │ │
   │ │          │ │   │ │          │ │   │ │          │ │
   │ │ 1.Parse  │ │   │ │ 1.Parse  │ │   │ │ 1.Parse  │ │
   │ │ 2.Filter │ │   │ │ 2.Filter │ │   │ │ 2.Filter │ │
   │ │ 3.Score  │ │   │ │ 3.Score  │ │   │ │ 3.Score  │ │
   │ │ 4.Local  │ │   │ │ 4.Local  │ │   │ │ 4.Local  │ │
   │ │   Top-N  │ │   │ │   Top-N  │ │   │ │   Top-N  │ │
   │ └────┬─────┘ │   │ └────┬─────┘ │   │ └────┬─────┘ │
   └──────┼───────┘   └──────┼───────┘   └──────┼───────┘
          │                  │                  │
          └──────────┬───────┴──────────────────┘
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      COORDINATING NODE                              │
│  Merges local top-N lists into a global sorted list                 │
│  Selects final top-N document IDs                                   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │   Shard 0    │   │   Shard 1    │   │   Shard 2    │
   │ ┌──────────┐ │   │ ┌──────────┐ │   │ ┌──────────┐ │
   │ │  FETCH   │ │   │ │  FETCH   │ │   │ │  FETCH   │ │
   │ │  PHASE   │ │   │ │  PHASE   │ │   │ │  PHASE   │ │
   │ │          │ │   │ │          │ │   │ │          │ │
   │ │ Retrieve │ │   │ │ Retrieve │ │   │ │ Retrieve │ │
   │ │ _source, │ │   │ │ _source, │ │   │ │ _source, │ │
   │ │ highlight│ │   │ │ highlight│ │   │ │ highlight│ │
   │ │ fields   │ │   │ │ fields   │ │   │ │ fields   │ │
   │ └────┬─────┘ │   │ └────┬─────┘ │   │ └────┬─────┘ │
   └──────┼───────┘   └──────┼───────┘   └──────┼───────┘
          │                  │                  │
          └──────────┬───────┴──────────────────┘
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      COORDINATING NODE                              │
│  Assembles final response with hits, _score, _source, highlights    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
                        Client Response
```

**Phase summary:**

1. **Query parsing** — the coordinating node validates JSON and broadcasts the query to every relevant shard.
2. **Query phase** — each shard applies filters, scores matching documents, and returns its local top-N (doc IDs + scores).
3. **Global merge** — the coordinating node merges shard results into a single sorted list and picks the final top-N IDs.
4. **Fetch phase** — only the shards holding winning documents retrieve full `_source`, highlights, and stored fields.
5. **Response assembly** — the coordinating node combines fetched data and returns the final JSON to the client.

---

### Query Types

#### Match Query

The `match` query is the standard full-text query. It analyzes the input text and constructs a boolean query from the resulting tokens.

```bash
GET /products/_search
```

```json
{
  "query": {
    "match": {
      "description": "lightweight laptop"
    }
  }
}
```

**Expected response structure:**

```json
{
  "hits": {
    "total": { "value": 42, "relation": "eq" },
    "max_score": 5.12,
    "hits": [
      {
        "_index": "products",
        "_id": "1",
        "_score": 5.12,
        "_source": {
          "description": "Ultra lightweight laptop with long battery life"
        }
      }
    ]
  }
}
```

By default the `match` query uses **OR** between tokens. To require all tokens, set `"operator": "and"`.

---

#### Term Query

The `term` query finds documents that contain an **exact** value in a field. It does not analyze the input, so it is best used on `keyword`, numeric, date, or boolean fields.

```bash
GET /products/_search
```

```json
{
  "query": {
    "term": {
      "status": {
        "value": "published"
      }
    }
  }
}
```

**Expected response structure:**

```json
{
  "hits": {
    "total": { "value": 15, "relation": "eq" },
    "max_score": 1.0,
    "hits": [
      {
        "_index": "products",
        "_id": "7",
        "_score": 1.0,
        "_source": {
          "name": "Wireless Keyboard",
          "status": "published"
        }
      }
    ]
  }
}
```

> **Tip:** Do not use `term` on analyzed `text` fields — the stored tokens are lowercased/stemmed and will rarely match a raw input string.

---

#### Range Query

The `range` query matches documents whose field value falls within a specified range. Commonly used for numbers, dates, and IP addresses.

| Operator | Meaning |
|----------|---------|
| `gt`     | Greater than |
| `gte`    | Greater than or equal to |
| `lt`     | Less than |
| `lte`    | Less than or equal to |

```bash
GET /products/_search
```

```json
{
  "query": {
    "range": {
      "price": {
        "gte": 500,
        "lte": 1500
      }
    }
  }
}
```

**Expected response structure:**

```json
{
  "hits": {
    "total": { "value": 38, "relation": "eq" },
    "max_score": 1.0,
    "hits": [
      {
        "_index": "products",
        "_id": "12",
        "_score": 1.0,
        "_source": {
          "name": "Budget Laptop",
          "price": 699.99
        }
      }
    ]
  }
}
```

For date ranges you can use date math: `"gte": "now-30d/d"` means "30 days ago rounded to the day."

---

#### Bool Query (must, must_not, should, filter)

The `bool` query is the cornerstone compound query. It lets you combine any number of sub-queries with boolean logic.

```
┌──────────────────────────────────────────────────────────────────┐
│                         BOOL QUERY                               │
│                                                                  │
│  ┌────────────┐   Clause MUST match.                             │
│  │   must     │   Contributes to _score.                         │
│  └────────────┘   (like AND)                                     │
│                                                                  │
│  ┌────────────┐   Clause SHOULD match.                           │
│  │  should    │   Contributes to _score if matched.              │
│  └────────────┘   (like OR — boosts relevance)                   │
│                                                                  │
│  ┌────────────┐   Clause MUST match.                             │
│  │  filter    │   Does NOT contribute to _score.                 │
│  └────────────┘   (binary yes/no — cacheable, fast)              │
│                                                                  │
│  ┌────────────┐   Clause MUST NOT match.                         │
│  │ must_not   │   Does NOT contribute to _score.                 │
│  └────────────┘   (exclusion — cacheable, fast)                  │
│                                                                  │
│  Scoring formula (simplified):                                   │
│  _score = sum(must scores) + sum(matched should scores)          │
│                                                                  │
│  A document is a hit only when:                                  │
│    ✓ ALL must clauses match                                      │
│    ✓ ALL filter clauses match                                    │
│    ✓ NO must_not clauses match                                   │
│    ✓ At least minimum_should_match should clauses match           │
│      (default 0 when must/filter exist, otherwise 1)             │
└──────────────────────────────────────────────────────────────────┘
```

**Full example:**

```bash
GET /products/_search
```

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "gaming laptop" } }
      ],
      "filter": [
        { "term":  { "brand": "dell" } },
        { "range": { "price": { "lte": 2000 } } }
      ],
      "should": [
        { "term": { "features": "rgb" } }
      ],
      "must_not": [
        { "term": { "status": "discontinued" } }
      ]
    }
  }
}
```

**Expected response structure:**

```json
{
  "hits": {
    "total": { "value": 5, "relation": "eq" },
    "max_score": 7.24,
    "hits": [
      {
        "_index": "products",
        "_id": "42",
        "_score": 7.24,
        "_source": {
          "name": "Dell Gaming Laptop G15",
          "brand": "dell",
          "price": 1299.99,
          "features": ["rgb", "144hz"],
          "status": "active"
        }
      }
    ]
  }
}
```

---

#### Multi-Match Query

The `multi_match` query searches the same text across multiple fields. It supports several strategies via the `type` parameter:

| Type | Behavior |
|------|----------|
| `best_fields` (default) | Score from the single best-matching field |
| `most_fields` | Sum of scores from all matching fields |
| `cross_fields` | Treats fields as one combined field |
| `phrase` | Runs a `match_phrase` on each field |
| `phrase_prefix` | Runs a `match_phrase_prefix` on each field |

```bash
GET /articles/_search
```

```json
{
  "query": {
    "multi_match": {
      "query": "distributed systems",
      "fields": ["title^3", "body", "tags^2"],
      "type": "best_fields"
    }
  }
}
```

**Expected response structure:**

```json
{
  "hits": {
    "total": { "value": 12, "relation": "eq" },
    "max_score": 9.87,
    "hits": [
      {
        "_index": "articles",
        "_id": "3",
        "_score": 9.87,
        "_source": {
          "title": "Introduction to Distributed Systems",
          "tags": ["distributed systems", "architecture"]
        }
      }
    ]
  }
}
```

The caret notation (`title^3`) is a **field boost** — it multiplies the relevance score from that field by the given factor.

---

#### Wildcard and Regexp Queries

**Wildcard** uses `*` (zero or more characters) and `?` (single character) for pattern matching.

```bash
GET /users/_search
```

```json
{
  "query": {
    "wildcard": {
      "email": {
        "value": "*@example.com"
      }
    }
  }
}
```

**Expected response structure:**

```json
{
  "hits": {
    "total": { "value": 320, "relation": "eq" },
    "max_score": 1.0,
    "hits": [
      {
        "_index": "users",
        "_id": "55",
        "_score": 1.0,
        "_source": { "email": "alice@example.com" }
      }
    ]
  }
}
```

**Regexp** supports a subset of regular expression syntax.

```bash
GET /logs/_search
```

```json
{
  "query": {
    "regexp": {
      "path": {
        "value": "/api/v[0-9]+/users",
        "flags": "ALL"
      }
    }
  }
}
```

**Expected response structure:**

```json
{
  "hits": {
    "total": { "value": 87, "relation": "eq" },
    "max_score": 1.0,
    "hits": [
      {
        "_index": "logs",
        "_id": "990",
        "_score": 1.0,
        "_source": { "path": "/api/v2/users", "status": 200 }
      }
    ]
  }
}
```

> **Warning:** Leading wildcards (`*term`) and complex regexps can be very slow because they require scanning every term in the inverted index. Use with caution on large indices.

---

#### Exists Query

The `exists` query returns documents that contain an indexed value for the given field.

```bash
GET /customers/_search
```

```json
{
  "query": {
    "exists": {
      "field": "phone_number"
    }
  }
}
```

**Expected response structure:**

```json
{
  "hits": {
    "total": { "value": 1024, "relation": "eq" },
    "max_score": 1.0,
    "hits": [
      {
        "_index": "customers",
        "_id": "8",
        "_score": 1.0,
        "_source": {
          "name": "Jane Doe",
          "phone_number": "+1-555-0199"
        }
      }
    ]
  }
}
```

To find documents **missing** a field, combine with `bool` / `must_not`:

```json
{
  "query": {
    "bool": {
      "must_not": {
        "exists": { "field": "phone_number" }
      }
    }
  }
}
```

---

#### Fuzzy Query

The `fuzzy` query finds documents that contain terms similar to the search term, measured by **Levenshtein edit distance**.

```bash
GET /products/_search
```

```json
{
  "query": {
    "fuzzy": {
      "name": {
        "value": "laptp",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

**Expected response structure:**

```json
{
  "hits": {
    "total": { "value": 10, "relation": "eq" },
    "max_score": 3.45,
    "hits": [
      {
        "_index": "products",
        "_id": "2",
        "_score": 3.45,
        "_source": { "name": "Laptop Pro 15" }
      }
    ]
  }
}
```

`"fuzziness": "AUTO"` lets Elasticsearch choose the edit distance based on term length (0 for 1-2 chars, 1 for 3-5 chars, 2 for 6+ chars).

---

#### Match Phrase Query

The `match_phrase` query analyzes the text and requires all tokens to appear **in the same order** and **adjacent** to each other (or within a configurable `slop`).

```bash
GET /articles/_search
```

```json
{
  "query": {
    "match_phrase": {
      "body": {
        "query": "quick brown fox",
        "slop": 1
      }
    }
  }
}
```

**Expected response structure:**

```json
{
  "hits": {
    "total": { "value": 3, "relation": "eq" },
    "max_score": 8.91,
    "hits": [
      {
        "_index": "articles",
        "_id": "17",
        "_score": 8.91,
        "_source": {
          "title": "Classic Pangrams",
          "body": "The quick brown fox jumps over the lazy dog."
        }
      }
    ]
  }
}
```

A `slop` of 1 allows one word to sit between the phrase tokens (e.g., "quick **dark** brown fox" would still match).

---

### Combining Filtering and Scoring — Practical Example

Elasticsearch enables highly customizable search queries that allow for precise filtering and relevance scoring. In the following query example, we combine **filtering** to narrow results to relevant categories and **boosting** to prioritize specific fields, improving both efficiency and relevance in search results.

```bash
GET /products/_search
```

```json
{
  "query": {
    "bool": {
      "filter": {
        "term": { "category": "laptop" }
      },
      "must": {
        "multi_match": {
          "query": "16GB RAM",
          "fields": ["name^3", "description", "specifications"]
        }
      }
    }
  }
}
```

### Breakdown of Components

#### Boolean Query

This example uses a **bool** query, allowing us to combine filtering and relevance scoring. Here, the query has two main parts: the **filter** and the **must** clause. The **filter** narrows the dataset to only include documents with specific attributes, while the **must** clause calculates a relevance score, ranking results by relevance. This approach is particularly useful when only a subset of documents is relevant to the search, as it reduces the number of documents to rank without compromising accuracy.

#### Filter

The **filter** in this query uses a **term filter** to restrict results to documents within the `"laptop"` category. Term filters are ideal for exact matches in structured data, such as categories, tags, or IDs. Here, only documents in the `category: "laptop"` will proceed to the scoring stage, streamlining the results to laptop-related documents. Filtering out unrelated documents early on makes the search more efficient, reducing the workload and enhancing speed.

#### Multi-match Query

The **multi_match** query performs the actual text search, looking for the phrase `"16GB RAM"` in fields such as `name`, `description`, and `specifications`. The `fields` parameter includes `"name^3"`, where the `name` field is boosted by a factor of 3, meaning that documents mentioning "16GB RAM" in the `name` will rank higher than those where it only appears in the description or specifications. Boosting fields in this way allows Elasticsearch to prioritize key fields, ensuring that the most relevant documents appear at the top of the results.

---

### Real-World Examples

> **Note:** The examples below use the modern `bool` + `filter` syntax. Earlier Elasticsearch versions supported a `filtered` query which has been **removed** since Elasticsearch 5.0. Always use `bool` with a `filter` clause instead.

#### Scenario 1: Movie Database Search with Genre Filtering and Title Boosting

In a movie database, users often search for films within specific genres while placing greater importance on titles. A user searching for "action" films with the phrase "Fast and Furious" would benefit from filtering on genre while prioritizing matches in the title field.

**Query:**

```bash
GET /movies/_search
```

```json
{
  "query": {
    "bool": {
      "filter": {
        "term": { "genre": "action" }
      },
      "must": {
        "multi_match": {
          "query": "Fast and Furious",
          "fields": ["title^3", "description"]
        }
      }
    }
  }
}
```

This query filters results to movies tagged with the `"action"` genre. The `multi_match` query then searches for `"Fast and Furious"` in the `title` and `description` fields, with a boost on `title^3`. This ensures that movies with the phrase in their title rank higher, giving preference to more relevant results.

**Example Response:**

```json
{
  "hits": {
    "total": { "value": 8, "relation": "eq" },
    "max_score": 12.34,
    "hits": [
      {
        "_index": "movies",
        "_id": "101",
        "_score": 12.34,
        "_source": {
          "title": "Fast and Furious 7",
          "genre": "action",
          "description": "Deckard Shaw seeks revenge..."
        }
      }
    ]
  }
}
```

Movies with "Fast and Furious" in the title field rank higher than those with the term only in the description, helping users find relevant movies quickly.

#### Scenario 2: Real Estate Listings Search with Location Filtering and Title Boosting

On a real estate platform, users might want to search for properties by location with additional criteria for key terms. For instance, a user may search for properties in `"New York"` containing the phrase "penthouse" in the title or description, with greater importance on the title.

**Query:**

```bash
GET /listings/_search
```

```json
{
  "query": {
    "bool": {
      "filter": {
        "term": { "location": "New York" }
      },
      "must": {
        "multi_match": {
          "query": "penthouse",
          "fields": ["title^4", "description"]
        }
      }
    }
  }
}
```

This query filters the results to properties with a `location` field set to `"New York"`. The `multi_match` query searches for the term `"penthouse"` in both `title` and `description`, but boosts the `title` by a factor of 4. This prioritizes listings with "penthouse" in the title, making it more likely for users to find relevant high-end listings in New York.

**Example Response:**

```json
{
  "hits": {
    "total": { "value": 14, "relation": "eq" },
    "max_score": 15.67,
    "hits": [
      {
        "_index": "listings",
        "_id": "250",
        "_score": 15.67,
        "_source": {
          "title": "Luxury Penthouse in Manhattan",
          "location": "New York",
          "description": "Stunning penthouse with skyline views..."
        }
      }
    ]
  }
}
```

Listings with "penthouse" in the title rank higher than those with it only in the description, optimizing the search experience by bringing prominent listings to the top.

#### Scenario 3: Job Listings Portal with Skill Tag Filtering and Title Boosting

In a job search portal, users may want to filter jobs by required skills and job title. For instance, a user might search for positions tagged with `"Data Science"` that include `"Machine Learning"` in the job title or description.

**Query:**

```bash
GET /jobs/_search
```

```json
{
  "query": {
    "bool": {
      "filter": {
        "term": { "skills": "Data Science" }
      },
      "must": {
        "multi_match": {
          "query": "Machine Learning",
          "fields": ["job_title^3", "job_description"]
        }
      }
    }
  }
}
```

The filter ensures only jobs tagged with `skills: "Data Science"` are considered, narrowing the search to positions relevant to data science. The `multi_match` query then searches for `"Machine Learning"` in both the `job_title` and `job_description` fields, with a boost on `job_title`. This prioritizes job listings with the keyword in the title, making it easier for candidates to find relevant positions.

**Example Response:**

```json
{
  "hits": {
    "total": { "value": 23, "relation": "eq" },
    "max_score": 10.02,
    "hits": [
      {
        "_index": "jobs",
        "_id": "789",
        "_score": 10.02,
        "_source": {
          "job_title": "Machine Learning Engineer",
          "skills": ["Data Science", "Python", "TensorFlow"],
          "job_description": "Build and deploy ML models..."
        }
      }
    ]
  }
}
```

Job postings with "Machine Learning" in the title rank higher, helping users find the most relevant data science positions with a focus on machine learning skills.

---

### Pagination and Sorting

By default Elasticsearch returns the top 10 hits sorted by `_score` descending. You control this with `from`, `size`, and `sort`.

#### Basic Pagination with `from` and `size`

```bash
GET /products/_search
```

```json
{
  "from": 20,
  "size": 10,
  "query": {
    "match_all": {}
  }
}
```

- `from` — the zero-based offset of the first hit to return (default `0`).
- `size` — the maximum number of hits to return (default `10`).

> **Deep pagination warning:** `from + size` cannot exceed the `index.max_result_window` setting (default 10,000). For deeper pagination use the `search_after` parameter or the Scroll API.

#### Custom Sorting

```bash
GET /products/_search
```

```json
{
  "sort": [
    { "price": { "order": "asc" } },
    { "rating": { "order": "desc" } },
    "_score"
  ],
  "query": {
    "match": { "category": "electronics" }
  }
}
```

**Expected response structure (note the `sort` array per hit):**

```json
{
  "hits": {
    "hits": [
      {
        "_id": "44",
        "_score": 2.1,
        "_source": { "name": "USB-C Cable", "price": 9.99, "rating": 4.8 },
        "sort": [9.99, 4.8, 2.1]
      }
    ]
  }
}
```

When you supply an explicit `sort`, Elasticsearch adds a `sort` array to each hit containing the values it sorted on. Documents are first sorted by `price` ascending, ties broken by `rating` descending, and further ties by `_score`.

---

### Highlighting Search Results

Highlighting returns fragments of text from matching fields with the matched tokens wrapped in configurable tags (default `<em>…</em>`).

```bash
GET /articles/_search
```

```json
{
  "query": {
    "match": { "body": "Elasticsearch performance" }
  },
  "highlight": {
    "pre_tags": ["<mark>"],
    "post_tags": ["</mark>"],
    "fields": {
      "body": {
        "fragment_size": 150,
        "number_of_fragments": 3
      }
    }
  }
}
```

**Expected response structure:**

```json
{
  "hits": {
    "hits": [
      {
        "_id": "5",
        "_score": 6.3,
        "_source": {
          "title": "Tuning Elasticsearch",
          "body": "Elasticsearch performance tuning involves many aspects..."
        },
        "highlight": {
          "body": [
            "<mark>Elasticsearch</mark> <mark>performance</mark> tuning involves many aspects...",
            "Improving <mark>Elasticsearch</mark> query <mark>performance</mark> requires..."
          ]
        }
      }
    ]
  }
}
```

Key options:

| Option | Description |
|--------|-------------|
| `pre_tags` / `post_tags` | HTML or custom tags wrapping matched tokens |
| `fragment_size` | Approximate character length of each fragment |
| `number_of_fragments` | Maximum fragments returned per field |
| `type` | Highlighter implementation (`unified`, `plain`, `fvh`) |

---

### Source Filtering (`_source`)

By default every hit includes its full `_source` document. When documents are large you can limit which fields are returned, reducing network overhead.

#### Disable `_source` entirely

```json
{
  "_source": false,
  "query": { "match_all": {} }
}
```

#### Include only specific fields

```json
{
  "_source": ["name", "price"],
  "query": { "match": { "category": "laptop" } }
}
```

#### Include/exclude with wildcards

```json
{
  "_source": {
    "includes": ["user.*", "meta.created_at"],
    "excludes": ["user.password_hash"]
  },
  "query": { "match_all": {} }
}
```

**Expected response structure:**

```json
{
  "hits": {
    "hits": [
      {
        "_id": "1",
        "_score": 1.0,
        "_source": {
          "user": {
            "name": "Alice",
            "email": "alice@example.com"
          },
          "meta": {
            "created_at": "2024-01-15T10:30:00Z"
          }
        }
      }
    ]
  }
}
```

> **Tip:** Prefer source filtering over `stored_fields` in most cases — it is simpler and requires no mapping changes.

---

### Command Reference

| Action | REST Verb & Endpoint | Key Body Parameters |
|--------|----------------------|---------------------|
| Full-text search | `GET /<index>/_search` | `query.match`, `query.multi_match` |
| Exact value lookup | `GET /<index>/_search` | `query.term`, `query.terms` |
| Range filter | `GET /<index>/_search` | `query.range` with `gt/gte/lt/lte` |
| Boolean combination | `GET /<index>/_search` | `query.bool` with `must/should/filter/must_not` |
| Phrase match | `GET /<index>/_search` | `query.match_phrase` with optional `slop` |
| Fuzzy match | `GET /<index>/_search` | `query.fuzzy` with `fuzziness` |
| Wildcard pattern | `GET /<index>/_search` | `query.wildcard` |
| Regular expression | `GET /<index>/_search` | `query.regexp` |
| Field existence | `GET /<index>/_search` | `query.exists` |
| Pagination | `GET /<index>/_search` | `from`, `size` |
| Sorting | `GET /<index>/_search` | `sort: [{ field: { order } }]` |
| Highlighting | `GET /<index>/_search` | `highlight.fields` |
| Source filtering | `GET /<index>/_search` | `_source: [fields]` or `_source: { includes, excludes }` |
| Count documents | `GET /<index>/_count` | `query` (same DSL, returns count only) |
| Validate query | `GET /<index>/_validate/query` | `query` (checks syntax, no execution) |
| Explain scoring | `GET /<index>/_explain/<id>` | `query` (details scoring for one doc) |
