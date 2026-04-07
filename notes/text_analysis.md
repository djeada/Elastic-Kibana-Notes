## Text Analysis

Text analysis is the foundation of full-text search in Elasticsearch. When a document is indexed
or a search query is executed, Elasticsearch does not store or compare raw strings directly.
Instead, it passes text through an **analysis pipeline** that breaks the text into individual
terms, normalizes those terms, and stores them in an optimized data structure called the
**inverted index**. This process is what allows Elasticsearch to match "running" when a user
searches for "run," to ignore HTML tags embedded in content, and to treat "cafe" and "cafe" as
equivalent terms.

Without text analysis, search engines would be limited to exact-match lookups. A query for
"Foxes" would fail to match a document containing "foxes" or "fox." By understanding the
linguistic structure of text -- including word boundaries, morphological roots, synonyms, and
character encodings -- Elasticsearch delivers results that align with user intent rather than
requiring users to guess the precise form stored in the index.

Text analysis applies in two contexts. At **index time**, the text of each document field is
analyzed and the resulting tokens are written into the inverted index. At **search time**, the
query string is analyzed with the same (or a compatible) analyzer so that the query tokens can
be matched against the index tokens. Choosing the right analyzer for each field is one of the
most important decisions in Elasticsearch schema design.

### The Analyzer Pipeline

Every analyzer in Elasticsearch is composed of exactly three stages that execute in order:

1. **Character Filters** -- transform the raw character stream before tokenization.
2. **Tokenizer** -- splits the character stream into individual tokens.
3. **Token Filters** -- modify, add, or remove tokens after tokenization.

The following diagram shows the complete pipeline from raw input to inverted index:

```
 Raw Document Text
 "The <b>Quick</b> Foxes were jumping &amp; leaping!"
  |
  v
 +------------------------------------------------------------------------+
 |                        CHARACTER FILTERS                               |
 |  Operate on the raw character stream before any tokenization occurs.   |
 |                                                                        |
 |  1. html_strip        "The Quick Foxes were jumping & leaping!"        |
 |     (removes HTML)                                                     |
 |                                                                        |
 |  2. mapping            (no mapping rules configured in this example)   |
 |     (char replacement)                                                 |
 |                                                                        |
 |  3. pattern_replace    (no pattern rules configured in this example)   |
 |     (regex replace)                                                    |
 +------------------------------------------------------------------------+
  |
  |  Cleaned stream: "The Quick Foxes were jumping & leaping!"
  v
 +------------------------------------------------------------------------+
 |                            TOKENIZER                                   |
 |  Splits the character stream into discrete tokens using rules such     |
 |  as whitespace boundaries, punctuation, or regex patterns.             |
 |                                                                        |
 |  Standard Tokenizer Output:                                            |
 |  [ "The", "Quick", "Foxes", "were", "jumping", "leaping" ]            |
 |                                                                        |
 |  (Note: "&" is removed because the standard tokenizer strips most     |
 |   punctuation and special characters.)                                 |
 +------------------------------------------------------------------------+
  |
  v
 +------------------------------------------------------------------------+
 |                          TOKEN FILTERS                                 |
 |  Applied in order to each token produced by the tokenizer.             |
 |                                                                        |
 |  1. lowercase          [ "the", "quick", "foxes", "were",             |
 |                          "jumping", "leaping" ]                        |
 |                                                                        |
 |  2. stop (English)     [ "quick", "foxes", "jumping", "leaping" ]     |
 |     (removes "the", "were")                                           |
 |                                                                        |
 |  3. stemmer (English)  [ "quick", "fox", "jump", "leap" ]             |
 |     ("foxes"->"fox", "jumping"->"jump", "leaping"->"leap")            |
 |                                                                        |
 |  4. synonym            [ "quick", "fox", "jump", "jump" ]             |
 |     ("leap" -> "jump" via synonym rule)                                |
 +------------------------------------------------------------------------+
  |
  |  Final tokens: [ "quick", "fox", "jump", "jump" ]
  v
 +------------------------------------------------------------------------+
 |                         INVERTED INDEX                                 |
 |                                                                        |
 |   Term       Document IDs       Positions                             |
 |  ---------  ------------------  ----------                             |
 |   "fox"      doc_1              [2]                                    |
 |   "jump"     doc_1              [3, 4]                                 |
 |   "quick"    doc_1              [1]                                    |
 +------------------------------------------------------------------------+
```

