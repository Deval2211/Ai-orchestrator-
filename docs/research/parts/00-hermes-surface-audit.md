# Hermes integration surface audit (source-verified)

Researcher's own verification, independent of the delegated deep-dives.
Research date: 2026-11-09. Local Hermes source at `/home/deval/.hermes/hermes-agent`.

## Hermes architecture facts (VERIFIED-IN-SOURCE)

| Fact | Evidence |
|------|----------|
| `PluginContext.register_tool(name, toolset, schema, handler, check_fn=None, requires_env=None, is_async=False, description="", emoji="", override=False)` | `hermes_cli/plugins.py:463` |
| `ctx.llm.complete(messages=..., max_tokens=..., purpose=...)` — plugin LLM calls, host-owned credentials, trust gate, JSON-Schema validation | https://hermes-agent.nousresearch.com/docs/developer-guide/plugin-llm-access |
| Delegation provider ABC: `route_task()` returns plain-JSON route (`model`, `provider`, `base_url`, `api_key`, `api_mode`, `request_overrides`, `reasoning_effort`, `service_tier`, `fallback_providers`, `profile`, `label`) | `agent/delegation_provider.py:35` (ROUTE_KEYS) |
| `register_delegation_provider` is a first-class plugin registrar; `delegation.backend` in config selects the active one | `hermes_cli/plugins_registrars.py:85` |
| Kanban dispatch accepts a plugin-supplied `spawn_fn(task, workspace, board=board) -> Optional[int]` | `hermes_cli/kanban_db_dispatch.py:1983`, `_call_spawn_fn` at `:2039` |
| Kanban board is a durable SQLite DB at `~/.hermes/kanban.db`; every task is a row | https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban |
| Kanban worker lanes are explicitly designed for "a CLI worker that wraps Codex / Claude Code / OpenCode, a containerised review worker, a non-Hermes service that pulls tasks via the API" | https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-worker-lanes |
| Kanban lifecycle terminators: `kanban_complete`, `kanban_request_review`, `kanban_request_changes`, `kanban_block`, `kanban_unblock` | kanban-worker-lanes doc |
| Kanban kernel reaps workers and emits `crashed` / `gave_up` / `timed_out`; a worker that calls no terminator and exits is treated as crashed | kanban-worker-lanes doc |
| Three external protocols: ACP (JSON-RPC/stdio), TUI gateway JSON-RPC (stdio or WebSocket, `hermes-gateway-v1`), API server (HTTP + SSE) | https://hermes-agent.nousresearch.com/docs/developer-guide/programmatic-integration |
| `hermes acp` exposes: session creation, prompt submission, streaming agent chunks, tool-call events, permission requests, session fork, cancel, auth | programmatic-integration doc |
| Plugin manifest format: `plugin.yaml` with `manifest_version`, `name`, `kind`, `version`, `requires_hermes`, `description`, `license`, `tags`, `capabilities` | `~/.hermes/plugins/opencode-free-acp/plugin.yaml` |
| Four hook systems: gateway hooks (`~/.hermes/hooks/`), plugin hooks (`ctx.register_hook()`), shell hooks (`hooks: block`), outbound webhooks (`hooks.outbound`) | https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks |

## What Hermes ALREADY provides — do not rebuild

This is the most consequential finding of the whole research.

### Credential pools — same-provider multi-account rotation (VERIFIED)
> "Credential pools let you register multiple API keys or OAuth tokens for the same provider.
> When one key hits a rate limit or billing quota, Hermes automatically rotates to the next
> healthy key — keeping your session alive without switching providers."

Strategy names: `round_robin`, `least_used`, `fill_first`, `random`.
Rotation semantics are already implemented and documented: 429-with-reset → bench that key;
transient 429 → retry same key once, then rotate; 402 → rotate with 1h cooldown; 401 → refresh
token, then rotate; 400 model-not-supported → bench that key for that model only.

Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/credential-pools

**Impact on the project:** the spec's "Account Manager" (per-account auth, limits, usage counters)
and "automatic fallback" overlap this heavily. The orchestrator's own account rotation for LLM API
providers is likely duplicate work.

