---
tags:
  - cli
  - reference
---

# CLI Reference

Missy's CLI is the primary interface for configuration, interaction, and operations. All commands share a common pattern:

```bash
missy [--config PATH] [--debug] COMMAND [OPTIONS]
```

## Global Options

| Option | Default | Description |
|--------|---------|-------------|
| `--config` | `~/.missy/config.yaml` | Path to config file (also `MISSY_CONFIG` env var) |
| `--debug` | off | Enable debug logging |

## Command Groups

| Group | Description |
|-------|-------------|
| [Core](core.md) | `init`, `setup`, `ask`, `run`, `doctor`, `providers` |
| [Schedule](schedule.md) | `schedule add/list/pause/resume/remove` |
| [Audit](audit.md) | `audit security`, `audit recent` |
| [Vault](vault.md) | `vault set/get/list/delete` |
| [Config](config.md) | `config plan/diff/rollback/backups` |
| [MCP](mcp.md) | `mcp list/add/remove/pin` |

Additional command groups: `discord`, `gateway`, `devices`, `voice`, `sessions`, `approvals`, `patches`, `presets`, `cost`, `recover`.
