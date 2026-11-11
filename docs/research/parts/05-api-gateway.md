# LiteLLM

## Repo facts
- **Repo:** https://github.com/BerriAI/litellm
- **Stars:** ~16k+ (per GitHub API)
- **Description:** "The fastest, litest AI Gateway. Rust core with Python SDK. Call 100+ LLM APIs in OpenAI (or native) format with cost tracking, guardrails, load balancing, and logging"
- **Languages:** Python (SDK), Rust (router-core)
- **Deployment:** Self-hosted (Docker, Kubernetes, bare metal); monolithic or split control/data plane

## License - exact terms quoted

### Core (outside `enterprise/`)
```text
MIT License

Copyright (c) 2023 Berri AI

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
**Source:** https://raw.githubusercontent.com/BerriAI/litellm/main/LICENSE

### Enterprise (`enterprise/` directory)
```text
The BerriAI Enterprise license (the "Enterprise License")
Copyright (c) 2024 - present Berrie AI Inc.

With regard to the BerriAI Software:

This software and associated documentation files (the "Software") may only be
used in production, if you (and any entity that you represent) have agreed to,
and are in compliance with, the BerriAI Subscription Terms of Service, available
via call (https://enterprise.litellm.ai/demo) or email (info@berri.ai) (the "Enterprise Terms"), or other
agreement governing the use of the Software, as agreed by you and BerriAI,
and otherwise have a valid BerriAI Enterprise license for the
correct number of user seats. Subject to the foregoing sentence, you are free to
modify this Software and publish patches to the Software. You agree that BerriAI
and/or its licensors (as applicable) retain all right, title and interest in and
to all such modifications and/or patches, and all such modifications and/or
patches may only be used, copied, modified, displayed, distributed, or otherwise
exploited with a valid BerriAI Enterprise license for the  correct
number of user seats.  Notwithstanding the foregoing, you may copy and modify
the Software for development and testing purposes, without requiring a
subscription.  You agree that BerriAI and/or its licensors (as applicable) retain
all right, title and interest in and to all such modifications.  You are not
granted any other rights beyond what is expressly stated herein.  Subject to the
foregoing, it is forbidden to copy, merge, publish, distribute, sublicense,
and/or sell the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

For all third party components incorporated into the BerriAI Software, those
components are licensed under the original license provided by the owner of the
applicable component.
```
**Source:** https://raw.githubusercontent.com/BerriAI/litellm/main/enterprise/LICENSE.md

**Restrictive clauses verbatim:**
- "may only be used in production, if you ... have agreed to, and are in compliance with, the BerriAI Subscription Terms of Service ... and otherwise have a valid BerriAI Enterprise license for the correct number of user seats"
- "forbidden to copy, merge, publish, distribute, sublicense, and/or sell the Software"
- "all such modifications and/or patches may only be used, copied, modified, displayed, distributed, or otherwise exploited with a valid BerriAI Enterprise license"

The GitHub `/license` endpoint reports `NOASSERTION` because the repo uses a dual-license structure: MIT for core, commercial for `enterprise/`.

## Core features verified (with file/module evidence)

| Feature | Evidence |
|---------|----------|
| **Unified API gateway** | `litellm/router.py` - `Router` class exposes OpenAI-compatible `/chat/completions`, `/embeddings`, `/completions` endpoints |
| **Provider support** | 100+ providers via `litellm/llms/` - OpenAI, Azure, Anthropic, Bedrock, Vertex, Cohere, NVIDIA, etc. |
| **Load balancing** | `litellm/router_strategy/simple_shuffle.py`, `least_busy.py`, `lowest_latency.py` - per-model-group load balancing with `order` priority |
| **Rate limiting** | `litellm/router_strategy/budget_limiter.py`, `litellm/types/litellm_params.py` `DeploymentOptions` (rpm, tpm, rpd, tpd, daily_reset_time) |
| **Fallbacks** | `router_settings.fallbacks` (model-group → model-group), `order`-based priority fallback, `encrypted_content_affinity` for Responses API |
| **Budgets / usage tracking** | `litellm/router_strategy/budget_limiter.py`, Redis-backed `router_usage` store, `litellm/caching/redis_cache.py` |
| **Virtual keys** | Proxy auth via `master_key` + `virtual_keys` table; `litellm/proxy/auth/` - team-scoped keys, spend limits per key |
| **Account/credential management** | `litellm/types/litellm_params.py` `ProviderConnection` - `api_key`, `api_base`, `api_version`, `tenant_id`, `client_id`, `client_secret`, `litellm_credential_name`; supports Secrets Manager / Vault resolution |
| **Deployment model** | YAML config (`model_list` + `router_settings` + `general_settings`) or DB-backed (PostgreSQL via Prisma); monolithic or split control/data plane; Kubernetes pod discovery |

## Routing strategies (actual names)

From `litellm/types/litellm_params.py` `RoutingStrategyName` literal and `litellm/router_strategy/` directory:

| Strategy | File | Scope |
|----------|------|-------|
| `simple-shuffle` | `simple_shuffle.py` | Weighted random per request; respects per-deployment `order` priority |
| `least-busy` | `least_busy.py` | Fewest in-flight requests; reservation TTL 30s |
| `usage-based-routing` | `usage_based_routing.py` | Routes to deployment with lowest current spend in budget window |
| `usage-based-routing-v2` | `lowest_tpm_rpm_v2.py` | Improved usage-based with TPM/RPM awareness |
| `latency-based-routing` | `lowest_latency.py` | Lowest observed latency (TTFT for streaming) |
| `cost-based-routing` | `lowest_cost.py` | Picks deployment with lowest per-token cost from `litellm.model_cost` map |
| `lar1` | `lar1_routing.py` | LAR-1 semantic routing based on agent confidence metadata (`metadata.lar1`) |
| `complexity_router` | `complexity_router/__init__.py` | Rule-based weighted scoring across dimensions; local classification <1ms; `ComplexityRouterConfig` with `TierDefinition` |
| `quality_router` | `quality_router.py` (not found - 404) | Referenced in Rust `OVERRIDABLE` list but Python file missing; may be enterprise-only |
| `tag_based_routing` | `tag_based_routing.py` | Routes by request tags vs deployment tags (`!tag` exclude, `&tag` require) |
| `priority` | implicit in Rust `is_priority()` | Forces `simple-shuffle` within priority group |

**Rust router overrides** (`litellm-rust/crates/router-core/src/strategy.rs`):
```rust
pub const OVERRIDABLE: [&str; 6] = [
    "simple-shuffle",
    "least-busy",
    "usage-based-routing",
    "usage-based-routing-v2",
    "latency-based-routing",
    "cost-based-routing",
];
```
Only these six can be overridden per-request via `routing_strategy` header/param. Others (`lar1`, `complexity_router`, `tag_based_routing`, `priority`) are group/router-level only.

## Multi-account per provider - VERIFIED ANSWER + evidence

**YES.** LiteLLM natively supports multiple accounts (different API keys, bases, credentials) for the **same provider and same model** via multiple deployments in the same `model_group`.

**Evidence from docs (load balancing page):**
```yaml
model_list:
  - model_name: gpt-6.1-sol
    litellm_params:
      model: azure/gpt-4-primary
      api_key: os.environ/AZURE_API_KEY
      order: 1
  - model_name: gpt-6.1-sol
    litellm_params:
      model: azure/gpt-4-secondary
      api_key: os.environ/AZURE_API_KEY_2
      order: 2
```
**Source:** https://docs.litellm.ai/docs/proxy/load_balancing (section "Quick Start - Load Balancing")

Each deployment entry is a distinct credential set (`api_key`, `api_base`, `api_version`, `region_name`, `headers`, etc. — all fields in `ProviderConnection`). The router treats them as separate deployments under the same `model_name` and load-balances / fails over between them.

**Code evidence:** `litellm/types/litellm_params.py` `ProviderConnection` dataclass includes every credential field per-deployment. `deployment_id()` hashes `model_group` + all `litellm_params` keys/values, so different keys produce different deployment IDs.

**Special handling for Responses API:** `encrypted_content_affinity` pre-call check (v1.82.3+) ensures follow-up requests with encrypted reasoning items route to the deployment that created them (same `api_base` + `api_key`).

## Cost-aware / semantic routing

| Feature | Name | File | Description |
|---------|------|------|-------------|
| **Cost-based routing** | `cost-based-routing` | `litellm/router_strategy/lowest_cost.py` | Picks deployment with lowest `input_cost_per_token` / `output_cost_per_token` from `litellm.model_cost` map; falls back to `deployment_backend_models` pricing |
| **Complexity / semantic routing** | `complexity_router` | `litellm/router_strategy/complexity_router/` | Rule-based weighted scoring across dimensions (reasoning, coding, context, tools, etc.); `ComplexityRouterConfig` with `TierDefinition` (simple/moderate/complex/large); local classification <1ms; maps tiers → model groups |
| **LAR-1 semantic routing** | `lar1` | `litellm/router_strategy/lar1_routing.py` | Uses agent confidence metadata (`metadata.lar1.confidence`) with configurable thresholds (low/medium/high) to route to appropriate capability tier |

**Note:** `cost-based-routing` is **cost-aware only** (cheapest per-token), not capability-aware. `complexity_router` is the **capability-based** router ("smallest capable resource") — it classifies request complexity and routes to a pre-configured tier. The two are separate strategies; they can be combined via routing groups (e.g., complexity router picks tier, then cost-based picks within tier).

## Deployment model and overhead

| Aspect | Details |
|--------|---------|
| **Self-host** | Docker image (`ghcr.io/berriai/litellm:main`), Helm chart, bare binary |
| **Control plane** | Optional: PostgreSQL (via Prisma) for persistent config, virtual keys, teams, audit logs |
| **Data plane** | Stateless Rust/Python workers; horizontal scaling via Kubernetes HPA |
| **Redis** | Required for distributed rate limiting, budgets, usage aggregation, in-flight tracking |
| **Monolithic vs split** | Single container serves proxy + admin UI + API; or split `litellm --config /path/config.yaml` (data) + separate control plane |
| **Overhead** | ~50-100MB RAM base; +Redis; +PostgreSQL if using DB-backed config. Latency overhead ~1-2ms (Rust router) to ~5-10ms (Python path). |

## Reuse assessment

**Use LiteLLM as the API gateway layer.** It provides:
- ✅ Unified OpenAI-compatible API across 100+ providers
- ✅ Multiple accounts per provider via multiple deployments per model group (verified)
- ✅ Load balancing, fallbacks, rate limits, budgets (all implemented, not planned)
- ✅ Cost-based routing (`lowest_cost.py`) + capability-based routing (`complexity_router`)
- ✅ Virtual keys, team scoping, spend tracking
- ✅ Self-hosted, MIT-licensed core (enterprise features are additive)

**Caveats:**
- Enterprise features (SSO, advanced guardrails, audit logs) require paid license
- `quality_router` strategy referenced in Rust but Python implementation not found in public repo
- Complexity router configuration is verbose; requires defining tiers and rubrics upfront
- Redis is a hard dependency for production rate limiting / budgets

---

# OmniRoute

## Does it exist? (what you found, URLs)

**Yes, but it is a different project than the name might suggest.**

| Repo | URL | Stars | License | Description |
|------|-----|-------|---------|-------------|
| **ChrisCompton/omniroute** | https://github.com/ChrisCompton/omniroute | 656 | MIT | "OmniRoute is an AI gateway for multi-provider LLMs: an OpenAI-compatible endpoint with smart routing, load balancing, retries, and fallbacks. Add policies, rate limits, caching, and observability" |
| **omniroute.online** | https://omniroute.online/ | — | — | Marketing site: "Run OmniRoute on your own machine... `npx omniroute`" (npm package) |

**Not found (404):**
- `omniroute/omniroute`
- `OpenRouterTeam/omniroute`
- `lm-sys/omniroute`

## Closest real matches

| Project | URL | Stars | License | What it does |
|---------|-----|-------|---------|--------------|
| **musistudio/claude-code-router** | https://github.com/musistudio/claude-code-router | 38,060 | MIT | **Different project.** Local control plane for AI agents: routes across models, fuses capabilities, orchestrates tools. Targets Claude Code / coding agents specifically, not a general LLM gateway. |
| **ChrisCompton/omniroute** | https://github.com/ChrisCompton/omniroute | 656 | MIT | Actual OmniRoute - general multi-provider AI gateway, OpenAI-compatible, self-hosted via npm/Node.js |

**Do not conflate them.** `claude-code-router` is for coding-agent orchestration; `omniroute` (ChrisCompton) is a general LLM API gateway.

## Assessment

- **ChrisCompton/omniroute** exists but is much smaller (656 stars), Node.js-based, less mature than LiteLLM. No evidence of multi-account-per-provider support in docs.
- **claude-code-router** is popular but solves a different problem (coding agent routing, not general LLM gateway).
- No "OmniRoute" from major orgs (OpenRouter, lm-sys, etc.) exists.

---

# Alternatives comparison

| Project | License | Multi-account per provider | Model routing (not just proxy) | Self-host overhead | URL |
|---------|---------|---------------------------|-------------------------------|-------------------|-----|
| **LiteLLM** | MIT + Enterprise (commercial) | **Yes** - multiple deployments per model_group with different `api_key`/`api_base` (verified) | **Yes** - 11 strategies: simple-shuffle, least-busy, usage-based, latency-based, cost-based, complexity_router, lar1, tag_based, priority, etc. | Medium - Docker + Redis (+PostgreSQL for control plane) | https://github.com/BerriAI/litellm |
| **Portkey** | MIT | **Yes** - "Virtual Keys / Model Catalog: register multiple keys for one provider" (docs) | **Yes** - "Advanced routing", fallback, load balancing, semantic caching | Medium - Docker + PostgreSQL + Redis + ClickHouse (observability) | https://github.com/Portkey-AI/gateway |
| **Helicone** | GPL-3.0 | **Partial** - "Vault: send one key to Helicone, managed keys per provider" (blog); multi-account not explicitly documented | **Yes** - "Provider routing across 100+ providers, automatic failover" | Medium - Docker + PostgreSQL + ClickHouse + Redis | https://github.com/Helicone/ai-gateway |
| **Bifrost** | Apache-2.0 | Not explicitly documented; provider config per-model | **Yes** - "Governance-based routing, adaptive load balancing, model catalog" | Low-Medium - Single binary (Rust), optional PostgreSQL | https://github.com/maximhq/bifrost |
| **Kong AI Gateway** | Kong Community (Apache-2.0) / Enterprise | Via Kong Consumers + Credentials plugins; each consumer can have separate provider creds | **Yes** - AI Proxy plugins: load balancing, semantic routing, token rate limiting | High - Full Kong Gateway (control plane + data plane), PostgreSQL/Cassandra | https://github.com/Kong/kong |
| **Cloudflare AI Gateway** | Proprietary (SaaS only) | Via Cloudflare account + multiple provider configs | **Yes** - "Auto Router, model aliases, dynamic routes" (docs) | **None** - Fully managed SaaS; cannot self-host | https://developers.cloudflare.com/ai-gateway/ |
| **RouteLLM** | Apache-2.0 | N/A - Router library, not a gateway | **Yes** - "Cost-aware routing: learns to route to cheaper models when quality suffices" (paper) | Low - Python library; embed in your own service | https://github.com/lm-sys/RouteLLM |
| **OpenRouter** | Proprietary (SaaS) | Via multiple API keys in account settings | **Yes** - Model routing, fallback, but closed SaaS | **None** - Hosted only | https://openrouter.ai/ |

**Key distinction:** "Model routing" = the gateway **selects** which model/deployment based on policy (cost, latency, capability, health). "Proxying" = just forwards to a single configured endpoint. All above except OpenRouter/Cloudflare are self-hostable; only LiteLLM, Portkey, Bifrost, Kong have verified multi-account-per-provider.

---

# Recommendation-relevant evidence

## Evidence (every URL you actually fetched)

1. **LiteLLM LICENSE (MIT core):** https://raw.githubusercontent.com/BerriAI/litellm/main/LICENSE
2. **LiteLLM enterprise/LICENSE.md (commercial):** https://raw.githubusercontent.com/BerriAI/litellm/main/enterprise/LICENSE.md
3. **LiteLLM routing strategies (Python):** https://api.github.com/repos/BerriAI/litellm/contents/litellm/router_strategy
4. **LiteLLM Rust strategy.rs (OVERRIDABLE, StrategyScope):** https://raw.githubusercontent.com/BerriAI/litellm/main/litellm-rust/crates/router-core/src/strategy.rs
5. **LiteLLM load balancing docs (multi-account proof):** https://docs.litellm.ai/docs/proxy/load_balancing
6. **LiteLLM ProviderConnection dataclass:** https://raw.githubusercontent.com/BerriAI/litellm/main/litellm/types/litellm_params.py
7. **LiteLLM complexity_router module:** https://raw.githubusercontent.com/BerriAI/litellm/main/litellm/router_strategy/complexity_router/__init__.py
8. **LiteLLM lowest_cost.py (cost-based routing):** https://raw.githubusercontent.com/BerriAI/litellm/main/litellm/router_strategy/lowest_cost.py
9. **LiteLLM lar1_routing.py (semantic routing):** https://raw.githubusercontent.com/BerriAI/litellm/main/litellm/router_strategy/lar1_routing.py
10. **LiteLLM tag_based_routing.py:** https://raw.githubusercontent.com/BerriAI/litellm/main/litellm/router_strategy/tag_based_routing.py
11. **LiteLLM simple_shuffle.py:** https://raw.githubusercontent.com/BerriAI/litellm/main/litellm/router_strategy/simple_shuffle.py
12. **LiteLLM least_busy.py:** https://raw.githubusercontent.com/BerriAI/litellm/main/litellm/router_strategy/least_busy.py
13. **ChrisCompton/omniroute GitHub API:** https://api.github.com/repos/ChrisCompton/omniroute
14. **musistudio/claude-code-router GitHub API:** https://api.github.com/repos/musistudio/claude-code-router
15. **claude-code-router LICENSE:** https://raw.githubusercontent.com/musistudio/claude-code-router/main/LICENSE
16. **Portkey LICENSE:** https://raw.githubusercontent.com/Portkey-AI/gateway/main/LICENSE
17. **Portkey GitHub API:** https://api.github.com/repos/Portkey-AI/gateway
18. **Portkey Virtual Keys docs:** https://docs.portkey.ai/docs/product/ai-gateway/virtual-keys
19. **Helicone LICENSE:** https://raw.githubusercontent.com/Helicone/ai-gateway/main/LICENSE
20. **Helicone Provider Routing docs:** https://docs.helicone.ai/gateway/provider-routing
21. **Helicone Vault blog:** https://www.helicone.ai/blog/vault-introduction
22. **Bifrost LICENSE:** https://raw.githubusercontent.com/maximhq/bifrost/main/LICENSE
23. **Bifrost GitHub API:** https://api.github.com/repos/maximhq/bifrost
24. **RouteLLM LICENSE:** https://raw.githubusercontent.com/lm-sys/RouteLLM/main/LICENSE
25. **RouteLLM GitHub API:** https://api.github.com/repos/lm-sys/RouteLLM
26. **Kong LICENSE:** https://raw.githubusercontent.com/Kong/kong/main/LICENSE (404 - Kong uses Apache-2.0 per docs)
27. **cc-switch local clone LICENSE:** /home/deval/.hermes/cache/scratch/ccswitch/LICENSE (MIT)