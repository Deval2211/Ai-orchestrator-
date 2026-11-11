# Spec Audit — ai-orchestrator

Audit of the existing specification in `specs/`, `phases/`, `references/` against the
research brief and against the live environment.

**Audit date:** 2026-11-09
**Auditor:** research pass 1 (source-verified where possible)
**Scope:** contradictions, unstated assumptions, missing decisions, dead plans, and
spec claims that are factually wrong in the current environment.

Severity:
- **BLOCKER** — implementation would produce a wrong result or fail outright.
- **HIGH** — a decision is being made by accident; a wrong turn is expensive to undo.
- **MEDIUM** — ambiguity that will cause churn during implementation.
- **LOW** — documentation hygiene.

---

## 1. BLOCKER — The "Hermes is read-only" rule contradicts the integration requirement

**Where:** `specs/architecture.md:13` ("HERMES (primary interface) -- DO NOT MODIFY"),
`specs/architecture.md:57` ("**Hermes is read-only.** The orchestrator never modifies Hermes
config, skills, memory, or plugins."), `phases/phase-1-foundation.md:9-11` ("Connect
orchestrator to Hermes via its supported interface" / "Do NOT modify Hermes itself").

**The problem:** these cannot both hold. The *only* supported ways to add a tool or service to
Hermes are:

1. A plugin dropped into `~/.hermes/plugins/<name>/` — a write to Hermes's plugin tree.
2. An MCP server entry in `~/.hermes/config.yaml` under `mcp_servers:` — a write to Hermes's config.
3. A Kanban worker-lane plugin — again a plugin write.
4. A delegation-provider plugin — again a plugin write.

An external orchestrator that Hermes can call *must* be registered through one of these.
"Read-only" is achievable only in the narrower sense of *"the orchestrator never mutates
Hermes's session state, memory or skills at runtime"* — not *"the orchestrator never adds
anything to Hermes"*.

**Also:** `specs/architecture.md:80-83` (data flow step 8) says "Session Sync records the full
history into the Hermes conversation", and `specs/session-continuity.md:49-51` says every task
is "recorded in the Hermes conversation context". **There is no supported API for an external
process to append to a Hermes session.** Verified two ways:

- Independent audit: session history lives in `state.db` (SQLite) with `sessions` and `messages`
  tables; the only write paths are the internal `SessionDB.append_message`, `PluginContext.post_message`
  (live gateway session only), and batch `hermes sessions import`. No public append API.
- Delegated research (`docs/research/parts/07-hermes-integration.md:292-315`) reached the same
  conclusion independently.

**Fix required:** rewrite the rule as *"Hermes is authoritative for conversation state, memory and
skills; the orchestrator never writes to them. It registers itself through exactly one supported
surface (a plugin or an MCP entry), and its own execution history lives in its own store."*
Then re-derive the session-continuity design on that basis.

---

## 2. BLOCKER — The spec still assumes local inference, which the brief forbids

**Where:** `specs/architecture.md:67-68` (Ollama listed as an API provider and as a "Local
models" resource type), `specs/resource-registry.md:10` (`type: ... | local_model | ...`),
`phases/phase-1-foundation.md:17` ("Local: Ollama (no model loaded yet)"),
`phases/phase-3-execution.md:11` ("Ollama adapter"),
`specs/undecided.md:24` (option A: "Should the router itself be a small local model (Ollama)").

**The constraint:** the brief states under §10 "Important constraint: No local AI inference" —
"the orchestrator itself can run locally during development... Local infrastructure is not the
same as local AI inference."

**Environment check:** `ollama` is not installed on this machine (`command -v ollama` → missing).
The `local_model` resource type has no candidate to register. Phase 1's deliverable would be
unbuildable as written.

**Also note the terminology trap:** `references/decisions.md` D9 records **"Local-first
execution"** as decided ("prefer local before cloud"). In the brief, "local" means local
*infrastructure* — the orchestrator may run locally — while local *inference* is banned. As
written, D9 reads as "prefer Ollama before cloud APIs", which contradicts the brief and would
also conflict with Principle 1 ("smallest capable resource, not cheapest available"). D9 must be
restated as **"CLI-before-API"** (prefer a local coding CLI over a cloud API for the same task),
which is what the rest of the spec actually means by it.

**Fix:** remove the `local_model` resource type and the Ollama adapter work item; restate D9.

---

## 3. BLOCKER — Phase 1 targets a model ID that NVIDIA does not serve

**Where:** `specs/resource-registry.md:12` (`model: "moonshotai/kimi-k3.1-pro"`),
`phases/phase-1-foundation.md:16` and `:36` (same ID, and "Orchestrator registers
`moonshotai/kimi-k3.1-pro`" as a verification criterion).

**Verified against the live endpoint:**
```
GET https://integrate.api.nvidia.com/v1/models   (authenticated with the stored NVIDIA key)
→ 84 models
→ models matching "kimi":   ['moonshotai/kimi-k2.6']
→ models matching "moonshotai": ['moonshotai/kimi-k2.6']
→ models matching "minimax":  ['minimaxai/minimax-m3.1-flash']
```

`moonshotai/kimi-k3.1-pro` is **not in NVIDIA's catalogue**. The Phase 1 deliverable ("reply
with HELLO_KIMI_TEST" through NVIDIA NIM) and its verification criterion would fail on first
run. The available Kimi model is `moonshotai/kimi-k2.6`.

**Fix:** pick a model ID from the live catalogue and treat `/v1/models` as the source of truth
for the registry (this also answers "model capability detection" from §2.A of the brief — the
provider already publishes the list; the orchestrator should read it rather than hard-code it).

---

## 4. HIGH — The spec rebuilds four things Hermes already ships

This is the most consequential finding. `specs/architecture.md:44-53` defines eight components;
four of them duplicate existing Hermes features.

| Spec component | Hermes already provides | Verified source |
|---|---|---|
| **Account Manager** — per-account auth, isolation, limits, usage counters | **Credential pools** — register multiple API keys/OAuth tokens for one provider, auto-rotate on 429/402/401/400. Strategies: `round_robin`, `least_used`, `fill_first`, `random`. | https://hermes-agent.nousresearch.com/docs/user-guide/features/credential-pools |
| *(escalation / fallback to next resource)* | **Fallback providers** — cross-provider failover with a `fallback_providers:` list, reset-aware benching, `min_switch_reset_seconds` | https://hermes-agent.nousresearch.com/docs/user-guide/features/fallback-providers |
| **Task Scheduler** — queue, parallelize, checkpoint tasks | **Kanban** — durable SQLite board (`~/.hermes/kanban.db`), 14 `kanban_*` tools, per-task pinned workspaces, parallel dispatch, circuit breaker, crash/timeout reaping | https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban |
| **Verification Engine** — re-check outputs, flag mistakes, trigger rework | **Kanban review loop** — `kanban_request_review` → reviewer approves / `kanban_request_changes` (routes back to the implementer) / `kanban_block` | https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-worker-lanes |
| **Checkpoint & Recovery** — save state, resume after failure | **Kanban lifecycle** — `crashed` / `gave_up` / `timed_out` events, respawn on `kanban_unblock`, per-task workspaces | same |

Plus partial overlap:
- **Provider routing** (`sort: price|throughput|latency`, `only`, `ignore`, `order`) already gives
  cost/latency-aware provider choice — https://hermes-agent.nousresearch.com/docs/user-guide/features/provider-routing
- **Delegation** already gives parallel subagents with an audit trail, and the
  `DelegationProvider` plugin interface (`agent/delegation_provider.py`) lets a plugin decide the
  route per task.

**The decisive detail:** the Kanban worker-lane contract explicitly targets this exact shape —
*"a CLI worker that wraps Codex / Claude Code / OpenCode, a containerised review worker, a
non-Hermes service that pulls tasks via the API"*. A plugin-supplied `spawn_fn(task, workspace,
board=board)` is the documented seam for an external orchestrator to drive coding CLIs.

**Consequence:** the orchestrator's value is NOT task lifecycle. It is **resource intelligence**:
knowing what resources exist, which accounts are healthy, what each can do, and which is the
smallest capable one. The spec should shrink to that and use Hermes for the rest.

---

## 5. HIGH — Session continuity has no mechanism

**Where:** `specs/session-continuity.md` (whole file), `specs/architecture.md:51,80-83`,
`phases/phase-5-continuity.md`.

The spec states the requirement correctly — "everything stays accessible from the same Hermes
conversation" — and correctly identifies the constraint that external CLIs keep private sessions.
But it never answers: **how does the orchestrator's history get into Hermes?**

The answer, per §1: it cannot, through a supported API. The spec's Phase 5 deliverable ("Continue
a conversation from an older task; see all history, artifacts, and context") therefore has no
implementation path as specified.

**Three viable reframes** (a decision the user must make — this is the single most important
open question):

- **(a) Hermes-native history.** Run the orchestrator's work *inside* a Hermes session (plugin
  tool / MCP tool / delegation). Hermes records the tool calls and results natively. The
  orchestrator keeps only execution *metadata* it needs for routing; the conversation stays
  Hermes's. Cost: long-running async work cannot stream back after the tool call returns.
- **(b) Kanban as the history.** The orchestrator writes to the Kanban board
  (`kanban_comment`, `kanban_attach`, `kanban_attach_url`); the board is durable, queryable, and
  visible from Hermes. The board becomes the unified execution history and the conversation
  references it. Cost: history lives in the board, not the transcript.
- **(c) Own store + import.** The orchestrator owns a real execution store and exports to Hermes
  via `hermes sessions import` (batch, not real-time). Cost: weak continuity; re-imports.

---

## 6. HIGH — The account-isolation table is factually wrong for Kiro and Claude Code

**Where:** `specs/resource-registry.md:47-60` (Account Isolation table),
`phases/phase-1-foundation.md:21-24` (per-account wrapper scripts).

The table says Kiro isolation = `KIRO_HOME` → `~/.kiro/`, and Claude Code isolation = `HOME` →
`~/.claude/`.

**Verified on this machine:** running `kiro-cli chat --no-interactive` with a fresh
`HOME` + `KIRO_HOME` did **not** use the existing account — it repeatedly attempted
"Opening browser..." (an interactive OAuth sign-in). The binary contains OAuth/device-flow code
and stores credentials in a SQLite DB at `~/.local/share/kiro-cli/data.sqlite3`, **outside**
`KIRO_HOME`. So `KIRO_HOME` isolates sessions/skills/settings but **not the auth credential**;
a second Kiro account needs a separate interactive login.

Note: the delegated CLI research (`docs/research/parts/08-coding-cli-landscape.md:104-110`)
reports `KIRO_HOME` does isolate OAuth sessions when the credential store is used. These two
findings conflict; the live test is direct evidence and the docs claim is narrower than it
looks. **Treat Kiro multi-account as unverified until tested with two real logins.**

Claude Code: `HOME` override is a blunt instrument — it relocates *everything* under home, not
just Claude's config, and the dedicated knob is `CLAUDE_CONFIG_DIR`. The table should use the
dedicated variable per tool, and `HOME` only as a fallback.

**Missing from the table entirely:** OpenCode's real isolation knobs are `XDG_CONFIG_HOME` +
`XDG_DATA_HOME`, but OpenCode also has a **background server mode** (`opencode service`,
`opencode serve`, `--server`, `--standalone`) and an **ACP server** (`opencode acp`). A shared
background server is a single point of auth and state — per-account isolation must mean one
server instance per account, not just one config dir per account. The spec's "wrapper script"
model does not express that.

---

## 7. HIGH — "Smallest capable resource" is not computable from the registry schema

**Where:** `specs/architecture.md:44-53`, `specs/resource-registry.md:8-32`,
`specs/system-1-router.md:31` ("Pick the smallest capable model from the available set").

The registry gives each resource a flat `capabilities:` list (`reasoning`, `code_generation`,
`agentic`) plus `cost_tier` and `latency_tier`. **There is no ordering.** "Smallest capable" needs
a total or partial order over resources — cheaper, smaller context, faster, weaker, stronger.
A flat capability *set* cannot express "this model is smaller but still capable".

The brief asks for the smallest capable resource, plus cost/latency/capability evaluation, plus
confidence-based routing, plus cascading, plus a way to measure routing accuracy. The current
schema supports none of that. Required additions to the resource object:

- `strength_tier` (an ordered scale), `context_window`, `cost_per_1k` (input/output),
  `typical_latency_ms`, `supports_tools`, `supports_streaming`, `supports_resume`.
- A **capability → minimum tier** mapping so "capable" is defined per task type.
- A **cost/time budget** per task, so "smallest" is smallest *that still succeeds*.

Related: `specs/system-1-router.md:62-64` lists router outputs "Estimated cost and time" but
nothing in the registry provides the data to estimate them.

---

## 8. HIGH — The router's decision loop has no learning signal or accuracy metric

**Where:** `specs/system-1-router.md`, `phases/phase-2-intelligence.md`.

The spec's own instruction is *"Do not treat a model's self-reported confidence as proof of
correctness"* (brief §3) and *"Investigate ways to measure routing accuracy"* — but the spec
defines no evaluation. `phases/phase-2-intelligence.md:10` lists "Training data: examples of
tasks mapped to complexity levels" as a work item with no source, no format, no held-out split,
and no accuracy metric.

Also missing from the router spec:
- **Which model runs the router?** (`OQ1` — open.) If the router itself is a cloud LLM, every
  task costs an extra call. A cheap local classifier is exactly what Laya was doing in Hermes,
  and that has now been removed (see §9).
- **What is the ground truth for "correct" routing?** Without a labelled task set and a
  success oracle (did the task pass verification?), routing accuracy is unmeasurable.
- **Cascading policy.** "Escalate on failure" needs a defined trigger: verification verdict,
  timeout, error class, or self-reported confidence. The spec's Verification Engine is
  "stateless" and its verdict format is undefined.
- **Cost of escalation.** No budget cap, no "stop escalating at $X" rule.

---

## 9. MEDIUM — The Hermes Laya router was removed, and the spec never noticed

Hermes had a local ONNX decision model (`laya-onnx` skill, `tools/laya_tool.py`, `router/`
module) used for skill/tool routing. All of it was removed on 2026-11-09 — the skill, the tool,
the router module, the model cache, and the `fact-check` skill that depended on it. Hermes now
imports 29 tool modules cleanly with no Laya dependency.

The spec never referenced Laya, so nothing in `specs/` is broken by this. But it matters for
`OQ1` (what runs the System 1 router): the local-decision-model option is gone, and the
`fact-check` skill that referenced it no longer exists.

**Consequence for the research stack:** extraction now runs through Scrapling
(`scrapling_extract`, `scrapling_search_extract`) and search through SearXNG; `web_extract`
runs on the keenable keyless backend. No firecrawl. Verified working this session.

---

## 10. MEDIUM — Storage: the spec's own answers conflict with the brief's proposal

**Where:** `references/open-questions.md:12-17` (OQ2: SQLite / JSON / Redis),
`specs/undecided.md:18`, `phases/phase-1-foundation.md:19` ("Store registry in a simple
persistent format"), `phases/phase-5-continuity.md:20` ("Save to persistent store + Hermes
session").

The brief proposes **PostgreSQL** as shared infrastructure. The spec's open question does not
list PostgreSQL as an option at all — it lists SQLite, JSON, Redis. Neither document has
reconciled with the other, and both are still marked undecided.

The brief also says: *"Do not introduce Kubernetes, Kafka, Redis or other infrastructure unless
the actual requirements justify it."* Redis is proposed in OQ2 without justification. For a
single-user, single-process orchestrator, **SQLite** satisfies every stated requirement
(durable, queryable, zero extra services, and it matches Hermes's own `state.db` / `kanban.db`
choice). PostgreSQL would be a real operational cost for no demonstrated benefit.

---

## 11. MEDIUM — Concurrency, idempotency, rate limits and cost accounting are absent

A word-frequency scan across `specs/*.md` and `phases/*.md`:

| Concept | Hits |
|---|---|
| `parallel` | 6 |
| `cost` | 5 |
| `quota` | 6 |
| `timeout` | 1 |
| `retry` | 1 |
| `idempot` | **0** |
| `concurren` | **0** |
| `rate-limit` | **0** |

Yet the system is explicitly multi-resource and multi-account, with automatic retry
(`specs/autonomy-and-safety.md:9`) and parallel agents (`phases/phase-6-advanced.md:9-11`).
Missing entirely:

- **Concurrency limits** — how many CLI processes at once, per account, per provider, per repo?
  Without this, a fan-out can exhaust the machine or trip provider limits.
- **Idempotency** — the same task submitted twice, or a retried task after a network timeout,
  must not run twice. Nothing specifies a task idempotency key.
- **Retry policy** — "Automatic retries and fallback" has no backoff, no max attempts, no
  distinction between retryable (429/5xx) and fatal (400/401) errors.
- **Cost accounting** — "usage and cost tracking" is a stated requirement (brief §2.A) and
  "Estimated cost" is a router output, but no cost model, no spend cap, no per-account spend
  tracking exists in the spec.
- **Per-provider rate-limit accounting** — `limits.rpm/tpm/daily` exist in the registry schema
  but nothing says who decrements them or how quota is discovered.

---

## 12. MEDIUM — The approval model does not work across a process boundary

**Where:** `specs/autonomy-and-safety.md:11,16-22` (risky actions requiring approval),
`specs/architecture.md:8` (USER: "approval requests").

The spec requires approval for file deletion/overwrite outside scratch, DB writes, secret
access, network calls to external services, and code that will be committed or deployed. But it
assumes the orchestrator can *ask* Hermes for approval and get an answer.

**Verified:** a `pre_tool_call` hook can block a tool call and surface a message to the user
(`plugins.py:1912-1984`), and hooks receive `turn_id`, `tool_call_id`, `session_id` for
correlation (`tools/approval_context.py:66-69`). **But this only fires when the action is invoked
as a Hermes tool.** A background orchestrator process with no active tool call has no way to
raise an approval prompt, and no way to receive the answer.

This is not a detail — "network calls to external services" is listed as requiring approval, which
under a naive reading means *every LLM API call needs approval*, which would make the whole
system unusable. The policy needs to be rewritten in terms of what is actually enforceable:
per-resource trust policy (e.g. "NVIDIA NIM reads are auto-approved; any adapter that writes
outside the task workspace requires approval"), enforced **inside** the orchestrator, with Hermes
approval reserved for the in-session tool path.

---

## 13. MEDIUM — Browser-based AI is scheduled as Phase 4 without resolving its legal risk

**Where:** `phases/phase-4-browser.md` (whole phase), `references/open-questions.md:42-46` (OQ6).

OQ6 asks for a policy on what may be bridged vs. what must use an official API — and is still
open. But Phase 4 already has a deliverable: "Orchestrator can drive the NVIDIA Build playground
programmatically."

The prior analysis (`/home/deval/Projects/multi-account-cli-isolation-analysis.md` §2.6, §5)
concluded browser-cookie bridging is *"against the ToS of both OpenAI and Anthropic — they can
revoke your subscription"*, *"unstable — any UI change, CSS selector update, or API endpoint
change breaks your bridge"*, and *"not a viable production approach"*.

Two unresolved sub-questions in the phase itself:
- **Captcha handling.** `phases/phase-4-browser.md:21` lists "Handle CAPTCHA / verification flows"
  as a work item. Solving CAPTCHAs is the clearest possible signal of adversarial automation.
- **The NVIDIA Build playground is a UI in front of an API you already have.** The
  `NVIDIA_API_KEY` gives direct API access. Driving the playground through a browser is strictly
  worse: slower, more fragile, and adds a ToS question with no upside. Unless the playground
  exposes a model the API does not, the phase has no justification.

**Recommendation:** drop browser-based AI from the MVP entirely (the brief explicitly invites
this: "Evaluate whether it should be excluded from the initial MVP"). If it returns later, scope
it to Playwright driving *permitted* services, with no CAPTCHA-solving, and never for
subscription-bypass.

---

## 14. LOW — Documentation hygiene

- **Three copies of the undecided list.** `README.md:68-86` (table), `specs/undecided.md` (table
  + numbered list), `references/open-questions.md` (OQ1-OQ6). They disagree: `undecided.md`
  lists 4 open questions, `open-questions.md` lists 6 (adding OQ5 Verification Engine Scope and
  OQ6 Browser AI Legal/Stability), and `README.md` mentions "Final project name" but not OQ5/OQ6.
  **Single source of truth needed.**
- **`references/decisions.md` has no date, author, or status column** and no D10+ for decisions
  made after the initial nine (e.g. the no-local-inference constraint, which is currently only in
  the brief, not in the decision log).
- **`README.md:7`** references `/home/deval/Projects/multi-account-cli-isolation-analysis.md`.
  That file exists and was read during this audit — it is the source of several findings above.
- **`specs/resource-registry.md:22-23`** has `rpm: 40`, `tpm: null`, `daily: null` as literal
  example values. `rpm: 40` is not NVIDIA's actual limit. Examples that look like real config
  get copy-pasted.
- **The registry schema mixes declared and observed state.** `limits:` (declared) and `state:`
  (observed: `quota_remaining`, `last_used`) sit in one object. The spec never says who updates
  `state`, or how often. If the registry is a static config file, `quota_remaining` is a lie
  within seconds. Either split it (registry = declared; status store = observed) or say plainly
  that it is a cache.

---

## 15. Decisions still required from the user

Ordered by how much downstream work they gate.

| # | Decision | Options | Gates | Evidence |
|---|----------|---------|-------|----------|
| **D-A** | **How the orchestrator registers with Hermes** | (1) MCP server + config entry; (2) plugin tool + Kanban worker lane; (3) delegation-provider plugin | Everything. Determines the whole integration shape. | `07-hermes-integration.md`, `00-hermes-surface-audit.md` |
| **D-B** | **Where execution history lives** | (a) Hermes-native (work inside a session); (b) Kanban board; (c) own store + import | Phase 5, session continuity, "non-negotiable" principle 4 | `07-hermes-integration.md:292-315` |
| **D-C** | **Which Hermes features are reused vs. rebuilt** | Reuse Kanban/credential-pools/fallback/delegate_task, or build a parallel stack | Phase 1, 3, 5, 6 — and total maintenance burden | §4 above |
| **D-D** | **Storage** | SQLite (recommended) vs JSON vs PostgreSQL | Phase 1 | §10 above; brief §10 constraint |
| **D-E** | **Language / runtime** | Python (matches Hermes plugin API + SDKs) vs Rust (matches herdr-core) vs Go | Phase 1, and D-A's implementation | `08-coding-cli-landscape.md`, `01-herdr.md` |
| **D-F** | **What runs the System 1 router** | Cheap cloud API model vs. a small local classifier (non-LLM) vs. heuristic pre-filter | Phase 2, per-task latency/cost | §8 above; Laya removed |
| **D-G** | **Is browser-based AI in scope at all?** | Exclude (recommended) vs. defer vs. build | Phase 4 | §13 above |
| **D-H** | **Verification depth per task type** | syntax/lint only vs. tests vs. semantic vs. human gate | Phase 2, 3, 6 | OQ5, still open |
| **D-I** | **Trust/approval policy** | Per-resource trust classes, enforced in-orchestrator | Phase 1, safety model | §12 above |
| **D-J** | **Account rotation policy + provider ToS compliance** | Same-provider rotation only; no quota circumvention | Phase 1 | Brief §11: "Do not design account switching as a way to bypass quotas" |
| **D-K** | **Concurrency + budget limits** | Max parallel CLIs, max spend per task, retry/backoff policy | Phase 1, 6 | §11 above |
| **D-L** | **Project name** | — | Cosmetic, but blocks final docs | `open-questions.md:26-33` |

**D-A, D-B and D-C are the ones that matter.** The rest are implementation details that can be
decided as each phase starts — except D-D and D-E, which are needed before Phase 1.

---

## 16. What the spec gets right (keep these)

- The parent-child model for session continuity (`specs/session-continuity.md:16-45`) is honest
  and correct — it does not pretend cross-CLI session sharing is possible. This is the single
  best-informed part of the spec.
- "Smallest capable resource, not cheapest available" (README:27) is a genuinely good product
  principle. It needs a computable schema, but the intent is right.
- The resource-as-adapter model (`specs/architecture.md:59`) is the right abstraction and makes
  the CLI landscape findings (HTTP server vs. subprocess vs. ACP) directly usable.
- The explicit "Hermes stays primary / extend, don't replace" stance (principle 2, D1, D8) is
  correct and should be preserved verbatim — only the "read-only" wording needs fixing.
- The risk that "Hermes Gateway" does not bridge external CLI sessions was correctly identified
  in the prior analysis and correctly carried into `specs/session-continuity.md`.
