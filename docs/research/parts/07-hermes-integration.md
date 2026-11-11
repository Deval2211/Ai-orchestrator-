# Hermes Integration Surfaces — Ranked Analysis

> **Source-verified** against local Hermes source at `/home/deval/.hermes/hermes-agent/` (Nous Research, repo: NousResearch/hermes-agent). Every file path and line number below is exact. Docs claims are marked **DOCS-CLAIMED**; unsourced claims are **UNKNOWN**.

---

## Hermes Architecture Facts (source-verified)

| Fact | Source |
|------|--------|
| **Repo root** | `/home/deval/.hermes/hermes-agent/` |
| **Plugin system** | `hermes_cli/plugins.py` — `PluginContext`, `PluginManager`, `PluginManifest` |
| **Delegation provider interface** | `agent/delegation_provider.py:62` (`DelegationProvider` ABC) + `agent/delegation_registry.py:51` (`register_delegation_provider`) |
| **Plugin manifest format** | `plugins/*/plugin.yaml` (YAML with `name`, `version`, `description`, `entry_point`, `requires`, `provides`, `isolation`) |
| **TUI Gateway JSON-RPC** | `tui_gateway/server.py` (aiohttp WS + JSON-RPC 2.0), `tui_gateway/rpc_dispatch.py`, `tui_gateway/methods_*.py` |
| **OpenAI-compatible HTTP proxy** | `hermes_cli/proxy/server.py:270` — aiohttp `/v1/{tail:.*}` forwarder on port **8645** |
| **MCP client config** | `hermes_cli/mcp_config.py:210` — `config.yaml` key `mcp_servers: {name: {command, args, url, headers, tools: {include, exclude}}}` |
| **Session DB** | `hermes_state_sessions.py` — SQLite `state.db` per profile under `~/.hermes/profiles/<name>/sessions/state.db` |
| **Message append API** | `hermes_state_messages.py:429` — `SessionDB.append_message(...)` (internal, no public REST) |
| **Plugin message injection** | `hermes_cli/plugins_injection.py:12` — `inject_plugin_message(ctx, content, role, session_key, origin)` |
| **Hook names** | `hermes_cli/plugins.py:106–204` — `VALID_HOOKS` set (see Surface 1) |
| **Kanban board** | `tools/kanban_tools.py` — SQLite `kanban.db`, tools gated by `HERMES_KANBAN_TASK` env / `kanban` toolset |
| **Approval hooks** | `tools/approval_context.py:54` — `pre_approval_request`, `post_approval_response` (observer-only) |

---

## Surface 1: Plugin System (capabilities, exact API names + file:line, limits)

### 1.1 Custom Tool Registration
**API**: `PluginContext.register_tool` — `hermes_cli/plugins.py:463–500`

```python
def register_tool(
    self,
    name: str,                    # tool name (unique per toolset)
    toolset: str,                 # e.g. "orchestrator"
    schema: dict,                 # JSON Schema (OpenAI function calling format)
    handler: Callable,            # sync or async; see contract below
    check_fn: Callable | None = None,      # visibility gate (session/toolset/platform)
    requires_env: list | None = None,      # env vars that must be set
    is_async: bool = False,                 # True → handler is async def
    description: str = "",                  # human description
    emoji: str = "",                        # optional emoji
    override: bool = False,                 # replace built-in (needs config opt-in)
) -> Optional[PluginRegistration]
```

**Handler contract** (`tools/registry.py` conventions, enforced by registry):
- **Sync**: `handler(args: dict, **kw) -> str` (return string = tool result shown to model)
- **Async**: `async def handler(args: dict, **kw) -> str`
- **Streaming**: **NOT SUPPORTED** — handler returns a single string. No chunked/yield protocol exists. Progress must be faked via multiple tool calls or `post_message`/`inject_message`.
- **Error**: return `tool_error("message")` from `tools.registry` (string starting with `tool_error:`).
- **Kwargs injected**: `session_id`, `turn_id`, `tool_call_id`, `profile_name`, `cwd`, etc. (see `tools/registry.py:_call_handler`).

**Manifest format** (`plugin.yaml` example at `plugins/delegation/default/plugin.yaml`):
```yaml
name: delegation-default
version: "1.0"
description: "Default delegation strategy"
entry_point: __init__.py:register
requires:
  hermes: ">=0.5.0"
provides:
  - delegation_provider
isolation: host  # or "subprocess" (not implemented for tools)
```

