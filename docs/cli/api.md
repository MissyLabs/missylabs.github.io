---
tags:
  - cli
---

# API Commands

Manage the Agent-as-a-Service REST API server.

## missy api start

Start the REST API server.

```bash
missy api start
missy api start --host 127.0.0.1 --port 8080
```

| Option | Default | Description |
|--------|---------|-------------|
| `--host` | `127.0.0.1` | Bind address (loopback only by default) |
| `--port` | `8080` | Listen port |

!!! warning "Security"
    The API server binds to loopback (`127.0.0.1`) by default. Binding to `0.0.0.0` exposes the API to all network interfaces — use only behind a reverse proxy with TLS.

### Authentication

Requests must include an API key in the `Authorization` header:

```
Authorization: Bearer <api-key>
```

### Rate Limiting

The API enforces per-key rate limits to prevent abuse.

## missy api status

Show API server configuration and status.

```bash
missy api status
```
