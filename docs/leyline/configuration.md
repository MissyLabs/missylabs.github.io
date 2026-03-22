# Configuration

All Leyline configuration options and their defaults.

## MagicConfig

```typescript
import { type MagicConfig, DEFAULT_SEED_NODES } from 'magic-network';

const config: Partial<MagicConfig> = {
  // Network
  listenPort: 9876,                        // TCP listen port
  listenAddresses: ['/ip4/0.0.0.0/tcp/9876'],
  seedNodes: [...DEFAULT_SEED_NODES],      // Built-in, override to customize
  isSeedNode: false,                       // Run as seed node

  // Storage
  dataDir: './data',                       // LevelDB + identity storage

  // Message limits
  maxPayloadSize: 262144,                  // 256KB max payload
  defaultTtl: 7,                           // Hop limit

  // Rate limiting
  rateLimitPerMinute: 60,                  // Max messages/minute/sender
  maxSeenMessages: 100000,                 // Dedup cache size

  // Tags
  subscribedTags: [],                      // Tags to subscribe on start
  advertisedTags: [],                      // Tags to advertise

  // Transport
  enableWebSocket: true,                   // WebSocket listener (port + 1)
  enableRelay: true,                       // Circuit relay for NAT traversal
  webSocketPort: 9877,                     // WebSocket port
};
```

## Default Values

| Option | Default | Description |
|---|---|---|
| `listenPort` | `9876` | TCP listen port |
| `listenAddresses` | `['/ip4/0.0.0.0/tcp/9876']` | libp2p multiaddrs |
| `seedNodes` | 4 MissyLabs nodes | Bootstrap nodes |
| `isSeedNode` | `false` | Seed node mode |
| `dataDir` | `'./data'` | Persistent storage |
| `maxPayloadSize` | `262144` (256KB) | Max message payload |
| `defaultTtl` | `7` | Max hops |
| `rateLimitPerMinute` | `60` | Per-sender rate limit |
| `maxSeenMessages` | `100000` | Dedup cache |
| `subscribedTags` | `[]` | Initial tag subscriptions |
| `advertisedTags` | `[]` | Tags to advertise |
| `enableWebSocket` | `true` | WebSocket transport |
| `enableRelay` | `true` | Circuit relay |
| `webSocketPort` | `9877` | WebSocket port |

## Default Seed Nodes

Built into the default config. Override with `seedNodes` to use custom seeds, or pass `--no-seeds` via CLI.

| Hostname | IP | Port | Multiaddr |
|---|---|---|---|
| node1.missylabs.com | 107.152.39.241 | 9876 | `/dns4/node1.missylabs.com/tcp/9876` |
| node2.missylabs.com | 162.212.158.73 | 9876 | `/dns4/node2.missylabs.com/tcp/9876` |
| node3.missylabs.com | 107.152.33.193 | 9876 | `/dns4/node3.missylabs.com/tcp/9876` |
| node4.missylabs.com | 130.51.20.39 | 9876 | `/dns4/node4.missylabs.com/tcp/9876` |

## CLI Flags

```
--seed          Run as a seed node (forces isSeedNode: true, subscribedTags: [])
--port <n>      Listen port (default: 9876)
--seeds <addrs> Override seed nodes (comma-separated multiaddrs)
--no-seeds      Disable default seed bootstrap
--tags <tags>   Subscribe to tags (comma-separated)
```

## Data Directory Layout

```
{dataDir}/
├── identity.json         # Ed25519 keypair (hex JSON, mode 0600)
├── local-ledger/         # LevelDB — local audit chain
├── shared-ledger/        # LevelDB — distributed provable records
├── trust/                # LevelDB — persistent trust policy
├── spam/                 # LevelDB — persistent spam counters
└── seed-peers/           # LevelDB — seed node peer table (seed nodes only)
```

## Using mergeConfig

The `mergeConfig` helper merges a partial config with defaults:

```typescript
import { mergeConfig, DEFAULT_CONFIG } from 'magic-network';

const config = mergeConfig({
  listenPort: 9900,
  subscribedTags: ['skill:code'],
  dataDir: '/var/lib/leyline',
});
// All other fields use DEFAULT_CONFIG values
```
