# 07 — Security and Account Management

**Date:** 2026-11-09 · **Status:** proposal, awaiting approval.

---

## 1. Security model in one page

The orchestrator holds credentials for many providers and spawns third-party coding agents that
can write files and run commands. Its threat model is therefore not "protect a web service" —
it is **"one compromised or confused child process must not escalate into every account we
hold."**

Four principles, in priority order:

1. **Secrets are references, never values.** The registry stores `env:VAR_NAME` or
   `keychain:ref`. The secret is resolved at the moment of use, in the child's environment only,
   and never written to the registry, the execution store, logs, prompts, or checkpoints.
2. **One account, one process, one environment.** Each account gets its own isolated HOME /
   config dir / credential set. A child cannot see another account's credentials because they are
   not in its environment.
3. **Least privilege per resource.** Each resource declares what it may do (read the repo, write
   the repo, run commands, reach the network). The adapter enforces it before dispatch.
4. **Human approval for the irreversible**, enforced where it can actually be enforced (see §5).

---

## 2. Account architecture

### 2.1 The model

An **account** is a usable credential set for a resource. A resource may have many accounts; an
account belongs to exactly one resource.

```
resource  "nvidia-nim:minimax-m3.1-flash"   (what it is, what it can do, what it costs)
  ├── account "…:acct-work"   auth_ref=env:NVIDIA_NIM_KEY_WORK   isolation={env:{}}
  └── account "…:acct-personal" auth_ref=env:NVIDIA_NIM_KEY_PERSONAL isolation={env:{}}
```

**Why this split matters.** The spec's current registry conflates the two: one `Resource` object
with a single `auth.ref`. With one `auth.ref` you cannot express "two NVIDIA accounts", which is
decision D4 (multiple API accounts) — a *decided* requirement. The split is forced by the
requirements, not invented.

### 2.2 Isolation mechanisms per resource type

| Resource type | Isolation | Verified |
|---|---|---|
| LLM API | Distinct env var per account (`NVIDIA_NIM_KEY_WORK` vs `…_PERSONAL`). No filesystem isolation needed — the key is the account. | Design |
| **OpenCode** | `XDG_CONFIG_HOME` + `XDG_DATA_HOME` per account. **Plus: one `opencode serve` instance per account** — the background server is a single point of shared auth/state, so a shared server defeats per-account isolation. | `opencode --help` shows `--server`, `--standalone`, `opencode service` |
| **Kiro CLI** | `KIRO_HOME` isolates sessions/skills/settings. **Auth is NOT isolated** — credentials live in `~/.local/share/kiro-cli/data.sqlite3`, outside `KIRO_HOME`, and a fresh `HOME`+`KIRO_HOME` triggers an interactive browser login. | Live test: fresh HOME → "Opening browser..." loop; auth DB path confirmed by directory listing + `strings` on the binary |
| **Claude Code** | `CLAUDE_CONFIG_DIR` (dedicated knob) — *not* a blanket `HOME` override, which would relocate unrelated state. | CLI research, `code.claude.com/docs/en/env-vars` |
| **Pi** | `HOME` or `XDG_CONFIG_HOME` | CLI research |
| **Cline** | `--config` / `--data-dir` | CLI research |
| Browser (deferred) | Separate Chrome `--user-data-dir` per account, never a copy of the user's real profile | Design |

**Open finding:** Kiro multi-account isolation is **unproven**. A second Kiro account needs a
second interactive browser login, which cannot be automated headlessly. Until tested with two real
accounts, treat Kiro as **single-account only** and do not build the wrapper to pretend
otherwise.

### 2.3 Provider ToS compliance

The brief is explicit: *"Do not design account switching as a way to bypass quotas, subscription
restrictions or provider policies."*

Rules this project will follow:

| Rule | Implementation |
|---|---|
| Rotation happens on **provider-signalled** failures only | Rotate on 429-with-reset, 402, 401-after-refresh-fails, and quota-exhausted responses. Never rotate to evade a *billing* or *plan* limit (402 → cooldown, not "try the next key" indefinitely). |
| No parallel bursting across accounts to exceed a plan's aggregate limit | The scheduler's concurrency limit is per **provider**, not per account. N accounts of one provider share one concurrency budget. |
| Credentials are the user's own | Every key belongs to an account the user controls. The orchestrator never sources keys it does not have. |
| Browser/subscription bridging is out of scope | Deferred past the MVP entirely — see §7. |

---

## 3. Credential storage

### 3.1 Where secrets live

