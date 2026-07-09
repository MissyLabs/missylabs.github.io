---
tags:
  - operations
  - api
---

# REST API Reference

Missy's Agent-as-a-Service HTTP API is served by `ApiServer` (`missy/api/server.py`) under the `/api/v1/*` prefix. The same process also serves the browser-based [Operator Console](operator-console.md) at `/` -- see that page for the human-facing UI. For the CLI commands that start and inspect this server (`missy api start`, `missy api status`), see [API Commands](../cli/api.md).

!!! warning "Loopback by default"
    The server binds to `127.0.0.1` by default. Binding to `0.0.0.0` exposes it to every network interface -- only do this behind a reverse proxy with TLS.

## Authentication

Every request must authenticate one of two ways:

| Method | Header | Notes |
|---|---|---|
| API key | `Authorization: Bearer <api-key>` | Preferred. Also accepted as `X-API-Key: <api-key>` |
| Browser session | `Cookie: missy_operator_session=<token>` | Set by `POST /login`; see [Operator Console](operator-console.md) |

The API key is compared with a constant-time check (`hmac.compare_digest`). If `ApiConfig.api_key` is empty (unset), **every** request is rejected with `401`, including from the console.

## Rate limiting

Requests are rate-limited per client IP using a 60-second sliding window, configurable via `ApiConfig.rate_limit_rpm` (default `60`). Exceeding the limit returns `429` with `{"status": "error", "error": "Rate limit exceeded"}`.

## CSRF on unsafe methods

CSRF tokens are only required for **browser-session-authenticated** requests -- a request authenticated with an `Authorization: Bearer` API key is not subject to the CSRF check, since it cannot be forged by a page loaded in the operator's browser.

For any `POST`, `PUT`, `PATCH`, or `DELETE` request authenticated via the session cookie, the client must also send:

```
X-CSRF-Token: <csrf-token-from-login>
```

A missing or mismatched token returns `403` and is recorded as a `web.csrf` audit deny event. The console's own JavaScript already attaches this header automatically.

## Response envelope

Every response is a JSON envelope:

```json
// Success
{"status": "ok", "data": { /* endpoint-specific payload */ }}

// Error
{"status": "error", "error": "human-readable message"}
```

`POST /api/v1/sessions` and `POST /api/v1/scheduler/jobs` return `201 Created`; `POST /api/v1/runs` returns `202 Accepted`; everything else that succeeds returns `200`. Unknown routes return `404`.

Every response also carries `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, and `Cache-Control: no-store`.

## Endpoints

### Status, providers, tools, diagnostics

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/health` | Liveness probe -- always `{"status": "healthy", "version": "..."}` if the server is up |
| `GET` | `/api/v1/status` | Agent runtime status: available providers, default provider, tool count, session count, memory presence |
| `GET` | `/api/v1/providers` | List registered providers with `available` and `is_default` flags |
| `GET` | `/api/v1/tools` | List registered tools with name, description, and JSON schema |
| `GET` | `/api/v1/diagnostics` | Section-by-section operator diagnostics (web entrypoint, providers, tools, memory, policy, gateway, Discord, scheduler, runtime) -- the same data source as `missy doctor` |
| `GET` | `/api/v1/costs/summary` | Aggregate spend/token totals, a per-session breakdown (`?limit=`, default 10, max 100), and the configured `max_spend_usd` budget cap |

### Sessions

API sessions are lightweight, in-memory conversation handles distinct from channel-level sessions; they do not survive an `ApiServer` restart.

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/sessions` | Create a session. Body: `name` (optional), `provider` (optional, defaults to the registry default) |
| `GET` | `/api/v1/sessions` | List sessions, most-recent first. Query: `limit` (default 20, max 200) |
| `GET` | `/api/v1/sessions/{id}` | Fetch a single session's metadata |
| `DELETE` | `/api/v1/sessions/{id}` | End a session |
| `GET` | `/api/v1/sessions/{id}/history` | Conversation turns for the session, chronological. Query: `limit` (default 50, max 500) |

### Chat (synchronous)

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/chat` | Send a message and block until the agent replies |

Body: `message` (required), `session_id` (optional -- a new session is created when omitted), `provider` (optional per-turn override). The response text is passed through the same `SecretCensor` used elsewhere in Missy before being returned.

### Runs (background + SSE streaming)