### 1.2 Calling External HTTP & Streaming Back Results
- **External calls**: Handler can `httpx.post(...)` or `asyncio.to_thread(requests.post, ...)` — fully allowed.
- **Streaming results back to conversation**: **NO native streaming**. Options:
  1. **`post_message` / `apost_message`** (`hermes_cli/plugins.py:607–616`) — posts a message handle into the *current* session; `handle.edit(new_text)` updates in-place. Works from async handlers.
  2. **`inject_message`** (`hermes_cli/plugins.py:588–595`) — queues content for a `session_key` or `origin` (messaging chat). Returns `bool` (accepted). Requires `plugins.entries.<id>.allow_gateway_injection: true` in config.
  3. **`emit`** (`hermes_cli/plugins.py:986–999`) — fire-and-forget event `<plugin_key>:<event>`; subscribers (other plugins, observability) receive it. No guarantee of ordering relative to model turn.

**Limit**: `post_message` only works when a gateway session is bound to the current context (i.e. during a tool call). Cannot push to arbitrary sessions from a background thread without a `session_key`.

### 1.3 Hook Points (exact names from `VALID_HOOKS`, `hermes_cli/plugins.py:106–204`)
| Hook | Fired | Kwargs (key ones) | Returns used? |
|------|-------|-------------------|---------------|
| `pre_tool_call` | Before any tool handler | `tool_name`, `args`, `turn_id`, `tool_call_id`, `session_id` | **YES** — `{action: "block", message}` vetoes; `{action: "modify", args: {...}}` rewrites args |
| `post_tool_call` | After tool returns | `tool_name`, `args`, `result`, `turn_id`, `tool_call_id`, `session_id` | No (observer) |
| `transform_tool_result` | After tool, before model sees it | `tool_name`, `args`, `result` | **YES** — last-writer-wins string replacement |
| `pre_approval_request` | Before approval prompt | `command`, `description`, `pattern_key`, `session_id`, `turn_id` | No (observer) |
| `post_approval_response` | After user answers | `command`, `description`, `choice`, `decided_by`, `session_id` | No |
| `on_human_input_request` / `on_human_input_resolved` | Human-input tool | `prompt`, `session_id`, `turn_id` | No |
| `pre_command` | Before slash command | `surface`, `command`, `alias_used`, `args_raw`, `session_key`, `platform` | No (observer; directive-shaped returns logged only) |
| `gateway_platform_event` | Telegram/Discord events | `platform`, `event_type`, `payload` | No |
| `kanban_task_claimed` / `completed` / `blocked` | Kanban lifecycle | `task_id`, `board`, `assignee`, `run_id`, `profile_name` | No |
| `on_kanban_worker_spawned` / `exited` / `stale_claim` | Worker lifecycle | `worker_pid`, `workspace_path`, `exit_kind`, `outcome` | No |
| `on_kanban_task_updated` | Task row write | `task_id`, `changed_fields` (names only) | No |
| `on_kanban_dispatch_tick` | After dispatch lock released | `board`, `profile_name`, `dry_run`, `outcome`, `result` | No |
| `post_compaction` | After conversation compaction | `session_id`, `profile_name` | No |
| `on_room_member_activity` | Hosted rooms | `member_id`, `turn_id`, `task_id`, `kind`, `seq`, `payload` | No |

**Stability**: `VALID_HOOKS` is the **allow-list** — new hooks only added with real fire-sites (`plugins.py:193–194`). Observer hooks never block; only `pre_tool_call` and `transform_tool_result` affect execution.

### 1.4 Plugin Access to LLM & Session Context
| Capability | Available? | How |
|------------|------------|-----|
| `ctx.llm` (make own LLM calls) | **YES** | `ctx.llm` property (`plugins.py:245+`) lazily returns an `LLMClient` bound to the active profile's model. Use `await ctx.llm.chat(messages, tools, ...)` — same as core. |
| Current session ID | **YES** | `ctx.session_id` (via `plugin_session.py`) — only valid *during* a tool call / command. |
| Conversation history | **NO direct API** | Must read `state.db` via `SessionDB` (internal) or use `session_search` tool. No `ctx.get_messages()`. |
| Append to conversation | **PARTIAL** | `ctx.post_message(text)` / `ctx.apost_message(text)` posts a message *into the current session* as the plugin (role `assistant`). `ctx.inject_message(content, role, session_key, origin)` targets arbitrary session/chat (needs config opt-in). |
| Read plugin config | **YES** | `ctx.config` (read-only profile config), `ctx.manifest`, `ctx.plugin_id`. |

