+++
title = "Share Memory by Communicating Part 4: Scaling & Production Patterns"
date = 2025-12-19
description = "Buffered channels, Kubernetes shutdown, errgroup, batching, and when this pattern breaks"
[taxonomies]
tags = ["go", "concurrency", "production", "kubernetes"]
+++

Parts 1-3 built the foundation. This final part covers production scaling: buffered channels, Kubernetes integration, errgroup supervision, performance tuning, and comparisons with Erlang/Akka.

<!-- more -->

## Buffered Channels: When They Help vs Hide Bugs

A buffered channel:

```go
ch := make(chan T, N)
```

Means: **send doesn't require a simultaneous receive—up to N items.**

### When Buffered Channels Help

**1. Decoupling producer/consumer latency**

```go
jobs := make(chan Job, 100)
```

Use when producers are bursty and consumers are steady. This is **load smoothing**, not concurrency control.

**2. Preventing response-path deadlocks**

```go
resp := make(chan int, 1)  // Buffered
```

Owner can send even if caller exits. Reduces blast radius.

**3. Semaphores**

```go
sem := make(chan struct{}, 10)
```

Classic concurrency limiting. Bounded by design.

### When Buffered Channels Hide Bugs

**1. Masking missing consumers**

```go
ch := make(chan Event, 1000)
ch <- e  // Succeeds!
// But no consumer exists
// Buffer fills slowly
// System degrades later
```

This converts a **correctness bug** into a **latency/memory bug**.

**2. "Fixing" deadlocks accidentally**

Adding a buffer makes the symptom disappear, but the caller still leaked.

**3. Using buffers as flow control**

```go
make(chan Job, 1_000_000)
```

This is unbounded memory in disguise.

### The Rule

> **Buffers absorb timing differences, not logic errors.**
> **If correctness depends on buffering, the design is wrong.**

## Kubernetes SIGTERM → Context

This is mechanical, not conceptual.

### What Kubernetes Does

1. Pod receives `SIGTERM`
2. Kubernetes waits `terminationGracePeriodSeconds`
3. If still running → `SIGKILL`

Go does **nothing automatically**.

### Canonical Production Wiring

```go
ctx, stop := signal.NotifyContext(
    context.Background(),
    os.Interrupt,
    syscall.SIGTERM,
)
defer stop()

go stateOwner(ctx, reads, writes)

<-ctx.Done()  // Wait for SIGTERM
wg.Wait()     // Wait for cleanup
```

Every goroutine must obey:

```go
select {
case <-ctx.Done():
    return
}
```

### The #1 Go-in-Kubernetes Bug

Not doing this:
- Pod keeps accepting requests
- Kubernetes sends SIGKILL
- In-flight work lost
- Partial writes
- Corrupt state

## errgroup: Fail-Fast Supervision

```go
g, ctx := errgroup.WithContext(parent)

g.Go(func() error {
    return worker1(ctx)
})

g.Go(func() error {
    return worker2(ctx)
})

if err := g.Wait(); err != nil {
    log.Fatal(err)
}
```

What errgroup provides:
1. **Shared cancellation** — one fails, all cancel
2. **Error propagation** — first error surfaces
3. **Join** — `Wait()` blocks until all done

What errgroup does NOT do:
- Restart
- Backoff
- Isolation

Use for: request fan-out, parallel RPCs, bounded lifetimes.

## Batching for Performance

The channel-per-request model can bottleneck at scale.

### Batch Reads

```go
type readBatch struct {
    keys []int
    resp chan []int
}

case batch := <-readBatches:
    result := make([]int, len(batch.keys))
    for i, k := range batch.keys {
        result[i] = state[k]
    }
    batch.resp <- result
```

Fewer wakeups, better cache locality, higher throughput.

### Time-Based Batching

```go
ticker := time.NewTicker(100 * time.Microsecond)
var pending []writeOp

for {
    select {
    case w := <-writes:
        pending = append(pending, w)
    case <-ticker.C:
        for _, w := range pending {
            state[w.key] = w.val
            w.resp <- true
        }
        pending = nil
    }
}
```

This is how high-throughput systems actually work.

## When This Pattern Breaks

### Where It Shines
- Strong invariants
- Complex state transitions
- Low-moderate throughput
- Correctness over speed

### Where It Fails
- High write throughput
- Hot keys
- CPU-bound logic in owner

### The Bottleneck

```
All ops → ONE goroutine → ONE P → ONE M
```

Caps throughput regardless of CPUs.

### Scaling: Sharded Owners

```go
const shards = 32
owners := make([]chan writeOp, shards)

func shardFor(key int) int {
    return key % shards
}
```

Each shard owns its own map and goroutine. Scales linearly.

## Comparison: Go vs Erlang vs Akka

| Aspect | Go | Erlang | Akka |
|--------|-----|--------|------|
| Actors | Manual | Built-in | Library |
| Mailbox | Channel | Built-in | Framework |
| Supervision | Manual | First-class | Framework |
| Restart | Your code | Automatic | Framework |
| Fault tolerance | Weak | Exceptional | Good |
| Performance | Excellent | Lower | Lower |

### The Philosophical Difference

**Go**: "Here are tools. Build what you need."

**Erlang**: "Here is a system. Obey its rules."

Go dominates in: infrastructure, databases, networking, storage.

Erlang shines in: telecom, fault-tolerant messaging, long-lived distributed systems.

## Proving Liveness

A goroutine is **live** if there exists a future execution path where it can make progress or exit.

For every blocking point, answer:

> **What event guarantees this unblocks?**

If the answer is "someone probably sends" or "shouldn't happen"—**not live**.

With context:

```go
select {
case <-resp:
case <-ctx.Done():
}
```

Two unblock paths. Both guaranteed eventually. **Live.**

## Final Synthesis

```
Buffered channels → timing
Context → lifetime
errgroup → failure propagation
Supervision → system stability
Liveness → correctness proof
```

The master rule:

> **Concurrency bugs are not about threads or channels—they are about ownership and lifetimes under failure.**

## Series Summary

- **Part 1**: The pattern—channel-owned state
- **Part 2**: Why it breaks—lifecycle and leaks
- **Part 3**: How to fix it—context and supervision
- **Part 4**: How to scale it—batching, sharding, Kubernetes

This is the complete model for production Go concurrency using the "share memory by communicating" philosophy.
