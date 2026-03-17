---
tags:
  - cli
  - operations
---

# Config Commands

Manage config backups, diffs, and rollback.

## missy config backups

List all config backups stored in `~/.missy/config.d/`.

```bash
missy config backups
```

Up to 5 backups are retained (oldest pruned automatically).

## missy config diff

Show unified diff between current config and the latest backup.

```bash
missy config diff
```

## missy config plan

Preview what changed since the last backup.

```bash
missy config plan
```

## missy config rollback

Restore config from the latest backup. The current config is backed up first.

```bash
missy config rollback
```

!!! tip
    Backups are created automatically whenever the setup wizard or config migration overwrites the config file.
