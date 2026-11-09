# AI Orchestrator

An intelligent, extensible AI resource orchestration engine that turns **Hermes** into a unified control center for LLMs, coding agents, CLI accounts, and browser-based AI services — with intelligent task allocation and complete session continuity.

> **Status:** Specification phase. No code written yet. Read-only access to Hermes and the codebase only.
> **Location:** `/home/deval/Projects/ai-orchestrator/`
> **Related analysis:** `/home/deval/Projects/multi-account-cli-isolation-analysis.md`

---

## What This Project Is

You do **not** want another conversational AI app. You want an orchestration *engine* that sits behind Hermes:

```
You → Hermes (primary interface) → AI Orchestrator (intelligence + resource management) → LLMs / CLIs / Browser AI
```

Hermes keeps its conversations, memory, skills, and session history intact. The Orchestrator is the intelligence and resource-management layer underneath.

## One-Line Definition

> An intelligent, extensible AI resource orchestration engine that turns Hermes into a unified control center for LLMs, coding agents, CLI accounts, and browser-based AI services, with intelligent task allocation and complete session continuity.

## Core Principles

1. **Smallest capable resource, not cheapest available.** Simple classification → small model; complex planning → strong reasoning model; large dev task → coding CLI.
2. **Hermes stays primary.** Preserve Hermes skills, memory, sessions, interface, and native skill-creation process. Extend, don't replace.
3. **Unified resource pool.** Every LLM API, local model, coding CLI, CLI account, browser profile, and browser AI service is one registered resource with its own auth, limits, and state.
4. **Session continuity is non-negotiable.** Everything stays accessible from the same Hermes conversation. CLIs keep their private sessions; Hermes keeps the unified task history and structured context.
5. **Autonomy with safety.** Auto-assignment, auto-retry, auto-escalation — but approval for risky actions, isolated accounts/profiles, checkpoints, and verification before "done."

## Folder Layout

```
ai-orchestrator/
├── README.md
├── specs/           # Architecture + component design
│   ├── architecture.md
│   ├── system-1-router.md
│   ├── resource-registry.md
│   ├── session-continuity.md
│   ├── autonomy-and-safety.md
│   └── undecided.md
├── phases/          # 6-phase implementation plan
│   ├── phase-1-foundation.md
│   ├── phase-2-intelligence.md
│   ├── phase-3-execution.md
│   ├── phase-4-browser.md
│   ├── phase-5-continuity.md
│   └── phase-6-advanced.md
└── references/      # Decision log + open questions
    ├── decisions.md
    └── open-questions.md
```

## Phase Summary

| Phase | Name | Goal |
|-------|------|------|
| 1 | Foundation | Hermes integration, resource registry, account isolation, unified execution records |
| 2 | Intelligence | System 1 router, task classification, capability-based model selection |
| 3 | Execution | API adapters + Kiro, OpenCode, Pi, Cline, Claude Code |
| 4 | Browser | Chrome profile management + authenticated browser-AI adapters |
| 5 | Continuity | Session sync, context sharing, checkpoints, recovery |
| 6 | Advanced | Parallel agents, dynamic escalation, verification, optional remote workers |

## Undecided Items (to be resolved)

| Area | Status |
|------|--------|
| Hermes as primary interface | **Decided** |
| Custom orchestration engine | **Decided** |
| Task-based model allocation | **Decided** |
| Multiple API accounts | **Decided** |
| Multiple CLI integrations/accounts | **Decided** |
| Browser-based AI access | **Decided** |
| Unified Hermes session history | **Decided** |
| Preserve Hermes skills | **Decided** |
| Local-first execution | **Decided** |
| Remote workers | Future expansion |
| Exact System 1 model | To decide |
| Database and storage architecture | To decide |
| Orchestrator programming language | To decide |
| Final project name | Not decided |

See [`references/decisions.md`](references/decisions.md) for the decision log and [`references/open-questions.md`](references/open-questions.md) for unresolved items.