# Real-World Usage Guide

## Quick Start

### Prerequisites
- Python 3.12+
- Node.js 22+
- pnpm 10+
- An LLM API key (OpenAI, Anthropic, Google, DeepSeek, etc.)

### First-Time Setup

```bash
# 1. Check system requirements (must run from project root!)
make check

# 2. Install all dependencies (backend + frontend)
make install

# 3. Generate configuration files
make config
# This creates config.yaml and extensions_config.json from examples

# 4. Edit config.yaml — uncomment and configure at least one model:
# models:
#   - name: claude-3-5-sonnet
#     use: langchain_anthropic:ChatAnthropic
#     model: claude-3-5-sonnet-20241022
#     api_key: $ANTHROPIC_API_KEY
#     max_tokens: 8192
#     supports_vision: true

# 5. Set API keys in .env file
echo "ANTHROPIC_API_KEY=your-key-here" >> .env

# 6. Start all services
make dev

# 7. Open browser
# http://localhost:2026
```

### Stopping Services

```bash
make stop
```

---

## Docker Workflow

### Development (Hot Reload)

```bash
# First time setup
make docker-init    # Build images, install dependencies

# Start services
make docker-start   # Starts all containers with hot-reload

# View logs
make docker-logs

# Stop services
make docker-stop
```

### Production Deployment

```bash
# Build and start production containers
make up

# Stop and remove containers
make down
```

---

## Using Skills

DeerFlow ships with 10+ public skills. Enable them in `extensions_config.json`:

```json
{
  "skills": {
    "deep-research": {"enabled": true},
    "report-generation": {"enabled": true},
    "slide-creation": {"enabled": true},
    "image-generation": {"enabled": true},
    "podcast-generation": {"enabled": true}
  }
}
```

### Example: Deep Research

In the chat interface, ask:
> "Research the latest developments in quantum computing and summarize the key breakthroughs from 2025-2026"

The agent will:
1. Use web_search to find relevant sources
2. Use web_fetch to read full articles
3. Synthesize findings into a structured research summary

### Example: Report Generation

> "Generate a comprehensive report on renewable energy adoption trends with charts and data"

The agent will:
1. Research the topic
2. Write the report as a file in the sandbox
3. Present the file using present_file tool

### Example: Slide Creation

> "Create a 10-slide presentation about our Q1 product roadmap"

The agent will:
1. Structure the content
2. Generate HTML/PPTX slides in the sandbox
3. Make them available as downloadable artifacts

---

## Custom Agent Creation

DeerFlow supports creating custom agents with specialized configurations.

### Via the Web Interface

Use the bootstrap mode to create an agent through conversation:
1. Start a new thread
2. The system enters bootstrap mode
3. Define the agent's name, purpose, tools, and model

### Via Configuration

Create a YAML file in `backend/.deer-flow/agents/{agent_name}/config.yaml`:

```yaml
name: code-reviewer
model: claude-3-5-sonnet
soul: |
  You are an expert code reviewer. Focus on:
  - Security vulnerabilities
  - Performance bottlenecks
  - Code style and maintainability
tool_groups:
  - file:read
  - bash
```

---

## MCP Server Integration

### Example: GitHub MCP Server

Add to `extensions_config.json`:

```json
{
  "mcpServers": {
    "github": {
      "enabled": true,
      "type": "stdio",
      "command": "uvx",
      "args": ["mcp-server-github"],
      "env": {
        "GITHUB_TOKEN": "$GITHUB_TOKEN"
      }
    }
  }
}
```

Then ask the agent:
> "List the open issues in our repository and summarize the most critical ones"

### Example: Database MCP Server

```json
{
  "mcpServers": {
    "postgres": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://user:pass@localhost:5432/mydb"]
    }
  }
}
```

### Example: HTTP-based MCP Server with OAuth

```json
{
  "mcpServers": {
    "enterprise-tools": {
      "enabled": true,
      "type": "http",
      "url": "https://tools.company.com/mcp",
      "oauth": {
        "token_url": "https://auth.company.com/oauth/token",
        "client_id": "deer-flow-client",
        "client_secret": "$MCP_CLIENT_SECRET",
        "grant_type": "client_credentials"
      }
    }
  }
}
```

---

## Embedded Python Client

Use DeerFlow as a library without running any servers:

### Basic Usage

```python
from deerflow.client import DeerFlowClient

# Initialize with config
client = DeerFlowClient(
    config_path="config.yaml",
    model_name="claude-3-5-sonnet",
    thinking_enabled=True,
)

# Simple one-shot
response = client.chat("What is the capital of France?")
print(response)  # "The capital of France is Paris."

# Multi-turn (requires checkpointer)
response1 = client.chat("My name is Alice", thread_id="session-1")
response2 = client.chat("What's my name?", thread_id="session-1")
print(response2)  # "Your name is Alice."
```