### 1.5 Limits & Gotchas
- **No tool streaming**: Handler runs to completion; no partial results to model.
- **`inject_message` requires gateway**: Only works when a gateway (TUI/Desktop/messaging) is running and `allow_gateway_injection: true` is set. Classic CLI REPL uses a different queue (`cli._pending_input`).
- **Session key vs session ID**: `session_key` = gateway routing key (ephemeral); `session_id` = durable `state.db` primary key. Plugins get `session_id` via context; `inject_message` needs `session_key`.
- **Isolation**: `isolation: host` runs in-process (full access). `isolation: subprocess` not implemented for tools (only for platform adapters).
- **MCP access**: `ctx.call_mcp(server, tool, arguments, timeout)` — gated by `plugins.entries.<id>.mcp_allowlist` in config (default-deny).

---

## Surface 2: TUI Gateway JSON-RPC (method names, stability, fitness)

### 2.1 Implementation
- **Server**: `tui_gateway/server.py` — aiohttp WebSocket at `/ws` (or `/api/gateway/ws`), JSON-RPC 2.0 over WS frames.
- **Dispatch**: `tui_gateway/rpc_dispatch.py` — `JsonRpcDispatcher` with `register_method(name, handler)`.
- **Method registry**: Scattered across `tui_gateway/methods_*.py` (session, tools, config, kanban, groups, mcp, etc.).

### 2.2 Key Method Names (from `methods_session.py`, `methods_tools.py`, `methods_config.py`, `contracts/*.py`)
| Method | Purpose | Direction |
|--------|---------|-----------|
| `session.create` | Create new session (returns `session_key`) | Client → Server |
| `session.submit` | Submit user turn (prompt + optional images) | Client → Server |
| `session.subscribe` | Subscribe to session events (stream) | Client → Server |
| `session.close` | End session | Client → Server |
| `session.list` | List sessions | Client → Server |
| `session.get` | Get session metadata | Client → Server |
| `session.inject_message` | **Inject message into running session** | Client → Server |
| `tools.call` | Call a registered tool by name | Client → Server |
| `tools.list` | List available tools | Client → Server |
| `config.get` / `config.set` | Read/write profile config | Client → Server |
| `kanban.*` | Board operations (create, list, claim, complete, etc.) | Client → Server |
| `mcp.servers.list` / `mcp.tools.list` | MCP server/tool discovery | Client → Server |
| `approval.respond` | Answer pending approval | Client → Server |
| `delegate_task` | Spawn delegation subagent | Client → Server |

### 2.3 Can External Client Inject Tool Result / Message?
**YES** — `session.inject_message` exists (`tui_gateway/methods_session.py`, `tui_gateway/server.py` handler). It accepts:
```json
{
  "jsonrpc": "2.0",
  "method": "session.inject_message",
  "params": {
    "session_key": "<gateway_session_key>",
    "content": "Tool result or note",
    "role": "user" | "assistant" | "system",
    "plugin_id": "optional"
  },
  "id": 1
}
```
- **Stability**: Used by Desktop/TUI internally. **Not documented as a public API** — no versioning guarantees. Method signatures can change between releases.
- **Auth**: WS connection requires `Authorization: Bearer <token>` where token is from `hermes auth` (session cookie / dashboard token). No mTLS, no API keys for external apps.

### 2.4 Fitness for Orchestrator Integration
| Criterion | Assessment |
|-----------|------------|
| **Supported/stable** | ❌ Internal; no stability guarantee |
| **Push results into conversation** | ✅ `session.inject_message` works |
| **Subscribe to events** | ✅ `session.subscribe` streams tool calls, approvals, messages |
| **Auth story** | ⚠️ Dashboard token / session cookie; not designed for service-to-service |
| **Multi-session** | ✅ Each `session_key` isolates a conversation |
| **Recommendation** | **Secondary** — usable for prototyping, but not a supported integration surface. |

---

## Surface 3: OpenAI-Compatible HTTP API Server (surface, auth, fitness)

