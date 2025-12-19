+++
title = "Go Channels: The Request-Response Pattern Demystified"
date = 2025-12-19
description = "Deep dive into how channels actually work - sender/receiver roles, request-response mechanics, and why resp must be a channel"
[taxonomies]
tags = ["go", "concurrency", "channels", "internals"]
+++

This post explains the mechanics that confuse even experienced engineers: why sender/receiver roles flip per channel, how request-response actually works, and why the response field must be a channel.

<!-- more -->

## The Core Confusion

When you first see this pattern:

```go
type readOp struct {
    key  int
    resp chan int
}

// Reader side
reads <- read
<-read.resp

// Owner side
read := <-reads
read.resp <- state[read.key]
```

Several things seem paradoxical:
- Who is the sender? Who is the receiver?
- The reader sends AND receives?
- Where does `read` come from in `read := <-reads`?
- Why can't we just put the value in the struct?

Let's resolve each one precisely.

## Roles Are Per Channel, Not Per Goroutine

This is the key insight that resolves most confusion:

> **A goroutine can be a sender on one channel and a receiver on another simultaneously.**

There is no such thing as a "sender goroutine" globally. Roles are defined per channel operation.

### Channel 1: `reads` (the request channel)

```go
// Reader goroutine: SENDER
reads <- read

// Owner goroutine: RECEIVER
read := <-reads
```

Data flows: `Reader → reads → Owner`

### Channel 2: `read.resp` (the response channel)

```go
// Owner goroutine: SENDER
read.resp <- state[read.key]

// Reader goroutine: RECEIVER
<-read.resp
```

Data flows: `Owner → read.resp → Reader`

Same two goroutines. Two channels. **Opposite directions**.

| Goroutine | Channel | Role |
|-----------|---------|------|
| Reader | `reads` | sender |
| Owner | `reads` | receiver |
| Owner | `read.resp` | sender |
| Reader | `read.resp` | receiver |

## Where Does `read` Come From?

This line confuses people:

```go
case read := <-reads:
```

The variable `read` didn't exist before. So where does it get its value?

**Answer**: It's created and assigned at the moment a value is received.

This is exactly equivalent to:

```go
read := <-reads
```

Just scoped to that `case` block.

### Step-by-step at runtime:

**1. Reader creates a value**

```go
read := readOp{
    key:  3,
    resp: make(chan int),
}
```

Memory now contains:

```
readOp {
  key: 3
  resp: 0xc00001a0c0  // channel pointer
}
```

**2. Reader sends it**

```go
reads <- read
```

This blocks until someone receives.

**3. Owner receives it**

```go
case read := <-reads:
```

At this moment:
- Go runtime matches send with receive
- The value is **copied** into the receiver's variable
- A new variable `read` is created in the owner's scope
- It's initialized with the received value

**4. But the channel is shared**

The struct is copied, but `read.resp` points to the **same channel** in both goroutines.

```
Reader's read.resp: 0xc00001a0c0 ─┐
                                  ├─► Same channel object
Owner's read.resp:  0xc00001a0c0 ─┘
```

Channels are references. The struct copy contains the same channel pointer.

## Why We Need Two Channels

You might think: "Why not reply on `reads`?"

Because `reads` carries **requests from many goroutines**. If you replied on `reads`:
- Any goroutine could receive the response
- Wrong requester gets the answer
- Total chaos

Each request must carry its **own private reply channel**:

```go
resp: make(chan int)
```

This ensures the response goes **only to the requester**.

## Why `resp` Must Be a Channel (Not a Field)

This is the deepest insight.

### What if `resp` were just an `int`?

```go
type readOp struct {
    key  int
    resp int  // result goes here?
}
```

And the owner did:

```go
read.resp = state[read.key]
```

**This doesn't work because:**
- The struct is copied when sent through the channel
- Owner and reader have **different struct instances**
- Mutating `read.resp` in the owner does nothing to the reader's copy
- There's no way for changes to "flow back"

### What if `resp` were `*int`?

```go
type readOp struct {
    key  int
    resp *int
}
```

This **could** work, but:
- Now you have shared memory
- Need synchronization (mutex or atomic)
- Breaks the message-passing model
- Reintroduces all the problems channels solve

### Why `chan int` is the answer

Channels provide:
1. **Memory transfer** — values flow between goroutines
2. **Synchronization** — send/receive form synchronization points
3. **Happens-before** — memory ordering guarantees
4. **Blocking semantics** — reader waits until response arrives

The channel is **the return path**, just like a function return value.

## Mental Model: Channels as Function Calls

Think of this:

```go
reads <- read
value := <-read.resp
```

As equivalent to:

```go
value := readFromState(key)
```

Where:
- `reads <- read` is the function call
- `<-read.resp` is the return value
- The owner goroutine is the "callee"

You've implemented **blocking RPC over channels**.

## The Complete Timeline

```
Reader goroutine              Owner goroutine
     │                              │
     │  create readOp{key:3, resp}  │
     │                              │
     ├────── reads <- read ─────────▶
     │                              │
     │         (blocked)            │ read := <-reads
     │                              │
     │                              │ read.resp <- state[3]
     │                              │
     ◀────── <-read.resp ───────────┤
     │                              │
     │  got value                   │
     ▼                              ▼
```

Two synchronization points:
1. Request handoff (`reads`)
2. Response handoff (`read.resp`)

## Common Mistakes

### Forgetting to create the response channel

```go
read := readOp{key: 3}  // resp is nil!
reads <- read
<-read.resp  // panic: send on nil channel
```

### Unbuffered response without context

```go
resp: make(chan int)  // unbuffered
```

If the reader abandons the request (timeout, disconnect), the owner blocks forever on:

```go
read.resp <- value  // no receiver, blocks forever
```

Fix: Buffer the response channel or use context:

```go
resp: make(chan int, 1)

// OR with context
select {
case read.resp <- value:
case <-read.ctx.Done():
}
```

### Reusing response channels

```go
resp := make(chan int)
for i := 0; i < 10; i++ {
    reads <- readOp{key: i, resp: resp}  // same channel!
    <-resp
}
```

This works for sequential requests, but if you parallelize, responses get mixed up. Each concurrent request needs its own channel.

## The Official Name

This pattern has established names:
- **Actor Model** — one goroutine owns state, others send messages
- **CSP (Communicating Sequential Processes)** — Hoare's formal model
- **Request-Response over Message Passing** — describes the two-channel pattern
- **"Share memory by communicating"** — Go's phrasing

It's the same model used in Erlang, Elixir, Akka, Orleans, and parts of Rust.

## Key Takeaways

1. **Sender/receiver roles are per channel**, not per goroutine
2. **Structs are copied**, but channel references inside them are shared
3. **The response must be a channel** because struct mutations don't propagate
4. **Two channels implement request-response** — one for request, one for reply
5. **This is blocking RPC** implemented with channels

Once you see channels as the communication mechanism between "caller" and "callee" across goroutine boundaries, the pattern becomes obvious.
