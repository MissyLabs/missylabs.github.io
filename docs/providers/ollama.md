---
tags:
  - providers
---

# Ollama

The Ollama provider connects Missy to locally-running models via the [Ollama](https://ollama.com) REST API. No API key required -- everything runs on your hardware.

## Setup

### Step 1: Install Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### Step 2: Pull a model

```bash
ollama pull llama3.2
```

### Step 3: Configure Missy

```yaml
providers:
  ollama:
    name: ollama
    model: "llama3.2"
    enabled: true
```

That is it. No API key, no network policy changes for the default local setup.

### Step 4: Verify

```bash
missy providers
missy ask "Hello" --provider ollama
```

## Configuration

```yaml
providers:
  ollama:
    name: ollama
    model: "llama3.2"
    base_url: "http://localhost:11434"    # Default Ollama address
    timeout: 60                           # Local inference can be slower
    enabled: true
```

### Remote Ollama server

If Ollama runs on a different machine:

```yaml
providers:
  ollama:
    name: ollama
    model: "llama3.2"
    base_url: "http://10.0.0.50:11434"
    enabled: true
```

The `base_url` host is automatically added to `provider_allowed_hosts`, so network policy is handled for you.

### Model tiers

```yaml
providers:
  ollama:
    name: ollama
    model: "llama3.2"
    fast_model: "llama3.2:1b"
    premium_model: "llama3.2:70b"
```

## Available models

Some popular models that work well with Missy:

| Model | Size | Good for |
|-------|------|----------|
| `llama3.2` | 3B | Fast, general purpose |
| `llama3.2:1b` | 1B | Very fast, simple queries |
| `llama3.2:70b` | 70B | Complex reasoning (needs large GPU) |
| `mistral` | 7B | Good balance of speed and quality |
| `codellama` | 7B | Code-focused tasks |
| `deepseek-coder-v2` | 16B | Code generation and analysis |

Pull any model with:

```bash
ollama pull MODEL_NAME
```

## How it works

Unlike the Anthropic and OpenAI providers, the Ollama provider does **not** use a vendor SDK. It communicates directly with the Ollama `/api/chat` endpoint via Missy's `PolicyHTTPClient`, which means all requests pass through the network policy engine.

The provider uses `stream=false` to receive the complete response in a single JSON payload.

## Configuration reference

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | `"ollama"` | Must be `"ollama"` |
| `model` | string | `"llama3.2"` | Model name (must be pulled first) |
| `base_url` | string | `"http://localhost:11434"` | Ollama server URL |
| `timeout` | int | `30` | Request timeout in seconds (increase for large models) |
| `enabled` | bool | `true` | Enable/disable |

!!! tip "Increase timeout for large models"
    Large models like `llama3.2:70b` can take 30+ seconds for first inference. Set `timeout: 120` or higher to avoid timeouts.
