# Security Analysis

## Executive Summary

DeerFlow demonstrates strong security awareness in its sandbox isolation, path validation, and guardrail systems. However, the absence of authentication and rate limiting makes it unsuitable for multi-user or internet-facing deployments without additional hardening.

---

## Current Security Posture

### Strengths

| Area | Implementation | Assessment |
|------|---------------|------------|
| **Sandbox Isolation** | Virtual path abstraction, Docker option | Strong |
| **Path Traversal Protection** | Rejects `..`, backslash variants in all file ops | Strong |
| **ZIP Bomb Defense** | 512MB uncompressed size limit on skill archives | Good |
| **Host Path Masking** | Real paths never appear in agent output | Strong |
| **Guardrails Middleware** | Configurable pre-tool-call authorization | Strong (when enabled) |
| **API Key Management** | `$VAR_NAME` env injection, .env gitignored | Good |
| **Atomic File I/O** | temp + rename for memory writes | Good |
| **Symlink Removal** | Skill archives stripped of symlinks post-extraction | Good |
| **Harness Boundary** | CI-enforced import firewall | Strong |

### Gaps

| Area | Current State | Risk Level |
|------|--------------|------------|
| **Authentication** | None (anyone with network access can use) | Critical |
| **Rate Limiting** | None (single user can exhaust API quotas) | High |
| **CORS** | Allows all origins via Nginx | Medium-High |
| **Input Sanitization** | No sanitization on LLM prompts | Medium |
| **Secrets in Memory** | Memory facts may contain sensitive info | Medium |
| **Local Sandbox** | Default provider executes on host | Medium |
| **Session Management** | No sessions, no token expiry | High |

---

## Detailed Analysis

### 1. Sandbox Security

**Virtual Path System**

The agent operates in a virtual filesystem:
```
/mnt/user-data/workspace  →  backend/.deer-flow/threads/{id}/user-data/workspace
/mnt/user-data/uploads    →  backend/.deer-flow/threads/{id}/user-data/uploads
/mnt/user-data/outputs    →  backend/.deer-flow/threads/{id}/user-data/outputs
/mnt/skills               →  deer-flow/skills/
```

**Path Traversal Protection** (`test_sandbox_tools_security.py`):
- Blocks `../` directory traversal
- Blocks `..\\` backslash traversal
- Blocks absolute paths outside virtual roots
- Post-processes tool output to replace physical paths with virtual paths

**Local vs Docker Sandbox**:
- **LocalSandboxProvider** (default): Executes commands directly on the host. The virtual path system provides path isolation but not process isolation. A malicious prompt could potentially execute arbitrary commands.
- **AioSandboxProvider** (Docker): Full container isolation with network restrictions. Recommended for production.

**Recommendation**: Default to Docker sandbox in production. Document the security implications of local sandbox clearly.

---

### 2. File Upload Security

**Location**: `backend/app/gateway/routers/uploads.py`

**Protections**:
- **Filename normalization**: `Path(filename).name` strips directory components
- **Path separators rejected**: Filenames with `/` or `\` are rejected
- **Safe ZIP extraction**: Rejects entries with absolute paths, `..` components, or symlinks
- **ZIP bomb defense**: Maximum 512MB uncompressed size per archive
- **Symlink removal**: Post-extraction scan removes all symlinks

**Potential concern**: The `markitdown` document conversion (PDF/PPT/Excel/Word → Markdown) processes untrusted files. If `markitdown` has vulnerabilities in its parsers, malicious documents could exploit them.

**Recommendation**: Run document conversion in the Docker sandbox rather than the gateway process.

---

### 3. Guardrails System

**Location**: `backend/packages/harness/deerflow/guardrails/`

Three provider options:

**AllowlistProvider** (built-in, zero dependencies):
```yaml
guardrails:
  enabled: true
  provider:
    use: deerflow.guardrails.builtin:AllowlistProvider
    config:
      denied_tools: ["bash", "write_file"]
```

**OAP Provider** (Open Agent Passport standard):
```yaml
guardrails:
  enabled: true
  provider:
    use: aport_guardrails.providers.generic:OAPGuardrailProvider
```

**Custom Provider** (implement `evaluate()`/`aevaluate()`):
```yaml
guardrails:
  enabled: true
  provider:
    use: my_package:MyGuardrailProvider
