# Model Configuration Guide

This document describes how to configure models in LocalAI, including YAML configuration files, parameters, and backend-specific options.

## Overview

LocalAI uses YAML configuration files to define model behavior, backend selection, and inference parameters. Each model can have its own configuration file placed in the models directory.

## Configuration File Structure

A model configuration file (`<model-name>.yaml`) supports the following top-level fields:

```yaml
# The name used to reference this model via the API
name: my-model

# Path to the model weights file (relative to the models directory)
parameters:
  model: llama-2-7b-chat.Q4_K_M.gguf

# Backend to use for inference
backend: llama-cpp

# Context window size
context_size: 4096

# Number of GPU layers to offload (-1 = all)
gpu_layers: 35

# Number of threads for CPU inference
threads: 8

# Enable memory mapping
mmap: true

# Enable memory locking
mlock: false
```

## Inference Parameters

Default inference parameters can be set under `parameters`:

```yaml
parameters:
  model: my-model.gguf
  temperature: 0.7
  top_p: 0.9
  top_k: 40
  max_tokens: 512
  repeat_penalty: 1.1
  seed: -1
```

These serve as defaults and can be overridden per-request via the API.

## Template Configuration

Prompt templates control how chat messages are formatted before being sent to the model:

```yaml
template:
  chat: |
    {{.Input}}
  chat_message: |
    {{if eq .RoleName "system"}}<<SYS>>
    {{.Content}}
    <</SYS>>
    {{else if eq .RoleName "user"}}[INST] {{.Content}} [/INST]
    {{else}}{{.Content}}
    {{end}}
  completion: |
    {{.Input}}
```

Templates use Go's `text/template` syntax. Available variables:
- `.Input` — the full assembled prompt
- `.RoleName` — the message role (`system`, `user`, `assistant`)
- `.Content` — the message content

## Feature Flags

```yaml
# Enable function calling / tool use
function_call_results: true

# Enable embeddings endpoint for this model
embeddings: false

# Enable image generation (for diffusion models)
diffusers:
  pipeline_type: StableDiffusionPipeline
  cuda: false
  scheduler_type: euler_a
  cfg_scale: 7
  img2img: false
```

## Backend-Specific Options

### llama-cpp

```yaml
backend: llama-cpp
gpu_layers: 35
f16: true
mmap: true
mlock: false
numa: false
low_vram: false
vocab_only: false
```

### whisper

```yaml
backend: whisper
parameters:
  model: ggml-base.en.bin
language: en
translate: false
```

### bert-embeddings

```yaml
backend: bert-embeddings
parameters:
  model: bert-base-uncased
embeddings: true
```

## Roles and System Prompts

A default system prompt can be baked into the configuration:

```yaml
system_prompt: |
  You are a helpful, respectful and honest assistant.
```

This is prepended to every conversation unless the request provides its own system message.

## Environment Variables in Config

Values can reference environment variables using the `${VAR}` syntax:

```yaml
parameters:
  model: ${MODELS_PATH}/llama-2-7b.gguf
```

## Validating a Configuration

Use the LocalAI CLI to validate a model config before deploying:

```bash
local-ai validate --config /path/to/models/my-model.yaml
```

Common validation errors:
- Missing `name` field
- Model file not found at the specified path
- Unsupported backend name
- Invalid template syntax

## Tips

- Use `gpu_layers: -1` to offload all layers to GPU when VRAM allows.
- Set `threads` to the number of physical CPU cores for best CPU performance.
- For chat models, always define a `chat_message` template matching the model's training format.
- The `context_size` should not exceed the model's maximum supported context length.
