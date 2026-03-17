---
tags:
  - providers
---

# Providers

Providers are the AI backends that power Missy's responses. The provider system is built around a registry pattern with a common interface, so switching between Claude, GPT, Ollama, or any OpenAI-compatible service requires only a config change.

## Supported providers

| Provider | Module | API key required | Local | Tool calling |
|----------|--------|------------------|-------|-------------|
| [Anthropic](anthropic.md) | `AnthropicProvider` | Yes | No | Yes |
| [OpenAI](openai.md) | `OpenAIProvider` | Yes | No | Yes |
| [OpenAI Codex](openai.md#codex) | `CodexProvider` | OAuth token | No | Yes |
| [Ollama](ollama.md) | `OllamaProvider` | No | Yes | Yes |

## How it works

### ProviderRegistry

The `ProviderRegistry` is the single source of truth for active providers. On startup, Missy reads the `providers:` section of your config and instantiates each enabled provider:

```yaml
providers:
  anthropic:
    name: anthropic
    model: "claude-sonnet-4-6"
    api_key: null                  # Falls back to ANTHROPIC_API_KEY env var
    enabled: true
  ollama:
    name: ollama
    model: "llama3.2"
    base_url: "http://localhost:11434"
    enabled: true
```

Check what is registered and available:

```bash
missy providers
```

### BaseProvider interface

All providers implement the same interface:

| Method | Description |
|--------|-------------|
| `complete(messages)` | Send conversation turns, get a response |
| `complete_with_tools(messages, tools, system)` | Completion with tool-calling support |
| `stream(messages, system)` | Stream partial response tokens |
| `is_available()` | Check if the provider is ready (credentials present, service reachable) |
| `get_tool_schema(tools)` | Convert tools to provider-native schema format |

The common data types:

```python
Message(role="user", content="Hello")

CompletionResponse(
    content="Hi there!",
    model="claude-sonnet-4-6",
    provider="anthropic",
    usage={"prompt_tokens": 10, "completion_tokens": 5, "total_tokens": 15},
    tool_calls=[],            # List[ToolCall] when model requests tool use
    finish_reason="stop",     # "stop" | "tool_calls" | "length"
)

ToolCall(id="call_123", name="calculator", arguments={"expression": "2+2"})
ToolResult(tool_call_id="call_123", name="calculator", content="4")
```

### ModelRouter

The `ModelRouter` automatically selects between fast, primary, and premium model tiers based on prompt complexity:

| Tier | When used | Example model |
|------|-----------|---------------|
| `fast` | Short queries, simple lookups | `claude-haiku-4-5` |
| `primary` | General purpose (default) | `claude-sonnet-4-6` |
| `premium` | Complex tasks, long context, many tools | `claude-opus-4-6` |

Configure tier models per provider:

```yaml
providers:
  anthropic:
    name: anthropic
    model: "claude-sonnet-4-6"
    fast_model: "claude-haiku-4-5"
    premium_model: "claude-opus-4-6"
```

### API key rotation

Providers support multiple API keys for load distribution and rate limit management:

```yaml
providers:
  anthropic:
    name: anthropic
    model: "claude-sonnet-4-6"
    api_keys:
      - "sk-ant-key1..."
      - "sk-ant-key2..."
      - "sk-ant-key3..."
```

Keys rotate round-robin when the registry's `rotate_key()` method is called (typically after rate limit errors).

## Network policy

All provider HTTP traffic passes through the `PolicyHTTPClient`, which enforces network policy. Provider hosts are automatically added to `provider_allowed_hosts` from their `base_url` configuration, so you generally do not need to add them manually.

If using `default_deny: true` and providers are not auto-detected, add them explicitly:

```yaml
network:
  provider_allowed_hosts:
    - "api.anthropic.com"
    - "api.openai.com"
```

## Next steps

- [Anthropic setup](anthropic.md)
- [OpenAI setup](openai.md)
- [Ollama setup](ollama.md)
- [Runtime switching](runtime-switching.md)
