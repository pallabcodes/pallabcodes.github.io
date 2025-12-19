+++
title = "Scaling Go Actors: Sharding State Owners"
date = 2025-12-19
description = "When a single state-owner goroutine becomes a bottleneck, shard by key to scale while preserving correctness"
[taxonomies]
tags = ["go", "concurrency", "scaling", "actors", "sharding"]
+++

The actor pattern serializes access to state through a single goroutine. This guarantees correctness but creates a throughput ceiling. When that becomes your bottleneck, the answer is sharding.

<!-- more -->

## The Limitation

```
many goroutines
      ↓
   ONE goroutine (state owner)
      ↓
   serialized access
```

This design guarantees correctness, but:
- All requests are serialized
- Throughput capped by one CPU core
- Latency increases under load

This isn't a bug—it's a design trade-off. And at some scale, you'll hit it.

## The Scaling Idea: Shard by Key

Instead of one state owner:

```
StateOwner
```

Create many, each owning part of the state:

```
StateOwner[0]  → map[0..999]
StateOwner[1]  → map[1000..1999]
StateOwner[2]  → map[2000..2999]
...
```

Each owner:
- Has its own map
- Has its own channels
- Processes requests independently
- Never touches another owner's data

## How Sharding Works

### Before: Single Owner

```
reads/writes ───► owner ───► state
```

### After: Sharded Owners

```
                 ┌─► owner 0 ──► state 0
reads/writes ────┼─► owner 1 ──► state 1
                 ├─► owner 2 ──► state 2
                 └─► owner 3 ──► state 3
```

### The Routing Rule

```go
ownerIndex := hash(key) % numShards
```

This ensures:
- Same key → same owner (always)
- No cross-owner races
- Per-key linearizability preserved

## Implementation

### Shard Structure

```go
const numShards = 32

type shard struct {
    reads  chan readOp
    writes chan writeOp
}

var shards [numShards]*shard

func init() {
    for i := 0; i < numShards; i++ {
        s := &shard{
            reads:  make(chan readOp),
            writes: make(chan writeOp),
        }
        shards[i] = s
        go s.run()
    }
}
```

### Shard Owner

```go
func (s *shard) run() {
    state := make(map[int]int)
    for {
        select {
        case r := <-s.reads:
            r.resp <- state[r.key]
        case w := <-s.writes:
            state[w.key] = w.val
            w.resp <- true
        }
    }
}
```

Each shard is an independent actor. Same pattern, multiplied.

### Routing Requests

```go
func getShard(key int) *shard {
    return shards[key%numShards]
}

func read(key int) int {
    s := getShard(key)
    resp := make(chan int, 1)
    s.reads <- readOp{key: key, resp: resp}
    return <-resp
}

func write(key, val int) {
    s := getShard(key)
    resp := make(chan bool, 1)
    s.writes <- writeOp{key: key, val: val, resp: resp}
    <-resp
}
```

The caller doesn't know about sharding. The API is unchanged.

## Why This Scales Linearly

With `N` shards:
- ~`N`× throughput (until CPU/IO saturation)
- State access is still serialized **per key**
- Different keys run in **parallel**

### Per-Key Linearizability

Critical property: operations for the **same key**:
- Always go to the same owner
- Remain strictly ordered
- See consistent state

You get parallelism **across keys** while maintaining correctness **per key**.

## Comparison with Mutex Sharding

You could also shard with mutexes:

```go
var locks [numShards]sync.Mutex
var states [numShards]map[int]int

func read(key int) int {
    i := key % numShards
    locks[i].Lock()
    defer locks[i].Unlock()
    return states[i][key]
}
```

### Trade-offs

| Aspect | Channel Sharding | Mutex Sharding |
|--------|-----------------|----------------|
| Memory model | Message passing | Shared memory |
| Ownership | Enforced by design | Enforced by discipline |
| Invariants | Localized in owner | Spread across code |
| Error modes | Blocked channel | Forgotten lock |
| Debugging | Easier (single owner) | Harder (any caller) |

Channel sharding is safer. Mutex sharding is faster (no channel overhead).

Choose based on:
- Complexity of invariants
- Team discipline
- Performance requirements

## When to Shard

### Symptoms You Need Sharding

- Single owner at 100% CPU
- Request latency grows under load
- Profiler shows time in channel recv
- Throughput plateaus despite available cores

### How Many Shards?

Rules of thumb:
- Start with `runtime.GOMAXPROCS(0)` (number of cores)
- Powers of 2 are common (16, 32, 64)
- More shards = more parallelism, but more channels
- Profile to find the right number

### Hash Function

Simple modulo works for integers:

```go
shard := key % numShards
```

For strings, use a real hash:

```go
import "hash/fnv"

func hash(s string) int {
    h := fnv.New32a()
    h.Write([]byte(s))
    return int(h.Sum32())
}

shard := hash(key) % numShards
```

## Real-World Examples

This exact pattern appears in:

| System | Sharding Unit |
|--------|---------------|
| Redis Cluster | Hash slots (16384) |
| Kafka | Partitions per topic |
| Cassandra | Token ranges |
| Akka | Router + Actors |
| Erlang | Process groups |
| Go maps | Internal bucket shards |

You're learning distributed systems primitives, not just Go.

## Adding Context to Shards

Production code needs shutdown:

```go
func (s *shard) run(ctx context.Context) {
    state := make(map[int]int)
    for {
        select {
        case <-ctx.Done():
            return
        case r := <-s.reads:
            select {
            case r.resp <- state[r.key]:
            case <-r.ctx.Done():
            }
        case w := <-s.writes:
            state[w.key] = w.val
            select {
            case w.resp <- true:
            case <-w.ctx.Done():
            }
        }
    }
}
```

Now each shard:
- Respects global shutdown
- Handles abandoned requests
- Won't block forever

## Key Takeaways

1. **Single owner is correct but limited** — throughput caps at one core
2. **Shard by key to scale** — each shard is an independent actor
3. **Same key → same shard** — preserves per-key linearizability
4. **Routing is transparent** — callers don't know about shards
5. **This is how production systems work** — Redis, Kafka, Cassandra all do this

The insight: you don't abandon the actor model to scale—you multiply it.
