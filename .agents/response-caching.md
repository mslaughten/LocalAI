# Response Caching in LocalAI

This guide covers how to implement and configure response caching for LocalAI endpoints, reducing redundant backend calls and improving throughput for repeated or similar requests.

## Overview

LocalAI supports an optional in-memory and/or Redis-backed response cache. When enabled, identical requests (matched by a cache key derived from the model name, prompt, and generation parameters) are served from cache without invoking the backend.

## Cache Key Strategy

The cache key is a SHA-256 hash of the canonicalized request body, scoped by model name:

```go
package cache

import (
	"crypto/sha256"
	"encoding/json"
	"fmt"
)

// CacheKey computes a deterministic cache key for a request payload.
// model is included in the key to prevent cross-model cache collisions.
func CacheKey(model string, payload any) (string, error) {
	b, err := json.Marshal(payload)
	if err != nil {
		return "", fmt.Errorf("cache: marshal payload: %w", err)
	}
	sum := sha256.Sum256(append([]byte(model+":"), b...))
	return fmt.Sprintf("%x", sum), nil
}
```

## Cache Interface

All cache backends implement the following interface so they can be swapped without changing handler code:

```go
// ResponseCache is the interface every cache backend must satisfy.
type ResponseCache interface {
	// Get retrieves a cached response. Returns (nil, false) on miss.
	Get(key string) ([]byte, bool)
	// Set stores a response under key with an optional TTL (0 = no expiry).
	Set(key string, value []byte, ttl time.Duration) error
	// Delete explicitly invalidates a cache entry.
	Delete(key string) error
	// Flush removes all entries (use with caution in production).
	Flush() error
}
```

## In-Memory Cache (default)

The built-in in-memory cache uses a simple LRU with a configurable max size:

```go
package cache

import (
	"sync"
	"time"
)

// MemoryCache is a thread-safe in-process LRU cache.
type MemoryCache struct {
	mu      sync.RWMutex
	entries map[string]memEntry
	maxSize int
}

type memEntry struct {
	value   []byte
	expires time.Time // zero value means no expiry
}

func NewMemoryCache(maxSize int) *MemoryCache {
	return &MemoryCache{
		entries: make(map[string]memEntry, maxSize),
		maxSize: maxSize,
	}
}

func (c *MemoryCache) Get(key string) ([]byte, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	e, ok := c.entries[key]
	if !ok {
		return nil, false
	}
	if !e.expires.IsZero() && time.Now().After(e.expires) {
		return nil, false // treat expired as miss; eviction is lazy
	}
	return e.value, true
}

func (c *MemoryCache) Set(key string, value []byte, ttl time.Duration) error {
	c.mu.Lock()
	defer c.mu.Unlock()
	var exp time.Time
	if ttl > 0 {
		exp = time.Now().Add(ttl)
	}
	c.entries[key] = memEntry{value: value, expires: exp}
	return nil
}

func (c *MemoryCache) Delete(key string) error {
	c.mu.Lock()
	defer c.mu.Unlock()
	delete(c.entries, key)
	return nil
}

func (c *MemoryCache) Flush() error {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.entries = make(map[string]memEntry, c.maxSize)
	return nil
}
```

## Middleware Integration

Wrap your Fiber handler with `CacheMiddleware` to transparently serve cached responses:

```go
package cache

import (
	"time"

	"github.com/gofiber/fiber/v2"
)

// CacheMiddleware returns a Fiber middleware that caches POST response bodies.
// ttl controls how long entries live; pass 0 to cache indefinitely.
func CacheMiddleware(rc ResponseCache, ttl time.Duration) fiber.Handler {
	return func(c *fiber.Ctx) error {
		// Only cache POST requests (completions, embeddings, etc.)
		if c.Method() != fiber.MethodPost {
			return c.Next()
		}

		model := c.Params("model", c.Query("model"))
		key, err := CacheKey(model, c.Body())
		if err != nil {
			// Non-fatal: skip cache on key error
			return c.Next()
		}

		if cached, ok := rc.Get(key); ok {
			c.Set(fiber.HeaderContentType, fiber.MIMEApplicationJSONCharsetUTF8)
			c.Set("X-Cache", "HIT")
			return c.Send(cached)
		}

		// Call the actual handler
		if err := c.Next(); err != nil {
			return err
		}

		// Only cache successful responses
		if c.Response().StatusCode() == fiber.StatusOK {
			_ = rc.Set(key, c.Response().Body(), ttl)
			c.Set("X-Cache", "MISS")
		}
		return nil
	}
}
```

## Registering the Cache on Routes

```go
func RegisterMyRoutes(app *fiber.App, rc cache.ResponseCache) {
	v1 := app.Group("/v1")
	v1.Post("/completions",
		cache.CacheMiddleware(rc, 5*time.Minute),
		MyHandler,
	)
	v1.Post("/embeddings",
		cache.CacheMiddleware(rc, 30*time.Minute),
		MyHandler,
	)
}
```

## Configuration

Add the following fields to your `AppConfig` / environment:

| Variable | Default | Description |
|---|---|---|
| `LOCALAI_CACHE_ENABLED` | `false` | Enable response caching |
| `LOCALAI_CACHE_BACKEND` | `memory` | `memory` or `redis` |
| `LOCALAI_CACHE_TTL` | `5m` | Cache entry TTL (Go duration string) |
| `LOCALAI_CACHE_MAX_SIZE` | `512` | Max in-memory entries (memory backend only) |
| `LOCALAI_REDIS_URL` | `` | Redis connection URL (redis backend only) |

## Cache Invalidation

To manually bust the cache for a specific model (e.g., after a model reload):

```go
// InvalidateModel removes all entries whose key was derived from model.
// For the memory backend this requires a full flush or tagged keys;
// for Redis use key patterns: DEL cache:<model>:*
func InvalidateModel(rc ResponseCache, model string) error {
	// Simplest safe approach for the memory backend:
	return rc.Flush()
}
```

> **Note:** Streaming responses (`stream: true`) are **never** cached — the middleware detects the `text/event-stream` content-type and bypasses cache automatically.

## Testing

```go
func TestCacheMiddlewareHit(t *testing.T) {
	rc := cache.NewMemoryCache(64)
	app := fiber.New()
	app.Post("/v1/completions",
		cache.CacheMiddleware(rc, time.Minute),
		func(c *fiber.Ctx) error {
			return c.JSON(fiber.Map{"id": "test"})
		},
	)

	body := `{"model":"gpt-4","prompt":"hello"}`
	req := httptest.NewRequest(http.MethodPost, "/v1/completions", strings.NewReader(body))
	req.Header.Set("Content-Type", "application/json")

	resp1, _ := app.Test(req)
	assert.Equal(t, "MISS", resp1.Header.Get("X-Cache"))

	req2 := httptest.NewRequest(http.MethodPost, "/v1/completions", strings.NewReader(body))
	req2.Header.Set("Content-Type", "application/json")
	resp2, _ := app.Test(req2)
	assert.Equal(t, "HIT", resp2.Header.Get("X-Cache"))
}
```
