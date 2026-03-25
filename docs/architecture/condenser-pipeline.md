---
tags:
  - architecture
---

# Condenser Pipeline

The `CondenserPipeline` is a 4-stage memory compression system that reduces conversation history size while preserving essential information. It works alongside [Sleep Mode](sleep-mode.md) and the [Memory Synthesizer](memory-synthesizer.md).

## Stages

```mermaid
graph LR
    A[Full History] --> B[Observation Masking]
    B --> C[Amortized Forgetting]
    C --> D[Summarizing]
    D --> E[Windowing]
    E --> F[Compressed History]
```

### 1. Observation Masking

Removes redundant tool outputs that have already been processed. For example, if a file was read and its contents incorporated into the conversation, the raw file content can be masked.

### 2. Amortized Forgetting

Progressively reduces detail in older messages. Recent messages retain full detail; older messages are condensed proportionally to their age.

### 3. Summarizing

Groups related messages and replaces them with summaries that capture key facts, decisions, and outcomes.

### 4. Windowing

Applies a sliding window that keeps the most recent N messages at full fidelity, ensuring the model always has immediate context.

## When It Runs

The condenser pipeline is triggered by the `CompactionManager` when:

- Context usage exceeds the configured threshold (default 80%)
- Sleep mode activates via `MemoryConsolidator`
- Manually via compaction scheduling

## Relationship to Other Systems

| System | Role |
|--------|------|
| **CondenserPipeline** | Compresses history (this page) |
| **MemoryConsolidator** | Triggers compression at 80% context |
| **CompactionManager** | Orchestrates leaf + condensation passes |
| **MemorySynthesizer** | Injects compressed results into context |
| **SleeptimeWorker** | Runs compression during idle periods |
