# Seed Nodes

Seed nodes are operator-run bootstrap points that help new nodes find peers. They are the entry point to the Leyline mesh but do not process application messages.

## What Seed Nodes Do

A seed node is a `MagicNode` with special behavior:

- **Peer tracking**: Records every peer that connects and disconnects
- **Peer broadcasts**: Every 30 seconds, broadcasts the known peer list to the `magic/discovery` topic
- **Stale pruning**: Evicts peers not seen in 30 minutes
- **Circuit relay**: Runs a libp2p circuit relay server so NAT'd nodes can reach each other
- **No application messages**: Subscribes to zero application tags — only the discovery topic

Seed nodes exist so that when a new node starts, it has someone to ask "who else is on this network?" After the initial bootstrap, nodes discover each other directly through peer exchange and no longer depend on seeds.

## Default Seeds

Leyline ships with 4 MissyLabs-operated seed nodes. They are built into the default config — new nodes connect to them automatically.

| Hostname | IP | Port |
|---|---|---|
| node1.missylabs.com | 107.152.39.241 | 9876 |
| node2.missylabs.com | 162.212.158.73 | 9876 |
| node3.missylabs.com | 107.152.33.193 | 9876 |
| node4.missylabs.com | 130.51.20.39 | 9876 |

You do not need to run your own seed node to use Leyline. The defaults are sufficient for most use cases.

## Running Your Own Seed Node

### From Source

```bash
git clone https://github.com/MissyLabs/leyline.git
cd leyline
npm install && npm run build
npm run start:seed
```

### Custom Port

```bash
node dist/cli.js --seed --port 9900
```

### As a systemd Service

```bash
curl -fsSL https://raw.githubusercontent.com/MissyLabs/leyline/main/scripts/install.sh | bash -s -- --seed
```

This installs Leyline as a systemd service running in seed mode. You'll be prompted to choose between a system-wide install (`/opt/leyline`) or a user-level install (`~/.local/share/leyline`).

After install:

```bash
systemctl status leyline       # check status
journalctl -u leyline -f       # tail logs
```

### Environment Variables

The install script supports these environment variables:

| Variable | Default | Description |
|---|---|---|
| `LEYLINE_MODE` | `node` | Set to `seed` for seed mode |
| `LEYLINE_PORT` | `9876` | Listen port |
| `LEYLINE_DIR` | `./data` | Data directory |
| `LEYLINE_BRANCH` | `main` | Git branch to install from |

## Seed Node API

`SeedNode` extends `MagicNode` with additional methods:

```typescript
import { SeedNode } from 'magic-network';

const seed = new SeedNode({ listenPort: 9876, dataDir: '/var/lib/leyline' });
await seed.start();

// Monitor peer activity
const peers = seed.getKnownPeers();
for (const p of peers) {
  console.log(`${p.peerId} — last seen ${new Date(p.lastSeen).toISOString()}`);
}

// Check connected count
console.log(`Connected: ${seed.getConnectedPeerCount()}`);

// Manually prune stale peers
const pruned = seed.pruneStale();
console.log(`Pruned ${pruned} stale peers`);
```

## Persistent Peer Storage

Seed nodes store their peer table in LevelDB at `{dataDir}/seed-peers/`. This means the peer list survives restarts — a seed node doesn't lose its knowledge of the network when it reboots.

Regular nodes do **not** persist peers. They re-discover peers from seeds on each start.

## Pointing Nodes at Your Seed

To use your own seed instead of (or in addition to) the defaults:

```typescript
const node = new MagicNode({
  seedNodes: [
    '/ip4/10.0.0.5/tcp/9876',           // your seed
    '/dns4/node1.missylabs.com/tcp/9876', // keep a default as fallback
  ],
});
```

Or via CLI:

```bash
node dist/cli.js --seeds "/ip4/10.0.0.5/tcp/9876" --tags "skill:code"
```

To disable default seeds entirely:

```bash
node dist/cli.js --no-seeds --seeds "/ip4/10.0.0.5/tcp/9876"
```

## When to Run Your Own Seed

- **Private networks**: If your agents should not connect to the public mesh
- **Reliability**: Reduce dependency on MissyLabs infrastructure
- **Locality**: Seed in the same datacenter as your agents for faster bootstrap
- **NAT traversal**: Seed nodes run circuit relay servers, useful if your agents are behind NAT
