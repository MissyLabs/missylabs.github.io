---
tags:
  - cli
---

# Core Commands

## missy init

Create the default `~/.missy/` directory structure and config file.

```bash
missy init
```

Creates:

- `~/.missy/config.yaml` — default secure config
- `~/.missy/audit.jsonl` — audit log
- `~/.missy/jobs.json` — scheduler store
- `~/.missy/secrets/` — vault directory (mode 700)
- `~/workspace/` — agent workspace

!!! note
    If a config file already exists, `init` skips it. Use `missy setup` for interactive configuration.

## missy setup

Interactive onboarding wizard or non-interactive config generation.

=== "Interactive"

    ```bash
    missy setup
    ```

    Walks through provider selection, API key entry, model tiers, and Discord setup.

=== "Non-interactive"

    ```bash
    missy setup \
      --provider anthropic \
      --api-key-env ANTHROPIC_API_KEY \
      --no-prompt
    ```

| Option | Description |
|--------|-------------|
| `--provider` | Provider name (`anthropic`, `openai`, `ollama`) |
| `--api-key` | Direct API key value |
| `--api-key-env` | Environment variable containing the API key |
| `--model` | Model identifier (defaults to provider's primary) |
| `--workspace` | Workspace directory path |
| `--no-prompt` | Non-interactive mode (requires `--provider`) |

## missy ask

Single-turn query.

```bash
missy ask "What is the capital of France?"
missy ask --provider ollama "Summarise this text: ..."
missy ask --mode safe-chat "Hello"
```

| Option | Default | Description |
|--------|---------|-------------|
| `--provider` | first configured | Provider to use |
| `--session` | none | Session ID for continuity |
| `--mode` | `full` | `full`, `safe-chat`, or `no-tools` |

## missy run

Interactive REPL session.

```bash
missy run
missy run --provider ollama --mode safe-chat
```

Type `quit`, `exit`, or press ++ctrl+d++ to end the session.

| Option | Default | Description |
|--------|---------|-------------|
| `--provider` | first configured | Provider to use |
| `--session` | `default` | Session ID for persistence |
| `--mode` | `full` | Capability mode |

## missy providers

List or switch providers.

```bash
missy providers              # list all providers
missy providers list         # same as above
missy providers switch ollama  # change active provider
```

## missy doctor

System health check.

```bash
missy doctor
```

Verifies config loading, provider availability, scheduler state, MCP connections, and voice channel readiness.

## missy presets list

Show all built-in network policy presets.

```bash
missy presets list
```

Displays a table of preset names with their hosts, domains, and CIDRs.
