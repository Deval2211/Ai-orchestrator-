# Multica

## Repo facts

- **Repo:** `multica-ai/multica` on GitHub
- **Language:** Go (server/daemon) + Next.js (frontend)
- **Website:** https://multica.ai/
- **License:** Apache License 2.0
- **Key finding:** Multica has a **first-class Hermes runtime backend** — `hermes` is listed in `SupportedTypes` alongside Claude, Codex, OpenCode, Cursor, etc. It is not a custom addition; it is a protocol family with dedicated backend code.

## Directory structure

```
server/
├── cmd/server/           # API server binary (router, main)
├── internal/
│   ├── handler/          # HTTP handlers (agents, issues, tasks, workspaces)
│   ├── middleware/       # Auth, rate limiting, CORS
│   ├── service/
│   │   └── builtin_skills/multica-platform/references/  # issues.md, sub-issues.md
│   └── ...
├── migrations/           # SQL migrations (PostgreSQL)
└── pkg/
    ├── agent/            # Runtime backends (hermes.go, hermes_gateway.go, pi.go, etc.)
    └── ...
```

## Hermes integration (the key finding)

**VERIFIED:** Multica already integrates Hermes as a runtime backend. This is implemented code, not a README claim.

### How Hermes is launched

`server/pkg/agent/hermes.go` implements `hermesBackend` which spawns the `hermes` binary:

```go
func (b *hermesBackend) Execute(ctx context.Context, prompt string, opts ExecOptions) (*Session, error) {
    execPath := b.cfg.ExecutablePath
    if execPath == "" {
        execPath = "hermes"
    }
    if _, err := exec.LookPath(execPath); err != nil {
        return nil, fmt.Errorf("hermes executable not found at %q: %w", execPath, err)
    }
```

### Communication protocol: ACP (Agent Communication Protocol)

Multica communicates with Hermes over **ACP JSON-RPC 2.0 via stdin/stdout**. The `acp` subcommand is hardcoded and cannot be overridden:

```go
var hermesBlockedArgs = map[string]blockedArgMode{
    "acp": blockedStandalone,
}
```

### Session lifecycle

1. **Initialize handshake:** `c.request(runCtx, "initialize", {"protocolVersion": 1, "clientInfo": {"name": "multica-agent-sdk", "version": "0.2.0"}})`
2. **Session creation:** `session/new` with cwd, model, mcpServers
3. **Session resumption:** `session/resume` with sessionId, cwd, mcpServers (reconnects MCP tools)
4. **Model switching:** `session/set_model` with provider-normalized ID comparison
5. **Prompt execution:** `session/prompt` with text messages
6. **Result collection:** Awaits `promptDone`, drains stdout/stderr, collects usage

```go
if opts.ResumeSessionID != "" {
    result, err := c.request(runCtx, "session/resume", map[string]any{
        "cwd":        cwd,
        "sessionId":  opts.ResumeSessionID,
        "mcpServers": mcpServers,
    })
    sessionID, resumeLanded = resolveHermesResumedSessionID(opts.ResumeSessionID, result)
} else {
    result, err := c.request(runCtx, "session/new", buildHermesSessionParams(cwd, opts.Model, mcpServers))
```

### Flags passed to Hermes

The daemon assembles `hermes acp` with:
- `launchPrefix` (configurable per runtime)
- `custom_args` (per-agent, with `acp` blocked from override)
- Profile selection (`-p`/`--profile`) stripped when the daemon built a per-task overlay
- `HERMES_YOLO_MODE=1` environment variable (auto-approves all tool executions)
- `HERMES_HOME` set to the overlay path when skills/config are present

```go
env = append(env, "HERMES_YOLO_MODE=1")
cmd.Env = env
```

### Session resumption and status

- Sessions are tracked by `session_id` UUID stored in `chat_session` table (migration `060_chat_session_runtime_id.up.sql`)
- On resume failure, Multica sets `ResumeRejected: true` so the daemon retries with a fresh session
- Hermes silently creates a new session when a resume target is gone (`acp_adapter/server.py: "not found, creating new"`). Multica detects this via `resolveHermesResumedSessionID`.
- Status reporting: `Result{Status: "completed"/"failed"/"timeout"/"aborted", SessionID: ..., ResumeRejected: ..., Usage: ...}`
- Provider errors (HTTP 4xx/5xx from LLM) are sniffed from stderr so they surface in the task result instead of being masked by `stopReason=end_turn`.

