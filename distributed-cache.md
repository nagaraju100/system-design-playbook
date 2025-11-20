# Distributed Cache System Design

Design a distributed caching system like Redis or Memcached that provides high-performance data access across multiple servers.

## Requirements

### Functional Requirements
- Store key-value pairs in memory
- Support TTL (Time To Live) for cache entries
- Support different data types (strings, lists, sets, hashes)
- Support cache eviction policies (LRU, LFU, FIFO)

### Non-Functional Requirements
- High availability (99.9% uptime)
- Low latency (< 1ms for cache hits)
- High throughput (millions of requests per second)
- Scalability (add/remove nodes dynamically)
- Data consistency across nodes

## Capacity Estimation

### Traffic Estimates
- 100M requests per day
- Average request size: 1KB
- Cache hit ratio: 80%
- Total cache size: 100GB
- Peak QPS: 10,000 requests/second

## System APIs

```
GET /cache/{key}
Response: { "value": "...", "ttl": 3600 }

PUT /cache/{key}
Body: { "value": "...", "ttl": 3600 }

DELETE /cache/{key}

POST /cache/batch
Body: { "keys": ["key1", "key2", ...] }
```

## High-Level Design

```
┌─────────┐     ┌──────────────┐     ┌─────────────┐
│ Client  │────▶│ Load Balancer│────▶│ Cache Nodes │
└─────────┘     └──────────────┘     └─────────────┘
                                              │
                                              ▼
                                    ┌─────────────────┐
                                    │  Consistent     │
                                    │  Hashing        │
                                    └─────────────────┘
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    ▼                         ▼                         ▼
            ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
            │  Cache Node  │         │  Cache Node  │         │  Cache Node  │
            │     1        │         │     2        │         │     3        │
            └──────────────┘         └──────────────┘         └──────────────┘
```

## Detailed Design

### Consistent Hashing
- Distribute keys across cache nodes using consistent hashing
- Each node is assigned multiple virtual nodes on the hash ring
- When a node is added/removed, only a small portion of keys need to be rehashed
- Reduces cache misses during node failures or scaling

### Cache Eviction Policies

**LRU (Least Recently Used)**
- Evict least recently accessed items
- Use doubly-linked list + hash map for O(1) operations
- Good for temporal locality patterns

**LFU (Least Frequently Used)**
- Evict least frequently accessed items
- Track access frequency for each key
- Good for long-term popularity patterns

**TTL-based Eviction**
- Automatically expire entries after TTL
- Use priority queue or sorted set for efficient expiration

### Replication Strategy

**Option 1: Master-Slave Replication**
- One master node handles writes
- Multiple slave nodes handle reads
- Asynchronous replication for better performance

**Option 2: Multi-Master Replication**
- All nodes can handle reads and writes
- Use conflict resolution strategies
- Better availability but more complex

### Cache-Aside Pattern
```
1. Application checks cache for data
2. If cache miss, fetch from database
3. Store data in cache for future requests
4. Return data to application
```

### Write-Through Pattern
```
1. Application writes to cache
2. Cache immediately writes to database
3. Return success to application
```

### Write-Behind Pattern
```
1. Application writes to cache
2. Cache returns success immediately
3. Cache asynchronously writes to database
```

## Handling Failures

### Node Failure
- Use consistent hashing to redistribute keys
- Replicate data to multiple nodes
- Use health checks to detect failures
- Automatic failover to replica nodes

### Network Partition
- Continue serving cached data
- Mark stale data appropriately
- Reconcile when partition heals

### Cache Stampede Prevention
- Use mutex/locks for cache misses
- Implement exponential backoff
- Pre-warm cache for popular items

## Performance Optimizations

### Memory Management
- Use memory pools to reduce allocation overhead
- Compress large values
- Implement memory limits per node

### Network Optimization
- Use binary protocols (Redis Protocol, Memcached Protocol)
- Batch multiple operations
- Use pipelining for multiple requests

### Connection Pooling
- Maintain persistent connections
- Reuse connections across requests
- Limit connection pool size

## Monitoring and Metrics

### Key Metrics
- Cache hit ratio
- Average response time
- Memory usage per node
- Network bandwidth usage
- Error rates

### Alerts
- Cache hit ratio drops below threshold
- High latency (> 5ms)
- Node failures
- Memory usage > 90%

## Scaling Strategies

### Horizontal Scaling
- Add more cache nodes
- Redistribute keys using consistent hashing
- No downtime required

### Vertical Scaling
- Increase memory per node
- Upgrade CPU for better performance
- Limited by single machine capacity

## Use Cases

1. **Session Storage**: Store user sessions in distributed cache
2. **Database Query Caching**: Cache frequently accessed database queries
3. **API Response Caching**: Cache API responses to reduce backend load
4. **Rate Limiting**: Track request counts per user/IP
5. **Leaderboards**: Store real-time leaderboard data

