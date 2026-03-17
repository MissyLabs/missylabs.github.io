---
tags:
  - extending
  - mcp
---

# MCP Servers

The [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) lets Missy connect to external tool servers. Each MCP server exposes a set of tools that the agent can call, with tools automatically namespaced to prevent collisions.

## How it works

```
Missy Agent
    └── McpManager
            ├── filesystem server  →  filesystem__read_file
            │                         filesystem__write_file
            │                         filesystem__list_dir
            └── postgres server    →  postgres__query
                                      postgres__schema
```

MCP servers run as separate processes and communicate with Missy over stdio or HTTP. The `McpManager` handles lifecycle (connect, health check, auto-restart) and security (name validation, digest verification, injection scanning).

## Adding an MCP server

### Via CLI

```bash
# stdio-based server
missy mcp add filesystem --command "npx @modelcontextprotocol/server-filesystem /tmp"

# HTTP-based server
missy mcp add myapi --url "http://localhost:3000/mcp"
```

### Via config file

Edit `~/.missy/mcp.json` directly:

```json
[
  {
    "name": "filesystem",
    "command": "npx @modelcontextprotocol/server-filesystem /tmp"
  },
  {
    "name": "postgres",
    "command": "npx @modelcontextprotocol/server-postgres postgresql://localhost/mydb"
  }
]
```

## Tool namespacing

All MCP tools are namespaced as `server__tool` to prevent name collisions:

| Server name | Tool name | Namespaced name |
|-------------|-----------|-----------------|
| `filesystem` | `read_file` | `filesystem__read_file` |
| `postgres` | `query` | `postgres__query` |

The double underscore (`__`) is the separator. Server names must not contain `__`.

### Naming rules

Server names must match `^[a-zA-Z0-9_\-]+$` (alphanumeric, hyphens, underscores only). Names containing `__` are rejected to preserve the namespace separator.

## Digest pinning

MCP servers can change their tool manifests between restarts. Digest pinning detects unexpected changes:

### Pin a digest

```bash
# After connecting and verifying the tools look correct
missy mcp pin filesystem
```

This computes a hash of the server's tool manifest and stores it in `~/.missy/mcp.json`:

```json
[
  {
    "name": "filesystem",
    "command": "npx @modelcontextprotocol/server-filesystem /tmp",
    "digest": "sha256:a1b2c3d4..."
  }
]
```

### What happens on mismatch

If a pinned server's tool manifest changes, Missy:

1. Disconnects the server immediately.
2. Emits a `mcp.digest_mismatch` audit event with category `security`.
3. Raises an error preventing the server from loading.

!!! warning "Supply chain protection"
    Digest pinning protects against supply chain attacks where an MCP server package is updated to expose malicious tools. Always pin digests for production servers.

## Injection scanning

MCP tool results are scanned for prompt injection patterns before being returned to the agent. This uses the same `InputSanitizer` that protects user input.

| Mode | Behavior |
|------|----------|
| `block_injection: true` (default) | Injected content is blocked entirely |
| `block_injection: false` | Warning prepended to output, content passes through |

When injection is detected, the tool result is replaced with:

```
[MCP BLOCKED] Tool 'server__tool' output contained injection patterns and was blocked: [patterns]
```

## Health checks

The `McpManager` periodically checks if connected servers are still alive. Dead servers are automatically restarted:

```python
# Called internally by the agent runtime
mcp_manager.health_check()
```

Servers that fail to restart are logged at ERROR level.

## Managing MCP servers

```bash
# List all connected servers
missy mcp list

# Remove a server
missy mcp remove filesystem

# Pin a tool manifest digest
missy mcp pin filesystem
```

## Calling MCP tools

MCP tools are called by the agent using their namespaced names:

```
Agent: I'll read the file for you.
[Tool call: filesystem__read_file(path="/tmp/example.txt")]
```

Behind the scenes, `McpManager.call_tool("filesystem__read_file", {"path": "/tmp/example.txt"})` splits the namespaced name, looks up the server, and dispatches the call.

## Security properties

| Property | Implementation |
|----------|---------------|
| Config file permissions | `0600` (owner read/write only) |
| Config ownership check | Refuses to load if not owned by current user |
| World-writable check | Refuses to load if group/world-writable |
| Name validation | Alphanumeric + hyphens + underscores only |
| Tool output scanning | InputSanitizer checks for 13+ injection patterns |
| Digest verification | SHA-256 hash of tool manifest |
| Atomic config writes | Write to temp file, then rename |
