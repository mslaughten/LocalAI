# Observability and Metrics in LocalAI

This guide covers how LocalAI exposes metrics, traces, and logs for monitoring and debugging production deployments.

## Overview

LocalAI uses [Prometheus](https://prometheus.io/) for metrics exposition and supports structured logging via `zerolog`. OpenTelemetry tracing is available for distributed tracing.

## Metrics Endpoint

By default, LocalAI exposes a `/metrics` endpoint compatible with Prometheus scraping.

```yaml
# prometheus scrape config example
scrape_configs:
  - job_name: 'localai'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/metrics'
```

### Available Metrics

| Metric Name | Type | Description |
|---|---|---|
| `localai_inference_duration_seconds` | Histogram | Time spent on model inference |
| `localai_requests_total` | Counter | Total number of API requests |
| `localai_requests_in_flight` | Gauge | Currently active requests |
| `localai_model_load_duration_seconds` | Histogram | Time to load a model into memory |
| `localai_loaded_models` | Gauge | Number of models currently loaded |
| `localai_tokens_generated_total` | Counter | Total tokens generated across all requests |
| `localai_tokens_prompt_total` | Counter | Total prompt tokens processed |

### Metric Labels

Most metrics include the following labels:
- `model`: the model name or alias used in the request
- `backend`: the backend handling the request (e.g., `llama-cpp`, `whisper`, `stablediffusion`)
- `method`: HTTP method (`GET`, `POST`)
- `status`: HTTP response status code

## Enabling Metrics

Metrics are enabled by default. To disable them, set the environment variable:

```bash
export LOCALAI_DISABLE_METRICS=true
```

Or via the config file:

```yaml
disable_metrics: true
```

## Structured Logging

LocalAI uses `zerolog` for structured JSON logging. Set the log level via:

```bash
export LOCALAI_LOG_LEVEL=debug   # trace, debug, info, warn, error
```

Example log output:

```json
{"level":"info","model":"gpt-4","backend":"llama-cpp","tokens":128,"duration_ms":340,"time":"2024-01-15T10:23:01Z","message":"inference complete"}
```

### Log Fields Reference

- `model` — model identifier
- `backend` — backend name
- `request_id` — unique request UUID (set via `X-Request-ID` header or auto-generated)
- `duration_ms` — request duration in milliseconds
- `tokens` — tokens generated
- `error` — error message if applicable

## OpenTelemetry Tracing (Experimental)

Distributed tracing can be enabled by providing an OTLP endpoint:

```bash
export LOCALAI_OTEL_ENDPOINT=http://jaeger:4318
export LOCALAI_OTEL_SERVICE_NAME=localai
```

This will emit spans for:
- HTTP request handling
- Model loading
- Backend inference calls
- Tokenization

## Health Check Endpoints

| Endpoint | Description |
|---|---|
| `GET /healthz` | Liveness probe — returns `200 OK` if the server is running |
| `GET /readyz` | Readiness probe — returns `200 OK` only when at least one model is loaded and ready |

Example response from `/readyz`:

```json
{
  "status": "ready",
  "models_loaded": 2,
  "uptime_seconds": 3600
}
```

## Grafana Dashboard

A community-maintained Grafana dashboard is available in `extra/grafana/localai-dashboard.json`. Import it into your Grafana instance and point it at your Prometheus data source.

Key panels include:
- Request rate and error rate
- p50/p95/p99 inference latency
- Tokens per second throughput
- Active model count and memory usage

## Adding Custom Metrics

If you are adding a new backend or feature and want to instrument it:

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var myBackendRequests = promauto.NewCounterVec(
    prometheus.CounterOpts{
        Name: "localai_mybackend_requests_total",
        Help: "Total requests handled by mybackend",
    },
    []string{"model", "status"},
)

// In your handler:
myBackendRequests.WithLabelValues(modelName, "success").Inc()
```

Follow the existing naming convention: `localai_<subsystem>_<metric>_<unit>`.

## Alerting Recommendations

```yaml
# Example Prometheus alerting rules
groups:
  - name: localai
    rules:
      - alert: HighInferenceLatency
        expr: histogram_quantile(0.95, rate(localai_inference_duration_seconds_bucket[5m])) > 30
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "LocalAI p95 inference latency exceeds 30s"

      - alert: LocalAIDown
        expr: up{job="localai"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "LocalAI instance is unreachable"
```

## See Also

- [API Endpoints and Auth](.agents/api-endpoints-and-auth.md)
- [Troubleshooting Common Issues](.agents/troubleshooting-common-issues.md)
- [Building and Testing](.agents/building-and-testing.md)
