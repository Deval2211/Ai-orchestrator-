# AionUi

## Repo facts
- **Repository**: iOfficeAI/AionUi (TypeScript frontend + Rust backend)
- **Clone path**: `/home/deval/.hermes/cache/scratch/aionui/` + `/home/deval/.hermes/cache/scratch/aioncore/`
- **License**: Apache-2.0 (confirmed from LICENSE file at root of aionui)
- **Primary language**: TypeScript (packages/ monorepo) + Rust (aioncore crate workspace, 24 crates)
- **Architecture**: Desktop app (Tauri/Electron) with Rust HTTP/WebSocket backend (aioncore) exposing REST + WebSocket APIs; TypeScript frontend packages (`web-host`, `desktop`, etc.)

## Directory structure
```
/home/deval/.hermes/cache/scratch/aionui/
├── packages/
│   ├── desktop/          # Main Tauri desktop app (renderer + main)
│   ├── web-host/         # Backend launcher + agent process registry (Node.js)
│   └── ...               # Other UI packages
├── docs/                 # Documentation (multilingual)
├── LICENSE               # Apache-2.0
/home/deval/.hermes/cache/scratch/aioncore/
├── crates/
│   ├── aionui-app/       # Binary entry, router assembly
│   ├── aionui-runtime/   # Managed Node runtime, subprocess spawn builder
│   ├── aionui-process/   # Supervised subprocess lifecycle, PTY
│   ├── aionui-db/        # SQLite repository layer, migrations
│   ├── aionui-ai-agent/  # Agent lifecycle, ACP integration, worker queues
│   ├── aionui-session/   # Unified session state (CLI + ACP)
│   ├── aionui-mcp/       # MCP protocol integration
│   └── ...               # 18+ other domain/capability/foundation crates
├── ARCHITECTURE.md       # Detailed architecture document
└── Cargo.toml            # Workspace with 24 crates
```

## Agent integrations (actual list, with evidence)

AionUi drives the following coding CLIs/agents via ACP (Agent Client Protocol) — **exact list from the seed data in `aioncore/crates/aionui-db/migrations/001_initial_schema.sql` (lines 186–352)**:

| Agent ID | Backend | Agent Type | Spawn Command |
|----------|---------|------------|---------------|
| `2d23ff1c` | `claude` | `acp` | `bun x --bun @agentclientprotocol/claude-agent-acp@0.29.2` |
| `8e1acf31` | `codex` | `acp` | `bun x --bun @zed-industries/codex-acp@0.9.5` |
| `cc126dd5` | `gemini` | `acp` | `gemini --experimental-acp` |
| `26a946ed` | `qwen` | `acp` | `qwen --acp` |
| `8b20fd41` | `codebuddy` | `acp` | `bun x --bun @tencent-ai/codebuddy-code@2.73.0 --acp` |
| `da386544` | `droid` | `acp` | `droid exec --output-format acp` |
| `600c6601` | `goose` | `acp` | `goose acp` |
| `eb895030` | `auggie` | `acp` | `auggie --acp` |
| `e241c49c` | `kimi` | `acp` | `kimi acp` |
| `53861a53` | `opencode` | `acp` | `opencode acp` |
| `3cd9d436` | `copilot` | `acp` | `copilot --acp --stdio` |
| `1e4afc51` | `qoder` | `acp` | `qoder --acp` |
| `65d0f5b2` | `vibe` | `acp` | `vibe` |
| `a0dfb1ec` | `cursor` | `acp` | `cursor acp` |
| `e044000d` | `kiro` | `acp` | `kiro acp` |
| `55f3ed1c` | `hermes` | `acp` | `hermes acp` |
| `346b0041` | `snow` | `acp` | `snow --acp` |
| `fb1083a5` | (none) | `nanobot` | `nanobot --experimental-acp` |
| `f9f61666` | (none) | `openclaw-gateway` | `openclaw` |
| `632f31d2` | (none) | `aionrs` (internal) | `null` (no CLI spawn) |

**Agent types** defined in `aionui/packages/desktop/src/renderer/utils/model/agentTypes.ts:20`:
```typescript
export type AgentType = 'acp' | 'remote' | 'aionrs' | 'openclaw-gateway' | 'nanobot';
```

All ACP agents use the `bun` bridge binary except `gemini`, `qwen`, `droid`, `auggie`, `goose`, `kimi`, `opencode`, `copilot`, `qoder`, `vibe`, `cursor`, `kiro`, `hermes`, `snow` which spawn directly.

