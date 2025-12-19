+++
title = "Go Closures and Goroutines: The Capture Bug"
date = 2025-12-19
description = "Why loop variables in goroutines break, how closures capture variables not values, and the correct patterns"
[taxonomies]
tags = ["go", "concurrency", "closures", "gotchas"]
+++

Every Go developer hits this bug at least once. Understanding why it happens requires understanding how closures work. This post explains the mechanics precisely.

<!-- more -->

## The Bug

```go
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i)
    }()
}
time.Sleep(time.Second)
```

Expected output:
```
0
1
2
```

Actual output (typically):
```
3
3
3
```

Why?

## Closures Capture Variables, Not Values

This is the fundamental rule:

> **A closure captures a reference to the variable, not a copy of its value.**

In the loop:

```go
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i)  // Captures variable 'i', not value
    }()
}
```

There is **one variable `i`**. All three goroutines capture a reference to that same variable.

By the time the goroutines actually run:
- The loop has finished
- `i` equals 3 (the exit condition)
- All goroutines read `i`, which is 3

## Timeline Visualization

```
Time →

Main goroutine:
  i=0, spawn goroutine (captures &i)
  i=1, spawn goroutine (captures &i)
  i=2, spawn goroutine (captures &i)
  i=3, loop exits
  sleep...

Goroutine 1 runs: reads i → 3
Goroutine 2 runs: reads i → 3
Goroutine 3 runs: reads i → 3
```

The goroutines don't run immediately. By the time they execute, `i` has changed.

## Why This Happens (Language Mechanics)

### What a closure is

A closure is a function that references variables from its enclosing scope:

```go
func outer() func() int {
    x := 10
    return func() int {
        return x  // This is a closure over 'x'
    }
}
```

The returned function "closes over" `x`. It doesn't copy `x`—it keeps a reference to the same memory location.

### Loop variables are one variable

In Go (before 1.22), the loop:

```go
for i := 0; i < 3; i++ {
```

Creates **one variable `i`** that is mutated each iteration. Not three separate variables.

So every closure that captures `i` captures the **same variable**.

## Fix 1: Shadow the Variable

```go
for i := 0; i < 3; i++ {
    i := i  // Shadow with a local copy
    go func() {
        fmt.Println(i)  // Captures the local 'i'
    }()
}
```

Now each iteration:
- Creates a **new local variable** named `i`
- Copies the current value into it
- The closure captures the local, not the loop variable

Each goroutine captures a **different variable** with a **different value**.

## Fix 2: Pass as Function Argument

```go
for i := 0; i < 3; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i)
}
```

Here:
- `i` is passed as an argument at spawn time
- The value is copied into parameter `n`
- The closure uses `n`, which is a per-call copy

This is often cleaner than shadowing.

## Fix 3: Go 1.22+ Changes

Starting with Go 1.22, loop variables are captured per-iteration by default:

```go
// Go 1.22+
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i)  // Now works correctly!
    }()
}
```

The compiler creates a new variable for each iteration. The capture bug is eliminated.

**But**: You should still understand the old behavior because:
- Legacy code exists
- The mental model matters for other closure scenarios
- Not everyone runs Go 1.22+

## Why the Original Example Didn't Break

In the actor pattern code:

```go
for r := 0; r < 100; r++ {
    go func() {
        for {
            read := readOp{
                key:  rand.Intn(5),
                resp: make(chan int),
            }
            reads <- read
            <-read.resp
            atomic.AddUint64(&readOps, 1)
            time.Sleep(time.Millisecond)
        }
    }()
}
```

**Why doesn't this break?**

Because `r` is **never used inside the goroutine**. The closure doesn't capture `r`. No bug.

If you did:

```go
for r := 0; r < 100; r++ {
    go func() {
        fmt.Println("Reader", r)  // BUG: captures r
    }()
}
```

Then you'd see the bug. All would print "Reader 100".

## Related Pattern: Range Captures

The same bug occurs with `range`:

```go
// Bug
items := []string{"a", "b", "c"}
for _, item := range items {
    go func() {
        fmt.Println(item)
    }()
}
// Prints "c" three times
```

Fix:

```go
for _, item := range items {
    item := item  // Shadow
    go func() {
        fmt.Println(item)
    }()
}
```

Or:

```go
for _, item := range items {
    go func(s string) {
        fmt.Println(s)
    }(item)
}
```

## The `var` vs `:=` Question

Why do we sometimes use `var` instead of `:=`?

```go
var readOps uint64
var writeOps uint64
```

Two reasons:

### 1. Addressability

Atomics require a pointer:

```go
atomic.AddUint64(&readOps, 1)
```

You need a stable address. `var` ensures this.

### 2. Initialization clarity

`var x uint64` clearly means "zero-initialized counter."

`:=` would require:

```go
readOps := uint64(0)
```

More noise, same result.

## When to Use Which

| Scenario | Recommendation |
|----------|---------------|
| Loop variable in goroutine | Shadow it or pass as arg |
| Counter for atomics | Use `var` |
| Complex state | Be explicit about scope |
| Go 1.22+ new code | Capture works, but consider clarity |

## Detecting the Bug

### Go vet

```bash
go vet ./...
```

Warns about loop variable capture:

```
loop variable i captured by func literal
```

### Race detector

```bash
go run -race ./...
```

May detect if the captured variable is also written.

### Code review

Look for pattern:
```go
for x := ... {
    go func() {
        ... x ...  // Captured!
    }()
}
```

## Key Takeaways

1. **Closures capture variables, not values** — the reference, not a copy
2. **Loop variables are one variable** (before Go 1.22) — mutated each iteration
3. **Shadow or pass as argument** — to get a per-iteration copy
4. **Go 1.22 fixes this** — but understand the mechanics anyway
5. **If you don't use the variable, no bug** — capture only happens on reference

This is one of Go's most common gotchas. Now you understand why it happens at the language level.