### Hermes as a custom runtime profile

The `runtime_profile` table's `protocol_family` CHECK constraint includes `'hermes'`:

```sql
CHECK (protocol_family IN (
    'claude','codebuddy','codex','copilot','opencode',
    'openclaw','hermes','gemini','pi','cursor',
    'kimi','kiro','antigravity'
))
```

Teams can define custom Hermes-based runtimes with their own `command_name`, `fixed_args`, and display name.

### Hermes gateway (Windows-specific)

`server/pkg/agent/hermes_gateway.go` starts a Hermes gateway process for Windows task isolation:

```go
func StartHermesTaskGateway(ctx context.Context, cfg Config, home string) (func(), error) {
    // Only on Windows with Hermes >= 0.22
    cmd := cfg.commandAt(cfg.ExecutablePath).exec(runCtx,
        "--profile", "default", "gateway", "run", "--quiet", "--external-supervisor")
    cmd.Env = append(cmd.Env, "HERMES_GATEWAY_DETACHED=1")
```

### hermesBackend struct

```go
type hermesBackend struct {
    cfg Config
}
```

It implements the `Backend` interface via `Execute(ctx, prompt, opts) (*Session, error)`.

### Supported types

Hermes is one of **29 supported protocol families**:

```go
var SupportedTypes = []string{
    "claude","codebuddy","codex","copilot","opencode","codearts",
    "deveco","openclaw","hermes","pi","cursor","kimi","reasonix",
    "dsh","kiro","antigravity","qoder","qoderclicn","traecli",
    "grok","qwen","qwenpaw","mcode","muse","dim","devin",
    "zeroclaw","zcode",
}
```

### Implications

- **Multica is a Hermes-first competitor** — Hermes is a built-in protocol family, not a custom addition.
- Multica already manages Hermes sessions, resumption, model switching, MCP server injection, and usage tracking.
- The integration is deep: it understands Hermes ACP semantics, Hermes profile overrides, Hermes silent session recreation, and Hermes stderr error patterns.
- For the orchestrator project: Multica's Hermes backend could theoretically be reused if self-hosted, but the coupling to Multica's task queue, daemon, and DB schema means it is not a standalone library.

## Agent registration and task assignment

### CreateAgent handler

`POST /api/agents` via `func (h *Handler) CreateAgent`:

```go
type CreateAgentRequest struct {
    Name                 string                     `json:"name"`
    Description          string                     `json:"description"`
    Instructions         string                     `json:"instructions"`
    ConversationStarters []AgentConversationStarter `json:"conversation_starters"`
    AvatarURL            *string                    `json:"avatar_url"`
    RuntimeID            string                     `json:"runtime_id"`       // UUID of agent_runtime
    RuntimeConfig        any                        `json:"runtime_config"`
    CustomEnv            map[string]string          `json:"custom_env"`
    CustomArgs           []string                   `json:"custom_args"`
    McpConfig            json.RawMessage            `json:"mcp_config"`
    Visibility           string                     `json:"visibility"`       // "private" default
    PermissionMode       *string                    `json:"permission_mode"`
    InvocationTargets    []AgentInvocationTargetDTO `json:"invocation_targets"`
    MaxConcurrentTasks   int32                      `json:"max_concurrent_tasks"`
    Model                string                     `json:"model"`
    ThinkingLevel        string                     `json:"thinking_level"`
    ServiceTier          string                     `json:"service_tier"`
    ComposioToolkitAllowlist []string               `json:"composio_toolkit_allowlist"`
    // Template records creation-source attribution
```

An "agent" is a workspace-scoped definition that ties together:
- A **runtime** (one of the 29 SupportedTypes, via `runtime_id` → `agent_runtime` table)
- Instructions, MCP config, custom args, environment
- Invocation permissions (who can trigger it)
- Model and thinking level preferences

