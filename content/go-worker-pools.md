+++
title = "Go Worker Pools: Channels as Synchronization"
date = 2025-12-20
description = "How worker pools use channels for both data transport and lifecycle coordination—eliminating the need for WaitGroups."
[taxonomies]
tags = ["go", "concurrency", "worker-pool", "channels", "patterns"]
+++

The worker pool is Go's workhorse pattern. What makes Go's version unique is how channels serve as *both* the data pipeline *and* the synchronization mechanism. This post dissects the mechanics.

<!-- more -->

## The Pattern

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        fmt.Println("worker", id, "started job", j)
        time.Sleep(time.Second) // Simulate work
        fmt.Println("worker", id, "finished job", j)
        results <- j * 2
    }
}

func main() {
    const numJobs = 5
    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)

    // Start 3 workers
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }

    // Send jobs
    for j := 1; j <= numJobs; j++ {
        jobs <- j
    }
    close(jobs)

    // Collect results
    for a := 1; a <= numJobs; a++ {
        <-results
    }
}
```

Notice: **No `WaitGroup`**. Yet the program doesn't exit early. Why?

## The Core Insight

> **Receiving from a channel is a blocking operation.**

This line:
```go
<-results
```
...is semantically equivalent to `wg.Wait()`. It blocks until a value arrives.

The loop:
```go
for a := 1; a <= numJobs; a++ {
    <-results
}
```
...blocks until **exactly 5 results** are received. This is the synchronization.

## Timeline Walkthrough

Let's trace execution step by step.

### T0: Setup

```go
jobs := make(chan int, numJobs)    // Buffered, capacity 5
results := make(chan int, numJobs) // Buffered, capacity 5
```

Two empty buffers. No blocking yet.

### T1: Start Workers

```go
for w := 1; w <= 3; w++ {
    go worker(w, jobs, results)
}
```

Three goroutines spawn. Each immediately tries:
```go
for j := range jobs { ... }
```

But `jobs` is empty. **All 3 workers block**, waiting for work.

### T2: Send Jobs

```go
for j := 1; j <= numJobs; j++ {
    jobs <- j
}
```

Main sends 5 jobs. Since `jobs` is buffered (capacity 5), all sends complete without blocking.

Meanwhile, workers wake up as jobs become available:
- Worker 1 grabs job 1
- Worker 2 grabs job 2
- Worker 3 grabs job 3
- Workers 1/2/3 each grab remaining jobs 4/5 as they finish

### T3: Close Jobs

```go
close(jobs)
```

This is **critical**. It signals:

> "No more jobs will ever be sent."

When workers finish their current job and call `range jobs` again, they see the channel is closed and exit cleanly.

**Without `close(jobs)`**: Workers block forever waiting for more work → goroutine leak.

### T4: Collect Results

```go
for a := 1; a <= numJobs; a++ {
    <-results
}
```

Main blocks here, receiving one result at a time. When all 5 results arrive, the loop exits, and `main` terminates.

## Why Buffered Channels Matter

### Unbuffered (`make(chan int)`)

```
Worker finishes → waits for main to receive
Main receives → worker unblocks
```

Like hand-to-hand delivery. Tight coupling. More context switches.

### Buffered (`make(chan int, 5)`)

```
Worker finishes → drops result in buffer → continues
Main collects later
```

Like a mailbox. Workers don't wait for the receiver. **Decoupled. Faster.**

### Rule of Thumb

| Scenario | Channel Type |
|----------|--------------|
| Signaling (done, cancel) | Unbuffered |
| Worker pools | Buffered |
| Pipelines | Buffered |
| Backpressure | Buffered (bounded) |
| Strict synchronization | Unbuffered |

## Channel Direction Annotations

Notice the function signature:

```go
func worker(id int, jobs <-chan int, results chan<- int)
```

| Annotation | Meaning |
|------------|---------|
| `<-chan int` | Receive-only |
| `chan<- int` | Send-only |
| `chan int` | Bidirectional |

This is **compile-time safety**. A worker *cannot* accidentally close `jobs` or read from `results`. The type system enforces the contract.

## Channels vs. WaitGroup

Both can coordinate goroutine lifecycle. When to use which?

| Need | Tool |
|------|------|
| Wait for N goroutines, no data | `WaitGroup` |
| Wait for N results, use data | Channel (count receives) |
| Pipeline (stage → stage) | Channel |
| Fan-out / Fan-in | Channel |
| Phased execution | `WaitGroup` (phase barriers) |
| Cancellation | `context.Context` |

In the worker pool, channels naturally encode **both data flow and lifecycle**:
- Each result on `results` = one completed job
- Receiving `numJobs` results = all jobs done

This eliminates the need for a separate counter.

## Common Mistakes

### 1. Forgetting to close the jobs channel

```go
// close(jobs) // ← Missing!
```

Workers block forever on `range jobs`. Goroutine leak.

### 2. Receiving fewer results than expected

```go
for a := 1; a <= numJobs-1; a++ { // ← Off by one
    <-results
}
```

One worker's `results <- ...` blocks forever (if unbuffered) or the result is orphaned (if buffered). Leak.

### 3. Closing the results channel from a worker

```go
func worker(...) {
    for j := range jobs {
        results <- j * 2
    }
    close(results) // ← PANIC if multiple workers
}
```

Only **one** goroutine should close a channel. If multiple workers try, you get a panic. Solution: close `results` in `main` after all workers finish, often coordinated with a `WaitGroup` if needed.

### 4. Sending on a closed channel

```go
close(jobs)
jobs <- 10 // ← PANIC
```

Sends to a closed channel panic. Always close from the sender side, never the receiver.

## Mental Model

```
                  ┌─────────────┐
                  │   Worker 1  │───┐
┌──────────┐      ├─────────────┤   │      ┌──────────┐
│  Main    │ ───► │   Worker 2  │ ──┼───►  │   Main   │
│ (sender) │      ├─────────────┤   │      │(receiver)│
└──────────┘      │   Worker 3  │───┘      └──────────┘
   jobs ──────────────────────────────────►   results
```

- `jobs` flows from main to workers
- `results` flows from workers to main
- `close(jobs)` signals "no more work"
- Receiving `numJobs` results signals "all done"

## Production Considerations

### Graceful Shutdown

In real systems, you need cancellation:

```go
func worker(ctx context.Context, jobs <-chan int, results chan<- int) {
    for {
        select {
        case <-ctx.Done():
            return // Shutdown signal
        case j, ok := <-jobs:
            if !ok {
                return // Channel closed
            }
            // Process job
            results <- j * 2
        }
    }
}
```

### Error Handling

Workers can fail. Propagate errors via a separate channel or use `errgroup`:

```go
import "golang.org/x/sync/errgroup"

g, ctx := errgroup.WithContext(context.Background())
for w := 0; w < 3; w++ {
    g.Go(func() error {
        return worker(ctx, jobs, results)
    })
}
if err := g.Wait(); err != nil {
    // Handle first error
}
```

### Backpressure

If workers are slow and jobs pile up, a bounded buffer provides natural backpressure:

```go
jobs := make(chan int, 100) // Buffer of 100
```

When full, `jobs <- j` blocks, slowing the producer.

## Summary

1. **Channels are synchronization**. Receiving blocks; counting receives replaces `WaitGroup`.
2. **`close(jobs)` is mandatory**. It signals workers to exit.
3. **Buffered = decoupled**. Workers don't wait for receivers.
4. **Direction annotations are contracts**. Enforced at compile time.
5. **One closer per channel**. Close from the sender side only.

The worker pool is foundational. Master it, and you understand Go concurrency patterns at a senior level.
