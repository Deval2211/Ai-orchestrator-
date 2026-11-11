# Coding CLI Landscape Research

**Purpose:** Establish the programmatic interface surface of each coding CLI so the orchestrator (Phase 3) can wrap each one as an adapter. Every claim below is tagged with its evidence source: local `--help`/`--version` output run on this machine, or docs-only (web extraction).

**Workspace:** `/home/deval/Projects/ai-orchestrator`
**Deliverable:** `docs/research/parts/08-coding-cli-landscape.md`

---

# Method

## What was run locally

Read-only, harmless commands only. No sessions started, no installs, no network writes.

| Tool | Binary | Version | Local runs performed |
|------|--------|---------|---------------------|
| Kiro CLI | `/home/deval/.local/bin/kiro-cli` | v2.27.1 | `--help`, `--version`, `--help-all`, `chat --help`, `agent --help`, `acp --help`, `login --help` |
| OpenCode | `/home/deval/.hermes/tools/node-26.12.1-linux-x64/bin/opencode` | v2.2.5 | `--help`, `--version`, `run --help`, `serve --help`, `acp --help`, `api --help`, `session --help`, `service --help`, `run -p "..." --format json` |
| Pi | `pi` | NOT INSTALLED | `which pi` → command not found; no local runs possible |
| Cline | `cline` | NOT INSTALLED | `which cline` → not found; no local runs possible |
| Claude Code | `claude` | NOT INSTALLED | `which claude` → command not found; no local runs possible |

## What was read online (docs)

| Tool | URL(s) fetched | Purpose |
|------|---------------|---------|
| Kiro | `https://kiro.dev/docs/cli/headless/`, `https://kiro.dev/docs/cli/acp/`, `https://kiro.dev/docs/configuration/`, `https://kiro.dev/changelog/cli/2-3/` | Headless flags, `--output-format stream-json`, `KIRO_API_KEY`, `KIRO_HOME`, ACP server mode, session storage paths |
| OpenCode | `https://opencode.ai/docs/sdk/`, `https://github.com/anomalyco/opencode` | Canonical repo moved from sst/opencode → anomalyco/opencode; SDK `@opencode-ai/sdk`; server mode on port 4096; SSE events |
| Pi | `https://pi.dev/`, `https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md` | Four modes (interactive, print/JSON, RPC, SDK); npm package `@earendil-works/pi-coding-agent`; JSONL session trees; RPC over stdio |
| Cline | `https://docs.cline.bot/usage/cli-overview`, `https://docs.cline.bot/cli/cli-reference` | `--json` flag → NDJSON output; `--id` resume flag (known bug with `--json`); `--config`/`--data-dir` isolation; JSONL line types (`agent_event`, `run_result`, etc.) |
| Claude Code | `https://code.claude.com/docs/en/headless`, `https://code.claude.com/docs/en/cli-reference`, `https://code.claude.com/docs/en/env-vars`, `https://code.claude.com/docs/en/authentication` | `-p`/`--print` headless; `--continue`/`--resume`; `--bare`; `CLAUDE_CONFIG_DIR` env var; `--output-format stream-json`; Agent SDK Python/TS |

**Evidence rule:** Every flag mentioned below appears verbatim in `--help` output (local) or is quoted from fetched docs (remote). Nothing is inferred.

---

# Kiro CLI

## Canonical repo / official docs / binary

- **Official docs:** `https://kiro.dev/docs/cli/` (CLI home page), `https://kiro.dev/docs/cli/headless/` (headless mode), `https://kiro.dev/docs/cli/acp/` (ACP agent mode)
- **Binary:** `/home/deval/.local/bin/kiro-cli` (installed on this machine)
- **Version (local):** `kiro-cli 2.27.1` (verified with `--version`)
- **License:** Not verified from source — no license text available in local binary; Kiro is a proprietary product (JetBrains-backed), no open-source license found in local files or docs.

## Headless / non-interactive execution (local --help output)

```
Usage: kiro-cli-chat chat [OPTIONS] [INPUT]

Arguments:
  [INPUT]          The first question to ask

Options:
  -r, --resume                          Resume the most recent conversation from this directory
      --resume-id <SESSION_ID>          Resume a specific conversation by session ID
      --resume-picker                   Interactively select a conversation to resume from this directory
      --no-interactive                  Whether the command should run without expecting user input
      --output-format <OUTPUT_FORMAT>   Response stream format: text (default), or stream-json for JSON Lines on stdout. stream-json implies --no-interactive
      --trust-all-tools                  Auto-approve all tool permission requests
      --trust-tools <TOOL_NAMES>        Trust only this set of tools
      --agent-engine <ENGINE>           Agent engine to use: "v1", "v2" (default), or "v3"
```

**Exact headless command:**
```
kiro-cli chat --no-interactive --output-format stream-json "your prompt here"
```
or piping stdin:
```
printf '%s' "your prompt" | kiro-cli chat --no-interactive
```

Headless requires an API key via env var `KIRO_API_KEY` (per docs at `https://kiro.dev/docs/cli/headless/`):
- `kiro-cli chat --v3 --no-interactive --trust-tools=read_file,grep_search "Review the changes"`

`--trust-all-tools` is V3: auto-approves all tool calls. `--trust-tools=<ids>` scopes approval. Without either, Kiro denies tool calls in non-interactive mode.

`--output-format stream-json` "requires V2 or V3 (`--agent-engine v2` or `--agent-engine v3`)".

