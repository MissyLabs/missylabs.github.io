# Configuration Reference

Complete annotated reference for `~/.missy/config.yaml`. Every field, its type, and its default value are documented below.

## Full Config File

```yaml
# Schema version stamp. Missy auto-migrates older configs on load.
# 0 = pre-migration (legacy), current = 2.
config_version: 2

# ---------------------------------------------------------------------------
# Network policy
# ---------------------------------------------------------------------------
network:
  # When true (default), ALL outbound HTTP is blocked unless the
  # destination matches an allow rule below.
  default_deny: true                    # bool, default: true

  # Named presets that expand to hosts/domains/CIDRs for common services.
  # See the Presets page for the full list.
  presets:                              # list[str], default: []
    - anthropic
    - github

  # CIDR blocks that are always reachable (e.g., local subnets).
  allowed_cidrs:                        # list[str], default: []
    - "127.0.0.0/8"
    - "10.0.0.0/8"

  # Fully-qualified domain names or suffix patterns.
  # "github.com" = exact match only.
  # Domain suffix matching is applied: "anthropic.com" matches
  # "api.anthropic.com" when resolved through the preset system.
  allowed_domains:                      # list[str], default: []
    - "anthropic.com"

  # Explicit host or host:port pairs.
  allowed_hosts:                        # list[str], default: []
    - "api.anthropic.com"

  # Per-category host overrides. These are merged (unioned) with
  # allowed_hosts when the request category matches.
  provider_allowed_hosts: []            # list[str], default: []
  tool_allowed_hosts: []                # list[str], default: []
  discord_allowed_hosts: []             # list[str], default: []

  # L7 REST policies: per-host HTTP method + path rules.
  # Evaluated after host-level network policy, before dispatch.
  # First matching rule wins; no match = fall through to network policy.
  rest_policies:                        # list[dict], default: []
    - host: "api.github.com"
      method: "GET"                     # HTTP method or "*" for any
      path: "/repos/**"                 # fnmatch glob pattern
      action: "allow"                   # "allow" or "deny"
    - host: "api.github.com"
      method: "DELETE"
      path: "/**"
      action: "deny"

# ---------------------------------------------------------------------------
# Filesystem policy
# ---------------------------------------------------------------------------
filesystem:
  # Absolute paths the agent may read from. Symlinks are resolved
  # before comparison. Empty list = no reads allowed.
  allowed_read_paths:                   # list[str], default: []
    - "/home/user/workspace"
    - "/home/user/documents"

  # Absolute paths the agent may write to. Symlinks are resolved
  # before comparison. Empty list = no writes allowed.
  allowed_write_paths:                  # list[str], default: []
    - "/home/user/workspace/output"

# ---------------------------------------------------------------------------
# Shell policy
# ---------------------------------------------------------------------------
shell:
  # Master switch. When false, ALL shell execution is blocked
  # regardless of allowed_commands.
  enabled: false                        # bool, default: false

  # Whitelist of command names allowed when enabled=true.
  # Empty list with enabled=true means ALL commands are allowed
  # (unrestricted shell).
  # Both bare names ("git") and path-qualified names work.
  allowed_commands:                     # list[str], default: []
    - git
    - ls
    - cat
    - grep
    - find
    - python3

# ---------------------------------------------------------------------------
# Plugin policy
# ---------------------------------------------------------------------------
plugins:
  # Master switch. When false, no plugins may be loaded.
  enabled: false                        # bool, default: false

  # Whitelist of plugin identifiers allowed when enabled=true.
  allowed_plugins: []                   # list[str], default: []

# ---------------------------------------------------------------------------
# Providers
# ---------------------------------------------------------------------------
providers:
  anthropic:
    name: anthropic                     # str, logical provider name
    model: "claude-sonnet-4-6"        # str, required — model identifier
    api_key: null                       # str|null — falls back to ANTHROPIC_API_KEY env var
    api_keys: []                        # list[str] — multiple keys for rotation
    base_url: null                      # str|null — API endpoint override
    timeout: 30                         # int, default: 30 — request timeout in seconds
    enabled: true                       # bool, default: true
    fast_model: "claude-haiku-4-5"    # str — model for fast/simple tier
    premium_model: "claude-opus-4-6"  # str — model for premium/complex tier

  openai:
    name: openai
    model: "gpt-4o"
    api_key: null                       # falls back to OPENAI_API_KEY env var
    timeout: 30
    enabled: true

  ollama:
    name: ollama
    model: "llama3.2"
    base_url: "http://localhost:11434"  # Ollama's default endpoint
    timeout: 120                        # local models may be slower
    enabled: true

# ---------------------------------------------------------------------------
# Scheduling
# ---------------------------------------------------------------------------
scheduling:
  enabled: true                         # bool, default: true
  max_jobs: 0                           # int, default: 0 (0 = unlimited)
  active_hours: ""                      # str, e.g. "08:00-22:00" (empty = always)

# ---------------------------------------------------------------------------
# Heartbeat
# ---------------------------------------------------------------------------
heartbeat:
  enabled: false                        # bool, default: false
  interval_seconds: 1800               # int, default: 1800 (30 minutes)
  workspace: "~/workspace"             # str, default: "~/workspace"
  active_hours: ""                      # str, e.g. "08:00-22:00"

# ---------------------------------------------------------------------------
# Observability (OpenTelemetry)
# ---------------------------------------------------------------------------
observability:
  otel_enabled: false                   # bool, default: false
  otel_endpoint: "http://localhost:4317" # str, OTLP collector endpoint
  otel_protocol: "grpc"                # str, "grpc" | "http/protobuf"
  otel_service_name: "missy"           # str, service name in traces
  log_level: "warning"                  # str, Python log level

# ---------------------------------------------------------------------------
# Vault (encrypted secrets)
# ---------------------------------------------------------------------------
vault:
  enabled: false                        # bool, default: false
  vault_dir: "~/.missy/secrets"        # str, directory for vault.key + vault.enc

# ---------------------------------------------------------------------------
# Proactive triggers
# ---------------------------------------------------------------------------
proactive:
  enabled: false                        # bool, master switch
  triggers:
    - name: "log-watcher"              # str, required — unique identifier
      trigger_type: "file_change"       # str, required — file_change|disk_threshold|load_threshold|schedule
      enabled: true                     # bool, default: true
      requires_confirmation: false      # bool, default: false — require ApprovalGate
      prompt_template: ""               # str — supports {trigger_name}, {trigger_type}, {timestamp}
      watch_path: "/var/log"            # str — directory to watch (file_change only)
      watch_patterns:                   # list[str] — glob patterns
        - "*.log"
      watch_recursive: false            # bool, default: false
      interval_seconds: 300             # int, default: 300 — polling/repeat interval
      cooldown_seconds: 300             # int, default: 300 — min seconds between firings

    - name: "disk-check"
      trigger_type: "disk_threshold"
      disk_path: "/"                    # str, default: "/"
      disk_threshold_pct: 90.0          # float, default: 90.0 — fire when exceeded

    - name: "load-check"
      trigger_type: "load_threshold"
      load_threshold: 4.0              # float, default: 4.0 — normalized 1-min load avg

# ---------------------------------------------------------------------------
# Container sandbox (Docker-based session isolation)
# ---------------------------------------------------------------------------
container:
  enabled: false                        # bool, default: false — master switch
  image: "python:3.12-slim"            # str, Docker image for sessions
  memory_limit: "256m"                  # str, Docker --memory value
  cpu_quota: 0.5                        # float, CPU fraction (--cpus)
  network_mode: "none"                  # str, Docker --network value

# ---------------------------------------------------------------------------
# Voice channel
# ---------------------------------------------------------------------------
voice:
  host: "0.0.0.0"                      # str, WebSocket bind address
  port: 8765                            # int, WebSocket port
  stt:
    engine: "faster-whisper"            # str, STT backend
    model: "base.en"                    # str, Whisper model name
  tts:
    engine: "piper"                     # str, TTS backend
    voice: "en_US-lessac-medium"        # str, Piper voice model

# ---------------------------------------------------------------------------
# Discord (see discord configuration docs for full schema)
# ---------------------------------------------------------------------------
# discord:
#   bot_token: "vault://DISCORD_BOT_TOKEN"
#   ...

# ---------------------------------------------------------------------------
# Global settings
# ---------------------------------------------------------------------------
workspace_path: "~/workspace"           # str, default: "~/workspace"
audit_log_path: "~/.missy/audit.jsonl" # str, default: "~/.missy/audit.jsonl"
max_spend_usd: 0.0                      # float, default: 0.0 (0 = unlimited)
```

