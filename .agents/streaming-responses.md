# Streaming Responses in LocalAI

This guide explains how to implement and handle streaming (Server-Sent Events / chunked) responses in LocalAI backends and API handlers.

## Overview

LocalAI supports streaming completions via Server-Sent Events (SSE), compatible with the OpenAI streaming API. When a client sends `"stream": true`, the handler writes incremental chunks as they arrive from the backend, rather than buffering the full response.

## Key Types

```go
// StreamWriter wraps a fiber.Ctx to provide SSE helper methods.
type StreamWriter struct {
    ctx *fiber.Ctx
}

// TokenCallback is called by the backend for each generated token.
type TokenCallback func(token string, usage TokenUsage) bool

// TokenUsage carries prompt/completion token counts for a chunk.
type TokenUsage struct {
    PromptTokens     int
    CompletionTokens int
}
```

## Enabling Streaming in a Handler

```go
func CompletionHandler(cm *config.ConfigLoader, lp *model.ModelLoader) fiber.Handler {
    return func(c *fiber.Ctx) error {
        req, err := GetValidatedBody[openai.CompletionRequest](c)
        if err != nil {
            return SendError(c, fiber.StatusBadRequest, err.Error())
        }

        if req.Stream {
            return handleStreamingCompletion(c, req, cm, lp)
        }
        return handleBlockingCompletion(c, req, cm, lp)
    }
}
```

## Writing SSE Chunks

Fiber does not have built-in SSE support, so we use `c.Context().SetBodyStreamWriter`:

```go
func handleStreamingCompletion(
    c *fiber.Ctx,
    req openai.CompletionRequest,
    cm *config.ConfigLoader,
    lp *model.ModelLoader,
) error {
    c.Set("Content-Type", "text/event-stream")
    c.Set("Cache-Control", "no-cache")
    c.Set("Connection", "keep-alive")
    c.Set("Transfer-Encoding", "chunked")

    c.Context().SetBodyStreamWriter(fasthttp.StreamWriter(func(w *bufio.Writer) {
        id := uuid.New().String()
        created := time.Now().Unix()

        cb := func(token string, usage TokenUsage) bool {
            chunk := buildChunk(id, created, req.Model, token, usage)
            data, _ := json.Marshal(chunk)
            fmt.Fprintf(w, "data: %s\\n\\n", data)
            if err := w.Flush(); err != nil {
                // Client disconnected
                return false
            }
            return true
        }

        if err := callBackendStreaming(req, cb); err != nil {
            errChunk := buildErrorChunk(err)
            data, _ := json.Marshal(errChunk)
            fmt.Fprintf(w, "data: %s\\n\\n", data)
            w.Flush()
        }

        // Send the [DONE] sentinel
        fmt.Fprint(w, "data: [DONE]\\n\\n")
        w.Flush()
    }))

    return nil
}
```

## Building Response Chunks

```go
func buildChunk(id string, created int64, model, token string, usage TokenUsage) openai.CompletionResponse {
    return openai.CompletionResponse{
        ID:      "cmpl-" + id,
        Object:  "text_completion",
        Created: created,
        Model:   model,
        Choices: []openai.Choice{
            {
                Text:         token,
                Index:        0,
                FinishReason: "",
            },
        },
    }
}
```

## Backend Streaming Callback

Backends that support streaming accept a `TokenCallback`. Return `false` from the callback to abort generation (e.g., client disconnected):

```go
func callBackendStreaming(req openai.CompletionRequest, cb TokenCallback) error {
    opts := gRPCPredictOptions(req)
    opts.TokenCallback = func(token string) bool {
        return cb(token, TokenUsage{})
    }
    return backend.Predict(context.Background(), req.Prompt, opts)
}
```

## Context Cancellation with Streaming

Always respect context cancellation to avoid leaking goroutines:

```go
func callBackendStreamingWithContext(
    ctx context.Context,
    req openai.CompletionRequest,
    cb TokenCallback,
) error {
    opts := gRPCPredictOptions(req)
    opts.TokenCallback = func(token string) bool {
        select {
        case <-ctx.Done():
            return false
        default:
            return cb(token, TokenUsage{})
        }
    }
    return backend.PredictWithContext(ctx, req.Prompt, opts)
}
```

## Testing Streaming Handlers

Use `httptest` with a pipe to capture SSE output:

```go
func TestStreamingHandler_EmitsChunks(t *testing.T) {
    app := fiber.New()
    app.Post("/completions", CompletionHandler(mockCM, mockLP))

    body := `{"model":"test","prompt":"hi","stream":true}`
    req := httptest.NewRequest(http.MethodPost, "/completions",
        strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    resp, err := app.Test(req, 5000)
    require.NoError(t, err)
    require.Equal(t, http.StatusOK, resp.StatusCode)
    require.Equal(t, "text/event-stream", resp.Header.Get("Content-Type"))

    scanner := bufio.NewScanner(resp.Body)
    var chunks []string
    for scanner.Scan() {
        line := scanner.Text()
        if strings.HasPrefix(line, "data: ") {
            chunks = append(chunks, strings.TrimPrefix(line, "data: "))
        }
    }

    require.True(t, len(chunks) >= 2, "expected at least one token chunk + [DONE]")
    assert.Equal(t, "[DONE]", chunks[len(chunks)-1])
}
```

## Common Pitfalls

- **Forgetting to flush**: Every `fmt.Fprintf` to the SSE writer must be followed by `w.Flush()`, or chunks will buffer and arrive late.
- **Ignoring callback return value**: If the client disconnects, `Flush()` returns an error. Return `false` from the callback to stop generation and free resources.
- **Missing `[DONE]` sentinel**: Some clients hang waiting for `[DONE]`. Always send it, even after an error.
- **Content-Type mismatch**: Must be `text/event-stream`, not `application/json`.
