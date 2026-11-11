# 12 — Research Gaps and Recommended Solution

**Status:** evidence-based audit of existing research against the live system  
**Date:** 2026-11-11  
**Scope:** all 14 research documents in `docs/research/` + `docs/research/parts/`, all 6 phase files, all 5 specs, and the live FreeLLMAPI service at `http://127.0.0.1:3001/v1`

---

## Executive Summary

The research corpus is thorough and well-structured — **except for one critical blind spot**: **FreeLLMAPI was never analysed**. It is mentioned exactly once across 14 research documents (a footnote in `11_SOURCE_REGISTER.md:210`). Meanwhile, the actual system at `/home/deval/freellmapi/server/src/` is a fully operational, self-hosted LLM gateway that already implements:

- A **System One decision model** endpoint (`POST /v1/decisions`) with two free decision models (`@cf/cloudflare/clef-flash`, `inception/mercury-decide:free`)
- **Per-model capability metadata**: `intelligence_rank`, `speed_rank`, `size_label`, `supports_tools`, `supports_vision`, `context_window`, rate limits (rpm/rpd/tpm/tpd), monthly token budget, fallback config
- **47 provider backends** (cloudflare, cohere, beatapi, google, exa, firecrawl, brave, linkup, router9, aihorde, electronhub, aclide, xkiro, airforce, nvidia, moondream, kilo, ovh, keenable, aihorde, and 27 more)
- **Model unification across providers** (same logical model → one group, gated by `unify_models_enabled`)
- **Bandit-based routing** with Thompson sampling, exploration (0.1), per-model promote/demote multipliers
- **Full fallback chain**: model→model keep-trying, cooldowns, penalties, provider failover
- **Free-model lifecycle**: catalog sync, probe, cooldown-probe, retirement, custom-model tombstones, model-state
- **159 curated provider quirks** (info/warning severity, model-glob targets)
- **Fusion** (panel of models + judge)
- **Context handoff** and **log redaction**

Live reality (verified 2026-11-11): **303 catalogued models, 53 reachable** (250 blocked `no_key`), 4 enabled API keys across 4 platforms, across 47 provider backends.

**Consequence**: multiple research documents plan to rebuild what FreeLLMAPI already does:

| Document | Claim | What FreeLLMAPI Already Provides |
|---|---|---|
| `06_TECHNOLOGY_STACK_AND_DEPENDENCIES.md:29` | "API gateway: None — Hermes already rotates keys and fails over" | Full gateway with rotation, failover, routing, scoring |
| `05_RECOMMENDED_SYSTEM_ARCHITECTURE.md §13` | Classifier = hosted model / local non-LLM / heuristic pre-filter | Decision endpoint (`POST /v1/decisions`) with free models |
| `09-spec-audit.md §7` | Build capability model with `strength_tier`, `cost_per_1k`, `supports_resume`... | `GET /api/models` returns `intelligenceRank`, `speedRank`, `sizeLabel`, `supportsTools`, `supportsVision`, `contextWindow`, limits, `fallbackEnabled`, `priority` |
| `04_BUILD_VS_BUY_INTEGRATION_ANALYSIS.md §5` | "No gateway needed" — assumes single path | Two live paths: main model → OpenRouter; delegated subagents → `freellmapi/auto` |
| `phases/phase-2-intelligence.md:10` | "Training data: examples of tasks mapped to complexity levels" | Decision model reads task + questions, returns probabilities |

---

## Nine Gaps (Evidence-Indexed)

### GAP 1 — CRITICAL: FreeLLMAPI Never Analysed

**Evidence:**
- Single mention: `11_SOURCE_REGISTER.md:210` — "FreeLLMAPI service active at `http://127.0.0.1:3001/v1`, model `auto` routes to a live model"
- `06_TECHNOLOGY_STACK_AND_DEPENDENCIES.md:27` plans to build adapter layer + router + registry + scoring + failover — **rebuilds FreeLLMAPI**
- Live: 47 provider backends (`/home/deval/freellmapi/server/src/providers/`), 303 models catalogued, 53 reachable (`GET /v1/models?available=true`)

### GAP 2 — CRITICAL: Spec's System 1 Router Is Rule-Based; Yours Is a Decision Model

**Evidence:**
- `specs/system-1-router.md:12-39` = deterministic pipeline: CLASSIFY → CAPABILITY MAP → LOOKUP → SELECT → EXECUTE → VERIFY+ESCALATE
- `phases/phase-2-intelligence.md:10` — "Training data: examples of tasks mapped to complexity levels" (a classifier)
- Actual intent: LLM reads task and picks resource → **FreeLLMAPI `POST /v1/decisions` does exactly this**
- `05_RECOMMENDED_SYSTEM_ARCHITECTURE.md §13` treats this as minor open item, never mentions the decision endpoint
- Decision models live: `@cf/cloudflare/clef-flash`, `inception/mercury-decide:free` (verified via `GET /v1/decisions/models`)

### GAP 3 — HIGH: Capability Model Already Exists

