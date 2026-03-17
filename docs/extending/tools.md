---
tags:
  - extending
  - tools
---

# Tools

Tools are discrete capabilities that the agent can invoke during a conversation. Each tool declares what permissions it needs, and the `ToolRegistry` validates those permissions against the policy engine before every execution.

## Built-in tools

Missy ships with these built-in tools:

| Tool | Description | Permissions needed |
|------|-------------|-------------------|
| `calculator` | Evaluate mathematical expressions | None |
| `shell_exec` | Execute shell commands | `shell: true` |
| `file_read` | Read file contents | `filesystem_read: true` |
| `file_write` | Write to files | `filesystem_write: true` |
| `file_delete` | Delete files | `filesystem_write: true` |
| `list_files` | List directory contents | `filesystem_read: true` |
| `web_fetch` | Fetch a URL | `network: true` |
| `memory_tools` | Search and manage conversation memory | None |
| `tts_speak` | Text-to-speech synthesis | None |
| `discord_upload` | Upload files to Discord channels | `network: true` |
| `browser_tools` | Browser automation | `network: true`, `shell: true` |
| `x11_tools` | X11 window management | `shell: true` |
| `x11_launch` | Launch X11 applications | `shell: true` |
| `atspi_tools` | AT-SPI accessibility automation | `shell: true` |
| `incus_tools` | Incus container management | `shell: true` |
| `code_evolve` | Code generation and evolution | `filesystem_write: true` |
| `self_create_tool` | Dynamically create new tools | `filesystem_write: true` |

!!! note "Policy enforcement"
    Tools only work if the required permissions are granted in your config. For example, `shell_exec` requires `shell.enabled: true` and the specific command in `shell.allowed_commands`.

## Creating a custom tool

### Step 1: Define the tool class

Create a new file in `missy/tools/builtin/` or your own package:

```python
from missy.tools.base import BaseTool, ToolPermissions, ToolResult


class WeatherTool(BaseTool):
    name = "weather"
    description = "Get the current weather for a location"
    permissions = ToolPermissions(
        network=True,
        allowed_hosts=["api.openweathermap.org"],
    )
    parameters = {
        "location": {
            "type": "string",
            "description": "City name or coordinates",
            "required": True,
        },
    }

    def execute(self, *, location: str) -> ToolResult:
        # Your implementation here
        try:
            weather_data = self._fetch_weather(location)
            return ToolResult(success=True, output=weather_data)
        except Exception as e:
            return ToolResult(success=False, output=None, error=str(e))

    def _fetch_weather(self, location: str) -> str:
        ...
```

### Step 2: Register the tool

Tools are registered with the `ToolRegistry`:

```python
from missy.tools.registry import get_tool_registry

registry = get_tool_registry()
registry.register(WeatherTool())
```

### Step 3: Grant network access

Since the weather tool makes HTTP requests, add the host to your network policy:

```yaml
network:
  tool_allowed_hosts:
    - "api.openweathermap.org"
```

## Tool permissions

The `ToolPermissions` dataclass declares what a tool needs:

```python
@dataclass
class ToolPermissions:
    network: bool = False           # Outbound HTTP
    filesystem_read: bool = False   # Read files
    filesystem_write: bool = False  # Write files
    shell: bool = False             # Execute commands
    allowed_paths: list[str] = []   # Specific paths (read or write)
    allowed_hosts: list[str] = []   # Specific hosts (network)
```

The registry checks these against the policy engine before every `execute()` call:

1. **Network** -- each host in `allowed_hosts` is checked against network policy.
2. **Filesystem read** -- declared paths and the actual path from kwargs are checked.
3. **Filesystem write** -- same as read, against write policy.
4. **Shell** -- the actual command string is checked against `shell.allowed_commands`.

If any check fails, the tool returns a `ToolResult(success=False)` with the policy violation message instead of raising an exception.

## Tool schema

The `get_schema()` method returns a JSON Schema dict that providers use for function calling:

```python
class MyTool(BaseTool):
    parameters = {
        "query": {
            "type": "string",
            "description": "Search query",
            "required": True,
        },
        "limit": {
            "type": "integer",
            "description": "Max results",
        },
    }
```

This generates:

```json
{
  "name": "my_tool",
  "description": "...",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {"type": "string", "description": "Search query"},
      "limit": {"type": "integer", "description": "Max results"}
    },
    "required": ["query"]
  }
}
```

## Audit trail

Every tool execution emits an audit event with the tool name, session ID, and result (`allow`, `deny`, or `error`). View tool events with:

```bash
missy audit recent --category plugin
```
