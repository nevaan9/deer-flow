# DeerFlow: Project Overview

## What is DeerFlow?

DeerFlow is an open-source **AI super agent harness** developed by ByteDance. Originally a Deep Research framework (v1.x), it was completely rewritten in v2.0 as a general-purpose task execution harness capable of orchestrating sub-agents, persistent memory, sandboxed execution, and extensible skills.

It enables AI agents to:
- Execute code in sandboxed environments
- Browse the web and analyze documents
- Manage files with path isolation per conversation thread
- Delegate tasks to parallel sub-agents
- Maintain long-term memory across conversations
- Integrate with external tools via Model Context Protocol (MCP)

**License**: MIT
**Origin**: ByteDance
**Status**: Ranked #1 on GitHub Trending (February 2026)

---

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| **Backend Runtime** | Python | 3.12+ |
| **Web Framework** | FastAPI | 0.115+ |
| **Agent Orchestration** | LangGraph | 0.2.52+ |
| **LLM Framework** | LangChain | 0.2+ |
| **ASGI Server** | Uvicorn | 0.34+ |
| **Frontend Framework** | Next.js (App Router) | 16+ |
| **UI Library** | React | 19 |
| **Frontend Language** | TypeScript | 5.8 |
| **CSS Framework** | Tailwind CSS | 4.0 |
| **Package Manager (Python)** | uv | latest |
| **Package Manager (Node)** | pnpm | 10.26 |
| **Reverse Proxy** | Nginx | latest |
| **Containerization** | Docker / Docker Compose | latest |

---

## Directory Structure

```
deer-flow/
├── backend/                        # Python backend (LangGraph + FastAPI)
│   ├── packages/harness/           # deerflow-harness publishable package
│   │   └── deerflow/               # Main harness module (import: deerflow.*)
│   │       ├── agents/             # Lead agent, middlewares, memory, thread state
│   │       │   ├── lead_agent/     # Agent factory + system prompt generation
│   │       │   ├── middlewares/    # 15 middleware components (AOP pattern)
│   │       │   ├── memory/         # Memory extraction, update queue, prompts
│   │       │   ├── checkpointer/   # State persistence providers
│   │       │   └── thread_state.py # ThreadState schema with custom reducers
│   │       ├── sandbox/            # Execution environments (local, Docker)
│   │       │   ├── local/          # LocalSandboxProvider (filesystem)
│   │       │   ├── sandbox.py      # Abstract Sandbox interface
│   │       │   ├── tools.py        # bash, ls, read_file, write_file, str_replace
│   │       │   └── middleware.py   # Sandbox lifecycle management
│   │       ├── subagents/          # Task delegation system
│   │       │   ├── builtins/       # general-purpose, bash agent definitions
│   │       │   ├── executor.py     # Background execution with thread pools
│   │       │   └── registry.py     # Agent discovery and configuration
│   │       ├── tools/              # Tool system (builtins, MCP, community)
│   │       │   └── builtins/       # present_file, ask_clarification, view_image, task
│   │       ├── mcp/                # Model Context Protocol integration
│   │       ├── models/             # Model factory with thinking/vision support
│   │       ├── skills/             # Skills discovery, loading, validation
│   │       ├── config/             # Configuration system (YAML + JSON)
│   │       ├── community/          # Community tools (tavily, jina, firecrawl, etc.)
│   │       ├── guardrails/         # Pre-tool-call authorization
│   │       ├── reflection/         # Dynamic module loading (resolve_variable/class)
│   │       ├── utils/              # Network, readability, file conversion
│   │       └── client.py           # Embedded Python client (DeerFlowClient)
│   ├── app/                        # Application layer (import: app.*)
│   │   ├── gateway/                # FastAPI REST API (port 8001)
│   │   │   ├── routers/            # 12 route modules
│   │   │   ├── app.py              # FastAPI initialization
│   │   │   └── config.py           # Gateway configuration
│   │   └── channels/               # IM integrations (Feishu, Slack, Telegram)
│   ├── tests/                      # 52+ test modules
│   ├── docs/                       # 20+ documentation files
│   ├── langgraph.json              # LangGraph server configuration
│   ├── pyproject.toml              # Project metadata + dependencies
│   └── Dockerfile                  # Backend container image
├── frontend/                       # Next.js 16 web interface
│   ├── src/
│   │   ├── app/                    # App Router pages (/workspace/chats/[thread_id])
│   │   ├── components/             # UI (shadcn), workspace (chat), landing
│   │   ├── core/                   # Business logic (threads, artifacts, memory, MCP)
│   │   ├── hooks/                  # Custom React hooks
│   │   ├── lib/                    # Utilities (cn from clsx + tailwind-merge)
│   │   └── styles/                 # Global CSS + Tailwind v4
│   ├── package.json
│   └── next.config.js
├── docker/                         # Docker deployment configs
│   ├── docker-compose.yaml         # Production compose
│   ├── docker-compose-dev.yaml     # Development compose (hot-reload)
│   ├── nginx/                      # Nginx reverse proxy config
│   └── provisioner/                # Optional Kubernetes provisioner
├── skills/                         # Agent skills (Markdown-based)
│   ├── public/                     # 10+ shipped skills
│   └── custom/                     # User-defined skills (gitignored)
├── scripts/                        # Automation scripts
│   ├── check.py                    # System requirements verification
│   ├── configure.py                # Config file generation
│   ├── serve.sh                    # Local development startup
│   └── deploy.sh                   # Production deployment
├── Makefile                        # Root commands (check, install, dev, stop)
├── config.example.yaml             # Configuration template (500+ lines)
├── extensions_config.example.json  # MCP + skills config template
├── CONTRIBUTING.md                 # Development guide
└── README.md                       # Main documentation (5 translations)
```

