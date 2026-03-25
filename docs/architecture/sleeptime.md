---
tags:
  - architecture
---

# Sleeptime Computing

The `SleeptimeWorker` performs background memory processing during idle periods — when no active conversation is happening. Inspired by [Letta's sleeptime computing](https://www.letta.com/blog/sleeptime-agents) concept.

## What Happens During Sleep

1. **Memory consolidation** — Old conversation turns are summarized and compressed via the [Condenser Pipeline](condenser-pipeline.md)
2. **Index maintenance** — Vector memory (FAISS) indices are rebuilt and optimized
3. **Pruning** — Stale entries are removed from the memory store based on age and relevance
4. **Learnings extraction** — Patterns from recent sessions are analyzed and stored

## Activation

The sleeptime worker activates when:

- No user input has been received for a configurable idle period
- The system detects it is in a low-activity state
- Scheduled via the heartbeat system

## Design Goals

- **Non-disruptive** — All processing happens in the background without affecting response latency
- **Idempotent** — Can be interrupted and resumed safely
- **Progressive** — Processes oldest/least-relevant memories first
