---
tags:
  - cli
  - extending
---

# MCP Commands

Manage Model Context Protocol server connections.

## missy mcp list

```bash
missy mcp list
```

Shows connected servers with alive status and tool count.

## missy mcp add

```bash
missy mcp add filesystem --command "npx @modelcontextprotocol/server-filesystem /tmp"
missy mcp add postgres --url "http://localhost:3000"
```

| Option | Description |
|--------|-------------|
| `--command` | Stdio command to launch the server |
| `--url` | HTTP URL for an already-running server |

## missy mcp remove

```bash
missy mcp remove filesystem
```

## missy mcp pin

Pin the tool manifest digest for supply chain verification.

```bash
missy mcp pin filesystem
```

Computes a SHA-256 digest of the server's tool manifest and writes it to `~/.missy/mcp.json`. On future connections, a digest mismatch refuses to load the server and emits a `mcp.digest_mismatch` audit event.

!!! danger
    Without pinning, a compromised MCP server could silently change its tool definitions between connections. Always pin production MCP servers.
