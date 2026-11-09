# Architecture

## 1. System Context

```
┌─────────────────────────────────────────────────────────────────┐
│  USER                                                            │
│  Prompts, instructions, approval requests                        │
└──────────────┬──────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│  HERMES (primary interface)  -- DO NOT MODIFY                   │
│  Conversations · Memory · Skills · Session history · Native tools│
│         │                                                       │
│         │ Delegates task execution to the orchestrator           │
└─────────┼───────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────┐
│  AI ORCHESTRATOR (this project)                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
│  │ System 1    │  │Resource Reg. │  │Session Sync          │    │
│  │ Router      │  │(registry)    │  │(Hermes continuity)   │    │
│  └──────┬──────┘  └──────┬───────┘  └──────────┬───────────┘    │
│         │                │                     │                │
│  ┌──────┴────────────────┴─────────────────────┴───────────┐   │
│  │              Execution Manager + Verification             │   │
│  └──────┬────────────────┬──────────────┬──────────────────┘   │
│         │               │              │                       │
│  ┌──────┴───┐  ┌───────┴──────┐  ┌────┴──────────────────┐    │
│  │LLM APIs  │  │Coding CLIs  │  │Browser AI services    │    │
│  │NIM,OR,Oll│  │Kiro,Op,Pi,  │  │Chrome profiles +     │    │
│  │          │  │Cline,Claude │  │authenticated web AI  │    │
│  └──────────┘  └─────────────┘  └───────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

Hermes stays the primary interface. The Orchestrator is the intelligence and
resource-management engine behind it. Hermes is never replaced.

## 2. Component Model

| Component | Responsibility | State |
|-----------|---------------|-------|
| System 1 Router | Classify task, pick smallest capable resource, escalate on failure | Stateless, calls Resource Registry |
| Resource Registry | Register/discover resources, track auth + quotas + availability | Persistent |
| Account Manager | Per-account auth, isolation, limits, usage counters | Persistent |
| Task Scheduler | Queue, parallelize, checkpoint tasks | Persistent |
| Execution Manager | Dispatch to adapters, collect logs/artifacts/errors | Ephemeral per task |
| Session Sync | Preserve Hermes conversation continuity across tasks | Persistent |
| Verification Engine | Re-check outputs, flag mistakes, trigger rework | Stateless |
| Checkpoint & Recovery | Save state, resume after failure | Persistent |

## 3. Key Design Rules

1. **Hermes is read-only.** The orchestrator never modifies Hermes config, skills, memory, or plugins.
2. **Accounts are isolated.** Each CLI account gets its own HOME/XDG_CONFIG_HOME/KIRO_HOME directory.
3. **Resources are adapters.** Every LLM, CLI, and browser service is a thin adapter behind one interface.
4. **Failure escalates.** If a small model can't solve it, the router retries with a stronger resource.
5. **Everything is checkpointed.** Long tasks save state so they can resume.

## 4. Resource Types

| Type | Examples | Auth |
|------|----------|------|
| API providers | NVIDIA NIM, OpenRouter, Ollama | API key |
| Local models | Ollama | local |
| Coding CLIs | Kiro, OpenCode, Pi, Cline, Claude Code | per-account config dir |
| CLI accounts | multiple per tool | HOME/XDG override |
| Browser profiles | multiple Chrome profiles | cookie/session |
| Browser AI | authenticated web AI services | CDP session |

## 5. Data Flow (one task)

1. Hermes sends a task to the Orchestrator.
2. Router classifies complexity and required capabilities.
3. Registry finds available matching resources.
4. Scheduler picks one and dispatches via the right adapter.
5. Execution Manager runs it, captures logs + artifacts.
6. Verification Engine checks the output.
7. If insufficient -> escalate to a stronger resource (back to step 2).
8. Session Sync records the full history into the Hermes conversation.
