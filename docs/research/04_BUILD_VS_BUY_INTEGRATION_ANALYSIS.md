# 04 — Build vs Buy vs Integrate

For every component in the project: what already exists, what to reuse, integrate, extend, or
build, and why. Every "reuse" claim is verified against source or official docs.

**Legend:** 🟢 REUSE (use it as-is) · 🔵 INTEGRATE (run it as a service, talk to it over a
supported interface) · 🟣 EXTEND (fork/adapt) · 🔴 BUILD (write it yourself) · ⛔ DROP (cut it)

---

## 1. Executive decision table

| # | Component | Decision | What | Complexity | Licence risk |
|---|---|---|---|---|---|
| 1 | **Hermes integration** | 🔵 INTEGRATE | MCP server + a small Kanban worker-lane plugin | Low | None (our own code) |
| 2 | **Conversation history** | 🟢 REUSE | Hermes `state.db` | — | — |
| 3 | **Task lifecycle / queue / parallelism** | 🟢 REUSE | Hermes Kanban (`~/.hermes/kanban.db`) | — | — |
| 4 | **Verification → rework loop** | 🟢 REUSE | Kanban review + `sdlc-review` skill | — | — |
| 5 | **Checkpoint / recovery (task level)** | 🟢 REUSE | Kanban crash/timeout/give-up handling + respawn | — | — |
| 6 | **Multi-account key rotation** | 🟢 REUSE | Hermes credential pools | — | — |
| 7 | **Cross-provider failover** | 🟢 REUSE | Hermes `fallback_providers` | — | — |
| 8 | **Provider cost/latency preference** | 🟢 REUSE | Hermes `provider_routing.sort` | — | — |
| 9 | **Resource registry** | 🔴 BUILD | Ours. Small. | Low | None |
| 10 | **Capability model (ordered tiers)** | 🔴 BUILD | Ours. Small. | Low | None |
| 11 | **System 1 router** | 🔴 BUILD | Ours, but small. | Medium | None |
| 12 | **Adapter layer (LLM APIs)** | 🔴 BUILD | Thin; httpx + provider adapters | Low | None |
| 13 | **Adapter layer (coding CLIs)** | 🔴 BUILD | Thin; each CLI has its own native interface | Medium | Per-CLI |
| 14 | **Account health / quota tracking** | 🔴 BUILD | Ours. Small. | Low | None |
| 15 | **Execution store** | 🔴 BUILD | SQLite, ours | Low | None |
| 16 | **API gateway (proxy/routing layer)** | ⛔ DROP | Hermes already routes + rotates | — | LiteLLM licence unresolved |
| 17 | **CLI terminal runtime** | 🔵 INTEGRATE (optional) | herdr-core sidecar over JSON-RPC | Medium | Apache-2.0 |
| 18 | **Multi-agent fan-out** | 🟢 REUSE | Hermes `delegate_task` | — | — |
| 19 | **Browser-based AI** | ⛔ DROP | Defer past the MVP | — | ToS exposure |
| 20 | **Local model serving** | ⛔ DROP | Forbidden by constraint; not installed | — | — |

**Net effect:** the spec's 8 components become **6 small ones**, all of them about *resource
intelligence*. Everything about *task orchestration* is delegated to Hermes.

---

## 2. Hermes integration

**Options evaluated**

| Option | Verdict |
|---|---|
| **MCP server** (orchestrator serves, Hermes calls) | ✅ **Recommended.** Native config (`mcp_servers:`), per-server tool filters, `${ENV_VAR}` auth headers, plugin `mcp_allowlist`. Hermes has **no MCP server**, so this is the only viable direction. |
| **Plugin tool** | ✅ Use *in addition*, for the in-session interactive path. `ctx.register_tool(name, toolset, schema, handler, check_fn, requires_env, is_async, ...)`. |
| **Kanban worker lane** | ✅ **Recommended** for long-running work. The lane contract explicitly targets "a CLI worker that wraps Codex / Claude Code / OpenCode" and "a non-Hermes service that pulls tasks via the API". Plugin supplies `spawn_fn(task, workspace, board=board)`. |
| **Delegation provider plugin** | ❌ Wrong shape. The route vocabulary is `model`/`provider`/`base_url`/`api_key`/`api_mode`/`request_overrides`/`reasoning_effort`/`service_tier`/`fallback_providers`/`profile`/`label` — it selects *where a Hermes child runs*, not an arbitrary execution backend. It cannot express "run Kiro CLI account 2". |
| **TUI gateway JSON-RPC** | ❌ Wrong direction — drives Hermes *from* an external client. |
| **ACP / OpenAI-compatible API server** | ❌ Same — Hermes as server. |
| **Writing to `state.db`** | ❌ No supported API. `SessionDB.append_message` is internal; `PluginContext.post_message` only works during a live gateway session; `hermes sessions import` is batch. The brief forbids this absent proof of support. |

