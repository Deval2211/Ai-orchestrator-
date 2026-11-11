# 08 — Session Continuity and Recovery

**Date:** 2026-11-09 · **Status:** proposal, awaiting approval.

---

## 1. The requirement, and the honest answer

**Requirement** (`specs/session-continuity.md:3`): *"Non-negotiable: everything stays accessible
from the same Hermes conversation."*

**The honest answer:** there is **no supported API** for an external process to append to a
Hermes session. Verified twice, independently:

| Path | Status |
|---|---|
| `SessionDB.append_message` (`hermes_state_messages.py:429`) | Internal. Needs a `SessionDB` instance; not a public API; not safe across processes. |
| `PluginContext.post_message` / `inject_message` | Works **only during a live gateway session** (inside a tool call or command). Cannot append to historical sessions from a background process. |
| `hermes sessions import` (`web_routers/sessions.py:56-121`) | Batch import of a JSONL export. Not real-time. |
| Write to `state.db` directly | **Forbidden by the brief** absent proof of a supported method. No such proof exists. |

**Therefore the requirement must be reframed**, not implemented literally. The reframing that
works:

> **Continuity is a link, not a copy.** Hermes owns the conversation. The orchestrator owns the
> execution. The two are joined by a **task reference**, and the link is visible from both sides.

This is a change to the spec's wording, not a reduction in what the user gets. The user still
sees one conversation, can ask "what did that Kiro run do?", and can continue from it — because
Hermes records the tool call, and the tool call's result carries the task reference, and the
orchestrator can answer any question about that task from its own store.

---

## 2. Four distinct layers

The prior analysis in this workspace identified the core problem correctly: external CLIs keep
private sessions, and true cross-CLI session sharing is impossible without passing history and
artifacts explicitly. This document formalises that into four separate layers with different
owners and different lifetimes.

| Layer | Owner | Contents | Lifetime | Survives orchestrator restart? |
|---|---|---|---|---|
| **L1 — Conversation** | Hermes (`state.db`) | user prompts, tool calls, tool results, Hermes compression lineage | days–months | n/a |
| **L2 — Task** | Hermes (Kanban, `~/.hermes/kanban.db`) | card, assignee, workspace, status, review verdict, crash history, comments, attachments | weeks | Yes |
| **L3 — Execution** | Orchestrator (SQLite) | plan, resource, account, adapter, timing, cost, error class, artifact index, native session id | weeks–months | Yes |
| **L4 — Native session** | The CLI itself | Kiro session files, OpenCode session store, Claude Code session | CLI-defined | Yes, if the CLI's store is on disk |

**The rule:** L1 and L2 are Hermes's. L3 is ours. L4 belongs to the tool. The orchestrator's job
is to hold L3 accurately enough that L4 is always findable again.

### 2.1 What this makes possible

| User action | How it works |
|---|---|
| "What did that Kiro run do?" | Hermes conversation → tool call → `task_ref` → L3 execution record → artifacts, logs, native session id. |
| "Resume that OpenCode session." | Hermes → orchestrator → L3 gives the native session id → adapter resumes it (`opencode --session <id>`). |
| "Continue from where we left off." | Hermes loads the conversation; the orchestrator loads L3; the next adapter invocation gets the L4 session id. The CLI starts fresh *with the right history*, which is the realistic model the spec already identified. |
| "What did it cost?" | L3 `cost_ledger`, aggregated per task and per account. |
| "Why did it fail?" | L3 `error_class` + `checkpoint.state` + the CLI's own stderr, captured in L3. |

### 2.2 What this explicitly does not give

- **Not a shared context window.** Different tools have different context limits and system
  prompts. Continuity is *passing the right context in*, not sharing memory. The spec already says
  this correctly ("lossy but workable") — keep that framing.
- **Not a live view into a finished CLI session.** Once a CLI process exits, its internal state
  is gone unless the CLI persists it. Kiro does (`~/.kiro/sessions/cli/<id>.jsonl`); Cline's
  `--json` mode does **not** resume reliably (known bug, `cline/cline#10856`).
