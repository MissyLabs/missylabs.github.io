---
tags:
  - edge
---

# Wake Word

missy-edge uses [openWakeWord](https://github.com/dscripka/openWakeWord) with ONNX Runtime for local wake word detection. Detection runs entirely on the Pi -- no audio leaves the device until the wake word is heard.

## How it works

1. The `WakeWordDetector` processes 80ms audio chunks from the microphone.
2. Each chunk is scored by the ONNX model (< 5ms on Pi 5).
3. When the score exceeds `wakeword_threshold`, the wake word is triggered.
4. The **pre-trigger buffer** (default 0.3s) is prepended to the audio stream, capturing the words spoken just before the wake word.
5. Audio streaming begins to the Missy server.
6. Silence detection (configurable threshold + timeout) ends the utterance.

### LED states during a conversation

| State | LED color | Meaning |
|-------|-----------|---------|
| Idle | Off | Listening for wake word |
| Listening | Green | Wake word detected, recording speech |
| Processing | Blue | Audio sent, waiting for agent response |
| Speaking | White | Playing TTS response |
| Muted | Red (solid) | Hardware mute active |
| Error | Red (flashing) | Connection or processing error |

## Default wake word

The default wake word model detects "Hey Missy" or "Missy" depending on the model file. The default model is bundled or downloaded by `setup_pi.sh` to `/opt/missy-edge/models/`.

## Configuration

```yaml
# Wake word settings in /etc/missy-edge/config.yaml
wakeword_model: ""                     # Empty = use default model
wakeword_threshold: 0.7               # Detection confidence [0.0, 1.0]
pre_trigger_seconds: 0.3              # Seconds of audio buffered before detection
```

### Tuning the threshold

| Threshold | Behavior |
|-----------|----------|
| 0.5 | Very sensitive -- more detections but more false positives |
| 0.7 | Balanced (default) |
| 0.85 | Conservative -- fewer false positives but may miss quiet speech |
| 0.95 | Very conservative -- only triggers on clear, close speech |

!!! tip "Start with the default"
    Begin with `0.7` and adjust based on your environment. If the Pi triggers on TV audio or background speech, increase the threshold. If it misses your wake word, decrease it.

## Custom wake word models

### Using a pre-trained model

openWakeWord provides several pre-trained models. Download an `.onnx` file and point to it:

```yaml
wakeword_model: "/opt/missy-edge/models/custom_wake_word.onnx"
```

### Training a custom model

To train your own wake word:

1. Follow the [openWakeWord training guide](https://github.com/dscripka/openWakeWord#training-new-models).
2. Generate positive examples (recordings of your desired wake phrase).
3. Generate negative examples (background noise, unrelated speech).
4. Train and export to `.onnx` format.
5. Place the model file on the Pi:

```bash
sudo cp my_wake_word.onnx /opt/missy-edge/models/
```

6. Update the config:

```yaml
wakeword_model: "/opt/missy-edge/models/my_wake_word.onnx"
```

7. Restart the service:

```bash
sudo systemctl restart missy-edge
```

## Pre-trigger buffer

The pre-trigger buffer is a ring buffer that keeps the last N seconds of audio in memory. When the wake word is detected, this buffered audio is prepended to the stream sent to the server.

This solves the problem of "clipping" the beginning of speech when a user says "Missy, what time is it" in one breath -- without the pre-trigger buffer, "what time" might be lost.

```yaml
pre_trigger_seconds: 0.3    # Default: 300ms
pre_trigger_seconds: 0.5    # Increase for users who speak quickly after the wake word
```

## Barge-in prevention

During TTS playback, wake word detection is **suppressed** to prevent the Pi from triggering on its own speaker output. Detection resumes automatically when playback ends.

## Dependencies

Wake word detection requires the `wakeword` extra:

```bash
pip install -e ".[wakeword]"
```

This installs:

- `openwakeword` -- wake word detection framework
- `onnxruntime` -- ONNX model inference (CPU-optimized for ARM64)

!!! note "tflite-runtime compatibility"
    Some openWakeWord features require `tflite-runtime`, which has specific Python version requirements. Check the openWakeWord docs for the latest compatibility matrix.

## Troubleshooting

### Wake word not triggering

1. Check the threshold: lower `wakeword_threshold` to 0.5 and test.
2. Verify the microphone: `arecord -l` should show the ReSpeaker device.
3. Check the model path: ensure the `.onnx` file exists and is readable.
4. Run with verbose logging: `missy-edge -v` to see detection scores.

### Too many false positives

1. Increase `wakeword_threshold` to 0.85 or higher.
2. Move the Pi away from TVs or speakers.
3. Check ambient noise: the heartbeat reports `noise_level` -- view with `missy devices status`.

### High CPU usage

1. openWakeWord should use < 5% CPU on Pi 5. If higher, check that you are using the ONNX Runtime (not TensorFlow Lite fallback).
2. Verify ARM64 optimizations: `uname -m` should show `aarch64`.
