+++
title = "The Mechanics of Go Execution: A Timeline View"
date = 2025-12-19
description = "Visualizing exactly how the main loop and worker goroutines interleave (and don't interleave) in time."
[taxonomies]
tags = ["go", "concurrency", "mental-model", "go-routines"]
+++

One of the hardest mental shifts in Go is understanding that `go func()` arranges for code to run *later*, not *now*. This post breaks down the execution timeline of a concurrent loop to kill the "sequential intuition" once and for all.

<!-- more -->

## The Confusion

Here is the code that confuses almost everyone at first glance:

```go
// Launch 50 workers
for i := 0; i < 50; i++ {
    wg.Add(1)
    go func() {
        // ... heavy work ...
        wg.Done()
    }()
}
wg.Wait()
```

The common (incorrect) intuition is that the loop waits for the goroutine to start, or that `wg.Done()` somehow happens "inside" the loop iteration.

Let's visualize reality.

## Two Separate Timelines

We must separate the **Main Goroutine** from the **Worker Goroutines**.

### Timeline: T0 (Start)

*   **Main**: Enters `main()`.
*   **Workers**: None exist.

### Timeline: T1 (Loop Start)

*   **Main**: `i = 0`.
*   **Main**: `wg.Add(1)`. Counter is now 1.
*   **Main**: `go func()`.
    *   **Action**: The runtime creates a goroutine object.
    *   **Status**: Queued. It has **NOT** started executing user code yet.
*   **Main**: Loop continues immediately.

### Timeline: T2 (Next Iteration)

*   **Main**: `i = 1`.
*   **Main**: `wg.Add(1)`. Counter is now 2.
*   **Main**: `go func()`.
    *   **Action**: Another goroutine queued.
*   **Main**: Loop continues.

*   **Worker #1**: *Maybe* starts running now? *Maybe* later? We don't know. The scheduler decides.

### Timeline: T3 (Loop Finishes)

*   **Main**: `i` reaches 50. Loop exits.
*   **Main**: Reaches `wg.Wait()`.
*   **Main**: **BLOCKS**.

At this exact moment:
*   50 Goroutines have been *created*.
*   Any number of them (0 to 50) might have *started* running.
*   Any number of them might have *finished*.
*   `wg.Wait()` effectively says: "Stop everything here until the counter is 0."

## The Worker Timeline

Let's zoom into a single worker. Its lifecycle is entirely independent of `main` once spawned.

```go
go func() {
    for c := 0; c < 1000; c++ {
        ops.Add(1)
    }
    wg.Done()
}()
```

1.  **Spawn**: Created by main.
2.  **Schedule**: Runtime picks it up (could be T1, T5, or T100).
3.  **Execute**: Runs its `for` loop sequentially. 1, 2, ... 1000.
4.  **Finish**: Calls `wg.Done()`.

**Critical:** `wg.Done()` cannot happen until the inner loop finishes. There is no magic jumping ahead.

## Visualizing the Interleaving

```text
Time →

Main:  [Add] [Spawn] [Add] [Spawn] ... [Add] [Spawn] -----------------> [Wait.......] [Resume]
          ↓       ↓       ↓       ↓             ↓                             ^
Worker 1:         [Start ...... Loop ...... Done]                             |
Worker 2:                 [Start ...... Loop ...... Done]                     |
Worker 50:                                      [Start ...... Loop ...... Done]
```

Notice:
1.  **Main** is a fast, tight loop creating work.
2.  **Workers** start whenever the scheduler has capacity.
3.  **Wait** is the synchronization barrier where timelines converge.

## The "Go" Mental Model

> **`go f()` means "Schedule this for later."**
> **`wg.Wait()` means "Wait for the schedule to clear."**

### Why this matters

If you assume `go` runs immediately, you might write code that depends on the worker doing something *before* the next loop iteration. That is a **race condition**.

**Wrong:**
```go
for i := 0; i < 50; i++ {
    go func() {
        results[i] = true // Race! "i" changes as we run!
    }()
}
```

**Right (Shadowing):**
```go
for i := 0; i < 50; i++ {
    i := i // Copy value for this specific timeline
    go func() {
        results[i] = true
    }()
}
```

(Note: Go 1.22+ fixes the loop variable issue, but the *concurrency* principle remains: you cannot rely on execution order without explicit synchronization.)

## Summary

1.  **Creation ≠ Execution**: The `for` loop creates workers; it doesn't run them.
2.  **Independent Timelines**: Once spawned, a goroutine is on its own time.
3.  **Explicit Barriers**: `wg.Wait()` (or a channel receive) is the *only* guarantee that work is done.