### Fallback providers — cross-provider failover (VERIFIED)
`fallback_providers:` list in config.yaml, each entry `{provider, model}`. Rate-limited primary
benched until its declared reset; `fallback.min_switch_reset_seconds` to avoid thrashing.
Auxiliary task fallback is a separate layer.

Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/fallback-providers

**Impact:** the spec's escalation ladder ("retry with a stronger resource", "fallback to next
resource") overlaps this for the API-provider case.

### Provider routing — cost/latency/throughput ordering (VERIFIED)
`provider_routing.sort:` accepts `price`, `throughput`, `latency`; plus `only`, `ignore`, `order`,
`require_parameters`, `data_collection`. Applies to OpenRouter only.

Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/provider-routing

**Impact:** partial overlap with "cost-aware routing" in the System 1 router spec — but only
within OpenRouter's provider set, and it is a provider-preference knob, not a task classifier.

### Kanban — durable task board, parallel workers, reviewer gate, recovery (VERIFIED)
- Durable SQLite board at `~/.hermes/kanban.db`, shared across Hermes profiles.
- 14 `kanban_*` tools: `kanban_show`, `kanban_list`, `kanban_complete`, `kanban_request_review`,
  `kanban_request_changes`, `kanban_block`, `kanban_heartbeat`, `kanban_comment`,
  `kanban_attach`, `kanban_attach_url`, `kanban_attachments`, `kanban_create`, `kanban_link`,
  `kanban_unblock`.
- Per-task pinned workspaces (`HERMES_KANBAN_WORKSPACE`, `HERMES_KANBAN_WORKSPACES_ROOT`) — this
  is a built-in isolation primitive for parallel agents.
- Circuit breaker on consecutive failures; `max_runtime` timeout; crash detection via PID.
- Reviewer gate: `kanban_request_review` → reviewer approves (`kanban_complete`), returns rework
  (`kanban_request_changes`), or blocks (`kanban_block`).
- Completion checkpoint near 90% of the iteration budget (`agent.budget_warning_ratio`).
- Descendant process fencing (`agent.delegation_context.delegated_child_subprocess_env`).

Sources: https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban and
.../kanban-worker-lanes

**Impact:** this is the single biggest overlap. The spec's "Task Scheduler" (queue, parallelize,
checkpoint), "Checkpoint & Recovery" (save state, resume after failure) and "Verification Engine"
(re-check outputs, flag mistakes, trigger rework) all have direct Hermes counterparts — including
the reviewer/rework loop, which is exactly "verification triggers rework".

**The decisive detail:** the Kanban worker-lane contract explicitly anticipates "a CLI worker that
wraps Codex / Claude Code / OpenCode" and "a non-Hermes service that pulls tasks via the API".
That is an existing, supported seam for an external orchestrator to drive coding CLIs — the
project's Phase 3/4 execution layer may not need to be built at all.

### Delegation — parallel subagents with audit (VERIFIED)
`delegate_task` spawns isolated child agents; the delegation-provider plugin interface lets a
plugin decide the route per task. Core keeps spawn, isolation, roles, depth, concurrency,
background units, stop fan-out, results, `output_schema` and progress.

Source: `agent/delegation_provider.py`, https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation

**Impact:** overlaps the spec's Phase 6.1 "Parallel Agents" and Phase 6.3 "Multi-stage
verification ... automated rework loop".

## Integration surfaces ranked

### Rank 1 — Kanban worker lane (external CLI worker) — BEST FIT
**Shape:** the orchestrator runs as a service; a Hermes plugin registers a worker lane whose
`spawn_fn(task, workspace, board=board)` launches the orchestrator's CLI adapter for that task;
the worker returns results through the `kanban_*` tools.

Why it fits:
- The lane contract was designed for exactly this shape (CLI workers wrapping Codex/Claude
  Code/OpenCode; non-Hermes services pulling tasks via the API).
