# URL Shortener System Design

Design a URL shortener service like bit.ly or TinyURL that converts long URLs into short, shareable links.

## Requirements

### Functional Requirements
- Shorten a long URL to a short URL
- Redirect short URL to original long URL
- Support custom short URLs (optional)
- Set expiration time for URLs (optional)
- Analytics: track click counts

### Non-Functional Requirements
- High availability
- Low latency for redirection
- Short URLs should be as short as possible
- System should be scalable to handle billions of URLs

## Capacity Estimation

### Traffic Estimates
- 100M URLs shortened per month
- Read:Write ratio = 100:1 (100 reads per write)
- 100M writes/month = ~40 writes/second
- 10B reads/month = ~3,800 reads/second

### Storage Estimates
- Average URL length: 500 characters
- Short URL length: 7 characters (using base62 encoding)
- Storage per URL: ~500 bytes
- 100M URLs × 500 bytes = 50GB per month
- 5 years of data: ~3TB

## System APIs

```
POST /api/v1/shorten
{
  "long_url": "https://example.com/very/long/url",
  "custom_alias": "optional_custom_alias",
  "expiration_date": "2024-12-31"
}

Response:
{
  "short_url": "https://short.ly/abc123",
  "expiration_date": "2024-12-31"
}

GET /api/v1/{short_url}
Response: 301 Redirect to original URL
```

## Database Design

### URL Table
```
- id: BIGINT (primary key)
- short_code: VARCHAR(7) (unique index)
- long_url: VARCHAR(2048)
- created_at: TIMESTAMP
- expiration_date: TIMESTAMP
- user_id: BIGINT (optional)
```

### Analytics Table
```
- id: BIGINT
- short_code: VARCHAR(7)
- clicked_at: TIMESTAMP
- ip_address: VARCHAR(45)
- user_agent: VARCHAR(255)
- referrer: VARCHAR(255)
```

## High-Level Design

```
┌─────────┐     ┌──────────────┐     ┌─────────────┐
│ Client  │────▶│ Load Balancer│────▶│ API Servers │
└─────────┘     └──────────────┘     └─────────────┘
                                              │
                                              ▼
                                    ┌─────────────────┐
                                    │  Application    │
                                    │  Logic Layer    │
                                    └─────────────────┘
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    ▼                         ▼                         ▼
            ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
            │   Cache      │         │   Database   │         │  Key Gen     │
            │   (Redis)    │         │   (MySQL/    │         │  Service     │
            │              │         │   Cassandra) │         │              │
            └──────────────┘         └──────────────┘         └──────────────┘
```

## Detailed Design

### URL Encoding
- Use base62 encoding (a-z, A-Z, 0-9) = 62 characters
- 7 characters = 62^7 = ~3.5 trillion unique URLs
- Algorithm: Convert auto-incrementing ID to base62

### Key Generation Service
**Option 1: Single Machine**
- Use auto-incrementing database ID
- Convert to base62

**Option 2: Distributed (Recommended)**
- Use range-based approach: Each server gets a range
- Server 1: 1-1M, Server 2: 1M-2M, etc.
- Use Zookeeper/etcd to manage ranges

**Option 3: UUID-based**
- Generate UUID, take first 7 characters
- Handle collisions with retry logic

### Caching Strategy
- Cache popular URLs in Redis
- Use LRU eviction policy
- Cache 20% of daily traffic = ~200M URLs
- Memory needed: 200M × 500 bytes = 100GB

### Database Sharding
- Shard by short_code hash
- Use consistent hashing for even distribution
- Each shard handles a subset of URLs

## Scaling Considerations

### Read Scaling
- Use read replicas for database
- Cache frequently accessed URLs
- Use CDN for static assets

### Write Scaling
- Use database sharding
- Pre-generate short codes in batches
- Use message queue for async processing

### Handling Hot URLs
- Cache popular URLs in multiple cache servers
- Use consistent hashing for cache distribution

## Security Considerations
- Prevent abuse with rate limiting
- Validate URLs to prevent malicious links
- Use HTTPS for all redirects
- Implement CAPTCHA for suspicious activity

## Monitoring and Analytics
- Track click counts per URL
- Monitor redirect latency
- Track geographic distribution of clicks
- Alert on system errors and high latency