## Hermes compatibility (explicit yes/no + evidence)

**YES — partial.** AionUi includes a builtin Hermes agent entry in the seed data (`aioncore/crates/aionui-db/migrations/001_initial_schema.sql:304–318`):
```sql
('55f3ed1c', '/api/assets/logos/brand/hermes.svg', 'Hermes',
 'hermes', 'acp', 'builtin', '{"binary_name":"hermes"}',
 1, 'hermes', '["acp"]', '[]', NULL,
 '{"supports_side_question":false}', 'yolo', 3130, ...);
```

A dedicated migration exists to NULL its `yolo_id` (`aioncore/crates/aionui-db/migrations/010_hermes_yolo_id_null.sql`):
```sql
UPDATE agent_metadata SET yolo_id = NULL WHERE id = '55f3ed1c' AND backend = 'hermes' AND agent_source = 'builtin';
```

Multiple documentation files reference Hermes (grep -ri hermes):
- `aionui/docs/readme/readme_*.md` (all languages)
- `aionui/readme.md`
- `aionui/CHANGELOG.md`
- `aioncore/crates/aionui-ai-agent/src/services/availability/mod.rs`
- `aioncore/crates/aionui-ai-agent/src/manager/acp/legacy_session_model.rs`
- `aioncore/crates/aionui-ai-agent/src/registry.rs`
- `aioncore/crates/aionui-db/src/repository/sqlite_agent_metadata.rs`
- `aioncore/crates/aionui-session/src/event.rs`

**Limitation**: Hermes is integrated only as an ACP-compatible agent (spawns `hermes acp`). There is no Hermes-specific protocol integration, no Hermes UI embedding, and no evidence of Hermes Agent (Nous Research) being used as the primary user interface. AionUi is its own desktop application.

## Execution architecture: UI-only or reusable runtime?

**Reusable runtime with process management.** AionUi is **not** UI-only — the Rust backend (`aioncore`) provides:

1. **Managed Node runtime** (`aioncore/crates/aionui-runtime/src/node_runtime/` per ARCHITECTURE.md) — bundles Node.js, activates from managed resources or downloads pinned version to `{data_dir}/runtime/node`. Adapter commands carry explicit Node executable; no ambient `PATH` dependence.

2. **Startup PATH enhancement** (`aionui_runtime::enhance_process_path()` called as first statement in `main()`) — merges interactive login-shell `$PATH`, current inherited PATH, platform fallbacks (`~/.cargo/bin`, `~/.local/bin`, version managers, Windows `%APPDATA%\npm`, Git, Scoop).

3. **Subprocess spawn builder** (`aionui_runtime::Builder::agent(program)` / `clean_cli(program)`) — centralised policy: `kill_on_drop(true)`, strips `NODE_OPTIONS`/`NODE_INSPECT`/`NODE_DEBUG`/`CLAUDECODE`, `clean_cli` pipes stdio + sets `NO_COLOR=1` + `TERM=dumb`.

4. **Agent process registry** (`aionui/packages/web-host/src/agent-process-registry.ts`) — tracks `pid`, `process_group_id`, `conversation_id`, `agent_type`, `backend`; on shutdown: SIGTERM → 1s grace → SIGKILL, re-reads registry to preserve concurrently registered processes.

5. **aionui-process crate** — supervised subprocess lifecycle, containment, startup cleanup for direct CLI sessions (ARCHITECTURE.md).

The backend is an Axum HTTP/WebSocket server (`aionui-app` crate) that can run standalone; the TypeScript desktop package connects to it. The runtime crates (`aionui-runtime`, `aionui-process`) are designed as library dependencies.

## Session handling

**SQLite-backed with ACP session persistence.** Schema from `aioncore/crates/aionui-db/migrations/001_initial_schema.sql`:

- **`conversations` table** (lines 65–88): `id`, `user_id`, `name`, `type`, `extra` (JSON), `model`, `status` (`pending`/`running`/`finished`), `source`, `channel_chat_id`, `pinned`, timestamps.
- **`messages` table** (lines 91–109): `id`, `conversation_id`, `msg_id`, `type`, `content` (JSON), `position`, `status` (`finish`/`pending`/`error`/`work`), `hidden`, `created_at`.
- **`acp_session` table** (lines 134–147): `conversation_id` (PK), `agent_backend`, `agent_source`, `agent_id`, `session_id`, `session_status` (`idle` default), `session_config` (JSON), `last_active_at`, `suspended_at`. Indexes on `session_status`, `suspended_at`, `agent_id`.
- **`agent_metadata` table** (lines 153–183): persists agent catalog with `handshake` JSON (capabilities, auth_methods, config_options, available_modes/models/commands), `yolo_id`, `behavior_policy`.

