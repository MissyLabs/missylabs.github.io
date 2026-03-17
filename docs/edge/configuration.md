---
tags:
  - edge
  - configuration
---

# Edge Configuration

## Config file locations

Configuration loads in order -- later sources override earlier:

1. `/etc/missy-edge/config.yaml` (system)
2. `~/.config/missy-edge/config.yaml` (user)
3. `--config` CLI flag
4. CLI flags (`--server`, `--name`, etc.)

## Full configuration reference

```yaml
# Connection
server_url: "ws://10.0.0.33:8765"     # Missy voice server WebSocket URL
name: "Office Pi"                       # Human-readable device name
room: "Office"                          # Room name (passed to agent as context)

# Audio capture
sample_rate: 16000                      # Hz (server expects 16000)
channels: 1                             # Mono (1) or stereo (2)
chunk_samples: 1280                     # Samples per chunk (80ms at 16kHz)

# Wake word
wakeword_model: ""                      # Path to custom .onnx model (empty = default)
wakeword_threshold: 0.7                 # Detection confidence [0.0, 1.0]
pre_trigger_seconds: 0.3               # Audio buffered before detection

# Utterance streaming
max_utterance_seconds: 30.0            # Hard cap on utterance length
silence_threshold: 0.01                 # RMS below this = silence
silence_timeout: 2.0                    # Seconds of silence to end utterance

# Heartbeat
heartbeat_interval: 60                  # Seconds between heartbeats

# Hardware
led_enabled: true                       # ReSpeaker LED ring feedback
mute_button_enabled: true               # ReSpeaker mute button monitoring
input_device: null                      # sounddevice index or name (null = default)
output_device: null                     # sounddevice index or name (null = default)

# TLS
ca_cert: ""                             # CA cert path for wss:// connections
```

## Sensitive files

| File | Permissions | Purpose |
|------|-------------|---------|
| `/etc/missy-edge/token` | `0600` | Auth token (client refuses to start if world-readable) |
| `/etc/missy-edge/node_id` | `0600` | Device UUID assigned during pairing |
| `/etc/missy-edge/config.yaml` | `0600` | Configuration |

!!! danger "File permissions matter"
    The edge client checks permissions on sensitive files at startup. If the token file is readable by group or others, the client exits immediately with an error. This is a security safeguard, not a bug.

## Common configurations

### Minimal (local network, no TLS)

```yaml
server_url: "ws://10.0.0.33:8765"
name: "Living Room Pi"
room: "Living Room"
```

### With TLS

```yaml
server_url: "wss://missy.example.com:8765"
name: "Kitchen Pi"
room: "Kitchen"
ca_cert: "/etc/missy-edge/ca.pem"
```

### Custom wake word sensitivity

```yaml
server_url: "ws://10.0.0.33:8765"
name: "Bedroom Pi"
room: "Bedroom"
wakeword_threshold: 0.8              # Higher = fewer false positives
pre_trigger_seconds: 0.5             # More pre-trigger audio
silence_timeout: 3.0                 # Longer pause before ending utterance
```

### Specific audio devices

List available devices:

```bash
python3 -c "import sounddevice; print(sounddevice.query_devices())"
```

Then reference by index or name:

```yaml
input_device: 3                        # Device index
output_device: "USB Audio Device"      # Device name
```

### Noisy environment

For rooms with high ambient noise:

```yaml
silence_threshold: 0.05                # Higher threshold to ignore background noise
silence_timeout: 2.5                   # Slightly longer silence window
wakeword_threshold: 0.85              # Reduce false wake word triggers
```

## CLI flags

CLI flags override config file values:

```bash
missy-edge --server ws://10.0.0.33:8765  # Override server URL
missy-edge --config /path/to/config.yaml  # Use specific config file
missy-edge --manual                       # Push-to-talk mode (Enter key)
missy-edge -v                             # Verbose/debug logging
```

## Reconnection behavior

If the server connection drops, the edge client automatically reconnects with exponential backoff:

| Parameter | Value |
|-----------|-------|
| Initial delay | 1 second |
| Maximum delay | 60 seconds |
| Jitter | +/-20% |
| Auth rate limit | 5 attempts per 60-second window |

The client distinguishes between connection failures and auth failures. Auth failures count toward a separate rate limit to prevent token brute-forcing.
