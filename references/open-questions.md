# Open Questions

Questions that need user input before we can proceed.

## OQ1: System 1 Model
**What model runs the router?**
- Option A: Small local model (Ollama, e.g., `qwen2.5:7b` or `nemotron-mini`)
- Option B: Fast API model (NVIDIA NIM free tier, e.g., `nemotron-3-8b`)
- Option C: Hermes's own model (current default: `qwen/qwen3.8-27b:free`)
- Tradeoff: latency vs cost vs reliability

## OQ2: Database / Storage Architecture
**How do we persist the registry, execution records, checkpoints?**
- Option A: SQLite — simple, portable, queryable
- Option B: JSON files — easy to inspect, version, no DB engine needed
- Option C: Redis — fast, but adds dependency
- Preference: portable, inspectable, no external services

## OQ3: Orchestrator Programming Language
**What language for the orchestrator core?**
- Option A: Python — matches Hermes, existing codebase, rapid prototyping
- Option B: Rust — safety, performance, single binary
- Option C: Go — fast, simple, good concurrency
- Tradeoff: dev speed vs runtime reliability vs ecosystem

## OQ4: Final Project Name
**What do we call this?**
- AI Orchestrator (placeholder)
- Hermes Orchestrator
- Nexus
- Conductor
- Maestro
- Suggestions welcome

## OQ5: Verification Engine Scope
**How deep does verification go?**
- Syntax/lint only?
- Run tests?
- Semantic check (does output match intent)?
- Human review gate?

## OQ6: Browser AI Legal/Stability
**The NVIDIA Build playground is a legit API endpoint. What about others?**
- ChatGPT/Claude web sessions: ToS violation risk (per isolation analysis)
- Other browser AIs: case by case
- Need policy on what we bridge vs what we use via official API

---

**Next step:** User answers OQ1–OQ4 before Phase 1 implementation starts.
