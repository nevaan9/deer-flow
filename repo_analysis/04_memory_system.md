# Memory System Deep Dive

## Overview

DeerFlow's memory system provides LLM-powered persistent context extraction, enabling the agent to learn from conversations and personalize future interactions. It operates entirely in the background with debounced processing.

---

## Architecture

```
User Conversation
        │
        ▼
MemoryMiddleware (in middleware chain)
  - Filters: user messages + final AI responses
  - Queues for background processing
        │
        ▼
MemoryUpdateQueue
  - Per-thread deduplication (newer replaces older)
  - Debounce timer: 30s default (configurable)
  - Batches multiple conversations
        │
        ▼ (after debounce expires)
MemoryUpdater (LLM-based)
  - Specialized prompt for fact extraction
  - Extracts context summaries + discrete facts
  - Whitespace-normalized deduplication
        │
        ▼
memory.json (atomic write: temp file → rename)
  - Cache invalidated on write
        │
        ▼
Next Request → Inject top 15 facts + context into system prompt
```

---

## Components

### MemoryMiddleware

**Location**: Part of the middleware chain in `lead_agent/agent.py`

**Behavior**:
- Runs in `after_model` phase
- Filters conversation messages to only user inputs and final AI responses
- Calls `MemoryUpdateQueue.add(thread_id, messages, agent_name)`
- Non-blocking — queues for background processing

### MemoryUpdateQueue

**Location**: `backend/packages/harness/deerflow/agents/memory/queue.py`

**Key behaviors**:
- **Per-thread deduplication**: If a thread sends multiple updates before the debounce fires, only the latest is kept
- **Debounce**: Configurable wait time (default 30s) prevents excessive LLM calls
- **Background thread**: Processing happens off the main request path
- **Batch processing**: Multiple threads' updates processed together after debounce

### MemoryUpdater

**Location**: `backend/packages/harness/deerflow/agents/memory/updater.py`

**What it does**:
1. Takes filtered conversation messages
2. Calls LLM with a specialized memory extraction prompt
3. Extracts structured output:
   - Context summaries (work, personal, top-of-mind)
   - History summaries (recent, earlier, long-term)
   - Discrete facts with confidence scores
4. Deduplicates facts using whitespace-normalized comparison
5. Writes atomically (temp file → rename)
6. Invalidates the in-process cache

---

## Data Structure

The memory is stored in `backend/.deer-flow/memory.json`:

```json
{
  "version": "1.0",
  "user": {
    "workContext": {
      "summary": "Senior engineer working on distributed systems at Acme Corp",
      "updatedAt": "2026-03-20T10:30:00Z"
    },
    "personalContext": {
      "summary": "Prefers concise explanations, experienced with Python and Go",
      "updatedAt": "2026-03-20T10:30:00Z"
    },
    "topOfMind": {
      "summary": "Currently investigating performance bottleneck in API gateway",
      "updatedAt": "2026-03-24T14:00:00Z"
    }
  },
  "history": {
    "recentMonths": {
      "summary": "Refactored authentication module, set up monitoring dashboards",
      "updatedAt": "2026-03-24T14:00:00Z"
    },
    "earlierContext": {
      "summary": "Migrated from monolith to microservices architecture",
      "updatedAt": "2026-02-15T09:00:00Z"
    },
    "longTermBackground": {
      "summary": "Has been at company for 3 years, led the platform team",
      "updatedAt": "2026-01-01T00:00:00Z"
    }
  },
  "facts": [
    {
      "id": "fact-001",
      "content": "Uses VS Code with Vim keybindings",
      "category": "preference",
      "confidence": 0.95,
      "createdAt": "2026-03-10T08:00:00Z",
      "source": "conversation"
    },
    {
      "id": "fact-002",
      "content": "Team uses PostgreSQL 16 for all production databases",
      "category": "knowledge",
      "confidence": 0.9,
      "createdAt": "2026-03-15T11:00:00Z",
      "source": "conversation"
    }
  ]
}
```

### Fact Categories

| Category | Description |
|----------|-------------|
| `preference` | User preferences and working styles |
| `knowledge` | Domain knowledge and technical facts |
| `context` | Current situation and project context |
| `behavior` | User behavioral patterns |
| `goal` | Objectives and targets |

---

## Per-Agent Memory

DeerFlow supports isolated memory per custom agent:

- Default agent: `backend/.deer-flow/memory.json`
- Custom agent: `backend/.deer-flow/agents/{agent_name}/memory.json`

This enables different agents to maintain separate knowledge bases — a research agent accumulates research-specific facts while a coding agent tracks development preferences.

---

## Memory Injection

On each request, the system prompt includes memory context:

```xml
<memory>
  <user_context>
    <work>Senior engineer working on distributed systems...</work>
    <personal>Prefers concise explanations...</personal>
    <top_of_mind>Investigating performance bottleneck...</top_of_mind>
  </user_context>
  <facts>
    1. Uses VS Code with Vim keybindings
    2. Team uses PostgreSQL 16 for all production databases
    ... (up to 15 facts)
  </facts>
</memory>
```

**Injection limits**:
- Maximum 15 facts injected per request
- Maximum 2000 tokens for the entire memory section
- Only injected when `memory.injection_enabled: true`

---

## Deduplication Strategy

Before appending new facts, the updater:
1. Strips leading/trailing whitespace from both existing and new fact content
2. Compares normalized strings for exact matches
3. Skips duplicates silently
4. New unique facts are appended with fresh metadata

This prevents memory bloat from repeated conversations about the same topics.

---

## Atomic I/O

All memory writes follow this pattern:
1. Write to a temporary file in the same directory
2. `os.replace()` (atomic rename) the temp file to the target path
3. If anything fails, the temp file is cleaned up

This ensures the memory file is never in a corrupted state, even if the process crashes mid-write.

---

## Configuration

```yaml
# config.yaml
memory:
  enabled: true                      # Master switch
  injection_enabled: true            # Whether to inject into system prompt
  storage_path: memory.json          # Path relative to .deer-flow/
  debounce_seconds: 30               # Wait before processing queued updates
  model_name: null                   # LLM for extraction (null = default)
  max_facts: 100                     # Maximum stored facts
  fact_confidence_threshold: 0.7     # Minimum confidence to store
  max_injection_tokens: 2000         # Token limit for prompt injection
```

### Tuning Recommendations

| Scenario | Adjustment |
|----------|-----------|
| High-frequency conversations | Increase `debounce_seconds` to 60-120 |
| Cost-sensitive | Use lightweight model for `model_name` |
| Privacy-focused | Set `injection_enabled: false` |
| Short-term focus | Reduce `max_facts` to 20-30 |
| High-quality memory | Increase `fact_confidence_threshold` to 0.85 |

---

## API Access

### Gateway API

```
GET  /api/memory          → Memory data
POST /api/memory/reload   → Force reload from file
GET  /api/memory/config   → Memory configuration
GET  /api/memory/status   → Config + current data
```

### Embedded Client

```python
client = DeerFlowClient()
client.get_memory()         # Current memory data
client.reload_memory()      # Force reload
client.get_memory_config()  # Configuration
client.get_memory_status()  # Config + data
```

---

## Test Coverage

`backend/tests/test_memory_updater.py` covers:
- Fact extraction from conversations
- Whitespace-normalized deduplication
- Atomic file I/O behavior
- Cache invalidation

`backend/tests/test_memory_prompt_injection.py` covers:
- Memory injection into system prompts
- Token limit enforcement
- Edge cases with empty/malformed memory data
