# herdr ecosystem

## Repo facts

| repo | stars | lang | license | last push | archived | what it is |
|------|-------|------|---------|-----------|----------|------------|
| herdrdev/herdr | unknown (API rate limited) | Rust | Apache-2.0 (from LICENSE file) | unknown | false | Terminal workspace manager for AI coding agents (GUI app + CLI). Contains agent detection, TUI, integrations. |
| herdrdev/herdr-core | unknown (API rate limited) | Rust | Apache-2.0 (from LICENSE file) | unknown | false | Core daemon library + binary (`herdr-core`). JSON-RPC server, terminal/PTY management, session persistence. |
| herdrdev/herdr-app | unknown (API rate limited) | unknown | unknown | unknown | unknown | Likely the macOS app bundle / packaging (referenced in herdr-core CHANGELOG as `herdr-app#...`). |
| herdrdev/herdr-layout-examples | unknown (API rate limited) | unknown | unknown | unknown | unknown | Layout examples for herdr TUI. |
| herdrdev/herdr-nix | unknown (API rate limited) | Nix | unknown | unknown | unknown | Nix packaging for herdr. |

**Note:** GitHub API rate limited (unauthenticated ~60 req/hr). License confirmed from local LICENSE files: both `herdr-core` and `herdr` use Apache-2.0. Language confirmed from Cargo.toml: Rust.

## Directory structure

### herdr-core (`/home/deval/.hermes/cache/scratch/hd_core/`)

```
hd_core/
├── Cargo.toml                    # Workspace root (members = ["crates/*"])
├── Cargo.lock
├── LICENSE                       # Apache-2.0
├── README.md                     # Project overview
├── CHANGELOG.md                  # Release notes
├── schema/
│   ├── README.md                 # OpenRPC documentation (13,638 chars)
│   ├── hc.openrpc.json           # Full OpenRPC document (73,226 chars)
│   └── released.openrpc.json     # Copy of hc.openrpc.json
├── crates/
│   ├── ghostty-vt/               # Terminal emulator VT parser (vendored)
│   └── herdr-core/               # Main crate
│       ├── Cargo.toml            # Library + binary (name = "herdr-core")
│       ├── src/
│       │   ├── main.rs           # Binary entry point
│       │   ├── lib.rs            # Library entry
│       │   ├── api.rs            # Method table (32 methods)
│       │   ├── api/
│       │   │   ├── types.rs      # Request/response types (800+ lines)
│       │   │   ├── run.rs        # terminal.run implementation
│       │   │   ├── waiting.rs    # terminal.wait implementation
│       │   │   └── doc.rs        # doc.get/doc.set
│       │   ├── server.rs         # JSON-RPC server (Unix socket)
│       │   ├── client.rs         # JSON-RPC client
│       │   ├── wire.rs           # Framed protocol implementation
│       │   ├── runtime.rs        # Core state, registry, persistence
│       │   ├── runtime/
│       │   │   └── persistence.rs # Atomic registry writer
│       │   ├── holder.rs         # PTY owner, process spawner
│       │   ├── holder_proto.rs   # Holder-core protocol
│       │   ├── terminals.rs      # Terminal registry, Public view
│       │   ├── templates.rs      # Template loading (markdown files)
│       │   ├── notes.rs          # Parent-child notes
│       │   ├── paths.rs          # Socket/state/config paths
│       │   ├── config.rs         # Config parsing
│       │   ├── exec.rs           # machine.exec implementation
│       │   ├── input.rs          # Input handling
│       │   ├── remote.rs         # Remote machine support
│       │   ├── detect/           # (empty in herdr-core)
│       │   └── platform/         # OS abstractions
│       └── tests/
├── docs/
│   ├── commands.md               # CLI reference (generated)
│   ├── templates.md              # Agent templates documentation
│   ├── machines.md               # Remote machines
│   └── security.md
└── skills/herdr-core/            # Skill for AI agents
```

### herdr (`/home/deval/.hermes/cache/scratch/hd_main/`)

