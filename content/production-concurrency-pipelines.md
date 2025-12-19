+++
title = "Production Concurrency: Backpressure, Pipelines & Real Failures"
date = 2025-12-19
description = "Formal backpressure reasoning, Go vs Rust async comparison, and designing a complete distributed pipeline from scratch"
[taxonomies]
tags = ["go", "concurrency", "production", "system-design", "backpressure"]
+++

This post goes beyond patterns into production system design: how backpressure actually works, why Go and Rust handle cancellation differently, and a complete walkthrough of designing a distributed pipeline that won't fail at 3am.

<!-- more -->

## A Real Goroutine Leak Postmortem

Let's start with something that actually happened.

### The System

A Go HTTP service handling ingestion requests. Each request validates input, sends work to a background processor, waits for the result.

```go
func handler(w http.ResponseWriter, r *http.Request) {
    resp := make(chan Result)

    workQueue <- Job{
        Data: r.Body,
        Resp: resp,
    }

    result := <-resp
    writeResponse(w, result)
}

func worker() {
    for job := range workQueue {
        res := process(job.Data)
        job.Resp <- res
    }
}
```

Looks clean. No races. No mutexes. Tests pass. Code review approved.

### What Happened in Production

Over 3-4 hours:
- P99 latency climbed slowly
- Memory usage grew steadily
- Goroutine count: 5k → 300k
- No panics, no errors
- CPU at ~30%

Eventually the load balancer marked the instance unhealthy. Kubernetes restarted the pod. Repeat.

### Root Cause Timeline

**T1 — Client disconnects early**

A mobile client times out. HTTP connection closes. The `handler` goroutine exits **before reading from `resp`**.

**T2 — Worker finishes processing**

```go
job.Resp <- res  // No receiver exists
```

The channel is unbuffered. No receiver exists. Worker goroutine **blocks forever**.

**T3 — Cascade failure**

- That worker never processes another job
- Work queue fills up
- New workers spawned (auto-scale)
- More workers block on abandoned channels
- Goroutine count explodes

### Why Go Didn't Save You

- Runtime sees runnable goroutines → fine
- Some goroutines blocked → fine
- No deadlock (not ALL blocked) → no panic
- Memory pressure grows slowly → no OOM yet

This is not a runtime bug. This is **missing lifecycle design**.

### The Missing Guarantees

| Missing | Consequence |
|---------|-------------|
| Request cancellation | Worker didn't know client left |
| Response buffering | Worker blocked on send |
| Ownership | No one responsible for goroutine exit |
| Backpressure | Queue grew unbounded |

### The Minimal Fix (Symptom Only)

```go
resp := make(chan Result, 1)
```

This stops the worker from blocking—but leaks work silently. The result goes nowhere.

### The Correct Fix

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // Request-scoped
    resp := make(chan Result, 1)

    select {
    case workQueue <- Job{Data: r.Body, Resp: resp, Ctx: ctx}:
    case <-ctx.Done():
        http.Error(w, "timeout", 504)
        return
    }

    select {
    case result := <-resp:
        writeResponse(w, result)
    case <-ctx.Done():
        return  // Client left, we abandon
    }
}

func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        case job := <-workQueue:
            res := process(job.Data)
            select {
            case job.Resp <- res:
            case <-job.Ctx.Done():
                // Client abandoned, drop result
            }
        }
    }
}
```

Now every blocking point has an escape. No permanent blocks possible.

## Formal Reasoning About Backpressure

Most systems fail silently because engineers don't think about backpressure formally.

### Definition

> **Backpressure** is the ability of a system to **refuse work** when downstream capacity is saturated.

If your system cannot say no, it will die.

### The Three Strategies

**1. Blocking**

```go
jobs <- job  // Blocks until receiver ready
```

Pros: Simple, natural flow control.
Cons: Ties caller lifecycle, dangerous in request paths.

**2. Bounded Queues with Rejection**

```go
select {
case jobs <- job:
    // Accepted
case <-time.After(100 * time.Millisecond):
    return ErrOverloaded
}
```

Or immediate rejection:

```go
select {
case jobs <- job:
default:
    return ErrQueueFull
}
```

Pros: Explicit overload signal, protects system.
Cons: Clients must handle rejection.

**3. Load Shedding**

Drop low-priority work, degrade functionality, preserve core correctness.

```go
func shed(job Job) bool {
    if queueLen > threshold && job.Priority < HIGH {
        metrics.DroppedJobs.Inc()
        return true  // Shed this job
    }
    return false
}
```

This is how production systems survive traffic spikes.

### The Backpressure Checklist

For **every queue** in your system, answer:

1. Is it bounded?
2. Who blocks when full?
3. Is that acceptable?
4. What is the failure mode?

If you cannot answer these → you have a bug waiting to happen.

### Why Buffered Channels Are Dangerous

Buffered channels:
- Delay backpressure signals
- Hide overload
- Move failure forward in time

They are **latency buffers**, not safety mechanisms.

```go
ch := make(chan Job, 10000)  // "Should be enough"
```

This just means you'll run out of memory 10000 jobs later.

### Real Example: Rate Limiter with Backpressure

```go
type RateLimiter struct {
    tokens chan struct{}
    refill *time.Ticker
}

