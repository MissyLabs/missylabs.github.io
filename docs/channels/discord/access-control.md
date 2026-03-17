---
tags:
  - channels
  - discord
  - security
---

# Discord Access Control

Missy enforces fine-grained access control on Discord interactions at two levels: **DM policies** (direct messages) and **guild policies** (server messages).

## DM policies

The `dm_policy` field controls how the bot handles direct messages. It applies per-account.

### disabled (default)

The bot ignores all direct messages entirely.

```yaml
discord:
  accounts:
    - dm_policy: disabled
```

### allowlist

Only users whose Discord user IDs appear in `dm_allowlist` can send DMs to the bot.

```yaml
discord:
  accounts:
    - dm_policy: allowlist
      dm_allowlist:
        - "123456789012345678"
        - "987654321098765432"
```

### pairing

Users must complete a pairing handshake before the bot responds to their DMs. This is useful for teams where you want to vet users before granting access.

```yaml
discord:
  accounts:
    - dm_policy: pairing
```

### open

Any user can send DMs without restriction. Use this only in trusted environments.

```yaml
discord:
  accounts:
    - dm_policy: open
```

!!! warning "Production recommendation"
    Use `allowlist` or `pairing` in production. The `open` policy lets any Discord user interact with your Missy instance.

## Guild policies

Guild policies control bot behavior per-server. Each guild (identified by its ID) gets its own policy.

```yaml
discord:
  accounts:
    - guild_policies:
        "111222333444555666":
          enabled: true
          require_mention: true
          allowed_channels: ["bot-commands", "general"]
          allowed_roles: ["Developers", "Admins"]
          allowed_users: []
          mode: full
```

### Policy fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | bool | `true` | Master switch for this guild |
| `require_mention` | bool | `false` | Bot only responds when @-mentioned |
| `allowed_channels` | list | `[]` | Channel names where bot responds (empty = all) |
| `allowed_roles` | list | `[]` | Roles required to interact (empty = all) |
| `allowed_users` | list | `[]` | User IDs permitted to interact (empty = all) |
| `mode` | string | `full` | Capability mode (see below) |

### Capability modes

| Mode | Description |
|------|-------------|
| `full` | All capabilities enabled (tools, shell, etc. -- subject to main policy) |
| `no_tools` | Chat only, no tool invocations |
| `safe_chat_only` | Chat only, no tools, no context from previous sessions |

### Channel filtering

When `allowed_channels` is empty, the bot responds in all channels. When populated, only messages in the listed channels are processed:

```yaml
guild_policies:
  "111222333444555666":
    allowed_channels: ["bot-commands"]
    require_mention: false
```

!!! tip "Combine require_mention with channel filtering"
    For busy servers, set `require_mention: true` in general channels and create a dedicated `bot-commands` channel with `require_mention: false`.

### Role-based access

When `allowed_roles` is populated, users must hold at least one of the listed roles:

```yaml
guild_policies:
  "111222333444555666":
    allowed_roles: ["AI-Users", "Admins"]
```

Users without a matching role are silently ignored.

### Bot-to-bot interaction

By default, messages from other bots are ignored (`ignore_bots: true`). To allow another bot to interact with Missy:

```yaml
discord:
  accounts:
    - ignore_bots: true
      allow_bots_if_mention_only: true
```

With `allow_bots_if_mention_only: true`, bot messages that explicitly @-mention Missy are processed even when `ignore_bots` is `true`.

## Audit trail

All Discord access control decisions are logged. View them with:

```bash
missy discord audit --limit 20
```

Events include:

- `discord.dm.denied` -- DM rejected by policy
- `discord.guild.denied` -- Guild message rejected by policy
- `discord.role.denied` -- User lacks required role
- `discord.channel.denied` -- Message in disallowed channel
