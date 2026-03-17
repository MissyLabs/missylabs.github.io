---
tags:
  - channels
  - voice
---

# Device Management

Edge nodes (Raspberry Pi devices) are managed through the `missy devices` CLI commands. The device registry is stored at `~/.missy/devices.json`.

## Listing devices

```bash
# Show all registered devices (paired and pending)
missy devices list

# Show online/offline status with heartbeat data
missy devices status
```

## Pairing a new device

Pairing is a two-step process that spans the edge node and the Missy server.

### Step 1: Initiate from the edge node

On the Raspberry Pi, run the pairing command:

```bash
sudo -u missy-edge /opt/missy-edge/venv/bin/missy-edge \
    --pair --name "Kitchen Pi" --room "Kitchen"
```

This sends a `pair_request` to the server. The server assigns a `node_id` and responds with `pair_pending`. The connection then closes.

### Step 2: Approve on the server

On the Missy server, approve the pending device:

```bash
missy devices pair --node-id <NODE_ID>
```

This generates an auth token and displays it once. Copy it immediately -- the server only stores a hash.

### Step 3: Provision the token

On the Pi, store the token:

```bash
echo 'THE_TOKEN' | sudo tee /etc/missy-edge/token
sudo chmod 600 /etc/missy-edge/token
sudo chown missy-edge:missy-edge /etc/missy-edge/token
```

!!! warning "Token security"
    The token is shown exactly once during `missy devices pair`. If you lose it, generate a new one by unpairing and re-pairing the device. The edge client refuses to start if the token file is world-readable.

## Unpairing a device

Remove a device from the registry:

```bash
missy devices unpair NODE_ID
```

The node will fail to authenticate on its next connection attempt and must be re-paired.

## Policy modes

Each edge node has a policy mode that controls its capabilities:

```bash
missy devices policy NODE_ID --mode full
```

| Mode | Description |
|------|-------------|
| `full` | Full agent capabilities (tools, shell, etc. per main policy) |
| `safe-chat` | Chat only, no tool invocations |
| `muted` | Node is rejected at connection time with a `muted` frame |

!!! tip "Use safe-chat for shared spaces"
    Set `safe-chat` for edge nodes in common areas (living room, kitchen) to prevent unintended tool execution. Use `full` only for private offices or workstations.

## Device registry fields

Each registered node stores:

| Field | Description |
|-------|-------------|
| `node_id` | UUID4 identifier assigned at pairing time |
| `friendly_name` | Human-readable label (e.g., "Living Room Pi") |
| `room` | Room name passed as context to the agent |
| `ip_address` | Last known IP address |
| `hardware_profile` | Device hardware details (mic type, channels, speaker) |
| `last_seen` | Unix timestamp of last successful contact |
| `status` | `online`, `offline`, or `muted` |
| `policy_mode` | `full`, `safe-chat`, or `muted` |
| `paired` | Whether the operator has approved the device |
| `sensor_data` | Latest occupancy, noise level, and update timestamp |

## Audio logging

Per-node audio logging can be enabled for debugging:

```yaml
# Set via registry update (not yet exposed in CLI)
audio_logging: true
audio_log_dir: "~/.missy/audio-logs/kitchen"
audio_log_retention_days: 7
```

Audio files are saved as timestamped `.pcm` files with restrictive permissions (`0600`). The registry automatically purges files older than the retention period.

## Registry storage

The device registry is stored at `~/.missy/devices.json` with the following security properties:

- Atomic writes via write-to-temp-then-rename.
- File permissions set to `0600` (owner read/write only).
- Refuses to load if owned by a different user or group/world-writable.
- Token hashes use PBKDF2-HMAC-SHA256 with 100,000 iterations.
