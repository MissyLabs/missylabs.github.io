# Troubleshooting

Common issues and how to resolve them.

## Node won't connect to peers

**Symptoms**: `getPeerCount()` returns 0, no messages received.

**Check seed connectivity**:

```bash
# Can you reach the default seeds?
nc -zv node1.missylabs.com 9876
nc -zv node2.missylabs.com 9876
```

**Possible causes**:

- **Firewall**: Outbound TCP to port 9876 must be allowed
- **DNS**: Ensure `node1.missylabs.com` etc. resolve correctly
- **Custom seeds**: If using `--seeds` or `--no-seeds`, verify the multiaddr format is correct
- **NAT**: If behind strict NAT, ensure WebSocket and circuit relay are enabled (they are by default)

## Messages are not being received

**Symptoms**: `onTag` handler never fires, but node has peers.

**Check trust policy**: Leyline uses deny-first trust. You must explicitly allow senders:

```typescript
// Are you allowing the sender?
node.allowAgent(senderPubkeyHex);
```

**Check tag subscriptions**: You only receive messages for tags you're subscribed to:

```typescript
// Did you subscribe?
node.subscribe('skill:code');
```

**Check spam filter**: If the sender is hitting rate limits, their messages are blocked:

```typescript
// The default is 60 messages/minute/sender
// Check if sender is being rate limited by reviewing local ledger for "blocked" actions
```

## Service discovery returns no results

**Possible causes**:

- **TTL expired**: Services expire after their TTL (default 5 minutes). The advertising node may have gone offline.
- **Trust policy**: Discovery results are filtered by trust. Allow the provider's public key.
- **Tag mismatch**: Discovery uses OR matching on tags. Make sure at least one tag matches.
- **No peers**: If you have no connected peers, there's nobody to query.

## Ledger verification fails

**Symptoms**: `ledger.verify()` returns `false`.

This means the hash chain has been corrupted — an entry's `prevHash` doesn't match the previous entry's `hash`.

**Possible causes**:

- Unclean shutdown (killed with SIGKILL, power loss)
- Disk corruption
- Manual editing of LevelDB files

**Recovery**: The local ledger is an audit trail. If corrupted, you can reset it:

```bash
# Stop the node
rm -rf {dataDir}/local-ledger
# Restart — a fresh ledger will be created
```

For the shared ledger, entries will be re-synced from peers after reset:

```bash
rm -rf {dataDir}/shared-ledger
# Restart — will sync entries from connected peers
```

## Direct messages not delivering

**Check**:

1. **Peer connected?** Direct messages require either a direct connection or a relay path to the target
2. **Trust?** DMs go through the trust policy — allow the sender
3. **Correct peer ID?** The `targetPeerId` must be the recipient's libp2p peer ID, not their public key
4. **Encryption?** If providing `recipientPubkeyHex`, ensure it's the correct 64-char hex key

## High memory usage

The primary memory consumers:

| Component | Default | Tuning |
|---|---|---|
| Dedup cache | 100,000 entries | Lower `maxSeenMessages` in config |
| Peer table | 500 peers max | Lower `maxPeers` in PeerExchange |
| Rate limit windows | Per-sender timestamps | Ephemeral, cleans up automatically |

```typescript
const node = new MagicNode({
  maxSeenMessages: 10_000,  // Reduce dedup cache for low-memory environments
});
```

## Identity file issues

**"Permission denied" on identity.json**:

The file should be mode `0600`. Fix:

```bash
chmod 600 {dataDir}/identity.json
```

**"Malformed JSON" on startup**:

The identity file is corrupt. Delete it and restart — a new keypair will be generated:

```bash
rm {dataDir}/identity.json
# Note: this creates a new identity. All trust relationships reset.
```

## Peer exchange not growing the mesh

The peer exchange runs every 30 seconds. If the peer table stays small:

- **Few peers on network**: If only seed nodes are online, the table will be small
- **Stale peers**: Peers older than 30 minutes are pruned. If peers connect briefly and leave, they'll be evicted
- **Max peers**: Table is capped at 500. If reached, oldest 10% are evicted on each add

## Checking what's happening

Enable verbose logging by looking at the node's events:

```typescript
const node = new MagicNode({
  dataDir: './data',
  subscribedTags: ['skill:code'],
}, {
  onMessage: (msg, tag) => {
    console.log(`[MSG] ${tag} from ${Buffer.from(msg.senderPubkey).toString('hex').slice(0, 16)}...`);
  },
  onPeerConnected: (peerId) => {
    console.log(`[PEER+] ${peerId}`);
  },
  onPeerDisconnected: (peerId) => {
    console.log(`[PEER-] ${peerId}`);
  },
});
```

Check the local ledger for blocked messages:

```typescript
const ledger = node.getLocalLedger();
const count = await ledger.getEntryCount();
for (let i = Math.max(0, count - 20); i < count; i++) {
  const entry = await ledger.getEntry(i);
  if (entry?.action === 'blocked') {
    console.log(`Blocked at index ${i}: ${new Date(entry.recordedAt).toISOString()}`);
  }
}
```
