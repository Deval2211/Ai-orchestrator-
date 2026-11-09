# Phase 1: Foundation

## Goal
Get the core infrastructure in place. Nothing fancy. Just working foundations.

## Work Items

### 1.1 Hermes Integration
- [ ] Connect orchestrator to Hermes via its supported interface
- [ ] Preserve all native Hermes behaviors (memory, skills, sessions)
- [ ] Do NOT modify Hermes itself

### 1.2 Resource Registry (minimal)
- [ ] Define Resource object schema
- [ ] Register initial resources:
  - NVIDIA NIM: `moonshotai/kimi-k3.1-pro` (free tier)
  - Local: Ollama (no model loaded yet)
  - Coding CLI: Kiro (installed)
- [ ] Store registry in a simple persistent format

### 1.3 Account Isolation
- [ ] Create per-account HOME/XDG dirs (e.g. `/home/deval/.accounts/`)
- [ ] Generate per-account wrapper scripts for Kiro
- [ ] Verify CLI launches under isolated env

### 1.4 Unified Execution Records
- [ ] Define task execution record schema
- [ ] Record every task: input, output, resource, status, errors, timestamp

## Deliverable
- A running orchestrator process that can register a resource and execute a
  trivial task (e.g., "reply with HELLO_KIMI_TEST") through NVIDIA NIM.
- This is the integration smoke test.

## Verification
- [ ] Orchestrator registers `moonshotai/kimi-k3.1-pro`
- [ ] Task executes, output captured
- [ ] Execution record persisted
