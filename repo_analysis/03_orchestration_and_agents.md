# Orchestration and Agents

## Complete Request Flow

Here's what happens when a user sends "Search for climate change impacts":

```
1. USER SUBMITS MESSAGE
   Frontend → LangGraph SDK → POST /api/langgraph/threads/{id}/runs (stream)

2. NGINX PROXY
   /api/langgraph/* → LangGraph Server (port 2024)

3. LANGGRAPH SERVER
   Invokes make_lead_agent(config) with:
   config.configurable = {
     thread_id: "abc-123",
     model_name: "claude-3-5-sonnet",
     thinking_enabled: true,
     subagent_enabled: true,
     max_concurrent_subagents: 3
   }

4. AGENT CREATION
   a) Resolve model → create_chat_model("claude-3-5-sonnet", thinking=true)
   b) Load tools → get_available_tools()
      - Config tools: web_search, web_fetch, image_search, ls, read_file, write_file, bash
      - Built-ins: present_file, ask_clarification, view_image
      - MCP tools: [from enabled servers]
      - Subagent: task (if enabled)
   c) Generate system prompt → apply_prompt_template()
      - Role + memory (top 15 facts) + skills list + subagent instructions
   d) Build middleware chain → _build_middlewares()
   e) create_agent(model, tools, middleware, prompt, state_schema=ThreadState)

5. AGENT EXECUTION LOOP
   ┌─────────────────────────────────────────────────────┐
   │ Iteration 1:                                        │
   │  before_model: ThreadData, Uploads, Sandbox, etc.   │
   │  LLM call → decides to use web_search_tool          │
   │  wrap_tool_call: web_search("climate change")       │
   │  after_model: Title generates, Memory queues         │
   │                                                     │
   │ Iteration 2:                                        │
   │  LLM sees search results → calls web_fetch(url)    │
   │  wrap_tool_call: web_fetch executed                  │
   │                                                     │
   │ Iteration 3 (final):                                │
   │  LLM synthesizes response (no tool calls)           │
   │  after_model: Title set, Memory queued              │
   │  Return ThreadState                                 │
   └─────────────────────────────────────────────────────┘

6. STREAMING RESPONSES (SSE)
   Server → Client:
   event: messages-tuple  data: {type: "ai", content: "...", tool_calls: [...]}
   event: messages-tuple  data: {type: "tool", name: "web_search", content: "..."}
   event: values          data: {title: "Climate Change Research", messages: [...]}
   event: end             data: {}

7. FRONTEND RENDERING
   Renders title, messages, artifacts, follow-up suggestions

8. BACKGROUND MEMORY UPDATE (30s debounce)
   MemoryUpdateQueue waits → LLM extracts facts → memory.json updated
```

---

## LangGraph Integration

DeerFlow registers its agent with LangGraph via `backend/langgraph.json`:

```json
{
  "graphs": {
    "lead_agent": "deerflow.agents:make_lead_agent"
  },
  "checkpointer": {
    "path": "deerflow.agents.checkpointer.async_provider:make_checkpointer"
  }
}
```

The LangGraph Server:
- Manages thread lifecycle (create, list, delete)
- Handles state checkpointing between turns
- Provides the SSE streaming protocol
- Routes `config.configurable` to `make_lead_agent()`

---

## Tool Assembly Pipeline

`get_available_tools()` in `backend/packages/harness/deerflow/tools/tools.py` assembles tools from five sources:

```
┌─────────────────────────────────────────────────┐
│            get_available_tools()                  │
│                                                  │
│  1. Config-Defined Tools (config.yaml)           │
│     ├── web_search (tavily)                      │
│     ├── web_fetch (jina_ai)                      │
│     ├── image_search (duckduckgo)                │
│     ├── ls, read_file, write_file, str_replace   │
│     └── bash                                     │
│     Resolved via resolve_variable() reflection   │
│     Filtered by tool_groups                      │
│                                                  │
│  2. MCP Tools (extensions_config.json)           │
│     ├── Lazy loaded on first use                 │
│     ├── Cached with mtime invalidation           │
│     └── Optional: deferred via tool_search       │
│                                                  │
│  3. Built-in Tools (always present)              │
│     ├── present_file  (show outputs to user)     │
│     └── ask_clarification (interrupt for input)  │
│                                                  │
│  4. Vision Tool (if model supports it)           │
│     └── view_image (read images as base64)       │
│                                                  │
│  5. Subagent Tool (if enabled)                   │
│     └── task (delegate to parallel subagents)    │
│                                                  │
│  6. Tool Search (if deferred loading enabled)    │
│     └── tool_search (discover MCP tools)         │
└─────────────────────────────────────────────────┘
```

### Tool Search (Deferred Loading)

When MCP servers expose hundreds of tools, sending all schemas to the model is expensive. The tool search system solves this:

1. When `tool_search.enabled: true`, MCP tools are NOT bound to the model
2. A `tool_search_tool` is added instead
3. The model calls `tool_search("query")` with a natural language description
4. The registry matches tools by description similarity
5. Matched tools are dynamically available for the current turn

---

## Subagent System

DeerFlow supports parallel task delegation via the `task` tool.

### Architecture

