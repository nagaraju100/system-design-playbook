# Social Media Feed System Design

Design a social media feed system like Twitter, Facebook, or Instagram that shows personalized content to users in real-time.

## Requirements

### Functional Requirements
- Users can post content (text, images, videos)
- Users can follow other users
- Generate personalized feed for each user
- Support for likes, comments, shares
- Real-time updates in feed
- Support for trending topics

### Non-Functional Requirements
- Low latency (< 200ms for feed generation)
- High availability (99.9% uptime)
- Handle millions of users and posts
- Support millions of reads per second
- Feed freshness (show recent content)

## Capacity Estimation

### Traffic Estimates
- 500 million daily active users
- Average 5 posts per user per day
- Total: 2.5 billion posts per day
- Average 200 follows per user
- Feed reads: 10 per user per day = 5 billion reads/day
- Peak QPS: 100,000 reads/second, 30,000 writes/second

## System APIs

```
# Create post
POST /api/v1/posts
Body: {
  "user_id": "user123",
  "content": "Hello world!",
  "media_urls": [],
  "visibility": "public"
}

# Get user feed
GET /api/v1/feeds/{user_id}?limit=20&cursor={cursor}

# Follow user
POST /api/v1/users/{user_id}/follow
Body: {
  "target_user_id": "user456"
}

# Like post
POST /api/v1/posts/{post_id}/like
```

## High-Level Design

```
┌─────────┐     ┌──────────────┐     ┌─────────────┐
│ Client  │────▶│ Load Balancer│────▶│ API Servers │
└─────────┘     └──────────────┘     └──────┬──────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    ▼                        ▼                        ▼
            ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
            │  Feed        │        │  Post        │        │  Social      │
            │  Service     │        │  Service     │        │  Graph       │
            └──────────────┘        └──────────────┘        └──────────────┘
                    │                        │                        │
                    └────────────────────────┼────────────────────────┘
                                             ▼
                                    ┌─────────────────┐
                                    │   Database      │
                                    │   (Posts, Users,│
                                    │    Follows)     │
                                    └─────────────────┘
```

## Feed Generation Strategies

### Pull Model (Fan-out on Read)

**How it works:**
- When user requests feed, fetch posts from all followed users
- Merge and rank posts in real-time
- Return top N posts

**Pros:**
- Simple implementation
- Always shows latest posts
- No storage overhead for feeds

**Cons:**
- Slow for users with many follows
- High database load on read
- Not scalable for power users

**Use case:** Small number of follows, read-heavy workload

### Push Model (Fan-out on Write)

**How it works:**
- When user posts, push to all followers' feed caches
- User's feed pre-computed and stored
- Read feed from cache (fast)

**Pros:**
- Fast feed reads (from cache)
- Low latency
- Good for read-heavy workload

**Cons:**
- High write cost (fan-out to all followers)
- Storage overhead (store feed for each user)
- Slow for users with many followers (celebrities)

**Use case:** Small number of followers, write-heavy workload

### Hybrid Model (Recommended)

**How it works:**
- Push model for users with < 1000 followers
- Pull model for users with > 1000 followers (celebrities)
- Combine both in feed generation

**Implementation:**
1. Pre-compute feed for regular users (push)
2. For celebrities, fetch posts on-demand (pull)
3. Merge both in feed generation
4. Cache result

## Database Schema

**Users Table**
```
- user_id: BIGINT (primary key)
- username: VARCHAR(50)
- email: VARCHAR(255)
- created_at: TIMESTAMP
- follower_count: INT
- following_count: INT
```

**Posts Table**
```
- post_id: BIGINT (primary key)
- user_id: BIGINT
- content: TEXT
- media_urls: JSON
- created_at: TIMESTAMP
- like_count: INT
- comment_count: INT
- INDEX (user_id, created_at)
```

**Follows Table**
```
- follower_id: BIGINT
- followee_id: BIGINT
- created_at: TIMESTAMP
- PRIMARY KEY (follower_id, followee_id)
- INDEX (followee_id)  # For getting followers
```

