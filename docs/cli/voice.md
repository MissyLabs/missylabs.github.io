---
tags:
  - cli
---

# Voice Commands

Manage the voice channel and edge node communication.

## missy voice status

Show voice channel configuration and STT/TTS engine status.

```bash
missy voice status
```

## missy voice test

Test TTS synthesis for a specific edge node.

```bash
missy voice test NODE_ID
missy voice test NODE_ID --text "Hello from Missy"
```

| Option | Default | Description |
|--------|---------|-------------|
| `--text` | `"Testing voice output"` | Text to synthesize |
