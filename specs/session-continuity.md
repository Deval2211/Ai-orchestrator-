# Session Continuity

Non-negotiable: everything stays accessible from the same Hermes conversation.

## The Problem

Each external CLI keeps its own private session:
- OpenCode has its own session store and context window
- Kiro has its own sessions
- Claude Code has its own sessions

These are **independent processes**. They do not share state with Hermes or
with each other. True cross-CLI session continuation is not possible without
passing history and artifacts explicitly.

## The Realistic Model: Parent-Child Orchestration

```
Hermes Main Session
│
├── Your original prompt
│
├── Task 1: Planning
│   ├── Model selection
│   ├── Prompt sent
│   └── Complete response
│
├── Task 2: Backend development
│   ├── OpenCode / Account 1
│   ├── Execution logs
│   ├── Code changes
│   └── Errors and corrections
│
├── Task 3: Frontend development
│   ├── Kiro / Account 2
│   └── Complete execution history
│
└── Task 4: Verification
    ├── Test results
    ├── Identified mistakes
    └── Fixes by another model
```

Hermes maintains the **unified, complete task history** and structured context.
The underlying CLIs retain their private execution sessions.

## What Session Sync Provides

1. **Unified history** - every task, model, prompt, output, and error is recorded
   in the Hermes conversation context.
2. **Context sharing** - relevant history and artifacts are passed into the next
   tool invocation (lossy but workable).
3. **Checkpoints** - long tasks save state so they can resume.
4. **Recovery** - on failure, resume from the last checkpoint.

## Continuation Protocol

To continue an old conversation:

1. Load the Hermes session history.
2. Extract the structured task context (decisions, artifacts, open issues).
3. Pass that context into the next tool invocation.
4. The CLI starts fresh but with the relevant history as input.

## Hermes Skills Must Remain Intact

Preserve:
- Existing Hermes skills
- Hermes's native skill creation process
- Skill updates and improvements
- Hermes's existing memory and conversation functionality
- Hermes's existing interface and workflow

External tools may receive relevant Hermes skills or compatible instructions
when necessary. We extend Hermes, we do not replace it.
