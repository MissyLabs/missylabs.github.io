# API Reference

Complete API documentation for all public exports of the `magic-network` package.

## MagicNode

`src/node/magic-node.ts` — The main node orchestrator.

### Constructor

```typescript
new MagicNode(config: Partial<MagicConfig>, events?: MagicNodeEvents)
```

```typescript
interface MagicNodeEvents {
  onMessage?: (msg: MagicMessage, tag: string) => void;
  onPeerConnected?: (peerId: string) => void;
  onPeerDisconnected?: (peerId: string) => void;
}
```

### Methods

| Method | Returns | Description |
|---|---|---|
| `start()` | `Promise<void>` | Initialize and start all subsystems |
| `stop()` | `Promise<void>` | Graceful shutdown |
| `broadcast(tags, payload, type?)` | `Promise<MagicMessage>` | Broadcast a signed message |
| `advertise(tags, payload)` | `Promise<MagicMessage>` | Broadcast with `ADVERTISE` type |
| `discover(tags, query)` | `Promise<MagicMessage>` | Broadcast with `DISCOVER` type |
| `sendDirect(peerId, payload, pubkeyHex)` | `Promise<boolean>` | Encrypted point-to-point message |
| `registerService(descriptor)` | `Promise<void>` | Register a service for discovery |
| `discoverServices(query)` | `Promise<ServiceDescriptor[]>` | Query service registry |
| `subscribe(tag)` | `void` | Subscribe to a tag at runtime |
| `unsubscribe(tag)` | `void` | Unsubscribe from a tag |
| `onTag(tag, handler)` | `void` | Register per-tag message handler |
| `allowAgent(pubkeyHex)` | `void` | Whitelist an agent |
| `blockAgent(pubkeyHex)` | `void` | Blacklist an agent |
| `allowTag(pubkeyHex, tag)` | `void` | Per-tag trust grant |
| `blockTag(pubkeyHex, tag)` | `void` | Per-tag trust block |
| `submitToSharedLedger(data)` | `Promise<void>` | Submit provable record |
| `getPublicKeyHex()` | `string` | 64-char hex public key |
| `getFingerprint()` | `string` | 16-char hex short ID |
| `getPeerCount()` | `number` | Connected peer count |
| `getMultiaddrs()` | `string[]` | Listening addresses |
| `getLocalLedger()` | `LocalLedger` | Local ledger instance |
| `getSharedLedger()` | `SharedLedger` | Shared ledger instance |
| `getPeerExchange()` | `PeerExchange \| null` | Peer exchange instance |
| `getLedgerSync()` | `LedgerSync \| null` | Ledger sync instance |
| `getServiceRegistry()` | `ServiceRegistry` | Service registry instance |
| `getLedgerConsensus()` | `Consensus` | Consensus state |

---

## SeedNode

`src/node/seed-node.ts` — Extends `MagicNode` for bootstrap nodes.

```typescript
new SeedNode(config: Partial<MagicConfig>)
```

Forces `isSeedNode: true` and `subscribedTags: []`.

| Method | Returns | Description |
|---|---|---|
| `getKnownPeers()` | `Array<{peerId, multiaddrs, lastSeen}>` | All seen peers |
| `getConnectedPeerCount()` | `number` | Currently connected peers |
| `pruneStale(maxAge?)` | `number` | Remove old peers (default: 30 min) |

---

## Identity

### generateKeypair

```typescript
async function generateKeypair(): Promise<Keypair>
// Keypair = { publicKey: Uint8Array, privateKey: Uint8Array }
```

### sign / verify

```typescript
async function sign(privateKey: Uint8Array, data: Uint8Array): Promise<Uint8Array>
async function verify(publicKey: Uint8Array, signature: Uint8Array, data: Uint8Array): Promise<boolean>
```

### Key encoding

```typescript
function publicKeyToHex(pubkey: Uint8Array): string      // 64-char hex
function hexToPublicKey(hex: string): Uint8Array          // 32 bytes
function getFingerprint(pubkey: Uint8Array): string       // 16-char hex
```

### IdentityStore

```typescript
const store = new IdentityStore(dataDir: string);
await store.load(): Promise<Keypair>      // Load or generate
await store.save(privateKey, publicKey)   // Save to disk
store.exists(): Promise<boolean>
store.getIdentityPath(): string
```

---

## Messages

### MagicMessage

```typescript
interface MagicMessage {
  id: Uint8Array;            // SHA-256 (32 bytes)
  senderPubkey: Uint8Array;  // Ed25519 public key (32 bytes)
  signature: Uint8Array;     // Ed25519 signature (64 bytes)
  tags: string[];            // 1-20 routing tags
  payload: Uint8Array;       // Max 256KB
  timestamp: number;         // Unix milliseconds
  nonce: Uint8Array;         // 16 random bytes
  type: MessageType;
  ttl: number;               // Hop count
}
```

### MessageType

```typescript
enum MessageType {
  BROADCAST = 1,
  DIRECT = 2,
  ADVERTISE = 3,
  DISCOVER = 4,
  DISCOVER_RESPONSE = 5,
}
```

### Message functions

