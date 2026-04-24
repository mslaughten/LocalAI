# Context and Timeout Handling in LocalAI

This guide covers how to propagate `context.Context` through request handlers, set per-request deadlines, and handle cancellations gracefully across backends.

---

## Why Context Matters

Long-running inference calls can block goroutines indefinitely. Using `context.Context` lets us:

- Honour client disconnects (the HTTP request context is cancelled when the client hangs up).
- Enforce per-request timeouts configured by the operator.
- Propagate trace/span IDs (see `request-logging-and-tracing.md`).
- Cancel in-flight backend calls when a graceful shutdown is triggered (see `graceful-shutdown.md`).

---

## Timeout Configuration

Add timeout values to your model or server config (see `model-configuration.md`):

```yaml
# config/my-model.yaml
name: my-model
backend: llama-cpp
timeout: 120s          # per-request inference timeout
connect_timeout: 10s   # time allowed to establish the backend gRPC connection
```

In Go, parse these with `time.ParseDuration` and store them on the config struct:

```go
type BackendConfig struct {
    Name           string        `yaml:"name"`
    Backend        string        `yaml:"backend"`
    Timeout        time.Duration `yaml:"timeout"`
    ConnectTimeout time.Duration `yaml:"connect_timeout"`
}
```

---

## Wrapping the Request Context

In your Fiber handler, extract the underlying `context.Context` and attach a deadline before passing it downstream:

```go
// TimeoutFromConfig returns a child context that expires after the model's
// configured timeout, or after the supplied fallback if none is set.
func TimeoutFromConfig(parent context.Context, cfg *BackendConfig, fallback time.Duration) (context.Context, context.CancelFunc) {
    d := cfg.Timeout
    if d <= 0 {
        d = fallback
    }
    return context.WithTimeout(parent, d)
}
```

Then in the handler:

```go
func CompletionHandler(cfg *BackendConfig, backend BackendClient) fiber.Handler {
    return func(c *fiber.Ctx) error {
        // Fiber's context is request-scoped; wrap it with our deadline.
        ctx, cancel := TimeoutFromConfig(c.Context(), cfg, 60*time.Second)
        defer cancel()

        req, err := GetValidatedBody[CompletionRequest](c)
        if err != nil {
            return SendError(c, fiber.StatusBadRequest, err.Error())
        }

        result, err := backend.Infer(ctx, req)
        if err != nil {
            return handleInferError(c, err)
        }
        return c.JSON(result)
    }
}
```

---

## Handling Cancellation Errors

Distinguish between a client disconnect, a timeout, and a real backend error so the correct HTTP status is returned:

```go
func handleInferError(c *fiber.Ctx, err error) error {
    switch {
    case errors.Is(err, context.DeadlineExceeded):
        return SendError(c, fiber.StatusGatewayTimeout,
            "inference timed out – try a shorter prompt or increase the model timeout")
    case errors.Is(err, context.Canceled):
        // Client disconnected; nothing useful to send back.
        return nil
    default:
        return SendError(c, fiber.StatusInternalServerError, err.Error())
    }
}
```

---

## Propagating Context into gRPC Backend Calls

All generated gRPC stubs accept a `context.Context` as their first argument. Pass the request context directly so that deadline and cancellation metadata are forwarded over the wire:

```go
func (c *LlamaCppClient) Infer(ctx context.Context, req *CompletionRequest) (*CompletionResponse, error) {
    pbReq := toProto(req)
    resp, err := c.stub.Predict(ctx, pbReq) // deadline propagated via gRPC metadata
    if err != nil {
        return nil, mapGRPCError(err)
    }
    return fromProto(resp), nil
}
```

Map gRPC status codes back to Go sentinel errors so `handleInferError` above works correctly:

```go
func mapGRPCError(err error) error {
    switch status.Code(err) {
    case codes.DeadlineExceeded:
        return context.DeadlineExceeded
    case codes.Canceled:
        return context.Canceled
    default:
        return err
    }
}
```

---

## Testing

```go
func TestCompletionHandler_Timeout(t *testing.T) {
    cfg := &BackendConfig{Timeout: 1 * time.Millisecond}
    slowBackend := &fakeBackend{
        inferFn: func(ctx context.Context, _ *CompletionRequest) (*CompletionResponse, error) {
            select {
            case <-time.After(500 * time.Millisecond):
                return &CompletionResponse{Text: "done"}, nil
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        },
    }

    app := fiber.New()
    app.Post("/completion", CompletionHandler(cfg, slowBackend))

    req := httptest.NewRequest(http.MethodPost, "/completion",
        strings.NewReader(`{"model":"x","prompt":"hi"}`))
    req.Header.Set("Content-Type", "application/json")

    resp, err := app.Test(req, 2000 /* ms overall test timeout */)
    require.NoError(t, err)
    assert.Equal(t, fiber.StatusGatewayTimeout, resp.StatusCode)
}
```

---

## Checklist

- [ ] Every handler creates a child context with `TimeoutFromConfig`.
- [ ] `defer cancel()` is always called immediately after the context is created.
- [ ] gRPC calls receive the request context, not `context.Background()`.
- [ ] `handleInferError` (or equivalent) differentiates timeout vs. cancellation vs. backend error.
- [ ] Timeout values are surfaced in model YAML config and documented.
- [ ] Unit tests cover the timeout and cancellation paths.
