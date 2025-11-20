# Distributed Lock System Design

Design a distributed locking mechanism that allows multiple processes or servers to coordinate access to shared resources in a distributed system.

## Requirements

### Functional Requirements
- Acquire lock with optional timeout
- Release lock explicitly
- Automatic lock expiration (deadlock prevention)
- Support for lock renewal/extension
- Read-write locks (shared/exclusive)

### Non-Functional Requirements
- High availability (survive node failures)
- Low latency (< 10ms for lock acquisition)
- Fairness (prevent starvation)
- Safety (only one process holds lock)
- Liveness (locks eventually released)

## Use Cases

1. **Leader Election**: Ensure only one leader in cluster
2. **Resource Access Control**: Prevent concurrent modifications
3. **Distributed Transactions**: Coordinate across services
4. **Scheduled Tasks**: Prevent duplicate job execution
5. **Cache Invalidation**: Coordinate cache updates

## High-Level Design

```
┌──────────┐         ┌──────────────┐         ┌──────────┐
│Process 1 │────────▶│              │◀────────│Process 2 │
└──────────┘         │  Distributed │         └──────────┘
                     │  Lock Service│
┌──────────┐         │              │         ┌──────────┐
│Process 3 │────────▶│              │◀────────│Process 4 │
└──────────┘         └──────┬───────┘         └──────────┘
                            │
                            ▼
                    ┌──────────────┐
                    │  Coordination │
                    │  Store (Redis,│
                    │  Zookeeper)   │
                    └──────────────┘
```

## Implementation Approaches

### 1. Redis-based Distributed Lock

**Simple Implementation (SET with NX)**
```python
def acquire_lock(lock_key, lock_value, ttl):
    result = redis.set(lock_key, lock_value, nx=True, ex=ttl)
    return result == "OK"

def release_lock(lock_key, lock_value):
    # Lua script for atomic check-and-delete
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    return redis.eval(script, 1, lock_key, lock_value)
```

**Redlock Algorithm (Multi-Node)**
- Acquire lock on majority of Redis nodes
- Prevents single point of failure
- More reliable but complex

### 2. ZooKeeper-based Distributed Lock

**Ephemeral Sequential Nodes**
- Create ephemeral sequential node
- Check if you have lowest sequence number
- Watch previous node for deletion
- Automatic cleanup on disconnection

**Implementation Steps:**
1. Create ephemeral sequential node: `/locks/resource-0000000001`
2. Get all children and check if you have lowest number
3. If yes, acquire lock
4. If no, watch previous node
5. When previous node deleted, retry

### 3. Database-based Distributed Lock

**Optimistic Locking**
- Use version numbers or timestamps
- Check version before update
- Retry on conflict

**Pessimistic Locking**
- Use SELECT FOR UPDATE
- Database holds lock until transaction commits
- Simple but can cause deadlocks

## Lock Properties

### Safety (Mutual Exclusion)
- Only one process can hold lock at a time
- Critical for correctness
- Must be guaranteed even with failures

### Liveness (Deadlock Freedom)
- Locks eventually released
- No process waits forever
- Achieved through TTL/timeout

### Fairness
- First-come-first-served ordering
- Prevent starvation
- Requires queue or priority mechanism

## Lock Expiration and Renewal

### Automatic Expiration
- Set TTL when acquiring lock
- Lock automatically released after TTL
- Prevents deadlocks from crashed processes

### Lock Renewal
- Extend lock before expiration
- Use background thread to renew
- Must check if still holding lock

```python
def renew_lock(lock_key, lock_value, ttl):
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("expire", KEYS[1], ARGV[2])
    else
        return 0
    end
    """
    return redis.eval(script, 1, lock_key, lock_value, ttl)
```

## Read-Write Locks

### Shared (Read) Lock
- Multiple readers can hold lock simultaneously
- Writers must wait for all readers
- Use case: Read-heavy workloads

### Exclusive (Write) Lock
- Only one writer can hold lock
- Blocks all readers and writers
- Use case: Write operations

### Implementation
- Track number of readers
- Track writer status
- Use separate locks or counters

## Handling Failures

### Process Crash
- Lock automatically expires (TTL)
- Other processes can acquire after expiration
- No manual cleanup needed

### Network Partition
- Lock holder may be partitioned away
- Lock expires after TTL
- New process can acquire (may cause issues)
- Consider fencing tokens for safety

### Clock Skew
- TTL based on server time
- Clock skew can cause early expiration
- Use logical clocks or NTP synchronization

## Fencing Tokens

- Incrementing token with each lock acquisition
- Include token in all operations
- Reject operations with stale tokens
- Prevents split-brain scenarios

**Example:**
```
Process A acquires lock, gets token 1
Network partition occurs
Lock expires, Process B acquires lock, gets token 2
Process A still thinks it holds lock
Process A's operations rejected (token 1 < token 2)
```

## Performance Considerations

### Lock Granularity
- Fine-grained locks: Better concurrency, more overhead
- Coarse-grained locks: Less overhead, lower concurrency
- Balance based on access patterns

### Lock Contention
- High contention reduces performance
- Consider lock-free data structures
- Use read-write locks for read-heavy workloads

### Caching
- Cache lock status locally
- Reduce coordination store calls
- Trade-off: Slightly less accurate

## Monitoring and Metrics

### Key Metrics
- Lock acquisition time
- Lock contention rate
- Lock hold duration
- Failed lock acquisitions
- Deadlock detection

### Alerts
- High lock contention
- Long lock hold times
- Frequent lock timeouts
- Coordination store failures

## Best Practices

1. **Always set TTL**: Prevent deadlocks
2. **Use unique lock values**: Prevent accidental release
3. **Implement retry logic**: Handle transient failures
4. **Monitor lock metrics**: Identify bottlenecks
5. **Use appropriate TTL**: Balance safety and performance
6. **Consider lock-free alternatives**: When possible

## Comparison of Approaches

| Approach | Pros | Cons | Use Case |
|----------|------|------|----------|
| Redis | Fast, simple | Single point of failure | High-performance apps |
| Redlock | Fault-tolerant | Complex, slower | Critical systems |
| ZooKeeper | Strong guarantees | Slower, complex | Coordination-heavy |
| Database | Simple, familiar | Slower, scalability | Low-contention scenarios |

## Popular Distributed Lock Solutions

- **Redis**: SET NX with expiration
- **ZooKeeper**: Ephemeral nodes
- **etcd**: Distributed key-value store with TTL
- **Consul**: Service discovery with sessions
- **Hazelcast**: In-memory data grid with locks

