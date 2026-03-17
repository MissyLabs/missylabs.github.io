---
tags:
  - channels
  - discord
---

# Discord Setup

Missy's Discord channel implements the full WebSocket Gateway API with slash commands (`/ask`, `/status`, `/model`, `/help`), access control, and audit logging.

## Prerequisites

1. A Discord application and bot token from the [Discord Developer Portal](https://discord.com/developers/applications).
2. The bot's **application ID** (found on the General Information page).
3. Network policy allowing Discord hosts.

## Step 1: Create a Discord bot

1. Go to [discord.com/developers/applications](https://discord.com/developers/applications).
2. Click **New Application**, give it a name.
3. Go to **Bot** in the sidebar, click **Reset Token**, and copy the token.
4. Under **Privileged Gateway Intents**, enable **Message Content Intent**.
5. Note the **Application ID** from the General Information page.

## Step 2: Configure Missy

Add the `discord:` section to `~/.missy/config.yaml`:

```yaml
discord:
  enabled: true
  accounts:
    - token_env_var: DISCORD_BOT_TOKEN
      application_id: "YOUR_APPLICATION_ID"
      dm_policy: disabled          # or: pairing, allowlist, open
      ignore_bots: true
      guild_policies:
        "YOUR_GUILD_ID":
          enabled: true
          require_mention: true
          allowed_channels: ["bot-commands"]
          mode: full               # full | no_tools | safe_chat_only
```

!!! tip "Never put the bot token in config.yaml"
    Store the token in an environment variable or the Missy vault:

    ```bash
    # Option 1: Environment variable
    export DISCORD_BOT_TOKEN="your-token-here"

    # Option 2: Missy vault
    missy vault set discord_bot_token "your-token-here"
    ```

    For vault storage, reference it in config:

    ```yaml
    discord:
      accounts:
        - token: "vault://discord_bot_token"
    ```

## Step 3: Allow Discord network hosts

Discord requires outbound access to several hosts. Add them to your network policy:

```yaml
network:
  discord_allowed_hosts:
    - "discord.com"
    - "gateway.discord.gg"
    - "cdn.discordapp.com"
```

Or use the `discord` preset if your config uses presets:

```yaml
network:
  presets:
    - discord
```

## Step 4: Test connectivity

```bash
missy discord probe
```

This verifies the bot token is valid and Discord's Gateway is reachable. Expected output:

```
Discord probe: OK
  Bot user: Missy#1234
  Gateway: wss://gateway.discord.gg
```

## Step 5: Register slash commands

Register slash commands for a specific guild (instant) or globally (up to 1 hour propagation):

```bash
# Guild-specific (recommended for testing)
missy discord register-commands --guild-id YOUR_GUILD_ID

# Global (all guilds the bot is in)
missy discord register-commands --global
```

## Step 6: Invite the bot

Generate an invite URL with the required permissions:

1. Go to **OAuth2 > URL Generator** in the Developer Portal.
2. Select scopes: `bot`, `applications.commands`.
3. Select permissions: `Send Messages`, `Read Message History`, `Use Slash Commands`, `Add Reactions`.
4. Use the generated URL to invite the bot to your server.

## Checking status

```bash
missy discord status    # Show configuration summary
missy discord audit     # Show Discord-specific audit events
```

## Configuration reference

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | bool | `false` | Master switch for Discord channel |
| `accounts[].token_env_var` | string | `DISCORD_BOT_TOKEN` | Env var containing the bot token |
| `accounts[].token` | string | null | Direct token (supports `vault://` refs) |
| `accounts[].application_id` | string | required | Discord application ID |
| `accounts[].dm_policy` | string | `disabled` | DM handling: `disabled`, `allowlist`, `pairing`, `open` |
| `accounts[].dm_allowlist` | list | `[]` | User IDs for allowlist DM policy |
| `accounts[].ignore_bots` | bool | `true` | Ignore messages from other bots |
| `accounts[].ack_reaction` | string | `""` | Emoji to react with on message receipt |
| `accounts[].auto_thread_threshold` | int | `0` | Create thread after N messages (0 = disabled) |
