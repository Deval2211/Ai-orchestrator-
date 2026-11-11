# 12 — Research Gaps and Recommended Solution

**Status:** evidence-based audit of existing research against the live system  
**Date:** 2026-11-11  
**Scope:** all 19 research documents in `docs/research/` + `docs/research/parts/` (8 top-level `04`–`11` + 10 in `parts/` + this file), all 6 phase files, all 5 specs, and the live FreeLLMAPI service at `http://127.0.0.1:3001/v1`

---

## Executive Summary

The research corpus is thorough and well-structured — **except for one critical blind spot**: **FreeLLMAPI was never analysed**. Across all 18 pre-existing research documents it is mentioned exactly **twice**, and neither mention analyses it: a one-line footnote in `11_SOURCE_REGISTER.md:210` ("FreeLLMAPI service active at `http://127.0.0.1:3001/v1`") and an env-var name in a secrets row in `06_TECHNOLOGY_STACK_AND_DEPENDENCIES.md:27`. Meanwhile, the actual system at `/home/deval/freellmapi/server/src/` is a fully operational, self-hosted LLM gateway that already implements:

- A **System One decision model** endpoint (`POST /v1/decisions`) with two free decision models configured (`@cf/cloudflare/clef-flash`, `inception/mercury-decide:free`)
- **Per-model capability metadata**: `intelligence_rank`, `speed_rank`, `size_label`, `supports_tools`, `supports_vision`, `context_window`, rate limits (rpm/rpd/tpm/tpd), monthly token budget, fallback config
- **47 dedicated provider backend modules** (`server/src/providers/*.ts`) plus **33 further OpenAI-compatible platforms registered inline** in `providers/index.ts` — roughly **80 platforms** in total; 42 distinct platforms appear in the live `models` table
- **Model unification across providers** (same logical model → one group, gated by the `unify_models_enabled` setting, default ON — `services/model-groups.ts:13,26`)
- **Bandit-based routing** with Thompson sampling, exploration (0.1), per-model promote/demote multipliers
- **Full fallback chain**: model→model keep-trying, cooldowns, penalties, provider failover
- **Free-model lifecycle**: catalog sync, probe, cooldown-probe, retirement, custom-model tombstones, model-state
- **159 curated provider quirks** (info/warning severity, model-glob targets)
- **Fusion** (panel of models + judge)
- **Context handoff** and **log redaction** (`lib/log-redaction.ts`)

Live reality (verified 2026-11-11): **303 catalogued models, 53 served** (250 blocked `no_key`), 4 enabled API keys across 4 platforms (kilo, ovh, keenable, aihorde), against ~80 platform backends. Note the 53 "available" rows include 2 pseudo-routers (`auto`, `fusion`) and 51 unified aliases (e.g. `claude-opus-4-5`, `kilo-auto`, `free-router`) — the alias layer comes from `services/model-groups.ts`, so 53 served rows ≠ 53 distinct upstream models.

**Consequence**: multiple research documents plan to rebuild what FreeLLMAPI already does:

| Document | Claim | What FreeLLMAPI Already Provides |
|---|---|---|
| `06_TECHNOLOGY_STACK_AND_DEPENDENCIES.md:29` | "API gateway: None — Hermes already rotates keys and fails over" | Full gateway with rotation, failover, routing, scoring |
| `05_RECOMMENDED_SYSTEM_ARCHITECTURE.md:465` | Classifier = hosted model / local non-LLM / heuristic pre-filter (D-F) | Decision endpoint (`POST /v1/decisions`) with free models configured |
| `parts/09-spec-audit.md §7` (lines 217-218) | Build capability model with `strength_tier`, `cost_per_1k`, `supports_resume`... | `GET /api/models` returns `intelligenceRank`, `speedRank`, `sizeLabel`, `supportsTools`, `supportsVision`, `contextWindow`, limits, `fallbackEnabled`, `priority` |
| `04_BUILD_VS_BUY_INTEGRATION_ANALYSIS.md §5` | "No gateway needed" — assumes single path | Two live paths: main model → `opencode-free-acp`; delegated subagents → `freellmapi/auto` |
| `phases/phase-2-intelligence.md:10` | "Training data: examples of tasks mapped to complexity levels" | Decision model reads task + questions, returns probabilities |

