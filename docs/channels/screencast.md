---
tags:
  - channels
  - screencast
---

# Screencast Channel

The screencast channel lets a user share their browser screen with Missy for live analysis. A user opens a link, grants screen-share permission, and their browser streams periodic JPEG/PNG frames over WebSocket to Missy, which runs each frame through a vision model and reports back what it sees.

`ScreencastChannel` (`missy/channels/screencast/channel.py`) is a combined HTTP + WebSocket server built from four sub-components: `ScreencastTokenRegistry` (auth), `SessionManager` (session/frame-queue state), `FrameAnalyzer` (background vision analysis), and `ScreencastServer` (the actual HTTP/WS listener).

!!! tip "Session creation is currently Discord-driven"
    Today the only built-in way to create a screencast session is Discord's `!screen share` command (or natural-language phrasing like "share my screen"). `ScreencastChannel.create_session()` is a plain Python method, so it can be called from other integrations too, but there is no CLI command for it yet.

## What it captures

- The browser's own screen/window/tab share (via `getDisplayMedia()`), not a webcam.
- Periodic still frames (JPEG or PNG), not continuous video — the client controls its own capture interval (2 seconds to 5 minutes, negotiated over the `config` WebSocket message).
- Each frame is handed to a vision model (Ollama by default, `minicpm-v` unless overridden) with a describe-the-screen prompt; results are stored per session and, when the session originated from Discord, posted back to the originating Discord channel.

## Step 1: Enable and configure the channel

Add a `screencast:` section to `~/.missy/config.yaml`:

```yaml
screencast:
  enabled: true
  host: "127.0.0.1"        # bind address; 0.0.0.0 exposes it on all interfaces
  port: 8780
  max_sessions: 20          # concurrent connected sessions
  frame_save_dir: ""        # if set, frames are saved to disk under this dir
  vision_model: ""           # override the default Ollama vision model
  analysis_prompt: ""        # override the default "describe this screen" prompt
  capture_url_base: ""       # override the share-URL host:port (e.g. behind a reverse proxy)
```

This block is read directly from raw YAML by `missy run` (there is no dataclass in `config/settings.py` for it) — the channel only starts if `screencast.enabled: true` is set, and it starts automatically as part of `missy run` alongside the CLI/Discord/Voice channels.

## Step 2: (optional) pair it with Discord

If a Discord channel is also configured and running, `missy run` wires the two together automatically: the screencast channel gets a reference to Discord's REST client (to post analysis results back into the channel that requested the share), and Discord's message handler gets a reference to the screencast channel (to serve `!screen ...` commands). No extra config is needed beyond enabling both `discord:` and `screencast:`.

## Step 3: Share a screen

From Discord, either the bang command or natural language works:

```
!screen share my-laptop
```

or simply:

```
share my screen, labelled my-laptop
```

The bot replies with a one-time share URL:

```
Screen share session created
Session: `AbCd1234...`
Label: my-laptop

Open this link in your browser to start sharing:
https://192.168.1.50:8780/?session_id=AbCd1234...&token=<token>

Analysis results will be posted here automatically.
```

Opening that URL loads `capture.html`, which requests `getDisplayMedia()` permission from the browser and starts streaming frames once the user grants it.

## Step 4: Review results

