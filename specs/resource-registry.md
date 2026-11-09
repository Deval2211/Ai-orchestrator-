# Resource Registry

Every LLM, CLI, account, browser profile, and browser AI service is one
registered resource. This is the single source of truth for what exists.

## Resource Object

```yaml
id: "nvidia-nim-kimi"                # unique
type: "llm_api"                      # llm_api | local_model | coding_cli | cli_account | browser_profile | browser_ai
provider: "nvidia_nim"
model: "moonshotai/kimi-k3.1-pro"
endpoint: "https://api.nvidia.com/v1/chat/completions"
auth:
  method: "api_key"
  ref: "env:NVIDIA_API_KEY"          # never store secrets in registry
capabilities:
  - "reasoning"
  - "code_generation"
  - "agentic"
limits:
  rpm: 40
  tpm: null
  daily: null
state:
  available: true
  quota_remaining: 100
  last_used: "2026-11-05T13:50:00Z"
metadata:
  cost_tier: "free"
  latency_tier: "medium"
```

## Registration

Resources register themselves on startup (config file) or dynamically when
detected (installed CLIs, browser profiles on disk).

## Discovery

The Router queries the registry by:
- Required capabilities
- Available (state.available == true)
- Quota remaining > 0
- Cost/latency preference

## Account Isolation

Each CLI account is an independent resource:

| CLI | Isolation mechanism | Config dir |
|-----|---------------------|------------|
| Kiro | `KIRO_HOME` | `~/.kiro/` |
| OpenCode | `XDG_CONFIG_HOME` + `XDG_DATA_HOME` | `~/.config/opencode/` |
| Pi | `HOME` or `XDG_CONFIG_HOME` | `~/.pi/` |
| Cline | per-account VS Code settings | `~/.vscode/` |
| Claude Code | `HOME` | `~/.claude/` |

Per-account wrapper scripts set the env vars. The orchestrator generates
these wrappers from the registry config.

## Browser Profiles

Each browser profile is a resource with:
- Path to Chrome user-data-dir
- Authenticated session state
- CDP endpoint (if remote debugging enabled)
- Isolation: separate process + profile per account
