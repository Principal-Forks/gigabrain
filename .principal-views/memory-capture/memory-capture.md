# Memory Capture Pipeline

The Memory Capture Pipeline extracts and stores memory notes from agent conversation outputs into a durable, queryable registry.

## Problem Statement

AI agents generate valuable contextual information during conversations that should be preserved across sessions. The capture pipeline solves this by:

- **Extracting** `<memory_note>` XML tags from agent outputs
- **Validating** content against policy rules (junk detection, plausibility)
- **Deduplicating** against existing memories (exact hash + semantic similarity)
- **Storing** to SQLite registry and/or native markdown files
- **Indexing** for world model entity mentions

## Core Operations

### Parse Phase
When agent output arrives, the pipeline:
1. Extracts all `<memory_note>` tags with their attributes (type, confidence, scope, durability)
2. Normalizes memory types (USER_FACT, PREFERENCE, DECISION, ENTITY, EPISODE, AGENT_IDENTITY, CONTEXT)
3. Detects nested tags and oversized content (>1200 chars)
4. Processes memory actions (remember, forget, update)

### Policy Phase
Each memory note is validated:
1. **Junk Detection** - Matches against system prompt patterns, API keys, benchmark artifacts
2. **Plausibility Check** - Detects token anomalies, broken phrases, entityless numeric facts
3. **Confidence Threshold** - Low confidence USER_FACTs with plausibility flags go to review

### Deduplication Phase
Prevents redundant storage:
1. **Exact Matching** - Normalized content + scope hash comparison
2. **Semantic Matching** - Jaccard similarity scoring against existing memories
3. **Threshold Routing** - Above auto-threshold: drop, between thresholds: queue for review

### Write Phase
Successful memories are persisted:
1. **Registry Insert** - SQLite `memory_current` table with full metadata
2. **Native Write** - Optional append to MEMORY.md or daily note
3. **Event Logging** - Append to `memory_events` for audit trail

### Post-Processing
After capture completes:
1. **Entity Rebuild** - Updates person service mention graph
2. **World Model Refresh** - Rebuilds beliefs, episodes, syntheses

## Design Decisions

### Policy-First Filtering
Junk and plausibility checks run before deduplication to avoid comparing garbage content against the registry.

### Dual Write Targets
Registry provides structured querying while native markdown enables human editing and version control.

### Semantic Dedup with Review Band
Rather than hard threshold, borderline duplicates (similarity 0.7-0.85) are queued for human/LLM review.

### Event Sourcing
All capture decisions are logged to `memory_events` with reason codes, enabling audit and replay.

## Common Workflows

### Standard Capture
```
Agent outputs: <memory_note type="PREFERENCE">User prefers dark mode</memory_note>
-> Parse: type=PREFERENCE, confidence=0.65
-> Policy: Not junk, plausible
-> Dedupe: No match
-> Write: Inserted to registry + native
-> Result: Memory stored successfully
```

### Duplicate Rejection
```
Agent outputs: <memory_note>User prefers dark mode</memory_note>
-> Parse: type=USER_FACT (inferred)
-> Policy: Valid
-> Dedupe: Exact match found (same normalized content)
-> Result: Dropped, logged as duplicate
```

### Review Queue
```
Agent outputs: <memory_note confidence="0.5">User born 1985</memory_note>
-> Parse: type=USER_FACT, confidence=0.5
-> Policy: Plausibility flag (entityless_numeric_fact)
-> Result: Queued for review
```

## Error Handling

- **Junk Rejection**: Logged with matched pattern, no storage
- **Plausibility Failure**: Queued for review with flags
- **Native Write Failure**: Logged as warning, registry write continues
- **World Model Failure**: Logged as warning, capture still succeeds
