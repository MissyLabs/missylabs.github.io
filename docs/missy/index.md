---
hide:
  - toc
  - tags
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

Every capability is **disabled by default**. Network, filesystem, shell, and plugin access require explicit opt-in. Three-layer policy engine enforces rules on every request. ChaCha20-Poly1305 encrypted vault for secrets. **Container-per-session sandbox** for OS-level isolation, **Ed25519 agent identity** for signed audit trails, and **trust scoring** (0--1000) for provider and tool reliability tracking.

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

Multi-step tool loop with circuit breaker, checkpoint/recovery, cost tracking, and budget caps. Sub-agents, learnings extraction, and self-tuning prompt patches. **AI Playbook** auto-captures successful tool patterns and promotes them to skills. **Sleep Mode** consolidates context when the token window fills. **Attention System** tracks urgency, focus, and topic continuity across turns. **Interactive approval TUI** surfaces policy-denied operations for real-time operator approval with session-scoped "allow always" memory.

</div>

<div class="feature-card" markdown>

### :material-eye: Full Auditability

Every action logged as structured JSONL — network requests, file access, shell commands, tool calls. OpenTelemetry export for production observability. No silent failures.

</div>

<div class="feature-card" markdown>

### :material-puzzle: Extensible

Built-in tools, skills, plugins, and **MCP server** integration. Digest-pinned MCP connections for supply chain safety. Config presets, auto-migration, plan/rollback for operations. **FAISS vector memory** for semantic search across conversation history. **SKILL.md dynamic discovery** for cross-agent skill portability. **Async Message Bus** with topic wildcards and priority queuing for event-driven subsystem coordination.

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

=== "With vision"

    ```bash
    pip install -e ".[vision]"
    ```

=== "With desktop automation"

    ```bash
    pip install -e ".[desktop]"
    playwright install firefox
    ```

## Architecture

```
CLI / Discord / Webhook / Voice / Screencast / API
         │
    AgentRuntime
    ├── InputSanitizer + SecretsDetector + PromptDriftDetector
    ├── PolicyEngine (network / filesystem / shell / REST L7)
    ├── AgentIdentity (Ed25519) + TrustScorer (0-1000)
    ├── CircuitBreaker + RateLimiter
    ├── ContextManager (token budget) + MemoryConsolidator (sleep mode)
    ├── AttentionSystem (alerting / orienting / sustained / selective / executive)
    ├── ProviderRegistry (Anthropic / OpenAI / Ollama)
    ├── ToolRegistry + MCP Manager + SKILL.md Discovery
    ├── Memory + Learnings + Vector Memory (FAISS) + Graph Memory
    ├── MemorySynthesizer + CondenserPipeline + SleeptimeWorker
    ├── Playbook (auto-capture → skill promotion)
    ├── MessageBus (async events, topic wildcards, priority queue)
    ├── ApprovalGate + InteractiveApproval TUI
    ├── ContainerSandbox (Docker) + LandlockPolicy (kernel LSM)
    ├── Checkpoint/Recovery + CostTracker + FailureTracker
    ├── PersonaManager + BehaviorLayer + HatchingManager
    ├── CodeEvolutionManager + StructuredOutput
    └── AuditLogger + OpenTelemetry
```

## Channels

| Channel | Description |
|---------|-------------|
| **CLI** | Interactive REPL and single-shot `missy ask` |
| **Discord** | Full Gateway API with DM/guild policies, slash commands |
| **Webhook** | HTTP ingress for automation pipelines |
| **Voice** | WebSocket server + Raspberry Pi edge nodes with wake word |
| **Screencast** | Browser-based screen capture with token auth |
| **REST API** | Agent-as-a-Service with API key auth and rate limiting |
