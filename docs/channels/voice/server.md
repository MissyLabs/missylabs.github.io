---
tags:
  - channels
  - voice
---

# Voice Server

The voice server accepts WebSocket connections from edge nodes (Raspberry Pi devices with microphones and speakers), handles speech-to-text, routes queries through the agent, and streams synthesized audio responses back.

## Architecture

```
Edge Node (Pi + ReSpeaker)
    │
    │  WebSocket (ws:// or wss://)
    ▼
VoiceServer (missy/channels/voice/server.py)
    ├── DeviceRegistry    — node auth + pairing
    ├── PairingManager    — first-contact registration
    ├── PresenceStore     — room occupancy + sensor data
    ├── STTEngine         — faster-whisper transcription
    ├── TTSEngine         — Piper synthesis
    └── AgentRuntime      — same agent as CLI/Discord
```

## Configuration

Add the `voice:` section to `~/.missy/config.yaml`:

```yaml
voice:
  host: "0.0.0.0"          # Listen on all interfaces
  port: 8765                # WebSocket port
  stt:
    engine: "faster-whisper"
    model: "base.en"        # Options: tiny.en, base.en, small.en, medium.en
  tts:
    engine: "piper"
    voice: "en_US-lessac-medium"
```

!!! warning "Binding to 0.0.0.0"
    Binding to `0.0.0.0` exposes the voice channel on all network interfaces. The server emits a `voice.bind.warning` audit event when this happens. For local-only use, bind to `127.0.0.1`.

## STT engines

### faster-whisper (default)

[faster-whisper](https://github.com/SYSTRAN/faster-whisper) provides fast, accurate transcription. Install the voice extras:

```bash
pip install -e ".[voice]"
```

Model selection affects accuracy vs. speed:

| Model | Size | Speed | Accuracy |
|-------|------|-------|----------|
| `tiny.en` | 39 MB | Fastest | Good for simple commands |
| `base.en` | 74 MB | Fast | Good general purpose |
| `small.en` | 244 MB | Moderate | Better accuracy |
| `medium.en` | 769 MB | Slow | Best accuracy |

## TTS engines

### Piper

[Piper](https://github.com/rhasspy/piper) is a fast, local TTS engine. It runs as a separate binary (not a pip package).

Install from the [Piper releases page](https://github.com/rhasspy/piper/releases):

```bash
# Download and extract piper
wget https://github.com/rhasspy/piper/releases/download/2023.11.14-2/piper_linux_x86_64.tar.gz
tar xzf piper_linux_x86_64.tar.gz
sudo mv piper /usr/local/bin/

# Download a voice model
mkdir -p ~/.local/share/piper
wget -O ~/.local/share/piper/en_US-lessac-medium.onnx \
  https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_US/lessac/medium/en_US-lessac-medium.onnx
```

## Server limits

The voice server enforces several safety limits:

| Limit | Value | Purpose |
|-------|-------|---------|
| Max audio buffer | 10 MB | Prevents memory exhaustion per connection |
| Max WebSocket frame | 1 MB | Limits individual frame size |
| Max concurrent connections | 50 | Connection flood protection |
| Auth timeout | 10 seconds | Closes unauthenticated connections |
| Sample rate range | 8,000--48,000 Hz | Rejects out-of-range values |
| Audio channels | 1--2 | Mono or stereo only |

## Starting the server

The voice server starts as part of the gateway:

```bash
missy gateway start --host 0.0.0.0 --port 8765
```

Check status:

```bash
missy gateway status
missy voice status
```

## Testing

Test TTS synthesis for a specific edge node:

```bash
missy voice test NODE_ID --text "Hello from Missy"
```

## Audit events

The voice server emits structured audit events for all operations:

| Event | Meaning |
|-------|---------|
| `voice.bind.warning` | Server bound to 0.0.0.0 |
| `voice.connection.auth_ok` | Node authenticated successfully |
| `voice.connection.auth_fail` | Authentication failed |
| `voice.connection.rejected_muted` | Muted node tried to connect |
| `voice.connection.closed` | Node disconnected |
| `voice.audio.received` | Audio buffer received from node |
| `voice.pair_request` | New device requested pairing |