- **Not a cross-CLI transcript.** Task 2's output does not become Task 3's system prompt. The
  orchestrator passes a *summary + artifacts*, deliberately.

---

## 3. Checkpointing

### 3.1 Two levels of checkpoint

| Level | What | Owner | Mechanism |
|---|---|---|---|
| **Task checkpoint** | The card's state, workspace, assignee, run history | Hermes | Kanban row + `kanban_heartbeat` |
| **Execution checkpoint** | The adapter's resume state for one attempt | Orchestrator | L3 `checkpoint` table |

The spec only has one (`specs/autonomy-and-safety.md:26-37`) and it is ambiguous about which.
Both are needed; they are different things.

### 3.2 Execution checkpoint format

The spec's current format stores `input` and `output_so_far` as strings. **That is wrong for
long-running coding tasks** — it duplicates the CLI's own session log and grows without bound.

Revised format:

```json
{
  "task_ref": "kanban-card-id-or-hermes-tool-call-id",
  "execution_id": "uuid",
  "attempt": 2,
  "resource_id": "opencode:default",
  "account_id": "opencode:default:acct-work",
  "adapter": "opencode_http",
  "workspace": "/home/deval/.hermes/kanban/workspaces/board/card",
  "native_session_id": "ses_7f3a...",
  "resume_token": null,
  "state": {
    "phase": "executing",
    "started_at": "2026-11-09T17:00:00Z",
    "last_event_at": "2026-11-09T17:12:33Z",
    "last_event_summary": "wrote src/auth.py (41 lines)"
  },
  "artifacts": [
    {"path": "src/auth.py", "sha256": "…", "bytes": 1420}
  ],
  "error": null,
  "created_at": "2026-11-09T17:00:01Z"
}
```

**Rules:**
- **Never store raw output.** Store a bounded `last_event_summary` and artifact hashes. The full
  transcript lives in the CLI's own store (L4) and in Hermes's tool-result (L1).
- **Never store secrets.** Redacted before write.
- **Store the native session id.** This is the single most important field: it is what makes
  resume possible after the orchestrator restarts.
- **Bounded size.** Cap `state.last_event_summary` at ~1 KB; keep only the last N artifact entries.

### 3.3 When a checkpoint is written

| Trigger | Checkpoint contains |
|---|---|
| Execution starts | attempt 1, native_session_id once known |
| Adapter emits a "phase" event (e.g. tool call completed, file written) | updated `state` + artifact delta |
| Every N seconds during a long run (N ≈ 30) | updated `state` |
| Execution fails | attempt + error class + resume token |
| Execution succeeds | final state + full artifact list |

**Why not just let Kanban do it?** Kanban checkpoints the *task*, not the *resource*. A CLI that
was killed mid-tool-call leaves a Kanban card in `running` → `crashed`, but Kanban does not know
the CLI's session id and cannot resume the CLI. That is precisely the orchestrator's L3 job.

---

## 4. Recovery

### 4.1 Recovery matrix

| Failure | Detected by | Recovery | Owner |
|---|---|---|---|
| Orchestrator process dies | — | On restart, scan L3 for `status='running'` rows older than a threshold → mark `interrupted` → re-dispatch with the stored `native_session_id` | Orchestrator |
| CLI process dies mid-task | Adapter sees exit / stream EOF | Retry with the same native session id (resume) up to `max_attempts`; then escalate to a stronger resource; then `kanban_block` | Orchestrator → Kanban |
| CLI process hangs | Adapter timeout | `terminal.interrupt`-equivalent (SIGINT → SIGTERM) → retry → give up | Orchestrator |
| Provider 429 | HTTP status + `Retry-After` | Cool the account until the reset time; rotate within the provider via Hermes's credential pool if the path is an LLM API | Hermes pool (API) / Orchestrator (CLI) |
| Provider 401 | HTTP status | Attempt token refresh; on failure mark the account `auth_required` and exclude it from routing | Orchestrator |
| Provider 402 (billing/quota) | HTTP status | **Cooldown, do not rotate indefinitely.** Surface to the user. Rotation here would be quota evasion | Orchestrator |
| Kanban circuit breaker trips | Kanban emits `gave_up` | Card → `blocked`; user decides | Kanban |
| Workspace deleted mid-task | Filesystem check before resume | Abort and report | Orchestrator |

