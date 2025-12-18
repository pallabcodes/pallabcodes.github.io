+++
title = "Concurrency Patterns in Go"
date = 2024-12-08
description = "Practical patterns for writing concurrent Go code safely and efficiently."
[taxonomies]
tags = ["go", "concurrency", "programming"]
+++

Go's concurrency primitives are simple, but combining them effectively takes practice. Here are battle-tested patterns.

<!-- more -->

## Goroutines and Channels: The Basics

```go
// Don't communicate by sharing memory;
// share memory by communicating.

ch := make(chan int)

go func() {
    ch <- 42  // Send
}()

value := <-ch  // Receive
```

## Pattern 1: Worker Pool

Process work items with bounded concurrency:

```go
func workerPool(numWorkers int, jobs <-chan Job, results chan<- Result) {
    var wg sync.WaitGroup
    
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }
    
    wg.Wait()
    close(results)
}

// Usage
jobs := make(chan Job, 100)
results := make(chan Result, 100)

go workerPool(10, jobs, results)

// Send jobs
for _, job := range allJobs {
    jobs <- job
}
close(jobs)

// Collect results
for result := range results {
    fmt.Println(result)
}
```

## Pattern 2: Fan-Out, Fan-In

Distribute work, then collect results:

```go
func fanOut(input <-chan int, numWorkers int) []<-chan int {
    outputs := make([]<-chan int, numWorkers)
    
    for i := 0; i < numWorkers; i++ {
        outputs[i] = worker(input)
    }
    
    return outputs
}

func fanIn(inputs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    merged := make(chan int)
    
    output := func(ch <-chan int) {
        defer wg.Done()
        for v := range ch {
            merged <- v
        }
    }
    
    wg.Add(len(inputs))
    for _, ch := range inputs {
        go output(ch)
    }
    
    go func() {
        wg.Wait()
        close(merged)
    }()
    
    return merged
}
```

## Pattern 3: Context for Cancellation

Always propagate context for proper cancellation:

```go
func fetchWithTimeout(ctx context.Context, url string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    return io.ReadAll(resp.Body)
}

// Caller can cancel
ctx, cancel := context.WithCancel(context.Background())
go func() {
    time.Sleep(2 * time.Second)
    cancel()  // Cancel if taking too long
}()

data, err := fetchWithTimeout(ctx, "https://api.example.com")
```

## Pattern 4: Semaphore

Limit concurrent access to a resource:

```go
type Semaphore struct {
    ch chan struct{}
}

func NewSemaphore(max int) *Semaphore {
    return &Semaphore{ch: make(chan struct{}, max)}
}

func (s *Semaphore) Acquire() {
    s.ch <- struct{}{}
}

func (s *Semaphore) Release() {
    <-s.ch
}

// Usage
sem := NewSemaphore(10)  // Max 10 concurrent

for _, url := range urls {
    sem.Acquire()
    go func(u string) {
        defer sem.Release()
        fetch(u)
    }(url)
}
```

## Pattern 5: Pipeline

Chain processing stages:

```go
func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

func print(in <-chan int) {
    for n := range in {
        fmt.Println(n)
    }
}

// Pipeline: generate → square → print
print(square(generator(1, 2, 3, 4, 5)))
// Output: 1, 4, 9, 16, 25
```

## Pattern 6: Select for Multiplexing

Handle multiple channels:

```go
func process(ctx context.Context, input <-chan Data) {
    for {
        select {
        case <-ctx.Done():
            return  // Graceful shutdown
        case data := <-input:
            handle(data)
        case <-time.After(30 * time.Second):
            log.Println("No data received, checking health...")
        }
    }
}
```

## Common Mistakes to Avoid

```go
// ❌ Goroutine leak - no way to stop
go func() {
    for {
        doWork()
    }
}()

// ✅ Cancellable goroutine
go func(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            doWork()
        }
    }
}(ctx)

// ❌ Race condition
counter := 0
for i := 0; i < 1000; i++ {
    go func() { counter++ }()
}

// ✅ Use sync/atomic or mutex
var counter int64
for i := 0; i < 1000; i++ {
    go func() { atomic.AddInt64(&counter, 1) }()
}
```

## Key Takeaways

1. **Prefer channels over mutexes** for communication
2. **Always use context** for cancellation
3. **Close channels from sender** side only
4. **Use select** for non-blocking operations
5. **Bound your concurrency** with semaphores or worker pools

Concurrency in Go is powerful but requires discipline. These patterns keep you safe.
