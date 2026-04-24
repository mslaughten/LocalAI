# Troubleshooting Common Issues

This guide covers common problems encountered when developing, running, or extending LocalAI, along with their solutions.

---

## Model Loading Failures

### Symptom
The model fails to load and you see errors like:
```
failed to load model: unsupported model type
```
or the backend process exits immediately.

### Causes & Fixes

1. **Wrong backend specified** — Ensure the model config YAML has the correct `backend:` field matching a registered backend (e.g., `llama-cpp`, `whisper`, `diffusers`).
2. **Missing model file** — Verify the model file exists at the path specified in `model_path` or that the gallery download completed successfully.
3. **Incompatible GGUF/GGML version** — The llama.cpp backend is version-sensitive. Re-download the model or rebuild the backend.
4. **Insufficient memory** — Reduce `context_size`, lower `n_gpu_layers`, or use a smaller quantization (e.g., Q4_K_M instead of Q8_0).

---

## Backend Process Crashes (gRPC)

### Symptom
```
ERROR backend process died unexpectedly
```

### Causes & Fixes

1. **CUDA/GPU driver mismatch** — Run `nvidia-smi` to confirm the driver is loaded. Ensure the LocalAI build matches the CUDA toolkit version.
2. **Shared library not found** — Set `LD_LIBRARY_PATH` to include the directory containing required `.so` files:
   ```bash
   export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
   ```
3. **Port conflict** — The gRPC backend defaults to a dynamic port. If a firewall is blocking loopback connections, allow `127.0.0.1` traffic.
4. **Enable verbose backend logging** — Set `DEBUG=true` or pass `--log-level debug` to capture the backend's stderr output. See `.agents/debugging-backends.md` for details.

---

## API Returns 404 or Unexpected Errors

### Symptom
Calling `/v1/chat/completions` returns `404 Not Found` or `500 Internal Server Error`.

### Causes & Fixes

1. **Model not loaded** — Call `GET /v1/models` to list available models. If your model is missing, check the models directory and config file naming.
2. **Route not registered** — If you added a custom endpoint, confirm `RegisterMyRoutes` is called during server startup. See `.agents/api-endpoints-and-auth.md`.
3. **Authentication failure** — If `API_KEY` is set in the environment, all requests must include `Authorization: Bearer <key>`. Omitting it returns `401`.
4. **Malformed request body** — Validate your JSON payload against the OpenAI-compatible schema. Required fields: `model`, `messages` (for chat).

---

## Slow Inference / High Latency

### Causes & Fixes

1. **CPU-only inference** — If GPU offloading is not configured, set `n_gpu_layers` to a positive integer (e.g., `35`) in the model config.
2. **Low thread count** — Increase `threads` in the model config to match your physical core count.
3. **Large context window** — Reduce `context_size` if you do not need long contexts.
4. **Streaming disabled** — Enable `stream: true` in the request body to receive tokens incrementally and reduce perceived latency.

---

## Gallery Model Download Fails

### Symptom
```
failed to download model: checksum mismatch
```
or the download hangs indefinitely.

### Causes & Fixes

1. **Checksum mismatch** — The remote file may have changed. Update the `sha256` field in the gallery YAML. See `.agents/adding-gallery-models.md`.
2. **Network timeout** — Set `LOCALAI_DOWNLOAD_TIMEOUT` environment variable (in seconds) to a higher value.
3. **Proxy / firewall** — Configure `HTTP_PROXY` / `HTTPS_PROXY` environment variables if your environment requires a proxy.

---

## Container / Docker Issues

### GPU not visible inside container
```bash
# Ensure the NVIDIA container toolkit is installed and pass the GPU flag:
docker run --gpus all ...
```

### Permission denied on model directory
```bash
# Match the UID of the container user or use a named volume:
docker run -v localai-models:/models ...
```

---

## Useful Environment Variables for Debugging

| Variable | Description |
|---|---|
| `DEBUG=true` | Enable verbose logging across all components |
| `LOCALAI_LOG_LEVEL=debug` | Set log verbosity (debug, info, warn, error) |
| `LOCALAI_SINGLE_ACTIVE_BACKEND=true` | Load only one backend at a time (reduces memory) |
| `LOCALAI_PARALLEL_REQUESTS=false` | Disable parallel request handling for easier debugging |
| `BACKENDS_PATH` | Override the directory scanned for backend binaries |

---

## Getting Further Help

- Check the [GitHub Issues](https://github.com/mudler/LocalAI/issues) for known bugs.
- Review backend-specific docs in `.agents/llama-cpp-backend.md` and `.agents/debugging-backends.md`.
- Join the community Discord for real-time assistance.
