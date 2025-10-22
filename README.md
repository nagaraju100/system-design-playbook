# System Design Playbook

Learn System Design concepts with practical examples and real-world use cases.

## Contents

### [Consistency Models](consistency-models.md)

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

## Overview

This playbook covers fundamental distributed systems concepts with a focus on practical understanding. Each topic includes real-world examples from popular services like Amazon S3, Google Drive, and blockchain networks to help illustrate how these concepts are applied in production systems.

## CAP Theorem

Understanding the trade-offs between:
- **Consistency (C)**: Data synchronization across nodes
- **Availability (A)**: System responsiveness to requests
- **Partition Tolerance (P)**: Handling network failures

Choosing the right consistency model depends on your application's specific requirements for data accuracy, system responsiveness, and fault tolerance.