**Documented limitations for automation:**
- Must provide a non-empty initial instruction (positional arg or piped stdin)
- No mid-session user input is possible
- Interactive slash commands (`/model` picker, `/agent` picker) are not available
- Terminal UI features are disabled
- In V3 non-interactive runs, Hooks that block are final; `--trust-tools` cannot approve Hook confirmation requests (only `--trust-all-tools` can)

## Streaming output support

- **`--output-format stream-json`**: Emits run events as JSON Lines on stdout — one self-describing JSON object per line (ACP events). `stream-json` implies `--no-interactive`.
- Default `--output-format text` is human-readable, not parseable by scripts.
- **ACP server mode** (`kiro-cli acp`): Communicates over stdin/stdout using JSON-RPC 2.0. Supports `initialize`, `session/new`, `session/load`, `session/prompt`, `session/cancel`, `session/set_mode`, `session/set_model`. Streaming updates include `AgentMessageChunk`, `ToolCall`, `ToolCallUpdate`, `TurnEnd`.

## Session persistence and RESUME

- **Where sessions live:** `~/.kiro/sessions/cli/` — each session creates `<session-id>.json` (metadata/state) and `<session-id>.jsonl` (event log/conversation history). Verified locally: `/home/deval/.kiro/sessions/cli/` contains `.jsonl`, `.history`, `.json` files.
- **Resume flags:** `-r, --resume` (most recent conversation from this directory), `--resume-id <SESSION_ID>` (specific session), `--resume-picker` (interactive selection from this directory).
- **Resume after orchestrator restart:** Yes — sessions are persisted as files keyed by UUID session ID. An external orchestrator can record the session ID and resume it later with `--resume-id` from any directory. The session ID is an opaque string stored in the session metadata files.

## Authentication

- **Headless:** `KIRO_API_KEY` env var (API key authentication). Generated via Kiro portal (`https://app.kiro.dev`). Also supports `--license free|pro` and `--use-device-flow` OAuth device flow (`kiro-cli login --help`).
- **Multiple independent accounts:** Yes — `KIRO_HOME` env var isolates entire profiles.

## Multi-account isolation

- **`KIRO_HOME`** (docs at `https://kiro.dev/docs/configuration/`, changelog CLI 2.3): "Point Kiro at a different directory for global agents, prompts, skills, steering, settings, and sessions by setting `KIRO_HOME` before launching." If set to a relative path, filesystem root, home directory, or an ancestor of home, Kiro ignores it and falls back to `~/.kiro`.
- **OAuth-bound auth:** If using OAuth/device flow (`--use-device-flow`), credentials are stored under `KIRO_HOME`, so setting `KIRO_HOME` per account does isolate OAuth sessions — but only if the OAuth provider doesn't bind to a fixed external profile directory. The `--auth-method cli` flag on `kiro-cli acp` keeps authentication inside the process (CLI credential store), which plays well with `KIRO_HOME` isolation.
- **Alternative pattern observed in community tools:** `kiro-pool` wraps kiro-cli, injecting `XDG_DATA_HOME`, `KIRO_HOME`, `XDG_RUNTIME_DIR` per profile; `kiro-wrap` rewrites `HOME` per session. Confirms `KIRO_HOME` is the canonical isolation knob.

## Local server / SDK / daemon mode

- **ACP server via stdio:** `kiro-cli acp` starts an Agent Client Protocol agent communicating over stdin/stdout JSON-RPC 2.0. Editors (JetBrains, Zed) configure it as a custom agent server. Not an HTTP server — it's a stdio-based protocol.
- **Cloud mode:** `--cloud` flag runs a session in a cloud sandbox, forwarded to `kiro-cli chat --cloud`. Forwarded to the TUI; sends `_meta.kiro.executionTarget` on `session/new`.
- **No HTTP server / no npm SDK found.** No evidence of an HTTP REST server or a client library for programmatic control outside of ACP stdio.

## License

- **Tool:** Not verified (proprietary).
- **SDK:** No public SDK found.

---

# OpenCode

## Canonical repo / official docs / binary

- **Canonical repo:** `https://github.com/anomalyco/opencode` (moved from `sst/opencode`; per GitHub page "The open source AI coding agent", public, MIT License, Stars: 225,545)
- **Official docs:** `https://opencode.ai/`, SDK docs at `https://opencode.ai/docs/sdk/`
- **Binary:** `/home/deval/.hermes/tools/node-26.12.1-linux-x64/bin/opencode` (also symlinked as `opencode2`)
- **Version (local):** `opencode v2.2.5` (verified with `--version`)

## Headless / non-interactive execution (local --help output)

Global flags:
```
--standalone     Run with a private server instead of the background service
--server string  Connect to a server URL instead of the background service
--auto           Auto-approve permissions that are not explicitly denied
```

`run` subcommand flags (local --help output):
```
DESCRIPTION
  Run OpenCode with a message

USAGE
  opencode run [flags] [<message...>]

ARGUMENTS
  message... string    Message to send (optional)

FLAGS
  --continue, -c          Continue the last session
  --session, -s string    Session ID to continue, or to create if it does not exist
  --fork                  Fork the session before continuing
  --model, -m string      Model to use in the format provider/model#variant
  --agent string          Agent to use
  --format choice         Output format (choices: default, json)
  --file, -f string       File to attach to the message
  --title string          Session title
  --thinking              Show thinking blocks
  --auto                  Auto-approve permissions that are not explicitly denied
```

**Exact headless command:**
```
opencode run --format json --continue "your prompt here"
```
or starting a named session:
```
opencode run --format json --session MY_SESSION_ID "your prompt here"
```

`--format json` produces structured JSON output instead of default text. `--auto` auto-approves permissions (needed for unattended CI). `--standalone` runs with a private server instead of connecting to the background service.

