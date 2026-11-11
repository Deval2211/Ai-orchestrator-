# 11 — Source Register

Every project examined during this research. Star counts, languages and SPDX licenses come from
the GitHub REST API (`https://api.github.com/repos/<owner>/<repo>` and its `/license` endpoint)
queried during the research window; where a query was rate-limited, the row says so explicitly
rather than carrying a guess.

**Research date:** 2026-11-09
**GitHub API rate limit note:** unauthenticated, 60 requests/hour per IP. The limit was hit
mid-research by parallel subagents; a small number of rows are marked `not re-verified` and
should be refreshed before publication.

---

## A. Projects named in the brief

### herdr ecosystem

| Field | Value |
|---|---|
| **Project** | herdr — "the runtime your coding agents live on" |
| **Website** | https://herdr.dev/ · docs https://herdr.dev/docs/ |
| **Repository** | https://github.com/herdrdev/herdr (Rust) · https://github.com/herdrdev/herdr-core (Rust) |
| **Official documentation** | https://herdr.dev/docs/ · `crates/herdr-core/schema/README.md` (OpenRPC contract) |
| **License source** | `LICENSE` file in each clone → **Apache-2.0** (verified by reading the file) |
| **Latest verified activity** | herdr-core last push 2026-11-08; herdr last push 2026-11-07 (API, verified earlier in the research window) |
| **Stars** | herdr 49,554 · herdr-core 529 (API, verified earlier in the research window) |
| **Key references** | `hc.openrpc.json` (32-method OpenRPC doc) · `crates/herdr-core/src/{server,client,runtime,terminals,holder}.rs` · `crates/herdr-core/src/api/*` · herdr-core CHANGELOG · herdr discussion #741 (agent-to-agent messaging request) |

Also in the org: `herdrdev/herdr-app` (macOS app bundle), `herdrdev/herdr-layout-examples`
(TUI layouts), `herdrdev/herdr-nix` (Nix packaging). Stars/languages for these three were not
re-verified after the rate limit hit; herdr-app is referenced from the herdr-core CHANGELOG.

### Multica

| Field | Value |
|---|---|
| **Project** | Multica — multi-agent coding platform |
| **Website** | https://multica.ai/ |
| **Repository** | https://github.com/multica-ai/multica (Go) |
| **Official documentation** | Repository `README.md` + `docs/` tree |
| **License source** | `LICENSE` file → **Apache License 2.0** (confirmed by reading the file; the GitHub API summary field reports `NOASSERTION`, which is a licence-detection artefact, not a custom licence) |
| **Latest verified activity** | 2026-11-09 (API) |
| **Stars** | 55,374 (API) |
| **Key references** | `README.md`, `LICENSE`, Go service layout, HTTP route table |

### OpenClaw

