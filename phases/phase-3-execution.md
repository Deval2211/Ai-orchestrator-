# Phase 3: Execution

## Goal
Build the adapters and execution engine that talks to the external resources.

## Work Items

### 3.1 API Adapters
- [ ] NVIDIA NIM adapter (REST API)
- [ ] OpenRouter adapter
- [ ] Ollama adapter
- [ ] Claude Code API adapter

### 3.2 CLI Adapters
- [ ] Kiro CLI adapter
- [ ] OpenCode adapter
- [ ] Pi CLI adapter
- [ ] Claude Code adapter
- [ ] Claude Code (Claude API)

### 3.3 Execution Manager
- [ ] Dispatch to adapters via CLI or REST
- [ ] Capture logs, stdout/stderr, artifacts
- [ ] Handle retries and timeouts
- [ ] Return structured result + artifacts

### 3.4 Verification
- [ ] Test execution (e.g., run code)
- [ ] Lint/format check
- [ ] Semantic verification (does it do what was asked?)
- [ ] Flag mistakes for rework

## Deliverable
- A working execution pipeline from task input to output + artifacts.
- Can execute a simple "hello world" Python script via Kiro CLI.

## Verification
- [ ] Task executes successfully
- [ ] Logs and artifacts are captured
- [ ] Verification engine passes test