**Session resumption**: The `acp_session` table stores `session_id` and `session_config` enabling ACP session load. `session_status = 'suspended'` with `suspended_at` timestamp supports pause/resume. The `aionui-session` crate provides "unified session state, capabilities, commands, and events across direct CLI and ACP backends" (ARCHITECTURE.md).

## License
**Apache-2.0** — full text in `/home/deval/.hermes/cache/scratch/aionui/LICENSE`. Copyright 2025 AionUi (aionui.com).

## Reuse assessment
| Aspect | Assessment |
|--------|------------|
| **Agent adapter coverage** | Excellent — 17 ACP agents + 3 native types seeded, extensible via `agent_metadata` table and extension registry |
| **Process management** | Production-grade: supervised subprocesses, process-group kill, registry survival, PATH enhancement |
| **Session persistence** | Full SQLite schema with ACP session_id/config, suspend/resume, conversation/message history |
| **MCP integration** | `aionui-mcp` crate + `mcp_servers` table (transport_type, transport_config, tools, OAuth tokens) |
| **Multi-user/team** | `teams`, `mailbox`, `team_tasks` tables; `aionui-team` crate |
| **License compatibility** | Apache-2.0 — compatible with MIT/Apache projects; requires NOTICE preservation |
| **Embeddability** | Backend is a library (`aionui-app` assembles crates); `aionui-runtime`/`aionui-process` usable standalone |
| **Hermes integration** | Only as an ACP agent (`hermes acp`); no Hermes UI or protocol coupling |
| **Gaps for Universal Orchestrator** | No multi-account per provider model; no built-in failover/routing; no cost tracking; designed as opinionated desktop app not headless library |

---

# T3 Code

## Repo facts
- **Repository**: pingdotgg/t3code (TypeScript monorepo)
- **Clone path**: `/home/deval/.hermes/cache/scratch/t3code/`
- **License**: MIT (confirmed from LICENSE file at root) — Copyright 2026 T3 Tools Inc.
- **Primary language**: TypeScript (Effect-TS ecosystem)
- **Architecture**: Monorepo with `packages/` (provider-core, provider-opencode, provider-kiro, provider-claude, contracts, shared, client-runtime) and `apps/` (server, web, mobile)

## Directory structure
```
/home/deval/.hermes/cache/scratch/t3code/
├── apps/
│   ├── server/           # Main backend (orchestration-v2, VCS, MCP)
│   ├── web/              # Web frontend
│   └── mobile/           # Mobile frontend
├── packages/
│   ├── provider-core/    # Core provider abstraction, adapters, driver
│   ├── provider-opencode/# OpenCode adapter (ACP via SSE)
│   ├── provider-kiro/    # Kiro adapter
│   ├── provider-claude/  # Claude Code adapter
│   ├── contracts/        # Shared schemas (Effect Schema), MCP toolkits
│   ├── shared/           # HostProcess, model, observability
│   └── client-runtime/   # Worktree setup, etc.
├── LICENSE               # MIT
└── pnpm-workspace.yaml
```

## Agent orchestration model

**Multiple agent threads via ProviderAdapterV2.** Each provider implements `ProviderAdapterV2` (from `@t3tools/provider-core/server/ProviderAdapter.ts`) which defines:

- **Capabilities** (`OrchestrationV2ProviderCapabilities`): declares per-driver support for sessions, threads, turns, streaming, tools, approvals, planning, subagents, context, checkpointing, identity, runtime policy.
- **Session lifecycle**: `openSession`, `closeSession` — maps to provider-native session (e.g., OpenCode SSE connection).
- **Thread model**: providers expose `providerThread` (native thread) ↔ `appThread` (T3 orchestration thread) mapping; supports fork, fork-from-turn, fork-from-subagent.
- **Turn execution**: streaming events → `node.updated`, `message.updated`, `turn_item.updated`, `provider_turn.updated`, `runtime_request.updated`, `subagent.updated`, `plan.updated`.
- **Subagents**: native `task` tool → child session → child `providerThread` + `appThread` with inherited permissions.

**Agents are registered per provider instance** (`ProviderInstanceId`). The `ProviderHost` manages multiple provider instances. Each adapter (OpenCode, Kiro, Codex, Claude, etc.) implements the same `ProviderAdapterV2` interface.

