# Operations

Running and managing Leyline nodes in production.

## Installation

### systemd Service

The install script sets up Leyline as a systemd service:

```bash
# Regular node
curl -fsSL https://raw.githubusercontent.com/MissyLabs/leyline/main/scripts/install.sh | bash

# Seed node
curl -fsSL https://raw.githubusercontent.com/MissyLabs/leyline/main/scripts/install.sh | bash -s -- --seed
```

You'll be prompted to choose:

- **System install**: `/opt/leyline`, runs as a system service
- **User install**: `~/.local/share/leyline`, runs as a user service

### Environment Variables

Customize the install by setting these before running the script:

| Variable | Default | Description |
|---|---|---|
| `LEYLINE_MODE` | `node` | `node` or `seed` |
| `LEYLINE_PORT` | `9876` | Listen port |
| `LEYLINE_DIR` | `./data` | Data directory |
| `LEYLINE_BRANCH` | `main` | Git branch to install |

### Uninstalling

```bash
curl -fsSL https://raw.githubusercontent.com/MissyLabs/leyline/main/scripts/uninstall.sh | bash
```

By default, the data directory is preserved. Add `--purge` to delete all data including identity and ledgers:

```bash
curl -fsSL https://raw.githubusercontent.com/MissyLabs/leyline/main/scripts/uninstall.sh | bash -s -- --purge
```

## Managing the Service

```bash
# Status
systemctl status leyline

# Logs
journalctl -u leyline -f

# Restart
systemctl restart leyline

# Stop
systemctl stop leyline
```

## Data Directory

All persistent state lives in the data directory:

```
{dataDir}/
├── identity.json         # Ed25519 keypair (mode 0600)
├── local-ledger/         # LevelDB — local audit chain
├── shared-ledger/        # LevelDB — distributed provable records
├── trust/                # LevelDB — persistent trust decisions
├── spam/                 # LevelDB — spam report counters
└── seed-peers/           # LevelDB — peer table (seed nodes only)
```

### Backup

Back up the entire data directory to preserve:

- **Identity**: `identity.json` — losing this means a new keypair and all trust relationships reset
- **Ledgers**: `local-ledger/` and `shared-ledger/` — the audit trail and provable records
- **Trust state**: `trust/` — allow/block decisions survive restarts

### Reset

To reset a node to a fresh state while keeping its identity:

```bash
# Stop the node first
rm -rf {dataDir}/local-ledger {dataDir}/shared-ledger {dataDir}/trust {dataDir}/spam
# Restart — ledgers and trust will be recreated empty
```

To fully reset (new identity):

```bash
rm -rf {dataDir}
# Restart — everything regenerated from scratch
```

## Monitoring

### Node Health

```typescript
// Connected peers
console.log(`Peers: ${node.getPeerCount()}`);

// Listening addresses
console.log(`Addrs: ${node.getMultiaddrs().join(', ')}`);

// Ledger sizes
const local = node.getLocalLedger();
const shared = node.getSharedLedger();
console.log(`Local ledger: ${await local.getEntryCount()} entries`);
console.log(`Shared ledger: ${await shared.getEntryCount()} entries`);

// Ledger integrity
console.log(`Local chain valid: ${await local.verify()}`);
console.log(`Shared chain valid: ${await shared.verify()}`);

// Peer exchange state
const pex = node.getPeerExchange();
if (pex) {
  console.log(`Known peers: ${pex.getPeerCount()}`);
}
```

### Seed Node Monitoring

```typescript
const seed = new SeedNode({ ... });
await seed.start();

// Peer tracking
console.log(`Connected: ${seed.getConnectedPeerCount()}`);
console.log(`Known: ${seed.getKnownPeers().length}`);

// Stale peer cleanup
const pruned = seed.pruneStale();
console.log(`Pruned ${pruned} stale peers`);
```

## Graceful Shutdown

Leyline handles `SIGINT` and `SIGTERM` for clean shutdown:

1. Stops the peer exchange interval
2. Stops the ledger sync interval
3. Closes both LevelDB ledgers
4. Stops the libp2p node

This ensures no data corruption. Always stop cleanly rather than killing with `SIGKILL`.

## Firewall

Leyline needs these ports open:

| Port | Protocol | Purpose |
|---|---|---|
| 9876 (default) | TCP | libp2p peer connections |
| 9877 (default) | TCP | WebSocket transport (if enabled) |

For seed nodes, incoming connections are essential. Regular nodes can operate with outbound-only connections to seeds, but peer-to-peer connections improve mesh quality.

## Resource Usage

| Resource | Typical | Notes |
|---|---|---|
| Memory | 50--150 MB | Depends on peer count and dedup cache size |
| Disk | Grows with ledger | LevelDB compacts automatically |
| CPU | Minimal | Spikes during Ed25519 signature verification |
| Bandwidth | Low | GossipSub is efficient; most traffic is peer exchange |

The dedup cache (`maxSeenMessages`, default 100k) is the primary memory consumer. Reduce it for memory-constrained environments.