```
hd_main/
├── Cargo.toml                    # Binary crate "herdr" (GUI app + CLI)
├── Cargo.lock
├── LICENSE                       # Apache-2.0
├── README.md                     # Project overview
├── CHANGELOG.md                  # Release notes
├── src/
│   ├── main.rs                   # Binary entry
│   ├── cli.rs                    # CLI commands
│   ├── api/                      # herdr-specific API
│   ├── app/                      # TUI application
│   ├── agent_resume.rs           # Agent session resume
│   ├── agent_view_eval.rs        # Agent state evaluation
│   ├── detect/                   # Agent detection engine
│   │   ├── mod.rs
│   │   ├── manifest.rs
│   │   └── manifests/            # 20+ agent TOML files (claude, codex, opencode, kiro, pi, etc.)
│   ├── integration/              # Agent integrations (Claude, OpenCode, etc.)
│   │   ├── assets/               # Per-agent scripts (e.g., herdr-agent-state.sh)
│   │   ├── config_file.rs        # Config file manipulation
│   │   └── opencode_config.rs
│   ├── server/                   # herdr's own server (distinct from herdr-core)
│   ├── session.rs                # Session management
│   ├── workspace.rs              # Workspace/tab/pane model
│   ├── pane.rs                   # Pane (terminal) management
│   ├── platform/                 # Platform-specific code
│   └── ...                       # UI, rendering, input, etc.
├── distribution/
│   ├── agent-detection/          # 20 agent TOML manifests
│   └── agent-guide.md            # Documentation for AI agents
├── skills/herdr/                 # Skill for AI agents (herdr CLI)
└── docs/next/                    # Website docs (agents.mdx, agent-automation.mdx, etc.)
```

## Public API / programmatic interface

herdr-core exposes a **JSON-RPC 2.0** API over a **Unix socket** (or Windows named pipe). The complete API is described in `schema/hc.openrpc.json` (OpenRPC 1.2.6 document). The `rpc.discover` method returns this document at runtime.

### Transport (server.rs:55-66, client.rs:72-102, paths.rs:17-49)

- **Type**: Unix domain socket (Linux/macOS) / Named pipe (Windows)
- **Default path**: `$XDG_RUNTIME_DIR/herdr-core/<session>.sock` (or `$XDG_STATE_HOME/herdr-core/<session>/<session>.sock` per paths.rs:22-35)
- **Configurable via**: `HC_SOCKET` environment variable or `socket` in `~/.config/herdr-core/config.toml`
- **Holder socket**: `<core_socket>.holder.sock` (holder.rs:94)
- **Protocol**: Plain JSON-RPC (one message per line) by default. After `hello` with `framed: true`, switches to length-prefixed framed mode: `[channel u32 LE][kind u8][len u32 LE][payload]` (wire.rs:1-2, api.rs:16-17 features list includes "framed")
- **No TCP/HTTP/WebSocket/stdin-stdout** (except `hc connect` which speaks the protocol over stdin/stdout for SSH tunneling)

### All 32 methods (from hc.openrpc.json and api.rs)

#### `hello` (handshake)
- **Params**: `generation?`, `features?`, `framed?`, `core?`, `calls_you?`
- **Result**: `HelloResult` — core identity, version, pid, methods, features, supported generations
- **Errors**: `unsupported_generation`

#### `server.*` (core lifecycle)
| method | params | result | description |
|--------|--------|--------|-------------|
| `server.status` | (none) | `StatusResult` | Core identity, version, pid, uptime, holder count, terminal count |
| `server.stop` | (none) | `StopResult` | Stop core and programs; kept terminals get new shells on next start |
| `server.restart` | (none) | `StopResult` | Planned restart: screens saved, programs keep running, next core adopts them |

