# Architecture Diagrams

Comprehensive Mermaid diagrams covering every architectural aspect of DeerFlow.

---

## 1. High-Level Service Architecture

```mermaid
graph TB
    Browser["Browser / API Client"]

    subgraph NGINX["Nginx Reverse Proxy :2026"]
        direction LR
        Router{Route}
    end

    subgraph LG["LangGraph Server :2024"]
        AgentRuntime["Agent Runtime"]
        StateManager["State Manager<br/>(Checkpointer)"]
        SSEStream["SSE Stream"]
    end

    subgraph GW["Gateway API :8001"]
        FastAPI["FastAPI"]
        ModelsRouter["/api/models"]
        MCPRouter["/api/mcp"]
        SkillsRouter["/api/skills"]
        MemoryRouter["/api/memory"]
        UploadsRouter["/api/uploads"]
        ArtifactsRouter["/api/artifacts"]
        ThreadsRouter["/api/threads"]
        AgentsRouter["/api/agents"]
        SuggestionsRouter["/api/suggestions"]
        ChannelsRouter["/api/channels"]
    end

    subgraph FE["Frontend :3000"]
        NextJS["Next.js 16 App Router"]
        ReactUI["React 19 UI"]
        LangGraphSDK["LangGraph SDK Client"]
    end

    Browser --> NGINX
    Router -->|"/api/langgraph/*"| LG
    Router -->|"/api/*"| GW
    Router -->|"/"| FE

    LangGraphSDK -.->|"SSE streaming"| LG
    ReactUI -.->|"REST calls"| GW

    style NGINX fill:#4a90d9,color:#fff
    style LG fill:#e74c3c,color:#fff
    style GW fill:#27ae60,color:#fff
    style FE fill:#8e44ad,color:#fff
```

---

## 2. Complete Request Flow

```mermaid
sequenceDiagram
    actor User
    participant FE as Frontend<br/>(Next.js)
    participant NX as Nginx<br/>(:2026)
    participant LG as LangGraph<br/>(:2024)
    participant Agent as Lead Agent
    participant MW as Middleware Chain
    participant LLM as LLM Provider
    participant Tools as Tool Executor
    participant GW as Gateway API<br/>(:8001)

    User->>FE: Type message
    FE->>NX: POST /api/langgraph/threads/{id}/runs (stream)
    NX->>LG: Forward to :2024

    LG->>LG: Load thread state (checkpointer)
    LG->>Agent: make_lead_agent(config)

    Note over Agent: 1. Resolve model<br/>2. Assemble tools<br/>3. Generate prompt<br/>4. Build middlewares

    loop Agent Execution Loop
        Agent->>MW: before_model()
        MW->>LLM: Invoke with messages + tools
        LLM-->>MW: Response (text + tool_calls)
        MW->>MW: after_model()

        alt Tool calls present
            MW->>Tools: wrap_tool_call() per tool
            Tools-->>MW: Tool results
        else No tool calls
            Note over MW: Loop ends
        end
    end

    Agent-->>LG: Final ThreadState

    loop SSE Events
        LG-->>NX: event: messages-tuple
        NX-->>FE: Forward (unbuffered)
        FE-->>User: Render incrementally
    end

    LG-->>NX: event: values (title, artifacts)
    NX-->>FE: Forward
    LG-->>NX: event: end
    NX-->>FE: Forward

    Note over Agent: Background (30s debounce)
    Agent-)GW: MemoryUpdateQueue
    GW-)GW: LLM extracts facts
    GW-)GW: Atomic write memory.json
```

---

## 3. Lead Agent Creation Pipeline

```mermaid
flowchart TD
    Config["RunnableConfig<br/>(configurable dict)"]

    Config --> Extract["Extract Parameters"]

    Extract --> P1["thread_id"]
    Extract --> P2["model_name"]
    Extract --> P3["thinking_enabled"]
    Extract --> P4["is_plan_mode"]
    Extract --> P5["subagent_enabled"]
    Extract --> P6["agent_name"]
    Extract --> P7["is_bootstrap"]
    Extract --> P8["reasoning_effort"]

    P6 --> AgentConfig{"agent_name<br/>provided?"}
    AgentConfig -->|Yes| LoadAgent["load_agent_config(name)"]
    AgentConfig -->|No| DefaultModel

    LoadAgent --> AgentModel["Agent's custom model"]
    AgentModel --> ModelResolve

    P2 --> ModelResolve["Model Resolution"]
    DefaultModel["Global default model<br/>(first in config.yaml)"] --> ModelResolve

    ModelResolve --> ValidateThinking{"supports_thinking?"}
    ValidateThinking -->|No| DisableThinking["Fallback: thinking=false"]
    ValidateThinking -->|Yes| CreateModel

    DisableThinking --> CreateModel["create_chat_model()<br/>(name, thinking, reasoning_effort)"]

    CreateModel --> AssembleTools["get_available_tools()"]
    AssembleTools --> BuildPrompt["apply_prompt_template()<br/>(memory + skills + subagent instructions)"]
    BuildPrompt --> BuildMW["_build_middlewares()"]
    BuildMW --> CreateAgent["create_agent()<br/>(model, tools, middleware,<br/>prompt, ThreadState)"]

    CreateAgent --> Agent["Compiled Agent"]

    style Config fill:#3498db,color:#fff
    style Agent fill:#27ae60,color:#fff
    style CreateModel fill:#e67e22,color:#fff
    style AssembleTools fill:#9b59b6,color:#fff
    style BuildMW fill:#e74c3c,color:#fff
```