### 4.2 Restart recovery procedure

```
on_startup:
  rows = SELECT * FROM execution WHERE status='running'
  for row in rows:
    if now - row.started_at > STALE_THRESHOLD:
        UPDATE execution SET status='interrupted', error_class='orchestrator_restart'
        if row.native_session_id and adapter.supports_resume:
            requeue(row)          # same session id, attempt+1
        else:
            requeue(row)          # fresh attempt, same plan
```

The key property: **the orchestrator can be killed at any moment and lose nothing but wall-clock
time**, because the plan, the account, the workspace, and the native session id were all persisted
before dispatch.

### 4.3 Retry policy

The spec says "automatic retries and fallback" with no detail. Proposed:

| Error class | Retry? | Action |
|---|---|---|
| `timeout` | Yes | Same resource, same account, attempt+1, with backoff. Up to 2 retries. |
| `429` | Yes | Cool the account to the provider's reset; retry after. Do not burn attempts while cooling. |
| `5xx` | Yes | Same resource, attempt+1, exponential backoff (2s, 8s, 30s). Up to 3 retries. |
| `401` | Once | Refresh token. On failure → account `auth_required`. |
| `402` | No | Cooldown the account, surface to user. |
| `400` (bad request) | No | The plan is wrong → re-plan with a different resource. |
| `process_exit_nonzero` | Depends | If the CLI exited with a tool/verification error → re-plan (escalate). If it exited cleanly but verification failed → escalate. |
| `cancelled` | No | User-initiated. |

**Escalation** (re-plan with a higher `strength_tier`) is bounded by:
- `max_escalations` per task (default 2), and
- a per-task spend cap.

Both are currently missing from the spec entirely.

### 4.4 Idempotency

Missing from the spec (zero mentions across `specs/` and `phases/`).

Every dispatch carries an **idempotency key** = `sha256(task_ref + plan_hash + attempt_group)`.
The orchestrator rejects a dispatch whose key matches a completed execution unless the caller
explicitly forces a new attempt. This prevents the classic failure: the orchestrator's HTTP call
times out, the caller retries, and the task runs twice — double-charging and double-writing.

For LLM APIs, add a `conversation_id`-style correlation id so the provider can dedupe where it
supports it (OpenAI and Anthropic both support request idempotency hints).

---

## 5. Cancellation

Missing from the spec.

| Level | Mechanism |
|---|---|
| Cancel the whole task | The orchestrator marks the execution `cancelled`, sends SIGINT (then SIGTERM, then SIGKILL after a grace period) to the child process group, and records the partial state. |
| Cancel one adapter | Same, scoped to one execution. Other parallel executions continue. |
| Cancel from Hermes | The tool call is interrupted; Hermes sends an interrupt to the running tool; the orchestrator's adapter sees the cancellation signal and terminates the child. The cancellation must propagate through the MCP call, which requires the orchestrator to handle client disconnects. |

**Practical note:** a CLI that has already written files does not "un-write" them on cancel.
Cancellation stops further work; it does not roll back. Rollback is the job of git worktrees
(T3 Code's model) — if the workspace is a git worktree, cancelling can discard the branch.

---

## 6. Context passing between tasks

The spec says *"relevant history and artifacts are passed into the next tool invocation (lossy but
workable)"* — correct, and it needs a concrete mechanism.

### 6.1 The context bundle