```

**Key design decisions**:
- **Fail-closed by default**: Policy evaluation errors → deny
- **Per-tool-call evaluation**: Each tool call is authorized independently
- **Error as ToolMessage**: Denied calls return error content, not exceptions

**Recommendation**: Enable guardrails by default in `config.example.yaml` with a sensible allowlist. Currently disabled by default.

---

### 4. Authentication Gap

**Current state**: No authentication layer at any service.

**Attack surface**:
- LangGraph Server (port 2024): Full agent access
- Gateway API (port 8001): Memory read/write, file upload, config modification
- Nginx (port 2026): Exposes both services

**Existing scaffolding**: `better-auth` is installed in the frontend with GitHub OAuth configuration started, but not active.

**Recommendation priority**:
1. Add API key authentication to Gateway endpoints
2. Activate better-auth for frontend sessions
3. Add authentication to LangGraph Server (or ensure it's only accessible via Nginx with auth)

---

### 5. CORS Configuration

**Location**: `docker/nginx/nginx.conf`

**Current**: Allows all origins (`Access-Control-Allow-Origin: *`)

**Risk**: Any website can make API calls to the DeerFlow instance if a user has it running locally.

**Recommendation**: Restrict to specific origins in production. In development, allow `localhost` only.

---

### 6. Memory System Security

**Concerns**:
- Memory facts are extracted by LLM from conversations — could inadvertently store sensitive information (API keys, passwords, personal data)
- Memory file (`memory.json`) has no encryption
- Memory injection into system prompts could be a prompt injection vector (tested in `test_memory_prompt_injection.py`)

**Existing mitigations**:
- Confidence threshold (0.7 default) filters low-quality facts
- Test coverage for prompt injection in memory

**Recommendations**:
- Add sensitive data detection before storing facts (PII, API keys, passwords)
- Consider encrypting memory at rest
- Add memory fact review UI for users to audit and delete sensitive facts

---

### 7. MCP Server Security

**Concerns**:
- MCP servers execute with the permissions of the DeerFlow process
- OAuth tokens are stored in config files
- Malicious MCP servers could exfiltrate data through tool responses

**Mitigations**:
- Environment variable injection (`$VAR_NAME`) keeps secrets out of config files
- Per-server enable/disable control

**Recommendations**:
- Add MCP server sandboxing (network restrictions, filesystem isolation)
- Validate MCP tool responses before passing to the agent
- Add MCP server trust levels (local = trusted, remote = sandboxed)

---

### 8. Prompt Injection Risks

**Attack vectors**:
- **Direct**: Malicious user input designed to override system instructions
- **Indirect**: Malicious content from web_search/web_fetch results
- **Memory-based**: Facts extracted from adversarial conversations

**Existing mitigations**:
- Guardrails middleware can block dangerous tool calls
- Memory prompt injection test coverage
- Agent system prompt includes behavioral guardrails

**Recommendations**:
- Add content filtering on web_fetch results
- Implement output validation for agent responses
- Consider a dedicated prompt injection detection layer

---

### 9. Environment Variable Handling

**Pattern**: Values starting with `$` in config files are resolved from environment.

```yaml
api_key: $OPENAI_API_KEY  # Resolved at runtime from env
```

**Strengths**:
- Secrets never in config files
- `.env` file is gitignored
- Docker compose passes env vars from host

**Concerns**:
- All environment variables are accessible to the DeerFlow process
- No secret rotation mechanism
- No encryption of secrets in transit between services

**Recommendation**: Consider integrating with a secrets manager (Vault, AWS Secrets Manager) for production deployments.

---

## Security Hardening Checklist

For production deployment:

- [ ] Enable Docker sandbox (`sandbox.use: deerflow.community.aio_sandbox:AioSandboxProvider`)
- [ ] Enable guardrails with tool allowlist
- [ ] Restrict CORS to specific origins
- [ ] Add authentication (activate better-auth or add API key auth)
- [ ] Restrict Nginx to internal network or add auth proxy
- [ ] Audit memory facts for sensitive data
- [ ] Set rate limits on all API endpoints
- [ ] Enable HTTPS (TLS termination at Nginx)
- [ ] Review MCP server configurations for data exposure
- [ ] Set `allowed_users` for IM channel integrations
- [ ] Regular dependency vulnerability scanning (`uv audit`, `pnpm audit`)
- [ ] Enable logging and monitoring for security events

---

## Security-Related Tests

| Test File | Coverage |
|-----------|----------|
| `test_sandbox_tools_security.py` | Path traversal, virtual path masking, command validation |
| `test_memory_prompt_injection.py` | Memory injection into prompts |
| `test_guardrail_middleware.py` | Pre-tool-call authorization |
| `test_skills_archive_root.py` | Skill archive extraction safety |
| `test_harness_boundary.py` | Import boundary enforcement |

---

## Responsible Disclosure

Security vulnerabilities should be reported at:
https://github.com/bytedance/deer-flow/security

Supported branches:
- `main` (v2.x) — actively maintained
- `main-1.x` (v1.x) — security fixes only