---

## 4. Middleware Chain

```mermaid
flowchart TD
    Request["Incoming Request"] --> M1

    subgraph ALWAYS["Always Active"]
        M1["1. ThreadDataMiddleware<br/>Creates per-thread dirs"]
        M2["2. UploadsMiddleware<br/>Injects file metadata"]
        M3["3. SandboxMiddleware<br/>Acquires sandbox"]
        M4["4. DanglingToolCallMiddleware<br/>Patches missing ToolMessages"]
    end

    subgraph CONDITIONAL["Conditionally Enabled"]
        M5["5. GuardrailMiddleware<br/>Pre-tool-call auth<br/><i>if guardrails.enabled</i>"]
        M6["6. ToolErrorHandlingMiddleware<br/>Exceptions → ToolMessages"]
        M7["7. SummarizationMiddleware<br/>Context reduction<br/><i>if summarization.enabled</i>"]
        M8["8. TodoListMiddleware<br/>Plan mode tasks<br/><i>if is_plan_mode</i>"]
        M9["9. TokenUsageMiddleware<br/>Usage tracking<br/><i>if token_usage.enabled</i>"]
    end

    subgraph POSTPROCESS["Post-Processing"]
        M10["10. TitleMiddleware<br/>Auto-generate title"]
        M11["11. MemoryMiddleware<br/>Queue for memory update"]
        M12["12. ViewImageMiddleware<br/>Base64 injection<br/><i>if supports_vision</i>"]
        M13["13. DeferredToolFilterMiddleware<br/>Hide deferred MCP tools<br/><i>if tool_search.enabled</i>"]
        M14["14. SubagentLimitMiddleware<br/>Truncate excess tasks<br/><i>if subagent_enabled</i>"]
        M15["15. LoopDetectionMiddleware<br/>Break repetitive loops"]
    end

    subgraph FINAL["Must Be Last"]
        M16["16. ClarificationMiddleware<br/>Intercepts ask_clarification<br/>→ Command(goto=END)"]
    end

    M1 --> M2 --> M3 --> M4
    M4 --> M5 --> M6 --> M7 --> M8 --> M9
    M9 --> M10 --> M11 --> M12 --> M13 --> M14 --> M15
    M15 --> M16

    M16 --> LLM["LLM Execution Loop"]

    style ALWAYS fill:#2ecc71,color:#fff
    style CONDITIONAL fill:#f39c12,color:#000
    style POSTPROCESS fill:#3498db,color:#fff
    style FINAL fill:#e74c3c,color:#fff
```

---

## 5. Tool Assembly Pipeline

```mermaid
flowchart LR
    subgraph CONFIG["Config Tools<br/>(config.yaml)"]
        T1["web_search<br/>(Tavily)"]
        T2["web_fetch<br/>(Jina AI)"]
        T3["image_search<br/>(DuckDuckGo)"]
        T4["ls"]
        T5["read_file"]
        T6["write_file"]
        T7["str_replace"]
        T8["bash"]
    end

    subgraph MCP["MCP Tools<br/>(extensions_config.json)"]
        M1["GitHub tools"]
        M2["Database tools"]
        M3["Custom servers"]
        MCPCache["Lazy loaded<br/>mtime cached"]
    end

    subgraph BUILTIN["Built-in Tools<br/>(always present)"]
        B1["present_file"]
        B2["ask_clarification"]
    end

    subgraph OPTIONAL["Conditional Tools"]
        O1["view_image<br/><i>if supports_vision</i>"]
        O2["task<br/><i>if subagent_enabled</i>"]
        O3["tool_search<br/><i>if deferred loading</i>"]
    end

    CONFIG --> GAT
    MCP --> GAT
    BUILTIN --> GAT
    OPTIONAL --> GAT

    GAT["get_available_tools()"] --> Agent["Agent Tool Belt"]

    style CONFIG fill:#27ae60,color:#fff
    style MCP fill:#8e44ad,color:#fff
    style BUILTIN fill:#2980b9,color:#fff
    style OPTIONAL fill:#e67e22,color:#fff
    style GAT fill:#c0392b,color:#fff
    style Agent fill:#2c3e50,color:#fff
```

