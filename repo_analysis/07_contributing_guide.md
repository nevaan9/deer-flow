# Contributing Guide

## Contribution Categories

### 1. Backend — Agent System
**Where**: `backend/packages/harness/deerflow/agents/`

- **New Middleware**: Add cross-cutting concerns to the middleware chain
  - Create in `middlewares/` following existing patterns
  - Register in `_build_middlewares()` in `lead_agent/agent.py` (order matters!)
  - Add configuration option in `config.yaml`
  - Write tests

- **Agent Improvements**: Enhance the lead agent or create new subagent types
  - Lead agent: `lead_agent/agent.py`
  - Subagent types: `subagents/builtins/`
  - System prompt: `lead_agent/prompt.py`

### 2. Backend — Tools
**Where**: `backend/packages/harness/deerflow/tools/` and `community/`

- **Built-in Tools**: Core tools available to all agents
  - Location: `tools/builtins/`
  - Examples: present_file, ask_clarification, view_image, task

- **Community Tools**: Third-party integrations
  - Location: `community/{provider_name}/`
  - Examples: tavily (web search), jina_ai (web fetch), firecrawl (scraping)
  - Each gets its own subdirectory with `tools.py`
  - Register in `config.example.yaml`

### 3. Backend — MCP
**Where**: `backend/packages/harness/deerflow/mcp/`

- MCP client improvements
- New transport support
- OAuth flow enhancements
- Tool caching optimizations

### 4. Backend — Sandbox
**Where**: `backend/packages/harness/deerflow/sandbox/`

- New sandbox providers (e.g., Firecracker, gVisor)
- Sandbox tool improvements
- Path translation enhancements
- Security hardening

### 5. Frontend — Components
**Where**: `frontend/src/components/`

- **Workspace**: Chat interface, message rendering, artifact display
  - Location: `components/workspace/`
- **Landing Page**: Homepage sections
  - Location: `components/landing/`
- **UI Primitives**: Shadcn-based components
  - Location: `components/ui/` (auto-generated, ESLint-ignored)

### 6. Frontend — Core Logic
**Where**: `frontend/src/core/`

- **Threads**: Thread management, streaming, state
- **Artifacts**: Artifact loading and caching
- **Memory**: Memory display and management
- **Skills**: Skill installation and management
- **MCP**: MCP server configuration UI
- **i18n**: Internationalization (en-US, zh-CN)

### 7. Skills
**Where**: `skills/public/` or `skills/custom/`

- Create new public skills for common workflows
- Each skill is a directory with `SKILL.md` (YAML frontmatter + instructions)
- Skills need: name, description, license, allowed-tools

### 8. IM Channels
**Where**: `backend/app/channels/`

- New platform integrations (Discord, WhatsApp, etc.)
- Improve existing: Feishu, Slack, Telegram
- Base class: `base.py` (abstract Channel)

### 9. Documentation & i18n
**Where**: `backend/docs/`, `README.md`, `frontend/src/core/i18n/`

- API documentation
- Architecture guides
- Setup tutorials
- README translations (already: EN, CN, JP, FR, RU)
- Frontend i18n strings

---

## Development Workflow

### 1. Fork and Clone

```bash
git clone https://github.com/your-fork/deer-flow.git
cd deer-flow
```

### 2. Create Feature Branch

```bash
git checkout -b feature/your-feature-name
```

### 3. Install Dependencies

```bash
make install
```

### 4. Start Development

```bash
make dev  # Starts all services with hot-reload
```

### 5. Make Changes and Test

```bash
# Backend tests
cd backend
make test                                          # All tests
PYTHONPATH=. uv run pytest tests/test_feature.py -v  # Specific test

# Frontend checks
cd frontend
pnpm check      # ESLint + TypeScript type check
pnpm lint:fix   # Auto-fix lint issues
```

### 6. Commit

Follow conventional commit format:
```bash
git commit -m "feat: add new web scraping tool"
git commit -m "fix: handle timeout in memory updater"
git commit -m "docs: update API reference for skills"
```

### 7. Push and Create PR

```bash
git push origin feature/your-feature-name
# Then create PR on GitHub
```

---

## Code Style

### Python (Backend)

- **Linter/Formatter**: `ruff`
- **Line length**: 240 characters
- **Quotes**: Double quotes
- **Indentation**: 4 spaces
- **Type hints**: Required
- **Target version**: Python 3.12

```bash
# Check style
cd backend
make lint

# Auto-format
make format
```

Key ruff rules enforced:
- `E` — Pycodestyle errors
- `F` — Pyflakes
- `I` — Import sorting (isort)
- `UP` — Pyupgrade

### TypeScript (Frontend)

