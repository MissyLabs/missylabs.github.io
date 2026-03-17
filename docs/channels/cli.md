---
tags:
  - channels
  - cli
---

# CLI Channel

The CLI channel is Missy's primary interface. It supports single-shot queries and an interactive REPL with full tool calling, session management, and streaming responses.

## Single-shot queries

Use `missy ask` to send a one-off prompt and get a response:

```bash
missy ask "What is the capital of France?"
```

Options:

```bash
# Use a specific provider
missy ask "Summarize this file" --provider anthropic

# Resume a named session
missy ask "Continue where we left off" --session my-project
```

## Interactive REPL

Use `missy run` for a multi-turn conversation:

```bash
missy run
```

The REPL supports:

- **Multi-turn context** -- previous messages are retained within the token budget.
- **Tool calling** -- the agent can invoke tools (shell, file read/write, web fetch, etc.) if policy permits.
- **Streaming** -- responses stream token-by-token as they arrive.
- **Session persistence** -- conversations are stored in the memory database for later recall.

Options:

```bash
# Start with a specific provider
missy run --provider ollama
```

## Sessions

Sessions group conversation turns under a shared identifier. This lets you resume context across CLI invocations.

```bash
# Start a named session
missy ask "Let's work on the API refactor" --session api-refactor

# Continue later
missy ask "What did we decide about error handling?" --session api-refactor
```

Session data is stored in `~/.missy/memory.db` (SQLite with FTS5 search). Clean up old sessions with:

```bash
# Delete sessions older than 30 days
missy sessions cleanup --older-than 30

# Preview what would be deleted
missy sessions cleanup --older-than 30 --dry-run
```

## Capability modes

What the CLI can do depends entirely on your `~/.missy/config.yaml` policy:

| Capability | Config section | Default |
|-----------|----------------|---------|
| Network access | `network:` | Denied (default_deny: true) |
| File reads | `filesystem.allowed_read_paths` | None |
| File writes | `filesystem.allowed_write_paths` | None |
| Shell commands | `shell.enabled` + `shell.allowed_commands` | Disabled |
| Plugins | `plugins.enabled` + `plugins.allowed_plugins` | Disabled |

!!! warning "Secure by default"
    Missy ships with everything disabled. You must explicitly allow each capability. See the [Configuration Reference](../configuration/reference.md) for the full schema.

## Provider selection

The CLI uses the first available provider by default. Override per-command:

```bash
missy ask "Quick question" --provider ollama
missy run --provider anthropic
```

Check which providers are configured and reachable:

```bash
missy providers
```

## Useful companion commands

```bash
missy doctor          # System health check
missy skills          # List registered skills
missy plugins         # List plugins and status
missy cost            # Show cost tracking and budget
missy cost --session  # Per-session cost breakdown
```