```
Lead Agent
    │
    ├── task(description="Research topic A", subagent_type="general-purpose")
    ├── task(description="Run tests", subagent_type="bash")
    └── task(description="Analyze data", subagent_type="general-purpose")
          │
          ▼
    SubagentExecutor
    ┌──────────────────┐     ┌──────────────────┐
    │ _scheduler_pool  │────▶│ _execution_pool  │
    │ (3 workers)      │     │ (3 workers)      │
    │ Schedule + Poll  │     │ Execute agents   │
    └──────────────────┘     └──────────────────┘
          │
          ▼
    SSE Events: task_started → task_running → task_completed
```

### Built-in Subagents

| Name | Access | Use Case |
|------|--------|----------|
| `general-purpose` | All tools except `task` | Research, analysis, code exploration |
| `bash` | Command execution | Build, test, deploy operations |

### Concurrency Control

- **Hard limit**: 3 concurrent subagents per turn (enforced by `SubagentLimitMiddleware`)
- **Timeout**: 15 minutes per subagent (configurable per agent type)
- **Batching**: For >3 sub-tasks, execute in batches of 3 across turns
- **Truncation**: Excess `task` calls are silently dropped by the middleware

---

## Sandbox System

The sandbox provides isolated code execution with virtual path translation.

### Interface

```python
class Sandbox(ABC):
    async def execute_command(self, command: str) -> str
    async def read_file(self, path: str, start_line=None, end_line=None) -> str
    async def write_file(self, path: str, content: str, mode="write") -> str
    async def list_dir(self, path: str, max_depth=2) -> str
```

### Implementations

| Provider | How It Works | When to Use |
|----------|-------------|-------------|
| `LocalSandboxProvider` | Direct filesystem execution on host | Local development |
| `AioSandboxProvider` | Docker container isolation | Production, security-sensitive |

### Virtual Path System

The agent never sees real host paths. Everything is abstracted:

```
Agent sees (virtual):                  Host reality (physical):
/mnt/user-data/workspace    ←→    backend/.deer-flow/threads/{id}/user-data/workspace
/mnt/user-data/uploads      ←→    backend/.deer-flow/threads/{id}/user-data/uploads
/mnt/user-data/outputs      ←→    backend/.deer-flow/threads/{id}/user-data/outputs
/mnt/skills                 ←→    deer-flow/skills/
```

**Security**: Path traversal protection blocks `..` and backslash variants. The `replace_virtual_path()` function handles bidirectional translation between virtual and physical paths. Host paths in tool output are automatically masked.

### Sandbox Tools

| Tool | Purpose |
|------|---------|
| `bash` | Execute shell commands with path translation |
| `ls` | Tree-format directory listing (max 2 levels) |
| `read_file` | Read with optional line range |
| `write_file` | Create/append with auto-mkdir |
| `str_replace` | Substring replacement (single or all) |

---

## MCP Integration

Model Context Protocol enables connecting to external tool servers.

### Transport Support

| Transport | Configuration | Use Case |
|-----------|--------------|----------|
| **stdio** | `command` + `args` + `env` | Local tool servers (npx, uvx) |
| **SSE** | `url` + `headers` | Remote streaming servers |
| **HTTP** | `url` + `headers` | REST-based tool servers |

### OAuth Support (HTTP/SSE)

```json
{
  "type": "http",
  "url": "https://tool-server.example.com",
  "oauth": {
    "token_url": "https://auth.example.com/token",
    "client_id": "...",
    "client_secret": "...",
    "grant_type": "client_credentials"
  }
}
```

Supports `client_credentials` and `refresh_token` flows with automatic token refresh and Authorization header injection.

### Caching Strategy

- Tools are lazy-loaded on first request
- Cached in-process with mtime-based invalidation
- When `extensions_config.json` is modified (via Gateway API or manually), the next request detects the mtime change and reloads tools

---

## Skills System

Skills are Markdown-based agent workflow definitions.

### Directory Structure

```
skills/
├── public/                    # 10+ shipped skills
│   ├── deep-research/
│   │   └── SKILL.md
│   ├── report-generation/
│   │   └── SKILL.md
│   ├── slide-creation/
│   │   └── SKILL.md
│   ├── image-generation/
│   │   └── SKILL.md
│   ├── podcast-generation/
│   │   └── SKILL.md
│   └── ...
└── custom/                    # User-defined (gitignored)
    └── my-skill/
        └── SKILL.md
```

### SKILL.md Format

```yaml
---
name: "deep-research"
description: "Perform deep web research with multi-source synthesis"
license: "MIT"
allowed-tools:
  - web_search
  - web_fetch
---
# Deep Research Skill

When the user asks for research on a topic...
[Markdown workflow instructions]
```

### Skill Lifecycle

1. **Discovery**: `load_skills()` scans `skills/{public,custom}/` recursively for `SKILL.md`
2. **Parsing**: YAML frontmatter extracted for metadata
3. **Enable/Disable**: Controlled via `extensions_config.json` or Gateway API
4. **Injection**: Enabled skills listed in the agent's system prompt with their container paths
5. **Installation**: `POST /api/skills/install` accepts `.skill` ZIP archives, validates, and extracts to `custom/`
