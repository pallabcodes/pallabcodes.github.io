+++
title = "The Go Scheduler Part 2: Leaks, Supervision & Backpressure"
date = 2025-12-19
description = "Real goroutine leak postmortems, supervision trees, backpressure design, and Go vs Rust async comparison"
[taxonomies]
tags = ["go", "concurrency", "production", "scheduler"]
+++

Part 1 explained the G-P-M model. This part covers what happens when things go wrong: real leak postmortems, proper supervision, backpressure design, and how Go compares to Rust's async model.

<!-- more -->

## A Real Goroutine Leak Postmortem

This is a common incident pattern. I've seen variations of this at multiple companies.

### The Symptoms

- p99 latency slowly increases
- Memory usage climbs
- No crashes
- CPU looks normal
- Restarts "fix" it temporarily

### The Code That Shipped

```go
func handler(w http.ResponseWriter, r *http.Request) {
    resp := make(chan Result, 1)

    go func() {
        result := callBackend()  // slow, sometimes hangs
        resp <- result
    }()

    select {
    case res := <-resp:
        writeResponse(w, res)
    case <-time.After(100 * time.Millisecond):
        http.Error(w, "timeout", 504)
        return
    }
}
```

### Why It Looked Correct

- Timeout exists ✅
- Buffered channel avoids blocking ✅
- No mutexes ✅
- No obvious deadlock ✅

This passed code review. This shipped to production.

### What Actually Happened

1. Client disconnects early
2. `handler` returns
3. `resp` channel is **never read again**
4. Backend goroutine eventually finishes
5. Backend goroutine executes `resp <- result`
6. **Blocks forever**

Wait—buffer size is 1, shouldn't it succeed?

Sometimes. But if:
- Backend returns twice (retry logic)
- Value already sent
- Reconnection logic fires

The second send blocks forever.

At 5k RPS, thousands of goroutines leak per minute. Memory climbs. Latency degrades. No crash, no alert.

### Why the Runtime Didn't Help

The Go runtime only panics when **all** goroutines are blocked. One blocked goroutine among thousands? Runtime doesn't care.

### The Fix

```go
ctx, cancel := context.WithTimeout(r.Context(), 100*time.Millisecond)
defer cancel()

select {
case res := <-resp:
    writeResponse(w, res)
case <-ctx.Done():
    return
}
```

And inside the backend goroutine:

```go
select {
case resp <- result:
case <-ctx.Done():
    return
}
```

**One missing `select` caused a production incident.**

### The Lesson

> **Buffered channels hide leaks. Context exposes them.**

## Supervision Trees in Go

Go doesn't have Erlang-style supervisors. You must build them.

### What Supervision Means in Go

A supervision tree is a hierarchy where parents:
- Start children
- Cancel children
- Wait for children
- Optionally restart children

### Production Structure

```
main
 └── root supervisor
      ├── http server
      │    └── request handlers (errgroup)
      ├── background worker supervisor
      │    ├── worker A (restartable)
      │    └── worker B (restartable)
      └── state owner
```

### Root Supervisor Pattern

```go
func main() {
    ctx, cancel := signal.NotifyContext(
        context.Background(),
        syscall.SIGTERM,
        os.Interrupt,
    )
    defer cancel()

    var wg sync.WaitGroup

    // Start function for supervised goroutines
    start := func(fn func(ctx context.Context)) {
        wg.Add(1)
        go func() {
            defer wg.Done()
            fn(ctx)
        }()
    }

    // Start your services
    start(runHTTPServer)
    start(runWorkerPool)
    start(runStateOwner)

    // Wait for shutdown signal
    <-ctx.Done()

    // Wait for all children to exit
    wg.Wait()
}
```

### Why This Matters

- No orphan goroutines
- Deterministic shutdown
- No leaks survive SIGTERM
- Kubernetes-friendly

## Backpressure: The Missing Design Element

Every production system must answer:

> **What happens if input arrives faster than output?**

### Without Backpressure

```go
for job := range jobs {
    go process(job)  // Infinite concurrency
}
```

This spawns unbounded goroutines. Memory explodes under load.

### With Explicit Backpressure

```go
sem := make(chan struct{}, 32)

for job := range jobs {
    sem <- struct{}{}  // Blocks when 32 active
    go func(job Job) {
        defer func() { <-sem }()
        process(job)
    }(job)
}
```

Now:
- Max 32 concurrent workers
- Senders block when full
- Pressure propagates upstream
- System stabilizes

### Channel-Owned State with Backpressure

```go
reads := make(chan readOp, 100)  // Bounded
```

When the buffer fills:
- Senders block
- Pressure propagates
- System finds equilibrium

### The Rule

> **Every unbounded queue eventually becomes a memory leak.**

Buffers must be:
- Sized intentionally
- Justified with math
- Monitored in production

## Go vs Rust Async: Real Differences

This isn't about performance—it's about semantics.

### Go Cancellation

- Explicit
- Opt-in
- Cooperative
- Goroutine continues unless it checks context

```go
select {
case <-ctx.Done():
    return
}
```

**Implication**: You must *prove* cancellation paths exist.

### Rust Async Cancellation (Tokio)

```rust
tokio::select! {
    _ = work() => {}
    _ = cancel_token.cancelled() => {}
}
```

Key difference: When a future is dropped, **it is cancelled**. Cancellation is structural. Compiler-enforced lifetimes prevent leaks at compile time.

### Comparison

| Aspect | Go | Rust |
|--------|-----|------|
| Cancellation | Manual | Structural |
| Safety | Runtime | Compile-time |
| Ergonomics | Simple | Verbose |
| Footguns | Many | Fewer |
| Control | High | High |

### The Critical Insight

> **Go favors explicit ownership.**
> **Rust favors implicit correctness.**

That's why Go needs **discipline**, not magic.

## Designing Real Systems

Everything we've covered comes together when you build:

### A Supervised Job Processing System

Components:
1. **Root supervisor** — SIGTERM handling, graceful shutdown
2. **Job queue** — Bounded channels, backpressure
3. **State owner** — Channel-owned state pattern
4. **Worker pool** — Restartable workers with context
5. **HTTP API** — Request-scoped goroutines
6. **Metrics** — Goroutine count, queue depth

This exercises:
- Supervision patterns
- Real failure modes
- Shutdown semantics
- Backpressure decisions

This is exactly the kind of system senior Go engineers build.

## Final Synthesis

> **Go concurrency is not about goroutines and channels—it is about controlling lifetimes, ownership, and pressure under failure.**

The scheduler handles the mechanics. You handle the architecture.

## Key Takeaways

1. **Leaks happen silently**—the runtime won't save you
2. **Supervision is manual**—build start/stop/wait patterns
3. **Backpressure is required**—unbounded queues are memory leaks
4. **Context is your friend**—every blocking op needs an escape
5. **Go vs Rust**—explicitness vs implicit correctness

The G-P-M model is elegant. Your job is to not break its guarantees.
