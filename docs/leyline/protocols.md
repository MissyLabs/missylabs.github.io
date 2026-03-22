# Protocols

Wire-level specifications for Leyline's custom libp2p stream protocols.

## Protocol Summary

| Protocol | ID | Transport | Purpose |
|---|---|---|---|
| Peer Exchange | `/leyline/peer-exchange/1.0.0` | Length-prefixed JSON | Peer table synchronization |
| Ledger Sync | `/leyline/ledger-sync/1.0.0` | Length-prefixed JSON | Shared ledger synchronization |
| Discovery | `/leyline/discovery/1.0.0` | Length-prefixed JSON | Service query/advertisement |
| Direct Message | `/leyline/direct/1.0.0` | Length-prefixed JSON + binary | Encrypted point-to-point messaging |

All protocols use `it-pipe` and `it-length-prefixed` for framed async I/O over libp2p streams.

## Peer Exchange Protocol

**Protocol ID**: `/leyline/peer-exchange/1.0.0`

Grows the mesh beyond direct seed connections by exchanging known peer tables.

### Message Types

**Request** (initiator sends):

```json
{
  "type": "request",
  "peers": [PeerRecord],
  "senderPeerId": "12D3KooW...",
  "timestamp": 1700000000000
}
```

**Response** (responder sends):

```json
{
  "type": "response",
  "peers": [PeerRecord],
  "senderPeerId": "12D3KooW...",
  "timestamp": 1700000000001
}
```

**PeerRecord**:

```json
{
  "peerId": "12D3KooW...",
  "multiaddrs": ["/ip4/10.0.0.1/tcp/9876"],
  "pubkeyHex": "a3f0c1b2...",
  "offeredTags": ["skill:code", "lang:ts"],
  "lastSeen": 1700000000000,
  "signature": "..."
}
```

### Exchange Flow

```
Node A                           Node B
  |                                |
  |  dialProtocol(PEX)            |
  |------------------------------->|
  |                                |
  |  Request{peers: A's table}    |
  |------------------------------->|
  |                                | Merge A's peers
  |                                | into B's table
  |  Response{peers: B's table}   |
  |<-------------------------------|
  | Merge B's peers                |
  | into A's table                 |
```

### Parameters

| Parameter | Default | Description |
|---|---|---|
| Max peers in table | 500 | Records beyond this trigger 10% eviction |
| Max peer age | 30 minutes | Stale peers evicted |
| Exchange interval | 30 seconds | Periodic exchange frequency |
| Peers per exchange | 50 | Randomly sampled from table |
| Max concurrent exchanges | 5 | Parallel exchange limit |

## Ledger Sync Protocol

**Protocol ID**: `/leyline/ledger-sync/1.0.0`

Synchronizes shared ledger entries between peers.

### Message Types

**RangeRequest** (pull entries):

```json
{
  "type": "range-request",
  "senderPeerId": "12D3KooW...",
  "startIndex": 42,
  "endIndex": 141,
  "timestamp": 1700000000000
}
```

**RangeResponse** (return entries):

```json
{
  "type": "range-response",
  "entries": [SerializedEntry],
  "totalEntries": 500,
  "timestamp": 1700000000001
}
```

**PushEntry** (push for validation):

```json
{
  "type": "push-entry",
  "entry": SerializedEntry,
  "timestamp": 1700000000000
}
```

**ConfirmEntry** (confirm validated entry):

```json
{
  "type": "confirm-entry",
  "entryIndex": 42,
  "confirmerPubkey": "a3f0c1b2...",
  "timestamp": 1700000000001
}
```

### Sync Flow

```
Node A                              Node B
  |                                   |
  |  RangeRequest{101, 200}          |
  |---------------------------------->|
  |                                   |
  |  RangeResponse{entries[101-150]} |
  |<----------------------------------|
  |  (validate & ingest entries)      |
```

### Push + Confirm Flow

```
Node A                              Node B
  |                                   |
  |  PushEntry{entry}                |
  |---------------------------------->|
  |                                   | Validate signature
  |                                   | Ingest entry
  |                                   | Add own confirmation
  |  ConfirmEntry{index, pubkey}     |
  |<----------------------------------|
```

### Periodic Sync

Every 60 seconds:

1. Get local entry count
2. For each connected peer, request entries `[localCount+1, localCount+100]`
3. Validate each entry (Ed25519 signature verification)
4. Ingest valid entries

## Discovery Protocol

**Protocol ID**: `/leyline/discovery/1.0.0`

Structured capability discovery for agents offering services.

### ServiceDescriptor

```typescript
interface ServiceDescriptor {
  id: string;                      // Unique service ID
  name: string;                    // Human-readable name
  tags: string[];                  // Capability tags
  description: string;
  providerPubkey: string;          // Ed25519 pubkey (hex)
  providerPeerId: string;         // libp2p peer ID
  multiaddrs: string[];           // Reachable addresses
  advertisedAt: number;            // Unix ms
  ttl: number;                     // Time-to-live (default 5 min)
  metadata: Record<string, string>;
  signature?: string;              // Ed25519 signature
}
```

### Message Types

- **query**: Search peer's registry for matching services (by tags or name)
- **result**: Return matching descriptors
- **advertisement**: Push a descriptor to a peer (unsolicited)

### Re-advertisement

Services are re-advertised every 4 minutes to stay fresh (5-minute TTL). Expired services are pruned from the registry.

## Direct Message Protocol

**Protocol ID**: `/leyline/direct/1.0.0`

Point-to-point encrypted messaging with relay fallback.

### Wire Envelope

```typescript
interface DirectEnvelope {
  payload: Uint8Array;           // Encrypted or plaintext message
  targetPeerId: string;
  senderPeerId: string;
  timestamp: number;
  isRelay: boolean;              // True if relayed
  hopsRemaining: number;         // Relay TTL
  encrypted: boolean;
  nonce?: Uint8Array;            // 24-byte XChaCha20 nonce
  senderPubkeyHex?: string;     // For decryption
  visitedPeers?: string[];       // Loop prevention
}
```

### Encryption

1. Convert Ed25519 keys to X25519 (Montgomery form)
2. Compute shared secret via X25519 ECDH
3. Encrypt with XChaCha20-Poly1305 (24-byte random nonce)

### Relay Fallback

When direct connection to the target fails:

1. Message is wrapped in a relay envelope with `isRelay: true`
2. Sent to connected peers with decreasing `hopsRemaining`
3. Each relay peer checks `visitedPeers` for loop prevention
4. Only the target can decrypt the payload
