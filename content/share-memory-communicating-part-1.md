+++
title = "Share Memory by Communicating Part 1: The Pattern"
date = 2025-12-19
description = "Understanding Go's channel-owned state pattern - the actor model in disguise"
[taxonomies]
tags = ["go", "concurrency", "patterns"]
+++

Go's famous proverb—"Don't communicate by sharing memory; share memory by communicating"—sounds abstract until you see it in action. This post breaks down the pattern, why it exists, and when to use it.

<!-- more -->

## The Core Idea

Instead of protecting shared state with mutexes:

```go
var mu sync.Mutex
var state map[string]int

func get(key string) int {
    mu.Lock()
    defer mu.Unlock()
    return state[key]
}
```

You confine state to a single goroutine and access it via channels:

```go
type readOp struct {
    key  string
    resp chan int
}

reads := make(chan readOp)

// State owner goroutine
go func() {
    state := make(map[string]int)
    for r := range reads {
        r.resp <- state[r.key]
    }
}()

// Client usage
resp := make(chan int)
reads <- readOp{"mykey", resp}
value := <-resp
```

The map is never touched by multiple goroutines. One goroutine owns it; others **request** access via channels.

## Why This Pattern Exists

Mutex-based code has well-known problems:

```go
mu.Lock()
state[key] = val
mu.Unlock()

// Problems:
// - Easy to forget to lock
// - Easy to deadlock (lock ordering)
// - Invariants spread across codebase
// - Hard to reason about correctness
```

Channel-owned state solves these by construction:

- **State has one owner** — no concurrent access possible
- **Invariants live in one place** — the owner goroutine
- **No accidental races** — the compiler can't even express them
- **Easier to reason about** — state transitions are explicit

This is essentially the **actor model**, disguised as idiomatic Go.

## Complete Working Example

```go
package main

import (
    "fmt"
    "sync/atomic"
    "time"
)

type readOp struct {
    key  int
    resp chan int
}

type writeOp struct {
    key  int
    val  int
    resp chan bool
}

func main() {
    var readOps, writeOps uint64

    reads := make(chan readOp)
    writes := make(chan writeOp)

    // State owner goroutine
    go func() {
        state := make(map[int]int)
        for {
            select {
            case read := <-reads:
                read.resp <- state[read.key]
            case write := <-writes:
                state[write.key] = write.val
                write.resp <- true
            }
        }
    }()

    // 100 reader goroutines
    for i := 0; i < 100; i++ {
        go func() {
            for {
                resp := make(chan int)
                reads <- readOp{key: i % 5, resp: resp}
                <-resp
                atomic.AddUint64(&readOps, 1)
                time.Sleep(time.Millisecond)
            }
        }()
    }

    // 10 writer goroutines
    for i := 0; i < 10; i++ {
        go func() {
            for {
                resp := make(chan bool)
                writes <- writeOp{key: i % 5, val: i, resp: resp}
                <-resp
                atomic.AddUint64(&writeOps, 1)
                time.Sleep(time.Millisecond)
            }
        }()
    }

    time.Sleep(time.Second)
    fmt.Printf("reads: %d, writes: %d\n",
        atomic.LoadUint64(&readOps),
        atomic.LoadUint64(&writeOps))
}
```

## How It Works

1. **State owner** runs in a dedicated goroutine
2. **Readers** send requests on `reads` channel, block waiting for response
3. **Writers** send requests on `writes` channel, block waiting for acknowledgment
4. **Owner** processes one request at a time via `select`
5. **No locks** — concurrency is handled by channel synchronization

## Trade-offs (Be Honest)

### Pros
- No locks
- No races
- Clear ownership
- Easy reasoning

### Cons
- One goroutine can be a bottleneck
- Higher latency per operation (channel overhead)
- More allocations (response channels)

This pattern shines when **correctness matters more than raw throughput**.

## When to Use What

| Situation | Use |
|-----------|-----|
| Simple counters | `atomic` |
| Small critical sections | `sync.Mutex` |
| Complex shared state | Channel-owned goroutine |
| High-throughput cache | Sharded mutex / lock-free |
| Distributed logic | Actor model (this) |

## Key Insight

The `time.Sleep(time.Second)` in the example is **not lifecycle management**. It's a demo hack. Real systems need proper shutdown, which we cover in Part 2.

## What's Missing

The example above has no:
- Graceful shutdown
- Cancellation support
- Failure handling
- Lifecycle management

These are not optional in production. Next: [Part 2](/share-memory-communicating-part-2) covers why `time.Sleep` is dangerous and how to fix it.
