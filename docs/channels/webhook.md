---
tags:
  - channels
  - webhook
---

# Webhook Channel

The webhook channel lets external systems — CI/CD pipelines, cron jobs, other services — hand Missy a task by sending a single HTTP POST request. It runs its own minimal, dependency-free HTTP server (`http.server.HTTPServer`), separate from the [REST API](../cli/api.md).

`WebhookChannel` lives at `missy/channels/webhook.py`.

!!! warning "No CLI command wires this up"
    Unlike the CLI, Discord, or Screencast channels, there is no `missy webhook start` command. `WebhookChannel` is meant to be embedded programmatically: instantiate it, call `start()`, and poll `receive()` in your own driver loop to feed messages into `AgentRuntime`. If you want a ready-to-run HTTP entry point, use `missy api start` (the [REST API](../cli/api.md)) instead.

## How it works

1. `WebhookChannel.start()` binds an `HTTPServer` on a background daemon thread and only accepts `POST /` requests — `GET`, `PUT`, `DELETE`, and `PATCH` all return `405`.
2. Each valid POST is parsed into a `ChannelMessage` and appended to an in-memory queue (capped at 1000 messages; new requests are rejected with `503` once full).
3. Your driver code calls `receive()` to pop the oldest queued message (non-blocking — it returns `None` immediately if the queue is empty; there is no long-poll or wait).
4. `send()` only logs the response length — the channel has no way to push a reply back to the original HTTP caller, since the request has already been closed with a `202 Accepted` at ingest time. Callers must retrieve results some other way (memory/session lookup, a different notification channel, etc.).

```python
from missy.channels.webhook import WebhookChannel

channel = WebhookChannel(host="127.0.0.1", port=9090, secret="my-hmac-secret")
channel.start()

# Elsewhere, in your own polling loop:
msg = channel.receive()
if msg is not None:
    ...  # hand msg.content to AgentRuntime
```

## Request format

Send a JSON body to `POST /`:

```bash
curl -X POST http://127.0.0.1:9090/ \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Deploy finished — summarize the CI log", "sender": "ci-bot"}'
```

| Field | Required | Description |
|-------|----------|--------------|
| `prompt` | Yes | The task text. Rejected with `400` if empty, `413` if over 32,000 characters. |
| `sender` | No | Free-text identifier for the caller. Capped at 64 characters; only alphanumerics and `-_. @` are kept, everything else is stripped. Defaults to `"webhook"`. |

Requirements enforced before the body is even parsed:

- `Content-Type` must be exactly `application/json` (no charset suffix tricks) — anything else gets `415`.
- `Content-Length` must be a valid non-negative integer, and the body must be no larger than 1 MB, or the request is rejected (`400` / `413`).

A subset of request headers is preserved on the resulting `ChannelMessage.metadata["webhook_headers"]`: `Content-Type`, `User-Agent`, `X-Request-Id`, and `X-Missy-Signature`.

### Responses

| Status | Meaning |
|--------|---------|
| `202` | Accepted and queued (`{"status": "queued"}`) |
| `400` | Malformed JSON, missing/empty `prompt`, or invalid `Content-Length` |
| `401` | HMAC signature missing or invalid (only when `secret` is configured) |
| `405` | Method other than POST |
| `413` | Body over 1 MB, or `prompt` over 32,000 characters |
| `415` | `Content-Type` is not `application/json` |
| `429` | Rate limit exceeded for the client IP |
| `503` | Internal queue is full (1000 pending messages) |

Every response includes `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, and `Cache-Control: no-store`.

## Authentication: HMAC signing

`WebhookChannel` has no API-key concept — instead it optionally verifies an HMAC-SHA256 signature over the raw request body:

```python
channel = WebhookChannel(secret="my-shared-secret")
```

When `secret` is set, every request must include an `X-Missy-Signature` header of the form:

```
X-Missy-Signature: sha256=<hex-hmac-of-body>
```

computed as `hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()`, prefixed with `sha256=`. The comparison uses `hmac.compare_digest` (constant-time). A missing or mismatched signature returns `401`.

!!! warning "Unauthenticated by default"
    If `secret` is empty and the channel is bound to anything other than `127.0.0.1`, it logs a warning and accepts **unauthenticated** requests from any client that can reach the port. Always set `secret` before exposing the webhook beyond localhost.

## Rate limiting

The channel tracks request timestamps per client IP in memory (sliding 60-second window, default limit 60 requests/window). Requests over the limit get `429` with a `Retry-After: 60` header. Up to 10,000 IPs are tracked before the oldest stale entries are evicted.

By default the client IP is the raw TCP peer address. If Missy sits behind a trusted reverse proxy, set `trust_proxy=True` to honor `X-Forwarded-For` (leftmost address) instead — only enable this if you control the proxy, since the header is otherwise spoofable.

```python
channel = WebhookChannel(host="0.0.0.0", port=9090, secret="...", trust_proxy=True)
```

## Network policy for the inbound port

`WebhookChannel` binds its own listening socket directly with `http.server` — it is not proxied through Missy's outbound `PolicyHTTPClient` gateway, and there is no `webhook` network preset. The port it listens on is not something `network.allowed_hosts` / presets control (those govern *outbound* calls Missy makes as a client). Instead:

- Use your OS firewall (`ufw`, `iptables`, etc.) to restrict which hosts can reach the bound port.
- Keep `host="127.0.0.1"` (the default) unless you specifically need remote callers — binding `0.0.0.0` exposes the port on every network interface.
- If you do expose it beyond localhost, always set `secret` (see above) so requests are signed.

```yaml
# Missy's config.yaml has no dedicated "webhook:" section — WebhookChannel
# is constructed and started in your own integration code, e.g.:
```

```python
WebhookChannel(
    host="127.0.0.1",   # bind address
    port=9090,           # listen port
    secret="vault-or-env-sourced-secret",
    trust_proxy=False,
)
```

## Webhook vs. REST API

Both accept HTTP input, but they solve different problems:

| | Webhook channel | [REST API](../cli/api.md) |
|---|---|---|
| Purpose | Fire-and-forget task ingestion | Full programmatic session/chat management |
| Auth | Optional HMAC-SHA256 body signature | Required API key (`Authorization: Bearer`) |
| Response | `202 Accepted`, no result in the response | Synchronous response with the agent's output |
| Endpoints | Single `POST /` | Multiple: sessions, chat, memory search, provider/tool introspection, status |
| CLI entry point | None — embed and drive manually | `missy api start` |
| Typical use | CI/CD pipelines pinging "something happened" | Applications that need request/response chat |

If you need the caller to receive the agent's answer synchronously, use the REST API. Use the webhook channel when you only need to *notify* Missy that it should act, and the result will surface elsewhere (memory, another channel, a scheduled follow-up).
