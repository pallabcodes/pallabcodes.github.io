+++
title = "Understanding the CAP Theorem in Distributed Systems"
date = 2024-12-17
description = "A deep dive into the CAP theorem and its practical implications for system design."
[taxonomies]
tags = ["distributed-systems", "system-design", "databases"]
+++

The CAP theorem is one of the most fundamental concepts in distributed systems, yet it's often misunderstood. Let's break it down.

<!-- more -->

## What is CAP?

The CAP theorem states that a distributed system can only provide two of these three guarantees simultaneously:

- **Consistency**: Every read receives the most recent write
- **Availability**: Every request receives a response
- **Partition Tolerance**: The system continues to operate despite network partitions

## The Real Trade-off

Here's the thing many engineers miss: **partition tolerance is not optional**. Networks fail. The real choice is between consistency and availability during a partition.

```
┌─────────────────────────────────────────┐
│           Network Partition             │
│                                         │
│   Node A ─────✕───── Node B             │
│                                         │
│   Choice:                               │
│   - Reject writes (Consistency)         │
│   - Accept writes (Availability)        │
└─────────────────────────────────────────┘
```

## Practical Examples

### CP Systems (Consistency + Partition Tolerance)

- **ZooKeeper**: Coordination service, rejects operations during partition
- **etcd**: Kubernetes' brain, consistency is critical
- **Traditional RDBMS with sync replication**

### AP Systems (Availability + Partition Tolerance)

- **Cassandra**: Always writable, eventual consistency
- **DynamoDB**: Highly available, tunable consistency
- **DNS**: Always responds, may be stale

## Code Example: Handling Eventual Consistency

```python
class EventuallyConsistentCache:
    def __init__(self, ttl_seconds: int = 60):
        self.cache = {}
        self.ttl = ttl_seconds
    
    def get(self, key: str) -> tuple[Any, bool]:
        """Returns (value, is_stale)"""
        entry = self.cache.get(key)
        if not entry:
            return None, False
        
        age = time.time() - entry['timestamp']
        is_stale = age > self.ttl
        return entry['value'], is_stale
    
    def set(self, key: str, value: Any):
        self.cache[key] = {
            'value': value,
            'timestamp': time.time()
        }
```

## Key Takeaways

1. **Pick your battles**: Understand which operations need strong consistency
2. **Design for failure**: Assume partitions will happen
3. **Use the right tool**: Don't use a CP database when you need AP semantics

The CAP theorem isn't about choosing two letters—it's about understanding trade-offs.