---

## Nine Gaps (Evidence-Indexed)

### GAP 1 — CRITICAL: FreeLLMAPI Never Analysed

**Evidence:**
- Two mentions, neither an analysis: `11_SOURCE_REGISTER.md:210` ("FreeLLMAPI service active at `http://127.0.0.1:3001/v1`, model `auto` routes to a live model") and `06_TECHNOLOGY_STACK_AND_DEPENDENCIES.md:27` (env-var name `FREELLMAPI_API_KEY` in a secrets row)
- `06_TECHNOLOGY_STACK_AND_DEPENDENCIES.md:293-312` plans a `registry/` (schema, store, loader), `router/` (classify, capability, select) and `adapters/` (base, llm_api, opencode, kiro, claude_code, pi, cline) module tree — **rebuilds FreeLLMAPI's registry, router and scoring**
- Live: 47 dedicated backend modules + 33 inline platform registrations (~80 platforms), 303 models catalogued, 53 served (`GET /v1/models?available=true`)

### GAP 2 — CRITICAL: Spec's System 1 Router Is Rule-Based; Yours Is a Decision Model

**Evidence:**
- `specs/system-1-router.md:12-39` = deterministic pipeline: CLASSIFY → CAPABILITY MAP → LOOKUP → SELECT → EXECUTE → VERIFY+ESCALATE
- `phases/phase-2-intelligence.md:10` — "Training data: examples of tasks mapped to complexity levels" (a classifier)
- Actual intent: LLM reads task and picks resource → **FreeLLMAPI `POST /v1/decisions` does exactly this** (System One shape: `{state, questions: {id: {type: choice|noul|score, instructions, criteria?}}}`)
- `05_RECOMMENDED_SYSTEM_ARCHITECTURE.md:465-466` treats this as minor open item (D-F), never mentions the decision endpoint
- Decision models configured: `@cf/cloudflare/clef-flash`, `inception/mercury-decide:free` (verified via `GET /v1/decisions/models`) — **but currently unreachable**: both need a decision-capable key that is not in the pool (see "Decision Endpoint" below)

### GAP 3 — HIGH: Capability Model Already Exists

**Evidence:**
- `parts/09-spec-audit.md §7` (lines 217-218) rates HIGH: needs `strength_tier`, `context_window`, `cost_per_1k`, `supports_tools`, `supports_streaming`, `supports_resume`, capability→tier map
- FreeLLMAPI `GET /api/models` (dashboard-session auth) returns: `intelligenceRank`, `speedRank`, `sizeLabel`, `supportsVision`, `supportsTools`, `contextWindow`, `rpmLimit`/`rpdLimit`/`tpmLimit`/`tpdLimit`, `monthlyTokenBudget`, `priority`, `fallbackEnabled`, `qualifiedModelId`, `source`, `keyLabel` — **superset** (handler: `server/src/routes/models.ts:430-505`)
- Schema verified in `server/data/freeapi.db`: `models` table has `intelligence_rank`, `speed_rank`, `size_label`, `context_window`, `supports_tools`, `supports_vision`, `rpm_limit`, `rpd_limit`, `tpm_limit`, `tpd_limit`, `monthly_token_budget`, `paid_input_per_m`, `paid_output_per_m`, `enabled`, `source`, `endpoint_scope` (447 rows, 445 enabled); `fallback_config` supplies `priority` + `enabled` per model (447 rows)
- **Not covered by FreeLLMAPI:** `supports_streaming` and `supports_resume` have no column — these remain orchestrator-owned
- **Access caveat:** `/api/models` requires a **dashboard session token**, not the unified inference key (`middleware/requireAuth.ts` → `validateSession`). The inference key works on `/v1/*` only, and `/v1/models` returns a reduced schema (`id`, `name`, `context_window`, `available`, `unavailable_reason`). The full capability set is readable from the SQLite DB directly, or via a dashboard session

