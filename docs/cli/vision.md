---
tags:
  - cli
---

# Vision Commands

Manage the vision subsystem — camera discovery, capture, analysis, and diagnostics.

Requires `pip install -e ".[vision]"`.

## missy vision devices

Enumerate and diagnose available USB cameras.

```bash
missy vision devices
```

Lists all detected cameras with vendor/product IDs, device paths, and preferred device status.

## missy vision capture

Capture frames from a camera.

```bash
missy vision capture
missy vision capture --device /dev/video0 --count 3 --output ~/captures/
```

| Option | Default | Description |
|--------|---------|-------------|
| `--device` | auto-detect | Camera device path |
| `--output` | `~/.missy/captures/` | Output directory |
| `--count` | `1` | Number of frames to capture |
| `--width` | `1920` | Capture width |
| `--height` | `1080` | Capture height |

## missy vision inspect

Run visual quality assessment on an image.

```bash
missy vision inspect --file photo.jpg
missy vision inspect --screenshot
missy vision inspect --device /dev/video0
```

## missy vision review

LLM-powered visual analysis.

```bash
missy vision review --file photo.jpg
missy vision review --mode puzzle --context "1000-piece landscape"
```

| Option | Default | Description |
|--------|---------|-------------|
| `--file` | none | Image file to analyze |
| `--mode` | `general` | Analysis mode: `general`, `puzzle`, `painting`, `inspection` |
| `--context` | none | Additional context for analysis |

## missy vision doctor

Run full vision subsystem diagnostics.

```bash
missy vision doctor
```

Checks: OpenCV installation, video group membership, device access, capture test, health status.

## missy vision health

Show capture statistics and device health.

```bash
missy vision health
```

## missy vision benchmark

Run capture performance benchmarks.

```bash
missy vision benchmark
```

## missy vision validate

Validate the vision pipeline end-to-end.

```bash
missy vision validate
```

## missy vision memory

Show vision scene memory status and history.

```bash
missy vision memory
```