### 3.1 What Exists
- **Proxy server**: `hermes_cli/proxy/server.py:194–271` — aiohttp app on **port 8645** (default), mounts at `/v1/{tail:.*}`.
- **Upstream**: Forwards to a configured upstream OpenAI-compatible endpoint (Nous, xAI, custom) with credential attachment.
- **Allowed paths**: Defined by `UpstreamAdapter.allowed_paths` (typically `/chat/completions`, `/models`, `/embeddings`, `/responses`).
- **Auth**: **None for inbound** — protected by `local_only` middleware (`proxy/server.py:209–223`) which:
  - Accepts only loopback Host (`localhost`, `127.0.0.1`, `::1`) unless bound to `0.0.0.0` (wildcard).
  - Rejects browser-origin requests via `Origin` + `Sec-Fetch-Site` checks.
  - **No API key / bearer token validation on inbound** — trust is network-level (loopback only).
- **Streaming**: Full SSE streaming passthrough (`_stream_back`, lines 162–191), including auto-append `[DONE]` frame.

### 3.2 Intended Use
- **Local development proxy** — attaches your Hermes-managed credentials to requests from a local client (e.g. `curl`, OpenAI SDK pointed at `http://localhost:8645/v1`).
- **NOT** a general-purpose "Hermes as an OpenAI-compatible agent server" — it does not run tools, maintain sessions, or expose Hermes tools.

### 3.3 Fitness for Orchestrator Integration
| Criterion | Assessment |
|-----------|------------|
| **Exposes Hermes tools** | ❌ No — pure credential-forwarding proxy |
| **Session continuity** | ❌ No — stateless forwarder |
| **Tool-call streaming** | ✅ Upstream SSE passed through |
| **Auth for external apps** | ❌ Loopback-only by design; wildcard bind = explicit LAN opt-in, still no API keys |
| **Recommendation** | **Not suitable** — this is a dev proxy, not an agent API. |

---

## Surface 4: MCP (Client / Server)

### 4.1 Hermes as MCP Client (Orchestrator as MCP Server)
**Config format** (`hermes_cli/mcp_config.py:210`, `config.yaml`):
```yaml
mcp_servers:
  my-orchestrator:
    command: "python"          # stdio transport
    args: ["-m", "orchestrator.mcp_server"]
    # OR HTTP/SSE transport:
    # url: "http://localhost:9000/mcp"
    # headers:
    #   Authorization: "Bearer ${MCP_MY_ORCHESTRATOR_API_KEY}"
    tools:
      include: ["plan", "execute", "query"]  # optional whitelist
      exclude: ["dangerous_tool"]             # optional blacklist
```
- **Discovery**: On startup, Hermes probes each server (`tools/mcp_tool_discovery.py`), registers tools under toolset `mcp:<server_name>`.
- **Filters**: `tools.include` / `tools.exclude` (arrays of tool names). `include: []` blocks all.
- **Auth**: Header templates with `${ENV_VAR}` resolved from `~/.hermes/.env` (`mcp_config.py:327–342`).
- **OAuth**: Device-code flow for remote servers (`tools/mcp_oauth.py`, `web_routers/mcp.py`).

**Fitness**: ✅ **Excellent fit for "orchestrator behind Hermes"**. The orchestrator exposes tools (plan, execute, query, delegate) via MCP stdio or HTTP. Hermes calls them as any other tool. Per-server allowlist in plugins (`plugins.entries.<id>.mcp_allowlist`) gates which plugins can call MCP.

### 4.2 Hermes as MCP Server (Exposing Hermes Tools to External Clients)
**STATUS: DOES NOT EXIST**. 
- Search of codebase (`grep -r "MCPServer\|serve.*mcp\|mcp_server"` across `tui_gateway/`, `hermes_cli/`, `agent/`) — **no MCP server implementation**.
- Hermes registers as MCP **client** only. The dashboard manages MCP server *configurations* (`web_routers/mcp.py`), but does not serve MCP.

**Implication**: External clients (VS Code, Claude Desktop, custom agents) **cannot** connect to Hermes via MCP to use its tools.

### 4.3 Which Direction Fits "Orchestrator Behind Hermes"?
| Direction | Fits? | Why |
|-----------|-------|-----|
| **Orchestrator → MCP Server, Hermes → MCP Client** | ✅ **YES** | Native, supported, per-server tool filters, credential management, plugin allowlists. |
| Hermes → MCP Server, Orchestrator → MCP Client | ❌ NO | Hermes has no MCP server. |

---

## Surface 5: Delegation Provider + Kanban (interfaces, fitness)

