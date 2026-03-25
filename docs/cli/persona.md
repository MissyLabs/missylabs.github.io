---
tags:
  - cli
---

# Persona Commands

Manage Missy's agent identity, tone, and style. Persona configuration is stored at `~/.missy/persona.yaml` with automatic backups and audit logging.

## missy persona show

Display the current persona configuration.

```bash
missy persona show
```

## missy persona edit

Edit persona fields.

```bash
missy persona edit --name "Missy" --tone "friendly and concise"
missy persona edit --identity "security-focused assistant"
```

| Option | Description |
|--------|-------------|
| `--name` | Agent display name |
| `--tone` | Communication style descriptor |
| `--identity` | Core identity/role description |

## missy persona reset

Reset persona to factory defaults.

```bash
missy persona reset
```

## missy persona backups

List available persona backups.

```bash
missy persona backups
```

## missy persona diff

Show differences between current persona and latest backup.

```bash
missy persona diff
```

## missy persona rollback

Restore persona from the latest backup.

```bash
missy persona rollback
```

## missy persona log

Show persona change audit log.

```bash
missy persona log
missy persona log --limit 20
```
