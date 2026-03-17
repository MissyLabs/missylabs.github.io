---
tags:
  - channels
  - voice
  - protocol
---

# WebSocket Protocol

The voice channel uses a stateful WebSocket protocol. Messages are JSON text frames except for audio data, which is sent as binary frames. The first message from any client **must** be `auth` or `pair_request`.

## Connection lifecycle

```
Client                                Server
  │                                      │
  ├── auth ─────────────────────────────►│
  │◄─────────────────────── auth_ok ─────┤
  │                                      │
  ├── audio_start ──────────────────────►│
  ├── [binary PCM chunks] ──────────────►│
  ├── audio_end ────────────────────────►│
  │                                      │  STT → Agent → TTS
  │◄──────────────────── response_text ──┤
  │◄──────────────────── audio_start ────┤
  │◄──────────────────── [binary WAV] ───┤
  │◄──────────────────── audio_end ──────┤
  │                                      │
  ├── heartbeat ────────────────────────►│
  │                                      │
```

## Client-to-server messages

### auth

Must be the first message. Authenticates using a node ID and token.

```json
{
  "type": "auth",
  "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "token": "URL_SAFE_BASE64_TOKEN"
}
```

### pair_request

Alternative first message for unregistered devices. The server assigns a `node_id` and closes the connection after responding with `pair_pending`.

```json
{
  "type": "pair_request",
  "friendly_name": "Living Room Pi",
  "room": "Living Room",
  "hardware_profile": {
    "mic_type": "respeaker",
    "channels": 4,
    "speaker": true
  }
}
```

!!! note "Server assigns the node_id"
    Do not include `node_id` in the pair request. The server generates it and returns it in the `pair_pending` response.

### audio_start

Begins an audio streaming session. Must be followed by binary frames and an `audio_end`.

```json
{
  "type": "audio_start",
  "sample_rate": 16000,
  "channels": 1,
  "format": "pcm_s16le"
}
```

### Binary audio frames

Raw PCM audio bytes sent as WebSocket binary frames. Typically 80ms chunks (1,280 samples at 16 kHz). Binary frames outside an active audio session are ignored.

### audio_end

Signals the end of the audio stream. Triggers STT transcription and agent processing.

```json
{
  "type": "audio_end"
}
```

### heartbeat

Periodic keepalive with optional sensor data.

```json
{
  "type": "heartbeat",
  "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "occupancy": true,
  "noise_level": 0.03,
  "wake_word_fp": false
}
```

| Field | Type | Description |
|-------|------|-------------|
| `occupancy` | bool or null | Room occupied (from sensor data) |
| `noise_level` | float or null | Ambient noise RMS [0.0, 1.0] |
| `wake_word_fp` | bool | False positive wake word detection flag |

## Server-to-client messages

### auth_ok

Sent after successful authentication.

```json
{
  "type": "auth_ok",
  "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "room": "Living Room"
}
```

### auth_fail

Sent when authentication fails. The connection is closed immediately after.

```json
{
  "type": "auth_fail",
  "reason": "invalid credentials"
}
```

### pair_pending

Sent after a successful pair request. The connection is closed after this message. The operator must approve the device via `missy devices pair` before the node can authenticate.

```json
{
  "type": "pair_pending",
  "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### transcript

Sent when `debug_transcripts` is enabled on the server. Contains the STT result.

```json
{
  "type": "transcript",
  "text": "What is the weather today?",
  "confidence": 0.95
}
```

### response_text

The agent's text response. Always sent before any TTS audio.

```json
{
  "type": "response_text",
  "text": "The current temperature is 72 degrees."
}
```

### audio_start (server)

Begins a TTS audio stream back to the client.

```json
{
  "type": "audio_start",
  "sample_rate": 22050,
  "format": "wav"
}
```

### Binary WAV frames

WAV audio data sent as binary WebSocket frames in 4 KB chunks (configurable via `audio_chunk_size`).

### audio_end (server)

Signals the end of TTS audio.

```json
{
  "type": "audio_end"
}
```

### error

Sent on processing failures. The connection may or may not be closed depending on severity.

```json
{
  "type": "error",
  "message": "Speech recognition failed"
}
```

### muted

Sent when a node with `policy_mode: muted` attempts to connect. The connection is closed immediately after.

```json
{
  "type": "muted"
}
```

## Protocol notes

!!! warning "Spec doc vs. actual implementation"
    The original spec document and the actual server implementation differ. Both missy-edge and the server use the **implementation**:

    | Spec doc | Actual server |
    |----------|---------------|
    | `stream_start` / `stream_end` | `audio_start` / `audio_end` |
    | `tts_audio` + `tts_end` (raw PCM) | `audio_start` + binary WAV + `audio_end` |
    | Client sends `node_id` in pair_request | Server assigns `node_id` |
    | Token via WebSocket `pair_ack` | Token shown in CLI `missy devices pair` output |

## Security

- The first frame must be `auth` or `pair_request`. Any other message type causes immediate disconnection.
- Binary frames before authentication are rejected.
- Nodes with `paired=False` are rejected even if the token is valid.
- Nodes with `policy_mode=muted` receive a `muted` frame and are disconnected.
- Token verification uses PBKDF2-HMAC-SHA256 with constant-time comparison.
