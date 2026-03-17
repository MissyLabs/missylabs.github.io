---
tags:
  - extending
  - plugins
---

# Plugins

Plugins are externally-loaded components that extend Missy's capabilities. Unlike tools (which are compiled into the codebase), plugins are loaded at runtime and gated behind a two-layer security check.

## Security model

Plugins are **disabled by default**. Loading a plugin requires both:

1. `plugins.enabled: true` in config.
2. The plugin's name in `plugins.allowed_plugins`.

Every load and execute attempt emits an audit event. Denied operations raise a `PolicyViolationError`.

```yaml
plugins:
  enabled: true
  allowed_plugins:
    - "weather"
    - "home-automation"
```

## Plugin permissions manifest

Every plugin must declare a `PluginPermissions` manifest listing all resources it needs:

```python
from missy.plugins.base import BasePlugin, PluginPermissions


class WeatherPlugin(BasePlugin):
    name = "weather"
    description = "Fetches weather data from an external API"
    version = "1.0.0"
    permissions = PluginPermissions(
        network=True,
        allowed_hosts=["api.openweathermap.org"],
    )

    def initialize(self) -> bool:
        # Validate API key, set up HTTP client, etc.
        # Return True on success, False on failure.
        return True

    def execute(self, *, location: str = "") -> dict:
        # Plugin logic here
        ...
```

### Permission fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `network` | bool | `false` | Outbound HTTP requests |
| `filesystem_read` | bool | `false` | Read from filesystem |
| `filesystem_write` | bool | `false` | Write to filesystem |
| `shell` | bool | `false` | Execute shell commands |
| `allowed_hosts` | list | `[]` | Specific hosts the plugin may contact |
| `allowed_paths` | list | `[]` | Specific filesystem paths the plugin may access |

## Creating a plugin

### Step 1: Define the class

```python
from missy.plugins.base import BasePlugin, PluginPermissions


class HomeAutomationPlugin(BasePlugin):
    name = "home-automation"
    description = "Controls smart home devices via Home Assistant"
    version = "0.1.0"
    permissions = PluginPermissions(
        network=True,
        allowed_hosts=["homeassistant.local:8123"],
    )

    def initialize(self) -> bool:
        """Called once when the plugin is loaded.

        Use this to validate configuration, set up clients,
        and verify connectivity. Return False to abort loading.
        """
        self.ha_url = "http://homeassistant.local:8123"
        self.ha_token = os.environ.get("HA_TOKEN")
        return self.ha_token is not None

    def execute(self, **kwargs) -> dict:
        """Called by the agent when using this plugin."""
        action = kwargs.get("action", "status")
        ...
```

### Step 2: Register

```python
from missy.plugins.loader import get_plugin_loader

loader = get_plugin_loader()
loader.load_plugin(HomeAutomationPlugin())
```

### Step 3: Enable in config

```yaml
plugins:
  enabled: true
  allowed_plugins:
    - "home-automation"

network:
  tool_allowed_hosts:
    - "homeassistant.local:8123"
```

## Plugin lifecycle

1. **Policy check** -- `PluginLoader` verifies `plugins.enabled` and `allowed_plugins`.
2. **Initialize** -- `plugin.initialize()` is called. Returns `True` on success.
3. **Register** -- Plugin is added to the internal registry with `enabled=True`.
4. **Execute** -- Agent can invoke the plugin. Each call emits an audit event.

If `initialize()` returns `False` or raises an exception, the plugin is not registered.

## Listing plugins

```bash
missy plugins
```

Shows all loaded plugins, their status, and version numbers.

## Plugins vs. tools vs. MCP

| Aspect | Tools | Plugins | MCP Servers |
|--------|-------|---------|-------------|
| Where they run | In-process | In-process | Separate process |
| Security gate | Permission declarations | Global enable + allowlist | Config file + digest pin |
| When to use | Core capabilities | Runtime extensions | External services |
| Audit granularity | Per-execution | Per-load + per-execution | Per-tool-call |
