---
tags:
  - getting-started
---

# Getting Started

Missy is a security-first, self-hosted AI assistant for Linux. It runs entirely on your own hardware, enforces strict access policies on every operation, and provides a full audit trail of everything it does.

## Who is Missy for?

- **Developers and sysadmins** who want an AI assistant that respects security boundaries
- **Teams** that need auditability and policy enforcement for AI-assisted workflows
- **Privacy-conscious users** who prefer self-hosted over cloud-managed agents
- **Homelab enthusiasts** interested in voice-controlled AI with Raspberry Pi edge nodes

## What you will learn

This guide walks you through going from zero to a working Missy installation:

| Page | What it covers |
|------|----------------|
| [Installation](installation.md) | System requirements, pip install, optional extras |
| [Quick Start](quickstart.md) | Fastest path: install, setup, first query |
| [Setup Wizard](setup-wizard.md) | Interactive and non-interactive configuration |
| [Your First Conversation](first-conversation.md) | Interactive REPL, tools, sessions, capability modes |

## How Missy works

Every capability -- network access, filesystem writes, shell commands, plugins -- is **disabled by default**. You explicitly enable what you need in `~/.missy/config.yaml`. The policy engine enforces these rules on every request, and every action is logged to a structured audit trail.

```
CLI / Discord / Webhook / Voice
         |
    AgentRuntime
    +-- PolicyEngine (network / filesystem / shell)
    +-- InputSanitizer + SecretsDetector
    +-- ProviderRegistry (Anthropic / OpenAI / Ollama)
    +-- ToolRegistry + MCP servers
    +-- Memory + Learnings
    +-- AuditLogger
```

Missy supports three AI providers out of the box:

- **Anthropic** (Claude) -- recommended default
- **OpenAI** (GPT-4o, Codex)
- **Ollama** (local models, fully offline)

!!! tip "Ready to start?"
    Jump straight to [Installation](installation.md) or, if you prefer the shortest path, head to the [Quick Start](quickstart.md).