#### `terminal.*` (terminal management)
| method | key params | result | description |
|--------|------------|--------|-------------|
| `terminal.run` | `argv?`, `line?`, `name?`, `cwd?`, `env?`, `keep?`, `labels?`, `parent?`, `on?`, `wait?`, `timeout?`, `rm?`, `die_with_parent?`, `outlive_parent?`, `no_parent?`, `no_notify?`, `size?`, `display?`, `caller?` | `RunResult` { terminal, first_message?, wait?, output? } | Start program or template; if template, sends body+task as first message when agent ready |
| `terminal.list` | `all?`, `labels?` | `ListResult` | Live terminals (all with `all`), filter by labels |
| `terminal.get` | `terminal` (req), `caller?` | `Terminal` | One terminal's facts, live or finished |
| `terminal.kill` | `terminal` (req), `caller?` | `Terminal` | End program: HUP → TERM → KILL |
| `terminal.rm` | `terminal` (req), `caller?` | `RemoveResult` | Remove finished terminal |
| `terminal.keep` | `terminal` (req), `keep` (req) | `Terminal` | Set restart-as-shell after loss (incl. signal death) |
| `terminal.label` | `terminal` (req), `labels` (req) | `Terminal` | Set/remove free-form key=value labels |
| `terminal.read` | `terminal` (req), `lines?`, `caller?` | `ReadResult` | Screen + history as text |
| `terminal.attach` | `terminal` (req), `cols` (req), `rows` (req), `history?`, `observe?`, `resize_notifications?`, `resumed?`, `display?`, `caller?` | `AttachResult` | Raw stream on framed connection; loss ends stream |
| `terminal.send` | `terminal` (req), `text` (req), `force?`, `if?`, `timeout?`, `wait?`, `caller?` | `SendResult` | Paste text + Enter; waits for draft clear or ~3s quiet; refuses blocked unless `force` |
| `terminal.wait` | `terminal?`, `output?`, `timeout?`, `children?`, `caller?` | `WaitResult` | Wait until not working (or for output match). `children` mode waits for child terminals |
| `terminal.keys` | `terminal` (req), `keys` (req array), `caller?` | `KeysResult` | Press named keys (enter, esc, up, ctrl+c, etc.) |
| `terminal.interrupt` | `terminal` (req), `timeout?` | `InterruptResult` | Press stop key (esc if reports status, else ctrl+c); never escalates |
| `terminal.put` | `terminal` (req), `name` (req), `data` (req base64) | `PutResult` | Write file beside terminal on its machine; lands in per-terminal folder |

#### `templates.*`
| method | params | result | description |
|--------|--------|--------|-------------|
| `templates.list` | (none) | `TemplatesResult` | User templates from `~/.config/herdr-core/templates/*.md` |

#### `machine.*` (remote machines)
| method | key params | result | description |
|--------|------------|--------|-------------|
| `machine.exec` | `argv?`, `line?`, `on?`, `cwd?`, `env?`, `input?`, `timeout?` | `ExecResult` | Run program without terminal; returns exit code + base64 output |
| `machine.add` | `target?`, `name?`, `session?`, `via?`, `temporary?` | `Machine` | Add machine via SSH; checks hc answers there |
| `machine.list` | (none) | `MachineListResult` | Machines this core has added |
| `machine.connect` | `machine` (req) | `MachineView` | Dial machine now (retry) |
| `machine.rename` | `machine` (req), `new_name` (req) | `Machine` | Rename machine locally |
| `machine.rm` | `machine` (req) | `RemoveResult` | Forget machine; terminals keep running there |

#### `file.*` (remote file ops)
| method | key params | result |
|--------|------------|--------|
| `file.read` | `path` (req), `on?`, `cwd?`, `offset?`, `limit?` | `FileReadResult` |
| `file.write` | `path` (req), `data` (req), `on?`, `cwd?` | `FileResult` |
| `file.edit` | `path` (req), `edits[]` (req), `on?`, `cwd?` | `FileResult` |

#### `doc.*` (opaque JSON documents)
| method | key params | result |
|--------|------------|--------|
| `doc.get` | `name` (req), `on?` | `DocGetResult` { value, version } |
| `doc.set` | `name` (req), `value` (req), `version` (req), `on?` | `DocSetResult` |

#### `events.*`
| method | key params | result |
|--------|------------|--------|
| `events.subscribe` | `terminal?`, `labels?`, `children_of?` | `SubscribeResult` (snapshot + event notifications) |

#### `rpc.*`
| method | params | result |
|--------|--------|--------|
| `rpc.discover` | (none) | OpenRPC document (same as hc.openrpc.json) |

### Critical method shapes (for orchestrator integration)

