# Search Engine System Design

Design a search engine like Google that can index billions of web pages and return relevant results in milliseconds.

## Requirements

### Functional Requirements
- Crawl and index web pages
- Support full-text search
- Rank results by relevance
- Support advanced search (filters, date range, etc.)
- Autocomplete/suggestions
- Spell correction
- Image search (optional)

### Non-Functional Requirements
- Low latency (< 100ms for search results)
- High availability (99.9% uptime)
- Handle billions of documents
- Support millions of queries per second
- Fresh index (updated regularly)

## Capacity Estimation

### Scale Estimates
- 1 trillion web pages to index
- Average page size: 100KB
- Total storage: 100PB
- 5 billion searches per day
- Peak QPS: 100,000 queries per second
- Index update: 10 million pages per day

## System APIs

```
# Search
GET /api/v1/search?q=query&page=1&limit=10

Response: {
  "results": [
    {
      "title": "...",
      "url": "...",
      "snippet": "...",
      "rank": 1
    },
    ...
  ],
  "total": 1000000,
  "time": "0.05s"
}

# Autocomplete
GET /api/v1/autocomplete?q=quer

Response: {
  "suggestions": ["query", "question", "queries"]
}
```

## High-Level Design

```
┌─────────┐     ┌──────────────┐     ┌─────────────┐
│ Client  │────▶│ Load Balancer│────▶│ Query       │
└─────────┘     └──────────────┘     └──────┬──────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    ▼                        ▼                        ▼
            ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
            │  Index       │        │  Ranking     │        │  Cache       │
            │  Servers     │        │  Service     │        │  (Redis)     │
            └──────────────┘        └──────────────┘        └──────────────┘
                    │
                    ▼
            ┌──────────────┐
            │  Document    │
            │  Store       │
            └──────────────┘

                    ┌──────────────┐
                    │  Web Crawler │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  URL Frontier│
                    └──────────────┘
```

## Detailed Design

### Web Crawling

**Crawler Components:**
1. **URL Frontier**: Queue of URLs to crawl
2. **DNS Resolver**: Resolve domain names to IPs
3. **Fetcher**: Download web pages
4. **Parser**: Extract links and content
5. **Deduplicator**: Remove duplicate URLs

**Crawling Strategy:**
- **BFS (Breadth-First Search)**: Crawl by depth levels
- **Priority Queue**: Prioritize important pages
- **Polite Crawling**: Respect robots.txt, rate limiting
- **Distributed Crawling**: Multiple crawlers in parallel

**URL Frontier:**
- Separate queues by domain
- Rate limit per domain
- Prioritize important URLs
- Handle URL deduplication

### Indexing

**Inverted Index:**
- Map each word to list of documents containing it
- Example: "python" → [doc1, doc5, doc10]
- Enables fast full-text search

**Index Structure:**
```
Term: "python"
Posting List: [
  {doc_id: 1, frequency: 5, positions: [10, 25, 30]},
  {doc_id: 5, frequency: 3, positions: [15, 20]},
  ...
]
```

**Index Partitioning:**
- Partition by term (hash or alphabetical)
- Each partition on different server
- Enables parallel query processing

**Document Store:**
- Store full document content
- Key: doc_id, Value: {url, title, content, metadata}
- Use distributed storage (HDFS, S3)

### Query Processing

**Query Flow:**
1. Parse query into terms
2. Lookup terms in inverted index
3. Find documents containing all terms (AND) or any term (OR)
4. Rank documents by relevance
5. Return top K results

**Boolean Query:**
- AND: Documents containing all terms
- OR: Documents containing any term
- NOT: Exclude documents with term

**Phrase Query:**
- Match exact phrase
- Check term positions in documents
- "machine learning" must have "machine" followed by "learning"

### Ranking Algorithm

**TF-IDF (Term Frequency-Inverse Document Frequency)**
- TF: How often term appears in document
- IDF: How rare term is across all documents
- Score = TF × IDF
- Higher score = more relevant

**PageRank:**
- Measure page importance based on links
- Pages with more backlinks rank higher
- Recursive algorithm (iterative computation)

**Combined Ranking:**
- Score = α × TF-IDF + β × PageRank + γ × Other factors
- Tune weights based on user feedback
- Machine learning models for better ranking

### Distributed Index

**Sharding Strategy:**
- **Term-based Sharding**: Hash term to shard
- **Document-based Sharding**: Hash doc_id to shard
- **Hybrid**: Both term and document sharding

**Query Processing:**
1. Query coordinator receives request
2. Determines which shards contain query terms
3. Sends query to relevant shards in parallel
4. Merges results from all shards
5. Ranks and returns top results

### Caching

**Query Result Cache:**
- Cache popular queries
- Store top 100 results per query
- TTL: 1 hour (balance freshness and performance)

**Index Cache:**
- Cache frequently accessed posting lists
- Keep hot terms in memory
- Reduce disk I/O

### Autocomplete

**Trie Data Structure:**
- Prefix tree for fast prefix matching
- Store top K completions per prefix
- Update based on query frequency

**Implementation:**
- Build trie from popular queries
- On input "quer", traverse to node
- Return top completions from that node
- Update weights based on user selections

### Spell Correction

**Edit Distance:**
- Calculate Levenshtein distance
- Find words with small edit distance
- Rank by frequency and distance

**N-gram Matching:**
- Break query into n-grams
- Match against n-grams in dictionary
- Faster than edit distance for large dictionaries

### Freshness

**Incremental Indexing:**
- Update index with new/changed pages
- Don't rebuild entire index
- Merge new postings with existing index

**Index Versioning:**
- Maintain multiple index versions
- Serve queries from current version
- Build new version in background
- Switch atomically when ready

## Performance Optimizations

### Compression
- Compress posting lists
- Use variable-length encoding
- Reduce storage and I/O

### Skip Lists
- Add skip pointers in posting lists
- Skip irrelevant documents quickly
- Faster intersection of posting lists

### Early Termination
- Stop processing after finding top K results
- Don't process entire posting list
- Significant performance improvement

## Monitoring and Metrics

### Key Metrics
- Query latency (p50, p95, p99)
- Index size and growth rate
- Crawl rate and coverage
- Cache hit ratio
- Query throughput

### Alerts
- High query latency
- Index build failures
- Crawler errors
- Low cache hit ratio

## Scaling Strategies

### Horizontal Scaling
- Add more index servers
- Add more crawlers
- Distribute load across machines

### Vertical Scaling
- Increase memory for caching
- Faster CPUs for ranking
- More storage for index

## Use Cases

1. **Web Search**: General web page search
2. **Site Search**: Search within specific website
3. **Document Search**: Search in document collections
4. **Product Search**: E-commerce product search
5. **News Search**: Real-time news article search

## Popular Search Engine Technologies

- **Elasticsearch**: Distributed search and analytics
- **Apache Solr**: Enterprise search platform
- **Google Search**: Proprietary large-scale search
- **Apache Lucene**: Search library (used by Solr/Elasticsearch)
- **Sphinx**: Full-text search engine