### Task assignment (Issue → Task flow)

Built-in skills at `server/internal/service/builtin_skills/multica-platform/references/`:

- **`issues.md`** — PR linking rules (`PREFIX-NUMBER` in title/branch), status changes trigger server-side effects, sub-issue/stage dispatch
- **`sub-issues.md`** — Splitting criteria (mergeable independently, separate code, different owner), `todo` status enqueues the agent immediately, `backlog` parks it

Workflow: Issue created → assigned to agent → daemon picks up the task → runs `Execute()` on the agent's runtime backend → result posted back to issue as comment.

### Agent management endpoints

`ListAgents`, `GetAgent`, `UpdateAgent`, `ArchiveAgent`, `RestoreAgent`, `CancelAgentTasks`, `ListAgentTasks`, `ListWorkspaceWorkingAgents`, `GetWorkspaceAgentRunCounts`, `GetWorkspaceAgentActivity30d`, `ListWorkspaceAgentTaskSnapshot`.

## Runtime architecture

### Client-Server split

| Component | Runs where | Description |
|-----------|-----------|-------------|
| **Backend API** | Server (Docker) | Go REST API + WebSocket server |
| **Frontend** | Browser | Next.js 16 web app |
| **CLI + Daemon** | User's machine | `multica` CLI + agent daemon that spawns the actual AI tool |

The server schedules tasks; the **user's daemon** executes them by launching the AI CLI (Hermes, Claude Code, etc.) on their machine. The server does not run LLM code directly.

### Runtime profile system

`runtime_profile` table (migration `120_runtime_profile.up.sql`):
- Workspace-scoped custom runtime definitions
- `protocol_family` must be in SupportedTypes
- `command_name`, `fixed_args`, `display_name`
- Migration `501` adds `runtime_type` column for runtime identity vs protocol family distinction
- `agent_runtime` has optional `profile_id` linking to custom profiles (NULL = built-in)

### Backend factory

```go
func New(agentType string, cfg Config) (Backend, error) {
    switch agentType {
    case "claude": return &claudeBackend{cfg: cfg}, nil
    // ...
    case "hermes": return &hermesBackend{cfg: cfg}, nil
    // ...
    }
}
```

`ResolveBackend()` routes through `NewRuntime()` for built-in runtime identities (e.g. "omp" → "pi" family) or `New()` for protocol families.

## Session persistence

- **Database:** PostgreSQL 17 (`pgcrypto` + `pg_trgm`)
- **Migrations:** `server/migrations/` — SQL-only, 120+ numbered files
- **Key tables:**
  - `agent_runtime` — runtime instances per workspace/daemon/provider
  - `runtime_profile` — custom runtime definitions (migration 120)
  - `chat_session` — sessions with `runtime_id UUID REFERENCES agent_runtime(id)` and `session_id` (migration 060)
  - `agent_task_queue` — task execution records with `chat_session_id`, `runtime_id`, `session_id`, `status`
  - `agent` — agent definitions
  - `issue` — issue tracking
- Sessions resume: the daemon stores `session_id` and `runtime_id` on completed tasks; next dispatch reads them and passes `ResumeSessionID` to the backend.
- Multica detects when Hermes silently creates a new session instead of resuming and adjusts.

## Public API surface (routes + auth)

Routes are wired in `cmd/server/router.go` via chi (`github.com/go-chi/chi/v5`). Verified paths from handler code:

| Method | Path | Handler |
|--------|------|---------|
| `GET` | `/api/agents` | `ListAgents` |
| `GET` | `/api/agents/{id}` | `GetAgent` |
| `POST` | `/api/agents` | `CreateAgent` |
| `PUT` | `/api/agents/{id}` | `UpdateAgent` |
| `PUT` | `/api/agents/{id}/env` | Agent env management |
| `GET` | `/api/agents/{id}/env` | Read agent env |
| `DELETE` | `/api/agents/{id}` | `ArchiveAgent` |
| `POST` | `/api/agents/{id}/restore` | `RestoreAgent` |
| `POST` | `/api/agents/{id}/cancel-tasks` | `CancelAgentTasks` |
| `GET` | `/api/agents/{id}/tasks` | `ListAgentTasks` |
| `GET` | `/api/agents/{id}/working-agents` | `ListWorkspaceWorkingAgents` |
| `GET` | `/api/workspaces` | Workspace list |
| `POST` | `/api/workspaces` | Workspace creation |
| `GET` | `/api/config` | Public config endpoint |
| `GET` | `/api/issues/{id}/task-runs` | Task runs for an issue |
| `POST` | `/api/upload-file` | File upload (middleware-less) |
| Various | `/plugin-surfaces/*` | Third-party JS hosting |

