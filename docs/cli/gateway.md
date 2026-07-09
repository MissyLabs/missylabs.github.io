---
tags:
  - cli
---

# Gateway Commands

Run Missy as a long-running service that processes tasks from all configured channels (Discord, CLI, proactive triggers) until stopped.

## missy gateway start

Start Missy in service mode.

```bash
missy gateway start
missy gateway start --host 0.0.0.0 --port 9000
```

| Option | Default | Description |
|--------|---------|-------------|
| `--host` | `127.0.0.1` | Bind address |
| `--port` | `8765` | Bind port |

The gateway runs the agent loop continuously, dispatching work from every enabled channel. If `proactive` triggers are configured (file-change watchers, disk/load thresholds, schedules), the gateway starts a `ProactiveManager` alongside the channel loop. Press ++ctrl+c++ to stop — the gateway shuts down proactive triggers and channels cleanly on `SIGINT`/`SIGTERM`.

!!! note
    `--host`/`--port` describe the gateway's own bind address, not the Discord or voice channel ports — those are configured separately (see [Voice Commands](voice.md) and [Discord setup](../channels/discord/setup.md)).

## missy gateway status

Show gateway configuration and channel status without starting the service.

```bash
missy gateway status
```

Prints a table of configured channels (Discord accounts and their token/DM-policy state, plus the always-available CLI channel) and the list of registered providers.

!!! note
    This command only reports configuration — it does not check whether a `missy gateway start` process is currently running.