### 5.1 Delegation Provider Interface
**ABC**: `agent/delegation_provider.py:62` — `DelegationProvider`
```python
class DelegationProvider(ProviderBase):
    @property
    def summary(self) -> str: ...                    # one-line for `/delegation` picker
    def is_available(self) -> bool: ...              # MUST NOT make network calls
    def task_parameters(self, config) -> dict: ...   # extra JSON-schema props for delegate_task
    def description(self, config) -> str: ...        # appended to delegate_task description
    @abc.abstractmethod
    def route_task(self, task: Mapping, config: Mapping) -> dict: ...
```
**Route vocabulary** (returned by `route_task`, `delegation_provider.py:15–30`):
```python
{
    "model": str, "provider": str, "base_url": str, "api_key": str,
    "api_mode": str, "request_overrides": dict, "reasoning_effort": str,
    "fallback_providers": list, "service_tier": "normal|fast|ultrafast",
    "profile": str,          # Hermes profile name — child runs AS that profile
    "label": str             # short text shown with child progress
}
```
**Registration**: `PluginContext.register_delegation_provider(provider)` → `agent/delegation_registry.py:51` (via `plugins_registrars.py:85`).
**Config selection**: `delegation.backend: <name>` in `config.yaml` selects active provider. `delegation.backends.<name>` holds provider-specific settings.

**Fitness for orchestrator**: 
- Orchestrator can register a `DelegationProvider` that routes `delegate_task` calls to itself (via HTTP, message queue, or in-process).
- The `profile` route is powerful: child runs as a **different Hermes profile** (separate config, model, skills, SOUL.md). Orchestrator could spin up a profile per task type.
- **Limitation**: Provider only decides *routing*; core owns spawn, isolation, progress, results. Orchestrator cannot intercept the child's execution stream.

### 5.2 Kanban Board
**Storage**: SQLite `kanban.db` per profile (`tools/kanban_db.py`).
**Tools**: `tools/kanban_tools.py` — `kanban_create`, `kanban_list`, `kanban_claim`, `kanban_complete`, `kanban_block`, `kanban_schedule`, `kanban_heartbeat`, `kanban_attach`, `kanban_comment`, etc.
**Gating**: 
- Only available when `HERMES_KANBAN_TASK` env var set (dispatcher worker) **OR** profile has `kanban` toolset enabled.
- `kanban_list` / `kanban_unblock` (orchestrator tools) hidden from task workers (`_check_kanban_orchestrator_mode`).
- Dispatcher sets `HERMES_KANBAN_TASK`, `HERMES_KANBAN_RUN_ID`, `HERMES_KANBAN_BOARD` on worker spawn.

**Fitness for orchestrator**:
- ✅ **Built-in task queue with persistence, heartbeats, retries, priorities**.
- ✅ **Multi-profile coordination** — board lives in profile's `kanban.db`; dispatcher can spawn workers across profiles.
- ❌ **Not a general scheduler** — tightly coupled to `delegate_task` / dispatcher architecture. No Cron-like API for arbitrary jobs.
- ❌ **No external API** — only accessible via tools (which require a running Hermes session).

---

## Surface 6: Session Continuity Mechanics (supported vs unsupported ways to record executions)

### 6.1 Where Session History Lives
- **Primary**: `state.db` (SQLite) per profile at `~/.hermes/profiles/<name>/sessions/state.db`.
- **Schema**: `sessions` table (id, parent_session_id, session_key, source, title, model, provider, cwd, git_repo_root, started_at, last_active, ended_at, end_reason, message_count, token_count, cost_usd, model_config JSON, pinned, hidden, archived).
- **Messages**: `messages` table (id, session_id, role, content, tool_calls JSON, tool_results JSON, created_at, metadata JSON).
- **Compression**: `compression_lineage` table tracks compression continuations.

### 6.2 Official Supported Way to Append Executions
**NONE EXISTS** as a public API.

**Internal (unsupported) ways**:
1. **`SessionDB.append_message`** (`hermes_state_messages.py:429`) — requires `SessionDB` instance (profile-scoped, not thread-safe across processes).
   ```python
   db.append_message(
       session_id=uuid,
       role="assistant",
       content="Orchestrator completed task X",
       tool_calls=[...],      # optional
       tool_results=[...],    # optional
       metadata={"orchestrator": True, "task_id": "..."}
   )
   ```
