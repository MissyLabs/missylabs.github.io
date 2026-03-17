---
tags:
  - providers
---

# Runtime Switching

Missy supports switching between providers at runtime without restarting. This lets you route different queries to different backends based on cost, speed, or capability needs.

## Switching providers

### Per-command

Override the provider for a single command:

```bash
# Use Ollama for a quick local query
missy ask "What time is it in Tokyo?" --provider ollama

# Use Anthropic for a complex task
missy ask "Analyze this codebase and suggest refactoring" --provider anthropic
```

### Interactive switching

During a `missy run` session, the provider can be switched for subsequent messages.

### Default provider

Set the default provider that is used when no `--provider` flag is given:

```bash
missy providers switch anthropic
```

The registry validates that the provider is registered and available before making it the default.

## How it works

The `ProviderRegistry` maintains a `_default_name` that can be changed at runtime via `set_default()`. When the agent runtime processes a request:

1. If a specific provider was requested (via `--provider` flag), that provider is used.
2. Otherwise, the default provider is used.
3. If no default is set, the first available provider is selected.

Provider availability is checked via `is_available()`, which verifies that credentials are present and (for some providers) that the upstream service is reachable.

## ModelRouter: automatic tier selection

The `ModelRouter` can automatically route queries to different model tiers within the same provider:

```yaml
providers:
  anthropic:
    name: anthropic
    model: "claude-sonnet-4-6"        # Primary tier
    fast_model: "claude-haiku-4-5"     # Fast tier
    premium_model: "claude-opus-4-6"   # Premium tier
```

The router scores prompt complexity based on:

| Signal | Routes to |
|--------|-----------|
| Short prompt (< 80 chars) + simple keywords | `fast` tier |
| Long prompt (> 500 chars) | `premium` tier |
| Many tools (> 3) | `premium` tier |
| Long conversation history (> 10 turns) | `premium` tier |
| Keywords: debug, architect, refactor, analyze, optimize, complex | `premium` tier |
| Everything else | `primary` tier |

## API key rotation

When a provider has multiple API keys configured, the registry can rotate to the next key:

```yaml
providers:
  anthropic:
    api_keys:
      - "sk-ant-key1..."
      - "sk-ant-key2..."
      - "sk-ant-key3..."
```

Rotation happens round-robin via `registry.rotate_key("anthropic")`, typically triggered after a rate limit error from the provider.

## Config hot-reload

Missy watches `~/.missy/config.yaml` for changes and reloads the provider configuration automatically. The `ConfigWatcher` polls the file every second and applies changes after a 2-second debounce window.

This means you can update provider settings (add a new provider, change models, update API keys) by editing the config file, and the changes take effect without restarting Missy.

## Checking provider status

```bash
# List all providers with availability status
missy providers

# System health check (includes provider connectivity)
missy doctor
```
