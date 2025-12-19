+++
title = "The Go Concurrency Toolbox: When to Use What"
date = 2025-12-19
description = "Atomics, mutexes, channels, condition variables - a decision framework for choosing the right concurrency primitive"
[taxonomies]
tags = ["go", "concurrency", "production", "decision-framework"]
+++

Go gives you multiple concurrency primitives. Each exists for a reason. This post provides a decision framework for choosing the right one—not by feature comparison, but by workload analysis.

<!-- more -->

## The Complete Toolbox

Go provides exactly these primitives for shared state:

1. **Atomics** (`sync/atomic`)
2. **Mutexes** (`sync.Mutex`, `sync.RWMutex`)
3. **Channels** (state ownership / actor pattern)
4. **Condition variables** (`sync.Cond`)
5. **RCU-style patterns** (advanced, using atomic pointer swaps)

No transactional memory. No STM. No implicit locking. Go prefers explicitness.

## 1. Atomics

### What They Are

CPU-level atomic instructions. Lock-free. Very low overhead.

```go
var counter uint64

atomic.AddUint64(&counter, 1)
atomic.LoadUint64(&counter)
```

### Use When

- Variable is a **simple primitive** (int, uint, pointer)
- Operations are **independent** (no invariants with other variables)
- Hot path where lock overhead matters

### Examples

- Request counters
- Throughput metrics
- Health flags
- Sequence numbers
- Feature flags (with atomic.Bool)

### Limitations

- Only primitive types
- No compound operations (check-then-act)
- Easy to misuse for complex logic

### Common Mistake

```go
if atomic.LoadUint64(&x) < 100 {
    atomic.AddUint64(&x, 1)  // Race: x might be >= 100 now
}
```

This is NOT atomic. The check and add are separate operations.

## 2. Mutexes

### What They Are

Mutual exclusion locks. Guard shared memory. Classic approach.

```go
var mu sync.Mutex
var state map[string]int

mu.Lock()
state[key] = val
mu.Unlock()
```

### Use When

- Multiple goroutines read/write shared structures
- State is moderately complex
- Performance matters
- Invariants span multiple fields

### RWMutex for Read-Heavy

```go
var mu sync.RWMutex

// Readers (concurrent)
mu.RLock()
val := state[key]
mu.RUnlock()

// Writers (exclusive)
mu.Lock()
state[key] = val
mu.Unlock()
```

### Trade-offs

| Pros | Cons |
|------|------|
| Flexible | Discipline-based |
| Performant | Easy to forget lock |
| Well-understood | Deadlock risk |
| Standard | Invariants spread across code |

### Common Mistakes

**Forgetting to unlock**:
```go
mu.Lock()
if err != nil {
    return err  // Lock never released!
}
mu.Unlock()
```

Fix: Always use `defer mu.Unlock()` or be very careful.

**Wrong lock granularity**:
```go
mu.Lock()
expensiveOperation()  // Holds lock too long
mu.Unlock()
```

**Lock ordering violations**:
```go
// Goroutine 1        // Goroutine 2
mu1.Lock()            mu2.Lock()
mu2.Lock()            mu1.Lock()  // Deadlock
```

## 3. Channels (State Ownership)

### What They Are

Message-passing synchronization. One goroutine owns the data. Others communicate.

```go
requests <- readOp{key: k, resp: ch}
value := <-ch
```

### Use When

- State is complex
- Invariants matter
- Correctness > raw throughput
- You want ownership enforced by design

### Trade-offs

| Pros | Cons |
|------|------|
| No accidental sharing | Serialized access |
| Clear ownership | Channel overhead |
| Composable | Potential bottleneck |
| Context integration | More allocations |

### When NOT to Use

- Simple counters (use atomics)
- Read-heavy caches (use RWMutex)
- Ultra-high throughput (profile first)

## 4. Condition Variables

### What They Are

Signaling mechanism for goroutines waiting on a condition. Used with mutexes.

```go
var mu sync.Mutex
var cond = sync.NewCond(&mu)
var ready bool

// Waiter
cond.L.Lock()
for !ready {
    cond.Wait()  // Releases lock, waits, reacquires
}
// proceed
cond.L.Unlock()

// Signaler
cond.L.Lock()
ready = true
cond.Signal()  // or cond.Broadcast()
cond.L.Unlock()
```

### Use When

- Goroutines must wait for a complex condition
- Producer-consumer with special logic
- Classic thread coordination patterns

### In Practice

Rare in idiomatic Go. Channels usually simpler:

```go
<-readyChan  // Wait for signal
```