### Tokenization

Tokenization is the process of splitting a stream of text into smaller units called **tokens**.
Each token typically represents a single word, although some tokenizers produce sub-word units
(n-grams) or treat the entire input as one token. Tokenization is the first step that gives
structure to unstructured text.

```
 +-----------------------------------------------+
 |               Input Text                       |
 |  "Hello world from Ahmed"                      |
 +-----------------------------------------------+
                      |
                      |  Split by whitespace
                      |  and punctuation rules
                      v
 +-----------------------------------------------+
 |              Tokenizer                         |
 +-----------------------------------------------+
      |          |          |          |
      v          v          v          v
 +---------+ +---------+ +---------+ +---------+
 | "Hello" | | "world" | | "from"  | | "Ahmed" |
 +---------+ +---------+ +---------+ +---------+
      |          |          |          |
      |    (each token carries metadata:         |
      |     position, start/end offset)          |
      v          v          v          v
 Token 0     Token 1     Token 2     Token 3
 pos=0       pos=1       pos=2       pos=3
 start=0     start=6     start=12    start=17
 end=5       end=11      end=16      end=22
```

- **Input:** The raw text string `"Hello world from Ahmed"`.
- **Operation:** The tokenizer splits the sentence based on whitespace and punctuation rules. Each token is assigned a position and character offsets that track where it appeared in the original text.
- **Output:** A list of tokens `["Hello", "world", "from", "Ahmed"]` that can be searched independently. The metadata enables features like highlighted search results and phrase queries.

### Normalization

Normalization standardizes tokens so that superficially different forms of the same word map to
the same index term. Without normalization, a search for "run" would miss documents containing
"Running" or "ran." Normalization includes lowercasing, stemming or lemmatization, and synonym
expansion.

```
 +-----------------------------------------------+
 |              Raw Tokens                        |
 |  [ "Hello", "World", "Foxes", "Running",      |
 |    "Leaping", "CAFE" ]                         |
 +-----------------------------------------------+
                      |
                      v
 +-----------------------------------------------+
 |              NORMALIZATION                     |
 +-----------------------------------------------+
          |              |              |              |
          v              v              v              v
 +--------------+ +---------------+ +---------------+ +----------------+
 |  Lowercasing | | Stemming /    | | Synonym       | | ASCII Folding  |
 |              | | Lemmatization | | Handling      | |                |
 | "Hello"      | | "foxes"       | | "leap" and   | | "cafe"         |
 |   -> "hello" | |   -> "fox"    | | "jump" map   | |   -> "cafe"    |
 | "World"      | | "running"     | | to "jump"    | | "naive"        |
 |   -> "world" | |   -> "run"    | |              | |   -> "naive"   |
 | "CAFE"       | | "leaping"     | |              | |                |
 |   -> "cafe"  | |   -> "leap"   | |              | |                |
 +--------------+ +---------------+ +---------------+ +----------------+
          |              |              |              |
          v              v              v              v
 +-----------------------------------------------+
 |  Normalized Tokens                             |
 |  [ "hello", "world", "fox", "run",            |
 |    "jump", "cafe" ]                            |
 +-----------------------------------------------+
```

- **Lowercasing:** Converts all tokens to lowercase so that `"Hello"` and `"hello"` match.
- **Stemming/Lemmatization:** Reduces words to their root form. `"foxes"` becomes `"fox"` and `"running"` becomes `"run"`, allowing different inflections to match.
- **Synonym Handling:** Maps semantically equivalent words to a single term. `"leap"` and `"jump"` can both be indexed as `"jump"`.
- **ASCII Folding:** Converts Unicode characters to their ASCII equivalents. `"cafe"` becomes `"cafe"` so users do not need to type accented characters.

### Summary Table