**terminal.run** — Primary way to start coding agents:
```json
{
  "argv": ["claude"],           // or template name "reviewer"
  "line": "cd repo && claude", // shell line alternative
  "name": "my-agent",           // optional fixed name
  "cwd": "/path/to/repo",
  "env": { "KEY": "value" },
  "keep": true,                 // restart as shell on loss
  "labels": { "role": "coder" },
  "parent": "t3",               // parent terminal ID (for notes)
  "on": "vps",                  // run on remote machine
  "wait": true,                 // wait until done (like shell)
  "timeout": 180.0,             // wait timeout seconds
  "rm": false,                  // auto-remove when finished
  "caller": "orchestrator-1"    // caller identity for notes/wait
}
```
Returns `RunResult` with `terminal` (Public), `first_message` (sent status), `wait` (if waited), `output` (tailed screen).

**terminal.send** — Send input to agent:
```json
{
  "terminal": "t7",
  "text": "Fix the bug in auth.rs",
  "force": false,        // if true, send even if agent reports blocked
  "if": "idle|working",  // only send if agent in this state
  "wait": false,         // if true, wait for turn to complete
  "timeout": 60.0,
  "caller": "orchestrator-1"
}
```

**terminal.wait** — Wait for agent completion:
```json
{
  "terminal": "t7",
  "output": "test passed",  // wait until this text appears
  "timeout": 300.0,
  "children": "recursive",  // wait for child terminals too
  "caller": "orchestrator-1"
}
```
Returns `WaitResult` with `ended`, `reason` ("exit", "status", "output", "turn"), `exit_code?`, `output?`.

**events.subscribe** — Real-time updates:
```json
{
  "terminal": "t7",
  "labels": { "role": "coder" },
  "children_of": "t3"
}
```
Returns snapshot then streams `event` notifications (created, lost, restarted, keep, finished, removed, status, turn, title, labels, size, doc) and `heartbeat` every 30s.

## Process lifecycle and session persistence

### Two-process architecture (holder.rs:1-5, 68-86, server.rs:55-66)

1. **Core** (`herdr-core` binary): JSON-RPC server, owns the Unix socket, manages terminal registry, handles client requests, persists state.
2. **Holder** (spawned by core, runs on `<socket>.holder.sock`): Owns every PTY, is every program's direct parent, always reads output (keeps ~1 MiB tail per terminal), restarts crashed core.

### Terminal survives client disconnect (holder.rs:20-33, runtime.rs, terminals.rs)

- The **holder** is the true process parent. When a client disconnects, the core stays running; the holder keeps the PTY alive and continues pumping output to its tail buffer.
- `terminal.attach` opens a raw stream on a **framed connection**; loss of that connection ends the stream but the terminal keeps running (holder.rs:35-36, api.rs:95-97).
- `terminal.keep: true` enables restart-as-shell after program loss (including death by signal without exit code). Recovery runs on every OS after a grace delay (default 3s, configurable via `HC_HOLDER_GRACE_MS`), except during core stop/restart or Windows session shutdown (holder.rs:95-98, api.rs:87-88).

### State persistence (runtime/persistence.rs, paths.rs:59-75)

- **Registry file**: `$XDG_STATE_HOME/herdr-core/<session>/registry.json` (atomic writes via `write_atomic` → write to `.tmp` → sync → rename)
- **Writer**: `RegistryWriter` runs on a dedicated thread; core calls `save(bytes)` under lock, writer flushes asynchronously; `flush()` awaits current snapshot (persistence.rs:9-70).
- **Screen tails**: Holder keeps ~1 MiB tail per terminal in memory (holder.rs:15). On core restart, `server.restart` saves screens, programs keep running, next core adopts them (api.rs:66-68).
- **Terminals.json**: Not a separate file; terminal state is part of the registry snapshot.

### Recovery after orchestrator restart

- If **core crashes**: Holder detects lost core connection, waits for new core to connect on holder socket, sends current terminal state (holder_proto.rs). Core rebuilds registry from `registry.json` + holder state.
- If **holder crashes**: Core detects (via heartbeat/frame loss), spawns new holder, but PTYs are lost (programs die). No automatic recovery of holder crash documented.
- If **orchestrator (client) restarts**: Connects to existing core via socket (client.rs:75-102 autostarts core if needed). Terminals continue running. `events.subscribe` replays snapshot.

