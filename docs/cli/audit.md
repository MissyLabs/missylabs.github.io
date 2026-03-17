---
tags:
  - cli
  - security
---

# Audit Commands

Query the structured JSONL audit log.

## missy audit security

Show recent policy violations and security events.

```bash
missy audit security
missy audit security --limit 100
```

## missy audit recent

Show recent audit events, optionally filtered by category.

```bash
missy audit recent
missy audit recent --limit 20 --category network
```

| Option | Default | Description |
|--------|---------|-------------|
| `--limit` | 50 | Maximum events to show |
| `--category` | all | Filter: `network`, `filesystem`, `shell`, `plugin`, `scheduler` |

## Event Categories

| Category | Events |
|----------|--------|
| `network` | HTTP requests, policy checks, DNS resolution |
| `filesystem` | File read/write, path validation |
| `shell` | Command execution, whitelist checks |
| `agent` | Tool calls, iterations, completions |
| `security` | Injection detection, secret redaction, MCP digest mismatches |
