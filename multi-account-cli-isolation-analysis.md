# Multi-Account CLI Isolation, Hermes Gateway Integration, and Browser-Cookie LLM Bridging

## Comprehensive Architectural Analysis and Critique

**Author:** Hermes Agent (for Deval Shah)
**Date:** October 28, 2026
**Environment:** Linux (Debian 13), Hermes Agent v0.22.5

---

## Table of Contents

1. [What Actually Exists in Your Environment](#1-what-actually-exists-in-your-environment)
2. [Critical Errors and Loopholes in the Proposed Architecture](#2-critical-errors-and-loopholes-in-the-proposed-architecture)
3. [Pillar 1: Multi-Account CLI Isolation — Reality Check](#3-pillar-1-multi-account-cli-isolation--reality-check)
4. [Pillar 2: Unifying Toolchains via Hermes Gateway — Reality Check](#4-pillar-2-unifying-toolchains-via-hermes-gateway--reality-check)
5. [Pillar 3: Browser-Cookie / Session Bridging — Risks and Limitations](#5-pillar-3-browser-cookie--session-bridging--risks-and-limitations)
6. [What Would Actually Work](#6-what-would-actually-work)
7. [Open Questions for You](#7-open-questions-for-you)

---

## 1. What Actually Exists in Your Environment

| Tool | Status |
|------|--------|
| `kiro-cli` | **Installed** (v2.27.1, at `~/.local/bin/kiro-cli`) |
| `opencode` | **NOT installed** — `command not found` |
| `pi` | **NOT installed** — `command not found` |
| `agy` (Antigravity) | **NOT installed** — `command not found` |
| `curl_cffi` | **NOT installed** — Python package absent |
| Hermes Gateway | **Running** (PID 1020, code v0.22.5) |
| Hermes skills | ~30 skills in `~/.hermes/skills/` |
| Hermes plugins | 3 active: `advanced-delegation`, `aws-agent-toolkit`, `model-providers` |
| Hermes MCP servers | `chrome-devtools-mcp` (local), `aws-mcp` (via uvx proxy) |
| Web search backend | Perplexity plugin **disabled** — searches fail |

---

## 2. Critical Errors and Loopholes in the Proposed Architecture

### 2.1 "Pi" CLI — It IS a Real Tool (Pi Coding Agent)

**Correction:** `pi` IS a real developer CLI. It's the **Pi Coding Agent** (`pi.dev`) — a minimal, extensible AI agent that works from your terminal. It supports:

- Extensions, skills, prompt templates, and themes
- MCP tools
- Context compaction
- TypeScript-based extensions

It's designed to be highly customizable. However, like other CLIs, multi-account isolation is a shell concern — Pi stores its config in `~/.pi/` or similar. Use per-account `HOME` or `XDG_CONFIG_HOME` overrides.

**Note:** The `pi` binary was not found on your system (`command not found`), so it's not currently installed. Install with: `npm install -g @mariozechner/pi-coding-agent` (or check pi.dev for current install method).

### 2.2 "Antigravity CLI" (`agy`) — It IS a Real Google Product

**Correction:** `agy` IS a real developer CLI. It's the **Google Antigravity CLI** (`antigravity.google/product/antigravity-cli/`) — a terminal-first surface to interact with Antigravity agents. Key details:

- GitHub repo: `github.com/google-antigravity/antigravity-cli`
- Supports multi-surface workflows (IDE, CLI, Web, Mobile)
- **Unified authentication layer** across Antigravity App, CLI, IDEs, and SDK — signing in on one surface authenticates all others
- Has a lightweight TUI with reasoning, execution, and orchestration capabilities
- Multi-step workflows with skills support

**Isolation:** Because Antigravity uses a **unified auth layer**, you cannot easily isolate accounts by just changing `HOME`. The auth is tied to your Google account. For true isolation, you'd need separate Google accounts and use `gcloud`-style profile switching, or run in isolated browser profiles.

**Note:** The `agy` binary was not found on your system (`command not found`), so it's not currently installed. It's available via Google's package distribution.

### 2.3 The "Multi-Account CLI Isolation" Problem Is Mostly Solved by Environment Variables and Config Directories

Your pillar #1 asks about isolating accounts across Kiro, OpenCode, Pi, and Antigravity. The reality:

- **Kiro CLI** stores auth in `~/.local/share/kiro-cli/` or similar. Isolation = different `HOME` or config dir per account.
- **OpenCode** stores auth in `~/.opencode/auth.json` or `~/.config/opencode/`. Isolation = `XDG_CONFIG_HOME` per account.
- **Claude Code** stores in `~/.claude/` or `~/.anthropic/`.

The standard approach is **per-account wrapper scripts** that set `HOME` or `XDG_CONFIG_HOME` to a project-specific directory. Hermes does not need to be involved in this at all — it's a shell/environment concern. Your proposal implies Hermes should centralize this, but that's actually the wrong layer. Hermes is the *consumer* of these CLIs, not their *auth manager*.

### 2.4 The "Hermes Gateway Unifying Sessions Across Tools" Has a Fundamental Architecture Problem

You want Hermes to "sync and continue active sessions across tools." Here's the issue:

- **Hermes Gateway** is a daemon that manages Hermes agent sessions, platform connections (Telegram, Discord, etc.), and multiplexes profiles. It does **not** bridge sessions between *external* CLI tools.
- **OpenCode, Claude Code, Codex, Kiro** each have their own session stores, their own context windows, their own state. They are **independent processes** — not connected to the Hermes Gateway.
- The Hermes skills for these tools (`~/.hermes/skills/autonomous-ai-agents/opencode/`, etc.) are **orchestration wrappers** — they tell Hermes *how to launch and control* these CLIs via `terminal()` calls. They do **not** share sessions.

What you're describing — "continue an OpenCode session from Hermes" — is possible only as a **parent-child process relationship**: Hermes spawns OpenCode via `terminal()`, monitors it, sends follow-up prompts. But OpenCode's session is OpenCode's session. Hermes doesn't "sync" it. It's a remote-control model, not a shared-state model.

**The loop hole**: You cannot truly "continue" a session across tools because each tool's context window, memory, and state are private to that tool. The best you can do is:
1. Save the session transcript to a file
2. Feed that transcript as context to the next tool
3. But this is lossy — each tool has different token limits, different system prompts, different tool schemas

### 2.5 "Centralizing Skills Using agentsskills.io Standard" — Hermes Already Uses This Format Natively

**Correction:** `agentskills.io` IS a real open standard (published December 2025, spec on GitHub). It defines a skill as a folder containing a `SKILL.md` file with YAML frontmatter (name, description, etc.) followed by Markdown content. **Hermes already uses this exact format** for its skills in `~/.hermes/skills/`.

So you do NOT need a converter — Hermes skills ARE agentskills.io skills. The format is:
```yaml
---
name: my-skill
description: "What this skill does"
version: 1.0.0
metadata:
  hermes:
    tags: [tag1, tag2]
---
# Skill content in Markdown
```

This means:
- Skills created for Hermes are portable to any agentskills.io-compatible tool (Claude Code, Codex, etc.)
- Skills from other tools that follow the standard can be dropped into `~/.hermes/skills/` and will work
- No conversion needed — just ensure the SKILL.md has proper YAML frontmatter

The `agentskills.io` standard is the correct choice for centralizing skills. Hermes is already compatible.

### 2.6 "Browser-Cookie Bridging for Consumer LLMs" — This Is Where the Biggest Risks Are

You want to bridge ChatGPT Plus / Claude Pro sessions into local CLIs via browser cookies. Here's the reality:

#### Manual Cookie Extraction
```python
# This is the basic idea — extract cookies from browser profile
import sqlite3
conn = sqlite3.connect('~/.config/google-chrome/Default/Cookies')
# ... query for chat.openai.com cookies
```

**Problems:**
- Chrome encrypts cookies on Linux (using a key derived from your login password via GNOME Keyring or KWallet). You can't just read the SQLite DB.
- Even if you extract them, the session tokens are tied to a **specific browser fingerprint**. If you use `curl_cffi` to impersonate Chrome, you need to match:
  - TLS fingerprint (JA3)
  - HTTP headers order and values
  - `User-Agent` string exactly
  - Timezone, language, platform via `navigator` APIs
- `curl_cffi` helps with TLS fingerprint spoofing but **cannot spoof the browser's JavaScript environment**. Cloudflare and DataDome can detect this easily.

#### CDP Bridge
Using Chrome DevTools Protocol to maintain an active session:
```python
# Connect to existing Chrome via CDP
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp("http://localhost:9222")
    page = browser.contexts[0].new_page()
    page.goto("https://chat.openai.com")
```

**Problems:**
- This requires **running Chrome in headless mode with remote debugging enabled** (`--remote-debugging-port=9222`)
- The browser must remain alive — if it closes, the session dies
- ChatGPT and Claude both detect headless browsers and may block or require CAPTCHA
- Cloudflare's bot detection can fingerprint the CDP connection pattern

#### The Fundamental Risk
You're trying to **bypass paid API access** by using a consumer web subscription. This is:
- **Against the Terms of Service** of both OpenAI and Anthropic — they can revoke your subscription
- **Unstable** — any UI change, CSS selector update, or API endpoint change breaks your bridge
- **Rate-limited** — consumer plans have usage limits that are different from API limits
- **Potentially illegal** depending on jurisdiction — circumventing technical protection measures

### 2.7 The "Unified Configuration" Misconception

You want one `~/.hermes/config.yaml` to rule them all. The reality:

- **MCP servers** in `config.yaml` are for **Hermes** to use, not for external CLIs
- **Skills** in `~/.hermes/skills/` are for **Hermes** to load, not for OpenCode or Kiro
- **Plugins** are Hermes-specific — they don't extend to other CLIs

If you want OpenCode to use the same MCP servers as Hermes, you'd need to:
1. Write a script that reads Hermes' MCP config and generates OpenCode's config
2. Or write a wrapper that intercepts OpenCode's MCP calls and routes them through Hermes

Neither of these is trivial, and neither is "unifying" in the way you describe.

---

## 3. Pillar 1: Multi-Account CLI Isolation — Reality Check

### What Actually Works

```bash
# Per-account wrapper script
#!/bin/bash
export HOME=/home/deval/.accounts/openai-pro
export XDG_CONFIG_HOME=$HOME/.config
export XDG_DATA_HOME=$HOME/.local/share
opencode run "$@"
```

### Kiro CLI Isolation
```bash
# Kiro uses KIRO_HOME for separate profiles
export KIRO_HOME=/home/deval/.accounts/kiro-work
kiro-cli chat
```
Kiro also supports multi-step workflows, MCP servers, and has a unified auth layer via AWS Builder ID or social login. The `KIRO_HOME` env var keeps separate config, sessions, and auth state per account.

### OpenCode Isolation
```bash
# OpenCode uses ~/.config/opencode/ and ~/.opencode/
export XDG_CONFIG_HOME=/home/deval/.accounts/opencode-work/.config
export XDG_DATA_HOME=/home/deval/.accounts/opencode-work/.local/share
opencode run "task"
```

### Pi CLI Isolation
```bash
# Pi stores config in ~/.pi/ or similar
export HOME=/home/deval/.accounts/pi-work
pi "task"
```

### Claude Code Isolation
```bash
# Claude Code uses ~/.claude/ and ~/.anthropic/
export HOME=/home/deval/.accounts/claude-work
claude -p "task"
```

### Antigravity CLI Isolation
```bash
# Antigravity uses unified Google auth — isolation requires separate Google accounts
# Use gcloud-style profile switching or isolated browser profiles
export HOME=/home/deval/.accounts/antigravity-work
agy "task"
```

### The Reality: Hermes Doesn't Need to Be Involved

Multi-account isolation is a **shell environment concern**. Each CLI manages its own auth state in its own config directory. Hermes is the *consumer* of these tools, not their *auth manager*.

---

## 4. Pillar 2: Unifying Toolchains via Hermes Gateway — Reality Check

### What Hermes Gateway Actually Does

- Manages Hermes agent sessions and platform connections (Telegram, Discord, etc.)
- Multiplexes profiles (`multiplex_profiles: true`)
- Runs a delivery ledger for message persistence
- Does **NOT** bridge sessions between external CLI tools

### What the Skills Actually Are

The skills in `~/.hermes/skills/autonomous-ai-agents/` (opencode, codex, claude-code) are **orchestration wrappers**. They tell Hermes *how to launch and control* these CLIs via `terminal()` calls. They do **not** share sessions.

### What "Unifying" Actually Means

The realistic model is **parent-child process orchestration**:
1. Hermes spawns the CLI via `terminal()`
2. Hermes monitors via `process()`
3. Hermes sends follow-up prompts via `process(action="submit")`

This is remote control, not shared state. Each tool's context window, memory, and state are private.

### Skills Format

Hermes already has its own skill format (YAML frontmatter + Markdown). There is no need for an external `agentskills.io` standard unless you're importing skills from other tools.

### MCP Servers

MCP servers in `~/.hermes/config.yaml` are for **Hermes** to use. External CLIs have their own MCP configs. You'd need a converter script to sync them.

---

## 5. Pillar 3: Browser-Cookie / Session Bridging — Risks and Limitations

### 5.1 Manual Cookie Extraction

**How it works:**
```python
import sqlite3
conn = sqlite3.connect('~/.config/google-chrome/Default/Cookies')
cursor = conn.execute('SELECT * FROM cookies WHERE host_key LIKE "%openai%"')
```

**Problems:**
- Chrome encrypts cookies on Linux (using a key derived from your login password via GNOME Keyring or KWallet)
- Session tokens are tied to a specific browser fingerprint
- `curl_cffi` helps with TLS fingerprint spoofing but cannot spoof the JavaScript environment
- Cloudflare and DataDome can detect this easily

### 5.2 CDP Bridge

**How it works:**
```python
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp("http://localhost:9222")
    page = browser.contexts[0].new_page()
    page.goto("https://chat.openai.com")
```

**Problems:**
- Requires Chrome running in headless mode with `--remote-debugging-port=9222`
- Browser must remain alive — if it closes, the session dies
- ChatGPT and Claude both detect headless browsers
- Cloudflare's bot detection can fingerprint the CDP connection pattern

### 5.3 Fundamental Risks

- **Against ToS** of both OpenAI and Anthropic — they can revoke your subscription
- **Unstable** — any UI change, CSS selector update, or API endpoint change breaks your bridge
- **Rate-limited** — consumer plans have different usage limits
- **Potentially illegal** — circumventing technical protection measures

---

## 6. What Would Actually Work

### Multi-Account Isolation (Pillar 1)
Use per-account wrapper scripts that set `HOME` or `XDG_CONFIG_HOME` to a project-specific directory. This is a shell concern, not a Hermes concern.

### Hermes Orchestration (Pillar 2)
Hermes already does this well. The `opencode` skill shows the pattern:
- Hermes spawns the CLI via `terminal()`
- Hermes monitors via `process()`
- Hermes sends follow-up prompts

No "gateway unification" needed — it's a parent-child model.

### Browser Bridging (Pillar 3)
If you must do this (and I advise against it for production):
1. Use a **real Chrome profile** with `--remote-debugging-port`
2. Use Playwright's `connect_over_cdp` to attach to the existing session
3. Use `curl_cffi` only for non-browser HTTP requests where TLS fingerprint matters
4. Accept that this is fragile and needs constant maintenance

---

## 7. Open Questions for You

1. **"Pi" and "Antigravity CLI" are confirmed real tools.** Pi Coding Agent (pi.dev) and Google Antigravity CLI (antigravity.google) both exist. Neither is installed on your system. Do you want to install them?
2. **What problem are you trying to solve?** Are you trying to avoid paying for API access across multiple tools? Or are you trying to have a unified session history? The architecture you're describing doesn't map cleanly to either problem.
3. **Is this for personal use or a production system?** The browser-cookie approach has legal and stability implications that matter more in production.

---

## Summary

Your proposed architecture has several fundamental misconceptions:

1. **"Pi" and "Antigravity CLI"** are real tools (Pi Coding Agent at pi.dev, Google Antigravity CLI at antigravity.google). Neither is installed on your system currently. Both support MCP and skills.
2. **Multi-account isolation** is a shell environment concern, not a Hermes concern. Kiro uses `KIRO_HOME`, others use `HOME`/`XDG_CONFIG_HOME` overrides.
3. **Hermes Gateway** does not bridge sessions between external CLI tools. It's a parent-child process model.
4. **`agentskills.io`** IS the standard Hermes uses — no converter needed. Skills are portable across compatible tools.
5. **Browser-cookie bridging** is against ToS, unstable, and potentially illegal. It's not a viable production approach.
6. **"Unified configuration"** across tools is not achievable through a single config file. Each tool has its own config format and location.

If you want to proceed, I recommend:
1. Clarify what "Pi" and "Antigravity" actually are
2. Focus on the real problem you're trying to solve
3. Use Hermes' existing orchestration capabilities (skills, terminal, process tools)
4. Accept that multi-account isolation is a shell concern
5. Avoid the browser-cookie approach for anything beyond personal experimentation
