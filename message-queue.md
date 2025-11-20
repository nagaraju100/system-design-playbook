# Message Queue System Design

Design a distributed message queue system like RabbitMQ, Kafka, or AWS SQS that enables asynchronous communication between services.

## Requirements

### Functional Requirements
- Producers can send messages to queues
- Consumers can receive messages from queues
- Support multiple queues and topics
- Message ordering guarantees
- At-least-once or exactly-once delivery
- Dead letter queue for failed messages

### Non-Functional Requirements
- High throughput (millions of messages per second)
- Low latency (< 10ms for message delivery)
- High availability (99.99% uptime)
- Durability (messages persisted to disk)
- Scalability (handle growing message volume)

## Capacity Estimation

### Traffic Estimates
- 1 billion messages per day
- Average message size: 1KB
- Peak message rate: 100,000 messages/second
- Message retention: 7 days
- Total storage: 1B × 1KB × 7 = 7TB

## System APIs

```
# Producer API
POST /queues/{queue_name}/messages
Body: {
  "message": "...",
  "priority": 5,
  "delay": 3600
}

# Consumer API
GET /queues/{queue_name}/messages?batch_size=10
Response: {
  "messages": [
    {"id": "msg1", "body": "...", "receipt_handle": "..."},
    ...
  ]
}

# Acknowledge message
DELETE /queues/{queue_name}/messages/{receipt_handle}
```

## High-Level Design

```
┌──────────┐         ┌──────────────┐         ┌──────────┐
│Producer 1│────────▶│              │◀────────│Consumer 1│
└──────────┘         │              │         └──────────┘
                     │ Message Queue│
┌──────────┐         │   Cluster    │         ┌──────────┐
│Producer 2│────────▶│              │◀────────│Consumer 2│
└──────────┘         │              │         └──────────┘
                     └──────┬───────┘
                            │
                            ▼
                    ┌──────────────┐
                    │   Storage    │
                    │   (Disk)     │
                    └──────────────┘
```

## Message Queue Models

### Point-to-Point (Queue)
- One producer, one consumer per message
- Message consumed by single consumer
- Use case: Task distribution, job processing

### Publish-Subscribe (Topic)
- One producer, multiple consumers
- Message broadcast to all subscribers
- Use case: Event notifications, logging

## Detailed Design

### Message Storage

**Option 1: Append-Only Log**
- Messages appended sequentially to disk
- Fast writes (O(1) append)
- Efficient for high throughput
- Used by Kafka

**Option 2: Database**
- Store messages in database tables
- More flexible querying
- Better for complex routing
- Used by RabbitMQ

**Option 3: Hybrid**
- Hot messages in memory
- Cold messages on disk
- Balance between speed and durability

### Partitioning and Sharding

- Partition queue by key or round-robin
- Each partition handled by different broker
- Enables parallel processing
- Maintains ordering within partition

### Message Ordering

**FIFO (First In First Out)**
- Strict ordering guarantee
- Single consumer per partition
- Lower throughput

**Best Effort Ordering**
- Ordering within partition
- Multiple consumers can process in parallel
- Higher throughput

### Delivery Guarantees

**At-Least-Once Delivery**
- Message delivered at least once
- May have duplicates
- Consumer must be idempotent
- Simpler implementation

**Exactly-Once Delivery**
- Message delivered exactly once
- No duplicates
- More complex (requires transactions)
- Lower performance

**At-Most-Once Delivery**
- Message delivered at most once
- May lose messages
- Fastest but least reliable

### Message Acknowledgment

**Automatic Acknowledgment**
- Message removed after sending
- Fast but may lose messages on consumer failure

**Manual Acknowledgment**
- Consumer explicitly acknowledges
- Message removed only after ack
- More reliable but requires ack management

**Acknowledgment Timeout**
- Auto-ack if consumer doesn't respond
- Message becomes available again
- Prevents message loss from slow consumers

### Dead Letter Queue (DLQ)

- Queue for messages that failed processing
- After N retry attempts, move to DLQ
- Allows manual inspection and reprocessing
- Prevents poison messages from blocking queue

## Replication and Durability

### Replication Strategy
- Replicate messages to multiple brokers
- Leader-follower replication
- Write to N replicas before acknowledging
- Read from any replica for availability

### Durability
- Write messages to disk before acknowledging
- Use write-ahead log (WAL)
- Periodic snapshots for faster recovery
- Trade-off: Durability vs. Performance

## Consumer Groups

- Multiple consumers in a group share work
- Each message consumed by one consumer in group
- Enables horizontal scaling
- Load balanced across consumers

## Message Retention

### Time-Based Retention
- Keep messages for N days
- Automatic deletion after retention period
- Configurable per queue

### Size-Based Retention
- Keep N messages or N GB
- Evict oldest messages when limit reached
- Prevents disk space issues

## Performance Optimizations

### Batching
- Batch multiple messages in single request
- Reduces network overhead
- Improves throughput

### Compression
- Compress messages before storage
- Reduces storage and network usage
- Trade-off: CPU usage

### Caching
- Cache frequently accessed messages
- Keep hot partitions in memory
- Faster consumer reads

## Monitoring and Metrics

### Key Metrics
- Message production rate
- Message consumption rate
- Queue depth (pending messages)
- Message latency (time in queue)
- Error rates
- Consumer lag

### Alerts
- Queue depth exceeds threshold
- High message latency
- Consumer lag increasing
- Broker failures

## Use Cases

1. **Asynchronous Processing**: Decouple producers and consumers
2. **Event-Driven Architecture**: Event notifications between services
3. **Log Aggregation**: Collect logs from multiple sources
4. **Task Queues**: Distribute work across workers
5. **Stream Processing**: Real-time data processing pipelines

## Popular Message Queue Systems

- **Apache Kafka**: High-throughput, distributed streaming platform
- **RabbitMQ**: Feature-rich message broker
- **AWS SQS**: Managed message queue service
- **Redis Pub/Sub**: Simple pub/sub messaging
- **Apache Pulsar**: Cloud-native messaging and streaming

