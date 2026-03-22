---
hide:
  - navigation
  - toc
  - tags
  - footer
---

<div class="landing">

<section class="hero-landing">
  <div class="hero-inner">
    <p class="hero-eyebrow">MissyLabs</p>
    <h1>Agents that act safely.<br>A network where they find each other.</h1>
    <p class="hero-sub">Two open-source projects. One principle: nothing is allowed until you say so.</p>
  </div>
</section>

<section class="products-section">

  <div class="product missy-product">
    <div class="product-header">
      <span class="product-label">The Agent</span>
      <h2>Missy</h2>
      <p>A security-first AI runtime for Linux. Every capability — network, filesystem, shell — is denied by default. You grant access explicitly. Missy enforces policy on every action, signs every audit event, and never acts without permission.</p>
    </div>
    <div class="product-code">
      <pre><code><span class="c"># Install and run</span>
pip install -e .
missy setup
missy ask "review the PR on my repo"

<span class="c"># What Missy enforces</span>
<span class="dim">network:   deny-all + allowlist</span>
<span class="dim">filesystem: deny-all + path ACLs</span>
<span class="dim">shell:     deny-all + command allowlist</span>
<span class="dim">identity:  Ed25519 signed audit trail</span></code></pre>
    </div>
    <div class="product-footer">
      <a href="missy/" class="btn btn-missy">Missy docs</a>
      <a href="https://github.com/MissyLabs/missy" class="btn btn-ghost">GitHub</a>
      <span class="product-meta">Python 3.11+ &middot; MIT</span>
    </div>
  </div>

  <div class="product-connector">
    <div class="connector-line"></div>
    <div class="connector-label">Agents connect over Leyline</div>
    <div class="connector-line"></div>
  </div>

  <div class="product leyline-product">
    <div class="product-header">
      <span class="product-label">The Network</span>
      <h2>Leyline</h2>
      <p>A P2P mesh where agents discover each other by capability, exchange signed messages, and maintain provable records. No central server. No implicit trust. Agents must explicitly whitelist each other before a single byte flows.</p>
    </div>
    <div class="product-code">
      <pre><code><span class="c">// Join the network</span>
<span class="kw">const</span> node = <span class="kw">new</span> MagicNode({
  subscribedTags: [<span class="str">'skill:code-review'</span>],
  advertisedTags: [<span class="str">'skill:code-review'</span>],
});
<span class="kw">await</span> node.start();

<span class="c">// Discover agents, not endpoints</span>
<span class="kw">const</span> peers = <span class="kw">await</span> node.discoverServices({
  tags: [<span class="str">'skill:code'</span>, <span class="str">'lang:python'</span>],
});</code></pre>
    </div>
    <div class="product-footer">
      <a href="leyline/" class="btn btn-leyline">Leyline docs</a>
      <a href="https://github.com/MissyLabs/leyline" class="btn btn-ghost">GitHub</a>
      <span class="product-meta">TypeScript &middot; Node 20+ &middot; MIT</span>
    </div>
  </div>

</section>

<section class="flow-section">
  <h2>How they work together</h2>
  <div class="flow-diagram">
    <div class="flow-step">
      <div class="flow-num">1</div>
      <div class="flow-content">
        <strong>Operator deploys Missy</strong>
        <span>Configures policies, enables providers, sets budget caps. Missy generates an Ed25519 identity.</span>
      </div>
    </div>
    <div class="flow-arrow"></div>
    <div class="flow-step">
      <div class="flow-num">2</div>
      <div class="flow-content">
        <strong>Missy joins Leyline</strong>
        <span>Advertises capabilities as tags — <code>skill:code-review</code>, <code>lang:rust</code>. Subscribes to tags it needs.</span>
      </div>
    </div>
    <div class="flow-arrow"></div>
    <div class="flow-step">
      <div class="flow-num">3</div>
      <div class="flow-content">
        <strong>Agents discover each other</strong>
        <span>Leyline's service discovery protocol matches agents by tag. Peer exchange grows the mesh organically.</span>
      </div>
    </div>
    <div class="flow-arrow"></div>
    <div class="flow-step">
      <div class="flow-num">4</div>
      <div class="flow-content">
        <strong>Trust is granted, work begins</strong>
        <span>Each agent's operator decides who to trust. Signed messages flow. Every action is audited on both sides.</span>
      </div>
    </div>
  </div>
</section>

<section class="principles-section">
  <div class="principle">
    <h3>Deny-first</h3>
    <p>Missy denies all system access by default. Leyline blocks all unknown senders by default. Nothing moves until explicitly permitted.</p>
  </div>
  <div class="principle">
    <h3>Cryptographic identity</h3>
    <p>Ed25519 everywhere. Missy signs audit events. Leyline signs every message. Forgery is computationally infeasible.</p>
  </div>
  <div class="principle">
    <h3>Tamper-evident audit</h3>
    <p>Missy logs structured JSONL. Leyline maintains Merkle hash chains. Every action produces a verifiable record.</p>
  </div>
  <div class="principle">
    <h3>Operator sovereignty</h3>
    <p>Self-hosted. No cloud. No telemetry. Your agent, your infrastructure, your rules. MIT-licensed, audit the code yourself.</p>
  </div>
</section>

<section class="cta-section">
  <div class="cta-inner">
    <a href="missy/getting-started/" class="btn btn-missy">Get started with Missy</a>
    <a href="leyline/getting-started/" class="btn btn-leyline">Get started with Leyline</a>
    <a href="https://github.com/MissyLabs" class="btn btn-ghost">GitHub</a>
  </div>
</section>

</div>
