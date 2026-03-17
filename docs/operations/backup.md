---
tags:
  - operations
---

# Backup & Recovery

Missy automatically backs up configuration before changes and provides CLI commands for rollback and diffing.

## Automatic backups

Every time the config is modified through the setup wizard or config commands, Missy creates a timestamped backup in `~/.missy/config.d/`:

```
~/.missy/config.d/
  config.yaml.20260315_143022
  config.yaml.20260316_091545
  config.yaml.20260317_102030
```

Up to **5 backups** are retained. The oldest backup is pruned when a new one is created.

## Listing backups

```bash
missy config backups
```

Shows all available backups with timestamps.

## Comparing configs

View the differences between the current config and the latest backup:

```bash
missy config diff
```

This produces a unified diff showing what changed:

```diff
--- config.yaml.20260316_091545
+++ config.yaml (current)
@@ -12,7 +12,7 @@
 providers:
   anthropic:
     name: anthropic
-    model: "claude-sonnet-4-6"
+    model: "claude-opus-4-6"
     enabled: true
```

## Rolling back

Restore the latest backup:

```bash
missy config rollback
```

This:

1. Creates a backup of the **current** config first (so you can undo the rollback).
2. Copies the latest backup over `~/.missy/config.yaml`.
3. Returns the path to the restored backup file.

!!! tip "Rollback is safe"
    The current config is always backed up before a rollback, so you can never lose your current config by rolling back.

## Config planning

Preview what a config change would do:

```bash
missy config plan
```

This compares the current running config with the file on disk and shows what would change if the config were reloaded.

## Manual backup

You can also back up manually:

```bash
cp ~/.missy/config.yaml ~/.missy/config.yaml.manual-backup
```

## What to back up

For a complete Missy backup, preserve these files:

| File | Contains | Priority |
|------|----------|----------|
| `~/.missy/config.yaml` | All configuration | Critical |
| `~/.missy/secrets/vault.key` | Vault encryption key | Critical (if using vault) |
| `~/.missy/secrets/vault.enc` | Encrypted secrets | Critical (if using vault) |
| `~/.missy/devices.json` | Edge node registry | Important (if using voice) |
| `~/.missy/mcp.json` | MCP server config | Important (if using MCP) |
| `~/.missy/jobs.json` | Scheduled jobs | Important (if using scheduler) |
| `~/.missy/memory.db` | Conversation history | Nice to have |
| `~/.missy/patches.json` | Prompt patches | Nice to have |

!!! danger "Back up the vault key separately"
    If you lose `~/.missy/secrets/vault.key`, all encrypted secrets in `vault.enc` become unrecoverable. Store a copy of the vault key in a secure location outside the Missy data directory.

## Config migration

Missy tracks a `config_version` in the config file. When the config schema changes between versions, the migration system:

1. Detects the old version on startup.
2. Creates a backup automatically.
3. Rewrites the config to the new format.
4. The migration is idempotent -- running it on an already-migrated config is a no-op.

The current config version is **2**. Configs without a `config_version` field are treated as version 0.
