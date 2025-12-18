+++
title = "Designing for Failure: Building Resilient Systems"
date = 2024-12-14
description = "Strategies for building systems that gracefully handle failures."
[taxonomies]
tags = ["reliability", "system-design", "distributed-systems"]
+++

In distributed systems, failure isn't a bug—it's a feature. The question isn't *if* things will fail, but *when* and *how* you'll handle it.

<!-- more -->

## The Reality of Distributed Systems

Everything fails. Networks partition. Disks die. Services crash. The only question is: does your system handle it gracefully?

```
Murphy's Law for Distributed Systems:
┌────────────────────────────────────────┐
│ If it can fail, it will.               │
│ If it can't fail, it still will.       │
│ If it seems to be working, you're not  │
│ looking hard enough.                   │
└────────────────────────────────────────┘
```

## Circuit Breaker Pattern

Stop cascading failures by failing fast:

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, reset_timeout=60):
        self.failures = 0
        self.threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.state = "CLOSED"
        self.last_failure_time = None
    
    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.reset_timeout:
                self.state = "HALF-OPEN"
            else:
                raise CircuitOpenError("Circuit breaker is open")
        
        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise
    
    def on_success(self):
        self.failures = 0
        self.state = "CLOSED"
    
    def on_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()
        if self.failures >= self.threshold:
            self.state = "OPEN"
```

## Retry with Exponential Backoff

Don't hammer a failing service:

```go
func retryWithBackoff(operation func() error, maxRetries int) error {
    for attempt := 0; attempt < maxRetries; attempt++ {
        err := operation()
        if err == nil {
            return nil
        }
        
        // Exponential backoff with jitter
        backoff := time.Duration(math.Pow(2, float64(attempt))) * time.Second
        jitter := time.Duration(rand.Intn(1000)) * time.Millisecond
        time.Sleep(backoff + jitter)
    }
    return fmt.Errorf("max retries exceeded")
}
```

## Bulkhead Pattern

Isolate failures to prevent total system collapse:

```
┌─────────────────────────────────────────────────────┐
│                   Your Service                       │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │
│  │ Payment API │ │ Inventory   │ │ Shipping    │    │
│  │ Thread Pool │ │ Thread Pool │ │ Thread Pool │    │
│  │   (10)      │ │   (10)      │ │   (10)      │    │
│  └─────────────┘ └─────────────┘ └─────────────┘    │
│        ↓              ↓               ↓             │
│   If Payment      Other pools    Other pools        │
│   hangs, only     unaffected     unaffected         │
│   that pool                                         │
│   is blocked                                        │
└─────────────────────────────────────────────────────┘
```

## Timeouts: Always. Everywhere.

Never make a network call without a timeout:

```python
# Bad - can hang forever
response = requests.get(url)

# Good - fails predictably
response = requests.get(url, timeout=(3.0, 10.0))
#                            connect, read timeout
```

## Graceful Degradation

When dependencies fail, provide reduced functionality instead of complete failure:

```python
def get_recommendations(user_id: str) -> list[Product]:
    try:
        # Try personalized recommendations
        return recommendation_service.get_personalized(user_id)
    except ServiceUnavailable:
        try:
            # Fall back to cached recommendations
            return cache.get(f"recs:{user_id}")
        except CacheMiss:
            # Ultimate fallback: popular items
            return get_popular_products()
```

## Key Takeaways

1. **Expect failure** - Design for it from day one
2. **Fail fast** - Use circuit breakers
3. **Retry smart** - Exponential backoff with jitter
4. **Isolate failures** - Bulkheads prevent cascading failures
5. **Timeout everything** - Never wait forever
6. **Degrade gracefully** - Partial functionality beats complete failure

The best systems don't avoid failure—they embrace it.
