---
tags:
  - providers
---

# OpenAI

Missy supports two OpenAI-compatible providers: the standard **OpenAI** provider (Chat Completions API) and the **Codex** provider (ChatGPT backend API with OAuth).

## OpenAI provider

### Setup

=== "Environment variable (recommended)"

    ```bash
    export OPENAI_API_KEY="sk-..."
    ```

    ```yaml
    providers:
      openai:
        name: openai
        model: "gpt-4o"
        enabled: true
    ```

=== "Config file"

    ```yaml
    providers:
      openai:
        name: openai
        model: "gpt-4o"
        api_key: "sk-..."
        enabled: true
    ```

### Verify

```bash
missy providers
missy ask "Hello" --provider openai
```

### OpenAI-compatible endpoints

The OpenAI provider works with any API that implements the Chat Completions format. Set `base_url` to target alternative services:

```yaml
providers:
  groq:
    name: openai
    model: "llama-3.1-70b-versatile"
    api_key: "gsk_..."
    base_url: "https://api.groq.com/openai/v1"
    enabled: true
  together:
    name: openai
    model: "meta-llama/Meta-Llama-3.1-405B-Instruct-Turbo"
    api_key: "..."
    base_url: "https://api.together.xyz/v1"
    enabled: true
```

!!! tip "Network policy"
    Provider `base_url` hosts are automatically added to `provider_allowed_hosts`, so you usually do not need to update the network policy manually.

### Models

| Model | Config value | Notes |
|-------|-------------|-------|
| GPT-4o | `gpt-4o` | Default -- multimodal, fast |
| GPT-4o mini | `gpt-4o-mini` | Cheaper, good for simple tasks |
| GPT-4 Turbo | `gpt-4-turbo` | Older, larger context |

### Configuration reference

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | `"openai"` | Must be `"openai"` |
| `model` | string | `"gpt-4o"` | Model identifier |
| `api_key` | string | null | API key (falls back to `OPENAI_API_KEY` env var) |
| `base_url` | string | null | Override API endpoint for compatible services |
| `timeout` | int | `30` | Request timeout in seconds |
| `enabled` | bool | `true` | Enable/disable |

## Codex provider { #codex }

The Codex provider uses ChatGPT's backend API with OAuth authentication. This is useful if you have a ChatGPT subscription and want to use it with Missy.

### Setup via wizard

The easiest way to configure the Codex provider is through the setup wizard:

```bash
missy setup
```

Select **OpenAI (OAuth/Codex)** when prompted. The wizard handles the OAuth flow and stores the token securely at `~/.missy/secrets/openai-oauth.json`.

### Manual configuration

```yaml
providers:
  openai-codex:
    name: openai-codex
    model: "gpt-4o"
    timeout: 60
    enabled: true
```

The OAuth token is loaded automatically from `~/.missy/secrets/openai-oauth.json` and refreshed when near expiry.

### How it differs from the standard OpenAI provider

| Aspect | OpenAI provider | Codex provider |
|--------|----------------|----------------|
| Endpoint | `api.openai.com` | `chatgpt.com/backend-api` |
| Auth | API key | OAuth bearer token |
| Request format | Chat Completions | Responses API |
| Extra headers | None | `chatgpt-account-id` from JWT |

## Dependencies

Both providers require the `openai` Python package:

```bash
pip install -e "."
```

The Codex provider additionally uses `httpx` (bundled with Missy's dependencies).