### GAP 4 — HIGH: Hermes Already Routes Through FreeLLMAPI

**Evidence:**
- `~/.hermes/config.yaml` (verified live):
  ```yaml
  model:
    default: opencode/glm-5.3-flash-free
    provider: opencode-free-acp
  delegation:
    provider: freellmapi
    model: auto
  ```
- Two live paths: main model → `opencode-free-acp` (ACP); delegated subagents → `freellmapi/auto` (FreeLLMAPI)
- Note: the main path changed on 2026-11-10 from `openrouter/liquid/lfm-2.5-2.6b:free` to `opencode/glm-5.3-flash-free` via the `opencode-free-acp` plugin — the research's build-vs-buy (§5) predates this and assumed only one path
- Research build-vs-buy assumed single path → orchestrator would become **third uncoordinated router**

### GAP 5 — HIGH: Fallback Ladder Is Single-Axis; Yours Needs Two

**Evidence:**
- Research ladder (`specs/system-1-router.md:44-50`): small → mid → strong reasoning → coding CLI → multiple agents (one axis: **strength**)
- FreeLLMAPI already has full **model→model** chain: `fallbackEnabled`, `priority`, provider unification, cooldown, keep-trying, exploration (0.1) — `services/router.ts` (2529 lines), `services/scoring.ts` (541 lines, presets `balanced`/`smartest`/`fastest`/`reliable`)
- Missing axis: **type axis** — LLM-work → CLI-work → browser/MCP-work → back to LLM
- **This type axis is genuinely yours to build — but only on top of FreeLLMAPI, never beside it**

### GAP 6 — MEDIUM: Browser Recommendation Conflates Two Things

**Evidence:**
- Research says exclude browser entirely (D-G in `10_OPEN_QUESTIONS_AND_ARCHITECTURAL_DECISIONS.md:73-79`)
- Own objection (`parts/09-spec-audit.md §13` lines 343-367, `07_SECURITY_AND_ACCOUNT_MANAGEMENT.md §7`): **bridging subscription web UIs via cookies + solving CAPTCHAs** — correct, should be honoured
- But FreeLLMAPI treats **search/scrape as first-class routing targets** (`providers/firecrawl.ts`, `exa.ts`, `brave.ts`, `linkup.ts`, `search-only.ts`, `serpsearch.ts`)
- Hermes has `chrome-devtools` MCP (30 tools) + `browser_exec` tool
- "Browser as a resource" is safe if scoped to **sanctioned APIs and Hermes's own CDP tooling** — only cookie-bridging subscription UIs and CAPTCHA-solving are banned

### GAP 7 — MEDIUM: `opencode-free-acp` Never Analysed

**Evidence:**
- `~/.hermes/plugins/opencode-free-acp/plugin.yaml` → `kind: model-provider`, version 0.4.0, license MIT, description: "Real OpenCode ACP inference with Hermes-owned tools and curated free models"
- Research cited this file only as manifest-format example (`11_SOURCE_REGISTER.md:190`)
- It is **OpenCode exposed to Hermes as an inference provider over ACP** — precedent for integration path + proof OpenCode is already Hermes-visible
- Verified working 2026-11-11: `opencode run -m opencode/glm-5.3-flash-free "Reply with exactly: HERMES-ACP-OK"` → `HERMES-ACP-OK` (exit 0); `opencode auth list` shows an "OpenCode Console" stored credential backing the `opencode/*` models

### GAP 8 — MEDIUM: Audit Fixes Never Applied to `phases/`