**Evidence:**
- `09-spec-audit.md §7` rates HIGH: needs `strength_tier`, `context_window`, `cost_per_1k`, `supports_tools`, `supports_streaming`, `supports_resume`, capability→tier map
- FreeLLMAPI `GET /api/models` (auth-required) returns: `intelligenceRank`, `speedRank`, `sizeLabel`, `supportsVision`, `supportsTools`, `contextWindow`, `rpmLimit`/`rpdLimit`/`tpmLimit`/`tpdLimit`, `monthlyTokenBudget`, `priority`, `fallbackEnabled`, `qualifiedModelId`, `source`, `keyLabel` — **superset**
- Schema verified in `server/data/freeapi.db`: `models` table has `intelligence_rank`, `speed_rank`, `size_label`, `context_window`, `supports_tools`, `supports_vision`, `rpm_limit`, `rpd_limit`, `tpm_limit`, `tpd_limit`, `monthly_token_budget`, `priority`, `enabled`, `source`, `endpoint_scope` (447 rows, 445 enabled)

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
- Research build-vs-buy assumed single path → orchestrator would become **third uncoordinated router**

### GAP 5 — HIGH: Fallback Ladder Is Single-Axis; Yours Needs Two

**Evidence:**
- Research ladder (`system-1-router.md:44-50`): small → mid → strong reasoning → coding CLI → multiple agents (one axis: **strength**)
- FreeLLMAPI already has full **model→model** chain: `fallbackEnabled`, `priority`, provider unification, cooldown, keep-trying, exploration (0.1) — `services/router.ts:2529` lines
- Missing axis: **type axis** — LLM-work → CLI-work → browser/MCP-work → back to LLM
- **This type axis is genuinely yours to build — but only on top of FreeLLMAPI, never beside it**

### GAP 6 — MEDIUM: Browser Recommendation Conflates Two Things

**Evidence:**
- Research says exclude browser entirely (D-G in `10_OPEN_QUESTIONS_AND_ARCHITECTURAL_DECISIONS.md`)
- Own objection (`09-spec-audit.md §13`, `07_SECURITY_AND_ACCOUNT_MANAGEMENT.md §7`): **bridging subscription web UIs via cookies + solving CAPTCHAs** — correct, should be honoured
- But FreeLLMAPI treats **search/scrape as first-class routing targets** (`firecrawl.ts`, `exa.ts`, `brave.ts`, `linkup.ts`, `search-only.ts`, `serpsearch.ts`)
- Hermes has `chrome-devtools` MCP (30 tools) + `browser_exec` tool
- "Browser as a resource" is safe if scoped to **sanctioned APIs and Hermes's own CDP tooling** — only cookie-bridging subscription UIs and CAPTCHA-solving are banned

### GAP 7 — MEDIUM: `opencode-free-acp` Never Analysed

**Evidence:**
- `~/.hermes/plugins/opencode-free-acp/plugin.yaml` → `kind: model-provider`, description: "Real OpenCode ACP inference with Hermes-owned tools and curated free models"
- Research cited this file only as manifest-format example (`11_SOURCE_REGISTER.md:190`)
- It is **OpenCode exposed to Hermes as an inference provider over ACP** — precedent for integration path + proof OpenCode is already Hermes-visible
- Verified working: `opencode run -m opencode/glm-5.3-flash-free "hi"` → response in <5s (tested 2026-11-11)

### GAP 8 — MEDIUM: Audit Fixes Never Applied to `phases/`

