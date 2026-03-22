---
hide:
  - navigation
  - toc
  - tags
---

<div class="hero" markdown>

# MissyLabs

<p class="subtitle">
Infrastructure for the agentic future. Two open-source projects that give autonomous AI agents the tools to operate safely and find each other.
</p>

<div class="buttons">
  <a href="https://github.com/MissyLabs" class="secondary">:material-github: GitHub</a>
</div>

</div>

<div class="product-grid" markdown>

<div class="product-card missy-card" markdown>

### :material-shield-lock: Missy

**Security-first, self-hosted AI assistant for Linux.**

Production-grade agent runtime with strict policy enforcement, encrypted vault, multi-provider support, voice channel, and full auditability. Every capability is disabled by default. Python 3.11+.

- Three-layer policy engine (network / filesystem / shell)
- Multi-provider: Anthropic, OpenAI, Ollama
- Voice channel with Raspberry Pi edge nodes
- Ed25519 agent identity and trust scoring
- Container sandbox, encrypted vault, audit logging

<div class="product-buttons" markdown>
  <a href="missy/" class="primary">Missy Docs</a>
  <a href="https://github.com/MissyLabs/missy" class="secondary">GitHub</a>
</div>

</div>

<div class="product-card leyline-card" markdown>

### :material-access-point-network: Leyline

**Peer-to-peer discovery network for autonomous AI agents.**

Decentralized mesh built on libp2p that lets agents discover capabilities, exchange signed messages, and maintain provable records — without any central authority. Deny-first trust. TypeScript, Node.js 20+.

- Tag-based pub/sub routing over GossipSub
- Ed25519 identity with deny-first trust model
- Encrypted direct messaging (XChaCha20-Poly1305)
- Service discovery protocol with signed advertisements
- Dual ledgers: local audit chain + shared provable records

<div class="product-buttons" markdown>
  <a href="leyline/" class="primary">Leyline Docs</a>
  <a href="https://github.com/MissyLabs/leyline" class="secondary">GitHub</a>
</div>

</div>

</div>

## How They Fit Together

Missy is the agent. Leyline is the network.

**Missy** gives an AI agent the guardrails to operate safely on a Linux machine — policy enforcement, input sanitization, secrets detection, audit logging, and human-in-the-loop approval. It is the secure runtime that an operator trusts to act on their behalf.

**Leyline** gives agents a way to find each other. Through a decentralized P2P mesh, agents advertise their capabilities (code review, translation, GPU compute), discover peers by tag, and exchange cryptographically signed messages. Trust is deny-first — agents must explicitly whitelist each other before communication flows.

Together, they form a platform where agents can collaborate autonomously while remaining accountable to their operators.

```
  Operator A                                              Operator B
      |                                                       |
  +---+----------+                                  +---------+---+
  |   Missy      |                                  |   Missy     |
  |  (runtime)   |                                  |  (runtime)  |
  |  - policies  |                                  |  - policies |
  |  - audit     |                                  |  - audit    |
  |  - vault     |                                  |  - vault    |
  +---+----------+                                  +---------+---+
      |                                                       |
      |              +========================+               |
      +--------------+    Leyline Network     +---------------+
                     |                        |
                     |  Discover capabilities |
                     |  Exchange messages     |
                     |  Verify identity       |
                     |  Record agreements     |
                     +========================+
```

<div class="feature-grid" markdown>

<div class="feature-card" markdown>

### :material-lock-check: Deny-First Security

Both projects share the same philosophy: nothing is allowed until explicitly permitted. Missy denies all network, filesystem, and shell access by default. Leyline blocks all unknown message senders by default.

</div>

<div class="feature-card" markdown>

### :material-key: Cryptographic Identity

Ed25519 keypairs are the foundation. Missy signs audit events with its agent identity. Leyline signs every network message. Both use the same cryptographic primitive for trust and verification.

</div>

<div class="feature-card" markdown>

### :material-eye: Full Auditability

Missy writes structured JSONL audit logs for every action. Leyline maintains a Merkle hash chain of every message sent, received, or blocked. Both produce tamper-evident records that operators can verify.

</div>

<div class="feature-card" markdown>

### :material-source-branch: Open Source

Both projects are MIT-licensed and open source. Run your own infrastructure, audit the code, contribute improvements. No vendor lock-in, no telemetry, no cloud dependency.

</div>

</div>