func NewRateLimiter(rate int, burst int) *RateLimiter {
    rl := &RateLimiter{
        tokens: make(chan struct{}, burst),
        refill: time.NewTicker(time.Second / time.Duration(rate)),
    }

    // Pre-fill tokens
    for i := 0; i < burst; i++ {
        rl.tokens <- struct{}{}
    }

    // Refill goroutine
    go func() {
        for range rl.refill.C {
            select {
            case rl.tokens <- struct{}{}:
            default:
                // At capacity, don't overflow
            }
        }
    }()

    return rl
}

func (rl *RateLimiter) Allow(ctx context.Context) error {
    select {
    case <-rl.tokens:
        return nil  // Allowed
    case <-ctx.Done():
        return ctx.Err()  // Caller gave up
    }
}
```

This is backpressure: callers block or timeout when rate exceeded.

## Go vs Rust Async Cancellation

This matters for architecture decisions.

### Go's Model: External, Cooperative

```go
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return  // Explicit check
        case job := <-jobs:
            process(ctx, job)
        }
    }
}
```

Cancellation is:
- **External** — passed via context
- **Cooperative** — goroutine must check
- **Advisory** — can be ignored

### Rust's Model: Structural, Implicit

```rust
let task = tokio::spawn(async {
    loop {
        do_work().await;
    }
});

// Cancellation = dropping the handle
drop(task);
// or
task.abort();
```

When a future is dropped, it's cancelled. The compiler enforces lifetimes.

### Comparison

| Aspect | Go | Rust |
|--------|-----|------|
| Cancellation | Explicit checks | Implicit via drop |
| Enforcement | Runtime discipline | Type system |
| Ergonomics | Easier | Harder |
| Guarantees | Weaker | Stronger |
| Blocking code | Natural | Dangerous |
| Learning curve | Lower | Higher |

### Why Go Still Dominates Infrastructure

- Easier operational debugging
- Predictable performance model
- Better blocking syscall story
- Faster iteration cycles
- Simpler mental model for most cases

But you must **design lifetimes yourself**. Rust's compiler does it for you; Go trusts you to do it right.

### Practical Implication

In Go, you need discipline:

```go
// Every goroutine you spawn
wg.Add(1)
go func() {
    defer wg.Done()
    worker(ctx)
}()

// At shutdown
cancel()
wg.Wait()
```

In Rust, the type system prevents you from forgetting this. In Go, you can forget, and your service will leak goroutines.

## Designing a Complete Pipeline

Let's design a real system end-to-end.

### The Problem

A distributed ingestion pipeline that:
- Ingests CSV/logs/events via HTTP
- Validates records
- Transforms data
- Persists to storage
- Exposes metrics

Constraints:
- Bounded memory (no unbounded queues)
- Graceful shutdown (no dropped work)
- Cancellation-safe (no leaked goroutines)
- Restartable workers (fault tolerance)
- Observable (metrics, tracing)

### Architecture

```
                    ┌─────────────┐
    HTTP ──────────▶│  Ingestion  │
                    │   Handler   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Validate   │◀── Worker Pool (N)
                    │   Queue     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Transform  │◀── Worker Pool (M)
                    │   Queue     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   Persist   │◀── Worker Pool (K)
                    │   Queue     │
                    └─────────────┘