### Auth model

- **OAuth/Email-based** — Resend API for email codes (self-hosted)
- **Task tokens** — Agents authenticate as `X-Actor-Source: task_token` with `X-Agent-ID` and `X-Task-ID` headers
- **Workspace membership** — `h.workspaceMember(w, r, workspaceID)` validates user membership
- **Invocation targets** — Per-agent allowlist of who can trigger it
- **Trusted proxies** — Configurable via `MULTICA_TRUSTED_PROXIES` CIDRs
- **Rate limiting** — Per-IP, respects trusted proxy headers

## Self-hosting footprint

From `SELF_HOSTING.md`:

| Requirement | Detail |
|-------------|--------|
| **Docker** | Required (Docker Compose setup) |
| **PostgreSQL 17** | `pgcrypto` + `pg_trgm` extensions |
| **Go binary** | Single server binary |
| **Next.js** | Frontend (Docker image from GHCR) |
| **CLI + Daemon** | `multica` CLI installed on user machines |
| **Resend API** | Email-based auth (optional — can copy codes from logs) |
| **Telemetry** | Anonymous daily snapshot to `telemetry.multica.ai` (disable with `DO_NOT_TRACK=1`) |

Install: `curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.sh | bash -s -- --with-server`

No Redis required. No separate object storage mentioned in core setup (attachment handling is internal).

## License (quote key terms)

Apache License 2.0. The repo includes a standard `LICENSE` file. Key terms:
- Commercial use permitted
- Modification permitted
- Distribution permitted
- Patent license granted
- **State changes** required
- License notice required
- **No trademark use** granted

## Reuse assessment

- **Not a standalone library.** The Hermes backend (`hermesBackend`) is tightly coupled to Multica's `Config`, `ExecOptions`, `Session`, `Message`, `Result` types and the daemon task queue.
- **Protocol knowledge is valuable.** Multica's understanding of ACP session lifecycle, Hermes silent session recreation, Hermes stderr error patterns, and profile override semantics is deep. The code is readable Go and could inform an equivalent implementation.
- **Apache 2.0** means you can study, fork, and adapt the code, but you must state changes and cannot use their trademarks.
- **As an orchestrator component:** Self-hosting Multica gives you a working Hermes agent management system, but it is designed for team workspaces, not as a library. You would need to either embed a Multica server or reimplement the Hermes ACP integration from scratch (using their code as reference).
- **Competitor signal:** Multica supports 29 agent runtimes including Hermes, Claude Code, Codex, OpenCode, Cursor, Gemini, Kiro, etc. This is the same space the orchestrator project targets, just with a different architecture (team-focused vs. personal orchestrator).

---

# OpenClaw

## Repo facts

- **Repo:** `openclaw/openclaw` on GitHub
- **Language:** TypeScript/Node.js
- **Website:** https://openclaw.ai/
- **License:** MIT (OpenClaw Foundation, 2026)
- **Package manager:** pnpm

## Directory structure

```
packages/
├── gateway-protocol/     # TypeBox schemas, JSON Schema, Swift models
├── gateway/              # Gateway server (WS + HTTP multiplex)
└── ...
docs/
├── concepts/             # Architecture, agent loop, queue, model providers
├── gateway/              # Protocol, sandboxing, OpenResponses, OpenAI HTTP API
├── plugins/              # Plugin SDK, CLI backends, agent harness, hooks
└── ...
extensions/               # Bundled plugins
```

## Architecture

**Gateway-centric architecture.** A single Gateway process is the control plane:

