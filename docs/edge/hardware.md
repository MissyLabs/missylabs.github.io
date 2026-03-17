---
tags:
  - edge
  - hardware
---

# Hardware Requirements

## Target platform

| Component | Recommendation |
|-----------|---------------|
| **Board** | Raspberry Pi 5 (4 GB or more) |
| **OS** | Raspberry Pi OS (Bookworm, 64-bit) |
| **Microphone** | ReSpeaker USB 4-mic Array v2.0 (`0x2886:0x0018`) |
| **Speaker** | Any USB or 3.5mm speaker |

## Why Raspberry Pi 5?

The Pi 5 provides enough CPU headroom for wake word detection (< 5% single core) while leaving resources for audio capture and network I/O. The Pi 4 (4 GB) also works but wake word latency will be higher.

## Why ReSpeaker?

The ReSpeaker USB 4-mic Array provides:

- **4 far-field microphones** -- picks up voice from across a room.
- **USB HID LED ring** -- 12 LEDs for visual feedback (listening, processing, speaking, muted, error states).
- **Physical mute button** -- hardware mute that missy-edge monitors via HID polling.
- **USB Audio Class** -- no drivers needed on Linux, works out of the box.

!!! tip "Other microphones"
    Any USB microphone that presents as a standard audio input device will work for audio capture. You will lose LED feedback and the hardware mute button, but the core functionality (wake word, streaming, playback) works with any sounddevice-compatible input.

## Resource budget

### Idle (listening for wake word)

| Resource | Target |
|----------|--------|
| CPU | < 5% single core |
| Memory | < 100 MB resident |
| Network | < 2 KB/s (heartbeats only) |

### During utterance (streaming audio to server)

| Resource | Target |
|----------|--------|
| CPU | ~10% single core |
| Memory | < 120 MB resident |
| Network | ~256 KB/s (16 kHz mono PCM-16LE) |

## Network requirements

The edge node needs WebSocket connectivity to the Missy server:

| Protocol | Port | Direction |
|----------|------|-----------|
| `ws://` or `wss://` | 8765 (default) | Pi to server |

For TLS (`wss://`), provide a CA certificate path in the edge config:

```yaml
ca_cert: "/etc/missy-edge/ca.pem"
```

## Storage

The edge node itself requires minimal storage:

| Component | Size |
|-----------|------|
| missy-edge package + venv | ~200 MB |
| Wake word model (ONNX) | ~5 MB |
| Config files | < 1 KB |

A standard 16 GB microSD card is more than sufficient.

## Shopping list

For a complete edge node setup:

1. Raspberry Pi 5 (4 GB) -- ~$60
2. ReSpeaker USB 4-mic Array v2.0 -- ~$30
3. USB or 3.5mm speaker -- ~$10-20
4. microSD card (16 GB+) -- ~$10
5. USB-C power supply (27W for Pi 5) -- ~$12
6. Case (optional) -- ~$10

Total: approximately $130-140 per room.