Use `sync.Cond` when you need:
- Broadcast to many waiters
- Complex predicates
- Integration with existing mutex-protected state

## 5. RCU-Style Patterns (Advanced)

### What It Is

Copy-on-write with atomic pointer swaps. Many readers, rare writers.

```go
type Config struct {
    // ... fields
}

var configPtr atomic.Pointer[Config]

// Readers (no lock)
cfg := configPtr.Load()
use(cfg)

// Writer (rare)
newCfg := &Config{...}
configPtr.Store(newCfg)
```

### Use When

- Extremely read-heavy (1000:1 read/write ratio)
- Config hot-reload
- Version bumps
- Lookup tables that rarely change

### Trade-offs

- Very fast reads (no synchronization)
- Writers must create full copy
- Memory overhead
- Complex if state is mutable

## The Decision Framework

Don't choose by feature. Choose by workload.

### Question 1: What kind of state?

| State Type | Best Tool |
|------------|-----------|
| Single counter | Atomic |
| Single flag | Atomic |
| Small struct, frequent mutation | Mutex |
| Complex invariants | Channel-owned |
| Large read-heavy data | RWMutex or RCU |

### Question 2: Access pattern?

| Pattern | Best Tool |
|---------|-----------|
| Many writers, simple ops | Atomic |
| Few writers, many readers | RWMutex |
| Mixed, with invariants | Mutex or Channel |
| Complex coordination | Channel |

### Question 3: Failure mode preference?

| Preferred Failure | Best Tool |
|-------------------|-----------|
| Race detector catches it | Mutex |
| Design prevents bugs | Channel |
| Performance over safety | Atomic |

### Question 4: Team context?

| Context | Best Tool |
|---------|-----------|
| Strict code review, experienced team | Mutex |
| Junior team, critical correctness | Channel |
| Performance-critical, well-tested | Atomic |

## Decision Cheat Sheet

```
┌─────────────────────────────────────────┐
│  Is it a simple counter/flag?           │
│     YES → Atomic                        │
│     NO ↓                                │
├─────────────────────────────────────────┤
│  Is it read-heavy (100:1+)?             │
│     YES → RWMutex or RCU                │
│     NO ↓                                │
├─────────────────────────────────────────┤
│  Are there complex invariants?          │
│     YES → Channel-owned goroutine       │
│     NO ↓                                │
├─────────────────────────────────────────┤
│  Is throughput critical?                │
│     YES → Mutex (profile to confirm)    │
│     NO → Channel (safer default)        │
└─────────────────────────────────────────┘
```

## Workload Examples

### Large File Processing

**Pattern**: Pipeline + fan-out/fan-in

**Tools**:
- Channels for data flow between stages
- Worker pools for parallelism
- Atomics for progress counters

```
Reader → Parser → Workers → Aggregator → Writer
         (channels)  (pool)   (channel)
```

### In-Memory Cache

**Pattern**: Depends on access ratio

**If read-heavy**: RWMutex or sharded maps

```go
type Cache struct {
    mu sync.RWMutex
    m  map[string]Value
}
```

**If write-heavy**: Channel-owned with batching

### Rate Limiter

**Pattern**: State owner

**Tools**: Channel-owned goroutine

```go
type RateLimiter struct {
    requests chan request
}

func (r *RateLimiter) Allow(ctx context.Context) bool {
    resp := make(chan bool, 1)
    r.requests <- request{resp: resp, ctx: ctx}
    return <-resp
}
```

### Metrics Collection

**Pattern**: Distributed counters

**Tools**: Atomics (or sharded atomics)

```go
type Metrics struct {
    requests atomic.Uint64
    errors   atomic.Uint64
    latency  atomic.Uint64  // For histogram, need more
}
```

### Configuration Hot-Reload

**Pattern**: RCU

**Tools**: Atomic pointer swap

```go
var config atomic.Pointer[Config]

// In reload handler
newConfig := parseConfig()
config.Store(newConfig)

// In request handlers
cfg := config.Load()
```

## The Golden Rule

> **If you must share memory, protect it.**
> **If you can avoid sharing memory, communicate instead.**

This is the principle behind all Go concurrency guidance. The tools exist to implement it in different contexts.

## Key Takeaways

1. **Atomics**: Simple values, independent operations, metrics
2. **Mutexes**: Structured state, performance-critical, team discipline
3. **Channels**: Complex invariants, ownership matters, correctness-first
4. **Cond**: Rare, for complex waiting conditions
5. **RCU**: Extreme read-heavy, rare writes

There's no universal best. There's only best **for this workload, this team, this context**.