## Git worktree management (exact module)

**Module**: `apps/server/src/vcs/worktreeRemoval.ts` (primary), `worktreeCheckoutCache.ts`, `worktreeCheckoutCache.test.ts`, `worktreesDirectory.ts`.

**Functionality** (from `worktreeRemoval.ts`):
- `removeLinkedWorktree` effect — safely removes a git worktree with proper locking checks.
- **Isolation per agent**: Each agent thread gets its own worktree (directory-bound). The `worktreeCheckoutCache` caches checkouts to avoid repeated clones.
- **Branch naming**: Not explicitly encoded in removal; checkout cache uses repository + ref.
- **Cleanup**: Handles Windows NTFS junctions vs. Git 2.54+ behavior; reads `.git` file → admin dir → `gitdir` → verifies it points back; checks `locked` file; refuses removal if locked unless double-force; checks for submodules and untracked/modified files (unless `--force`); restores `.git` file on partial delete failure.
- **Cache**: `WorktreeCheckoutCache` (in `worktreeCheckoutCache.ts`) — LRU cache with size limit, background eviction, testkit for deterministic tests.

**Integration**: MCP toolkit `worktree` (in `packages/contracts/src/worktreeMcp.ts` + `worktreeSetup.ts`) exposes worktree operations to agents.

## Model switching

**Supported mid-session via `supportsModelSwitchInSession: true`** in provider capabilities (e.g., OpenCode adapter: `aionui/packages/provider-opencode/src/server/adapter.ts:116`).

**Mechanism**: The `ProviderAdapterV2` runtime policy includes `modelSelection` (`ModelSelection` type from contracts with `instanceId` + `model`). The `turnScopedSelectionTransition` (from `provider-core/server/selectionTransition.ts`) handles model/provider switching within a turn. OpenCode adapter passes `modelSelection.model` to `openSession` and can update via SDK `session.update({ model })`.

**Runtime policy** (`ProviderAdapterV2RuntimePolicy` in `ProviderAdapter.ts:15–30`):
```typescript
{
  runtimeMode: RuntimeMode,
  interactionMode: ProviderInteractionMode,
  cwd: string | null,
  approvalPolicy: unknown | undefined,
  sandboxPolicy: unknown | undefined,
  maxRuntimeMode: RuntimeMode | undefined, // admin ceiling
}
```

## Agent adapter interface

**Exact file**: `/home/deval/.hermes/cache/scratch/t3code/packages/provider-core/src/server/ProviderAdapter.ts`

**Core type**: `ProviderAdapterV2` (lines 930+) — an Effect Service with methods:

```typescript
ProviderAdapterV2.of({
  instanceId: ProviderInstanceId,
  driver: ProviderDriverKind,
  getCapabilities: Effect<OrchestrationV2ProviderCapabilities>,
  planSelectionTransition: Effect<ProviderSelectionTransitionPlan>,
  openSession: Effect<ProviderAdapterV2OpenSessionInput, ..., ProviderAdapterV2OpenSessionOutput>,
  closeSession: Effect<ProviderAdapterV2CloseSessionInput, ..., void>,
  // Event stream: Stream<ProviderAdapterV2Event>
})
```

**Event types** (`ProviderAdapterV2Event` union, lines 100–170):
- `app_thread.created`
- `provider_session.updated`
- `provider_thread.updated`
- `node.updated`
- `message.updated`
- `turn_item.updated`
- `runtime_request.updated`
- `subagent.updated`
- `plan.updated`
- `provider_failure`

**Provider adapters implemented** (in `apps/server/src/orchestration-v2/Adapters/`):
- `ClaudeAdapterV2.ts` — Claude Code
- `CodexAdapterV2.ts` — Codex CLI
- `OpenCode2AdapterV2.ts` — OpenCode (v2 SDK)
- `KiroAdapterV2.ts` — Kiro
- `AntigravityAdapterV2.ts` — Antigravity
- `CursorAdapterV2.ts` — Cursor
- `AcpRegistryAdapterV2.ts` — generic ACP registry

## MCP integration

**Barrel module**: `/home/deval/.hermes/cache/scratch/t3code/packages/contracts/src/mcpToolkits.ts` (re-exports from `index.ts`).

**MCP servers supported**: Dynamic — any MCP server via stdio/HTTP/SSE. The `McpProviderSessions` service (`provider-core/server/McpProviderSessions.ts`) manages per-thread MCP sessions. On `openSession`, OpenCode adapter adds `t3-code` MCP server pointing to local endpoint with auth header (adapter.ts:969–987).

