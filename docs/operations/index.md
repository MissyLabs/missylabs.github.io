---
tags:
  - operations
---

# Operations

This section covers day-to-day operational tasks for running Missy in production: backups, monitoring, and troubleshooting.

## Key operational areas

| Area | What it covers |
|------|----------------|
| [Backup & Recovery](backup.md) | Config backups, rollback, diffing between versions |
| [Observability](observability.md) | Audit logging, OpenTelemetry, cost tracking |
| [Troubleshooting](troubleshooting.md) | Common issues and diagnostic steps |

## Important file locations

| Purpose | Path |
|---------|------|
| Config | `~/.missy/config.yaml` |
| Config backups | `~/.missy/config.d/` |
| Audit log | `~/.missy/audit.jsonl` |
| Memory DB | `~/.missy/memory.db` |
| Scheduler jobs | `~/.missy/jobs.json` |
| MCP config | `~/.missy/mcp.json` |
| Device registry | `~/.missy/devices.json` |
| Vault key | `~/.missy/secrets/vault.key` |
| Vault data | `~/.missy/secrets/vault.enc` |
| Prompt patches | `~/.missy/patches.json` |
| Workspace | `~/workspace` |

## Health check

Run the built-in health check to verify all subsystems:

```bash
missy doctor
```

This checks:

- Config file validity
- Provider availability and connectivity
- Policy engine status
- Memory database accessibility
- Vault status (if enabled)
- MCP server health

## Routine maintenance

### Memory cleanup

Remove old conversation history to keep the database lean:

```bash
# Delete turns older than 30 days
missy sessions cleanup --older-than 30

# Preview first
missy sessions cleanup --older-than 30 --dry-run
```

### Recovering from crashes

If Missy was interrupted during a task, check for incomplete checkpoints:

```bash
# List incomplete checkpoints
missy recover

# Abandon all incomplete sessions
missy recover --abandon-all
```