---

## 6. Memory System

```mermaid
flowchart TD
    Conv["Conversation<br/>(user + AI messages)"]

    Conv --> MMW["MemoryMiddleware<br/>(in middleware chain)"]

    MMW -->|"Filters: user msgs +<br/>final AI responses"| Queue["MemoryUpdateQueue"]

    Queue -->|"Per-thread dedup<br/>newer replaces older"| Timer{"Debounce<br/>30s elapsed?"}

    Timer -->|"No, reset timer"| Queue
    Timer -->|"Yes"| Updater["MemoryUpdater<br/>(LLM-based)"]

    Updater --> Extract["Extract from conversation"]

    Extract --> Facts["Discrete Facts<br/>id, content, category,<br/>confidence, source"]
    Extract --> UserCtx["User Context<br/>work, personal, top-of-mind"]
    Extract --> History["History<br/>recent, earlier, long-term"]

    Facts --> Dedup{"Whitespace-normalized<br/>duplicate?"}
    Dedup -->|"Yes"| Skip["Skip"]
    Dedup -->|"No"| Write

    UserCtx --> Write["Atomic Write"]
    History --> Write

    Write --> Temp["Write temp file"]
    Temp --> Rename["os.replace()<br/>(atomic rename)"]
    Rename --> File["memory.json"]
    Rename --> Invalidate["Cache invalidated"]

    File --> NextReq["Next Request"]
    NextReq --> Inject["Inject into system prompt<br/>Top 15 facts<br/>Max 2000 tokens"]

    subgraph PerAgent["Per-Agent Memory"]
        Default["Default:<br/>.deer-flow/memory.json"]
        Custom["Custom agent:<br/>.deer-flow/agents/{name}/memory.json"]
    end

    File --- PerAgent

    style MMW fill:#9b59b6,color:#fff
    style Queue fill:#e67e22,color:#fff
    style Updater fill:#e74c3c,color:#fff
    style File fill:#27ae60,color:#fff
    style Inject fill:#3498db,color:#fff
```

---

## 7. Subagent Execution Model

```mermaid
flowchart TD
    Lead["Lead Agent"]

    Lead -->|"task(desc, type)"| Task1["Task 1:<br/>general-purpose"]
    Lead -->|"task(desc, type)"| Task2["Task 2:<br/>bash"]
    Lead -->|"task(desc, type)"| Task3["Task 3:<br/>general-purpose"]
    Lead -->|"task(desc, type)"| Task4["Task 4<br/>(TRUNCATED by<br/>SubagentLimitMiddleware)"]

    subgraph Executor["SubagentExecutor"]
        subgraph Sched["Scheduler Pool<br/>(3 workers)"]
            S1["Schedule"]
            S2["Monitor"]
            S3["Emit SSE"]
        end

        subgraph Exec["Execution Pool<br/>(3 workers)"]
            E1["Worker 1"]
            E2["Worker 2"]
            E3["Worker 3"]
        end

        Sched --> Exec
    end

    Task1 --> Executor
    Task2 --> Executor
    Task3 --> Executor
    Task4 -.->|"Silently dropped"| X["Not executed"]

    E1 --> R1["Result 1"]
    E2 --> R2["Result 2"]
    E3 --> R3["Result 3"]

    R1 --> Lead
    R2 --> Lead
    R3 --> Lead

    subgraph Events["SSE Events"]
        EV1["task_started"]
        EV2["task_running"]
        EV3["task_completed"]
    end

    Executor --> Events

    subgraph Limits["Concurrency Controls"]
        L1["Hard limit: 3 concurrent"]
        L2["Timeout: 15min/agent"]
        L3["Batching: >3 across turns"]
    end

    style Lead fill:#2c3e50,color:#fff
    style Executor fill:#8e44ad,color:#fff
    style Task4 fill:#95a5a6,color:#fff
    style X fill:#95a5a6,color:#fff
    style Limits fill:#e74c3c,color:#fff
```

---

## 8. Sandbox & Virtual Path System