**Per-server tool filter**: Yes. In OpenCode adapter (`adapter.ts:1584–1606`), tool calls with `_` prefix are matched against `mcpServerNames` (fetched via `mcp.status`). The `mcpToolPresentation` helper (`provider-core/server/mcpToolPresentation.ts`) builds presentation metadata per server+tool. Tools are emitted as `dynamic_tool` type with `serverName`, `toolName`, `title`, `input`, `output`.

**Registry**: `McpSessionRegistry` (`apps/server/src/mcp/McpSessionRegistry.ts`) — tracks active MCP sessions per thread, capabilities, endpoints.

## Diff inspection and PR workflow

**Not a dedicated feature in the inspected codebase.** The VCS layer (`apps/server/src/vcs/`) provides:
- `GitVcsDriver` — git operations (status, diff, log, worktree)
- `GitVcsDriverCore` — effectful git execution
- `gitIndexStat` — index vs working tree comparison

No PR creation, diff review UI, or GitHub/GitLab API integration found in the local clone. The `contracts` package has `git.ts` with basic schemas but no PR types.

## License
**MIT** — full text in `/home/deval/.hermes/cache/scratch/t3code/LICENSE`. Copyright 2026 T3 Tools Inc.

## Reuse assessment
| Aspect | Assessment |
|--------|------------|
| **Adapter interface** | Excellent — `ProviderAdapterV2` is a clean Effect-TS service interface; 7+ adapters implemented |
| **Multi-thread orchestration** | Native support: threads, forks, subagents, handoff, checkpointing |
| **Worktree isolation** | Production-grade: per-thread worktrees, caching, safe removal with lock/submodule checks |
| **Model switching** | Mid-session via runtime policy; declared in capabilities |
| **MCP integration** | First-class: per-thread sessions, dynamic tool discovery, presentation metadata |
| **License** | MIT — maximally permissive |
| **Embeddability** | Core packages (`provider-core`, `contracts`, `shared`) are library crates; `apps/server` is the composed application |
| **Gaps for Universal Orchestrator** | No multi-account per provider; no failover/routing; no cost tracking; Effect-TS runtime required; tightly coupled to T3 orchestration model |

---

# CC Switch

## Repo facts
- **Repository**: farion1231/cc-switch (Rust + TypeScript Tauri desktop app)
- **Clone path**: `/home/deval/.hermes/cache/scratch/ccswitch/`
- **License**: MIT (confirmed from LICENSE file at root) — Copyright 2025 Jason Young
- **Primary language**: Rust (Tauri backend) + TypeScript (frontend)
- **Architecture**: Tauri 2.x desktop app (`src-tauri/`); core logic in `src-tauri/src/`; `src-tauri/Cargo.toml` defines `crate-type = ["rlib"]` (library + Tauri binary)

## Directory structure
```
/home/deval/.hermes/cache/scratch/ccswitch/
├── src-tauri/
│   ├── src/
│   │   ├── main.rs              # Tauri entry, windows_subsystem = "windows"
│   │   ├── lightweight.rs       # Headless/tray-only mode (destroys main window)
│   │   ├── store.rs             # Global AppState (DB, ProxyService, UsageCache, CodexOAuthManager)
│   │   ├── provider.rs          # Provider struct, UniversalProvider, ManagedAccount, auth bindings
│   │   ├── hermes_config.rs     # Reads/writes ~/.hermes/config.yaml
│   │   ├── settings.rs          # App settings, WebDAV sync, encryption helpers
│   │   ├── proxy/
│   │   │   ├── provider_router.rs  # Circuit breaker, failover chain, request routing
│   │   │   ├── handler_context.rs  # Per-request context, auto_failover logic
│   │   │   └── handlers.rs         # HTTP handlers, streaming, non-streaming
│   │   ├── services/
│   │   │   ├── provider/       # Live config, Codex official models, Codex direct
│   │   │   ├── claude_account/ # Account snapshot, switch, login, layout
│   │   │   ├── usage_stats.rs  # Token usage, cost calculation, per-app breakdown
│   │   │   └── proxy.rs        # Proxy service lifecycle
│   │   ├── database/           # SQLite (rusqlite), DAOs, migrations
│   │   └── commands/           # Tauri command handlers
│   ├── Cargo.toml
│   └── tauri.conf.json
├── src/                        # TypeScript frontend (React)
├── LICENSE                     # MIT
└── README.md
```

## Provider + multi-account model (exact answer: N accounts per provider supported? how?)

