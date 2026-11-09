# Undecided Items

These are design decisions, not settled requirements.

| Area | Status | Notes |
|------|--------|-------|
| Hermes as primary interface | **Decided** | Hermes stays primary, orchestrator is behind it |
| Custom orchestration engine | **Decided** | Yes, separate from Hermes |
| Task-based model allocation | **Decided** | Router picks smallest capable resource |
| Multiple API accounts | **Decided** | Supported via registry |
| Multiple CLI integrations/accounts | **Decided** | Per-account isolation via env vars |
| Browser-based AI access | **Decided** | Chrome profiles + CDP |
| Unified Hermes session history | **Decided** | Parent-child model, Hermes keeps history |
| Preserve Hermes skills | **Decided** | Yes, extend not replace |
| Local-first execution | **Decided** | Prefer local/CLI before cloud APIs |
| Remote workers | Future expansion | Not in initial phases |
| Exact System 1 model | **To decide** | Which model runs the router itself? |
| Database and storage architecture | **To decide** | SQLite? JSON files? |
| Orchestrator programming language | **To decide** | Python? Rust? |
| Final project name | **Not decided** | "AI Orchestrator" is placeholder |

## Open Questions

1. **System 1 model**: Should the router itself be a small local model (Ollama),
   or a fast API model (NVIDIA NIM free tier)? Tradeoff: latency vs cost vs
   reliability.

2. **Storage**: SQLite is simple and portable, but JSON files are easier to
   inspect and version. Which fits better with Hermes's existing patterns?

3. **Language**: Python matches Hermes and the existing codebase. Rust would
   give better performance and safety but adds complexity. What's the right
   tradeoff for this project?

4. **Name**: Needs to reflect the orchestration + intelligence + Hermes
   integration. Suggestions welcome.
