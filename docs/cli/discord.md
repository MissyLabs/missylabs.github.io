---
tags:
  - cli
---

# Discord Commands

CLI commands for inspecting and operating the Discord channel: connection status, token probing, slash-command registration, and audit history. For end-to-end setup instructions (bot creation, token configuration, network policy, invite flow), see [Discord setup](../channels/discord/setup.md); for DM/guild/role access control, see [Discord access control](../channels/discord/access-control.md).

## missy discord status

Show a summary of configured Discord accounts.

```bash
missy discord status
```

Reads the Discord configuration from the Missy config file and prints whether the integration is enabled, followed by a table listing each account's token environment variable, application ID, DM policy, number of guild policies configured, and whether bot messages are ignored.

## missy discord probe

Test Discord API connectivity and bot token validity.

```bash
missy discord probe
```

For each configured account, calls `GET /users/@me` through the policy-enforced HTTP gateway and reports whether the token is valid and the request was allowed by network policy. Prints the connected bot username and ID on success, or an error if the token env var is unset or the call fails.

## missy discord register-commands

Register the built-in slash commands (`/ask`, `/status`, `/model`, `/help`) with Discord.

```bash
# Guild-scoped (registers instantly, recommended for testing)
missy discord register-commands --guild-id YOUR_GUILD_ID

# Global (omit --guild-id; propagates to all guilds, can take up to ~1 hour)
missy discord register-commands
```

| Option | Default | Description |
|--------|---------|-------------|
| `--guild-id` | none | Register commands scoped to this guild instead of globally |
| `--account-index` | `0` | Index into the configured `discord.accounts` list |

Fails with an error if the account's token env var is unset or `application_id` is not configured.

## missy discord audit

Show recent Discord-related audit events.

```bash
missy discord audit
missy discord audit --limit 20
```

| Option | Default | Description |
|--------|---------|-------------|
| `--limit` | `50` | Maximum number of events to show |

Filters the audit log (`~/.missy/audit.jsonl`) to events whose `event_type` starts with `discord.` and prints a table of timestamp, event type, result (`allow`/`deny`/`error`), and a truncated detail payload.
