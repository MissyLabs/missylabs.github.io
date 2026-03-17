---
hide:
  - navigation
  - toc
tags:
  - home
---

<div class="hero" markdown>

# Missy

<p class="subtitle">
Security-first, self-hosted AI assistant for Linux. Production-grade agent platform with strict policy enforcement, encrypted vault, multi-provider support, voice channel, and full auditability.
</p>

<div class="buttons">
  <a href="getting-started/" class="primary">Get Started</a>
  <a href="https://github.com/MissyLabs/missy" class="secondary">View on GitHub</a>
</div>

</div>

<div class="feature-grid" markdown>

<div class="feature-card" markdown>

### :material-shield-lock: Security First

Every capability is **disabled by default**. Network, filesystem, shell, and plugin access require explicit opt-in. Three-layer policy engine enforces rules on every request. ChaCha20-Poly1305 encrypted vault for secrets.

</div>

<div class="feature-card" markdown>

### :material-swap-horizontal: Multi-Provider

Switch between **Anthropic**, **OpenAI**, and **Ollama** (local models) — even at runtime. API key rotation, model tiers (fast/primary/premium), and automatic fallback when a provider is down.

</div>

<div class="feature-card" markdown>

### :material-microphone: Voice Native

WebSocket voice channel with dedicated **Raspberry Pi edge nodes**. Wake word detection, local STT (faster-whisper), local TTS (Piper). Per-node policy modes with PBKDF2 device authentication.

</div>

<div class="feature-card" markdown>

### :material-robot: Agentic Runtime

Multi-step tool loop with circuit breaker, checkpoint/recovery, cost tracking, and budget caps. Sub-agents, approval gates, learnings extraction, and self-tuning prompt patches.

</div>

<div class="feature-card" markdown>

### :material-eye: Full Auditability

Every action logged as structured JSONL — network requests, file access, shell commands, tool calls. OpenTelemetry export for production observability. No silent failures.

</div>

<div class="feature-card" markdown>

### :material-puzzle: Extensible

Built-in tools, skills, plugins, and **MCP server** integration. Digest-pinned MCP connections for supply chain safety. Config presets, auto-migration, plan/rollback for operations.

</div>

</div>

## Quick Install

```bash
curl -fsSL https://raw.githubusercontent.com/MissyLabs/missy/master/install.sh | bash
```

Or install manually:

=== "pip"

    ```bash
    git clone https://github.com/MissyLabs/missy.git
    cd missy
    pip install -e .
    missy setup
    ```

=== "With voice support"

    ```bash
    pip install -e ".[voice]"
    ```

=== "With OpenTelemetry"

    ```bash
    pip install -e ".[otel]"
    ```

## Architecture

```
CLI / Discord / Webhook / Voice
         │
    AgentRuntime
    ├── InputSanitizer + SecretsDetector
    ├── PolicyEngine (network / filesystem / shell)
    ├── CircuitBreaker + RateLimiter
    ├── ContextManager (token budget)
    ├── ProviderRegistry (Anthropic / OpenAI / Ollama)
    ├── ToolRegistry + MCP Manager
    ├── Memory + Learnings
    ├── ApprovalGate
    └── AuditLogger + OpenTelemetry
```

## Channels

| Channel | Description |
|---------|-------------|
| **CLI** | Interactive REPL and single-shot `missy ask` |
| **Discord** | Full Gateway API with DM/guild policies, slash commands |
| **Webhook** | HTTP ingress for automation pipelines |
| **Voice** | WebSocket server + Raspberry Pi edge nodes with wake word |

---

<div style="text-align: center; margin: 3rem 0 1rem;">

[:material-github: MissyLabs](https://github.com/MissyLabs){ .md-button }
[:material-raspberry-pi: Missy Edge](https://github.com/MissyLabs/missy-edge){ .md-button }

</div>
