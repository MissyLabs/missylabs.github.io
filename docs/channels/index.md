---
tags:
  - channels
---

# Channels

Channels are the communication interfaces through which users interact with Missy. Each channel connects to the same AgentRuntime, so the same policies, tools, and provider configuration apply regardless of how you reach the agent.

## Available channels

| Channel | Transport | Use case | Requires setup |
|---------|-----------|----------|----------------|
| [CLI](cli.md) | stdin/stdout | Interactive terminal sessions, scripts | None (built-in) |
| [Discord](discord/setup.md) | WebSocket Gateway API | Team collaboration, remote access | Bot token + application ID |
| [Voice](voice/server.md) | WebSocket + binary audio | Hands-free via Raspberry Pi edge nodes | STT/TTS engines + edge hardware |
| [Webhook](webhook.md) | HTTP POST | CI/CD pipelines, external integrations | Network policy for inbound port |
| [Screencast](screencast.md) | HTTP + WebSocket | Browser-based screen capture and analysis | Token auth |
| [REST API](../operations/rest-api.md) | HTTP REST | Agent-as-a-Service programmatic access | API key |
| [Operator Console](../operations/operator-console.md) | HTTP + Server-Sent Events | Browser dashboard: chat with live run streaming, providers/tools/sessions status, scheduler jobs, memory browser, audit trail | API key (used as the login password) |

!!! tip "Just want a web UI? Start here"
    `missy api start` launches both the REST API *and* a full browser dashboard at `http://127.0.0.1:8080/` — no separate install or config needed. Sign in with your API key and you get a live chat panel plus admin views for everything else. See [Operator Console](../operations/operator-console.md).

## How channels work

All channels follow the same pattern:

1. **Receive input** -- text from the user (or transcribed audio).
2. **Forward to AgentRuntime** -- the runtime applies policy checks, selects a provider, runs tools if needed, and returns a response.
3. **Deliver output** -- text back to the user (or synthesized audio).

```
User ──► Channel ──► AgentRuntime ──► Provider (Claude / GPT / Ollama)
                          │
                     PolicyEngine
                     ToolRegistry
                     Memory / Learnings
```

## Choosing a channel

**Start with the CLI.** It requires no additional configuration and gives you full access to every Missy feature. Add Discord or Voice channels when you need remote or hands-free access.

!!! tip "Multiple channels at once"
    Missy can serve multiple channels simultaneously. A typical setup runs the CLI for admin tasks while Discord and Voice handle day-to-day queries.

## Channel-specific capabilities

| Feature | CLI | Discord | Voice | Webhook | Screencast | API | Console |
|---------|-----|---------|-------|---------|------------|-----|---------|
| Interactive REPL | Yes | -- | -- | -- | -- | -- | Yes (browser chat panel) |
| Single-shot queries | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| Tool calling | Yes | Configurable per guild | Per-node policy | Yes | Yes | Yes | Yes |
| Session persistence | Yes | Per-user/channel | Per-node | Per-request | Per-session | Per-request | Per-request |
| Streaming responses | Yes | -- | Audio streaming | -- | -- | -- | Yes (SSE run events) |
| Access control | OS user | DM policy + guild roles | Device pairing + policy modes | Network policy | Token auth | API key | API key (cookie session) |

!!! note "Console vs. REST API"
    The Operator Console and REST API are the same server (`missy api start`) exposing two faces of the same thing: the console is the browser UI, the REST API is what the console (or your own scripts) call underneath. You don't run them separately.