```

### Supervision Tree

```go
func main() {
    ctx, cancel := signal.NotifyContext(
        context.Background(),
        syscall.SIGTERM,
        os.Interrupt,
    )
    defer cancel()

    var wg sync.WaitGroup

    // Bounded queues
    validateQ := make(chan Record, 1000)
    transformQ := make(chan Record, 1000)
    persistQ := make(chan Record, 1000)

    // Start worker pools
    startPool(ctx, &wg, "validate", 10, validateQ, transformQ, validate)
    startPool(ctx, &wg, "transform", 5, transformQ, persistQ, transform)
    startPool(ctx, &wg, "persist", 3, persistQ, nil, persist)

    // Start HTTP server
    wg.Add(1)
    go func() {
        defer wg.Done()
        runHTTPServer(ctx, validateQ)
    }()

    // Wait for shutdown signal
    <-ctx.Done()
    log.Println("Shutting down...")

    // Wait for all workers to finish
    wg.Wait()
    log.Println("Shutdown complete")
}
```

### Worker Pool with Restart

```go
func startPool(
    ctx context.Context,
    wg *sync.WaitGroup,
    name string,
    count int,
    in <-chan Record,
    out chan<- Record,
    fn func(Record) (Record, error),
) {
    for i := 0; i < count; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            runWorkerWithRestart(ctx, name, id, in, out, fn)
        }(i)
    }
}

func runWorkerWithRestart(
    ctx context.Context,
    name string,
    id int,
    in <-chan Record,
    out chan<- Record,
    fn func(Record) (Record, error),
) {
    for {
        err := runWorker(ctx, name, id, in, out, fn)
        if err == nil || ctx.Err() != nil {
            return  // Clean exit or shutdown
        }

        log.Printf("Worker %s-%d crashed, restarting: %v", name, id, err)
        metrics.WorkerRestarts.WithLabelValues(name).Inc()

        // Backoff before restart
        select {
        case <-time.After(time.Second):
        case <-ctx.Done():
            return
        }
    }
}

func runWorker(
    ctx context.Context,
    name string,
    id int,
    in <-chan Record,
    out chan<- Record,
    fn func(Record) (Record, error),
) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()

    for {
        select {
        case <-ctx.Done():
            return nil

        case record, ok := <-in:
            if !ok {
                return nil  // Channel closed
            }

            result, err := fn(record)
            if err != nil {
                metrics.ProcessingErrors.WithLabelValues(name).Inc()
                continue  // Skip bad records
            }

            if out != nil {
                select {
                case out <- result:
                case <-ctx.Done():
                    return nil
                }
            }

            metrics.RecordsProcessed.WithLabelValues(name).Inc()
        }
    }
}
```

### HTTP Handler with Backpressure

```go
func runHTTPServer(ctx context.Context, validateQ chan<- Record) {
    mux := http.NewServeMux()

    mux.HandleFunc("/ingest", func(w http.ResponseWriter, r *http.Request) {
        record, err := parseRecord(r.Body)
        if err != nil {
            http.Error(w, "invalid record", 400)
            return
        }

        // Backpressure: reject if queue full
        select {
        case validateQ <- record:
            w.WriteHeader(202)  // Accepted
            metrics.RecordsAccepted.Inc()

        case <-time.After(100 * time.Millisecond):
            http.Error(w, "overloaded", 503)
            metrics.RecordsRejected.Inc()

        case <-r.Context().Done():
            return  // Client gave up
        }
    })

    server := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    go func() {
        <-ctx.Done()
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        server.Shutdown(shutdownCtx)
    }()

    if err := server.ListenAndServe(); err != http.ErrServerClosed {
        log.Printf("HTTP server error: %v", err)
    }
}
```

### Failure Scenarios Addressed

| Scenario | How It's Handled |
|----------|------------------|
| Worker panic | Recover + restart with backoff |
| Queue full | HTTP 503 + client retry |
| Slow downstream | Backpressure propagates up |
| SIGTERM | Graceful drain + wait |
| Client disconnect | Context cancellation |
| Memory pressure | Bounded queues cap growth |

### Metrics to Monitor

```go
var (
    RecordsAccepted = prometheus.NewCounter(...)
    RecordsRejected = prometheus.NewCounter(...)
    RecordsProcessed = prometheus.NewCounterVec(..., []string{"stage"})
    ProcessingErrors = prometheus.NewCounterVec(..., []string{"stage"})
    WorkerRestarts = prometheus.NewCounterVec(..., []string{"stage"})
    QueueDepth = prometheus.NewGaugeVec(..., []string{"stage"})
)
```

Alert on:
- `RecordsRejected` rate > threshold (backpressure hitting clients)
- `WorkerRestarts` rate > threshold (instability)
- `QueueDepth` near capacity (approaching backpressure)

## Key Takeaways

1. **Every goroutine needs an owner** — someone responsible for its lifecycle
2. **Every blocking point needs an escape** — context cancellation
3. **Every queue needs bounds** — unbounded = eventual OOM
4. **Backpressure is not optional** — design it explicitly
5. **Go trusts you** — Rust's compiler catches mistakes you must catch yourself
6. **Test failure modes** — not just happy paths

This is the difference between software that works in development and software that survives production.