**Decision: 🔵 INTEGRATE** via MCP + a Kanban lane plugin. No Hermes source modification.

**Complexity:** Low. One `config.yaml` block + one ~100-line plugin.

---

## 3. Task lifecycle, queue, parallelism, checkpoints, verification

This is where the spec is heaviest and the reuse is largest.

**What Hermes Kanban already does** (all verified):

| Spec requirement | Hermes equivalent |
|---|---|
| Task Scheduler: "queue, parallelize, checkpoint tasks" | Kanban board — 14 `kanban_*` tools, per-task pinned workspaces, parallel dispatch |
| Checkpoint & Recovery: "save state, resume after failure" | Kanban lifecycle — `crashed` / `gave_up` / `timed_out` events, respawn on `kanban_unblock` |
| Verification Engine: "re-check outputs, flag mistakes, trigger rework" | Kanban review — `kanban_request_review` → `kanban_complete` (approve) / `kanban_request_changes` (rework, routes back to the implementer) / `kanban_block` |
| "Everything is checkpointed" (architecture.md rule 5) | Kanban writes every task as a durable row; workspaces are pinned per task |
| Escalation on failure | Kanban's consecutive-failure circuit breaker + `gave_up` |

**Decision: 🟢 REUSE** for all five. Delete these four components from the spec.

**What the orchestrator still keeps** (the part Kanban cannot do): a *resource-level* checkpoint —
which CLI session id, which worktree, what adapter state — so a killed adapter can resume the
**underlying tool**, not just the card.

**Complexity of reuse:** Zero — it is already installed and working.

**Risk:** if the orchestrator's work runs outside a Hermes session, none of it appears in the
conversation. **Mitigation:** every orchestrator action must be invoked *as* a Hermes tool
(MCP tool call or Kanban lane). Enforce by requiring a correlation id (`task_ref` / Hermes
`tool_call_id`) on every orchestrator API call and rejecting uncorrelated calls.

---

## 4. Account management and API-key rotation

**What Hermes already does:**
- **Credential pools** — register multiple API keys or OAuth tokens for one provider; auto-rotate
  on 429 (with reset-time benching), 402 (1h cooldown), 401 (refresh-then-rotate), and
  400-model-not-supported (bench that key for that model only). Strategies: `round_robin`,
  `least_used`, `fill_first`, `random`.
- **Fallback providers** — cross-provider failover with reset-aware benching and
  `min_switch_reset_seconds`.
- **Provider routing** — `sort: price | throughput | latency`, plus `only` / `ignore` / `order`.

**What Hermes does NOT do:** know that "Kiro account 2 is currently sitting at a browser-login
screen" or "this model has a 32k context but that one has 128k". That is capability + health
metadata — the orchestrator's job.

**Decision:**
- Same-provider key rotation for **LLM APIs** → 🟢 **REUSE** Hermes credential pools.
- Cross-provider failover → 🟢 **REUSE** Hermes fallback providers.
- CLI account isolation (Kiro/OpenCode/Claude Code) → 🔴 **BUILD** (env-var wrappers; ~50 lines).
- Capability + health metadata per account → 🔴 **BUILD** (small table).

**Complexity:** Low.

**Licence risk:** None — this is Hermes's own code, and the orchestrator just points at it.

---

## 5. API gateway — LiteLLM, OmniRoute, alternatives

**The question:** should the orchestrator proxy LLM calls through a gateway, or call providers
directly?

**Arguments against adopting a gateway in the MVP:**

1. **Hermes already does the two things a gateway provides.** Same-provider key rotation and
   cross-provider failover are built into Hermes. A gateway would duplicate them.
