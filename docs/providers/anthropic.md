---
tags:
  - providers
---

# Anthropic

The Anthropic provider connects Missy to Claude via the Messages API. It supports tool calling, streaming, and model tiers.

## Setup

### Step 1: Get an API key

1. Go to [console.anthropic.com](https://console.anthropic.com/).
2. Create an API key under **API Keys**.
3. Copy the key (starts with `sk-ant-api`).

!!! danger "Setup tokens are not supported"
    Keys starting with `sk-ant-oat` are setup tokens used for OAuth flows, not API keys. They will not work with the Messages API. Make sure your key starts with `sk-ant-api`.

### Step 2: Configure

=== "Environment variable (recommended)"

    ```bash
    export ANTHROPIC_API_KEY="sk-ant-api03-..."
    ```

    ```yaml
    providers:
      anthropic:
        name: anthropic
        model: "claude-sonnet-4-6"
        enabled: true
    ```

=== "Config file"

    ```yaml
    providers:
      anthropic:
        name: anthropic
        model: "claude-sonnet-4-6"
        api_key: "sk-ant-api03-..."
        enabled: true
    ```

=== "Vault"

    ```bash
    missy vault set anthropic_api_key "sk-ant-api03-..."
    ```

    ```yaml
    providers:
      anthropic:
        name: anthropic
        model: "claude-sonnet-4-6"
        api_key: "vault://anthropic_api_key"
        enabled: true
    ```

### Step 3: Verify

```bash
missy providers
```

Expected output:

```
Providers:
  anthropic: available (claude-sonnet-4-6)
```

Test with a query:

```bash
missy ask "Hello, which model are you?" --provider anthropic
```

## Models

| Model | Config value | Use case |
|-------|-------------|----------|
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | Default -- balanced speed and capability |
| Claude Haiku 4.5 | `claude-haiku-4-5` | Fast tier -- simple queries, low cost |
| Claude Opus 4.6 | `claude-opus-4-6` | Premium tier -- complex reasoning |

### Model tiers

```yaml
providers:
  anthropic:
    name: anthropic
    model: "claude-sonnet-4-6"
    fast_model: "claude-haiku-4-5"
    premium_model: "claude-opus-4-6"
```

The `ModelRouter` automatically selects the tier based on prompt complexity. Short, simple queries go to `fast_model`; complex tasks with many tools or long context go to `premium_model`.

## Configuration reference

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | `"anthropic"` | Provider identifier |
| `model` | string | `"claude-sonnet-4-6"` | Primary model |
| `api_key` | string | null | API key (falls back to `ANTHROPIC_API_KEY` env var) |
| `api_keys` | list | `[]` | Multiple keys for rotation |
| `timeout` | int | `30` | Request timeout in seconds |
| `fast_model` | string | `""` | Model for fast/simple tier |
| `premium_model` | string | `""` | Model for premium/complex tier |
| `enabled` | bool | `true` | Enable/disable this provider |

## Dependencies

The Anthropic provider requires the `anthropic` Python package. It is installed automatically with Missy's base dependencies:

```bash
pip install -e "."
```

If the package is not installed, `is_available()` returns `false` and the provider is skipped during registry initialization.