**YES — multiple accounts per provider supported via `ManagedAccount` + `authBinding`.**

**Evidence from `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/provider.rs`:**

1. **`UniversalProvider` struct** (lines 786+) contains `managed_accounts: Vec<ManagedAccount>`.
2. **`ManagedAccount` struct** (lines 382–420):
   ```rust
   pub struct ManagedAccount {
       pub id: String,                    // unique account ID
       pub name: String,                  // display name
       #[serde(rename = "accountId")]
       pub account_id: Option<String>,    // provider-specific account identifier
       pub auth_binding: AuthBinding,     // links to provider auth
       pub real_model: Option<String>,    // upstream model name (internal only)
       pub display_model: Option<String>, // labelOverride for Claude Desktop
       pub enabled: bool,
       // ...
   }
   ```
3. **`AuthBinding`** (lines 548–704) — generic credential binding:
   ```rust
   pub enum AuthBindingSource { ManagedAccount, EnvironmentVariable, Profile, ... }
   pub struct AuthBinding {
       pub source: AuthBindingSource,
       #[serde(rename = "accountId")]
       pub account_id: Option<String>,
       pub auth_provider: String,  // e.g., "github", "google", "anthropic"
       // legacy: github_account_id for backward compat
   }
   ```
4. **`Provider::managed_account_id_for(&self, auth_provider: &str) -> Option<String>`** (lines 693–705) — resolves account ID for a given auth provider, reading from `authBinding` (new) with fallback to `github_account_id` (legacy).

**How a second account is modelled**: Add a `ManagedAccount` entry to `UniversalProvider.managed_accounts` with its own `auth_binding` pointing to the credential source. The provider routes requests via `auth_binding.account_id` to select the correct upstream identity.

**Provider config schema** (`Provider` struct, lines 10–44): `id`, `name`, `settings_config` (JSON), `website_url`, `category`, `meta` (ProviderMeta with `provider_type`, `api_key_field`, `opencode_config_format`), `icon`, `icon_color`, `in_failover_queue`.

**API endpoint management**: `settings_config` holds `base_url`, `api_key`, `models[]`, `capabilities[]`, `context_limit`, `model_protocols`, `model_enabled`, `model_health`, `bedrock_config`. The proxy reads this per-request.

**Model configuration**: Per-provider `models` array in `settings_config`; `UniversalProvider` adds `real_model` (upstream) + `display_model` (Claude Desktop labelOverride) per managed account.

## Failover / routing

**YES — local proxy with circuit breaker + failover chain.**

**Evidence from `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/proxy/provider_router.rs` and `handler_context.rs`:**

- **`ProviderRouter`** (provider_router.rs) — wraps `reqwest::Client` + `CircuitBreaker` per provider.
- **Circuit breaker config** (from `circuit_breaker.rs`, accessed via `db.get_proxy_config_for_app(app_type)`): `failure_threshold`, `success_threshold`, `timeout`, `half_open_requests`.
- **Failover chain** (`handler_context.rs:299`): `failover_chain(auto_failover_enabled)` — when enabled, on upstream error (5xx, timeout, circuit open), automatically retries next provider in queue.
- **`in_failover_queue` flag** on `Provider` struct (line 42) — marks providers eligible for failover.
- **Auto-failover toggle** per app type (Claude, Codex) via `proxy_config` table; `handler_context.rs:133, 168, 181, 256, 273, 322` gate failover logic.
- **Request routing**: `handler_context.rs` builds `ProviderRequestContext` with resolved provider + fallback chain; `handlers.rs:759, 1712` apply timeout + failover for streaming/non-streaming.

**Not a generic LLM proxy** — tailored for Claude Code, Codex, Gemini CLI protocols; translates between client (Claude Desktop format) and upstream APIs.

## Usage + cost tracking

**YES — per-request token usage + cost persisted in SQLite.**

**Evidence from `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/services/usage_stats.rs`:**

- **`UsageSummary` struct** (lines 24–33): `total_cost` (String), `total_input_tokens`, `total_output_tokens`, `total_cache_creation_tokens`, `total_cache_read_tokens`, `real_total_tokens` (input+output+cache_creation+cache_read), `cache_hit_rate`.
- **`AppTypeUsageSummary`** (lines 39–47) — breakdown per app type (Claude, Codex, etc.).
- **Pricing** (`proxy/usage/calculator.rs::ModelPricing`) — per-model input/output/cache pricing.
- **Storage**: `usage_stats` table (inferred from `usage_summary_from_row` reading 6 columns: cost, input, output, cache_creation, cache_read, real_total).
- **Per-session**: `get_session_usage_summary` — filters by session.
- **Dashboard**: hero "real consumption" number = `real_total_tokens`.