---

## Key Dependencies

### Backend (Python)

| Package | Role |
|---------|------|
| `langgraph` | Agent graph orchestration, state management, checkpointing |
| `langchain` | LLM abstraction, agent creation, middleware system |
| `langchain-openai` / `langchain-anthropic` / `langchain-google-genai` | LLM provider integrations |
| `langchain-mcp-adapters` | Model Context Protocol tool integration |
| `fastapi` + `uvicorn` | Gateway REST API server |
| `pydantic` | Data validation and serialization |
| `tavily-python` | Web search tool |
| `markitdown` | Document conversion (PDF/PPT/Excel/Word to Markdown) |
| `httpx` | Async HTTP client |
| `sse-starlette` | Server-sent events for streaming |
| `slack-sdk` / `python-telegram-bot` / `lark-oapi` | IM platform integrations |

### Frontend (Node.js)

| Package | Role |
|---------|------|
| `next` (16+) | Meta-framework (App Router, SSR, API routes) |
| `react` (19) | UI library |
| `@langchain/langgraph-sdk` | Backend communication (threads, runs, streaming) |
| `@tanstack/react-query` | Server state management |
| `ai` (Vercel AI SDK) | Streaming UI components |
| `@radix-ui/*` + `shadcn/ui` | Component primitives |
| `tailwindcss` (4) | Utility-first CSS |
| `@uiw/react-codemirror` | Code editor with syntax highlighting |
| `shiki` / `remark-gfm` / `rehype-katex` | Markdown rendering with code + math |
| `gsap` / `motion` | Animations |

---

## Harness / App Architectural Split

The backend enforces a strict two-layer architecture:

```
┌──────────────────────────────────────────┐
│  App Layer (app/*)                        │
│  - FastAPI Gateway API                   │
│  - IM Channel integrations               │
│  - Imports deerflow.* (ALLOWED)          │
└────────────────┬─────────────────────────┘
                 │ depends on
┌────────────────▼─────────────────────────┐
│  Harness Layer (deerflow/*)              │
│  - Agent orchestration                   │
│  - Tools, sandbox, models                │
│  - MCP, skills, config                   │
│  - NEVER imports app.* (ENFORCED BY CI)  │
└──────────────────────────────────────────┘
```