For anything that may call tools or run long, prefer `/runs` over `/chat`: it starts the agent on a background thread and lets the client watch progress live instead of holding the HTTP connection open.

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/runs` | Start a background run. Body: `message` (required), `session_id` (optional), `provider` (optional). Returns `202` with `run_id` immediately |
| `GET` | `/api/v1/runs?session_id=` | List recent runs for a session. Query: `session_id` (required), `limit` (default 20, max 100) |
| `GET` | `/api/v1/runs/{id}` | Poll a run's current status/result -- use this if you don't want to hold an SSE connection open |
| `GET` | `/api/v1/runs/{id}/events` | Server-Sent Events stream of run progress |

Only one run may be in flight per session at a time. A second `POST /api/v1/runs` against a busy session returns `409` and is recorded as a `web.run` audit deny event.

The SSE stream (`text/event-stream`) emits, in order: `run.start`, zero or more `tool.request` / `tool.result` pairs, then a terminal `run.complete` or `run.error` event. Reconnecting to a run that has already finished (or polling `GET /api/v1/runs/{id}`) returns the terminal state immediately rather than hanging. The terminal event carries `resolved_provider`, `tools_used`, and `cost` -- the same summary already recorded on the `agent.run.complete` audit event -- so a client never needs a second request to know what actually served the run.

### Memory

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/memory/search?q=` | Full-text search over conversation history. Query: `q` (required), `limit` (default 10, max 100), `session_id` (optional scope) |
| `POST` | `/api/v1/memory/turns/{id}/pin` | Set a turn's pinned state. Body: `pinned` (bool, default `true`). Pinned turns survive `missy sessions cleanup` and the memory store's age-based `cleanup()` |
| `DELETE` | `/api/v1/memory/turns/{id}` | Permanently delete a single turn and its full-text index entry |

Search and session-history results both include each turn's `id` and `pinned` state.

### Scheduler

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/scheduler/jobs` | List scheduled jobs in full detail |
| `POST` | `/api/v1/scheduler/jobs` | Create a job. Body: `name`, `schedule`, `task` (all required); `provider`, `description`, `active_hours`, `timezone` (all optional) |
| `DELETE` | `/api/v1/scheduler/jobs/{id}` | Remove a job -- a thin alias for the `scheduler.remove_job` control below, so it goes through the same confirmation/audit path |

### Audit

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/audit` | Browse redacted, paginated audit events |

Query parameters: `q` (substring search), `event_type`, `category`, `result`, `session_id`, `task_id`, `severity`, `actor`, `source`, `subsystem`, `action`, `since`, `until` (ISO-8601 timestamp bounds), `limit` (default 50, max 500), `offset`. The response includes `total`, `has_more`, and `facets` (counts by category/result/severity/subsystem) for building filter UIs. Every event is redacted -- keys such as `api_key`, `password`, `secret`, and `token` are replaced with `[REDACTED]`, and string values are run through the secrets detector before being returned.

### Operator controls

Controls are a small, deliberately narrow set of state-changing operations exposed with built-in confirmation, distinct from the general-purpose `/chat` and `/runs` endpoints.

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/controls` | List available controls and their current targets/state |
| `POST` | `/api/v1/controls/{id}` | Execute a control |

Current control IDs: `provider.set_default`, `scheduler.pause_job`, `scheduler.resume_job`, `scheduler.remove_job`. Each requires a `confirm` field in the body matching a per-target template (e.g. `set-default:<provider>`, `remove-job:<job-id>`) -- a request without the exact matching confirmation string is denied. Every execution (allowed or denied) is recorded as a `web.control` audit event.

## curl examples

### Send a message and get an immediate reply

```bash
curl -s -X POST http://127.0.0.1:8080/api/v1/chat \
  -H "Authorization: Bearer $MISSY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "What is the capital of France?"}'
# => {"status": "ok", "data": {"response": "...", "session_id": "...", "usage": {}}}
```

### Start a background run and stream its progress

```bash
# 1. Start the run -- returns immediately with a run_id
curl -s -X POST http://127.0.0.1:8080/api/v1/runs \
  -H "Authorization: Bearer $MISSY_API_KEY" -H "Content-Type: application/json" \
  -d '{"message": "list the files in ~/workspace"}'
# => {"status": "ok", "data": {"run_id": "...", "session_id": "...", "status": "pending", ...}}

# 2. Watch it live over Server-Sent Events
curl -N -H "Authorization: Bearer $MISSY_API_KEY" \
  http://127.0.0.1:8080/api/v1/runs/<run_id>/events
# event: run.start
# event: tool.request
# data: {"tool": "list_files", ...}
# event: tool.result
# data: {"tool": "list_files", "is_error": false, ...}
# event: run.complete
# data: {"response": "...", "resolved_provider": "anthropic", "tools_used": ["list_files"], "cost": {...}}
```

## See also

- [Operator Console](operator-console.md) -- the browser UI built on top of this API
- [API Commands](../cli/api.md) -- `missy api start` / `missy api status` CLI reference
- [Observability](observability.md) -- audit log structure and event categories