| Feature            | Description & Examples |
|--------------------|------------------------|
| **Tokenization**   | Breaking text into smaller searchable units (tokens). `"Hello world from Ahmed"` becomes `["Hello", "world", "from", "Ahmed"]`. This is the foundational step that gives structure to raw text and enables term-level matching. |
| **Normalization**  | Standardizing tokens so that variant forms resolve to the same index term. Includes lowercasing (`"Quick"` -> `"quick"`), stemming (`"foxes"` -> `"fox"`), synonym mapping (`"leap"` -> `"jump"`), and ASCII folding (`"cafe"` -> `"cafe"`). |
| **Character Filters** | Pre-tokenization transformations on the raw character stream. Examples: stripping HTML tags, replacing characters, regex substitution. |
| **Token Filters**  | Post-tokenization transformations on individual tokens. Examples: lowercase, stop word removal, stemming, synonym expansion, n-gram generation. |
| **Inverted Index** | The data structure that maps each unique term to the list of documents (and positions) that contain it. This is what makes full-text search fast. |

### Character Filters

Character filters operate on the raw character stream **before** the tokenizer runs. An analyzer
can have zero or more character filters, and they execute in the order they are defined.

#### html_strip

Removes HTML tags and decodes HTML entities. Useful when indexing content from web pages or
rich-text editors.

```
Input:   "<p>Elasticsearch is <b>fast</b> &amp; scalable.</p>"
Output:  "Elasticsearch is fast & scalable."
```

Configuration example:
```json
{
  "char_filter": {
    "my_html_strip": {
      "type": "html_strip",
      "escaped_tags": ["b"]
    }
  }
}
```
With `escaped_tags` set to `["b"]`, the `<b>` tags are preserved while all other HTML is removed.

#### mapping

Replaces specific character sequences with defined substitutions. This is useful for normalizing
special characters or symbols before tokenization.

```
Input:   "I love C++ and C#"

Mapping rules:  "+" => "plus", "#" => "sharp"

Output:  "I love Cplusplus and Csharp"
```

Configuration example:
```json
{
  "char_filter": {
    "programming_symbols": {
      "type": "mapping",
      "mappings": ["+=>plus", "#=>sharp"]
    }
  }
}
```

#### pattern_replace

Uses a regular expression to find and replace character sequences in the stream.

```
Input:   "Phone: (555) 123-4567"

Pattern: "[^0-9]"    Replacement: ""

Output:  "5551234567"
```

Configuration example:
```json
{
  "char_filter": {
    "digits_only": {
      "type": "pattern_replace",
      "pattern": "[^0-9]",
      "replacement": ""
    }
  }
}
```

### Tokenizers

The tokenizer is the core of the analyzer. It receives the character stream (after character
filters have been applied) and produces a stream of tokens. Every analyzer has exactly **one**
tokenizer.

| Tokenizer      | Description                                          | Input Example                     | Output Tokens                                  |
|----------------|------------------------------------------------------|-----------------------------------|-------------------------------------------------|
| **standard**   | Divides text at word boundaries (Unicode Text Segmentation). Removes most punctuation. | `"Hello-world's test"` | `["Hello", "world's", "test"]` |
| **whitespace** | Splits text at whitespace characters only. Keeps punctuation attached to tokens. | `"Hello-world's test"` | `["Hello-world's", "test"]` |
| **keyword**    | Emits the entire input as a single token. No splitting occurs. | `"Hello world"` | `["Hello world"]` |
| **pattern**    | Splits text using a regular expression (default: `\W+`). | `"one,two;three"` | `["one", "two", "three"]` |
| **ngram**      | Produces n-grams of specified lengths from the text. Useful for partial matching and autocomplete on arbitrary substrings. | `"fox"` (min=2, max=3) | `["fo", "fox", "ox"]` |
| **edge_ngram** | Produces n-grams anchored to the **beginning** of each token. Ideal for search-as-you-type and prefix matching. | `"fox"` (min=1, max=3) | `["f", "fo", "fox"]` |

**When to choose which tokenizer:**
- Use **standard** for general-purpose full-text search in most languages.
- Use **whitespace** when punctuation is meaningful (e.g., programming code, part numbers).
- Use **keyword** for fields that should not be analyzed at all but still need token filters (e.g., email addresses with lowercasing).
- Use **pattern** when you need custom split logic using regex.
- Use **ngram** / **edge_ngram** for autocomplete and partial-match use cases.