- **Linter**: ESLint with TypeScript support
- **Formatter**: Prettier
- **Strict mode**: Enabled
- **Unused variables**: Prefix with `_`
- **Path aliases**: `@/*` maps to `src/*`

```bash
# Check everything
cd frontend
pnpm check       # ESLint + TypeScript

# Auto-fix
pnpm lint:fix
```

---

## Testing

### Mandatory TDD

Every feature or bug fix **must** include tests. No exceptions.

### Test Structure

```
backend/tests/
├── test_memory_updater.py              # Memory fact extraction
├── test_memory_prompt_injection.py     # Memory injection security
├── test_sandbox_tools_security.py      # Path traversal protection
├── test_guardrail_middleware.py        # Authorization middleware
├── test_task_tool_core_logic.py        # Subagent tool behavior
├── test_present_file_tool_core_logic.py # File presentation
├── test_subagent_executor.py           # Background execution
├── test_subagent_timeout_config.py     # Timeout handling
├── test_harness_boundary.py            # Import boundary enforcement
├── test_docker_sandbox_mode_detection.py # Docker mode detection
├── test_provisioner_kubeconfig.py      # Kubernetes config
├── test_client.py                      # DeerFlowClient (77 unit tests)
├── test_client_live.py                 # Live integration tests
├── test_mcp_client_config.py           # MCP configuration
├── test_mcp_sync_wrapper.py            # MCP async wrapper
├── test_skills_archive_root.py         # Skill installation
├── test_config_version.py              # Config versioning
├── test_app_config_reload.py           # Config auto-reload
└── ... (52+ total test modules)
```

### Running Tests

```bash
cd backend

# All tests
make test

# Specific test
PYTHONPATH=. uv run pytest tests/test_memory_updater.py -v

# With coverage
PYTHONPATH=. uv run pytest --cov=deerflow tests/
```

### Test Patterns

**Circular import avoidance**: Some modules have complex import graphs. Use `sys.modules` mocks in `tests/conftest.py`:

```python
# conftest.py — prevents circular imports during test collection
sys.modules["deerflow.subagents.executor"] = MagicMock()
```

**Gateway conformance tests**: `TestGatewayConformance` in `test_client.py` validates that every `DeerFlowClient` method's return dict passes through the corresponding Gateway Pydantic model.

---

## CI/CD

### GitHub Actions

**Backend Tests** (`.github/workflows/backend-unit-tests.yml`):
- Trigger: Push to main, pull requests
- Python 3.12, uv package manager
- Runs full test suite including regression tests:
  - Docker sandbox mode detection
  - Provisioner kubeconfig handling
  - Harness/app import boundary

### Regression Tests (Always Run in CI)

These tests protect against regressions in critical subsystems:

| Test | Protects |
|------|----------|
| `test_harness_boundary.py` | Harness never imports from app |
| `test_docker_sandbox_mode_detection.py` | Docker mode detected correctly |
| `test_provisioner_kubeconfig.py` | Kubernetes config handling |

---

## Harness Boundary Rules

The most important architectural rule:

> **`deerflow.*` (harness) must NEVER import from `app.*`**

This is enforced by `test_harness_boundary.py` which scans all Python files in `packages/harness/deerflow/` and fails if any `from app.*` or `import app.*` is found.

**Why**: The harness is a publishable pip package. It must work without the app layer.

**Allowed**:
```python
# In app/ code:
from deerflow.config import get_app_config     # App → Harness ✓
from deerflow.models import create_chat_model   # App → Harness ✓
```

**Forbidden**:
```python
# In deerflow/ code:
from app.gateway.routers import uploads         # Harness → App ✗ (CI fails)
```

---

## Where to Add Things

### New Tool

1. Create `backend/packages/harness/deerflow/community/{provider}/tools.py`
2. Define tool using LangChain's `@tool` decorator
3. Add configuration to `config.example.yaml` under `tools:`
4. Write tests in `backend/tests/test_{provider}_tools.py`

### New Middleware

1. Create `backend/packages/harness/deerflow/agents/middlewares/{name}_middleware.py`
2. Register in `_build_middlewares()` in `lead_agent/agent.py`
3. Add config option if needed
4. Write tests
5. Document ordering constraints

### New Skill

1. Create `skills/public/{skill-name}/SKILL.md`
2. Add YAML frontmatter: name, description, license, allowed-tools
3. Write workflow instructions in Markdown
4. Add to `extensions_config.example.json`

### New MCP Server

1. Add example to `extensions_config.example.json`
2. Document required API keys in `.env.example`
3. Test with `extensions_config.json` locally

### New IM Channel

1. Create `backend/app/channels/{platform}.py`
2. Extend `base.py` Channel class
3. Add to `manager.py` for lifecycle management
4. Add config section to `config.example.yaml`
5. Document setup in `backend/docs/`