**Evidence:**
- `parts/09-spec-audit.md` Phase 0 checklist only covers `specs/`
- Still live in `phases/`:
  - `phase-1-foundation.md:16-17`: Ollama, `moonshotai/kimi-k3.1-pro` (not in NVIDIA's catalogue)
  - `phase-3-execution.md:11`: Ollama adapter; `phase-3-execution.md:17-19`: Pi, Claude Code, Claude Code (Claude API) adapters — **none of pi/cline/claude installed** (`command -v pi cline claude` → not found)
  - `phase-4-browser.md:21`: "Handle CAPTCHA / verification flows" (ToS violation)
- Also in `specs/`: `specs/architecture.md:69` lists "Kiro, OpenCode, Pi, Cline, Claude Code" as coding CLIs; `specs/resource-registry.md:56` has a Cline account-isolation row. Only `kiro-cli` and `opencode` are actually installed

### GAP 9 — MEDIUM: Free-Model Churn Already Solved

**Evidence:**
- User said "considering free models for now" — churn is the hardest part
- FreeLLMAPI has: `catalog-sync`, `model-probe`, `cooldown-probe`, `model-retirement`, `custom-model-tombstone`, `model-state` — full lifecycle for catalogue where models die weekly
- Research never mentions it, never treats churn as solved problem
- Verified live: `settings.catalog_applied_version` = `2026.11.10.a97a6d`, `catalog_applied_tier` = `monthly`, `catalog_last_sync_ms` = 2026-11-11 10:54:28 local — **the catalogue re-syncs itself on a schedule, so the orchestrator never hardcodes a model list**

---

## Additional Findings (Supporting Evidence)

### Provider Quirks — 159 Curated Entries
- Table `quirks` (159 rows: 114 info, 45 warning) + `quirk_targets` (225 rows with model-glob patterns)
- Examples: `nvidia-route-hangs` (2 targets), `cn-real-name-verification` (3 targets), `freemodel-shared-monthly-byok` (29 targets)
- Sourced from a signed upstream catalogue (`catalog-sync.ts` → `catalogBaseUrl()/v1/latest`), so the orchestrator inherits upstream's curation for free
- **Not exposed on any HTTP route** — served internally by `services/quirks.ts` (`resolveQuirksByModel`, `listQuirkDefinitions`) and exported only via `scripts/export-catalog.ts`. Reading them requires the SQLite DB or a new endpoint

### Decision Endpoint — Configured But Unusable Without a Decision-Capable Key
- `POST /v1/decisions` accepts System One shape: `{state, questions: {id: {type: choice|noul|score, instructions, criteria?}}}` (also aliased at `POST /v1/systemone`)
- Requires a key for one of the decision-capable platforms — adapters exist for `blockrun`, `apinex`, `beatapi`, `cloudflare`, `openrouter` (`services/decision.ts:192-340`)
- Current FreeLLMAPI key pool: `kilo`, `ovh`, `keenable`, `aihorde` — **none support decisions**
- Live test 2026-11-11: `POST /v1/decisions` → **HTTP 503**, `{"error":{"code":"no_decision_keys","message":"decision error: No decision model is available. Add a Cloudflare or OpenRouter key on the Keys page."}}` (`services/decision.ts:462-476`)
- OpenRouter path also blocked: `inception/mercury-decide:free` → 404 "No endpoints found" on `https://openrouter.ai/api/alpha/decisions`; OpenRouter's live `/api/v1/models` catalogue lists no `inception/*` model at all
- Cloudflare path viable but needs the `account_id:token` credential form (quirk `cloudflare-key-format`)
- **Net effect:** the decision layer exists in code and is configured, but is dark until a decision-capable key is added to the FreeLLMAPI key pool

### MCP Surface Already Configured
- `~/.hermes/config.yaml` has 3 MCP servers: `chrome-devtools`, `aws-mcp`, `mcp-libre`
- `opencode-free-acp` plugin enabled → OpenCode already a Hermes-visible resource via ACP

### Live Model Counts (2026-11-11)
| Metric | Value | Source |
|---|---|---|
| Catalogued models (`GET /v1/models`) | 303 | live API |
| Served rows (`?available=true`) | 53 | live API |
| Blocked `no_key` | 250 | live API |
| — of which pseudo-routers (`auto`, `fusion`) | 2 | live API |
| — of which unified aliases | 51 | live API + `services/model-groups.ts` |
| `models` table rows / enabled | 447 / 445 | `freeapi.db` |
| Distinct platforms in `models` table | 42 | `freeapi.db` |
| Dedicated provider backend modules | 47 | `server/src/providers/*.ts` (49 files − `base.ts` − `index.ts`) |
| Inline OpenAI-compatible platform registrations | 33 | `providers/index.ts` |
| Enabled API keys | 4 (kilo, ovh, keenable, aihorde) | `freeapi.db` |
| Provider backends (total, ~) | ~80 | 47 files + 33 inline |
| Decision models configured | 2 (Cloudflare Clef, OpenRouter Mercury Decide) | `GET /v1/decisions/models` |
| Decision models reachable | **0** | `POST /v1/decisions` → 503 `no_decision_keys` |
| Curated provider quirks | 159 (114 info / 45 warning) | `freeapi.db` |
| Catalog last sync | 2026-11-11 10:54:28 | `settings.catalog_last_sync_ms` |

---

## Recommended Solution

### Build the Type Axis, Not the Model Axis

Three layers:

```
Layer 3 — SYSTEM 1 DECISION  (extend FreeLLMAPI, don't replace)
  POST /v1/decisions, free models: @cf/cloudflare/clef-flash, inception/mercury-decide:free
  CHANGE ONE THING: criteria becomes "pick a RESOURCE" instead of "pick a preset"
  Copy its own pattern: 30-min per-turn cache (routing-decision.ts CACHE_TTL_MS)
                        + 1500 ms timeout + 0.5 min-probability
  BLOCKER: needs a decision-capable key first (currently 503 no_decision_keys)

Layer 2 — UNIFIED RESOURCE REGISTRY  ← THIS IS THE NEW BUILD
  llm_api     → 53 served FreeLLMAPI models. DO NOT COPY — read live:
                 /v1/models (available, context_window) for reachability
                 /api/models (dashboard session) OR freeapi.db directly for
                 intelligenceRank, speedRank, supportsTools, supportsVision, limits
  coding_cli  → opencode (serve/SSE + SDK, --session id, --auto)
                 kiro-cli (chat --no-interactive --output-format stream-json,
                          --resume-id <SESSION_ID>, KIRO_HOME)
  browser     → chrome-devtools MCP, browser_exec  (sanctioned only — no CAPTCHA, no cookie-bridging)
  mcp         → aws-mcp, mcp-libre
  + speciality overlay (FreeLLMAPI quirks table, provider ToS flags, cost annotations)
                 ORCHESTRATOR-OWNED: supports_streaming, supports_resume

Layer 1 — EXECUTION + CONTINUITY
  Adapters, checkpoints, Kanban lane, session sync (per 08_SESSION_CONTINUITY_AND_RECOVERY.md)
```

### Revised Decision Set (Updating `10_OPEN_QUESTIONS_AND_ARCHITECTURAL_DECISIONS.md`)

| ID | Decision | Revised Answer | Evidence |
|---|---|---|---|
| D-F | System 1 router | **Extend FreeLLMAPI decision endpoint** — change criteria to "pick resource type" | GAP 2, GAP 3 |
| D-G | Browser in scope? | **Scoped yes** — sanctioned APIs (search/scrape via FreeLLMAPI providers) + Hermes CDP only | GAP 6 |
| D-C | Reuse vs rebuild | **Reuse FreeLLMAPI entirely** for model axis; build only type-axis registry + adapters | GAP 1, 3, 5 |
| (new) D-M | Decision model keys | Need Cloudflare or OpenRouter key for decision endpoint; budget for this | Decision endpoint test → 503 |
| (new) D-N | How to read FreeLLMAPI's capability model | `/v1/models` (reduced schema, inference key) vs `/api/models` (full, dashboard session) vs direct SQLite read | GAP 3 access caveat |
| (new) D-O | Does the orchestrator own a model list? | **No** — read FreeLLMAPI live; catalogue re-syncs monthly and models retire weekly | GAP 9 |

### Immediate Actions

1. **Add a decision-capable key to FreeLLMAPI** — Cloudflare Workers AI (`account_id:token`) is the only currently-viable option; OpenRouter's `inception/mercury-decide:free` no longer resolves → unblocks `POST /v1/decisions` (currently 503)
2. **Decide D-N** — pick one read path for the capability model. Direct SQLite read of `server/data/freeapi.db` is the simplest today; a small read-only endpoint in FreeLLMAPI is the cleaner long-term shape
3. **Remove Ollama, Pi, Cline, Claude Code, CAPTCHA items** from `phases/` and `specs/` — they are uninstalled or ToS-violating
4. **Rewrite `06_TECHNOLOGY_STACK_AND_DEPENDENCIES.md`** §API gateway (line 29): "FreeLLMAPI (self-hosted, running)" not "None"; drop the `registry/`+`router/` module tree (lines 293-312) in favour of a FreeLLMAPI client
5. **Rewrite `05_RECOMMENDED_SYSTEM_ARCHITECTURE.md`** §2.2 + line 465: capability model = read FreeLLMAPI live, not build it
6. **Add FreeLLMAPI deep-dive** to research corpus (new `parts/10-freellmapi.md`)
7. **Update `specs/system-1-router.md`** to describe a decision-model-based router, not a rule-based classifier

---

## Document Discrepancy Table

| Document | Claim | Live Reality | Action |
|---|---|---|---|
| `06_TECHNOLOGY_STACK_AND_DEPENDENCIES.md:27` | API gateway: None | FreeLLMAPI running, 47 backends | Rewrite |
| `05_RECOMMENDED_SYSTEM_ARCHITECTURE.md:465` | Classifier: hosted/local/heuristic | Decision endpoint exists, free models configured | Rewrite line 465 |
| `parts/09-spec-audit.md §7` | Build capability model | FreeLLMAPI `/api/models` returns superset | Mark resolved, cite FreeLLMAPI |
| `phases/phase-1-foundation.md:16` | Register `moonshotai/kimi-k3.1-pro` | Not in NVIDIA catalogue | Remove |
| `phases/phase-3-execution.md:17` | Pi/Cline/Claude Code adapters | Not installed | Remove or mark future |
| `phases/phase-4-browser.md:21` | Handle CAPTCHA | ToS violation | Remove |
| `10_OPEN_QUESTIONS...md D-G` | Exclude browser | Search/scrape already in FreeLLMAPI | Revise to scoped yes |
| `specs/system-1-router.md` | Rule-based classifier | Decision model endpoint | Rewrite |

---

## Evidence Index (Every Claim Has a Verifiable Source)

| Claim | Source |
|---|---|
| FreeLLMAPI service active, 53 models reachable | `systemctl --user is-active freellmapi` + `GET /v1/models?available=true` |
| 47 provider backends | `ls /home/deval/freellmapi/server/src/providers/*.ts` |
| Decision endpoint + models | `GET /v1/decisions/models` → `['@cf/cloudflare/clef-flash','inception/mercury-decide:free']` |
| Capability fields in `/api/models` | `GET /api/models` (auth) → `intelligenceRank`, `speedRank`, `sizeLabel`, `supportsTools`, `supportsVision`, `contextWindow`, limits, `fallbackEnabled`, `priority` |
| DB schema | `server/data/freeapi.db` → `models` table columns |
| Hermes config dual-path | `~/.hermes/config.yaml` lines 1-5, 102-105 |
| Fallback chain in router | `services/router.ts` (2529 lines), `fallback_config` table (447 rows) |
| Provider quirks | `server/data/freeapi.db` → `quirks` (159) + `quirk_targets` (225) |
| opencode-free-acp working | `opencode run -m opencode/glm-5.3-flash-free "hi"` → response |
| OpenRouter decision 404 | `POST /api/alpha/decisions` with `inception/mercury-decide:free` |
| Phases uninstalled tools | `which pi/cline/claude` → not found |

---

*Document: docs/research/12_RESEARCH_GAPS_AND_RECOMMENDED_SOLUTION.md*