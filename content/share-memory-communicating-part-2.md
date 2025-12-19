+++
title = "Share Memory by Communicating Part 2: Lifecycle & Failure"
date = 2025-12-19
description = "Why time.Sleep is dangerous, goroutine leaks, and why every goroutine needs an owner"
[taxonomies]
tags = ["go", "concurrency", "production"]
+++

Part 1 showed the pattern. This part explains why it breaks in production without proper lifecycle management—and how goroutine leaks silently kill systems.

<!-- more -->

## The Problem with time.Sleep

The Part 1 example ended with:

```go
time.Sleep(time.Second)
fmt.Printf("reads: %d\n", atomic.LoadUint64(&readOps))
```

This is **not lifecycle control**. It's a demo hack.

What `time.Sleep` actually does:
- Keeps `main` alive for 1 second
- Lets goroutines run opportunistically
- Then process exits abruptly

When `main` returns:
- All goroutines are **force-killed**
- No cleanup
- No guarantees
- No final responses

This is **process termination**, not shutdown.

## What "Blocked Forever" Actually Means

Consider this timeline:

```go
// Reader goroutine
reads <- readOp{key: 1, resp: resp}
<-resp  // WAITING HERE
```

If the state owner goroutine panics, exits, or deadlocks—**who sends to `resp`?**

Answer: no one. The reader goroutine is **blocked forever**.

But here's what confuses people: **the program doesn't crash**.

### Goroutines Block Independently

**Critical rule**: One goroutine blocking does NOT block others—including `main`.

```go
go func() {
    <-ch  // blocks forever
}()

time.Sleep(time.Second)
// main exits normally, leaked goroutine is killed by OS
```

The Go runtime doesn't care. It only panics when **all** goroutines are blocked.

## Why This Matters in Production

In a toy program, the bug is hidden:
- Process exits
- OS kills everything
- Blocked goroutines never accumulate

In a real server:
- Handler goroutine blocks forever
- Request never completes
- Connection stays open
- Memory retained
- File descriptors leak
- Retries happen
- Traffic multiplies

This is **silent partial failure**—the worst kind.

## Two Perspectives on Blocked Goroutines

### Process Perspective (Go Runtime)

From Go's view:
- `main` is running → fine
- some goroutine is blocked → not fatal
- program exits → OS cleans up

No invariant is violated.

### System Perspective (Production)

From a service's view:
- Requests hang
- Memory grows
- Goroutines accumulate
- Throughput collapses
- No crash, no alert, no panic

The real problem:

> **You lost control over the lifecycle of work you started.**

## Every Goroutine Needs an Owner

The rule that Go doesn't enforce but production systems must:

> **Every goroutine must have an owner.**

If you can't answer:
- Who started this goroutine?
- Who stops it?
- When does it exit?

—then you've written a bug.

### Three Classes of Goroutines

**1. Request-scoped**: Tied to a request, must exit when request ends

**2. Background services**: Long-lived, must support graceful shutdown

**3. Fire-and-forget**: Almost always a bug in production

## The Mental Model

Lock this in:

```
Mutexes protect memory.
Channels protect sequencing.
Context protects lifetimes.
Supervision protects systems.
```

Most Go bugs happen when engineers use only the first two.

## The Root Cause

The Part 1 example has:
- No graceful shutdown
- No cancellation support
- No failure handling
- No lifecycle management

Without these, your state owner pattern will leak goroutines under any failure condition.

## Key Takeaway

> **"The program didn't deadlock" is meaningless.**
> **"A goroutine had no exit path" is the real bug.**

Next: [Part 3](/share-memory-communicating-part-3) adds context-based cancellation and proper shutdown.
