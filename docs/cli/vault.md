---
tags:
  - cli
  - security
---

# Vault Commands

Manage the ChaCha20-Poly1305 encrypted secrets vault.

!!! warning
    The vault must be enabled in config before use: `vault: { enabled: true }`.

## missy vault set

```bash
missy vault set ANTHROPIC_API_KEY sk-ant-api03-...
```

Stores a key-value pair encrypted at `~/.missy/secrets/vault.enc`.

## missy vault get

```bash
missy vault get ANTHROPIC_API_KEY
```

Retrieves and prints the decrypted value.

## missy vault list

```bash
missy vault list
```

Lists all stored key names (values are not shown).

## missy vault delete

```bash
missy vault delete ANTHROPIC_API_KEY
```

## Using Vault References

Reference vault secrets in `config.yaml` with the `vault://` prefix:

```yaml
providers:
  anthropic:
    name: anthropic
    model: "claude-sonnet-4-6"
    api_key: "vault://ANTHROPIC_API_KEY"
```

Vault references are resolved transparently at config load time.
