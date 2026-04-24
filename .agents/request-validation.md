# Request Validation in LocalAI

This guide covers how to add input validation to API endpoints in LocalAI, ensuring requests are well-formed before they reach backend processing.

## Overview

LocalAI uses a middleware-based validation approach. Validators are registered per-route and run before handlers. Validation errors return structured JSON with HTTP 400 or 422 status codes.

## Validation Middleware

```go
// pkg/middleware/validate.go
package middleware

import (
	"encoding/json"
	"net/http"

	"github.com/go-playground/validator/v10"
	"github.com/gofiber/fiber/v2"
	"github.com/rs/zerolog/log"
)

var validate = validator.New()

// ValidationError represents a structured validation failure.
type ValidationError struct {
	Field   string `json:"field"`
	Tag     string `json:"tag"`
	Message string `json:"message"`
}

// ValidateBody parses and validates the request body against the given struct type.
// Usage: app.Post("/v1/completions", ValidateBody[CompletionRequest](), handler)
func ValidateBody[T any]() fiber.Handler {
	return func(c *fiber.Ctx) error {
		var body T
		if err := c.BodyParser(&body); err != nil {
			log.Warn().Err(err).Str("path", c.Path()).Msg("failed to parse request body")
			return c.Status(http.StatusBadRequest).JSON(fiber.Map{
				"error": "invalid JSON body",
			})
		}

		if errs := validate.Struct(body); errs != nil {
			var validationErrors []ValidationError
			for _, e := range errs.(validator.ValidationErrors) {
				validationErrors = append(validationErrors, ValidationError{
					Field:   e.Field(),
					Tag:     e.Tag(),
					Message: e.Translate(nil),
				})
			}
			log.Warn().
				Str("path", c.Path()).
				Int("num_errors", len(validationErrors)).
				Msg("request validation failed")
			return c.Status(http.StatusUnprocessableEntity).JSON(fiber.Map{
				"error":   "validation failed",
				"details": validationErrors,
			})
		}

		// Store validated body in locals for the handler to retrieve.
		c.Locals("validatedBody", body)
		return c.Next()
	}
}

// GetValidatedBody retrieves the validated body stored by ValidateBody middleware.
func GetValidatedBody[T any](c *fiber.Ctx) (T, bool) {
	body, ok := c.Locals("validatedBody").(T)
	return body, ok
}
```

## Defining Validated Request Structs

Annotate request structs with `validate` tags:

```go
// api/types/requests.go
package types

type CompletionRequest struct {
	Model       string    `json:"model"       validate:"required,min=1,max=256"`
	Prompt      string    `json:"prompt"      validate:"required,min=1"`
	MaxTokens   int       `json:"max_tokens"  validate:"omitempty,min=1,max=32768"`
	Temperature float64   `json:"temperature" validate:"omitempty,min=0,max=2"`
	Stop        []string  `json:"stop"        validate:"omitempty,max=4"`
	Stream      bool      `json:"stream"`
}

type ChatCompletionRequest struct {
	Model       string        `json:"model"       validate:"required,min=1,max=256"`
	Messages    []ChatMessage `json:"messages"    validate:"required,min=1,dive"`
	MaxTokens   int           `json:"max_tokens"  validate:"omitempty,min=1,max=32768"`
	Temperature float64       `json:"temperature" validate:"omitempty,min=0,max=2"`
	Stream      bool          `json:"stream"`
}

type ChatMessage struct {
	Role    string `json:"role"    validate:"required,oneof=system user assistant tool"`
	Content string `json:"content" validate:"required"`
}
```

## Registering Validation on Routes

```go
func RegisterMyRoutes(app *fiber.App, deps *Dependencies) {
	v1 := app.Group("/v1")

	v1.Post("/completions",
		AuthMiddleware(deps.Config),
		RateLimitMiddleware(deps.Config),
		ValidateBody[types.CompletionRequest](),
		CompletionHandler(deps),
	)

	v1.Post("/chat/completions",
		AuthMiddleware(deps.Config),
		RateLimitMiddleware(deps.Config),
		ValidateBody[types.ChatCompletionRequest](),
		ChatCompletionHandler(deps),
	)
}
```

## Retrieving the Validated Body in a Handler

```go
func CompletionHandler(deps *Dependencies) fiber.Handler {
	return func(c *fiber.Ctx) error {
		req, ok := middleware.GetValidatedBody[types.CompletionRequest](c)
		if !ok {
			// Should not happen if middleware is registered correctly.
			return c.Status(http.StatusInternalServerError).JSON(fiber.Map{
				"error": "internal: missing validated body",
			})
		}

		// Use req.Model, req.Prompt, etc.
		_ = req
		return c.JSON(fiber.Map{"status": "ok"})
	}
}
```

## Testing Validation

```go
func TestValidateBody_MissingModel(t *testing.T) {
	app := fiber.New()
	app.Post("/v1/completions", ValidateBody[types.CompletionRequest](), func(c *fiber.Ctx) error {
		return c.SendStatus(200)
	})

	body := `{"prompt": "hello"}`  // model is missing
	req := httptest.NewRequest(http.MethodPost, "/v1/completions", strings.NewReader(body))
	req.Header.Set("Content-Type", "application/json")

	resp, _ := app.Test(req)
	assert.Equal(t, http.StatusUnprocessableEntity, resp.StatusCode)

	var result map[string]interface{}
	json.NewDecoder(resp.Body).Decode(&result)
	assert.Equal(t, "validation failed", result["error"])
}

func TestValidateBody_Valid(t *testing.T) {
	app := fiber.New()
	app.Post("/v1/completions", ValidateBody[types.CompletionRequest](), func(c *fiber.Ctx) error {
		return c.SendStatus(200)
	})

	body := `{"model": "gpt-4", "prompt": "hello"}`
	req := httptest.NewRequest(http.MethodPost, "/v1/completions", strings.NewReader(body))
	req.Header.Set("Content-Type", "application/json")

	resp, _ := app.Test(req)
	assert.Equal(t, http.StatusOK, resp.StatusCode)
}
```

## Custom Validators

Register domain-specific rules once at startup:

```go
func init() {
	// Validate that a model name contains no path traversal sequences.
	validate.RegisterValidation("safemodelname", func(fl validator.FieldLevel) bool {
		v := fl.Field().String()
		return !strings.Contains(v, "..") && !strings.Contains(v, "/")
	})
}
```

Then use `validate:"required,safemodelname"` in your struct tags.

## Error Response Format

All validation errors follow this schema so clients can handle them uniformly:

```json
{
  "error": "validation failed",
  "details": [
    {"field": "Model", "tag": "required", "message": "Model is a required field"},
    {"field": "Temperature", "tag": "max", "message": "Temperature must be at most 2"}
  ]
}
```

## Notes

- Always place `ValidateBody` **after** auth/rate-limit middleware so unauthenticated requests are rejected cheaply before parsing.
- Use `omitempty` for optional fields to avoid false positives on zero values.
- The `dive` tag is required to validate slice elements (e.g., `Messages`).
- Keep struct definitions in `api/types/` so they can be shared across handlers and tests.