## Field Type Summary

| Field | Type | Default | Notes |
|---|---|---|---|
| `config_version` | `int` | `0` | Schema version stamp |
| `network.default_deny` | `bool` | `true` | Master network deny switch |
| `network.presets` | `list[str]` | `[]` | Named service presets |
| `network.allowed_cidrs` | `list[str]` | `[]` | CIDR notation |
| `network.allowed_domains` | `list[str]` | `[]` | FQDN or suffix patterns |
| `network.allowed_hosts` | `list[str]` | `[]` | `host` or `host:port` |
| `filesystem.allowed_read_paths` | `list[str]` | `[]` | Absolute paths |
| `filesystem.allowed_write_paths` | `list[str]` | `[]` | Absolute paths |
| `shell.enabled` | `bool` | `false` | Master shell switch |
| `shell.allowed_commands` | `list[str]` | `[]` | Command basenames |
| `plugins.enabled` | `bool` | `false` | Master plugin switch |
| `plugins.allowed_plugins` | `list[str]` | `[]` | Plugin identifiers |
| `providers.<name>.model` | `str` | *required* | Model identifier |
| `providers.<name>.api_key` | `str\|null` | `null` | Falls back to env var |
| `providers.<name>.api_keys` | `list[str]` | `[]` | Rotation pool |
| `providers.<name>.timeout` | `int` | `30` | Seconds |
| `providers.<name>.fast_model` | `str` | `""` | Fast tier model |
| `providers.<name>.premium_model` | `str` | `""` | Premium tier model |
| `scheduling.enabled` | `bool` | `true` | Scheduler switch |
| `scheduling.max_jobs` | `int` | `0` | 0 = unlimited |
| `heartbeat.enabled` | `bool` | `false` | Heartbeat switch |
| `heartbeat.interval_seconds` | `int` | `1800` | 30 minutes |
| `observability.otel_enabled` | `bool` | `false` | OTEL switch |
| `observability.log_level` | `str` | `"warning"` | Python log level |
| `vault.enabled` | `bool` | `false` | Vault switch |
| `network.rest_policies` | `list[dict]` | `[]` | L7 REST policy rules |
| `container.enabled` | `bool` | `false` | Container sandbox switch |
| `container.image` | `str` | `"python:3.12-slim"` | Docker image |
| `container.memory_limit` | `str` | `"256m"` | Memory limit |
| `container.cpu_quota` | `float` | `0.5` | CPU fraction |
| `container.network_mode` | `str` | `"none"` | Docker network mode |
| `proactive.enabled` | `bool` | `false` | Proactive triggers switch |
| `max_spend_usd` | `float` | `0.0` | Per-session cost cap |

## Environment Variable Fallbacks

When `api_key` is `null` or omitted for a provider, Missy checks the environment variable `{PROVIDER_NAME}_API_KEY` (uppercased). For example:

| Provider | Environment Variable |
|---|---|
| `anthropic` | `ANTHROPIC_API_KEY` |
| `openai` | `OPENAI_API_KEY` |

## Vault References

Any `api_key` or `api_keys` value can use a `vault://` prefix to resolve secrets from Missy's encrypted vault at runtime:

```yaml
providers:
  anthropic:
    api_key: "vault://ANTHROPIC_API_KEY"
```

See the [Vault documentation](../security/index.md) for setup instructions.