**Cost calculation**: `total_cost = Σ (tokens × model_price)` using `ModelPricing` table; stored as string to preserve precision.

## Credential storage (exact location and protection)

**Location**: `~/.config/cc-switch/` (Linux/macOS) or `%APPDATA%\cc-switch\` (Windows) — via `config::get_app_config_dir()`.

**Files** (from `settings.rs`, `provider.rs`, `claude_account` module):
- `config.json` — main config (providers, accounts, settings); **plaintext JSON**.
- `auth.json` — Claude/Codex OAuth tokens (managed by `claude_account` snapshot logic); **plaintext JSON**.
- `hermes_config.yaml` — read/written by `hermes_config.rs` for Hermes Agent integration; **plaintext YAML**.

**Protection**: **No OS keychain / no encryption at rest found.**
- `settings.rs` has `atomic_write` (write to temp + rename) for durability, not confidentiality.
- `Cargo.toml` dependencies: no `keyring`, `secret-service`, `security-framework`, `windows-credman`, or crypto crates (e.g., `aes-gcm`, `chacha20poly1305`, `age`, `scrypt`).
- `provider.rs` comment (lines 90–98): "Third-party managed OAuth … the real credential is injected per-request by the local proxy, so the card is keyless by design and its stored config is only an upstream snapshot." — implies OAuth tokens live in upstream `auth.json`, not encrypted locally.

**Conclusion**: Credentials stored in **plaintext files** in app config directory. Relies on filesystem permissions only. Not suitable for high-security environments without external encryption (e.g., encrypted home, Vault).

## Headless/library usability

**Partially — core is a Rust library (`cc_switch_lib` rlib), but wired to Tauri lifecycle.**

**Evidence**:
- `src-tauri/Cargo.toml`: `crate-type = ["rlib"]` — produces a Rust library.
- `src-tauri/src/main.rs` — Tauri entry point; `tauri::Builder` invokes commands, manages windows, tray, plugins.
- `src-tauri/src/lightweight.rs` — `enter_lightweight_mode()` / `exit_lightweight_mode()`: destroys main WebView window, keeps tray + background services (proxy, session manager) running. `LIGHTWEIGHT_MODE` atomic flag gates UI.
- `src-tauri/src/store.rs` — `AppState` (DB, ProxyService, UsageCache, CodexOAuthManager) constructed in `AppState::new()`; no Tauri types in signature — can be instantiated independently.
- `src-tauri/src/services/proxy.rs` — `ProxyService` runs background HTTP proxy; `start()`/`stop()` methods.

**Limitations**:
- Database path hardcoded to `get_app_config_dir()` (uses Tauri `app.handle().path().app_config_dir()` in practice).
- Tauri plugins (log, opener, process, updater, dialog, store, deep-link, window-state) initialized in `main.rs`.
- Commands (`src-tauri/src/commands/`) use `#[tauri::command]` — need Tauri IPC to invoke.
- No documented public Rust API surface; internal modules not re-exported from `lib.rs`.

**Verdict**: The **core logic (provider routing, usage tracking, account management) is in a reusable rlib**, but the **application wiring (config dir, DB, Tauri plugins, command IPC) assumes Tauri runtime**. Could be embedded in another Rust binary with manual `AppState` construction, but not a turnkey headless library.

## License
**MIT** — full text in `/home/deval/.hermes/cache/scratch/ccswitch/LICENSE`. Copyright 2025 Jason Young.

## Reuse assessment
| Aspect | Assessment |
|--------|------------|
| **Multi-account per provider** | ✅ Yes — `ManagedAccount` + `AuthBinding` model; N accounts per provider |
| **Provider config + endpoints** | ✅ JSON `settings_config` with base_url, models, capabilities; per-account model override |
| **Failover/routing** | ✅ Circuit breaker + failover chain per app type; proxy injects OAuth per-request |
| **Usage + cost tracking** | ✅ Per-request tokens + cost; SQLite persistence; per-app/session breakdown |
| **Credential storage** | ❌ Plaintext files in config dir; no OS keychain / encryption |
| **Headless/library** | ⚠️ Partial — rlib exists but tied to Tauri config dir, plugins, IPC |
| **License** | ✅ MIT — permissive |
| **Gaps for Universal Orchestrator** | No encryption; no documented library API; Tauri-coupled config paths; Claude/Codex/Gemini-specific protocol translation |

