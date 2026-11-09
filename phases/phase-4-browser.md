# Phase 4: Browser

## Goal
Manage Chrome profiles and connect to browser-based AI services.

## Work Items

### 4.1 Chrome Profile Management
- [ ] Create isolated Chrome profiles on disk
- [ ] Launch Chrome with `--remote-debugging-port` per profile
- [ ] Manage profile lifecycle (start/stop/cleanup)

### 4.2 CDP Connection
- [ ] Connect to Chrome via CDP (Playwright / raw CDP)
- [ ] Maintain session while Chrome runs
- [ ] Handle disconnection/reconnection

### 4.3 Browser AI Adapters
- [ ] NVIDIA Build playground adapter (proven working)
- [ ] Generic authenticated web AI adapter pattern
- [ ] Handle CAPTCHA / verification flows

### 4.4 Session Persistence
- [ ] Save/restore browser session state
- [ ] Cookie/session isolation per profile

## Deliverable
- Orchestrator can drive the NVIDIA Build playground programmatically.
- Can run a prompt through Kimi K3.1 Pro in the browser and capture output.

## Verification
- [ ] Prompt sent, response captured
- [ ] Output matches manual playground result
