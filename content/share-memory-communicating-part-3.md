+++
title = "Share Memory by Communicating Part 3: Context & Supervision"
date = 2025-12-19
description = "Adding proper cancellation, shutdown, and supervision to channel-owned state"
[taxonomies]
tags = ["go", "concurrency", "context", "production"]
+++

Part 2 explained why the naive pattern breaks. This part fixes it with proper context-based cancellation and supervision.

<!-- more -->

## The Core Rule

> **The state owner must respect cancellation. Only it decides when to exit.**

Callers can *request* cancellation, but the owner goroutine controls its own shutdown.

## Step 1: Context-Aware Request Types

```go
type readOp struct {
    ctx  context.Context
    key  int
    resp chan int
}

type writeOp struct {
    ctx  context.Context
    key  int
    val  int
    resp chan bool
}
```

Why this matters:
- Allows per-request timeout
- Allows request abandonment
- Prevents goroutine leaks

## Step 2: Context-Aware State Owner

```go
func stateOwner(ctx context.Context, reads <-chan readOp, writes <-chan writeOp) {
    state := make(map[int]int)

    for {
        select {
        case <-ctx.Done():
            return  // Graceful shutdown

        case r := <-reads:
            select {
            case r.resp <- state[r.key]:
            case <-r.ctx.Done():
                // Caller abandoned request
            }

        case w := <-writes:
            select {
            case <-w.ctx.Done():
                // Caller abandoned
            default:
                state[w.key] = w.val
                select {
                case w.resp <- true:
                case <-w.ctx.Done():
                }
            }
        }
    }
}
```

### Why the Double Select?

The inner `select` is critical:

```go
select {
case r.resp <- state[r.key]:
case <-r.ctx.Done():
}
```

Because the caller may cancel **before receiving the response**. Without this, the owner blocks forever on `r.resp <-`.

This is a **classic goroutine leak bug**.

## Step 3: Context-Aware Callers

```go
func read(ctx context.Context, reads chan<- readOp, key int) (int, error) {
    resp := make(chan int, 1)  // Buffered!

    req := readOp{
        ctx:  ctx,
        key:  key,
        resp: resp,
    }

    select {
    case reads <- req:
    case <-ctx.Done():
        return 0, ctx.Err()
    }

    select {
    case val := <-resp:
        return val, nil
    case <-ctx.Done():
        return 0, ctx.Err()
    }
}
```

**Note the buffered response channel**. This prevents the owner from blocking if the caller exits early.

## What "Supervision" Means in Go

Go doesn't have supervisors like Erlang. Supervision in Go means **three explicit guarantees**:

1. **Start** — goroutine is launched
2. **Stop** — goroutine can be cancelled
3. **Wait** — caller can block until goroutine exits

Nothing more, nothing less.

## Canonical Supervision Structure

```go
type Supervisor struct {
    ctx    context.Context
    cancel context.CancelFunc
    wg     sync.WaitGroup
}

func NewSupervisor(parent context.Context) *Supervisor {
    ctx, cancel := context.WithCancel(parent)
    return &Supervisor{ctx: ctx, cancel: cancel}
}

func (s *Supervisor) Go(fn func(ctx context.Context)) {
    s.wg.Add(1)
    go func() {
        defer s.wg.Done()
        fn(s.ctx)
    }()
}

func (s *Supervisor) Stop() {
    s.cancel()
    s.wg.Wait()
}
```

This is **90% of production Go concurrency**.

## Putting It Together

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    reads := make(chan readOp)
    writes := make(chan writeOp)

    var wg sync.WaitGroup

    // Start state owner
    wg.Add(1)
    go func() {
        defer wg.Done()
        stateOwner(ctx, reads, writes)
    }()

    // Start workers...

    // Wait for shutdown signal
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, os.Interrupt, syscall.SIGTERM)
    <-sigCh

    // Graceful shutdown
    cancel()
    wg.Wait()
}
```

## The Original Failure Scenario — Fixed

### Before (Part 1)
- Owner dies → reader blocks forever → goroutine leaks → system degrades

### After (This Pattern)
- Owner dies → supervisor cancels context → reader sees `ctx.Done()` → reader exits → resources released → system remains healthy

The runtime behavior is unchanged. **Your guarantees changed.**

## Comparison: Mutex vs Channel Pattern

| Concern | Mutex | Channel-owned |
|---------|-------|---------------|
| Data races | Solved | Solved |
| Lifecycle | Not addressed | Explicit |
| Cancellation | Not addressed | Built-in |
| Ownership | Implicit | Explicit |
| Partial failure | Breaks silently | Handled |

## The Final Rule

> **If a goroutine can block, it must be cancellable.**
> **If it is cancellable, someone must own that cancellation.**

Next: [Part 4](/share-memory-communicating-part-4) covers advanced topics—buffered channels, Kubernetes shutdown, errgroup supervision, and scaling.