### Token Filters

Token filters receive the token stream from the tokenizer and can modify, add, or remove tokens.
An analyzer can have zero or more token filters, applied in order.

#### lowercase

Converts all token text to lowercase. This is the most commonly used token filter.

```
Input tokens:   [ "The", "Quick", "FOX" ]
Output tokens:  [ "the", "quick", "fox" ]
```

#### stop

Removes common words (stop words) that carry little semantic meaning. Elasticsearch ships with
stop word lists for many languages.

```
Input tokens:   [ "the", "quick", "fox", "is", "over", "the", "fence" ]
Output tokens:  [ "quick", "fox", "fence" ]

Removed (English stop words): "the", "is", "over", "the"
```

#### stemmer

Reduces words to their morphological root using language-specific rules.

```
Input tokens:   [ "running", "foxes", "easily", "production" ]
Output tokens:  [ "run", "fox", "easili", "product" ]
```

Note that stemming is an approximation -- `"easily"` becomes `"easili"`, not `"easy"`. This is
expected behavior for algorithmic stemmers like the English (Porter) stemmer.

#### synonym

Expands or replaces tokens using a synonym list. Synonyms can be defined inline or loaded from a
file.

```
Synonym rules:  "leap, jump, hop"

Input tokens:   [ "the", "fox", "leaps" ]
Output tokens:  [ "the", "fox", "leaps", "jumps", "hops" ]
                (in expand mode, all synonyms are added)
```

Configuration example:
```json
{
  "filter": {
    "my_synonyms": {
      "type": "synonym",
      "synonyms": ["leap,jump,hop", "fast,quick,speedy"]
    }
  }
}
```

#### ascii_folding

Converts Unicode characters to their ASCII equivalents. Essential for multilingual search where
users may not type accented characters.

```
Input tokens:   [ "cafe", "naive", "resume" ]
Output tokens:  [ "cafe", "naive", "resume" ]
```

#### ngram (token filter)

Generates n-grams from each token. Unlike the ngram tokenizer, this operates on already-split
tokens.

```
Token: "fox"  (min_gram=2, max_gram=3)

Output: [ "fo", "ox", "fox" ]
```

### Built-in Analyzers

Elasticsearch ships with several pre-configured analyzers that combine a tokenizer and token
filters for common use cases.

| Analyzer       | Tokenizer   | Token Filters                        | Description                                     |
|----------------|-------------|--------------------------------------|-------------------------------------------------|
| **standard**   | standard    | lowercase                            | General-purpose analyzer. Good default choice for most full-text fields. Splits on word boundaries, lowercases tokens. |
| **simple**     | lowercase   | *(none)*                             | Splits at non-letter characters and lowercases. Numbers are discarded entirely. |
| **whitespace** | whitespace  | *(none)*                             | Splits at whitespace only. No lowercasing or other normalization. |
| **stop**       | lowercase   | stop (English)                       | Like `simple` but also removes English stop words. |
| **keyword**    | keyword     | *(none)*                             | No-op analyzer -- emits the entire input as one token. Useful with `normalizer` on keyword fields. |
| **pattern**    | pattern     | lowercase                            | Splits using a regex (default `\W+`) and lowercases. |
| **language**   | standard    | lowercase, stop, stemmer (per lang)  | Language-specific analyzers (e.g., `english`, `french`, `german`). Apply language-appropriate stop words and stemming. |

**Choosing the right built-in analyzer:**
- Start with **standard** unless you have a specific reason not to.
- Use **language** analyzers (e.g., `english`) when stemming and stop word removal improve relevance.
- Use **whitespace** when you need to preserve the exact token boundaries.
- Use **keyword** for fields that must not be split.

### Custom Analyzer Configuration

When built-in analyzers do not meet your needs, you can define a custom analyzer that specifies
its own character filters, tokenizer, and token filters. Custom analyzers are defined in the
index settings under `settings.analysis`. The structure is:

