# Consensus

The shared ledger uses a quorum-based pre-commit consensus mechanism to finalize entries. This ensures that provable records are confirmed by multiple peers before being considered committed.

## Overview

When a node submits data to the shared ledger, the entry goes through a consensus process:

1. **Propose**: The submitter signs the data and broadcasts it to connected peers
2. **Validate**: Each peer verifies the Ed25519 signature
3. **Confirm**: Valid entries receive confirmations from peers
4. **Commit**: Once the quorum threshold is reached, the entry is considered finalized

## Configuration

```typescript
interface ConsensusConfig {
  quorumSize: number;           // Min confirmations (default: 2)
  proposalTimeoutMs: number;    // Proposal expiry (default: 30,000 ms)
  maxPendingEntries: number;    // Max in-flight proposals (default: 100)
  maxClockSkewMs: number;       // Future timestamp tolerance (default: 60,000 ms)
}
```

## Proposal Lifecycle

```
PENDING ──(quorum reached)──> CONFIRMED
   |
   └──(timeout expired)──> REJECTED (eligible for re-proposal)
```

### States

| State | Meaning |
|---|---|
| `pending` | Broadcast to peers, awaiting confirmations |
| `confirmed` | Received enough confirmations to meet quorum |
| `rejected` | Timed out without reaching quorum |

### Deduplication

Each proposal is identified by a content hash: `SHA-256(data || timestamp || submitterPubkey)`. If a proposal with the same hash already exists and is still pending, the duplicate is rejected.

### Ordering

When multiple proposals are confirmed, they are ordered by:

1. **Timestamp** (earliest first)
2. **Hash** (lexicographic, for ties)

This deterministic ordering ensures all nodes agree on entry sequence.

## How Confirmations Work

When a peer receives a proposal via the ledger sync protocol:

1. Verify the submitter's Ed25519 signature over the data
2. If valid, add the entry to the local shared ledger
3. Send a `confirm-entry` message back with the confirmer's public key
4. The originator records the confirmation

Confirmations are **idempotent** — a peer's public key can only appear once in the confirmer list.

## Submitting Data

```typescript
// Submit a provable record
await node.submitToSharedLedger(
  new TextEncoder().encode(JSON.stringify({
    type: 'task-completion',
    task: 'code-review-pr-42',
    result: 'approved',
    agent: node.getPublicKeyHex(),
    timestamp: Date.now(),
  })),
);
```

The node signs the data automatically and broadcasts it to all connected peers for confirmation.

## Querying Consensus State

```typescript
const consensus = node.getLedgerConsensus();

// Check pending proposals
const pending = consensus.getPending();
console.log(`${pending.length} proposals awaiting confirmation`);

// Check if a specific entry is confirmed
const entry = await node.getSharedLedger().getEntry(42);
if (entry) {
  console.log(`Entry 42: ${entry.confirmations} confirmations`);
  console.log(`Confirmers: ${entry.confirmerPubkeys.length}`);
}
```

## Limitations

!!! warning "Not Byzantine fault tolerant"
    The consensus mechanism assumes an honest majority of peers. It is designed for networks where seed operators are trusted. A colluding majority could confirm invalid entries.

- **No finality guarantee**: Entries can be confirmed but there is no formal finality — the quorum threshold is a practical measure, not a cryptographic proof
- **Clock dependence**: Ordering relies on timestamps; nodes with significantly skewed clocks may produce entries that are rejected or misordered
- **Network partitions**: If a node cannot reach enough peers to meet quorum, its proposals will time out
