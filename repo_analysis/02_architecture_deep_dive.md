# Architecture Deep Dive

## Three-Service Architecture

DeerFlow runs as three services behind an Nginx reverse proxy:

```
                        Browser / API Client
                              │
                    ┌─────────▼──────────┐
                    │  Nginx (port 2026)  │
                    │  Unified Entry Point│
                    └─────┬───┬───┬──────┘
                          │   │   │
          ┌───────────────┘   │   └───────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ LangGraph Server│ │ Gateway API     │ │ Frontend         │
│ Port 2024       │ │ Port 8001       │ │ Port 3000        │
│                 │ │                 │ │                  │
│ Agent runtime   │ │ FastAPI REST    │ │ Next.js 16       │
│ State mgmt      │ │ Models, MCP,   │ │ React 19         │
│ Thread handling │ │ Skills, Memory,│ │ Chat workspace   │
│ Tool execution  │ │ Uploads, etc.  │ │ Landing page     │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

### Nginx Routing Rules

| Path Pattern | Destination | Purpose |
|---|---|---|
| `/api/langgraph/*` | LangGraph Server (2024) | Agent conversations, streaming |
| `/api/*` (other) | Gateway API (8001) | Models, MCP, skills, memory, uploads, artifacts |
| `/` (non-API) | Frontend (3000) | Web interface |

### Key Nginx Configuration Details
- **SSE/Streaming**: `proxy_buffering off` for real-time agent responses
- **Timeouts**: 600s for long-running agent operations
- **CORS**: Centralized handling (allows all origins by default)
- **Chunked Transfer**: Enabled for streaming support

---

## Lead Agent Creation Flow

The core of DeerFlow is the Lead Agent, created via `make_lead_agent(config: RunnableConfig)` in `backend/packages/harness/deerflow/agents/lead_agent/agent.py`.

```
make_lead_agent(config: RunnableConfig)
    │
    ├── 1. Extract runtime config from config.configurable
    │       - thinking_enabled, model_name, is_plan_mode
    │       - subagent_enabled, max_concurrent_subagents
    │       - reasoning_effort, agent_name, is_bootstrap
    │
    ├── 2. Model Resolution
    │       - Load agent-specific config if agent_name provided
    │       - Check agent's custom model override
    │       - Fall back to global default model
    │       - Validate thinking support
    │       → create_chat_model(name, thinking_enabled, reasoning_effort)
    │
    ├── 3. Tools Assembly via get_available_tools()
    │       - Config-defined tools (from config.yaml via reflection)
    │       - MCP tools (from enabled servers, cached with mtime)
    │       - Built-ins: present_file, ask_clarification
    │       - Conditional: view_image (if vision), task (if subagent)
    │       - Conditional: tool_search (if deferred loading)
    │
    ├── 4. System Prompt via apply_prompt_template()
    │       - Role definition (agent name + soul)
    │       - Memory injection (top 15 facts + context)
    │       - Skills list with container paths
    │       - Subagent orchestration instructions
    │       - Working directory and file handling guidance
    │
    ├── 5. Middleware Chain via _build_middlewares()
    │       - 15 middlewares in strict order (see below)
    │
    └── 6. create_agent(model, tools, middleware, system_prompt, state_schema=ThreadState)
```

---

## ThreadState Schema

The agent's state is managed through `ThreadState`, defined in `backend/packages/harness/deerflow/agents/thread_state.py`:

```python
class ThreadState(AgentState):
    # Inherited from AgentState:
    messages: list[BaseMessage]           # Conversation history

    # DeerFlow extensions:
    sandbox: SandboxState | None          # {sandbox_id: str | None}
    thread_data: ThreadDataState | None   # {workspace_path, uploads_path, outputs_path}
    title: str | None                     # Auto-generated thread title
    artifacts: Annotated[list[str], merge_artifacts]  # Deduplicating list
    todos: list | None                    # Plan mode task list
    uploaded_files: list[dict] | None     # File metadata from uploads
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]  # Image cache
```

### Custom Reducers

- **`merge_artifacts`**: Deduplicates artifact paths while preserving insertion order (uses `dict.fromkeys`)
- **`merge_viewed_images`**: Merges image dictionaries; sending an empty dict `{}` clears all images (used by ViewImageMiddleware after processing)

---

## Middleware Chain

Middlewares are the backbone of DeerFlow's cross-cutting concerns. They follow the **Aspect-Oriented Programming (AOP)** pattern, with strict execution order.

Each middleware can intercept at three points:
- **`before_model`**: Before the LLM is called
- **`wrap_tool_call`**: Around each tool invocation
- **`after_model`**: After the LLM responds

### Execution Order (Critical)

```
Request
  │
  ├── 1.  ThreadDataMiddleware       Creates per-thread directories
  ├── 2.  UploadsMiddleware          Injects uploaded file metadata
  ├── 3.  SandboxMiddleware          Acquires sandbox, stores sandbox_id
  ├── 4.  DanglingToolCallMiddleware Patches missing ToolMessage responses
  ├── 5.  GuardrailMiddleware        Pre-tool-call authorization (optional)
  ├── 6.  ToolErrorHandlingMiddleware Converts tool exceptions to ToolMessages
  ├── 7.  SummarizationMiddleware    Context reduction at token limits (optional)
  ├── 8.  TodoListMiddleware         Task tracking in plan mode (optional)
  ├── 9.  TokenUsageMiddleware       Token usage accumulation (optional)
  ├── 10. TitleMiddleware            Auto-generates thread title
  ├── 11. MemoryMiddleware           Queues conversations for memory update
  ├── 12. ViewImageMiddleware        Injects base64 images for vision models (optional)
  ├── 13. DeferredToolFilterMiddleware Hides deferred MCP tools (optional)
  ├── 14. SubagentLimitMiddleware    Truncates excess task calls (optional)
  ├── 15. LoopDetectionMiddleware    Breaks repetitive tool call loops
  ├── 16. ClarificationMiddleware    Intercepts ask_clarification (MUST BE LAST)
  │
  └── Agent Execution Loop
```

### Why Order Matters

- **ThreadData before Sandbox**: Thread directories must exist before sandbox acquires them
- **Dangling before Guardrail**: Incomplete tool calls must be patched before authorization checks
- **Summarization before Title/Memory**: Context reduction happens before downstream processing
- **Memory after Title**: Title is generated first, then conversation is queued for memory
- **Clarification last**: Must intercept after all other processing to properly interrupt the flow via `Command(goto=END)`

---

## Configuration System

DeerFlow uses a layered configuration approach:

### config.yaml (Main Configuration)

**Resolution priority:**
1. Explicit `config_path` argument
2. `DEER_FLOW_CONFIG_PATH` environment variable
3. `config.yaml` in current directory
4. `config.yaml` in parent directory (project root — recommended)

**Key sections:**
- `models[]` — LLM configurations (name, use class path, API key, thinking/vision flags)
- `tools[]` — Tool definitions with group assignments
- `tool_groups[]` — Logical groupings for access control
- `sandbox` — Provider selection (local or Docker)
- `skills` — Path configuration
- `title` — Auto-title generation
- `summarization` — Context compression settings
- `memory` — Persistent memory configuration
- `checkpointer` — State persistence (memory, sqlite, postgres)
- `guardrails` — Pre-tool-call authorization
- `channels` — IM platform integrations

**Environment variable resolution**: Any value starting with `$` is resolved from environment (e.g., `api_key: $OPENAI_API_KEY`).

**Config versioning**: `config_version` field enables schema migration. Run `make config-upgrade` to merge new fields.

**Auto-reload**: `get_app_config()` caches parsed config but detects mtime changes for automatic reload without process restart.

### extensions_config.json (MCP + Skills)

**Resolution priority:** Same as config.yaml but with `DEER_FLOW_EXTENSIONS_CONFIG_PATH`.

```json
{
  "mcpServers": {
    "github": {
      "enabled": true,
      "type": "stdio",
      "command": "uvx",
      "args": ["mcp-server-github"],
      "env": {"GITHUB_TOKEN": "$GITHUB_TOKEN"}
    }
  },
  "skills": {
    "deep-research": {"enabled": true},
    "report-generation": {"enabled": true}
  }
}
```

**Runtime modifiable**: Gateway API endpoints allow updating MCP servers and skills without restart. Changes are detected by the LangGraph server via mtime-based cache invalidation.

---

## Checkpointer System

State persistence for multi-turn conversations:

| Type | Backend | Use Case |
|------|---------|----------|
| `memory` | In-process dict | Development, single-turn |
| `sqlite` | File-based SQLite | Single-process persistence |
| `postgres` | PostgreSQL | Production, multi-process |

**For LangGraph Server**: The server manages its own checkpointing infrastructure — `config.yaml` checkpointer settings don't affect it.

**For DeerFlowClient**: The embedded client uses the configured checkpointer. Without one, each call is stateless — `thread_id` is only used for file isolation.

```yaml
# config.yaml
checkpointer:
  type: sqlite
  connection_string: checkpoints.db
```

---

## Reflection System

DeerFlow uses runtime reflection for extensibility. The `resolve_variable()` and `resolve_class()` functions in `deerflow/reflection/` dynamically import modules and validate classes:

```python
# config.yaml uses string paths like:
use: langchain_anthropic:ChatAnthropic
use: deerflow.sandbox.local:LocalSandboxProvider
use: deerflow.community.tavily.tools:web_search_tool

# At runtime, these are resolved to actual Python objects:
model_class = resolve_class("langchain_anthropic:ChatAnthropic", BaseChatModel)
tool_var = resolve_variable("deerflow.community.tavily.tools:web_search_tool")
```

This enables **zero-code extensibility** — add a new model provider or tool by editing config.yaml, no Python changes needed.
