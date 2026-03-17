---
tags:
  - edge
  - security
---

# Pairing

Pairing is the process of registering a new edge node with the Missy server. It is a cross-repo workflow that spans the Pi (missy-edge) and the server (missy).

## Overview

```
Pi (missy-edge)                          Server (missy)
     |                                        |
     |-- pair_request ----------------------->|  Server assigns node_id
     |<----------------------- pair_pending --|  Connection closes
     |                                        |
     |                    [Operator approves]  |
     |                    missy devices pair   |
     |                    --node-id <ID>       |
     |                                        |  Token generated & displayed
     |                                        |
     |  [Token provisioned to Pi]             |
     |                                        |
     |-- auth (node_id + token) ------------->|
     |<---------------------------- auth_ok --|
     |                                        |
     |  Ready for voice interaction           |
```

## Step 1: Initiate pairing from the Pi

On the Raspberry Pi:

```bash
sudo -u missy-edge /opt/missy-edge/venv/bin/missy-edge \
    --pair --name "Kitchen Pi" --room "Kitchen"
```

This sends a `pair_request` WebSocket message to the server with the device's friendly name, room, and hardware profile. The server assigns a `node_id`, responds with `pair_pending`, and closes the connection.

!!! note "The Pi does not send a node_id"
    The server generates the node_id during pairing. Do not try to specify one.

## Step 2: Approve on the server

On the machine running Missy, check for pending devices:

```bash
missy devices list
```

Approve the pending device:

```bash
missy devices pair --node-id <NODE_ID>
```

This:

1. Marks the device as `paired=True` in the registry.
2. Generates a cryptographic auth token (32 bytes, URL-safe base64).
3. Stores only the PBKDF2-HMAC-SHA256 hash (100,000 iterations) in the registry.
4. Displays the plaintext token **once**.

!!! danger "Copy the token now"
    The plaintext token is shown exactly once. The server only stores a hash. If you lose the token, you must unpair and re-pair the device.

## Step 3: Provision the token to the Pi

On the Pi, store the token:

```bash
echo 'THE_TOKEN_FROM_STEP_2' | sudo tee /etc/missy-edge/token
sudo chmod 600 /etc/missy-edge/token
sudo chown missy-edge:missy-edge /etc/missy-edge/token
```

The edge client **refuses to start** if the token file is world-readable. This prevents accidental exposure on shared systems.

## Step 4: Start the edge client

```bash
sudo systemctl start missy-edge
```

The client reads the token from `/etc/missy-edge/token`, sends an `auth` message with the node_id and token, and receives `auth_ok` on success.

Check the logs:

```bash
sudo journalctl -u missy-edge -f
```

## Re-pairing a device

If you need to re-pair (lost token, changed server, etc.):

```bash
# On the server: remove the old registration
missy devices unpair NODE_ID

# On the Pi: run pairing again
sudo -u missy-edge /opt/missy-edge/venv/bin/missy-edge \
    --pair --name "Kitchen Pi" --room "Kitchen"

# On the server: approve and get new token
missy devices pair --node-id <NEW_NODE_ID>

# On the Pi: provision new token
echo 'NEW_TOKEN' | sudo tee /etc/missy-edge/token
sudo chmod 600 /etc/missy-edge/token
sudo chown missy-edge:missy-edge /etc/missy-edge/token

# Restart
sudo systemctl restart missy-edge
```

## Security properties

| Property | Implementation |
|----------|---------------|
| Token entropy | 32 bytes (256 bits) via `secrets.token_urlsafe` |
| Token storage (server) | PBKDF2-HMAC-SHA256 hash, 100,000 iterations, node_id as salt |
| Token storage (Pi) | Plaintext file at `/etc/missy-edge/token`, permissions `0600` |
| Token verification | Constant-time comparison via `hmac.compare_digest` |
| File permission check | Edge client refuses to start if token is world-readable |
| Unpaired nodes | Rejected at auth even if token is correct |
| Muted nodes | Receive `muted` frame and are disconnected |
