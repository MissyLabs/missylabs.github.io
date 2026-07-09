---
tags:
  - cli
---

# Sandbox Commands

Inspect the Docker-based container-per-session sandbox (`ContainerSandbox`) used to isolate tool execution.

## missy sandbox status

Check Docker availability and show the current container sandbox configuration.

```bash
missy sandbox status
```

Prints a table with:

| Setting | Description |
|---------|-------------|
| `Docker` | Whether a working Docker installation was detected on this host |
| `enabled` | Whether `container.enabled` is set in config |
| `image` | Docker image used for session containers (default `python:3.12-slim`) |
| `memory_limit` | Docker `--memory` limit (default `256m`) |
| `cpu_quota` | Docker `--cpus` fraction (default `0.5`) |
| `network_mode` | Docker `--network` mode (default `none`, i.e. no networking inside the sandbox) |

If no `container:` block is present in config, `enabled` is reported as `false (not configured)`.

!!! note
    Enable the sandbox by adding a `container:` block to `~/.missy/config.yaml`:

    ```yaml
    container:
      enabled: true
      image: "python:3.12-slim"
      memory_limit: "256m"
      cpu_quota: 0.5
      network_mode: "none"
    ```

    Session containers are created per session, run without network access by default, and are subject to the configured memory/CPU limits.