| Field | Value |
|---|---|
| **Project** | OpenClaw — agent + gateway runtime |
| **Website** | https://openclaw.ai/ |
| **Repository** | https://github.com/openclaw/openclaw (TypeScript) |
| **Official documentation** | Repository docs tree |
| **License source** | `/license` endpoint → **MIT** (verified via API) |
| **Latest verified activity** | 2026-11-09 (API) |
| **Stars** | 394,224 (API) |
| **Key references** | Gateway/agent architecture, provider integration, auth profiles, browser automation, persistence layer, security model |
| **Hermes relationship** | Hermes ships a one-command *import* of an OpenClaw install (https://hermes-agent.nousresearch.com/docs/user-guide/import-from-other-agents) — the migration direction is OpenClaw → Hermes, not the reverse |

### AionUi

| Field | Value |
|---|---|
| **Project** | AionUi — agent UI |
| **Repository** | https://github.com/iOfficeAI/AionUi (TypeScript) |
| **Website** | Project homepage from the repository |
| **Official documentation** | Repository `README.md` |
| **License source** | `/license` endpoint → **Apache-2.0** (verified via API) |
| **Latest verified activity** | 2026-11-05 (API) |
| **Stars** | 34,143 (API) |
| **Key references** | `packages/` tree (1,195 entries), agent integration adapters, process-management path |

### T3 Code

| Field | Value |
|---|---|
| **Project** | T3 Code — coding-agent orchestration with worktrees |
| **Repository** | https://github.com/pingdotgg/t3code (TypeScript) |
| **License source** | `/license` endpoint → **MIT** (verified via API) |
| **Latest verified activity** | 2026-11-09 (API) |
| **Stars** | 53,007 (API) |
| **Key references** | Worktree module, agent adapter interface, model-switching path, diff/PR workflow |

### CC Switch

| Field | Value |
|---|---|
| **Project** | CC Switch — provider & account manager for coding CLIs |
| **Website** | https://ccswitch.io/ |
| **Repository** | https://github.com/farion1231/cc-switch (Rust) |
| **License source** | `/license` endpoint → **MIT** (verified via API) |
| **Latest verified activity** | 2026-11-09 (API) |
| **Stars** | 152,936 (API) |
| **Key references** | `aioncore/` and `ccswitch/` Rust crates, provider config model, credential storage |

### Shep

| Field | Value |
|---|---|
| **Project** | Shep CLI |
| **Repository** | `https://github.com/shep-ai/cli` → **HTTP 301 Moved Permanently**; canonical URL to be confirmed from the org's repo list |
| **License / activity / stars** | **Not verified** — repo did not resolve during the research window |
| **Status** | Unresolved identity. Treated as a low-confidence lead; see `parts/04-lowpromise-leads.md` |

### deadreckon

| Field | Value |
|---|---|
| **Project** | deadreckon — agent supervision / checkpointing |
| **Repository** | https://github.com/gregce/deadreckon (Rust) |
| **License source** | `/license` endpoint returns **null**; no SPDX id |
| **Latest verified activity** | 2026-08-21 (API) |
| **Stars** | 15 (API) |
| **Key references** | `Cargo.toml` workspace (`crates/deadreckon`, `crates/deadreckon-core`, `crates/deadreckon-protocol`, `crates/deadreckon-providers`) |
| **Status** | Very low maturity; **no license file confirmed** → not usable as a dependency without a license |

### Rinne

| Field | Value |
|---|---|
| **Project** | Rinne — "Local, open-source, terminal-first AI orchestration" |
| **Repository** | https://github.com/GIKSN-RESEARCH/Rinne (MIT) — best match for the brief's "Rinne" |
| **License source** | `/license` endpoint → **MIT** (verified via API) |
| **Stars** | 21 (API) |
| **Key references** | README, repo tree |
| **Status** | Very low maturity. Its local-inference orientation conflicts with this project's no-local-inference constraint |

### LiteLLM

| Field | Value |
|---|---|
| **Project** | LiteLLM — unified LLM API gateway |
| **Repository** | https://github.com/BerriAI/litellm (Python) |
| **License source** | `/license` endpoint → **NOASSERTION** — the summary field is a detection artefact; the **actual `LICENSE` file** must be read and its non-MIT clauses quoted before any commercial use |
| **Latest verified activity** | 2026-11-09 (API) |
| **Stars** | 62,948 (API) |
| **Key references** | `litellm/router.py`, `litellm/router_strategy/` (strategy names), `litellm/proxy/`, Rust `litellm-rust/crates/router-core/src/strategy.rs` (`StrategyScope`, `OVERRIDABLE`) |

### OmniRoute

| Field | Value |
|---|---|
| **Status** | **No repository found** under the obvious names (`omniroute/omniroute`, `OpenRouterTeam/omniroute`, `lm-sys/omniroute` all 404). The brief's "OmniRoute" could not be matched to a real project. |
| **Closest real match** | https://github.com/musistudio/claude-code-router — **MIT**, 38,060 stars, an LLM proxy/router for Claude Code. A **different project**; must not be conflated with "OmniRoute". |
| **Action** | Ask the user which project was meant, or drop it |

---

## B. Projects discovered during the research

| Project | Repository | Stars | Lang | License | Last push | Overlap |
|---|---|---|---|---|---|---|
| OpenCode | https://github.com/anomalyco/opencode | not re-verified | TypeScript | MIT (from docs research) | — | Coding CLI execution, server mode + SDK |
| Claude Code | https://github.com/anthropics/claude-code | not re-verified | TypeScript | proprietary | — | Coding CLI execution, Agent SDK |
| Cline | https://github.com/cline/cline | not re-verified | TypeScript | Apache-2.0 (from docs research) | — | Coding CLI execution, ACP mode |
| Pi Coding Agent | https://github.com/earendil-works/pi (per CLI research; verify canonical URL) | not re-verified | TypeScript | MIT (from docs research) | — | Coding CLI execution, RPC + SDK |
| RouteLLM | https://github.com/lm-sys/RouteLLM | not re-verified | Python | verify — historically Apache-2.0 with a non-commercial restriction | — | Cost-aware model routing |
| Hermes Agent | https://github.com/NousResearch/hermes-agent | — | Python | verify | — | **The primary interface** |
| OpenAI-compatible route | `musistudio/claude-code-router` | 38,060 | TypeScript | MIT | — | Provider routing for coding CLIs |

Full discovery table with all candidates and their category overlap:
`docs/research/parts/06-discovered-competitors.md`.

---

## C. Hermes Agent — the primary interface (reference, not a candidate)

| Field | Value |
|---|---|
| **Website / docs** | https://hermes-agent.nousresearch.com/docs/ · full index at https://hermes-agent.nousresearch.com/docs/llms.txt |
| **Repository** | https://github.com/NousResearch/hermes-agent |
| **Local source** | `/home/deval/.hermes/hermes-agent` (authoritative for this audit) |
| **Docs pages used** | programmatic-integration, plugins, plugin-llm-access, hooks, credential-pools, fallback-providers, provider-routing, kanban, kanban-worker-lanes, delegation, mcp, session-storage, web-search, tools, security |

### Hermes source files inspected (with what they establish)

| File | Establishes |
|---|---|
| `hermes_cli/plugins.py:463` | `PluginContext.register_tool(name, toolset, schema, handler, check_fn, requires_env, is_async, description, emoji, override)` |
| `hermes_cli/plugins.py:1912-1984` | `pre_tool_call` hook can block a tool call and surface a message to the user (the approval-veto path) |
| `hermes_cli/plugins.py:106-204` | `VALID_HOOKS` — the exact hook names available to plugins |
| `hermes_cli/plugins_registrars.py:85` | `register_delegation_provider` is a first-class plugin registrar |
| `agent/delegation_provider.py:35` | Delegation route vocabulary (`model`, `provider`, `base_url`, `api_key`, `api_mode`, `request_overrides`, `reasoning_effort`, `service_tier`, `fallback_providers`, `profile`, `label`) |
| `hermes_cli/kanban_db_dispatch.py:1983,2039` | Plugin-supplied `spawn_fn(task, workspace, board=board) -> Optional[int]` — the external-worker-lane seam |
| `hermes_state_messages.py:429` | `SessionDB.append_message` — the internal (unsupported) session-append path |
| `tools/approval_context.py:66-69` | Hook kwargs `turn_id`, `tool_call_id`, `session_id` |
| `tools/registry.py:121-175` | `discover_builtin_tools()` — self-registering tool discovery |
| `agent/delegation_context.py` | `delegated_child_subprocess_env` — the supported way for spawned code to inherit credential/profile policy |
| `~/.hermes/plugins/opencode-free-acp/plugin.yaml` | The plugin manifest format (`manifest_version`, `name`, `kind`, `version`, `requires_hermes`, `license`, `capabilities`) |

---

## D. Environment facts verified on this machine

| Fact | How verified |
|---|---|
| `kiro-cli` **v2.27.1** installed at `/home/deval/.local/bin/kiro-cli` | `kiro-cli --version` |
| Kiro headless flags: `--no-interactive`, `--output-format stream-json`, `--trust-all-tools`, `--trust-tools`, `-r/--resume`, `--resume-id` | `kiro-cli chat --help` |
| Kiro sessions live in `~/.kiro/sessions/cli/` | directory listing |
| Kiro auth DB at `~/.local/share/kiro-cli/data.sqlite3` (outside `KIRO_HOME`) | directory listing + `strings` on the binary |
| A fresh `HOME`+`KIRO_HOME` makes Kiro attempt an interactive browser login | live run with `KIRO_HOME=/tmp/... HOME=/tmp/...` |
| `opencode` **v2.2.5** installed at `/home/deval/.hermes/tools/node-26.12.1-linux-x64/bin/opencode` | `opencode --version` |
| OpenCode exposes `--server`, `--standalone`, `--session/-s`, `--continue/-c`, `--auto`, `opencode serve`, `opencode api`, `opencode acp`, `opencode service` | `opencode --help` |
| `pi`, `cline`, `claude`, `codex`, `ollama`, `kiro` are **NOT installed** | `command -v` |
| `google-chrome` / `google-chrome-stable` present at `/usr/bin/` | `command -v` |
| NVIDIA NIM `/v1/models` returns **84 models**; `moonshotai/kimi-k3.1-pro` is **absent**; `moonshotai/kimi-k2.6` and `minimaxai/minimax-m3.1-flash` are present | authenticated `GET https://integrate.api.nvidia.com/v1/models` |
| `~/.accounts/` does not exist | `ls` |
| SearXNG reachable at `http://127.0.0.1:8080` (HTTP 200) | `curl` |
| FreeLLMAPI service active at `http://127.0.0.1:3001/v1`, model `auto` routes to a live model | `systemctl --user is-active`, `POST /v1/chat/completions` |
| Hermes loads **29** builtin tool modules with no Laya dependency | `discover_builtin_tools()` |

---

## E. Source URLs — complete list

**Hermes docs**
- https://hermes-agent.nousresearch.com/docs/llms.txt
- https://hermes-agent.nousresearch.com/docs/developer-guide/programmatic-integration
- https://hermes-agent.nousresearch.com/docs/developer-guide/plugins
- https://hermes-agent.nousresearch.com/docs/developer-guide/plugin-llm-access
- https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins
- https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks
- https://hermes-agent.nousresearch.com/docs/user-guide/features/credential-pools
- https://hermes-agent.nousresearch.com/docs/user-guide/features/fallback-providers
- https://hermes-agent.nousresearch.com/docs/user-guide/features/provider-routing
- https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban
- https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-worker-lanes
- https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation
- https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp
- https://hermes-agent.nousresearch.com/docs/user-guide/features/web-search
- https://hermes-agent.nousresearch.com/docs/user-guide/features/security
- https://hermes-agent.nousresearch.com/docs/user-guide/import-from-other-agents
- https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage

**Project sites and repos**
- https://herdr.dev/ · https://herdr.dev/docs/
- https://github.com/herdrdev/herdr · https://github.com/herdrdev/herdr-core
- https://github.com/herdrdev/herdr-app · https://github.com/herdrdev/herdr-nix · https://github.com/herdrdev/herdr-layout-examples
- https://github.com/herdrdev/herdr/discussions/741
- https://multica.ai/ · https://github.com/multica-ai/multica
- https://openclaw.ai/ · https://github.com/openclaw/openclaw
- https://github.com/iOfficeAI/AionUi
- https://github.com/pingdotgg/t3code
- https://ccswitch.io/ · https://github.com/farion1231/cc-switch
- https://github.com/shep-ai/cli (301)
- https://github.com/gregce/deadreckon
- https://github.com/GIKSN-RESEARCH/Rinne
- https://github.com/BerriAI/litellm
- https://github.com/musistudio/claude-code-router
- https://github.com/lm-sys/RouteLLM
- https://github.com/anomalyco/opencode
- https://github.com/cline/cline
- https://github.com/anthropics/claude-code
- https://github.com/NousResearch/hermes-agent

**CLI documentation**
- https://kiro.dev/docs/cli/ · https://kiro.dev/docs/cli/headless/ · https://kiro.dev/docs/cli/acp/ · https://kiro.dev/docs/configuration/ · https://kiro.dev/changelog/cli/2-3/
- https://opencode.ai/docs/sdk/
- https://pi.dev/
- https://docs.cline.bot/usage/cli-overview · https://docs.cline.bot/cli/cli-reference
- https://code.claude.com/docs/en/headless · https://code.claude.com/docs/en/cli-reference · https://code.claude.com/docs/en/env-vars · https://code.claude.com/docs/en/authentication

**Prior analysis in this workspace**
- `/home/deval/Projects/multi-account-cli-isolation-analysis.md` (Oct 28, 2026) — the earlier
  independent analysis whose conclusions this research independently confirms and extends