### Session isolation (paths.rs:80-93)

- `HC_SESSION` env var names an isolated core namespace (default: "default").
- Each session gets its own socket, holder socket, state directory, config.
- Multiple sessions = fully separate runtime namespaces.

## Agent coordination: what actually exists

### No dedicated agent-to-agent messaging API

- **OpenRPC has no `agent.*` namespace**. No method like `agent.send`, `agent.message`, `agent.broadcast`.
- **CHANGELOG entry** (herdr-core/CHANGELOG.md:5): "Agent messages and parent notes wait for you to send or clear an unfinished draft, instead of submitting it with their message. A draft left alone holds messages for at most a minute. Sends to another machine wait too. (#64)"
  - This refers to `terminal.send` behavior when a human has a draft in their editor — it waits for the human to send/clear the draft before delivering the agent's message. **Not agent-to-agent messaging.**
- **Previous investigation** (grep for 'message' across crates/herdr-core/src): Found only:
  - `notes.rs`: Parent-child notes (when a terminal finishes, parent gets a note)
  - `runtime.rs:66`: `messages` field in `State` — "A message was typed into this agent while it worked (so it sits in its queue) and no wait of its has returned since: who sent it" — this is for `terminal.wait` tracking, not inter-agent messaging.
  - `api/run.rs:32-35`: `first_message` — template's body sent to new terminal when agent ready.
  - `terminal.send` — sends text to a terminal (human or agent), not agent-to-agent.

### What exists instead: Parent-child coordination via `terminal.send` + `terminal.wait`

1. **Parent starts child**: `terminal.run { parent: "t3", ... }` — child terminal records parent ID (terminals.rs).
2. **Parent sends task**: `terminal.send { terminal: "t7", text: "review src/x.rs", wait: true }` — delivers text, optionally waits for child's turn to complete.
3. **Child finishes**: Parent's `terminal.wait { children: "recursive" }` returns when all children done (waiting.rs:16-19).
4. **Notes**: When child finishes, parent gets a "note" in its terminal (notes.rs) — human-visible, not programmatic API.

### No agent identity / routing in core