```typescript
// Create a signed message
async function createMessage(opts: {
  tags: string[];
  payload: Uint8Array;
  type: MessageType;
  privateKey: Uint8Array;
  publicKey: Uint8Array;
  ttl?: number;  // default: 7
}): Promise<MagicMessage>

// Serialize / deserialize (format: 'protobuf' | 'json')
function serializeMessage(msg: MagicMessage, format?: 'protobuf'): Uint8Array
function deserializeMessage(data: Uint8Array, format?: 'protobuf'): MagicMessage
function serializeMessageJson(msg: MagicMessage): Uint8Array
function deserializeMessageJson(data: Uint8Array): MagicMessage

// Validate structure (NOT signature)
function validateMessage(msg: MagicMessage): { valid: boolean; error?: string }

// Verify Ed25519 signature
async function verifyMessageSignature(msg: MagicMessage): Promise<boolean>

// Load protobuf schema (call once before serialization)
async function initProto(): Promise<void>
```

---

## TagPubSub

```typescript
const pubsub = new TagPubSub(gossipsub: GossipSub);

pubsub.subscribe(tag)
pubsub.unsubscribe(tag)
pubsub.subscribeDiscovery()
pubsub.onTag(tag, handler: (data: Uint8Array, tag: string) => void)
pubsub.offTag(tag, handler)
pubsub.onMessage(handler)
pubsub.offMessage(handler)
await pubsub.publish(tags: string[], data: Uint8Array)
await pubsub.publishDiscovery(data: Uint8Array)
pubsub.handleMessage(topic: string, data: Uint8Array)
pubsub.getSubscribedTags(): string[]
pubsub.getTopics(): string[]
pubsub.getTagPeerCount(tag): number
```

**Topic mapping**: Tag `T` maps to GossipSub topic `magic/tag/T`. Discovery uses `magic/discovery`.

---

## Trust

### TrustPolicy

```typescript
const policy = new TrustPolicy();

policy.allowAgent(pubkeyHex)
policy.blockAgent(pubkeyHex)
policy.allowTag(pubkeyHex, tag)
policy.blockTag(pubkeyHex, tag)
policy.isAllowed(pubkeyHex, tags: string[]): boolean
policy.getBlockedAgents(): string[]
policy.getAllowedAgents(): string[]
```

### SpamFilter

```typescript
const filter = new SpamFilter(maxSeenSize?: number);  // default: 100_000

filter.isDuplicate(messageIdHex): boolean
filter.isRateLimited(pubkeyHex, maxPerMinute: number): boolean
filter.reportSpam(pubkeyHex): void
filter.getSpamCount(pubkeyHex): number
```

---

## Ledgers

### LocalLedger

```typescript
const ledger = new LocalLedger(dataDir);
await ledger.open()
await ledger.close()
await ledger.append(message: Uint8Array, action: string): Promise<LedgerEntry>
await ledger.getEntry(index): Promise<LedgerEntry | null>
await ledger.getLatest(): Promise<LedgerEntry | null>
await ledger.verify(): Promise<boolean>
await ledger.getEntryCount(): Promise<number>
```

### SharedLedger

```typescript
const ledger = new SharedLedger(dataDir);
await ledger.open()
await ledger.close()
await ledger.submit(data, submitterPubkey, signature): Promise<SharedLedgerEntry>
await ledger.addConfirmation(index, confirmerPubkey): Promise<SharedLedgerEntry | null>
await ledger.getEntry(index): Promise<SharedLedgerEntry | null>
await ledger.getLatest(): Promise<SharedLedgerEntry | null>
await ledger.getEntryCount(): Promise<number>
await ledger.verify(): Promise<boolean>
await ledger.getRange(startIndex, endIndex): Promise<SharedLedgerEntry[]>
```

### LedgerSync

```typescript
const sync = new LedgerSync(libp2p, ledger, localPubkey, localPrivkey, {
  syncIntervalMs?: number,  // default: 60_000
  events?: {
    onEntryReceived?: (entry) => void,
    onEntryConfirmed?: (index, pubkey) => void,
    onSyncComplete?: (peerId, count) => void,
  },
});

await sync.start()
await sync.stop()
await sync.requestRange(peerId, startIndex, endIndex): Promise<SharedLedgerEntry[]>
await sync.pushEntry(peerId, entry): Promise<void>
await sync.broadcastEntry(entry): Promise<void>
await sync.syncWithAllPeers(): Promise<void>
```

---

## PeerExchange

```typescript
const pex = new PeerExchange(libp2p, {
  maxPeers?: number,           // default: 500
  maxPeerAge?: number,         // default: 30 min
  exchangeIntervalMs?: number, // default: 30s
});

await pex.start()
await pex.stop()
pex.addPeer(record: PeerRecord)
pex.removePeer(peerId)
pex.getPeers(): PeerRecord[]
pex.getPeer(peerId): PeerRecord | undefined
pex.getPeerCount(): number
pex.pruneStale(): number
await pex.exchangeWithPeer(peerId): Promise<PeerRecord[]>
```

---

## Configuration

```typescript
import { type MagicConfig, DEFAULT_CONFIG, mergeConfig } from 'magic-network';

const config = mergeConfig({ listenPort: 9900 });
// Spreads partial over DEFAULT_CONFIG
```

See [Configuration](configuration.md) for all options and defaults.