- **In Discord**, results post automatically to the channel where `!screen share` was invoked (truncated to fit Discord's 2000-character message limit).
- **On demand**, ask for the latest analysis:

```
!screen analyze
!screen analyze AbCd1234
```

- **List active sessions**: `!screen list`
- **Check server health**: `!screen status`
- **Stop sharing**: `!screen stop` (stops the most recent session) or `!screen stop AbCd1234`

All five subcommands also work as natural language (e.g. "what's on the screen", "stop the screen share", "screencast status") — see `infer_screen_intent()` in `missy/channels/discord/screen_commands.py`.

## Token authentication

`ScreencastTokenRegistry` issues a fresh, random `(session_id, token)` pair per session (`secrets.token_urlsafe`, 16 and 32 bytes respectively). Only a PBKDF2-HMAC-SHA256 hash of the token (100,000 iterations, salted with the session ID) is kept in memory — the plaintext token is returned exactly once, embedded in the share URL, and never stored.

The browser authenticates by sending the `session_id` and plaintext `token` as the first WebSocket message:

```json
{"type": "auth", "session_id": "AbCd1234...", "token": "..."}
```

The server re-derives the PBKDF2 hash and compares it to the stored hash with `hmac.compare_digest` (constant-time). A missing/invalid token, or a stale/revoked session, gets an `auth_fail` message and the connection is closed. Auth must complete within 10 seconds of connecting or the socket is closed.

!!! warning "Tokens travel in the URL"
    The share URL embeds the plaintext token as a query parameter. Treat it like a credential — only share it with the intended viewer, and revoke the session (`!screen stop`) when you're done. Sessions are also capped at `max_sessions` concurrent connections; once at capacity new connections are refused with a `"Too many active sessions"` auth failure.

## Session lifecycle (`SessionManager`)

`SessionManager` tracks two things: the live WebSocket connections (`SessionState`, keyed by `session_id`) and a rolling window of the last 50 analysis results per session (older ones are evicted). Frames flow through a single bounded `asyncio.Queue` (max 50 pending) shared across all sessions; if the queue is full when a frame arrives, the client gets a `{"type": "backpressure"}` message and the frame is dropped rather than queued.

Per-connection state (`SessionState`) is discarded on disconnect, but stored analysis results persist for the session's lifetime (or until the process restarts — everything is in-memory, nothing is written to `~/.missy/`). A session is fully removed only when revoked via `revoke_session()` (`!screen stop`), which just flips `active = False` — a revoked session's token will never authenticate again.

## Frame protocol and validation

Once authenticated, the browser sends a JSON `frame` message describing the next frame (`format`, `width`, `height`, `seq`), immediately followed by the binary frame bytes. The server enforces, per frame:

| Check | Limit |
|-------|-------|
| Frame size | 5 MB max |
| Format | Must be `jpeg` or `png`, validated by magic bytes (not just the claimed format) |
| Dimensions | 64–7680 px per side, if provided |
| Rate | At most one frame every 2 seconds per session (extras are silently dropped) |

Valid frames are enqueued for `FrameAnalyzer`, which runs in the background, calls the vision model (via `missy.channels.discord.image_analyze.analyze_image_bytes`), and stores an `AnalysisResult` (text, model name, processing time) back on the session. If `frame_save_dir` is configured, each accepted frame is also written to `{frame_save_dir}/{session_id}/{timestamp}_{seq}.{jpg|png}` (directory mode `0700`, file mode `0600`, atomic write via temp-file rename).

## Transport security

- **HTTP**: the capture page is served with `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `Cache-Control: no-store`, and a restrictive `Content-Security-Policy` (no external script/style/image sources).
- **TLS**: browsers require a secure context for `getDisplayMedia()`. On `127.0.0.1`/`localhost` binds, Missy skips TLS since localhost already counts as secure. On any other bind address, `ScreencastServer` auto-generates a self-signed EC certificate (stored at `~/.missy/secrets/screencast.{crt,key}`, reused across restarts) and serves over HTTPS — expect a one-time browser warning for the self-signed cert.
- **Connection caps**: at most 20 concurrent WebSocket connections server-wide (independent of the per-session `max_sessions` cap), enforced before authentication.

!!! warning "Binding to 0.0.0.0"
    Binding `host: "0.0.0.0"` exposes the capture server on every network interface and emits a `screencast.bind.warning` audit event. Prefer `127.0.0.1` unless you specifically need LAN/remote access, and rely on the token auth plus TLS above if you do expose it.

## Audit events

All screencast activity publishes structured audit events (category `plugin`) via the core event bus, including `screencast.session.created`, `screencast.session.revoked`, `screencast.connection.auth_ok` / `auth_fail`, `screencast.connection.rejected_capacity`, `screencast.frame.received` / `rejected`, `screencast.analysis.completed` / `failed`, and `screencast.bind.warning`.

## Screencast vs. Voice vs. Webhook

Screencast is visual and session-based; it does not replace the [Voice channel](voice/server.md) (audio) or the [Webhook channel](webhook.md) (fire-and-forget HTTP task ingestion). Use screencast specifically when you want Missy to watch and describe what's happening on a screen in near-real time.
