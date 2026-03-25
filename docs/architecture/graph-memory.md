---
tags:
  - architecture
---

# Graph Memory

The `GraphMemoryStore` provides structured knowledge storage as an entity-relationship graph, complementing the text-based `SQLiteMemoryStore` and vector-based `VectorMemoryStore`.

## Storage

Graph memory is stored in SQLite at `~/.missy/graph_memory.db`. Entities and relationships are persisted as rows with metadata.

## Concepts

| Concept | Description |
|---------|-------------|
| **Entity** | A named node (person, project, concept, file) |
| **Relationship** | A typed edge between two entities (e.g., "authored", "depends-on") |
| **Pattern** | A rule-based query that matches subgraphs |

## Use Cases

- **Knowledge tracking** — "User works on project X", "File A depends on File B"
- **Context enrichment** — Relevant entities and relationships are injected into the agent's context
- **Pattern matching** — Rules like "find all files related to the current task" traverse the graph

## Relationship to Other Memory Systems

| Store | Best For |
|-------|----------|
| `SQLiteMemoryStore` | Conversation history, full-text search |
| `VectorMemoryStore` | Semantic similarity search |
| `GraphMemoryStore` | Structured relationships, entity tracking |
| `MemorySynthesizer` | Merges all stores into a single context block |