2. **`PluginContext.post_message` / `inject_message`** — only works **during a live gateway session** (tool call or command). Cannot append to historical sessions from a background process.
3. **`hermes sessions import` CLI** — imports JSONL export (`web_routers/sessions.py:56–121`). Batch, not real-time.

### 6.3 Critical Finding
> **There is no supported, stable API for an external system to record its executions into a Hermes session so the user can "inspect previous executions and continue working from the same conversation."**

The only viable path today:
- **Run orchestrator work *inside* a Hermes session** (via plugin tool, delegation, or MCP tool) so Hermes natively records the tool calls/results.
- **Or** accept that orchestrator executions are invisible in Hermes history unless manually imported.

### 6.4 What Hermes Already Provides (Do Not Duplicate)
| Feature | Hermes Owns It | Orchestrator Should Not Reimplement |
|---------|----------------|--------------------------------------|
| Session history (messages, tools, costs) | ✅ `state.db` | ❌ Don't build parallel history |
| Conversation compression / summarization | ✅ `hermes_state_compression.py` | ❌ |
| Memory (MEMORY.md, USER.md) | ✅ `tools/memory_tool.py` | ❌ |
| Skills (SKILL.md, tool definitions) | ✅ `tools/skills_tool.py` | ❌ |
| Model routing / provider abstraction | ✅ `models.py`, `providers.py` | ❌ |
| Approval gates (write_approval, command approval) | ✅ `tools/approval.py`, `write_approval.py` | ❌ |
| MCP client management | ✅ `mcp_config.py`, `mcp_tool_*.py` | ❌ |
| Kanban task coordination | ✅ `kanban_tools.py`, `kanban_db.py` | ❌ |
| Profile isolation (config, credentials, skills) | ✅ `profiles.py`, `config.py` | ❌ |
| Cron / scheduled tasks | ✅ `cron/scheduler_provider.py` | ❌ |

---

## Surface 7: Approval Propagation

### 7.1 How Hermes Surfaces Approval
- **Command approval** (`tools/approval.py`): Pattern-matched commands (shell, file writes, etc.) trigger `pre_approval_request` hook → prompt user (CLI/TUI/Desktop) → `post_approval_response` hook.
- **Write approval** (`tools/write_approval.py:74`): `memory.write_approval` / `skills.write_approval` config gates stage pending writes to `~/.hermes/pending/{memory,skills}/<id>.json` for out-of-band review.
- **MCP tool approval**: Not a separate gate — MCP tools are regular tools; approval applies if the tool matches a pattern.

### 7.2 Can External Service Request Approval Through Hermes?
**YES, via `pre_tool_call` hook veto** (`plugins.py:1912–1984`):
```python
# In a plugin hook callback for pre_tool_call:
if tool_name == "my_orchestrator_tool" and args.get("needs_approval"):
    return {"action": "block", "message": "Orchestrator requests approval for X"}
```
- The block message is shown to the user in the approval UI.
- User's choice (allow/deny/modify) flows through `post_tool_call` / `post_approval_response` hooks.
- **Limitation**: Only works when the orchestrator action is invoked **as a Hermes tool** (plugin tool, MCP tool, or delegation). A purely external async process cannot inject an approval prompt without a tool call context.

### 7.3 Approval Hook Kwargs (`tools/approval_context.py:66–69`)
```python
kwargs.setdefault("turn_id", _approval_turn_id.get())
kwargs.setdefault("tool_call_id", _approval_tool_call_id.get())
if _approval_session_id.get():
    kwargs.setdefault("session_id", _approval_session_id.get())
```
Plugins receive `turn_id`, `tool_call_id`, `session_id` for correlation.

---

## Recommended Integration Pattern (Ranked)

| Rank | Pattern | Mechanism | Why |
|------|---------|-----------|-----|
| **1** | **Orchestrator as MCP Server** | `mcp_servers` in config.yaml + plugin tool `call_mcp` or direct tool exposure | Native, supported, per-tool filters, credential mgmt, plugin allowlists. Hermes calls orchestrator tools as first-class tools. Session history captured automatically. |
| **2** | **Plugin Tool + Delegation Provider** | Custom plugin registers `orchestrator_*` tools + `DelegationProvider` for `delegate_task` routing | Full in-process control; `post_message` for streaming updates; `profile` route spins isolated child profiles. Requires plugin development. |
| **3** | **Plugin Tool + Kanban** | Plugin registers tools that create/claim Kanban tasks; dispatcher workers (or orchestrator) execute | Built-in persistence, heartbeats, retries, multi-profile. Good for long-running async work. |
| **4** | **TUI Gateway JSON-RPC** | External client connects WS → `session.inject_message`, `session.subscribe` | Works for real-time UI embedding. **Not stable/supported**; auth is dashboard-token only. |
| **5** | **OpenAI-compatible Proxy** | Point orchestrator at `localhost:8645/v1` | **Wrong direction** — this is Hermes→upstream, not external→Hermes. |

