# Enhancement Ideas

These are opportunities for improving DeerFlow, organized by complexity and impact.

---

## High Impact, Medium Complexity

### 1. Authentication & Authorization System

**Current state**: No authentication. The `better-auth` library is scaffolded in the frontend (`frontend/src/server/`) but not yet active. Anyone with network access can use the system.

**Proposal**:
- Activate better-auth with GitHub OAuth (already partially implemented)
- Add JWT-based API authentication for Gateway endpoints
- Per-user thread isolation (users can only access their own threads)
- Role-based access control (admin, user, viewer)
- API key management for programmatic access

**Impact**: Required for any multi-user or production deployment.

---

### 2. Database-Backed Storage

**Current state**: Memory stored in `memory.json`, thread data in filesystem directories. No query capability, no concurrent write safety beyond atomic rename.

**Proposal**:
- Replace `memory.json` with SQLite/PostgreSQL storage
- Store thread metadata and artifacts in database
- Enable full-text search across memory facts
- Support concurrent users without file locking issues
- Migration path: JSON import/export for backward compatibility

**Impact**: Prerequisite for multi-user support and production scalability.

---

### 3. Observability Stack

**Current state**: Python `logging` module with configurable log level. Token usage tracking via middleware. No metrics, no tracing, no dashboards.

**Proposal**:
- **OpenTelemetry integration**: Traces for agent execution, tool calls, LLM invocations
- **Structured logging**: JSON log format for log aggregation (ELK, Loki)
- **Metrics**: Prometheus metrics for latency, token usage, error rates, active threads
- **Dashboard**: Grafana dashboard template for monitoring agent health
- **LangSmith integration**: Already partially supported via metadata injection — expand with proper trace context

**Impact**: Essential for understanding agent behavior in production.

---

### 4. RAG Integration (Vector Store for Long-Term Knowledge)

**Current state**: Memory system stores discrete facts (max 100) with text-based deduplication. No semantic search or retrieval-augmented generation.

**Proposal**:
- Add vector store backend (Chroma, Qdrant, Pinecone) alongside fact storage
- Embed conversation summaries and uploaded documents
- Semantic retrieval during system prompt generation
- Hybrid approach: structured facts + vector similarity search
- Configuration in `config.yaml → memory.vector_store`

**Impact**: Dramatically improves agent's long-term knowledge and context recall.

---

### 5. Agent Evaluation Framework

**Current state**: No systematic way to evaluate agent quality, benchmark tool selection accuracy, or regression-test prompt changes.

**Proposal**:
- Define evaluation datasets: (input, expected_behavior) pairs
- Metrics: tool selection accuracy, response quality (LLM-as-judge), latency, token efficiency
- CI integration: Run evaluations on prompt/middleware changes
- A/B testing support: Compare system prompt variations
- Benchmark suite for different model providers

**Impact**: Enables data-driven agent improvement rather than trial-and-error.

---

## Medium Impact, Low-Medium Complexity

### 6. Rate Limiting and Quota Management

**Current state**: No rate limiting at any layer. A single user could exhaust LLM API quotas.

**Proposal**:
- Token-based rate limiting per user/API key
- Configurable limits in `config.yaml`
- Queue-based overflow handling (return 429 with retry-after)
- Admin dashboard for quota visibility

---

### 7. Conversation Branching / Forking

**Current state**: Linear conversation threads. No way to explore alternative approaches from a midpoint.

**Proposal**:
- Fork a thread at any message to create a branch
- Compare outcomes across branches
- Merge useful results back to main thread
- UI: Tree visualization of conversation branches

---

### 8. Workflow Templates

**Current state**: Skills provide workflow instructions but no structured execution pipeline.

**Proposal**:
- Define multi-step workflow templates (YAML/JSON)
- Sequential and parallel step execution
- Conditional branching based on intermediate results
- Template marketplace (community-contributed)
- Example: "Code Review Pipeline" → lint → test → security scan → summary

---

### 9. Plugin Architecture for Custom Middleware

**Current state**: Adding middleware requires modifying `_build_middlewares()` in the source code.

**Proposal**:
- Config-driven middleware registration (like tools):
  ```yaml
  middlewares:
    - use: my_package:CustomMiddleware
      config:
        key: value
      position: before:ClarificationMiddleware
  ```
- Allow third-party middleware packages
- Middleware ordering DSL for dependency declaration

---

### 10. Multi-User / Multi-Tenant Support

**Current state**: Single-user system. All threads share the same memory, config, and API keys.

**Proposal**:
- Per-user memory isolation
- Per-user API key management
- Tenant-level configuration overrides
- Usage tracking per user/tenant
- Admin panel for user management

---

## Lower Complexity Improvements

### 11. WebSocket Support

**Current state**: SSE (Server-Sent Events) for streaming. Unidirectional.

**Proposal**:
- Add WebSocket transport as alternative to SSE
- Enable bidirectional communication (e.g., cancel mid-stream)
- Better mobile browser compatibility
- Multiplexed connections for concurrent streams

---

### 12. Prompt Versioning

**Current state**: System prompts generated dynamically with no version tracking.

**Proposal**:
- Version-control system prompt templates
- Track which prompt version generated each response
- A/B testing between prompt versions
- Rollback capability for prompt regressions

---

### 13. Skill Store / Marketplace

**Current state**: Skills installed manually via ZIP archive or placed in `skills/custom/`.

**Proposal**:
- Centralized skill registry (GitHub-based or self-hosted)
- One-click install from the web UI
- Skill ratings, reviews, and download counts
- Dependency management between skills
- Auto-update mechanism

---

### 14. Enhanced File Handling

**Current state**: File upload with PDF/PPT/Excel/Word conversion to Markdown. Basic read/write in sandbox.

**Proposal**:
- Image OCR and analysis for uploaded images
- Video/audio transcription
- Spreadsheet formula preservation
- Version history for sandbox files
- Large file chunking for context-limited models

---

### 15. Agent Collaboration

**Current state**: Subagents work independently and return results to the lead agent.

**Proposal**:
- Inter-agent communication (subagent A shares findings with subagent B)
- Shared scratchpad for collaborative problem-solving
- Debate/critique pattern: Agent A proposes, Agent B critiques
- Consensus mechanism for conflicting results

---

## Infrastructure Improvements

### 16. Horizontal Scaling

**Current state**: Single-process architecture. SQLite checkpointer supports one process.

**Proposal**:
- PostgreSQL checkpointer for multi-process deployment
- Redis-based memory cache for shared state
- Load balancer support with sticky sessions
- Kubernetes Helm chart

---

### 17. Offline / Local-Only Mode

**Current state**: Requires external LLM API access.

**Proposal**:
- Support for local models via Ollama/vLLM
- Pre-configured config templates for popular local models
- Reduced-feature mode when internet is unavailable
- Local embedding models for RAG

---

### 18. Audit Logging

**Current state**: Standard application logging with no audit trail.

**Proposal**:
- Immutable audit log of all agent actions (tool calls, file modifications, API calls)
- Who did what, when, via which agent
- Compliance-ready format (SOC 2, GDPR)
- Searchable audit trail via Gateway API
