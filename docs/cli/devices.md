---
tags:
  - cli
---

# Device Commands

Manage voice edge nodes (Raspberry Pi devices running [missy-edge](https://github.com/MissyLabs/missy-edge)).

## missy devices list

List all registered edge nodes.

```bash
missy devices list
```

## missy devices pair

Approve a pending pairing request.

```bash
missy devices pair --node-id NODE_ID
```

The generated auth token must be provisioned to the edge node at `/etc/missy-edge/token`.

## missy devices unpair

Remove a paired edge node.

```bash
missy devices unpair NODE_ID
```

## missy devices status

Show online/offline status and sensor data for all nodes.

```bash
missy devices status
```

## missy devices policy

Set the policy mode for an edge node.

```bash
missy devices policy NODE_ID --mode full
missy devices policy NODE_ID --mode safe-chat
missy devices policy NODE_ID --mode muted
```

| Mode | Description |
|------|-------------|
| `full` | Full agent capabilities |
| `safe-chat` | Chat only, no tools |
| `muted` | Node silenced, no responses |