```json
{
  "settings": {
    "analysis": {
      "char_filter": { ... },
      "tokenizer":   { ... },
      "filter":      { ... },
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "char_filter": ["<char_filter_names>"],
          "tokenizer": "<tokenizer_name>",
          "filter": ["<token_filter_names>"]
        }
      }
    }
  }
}
```

A full working example is shown in the Blog Search Engine section below.

### The _analyze API

The `_analyze` API lets you test how an analyzer processes text without indexing any documents.
This is invaluable for debugging and tuning your analysis configuration.

**Test with a built-in analyzer:**
```json
POST /_analyze
{
  "analyzer": "standard",
  "text": "The Quick Brown Foxes jumped!"
}
// Returns tokens: ["the", "quick", "brown", "foxes", "jumped"]
// Each token includes start_offset, end_offset, and position metadata.
```

**Test with individual components:**
```json
POST /_analyze
{
  "tokenizer": "standard",
  "filter": ["lowercase", "stop"],
  "text": "The Quick Brown Foxes jumped!"
}
```

**Test against an index's configured analyzer:**
```json
POST /my_blog_index/_analyze
{
  "analyzer": "blog_analyzer",
  "text": "<p>The foxes &amp; hares were leaping!</p>"
}
```

### Multi-fields

Elasticsearch allows a single source field to be indexed in multiple ways using **multi-fields**.
This is useful when you need both full-text search and exact-match aggregations on the same data.

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "raw":          { "type": "keyword" },
          "autocomplete": { "type": "text", "analyzer": "autocomplete_analyzer" },
          "english":      { "type": "text", "analyzer": "english" }
        }
      }
    }
  }
}
```

With this mapping:
- `name` -- analyzed with the standard analyzer for general full-text search.
- `name.raw` -- stored as an exact keyword for sorting, aggregations, and exact-match filtering.
- `name.autocomplete` -- analyzed with a custom autocomplete analyzer (e.g., edge_ngram) for search-as-you-type.
- `name.english` -- analyzed with the English analyzer for stemmed, language-aware search.

### How the Inverted Index Works

The inverted index is the core data structure that makes full-text search fast. Instead of
scanning every document for a query term, Elasticsearch looks up the term in the inverted index
and immediately finds all documents that contain it.

```
 Documents:
 +----------------------------------------------------+
 | doc_1: "The quick brown fox jumps over the fence"   |
 | doc_2: "A quick red car drove over the bridge"      |
 | doc_3: "The brown fox sat quietly on the fence"     |
 +----------------------------------------------------+
                          |
                 (analyzed with English analyzer)
                          |
                          v
 Inverted Index:
 +----------+----------------------+------------------+
 |  Term    |  Document IDs        |  Positions        |
 +----------+----------------------+------------------+
 | "bridge" | doc_2                | [7]               |
 | "brown"  | doc_1, doc_3         | [2], [1]          |
 | "car"    | doc_2                | [3]               |
 | "drove"  | doc_2                | [4]               |
 | "fence"  | doc_1, doc_3         | [7], [5]          |
 | "fox"    | doc_1, doc_3         | [3], [2]          |
 | "jump"   | doc_1                | [4]               |
 | "over"   | doc_1, doc_2         | [5], [5]          |
 | "quick"  | doc_1, doc_2         | [1], [1]          |
 | "quietli"| doc_3                | [3]               |
 | "red"    | doc_2                | [2]               |
 | "sat"    | doc_3                | [3]               |
 +----------+----------------------+------------------+

 Query: "quick fox"
         |
         v
 Look up "quick" -> doc_1, doc_2
 Look up "fox"   -> doc_1, doc_3
                         |
                         v
 Merge results:  doc_1 (matches both terms, highest score)
                 doc_2 (matches "quick" only)
                 doc_3 (matches "fox" only)
