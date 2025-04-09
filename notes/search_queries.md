## Search Queries

Elasticsearch enables highly customizable search queries that allow for precise filtering and relevance scoring. In the following query example, we combine **filtering** to narrow results to relevant categories and **boosting** to prioritize specific fields, improving both efficiency and relevance in search results.

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

### Real-World Examples

#### Scenario 1: Movie Database Search with Genre Filtering and Title Boosting

In a movie database, users often search for films within specific genres while placing greater importance on titles. A user searching for "action" films with the phrase "Fast and Furious" would benefit from filtering on genre while prioritizing matches in the title field.

**Modified Query**:

```json
{
  "query": {
    "filtered": {
      "filter": {
        "term": { "genre": "action" }
      },
      "query": {
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

**Example Response**:
Movies with "Fast and Furious" in the title field rank higher than those with the term only in the description, helping users find relevant movies quickly.

#### Scenario 2: Real Estate Listings Search with Location Filtering and Title Boosting

On a real estate platform, users might want to search for properties by location with additional criteria for key terms. For instance, a user may search for properties in `"New York"` containing the phrase "penthouse" in the title or description, with greater importance on the title.

**Modified Query**:

```json
{
  "query": {
    "filtered": {
      "filter": {
        "term": { "location": "New York" }
      },
      "query": {
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

**Example Response**:
Listings with "penthouse" in the title rank higher than those with it only in the description, optimizing the search experience by bringing prominent listings to the top.

#### Scenario 3: Job Listings Portal with Skill Tag Filtering and Title Boosting

In a job search portal, users may want to filter jobs by required skills and job title. For instance, a user might search for positions tagged with `"Data Science"` that include `"Machine Learning"` in the job title or description.

**Modified Query**:

```json
{
  "query": {
    "filtered": {
      "filter": {
        "term": { "skills": "Data Science" }
      },
      "query": {
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

**Example Response**:
Job postings with "Machine Learning" in the title rank higher, helping users find the most relevant data science positions with a focus on machine learning skills.