**Evidence:**
- `09-spec-audit.md` Phase 0 checklist only covers `specs/`
- Still live in `phases/`:
  - `phase-1-foundation.md:16-17`: Ollama, `moonshotai/kimi-k3.1-pro` (not in NVIDIA's catalogue)
  - `phase-3-execution.md:17-19`: Pi, Cline, Claude Code adapters — **none installed** (`which pi/cline/claude` → not found)
  - `phase-4-browser.md:21`: "Handle CAPTCHA / verification flows" (ToS violation)

### GAP 9 — MEDIUM: Free-Model Churn Already Solved

**Evidence:**
- User said "considering free models for now" — churn is the hardest part
- FreeLLMAPI has: `catalog-sync`, `model-probe`, `cooldown-probe`, `model-retirement`, `custom-model-tombstone`, `model-state` — full lifecycle for catalogue where models die weekly
- Research never mentions it, never treats churn as solved problem
- Verified: `settings.catalog_applied_version` = `2026.11.10.a97a6d`, monthly tier, last sync `2026-11-10`

---

## Additional Findings (Supporting Evidence)

### Provider Quirks — 159 Curated Entries
- Table `quirks` (159 rows: 114 info, 45 warning) + `quirk_targets` (225 rows with model-glob patterns)
- Examples: `nvidia-route-hangs` (2 targets), `cn-real-name-verification` (3 targets), `freemodel-shared-monthly-byok` (29 targets)
- Not exposed via API (used internally by catalog-sync, exported via `scripts/export-catalog.ts`)

### Decision Endpoint — Operational But Unusable Without Keys
- `POST /v1/decisions` accepts System One shape: `{state, questions: {id: {type: choice|noul|score, instructions, criteria?}}}`
- Requires Cloudflare or OpenRouter key for decision models (`blockrun`, `apinex`, `beatapi`, `cloudflare`, `openrouter` adapters)
- Current FreeLLMAPI key pool: `kilo`, `ovh`, `keenable`, `aihorde` — **none support decisions** → returns `no_decision_keys` (503)
- OpenRouter free tier: `inception/mercury-decide:free` not available on account (404 on alpha endpoint)
- Cloudflare Workers AI: requires `account_id:token` format (quirk `cloudflare-key-format`)

### MCP Surface Already Configured
- `~/.hermes/config.yaml` has 3 MCP servers: `chrome-devtools`, `aws-mcp`, `mcp-libre`
- `opencode-free-acp` plugin enabled → OpenCode already a Hermes-visible resource via ACP

### Live Model Counts (2026-11-11)
| Metric | Value |
|---|---|
| Catalogued models | 303 |
| Available (has key) | 53 |
| Blocked `no_key` | 250 |
| Enabled API keys | 4 (kilo, ovh, keenable, aihorde) |
| Provider backends | 47 |
| Decision models configured | 2 (Cloudflare Clef, OpenRouter Mercury Decide) |

---

## Recommended Solution

### Build the Type Axis, Not the Model Axis

Three layers:

```
Layer 3 — SYSTEM 1 DECISION  (extend FreeLLMAPI, don't replace)
  POST /v1/decisions, free models: @cf/cloudflare/clef-flash, inception/mercury-decide:free
  CHANGE ONE THING: criteria becomes "pick a RESOURCE" instead of "pick a preset"
  Copy its own pattern: 30-min per-turn cache + heuristic pre-filter for obvious tasks

Layer 2 — UNIFIED RESOURCE REGISTRY  ← THIS IS THE NEW BUILD
  llm_api     → 53 reachable FreeLLMAPI models. DO NOT COPY — read /api/models live
                 (intelligenceRank, speedRank, supportsTools, supportsVision, limits)
  coding_cli  → opencode (serve/SSE + SDK, session id, --auto)
                 kiro-cli (stream-json, --resume-id, KIRO_HOME)
  browser     → chrome-devtools MCP, browser_exec  (sanctioned only — no CAPTCHA, no cookie-bridging)
  mcp         → aws-mcp, mcp-libre
  + speciality overlay (FreeLLMAPI quirks, provider ToS flags, cost annotations)

Layer 1 — EXECUTION + CONTINUITY
  Adapters, checkpoints, Kanban lane, session sync (per 08_SESSION_CONTINUITY_AND_RECOVERY.md)
```

### Revised Decision Set (Updating `10_OPEN_QUESTIONS_AND_ARCHITECTURAL_DECISIONS.md`)

| ID | Decision | Revised Answer | Evidence |
|---|---|---|---|
| D-F | System 1 router | **Extend FreeLLMAPI decision endpoint** — change criteria to "pick resource type" | GAP 2, GAP 3 |
| D-G | Browser in scope? | **Scoped yes** — sanctioned APIs (search/scrape via FreeLLMAPI providers) + Hermes CDP only | GAP 6 |
| D-C | Reuse vs rebuild | **Reuse FreeLLMAPI entirely** for model axis; build only type-axis registry + adapters | GAP 1, 3, 5 |
| (new) D-M | Decision model keys | Need Cloudflare or OpenRouter key for decision endpoint; budget for this | Decision endpoint test |

### Immediate Actions

1. **Add Cloudflare Workers AI key** (`account_id:token`) or **OpenRouter key** to FreeLLMAPI key pool → unblocks decision endpoint
2. **Remove Ollama, Pi, Cline, Claude Code, CAPTCHA items** from `phases/` — they are uninstalled/ToS-violating
3. **Rewrite `06_TECHNOLOGY_STACK_AND_DEPENDENCIES.md`** §API gateway: "FreeLLMAPI (self-hosted, running)" not "None"
4. **Rewrite `05_RECOMMENDED_SYSTEM_ARCHITECTURE.md`** §2.2: capability model = read FreeLLMAPI `/api/models` live
5. **Add FreeLLMAPI deep-dive** to research corpus (new `parts/10-freellmapi.md` or similar)
6. **Update `specs/system-1-router.md`** to describe decision-model-based router, not rule-based classifier

---

## Document Discrepancy Table

| Document | Claim | Live Reality | Action |
|---|---|---|---|
| `06_TECHNOLOGY_STACK_AND_DEPENDENCIES.md:27` | API gateway: None | FreeLLMAPI running, 47 backends | Rewrite |
| `05_RECOMMENDED_SYSTEM_ARCHITECTURE.md:465` | Classifier: hosted/local/heuristic | Decision endpoint exists, free models | Rewrite §13 |
| `09-spec-audit.md §7` | Build capability model | FreeLLMAPI `/api_models` returns superset | Mark resolved, cite FreeLLMAPI |
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