### Recommended: Pattern 1 (MCP Server) + Pattern 2 (Plugin) Hybrid
1. **Orchestrator exposes MCP server** (stdio or HTTP/SSE) with tools: `plan`, `execute`, `query_status`, `delegate_subtask`.
2. **Hermes config** adds it to `mcp_servers` with `tools.include` allowlist.
3. **Optional plugin** (`plugins/orchestrator-bridge/`) registers:
   - `orchestrator_plan` tool (thin wrapper over MCP `plan`) for slash-command UX.
   - `DelegationProvider` that routes `delegate_task` → orchestrator's `delegate_subtask` (via MCP or HTTP).
   - `pre_tool_call` hook for approval propagation.
   - `post_message` / `inject_message` for streaming progress into conversation.
4. **Session continuity**: All tool calls/results recorded natively in `state.db` because they run *as Hermes tools*.

---

## What Hermes Already Provides (Do Not Duplicate)

| Capability | Module | Notes |
|------------|--------|-------|
| Session persistence (SQLite) | `hermes_state_sessions.py`, `hermes_state_messages.py` | Per-profile `state.db` |
| Conversation compression | `hermes_state_compression.py` | Automatic, configurable |
| Cross-session memory (MEMORY.md/USER.md) | `tools/memory_tool.py` | Scoped to profile |
| Skills (SKILL.md + files) | `tools/skills_tool.py`, `tools/skill_manager_tool.py` | Versioned, importable |
| Model/provider routing | `models.py`, `providers.py`, `runtime_provider.py` | 20+ providers, API mode detection |
| MCP client (discovery, auth, tool reg) | `mcp_config.py`, `mcp_tool_*.py` | OAuth, stdio, HTTP/SSE |
| Kanban task board | `kanban_tools.py`, `kanban_db.py` | Multi-profile, dispatcher workers |
| Delegation/subagents | `delegate_task` tool, `delegation_provider.py` | Profile-aware, depth/conc limits |
| Approval gates (commands, writes) | `tools/approval.py`, `write_approval.py` | Pattern-based, staged pending store |
| Cron/scheduled tasks | `cron/scheduler_provider.py` | In-process or external (Redis) |
| Multi-profile isolation | `profiles.py`, `config.py` | Separate home, config, skills, sessions |
| Plugin system (tools, hooks, providers) | `plugins.py`, `plugins_registrars.py` | Host isolation, capability grants |
| TUI/Desktop gateway (WS + JSON-RPC) | `tui_gateway/` | Multi-session, platform adapters |
| Web dashboard (FastAPI) | `web_server.py`, `web_routers/` | Session mgmt, config, MCP, logs |

---

## Open Integration Questions

1. **Session append API**: Will Hermes expose a supported `POST /api/sessions/{id}/messages` or `SessionDB.append_message` as a public SDK function? (Currently internal only.)
2. **MCP Server on Hermes**: Roadmap for Hermes exposing its tools via MCP? (Currently client-only.)
3. **Plugin tool streaming**: Any plan for `handler` yielding chunks (`AsyncGenerator[str, None]`) for real-time model streaming?
4. **Delegation provider streaming**: Can a provider return a stream/async iterator for child progress, or is `route_task` strictly synchronous routing?
5. **Gateway injection auth**: Can `session.inject_message` accept a service API key instead of dashboard token? (Currently no.)
6. **Kanban external API**: Will `kanban.db` get a supported HTTP/JSON-RPC interface for external schedulers?
7. **Profile-scoped MCP allowlist**: `plugins.entries.<id>.mcp_allowlist` is profile-scoped — can an orchestrator plugin get cross-profile MCP access?

---

## Evidence (Every File Path and URL)