```mermaid
flowchart LR
    subgraph AgentView["Agent's View (Virtual Paths)"]
        V1["/mnt/user-data/workspace"]
        V2["/mnt/user-data/uploads"]
        V3["/mnt/user-data/outputs"]
        V4["/mnt/skills"]
    end

    subgraph Translation["Path Translation Layer"]
        PT["replace_virtual_path()<br/>Bidirectional mapping<br/>Host paths masked in output"]
    end

    subgraph HostView["Host Reality (Physical Paths)"]
        H1["backend/.deer-flow/<br/>threads/{id}/user-data/workspace"]
        H2["backend/.deer-flow/<br/>threads/{id}/user-data/uploads"]
        H3["backend/.deer-flow/<br/>threads/{id}/user-data/outputs"]
        H4["deer-flow/skills/"]
    end

    V1 <-->|translate| PT
    V2 <-->|translate| PT
    V3 <-->|translate| PT
    V4 <-->|translate| PT

    PT <-->|translate| H1
    PT <-->|translate| H2
    PT <-->|translate| H3
    PT <-->|translate| H4

    subgraph Tools["Sandbox Tools"]
        BT["bash<br/>(execute commands)"]
        LS["ls<br/>(tree listing, 2 levels)"]
        RF["read_file<br/>(optional line range)"]
        WF["write_file<br/>(create/append)"]
        SR["str_replace<br/>(substring replace)"]
    end

    subgraph Providers["Sandbox Providers"]
        LP["LocalSandboxProvider<br/>Host filesystem<br/>(development)"]
        AP["AioSandboxProvider<br/>Docker container<br/>(production)"]
    end

    Tools --> PT
    Providers --> PT

    subgraph Security["Path Security"]
        S1["Block ../"]
        S2["Block ..\\"]
        S3["Block absolute<br/>outside roots"]
        S4["Mask host paths<br/>in output"]
    end

    style AgentView fill:#3498db,color:#fff
    style HostView fill:#27ae60,color:#fff
    style Translation fill:#e67e22,color:#fff
    style Security fill:#e74c3c,color:#fff
```

---

## 9. MCP Integration

```mermaid
flowchart TD
    ExtConfig["extensions_config.json"]

    ExtConfig --> Servers["MCP Server Definitions"]

    Servers --> S1["stdio Transport<br/>command + args + env<br/>(npx, uvx)"]
    Servers --> S2["SSE Transport<br/>url + headers<br/>(remote streaming)"]
    Servers --> S3["HTTP Transport<br/>url + headers<br/>(REST-based)"]

    S2 --> OAuth{"OAuth<br/>configured?"}
    S3 --> OAuth

    OAuth -->|Yes| TokenFlow["Token Flow"]
    OAuth -->|No| Direct["Direct connection"]

    TokenFlow --> CC["client_credentials"]
    TokenFlow --> RT["refresh_token"]
    CC --> AutoRefresh["Auto token refresh<br/>Authorization header injection"]
    RT --> AutoRefresh

    S1 --> Cache
    Direct --> Cache
    AutoRefresh --> Cache

    Cache["Tool Cache<br/>(mtime-based invalidation)"]

    Cache --> Check{"extensions_config.json<br/>mtime changed?"}
    Check -->|"Yes"| Reload["Reload tools"]
    Check -->|"No"| Return["Return cached tools"]

    Reload --> Return

    Return --> Deferred{"tool_search<br/>enabled?"}
    Deferred -->|"Yes"| TS["tool_search_tool<br/>(deferred loading)"]
    Deferred -->|"No"| Bind["Bind all to model"]

    TS --> Model["Agent Model"]
    Bind --> Model

    subgraph Runtime["Runtime Modification"]
        GWAPI["Gateway API<br/>PUT /api/mcp/config"]
        GWAPI --> WriteCfg["Write extensions_config.json"]
        WriteCfg --> MtimeChange["mtime updated"]
        MtimeChange --> Check
    end

    style ExtConfig fill:#9b59b6,color:#fff
    style Cache fill:#e67e22,color:#fff
    style OAuth fill:#e74c3c,color:#fff
    style Runtime fill:#27ae60,color:#fff
```

---

## 10. Configuration System

