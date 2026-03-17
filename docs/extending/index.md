---
tags:
  - extending
---

# Extending Missy

Missy provides three extension mechanisms, each with different security trade-offs:

| Mechanism | What it does | Security model | Config required |
|-----------|-------------|----------------|-----------------|
| [Tools](tools.md) | Add capabilities the agent can invoke | Permission declarations checked against policy engine | Registered in code |
| [Plugins](plugins.md) | Load external components at runtime | Disabled by default, explicit allowlist | `plugins:` section |
| [MCP Servers](mcp.md) | Connect to Model Context Protocol servers | Namespaced tools, digest pinning, injection scanning | `~/.missy/mcp.json` |

## How extensions fit into the architecture

```
AgentRuntime
    ├── ToolRegistry
    │       ├── Built-in tools (calculator, shell_exec, file_read, ...)
    │       └── Custom tools (your code)
    ├── PluginLoader
    │       └── Loaded plugins (weather, custom integrations, ...)
    └── McpManager
            ├── filesystem server → filesystem__read_file, filesystem__write_file
            └── postgres server → postgres__query, postgres__schema
```

All three extension types are subject to Missy's policy engine. A tool that declares `network: true` must pass the network policy check. A plugin that needs filesystem access must declare it in its manifest and the paths must be allowed in config.

## Quick comparison

**Use tools** when you want to add a capability that is tightly integrated with Missy's codebase. Tools are registered in Python code and have full access to the type system.

**Use plugins** when you want to load external code at runtime without modifying Missy itself. Plugins are gated behind a two-layer security check (global enable + allowlist).

**Use MCP servers** when you want to expose capabilities from an external process (a database, a file server, a custom API). MCP servers run as separate processes and communicate over stdio or HTTP.
