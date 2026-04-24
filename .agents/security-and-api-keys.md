# Security and API Keys in LocalAI

This guide covers how LocalAI handles authentication, API key management, and security best practices for deployments.

## Overview

LocalAI supports multiple authentication mechanisms to secure your API endpoints. By default, LocalAI runs without authentication (suitable for local development), but production deployments should always use API key protection.

## Configuration

### Environment Variables

```bash
# Single API key
LOCALAI_API_KEY=your-secret-key

# Multiple API keys (comma-separated)
LOCALAI_API_KEYS=key1,key2,key3

# Disable auth entirely (default for local use)
LOCALAI_API_KEY=
```

### Config File (`localai.yaml`)

```yaml
apiKeys:
  - sk-localai-key1
  - sk-localai-key2
cors: true
corsAllowOrigins: "*"
```

## How Authentication Works

LocalAI uses bearer token authentication compatible with the OpenAI API format:

```
Authorization: Bearer <your-api-key>
```

The middleware checks the `Authorization` header on all protected routes. If no API keys are configured, all requests are allowed through.

### Auth Middleware (internal reference)

The auth middleware is registered in `pkg/http/middleware/auth.go`:

```go
// AuthMiddleware validates bearer tokens against configured API keys.
// If no keys are configured, all requests pass through.
func AuthMiddleware(appConfig *config.ApplicationConfig) fiber.Handler {
    return func(c *fiber.Ctx) error {
        if len(appConfig.ApiKeys) == 0 {
            return c.Next()
        }

        authHeader := c.Get("Authorization")
        if authHeader == "" {
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
                "error": "missing authorization header",
            })
        }

        token := strings.TrimPrefix(authHeader, "Bearer ")
        for _, key := range appConfig.ApiKeys {
            if token == key {
                return c.Next()
            }
        }

        return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
            "error": "invalid api key",
        })
    }
}
```

## CORS Configuration

For browser-based clients, configure CORS appropriately:

```bash
# Allow all origins (development only)
LOCALAI_CORS=true
LOCALAI_CORS_ALLOW_ORIGINS=*

# Restrict to specific origins (production)
LOCALAI_CORS=true
LOCALAI_CORS_ALLOW_ORIGINS=https://myapp.example.com,https://admin.example.com
```

## TLS / HTTPS

LocalAI can be configured to serve over HTTPS directly:

```bash
LOCALAI_TLS=true
LOCALAI_TLS_CERT=/path/to/cert.pem
LOCALAI_TLS_KEY=/path/to/key.pem
```

Alternatively, place LocalAI behind a reverse proxy (nginx, Caddy, Traefik) that handles TLS termination.

### Example Nginx Config

```nginx
server {
    listen 443 ssl;
    server_name localai.example.com;

    ssl_certificate /etc/ssl/certs/localai.crt;
    ssl_certificate_key /etc/ssl/private/localai.key;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header Authorization $http_authorization;
        proxy_read_timeout 300s;
    }
}
```

## Adding Auth to Custom Endpoints

When adding new API endpoints (see `api-endpoints-and-auth.md`), register them under the authenticated router group:

```go
func RegisterMyRoutes(router *fiber.App, appConfig *config.ApplicationConfig) {
    // Public routes (no auth)
    router.Get("/healthz", HealthHandler)

    // Protected routes (require API key)
    protected := router.Group("/", AuthMiddleware(appConfig))
    protected.Post("/v1/my-feature", MyHandler)
}
```

## Security Best Practices

1. **Always use HTTPS in production** — API keys sent over HTTP can be intercepted.
2. **Rotate keys regularly** — Update `LOCALAI_API_KEYS` and restart the service.
3. **Use strong keys** — Generate with `openssl rand -hex 32` or similar.
4. **Limit network exposure** — Bind to `127.0.0.1` if only accessed locally (`LOCALAI_ADDRESS=127.0.0.1:8080`).
5. **Avoid logging keys** — Ensure your log level doesn't expose request headers.
6. **Use separate keys per client** — Makes it easy to revoke access for a single consumer.

## Generating Secure API Keys

```bash
# Linux/macOS
openssl rand -hex 32

# Or using Python
python3 -c "import secrets; print(secrets.token_hex(32))"
```

## Troubleshooting Auth Issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| `401 Unauthorized` on all requests | API key mismatch | Double-check key in `Authorization: Bearer <key>` |
| `401` even with correct key | Whitespace/newline in key | Trim the key value |
| All requests pass without auth | No keys configured | Set `LOCALAI_API_KEY` |
| CORS errors in browser | CORS not enabled | Set `LOCALAI_CORS=true` |

For additional troubleshooting, see `troubleshooting-common-issues.md`.