### Core Source Files (Inspected)
| File | Lines | Purpose |
|------|-------|---------|
| `/home/deval/.hermes/hermes-agent/hermes_cli/plugins.py` | 1–2163 | Plugin system, `PluginContext`, `register_tool`, `register_hook`, `register_delegation_provider`, `inject_message`, `post_message`, `call_mcp`, `VALID_HOOKS` |
| `/home/deval/.hermes/hermes-agent/agent/delegation_provider.py` | 1–110 | `DelegationProvider` ABC, route vocabulary |
| `/home/deval/.hermes/hermes-agent/agent/delegation_registry.py` | 1–134 | Provider registry, `register_delegation_provider`, `get_active_provider` |
| `/home/deval/.hermes/hermes-agent/agent/delegation_backends.py` | 1–200 | Built-in backends, `list_backends`, `set_global_backend` |
| `/home/deval/.hermes/hermes-agent/hermes_cli/plugins_registrars.py` | 1–135 | `register_delegation_provider` registrar wiring |
| `/home/deval/.hermes/hermes-agent/plugins/delegation/default/plugin.yaml` | 1–15 | Example plugin manifest |
| `/home/deval/.hermes/hermes-agent/plugins/delegation/default/__init__.py` | 1–50 | Bundled provider registration |
| `/home/deval/.hermes/hermes-agent/tui_gateway/server.py` | 1–1000+ | JSON-RPC WS server, method registration |
| `/home/deval/.hermes/hermes-agent/tui_gateway/rpc_dispatch.py` | 1–200 | `JsonRpcDispatcher` |
| `/home/deval/.hermes/hermes-agent/tui_gateway/methods_session.py` | 1–100 | `session.inject_message`, `session.submit`, `session.subscribe` |
| `/home/deval/.hermes/hermes-agent/hermes_cli/proxy/server.py` | 1–303 | OpenAI-compatible forwarder on :8645 |
| `/home/deval/.hermes/hermes-agent/hermes_cli/mcp_config.py` | 1–1170 | MCP server config, `_get_mcp_servers`, tool filters |
| `/home/deval/.hermes/hermes-agent/hermes_state_sessions.py` | 1–1981 | `SessionDB` lifecycle, `state.db` schema |
| `/home/deval/.hermes/hermes-agent/hermes_state_messages.py` | 1–1000+ | `append_message`, message schema |
| `/home/deval/.hermes/hermes-agent/hermes_cli/plugins_injection.py` | 1–70 | `inject_plugin_message` routing |
| `/home/deval/.hermes/hermes-agent/tools/approval_context.py` | 1–385 | Approval contextvars, `pre_approval_request`, `post_approval_response` |
| `/home/deval/.hermes/hermes-agent/tools/write_approval.py` | 1–413 | Staged pending writes for memory/skills |
| `/home/deval/.hermes/hermes-agent/tools/kanban_tools.py` | 1–1315 | Kanban tool implementations |
| `/home/deval/.hermes/hermes-agent/tools/kanban_db.py` | (not fully read) | SQLite `kanban.db` schema |
| `/home/deval/.hermes/hermes-agent/hermes_cli/lifecycle.py` | 1–97 | `invoke_hook`, `ainvoke_hook`, hook dispatch |
| `/home/deval/.hermes/hermes-agent/hermes_cli/web_routers/sessions.py` | 1–991 | Session REST API (list, prune, import, mutate) |
| `/home/deval/.hermes/hermes-agent/hermes_cli/web_routers/delegation.py` | 1–262 | Delegation config API |
| `/home/deval/.hermes/hermes-agent/hermes_cli/web_routers/mcp.py` | 1–538 | MCP dashboard routes |

### Official Docs (URLs)
| URL | Topic |
|-----|-------|
| `https://hermes-agent.nousresearch.com/docs/developer-guide/programmatic-integration` | Programmatic integration overview |
| `https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins` | Plugin system user guide |
| `https://hermes-agent.nousresearch.com/docs/developer-guide/plugins` | Plugin development guide |
| `https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp` | MCP client usage |
| `https://hermes-agent.nousresearch.com/docs/developer-guide/native-mcp` | MCP internals |
| `https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation` | Delegation/subagents |
| `https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban` | Kanban board |
| `https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage` | Session DB internals |
| `https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks` | Hook system reference |

> **Note**: All source claims above are **VERIFIED-IN-SOURCE** with exact file paths and line numbers. Docs URLs are **DOCS-CLAIMED** — not fetched in this analysis. No invented method names or APIs.