# Discovered Competitors — Full Discovery Table

All candidates found during the research window, categorized by overlap with the ai-orchestrator
project. Star counts and license info come from GitHub API queries during the research window;
rows marked "not re-verified" were affected by API rate limiting.

---

## Tier 1 — Direct overlap (agent orchestration platforms)

These projects solve a similar problem (multi-agent coding / LLM orchestration) but differ in
architecture: they aim to be the primary interface, whereas our project keeps Hermes primary.

| Project | Repository | Stars | Lang | License | Last push | Overlap | Disposition |
|---|---|---|---|---|---|---|---|
| **Multica** | https://github.com/multica-ai/multica | 55,374 | Go | Apache-2.0 | 2026-11-09 | Multi-runtime agent platform; already drives Hermes over ACP (29 runtimes) | ⛔ Do not adopt — team/workspace platform, not a personal resource-intelligence layer. 🟣 Reference: `server/pkg/agent/hermes.go` is the best available Hermes-over-ACP implementation. |
| **OpenClaw** | https://github.com/openclaw/openclaw | 394,224 | TypeScript | MIT | 2026-11-09 | Gateway-centric agent platform; WS + HTTP multiplex; plugin SDK | ⛔ Do not adopt — complete agent platform, wrong direction (would replace Hermes). 🟣 Reference: gateway protocol shape. |
| **AionUi** | https://github.com/iOfficeAI/AionUi | 34,143 | TypeScript | Apache-2.0 | 2026-11-05 | Agent UI desktop app; drives 17+ ACP coding agents | ⛔ Do not adopt — competing UI; we want Hermes as the UI. 🟣 Reference: process management, SQLite session schema. |
| **T3 Code** | https://github.com/pingdotgg/t3code | 53,007 | TypeScript | MIT | 2026-11-09 | Coding-agent orchestration; git worktrees; adapter interface; model switching | 🟣 Reference only — worktree module and ProviderAdapterV2 are good models, not dependencies. |
| **CC Switch** | https://github.com/farion1231/cc-switch | 152,936 | Rust | MIT | 2026-11-09 | Provider + account manager for coding CLIs; multi-account per provider | 🟣 Reference only — managed_account + AuthBinding model is directly relevant to registry design. Desktop app, not headless library. |

---

## Tier 2 — Adjacent (model gateways / routing)

These solve the LLM API routing problem but not the heterogeneous-resource (CLI + API) problem.

| Project | Repository | Stars | Lang | License | Last push | Overlap | Disposition |
|---|---|---|---|---|---|---|---|
| **LiteLLM** | https://github.com/BerriAI/litellm | 62,948 | Python/Rust | MIT core + Enterprise commercial | 2026-11-09 | Unified API gateway; 100+ providers; cost/latency/semantic routing; multi-account per provider | ⛔ Not in MVP — duplicates Hermes's credential pools + fallback providers. Revisit if multi-tenant billing appears. **License note:** `/license` reports NOASSERTION; actual `LICENSE` must be read before any commercial use. |
| **Portkey** | https://github.com/Portkey-AI/gateway | not re-verified | — | MIT | — | AI gateway; virtual keys; advanced routing | Not evaluated in depth — similar position to LiteLLM. |
| **Helicone** | https://github.com/Helicone/ai-gateway | not re-verified | — | GPL-3.0 | — | AI gateway; provider routing; vault | Not evaluated in depth — GPL-3.0 is a license concern. |
| **Bifrost** | https://github.com/maximhq/bifrost | not re-verified | Rust | Apache-2.0 | — | Governance-based routing; adaptive load balancing | Not evaluated in depth. |
| **Kong AI Gateway** | https://github.com/Kong/kong | — | — | Apache-2.0 (per docs) | — | Full API gateway with AI Proxy plugins | Too heavyweight for this project. |
| **RouteLLM** | https://github.com/lm-sys/RouteLLM | not re-verified | Python | verify | — | Cost-aware model routing (library, not gateway) | Interesting for cost-aware routing research; not a dependency. |
| **Cloudflare AI Gateway** | https://developers.cloudflare.com/ai-gateway/ | — | — | Proprietary (SaaS) | — | SaaS-only AI gateway | Not self-hostable — excluded. |
| **OpenRouter** | https://openrouter.ai/ | — | — | Proprietary (SaaS) | — | Hosted model routing | Not self-hostable — excluded. |
| **"OmniRoute"** | not found | — | — | — | — | — | **Does not exist.** Closest match is `musistudio/claude-code-router` (different project). |

---

## Tier 3 — Coding CLI execution (individual tools)

These are the resources the orchestrator adapts to, not components it adopts.

| Project | Repository | Stars | Lang | License | Overlap |
|---|---|---|---|---|---|
| OpenCode | https://github.com/anomalyco/opencode | not re-verified | TypeScript | MIT | `opencode serve` HTTP + SDK; ACP mode |
| Claude Code | https://github.com/anthropics/claude-code | not re-verified | TypeScript | proprietary | Agent SDK; `--output-format stream-json` |
| Cline | https://github.com/cline/cline | not re-verified | TypeScript | Apache-2.0 | `--json` NDJSON; ACP mode |
| Pi | https://github.com/earendil-works/pi | not re-verified | TypeScript | MIT | RPC mode + SDK; JSON events |

---

## Tier 4 — Low-promise leads (dismissed)

See `parts/04-lowpromise-leads.md` for full findings.

| Project | Repository | Stars | License | Disposition |
|---|---|---|---|---|
| Shep | https://github.com/shep-ai/cli | unknown | unknown | Identity unresolved (HTTP 301); not evaluated |
| deadreckon | https://github.com/gregce/deadreckon | 15 | none confirmed | No license → not usable |
| Rinne | https://github.com/GIKSN-RESEARCH/Rinne | 21 | MIT | Local-inference orientation conflicts with project constraint |

---

## What this means for the project

The research confirms **no existing project is a drop-in component** for the ai-orchestrator:

- **Multica** already drives Hermes over ACP — but it is a team platform with PostgreSQL + web UI.
- **LiteLLM** already routes across LLM providers — but it duplicates Hermes's credential pools and
  fallback providers, and its enterprise license restricts production use.
- **AionUi / T3 Code / CC Switch** have useful subsystems (process management, worktrees,
  multi-account models) — but all are desktop apps or full platforms, not headless libraries.

**Conclusion:** the orchestrator is built from scratch (~3,700 lines per the roadmap), using
Hermes for task lifecycle and building only the resource-intelligence layer (registry, router,
adapters, MCP server, execution store). All external projects above are **references only** —
none are adopted as dependencies in the MVP.