**Why this matters:**
- The Harness (`deerflow-harness`) is a **publishable pip package** — anyone can `pip install deerflow-harness` and use the agent system without the Gateway or IM channels.
- The App layer is the **unpublished application** that adds HTTP APIs and platform integrations on top.
- The boundary is enforced by `tests/test_harness_boundary.py` which scans all harness source files for `app.*` imports and fails CI if any are found.

This pattern is a textbook example of the **Dependency Inversion Principle** — the stable, reusable core has zero knowledge of the volatile application layer that depends on it.

---

## What is Make and the Makefile?

### The Concept

`make` is a build automation tool that originated in Unix (1976). It reads instructions from a file called `Makefile` and executes shell commands based on named **targets**. Think of it as a task runner — similar to `npm scripts` in Node.js or `gradle tasks` in Java — but language-agnostic and universally available on Unix/macOS/Linux systems.

### Why DeerFlow Uses Make

DeerFlow is a polyglot project: Python backend, Node.js frontend, Nginx proxy, Docker containers, and shell scripts all need to work together. Make provides a **single entry point** for all operations regardless of which language or tool is involved:

```bash
make check     # Runs a Python script (scripts/check.py)
make install   # Runs uv sync (Python) AND pnpm install (Node.js)
make dev       # Runs a Bash script that starts 4 services
make up        # Runs Docker Compose via deploy.sh
```

Without Make, you'd need to remember different commands for different tools in different directories. Make unifies them.

### How a Makefile Works

```makefile
# A target with a recipe (the indented lines MUST use actual Tab characters, not spaces)
target_name:
	@echo "This is a shell command"
	@python scripts/check.py

# Phony targets don't produce files — they just run commands
.PHONY: check install dev
```

**Key concepts**:
- **Target**: A named task (e.g., `check`, `install`, `dev`)
- **Recipe**: Shell commands to execute (MUST be indented with a real Tab character)
- **`.PHONY`**: Declares that a target is a command, not a file to build
- **`@` prefix**: Suppresses printing the command itself (only shows output)
- **Variables**: `PYTHON ?= python` sets a default that can be overridden

### DeerFlow's Makefile Hierarchy

DeerFlow has **two Makefiles** that serve different scopes:

**Root `Makefile`** (full application orchestration):

| Target | What It Does | Underlying Command |
|--------|-------------|-------------------|
| `make check` | Verify system requirements (Python, Node, pnpm, uv, nginx) | `python scripts/check.py` |
| `make config` | Generate config.yaml and extensions_config.json from examples | `python scripts/configure.py` |
| `make config-upgrade` | Merge new fields into existing config.yaml | `scripts/config-upgrade.sh` |
| `make install` | Install ALL dependencies (backend + frontend) | `cd backend && uv sync` + `cd frontend && pnpm install` |
| `make dev` | Start all 4 services with hot-reload | `scripts/serve.sh --dev` |
| `make start` | Start all 4 services in production mode | `scripts/serve.sh --prod` |
| `make dev-daemon` | Start services in background | `scripts/start-daemon.sh` |
| `make stop` | Stop all running services | `pkill` for each process + nginx quit |
| `make clean` | Stop + delete runtime data | Removes `.deer-flow/`, `.langgraph_api/`, logs |
| `make setup-sandbox` | Pre-pull Docker sandbox image | `docker pull` / `container pull` |
| `make up` | Build + start production Docker containers | `scripts/deploy.sh` |
| `make down` | Stop + remove production containers | `scripts/deploy.sh down` |
| `make docker-init` | Initialize Docker dev environment | `scripts/docker.sh init` |
| `make docker-start` | Start Docker dev services | `scripts/docker.sh start` |
| `make docker-stop` | Stop Docker dev services | `scripts/docker.sh stop` |
| `make docker-logs` | View Docker dev logs | `scripts/docker.sh logs` |