| Secret class | Storage | Reference format in the registry |
|---|---|---|
| LLM API keys | `~/.hermes/.env` (already the user's convention) or a system keyring | `env:NVIDIA_NIM_KEY_WORK` |
| CLI OAuth tokens | The CLI's own credential store, inside the per-account config dir | `path:/home/deval/.accounts/kiro-work/.local/share/kiro-cli` |
| The orchestrator's own API key (for the MCP/HTTP surface) | `~/.hermes/.env` as `ORCHESTRATOR_API_KEY` | `env:ORCHESTRATOR_API_KEY` |
| Database password, if ever needed | Not applicable — SQLite, no server | — |

**The registry never contains a secret.** It contains a pointer. This is already the spec's
intent (`specs/resource-registry.md:15-16`: *"never store secrets in registry"*) — the audit just
confirms it must be enforced at the code level, not just documented.

### 3.2 Resolution rules

- Resolve the secret **at call time**, in the child's environment, never in the parent's.
- The orchestrator's own process should hold **only** the secrets it needs to proxy, and even
  then, resolve-and-scope them per call.
- A secret must never appear in: the execution record's `request` field, error messages, tool
  output that gets logged, checkpoint `state`, or the cost ledger.

### 3.3 Redaction

Implement a redaction filter on everything that leaves the orchestrator:

1. **Registry-driven** — before an execution, resolve the set of secret values that could be
   present for that account, and mask any exact occurrence in logs, output digests, and error
   text.
2. **Pattern-driven** — mask common shapes regardless of the registry: `sk-…`, `nvapi-…`,
   `Bearer <token>`, `AKIA…`, `ghp_…`, `-----BEGIN … PRIVATE KEY-----`, and any
   `*_API_KEY=…` / `*_TOKEN=…` assignment.
3. **Apply at the boundary** — redaction happens in the adapter, before the event is emitted, so
   no unredacted copy ever reaches the store.

---

## 4. Permissions and least privilege

### 4.1 Per-resource capability declaration

```yaml
# in the registry's `declared` block
permissions:
  filesystem:
    read:  ["/home/deval/Projects/**"]      # workspace + repo
    write: ["/home/deval/Projects/**"]      # workspace only (Kanban pins it)
    deny:  ["/home/deval/.ssh", "/home/deval/.aws", "/home/deval/.hermes/.env"]
  network:
    allow: ["api.nvidia.com", "openrouter.ai", "127.0.0.1"]
    deny:  ["*"]                             # default deny
  subprocess:
    allow: ["/home/deval/.hermes/tools/**/opencode", "/home/deval/.local/bin/kiro-cli"]
  credentials:
    env: ["NVIDIA_NIM_KEY_WORK"]
```

The adapter enforces `permissions` **before** dispatch: it builds the child's environment and
argv, then validates that the resolved command and paths are inside the allowlist.

### 4.2 The deny list is not optional

The highest-value rules in the whole file are the denies. A coding agent that can read
`~/.ssh/id_ed25519` or `~/.hermes/.env` converts "the agent ran a bad command" into "the agent
exfiltrated every credential on the machine". The orchestrator must set the child's HOME to the
per-account dir (so `~/.ssh` resolves inside the account, not the user's real home) **and** deny
the absolute paths as a second layer.

### 4.3 Kanban workspaces as the write boundary

Hermes Kanban already provides per-task pinned workspaces
(`HERMES_KANBAN_WORKSPACE`, `HERMES_KANBAN_WORKSPACES_ROOT`). The orchestrator should require
that every coding-CLI execution declares its workspace, and refuse to dispatch into a path
outside the board's workspace root. This is the same idea as T3 Code's git-worktree isolation,
using a primitive Hermes already ships.

### 4.4 Kanban fencing is not a security boundary

Hermes's own docs are explicit: the descendant-process fence is *"cooperative runtime scoping,
not OS confinement — it does not prevent arbitrary code from deliberately erasing lineage
metadata."* It prevents accidents, not adversaries. Real isolation is separate OS processes,
separate HOME/config dirs, and a restricted environment.

---

## 5. Approval — what can actually be enforced

**The finding that changes the design:** Hermes can surface an approval request **only when the
action is invoked as a Hermes tool**. A `pre_tool_call` plugin hook can block the call and show
the user a message (`hermes_cli/plugins.py:1912-1984`), and the hook receives `turn_id`,
`tool_call_id`, `session_id` for correlation (`tools/approval_context.py:66-69`). But a
background orchestrator process with no active tool call has **no way to raise an approval
prompt and no way to receive the answer**.

The current spec (`specs/autonomy-and-safety.md:16-22`) assumes otherwise, and lists
"network calls to external services" as approval-requiring — which, read literally, would mean
*every LLM API call needs approval*. That would make the system unusable.

### 5.1 The enforceable policy

Split actions into two classes:

**Class A — auto-approved, enforced inside the orchestrator.**
All actions covered by a resource's declared permission set. Read-only repo access, workspace
writes, calls to allowlisted provider endpoints, subprocess launches of allowlisted binaries.
These never need a human because they are already bounded by the permission model.

**Class B — approval-required, and therefore must run inside a Hermes tool call.**
Destructive filesystem operations outside the workspace, database writes, secret *reads* beyond
the account's own credential, git push / deploy, network calls to hosts not on the allowlist.

Implementation for Class B:
1. The orchestrator, invoked as an MCP tool, detects a Class-B action.
2. It does **not** execute. It returns a structured "approval required" result with the exact
   action, target, and diff.
3. The MCP tool's `check_fn` (or the plugin's `pre_tool_call` hook) blocks and shows the user.
4. The user approves; the orchestrator re-executes with an approval token bound to that exact
   action (single-use, expiring, scoped to the action hash).

**Class-B actions are therefore unavailable to background/async runs.** This is a real constraint
and must be stated plainly, not papered over.

### 5.2 Approval tokens

```json
{
  "approval_id": "uuid",
  "action_hash": "sha256(method|target|argv|workspace)",
  "task_ref": "kanban-card-id or tool_call_id",
  "granted_by": "user",
  "expires_at": "2026-11-09T18:00:00Z",
  "single_use": true
}
```

The orchestrator refuses an approved action whose `action_hash` does not match what was approved.
This prevents "approve `rm file.txt`" from becoming "approve `rm -rf /`".

---

## 6. Threats and mitigations

| # | Threat | Impact | Mitigation | Status |
|---|---|---|---|---|
| T1 | **Secret in prompt / logs / checkpoint** | Credential leak into a model provider, a log file, or git | Registry stores refs only; resolve at call time; redact at the adapter boundary; never log env values | To build |
| T2 | **Child process reads another account's credentials** | Cross-account compromise | Per-account HOME/config dir; env allowlist per account; absolute-path deny list | To build |
| T3 | **Coding agent writes outside the workspace** | Damage to the user's real projects | Workspace pinning from Kanban; permission allowlist; Class-B approval | To build |
| T4 | **Destructive action without approval** | Data loss | Class B + single-use, action-hash-bound approval tokens | To build |
| T5 | **Approval reuse / TOCTOU** | Approved action changed before execution | Action hash re-checked at execution time; single use; expiry | To build |
| T6 | **Runaway spend** | Unexpected cost | Per-task spend cap; per-provider concurrency budget; escalation budget | **Missing from spec** |
| T7 | **Runaway concurrency** | Machine exhaustion, provider bans | Max parallel executions globally and per provider; max per workspace | **Missing from spec** |
| T8 | **Prompt injection from repo content** | Agent performs attacker-chosen actions | Permissions bound the blast radius regardless of prompt content; Class-B approval for anything irreversible | Design |
| T9 | **Untrusted third-party code (the CLI itself) is compromised** | Full compromise of the orchestrator's process | Run each CLI as a separate OS process with a scoped env and cwd; the orchestrator holds no long-lived secret it does not need | Design |
| T10 | **Duplicate execution after a retry** | Double-charging, double-writing | Idempotency key per task; dedupe on `(task_ref, idempotency_key)` | **Missing from spec** |
| T11 | **Account rotation used to evade quotas** | ToS violation, account ban | Rotate only on provider-signalled failures; per-provider concurrency budget; 402 → cooldown not rotation | Design |
| T12 | **MCP surface exposed beyond localhost** | Remote access to the orchestrator | Bind to 127.0.0.1; bearer token; if remote is ever needed, mTLS or a tunnel | Design |
| T13 | **Browser profile theft** | Stolen web sessions | Browser phase deferred; when built: fresh dedicated profiles only, never the user's real profile | Deferred |
| T14 | **Malicious repository** | Agent tricked into exfiltrating secrets | T8 + T2 combined; no real credentials in the child's env beyond its own account | Design |

---

## 7. Browser-based AI — the security case for dropping it

Independent of the ToS argument in the architecture doc, the security case is decisive:

1. **Browser profiles hold live web sessions.** A compromised profile is a compromised account
   on a third-party service, often with payment attached.
2. **CDP endpoints are unauthenticated by default.** A Chrome instance started with
   `--remote-debugging-port` accepts any local connection — any process on the machine can
   read the page, read cookies, and drive the session.
3. **CAPTCHA handling is a red flag.** `phases/phase-4-browser.md:21` lists it as a work item.
   Solving CAPTCHAs is deliberate evasion of anti-automation controls — it is both a ToS breach
   and the kind of capability that must never exist in a component with access to the user's
   other credentials.
4. **The upside is nil for the NVIDIA case.** The API key already gives direct access.

**Decision: browser-based AI is removed from the MVP.** If it ever returns: Playwright only,
fresh dedicated profiles per account (never a copy of the real profile), CDP bound to a unix
socket or a random port with a token, **no CAPTCHA solving**, and only services whose terms
permit automation.

---

## 8. Concrete checklist before Phase 1

- [ ] Registry stores `auth_ref` only; a test asserts no secret value appears in the registry file.
- [ ] Redaction filter applied in every adapter before events leave the orchestrator.
- [ ] Every child process gets a scoped env (allowlisted vars only) and a pinned cwd.
- [ ] Deny list covers `~/.ssh`, `~/.aws`, `~/.gnupg`, `~/.hermes/.env`, `~/.config/gh`.
- [ ] Class-B actions are impossible from a background run (enforced server-side, not by the client).
- [ ] Approval tokens are single-use, expiring, and action-hash-bound.
- [ ] Per-task spend cap and per-provider concurrency budget are enforced, not just logged.
- [ ] Idempotency key present on every dispatch; duplicates rejected.
- [ ] MCP server binds `127.0.0.1` only and requires a bearer token.
- [ ] SQLite file permissions `0600`; no secrets in the DB.
- [ ] Kiro multi-account is either proven with two real logins or marked single-account in the registry.
