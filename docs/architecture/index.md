# Architecture Overview

Missy is a security-first, self-hosted agentic AI assistant for Linux. This page describes the system architecture: how components fit together and how a request flows from input to response.

## System Diagram

```mermaid
graph TB
    subgraph Channels
        CLI[CLIChannel]
        Discord[DiscordChannel]
        Webhook[WebhookChannel]
        Voice[VoiceChannel]
    end

    subgraph Core["Agent Core"]
        Runtime[AgentRuntime]
        Context[ContextManager]
        CB[CircuitBreaker]
        Progress[ProgressReporter]
    end

    subgraph Security
        Sanitizer[InputSanitizer]
        Secrets[SecretsDetector]
        Censor[SecretCensor]
        Policy[PolicyEngine]
    end

    subgraph Providers
        Registry[ProviderRegistry]
        Router[ModelRouter]
        RL[RateLimiter]
        Anthropic[AnthropicProvider]
        OpenAI[OpenAIProvider]
        Ollama[OllamaProvider]
    end

    subgraph Tools
        ToolReg[ToolRegistry]
        MCP[McpManager]
        BuiltIn[Built-in Tools]
    end

    subgraph Persistence
        Memory[SQLiteMemoryStore]
        Audit[AuditLogger]
        Learnings[Learnings]
        Checkpoint[CheckpointManager]
    end

    CLI --> Runtime
    Discord --> Runtime
    Webhook --> Runtime
    Voice --> Runtime

    Runtime --> Sanitizer
    Runtime --> Context
    Runtime --> CB
    Runtime --> Progress

    Sanitizer --> Policy
    Runtime --> Registry
    Registry --> Router
    Router --> RL
    RL --> Anthropic
    RL --> OpenAI
    RL --> Ollama

    Runtime --> ToolReg
    ToolReg --> BuiltIn
    ToolReg --> MCP

    Policy -.->|enforces| Registry
    Policy -.->|enforces| ToolReg

    Runtime --> Memory
    Runtime --> Audit
    Runtime --> Learnings
    Runtime --> Checkpoint
```

## Key Subsystems

### Channels

Channels are the communication interfaces through which users interact with Missy. Each channel adapts its input/output format but delegates all processing to the same `AgentRuntime`.

| Channel | Transport | Use Case |
|---|---|---|
| `CLIChannel` | stdin/stdout | Interactive terminal, scripting |
| `DiscordChannel` | WebSocket + REST | Chat bot with slash commands |
| `WebhookChannel` | HTTP POST | External integrations, automation |
| `VoiceChannel` | WebSocket + PCM audio | Edge node voice assistants |

### Agent Runtime

The `AgentRuntime` is the central orchestrator. It binds a provider, session, tool registry, and all supporting subsystems into a single execution loop. See [Agent Runtime](agent-runtime.md) for details.

### Policy Engine

The `PolicyEngine` is a three-layer enforcement facade that gates all external interactions:

- **NetworkPolicyEngine** -- Controls outbound HTTP (CIDR blocks, domain matching, per-category hosts).
- **FilesystemPolicyEngine** -- Controls file read/write access (path allowlists, symlink resolution).
- **ShellPolicyEngine** -- Controls command execution (enabled flag, command whitelist).

Every check emits an audit event regardless of outcome.

### Provider System

The `ProviderRegistry` manages multiple AI providers with fallback resolution. The `ModelRouter` selects the appropriate model tier (fast, default, premium) based on task complexity. A `RateLimiter` throttles API calls per provider.

Supported providers: Anthropic, OpenAI, Ollama (and any OpenAI-compatible endpoint via `base_url` override).

### Tool System

The `ToolRegistry` manages built-in tools and MCP server connections. Tools are invoked during the agentic loop when the model requests them. Built-in tools include file operations, shell execution, web fetch, calculator, browser automation, and more.

MCP (Model Context Protocol) servers extend the tool set dynamically. Tools from MCP servers are namespaced as `server__tool`.

### Security Layer

