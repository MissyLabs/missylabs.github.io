---
tags:
  - operations
  - observability
---

# Observability

Missy provides three observability mechanisms: structured audit logging, OpenTelemetry export, and cost tracking.

## Audit logging

Every significant action emits a structured event that is written to `~/.missy/audit.jsonl` as newline-delimited JSON (JSONL).

### Event structure

Each event contains:

```json
{
  "timestamp": "2026-03-17T10:30:45.123456",
  "session_id": "abc-123",
  "task_id": "task-456",
  "event_type": "tool_execute",
  "category": "plugin",
  "result": "allow",
  "detail": {
    "tool": "shell_exec",
    "message": "Command completed successfully"
  }
}
```

### Event categories

| Category | Events |
|----------|--------|
| `security` | Policy violations, injection detection, secrets detection |
| `plugin` | Tool execution, plugin load/execute, MCP calls |
| `voice` | Connection auth, audio received, pair requests |
| `config` | Config load, migration, hot-reload |

### Result values

| Result | Meaning |
|--------|---------|
| `allow` | Action permitted and completed |
| `deny` | Action blocked by policy |
| `error` | Action attempted but failed |

### Querying the audit log

```bash
# Recent events (all categories)
missy audit recent --limit 20

# Security events only
missy audit security --limit 50

# Filter by category
missy audit recent --category plugin --limit 30
```

### Secrets in audit events

Audit event detail messages are automatically passed through the `SecretCensor` to redact potential credentials before being written to the log file.

## OpenTelemetry

Missy can export traces and metrics to any OTLP-compatible collector (Jaeger, Grafana Tempo, Datadog, etc.).

### Setup

Install the OTEL extras:

```bash
pip install -e ".[otel]"
```

Enable in config:

```yaml
observability:
  otel_enabled: true
  otel_endpoint: "http://localhost:4317"
  otel_protocol: "grpc"           # "grpc" or "http/protobuf"
  otel_service_name: "missy"
```

### Protocol options

| Protocol | Endpoint example | Notes |
|----------|-----------------|-------|
| `grpc` | `http://localhost:4317` | Default, lower overhead |
| `http/protobuf` | `http://localhost:4318/v1/traces` | Works through HTTP proxies |

### What gets exported

The `OtelExporter` subscribes to the event bus and creates OTLP spans for:

- Agent completions (provider, model, token usage)
- Tool executions (tool name, result)
- Policy decisions (allow/deny with reason)
- Voice channel events (audio received, STT, TTS)

### Example: Jaeger

```bash
# Start Jaeger with OTLP support
docker run -d --name jaeger \
  -p 16686:16686 \
  -p 4317:4317 \
  jaegertracing/all-in-one:latest
```

```yaml
observability:
  otel_enabled: true
  otel_endpoint: "http://localhost:4317"
```

Then open `http://localhost:16686` to view traces.

## Cost tracking

Missy tracks token usage and estimated costs per session.

### Configuration

```yaml
max_spend_usd: 0.0    # Per-session budget cap; 0 = unlimited
```

When `max_spend_usd` is set, Missy stops making provider calls once the session cost exceeds the limit.

### Viewing costs

```bash
# Show cost tracking config and current session
missy cost

# Per-session breakdown
missy cost --session
```

## Log level

Control the verbosity of Missy's internal logging:

```yaml
observability:
  log_level: "warning"    # debug | info | warning | error | critical
```

!!! tip "Use debug for troubleshooting"
    Set `log_level: "debug"` temporarily when diagnosing issues. This logs every policy check, provider call, and tool execution. Switch back to `warning` for normal operation to avoid log noise.

## Configuration reference

```yaml
observability:
  otel_enabled: false                        # Enable OpenTelemetry export
  otel_endpoint: "http://localhost:4317"     # OTLP collector endpoint
  otel_protocol: "grpc"                      # "grpc" or "http/protobuf"
  otel_service_name: "missy"                 # Service name in traces
  log_level: "warning"                       # Python log level

max_spend_usd: 0.0                           # Per-session budget (0 = unlimited)
audit_log_path: "~/.missy/audit.jsonl"       # Audit log file path
```