- Kanban owns the task lifecycle, workspaces, parallel dispatch, reviewer gate, crash/timeout
  reaping, and the audit trail — all things the spec wants the orchestrator to build.
- The orchestrator keeps its own resource registry / router / account logic; it does NOT
  reimplement task lifecycle.
- The user still sees everything through Hermes (kanban board + conversation).

Cost: the orchestrator worker must terminate its run with exactly one `kanban_complete` /
`kanban_request_review` / `kanban_block`, and must not write to the board directly.

### Rank 2 — Plugin tool (synchronous request/response)
**Shape:** `ctx.register_tool("orchestrate", toolset="orchestrator", schema=..., handler=...)`;
the handler POSTs to the orchestrator's HTTP API and returns the result.

Fits the simple path: user asks Hermes → Hermes calls one tool → orchestrator picks a resource
and executes. Verified API: `hermes_cli/plugins.py:463`. `ctx.llm` available if the router itself
needs an LLM call.

Weakness: the whole task blocks one tool call. Streaming progress back into the conversation is
limited to what the tool returns (plus `ctx.register_hook()` for observability).

### Rank 3 — MCP server (orchestrator exposes tools to Hermes)
Hermes already speaks MCP as a client (`mcp_servers:` in config.yaml, with per-plugin
`mcp_allowlist` and tool filters). The orchestrator would publish an MCP server exposing
`orchestrate`, `list_resources`, `get_execution`, `resume_execution`.

Fits: clean, language-agnostic, standard. Also gives the orchestrator's registry a natural
public API.

Weakness: MCP is request/response over stdio or one HTTP server — it does not carry Hermes's
conversation history, so session continuity has to be solved separately anyway.

### Rank 4 — Delegation provider plugin
**Shape:** register a `DelegationProvider` whose `route_task()` returns a `profile` route, so
`delegate_task` spawns the child as a specific Hermes profile wired to the orchestrator.

Fits: Hermes keeps the subagent lifecycle; the orchestrator decides WHERE work goes.
Weakness: the route vocabulary is model/provider/profile-shaped. It cannot express "send this
task to Kiro CLI account 2". A delegation provider picks a route, not an arbitrary execution
backend. **This is a real limitation, not a solution.**

### Rank 5 — TUI gateway JSON-RPC / API server / ACP
These drive Hermes FROM an external client. That is the opposite direction from what the project
needs (Hermes primary, orchestrator behind). Not applicable as the primary seam, though the
gateway JSON-RPC is relevant if the orchestrator needs to push async completion events back into
a running Hermes session — that needs separate verification (see below).

## CRITICAL open question — session continuity

The spec requires: "I can inspect previous executions and continue working from the same
conversation" and "Session Sync records the full history into the Hermes conversation."

Hermes stores sessions in `~/.hermes/state.db` (SQLite + FTS5). The brief explicitly says:
"Do not recommend directly modifying Hermes's internal session database without establishing that
it is a supported integration method."

**Finding: writing into a Hermes session from an external service is NOT a documented supported
operation.** The plugin API exposes `ctx.llm` and tool registration, but no documented
"append a message to session N" API was found. Kanban gives a supported alternative — the
`kanban_comment` / `kanban_attach` / `kanban_attach_url` tools write durable records the user can
read back through Hermes, and the board IS the unified history. That reframes "session
continuity" from "inject into the Hermes transcript" to "keep the authoritative record in the
orchestrator's own store, and surface it into Hermes via supported Kanban/Plugin channels."

This must be decided before implementation — it changes the architecture.

## Approval propagation (PARTIALLY VERIFIED)

- ACP exposes `permission requests` to the IDE client (documented).
- Hermes has an approval transport registrar (`register_approval_transport`, `hermes_cli/plugins.py:457`)
  and `tools/approval.py` smart approvals.
- A plugin tool can request approval via the registry's `check_fn` parameter on `register_tool`.
- **Not verified:** whether an out-of-process service (the orchestrator) can raise an approval
  request in a Hermes conversation. The in-process plugin path can; the cross-process path needs
  confirmation.