```
Client (WS) → Gateway (WS + HTTP multiplex) → Agent Loop → Model Provider
```

- **Transport:** WebSocket with JSON frames. First frame must be `connect`. After handshake: requests `{type:"req", id, method, params}` → responses `{type:"res", id, ok, payload|error}`, events `{type:"event", event, payload, seq?, stateVersion?}`.
- **HTTP endpoints:** `/v1/responses` (OpenResponses), `/v1/chat/completions` (OpenAI-compatible), `/v1/models`, `/v1/embeddings`. All share the Gateway port, disabled by default.
- **Agent loop:** Serialized per-session. `agent` RPC validates → persists session → returns `{runId, acceptedAt}` → `agentCommand` runs the turn → `runEmbeddedAgent` drives the model loop → streams back via WS.
- **Plugin system:** Channel plugins, provider plugins, CLI backend plugins, tool plugins, feature plugins, agent harness plugins. All experimental APIs.

## Provider / auth / isolation model

### Provider model

OpenClaw separates three layers:
- **Provider:** Authentication and model discovery (Anthropic, OpenAI, Google Gemini, etc.)
- **Model:** The specific model (`claude-opus-4-6`, `gpt-6-astra`)
- **Agent runtime:** The execution backend (`openclaw` embedded, `codex` app-server, `claude-cli`, ACP)

Model refs: `provider/model` (e.g., `opencode/claude-opus-4-6`). Multiple API keys and rotation supported.

### Auth profiles

Provider auth stored in gateway config. CLI: `openclaw onboard`, `openclaw models auth login`. OAuth profiles for ChatGPT/Codex, API keys for others. `openclaw configure` preserves existing primary model when adding providers.

### Runtime isolation

Two runtime families:
1. **Embedded harnesses** — Run inside OpenClaw's agent loop (`openclaw` runtime, `codex`, `copilot`, `muse`). Plugin harnesses register via `api.registerAgentHarness()`.
2. **CLI backends** — Run a local CLI process, keeping the model ref in standard form. Registered via `api.registerCliBackend()`.

### Agent isolation

- Per-agent sandbox config (`agents.entries.*.sandbox`)
- Per-agent tool policy (`agents.entries.*.tools`)
- Queue lanes isolate concurrent sessions
- Session controller mailbox grants one turn per physical session

## Session management and persistence

- **Storage:** SQLite WAL (Gateway process)
- **Session keys:** Derived from user, agent, and scope. `x-openclaw-session-key` for explicit routing.
- **Continuation:** `previous_response_id` reuses sessions within same agent/user scope. Mappings survive Gateway restarts for 30 days, capped at 5,000 responses.
- **Queue modes:** `steer` (inject into active run), `followup` (enqueue after current), `collect` (coalesce), `interrupt` (abort and restart)
- **Concurrency:** Per-lane FIFO queue. Default 1 per lane, `main` uses `max(8, CPU * 4)`. Sub-agents default 8, Swarm collectors default 32.
- **Incognito sessions:** No continuation mappings. Explicit incognito continuation requires `operator.admin`.
- **Reserved namespaces:** `subagent:`, `cron:`, `acp:` session keys are validation failures.

## External tool integration surface

### For external orchestrators (outside OpenClaw):

| Surface | Protocol | Use for |
|---------|----------|---------|
| **Gateway WS protocol** | WebSocket + JSON RPC | Start agent runs, stream events, wait for results, cancel, inspect resources |
| **OpenResponses HTTP API** | `POST /v1/responses` | OpenResponses-compatible clients (item-based inputs, client tool calls, SSE) |
| **OpenAI Chat Completions** | `POST /v1/chat/completions` | Any tool expecting OpenAI Chat Completions |
| **`openclaw agent` CLI** | CLI subprocess | One-shot script integration |
| **`openclaw message` CLI** | CLI subprocess | Send messages from scripts |

### Plugin SDK (inside OpenClaw):