All inputs pass through the `InputSanitizer` (prompt injection detection) before reaching the LLM. All outputs pass through the `SecretCensor` (credential redaction) before reaching the user. The `SecretsDetector` identifies 9+ credential patterns (API keys, JWTs, private keys, etc.).

### Persistence

- **SQLiteMemoryStore** -- Conversation history with FTS5 full-text search (`~/.missy/memory.db`).
- **AuditLogger** -- Structured JSONL audit trail (`~/.missy/audit.jsonl`).
- **Learnings** -- Task outcomes extracted from tool-augmented runs, persisted in SQLite.
- **CheckpointManager** -- Saves tool loop state for crash recovery.

## Request Flow

Here is how a single user request flows through the system:

```mermaid
sequenceDiagram
    participant User
    participant Channel
    participant Runtime as AgentRuntime
    participant Sanitizer as InputSanitizer
    participant Context as ContextManager
    participant Provider
    participant Tools as ToolRegistry
    participant Memory
    participant Censor as SecretCensor

    User->>Channel: "Summarize README.md"
    Channel->>Runtime: run(user_input)
    Runtime->>Sanitizer: sanitize(input)
    Sanitizer-->>Runtime: cleaned input
    Runtime->>Memory: load_history(session_id)
    Memory-->>Runtime: history + learnings
    Runtime->>Context: build_messages(system, input, history)
    Context-->>Runtime: enriched system prompt + messages

    loop Tool Loop (up to max_iterations)
        Runtime->>Provider: complete_with_tools(messages, tools)
        Provider-->>Runtime: response with tool_calls
        Runtime->>Tools: execute(file_read, {path: "README.md"})
        Tools-->>Runtime: file contents
        Runtime->>Runtime: inject verification prompt
    end

    Provider-->>Runtime: final text response
    Runtime->>Memory: save_turn(user + assistant)
    Runtime->>Censor: censor_response(text)
    Censor-->>Runtime: redacted text
    Runtime->>Channel: response text
    Channel->>User: "Here is the summary..."
```

### Step-by-Step

1. **Input** -- User sends a message through any channel.
2. **Sanitization** -- `InputSanitizer` checks for prompt injection patterns and truncates oversized payloads.
3. **Session resolution** -- A session is created or resumed from the session ID.
4. **History loading** -- Previous conversation turns and relevant memory snippets are retrieved from SQLite.
5. **Context assembly** -- `ContextManager` builds the message list within the token budget, enriching the system prompt with memory and learnings.
6. **Provider resolution** -- `ProviderRegistry` resolves the configured provider (with fallback).
7. **Tool loop** -- If tools are available and `max_iterations > 1`, the runtime enters the agentic loop:
    - Call the provider with tool schemas.
    - Execute any requested tool calls (gated by the policy engine).
    - Inject verification prompts.
    - Repeat until the model produces a final response or the iteration limit is reached.
8. **Persistence** -- User and assistant turns are saved to the memory store. Learnings are extracted from tool-augmented runs.
9. **Censorship** -- `SecretCensor` redacts any detected credentials from the response.
10. **Output** -- The final response is returned to the user through the channel.

## Design Principles

- **Secure by default** -- Every capability is disabled until explicitly enabled in config.
- **Policy enforcement at the edge** -- All external interactions (network, filesystem, shell) pass through the policy engine before execution.
- **Graceful degradation** -- Subsystems (memory, learnings, checkpoints, cost tracking) are loaded lazily and fail silently, so the core agent loop is always functional.
- **Full auditability** -- Every policy check, tool call, and provider interaction emits a structured audit event.
- **Provider agnostic** -- The same agent loop works with any provider that implements the `BaseProvider` interface.

## Further Reading

- [Agent Runtime](agent-runtime.md) -- session management, tool loop, checkpoint/recovery
- [Context Management](context-management.md) -- token budget, memory injection, pruning
- [Circuit Breaker](circuit-breaker.md) -- failure isolation with exponential backoff
- [Progress Reporting](progress-reporting.md) -- structured progress protocol
