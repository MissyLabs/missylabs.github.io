---
tags:
  - cli
---

# Evolve Commands

Manage code evolution proposals — self-modifying code changes with approval workflow and git-backed rollback.

## missy evolve list

List all code evolution proposals.

```bash
missy evolve list
```

## missy evolve show

Show details of a specific evolution proposal.

```bash
missy evolve show EVOLUTION_ID
```

## missy evolve approve

Approve a proposed code evolution.

```bash
missy evolve approve EVOLUTION_ID
```

## missy evolve reject

Reject a proposed code evolution.

```bash
missy evolve reject EVOLUTION_ID
```

## missy evolve apply

Apply an approved code evolution.

```bash
missy evolve apply EVOLUTION_ID
```

!!! note
    Applied evolutions create a git commit that can be rolled back.

## missy evolve rollback

Roll back a previously applied code evolution.

```bash
missy evolve rollback EVOLUTION_ID
```
