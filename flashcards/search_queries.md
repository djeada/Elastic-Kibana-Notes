### 1. **What is the Query DSL in Elasticsearch?**

The Query DSL (Domain Specific Language) is a JSON-based language used to define search queries, combining leaf queries that match on individual fields (match, term, range) with compound queries (bool, dis_max, boosting) that combine multiple clauses, and supporting filters, aggregations, sorting, pagination, and highlighting in a single request body.

### 2. **What is the difference between leaf queries and compound queries?**

Leaf queries operate on a single field to check for a specific value or condition (e.g., match, term, range, exists), while compound queries combine multiple leaf or compound queries using boolean logic (bool), disjunction (dis_max), or score adjustment (boosting, constant_score) to build complex search expressions.

### 3. **How does the match query work?**

The match query analyzes the input text using the same analyzer as the target field, then searches for any of the resulting tokens in the inverted index; by default it uses OR logic between tokens so a document matching any token is returned, but you can set operator to "and" to require all tokens to match.

### 4. **When should you use a term query instead of a match query?**

Use a term query when searching keyword, numeric, date, or boolean fields for an exact value without analysis, because the term query does not analyze the input; never use a term query on a text field since the stored tokens will not match the unanalyzed input, leading to unexpected zero results.

### 5. **What is a range query?**

A range query matches documents where a numeric, date, or keyword field falls within a specified range defined by gt (greater than), gte (greater than or equal), lt (less than), and lte (less than or equal) parameters, and is commonly used for date filtering, price ranges, or age brackets.

### 6. **What is a bool query and what are its clauses?**

A bool query combines multiple query clauses: must clauses that must match and contribute to the score, filter clauses that must match but do not score and are cached, should clauses that optionally match and boost the score, and must_not clauses that exclude matching documents without scoring; this is the most commonly used compound query.

### 7. **What is the difference between `must` and `filter` in a bool query?**

Both must and filter require matching documents, but must calculates a relevance score for each match while filter does not score and its results are cached by the query cache; use filter for exact-value conditions (status, date ranges, IDs) and must for full-text relevance searches.

### 8. **How does the multi_match query work?**

The multi_match query runs a match query across multiple fields simultaneously, supporting strategies like best_fields (score from the best matching field), most_fields (combine scores from all matching fields), and cross_fields (analyze the query as if all fields were one), with optional per-field boosting via the caret (^) syntax.

### 9. **What is field boosting?**

Field boosting multiplies the relevance score contribution of a particular field using the caret syntax (e.g., title^3) so that matches in boosted fields rank higher in results; it is applied within multi_match, bool, or dis_max queries to emphasize certain fields like title or name over less important fields like description.

### 10. **What is the match_phrase query and the slop parameter?**

The match_phrase query matches documents containing the exact sequence of analyzed tokens in order, and the slop parameter allows a specified number of positional moves (token swaps or insertions) between terms, so a slop of 1 lets "quick fox" match "quick brown fox" by tolerating one intervening term.

### 11. **What is a fuzzy query?**

A fuzzy query matches terms that are within a specified edit distance (Levenshtein distance) of the search term, handling typos and minor misspellings; the fuzziness parameter can be set to AUTO (which varies tolerance by term length), or to a fixed integer like 1 or 2, and it can also be used within match queries via the fuzziness option.

### 12. **Why are leading wildcard queries expensive?**

A query starting with an asterisk (e.g., *foo) cannot use the inverted index efficiently because there is no known prefix to look up, forcing Elasticsearch to scan every term in the field across every segment, which is extremely slow on large indices; prefer ngram or edge_ngram tokenizers if prefix or infix matching is needed.

### 13. **How does pagination work with `from` and `size`?**

The from parameter specifies the starting offset and size specifies the number of documents to return; Elasticsearch must still fetch from + size documents from every shard and sort them on the coordinating node, so deep pagination (large from values) becomes increasingly expensive and should be avoided in favor of search_after for scrolling through large result sets.

### 14. **What is `search_after` and when should it be used?**

search_after provides efficient deep pagination by using the sort values of the last document in the previous page as the starting point for the next page, avoiding the cost of skipping over from documents; it requires a deterministic sort order (usually including _id as a tiebreaker) and replaces the scroll API for live pagination use cases.

### 15. **How does highlighting work in search results?**

Highlighting identifies the portions of a field's text that match the query and wraps them in configurable tags (e.g., <em>), returning these fragments in a highlight section of each hit; Elasticsearch supports three highlighter types—unified (default, best for most cases), plain, and fvh (fast vector highlighter for large fields).

### 16. **What is source filtering?**

Source filtering controls which fields from the _source document are included in search results using _source_includes and _source_excludes parameters, reducing network transfer and client-side parsing overhead when you only need a subset of fields rather than the entire document.

### 17. **What is the exists query?**

The exists query matches documents where the specified field has any non-null value, which is useful for finding documents with or without optional fields; combined with must_not in a bool query, it effectively finds documents where a field is missing or null.

### 18. **How does the two-phase search execution work?**

In the query phase each shard returns the top N document IDs and sort values to the coordinating node which merges and sorts them globally, then in the fetch phase the coordinating node retrieves the full _source and stored fields for only the final result set, minimizing data transfer by fetching full documents only for hits that will actually be returned.

### 19. **What is the dis_max query?**

The dis_max (disjunction max) query runs multiple sub-queries and scores each document using the highest score from any single sub-query rather than summing scores, with an optional tie_breaker parameter that adds a fraction of the other matching sub-query scores; it is useful when you want the best single-field match to dominate ranking.

### 20. **What is the `minimum_should_match` parameter?**

The minimum_should_match parameter specifies how many should clauses in a bool query (or tokens in a match query) must match for a document to be returned; it can be an absolute number, a percentage, or a combination expression, giving fine-grained control over recall versus precision in search results.
