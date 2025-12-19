+++
title = "Go Worker Pools: Channels as Synchronization"
date = 2025-12-19
description = "How channels replace WaitGroups in worker pools, and why buffered channels outperform unbuffered ones."
[taxonomies]
tags = ["go", "concurrency", "worker-pool", "channels"]
+++

A common confusion when learning Go: "If there's no `WaitGroup`, how does this program not exit early?" The answer reveals one of Go's most elegant design choices: **channels can synchronize *and* transport data simultaneously**.

<!-- more -->

## The Worker Pool Pattern

```go
func worker(id int, jobs <-chan int, results chan<- int) {
	for j := range jobs {
		fmt.Println("worker", id, "started  job", j)
		time.Sleep(time.Second)
		fmt.Println("worker", id, "finished job", j)
		results <- j * 2
	}
}

func main() {
	const numJobs = 5
	jobs := make(chan int, numJobs)
	results := make(chan int, numJobs)

	for w := 1; w <= 3; w++ {
		go worker(w, jobs, results)
	}

	for j := 1; j <= numJobs; j++ {
		jobs <- j
	}
	close(jobs)

	for a := 1; a <= numJobs; a++ {
		<-results
	}
}
```

## The Key Insight

**Receiving from a channel blocks the goroutine until a value is available.**

This single line is the entire synchronization mechanism:

```go
for a := 1; a <= numJobs; a++ {
	<-results
}
```

Main will block here until all 5 results have been received. No `WaitGroup` needed.

## Timeline Visualization

```
Time →

Main:
  [Create channels]
  [Spawn 3 workers] (all block waiting for jobs)
  [Send 5 jobs]
  [Close jobs channel]
  [Wait for 5 results] ← BLOCKS until all done
  [Exit]

Worker 1:  [Wait] [Process job 1] [Send result] [Process job 4] [Send result] [Exit]
Worker 2:  [Wait] [Process job 2] [Send result] [Exit]
Worker 3:  [Wait] [Process job 3] [Send result] [Process job 5] [Send result] [Exit]
```

## Why No WaitGroup?

| Coordination Need   | How It's Handled                |
|---------------------|---------------------------------|
| Workers finish work | `range jobs` exits on close     |
| Main waits          | `<-results` blocks              |
| Completion count    | Number of results expected      |

The **data flow encodes the lifecycle**. Each result received is proof of one job completed.

## Buffered vs. Unbuffered: The Mailbox Analogy

### Unbuffered Channel (Hand-to-Hand)

```go
ch := make(chan int) // capacity = 0
```

**Rule:** Send and receive must meet simultaneously.

```
Worker:  "I have a result!" → [BLOCKS until main is ready]
Main:    "I'm ready!"        → [Both unblock together]
```

**Consequence:**
*   Tight coupling.
*   Worker must wait for main.
*   Slower in practice.

### Buffered Channel (Drop Box)

```go
results := make(chan int, 5) // capacity = 5
```

**Rule:** Send succeeds if there's free space.

```
Worker:  "I have a result!" → [Drop in box] → [Continue immediately]
Main (later): "Give me results" → [Take from box]
```

**Consequence:**
*   Decoupled.
*   Workers don't wait.
*   Higher throughput.

## Why Buffered Is Faster

### Unbuffered Timeline

```
Worker finishes → waits for main
Main receives   → wakes worker
(repeated handshake for each result)
```

Like calling someone for every single delivery.

### Buffered Timeline

```
Worker finishes → drops package → leaves
Main picks up packages later
```

Like using a mailbox.

## Common Mistakes

### ❌ Forgetting to receive all results

```go
for a := 1; a <= numJobs-1; a++ { // Off by one!
	<-results
}
```

**Result:** One worker blocks forever. Goroutine leak.

### ❌ Forgetting to close the jobs channel

```go
// close(jobs) ← MISSING
```

**Result:** Workers block forever in `range jobs`. Leak.

## When to Use Channels vs. WaitGroup

| Use Case         | Tool        |
|------------------|-------------|
| Only waiting     | `WaitGroup` |
| Data + waiting   | Channels    |
| Pipelines        | Channels    |
| Fan-out/fan-in   | Channels    |
| Phases           | `WaitGroup` |
| Cancellation     | `context`   |

## Summary

1.  **Channels synchronize implicitly** via blocking semantics.
2.  **Buffered channels decouple** senders and receivers, improving throughput.
3.  **Worker pools leverage channels** for both data flow and lifecycle management.
4.  **The number of results received** is the completion signal—no `WaitGroup` needed.
