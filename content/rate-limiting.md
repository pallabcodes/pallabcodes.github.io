+++
title = "Rate Limiting: Algorithms and Trade-offs"
date = 2024-12-12
description = "Comparing token bucket, sliding window, and other rate limiting algorithms."
[taxonomies]
tags = ["system-design", "api-design", "performance"]
+++

Rate limiting protects your APIs from abuse and ensures fair resource allocation. But which algorithm should you use?

<!-- more -->

## Why Rate Limiting?

Without rate limiting:
- One client can consume all resources
- DDoS attacks overwhelm your service
- Runaway scripts cause outages
- Costs spiral out of control

## Token Bucket Algorithm

The most common algorithm. Tokens are added at a fixed rate; requests consume tokens.

```python
import time
from threading import Lock

class TokenBucket:
    def __init__(self, rate: float, capacity: int):
        self.rate = rate          # Tokens per second
        self.capacity = capacity  # Max tokens
        self.tokens = capacity
        self.last_update = time.time()
        self.lock = Lock()
    
    def allow_request(self) -> bool:
        with self.lock:
            now = time.time()
            elapsed = now - self.last_update
            
            # Add tokens based on elapsed time
            self.tokens = min(
                self.capacity,
                self.tokens + elapsed * self.rate
            )
            self.last_update = now
            
            if self.tokens >= 1:
                self.tokens -= 1
                return True
            return False
```

**Pros**: Allows bursts up to capacity, smooth rate limiting
**Cons**: Burst at window boundaries possible

## Sliding Window Log

Track exact timestamps of each request:

```python
class SlidingWindowLog:
    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window = window_seconds
        self.requests = []
    
    def allow_request(self) -> bool:
        now = time.time()
        cutoff = now - self.window
        
        # Remove old requests
        self.requests = [t for t in self.requests if t > cutoff]
        
        if len(self.requests) < self.limit:
            self.requests.append(now)
            return True
        return False
```

**Pros**: Precise, no boundary issues
**Cons**: Memory-intensive for high traffic

## Sliding Window Counter

Approximate sliding window using fixed windows:

```
Window 1 (75% weight)    Window 2 (25% weight)
     ┌────────────┐      ┌────────────┐
     │ 40 requests│      │ 20 requests│
     └────────────┘      └────────────┘
                    ↑
              Current time (25% into window 2)

Weighted count = 40 * 0.75 + 20 * 0.25 = 35
```

```python
class SlidingWindowCounter:
    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window = window_seconds
        self.prev_count = 0
        self.curr_count = 0
        self.window_start = int(time.time() / window_seconds)
    
    def allow_request(self) -> bool:
        now = time.time()
        current_window = int(now / self.window)
        
        if current_window > self.window_start:
            self.prev_count = self.curr_count
            self.curr_count = 0
            self.window_start = current_window
        
        # Calculate position in current window
        elapsed = now - (current_window * self.window)
        weight = elapsed / self.window
        
        # Weighted sum
        count = self.prev_count * (1 - weight) + self.curr_count
        
        if count < self.limit:
            self.curr_count += 1
            return True
        return False
```

**Pros**: Memory-efficient, reasonably precise
**Cons**: Approximate

## Algorithm Comparison

| Algorithm | Memory | Precision | Burst Handling |
|-----------|--------|-----------|----------------|
| Token Bucket | O(1) | Good | Allows bursts |
| Sliding Log | O(n) | Exact | No bursts |
| Sliding Counter | O(1) | ~99% | Smooth |

## Distributed Rate Limiting

For multiple servers, use Redis:

```python
import redis

def check_rate_limit(redis_client, user_id: str, limit: int, window: int) -> bool:
    key = f"rate_limit:{user_id}"
    current = redis_client.incr(key)
    
    if current == 1:
        redis_client.expire(key, window)
    
    return current <= limit
```

## Key Takeaways

1. **Token bucket** for most use cases—allows bursts, simple
2. **Sliding window counter** when precision matters
3. **Use Redis** for distributed rate limiting
4. **Include headers** - Tell clients their limits: `X-RateLimit-Remaining`
5. **Fail open carefully** - What if Redis is down?

Rate limiting is your API's immune system. Design it thoughtfully.