## Limitations

1. **Plaintext credentials** — `config.json`, `auth.json`, `hermes_config.yaml` unencrypted. Blocker for enterprise / shared-device use.
2. **No public library API** — Internal modules (`provider`, `proxy`, `services`, `database`) not re-exported; `AppState` constructible but config dir resolution uses Tauri APIs.
3. **Protocol coupling** — Proxy translates Claude Desktop ↔ upstream (Anthropic, OpenAI, Gemini, Codex); adding new client protocols requires Rust changes.
4. **Tauri runtime required** — Even in lightweight mode, `tauri::AppHandle` needed for tray, plugins, config dir. Cannot run as pure systemd service without Tauri.
5. **Single-user design** — `AppState` is global singleton; no multi-tenant isolation.
6. **No Hermetic builds** — `tauri-build` + system WebKitGTK on Linux; not trivially reproducible.

## Evidence (every file path and URL)

### AionUi
- `/home/deval/.hermes/cache/scratch/aionui/LICENSE` — Apache-2.0 license
- `/home/deval/.hermes/cache/scratch/aioncore/ARCHITECTURE.md` — Full architecture
- `/home/deval/.hermes/cache/scratch/aioncore/crates/aionui-db/migrations/001_initial_schema.sql` — SQLite schema + seed data (agent_metadata, acp_session, conversations, messages, providers, teams, cron)
- `/home/deval/.hermes/cache/scratch/aioncore/crates/aionui-db/migrations/010_hermes_yolo_id_null.sql` — Hermes yolo_id migration
- `/home/deval/.hermes/cache/scratch/aionui/packages/desktop/src/renderer/utils/model/agentTypes.ts` — AgentType enum, AgentMetadata, ManagedAgent
- `/home/deval/.hermes/cache/scratch/aionui/packages/web-host/src/agent-process-registry.ts` — Process registry, cleanup, SIGTERM/SIGKILL
- `/home/deval/.hermes/cache/scratch/aionui/readme.md` — Project readme
- `grep -ri hermes` results — Hermes references across codebase

### T3 Code
- `/home/deval/.hermes/cache/scratch/t3code/LICENSE` — MIT license
- `/home/deval/.hermes/cache/scratch/t3code/packages/provider-core/src/server/ProviderAdapter.ts` — ProviderAdapterV2 interface, events, runtime policy
- `/home/deval/.hermes/cache/scratch/t3code/packages/provider-opencode/src/server/adapter.ts` — OpenCode adapter (3801 lines), capabilities, session/thread/turn/subagent handling, MCP tool filter
- `/home/deval/.hermes/cache/scratch/t3code/apps/server/src/vcs/worktreeRemoval.ts` — Worktree removal with lock/submodule checks
- `/home/deval/.hermes/cache/scratch/t3code/apps/server/src/vcs/worktreeCheckoutCache.ts` — Worktree checkout caching
- `/home/deval/.hermes/cache/scratch/t3code/packages/contracts/src/mcpToolkits.ts` — MCP toolkits barrel (via index.ts)
- `/home/deval/.hermes/cache/scratch/t3code/apps/server/src/orchestration-v2/Adapters/` — Adapter implementations (Claude, Codex, OpenCode, Kiro, Antigravity, Cursor, ACP Registry)

### CC Switch
- `/home/deval/.hermes/cache/scratch/ccswitch/LICENSE` — MIT license
- `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/provider.rs` — Provider, UniversalProvider, ManagedAccount, AuthBinding, auth_binding resolution
- `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/proxy/provider_router.rs` — Circuit breaker, failover chain, provider routing
- `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/proxy/handler_context.rs` — Request context, auto_failover gating
- `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/proxy/handlers.rs` — HTTP handlers with failover
- `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/services/usage_stats.rs` — UsageSummary, cost calculation, per-app breakdown
- `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/settings.rs` — Config dir, atomic_write, hermes_config helpers
- `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/hermes_config.rs` — Reads/writes ~/.hermes/config.yaml
- `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/lightweight.rs` — Headless/tray mode, window destruction
- `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/store.rs` — AppState (DB, ProxyService, UsageCache)
- `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/main.rs` — Tauri entry, plugin init
- `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/Cargo.toml` — Crate config, rlib, Tauri deps
- `/home/deval/.hermes/cache/scratch/ccswitch/src-tauri/src/services/claude_account/snapshot/switch.rs` — Account switch (check_out/vacate)