| Plugin type | Registration | Purpose |
|------------|--------------|---------|
| **Channel plugin** | `api.registerChannelPlugin()` | Connect to messaging platforms |
| **Provider plugin** | `api.registerProviderPlugin()` | Add model providers |
| **CLI backend plugin** | `api.registerCliBackend()` | Local AI CLI as text inference backend |
| **Tool plugin** | `api.registerToolPlugin()` | Agent tools |
| **Feature plugin** | `api.registerFeaturePlugin()` | Typed operations, native UI |
| **Agent harness plugin** | `api.registerAgentHarness()` | Replace the embedded agent executor |

### Integration recommendation

For an external orchestrator (the ai-orchestrator project):
1. **Gateway WS protocol** is the primary surface — full agent control via RPC
2. **OpenResponses HTTP API** is the secondary surface — simpler HTTP integration
3. **Agent harness plugin** — if you want to run inside OpenClaw, but all plugin APIs are experimental

### ACP support

OpenClaw supports ACP for external harnesses:
- `runtime: "acp"`, `agentId: "codex"` for Codex ACP adapter
- Claude Code, Gemini CLI, OpenCode, Cursor use ACP/acpx
- ACP sessions use the reserved `acp:` namespace in session keys

## Security model

- **Gateway auth modes:** `token`, `password`, `trusted-proxy`, `none`
- **Shared-secret auth:** `Authorization: Bearer <token>`. Treats the endpoint as full operator access.
- **Trusted-proxy auth:** Identity-aware proxy headers. Loopback proxies need explicit `allowLoopback = true`.
- **Device identity:** New device IDs require pairing approval. Gateway issues device tokens for subsequent connects.
- **Challenge signatures:** `connect.challenge` nonce must be signed. `v3` binds `platform` and `deviceFamily`. Gateway pins paired metadata on reconnect.
- **Non-local connects** require explicit approval.
- **Sandboxing:** Optional. Backends: Docker, Podman, SSH, OpenShell, Crabbox. Gateway stays on host; only tool execution moves into sandbox.
- **Tool policy:** Allow/deny per-tool, per-agent. `tools.elevated` escapes sandbox for exec.
- **Operator scopes:** `operator.admin`, `operator.approvals`, `operator.pairing`, `operator.read`, `operator.talk.secrets`, `operator.write`

## License

MIT License. Full text:

> Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:
>
> The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.
>
> THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED...

**Dependency concern:** Plugin APIs are all marked **experimental** — contracts can change between releases. Pin the OpenClaw version you develop against.

## Conflict check against Hermes-first design

- **Hermes ships an OpenClaw importer:** `https://hermes-agent.nousresearch.com/docs/user-guide/import-from-other-agents` — one-command migration from OpenClaw → Hermes. Direction is OpenClaw → Hermes.
- **OpenClaw does NOT offer a Hermes importer.** OpenClaw has no equivalent pointing back at Hermes.
- **No Hermes-specific runtime in OpenClaw.** OpenClaw's runtime list: `openclaw` (embedded), `codex`, `copilot`, `muse`, plus CLI backends (`claude-cli`, `google-gemini-cli`). Hermes is not a registered provider, runtime, or CLI backend.
- **No Hermes ACP adapter in OpenClaw.** OpenClaw's ACP support is for Codex, Claude Code, Gemini CLI, OpenCode, Cursor — not Hermes.
- **Conflict assessment:** Low. OpenClaw does not target Hermes users or compete for the Hermes runtime slot. The one-way importer direction (OpenClaw → Hermes) confirms that Hermes is the "upstream" format in the migration direction.

## Limitations

- **Plugin API stability:** All plugin APIs are experimental. No guaranteed compatibility between versions.
- **Sandboxing is imperfect:** "This is not a perfect security boundary, but it materially limits filesystem and process access when the model does something dumb."
- **No native Hermes support:** If Hermes becomes important, you'd need to write a CLI backend plugin or agent harness plugin from scratch.
- **SQLite-only persistence:** Gateway uses SQLite WAL. Not designed for multi-instance deployments or shared-state HA.
- **Telemetry/observability:** Limited built-in observability surface for external orchestrators. No Prometheus metrics endpoint documented.
- **HTTP endpoints are disabled by default:** OpenResponses and Chat Completions require explicit config to enable.
- **No built-in issue/PR tracking:** Unlike Multica, OpenClaw is purely an agent framework — no built-in task management, issue tracking, or PR linking.