```

Key observations:
- Stop words like `"the"`, `"a"`, and `"on"` have been removed and do not appear in the index.
- `"jumps"` was stemmed to `"jump"` and `"quietly"` to `"quietli"`.
- Position data enables phrase queries -- Elasticsearch can verify that `"quick"` appears immediately before `"fox"` in `doc_1`.
- Document frequency (how many documents contain a term) is used for relevance scoring.

### `text` vs `keyword` Field Types

Choosing between `text` and `keyword` is one of the most important mapping decisions.

| Aspect              | `text`                                      | `keyword`                                  |
|---------------------|---------------------------------------------|--------------------------------------------|
| **Analysis**        | Full analysis pipeline (tokenizer + filters) | No analysis; stored as-is                 |
| **Use case**        | Full-text search (match queries)            | Exact matching, sorting, aggregations      |
| **Stored as**       | Multiple tokens in inverted index           | Single term in inverted index              |
| **Example value**   | `"The Quick Brown Fox"`                     | `"The Quick Brown Fox"`                    |
| **Indexed terms**   | `["the", "quick", "brown", "fox"]`          | `["The Quick Brown Fox"]`                  |
| **Sortable**        | Not directly (use `.keyword` multi-field)   | Yes                                        |
| **Aggregatable**    | Not directly                                | Yes                                        |
| **Supports match**  | Yes                                         | Yes (exact match only)                     |
| **Supports wildcard** | Limited                                   | Yes                                        |
| **Max length**      | Unlimited (tokens are bounded)              | 32,766 bytes (default `ignore_above`)      |

**Rule of thumb:**
- Use `text` when users will search for the field using natural language.
- Use `keyword` for identifiers, status codes, tags, email addresses, and any value where exact matching is required.
- When in doubt, use **multi-fields** to get both behaviors.

### Analysis-Related API Reference

| API Endpoint                                  | Method | Description                                                      |
|-----------------------------------------------|--------|------------------------------------------------------------------|
| `POST /_analyze`                              | POST   | Test how an analyzer processes text (global scope).              |
| `POST /{index}/_analyze`                      | POST   | Test analysis using an index's configured analyzers.             |
| `PUT /{index}`                                | PUT    | Create an index with custom analysis settings.                   |
| `POST /{index}/_close`                        | POST   | Close an index (required before updating analysis settings).     |
| `PUT /{index}/_settings`                      | PUT    | Update analysis settings on a closed index.                      |
| `POST /{index}/_open`                         | POST   | Re-open an index after updating settings.                        |
| `PUT /{index}/_mapping`                       | PUT    | Update field mappings (add new fields with analyzers).           |
| `GET /{index}/_mapping`                       | GET    | Retrieve current mappings, including analyzer configuration.     |
| `GET /{index}/_settings`                      | GET    | Retrieve current index settings, including analysis definitions. |

### Example: Text Analysis for a Blog Search Engine

Let's walk through a complete example of configuring and using text analysis for a hypothetical
blog search engine. The goal is to provide high-quality full-text search across blog posts that
may contain HTML content, synonyms, and natural language variations.

#### Step 1: Define the Index with a Custom Analyzer

```json
PUT /blog
{
  "settings": {
    "analysis": {
      "char_filter": {
        "blog_html_strip": { "type": "html_strip" }
      },
      "filter": {
        "blog_stop":    { "type": "stop",    "stopwords": "_english_" },
        "blog_stemmer": { "type": "stemmer", "language":   "english"  },
        "blog_synonyms": {
          "type": "synonym",
          "synonyms": ["leap,jump,hop", "fast,quick,speedy"]
        }
      },
      "analyzer": {
        "blog_analyzer": {
          "type":        "custom",
          "char_filter": ["blog_html_strip"],
          "tokenizer":   "standard",
          "filter":      ["lowercase", "blog_stop", "blog_stemmer", "blog_synonyms"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title":       { "type": "text", "analyzer": "blog_analyzer" },
      "body":        { "type": "text", "analyzer": "blog_analyzer" },
      "author":      { "type": "keyword" },
      "tags":        { "type": "keyword" },
      "published_at": { "type": "date"    }
    }
  }
}
```

#### Step 2: Index a Blog Post

```json
POST /blog/_doc/1
{
  "title": "The Quick Brown Foxes",
  "body": "<p>The quick brown foxes were <b>jumping</b> over obstacles, leaping with grace.</p>",
  "author": "Ahmed",
  "tags": ["nature", "animals"],
  "published_at": "2024-01-15"
}
```

#### Step 3: Trace the Analysis Pipeline

Suppose a blog post body includes:
> `"<p>The quick brown foxes were <b>jumping</b> over obstacles, leaping with grace.</p>"`

The `blog_analyzer` processes this as follows:

```
 Raw input:
 "<p>The quick brown foxes were <b>jumping</b> over obstacles, leaping with grace.</p>"
  |
  |  [html_strip char filter]
  v
 "The quick brown foxes were jumping over obstacles, leaping with grace."
  |
  |  [standard tokenizer]
  v
 ["The","quick","brown","foxes","were","jumping","over","obstacles","leaping","with","grace"]
  |
  |  [lowercase]
  v
 ["the","quick","brown","foxes","were","jumping","over","obstacles","leaping","with","grace"]
  |
  |  [stop]  removes: "the", "were", "over", "with"
  v
 ["quick", "brown", "foxes", "jumping", "obstacles", "leaping", "grace"]
  |
  |  [stemmer]  "foxes"->"fox", "jumping"->"jump", "obstacles"->"obstacl",
  |             "leaping"->"leap", "grace"->"grace"
  v
 ["quick", "brown", "fox", "jump", "obstacl", "leap", "grace"]
  |
  |  [synonym]  "leap" expands to also include "jump", "hop"
  v
 ["quick", "brown", "fox", "jump", "obstacl", "leap", "jump", "hop", "grace"]
```

These tokens are stored in the inverted index for `doc_1`.

#### Step 4: Search

```json
GET /blog/_search
{
  "query": {
    "match": {
      "body": "fox jumps over an obstacle"
    }
  }
}
```

The search query `"fox jumps over an obstacle"` is analyzed with the same `blog_analyzer`:
- `"fox"` -> `"fox"`
- `"jumps"` -> `"jump"`
- `"over"` -> removed (stop word)
- `"an"` -> removed (stop word)
- `"obstacle"` -> `"obstacl"`

Elasticsearch looks up `"fox"`, `"jump"`, and `"obstacl"` in the inverted index and finds
`doc_1`, which contains all three terms. The document is returned as a highly relevant match
even though the original text said "foxes were jumping over obstacles" -- demonstrating the
power of text analysis.

#### Step 5: Verify with the _analyze API

```json
POST /blog/_analyze
{
  "analyzer": "blog_analyzer",
  "text": "<p>The quick brown foxes were <b>jumping</b> over obstacles, leaping with grace.</p>"
}
```

This returns the exact token list produced by the analyzer, allowing you to verify that your
configuration behaves as expected.

### Text Analysis in Action: Search Query Example

Imagine a user searches for `"fox jumps over an obstacle."` With tokenization and normalization
applied, Elasticsearch returns results from the original phrase `"foxes were jumping over
obstacles, leaping with grace"` because it recognizes the morphological and semantic variations:

- `"fox"` matches `"foxes"` (stemming reduced both to `"fox"`).
- `"jumps"` matches `"jumping"` and `"leaping"` (stemming + synonym expansion).
- `"obstacle"` matches `"obstacles"` (stemming reduced both to `"obstacl"`).
- Stop words `"over"` and `"an"` are ignored in both the document and query.

This process allows Elasticsearch to handle complex language patterns, improve accuracy, and
enhance the user search experience -- returning relevant results even when the query and document
use different word forms.

### Importance of Text Analysis in Elasticsearch

Through the analysis pipeline -- character filters, tokenizers, and token filters -- Elasticsearch
transforms raw text into a structured, searchable form. This enables:

- **Improved relevance:** By understanding linguistic variations, Elasticsearch returns the most relevant results for a query, even when the exact words differ from the indexed text.
- **Efficient indexing:** With standardized terms, the inverted index is smaller and lookups are faster. Stemming and stop word removal reduce redundancy.
- **Better user experience:** Users find what they need without needing to know the exact phrasing stored in the index. Synonyms, stemming, and case normalization compensate for natural language complexity.
- **Flexibility:** Custom analyzers allow fine-grained control over how each field is processed, enabling the same Elasticsearch cluster to handle blog posts, product catalogs, log files, and more -- each with analysis tuned to its domain.
