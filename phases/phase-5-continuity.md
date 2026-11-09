# Phase 5: Continuity

## Goal
Full session synchronization, context sharing, checkpoints, and recovery.

## Work Items

### 5.1 Task History Recording
- [ ] Record every task in the Hermes conversation context
- [ ] Store structured context: decisions, artifacts, open issues

### 5.2 Context Passing
- [ ] Pass relevant history + artifacts into next tool invocation
- [ ] Handle token limits (truncation strategy)
- [ ] Log what was passed and where

### 5.3 Checkpoints
- [ ] Periodic checkpoint save during long tasks
- [ ] Checkpoint schema defined (see autonomy-and-safety.md)
- [ ] Save to persistent store + Hermes session

### 5.4 Recovery
- [ ] On failure, resume from last checkpoint
- [ ] Context restored from session history
- [ ] Partial work preserved

## Deliverable
- Continue a conversation from an older task; see all history, artifacts,
  and context. Resume after failure.

## Verification
- [ ] Mid-task failure resumes from checkpoint
- [ ] New task inherits prior context
- [ ] Session length grows but stays bounded