2. **The orchestrator's own routing is at a different layer.** A gateway routes *requests to
   models*. The orchestrator routes *tasks to resources* — which may be a model, a coding CLI, or
   (later) a browser service. A gateway cannot express "this task needs Kiro, not an API".
3. **Licence risk.** LiteLLM's `/license` endpoint reports `NOASSERTION`; its actual `LICENSE`
   file must be read and its enterprise/commercial clauses quoted before any adoption. That is
   unresolved — and it is a real blocker for a project that might become commercial.
4. **Operational cost.** Each gateway is another service to run, secure, monitor and keep patched.
   The brief explicitly says not to add infrastructure without justification.
5. **"OmniRoute" does not appear to exist** under that name. The closest real match,
   `musistudio/claude-code-router` (MIT, 38,060 stars), is a different project — a Claude Code
   provider router. The brief's premise here is wrong and should be corrected.

**Arguments for a gateway later:**
- Unified usage/billing across many provider accounts.
- Team-wide virtual keys and budgets.
- Semantic/cost-aware routing across many providers (LiteLLM's router strategies; RouteLLM's
  cost-predictor).

**Decision: ⛔ DROP from the MVP.** Call provider APIs directly from thin adapters. Record
per-account usage in our own `cost_ledger` table. Revisit only if the project grows a
multi-tenant or billing requirement — at which point LiteLLM (Apache-2.0 core, enterprise terms
for some features) is the leading candidate, **after** reading its actual licence.

**Complexity of dropping:** Low — a thin `httpx` adapter per provider is ~40 lines.

---

## 6. Coding CLI execution runtime — herdr-core

**What herdr-core actually is** (verified from source):

- A **terminal/PTY runtime**, not an agent manager. No agent registry, no agent-to-agent
  messaging, no built-in agent configs.
- **32-method JSON-RPC 2.0 API** over a **Unix domain socket**, fully described by an OpenRPC
  document (`hc.openrpc.json`) that is also retrievable at runtime via `rpc.discover`.
- Methods include `terminal.run`, `terminal.send`, `terminal.read`, `terminal.wait`
  (including `--children`), `terminal.attach`, `terminal.keys`, `terminal.interrupt`,
  `terminal.put`, `machine.exec`, `machine.add`, `file.read/write/edit`, `server.status/restart`,
  `events.subscribe`.
- **Keeps terminals alive across client disconnect** — the core daemon outlives the client.
  `server.restart` saves screens while programs keep running.
- **Persistence**: atomic disk writes; terminal state recoverable after a core restart.
- **Licence:** Apache-2.0 for both herdr and herdr-core.
- **Deployable as a sidecar daemon** over the Unix socket — callable from any language.

**What the orchestrator would still have to build on top:** agent registry, agent lifecycle,
inter-agent messaging (can be approximated with `terminal.send` + `terminal.wait --children`),
and agent detection (which lives in the herdr **GUI app**, not the core).

**Evaluation:**

| Scenario | Verdict |
|---|---|
| Terminal lifetime / crash-safety / remote machines / screen capture | herdr-core is excellent — it solves a hard problem well |
| **Driving coding CLIs with structured output and session resume** | **herdr-core is the wrong tool.** It manages *screens*, not protocols. Every CLI in scope already has a native structured interface: OpenCode has an HTTP server + SDK, Kiro has `--output-format stream-json` and ACP, Claude Code has `--output-format stream-json`, Pi has JSON mode + RPC, Cline has `--json` NDJSON. A PTY plus screen-scraping is strictly worse than each tool's own protocol. |
| Using it as a *fallback* for tools with no structured mode | Reasonable — a generic "run this command in a durable PTY and read its output" adapter. |

**Decision: 🔵 INTEGRATE — optional, not core.** Do **not** make herdr-core the execution runtime
for the MVP. Use each CLI's native interface. Keep herdr-core as a candidate for (a) a generic
fallback adapter and (b) remote-machine execution (`machine.add` over SSH, `machine.exec`) if
that ever enters scope.

**Complexity of integrating:** Medium (Rust sidecar + JSON-RPC client + socket lifecycle).
**Licence:** Apache-2.0 — permissive, attribution required, no copyleft. Low risk.