### Streaming

```python
for event in client.stream("Explain quantum computing"):
    if event.type == "messages-tuple":
        if event.data.get("type") == "ai":
            print(event.data["content"], end="", flush=True)
        elif event.data.get("type") == "tool":
            print(f"\n[Tool: {event.data['name']}]")
    elif event.type == "end":
        usage = event.data.get("usage", {})
        print(f"\nTokens: {usage.get('total_tokens', 0)}")
```

### Configuration Queries

```python
# List available models
models = client.list_models()
for m in models["models"]:
    print(f"{m['name']}: thinking={m['supports_thinking']}")

# Manage skills
skills = client.list_skills()
client.update_skill("deep-research", enabled=True)

# Access memory
memory = client.get_memory()
print(f"Known facts: {len(memory.get('facts', []))}")

# Upload files
result = client.upload_files("thread-1", ["/path/to/document.pdf"])
print(result["files"][0]["virtual_path"])  # /mnt/user-data/uploads/document.pdf
```

### With Persistent State

```python
from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver

checkpointer = AsyncSqliteSaver.from_conn_string("checkpoints.db")

client = DeerFlowClient(
    checkpointer=checkpointer,
    subagent_enabled=True,
)

# Conversations persist across process restarts
for event in client.stream("Continue our discussion", thread_id="persistent-thread"):
    ...
```

---

## IM Channel Integrations

### Slack

1. Create a Slack App with Socket Mode enabled
2. Add bot token scopes: `chat:write`, `app_mentions:read`, `channels:history`
3. Configure:

```yaml
# config.yaml
channels:
  langgraph_url: http://localhost:2024
  gateway_url: http://localhost:8001
  slack:
    enabled: true
    bot_token: $SLACK_BOT_TOKEN
    app_token: $SLACK_APP_TOKEN
    allowed_users: []  # Empty = allow all
```

### Telegram

1. Create a bot via @BotFather
2. Configure:

```yaml
channels:
  telegram:
    enabled: true
    bot_token: $TELEGRAM_BOT_TOKEN
    allowed_users: []
```

### Feishu (Lark)

1. Create a Feishu app with message webhook
2. Configure:

```yaml
channels:
  feishu:
    enabled: true
    app_id: $FEISHU_APP_ID
    app_secret: $FEISHU_APP_SECRET
```

### Supported Commands (All Platforms)

| Command | Action |
|---------|--------|
| `/new` | Start a new conversation thread |
| `/status` | Show system status |
| `/models` | List available models |
| `/memory` | Show memory status |
| `/help` | Show help information |

---

## Configuring Different LLM Providers

### OpenAI

```yaml
models:
  - name: gpt-4
    display_name: GPT-4
    use: langchain_openai:ChatOpenAI
    model: gpt-4
    api_key: $OPENAI_API_KEY
    max_tokens: 4096
    supports_vision: true
```

### Anthropic Claude

```yaml
models:
  - name: claude-3-5-sonnet
    display_name: Claude 3.5 Sonnet
    use: langchain_anthropic:ChatAnthropic
    model: claude-3-5-sonnet-20241022
    api_key: $ANTHROPIC_API_KEY
    max_tokens: 8192
    supports_vision: true
    supports_thinking: true
    when_thinking_enabled:
      thinking:
        type: enabled
        budget_tokens: 10000
```

### Google Gemini

```yaml
models:
  - name: gemini-2.5-pro
    display_name: Gemini 2.5 Pro
    use: langchain_google_genai:ChatGoogleGenerativeAI
    model: gemini-2.5-pro
    google_api_key: $GOOGLE_API_KEY
    max_tokens: 8192
    supports_vision: true
```

### DeepSeek (with Thinking)

```yaml
models:
  - name: deepseek-v3
    display_name: DeepSeek V3
    use: deerflow.models.patched_deepseek:PatchedChatDeepSeek
    model: deepseek-reasoner
    api_key: $DEEPSEEK_API_KEY
    max_tokens: 16384
    supports_thinking: true
    when_thinking_enabled:
      extra_body:
        thinking:
          type: enabled
```

### OpenAI-Compatible Providers (Novita, MiniMax, OpenRouter)

```yaml
models:
  - name: novita-deepseek
    display_name: Novita DeepSeek
    use: langchain_openai:ChatOpenAI
    model: deepseek/deepseek-v3.2
    api_key: $NOVITA_API_KEY
    base_url: https://api.novita.ai/openai
    max_tokens: 4096
```

---

## Plan Mode

Enable plan mode for complex multi-step tasks:

In the frontend, toggle plan mode in settings. This activates the TodoList middleware which provides the agent with a `write_todos` tool for tracking progress.

The agent will:
1. Break down the task into steps
2. Create a todo list visible in the UI
3. Mark tasks as in_progress and completed in real-time
4. Adapt the plan based on intermediate results
