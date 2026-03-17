# Network Presets

Network presets are named shorthand that expand to predefined `allowed_hosts`, `allowed_domains`, and `allowed_cidrs` entries for common services. They are the easiest way to configure network access.

## Using Presets

Add preset names to the `network.presets` list:

```yaml
network:
  default_deny: true
  presets:
    - anthropic
    - github
    - ollama
```

Preset entries are **merged** with any explicit allow-list entries you define. Duplicates are automatically deduplicated.

## Built-in Presets

### anthropic

Allows access to the Anthropic API.

| Type | Entries |
|---|---|
| Hosts | `api.anthropic.com` |
| Domains | `anthropic.com` |
| CIDRs | -- |

### openai

Allows access to the OpenAI API and authentication endpoints.

| Type | Entries |
|---|---|
| Hosts | `api.openai.com`, `auth.openai.com`, `chatgpt.com` |
| Domains | `openai.com` |
| CIDRs | -- |

### ollama

Allows access to a local Ollama instance on the default port.

| Type | Entries |
|---|---|
| Hosts | `localhost:11434`, `127.0.0.1:11434` |
| Domains | -- |
| CIDRs | `127.0.0.0/8` |

### github

Allows access to GitHub API and content delivery.

| Type | Entries |
|---|---|
| Hosts | `api.github.com`, `github.com` |
| Domains | `github.com`, `githubusercontent.com` |
| CIDRs | -- |

### discord

Allows access to Discord API and WebSocket gateway.

| Type | Entries |
|---|---|
| Hosts | `discord.com`, `gateway.discord.gg` |
| Domains | `discord.com`, `discord.gg`, `discordapp.com` |
| CIDRs | -- |

### home-assistant

Allows access to a local Home Assistant instance on the default port.

| Type | Entries |
|---|---|
| Hosts | `localhost:8123`, `127.0.0.1:8123` |
| Domains | -- |
| CIDRs | `127.0.0.0/8` |

### pypi

Allows access to the Python Package Index for package downloads.

| Type | Entries |
|---|---|
| Hosts | `pypi.org`, `files.pythonhosted.org` |
| Domains | `pypi.org`, `pythonhosted.org` |
| CIDRs | -- |

### npm

Allows access to the npm registry.

| Type | Entries |
|---|---|
| Hosts | `registry.npmjs.org` |
| Domains | `npmjs.org`, `npmjs.com` |
| CIDRs | -- |

### docker-hub

Allows access to Docker Hub for container image pulls.

| Type | Entries |
|---|---|
| Hosts | `registry-1.docker.io`, `auth.docker.io`, `index.docker.io` |
| Domains | `docker.io`, `docker.com` |
| CIDRs | -- |

### huggingface

Allows access to Hugging Face for model downloads.

| Type | Entries |
|---|---|
| Hosts | `huggingface.co`, `cdn-lfs.huggingface.co` |
| Domains | `huggingface.co` |
| CIDRs | -- |

## Preset Summary Table

| Preset | Primary Use | Local? |
|---|---|---|
| `anthropic` | Anthropic Claude API | No |
| `openai` | OpenAI GPT API | No |
| `ollama` | Local LLM inference | Yes |
| `github` | GitHub API, raw content | No |
| `discord` | Discord bot connectivity | No |
| `home-assistant` | Home automation API | Yes |
| `pypi` | Python package installs | No |
| `npm` | Node.js package installs | No |
| `docker-hub` | Container image pulls | No |
| `huggingface` | Model downloads | No |

## Combining Presets with Explicit Rules

Presets merge with explicit entries. Use presets for standard services and explicit rules for custom endpoints:

```yaml
network:
  default_deny: true
  presets:
    - anthropic
    - github
  allowed_hosts:
    - "my-internal-api.corp.local:8080"
  allowed_cidrs:
    - "10.0.0.0/8"
  tool_allowed_hosts:
    - "api.weather.com"
```

The effective allow list is the union of all preset expansions plus all explicit entries.

## Unknown Presets

If you specify a preset name that does not match any built-in preset, Missy logs a warning and ignores it:

```
WARNING: Unknown network presets (ignored): my-custom-preset
```

Your config continues to load normally with the remaining valid presets.

## Common Preset Combinations

=== "Anthropic + tools"
    ```yaml
    network:
      default_deny: true
      presets:
        - anthropic
        - github
        - pypi
    ```

=== "Multi-provider"
    ```yaml
    network:
      default_deny: true
      presets:
        - anthropic
        - openai
        - ollama
    ```

=== "Full development"
    ```yaml
    network:
      default_deny: true
      presets:
        - anthropic
        - github
        - pypi
        - npm
        - docker-hub
        - huggingface
    ```

=== "Discord bot"
    ```yaml
    network:
      default_deny: true
      presets:
        - anthropic
        - discord
    ```
