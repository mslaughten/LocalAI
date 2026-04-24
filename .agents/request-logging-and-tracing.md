# Request Logging and Tracing in LocalAI

This guide covers how to add structured request logging and distributed tracing to LocalAI API endpoints.

## Overview

LocalAI uses a middleware-based approach for request logging and tracing. Each incoming HTTP request is assigned a unique trace ID, logged with relevant metadata, and correlated across backend calls.

## Request Logger Middleware

The `RequestLogger` middleware attaches a trace ID to every request and logs structured fields:

```go
package middleware

import (
	"time"

	"github.com/gofiber/fiber/v2"
	"github.com/google/uuid"
	"github.com/rs/zerolog/log"
)

// RequestLogger logs each incoming request with a unique trace ID,
// method, path, status code, latency, and client IP.
func RequestLogger() fiber.Handler {
	return func(c *fiber.Ctx) error {
		start := time.Now()

		// Generate or propagate a trace ID
		traceID := c.Get("X-Trace-Id")
		if traceID == "" {
			traceID = uuid.New().String()
		}
		c.Locals("traceID", traceID)
		c.Set("X-Trace-Id", traceID)

		err := c.Next()

		latency := time.Since(start)
		statusCode := c.Response().StatusCode()

		logger := log.With().
			Str("trace_id", traceID).
			Str("method", c.Method()).
			Str("path", c.Path()).
			Int("status", statusCode).
			Dur("latency_ms", latency).
			Str("ip", c.IP()).
			Logger()

		switch {
		case statusCode >= 500:
			logger.Error().Msg("server error")
		case statusCode >= 400:
			logger.Warn().Msg("client error")
		default:
			logger.Info().Msg("request completed")
		}

		return err
	}
}
```

## Registering the Middleware

In your route registration function, apply `RequestLogger` before other middleware:

```go
func RegisterMyRoutes(app *fiber.App, ...) {
	app.Use(middleware.RequestLogger())
	app.Use(middleware.AuthMiddleware(...))
	app.Use(middleware.RateLimitMiddleware(...))
	// ... register handlers
}
```

## Accessing the Trace ID in Handlers

Retrieve the trace ID from `c.Locals` to correlate log lines within a handler:

```go
func MyHandler(c *fiber.Ctx) error {
	traceID, _ := c.Locals("traceID").(string)
	log.Info().Str("trace_id", traceID).Msg("handling request")
	// ...
}
```

## Propagating Trace IDs to Backends

When calling a backend (e.g., llama.cpp gRPC), forward the trace ID as metadata:

```go
import "google.golang.org/grpc/metadata"

func callBackend(ctx context.Context, traceID string, req *pb.PredictRequest) (*pb.Reply, error) {
	md := metadata.Pairs("x-trace-id", traceID)
	ctx = metadata.NewOutgoingContext(ctx, md)
	return client.Predict(ctx, req)
}
```

## Log Levels

| Level   | When to use                                      |
|---------|--------------------------------------------------|
| `Debug` | Verbose internal state, model load parameters    |
| `Info`  | Normal request completion, model loaded          |
| `Warn`  | 4xx errors, recoverable issues, slow requests    |
| `Error` | 5xx errors, backend failures, panics             |

## Slow Request Detection

Add a threshold check to flag slow requests automatically:

```go
const slowRequestThreshold = 5 * time.Second

if latency > slowRequestThreshold {
	log.Warn().
		Str("trace_id", traceID).
		Dur("latency_ms", latency).
		Str("path", c.Path()).
		Msg("slow request detected")
}
```

## Environment Configuration

| Variable              | Default | Description                              |
|-----------------------|---------|------------------------------------------|
| `LOG_LEVEL`           | `info`  | Zerolog level: debug, info, warn, error  |
| `LOG_FORMAT`          | `json`  | `json` or `pretty` for development       |
| `TRACE_HEADER`        | `X-Trace-Id` | Header name for trace propagation   |

Set `LOG_FORMAT=pretty` locally for human-readable output during development.

## Integration with Observability

Trace IDs emitted in logs can be correlated with Prometheus metrics (see `observability-and-metrics.md`) by including the `trace_id` label on histogram observations where appropriate.

## Testing

```go
func TestRequestLoggerSetsTraceID(t *testing.T) {
	app := fiber.New()
	app.Use(middleware.RequestLogger())
	app.Get("/ping", func(c *fiber.Ctx) error {
		return c.SendString("pong")
	})

	req := httptest.NewRequest("GET", "/ping", nil)
	resp, err := app.Test(req)
	require.NoError(t, err)
	assert.NotEmpty(t, resp.Header.Get("X-Trace-Id"))
	assert.Equal(t, 200, resp.StatusCode)
}
```
