# Error Handling in LocalAI

This guide covers patterns and best practices for error handling across LocalAI's HTTP API layer, backends, and middleware.

## Overview

LocalAI uses structured error responses to ensure clients receive consistent, actionable error messages. All errors returned from API endpoints should follow the OpenAI-compatible error format where applicable.

## Standard Error Response Format

```go
// APIError represents a structured error response compatible with OpenAI's API format.
type APIError struct {
    Error *APIErrorBody `json:"error"`
}

type APIErrorBody struct {
    Message string `json:"message"`
    Type    string `json:"type"`
    Code    any    `json:"code,omitempty"`
}
```

Example JSON response:
```json
{
  "error": {
    "message": "model 'gpt-4' not found",
    "type": "invalid_request_error",
    "code": 404
  }
}
```

## Error Types

| Type | Usage |
|------|-------|
| `invalid_request_error` | Malformed input, missing fields, validation failures |
| `authentication_error` | Missing or invalid API key |
| `not_found_error` | Model or resource not found |
| `server_error` | Internal backend or inference errors |
| `rate_limit_error` | Too many requests |

## Sending Errors from Handlers

Use the `SendError` helper to write a consistent error response:

```go
func SendError(c *fiber.Ctx, status int, errType, message string) error {
    return c.Status(status).JSON(APIError{
        Error: &APIErrorBody{
            Message: message,
            Type:    errType,
            Code:    status,
        },
    })
}
```

Example usage in a handler:

```go
func CompletionHandler(cl *config.BackendConfigLoader, ml *model.ModelLoader, appConfig *config.ApplicationConfig) fiber.Handler {
    return func(c *fiber.Ctx) error {
        req, err := GetValidatedBody[schema.OpenAIRequest](c)
        if err != nil {
            return SendError(c, fiber.StatusBadRequest, "invalid_request_error", err.Error())
        }

        cfg, err := cl.LoadBackendConfigFileByName(req.Model, appConfig.ModelPath)
        if err != nil {
            return SendError(c, fiber.StatusNotFound, "not_found_error",
                fmt.Sprintf("model '%s' not found", req.Model))
        }

        result, err := backend.ModelInference(c.Context(), req, ml, cfg, appConfig, nil)
        if err != nil {
            log.Error().Err(err).Str("model", req.Model).Msg("inference error")
            return SendError(c, fiber.StatusInternalServerError, "server_error",
                "inference failed: "+err.Error())
        }

        return c.JSON(result)
    }
}
```

## Wrapping Backend Errors

When calling backends, always wrap errors with context using `fmt.Errorf`:

```go
result, err := grpcClient.Predict(ctx, request)
if err != nil {
    return nil, fmt.Errorf("backend predict call failed for model %s: %w", modelName, err)
}
```

This preserves the original error for logging while adding context for debugging.

## Panic Recovery Middleware

All routes should be protected by Fiber's built-in recover middleware to prevent crashes from propagating:

```go
import "github.com/gofiber/fiber/v2/middleware/recover"

app.Use(recover.New(recover.Config{
    EnableStackTrace: true,
    StackTraceHandler: func(c *fiber.Ctx, e interface{}) {
        log.Error().Interface("panic", e).Str("path", c.Path()).Msg("recovered from panic")
    },
}))
```

## Logging Errors

Use `zerolog` for structured error logging. Always include relevant context fields:

```go
log.Error().
    Err(err).
    Str("model", modelName).
    Str("request_id", c.Locals("trace_id").(string)).
    Msg("failed to load model")
```

Avoid logging sensitive fields such as:
- API keys or tokens
- Full request bodies containing user data
- File paths that reveal system structure

## gRPC / Backend-Specific Errors

Backend errors from gRPC calls should be inspected using the `status` and `codes` packages:

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

if err != nil {
    st, ok := status.FromError(err)
    if ok && st.Code() == codes.Unavailable {
        return SendError(c, fiber.StatusServiceUnavailable, "server_error",
            "backend is currently unavailable")
    }
    return SendError(c, fiber.StatusInternalServerError, "server_error", err.Error())
}
```

## Context Cancellation

Always check for context cancellation in long-running operations and return a meaningful error:

```go
select {
case <-ctx.Done():
    return nil, fmt.Errorf("request cancelled or timed out: %w", ctx.Err())
default:
}
```

## Testing Error Paths

Every handler should have tests for its error branches:

```go
func TestCompletionHandler_ModelNotFound(t *testing.T) {
    app := fiber.New()
    app.Post("/v1/completions", CompletionHandler(mockLoader, mockModelLoader, mockConfig))

    req := httptest.NewRequest("POST", "/v1/completions",
        strings.NewReader(`{"model":"nonexistent","prompt":"hello"}`))
    req.Header.Set("Content-Type", "application/json")

    resp, _ := app.Test(req)
    assert.Equal(t, 404, resp.StatusCode)

    var body APIError
    json.NewDecoder(resp.Body).Decode(&body)
    assert.Equal(t, "not_found_error", body.Error.Type)
}
```

## Related Guides

- [Request Validation](.agents/request-validation.md) — validate inputs before processing
- [Request Logging and Tracing](.agents/request-logging-and-tracing.md) — correlate errors with trace IDs
- [Observability and Metrics](.agents/observability-and-metrics.md) — track error rates via metrics
- [API Endpoints and Auth](.agents/api-endpoints-and-auth.md) — register handlers with error middleware