```json
{
  "task_ref": "...",
  "goal": "one-line statement of what this task must achieve",
  "decisions": [
    {"decision": "Use SQLite, not Postgres", "rationale": "single-user, no infra budget", "at": "..."}
  ],
  "artifacts": [
    {"path": "src/auth.py", "summary": "implements token refresh", "sha256": "..."}
  ],
  "open_issues": [
    {"issue": "rate-limit retry untested", "severity": "medium"}
  ],
  "prior_executions": [
    {"execution_id": "...", "resource": "kiro-cli", "status": "failed", "summary": "..."}
  ],
  "budget": {"spent_usd": 0.02, "cap_usd": 0.50, "attempts": 1, "max_attempts": 3}
}
```

### 6.2 Rules

- **Never pass raw transcripts.** Pass the bundle.
- **Always pass artifact paths, not contents** — unless the task explicitly needs the content,
  in which case pass a bounded excerpt plus the path.
- **Always pass the failure summary from the previous attempt**, so attempt 2 does not repeat
  attempt 1's mistake.
- **Token budget**: the bundle is capped (default ~4k tokens). If it exceeds the cap, prune
  `prior_executions` first, then `decisions`, then truncate `artifacts` summaries. Never prune
  `goal` or `open_issues`.
- **The bundle is written to the workspace** as `context.json` so the CLI can read it, and passed
  in the prompt so a CLI without file access still sees it.

---

## 7. Artifact storage

| Artifact type | Where | Metadata in |
|---|---|---|
| Source files the CLI produced | The task workspace (in place) | L3 `artifact` table (path, sha256, bytes) |
| Build/test output | `<workspace>/.orchestrator/artifacts/` | L3 `artifact` table |
| Final deliverables (reports, charts) | Copied to `~/.hermes/cache/orchestrator/artifacts/<execution_id>/` | L3 `artifact` table |
| Logs | `<workspace>/.orchestrator/logs/<adapter>.log` | path + size in L3 |

**Rules:**
- Artifacts are **content-addressed** by sha256, so a resumed attempt can tell what actually
  changed rather than re-running everything.
- The orchestrator never writes into the user's real project directories unless the task's
  workspace *is* that project and the plan says so (and then it is a Class-B action).
- Retention: artifacts older than N days are pruned by size, not age alone. The L3 record keeps
  the metadata even after the file is gone (so the record stays truthful).

---

## 8. What Hermes already gives us here — do not rebuild

| Capability | Hermes mechanism | Verified |
|---|---|---|
| Conversation history + compression | `state.db`, `hermes_state_compression.py` | `07-hermes-integration.md:286-290` |
| Task state, workspace pinning, crash/timeout/give-up reaping | Kanban | kanban-worker-lanes doc |
| Respawn after `kanban_unblock` | Kanban dispatcher | kanban-worker-lanes doc |
| Comment/attachment history on a task | `kanban_comment`, `kanban_attach`, `kanban_attach_url` | kanban doc |
| Retry policy on a card | Kanban consecutive-failure circuit breaker | kanban doc |
| Review → rework routing | `kanban_request_review` → `kanban_request_changes` | kanban-worker-lanes doc |
| Budget checkpoint near the iteration cap | `agent.budget_warning_ratio` | kanban doc |
| Credential rotation across retries | Credential pools | credential-pools doc |
| Tool-result recording in the transcript | Native | — |

**The orchestrator adds exactly three things Hermes cannot:**
1. the **native session id** for each CLI, so a *tool* can be resumed;
2. the **artifact index** with hashes, so a resumed attempt knows what changed;
3. the **cost ledger**, so spend is attributable per task and per account.

---

## 9. Open questions this document does not resolve

1. **Should the orchestrator's execution history be visible in the Hermes transcript, or only
   referenced from it?** Referenced-only is what the evidence supports. If the user wants
   transcript-native history, the only supported route is to run the work *inside* the Hermes
   session (via the MCP tool), which constrains long-running work.
2. **Retention policy** for L3 and L4 — how long should execution records and CLI sessions live?
   Needs a user decision.
3. **Should cancelled tasks roll back?** Requires git worktree isolation, which the spec never
   mentions. T3 Code's worktree module is the reference implementation.
4. **Multi-repo tasks** — Kanban pins one workspace per card. A task spanning two repos needs a
   workspace policy. Not addressed in the spec.