**Alternative that is a better fit:** each CLI's own server/SDK mode —
`opencode serve` + `@opencode-ai/sdk` (best), Pi's RPC/SDK, Claude Code's Agent SDK, and ACP
for Kiro/Cline.

---

## 7. Multica — the closest competitor

**This is the most important competitive finding.** Multica already implements a Hermes backend.

**Verified from source** (`server/pkg/agent/hermes.go`):

- Multica spawns the `hermes` binary and talks to it over **ACP JSON-RPC 2.0 via stdin/stdout**.
- Full session lifecycle: `initialize` → `session/new` → `session/prompt` → `session/resume`
  (with `sessionId`) → `session/set_model`.
- Per-task overlay: `HERMES_HOME` pointed at an overlay path, `HERMES_YOLO_MODE=1` to
  auto-approve tools, per-agent `custom_args`, and the `acp` subcommand blocked from override.
- Multica supports **29 agent runtimes** including Hermes, Claude Code, Codex, OpenCode, Cursor,
  Gemini and Kiro.

**Multica's footprint:** Docker + **PostgreSQL 17** + a Go server binary + a Next.js frontend.
No Redis. Telemetry to `telemetry.multica.ai` (disable with `DO_NOT_TRACK=1`).

**Multica's licence:** Apache-2.0 (commercial use permitted, modification permitted, patent
licence granted, state changes required, no trademark use).

**Assessment:**

| Question | Answer |
|---|---|
| Is anyone building almost exactly this? | **Yes — Multica is close.** It is a multi-runtime agent platform that already drives Hermes over ACP. The differences: it is **team/workspace-oriented** (issues, sub-issues, shared runtimes, PostgreSQL, web UI) rather than a personal resource-intelligence layer; it does not appear to do "smallest capable resource" task routing; and its UX is its own web app, not Hermes-as-primary. |
| Should we use Multica? | **No, not as a dependency.** It is a whole server + Postgres + web app. Embedding it is heavier than building the small thing we actually need. **But its `hermes.go` is the single best reference implementation available** for driving Hermes over ACP from an external service — Apache-2.0, readable Go, and it encodes hard-won details (silent session recreation, stderr error patterns, profile override semantics, `HERMES_HOME` overlays). |
| Should we fork it? | **No.** Forking a 55k-star team platform to build a personal orchestrator is a maintenance trap, and it would make Hermes *one* runtime among 29 instead of the primary interface. |
| Is there a shortcut? | **Yes — and it is worth evaluating.** Multica proves that driving Hermes over ACP is tractable. If the orchestrator wants to run *Hermes itself* as one of its resources (e.g. to delegate a subtask to a Hermes profile), ACP is the protocol to use. That is a genuinely useful capability our MVP should consider. |

**Decision: ⛔ DO NOT ADOPT. 🟣 USE AS REFERENCE.** Read `server/pkg/agent/hermes.go` before
implementing any Hermes-over-ACP path.

---

## 8. OpenClaw

**What it is:** a gateway-centric agent platform (TypeScript). One Gateway process is the control
plane: `Client (WS) → Gateway (WS + HTTP multiplex) → Agent Loop → Model Provider`. HTTP
endpoints `/v1/responses` (OpenResponses), `/v1/chat/completions`, `/v1/models`,
`/v1/embeddings` share the gateway port. MIT licensed. 394,224 stars.

**Why it is not our integration target:** it is a *complete agent platform*, not a
resource-intelligence layer. Adopting it would replace Hermes as the primary interface —
directly contradicting the project's non-negotiable constraint. It also sits in a different
direction: it makes *itself* the gateway, whereas we need to sit *behind* Hermes.

**Hermes's own relationship with it:** Hermes ships a one-command **importer** for OpenClaw
installs (`hermes chat` → import from OpenClaw), i.e. the migration direction is
OpenClaw → Hermes. OpenClaw does not offer the reverse.

**Decision: ⛔ DO NOT ADOPT.** Its gateway-protocol package is a possible reference for
event-streaming shapes, nothing more.

---

## 9. AionUi, T3 Code, CC Switch

