+++
title = "The Go Scheduler: G-P-M Model Explained"
date = 2025-12-19
description = "Understanding how Go runs millions of goroutines on few OS threads - the G-P-M model demystified"
[taxonomies]
tags = ["go", "concurrency", "internals", "scheduler"]
+++

Go runs many goroutines on few OS threads. Understanding how is the difference between hoping your concurrent code works and *knowing* it works.

<!-- more -->

## The Big Picture

> Go runs **many goroutines** on **few OS threads** using a **cooperative + preemptive scheduler** managed by the runtime.

You don't control threads. You control concurrency structure.

## The G-P-M Model

Go scheduling is built around three entities:

```
G = Goroutine  (unit of work)
P = Processor  (scheduler context)
M = Machine    (OS thread)
```

The one-sentence summary:

> **G runs on M, but only when it has a P.**

### G — Goroutine

- Lightweight task
- Starts with ~2KB stack (grows as needed)
- Can be parked, resumed, migrated
- Millions are fine

Think: **task / coroutine**

### M — Machine (OS Thread)

- Actual kernel thread
- Expensive to create
- Blocks on syscalls
- Created/destroyed by runtime

Think: **CPU execution vessel**

### P — Processor (Scheduler Token)

- Holds run queue
- Owns resources (timers, GC state)
- Limits parallelism

Think: **permission to run Go code**

### Key Invariant

```
At most GOMAXPROCS Ps exist
At most GOMAXPROCS Ms actively executing Go code
```

Default: `GOMAXPROCS = number of CPU cores`

## Visual Model

```
[G] [G] [G] [G]   ← goroutines
     ↓   ↓
   [P] [P]        ← processors (scheduler slots)
     ↓   ↓
   [M] [M]        ← OS threads
     ↓   ↓
   CPU CPU
```

Many G. Few P. Few active M.

## Scheduling Lifecycle

When you write `go work()`:

1. A new **G** is created
2. G is put into a **run queue**
3. A **P** picks G
4. P binds to an **M**
5. M executes G
6. G blocks / yields / finishes
7. P schedules next G

## Run Queues

### Local Run Queue (per P)

Each P has its own queue:

```
P0: G1 → G2 → G3
P1: G4 → G5
```

Fast (mostly lock-free), FIFO-ish.

### Global Run Queue

Shared across Ps. Used when:
- New goroutine created
- Local queue overflows
- Fairness balancing

### Priority

1. Local run queue
2. Global run queue
3. Work stealing

## Work Stealing

If a P runs out of work:

```
P0 queue empty → steal half from another P
```

Why this matters:
- Keeps CPUs busy
- Avoids hotspots
- Enables scalability

**You don't need to balance goroutines manually.** The scheduler already does.

## Blocking Behavior

This is where Go shines.

### Blocking Syscall

```go
data := read(fd)
```

What happens:
1. M enters syscall
2. M is **detached from P**
3. P is handed to another M
4. Other goroutines continue running

> Blocking syscalls do NOT block the scheduler.

### Channel/Mutex Blocking

```go
<-ch
```

What happens:
1. G is parked
2. P schedules another G
3. No OS thread blocked

This is why Go scales.

## Preemption (Since Go 1.14)

Older Go was cooperative-only. That caused starvation.

Now Go has **asynchronous preemption**.

```go
for {
    // tight loop - no function calls
}
```

Before 1.14: Could starve other goroutines.
After 1.14: Runtime can interrupt this safely.

How it works:
- Runtime inserts safe points
- Signal-based interruption
- Stack is safely inspected

> Prevents one goroutine from hogging the CPU.

## Fairness

**Does Go guarantee fairness?** No strict fairness.

**Does Go prevent starvation?** Practically, yes.

Mechanisms:
- Global queue injection
- Preemption
- Work stealing

If your goroutine starves others, it's usually a design bug, not scheduler failure.

## GC Interaction

Go GC is concurrent, mostly non-blocking, and scheduler-aware.

### Coordination

- GC needs all Ps to reach safe points
- Short STW (stop-the-world) phases
- Scheduler cooperates with GC

Before Go 1.14, tight loops could cause GC starvation. Preemption fixed this.

## GOMAXPROCS Tuning

```go
runtime.GOMAXPROCS(n)
```

| Scenario | Recommendation |
|----------|----------------|
| CPU-bound | cores |
| I/O-heavy | cores |
| Container with CPU limit | Set explicitly |
| Latency-sensitive | Test and measure |

### Container Gotcha

Kubernetes limits CPU, but Go sees host CPUs.

```bash
# Wrong: sees 64 cores in container limited to 2
# Right: set explicitly
GOMAXPROCS=2
```

Or use [automaxprocs](https://github.com/uber-go/automaxprocs).

## Go vs async/await

| Go Scheduler | JS async/await |
|--------------|----------------|
| Preemptive | Cooperative |
| Multi-core | Single-threaded (mostly) |
| True parallelism | Concurrency illusion |
| No event loop | Central event loop |

> Go doesn't "await"—it **schedules**.

## Debugging the Scheduler

### Trace Tool

```bash
go test -run TestX -trace trace.out
go tool trace trace.out
```

You can see:
- Goroutine creation
- Blocking events
- Preemption
- GC pauses

### pprof Indicators

Look for:
- Too many runnable goroutines
- Long-running goroutines
- Scheduler contention

## Common Scheduler-Related Bugs

### Goroutine Leaks

```go
go func() {
    <-ch  // never receives
}()
```

### Unbounded Goroutines

```go
for {
    go work()  // spawns forever
}
```

### CPU Hogs

```go
for {
    // no yields, no function calls
}
```

## Final Mental Model

```
G = work
P = permission to run
M = thread
Scheduler = traffic controller
```

## Key Takeaway

> **The Go scheduler multiplexes millions of goroutines onto a small number of OS threads using the G-P-M model, work stealing, and preemption—making concurrency cheap, parallelism real, and blocking mostly invisible unless you design it poorly.**

Understanding this model helps you:
- Debug hanging programs
- Reason about performance
- Design correct concurrent systems
- Tune for containers
