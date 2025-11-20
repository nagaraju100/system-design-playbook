# System Design Playbook

Learn System Design concepts with practical examples and real-world use cases.

## Contents

### Core Concepts

#### [Consistency Models](consistency-models.md)

Learn about different consistency models used in distributed systems:

- **Eventual Consistency** - Understanding how systems like Amazon S3 and DNS handle temporary inconsistencies
- **Linearizable Consistency** - Strong consistency guarantees in bank transfers and ticket booking systems
- **Causal Consistency** - Preserving cause-and-effect relationships in social media and email threads
- **Quorum-based Consistency** - Majority agreement in collaborative editing and blockchain systems

Each model includes:
- Clear definitions
- Real-world examples
- Trade-offs and use cases
- Guidance on when to use each model

### Infrastructure & Components

#### [Load Balancer](load-balancer.md)
Design a load balancer that distributes incoming network traffic across multiple servers. Learn about different load balancing algorithms, health checking, session persistence, and high availability configurations.

#### [Distributed Cache](distributed-cache.md)
Design a distributed caching system like Redis or Memcached. Understand caching strategies, eviction policies, replication, and performance optimizations for high-throughput systems.

#### [Message Queue](message-queue.md)
Design a distributed message queue system for asynchronous communication. Learn about different queue models, delivery guarantees, partitioning, and popular solutions like Kafka and RabbitMQ.

#### [Rate Limiting](rate-limiting.md)
Design a rate limiting system to control request rates and prevent abuse. Explore different algorithms (token bucket, sliding window), distributed rate limiting, and implementation strategies.

#### [Distributed Lock](distributed-lock.md)
Design a distributed locking mechanism for coordinating access to shared resources. Learn about Redis-based locks, ZooKeeper implementations, and handling failures in distributed systems.

### System Designs

#### [URL Shortener](url-shortener.md)
Design a URL shortener service like bit.ly or TinyURL. Learn about URL encoding, key generation, caching strategies, and scaling to handle billions of URLs.

#### [Chat System](chat-system.md)
Design a real-time chat system like WhatsApp or Slack. Understand WebSocket connections, message delivery, presence services, and scaling for millions of concurrent users.

#### [Search Engine](search-engine.md)
Design a search engine that can index billions of web pages. Learn about web crawling, inverted indexes, ranking algorithms, and distributed search architectures.

#### [Social Media Feed](social-media-feed.md)
Design a social media feed system like Twitter or Facebook. Explore feed generation strategies (push vs pull), ranking algorithms, and handling millions of posts per day.

#### [File Storage System](file-storage.md)
Design a distributed file storage system like Google Drive or AWS S3. Learn about metadata separation, replication, deduplication, and scaling to exabytes of storage.

## Overview

This playbook covers fundamental distributed systems concepts with a focus on practical understanding. Each topic includes real-world examples from popular services like Amazon S3, Google Drive, and blockchain networks to help illustrate how these concepts are applied in production systems.

## CAP Theorem

Understanding the trade-offs between:
- **Consistency (C)**: Data synchronization across nodes
- **Availability (A)**: System responsiveness to requests
- **Partition Tolerance (P)**: Handling network failures

Choosing the right consistency model depends on your application's specific requirements for data accuracy, system responsiveness, and fault tolerance.
