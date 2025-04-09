## Task 9: Modeling Relational Data with Nested Objects and Parent‑Child Relationships

**Objectives:**
- Model complex relationships using nested fields and join (parent‑child) relationships.
- Query these relationships using nested queries and parent‑child queries.
- Understand how data modeling impacts both indexing and search performance.

**Lab Steps:**

1. **Nested Mapping:**  
   Create an index `library` with nested objects for reviews:
   ```json
   PUT /library
   {
     "mappings": {
       "properties": {
         "title": { "type": "text" },
         "reviews": {
           "type": "nested",
           "properties": {
             "reviewer": { "type": "keyword" },
             "rating": { "type": "integer" },
             "comment": { "type": "text" }
           }
         }
       }
     }
   }
   ```
2. **Parent‑Child Relationships with Join Field:**  
   Create an index `books` that relates authors (parent) with books (child):
   ```json
   PUT /books
   {
     "mappings": {
       "properties": {
         "my_join_field": {
           "type": "join",
           "relations": {
             "author": "book"
           }
         },
         "name": { "type": "text" }
       }
     }
   }
   ```
   Index a parent document:
   ```json
   POST /books/_doc/1
   {
     "name": "George Orwell",
     "my_join_field": "author"
   }
   ```
   Index a child document (using routing):
   ```json
   POST /books/_doc/2?routing=1
   {
     "name": "1984",
     "my_join_field": {
       "name": "book",
       "parent": "1"
     }
   }
   ```
3. **Querying Relationships:**  
   For nested objects, run a nested query:
   ```json
   GET /library/_search
   {
     "query": {
       "nested": {
         "path": "reviews",
         "query": {
           "bool": {
             "must": [
               { "match": { "reviews.reviewer": "John Doe" } },
               { "range": { "reviews.rating": { "gte": 4 } } }
             ]
           }
         }
       }
     }
   }
   ```
   For parent-child, execute a `has_parent` query:
   ```json
   GET /books/_search
   {
     "query": {
       "has_parent": {
         "parent_type": "author",
         "query": { "match": { "name": "George Orwell" } }
       }
     }
   }
   ```
4. **Reflection:**  
   Discuss the pros and cons of nested versus parent-child modeling in your lab journal.
