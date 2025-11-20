# Load Balancer System Design

Design a load balancer that distributes incoming network traffic across multiple servers to ensure high availability and optimal resource utilization.

## Requirements

### Functional Requirements
- Distribute incoming requests across multiple backend servers
- Support multiple load balancing algorithms
- Health checking for backend servers
- Session persistence (sticky sessions)
- SSL/TLS termination

### Non-Functional Requirements
- High availability (99.99% uptime)
- Low latency overhead (< 1ms)
- Handle millions of requests per second
- Automatic failover
- Support for dynamic server addition/removal

## Load Balancing Algorithms

### Round Robin
- Distribute requests sequentially across servers
- Simple and fair distribution
- Doesn't consider server load or capacity

### Weighted Round Robin
- Assign weights to servers based on capacity
- Higher capacity servers receive more requests
- Good for heterogeneous server clusters

### Least Connections
- Route requests to server with fewest active connections
- Good for long-lived connections
- Requires tracking connection counts

### Least Response Time
- Route to server with lowest response time
- Considers both server load and network latency
- More complex but better performance

### IP Hash
- Hash client IP to determine target server
- Ensures same client always hits same server
- Good for session persistence

### Geographic Routing
- Route based on client geographic location
- Route to nearest data center
- Reduces latency for global applications

## High-Level Architecture

```
                    ┌─────────────────┐
                    │   DNS Server    │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  Load Balancer  │
                    │   (Layer 4/7)   │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Server 1   │    │   Server 2   │    │   Server 3   │
└──────────────┘    └──────────────┘    └──────────────┘
```

## Layer 4 vs Layer 7 Load Balancing

### Layer 4 (Transport Layer)
- Works at TCP/UDP level
- Routes based on IP and port
- Faster (less processing)
- No application awareness
- Use cases: Database load balancing, simple HTTP routing

### Layer 7 (Application Layer)
- Works at HTTP/HTTPS level
- Routes based on URL, headers, cookies
- More intelligent routing decisions
- SSL termination capability
- Use cases: Web applications, API gateways

## Health Checking

### Active Health Checks
- Load balancer periodically sends requests to servers
- Check endpoints: `/health`, `/ping`
- Mark server as unhealthy if checks fail
- Configurable intervals and thresholds

### Passive Health Checks
- Monitor actual request responses
- Track error rates and response times
- Mark unhealthy based on failure patterns
- Less overhead than active checks

### Health Check Parameters
- Interval: How often to check (e.g., 10 seconds)
- Timeout: Max wait time for response (e.g., 5 seconds)
- Healthy threshold: Consecutive successes to mark healthy
- Unhealthy threshold: Consecutive failures to mark unhealthy

## Session Persistence

### Cookie-Based Sticky Sessions
- Load balancer sets cookie with server identifier
- Client sends cookie in subsequent requests
- Route to same server based on cookie
- Works across multiple load balancers

### IP-Based Sticky Sessions
- Hash client IP to determine server
- Same IP always routes to same server
- Simple but breaks with NAT/proxies

### Application-Level Sessions
- Store sessions in shared cache (Redis)
- Any server can handle any request
- Better scalability and fault tolerance

## High Availability

### Active-Passive Configuration
- Primary load balancer handles traffic
- Secondary load balancer in standby
- Automatic failover on primary failure
- Uses VRRP/HSRP protocols

### Active-Active Configuration
- Multiple load balancers share traffic
- Better resource utilization
- More complex configuration
- Requires session synchronization

## SSL/TLS Termination

### Benefits
- Offload SSL processing from backend servers
- Centralized certificate management
- Better performance (dedicated SSL hardware)
- Easier certificate rotation

### SSL Passthrough
- Forward encrypted traffic to backend
- Backend servers handle SSL
- More secure (end-to-end encryption)
- Higher backend server load

## Rate Limiting

### Per-IP Rate Limiting
- Limit requests per client IP
- Prevent DDoS attacks
- Fair resource distribution

### Per-User Rate Limiting
- Limit based on authenticated user
- More accurate than IP-based
- Requires user identification

### Global Rate Limiting
- Limit total requests across all clients
- Protect backend from overload
- Circuit breaker pattern

## Monitoring and Metrics

### Key Metrics
- Request rate (requests per second)
- Response times (p50, p95, p99)
- Error rates (4xx, 5xx)
- Active connections
- Backend server health status

### Logging
- Access logs (client IP, request path, response code)
- Error logs (failed health checks, connection errors)
- Performance logs (latency, throughput)

## Scaling Strategies

### Horizontal Scaling
- Add more load balancer instances
- Use DNS round-robin or anycast
- Distribute load across multiple load balancers

### Vertical Scaling
- Increase CPU and memory
- Use dedicated SSL hardware
- Limited by single machine capacity

## Use Cases

1. **Web Application Load Balancing**: Distribute HTTP/HTTPS traffic
2. **Database Load Balancing**: Distribute database connections
3. **API Gateway**: Route API requests to microservices
4. **CDN Edge Servers**: Route to nearest edge server
5. **Multi-Region Routing**: Route to closest data center

## Popular Load Balancer Solutions

- **AWS ELB/ALB**: Managed load balancing service
- **NGINX**: Open-source web server and load balancer
- **HAProxy**: High-performance load balancer
- **F5 BIG-IP**: Enterprise load balancer
- **Cloudflare**: Global load balancing and DDoS protection

