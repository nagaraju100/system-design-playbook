# Chat System Design

Design a real-time chat system like WhatsApp, Slack, or Discord that supports one-on-one and group messaging.

## Requirements

### Functional Requirements
- Send and receive messages in real-time
- Support one-on-one chats
- Support group chats (multiple participants)
- Message delivery status (sent, delivered, read)
- Online/offline status
- Message history
- File and media sharing
- Push notifications

### Non-Functional Requirements
- Low latency (< 100ms for message delivery)
- High availability (99.9% uptime)
- Scalability (millions of concurrent users)
- Message ordering guarantees
- At-least-once message delivery

## Capacity Estimation

### Traffic Estimates
- 500 million daily active users
- Average 50 messages per user per day
- Total: 25 billion messages per day
- Peak QPS: 1 million messages per second
- Average message size: 100 bytes
- Storage: 25B × 100 bytes × 30 days = 75TB per month

## System APIs

```
# Send message
POST /api/v1/messages
Body: {
  "chat_id": "chat123",
  "message": "Hello!",
  "type": "text",
  "attachments": []
}

# Get messages
GET /api/v1/chats/{chat_id}/messages?limit=50&before={message_id}

# Update delivery status
PUT /api/v1/messages/{message_id}/status
Body: {
  "status": "delivered" | "read"
}

# Get online status
GET /api/v1/users/{user_id}/status
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
            │  WebSocket   │        │   Message    │        │   Presence   │
            │   Servers    │        │   Queue      │        │   Service    │
            └──────────────┘        └──────────────┘        └──────────────┘
                    │                        │                        │
                    └────────────────────────┼────────────────────────┘
                                             ▼
                                    ┌─────────────────┐
                                    │   Database      │
                                    │   (Messages,    │
                                    │    Users, Chats)│
                                    └─────────────────┘
```

## Detailed Design

### Real-Time Communication

**WebSocket Connection**
- Persistent TCP connection
- Full-duplex communication
- Low latency
- Server can push messages to client

**Connection Management**
- Each user maintains WebSocket connection
- Connection pool per server
- Handle reconnection on disconnect
- Heartbeat to detect dead connections

### Message Flow

**Sending Message:**
1. Client sends message via WebSocket/HTTP
2. API server validates and stores message
3. Message added to message queue
4. Queue worker processes message
5. Message delivered to recipient's WebSocket server
6. Recipient receives message in real-time

**Offline User:**
1. Message stored in database
2. Push notification sent
3. Message delivered when user comes online

### Database Schema

**Users Table**
```
- user_id: BIGINT (primary key)
- username: VARCHAR(50)
- email: VARCHAR(255)
- created_at: TIMESTAMP
- last_seen: TIMESTAMP
```

**Chats Table**
```
- chat_id: BIGINT (primary key)
- type: ENUM('one_on_one', 'group')
- created_at: TIMESTAMP
- updated_at: TIMESTAMP
```

**Chat Participants Table**
```
- chat_id: BIGINT
- user_id: BIGINT
- joined_at: TIMESTAMP
- last_read_message_id: BIGINT
- PRIMARY KEY (chat_id, user_id)
```

**Messages Table**
```
- message_id: BIGINT (primary key)
- chat_id: BIGINT
- sender_id: BIGINT
- message: TEXT
- type: ENUM('text', 'image', 'file')
- created_at: TIMESTAMP
- INDEX (chat_id, created_at)
```

**Message Status Table**
```
- message_id: BIGINT
- user_id: BIGINT
- status: ENUM('sent', 'delivered', 'read')
- updated_at: TIMESTAMP
- PRIMARY KEY (message_id, user_id)
```

### Message Storage

**Option 1: Relational Database**
- Store all messages in MySQL/PostgreSQL
- Simple but may become bottleneck
- Use partitioning for large scale

**Option 2: NoSQL Database**
- Store messages in Cassandra/MongoDB
- Better for write-heavy workloads
- Partition by chat_id

**Option 3: Hybrid Approach**
- Recent messages in cache (Redis)
- Older messages in database
- Archive very old messages

### Message Ordering

**Within Chat Ordering**
- Use message_id (auto-increment) or timestamp
- Ensure messages ordered by creation time
- Handle clock skew in distributed systems

**Global Ordering**
- Use vector clocks or logical timestamps
- More complex but handles edge cases
- Required for strict ordering guarantees

### Presence Service

**Online/Offline Status**
- Track user's WebSocket connection
- Update status on connect/disconnect
- Store in Redis for fast access
- Broadcast status changes to contacts

**Last Seen**
- Update timestamp on user activity
- Show "last seen X minutes ago"
- Privacy controls (hide from certain users)

### Message Delivery Status

**Sent**: Message stored in database
**Delivered**: Message received by recipient's device
**Read**: Recipient opened and viewed message

**Implementation:**
- Track status per message per user
- Update status via WebSocket events
- Store in database for persistence

### Group Chat Features

**Participant Management**
- Add/remove participants
- Admin roles and permissions
- Broadcast participant changes

**Message Routing**
- Send message to all participants
- Filter based on user preferences
- Handle large groups efficiently

### File and Media Sharing

**Upload Flow:**
1. Client uploads file to object storage (S3)
2. Store file metadata in database
3. Send message with file URL
4. Recipient downloads from object storage

**Optimization:**
- Compress images before storage
- Generate thumbnails for images
- Use CDN for faster delivery

### Push Notifications

**When to Send:**
- User is offline
- User is online but not in chat
- Message in group chat

**Implementation:**
- Use FCM (Firebase Cloud Messaging) for Android
- Use APNs (Apple Push Notification) for iOS
- Use web push for browsers

### Scaling Strategies

**Horizontal Scaling**
- Multiple WebSocket servers
- Use consistent hashing for user routing
- Load balance WebSocket connections

**Database Sharding**
- Shard by user_id or chat_id
- Distribute load across shards
- Handle cross-shard queries

**Caching**
- Cache user presence in Redis
- Cache recent messages
- Cache chat metadata

## Performance Optimizations

### Message Batching
- Batch multiple messages in single request
- Reduce network overhead
- Improve throughput

### Lazy Loading
- Load messages on demand
- Paginate message history
- Load older messages when scrolling up

### Connection Pooling
- Reuse WebSocket connections
- Maintain connection pool per server
- Efficient resource utilization

## Monitoring and Metrics

### Key Metrics
- Message delivery latency
- WebSocket connection count
- Messages per second
- Online user count
- Message delivery success rate

### Alerts
- High message latency
- WebSocket connection failures
- Database connection issues
- High error rates

## Security Considerations

- End-to-end encryption for messages
- Authentication and authorization
- Rate limiting to prevent spam
- Input validation and sanitization
- Secure file uploads

## Use Cases

1. **One-on-One Chat**: Direct messaging between two users
2. **Group Chat**: Multiple participants in single chat
3. **Channel Chat**: Public channels with many members
4. **Broadcast Messages**: One-to-many messaging
5. **Customer Support**: Chat with support agents

