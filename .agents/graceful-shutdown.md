# Graceful Shutdown in LocalAI

This guide covers how to implement graceful shutdown handling in LocalAI backends and HTTP servers, ensuring in-flight requests complete and resources are released cleanly.

## Overview

Graceful shutdown ensures:
- In-flight requests finish before the server stops
- Backend model resources (GPU memory, file handles) are released
- Background goroutines (cache eviction, metrics collection) are stopped
- Clients receive proper error responses for requests that cannot complete

## Core Shutdown Pattern

```go
package core

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

// ShutdownTimeout is the maximum time allowed for graceful shutdown.
const ShutdownTimeout = 30 * time.Second

// RunWithGracefulShutdown starts the HTTP server and blocks until a
// termination signal (SIGINT or SIGTERM) is received, then shuts down
// cleanly within ShutdownTimeout.
func RunWithGracefulShutdown(srv *http.Server, onShutdown func()) error {
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

    errCh := make(chan error, 1)
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            errCh <- err
        }
    }()

    select {
    case err := <-errCh:
        return err
    case sig := <-quit:
        slog.Info("received signal, shutting down", "signal", sig)
    }

    ctx, cancel := context.WithTimeout(context.Background(), ShutdownTimeout)
    defer cancel()

    // Run any cleanup hooks (flush caches, unload models, etc.)
    if onShutdown != nil {
        onShutdown()
    }

    if err := srv.Shutdown(ctx); err != nil {
        return fmt.Errorf("server forced to shutdown: %w", err)
    }

    slog.Info("server exited cleanly")
    return nil
}
```

## Registering Shutdown Hooks

Use a `ShutdownManager` to collect cleanup functions from multiple subsystems:

```go
type ShutdownManager struct {
    mu    sync.Mutex
    hooks []func(ctx context.Context) error
}

func (m *ShutdownManager) Register(fn func(ctx context.Context) error) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.hooks = append(m.hooks, fn)
}

// RunAll executes all registered hooks concurrently and waits for them
// to finish or the context to expire.
func (m *ShutdownManager) RunAll(ctx context.Context) {
    m.mu.Lock()
    hooks := append([]func(ctx context.Context) error(nil), m.hooks...)
    m.mu.Unlock()

    var wg sync.WaitGroup
    for _, h := range hooks {
        wg.Add(1)
        go func(fn func(ctx context.Context) error) {
            defer wg.Done()
            if err := fn(ctx); err != nil {
                slog.Warn("shutdown hook error", "err", err)
            }
        }(h)
    }

    done := make(chan struct{})
    go func() { wg.Wait(); close(done) }()

    select {
    case <-done:
        slog.Info("all shutdown hooks completed")
    case <-ctx.Done():
        slog.Warn("shutdown timed out, some hooks may not have finished")
    }
}
```

## Backend Unload Hook

Backends should register an unload hook so GPU/CPU memory is freed:

```go
// In your backend initialisation:
shutdownMgr.Register(func(ctx context.Context) error {
    slog.Info("unloading model", "model", modelName)
    return backend.Unload(ctx)
})
```

## Cache Flush Hook

If using the response cache (see `response-caching.md`), register a flush:

```go
shutdownMgr.Register(func(ctx context.Context) error {
    cache.Flush()
    slog.Info("response cache flushed")
    return nil
})
```

## Wiring It All Together

```go
func main() {
    mgr := &ShutdownManager{}

    // … initialise backends, register hooks …

    srv := &http.Server{
        Addr:    ":8080",
        Handler: router,
    }

    if err := RunWithGracefulShutdown(srv, func() {
        ctx, cancel := context.WithTimeout(context.Background(), ShutdownTimeout)
        defer cancel()
        mgr.RunAll(ctx)
    }); err != nil {
        slog.Error("shutdown error", "err", err)
        os.Exit(1)
    }
}
```

## Testing

```go
func TestGracefulShutdown_HooksRun(t *testing.T) {
    called := false
    mgr := &ShutdownManager{}
    mgr.Register(func(ctx context.Context) error {
        called = true
        return nil
    })

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    mgr.RunAll(ctx)

    if !called {
        t.Fatal("expected shutdown hook to be called")
    }
}
```

## Tips

- Keep `ShutdownTimeout` generous enough for large models to unload (GPU teardown can take several seconds).
- Log the signal received so operators know whether the shutdown was intentional.
- Avoid calling `os.Exit` directly in handlers; let the shutdown path handle it so hooks always run.
- In Kubernetes, set `terminationGracePeriodSeconds` to at least `ShutdownTimeout + 5s` to avoid SIGKILL races.
