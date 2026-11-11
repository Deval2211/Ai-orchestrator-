# Open Questions and Architectural Decisions

**Status:** awaiting user decisions. These twelve items gate Phase 1 implementation.

Consolidated from `specs/undecided.md`, `references/open-questions.md`, and `parts/09-spec-audit.md`
(§15). `specs/undecided.md` and `references/open-questions.md` should be superseded by this file
as the single source of truth.

---

## Decisions that gate Phase 1 (must be answered before code)

### D-A — Integration surface with Hermes

**Options:**
1. **MCP server + Kanban worker-lane plugin** (recommended) — Hermes calls orchestrator tools
   via MCP; long-running work rides on a Kanban lane. No Hermes source modification.
2. Plugin tool only — blocks one tool call for the whole task; no parallelism, no workspaces,
   no crash recovery.
3. Delegation-provider plugin — wrong shape; route vocabulary is model/provider/profile, cannot
   express "run Kiro CLI account 2 in workspace W".

**Recommendation: Option 1.** Verified in `07-hermes-integration.md` and `00-hermes-surface-audit.md`.

### D-B — Where execution history lives

**Options:**
1. **Hermes-native** — run work inside a Hermes session (MCP tool / delegation). Hermes records
   tool calls natively. Constrains long-running async work.
2. **Kanban board** — orchestrator writes to `kanban_comment`/`kanban_attach`; board is durable
   and queryable from Hermes. History lives in the board, not the transcript.
3. **Own store + import** — orchestrator owns SQLite execution store; exports via `hermes sessions
   import` (batch, not real-time). Weakest continuity.

**Recommendation: Option 1 with Option 2 as backup.** Verified: no supported API exists for an
external process to append to `state.db` (`07-hermes-integration.md:292-315`,
`parts/00-hermes-surface-audit.md`).

### D-C — Which Hermes features to reuse vs rebuild

**Reuse (do not build):** credential pools, fallback providers, Kanban task lifecycle,
Kanban review/rework loop, delegation/subagents, `pre_tool_call` approval veto, MCP client
management, profiles, memory, skills.

**Build (orchestrator-owned):** resource registry, capability model, System 1 router, account
health tracking, adapter layer, execution store, MCP server.

### D-D — Storage

**Options:** SQLite (recommended) vs JSON files vs PostgreSQL.

**Recommendation: SQLite.** Zero extra services, matches Hermes's own `state.db`/`kanban.db`,
single-writer semantics fine for a single process. PostgreSQL unjustified at this scale
(per `06_TECHNOLOGY_STACK_AND_DEPENDENCIES.md` §4).

### D-E — Language / runtime

**Options:** Python (recommended) vs Rust vs Go.

**Recommendation: Python.** The Hermes plugin API is Python; MCP's reference SDK is Python;
every CLI in scope has a Python SDK or is trivially driven by `asyncio`. Rust/Go forfeit the
in-process plugin path.

### D-F — What runs the System 1 router

**Options:** cheap cloud API model vs small local non-LLM classifier vs heuristic pre-filter +
model fallback.

**Note:** The brief forbids local AI inference (no Ollama). Laya (Hermes's old local ONNX
router) was removed on 2026-11-09. A heuristic pre-filter with model fallback for ambiguous
tasks is the pragmatic choice.

### D-G — Browser-based AI in scope?

**Options:** exclude (recommended) vs defer vs build.

**Recommendation: exclude.** ToS exposure, CAPTCHA-handling red flag, no upside for API-capable
providers (NVIDIA key already gives direct API access). Deferred past the MVP entirely
(`07_SECURITY_AND_ACCOUNT_MANAGEMENT.md` §7).

### D-H — Verification depth per task type

**Options:** syntax/lint only vs run tests vs semantic check vs human review gate.

**Recommendation:** adapter-level verification (exit code, declared test command, lint) with
semantic judgment delegated to Kanban's reviewer (`kanban_request_review` → `sdlc-review` skill).
No bespoke verification engine.

