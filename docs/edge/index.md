---
tags:
  - edge
---

# Missy Edge

**missy-edge** is the standalone Raspberry Pi voice client for Missy. Put a Pi with a microphone in every room, say "Missy," and talk. The edge node handles wake word detection locally, streams audio to the Missy server over WebSocket, and plays back the spoken response.

All AI processing stays on the server -- the Pi handles only audio capture, wake word detection, and playback.

## How it works

```
  "Hey Missy, what's the weather?"

  [Pi + ReSpeaker]                      [Missy Server]
       |                                      |
       |  1. Wake word detected locally       |
       |  2. LED -> green                     |
       |                                      |
       |-- audio_start ---------------------->|
       |-- PCM chunks (80ms each) ----------->|  STT (faster-whisper)
       |-- silence detected                   |  Agent (Claude/GPT/Ollama)
       |-- audio_end ------------------------>|  TTS (Piper)
       |                                      |
       |  3. LED -> blue (processing)         |
       |                                      |
       |<----------------------- audio_start  |
       |<----------------------- WAV chunks   |
       |<----------------------- audio_end    |
       |                                      |
       |  4. LED -> white (speaking)          |
       |  5. Speaker plays response           |
       |  6. LED -> off                       |
```

## Relationship to Missy

missy-edge is a **separate repository** ([MissyLabs/missy-edge](https://github.com/MissyLabs/missy-edge)) that connects to Missy's voice server.

| Component | Repository | Runs on |
|-----------|-----------|---------|
| Voice server | [MissyLabs/missy](https://github.com/MissyLabs/missy) (`missy/channels/voice/`) | Your server |
| Edge client | [MissyLabs/missy-edge](https://github.com/MissyLabs/missy-edge) | Raspberry Pi |
| Device management CLI | MissyLabs/missy (`missy devices ...`) | Your server |

The server handles STT, agent processing, and TTS. The Pi handles audio I/O and wake word detection.

## Features

- **Wake word detection** -- openWakeWord + ONNX Runtime, < 5ms per chunk on Pi 5, pre-trigger buffer captures audio before the wake word
- **Streaming audio** -- real-time PCM-16LE via sounddevice, automatic silence detection ends the utterance
- **Auto-reconnect** -- exponential backoff (1s to 60s, +/-20% jitter) with auth rate limiting (5 attempts per 60s window)
- **LED feedback** -- ReSpeaker USB ring: green (listening), blue (processing), white (speaking), red (muted), flashing red (error)
- **Hardware mute** -- physical button on ReSpeaker toggles microphone, LED confirms state
- **Barge-in prevention** -- wake word suppressed during TTS playback to avoid self-triggering
- **Noise estimation** -- sliding-window RMS level reported in heartbeats for ambient awareness
- **Security** -- refuses to start if token file is world-readable, TLS support for `wss://`, systemd sandboxing

## Getting started

1. [Check hardware requirements](hardware.md)
2. [Set up the Pi](pi-setup.md)
3. [Pair with the server](pairing.md)
4. [Configure the edge node](configuration.md)
5. [Set up wake word detection](wake-word.md)

## Architecture

```
missy_edge/
|-- client.py        EdgeClient: reconnect loop, auth, concurrent async loops
|-- audio.py         AudioCapture (sounddevice -> Queue) + AudioPlayback (WAV)
|-- wakeword.py      WakeWordDetector (openWakeWord/ONNX, pre-trigger buffer)
|-- protocol.py      Message builders matching server.py protocol
|-- reconnect.py     ExponentialBackoff + AuthRateLimiter
|-- config.py        EdgeConfig dataclass, YAML load chain, token checks
|-- noise.py         Sliding-window RMS noise estimator
|-- led.py           ReSpeaker USB HID LED controller (6 states)
|-- mute.py          Physical mute button monitor (HID polling thread)
+-- __main__.py      CLI entry point with signal handling
```

**Threading model:** Single asyncio event loop. The sounddevice callback runs in a separate thread and pushes 80ms PCM chunks into an `asyncio.Queue` via `loop.call_soon_threadsafe()`. The mute button polls HID in a daemon thread. Everything else is async.