- Core knows terminals, not agents. An "agent" is just a terminal running a command like `claude` or `pi`.
- Agent **detection** (working/blocked/idle/done) is in `herdr` (the GUI app), not `herdr-core`. Detection uses screen pattern matching via TOML manifests (hd_main/src/detect/manifests/*.toml).
- No agent registry, no agent addressing, no message routing between agents.

## Does herdr-core wrap/manage CODING AGENTS?

**No.** herdr-core is a **terminal/PTY runtime**, not an agent manager.

### Evidence

1. **Templates are user-defined markdown files** (templates.rs:12-25, docs/templates.md):
   ```markdown
   ---
   description: Read-only code review
   command: pi
   ---
   You are a focused review agent...
   ```
   - `command` is any shell command (e.g., `pi`, `claude`, `codex`, `opencode`, `kiro`, `bash`).
   - Core does not know what `pi` or `claude` are; it just runs the command.
   - No built-in templates ship with herdr-core.

2. **No agent adapters in herdr-core**:
   - `crates/herdr-core/src/` has no `agent/`, `adapter/`, or `integration/` directories.
   - No code references `claude`, `codex`, `opencode`, `kiro`, `pi` as special cases.

3. **Agent detection lives in herdr (GUI app)**:
   - `hd_main/src/detect/manifests/` contains 20+ TOML files (claude.toml, codex.toml, opencode.toml, kiro.toml, pi.toml, etc.) with regex rules for detecting agent state from screen content.
   - `hd_main/src/integration/` has per-agent scripts (e.g., `assets/claude/herdr-agent-state.sh`) for session restore.
   - This is **herdr-specific**, not in herdr-core.

4. **herdr-core README** (README.md): "run programs and coding agents in terminals that stay alive" — "coding agents" are just programs you run.

5. **CHANGELOG** (herdr-core/CHANGELOG.md:25): "`hc wait --children` waits for a helper that was given a task when it started until that task is done" — "helper" = child terminal, not a managed agent.

### What an external orchestrator would need to build

- Agent registry (map agent role → terminal ID)
- Agent lifecycle (start via `terminal.run` with template, monitor via `events.subscribe` + `terminal.wait`)
- Inter-agent messaging (build on `terminal.send` + `terminal.wait` with caller tracking)
- Agent-specific configs (templates stored in orchestrator, pushed to `~/.config/herdr-core/templates/`)

## Reuse assessment for an external orchestrator

### What works well

| Capability | Assessment | Evidence |
|------------|------------|----------|
| **JSON-RPC API** | Complete, well-defined, versioned | `hc.openrpc.json` (32 methods), `rpc.discover`, generation handshake |
| **Transport** | Standard Unix socket, framed mode for streaming | `server.rs`, `wire.rs`, `hello` with `framed: true` |
| **Terminal lifecycle** | Robust: spawn, keep-alive, restart, detach | `holder.rs`, `terminals.rs`, `server.restart` |
| **Session persistence** | Atomic disk writes, survives core restart | `persistence.rs`, `paths.rs`, `registry.json` |
| **Remote execution** | `machine.exec`, `machine.add`, file ops | `exec.rs`, `remote.rs`, `api.rs` machine/file methods |
| **Real-time events** | `events.subscribe` with snapshot + streaming | `api.rs` events.subscribe, `runtime.rs` |
| **Wait-for-completion** | `terminal.wait` with output/children/timeout | `waiting.rs`, `api.rs` terminal.wait |

### Critical gaps for orchestrator use case

| Gap | Impact | Workaround |
|-----|--------|------------|
| **No agent abstraction** | Orchestrator must build agent registry, templates, detection | Build in orchestrator layer |
| **No agent-to-agent messaging** | Cannot route messages between agents natively | Use `terminal.send` + caller tracking + `terminal.wait --children` |
| **No built-in agent configs** | Must manage templates externally | Store templates in orchestrator, write to config dir |
| **Single-tenant core** | One core = one session; no multi-user isolation beyond `HC_SESSION` | Run multiple cores (one per user/project) |
| **No auth/authorization** | Socket access = full control | File permissions on socket dir; SSH tunneling via `hc connect` |
| **No metrics/observability API** | Cannot query resource usage, queue depths | `server.status` only gives counts |
| **Holder crash = data loss** | PTYs die if holder crashes | Accept risk; design for core restart, not holder restart |

### Embedding vs. standalone

- **herdr-core is BOTH a library and a binary** (Cargo.toml: `[[bin]] name = "herdr-core" path = "src/main.rs"` + library crate).
- **But**: The binary is the intended deployment unit. The library API is not documented for external embedding; internal types (`Core`, `State`, `Terminal`) are not public API.
- **Recommended**: Run `herdr-core` as a sidecar daemon, communicate via Unix socket + JSON-RPC. This matches the project's own CLI (`hc`) architecture.

## Limitations and license obligations

### License: Apache-2.0 (both repos)

**File**: `/home/deval/.hermes/cache/scratch/hd_core/LICENSE` and `/home/deval/.hermes/cache/scratch/hd_main/LICENSE`

**Key obligations**:
1. **Preserve license and copyright notices** in all copies/substantial portions (Section 4).
2. **Include Apache-2.0 license text** in distributions (Section 4).
3. **State changes** made to licensed files (Section 4).
4. **Trademark clause** (Section 6): "This License does not grant permission to use the trade names, trademarks, service marks, or product names of the Licensor, except as required for reasonable and customary use in describing the origin of the Work and reproducing the content of the NOTICE file."
   - Project name "Herdr" / "herdr-core" likely trademarked. Cannot name forked product "Herdr" without permission.
5. **Patent grant** (Section 3): Contributors grant patent license; litigation terminates license.
6. **No warranty** (Section 7): "AS IS" basis.

### Technical limitations

1. **Unix socket only** — No network exposure without SSH tunnel (`hc connect`) or reverse proxy.
2. **Single-threaded core state lock** — All API calls serialize on `Core` mutex (runtime.rs). High-throughput orchestration may contend.
3. **No horizontal scaling** — One core per session. Multiple machines via `machine.add` but they run their own cores.
4. **Linux/macOS/Windows only** — No WASM, no browser, no mobile.
5. **Rust-only implementation** — Orchestrator in another language must speak JSON-RPC over Unix socket (doable but needs socket library).
6. **No built-in secret management** — Env vars passed to `terminal.run` are visible in process table.
7. **Template system is file-based** — No API to create/update templates programmatically (only `templates.list` reads filesystem).

## Evidence

### Local files (verified by read_file)
- `/home/deval/.hermes/cache/scratch/hd_core/schema/hc.openrpc.json` — OpenRPC document (73,226 chars, 32 methods)
- `/home/deval/.hermes/cache/scratch/hd_core/schema/released.openrpc.json` — Duplicate
- `/home/deval/.hermes/cache/scratch/hd_core/schema/README.md` — Schema documentation (13,638 chars)
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/Cargo.toml` — Library + binary config
- `/home/deval/.hermes/cache/scratch/hd_core/Cargo.toml` — Workspace config
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/api.rs` — Method table (32 methods)
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/server.rs` — Unix socket server (lines 55-66)
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/client.rs` — Client connection (lines 72-115)
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/wire.rs` — Framed protocol
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/runtime.rs` — Core state
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/runtime/persistence.rs` — Atomic registry writer
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/holder.rs` — PTY owner, process parent
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/terminals.rs` — Terminal registry
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/templates.rs` — Template loading
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/notes.rs` — Parent-child notes
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/paths.rs` — Socket/state paths
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/api/run.rs` — terminal.run impl
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/api/waiting.rs` — terminal.wait impl
- `/home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/api/types.rs` — Request/response types
- `/home/deval/.hermes/cache/scratch/hd_core/CHANGELOG.md` — Release notes (agent messages entry line 5)
- `/home/deval/.hermes/cache/scratch/hd_core/docs/templates.md` — Template documentation
- `/home/deval/.hermes/cache/scratch/hd_core/docs/commands.md` — CLI reference
- `/home/deval/.hermes/cache/scratch/hd_core/LICENSE` — Apache-2.0
- `/home/deval/.hermes/cache/scratch/hd_main/Cargo.toml` — herdr binary config
- `/home/deval/.hermes/cache/scratch/hd_main/src/detect/manifests/*.toml` — 20 agent detection manifests
- `/home/deval/.hermes/cache/scratch/hd_main/src/integration/` — Agent integrations
- `/home/deval/.hermes/cache/scratch/hd_main/distribution/agent-guide.md` — Agent documentation
- `/home/deval/.hermes/cache/scratch/hd_main/LICENSE` — Apache-2.0

### GitHub URLs (fetched via curl, rate limited)
- `https://api.github.com/repos/herdrdev/herdr` — Rate limited
- `https://api.github.com/repos/herdrdev/herdr-core` — Rate limited
- `https://api.github.com/repos/herdrdev/herdr-app` — Rate limited
- `https://api.github.com/repos/herdrdev/herdr-layout-examples` — Rate limited
- `https://api.github.com/repos/herdrdev/herdr-nix` — Rate limited
- `https://api.github.com/repos/herdrdev/herdr/license` — Not directly fetched (rate limited)
- `https://api.github.com/repos/herdrdev/herdr-core/license` — Not directly fetched (rate limited)

### Search/grep evidence (verified by execute_code)
- `grep -rn "agent" /home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/` — No agent adapters in core
- `grep -rn "message" /home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/` — Only notes, wait tracking, first_message
- `grep -rn "claude\|codex\|opencode\|kiro" /home/deval/.hermes/cache/scratch/hd_core/crates/herdr-core/src/` — No matches in core
- `grep -rn "agent.*message" /home/deval/.hermes/cache/scratch/hd_core/CHANGELOG.md` — Line 5: human draft handling, not agent-to-agent
- `find /home/deval/.hermes/cache/scratch/hd_main/src/detect/manifests -name "*.toml"` — 20 agent manifests in herdr (not core)