---
tags:
  - getting-started
  - tutorial
---

# Your First Conversation

This page walks through Missy's interactive mode, capability modes, tool usage, and session management.

## Starting the REPL

Launch an interactive session:

```bash
missy run
```

You will see a session banner:

```
╭──────────────────────────────────────────────────╮
│ Missy interactive session                        │
│                                                  │
│ Provider : anthropic                             │
│ Mode     : full (all tools)                      │
│ Type quit or exit to end, or press Ctrl-D.       │
╰──────────────────────────────────────────────────╯
```

Type a message at the `You>` prompt and press Enter. Missy responds in the same terminal.

```
You> What time is it?
╭─ Missy ──────────────────────────────────────────╮
│ The current time is 2026-03-17 14:32:07 UTC.     │
╰──────────────────────────────────────────────────╯
You>
```

To exit, type `quit`, `exit`, or press ++ctrl+d++.

## Single-shot mode

For one-off questions without entering the REPL, use `missy ask`:

```bash
missy ask "Summarize the LICENSE file in ~/workspace/myproject"
```

The `ask` command prints the response and exits immediately. Useful for scripts and pipelines.

## Capability modes

Missy supports three capability modes that control which tools are available during a conversation:

=== "full"

    All registered tools are available. This is the default.

    ```bash
    missy run --mode full
    ```

    Tools include file reading/writing, web search, code execution (if shell is enabled in config), memory operations, and any MCP server tools.

=== "safe-chat"

    Only read-only tools are available. Missy can read files and search, but cannot write files, execute commands, or modify state.

    ```bash
    missy run --mode safe-chat
    ```

    !!! tip
        Use `safe-chat` when you want Missy to answer questions about your codebase without risk of accidental changes.

=== "no-tools"

    Pure conversational mode. No tools are available -- Missy responds using only its training data and conversation context.

    ```bash
    missy run --mode no-tools
    ```

The same `--mode` flag works with `missy ask`:

```bash
missy ask --mode safe-chat "What does the main() function do in app.py?"
```

## Working with tools

In `full` mode, Missy automatically invokes tools when needed. You do not need to call tools explicitly -- describe what you want, and Missy selects the appropriate tool.

```
You> Read the first 10 lines of ~/workspace/myproject/README.md
╭─ Missy ──────────────────────────────────────────╮
│ Here are the first 10 lines of README.md:        │
│                                                   │
│ # My Project                                      │
│ A sample application for demonstrating Missy.     │
│ ...                                               │
╰──────────────────────────────────────────────────╯
```

To see which tools are registered:

```bash
missy skills
```

!!! note "Policy enforcement"
    Even in `full` mode, all tool actions are checked against the policy engine. A file write outside `allowed_write_paths` is denied. A network request to a host not in the allowlist is blocked. Tool availability does not bypass security policy.

## Sessions

Sessions allow Missy to maintain conversation context across multiple interactions.

### Default session

By default, `missy run` uses a session named `default`. This means conversation history persists between runs:

```bash
missy run              # First session — ask some questions
# ... exit and come back later ...
missy run              # Same "default" session — Missy remembers context
```

### Named sessions

Create separate sessions for different tasks:

```bash
missy run --session project-alpha
missy run --session debugging-issue-42
```

The `--session` flag also works with `missy ask`:

```bash
missy ask --session project-alpha "What did we decide about the database schema?"
```

### Managing sessions

List recent sessions:

```bash
missy sessions list
```

Clean up old conversation history:

```bash
missy sessions cleanup --older-than 30         # delete sessions older than 30 days
missy sessions cleanup --older-than 7 --dry-run # preview what would be deleted
```

## Choosing a provider

Override the default provider for a single session:

```bash
missy run --provider ollama
missy ask --provider openai "Translate this to French: hello world"
```

List configured providers and their status:

```bash
missy providers
```

!!! tip "Runtime switching"
    Inside a REPL session, the provider is fixed for the duration of that session. To switch providers, exit and start a new session with `--provider`.

## Recovery

If a session is interrupted (network failure, crash, ++ctrl+c++ during a long operation), Missy can detect incomplete tasks on the next run:

```
Found 1 resumable task(s) from previous sessions.
  • session=default — Analyze the test coverage for the auth module...
```

To view and manage incomplete tasks:

```bash
missy recover                    # list incomplete checkpoints
missy recover --abandon-all      # discard all incomplete checkpoints
```

## Next steps

- [Configuration Reference](../configuration/reference.md) -- enable shell access, add network allowlist entries, configure filesystem policies
- [Security overview](../security/index.md) -- understand the policy engine, input sanitization, and audit logging
- [Extending Missy](../extending/index.md) -- add custom tools, plugins, and MCP servers
