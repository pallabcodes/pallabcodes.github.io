+++
title = "Handling OS Signals in Go"
date = 2025-12-18
description = "A practical guide to graceful shutdown and signal handling in Go applications"
[taxonomies]
tags = ["go", "systems", "concurrency"]
+++

Every long-running Go program—whether it's an HTTP server, a worker process, or a CLI tool—needs to handle shutdown gracefully. You don't want your database connections left hanging or your in-flight requests aborted mid-write when someone hits Ctrl+C.

Go makes this surprisingly elegant by treating OS signals as just another event source you can receive on a channel. Let's dig into how it actually works.

<!-- more -->

## The Basic Pattern

Here's the minimal version you'll see everywhere:

```go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "syscall"
)

func main() {
    sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)

    done := make(chan bool, 1)

    go func() {
        sig := <-sigs
        fmt.Println()
        fmt.Println("received:", sig)
        done <- true
    }()

    fmt.Println("awaiting signal")
    <-done
    fmt.Println("exiting")
}
```

Run this, press Ctrl+C, and you'll see:

```
awaiting signal
^C
received: interrupt
exiting
```

Let's break down why each piece exists.

## Why a Buffered Channel?

```go
sigs := make(chan os.Signal, 1)
```

The buffer of 1 matters more than you'd think. Signals can arrive at any time—even before your goroutine is ready to receive. An unbuffered channel would cause the signal to be dropped if no one's listening at that exact moment.

I've debugged production issues where someone "optimized" this to an unbuffered channel, then wondered why their process occasionally ignored SIGTERM. Don't be that person.

## Registering for Signals

```go
signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
```

This tells Go's runtime: "When the OS sends these signals, deliver them to this channel instead of using the default behavior."

The signals you'll care about most:

- **SIGINT** — What you get when you press Ctrl+C
- **SIGTERM** — The polite "please shut down" signal (what Kubernetes sends first, then waits `terminationGracePeriodSeconds` before SIGKILL)
- **SIGKILL** — Instant death. You can't catch this one—don't try.
- **SIGHUP** — Traditionally "reload config" in Unix daemons
- **SIGQUIT** — Ctrl+\ in terminal. Go's runtime prints all goroutine stacks—invaluable for debugging hangs in production

A common mistake: registering for SIGKILL. It won't work. The OS terminates your process before Go even sees it.

## The Coordination Dance

The `done` channel is just for internal coordination:

```go
done := make(chan bool, 1)
```

The signal-handling goroutine receives the signal, does whatever cleanup it needs, then signals `done`. The main goroutine blocks on `done` until shutdown is triggered.

Why a separate goroutine? Because `<-sigs` blocks. If you put that in `main()`, you couldn't do anything else—no HTTP server, no background workers, nothing.

## A More Realistic Example

In practice, you're usually dealing with something like an HTTP server:

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    server := &http.Server{Addr: ":8080"}

    // Start server in background
    go func() {
        log.Println("listening on :8080")
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    // Wait for shutdown signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("shutting down...")

    // Give in-flight requests 30 seconds to complete
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Fatalf("forced shutdown: %v", err)
    }

    log.Println("server stopped")
}
```

The key insight: `server.Shutdown()` is graceful—it stops accepting new connections while letting existing requests finish. The context timeout prevents a hung request from blocking shutdown forever.

## The Modern Way: signal.NotifyContext

Go 1.16 introduced `signal.NotifyContext`, which is what you'll see in most new code:

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    server := &http.Server{Addr: ":8080"}

    go func() {
        <-ctx.Done()
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        server.Shutdown(shutdownCtx)
    }()

    log.Println("listening on :8080")
    if err := server.ListenAndServe(); err != http.ErrServerClosed {
        log.Fatalf("server error: %v", err)
    }
}
```

This integrates directly with Go's context system, so you can pass `ctx` to database calls, HTTP clients, worker pools—anything that respects context cancellation.

## Handling Multiple Signals Differently

Sometimes you want different behavior for different signals:

```go
sigs := make(chan os.Signal, 1)
signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM, syscall.SIGHUP)

for {
    switch <-sigs {
    case syscall.SIGHUP:
        log.Println("reloading config...")
        reloadConfig()
    case syscall.SIGINT, syscall.SIGTERM:
        log.Println("shutting down...")
        return
    }
}
```

This is a common pattern for daemons that support config reload without restart.

## Double Ctrl+C for Force Quit

A nice UX touch: allow a second Ctrl+C to force immediate termination:

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    // ... start your server ...

    <-ctx.Done()
    stop() // Stop catching signals—next one will kill us

    log.Println("shutting down gracefully, press Ctrl+C again to force")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    server.Shutdown(shutdownCtx)
}
```

After the first signal, `stop()` re-enables default signal handling. A second Ctrl+C terminates immediately.

## Common Gotchas

**Forgetting to call `signal.Stop`**: If you're done handling signals (say, in a test), call `signal.Stop(sigs)` to stop delivery. Otherwise signals keep coming.

**Using the wrong signal in containers**: Docker sends SIGTERM by default, but your app might only handle SIGINT. Always handle both.

**Not giving cleanup enough time**: That 30-second timeout isn't arbitrary. If your handlers might take a while, size it appropriately—but not so long that deploys take forever.

**Testing signal handling**: You can send signals programmatically with `syscall.Kill(syscall.Getpid(), syscall.SIGINT)`, which is useful for integration tests.

## The Mental Model

```
OS → signal → Go runtime → channel → your code → graceful shutdown
```

Go treats the OS as just another event source. Signals become channel values. You handle them with the same primitives you use for everything else in Go.

This is one of those tiny design decisions that makes Go feel cohesive. No special signal handler syntax, no callback registration—just channels all the way down.