**Feed Table (Push Model)**
```
- user_id: BIGINT
- post_id: BIGINT
- post_user_id: BIGINT
- created_at: TIMESTAMP
- score: FLOAT  # For ranking
- PRIMARY KEY (user_id, post_id)
- INDEX (user_id, score DESC, created_at DESC)
```

**Likes Table**
```
- user_id: BIGINT
- post_id: BIGINT
- created_at: TIMESTAMP
- PRIMARY KEY (user_id, post_id)
```

## Feed Ranking

### Ranking Factors

**Recency:**
- Newer posts rank higher
- Exponential decay: score = e^(-λt)
- λ controls decay rate

**Engagement:**
- Likes, comments, shares
- Higher engagement = higher rank
- Normalize by post age

**User Relationship:**
- Close friends' posts rank higher
- Based on interaction history
- Mutual follows boost score

**Content Type:**
- Videos may rank higher than text
- Images vs. text preferences
- User preferences matter

### Ranking Algorithm

```
score = w1 × recency_score + 
        w2 × engagement_score + 
        w3 × relationship_score + 
        w4 × content_score
```

Tune weights using A/B testing and machine learning.

## Detailed Design

### Post Creation Flow

1. User creates post via API
2. Store post in database
3. If user has < 1000 followers:
   - Fan-out to all followers' feed caches
   - Update feed table for each follower
4. If user has > 1000 followers:
   - Skip fan-out (use pull model)
5. Invalidate user's own feed cache
6. Return success

### Feed Retrieval Flow

1. User requests feed
2. Check cache for pre-computed feed
3. If cache hit:
   - Return cached feed
4. If cache miss:
   - Fetch from feed table (push model)
   - Fetch from followed celebrities (pull model)
   - Merge and rank posts
   - Cache result
   - Return feed

### Real-Time Updates

**WebSocket Connection:**
- Maintain persistent connection
- Push new posts to followers in real-time
- Update feed without refresh

**Polling:**
- Client polls for new posts
- Simpler but less efficient
- Higher latency

## Caching Strategy

### Feed Cache
- Cache top 100 posts per user
- TTL: 5 minutes
- Invalidate on new post from followed users

### Post Cache
- Cache individual posts
- TTL: 1 hour
- Reduce database load

### Social Graph Cache
- Cache follow relationships
- TTL: 1 day
- Fast lookup for feed generation

## Scaling Strategies

### Database Sharding

**Shard by User ID:**
- Distribute users across shards
- Each shard handles subset of users
- Challenge: Cross-shard queries for follows

**Shard by Post ID:**
- Distribute posts across shards
- Better for post storage
- Need to aggregate across shards for feed

### Read Replicas
- Use read replicas for feed reads
- Reduce load on primary database
- Eventual consistency acceptable

### CDN for Media
- Store images/videos in CDN
- Faster content delivery
- Reduce origin server load

## Performance Optimizations

### Pagination
- Use cursor-based pagination
- More efficient than offset-based
- Consistent results

### Lazy Loading
- Load feed incrementally
- Load more on scroll
- Better initial load time

### Batch Operations
- Batch fan-out operations
- Use message queue for async processing
- Reduce write latency

## Trending Topics

### Algorithm
- Track hashtag/post frequency over time
- Calculate velocity (rate of increase)
- Rank by velocity and current frequency
- Update every few minutes

### Implementation
- Use time-windowed counters
- Store in Redis for fast access
- Update asynchronously

## Monitoring and Metrics

### Key Metrics
- Feed generation latency
- Post creation latency
- Cache hit ratio
- Fan-out latency
- Active users
- Posts per second

### Alerts
- High feed generation latency
- Low cache hit ratio
- Fan-out failures
- Database connection issues

## Use Cases

1. **Personal Feed**: Chronological or ranked feed
2. **Hashtag Feed**: Posts with specific hashtag
3. **User Profile**: All posts from specific user
4. **Trending Feed**: Popular/trending content
5. **Explore Feed**: Discover new content

## Popular Social Media Architectures

- **Twitter**: Hybrid model (push for regular users, pull for celebrities)
- **Facebook**: Push model with smart ranking
- **Instagram**: Push model with heavy caching
- **TikTok**: Algorithm-driven feed (not chronological)