```mermaid
flowchart TD
    subgraph Resolution["Config File Resolution (Priority Order)"]
        P1["1. Explicit config_path argument"]
        P2["2. DEER_FLOW_CONFIG_PATH env var"]
        P3["3. config.yaml in current dir"]
        P4["4. config.yaml in parent dir (recommended)"]
        P1 --> P2 --> P3 --> P4
    end

    Resolution --> Parse["Parse YAML"]

    Parse --> EnvResolve["Environment Variable Resolution<br/>$VAR_NAME → os.environ[VAR_NAME]"]

    EnvResolve --> ConfigObj["AppConfig Object"]

    subgraph Sections["Config Sections"]
        S1["models[] — LLM configurations"]
        S2["tools[] — Tool definitions + groups"]
        S3["sandbox — Provider selection"]
        S4["memory — Persistence settings"]
        S5["summarization — Context compression"]
        S6["checkpointer — State persistence"]
        S7["guardrails — Authorization"]
        S8["channels — IM integrations"]
        S9["title — Auto-generation"]
        S10["token_usage — Tracking"]
    end

    ConfigObj --> Sections

    subgraph ExtConfig["extensions_config.json"]
        EC1["mcpServers — MCP server configs"]
        EC2["skills — Skill enable/disable"]
    end

    subgraph AutoReload["Auto-Reload Mechanism"]
        AR1["get_app_config() called"]
        AR2{"mtime changed?"}
        AR3["Return cached"]
        AR4["Reload + cache"]
        AR1 --> AR2
        AR2 -->|"No"| AR3
        AR2 -->|"Yes"| AR4
    end

    subgraph Versioning["Config Versioning"]
        V1["config_version: 3"]
        V2["make config-upgrade"]
        V3["Merge new fields from<br/>config.example.yaml"]
        V1 --> V2 --> V3
    end

    style Resolution fill:#3498db,color:#fff
    style EnvResolve fill:#e67e22,color:#fff
    style AutoReload fill:#27ae60,color:#fff
    style Versioning fill:#9b59b6,color:#fff
```

---

## 11. Harness / App Boundary

```mermaid
flowchart TD
    subgraph APP["App Layer (app/*)"]
        direction TB
        GW["FastAPI Gateway<br/>12 route modules"]
        CH["IM Channels<br/>Feishu, Slack, Telegram"]
        GWConfig["Gateway Config"]
    end

    subgraph HARNESS["Harness Layer (deerflow/*)"]
        direction TB
        Agents["agents/<br/>Lead agent, middlewares,<br/>memory, thread state"]
        Tools2["tools/<br/>builtins, assembly"]
        Sandbox2["sandbox/<br/>providers, tools, paths"]
        Models["models/<br/>factory, thinking, vision"]
        MCP2["mcp/<br/>client, cache, transports"]
        Skills2["skills/<br/>loader, validation"]
        Config2["config/<br/>app, extensions, paths"]
        Community["community/<br/>tavily, jina, firecrawl"]
        Guard["guardrails/<br/>allowlist, OAP, custom"]
        Reflect["reflection/<br/>resolve_variable, resolve_class"]
        Client["client.py<br/>DeerFlowClient"]
        SubAgents["subagents/<br/>executor, registry"]
    end

    APP -->|"imports deerflow.*<br/>ALLOWED"| HARNESS
    HARNESS -.-x|"imports app.*<br/>FORBIDDEN<br/>(CI enforced)"| APP

    subgraph CI["CI Enforcement"]
        Test["test_harness_boundary.py<br/>Scans all .py in deerflow/<br/>Fails if 'from app' found"]
    end

    HARNESS -.-> CI

    subgraph WHY["Why This Matters"]
        PIP["pip install deerflow-harness<br/>(standalone, no Gateway needed)"]
        DIP["Dependency Inversion Principle<br/>Stable core ← volatile app"]
    end

    style APP fill:#e74c3c,color:#fff
    style HARNESS fill:#27ae60,color:#fff
    style CI fill:#f39c12,color:#000
    style WHY fill:#3498db,color:#fff
```

---

## 12. Skills Lifecycle

```mermaid
flowchart TD
    subgraph Discovery["1. Discovery"]
        Scan["load_skills() scans<br/>skills/{public,custom}/"]
        Scan --> FindSKILL["Find SKILL.md files<br/>(recursive)"]
    end

    subgraph Parsing["2. Parsing"]
        FindSKILL --> ParseFM["Parse YAML frontmatter"]
        ParseFM --> Meta["Metadata:<br/>name, description,<br/>license, allowed-tools"]
    end

    subgraph Enable["3. Enable/Disable"]
        Meta --> CheckExt["Check extensions_config.json"]
        CheckExt --> Enabled{"enabled?"}
        Enabled -->|Yes| Active["Active Skill"]
        Enabled -->|No| Inactive["Inactive Skill"]
    end

    subgraph Injection["4. Prompt Injection"]
        Active --> SysPrompt["System Prompt:<br/>&lt;skills_section&gt;<br/>- skill-name (/mnt/skills/...)"]
    end

    subgraph Install["5. Installation"]
        Upload["POST /api/skills/install<br/>(.skill ZIP archive)"]
        Upload --> Validate["Validate"]
        Validate --> SizeCheck["Size < 512MB"]
        Validate --> PathCheck["No ../ or absolute paths"]
        Validate --> SymlinkCheck["Remove symlinks"]
        Validate --> FMCheck["Valid SKILL.md frontmatter"]
        SizeCheck --> Extract
        PathCheck --> Extract
        SymlinkCheck --> Extract
        FMCheck --> Extract["Extract to<br/>skills/custom/{name}/"]
    end

    subgraph Manage["6. Runtime Management"]
        GWAPI["Gateway API"]
        GWAPI --> List["GET /api/skills"]
        GWAPI --> Toggle["PUT /api/skills/{name}"]
        GWAPI --> InstallAPI["POST /api/skills/install"]
        Toggle --> WriteExt["Update<br/>extensions_config.json"]
    end

    style Discovery fill:#3498db,color:#fff
    style Parsing fill:#9b59b6,color:#fff
    style Enable fill:#e67e22,color:#fff
    style Injection fill:#27ae60,color:#fff
    style Install fill:#e74c3c,color:#fff
```

