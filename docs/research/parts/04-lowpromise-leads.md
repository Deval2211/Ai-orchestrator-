# Low-Promise Leads — Shep, deadreckon, Rinne

Researched and dismissed. This file consolidates the findings that were previously
scattered across delegation results.

---

## Shep (`shep-ai/cli`)

**Status: unresolved identity — not evaluated.**

- Repository `https://github.com/shep-ai/cli` returns **HTTP 301 Moved Permanently**.
- The redirect target was not followed during the research window (GitHub API rate limit
  prevented `api.github.com/orgs/shep-ai/repos` lookup).
- No stars, language, license, or activity data available.
- **Decision:** treat as an unresolved lead. Ask the user which project was meant, or drop it.
- **Usable as a dependency?** No — identity not confirmed, no license data, no source inspected.

---

## deadreckon (`gregce/deadreckon`)

**Status: not usable as a dependency.**

| Field | Value |
|---|---|
| Repository | https://github.com/gregce/deadreckon |
| Language | Rust |
| Stars | 15 |
| License | `/license` endpoint returns **null** — no SPDX ID confirmed |
| Last push | 2026-08-21 |
| Workspace crates | `deadreckon`, `deadreckon-core`, `deadreckon-protocol`, `deadreckon-providers` |

**What it does (from docs research):** agent supervision, checkpointing, independent verification,
acceptance criteria, multi-CLI execution, session management.

**Why dismissed:**
1. **No confirmed license.** `/license` returns null; no LICENSE file found. Without a license there
   is no right to use, copy, modify, or distribute the code. This is a hard blocker — not a
   judgment on code quality.
2. **Very low maturity.** 15 stars, last push August 2026, small surface area.
3. **Right idea, wrong vehicle.** The checkpointing/supervision concept is relevant to our
   execution-layer design, but we should implement it ourselves rather than depend on an
   unlicensed, obscure crate.

**Study the idea, not the code.** The concepts (supervision trees, checkpoint acceptance
criteria, independent verification) are worth borrowing; the implementation is not.

---

## Rinne (`GIKSN-RESEARCH/Rinne`)

**Status: not usable — local-inference orientation + low maturity.**

| Field | Value |
|---|---|
| Repository | https://github.com/GIKSN-RESEARCH/Rinne |
| Stars | 21 |
| License | MIT (verified via API) |
| Last push | not re-verified (rate-limited) |
| Description | "Local, open-source, terminal-first AI orchestration" |

**Why dismissed:**
1. **Conflicts with the project's no-local-inference constraint.** The brief (§10) explicitly
   forbids local AI inference. Rinne's entire premise is local-first inference.
2. **Very low maturity.** 21 stars, small team, unknown maintenance cadence.
3. **Not Hermes-first.** It is its own orchestrator/CLI, not a resource-intelligence layer behind
   Hermes.

**MIT licensed** — so the code could be studied, but there is nothing in it that justifies
adopting it as a component.

---

## Summary

| Project | License | Usable? | Primary reason |
|---|---|---|---|
| Shep | unknown | No | Identity unresolved; no source inspected |
| deadreckon | none confirmed | No | No license → no right to use |
| Rinne | MIT | No | Local-inference orientation conflicts with project constraint |

**None of these three are adopted, forked, or referenced as dependencies.** They are
documented here so the research trail is complete and the next researcher does not re-open them.