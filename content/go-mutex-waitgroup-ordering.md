+++
title = "Go Mutex vs WaitGroup: Safety, Coordination, and Ordering"
date = 2025-12-19
description = "A mechanical deep dive into why Mutexes don't guarantee ordering, how WaitGroups coordinate lifecycles, and the mental model for correct concurrency."
[taxonomies]
tags = ["go", "concurrency", "mutex", "waitgroup", "mental-model"]
+++

We often confuse "safety" with "ordering". This post dissects the mechanical difference between `sync.Mutex` (safety) and `sync.WaitGroup` (coordination), and why a race-free program can still be nondeterministic.

<!-- more -->

## The Canonical Pattern

Here is the classic "shared state with workers" pattern in Go:

```go
type Container struct {
    mu       sync.Mutex
    counters map[string]int
}

func (c *Container) inc(name string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.counters[name]++
}

func main() {
    c := Container{
        counters: map[string]int{"a": 0, "b": 0},
    }

    var wg sync.WaitGroup

    doIncrement := func(name string, n int) {
        for i := 0; i < n; i++ {
            c.inc(name)
        }
        wg.Done()
    }

    wg.Add(3)
    go doIncrement("a", 10000)
    go doIncrement("a", 10000)
    go doIncrement("b", 10000)

    wg.Wait()
    fmt.Println(c.counters)
}
```

This code is **100% correct and race-free**. It outputs:
```
map[a:20000 b:10000]
```

But why exactly? And what does it *not* leverage?

## The Mental Model: Safety vs. Coordination

These two primitives solve orthogonal problems.

### 1. Safety (`sync.Mutex`)

**Problem**: "Can two goroutines touch this data at the same time?"
**Answer**: No.

The mutex guarantees **mutual exclusion**. Inside `inc()`, the lock ensures that the read-modify-write cycle (`counters[name]++`) is atomic.

> **Rule**: The mutex protects the *critical section*, not the goroutine.

It does not lock the goroutine for its entire life. It locks for nanoseconds—just long enough to increment the integer.

### 2. Coordination (`sync.WaitGroup`)

**Problem**: "Is all concurrent work finished?"
**Answer**: Wait until yes.

The `WaitGroup` manages **goroutine lifecycles**. It answers the question: "How many goroutines are still running?"

> **Rule**: `WaitGroup` has NOTHING to do with data safety.

You can use a WaitGroup without a mutex (e.g., parallel independent tasks), and a mutex without a WaitGroup (e.g., a long-running server).

## The Execution Timeline

Let's trace the execution mechanically:

1.  **Setup**: `main` creates `Container` and sets `wg.Add(3)`.
2.  **Spawn**: Three goroutines start. `main` hits `wg.Wait()` and blocks.
3.  **Interleaving**:
    *   Goroutine 1 calls `c.inc("a")`. It acquires the lock.
    *   Goroutine 2 calls `c.inc("a")`. It tries to acquire the lock but **blocks** because G1 holds it.
    *   G1 increments, unlocks.
    *   G2 unblocks, acquires lock, increments, unlocks.
    *   G3 runs freely on key "b" (but still contends for the same mutex `mu`).
4.  **Completion**:
    *   G1 finishes loop, calls `wg.Done()`. Counter → 2.
    *   G2 finishes loop, calls `wg.Done()`. Counter → 1.
    *   G3 finishes loop, calls `wg.Done()`. Counter → 0.
5.  **Resume**: `wg.Wait()` unblocks. `main` prints the result.

## The Ordering Misconception

Here is the critical insight for senior engineers.

**Hypothesis**: Since we spawn G1 ("a"), then G2 ("a"), then G3 ("b"), will "a" always be updated before "b"?

**Reality**: **NO.**

A mutex guarantees **safety**, not **ordering**.

The Go scheduler is free to run these goroutines in any order. The interleaving is nondeterministic. You might see:
*   `b++`, `a++`, `a++`
*   `a++`, `b++`, `a++`

### Mutexes Protect Memory, Not Meaning

If your business logic requires that "User A must be processed before User B", a mutex **cannot** enforce that.

*   **Mutex**: Prevents corruption (technical correctness).
*   **Ordering**: Enforces sequence (semantic correctness).

If you need ordering, you need:
1.  **Channels**: A single unbuffered channel enforces run-order (requests processed as received).
2.  **Sequencing**: Run step 1, wait, then run step 2.
3.  **State Owner**: A single goroutine reading from a channel applies updates in the order they arrive.

## When to Use This Pattern

Use the Mutex + WaitGroup pattern when:
*   State is **small** and shared.
*   Operations are **simple** (increments, swaps).
*   **Performance** matters (mutexes are faster than channels for simple state).
*   **Order doesn't matter** (commutative operations).

## Summary

| Tool | Purpose | Mental Model |
|------|---------|--------------|
| `sync.Mutex` | **Safety** | "One at a time." Protects memory access. |
| `sync.WaitGroup` | **Coordination** | "Wait for all." Protects program lifecycle. |
| Channels | **Flow/Order** | "Pass the baton." Enforces sequence and ownership. |

Don't use a WaitGroup to protect data. Don't use a Mutex to signal completion. And don't expect either to guarantee execution order.