---

## 13. ThreadState Schema

```mermaid
classDiagram
    class AgentState {
        +list~BaseMessage~ messages
    }

    class ThreadState {
        +SandboxState sandbox
        +ThreadDataState thread_data
        +str title
        +list~str~ artifacts
        +list todos
        +list~dict~ uploaded_files
        +dict~str,ViewedImageData~ viewed_images
    }

    class SandboxState {
        +str sandbox_id
    }

    class ThreadDataState {
        +str workspace_path
        +str uploads_path
        +str outputs_path
    }

    class ViewedImageData {
        +str base64
        +str mime_type
    }

    AgentState <|-- ThreadState
    ThreadState *-- SandboxState
    ThreadState *-- ThreadDataState
    ThreadState *-- ViewedImageData

    note for ThreadState "artifacts: Annotated[list, merge_artifacts]\n  → Deduplicates via dict.fromkeys\n\nviewed_images: Annotated[dict, merge_viewed_images]\n  → Merge dicts; empty {} clears all"
```

---

## 14. Package Management & Workspace Structure

```mermaid
flowchart TD
    subgraph Root["Project Root"]
        Makefile["Makefile<br/>(unified task runner)"]
        RootEnv[".env<br/>(API keys, secrets)"]
        ConfigYAML["config.yaml"]
        ExtJSON["extensions_config.json"]
    end

    subgraph Backend["backend/"]
        BPyproject["pyproject.toml<br/>name: deer-flow (app)"]
        BLock["uv.lock"]
        BMakefile["Makefile<br/>(backend tasks)"]
        BVenv[".venv/<br/>(managed by uv)"]

        subgraph Workspace["uv Workspace"]
            App["Root Package<br/>deer-flow<br/>FastAPI, IM SDKs,<br/>langgraph-sdk"]
            Harness["Workspace Member<br/>packages/harness/<br/>deerflow-harness<br/>LangChain, LangGraph,<br/>all agent deps"]
        end

        App -->|"depends on<br/>workspace = true"| Harness
    end

    subgraph Frontend2["frontend/"]
        FPkg["package.json<br/>name: deer-flow-frontend"]
        FLock["pnpm-lock.yaml"]
        FNode["node_modules/"]
    end

    Makefile -->|"make install"| BPyproject
    Makefile -->|"make install"| FPkg

    BPyproject -->|"uv sync"| BVenv
    FPkg -->|"pnpm install"| FNode

    subgraph Build["Build & Publish"]
        Hatch["hatchling<br/>(build backend)"]
        Wheel["deerflow-harness.whl"]
        Harness --> Hatch --> Wheel
        Wheel --> PyPI["pip install<br/>deerflow-harness"]
    end

    style Root fill:#f39c12,color:#000
    style Backend fill:#3498db,color:#fff
    style Frontend2 fill:#8e44ad,color:#fff
    style Workspace fill:#27ae60,color:#fff
    style Build fill:#e74c3c,color:#fff
```

---

## 15. Deployment Options

```mermaid
flowchart TD
    subgraph Local["Option 1: Local Dev (make dev)"]
        L_NX["Nginx :2026<br/>(native process)"]
        L_LG["LangGraph :2024<br/>(uv run langgraph dev)"]
        L_GW["Gateway :8001<br/>(uvicorn)"]
        L_FE["Frontend :3000<br/>(pnpm dev --turbo)"]
        L_NX --> L_LG
        L_NX --> L_GW
        L_NX --> L_FE
        L_Logs["logs/ directory"]
        L_Hot["Hot-reload: ALL"]
    end

    subgraph Docker["Option 2: Docker Prod (make up)"]
        D_NX["nginx container<br/>:2026 exposed"]
        D_LG["langgraph container<br/>--no-reload"]
        D_GW["gateway container<br/>--workers 2"]
        D_FE["frontend container<br/>next start (prod)"]
        D_NX --> D_LG
        D_NX --> D_GW
        D_NX --> D_FE

        subgraph Volumes["Mounted Volumes"]
            V1["config.yaml (RO)"]
            V2["extensions_config.json (RO)"]
            V3[".deer-flow/ (RW)"]
            V4["skills/ (RO)"]
            V5["docker.sock (DooD)"]
        end

        D_Optional["Optional:<br/>provisioner container<br/>(Kubernetes sandbox)"]
    end

    subgraph DockerDev["Option 3: Docker Dev (make docker-start)"]
        DD["Docker containers<br/>WITH hot-reload"]
        DD_Init["make docker-init"]
        DD_Start["make docker-start"]
        DD_Init --> DD_Start --> DD
    end

    subgraph SandboxModes["Sandbox Mode Detection"]
        SM_Local["local<br/>Host filesystem"]
        SM_AIO["aio<br/>Docker per sandbox"]
        SM_K8s["provisioner<br/>Kubernetes pods"]
    end

    Docker -.-> SandboxModes

    style Local fill:#27ae60,color:#fff
    style Docker fill:#3498db,color:#fff
    style DockerDev fill:#9b59b6,color:#fff
    style SandboxModes fill:#e74c3c,color:#fff
```

