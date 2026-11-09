# System 1 Router

The router is the intelligence layer. It decides **what runs where**.

## Core Principle

> Use the smallest capable resource, not simply the cheapest available resource.

## Decision Logic

```
TASK IN
  │
  ├─ 1. CLASSIFY
  │     complexity: simple / moderate / complex / large
  │     domain: code / reasoning / classification / summarization / research
  │     required: tools? files? browser? multi-step?
  │
  ├─ 2. CAPABILITY MAP
  │     simple classification  -> small, fast model
  │     summarization          -> economical model
  │     code generation        -> coding-specialized model
  │     complex planning       -> strong reasoning model
  │     large development      -> coding CLI or multiple agents
  │     difficult debugging    -> stronger model after escalation
  │
  ├─ 3. RESOURCE LOOKUP
  │     Query Resource Registry for available resources matching capabilities
  │     Check provider availability and quotas
  │
  ├─ 4. SELECT
  │     Pick the smallest capable model from the available set
  │
  ├─ 5. EXECUTE
  │     Dispatch to Execution Manager
  │
  └─ 6. VERIFY + ESCALATE
        If result insufficient -> go back to step 4 with next-stronger resource
        Repeat until acceptable or resources exhausted
```

## Escalation Ladder

| Level | Resource class | Trigger |
|-------|---------------|---------|
| 1 | Small/fast model | default for simple tasks |
| 2 | Mid-tier model | moderate complexity |
| 3 | Strong reasoning model | complex planning, debugging |
| 4 | Coding CLI | large development tasks |
| 5 | Multiple agents | task too big for one CLI |

## Inputs to the Router

- Task description
- Required capabilities (tools, files, browser, multi-step)
- Constraints (budget, time, privacy)
- Available resources from registry
- Current quotas and usage

## Outputs

- Selected resource + adapter
- Estimated cost and time
- Escalation plan if it fails
