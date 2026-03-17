---
tags:
  - getting-started
  - quickstart
---

# Quick Start

Get from zero to a working Missy instance in under two minutes.

## 1. Install

```bash
git clone https://github.com/MissyLabs/missy.git
cd missy
pip install -e .
```

## 2. Run the setup wizard

```bash
missy setup
```

The wizard walks you through five steps:

1. **Workspace directory** -- where Missy reads and writes files (default: `~/workspace`)
2. **Provider selection** -- choose Anthropic, OpenAI, Ollama, or a combination
3. **API key entry** -- paste your key or let Missy detect it from the environment
4. **Model tiers** -- pick primary, fast, and premium models
5. **Write config** -- review and confirm

!!! tip "Have your API key ready"
    For Anthropic, get a key from [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys). For OpenAI, visit [platform.openai.com/api-keys](https://platform.openai.com/api-keys).

The wizard writes `~/.missy/config.yaml` with secure defaults: network deny-all, no shell access, no plugins. Only the provider host you selected is allowed through.

## 3. Ask your first question

```bash
missy ask "What can you help me with?"
```

Expected output:

```
╭─ Missy ──────────────────────────────────────────────────╮
│ I can help you with a wide range of tasks:               │
│                                                          │
│ • Answer questions and explain concepts                  │
│ • Read and analyze files in your workspace               │
│ • Search your codebase                                   │
│ • Schedule recurring tasks                               │
│ • Manage secrets in an encrypted vault                   │
│ • ...and more, depending on which capabilities you       │
│   enable in your config.                                 │
╰──────────────────────────────────────────────────────────╯
```

## 4. Start an interactive session

```bash
missy run
```

This opens a REPL where you can have a multi-turn conversation. Type `quit` or press ++ctrl+d++ to exit.

## 5. Verify your setup

```bash
missy doctor
```

This runs health checks on all subsystems -- config, providers, policy engine, memory, audit log -- and reports any issues.

## What just happened

After completing these steps, Missy has:

- Created `~/.missy/config.yaml` with your provider credentials and security policies
- Created `~/.missy/audit.jsonl` to log all actions
- Created `~/.missy/memory.db` for conversation history
- Created `~/workspace/` as the default working directory

!!! note "Security defaults"
    Out of the box, Missy operates with **all capabilities disabled**. It can answer questions and use its built-in tools, but it cannot execute shell commands, write to arbitrary paths, or make network requests outside the configured provider. Enable additional capabilities in `~/.missy/config.yaml` as needed.

## Next steps

- [Setup Wizard reference](setup-wizard.md) -- all wizard options including non-interactive mode
- [Your First Conversation](first-conversation.md) -- interactive REPL, tools, and sessions
- [Configuration Reference](../configuration/reference.md) -- full config.yaml schema