---

## 16. Security Layers

```mermaid
flowchart TD
    UserInput["User Input"]

    UserInput --> L1["Layer 1: Nginx<br/>CORS, Timeouts, TLS termination"]

    L1 --> L2["Layer 2: API Gateway<br/>Path validation, filename normalization"]

    L2 --> L3["Layer 3: Guardrails Middleware<br/>Pre-tool-call authorization<br/>Fail-closed by default"]

    subgraph Providers["Guardrail Providers"]
        GP1["AllowlistProvider<br/>(zero deps, block/allow)"]
        GP2["OAP Provider<br/>(Open Agent Passport)"]
        GP3["Custom Provider<br/>(implement evaluate())"]
    end

    L3 --> Providers

    L3 --> L4["Layer 4: Sandbox Isolation"]

    subgraph SandboxSec["Sandbox Security"]
        SS1["Virtual path abstraction"]
        SS2["Path traversal blocking<br/>(../ and ..\\)"]
        SS3["Host path masking in output"]
        SS4["Per-thread directory isolation"]
    end

    L4 --> SandboxSec

    L4 --> L5["Layer 5: File Upload Security"]

    subgraph UploadSec["Upload Protection"]
        US1["Filename normalization"]
        US2["Directory separator rejection"]
        US3["ZIP bomb defense (512MB)"]
        US4["Symlink removal"]
        US5["Absolute path rejection"]
    end

    L5 --> UploadSec

    L5 --> L6["Layer 6: Memory Security"]

    subgraph MemSec["Memory Protection"]
        MS1["Atomic I/O (temp + rename)"]
        MS2["Confidence threshold filter"]
        MS3["Prompt injection test coverage"]
    end

    L6 --> MemSec

    L6 --> L7["Layer 7: API Key Management"]

    subgraph KeySec["Secret Handling"]
        KS1["$VAR_NAME env injection"]
        KS2[".env gitignored"]
        KS3["No secrets in config files"]
    end

    L7 --> KeySec

    L7 --> L8["Layer 8: Architectural Boundary"]

    subgraph BoundarySec["Boundary Enforcement"]
        BS1["Harness never imports App"]
        BS2["CI test enforcement"]
        BS3["Publishable harness isolation"]
    end

    L8 --> BoundarySec

    subgraph Gaps["Known Gaps"]
        G1["No authentication"]
        G2["No rate limiting"]
        G3["CORS allows all origins"]
        G4["Memory facts unencrypted"]
        G5["No session management"]
    end

    style L1 fill:#3498db,color:#fff
    style L2 fill:#27ae60,color:#fff
    style L3 fill:#e74c3c,color:#fff
    style L4 fill:#e67e22,color:#fff
    style L5 fill:#9b59b6,color:#fff
    style L6 fill:#1abc9c,color:#fff
    style L7 fill:#34495e,color:#fff
    style L8 fill:#2c3e50,color:#fff
    style Gaps fill:#c0392b,color:#fff
```

---

## 17. Frontend Architecture

