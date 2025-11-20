# Rate Limiting System Design

Design a rate limiting system that controls the rate of requests from clients to prevent abuse and ensure fair resource usage.

## Requirements

### Functional Requirements
- Limit number of requests per time window
- Support multiple rate limiting algorithms
- Different limits for different users/APIs
- Return appropriate HTTP status codes (429 Too Many Requests)
- Provide rate limit headers in response

### Non-Functional Requirements
- Low latency overhead (< 1ms)
- High throughput (millions of requests per second)
- Distributed rate limiting across multiple servers
- Accurate rate limiting (no race conditions)

## Rate Limiting Algorithms

### Fixed Window Counter
- Divide time into fixed windows (e.g., 1 minute)
- Count requests in current window
- Reset counter at window boundary
- Simple but allows bursts at window boundaries

**Example**: 100 requests per minute
- Window 1 (0:00-0:59): Count requests
- Window 2 (1:00-1:59): Reset and count again

### Sliding Window Log
- Track timestamp of each request
- Count requests in sliding window
- More accurate but memory intensive
- O(n) memory where n is number of requests

### Sliding Window Counter
- Combine fixed window with sliding window
- Use multiple fixed windows to approximate sliding
- Balance between accuracy and memory
- O(1) memory complexity

### Token Bucket
- Tokens added to bucket at fixed rate
- Request consumes one token
- Request allowed if tokens available
- Allows bursts up to bucket capacity

**Example**: 10 tokens/second, bucket size 100
- Tokens refill at 10/second
- Can burst 100 requests immediately
- Then limited to 10/second

### Leaky Bucket
- Requests added to bucket
- Requests processed at fixed rate
- Requests rejected if bucket full
- Smooths out traffic spikes

## High-Level Design

```
┌─────────┐     ┌──────────────┐     ┌─────────────┐
│ Client  │────▶│ Load Balancer│────▶│ API Gateway │
└─────────┘     └──────────────┘     └──────┬──────┘
                                             │
                                             ▼
                                    ┌─────────────────┐
                                    │  Rate Limiter   │
                                    └────────┬────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    ▼                        ▼                        ▼
            ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
            │   Redis      │        │   Redis      │        │   Redis      │
            │  (Distributed│        │  (Distributed│        │  (Distributed│
            │   Counter)   │        │   Counter)   │        │   Counter)   │
            └──────────────┘        └──────────────┘        └──────────────┘
```

## Detailed Design

### Distributed Rate Limiting

**Option 1: Centralized Redis**
- All servers check same Redis instance
- Accurate but single point of failure
- Network latency for each check

**Option 2: Distributed Counters**
- Each server maintains local counter
- Periodically sync with central store
- Faster but less accurate
- May allow slightly more requests

**Option 3: Consistent Hashing**
- Hash client identifier to specific Redis node
- Each client always hits same node
- Better distribution and accuracy

### Rate Limit Key Strategy

**Per-IP Rate Limiting**
- Key: `rate_limit:ip:{ip_address}`
- Simple but affected by NAT/proxies
- Good for DDoS protection

**Per-User Rate Limiting**
- Key: `rate_limit:user:{user_id}`
- More accurate for authenticated users
- Requires user identification

**Per-API Key Rate Limiting**
- Key: `rate_limit:api_key:{api_key}`
- Different limits for different API tiers
- Common in API services

**Per-Endpoint Rate Limiting**
- Key: `rate_limit:endpoint:{endpoint}:{identifier}`
- Different limits for different endpoints
- More granular control

### Redis Implementation (Token Bucket)

```python
def check_rate_limit(user_id, limit, window):
    key = f"rate_limit:{user_id}"
    current = redis.incr(key)
    
    if current == 1:
        redis.expire(key, window)
    
    if current > limit:
        return False  # Rate limit exceeded
    
    return True  # Within limit
```

### Sliding Window Log Implementation

```python
def check_rate_limit(user_id, limit, window):
    key = f"rate_limit:{user_id}"
    now = time.time()
    
    # Remove old entries
    redis.zremrangebyscore(key, 0, now - window)
    
    # Count current requests
    current = redis.zcard(key)
    
    if current >= limit:
        return False
    
    # Add current request
    redis.zadd(key, {str(uuid.uuid4()): now})
    redis.expire(key, window)
    
    return True
```

## Rate Limit Headers

Standard HTTP headers to inform clients about rate limits:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1609459200
Retry-After: 60
```

## Handling Rate Limit Exceeded

### HTTP 429 Too Many Requests
- Standard status code for rate limiting
- Include `Retry-After` header
- Return error message with details

### Response Body
```json
{
  "error": "Rate limit exceeded",
  "message": "You have exceeded the rate limit of 100 requests per minute",
  "retry_after": 60
}
```

## Performance Optimizations

### Caching
- Cache rate limit status in local memory
- Reduce Redis calls
- Trade-off: Slightly less accurate

### Batch Operations
- Batch multiple rate limit checks
- Use Redis pipeline
- Reduce network round trips

### Lazy Evaluation
- Only check rate limit when needed
- Skip for internal/system requests
- Whitelist certain IPs/users

## Rate Limiting Strategies by Use Case

### API Rate Limiting
- Per-API key limits
- Tiered limits (free, paid, enterprise)
- Different limits per endpoint

### DDoS Protection
- Per-IP limits
- Aggressive limits for suspicious IPs
- Geographic-based limits

### User Action Limiting
- Per-user limits
- Prevent spam/abuse
- Different limits for different actions

### Resource Protection
- Per-resource limits
- Protect expensive operations
- Prevent resource exhaustion

## Monitoring and Metrics

### Key Metrics
- Rate limit violations per second
- Requests per client/IP
- Top rate-limited clients
- False positive rate
- Rate limiter latency

### Alerts
- Sudden spike in rate limit violations
- High rate limiter latency
- Redis connection failures
- Unusual traffic patterns

## Scaling Considerations

### Horizontal Scaling
- Use distributed Redis cluster
- Consistent hashing for key distribution
- Handle Redis failover gracefully

### Vertical Scaling
- Increase Redis memory
- Use Redis Cluster for sharding
- Optimize Redis configuration

## Use Cases

1. **API Protection**: Prevent API abuse and ensure fair usage
2. **DDoS Mitigation**: Protect against distributed attacks
3. **Cost Control**: Limit expensive operations
4. **Fair Resource Sharing**: Ensure equal access for all users
5. **Spam Prevention**: Limit user actions (comments, posts)

## Popular Rate Limiting Solutions

- **Redis**: Distributed counters and sliding windows
- **NGINX**: Built-in rate limiting module
- **AWS API Gateway**: Managed rate limiting
- **Kong**: API gateway with rate limiting
- **Envoy Proxy**: Service mesh with rate limiting

