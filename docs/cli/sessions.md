---
tags:
  - cli
---

# Sessions Commands

Manage conversation history stored in the memory store (`~/.missy/memory.db`).

## missy sessions list

List recent sessions with their names and turn counts.

```bash
missy sessions list
missy sessions list --limit 50
```

| Option | Default | Description |
|--------|---------|-------------|
| `--limit` | `20` | Max sessions to show |

Shows a table with session ID, friendly name (if set), turn count, provider, channel, and last-updated timestamp.

## missy sessions rename

Set a friendly name for a session.

```bash
missy sessions rename SESSION_ID "my project"
```

`SESSION_ID` accepts either the full session ID or, if it is shorter than 32 characters and contains no hyphen, an existing friendly name to resolve from.

## missy sessions cleanup

Delete old conversation history from the memory store.

```bash
missy sessions cleanup
missy sessions cleanup --older-than 7
missy sessions cleanup --dry-run
```

| Option | Default | Description |
|--------|---------|-------------|
| `--older-than` | `30` | Delete turns older than N days |
| `--dry-run` | off | Show what would be deleted without deleting |

!!! warning
    Cleanup is irreversible. Use `--dry-run` first to confirm what would be removed.