```mermaid
flowchart TD
    subgraph Pages["App Router Pages"]
        Landing["/ — Landing Page"]
        Chat["/workspace/chats/[thread_id]<br/>— Chat Interface"]
    end

    subgraph Components["Component Layer"]
        WS["workspace/<br/>Chat messages, artifacts,<br/>settings panels"]
        LN["landing/<br/>Homepage sections"]
        UI["ui/<br/>Shadcn primitives<br/>(auto-generated)"]
    end

    subgraph Core["Core Business Logic"]
        Threads["threads/<br/>Management, streaming,<br/>state hooks"]
        Artifacts["artifacts/<br/>Loading, caching"]
        Memory2["memory/<br/>Display, management"]
        Skills3["skills/<br/>Install, toggle"]
        MCP3["mcp/<br/>Server configuration"]
        I18n["i18n/<br/>en-US, zh-CN"]
        Models2["models/<br/>TypeScript types"]
        Settings["settings/<br/>localStorage prefs"]
    end

    subgraph DataFlow["Data Flow"]
        LGSDK["@langchain/langgraph-sdk<br/>Thread + Run management"]
        RQ["@tanstack/react-query<br/>Server state caching"]
        Stream["SSE Streaming<br/>client.runs.stream()"]
    end

    Pages --> Components
    Components --> Core
    Core --> DataFlow

    LGSDK -->|"/api/langgraph/*"| NX["Nginx :2026"]
    RQ -->|"/api/*"| NX

    subgraph Rendering["Rich Rendering"]
        MD["Markdown: remark-gfm"]
        Math["Math: rehype-katex"]
        Code["Code: shiki + CodeMirror"]
        Anim["Animation: gsap + motion"]
    end

    Components --> Rendering

    style Pages fill:#8e44ad,color:#fff
    style Components fill:#3498db,color:#fff
    style Core fill:#27ae60,color:#fff
    style DataFlow fill:#e67e22,color:#fff
    style Rendering fill:#e74c3c,color:#fff
```

---

## 18. Checkpointer System

```mermaid
flowchart LR
    subgraph Types["Checkpointer Types"]
        Mem["InMemorySaver<br/>Ephemeral, per-process<br/>Lost on restart"]
        SQL["AsyncSqliteSaver<br/>File-based (store.db)<br/>Survives restarts"]
        PG["AsyncPostgresSaver<br/>Database-backed<br/>Multi-process ready"]
    end

    subgraph Usage["Who Uses What"]
        LGServer["LangGraph Server<br/>Manages own checkpointer<br/>(config.yaml ignored)"]
        EmbClient["DeerFlowClient<br/>Uses config.yaml checkpointer<br/>(or explicit argument)"]
    end

    Mem --> EmbClient
    SQL --> EmbClient
    PG --> EmbClient

    subgraph Config3["config.yaml"]
        CK["checkpointer:<br/>  type: sqlite<br/>  connection_string: checkpoints.db"]
    end

    Config3 --> EmbClient

    subgraph State["Persisted State"]
        TS["ThreadState<br/>messages, title, artifacts,<br/>sandbox, todos, images"]
    end

    Types --> State
    State -->|"Load on<br/>next request"| Agent2["Agent Execution"]
    Agent2 -->|"Save after<br/>each turn"| State

    style Mem fill:#f39c12,color:#000
    style SQL fill:#27ae60,color:#fff
    style PG fill:#3498db,color:#fff
    style State fill:#9b59b6,color:#fff
```

---

## 19. Development Lifecycle

```mermaid
flowchart LR
    subgraph Setup["1. SETUP (one-time)"]
        S1["make check"] --> S2["make install"] --> S3["make config"] --> S4["Edit config.yaml<br/>+ .env API keys"]
    end

    subgraph Dev["2. DEVELOP (daily)"]
        D1["make dev"] --> D2["Edit code"] --> D3["Hot-reload"] --> D4["Test in browser"]
        D4 --> D2
        D5["cd backend<br/>make test"]
        D6["cd backend<br/>make lint"]
    end

    subgraph Test["3. TEST (before PR)"]
        T1["cd backend<br/>make test"]
        T2["cd frontend<br/>pnpm check"]
        T1 --> T2
    end

    subgraph Deploy["4. DEPLOY"]
        DP1["make up<br/>(Docker prod)"]
        DP2["make start<br/>(bare-metal prod)"]
        DP3["make docker-start<br/>(Docker dev)"]
    end

    subgraph Maintain["5. MAINTAIN"]
        M1["make config-upgrade"]
        M2["make stop / down"]
        M3["make clean"]
    end

    Setup --> Dev --> Test --> Deploy --> Maintain

    style Setup fill:#3498db,color:#fff
    style Dev fill:#27ae60,color:#fff
    style Test fill:#f39c12,color:#000
    style Deploy fill:#e74c3c,color:#fff
    style Maintain fill:#9b59b6,color:#fff
```

---

## How to Render These Diagrams

**VS Code**: Install the "Markdown Preview Mermaid Support" extension, then preview this file with `Cmd+Shift+V`.

**GitHub**: Mermaid diagrams render natively in GitHub Markdown — just push this file.

**CLI**: Use `mmdc` (Mermaid CLI) to export as PNG/SVG:
```bash
npm install -g @mermaid-js/mermaid-cli
mmdc -i 10_architecture_diagrams.md -o diagrams/ -e png
```
