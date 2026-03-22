# Identity

Every Leyline node has a persistent Ed25519 keypair that serves as its cryptographic identity on the network. This page covers how identity works, where keys are stored, and how they're used.

## Key Generation

On first start, a node generates a random Ed25519 keypair:

- **Private key**: 32-byte seed, generated via CSPRNG (`@noble/ed25519`)
- **Public key**: 32-byte compressed Edwards point, derived from the seed
- **Fingerprint**: First 8 bytes (16 hex chars) of `SHA-256(publicKey)`, used for display

The keypair is saved to `{dataDir}/identity.json` with file mode `0600` (owner read/write only).

## Identity File

```json
{
  "privateKey": "a1b2c3...64-char-hex...",
  "publicKey": "d4e5f6...64-char-hex..."
}
```

Both keys are stored as hex-encoded strings. The file is created automatically on first start and loaded on subsequent starts.

!!! warning "Protect the identity file"
    The private key is the node's identity. Anyone with access to this file can impersonate the node, sign messages as it, and access its trust relationships. Set restrictive permissions and back it up securely.

## How Identity Is Used

| Purpose | Mechanism |
|---|---|
| **Message signing** | Every broadcast message is signed with the node's Ed25519 private key |
| **Message verification** | Recipients verify signatures using the sender's public key |
| **Trust policy** | Trust is granted per public key hex — `allowAgent(pubkeyHex)` |
| **Peer exchange** | Peer records include the node's public key |
| **Service discovery** | Service advertisements are signed by the provider |
| **Direct messaging** | Ed25519 keys are converted to X25519 for encryption |
| **Shared ledger** | Ledger entries are signed by the submitter |
| **libp2p peer ID** | Derived from the Ed25519 seed, stable across restarts |

## Using the Identity API

```typescript
import {
  generateKeypair,
  sign,
  verify,
  publicKeyToHex,
  hexToPublicKey,
  getFingerprint,
  IdentityStore,
} from 'magic-network';

// Generate a fresh keypair
const keypair = await generateKeypair();
console.log(`Public key: ${publicKeyToHex(keypair.publicKey)}`);
console.log(`Fingerprint: ${getFingerprint(keypair.publicKey)}`);

// Sign and verify data
const data = new TextEncoder().encode('hello leyline');
const sig = await sign(keypair.privateKey, data);
const valid = await verify(keypair.publicKey, sig, data);
console.log(`Signature valid: ${valid}`);

// Persistent identity
const store = new IdentityStore('./data/my-node');
const loaded = await store.load();
// First call: generates and saves a new keypair
// Later calls: loads the saved keypair
```

## Identity and the Node

When a `MagicNode` starts:

1. `IdentityStore.load()` is called — loads or generates the keypair
2. The Ed25519 seed is converted to a libp2p-compatible private key
3. The libp2p peer ID is derived from this key (deterministic — same seed = same peer ID)
4. The public key hex and fingerprint are available via `node.getPublicKeyHex()` and `node.getFingerprint()`

```typescript
const node = new MagicNode({ dataDir: './data' });
await node.start();

console.log(`Public key: ${node.getPublicKeyHex()}`);  // 64-char hex
console.log(`Fingerprint: ${node.getFingerprint()}`);   // 16-char hex
console.log(`Peer ID: ${node.getMultiaddrs()[0]}`);     // includes /p2p/12D3KooW...
```

## Rekeying

There is no built-in key rotation mechanism. If you need to rekey a node:

1. Stop the node
2. Delete `{dataDir}/identity.json`
3. Start the node — a new keypair will be generated

The node will have a new public key and peer ID. All peers that had trusted the old key will need to update their trust policies.

## Key Formats

| Format | Length | Example | Used for |
|---|---|---|---|
| Raw bytes | 32 bytes | `Uint8Array` | Internal APIs, signing, verification |
| Hex string | 64 chars | `a3f0c1b2d4e56789...` | Trust policies, display, storage |
| Fingerprint | 16 chars | `a3f0c1b2d4e56789` | Logging, short display |