### D-I — Trust / approval policy

**Options:** per-resource trust classes enforced in-orchestrator vs broad Hermes approval for
every action.

**Recommendation:** Class A (auto-approved, bounded by declared permissions) vs Class B
(approval-required, must run inside a Hermes tool call). Network calls to allowlisted providers
are Class A; destructive filesystem ops, DB writes, git push/deploy are Class B
(`07_SECURITY_AND_ACCOUNT_MANAGEMENT.md` §5).

### D-J — Account rotation policy + provider ToS compliance

**Options:** rotate on provider-signalled failures only vs rotate to evade quotas.

**Recommendation:** rotate only on 429-with-reset, 402 (cooldown), 401-after-refresh-fails.
Never rotate to evade billing/plan limits. Per-provider concurrency budget shared across all
accounts of that provider
(`07_SECURITY_AND_ACCOUNT_MANAGEMENT.md` §2.3).

### D-K — Concurrency + budget limits

**Missing from the spec entirely.** Needed:
- Max parallel executions globally and per provider
- Per-task spend cap (hard stop)
- Per-workspace lock (two cards cannot write the same tree)
- Idempotency key per dispatch

### D-L — Project name

**Options:** "AI Orchestrator" (placeholder) vs Hermes Orchestrator vs Nexus vs Conductor vs
Maestro vs user suggestion.

---

## Decisions already made (from prior sessions)

| # | Decision | Source |
|---|---|---|
| D1 | Hermes remains the primary interface | `references/decisions.md` D1 |
| D2 | Custom orchestrator engine behind Hermes | `references/decisions.md` D2 |
| D3 | Task-based model allocation | `references/decisions.md` D3 |
| D4 | Multiple API accounts supported | `references/decisions.md` D4 |
| D5 | Multiple CLI integrations/accounts | `references/decisions.md` D5 |
| D6 | Browser-based AI via CDP + Chrome profiles | `references/decisions.md` D6 |
| D7 | Unified Hermes session history (parent-child) | `references/decisions.md` D7 |
| D8 | Preserve Hermes skills (extend, don't replace) | `references/decisions.md` D8 |
| D9 | Local-first execution | `references/decisions.md` D9 |

**Note on D9:** The spec restates this as "prefer local before cloud." The brief clarifies
"local" means local infrastructure, not local inference. Restate as **CLI-before-API**.

---

## Corrections needed in existing specs

From `parts/09-spec-audit.md`:

1. **`specs/architecture.md` rule 1** — "Hermes is read-only" must be narrowed to: Hermes owns
   conversation state, memory, and skills; the orchestrator never writes to them. The orchestrator
   *must* register via MCP config or a plugin (both are writes to Hermes's tree) — that is
   supported and required.

2. **`specs/architecture.md`** — remove `local_model` resource type and Ollama adapter work item
   (brief forbids local inference; `moonshotai/kimi-k3.1-pro` is not in NVIDIA's catalogue).

3. **`specs/architecture.md`** — remove "Account Manager", "Task Scheduler", "Checkpoint &
   Recovery", "Verification Engine" components (replaced by Hermes/Kanban).

4. **`specs/resource-registry.md`** — fix Account Isolation table for Kiro (auth DB outside
   `KIRO_HOME`) and Claude Code (use `CLAUDE_CONFIG_DIR`, not blanket `HOME`).

5. **`specs/system-1-router.md`** — add ordered capability tiers (`strength_tier`, `context_window`,
   `cost_per_1k`, etc.) so "smallest capable" is computable.

6. **Merge the three copies of the undecided list** into this file (`10_OPEN_QUESTIONS_AND_...
.md`).

7. **Add D10+** to `references/decisions.md` with date/author/status columns.

---

## Next step

User answers D-A through D-L. Once D-A, D-D, and D-E are settled, Phase 1 (registry + one
adapter + MCP surface) can begin.