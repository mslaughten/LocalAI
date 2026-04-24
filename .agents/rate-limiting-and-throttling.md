# Rate Limiting and Throttling in LocalAI

This guide covers how to implement and configure rate limiting and throttling for LocalAI API endpoints.

## Overview

LocalAI supports per-API-key and per-IP rate limiting to prevent abuse and ensure fair usage across clients. Rate limiting is implemented as middleware that integrates with the existing authentication and routing layers.

## Configuration

Rate limiting can be configured via environment variables or the LocalAI config file:

```yaml
# config.yaml
rate_limiting:
  enabled: true
  # Requests per minute per API key (0 = unlimited)
  requests_per_minute: 60
  # Requests per minute per IP when no API key is provided
  anonymous_requests_per_minute: 10
  # Burst size (allows short spikes above the rate limit)
  burst_size: 10
  # Backend to use for rate limit state: memory, redis
  backend: memory
  redis_url: "redis://localhost:6379"
```

Environment variable equivalents:

```bash
LOCALAI_RATE_LIMIT_ENABLED=true
LOCALAI_RATE_LIMIT_RPM=60
LOCALAI_RATE_LIMIT_ANON_RPM=10
LOCALAI_RATE_LIMIT_BURST=10
LOCALAI_RATE_LIMIT_BACKEND=memory
LOCALAI_REDIS_URL=redis://localhost:6379
```

## Implementation

### Middleware Registration

Rate limiting middleware is registered in the router setup, after authentication middleware:

```go
// In your router setup (see api-endpoints-and-auth.md)
func RegisterMyRoutes(app *fiber.App, cl *config.BackendConfigLoader, ml *model.ModelLoader, appConfig *config.ApplicationConfig) {
    // Auth middleware runs first
    app.Use(AuthMiddleware(appConfig))
    // Rate limiting runs after auth so we can key on the resolved API key
    app.Use(RateLimitMiddleware(appConfig))

    app.Get("/v1/models", ListModelsHandler(cl, appConfig))
    // ... other routes
}
```

### RateLimitMiddleware

```go
func RateLimitMiddleware(appConfig *config.ApplicationConfig) fiber.Handler {
    limiter := newRateLimiter(appConfig)
    return func(c *fiber.Ctx) error {
        key := resolveRateLimitKey(c)
        if !limiter.Allow(key) {
            return c.Status(fiber.StatusTooManyRequests).JSON(fiber.Map{
                "error": fiber.Map{
                    "message": "rate limit exceeded",
                    "type":    "rate_limit_error",
                    "code":    429,
                },
            })
        }
        // Expose rate limit headers
        info := limiter.Info(key)
        c.Set("X-RateLimit-Limit", strconv.Itoa(info.Limit))
        c.Set("X-RateLimit-Remaining", strconv.Itoa(info.Remaining))
        c.Set("X-RateLimit-Reset", strconv.FormatInt(info.ResetAt.Unix(), 10))
        return c.Next()
    }
}
```

### Resolving the Rate Limit Key

The key used for rate limiting is determined in priority order:

1. API key (from `Authorization: Bearer <key>` header) — preferred for authenticated clients
2. `X-Forwarded-For` header — for clients behind proxies
3. Remote IP address — fallback

```go
func resolveRateLimitKey(c *fiber.Ctx) string {
    if apiKey := extractBearerToken(c); apiKey != "" {
        return "key:" + apiKey
    }
    if xff := c.Get("X-Forwarded-For"); xff != "" {
        // Use only the first IP in the chain
        return "ip:" + strings.SplitN(xff, ",", 2)[0]
    }
    return "ip:" + c.IP()
}
```

## Response Headers

When rate limiting is active, LocalAI adds standard rate limit headers to every response:

| Header | Description |
|---|---|
| `X-RateLimit-Limit` | Maximum requests allowed per window |
| `X-RateLimit-Remaining` | Requests remaining in the current window |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |
| `Retry-After` | Seconds to wait before retrying (only on 429 responses) |

## Per-Model Throttling

In addition to global rate limiting, you can configure per-model concurrency limits to prevent a single heavy model from monopolizing resources:

```yaml
# In your model config (see model-configuration.md)
name: my-large-model
backend: llama-cpp
parameters:
  model: models/llama-3-70b.gguf
throttling:
  max_concurrent_requests: 2
  queue_timeout_seconds: 30
```

Requests that exceed `max_concurrent_requests` are queued. If the queue wait exceeds `queue_timeout_seconds`, a `503 Service Unavailable` is returned.

## Testing Rate Limiting

```bash
# Send 70 requests rapidly and observe 429s after the limit
for i in $(seq 1 70); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -H "Authorization: Bearer $LOCALAI_API_KEY" \
    http://localhost:8080/v1/models
done
```

## Disabling Rate Limiting in Development

Set `LOCALAI_RATE_LIMIT_ENABLED=false` or `rate_limiting.enabled: false` to disable all rate limiting. This is the default when no configuration is provided.

## Related

- [API Endpoints and Auth](api-endpoints-and-auth.md)
- [Security and API Keys](security-and-api-keys.md)
- [Observability and Metrics](observability-and-metrics.md)
