### 1. **What are the three stages of the analyzer pipeline?**

An analyzer processes text through three stages: character filters that modify the raw text (like stripping HTML or replacing characters), a tokenizer that splits the filtered text into individual tokens, and token filters that transform each token (lowercasing, removing stop words, stemming, adding synonyms) before the tokens are stored in the inverted index.

### 2. **What does the standard analyzer do?**

The standard analyzer is the default in Elasticsearch; it uses the Unicode Text Segmentation tokenizer to split text on word boundaries, applies the lowercase token filter, and removes punctuation, making it a good general-purpose choice for most European languages but not ideal when you need to preserve special characters or apply stemming.

### 3. **What does the `html_strip` character filter do?**

The html_strip character filter removes HTML tags and decodes HTML entities (like &amp; to &) from the input text before tokenization, which is useful when indexing web page content or rich text fields where the markup should not be treated as searchable terms.

### 4. **What is the difference between the standard and whitespace tokenizers?**

The standard tokenizer splits text at word boundaries as defined by Unicode, also removing most punctuation, while the whitespace tokenizer splits text only on whitespace characters and preserves punctuation and special characters attached to tokens; use whitespace when you need to keep hyphenated terms or other punctuated values intact.

### 5. **When should you use the keyword tokenizer?**

Use the keyword tokenizer when the entire field value should be treated as a single token without any splitting, which is useful for email addresses, URLs, product codes, or any value that must be matched exactly; it is the tokenizer behind keyword fields and is often combined with token filters like lowercase in a custom normalizer.

### 6. **What are ngram and edge_ngram tokenizers used for?**

The ngram tokenizer generates all substrings of a configurable length from each token enabling partial and infix matching (e.g., "qui", "uic", "ick" from "quick"), while the edge_ngram tokenizer generates substrings only from the beginning of each token enabling autocomplete and prefix matching (e.g., "q", "qu", "qui").

### 7. **What does the lowercase token filter do?**

The lowercase token filter converts every token to lowercase letters so that searches become case-insensitive; it is included by default in the standard analyzer and is one of the most commonly used normalization steps because users rarely type queries with exact casing.

### 8. **What does the stop token filter do?**

The stop token filter removes common words (stop words) like "the", "is", "and", "of" that appear in almost every document and contribute little to relevance, reducing index size and improving query precision; the list of stop words can be customized per language.

### 9. **What is stemming and why is it useful?**

Stemming reduces words to their root form (e.g., "running", "runs", "ran" all become "run") so that queries match all morphological variants of a term; Elasticsearch provides stemmer token filters for many languages, improving recall without requiring users to search for every possible word form.

### 10. **What is the difference between stemming and lemmatization?**

Stemming uses rule-based algorithms to chop suffixes and produce approximate root forms (sometimes resulting in non-words), while lemmatization uses a dictionary to produce the actual linguistic base form (lemma) of each word; stemming is faster and built into Elasticsearch token filters, whereas lemmatization requires external plugins or preprocessing.

### 11. **How does the synonym token filter work?**

The synonym token filter expands or replaces tokens based on a synonym mapping (e.g., "laptop" → "notebook, portable computer") defined in a file or inline, so that a search for any synonym matches documents containing any of the related terms; synonyms can be applied at index time, search time, or both.

### 12. **What does the `ascii_folding` token filter do?**

The ascii_folding token filter converts Unicode characters to their ASCII equivalents (e.g., "café" becomes "cafe", "über" becomes "uber"), which is useful for making searches accent-insensitive so that users do not need to type diacritics to find matching documents.

### 13. **How do you create a custom analyzer?**

Define a custom analyzer in the index settings under analysis.analyzer with a type of "custom", specifying a char_filter array, a tokenizer, and a filter array of token filters; each referenced component must either be a built-in name or defined in the corresponding analysis.char_filter, analysis.tokenizer, or analysis.filter section of the settings.

### 14. **What is the `_analyze` API used for?**

The _analyze API lets you test how text is processed by a specific analyzer, tokenizer, or set of filters by returning the list of tokens produced from the input text along with their positions and offsets, which is invaluable for debugging analysis chains and understanding why searches match or miss.

### 15. **What are multi-fields?**

Multi-fields use the fields mapping parameter to index the same source value in multiple ways; for example, a title field can be analyzed as text for full-text search and also stored as a keyword sub-field for sorting and aggregations, and each sub-field can use a different analyzer without duplicating data in the source document.

### 16. **What is the inverted index structure?**

The inverted index is a mapping from each unique analyzed term to a postings list containing the document IDs, term frequencies, and position offsets where that term appears, enabling Elasticsearch to find all documents matching a term in constant time and supporting phrase queries and proximity matching through positional data.

### 17. **When should you use a `text` field versus a `keyword` field?**

Use a text field when you need full-text search with analysis, scoring, and matching on individual terms, and use a keyword field when you need exact-value matching, sorting, aggregations, or case-sensitive filtering; many string fields benefit from being mapped as both using the multi-field approach.

### 18. **What are position offsets and why are they important?**

Position offsets record the start and end character positions of each token in the original text, and term positions record the sequential order of tokens; these are stored in the inverted index and used by phrase queries for term-order matching, by the highlighter for locating matched text in the source, and by the fast vector highlighter for efficient fragment extraction.

### 19. **What is the pattern tokenizer?**

The pattern tokenizer splits text using a regular expression pattern, where the default pattern splits on non-word characters (\W+); it is useful when your text has a custom delimiter that does not match any built-in tokenizer's splitting rules, though it is slower than the standard or whitespace tokenizers.

### 20. **How does the mapping character filter work?**

The mapping character filter replaces specified character sequences with other sequences before tokenization (e.g., replacing ":)" with "_happy_" or "+" with " plus "), which is useful for normalizing emoticons, special symbols, or domain-specific notations into searchable text.
