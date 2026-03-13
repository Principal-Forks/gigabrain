# Gigabrain Architecture

Gigabrain is a local-first hybrid memory stack for intelligent agents. It enables AI agents to maintain persistent, queryable memory across sessions without requiring cloud vector databases.

## Problem Statement

AI agents lose context between conversations. Each session starts fresh, forcing users to re-explain preferences, decisions, and context. Gigabrain solves this by:

- **Capturing** explicit memory intents from agent outputs
- **Storing** memories in a durable, auditable SQLite database
- **Recalling** relevant context before each prompt
- **Maintaining** memory quality through automated nightly pipelines

## Core Operations

### Capture Pipeline
When an agent outputs a `<memory_note>` XML tag, the capture pipeline:
1. Parses the memory content and metadata
2. Validates against policy (junk filter, plausibility checks)
3. Performs deduplication (exact and semantic)
4. Appends to the event store (immutable log)
5. Updates the projection store (queryable state)
6. Optionally writes to native MEMORY.md files

### Recall Pipeline
Before each prompt, the recall pipeline:
1. Extracts queries from the user's input
2. Resolves entity coreferences (person names, aliases)
3. Classifies the query type (entity brief, timeline, verification)
4. Searches FTS5 index and native chunks
5. Ranks and budgets results by relevance
6. Injects as `<gigabrain-context>` into the prompt

### Nightly Maintenance
An automated pipeline runs quality maintenance:
1. Syncs native MEMORY.md content
2. Promotes native notes to the registry
3. Runs quality sweeps and deduplication
4. Refreshes entity graph and world model
5. Detects contradictions
6. Builds synthesis briefs and vault surface

## Design Decisions

### Event Sourcing + CQRS
All memory changes are logged to an append-only event store, enabling:
- Full audit trails
- Rollback and restore capabilities
- Temporal queries
- Quality scoring over time

The read path uses materialized projections for fast FTS5 search.

### No Vector Database Required
Core recall uses SQLite FTS5 full-text search. This keeps the system:
- Fully local (no API calls for recall)
- Deterministic and auditable
- Fast for common queries
- Simple to deploy

### Dual Integration Paths
- **OpenClaw Plugin**: Direct gateway integration with memory slot provider
- **Codex MCP Server**: Standalone server for Codex App/CLI

Both paths share the same core engine.

## Common Workflows

### User Fact Capture
```
User: "My favorite programming language is Rust"
Agent: <memory_note type="PREFERENCE">User's favorite programming language is Rust</memory_note>
```

### Entity Brief Recall
```
User: "What do you know about Alex?"
→ Orchestrator classifies as entity_brief
→ Recall searches entity graph + memories mentioning Alex
→ Returns synthesized brief about Alex
```

### Nightly Maintenance
```bash
npx gigabrainctl nightly
```
Runs the full maintenance pipeline: sync, dedupe, audit, vault build.

## Error Handling

- **Junk Filter Rejection**: Memories matching system prompt patterns are rejected
- **Plausibility Failure**: Malformed or nonsensical memories are flagged for review
- **Dedup Conflict**: Duplicate memories are merged, keeping the higher-quality version
- **Contradiction Detection**: Conflicting facts surface in the review queue