| Project | What it actually is | Decision |
|---|---|---|
| **AionUi** (Apache-2.0, 34k★) | An agent **UI**. It drives coding CLIs, but the value is the interface. | ⛔ DO NOT ADOPT. We want Hermes as the UI — AionUi would be a second, competing one. |
| **T3 Code** (MIT, 53k★) | Coding-agent orchestration with **git worktree** management, agent threads, diff inspection, PR workflow. | 🟣 **REFERENCE for worktree handling.** Hermes Kanban already provides per-task workspaces; T3 Code's worktree module is a good model if we need git-isolated parallel workspaces, but it is not a dependency. |
| **CC Switch** (MIT, 152k★) | Provider + account manager for coding CLIs (Rust). | 🟣 **REFERENCE.** Its model of "multiple accounts per provider" is directly relevant to our registry design. But it is a desktop app (Tauri), not a headless library, and it manages *coding CLI* providers, not the general LLM-API + CLI + browser surface we need. |

---

## 10. Low-promise leads — verified and dismissed

| Project | Reality | Decision |
|---|---|---|
| **Shep** (`shep-ai/cli`) | HTTP 301 Moved Permanently — canonical identity unresolved during the research window. | ⛔ Not evaluated. Ask the user which project was meant, or drop. |
| **deadreckon** (`gregce/deadreckon`) | 15 stars, Rust, **no confirmed licence file** (`/license` returns null). Agent supervision / checkpointing is the right *idea*. | ⛔ Not usable as a dependency — no licence means no right to use. Study the idea, not the code. |
| **Rinne** (`GIKSN-RESEARCH/Rinne`) | 21 stars, MIT, "Local, open-source, terminal-first AI orchestration". | ⛔ Not usable — it is local-inference-oriented, which this project has ruled out, and the maturity is far too low. |
| **LiteLLM** | 62,948 stars, powerful, but licence `NOASSERTION` pending a real read, and it duplicates Hermes's routing. | ⛔ Not in the MVP. Revisit if multi-tenant billing appears. |
| **"OmniRoute"** | **Does not exist** under that name. Closest match `musistudio/claude-code-router` (MIT) is a different project. | ⛔ Correct the brief. |

---

## 11. What we must build ourselves — the complete list

| Component | Size | Why it cannot be reused |
|---|---|---|
| **Resource registry** | ~200 lines + schema | No existing tool models "LLM APIs + coding CLIs + browser services + per-account health" uniformly. CC Switch is the closest but is a desktop app for CLI providers. |
| **Capability model** | ~50 lines + schema | Nothing existing has an ordered `strength_tier` that makes "smallest capable" computable across heterogeneous resource types. |
| **System 1 router** | ~300 lines | No existing tool does task-shape → heterogeneous-resource selection. LiteLLM/RouteLLM route *models* by cost/quality. Hermes routes *providers*. |
| **Account health tracking** | ~150 lines + table | Hermes pools rotate keys; they don't track CLI account states. |
| **Adapter layer** | ~600 lines total (5 CLIs + 2 APIs) | Inherently bespoke — each tool's native interface is different. |
| **Execution store** | ~200 lines + schema | Hermes's store is the conversation; ours is resource execution (native session ids, artifacts, cost). |
| **MCP server** | ~200 lines | Trivial; the value is in the tools it exposes. |

**Total: roughly 1,700 lines** for the entire orchestrator — versus the spec's implied
multi-thousand-line engine. That is the point of this analysis.

---

## 12. Risks of the reuse-heavy plan

| Risk | Severity | Mitigation |
|---|---|---|
| Over-reliance on Hermes internals that may change | Medium | Use only documented surfaces: MCP config, plugin API, Kanban tools. Avoid `state.db` and internal imports entirely. |
| Kanban's 90%-of-budget completion checkpoint is advisory, not enforced | Low | Read it as a signal, not a guarantee; the orchestrator's own adapter-level verification is the real check. |
| Kanban descendant fencing is "cooperative runtime scoping, not OS confinement" | Medium | Do not treat it as a security boundary. Real isolation = separate OS processes, separate HOME/config dirs, per-account credentials. |
| MCP tools run in the Hermes process context | Low | The orchestrator's MCP server is a separate process (stdio child) — it only receives tool calls. |
| A dependency (LiteLLM, Multica, T3 Code, CC Switch) changes licence terms | Low | None are adopted as dependencies in the MVP. All are references only. |
