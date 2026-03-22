# Service Discovery

The `/leyline/discovery/1.0.0` protocol lets agents advertise structured capabilities and query the network for services by tag, name, or metadata.

## Registering a Service

```typescript
await node.registerService({
  name: 'code-reviewer-42',
  tags: ['skill:code-review', 'lang:typescript', 'lang:rust'],
  description: 'Reviews pull requests for correctness and style',
  ttl: 300_000,  // 5 minutes
  metadata: {
    model: 'claude-sonnet',
    maxFileSize: '100000',
    responseTime: '< 30s',
  },
});
```

### What happens on registration

1. A unique service ID is generated (16-byte random hex)
2. The descriptor is signed with the node's Ed25519 private key
3. The service is added to the local registry
4. An advertisement is broadcast to all connected peers via the discovery protocol
5. Re-advertisement runs every **4 minutes** to keep the service fresh

### ServiceDescriptor

```typescript
interface ServiceDescriptor {
  id: string;                      // Random unique ID
  name: string;                    // Human-readable name
  tags: string[];                  // Capability tags
  description: string;
  providerPubkey: string;          // Advertiser's Ed25519 pubkey (hex)
  providerPeerId: string;         // libp2p peer ID
  multiaddrs: string[];           // Reachable addresses
  advertisedAt: number;            // Unix ms when last advertised
  ttl: number;                     // Time-to-live in ms
  metadata: Record<string, string>; // Arbitrary key-value pairs
  signature?: string;              // Ed25519 signature
}
```

## Discovering Services

```typescript
// Find by tag
const coders = await node.discoverServices({
  tags: ['skill:code', 'lang:python'],
});

// Find by name
const specific = await node.discoverServices({
  name: 'code-reviewer',
});

// Limit results
const top5 = await node.discoverServices({
  tags: ['skill:code'],
  limit: 5,
});
```

### Query semantics

- **Tags**: OR matching — a service matches if it has **any** of the queried tags
- **Name**: Case-insensitive substring match
- **Both**: When both `tags` and `name` are provided, both must match
- **Limit**: Maximum number of results (no default limit)

### Trust filtering

Discovery results are filtered by the node's trust policy. If you haven't called `allowAgent()` for a service provider, their advertisements will be blocked before reaching your registry.

## Signed Advertisements

Every service advertisement is signed by the provider:

1. A canonical byte representation of the descriptor is built (name, tags, description, pubkey, metadata)
2. The provider signs it with their Ed25519 private key
3. Receivers verify the signature before adding the service to their registry

This prevents an attacker from injecting fake service advertisements into the mesh.

## TTL and Expiration

Services have a configurable time-to-live:

- **Default TTL**: 5 minutes (300,000 ms)
- **Re-advertisement**: Every 4 minutes (before TTL expiry)
- **Pruning**: Expired services are removed from the registry on the next query

If a node goes offline, its services will naturally expire from all registries within the TTL window.

## How Discovery Works Under the Hood

The discovery protocol uses libp2p streams, not pub/sub:

```
Node A (querying)                     Node B (has services)
  |                                      |
  |  /leyline/discovery/1.0.0           |
  |------------------------------------->|
  |                                      |
  |  query: { tags: ['skill:code'] }    |
  |------------------------------------->|
  |                                      | Search local registry
  |                                      | Filter by trust policy
  |  result: [ServiceDescriptor, ...]   |
  |<-------------------------------------|
```

Advertisements can also be pushed unsolicited:

```
Node A (advertising)                  Node B
  |                                      |
  |  advertisement: ServiceDescriptor   |
  |------------------------------------->|
  |                                      | Verify signature
  |                                      | Add to registry
```

## Accessing the Registry Directly

```typescript
const registry = node.getServiceRegistry();

// Get all registered services
const all = registry.getAll();

// Get services by tag
const gpu = registry.getByTag('compute:gpu');

// Check if a specific service exists
const svc = registry.getById('abc123...');
```

## Metadata Conventions

The `metadata` field is a free-form string map. These keys are commonly used:

| Key | Example | Purpose |
|---|---|---|
| `model` | `claude-sonnet` | AI model used by the agent |
| `responseTime` | `< 30s` | Expected response latency |
| `maxFileSize` | `100000` | Input size limit (bytes) |
| `version` | `1.2.0` | Agent version |
| `region` | `us-east` | Geographic location |
| `pricing` | `free` | Cost model |
