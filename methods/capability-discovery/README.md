# M03 — Capability discovery

Status: collected reference material; not an adopted methodology or implementation requirement.

## Reference scope

These supplied blueprints concern discovery and invocation of callable tools and agents at runtime. They are relevant to a subset of existing-capability discovery, but do not cover the broader WHY research question of identifying existing software, libraries, services, algorithms or other solutions before deciding what to build. That broader method remains open for discussion.

- [M03-A01: Blueprint A — Minimal Baseline](minimal-static-manifest.md)
- [M03-A02: Blueprint B — Complete Modular Architecture](complete-modular-discovery.md)

## Supplied dynamic selection rubric

An implementing AI agent should always start from **Blueprint A (Minimal)** and activate individual modules from **Blueprint B (Complete)** only when the requirement trigger is met:

| Project Requirement | Trigger Condition | Module to Activate |
| --- | --- | --- |
| **Small, local, trusted tools/agents** | Fewer than ~30 capabilities; known endpoints | **Blueprint A only** (Static Manifest) |
| **Large tool/agent catalog** | Greater than 30 capabilities or tight LLM context budget | **+ Active Semantic Router** (2-stage coarse-to-fine) |
| **Dynamic / unknown remote agents** | Agents join/leave dynamically across networks | **+ Federated Naming (ANS / ANP)** |
| **Untrusted / cross-org agents** | External vendors, zero-trust, or compliance needs | **+ Cryptographic Attestation & W3C DIDs** |
| **Unreliable or competing providers** | Multiple agents offer the same skill with varying quality | **+ Trust-Weighted Reputation Scoring** |
| **Multi-step / composite tasks** | No single agent can fulfill the entire request | **+ DAG Coalition Planner** |
| **Strict SLAs, budgets, or billing** | Hard latency/cost caps or audit-trail mandates | **+ Mutual Binding Contract & Fallback** |

The supplied approach is to read the rubric, use Blueprint A as the base, and selectively integrate modules from Blueprint B as requirements justify them. The two share core capability fields; Blueprint B adds optional metadata around those fields.

## Engineering adaptation

Apply [engineering foundations](../../00-engineering-foundations.md): 守時 (current capabilities), 節事 (least sufficient effort), 持盈 (preserve required guarantees and useful knowledge).

The rubric above is preserved as a reference, not an approved automatic selection policy. In particular:

- Skip implementation if existing discovery mechanisms are sufficient. A minimal custom manifest is not mandatory.
- Treat the approximately 30-capability threshold as an unvalidated heuristic, not a measured performance boundary or rule. The source does not resolve exactly 30 capabilities; select based on actual retrieval accuracy, context cost and latency.
- Activate a module only when a demonstrated requirement or credible risk justifies it, considering existing implementations and module dependencies. Do not assume that copying feature-flag blocks guarantees compatibility or a clean upgrade path.
- Preserve authorization, input validation and applicable security boundaries even in a minimal design. Never disable required safeguards for cost reasons.
- Discovery and invocation are distinct decisions. Select and adapt the references within task authority; no new runtime, networking integration or execution is authorized by this collection.

Neither example has been executed or checked against the named protocols. No standards compatibility or runtime benchmark is claimed.
