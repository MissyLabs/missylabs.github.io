---
tags:
  - operations
  - api
---

# Operator Console

The operator console (internally also called the "Web TUI") is a browser-based dashboard served from the same process as the [REST API](rest-api.md), at `/`. It gives an operator a visual view of a running Missy instance -- and a way to drive it -- without touching the terminal.

```bash
missy api start                   # loopback-only by default
missy api start --host 127.0.0.1 --port 8080
missy api status
```

See [API Commands](../cli/api.md) for the full CLI flag reference. `MISSY_API_KEY` (or `ApiConfig.api_key`) must be set before starting the server -- with no key configured, every request, including the console's own login page, is rejected with `401`.

Open `http://127.0.0.1:8080/` and sign in with the operator key.

## Login flow

`GET /login` renders a single-field form (the operator key, submitted as a password input). `POST /login` compares the submitted value against the configured API key with a constant-time check:

- **On success**, the server creates a server-side `WebSession` (random 32-byte token + a separate random 32-byte CSRF token, both from `secrets.token_urlsafe`) and sets it as a cookie, then redirects to `/`.
- **On failure**, it redirects back to `/login?error=1` and records a `web.login` audit deny event.

The session cookie (`missy_operator_session` by default) is `HttpOnly` and `SameSite=Strict`, with a default 8-hour TTL (`ApiConfig.web_session_ttl_seconds`). `POST /logout` requires the CSRF header, revokes the session server-side, clears the cookie, and records a `web.logout` audit event.

Every unsafe request the console's own JavaScript makes (`POST`/`PUT`/`PATCH`/`DELETE`) automatically attaches the session's CSRF token as an `X-CSRF-Token` header -- see [REST API: CSRF on unsafe methods](rest-api.md#csrf-on-unsafe-methods) for the enforcement details.

## What's on screen

The console is a single page (`render_console()` in `missy/api/web_console.py`) that polls the JSON API every 15 seconds and re-renders. All panels read from the same `/api/v1/*` endpoints documented in the [REST API reference](rest-api.md) -- anything the console can do, a script can do too.

### Runtime status

The hero section at the top shows overall runtime health plus four counters: providers available, tool count, session count, and whether memory is active. Backed by `GET /api/v1/status`.

### Ask Missy (live run console)

The primary "ask the bot and watch it work" panel. Submitting the textarea calls `POST /api/v1/runs`, which starts the run on a background thread and returns a `run_id` immediately without blocking the request. The browser then opens `GET /api/v1/runs/{run_id}/events` as an `EventSource` (Server-Sent Events) and renders each event live in a scrolling log: a `run.start` marker, one line per `tool.request`/`tool.result` pair (tool name and success/failure), and finally a `run.complete` or `run.error` entry. On completion, the panel also displays which provider actually served the request, which tools it used, and the run's cost -- all carried on the terminal SSE event, so no follow-up request is needed. Only one run may be in flight per session; starting a second one while a run is active is rejected with `409`.

### Providers / Tools / Sessions

Three read-only panels backed by `GET /api/v1/providers`, `GET /api/v1/tools`, and `GET /api/v1/sessions?limit=8`: each configured provider with its availability and default status, the registered tool catalog with descriptions, and the most recently active API sessions.

### Diagnostics

Backed by `GET /api/v1/diagnostics`. Renders the same section-by-section health checks (web entrypoint, providers, tools, memory, policy, gateway, Discord, scheduler, runtime) that power `missy doctor`, including remediation hints where a check has one.

### Controls

Backed by `GET /api/v1/controls`. Lists the narrow set of confirmable operator actions -- switching the default provider, pausing/resuming/removing a scheduled job -- as buttons. Clicking one shows a native browser confirmation dialog before issuing `POST /api/v1/controls/{id}` with the required `confirm` string; disabled or unavailable targets are grayed out.

### Scheduled Jobs

Lists existing jobs (`GET /api/v1/scheduler/jobs`) alongside a creation form that posts `name`, `schedule`, `task`, and optionally `provider` and `active_hours` to `POST /api/v1/scheduler/jobs`. Each job row has a **Remove** button that confirms, then calls `DELETE /api/v1/scheduler/jobs/{id}` -- routed through the same `scheduler.remove_job` control and audit path as the Controls panel.

### Memory Browser

A debounced search box calls `GET /api/v1/memory/search` (optionally scoped by session ID) and renders matching turns with a preview, role, provider, and timestamp. Each result has **Pin/Unpin** (`POST /api/v1/memory/turns/{id}/pin`) and **Delete** (`DELETE /api/v1/memory/turns/{id}`) actions; pinned turns are marked with a star and are protected from `missy sessions cleanup`'s age-based pruning.

### Audit Trail

Backed by `GET /api/v1/audit`, with filter controls for result, severity, subsystem, actor, source, a free-text query, and since/until timestamps, plus prev/next pagination. Selecting a row shows its full redacted JSON detail in a side panel.

### Security panel

A static informational panel summarizing the posture below: cookie session + API key authentication, CSRF required for browser actions, CSP/no-store/frame-deny response headers, and loopback-by-default networking.

## Security posture

| Property | Detail |
|---|---|
| Network exposure | Binds to `127.0.0.1` by default; a warning is logged if started on a non-loopback host |
| Authentication | API key (`Authorization: Bearer` / `X-API-Key`) for scripts, or a signed session cookie for the browser |
| Session cookie | `HttpOnly`, `SameSite=Strict`, random 32-byte token, default 8-hour TTL |
| CSRF | Every unsafe request from a cookie-authenticated session requires a matching `X-CSRF-Token`; mismatches are denied (`403`) and audited as `web.csrf` |
| Response headers | `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Cache-Control: no-store` on every response |
| HTML-specific headers | `Referrer-Policy: no-referrer` and a `Content-Security-Policy` of `default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self'; frame-ancestors 'none'; base-uri 'none'; form-action 'self'` on every HTML page (console, login, message pages) |
| Secrets in output | Audit event details are redacted (keys like `api_key`, `password`, `secret`, `token` become `[REDACTED]`; string values pass through the secrets detector) before they reach the browser |
| Rate limiting | Shared with the JSON API -- 60 requests/minute per client IP by default |

!!! security "Loopback is the safety boundary"
    The console has no independent authorization model beyond "holds the operator API key" -- there are no per-operator accounts or roles. Treat the operator key with the same care as root/sudo credentials, and if you must expose the console beyond localhost, put it behind a reverse proxy that adds TLS and its own access control.

## See also

- [REST API Reference](rest-api.md) -- the full endpoint list the console is built on
- [API Commands](../cli/api.md) -- `missy api start` / `missy api status` CLI reference
- [Observability](observability.md) -- audit log structure and event categories