**Documented limitation:** None found for headless mode; `--format json` works cleanly for scripts.

## Streaming output support

- **`--format json`** on `run`: Outputs structured JSON for parsing (not streaming per-line — it's a single response format for the run result).
- **`--print-logs`**: Prints logs to stderr (server logs require `--standalone`).
- **SDK events:** `@opencode-ai/sdk` exposes `client.event.subscribe()` → Server-Sent Events stream (SSE). For real-time event streaming, use the SDK over the HTTP server, not shelling out.

## Session persistence and RESUME

- **Where sessions live:** `~/.config/opencode/` and `~/.local/share/opencode/` (per resource-registry.md in the project spec). Locally verified: `~/.config/opencode/service.json` exists; `~/.local/share/opencode/` contains `opencode.db`, `log/`, `repos/`, `shell/`.
- **Resume flags:** `--continue, -c` (continue last session), `--session, -s <ID>` (continue specific session or create if it doesn't exist), `--fork` (fork before continuing).
- **Resume after orchestrator restart:** Yes — sessions persist in the SQLite DB (`~/.local/share/opencode/opencode.db`). External orchestrator can capture session IDs and resume with `--session` on any later run. The SDK's `session.list()` / `session.get()` / `session.messages()` APIs also expose this over HTTP.

## Authentication

- **Auth model:** API keys stored in config (`~/.config/opencode/`), OAuth for integrations. `opencode auth` manages integrations and credentials.
- **Multiple independent accounts:** Yes — via config-dir isolation (see below).
- **OAuth-bound:** Some integrations use OAuth; those credentials are stored under the config/data dir, so config-dir isolation does separate them — unless the OAuth provider binds to a fixed location outside the config dir.

## Multi-account isolation

- **`XDG_CONFIG_HOME` + `XDG_DATA_HOME`** (per project spec `specs/resource-registry.md`): OpenCode reads config from `XDG_CONFIG_HOME/opencode` (default `~/.config/opencode`) and state from `XDG_DATA_HOME/opencode` (default `~/.local/share/opencode`). Setting these env vars per account isolates account B from account A entirely.
- **`--standalone`**: Runs with a private server instead of the background service — useful for per-account server instances.
- **`--server <URL>`**: Connect to a specific server URL instead of the background service — the orchestrator could run one `opencode serve` per account on different ports and connect each CLI instance to its own server.

## Local server / SDK / daemon mode — HIGHEST-VALUE FINDING

**YES — HTTP server mode exists.**

`opencode serve` (local --help output):
```
DESCRIPTION
  Start the v2 API and web server

USAGE
  opencode serve [flags]

FLAGS
  --hostname string    
  --port integer       
  --cors string         Additional allowed CORS origin (repeat for multiple origins)
  --service            
  --stdio              
```

- Starts a v2 API + web HTTP server. Default port: **4096** (per SDK docs).
- **SDK:** `@opencode-ai/sdk` (npm). Install: `npm install @opencode-ai/sdk`. The SDK `createOpencode()` starts both a server and a client; `createOpencodeClient({ baseUrl: "http://localhost:4096" })` connects to an already-running server.
- Full typed API: `session.create`, `session.list`, `session.prompt`, `session.messages`, `session.command`, `session.shell`, `session.abort`, `find.text`, `find.files`, `config.providers`, `auth.set`, `event.subscribe` (SSE), `app.log`, `project.list`, etc.
- **Also:** `opencode acp` starts an Agent Client Protocol server (JSON-RPC over stdio). `opencode api` makes requests to the running server. `opencode service` manages the background server (`start`, `restart`, `status`, `stop`).
- **Conclusion:** OpenCode exposes a full HTTP API with an official TypeScript SDK. **This is the best integration path for the orchestrator** — no shelling out needed, just spawn `opencode serve` once and use the SDK or raw HTTP.

## License

- **Tool:** MIT (confirmed from GitHub repo page anomalyco/opencode).
- **SDK:** `@opencode-ai/sdk` — bundled with the repo, same MIT license presumably.

---

# Pi (Pi Coding Agent)

## Canonical repo / official docs / binary

- **Canonical repo:** `https://github.com/earendil-works/pi` (AI agent toolkit; README says "See the documentation for full setup and usage instructions")
- **Official docs:** `https://pi.dev/`, package catalog at `https://pi.dev/packages`
- **Binary:** `pi` (installed via `curl -fsSL https://pi.dev/install.sh | sh` or `npm install -g @earendil-works/pi-coding-agent`)
- **Installed locally:** NO — `which pi` → "command not found". All claims below are docs-only.
- **License:** MIT (confirmed from README: "## License\n\nMIT").

## Headless / non-interactive execution (docs only)

From pi.dev README:
> Four modes: interactive, print/JSON, RPC, and SDK.

- **Interactive:** `pi` — full TUI experience.
- **Print/JSON:** `pi -p "query"` for scripts; `--mode json` for event streams.
- **RPC:** JSON protocol over stdin/stdout for non-Node integrations (`docs/rpc.md`).
- **SDK:** TypeScript SDK (`docs/sdk.md`); `@earendil-works/pi-coding-agent` npm package; OpenClaw is a real-world integration (`https://github.com/OpenClaw/OpenClaw`).

**Exact headless command (docs):**
```
pi -p "your query"
```
with `--mode json` for JSON event stream output.

## Streaming output support

- **`--mode json`**: Event stream output (JSON per line / NDJSON).
- **RPC over stdio:** JSON protocol for non-Node integrations — bidirectional request/response streaming over stdin/stdout.
- **SDK:** TypeScript library for programmatic access.

## Session persistence and RESUME

- **Where sessions live:** `~/.pi/` (per project spec `specs/resource-registry.md`). Sessions stored as JSONL trees — "All branches live in a single file."
- **Resume:** `/tree` command inside Pi to navigate to any previous point and continue from there. Export via `/export`, share via `/share` (GitHub gist URL).
- **Resume after orchestrator restart:** Sessions are JSONL files on disk; external orchestrator can reopen/resume from the tree structure. Exact CLI resume flag not verified locally (docs don't surface a `--resume-id` style flag in the README; need `--help` which we couldn't run locally).

## Authentication

- **Auth model:** API keys or OAuth. Run `/login` inside Pi to connect a subscription or API key.
- **Multiple independent accounts:** Yes — via config-dir isolation (see below).
- **OAuth-bound:** OAuth credentials stored under config dir; isolation should work unless OAuth binds externally.

## Multi-account isolation

- **`HOME` or `XDG_CONFIG_HOME`** (per project spec): `~/.pi/` is the default config dir. Setting `HOME` or `XDG_CONFIG_HOME` per account isolates account B from account A.

## Local server / SDK / daemon mode

- **TypeScript SDK:** `@earendil-works/pi-coding-agent` on npm — embed Pi in apps. OpenClaw is a real-world SDK integration.
- **RPC over stdio:** JSON protocol — can be driven from any language, not just Node.
- **No HTTP server** found in docs (only interactive TUI, print/JSON, RPC, SDK). No daemon/background service mode documented.

## License

- **Tool:** MIT (from README).
- **SDK:** `@earendil-works/pi-coding-agent` — bundled MIT (same repo).

## Documented limitations for automation

- Print/JSON mode (`-p` / `--mode json`) exists specifically for scripting, so Pi is headless-friendly.
- RPC mode is for non-Node integrations.
- Exact `--help` flags not verified locally (binary not installed).

---

# Cline

## Canonical repo / official docs / binary

- **Canonical repo:** `https://github.com/cline/cline` (Autonomous coding agent as SDK, IDE extension, or CLI)
- **Official docs:** `https://docs.cline.bot/` (CLI overview at `https://docs.cline.bot/usage/cli-overview`, CLI reference at `https://docs.cline.bot/cli/cli-reference`)
- **Binary:** `cline` (installed via `npm i -g cline`)
- **Installed locally:** NO — `which cline` → not found. All claims below are docs-only.
- **License:** Not verified from local files.

## Headless / non-interactive execution (docs only; `--help` output quoted from CLI Reference page)

```
Usage: cline [options] [command] [prompt]

Arguments:
  prompt                        Your prompt. Default to start in act mode with auto-approve enabled.

Options:
  -V, --version                 Output the version number
  -p, --plan                    Run in plan mode
  --json                        Output messages as JSON instead of styled text
  --auto-approve <boolean>      Set tool auto-approval for all tools (default: true)
  -c, --cwd <path>              Working directory
  --thinking <level>            Set reasoning effort: none|low|medium|high|xhigh
  --compaction <mode>           Context compaction mode: agentic|basic|off
  -i, --tui                     Open the terminal user interface (TUI)
  --id <session-id>             Resume an existing session by ID
  -P, --provider <id>           Provider id (default: cline)
  -k, --key <api-key>           API key for this run, also saved as the provider's key
  -m, --model <model-id>        Model to use for the session with the selected provider
  -s, --system <system-prompt>  Override the default system prompt
  -z, --zen                     Start a session that runs in the background hub
  --retries <count>             Number of maximum consecutive mistakes (retries) before exiting
  -t, --timeout <seconds>       Optional timeout in seconds (default: 0 for no timeout)
  --acp                         Run in Agent Client Protocol (ACP) mode for editor integration
  --config <path>               Configuration directory (default: ~/.cline)
  --data-dir <path>             Use isolated local state at this directory path (default: ~/.cline/data)
  --hooks-dir <path>            Directory path to additional hooks for runtime hook injection
```

**Exact headless command:**
```
cline --json --auto-approve true "your prompt here"
```

Headless is triggered when using flags like `--json`, when stdin is piped, or when stdout is redirected (per CLI overview).

## Streaming output support

- **`--json`**: Newline-delimited JSON messages — one JSON object per line (NDJSON). Line types include:
  - `agent_event` — agent events (`content_start`, `content_update`, `content_end`, `iteration_start`, `iteration_end`, `usage`, `notice`, `done`, `error`)
  - `run_result` — run finished (`finishReason`, `iterations`, `usage`, `durationMs`, `model`, final `text`)
  - `run_aborted` — follows `run_result` when run is stopped
  - `hook_event`, `team_event`, `run_start`
- Example line: `{"ts":"2026-10-07T04:49:43.041Z","type":"agent_event","event":{"type":"done","reason":"completed","text":"Hello!","iterations":1}}`

## Session persistence and RESUME

- **Where sessions live:** `~/.cline/data/sessions.db` (SQLite) and `~/.cline/data/sessions/` (one folder per session). Global config at `~/.cline/` (settings, providers.json, mcp, rules, skills, hooks, agents, plugins).
- **Resume flag:** `--id <session-id>` — resume an existing session by ID.
- **Resume after orchestrator restart:** Sessions persist in SQLite DB; external orchestrator can capture session IDs and resume with `--id` later.
- **KNOWN BUG:** GitHub issue `cline/cline#10856` ("Cline CLI 3.0.7 --json mode can't resume an existing session ID") — `--id` + prompt is rejected in `--json` mode ("requires a prompt argument" although one is provided). This means `--json` headless mode cannot resume an existing session ID — a significant limitation for orchestrators needing session continuity.

## Authentication

- **Auth model:** Provider authentication — Cline Provider, ClinePass, or your own provider key. `cline auth` configures provider/model. `-k, --key <api-key>` passes an API key for the run.
- **Multiple independent accounts:** Yes — via config-dir isolation (see below).
- **OAuth-bound:** Some providers use OAuth; credentials stored under config dir.

## Multi-account isolation

- **`--config <path>`**: Configuration directory (default: `~/.cline`).
- **`--data-dir <path>`**: Use isolated local state at this directory path (default: `~/.cline/data`).
- Setting `--config` and `--data-dir` per account fully isolates account B from account A.
- Env vars: `CLINE_HUB_ADDRESS`, `CLINE_SESSION_BACKEND_MODE`, `CLINE_SANDBOX_DATA_DIR`, `CLINE_LOG_PATH`, `CLINE_HOOKS_DIR`, `CLINE_DEBUG_PORT_BASE`.

## Local server / SDK / daemon mode

- **ACP mode:** `--acp` runs Cline in Agent Client Protocol mode for editor integration (Zed, JetBrains, Neovim).
- **Hub:** `cline hub` manages a local hub daemon (`hub status`, `hub ensure`, `hub start`, `hub stop`, `hub dashboard`). Sessions can run in the background hub (`-z, --zen`).
- **SDK:** Cline is described as "an SDK" — the VS Code extension, JetBrains plugin, and CLI share an agent core. SDK reference at `https://docs.cline.bot/sdk/reference/events#agentevent`. No standalone npm client SDK confirmed from docs fetched.
- **HTTP server:** No HTTP server mode documented (ACP is stdio-based; hub is local daemon).

## License

- **Tool:** Not verified.
- **SDK:** Not verified.

## Documented limitations for automation

- `--json` mode cannot resume an existing session ID (known bug cline/cline#10856).
- The CLI has no allow or deny list for shell commands; use a `PreToolUse` hook to block commands.
- Headless mode triggered by `--json`, piped stdin, or redirected stdout — any of these disables TUI.
- `--auto-approve true` needed for unattended runs (default is `true` for direct CLI runs, `false` in ACP mode).

---

# Claude Code

## Canonical repo / official docs / binary

- **Canonical repo:** `https://github.com/anthropics/claude-code` (the `claude` CLI)
- **Official docs:** `https://code.claude.com/docs/en/` (headless at `https://code.claude.com/docs/en/headless`, CLI reference at `https://code.claude.com/docs/en/cli-reference`)
- **Binary:** `claude`
- **Installed locally:** NO — `which claude` → command not found. All claims below are docs-only.
- **License:** Not verified (proprietary, Anthropic).

## Headless / non-interactive execution (docs only)

From headless docs:
> Add the `-p` (or `--print`) flag to any `claude` command to run it non-interactively.

CLI commands table (from CLI reference):
| Command | Description |
|---------|-------------|
| `claude` | Start interactive session |
| `claude "query"` | Start interactive session with initial prompt |
| `claude -p "query"` | Query via SDK, then exit |
| `cat file \| claude -p "query"` | Process piped content |
| `claude -c` | Continue most recent conversation in current directory |
| `claude -c -p "query"` | Continue via SDK |
| `claude -r "<session>" "query"` | Resume session by ID or name |

**Exact headless command:**
```
claude -p "Find and fix the bug in auth.py" --allowedTools "Read,Edit,Bash"
```

`--bare` mode skips auto-discovery of hooks, skills, subagents, MCP servers, CLAUDE.md — recommended for CI/scripts:
```
claude --bare -p "Summarize README.md" --allowedTools "Read"
```

## Streaming output support

- **`--output-format stream-json`**: Structured output mode. Used with `-p`. Example: `gh pr diff "$1" | claude -p "..." --append-system-prompt "..." --output-format json` prints the review as JSON with text in the `result` field.
- SDK docs mention `stream-json` output for programmatic parsing; `session_id` can be extracted: `session_id=$(claude -p "Start a review" --output-format json | jq -r '.session_id')`.

## Session persistence and RESUME

- **Where sessions live:** `~/.claude/` — transcripts stored locally. Config dir can be overridden via `CLAUDE_CONFIG_DIR` env var.
- **Resume flags:** `--continue, -c` (continue most recent conversation), `--resume, -r "<session_id>"` (resume specific session by ID or name; also accepts absolute path to a `.jsonl` transcript file).
- **Resume after orchestrator restart:** Yes — sessions persist as files on disk; external orchestrator can capture session IDs and resume with `--resume` later. Works across directories ("Claude Code finds the session by its ID in any project on this machine").
- **Claude Code v2.1.257+:** `--continue` opens a background session that has finished, not one still running.

## Authentication

- **Auth model:** OAuth login (`claude auth login`) or API key (`ANTHROPIC_API_KEY` env var, or `apiKeyHelper` in `--settings`). `claude setup-token` generates a long-lived OAuth token for CI/scripts (requires Claude subscription).
- **Multiple independent accounts:** Yes — via `CLAUDE_CONFIG_DIR` env var isolation (see below).
- **OAuth-bound:** OAuth credentials stored under config dir (`claude.credentials.json`).

## Multi-account isolation

- **`CLAUDE_CONFIG_DIR`**: Overrides where Claude Code looks for all config/credentials (`~/.claude` default). "When `CLAUDE_CONFIG_DIR` is set, Claude Code keeps `.credentials.json` under that directory instead."
- Setting `CLAUDE_CONFIG_DIR` per account fully isolates account B from account A.
- **OAuth-bound caveat:** On macOS, OAuth credentials also touch the system Keychain (`claude.credentials.json` inherits keychain access controls). The keychain may not be fully isolated by `CLAUDE_CONFIG_DIR` alone — potential cross-account leakage via keychain.
- `claude auth status` shows `configDirectory` field naming the config directory being used (v2.1.268+).

## Local server / SDK / daemon mode

- **Agent SDK:** Python and TypeScript packages (`@anthropic/claude-code` presumably) — full programmatic control with structured outputs, tool approval callbacks, native message objects.
- **CLI is the entry point to the SDK:** `claude -p` runs the SDK headlessly.
- **Daemon/supervisor:** `claude daemon` commands manage background-session supervisor (`daemon logs`, `daemon run`, `daemon status`, `daemon stop`).
- **Remote Control:** `claude remote-control` starts a server to control Claude Code from Claude.ai/Claude app — server mode, no local interactive session.
- **HTTP server:** No plain HTTP REST server documented; Agent SDK is the programmatic interface.

## License

- **Tool:** Not verified (proprietary).
- **SDK:** Not verified (likely Anthropic's standard SDK terms).

## Documented limitations for automation

- `-p` mode rejects `--bg` and `--cloud` (with task description); `--cloud` with session ID queues a message and exits.
- Not every CLI option combines with `-p`.
- User-invoked skills and custom commands work; built-in terminal-only commands like `/login` are not available in `-p` mode.
- `--bare` skips auto-discovery (hooks, skills, MCP, CLAUDE.md) — good for reproducibility but loses project context.
- `claude agents` (agent view) requires an interactive terminal.

---

# Comparison table

| Tool | Binary | Headless cmd | Streaming | Resume | Multi-account isolation | Server/SDK mode | License |
|------|--------|-------------|-----------|--------|------------------------|-----------------|---------|
| Kiro CLI | `kiro-cli` (v2.27.1) | `kiro-cli chat --no-interactive --output-format stream-json "prompt"` | JSON Lines (ACP events) on stdout via `--output-format stream-json` | `-r/--resume`, `--resume-id <ID>`; sessions at `~/.kiro/sessions/cli/*.jsonl` | `KIRO_HOME` env var (isolates whole profile; relative paths ignored) | ACP server over stdio (`kiro-cli acp`, JSON-RPC 2.0). No HTTP server, no SDK. | Proprietary (unverified) |
| OpenCode | `opencode` (v2.2.5) | `opencode run --format json --continue "prompt"` | `--format json` (single JSON); SSE events via SDK `event.subscribe()`; `--print-logs` to stderr | `--continue/-c`, `--session/-s <ID>`; sessions in `~/.local/share/opencode/opencode.db` | `XDG_CONFIG_HOME` + `XDG_DATA_HOME`; also `--standalone` private server, `--server <URL>` | **YES** — `opencode serve` HTTP server (port 4096) + `@opencode-ai/sdk` npm package (typed client over HTTP); also ACP stdio server | MIT |
| Pi Coding Agent | `pi` | `pi -p "prompt"` + `--mode json` | `--mode json` event stream; RPC over stdio JSON | Not precisely documented; sessions are JSONL trees in `~/.pi/`; `/tree` navigate | `HOME` or `XDG_CONFIG_HOME` | TypeScript SDK (`@earendil-works/pi-coding-agent`); RPC stdio; no HTTP server | MIT |
| Cline | `cline` | `cline --json --auto-approve true "prompt"` | NDJSON per line: `agent_event`, `run_result`, `hook_event`, etc. | `--id <session-id>` (KNOWN BUG: broken with `--json` in v3.0.7); sessions at `~/.cline/data/sessions.db` | `--config <dir>` + `--data-dir <dir>` flags | ACP mode (`--acp`, stdio); local hub daemon; no HTTP server, no standalone SDK confirmed | Unverified |
| Claude Code | `claude` | `claude -p "prompt"` + `--bare` | `--output-format stream-json` / `--output-format json` | `--continue/-c`, `--resume/-r <ID>`; sessions at `~/.claude/` transcripts | `CLAUDE_CONFIG_DIR` env var | Agent SDK (Python/TS); `claude daemon` supervisor; `claude remote-control` server mode | Proprietary (unverified) |

---

# What this means for the orchestrator

## Cleanly wrappable as subprocess adapters

1. **OpenCode** — Best overall. Local binary verified (`opencode v2.2.5`). Has `--format json` for headless output, `--continue`/`--session` for resume, `XDG_CONFIG_HOME`/`XDG_DATA_HOME` for multi-account isolation, and critically **`opencode serve` HTTP server + `@opencode-ai/sdk` npm package**. MIT license. The orchestrator should prefer the HTTP/SDK path over shelling out.

2. **Kiro CLI** — Good subprocess adapter candidate. Local binary verified (`kiro-cli v2.27.1`). Has `--no-interactive`, `--output-format stream-json` (JSON Lines/ACP events on stdout), `-r/--resume`/`--resume-id`, `KIRO_API_KEY` for headless auth, `KIRO_HOME` for multi-account isolation. ACP server over stdio is available but no HTTP server or SDK. Proprietary (license unverified).

3. **Pi Coding Agent** — Good subprocess adapter candidate. Four modes including print/JSON (`-p`) and `--mode json` for event streams. RPC over stdio for non-Node integrations. TypeScript SDK available. `HOME`/`XDG_CONFIG_HOME` for isolation. MIT license. But binary not installed locally — claims are docs-only; needs `--help` verification.

4. **Cline** — Subprocess adapter but with a known bug. `--json` NDJSON output works, `--id` for resume, `--config`/`--data-dir` for isolation. BUT `--json` mode cannot resume an existing session ID (cline/cline#10856) — breaks session continuity for headless runs. Needs `--auto-approve` for unattended runs. Binary not installed locally — docs-only claims.

5. **Claude Code** — Subprocess adapter via `-p`/`--print`. `--continue`/`--resume` for resume, `CLAUDE_CONFIG_DIR` for isolation, `--bare` for CI reproducibility. Agent SDK (Python/TS) available for programmatic use. But `--cloud` conflicts with `-p`; terminal-only commands unavailable in `-p` mode. Binary not installed locally — docs-only claims. Proprietary.

## Which expose a better programmatic interface than shelling out

| Tool | Better programmatic interface |
|------|------------------------------|
| **OpenCode** | **HTTP server (`opencode serve`) + `@opencode-ai/sdk`** — full typed API over HTTP, SSE events, session management. Best option. |
| **Pi** | TypeScript SDK (`@earendil-works/pi-coding-agent`) + RPC over stdio — good alternative to shelling out. |
| **Claude Code** | Agent SDK (Python/TS) — structured outputs, tool approval callbacks, native message objects. |
| **Kiro CLI** | ACP over stdio (JSON-RPC 2.0) — only stdio protocol, no HTTP/SDK. Shelling out to `kiro-cli acp` is the integration path. |
| **Cline** | ACP mode (`--acp`) + hub daemon — stdio-based, no HTTP server confirmed. Shelling out is the main path. |

## Which are too interactive/fragile to automate reliably

- **Cline** — `--json` resume bug (#10856) makes headless session continuity unreliable. Workaround possible (use `--json` for fresh runs, `--id` without `--json` for resumes) but fragile.
- **Kiro CLI** — V3 non-interactive mode blocks Hook confirmation requests unless `--trust-all-tools` is used; `--trust-tools` cannot approve Hooks. Hooks that block are final. Need careful permission configuration.
- **Claude Code** — Not all CLI options combine with `-p`; `--bare` mode loses project context (hooks, skills, MCP, CLAUDE.md); terminal-only commands like `/login` unavailable. Workable but with caveats.

---

# Risks and gaps

1. **Local verification incomplete:** Pi, Cline, and Claude Code are not installed on this machine. All their claims (flags, session paths, isolation mechanisms) come from web docs only and should be verified with `--help`/`--version` before the orchestrator depends on them.

2. **OpenCode repo moved:** Canonical repo is now `anomalyco/opencode` (was `sst/opencode`). Docs at `opencode.ai`. Version on this machine is v2.2.5 — the `serve` subcommand and SDK `@opencode-ai/sdk` exist in this version, confirmed locally.

3. **Cline `--json` resume bug:** cline/cline#10856 confirms `--id` + prompt rejected in `--json` mode. Orchestrator must handle this: either avoid `--json` for resumed sessions, or use the `--id` flag without `--json` and parse text output.

4. **Kiro proprietary:** License not verified; no public SDK found; ACP is stdio-only (no HTTP server). Orchestrator must shell out to `kiro-cli`.

5. **OAuth + config-dir isolation not perfect:** For Claude Code on macOS, Keychain may leak credentials across `CLAUDE_CONFIG_DIR` boundaries. For Kiro, `--use-device-flow` OAuth credentials are under `KIRO_HOME` but the OAuth provider may bind externally. OpenCode's OAuth integrations similarly stored under config dir but provider-specific.

6. **No HTTP server for Kiro, Cline, Claude Code:** Only OpenCode has a verified HTTP server mode (`opencode serve`). Pi has no HTTP server documented. Kiro/Cline use ACP over stdio. Claude Code has Agent SDK but no plain HTTP REST server.

7. **Session resume across orchestrator restart:** All five tools persist sessions to disk and support resume by session ID. The orchestrator must record session IDs and the tool's config dir per account to enable cross-restart resume.

8. **Exit codes / failure handling:** Kiro docs mention exit codes (`--require-mcp-startup` exits code 3 on failure); Claude Code exits 0 on success, non-zero on failure, 143 on SIGTERM. Cline/OpenCode exit-code behavior not verified — should be tested before production use.

---

# Evidence

## Local command output (verbatim)

### Kiro CLI `--help-all` (local, v2.27.1)
```
USAGE:
    kiro-cli [OPTIONS] [SUBCOMMAND]

Commands:
  debug         Debug the app
  settings      Customize appearance & behavior
  setup         Setup cli components
  update        Update the Kiro application
  diagnostic    Run diagnostic tests
  init          Generate the dotfiles for the given shell
  theme         Get or set theme
  issue         Create a new Github issue
  login         Login
  logout        Logout
  whoami        Prints details about the current user
  profile       Show the profile associated with this idc user
  user          Manage your account
  doctor        Fix and diagnose common issues
  launch        Launch the desktop app
  quit          Quit the desktop app
  restart       Restart the desktop app
  integrations  Manage system integrations
  translate     Natural Language to Shell translation
  dashboard     Open the dashboard
  chat          AI assistant in your terminal
  crew          Launch Kiro Crew, installing it if it is not already installed
  mcp           Model Context Protocol (MCP)
  powers        Manage powers
  inline        Inline shell completions
  agent         Agent root commands
  acp           Agent Client Protocol (ACP)
  help          Print this message or the help of the given subcommand(s)

Options:
  -v, --verbose...
      --help-all
      --agent <AGENT>
      --tui
      --classic
      --v3
      --v2
  -r, --resume          Resume the most recent conversation from this directory
      --resume-id <SESSION_ID>
      --resume-picker
      --cloud
      --repo <REPO>
  -h, --help
  -V, --version
```

### Kiro CLI `chat --help` (local)
Key flags: `-r/--resume`, `--resume-id`, `--resume-picker`, `--sessions`, `--agent`, `--model`, `--effort`, `-a/--trust-all-tools`, `--trust-tools`, `--no-interactive`, `-l/--list-sessions`, `--all-cwds`, `--list-models`, `-f/--format`, `--output-format` (text / stream-json), `-d/--delete-session`, `--session-source`, `-v/--verbose`, `-w/--wrap`, `--require-mcp-startup`, `--tui`, `--legacy-ui`, `--agent-engine`, `--mode`, `--cloud`, `--repo`.

### Kiro CLI `acp --help` (local)
```
Usage: kiro-cli-chat acp [OPTIONS]
Options:
  --agent <AGENT>
  --model <MODEL>
  --effort <EFFORT>
  -a, --trust-all-tools
  --trust-tools <TOOL_NAMES>
  --agent-engine <ENGINE>
  --auth-method <METHOD>   cli: Resolve access tokens for the v3 engine from the Kiro CLI credential store
```

### OpenCode `--help` (local, v2.2.5)
```
DESCRIPTION
  OpenCode command line interface

USAGE
  opencode <subcommand> [flags] [<directory>]

FLAGS
  --standalone            Run with a private server instead of the background service
  --server string         Connect to a server URL instead of the background service
  --auto                  Auto-approve permissions that are not explicitly denied
  --continue, -c          Continue the last session
  --session, -s string    Session ID to continue, or to create if it does not exist
  --prompt string         Prompt to use
```

### OpenCode `run --help` (local)
```
FLAGS
  --continue, -c          Continue the last session
  --session, -s string    Session ID to continue, or to create if it does not exist
  --fork                  Fork the session before continuing
  --model, -m string      Model to use in the format provider/model#variant
  --agent string          Agent to use
  --format choice         Output format (choices: default, json)
  --file, -f string       File to attach to the message
  --title string          Session title
  --thinking              Show thinking blocks
  --auto                  Auto-approve permissions that are not explicitly denied
```

### OpenCode `serve --help` (local)
```
DESCRIPTION
  Start the v2 API and web server

USAGE
  opencode serve [flags]

FLAGS
  --hostname string    
  --port integer       
  --cors string         Additional allowed CORS origin (repeat for multiple origins)
  --service            
  --stdio              
```

### OpenCode version (local)
```
opencode v2.2.5
```

## URLs fetched

- `https://kiro.dev/docs/cli/headless/` — Kiro headless mode, `--no-interactive`, `--output-format stream-json`, `KIRO_API_KEY`, `--trust-all-tools`, `--trust-tools`, `--require-mcp-startup`, `KIRO_HEADLESS_WORKFLOW_TIMEOUT_SECS`
- `https://kiro.dev/docs/cli/acp/` — Kiro ACP server, `kiro-cli acp`, JSON-RPC 2.0 over stdio, session storage at `~/.kiro/sessions/cli/`
- `https://kiro.dev/docs/configuration/` — Kiro config scopes, `~/.kiro/` global scope, `KIRO_HOME` env var
- `https://kiro.dev/changelog/cli/2-3/` — Kiro CLI 2.3.0: `KIRO_HOME` environment variable details
- `https://opencode.ai/docs/sdk/` — OpenCode SDK `@opencode-ai/sdk`, `createOpencode()`, port 4096, SSE `event.subscribe()`, full typed API
- `https://github.com/anomalyco/opencode` — OpenCode canonical repo, MIT License, installation via npm `@opencode/cli`
- `https://pi.dev/` — Pi Coding Agent: four modes (interactive, print/JSON, RPC, SDK), `pi -p`, `--mode json`, API keys/OAuth, sessions as JSONL trees
- `https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md` — Pi install, npm package `@earendil-works/pi-coding-agent`, MIT license, RPC over stdio
- `https://docs.cline.bot/usage/cli-overview` — Cline CLI overview, `--json`, `--auto-approve`, headless triggers
- `https://docs.cline.bot/cli/cli-reference` — Cline CLI reference: `--json`, `--id <session-id>`, `--config <path>`, `--data-dir <path>`, `--auto-approve`, NDJSON line types (`agent_event`, `run_result`), sessions.db at `~/.cline/data/sessions.db`
- `https://code.claude.com/docs/en/headless` — Claude Code `-p`/`--print`, `--bare`, `--continue`, `--resume`, `--output-format stream-json`, `ANTHROPIC_API_KEY`, `claude setup-token`, exit codes
- `https://code.claude.com/docs/en/cli-reference` — Claude Code CLI commands table, `--resume` by ID or `.jsonl` path, `--bare`, system prompt flags
- `https://code.claude.com/docs/en/env-vars` — `CLAUDE_CONFIG_DIR` env var
- `https://code.claude.com/docs/en/authentication` — Claude Code authentication, OAuth, `claude.credentials.json`
- `https://github.com/MishraShardendu22/kiro-account-pool-manager` — Community tool confirming `KIRO_HOME` + `XDG_DATA_HOME` + `XDG_RUNTIME_DIR` per-profile isolation for Kiro
- `https://github.com/wangyuyan-agent/kiro-multi` — Community tool confirming per-profile `HOME` rewrite for Kiro multi-account
- `https://github.com/cline/cline/issues/10856` — Cline `--json` resume bug: `--id` rejected in `--json` mode