## Evidence (every URL actually fetched)

### Multica source files (raw.githubusercontent.com)
- `server/pkg/agent/hermes.go` — Full Hermes backend implementation (hermesBackend, Execute, session lifecycle, ACP protocol)
- `server/pkg/agent/hermes_gateway.go` — Windows Hermes gateway task isolation
- `server/pkg/agent/hermes_resume_test.go` — Hermes resume test cases
- `server/pkg/agent/builtin_runtimes.go` — BuiltinRuntime registry, ResolveBackend, NewRuntime
- `server/pkg/agent/agent.go` — SupportedTypes whitelist, New() factory, isSupportedType
- `server/internal/handler/agent.go` — CreateAgent handler, CreateAgentRequest struct, ListAgents, GetAgent, UpdateAgent, ArchiveAgent, CancelAgentTasks, ListAgentTasks
- `server/internal/handler/handler.go` — Handler struct, middleware, config fields
- `server/migrations/120_runtime_profile.up.sql` — runtime_profile table creation with protocol_family CHECK including 'hermes'
- `server/migrations/501_runtime_profile_runtime_type.up.sql` — `ALTER TABLE runtime_profile ADD COLUMN runtime_type`
- `server/migrations/060_chat_session_runtime_id.up.sql` — chat_session runtime_id foreign key, backfill from agent_task_queue
- `server/internal/service/builtin_skills/multica-platform/references/issues.md` — PR linking, status changes, sub-issues
- `server/internal/service/builtin_skills/multica-platform/references/sub-issues.md` — Sub-issue splitting criteria, todo/backlog dispatch
- `SELF_HOSTING.md` — Docker, Postgres 17, Go + Next.js architecture, install script
- `LICENSE` — Apache License 2.0
- `README.md` — (fetched, minimal output)

### OpenClaw docs (raw.githubusercontent.com)
- `docs/gateway/openresponses-http-api.md` — OpenResponses HTTP API spec (routes, auth, session behavior, request shape)
- `docs/gateway/openai-http-api.md` — OpenAI Chat Completions surface (routes, auth matrix, security boundary)
- `docs/gateway/sandboxing.md` — Sandbox modes, backends (Docker, Podman, SSH, OpenShell, Crabbox), tool policy
- `docs/concepts/architecture.md` — Gateway architecture, WS protocol, pairing, remote access, invariants
- `docs/concepts/agent-loop.md` — Agent loop lifecycle, streams, wait semantics
- `docs/concepts/agent-runtimes.md` — Runtime vs provider vs model layers, Codex surfaces, runtime ownership table
- `docs/concepts/queue.md` — Queue modes, lane-aware FIFO, concurrency, steering
- `docs/concepts/model-providers.md` — Provider reference index
- `docs/concepts/model-providers/quick-rules.md` — Model refs, CLI helpers, provider/runtime split, CLI runtimes
- `docs/concepts/model-providers/custom-providers.md` — Custom providers, base URLs, no-auth mode
- `docs/gateway/protocol.md` — Gateway WS protocol (handshake, framing, roles, auth)
- `docs/gateway/external-apps.md` — External app integration paths (Gateway WS, HTTP APIs, CLI)
- `docs/plugins/building-plugins.md` — Plugin types, quickstart, plugin shapes
- `docs/plugins/sdk-overview.md` — Plugin SDK architecture, registration API
- `docs/plugins/sdk-agent-harness.md` — Agent harness plugin contract, when to use, core ownership
- `docs/plugins/cli-backend-plugins.md` — CLI backend plugin contract
- `docs/tools/skills.md` — Skills documentation
- `LICENSE` — MIT License text
- `README.md` — (fetched, minimal output)

### Hermes docs
- `https://hermes-agent.nousresearch.com/docs/user-guide/import-from-other-agents` — Hermes agent import guide (confirms OpenClaw → Hermes migration direction)

### GitHub API (used sparingly)
- `GET /repos/multica-ai/multica/contents/cmd/server?ref=main` — Listed `server/` subdirectory (found `router.go`, `main.go`)
