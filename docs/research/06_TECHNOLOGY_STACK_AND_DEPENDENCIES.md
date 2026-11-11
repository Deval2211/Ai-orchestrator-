# 06 — Technology Stack and Dependencies

**Date:** 2026-11-09 · **Status:** proposal, awaiting approval.

---

## 1. Recommended stack

| Concern | Choice | Why | Alternatives evaluated |
|---|---|---|---|
| **Language** | **Python 3.12+** | Hermes is Python; the plugin API is Python (`ctx.register_tool`, `ctx.llm`, `ctx.register_hook`); every CLI in scope ships a Python SDK or is trivially driven by `asyncio.create_subprocess_exec`; MCP SDK is Python-first. | **Rust** — matches herdr-core and gives a single binary, but forfeits the in-process plugin path and every Python SDK. **Go** — matches Multica; fine, but no advantage here. |
| **Async runtime** | **asyncio** | Every adapter is I/O-bound (HTTP calls, subprocess streams). `asyncio.create_subprocess_exec` + `StreamReader` gives native streaming. | Twisted/trio — no benefit. |
| **HTTP client** | **httpx** (async) | Async streaming, HTTP/2, timeouts. Already Hermes's HTTP stack. | `aiohttp` — heavier; `requests` — sync only. |
| **Data validation** | **Pydantic v2** | The registry, the resource object and the adapter interface are all schemas. Pydantic gives validation + JSON Schema for the MCP tool definitions for free. | dataclasses + manual validation — more code, no schema export. |
| **Storage** | **SQLite** (stdlib `sqlite3`, or `aiosqlite`) | Durable, queryable, zero services, single-writer. Matches Hermes (`state.db`, `kanban.db`). The brief forbids unjustified infra. | **PostgreSQL** (the brief's proposal) — real operational cost, no demonstrated need. **JSON files** — poor queryability, no concurrent-writer safety, awkward migrations. **Redis** — no justification. |
| **Schema migrations** | **Alembic** (SQLAlchemy) *or* plain numbered SQL files | Alembic if SQLAlchemy is used; plain SQL if `sqlite3` directly. Either way, migrations must be versioned. | — |
| **ORM** | **None in the MVP** — raw SQL with `aiosqlite` | The schema is 7 tables with simple queries. An ORM adds a dependency and hides the SQL. Revisit if the schema grows. | SQLAlchemy — fine, but not needed yet. |
| **MCP server** | **FastMCP** (`mcp` package) | Python's reference MCP SDK; FastMCP gives tool decorators + schema generation from type hints, which pairs perfectly with Pydantic. | Hand-rolled JSON-RPC over stdio — more code, no benefit. |
| **Hermes integration** | **Plugin** (`~/.hermes/plugins/`) + **MCP** (`mcp_servers:` in config.yaml) | Both documented and supported. The plugin carries the Kanban lane; MCP carries the tools. | Delegation-provider plugin — wrong shape. Writing to `state.db` — unsupported. |
| **CLI execution** | **`asyncio.create_subprocess_exec`**, one process group per task (`start_new_session=True`) | Native, no dependency. Process-group isolation lets us signal the whole tree. | **herdr-core** (Apache-2.0, 32-method JSON-RPC over a Unix socket) — excellent for durable PTYs and remote machines, but it manages *screens*, not protocols. Use it as an optional fallback adapter, not the core. **ptyprocess** — only needed for interactive tools; every CLI in scope has a headless mode. |
| **Browser automation** | **Playwright** (deferred, not in the MVP) | If browser AI returns. | — |
| **Containerisation** | **Docker Compose** (optional) | Only if the orchestrator needs to run somewhere other than this machine. | — |
| **Testing** | **pytest** + **pytest-asyncio** | Standard, async-native. | — |
| **Linting / formatting** | **Ruff** | Fast, single tool, replaces flake8+black+isort. | — |
| **Type checking** | **mypy** (or **pyright**) | The adapter interface is the load-bearing contract; it must be typed. | — |
| **Logging** | **stdlib `logging`** + structured JSON formatter | No dependency. **Never log secret values** — redact at the adapter boundary. | structlog — nice, but optional. |
| **Secrets** | **env refs** (`env:VAR_NAME`) resolved at call time; `~/.hermes/.env` as the store | Matches Hermes's existing convention (`FREELLMAPI_API_KEY`, `NVIDIA_API_KEY`, `SEARXNG_URL` already live there). | OS keyring — better, but adds a dependency and a prompt per access. |
| **Monitoring** | **SQLite cost/execution ledger** + a `hermes dashboard`-style read path | The ledger is the source of truth; a dashboard is later. | OpenTelemetry / Prometheus — unjustified for a single-user tool. |
| **API gateway** | **None** | Hermes already rotates keys and fails over. A gateway duplicates it and adds a licence question (LiteLLM's enterprise directory has production-use restrictions). | LiteLLM — revisit only if multi-tenant billing appears. |

---

## 2. Dependency list (MVP)

### Runtime

```
httpx>=0.27          # async HTTP for LLM API adapters
mcp>=1.0             # MCP server SDK (FastMCP)
pydantic>=2.7        # schemas + validation + JSON Schema export
aiosqlite>=0.20      # async SQLite (or stdlib sqlite3 in a thread executor)
```

**That's it.** Four runtime dependencies. Everything else is stdlib.

### Development

```
pytest
pytest-asyncio
ruff
mypy
```

### Deliberately NOT dependencies

| Not used | Why |
|---|---|
| **LangChain / LlamaIndex / CrewAI / AutoGen** | Agent frameworks that wrap model calls. This project routes to *resources* and dispatches to *CLIs*; an agent framework would be a second, competing control plane. |
| **SQLAlchemy** | 7 simple tables; raw SQL is clearer and has zero dependency cost. |
| **PostgreSQL driver / Redis client** | No such services. The brief explicitly warns against unjustified infrastructure. |
| **Docker SDK** | Not needed — the orchestrator spawns processes directly. |
| **Prometheus / OpenTelemetry** | No metrics infrastructure for a single-user tool. The SQLite ledger is enough. |
| **LiteLLM** | Duplicates Hermes's rotation + failover; licence has an enterprise restriction on production use of the `enterprise/` tree. |
| **herdr-core (Rust)** | Not needed in the MVP — every CLI has a structured native interface. Optional sidecar later. |
| **Playwright** | Browser AI is deferred. |

---

## 3. Language decision, in depth

The brief leaves the language open (Python / Rust / Go). The deciding factor is **not**
performance — it is the integration surface.

**Python wins because the integration is Python:**

1. **The Hermes plugin API is Python.** `ctx.register_tool(name, toolset, schema, handler, ...)`,
   `ctx.llm.complete(messages=...)`, `ctx.register_hook(...)`. A plugin is a directory with
   `plugin.yaml` and Python code. Writing the Kanban lane in Python is ~100 lines; in Rust it
   would need a separate process and an IPC protocol.
2. **The CLI SDKs are Python or trivially scriptable.** OpenCode has `@opencode-ai/sdk`
   (TypeScript) but also `opencode serve` over HTTP (any language). Claude Code ships a
   **Python** Agent SDK. Pi has an RPC mode. Only Kiro needs a subprocess wrapper.
3. **MCP's Python SDK is the reference implementation.** FastMCP's decorators generate the tool
   schemas the model sees — which pairs directly with the Pydantic models the registry already
   uses.
4. **Async I/O is where the complexity is, and Python's asyncio is sufficient.** Every adapter is
   I/O-bound. A Rust async runtime would not make this meaningfully faster; it would make
   integration harder.

**Where Rust would win:** if the orchestrator needed to embed herdr-core as a library (rather
than talk to it over JSON-RPC), or needed sub-millisecond routing latency (it does not — routing
is a decision, not a hot loop).

**Recommendation: Python.** Revisit only if the orchestrator becomes a long-running, high-fan-out
service where process-level overhead dominates.

---

## 4. Storage decision, in depth

**Recommendation: SQLite, one file, stdlib or `aiosqlite`.**

| Requirement | SQLite | PostgreSQL | JSON files |
|---|---|---|---|
| Durable | ✅ WAL mode | ✅ | ✅ |
| Queryable | ✅ SQL | ✅ | ❌ (load + parse) |
| Zero extra services | ✅ | ❌ needs a server | ✅ |
| Concurrent writers | ⚠️ one writer at a time (we have one process) | ✅ | ❌ race-prone |
| Schema migrations | ✅ versioned SQL | ✅ | ⚠️ hand-rolled |
| Matches Hermes | ✅ `state.db`, `kanban.db` | ❌ | — |
| Operational cost | none | a service to run, secure, back up, patch | none |
| Honest verdict for this project | **right size** | over-engineered | too weak |

The brief proposes PostgreSQL as "shared infrastructure". That proposal is not justified at this
scale: single user, single process, seven tables, thousands of rows. SQLite satisfies every
stated requirement with zero operational burden. If the project later becomes multi-tenant or
needs concurrent writers from multiple processes, PostgreSQL is the migration target — and the
schema in `05_RECOMMENDED_SYSTEM_ARCHITECTURE.md` is deliberately written in portable SQL for
that reason.

**WAL mode matters:** set `PRAGMA journal_mode=WAL` so a long write does not block the
orchestrator's reads.

---

## 5. MCP server design

The MCP server is the orchestrator's public face to Hermes. Tool design matters: every tool
description is read by the model.

```python
# orchestrator/mcp_server.py
from mcp.server.fastmcp import FastMCP
from pydantic import BaseModel, Field

mcp = FastMCP("ai-orchestrator")

class PlanRequest(BaseModel):
    goal: str = Field(description="What the task must achieve, in one or two sentences.")
    workspace: str | None = Field(default=None,
        description="Absolute path to the working directory. Required for tasks that modify files.")
    capabilities: list[str] = Field(default_factory=list,
        description="Required capabilities, e.g. ['code_generation','browser']. Omit to let the router infer them.")
    budget_usd: float | None = Field(default=None, description="Maximum spend for this task.")
    idempotency_key: str | None = Field(default=None,
        description="Caller-generated key. Reusing a key returns the existing execution instead of re-running.")

@mcp.tool()
async def plan_task(req: PlanRequest) -> dict:
    """Classify a task and select the smallest capable resource for it.
    Returns a plan (resource, account, estimated cost and time). Does not execute."""
    ...

@mcp.tool()
async def execute_task(execution_id: str) -> dict:
    """Execute a previously-created plan and stream progress. Returns the final result, artifacts and cost."""
    ...

@mcp.tool()
async def get_execution(execution_id: str) -> dict:
    """Return the full record of an execution: plan, status, artifacts, cost, native session id."""
    ...

@mcp.tool()
async def list_resources(type: str | None = None) -> list[dict]:
    """List registered resources and accounts, with availability and health."""
    ...
```

**Tool-count discipline:** five tools. Each extra tool costs tokens in every turn where it is
available and dilutes the model's choice. `list_resources` is for the *user* asking what is
available; `plan_task` + `execute_task` are the normal path.

**`plan_task` and `execute_task` are separate on purpose.** A plan is cheap, inspectable, and
cancellable — and splitting them lets the user see (and edit) the routing decision before a
dollar is spent. It also gives the approval flow a natural place to intervene.

---

## 6. Adapter interface

```python
# orchestrator/adapters/base.py
from dataclasses import dataclass
from typing import AsyncIterator, Protocol, Literal

class ResourceType(Literal["llm_api", "coding_cli", "browser_service"]:
    ...

@dataclass(frozen=True)
class ExecutionRequest:
    goal: str
    workspace: str | None
    context_bundle: dict
    idempotency_key: str
    timeout_s: int
    trust_all_tools: bool          # maps to --trust-all-tools / --auto / --yolo

@dataclass(frozen=True)
class ExecutionEvent:
    kind: Literal["started", "output", "tool_call", "artifact", "progress", "finished"]
    text: str | None = None
    path: str | None = None
    data: dict | None = None

@dataclass(frozen=True)
class ExecutionResult:
    status: Literal["succeeded", "failed", "cancelled"]
    output: str
    artifacts: list[str]
    native_session_id: str | None
    error_class: str | None
    cost_usd: float
    usage: dict

class Adapter(Protocol):
    resource_type: ResourceType

    async def health(self) -> dict:
        """Cheap liveness probe. MUST NOT make a billable model call."""

    async def execute(self, req: ExecutionRequest) -> AsyncIterator[ExecutionEvent]:
        """Run the task. Yields events; the final event carries the result."""

    def supports_resume(self) -> bool: ...
```

**Why `AsyncIterator`:** every CLI in scope streams (`--output-format stream-json`, ACP frames,
SSE from `opencode serve`), and the Hermes conversation should show progress rather than a
multi-minute silence.

**Why `health()` must not call a model:** the router probes every candidate resource before
choosing. A billable probe would make routing cost money.

---

## 7. Per-CLI adapter mapping

| CLI | Transport | Headless invocation | Streaming | Resume | Isolation |
|---|---|---|---|---|---|
| **OpenCode** | HTTP (`opencode serve`) — preferred | `POST /session` + prompt | SSE | session id | `XDG_CONFIG_HOME` + `XDG_DATA_HOME`, **one server per account** |
| **Kiro CLI** | subprocess | `kiro-cli chat --no-interactive --output-format stream-json "<prompt>"` | NDJSON | `--resume-id <id>` | `KIRO_HOME` + `KIRO_API_KEY`; **auth not isolated — single account only until proven** |
| **Claude Code** | subprocess | `claude -p --output-format stream-json "<prompt>"` | stream-json | `--resume <id>` | `CLAUDE_CONFIG_DIR` |
| **Pi** | subprocess (RPC mode preferred) | `pi -p` / `--mode json` | JSON events | session tree | `HOME` or `XDG_CONFIG_HOME` |
| **Cline** | subprocess | `cline --json "<prompt>"` | NDJSON | `--id` — **known broken with `--json`** | `--config` / `--data-dir` |
| **LLM APIs** | httpx | `POST {endpoint}/chat/completions` | SSE | n/a (stateless) | per-account env var |

---

## 8. Hermes plugin (Kanban lane)

```python
# ~/.hermes/plugins/orchestrator-lane/__init__.py
import httpx

def register(ctx):
    @ctx.kanban_worker_lane(assignee="orchestrator")
    def spawn(task, workspace, board=None):
        """Route a Kanban card to the ai-orchestrator resource router."""
        client = httpx.Client(base_url="http://127.0.0.1:9100", timeout=30)
        resp = client.post("/internal/spawn", json={
            "task_ref": task.id,
            "goal": task.description,
            "workspace": workspace,
            "board": board,
        })
        resp.raise_for_status()
        return resp.json().get("pid")   # optional PID for crash detection
```

**Requirements the lane must satisfy** (from Hermes' own contract):
- The worker MUST terminate its run with exactly one of `kanban_complete`,
  `kanban_request_review`, `kanban_block`.
- A worker that exits without calling any of them is treated as **crashed** and reaped.
- It must respect the fence: only its own card's lifecycle handoffs, not `kanban_unblock`.
- It must inherit the caller's credential/profile policy via
  `agent.delegation_context.delegated_child_subprocess_env`.

---

## 9. Repository layout

```
ai-orchestrator/
├── pyproject.toml
├── README.md
├── docs/research/                 # this research
├── specs/                         # to be REWRITTEN per the audit
├── phases/                        # to be REWRITTEN per the roadmap
├── references/
│   ├── decisions.md                # decision log (add D10+)
│   └── open-questions.md          # the 12 decisions needing approval
└── src/ai_orchestrator/
    ├── __init__.py
    ├── config.py                  # settings loading, paths
    ├── registry/
    │   ├── schema.py              # Pydantic: Resource, Account, Capability
    │   ├── store.py               # SQLite CRUD for resource/account/state
    │   └── loader.py              # registry YAML → store
    ├── router/
    │   ├── classify.py            # task → domain, complexity, capabilities
    │   ├── capability.py          # capability → minimum tier mapping
    │   └── select.py              # registry filter + scoring
    ├── adapters/
    │   ├── base.py                # the Adapter protocol
    │   ├── llm_api.py             # generic OpenAI-compatible adapter
    │   ├── opencode.py
    │   ├── kiro.py
    │   ├── claude_code.py
    │   ├── pi.py
    │   └── cline.py
    ├── execution/
    │   ├── manager.py             # dispatch, retry, escalation
    │   ├── checkpoint.py          # L3 checkpoint read/write
    │   ├── recovery.py            # restart recovery
    │   └── idempotency.py
    ├── accounts/
    │   ├── health.py              # cooldown, quota, auth state
    │   └── isolation.py           # per-account env / config dir
    ├── security/
    │   ├── redact.py              # secret redaction at the boundary
    │   ├── permissions.py         # permission declaration + enforcement
    │   └── approval.py            # Class-B approval tokens
    ├── storage/
    │   ├── schema.sql             # the 7-table schema + migrations
    │   └── ledger.py              # cost ledger
    └── mcp_server.py              # the MCP tool surface
```

**Tests mirror it:** `tests/` with `test_registry.py`, `test_router.py`, `test_adapters/`,
`test_execution.py`, `test_security.py`. Adapter tests must use **recorded fixtures**, never
live API calls.

---

## 10. Version pinning and reproducibility

```
# pyproject.toml — every runtime dependency pinned exactly
dependencies = [
  "httpx==0.28.1",
  "mcp==1.12.0",
  "pydantic==2.11.7",
  "aiosqlite==0.21.0",
]
```

Pin exactly. This is a single-developer tool that must keep working when the orchestrator is
restarted months later; a floating minor version is how that breaks.

---

## 11. What this stack deliberately avoids

| Avoided | Reason |
|---|---|
| Kubernetes / Kafka / Redis / Postgres | The brief forbids unjustified infrastructure; a single-user tool needs none of them. |
| Microservices | One process, one SQLite file, one MCP server. Splitting it adds deployment cost for no benefit. |
| An agent framework (LangChain/CrewAI/AutoGen) | Would be a second control plane competing with Hermes. |
| A plugin framework / DI container | Python's protocols and dataclasses are enough. |
| Async ORM | Seven tables. Raw SQL is clearer. |
| A metrics server | The SQLite ledger plus Hermes's own logs are sufficient. |
| Docker for the core | The orchestrator runs as a local process. Docker is for optional remote workers later. |