**Backend `Makefile`** (backend-only development):

| Target | What It Does | Underlying Command |
|--------|-------------|-------------------|
| `make install` | Install backend Python dependencies | `uv sync` |
| `make dev` | Start LangGraph server only | `uv run langgraph dev --no-browser --allow-blocking --no-reload` |
| `make gateway` | Start Gateway API only | `PYTHONPATH=. uv run uvicorn app.gateway.app:app --host 0.0.0.0 --port 8001` |
| `make test` | Run all backend tests | `PYTHONPATH=. uv run pytest tests/ -v` |
| `make lint` | Check code style | `uvx ruff check .` |
| `make format` | Auto-fix code style | `uvx ruff check . --fix && uvx ruff format .` |

### Common Gotcha: "No rule to make target"

This error means one of:
1. **Wrong directory**: You're not in the directory that contains the Makefile. The root Makefile must be run from the project root (`deer-flow/`), and the backend Makefile from `backend/`.
2. **Tab vs spaces**: Makefile recipes require literal Tab characters. If someone's editor converted tabs to spaces, recipes break silently.
3. **Typo**: The target name doesn't exist in the Makefile.

---

## Package Management

DeerFlow uses different package managers for different ecosystems, unified through Make targets.

### Python: uv + Workspaces

**`uv`** is an extremely fast Python package manager (written in Rust) that replaces pip, pip-tools, virtualenv, and poetry in a single tool.

**Workspace structure**:
```
backend/
├── pyproject.toml              # Root project: "deer-flow" (the app)
│   └── [tool.uv.workspace]
│       members = ["packages/harness"]
│   └── [tool.uv.sources]
│       deerflow-harness = { workspace = true }
│
└── packages/harness/
    └── pyproject.toml          # Sub-package: "deerflow-harness" (publishable)
```

**How it works**:
- `backend/pyproject.toml` defines the **application** (`deer-flow`) — depends on `deerflow-harness` + app-specific deps (FastAPI, IM SDKs)
- `backend/packages/harness/pyproject.toml` defines the **library** (`deerflow-harness`) — all agent, tool, model, MCP dependencies
- `[tool.uv.workspace]` tells uv these are linked packages — changes to `deerflow-harness` are immediately available to the app without reinstalling
- `[tool.uv.sources]` with `workspace = true` means "use the local copy, not PyPI"

**Key commands**:
```bash
cd backend
uv sync              # Install all dependencies (both root + workspace members)
uv run pytest ...    # Run command inside the managed virtualenv
uv add httpx         # Add a dependency to backend/pyproject.toml
uvx ruff check .     # Run a tool without installing it permanently
```

**Lock file**: `uv.lock` pins exact versions for reproducible builds. Committed to git.

**Build system**: The harness uses `hatchling` as its build backend — this is what enables `pip install deerflow-harness` from the built wheel.

### Node.js: pnpm

**`pnpm`** is a fast, disk-efficient Node.js package manager. DeerFlow pins it to version 10.26.2 via `packageManager` in `package.json`.

**Key commands**:
```bash
cd frontend
pnpm install         # Install all dependencies from pnpm-lock.yaml
pnpm dev             # Start dev server with Turbopack (Next.js 16)
pnpm build           # Production build
pnpm check           # ESLint + TypeScript type checking
pnpm lint:fix        # Auto-fix lint issues
```

**Lock file**: `pnpm-lock.yaml` pins exact versions. Committed to git.

### Why Two Package Managers?

| Ecosystem | Manager | Lock File | Reason |
|-----------|---------|-----------|--------|
| Python | `uv` | `uv.lock` | Backend runtime, agent system, LLM integrations |
| Node.js | `pnpm` | `pnpm-lock.yaml` | Frontend UI, Next.js framework, React components |

There's no cross-ecosystem dependency. The backend and frontend communicate only via HTTP (through Nginx).

---

## Scripts System

The `scripts/` directory contains automation that the Makefile targets delegate to:

### `scripts/check.py` — System Requirements Verification

Called by `make check`. Validates:
- Python 3.12+ installed
- Node.js 22+ installed
- pnpm 10+ installed
- `uv` installed
- `nginx` installed

Outputs clear pass/fail with installation instructions for each missing tool.

### `scripts/configure.py` — Config File Generation

Called by `make config`. Copies example files to create initial configs:
- `config.example.yaml` → `config.yaml`
- `extensions_config.example.json` → `extensions_config.json`

Aborts if config files already exist (won't overwrite).

### `scripts/config-upgrade.sh` — Config Migration

Called by `make config-upgrade` and automatically on `make dev`. Merges new fields from `config.example.yaml` into an existing `config.yaml` without overwriting user customizations. Uses the `config_version` field to detect outdated configs.

### `scripts/serve.sh` — Local Development Startup

Called by `make dev` (with `--dev`) or `make start` (with `--prod`). This is the main orchestration script:

1. **Stops** any existing services (pkill + nginx quit)
2. **Validates** config.yaml exists
3. **Auto-upgrades** config if needed
4. **Starts 4 services** sequentially with port-readiness checks:
   - LangGraph Server (port 2024) — `uv run langgraph dev`
   - Gateway API (port 8001) — `uvicorn app.gateway.app:app`
   - Frontend (port 3000) — `pnpm run dev` (dev) or `pnpm run preview` (prod)
   - Nginx (port 2026) — reverse proxy
5. **Registers cleanup trap** — Ctrl+C gracefully stops all services
6. **Logs** written to `logs/{langgraph,gateway,frontend,nginx}.log`

**Dev vs Prod differences**:

| Aspect | Dev (`make dev`) | Prod (`make start`) |
|--------|-----------------|-------------------|
| Frontend | `next dev --turbo` (hot reload) | `next build && next start` |
| Gateway | `--reload` (watches .yaml, .env) | No reload |
| LangGraph | Default (reload enabled) | `--no-reload` |

### `scripts/deploy.sh` — Docker Production Deployment

Called by `make up` / `make down`. Handles:

1. **Environment setup**: Sets `DEER_FLOW_HOME`, config paths, Docker socket path
2. **Config seeding**: Copies `config.example.yaml` if no `config.yaml` exists
3. **Auth secret**: Generates and persists `BETTER_AUTH_SECRET` for frontend sessions
4. **Sandbox mode detection**: Reads `config.yaml` to determine `local`, `aio`, or `provisioner` mode
5. **Docker Compose**: Builds and starts containers with appropriate profiles

### `scripts/wait-for-port.sh` — Port Readiness Check

Used by `serve.sh` to block until a service is accepting connections on its port. Takes port number, timeout (seconds), and service name as arguments.

### `scripts/docker.sh` — Docker Dev Environment

Called by `make docker-init`, `make docker-start`, etc. Manages the development Docker Compose environment (separate from the production `deploy.sh`).

---

## Deployment Options

DeerFlow supports three deployment strategies:

### Option 1: Local Development (`make dev`)

```
┌─────────────────────────────────────────────────┐
│  Your Machine (all processes run natively)        │
│                                                  │
│  Nginx (2026) ─┬─▶ LangGraph Server (2024)      │
│                ├─▶ Gateway API (8001)             │
│                └─▶ Frontend (3000)                │
│                                                  │
│  All logs → logs/ directory                      │
│  Hot-reload enabled for all services             │
└─────────────────────────────────────────────────┘
```

**When to use**: Day-to-day development, debugging, testing changes.

**Requirements**: Python 3.12+, Node.js 22+, pnpm, uv, nginx (all installed locally).

**Commands**:
```bash
make check     # Verify requirements
make install   # Install dependencies
make config    # Generate config files (first time)
# Edit config.yaml to add model API keys
make dev       # Start everything
# Open http://localhost:2026
make stop      # Shut down
```

### Option 2: Docker Production (`make up`)

```
┌─────────────────────────────────────────────────┐
│  Docker Network: deer-flow                       │
│                                                  │
│  ┌───────────────────────────────┐               │
│  │ nginx (port 2026 exposed)     │               │
│  │  ├─▶ frontend container       │               │
│  │  ├─▶ gateway container        │               │
│  │  └─▶ langgraph container      │               │
│  └───────────────────────────────┘               │
│                                                  │
│  Volumes:                                        │
│  ├── config.yaml (read-only mount)               │
│  ├── extensions_config.json (read-only mount)    │
│  ├── .deer-flow/ (read-write, persistent data)   │
│  ├── skills/ (read-only mount)                   │
│  └── docker.sock (for DooD sandbox)              │
│                                                  │
│  Optional:                                       │
│  └── provisioner container (Kubernetes mode)     │
└─────────────────────────────────────────────────┘
```

**When to use**: Staging, demo environments, single-server production.

**Requirements**: Docker + Docker Compose only (no local Python/Node needed).

**Key environment variables** (set in `.env` or by `deploy.sh`):

| Variable | Purpose | Default |
|----------|---------|---------|
| `DEER_FLOW_HOME` | Runtime data directory | `backend/.deer-flow` |
| `DEER_FLOW_CONFIG_PATH` | Path to config.yaml | `config.yaml` |
| `DEER_FLOW_EXTENSIONS_CONFIG_PATH` | Path to extensions config | `extensions_config.json` |
| `DEER_FLOW_DOCKER_SOCKET` | Docker socket for DooD sandbox | `/var/run/docker.sock` |
| `BETTER_AUTH_SECRET` | Frontend session encryption | Auto-generated |
| `PORT` | External port | `2026` |
| `LANGCHAIN_TRACING_V2` | Enable LangSmith tracing | `false` |
| `LANGSMITH_API_KEY` | LangSmith API key | (unset) |

**Commands**:
```bash
make config    # Generate config files
# Edit config.yaml, set API keys in .env
make up        # Build images + start containers
# Open http://localhost:2026
make down      # Stop and remove containers
```

**Docker-on-Docker (DooD)**: When using the AIO sandbox provider in Docker, containers need access to the host's Docker daemon. The `docker.sock` volume mount enables this — sandbox containers are siblings, not nested.

### Option 3: Docker Development (`make docker-start`)

A hybrid that runs services in Docker but with development features:

```bash
make docker-init   # Pull images, install deps in containers
make docker-start  # Start with hot-reload in containers
make docker-logs   # Watch logs
make docker-stop   # Tear down
```

**When to use**: When you want Docker isolation but still need hot-reload for development.

### Sandbox Modes in Deployment

The deployment script auto-detects the sandbox mode from `config.yaml`:

| Mode | Config | What Happens |
|------|--------|-------------|
| **local** | `use: deerflow.sandbox.local:LocalSandboxProvider` | Code runs on the host filesystem (or inside the gateway container in Docker). No extra containers. |
| **aio** | `use: deerflow.community.aio_sandbox:AioSandboxProvider` | Each sandbox gets a dedicated Docker container. Requires Docker socket access. |
| **provisioner** | `aio` + `provisioner_url` set | Sandboxes are Kubernetes pods managed by the provisioner service. Adds the `provisioner` container to the Docker Compose stack. |

---

## Development Lifecycle Summary

```
1. SETUP (one-time)
   make check → make install → make config → edit config.yaml

2. DEVELOP (daily)
   make dev → edit code → services hot-reload → test in browser
   cd backend && make test → run tests
   cd backend && make lint → check style

3. TEST (before PR)
   cd backend && make test
   cd frontend && pnpm check

4. DEPLOY (release)
   Option A: make up              (Docker production)
   Option B: make start           (bare-metal production)
   Option C: make docker-start    (Docker development)

5. MAINTAIN
   make config-upgrade            (merge new config fields)
   make stop / make down          (shut down)
   make clean                     (reset runtime data)
```
