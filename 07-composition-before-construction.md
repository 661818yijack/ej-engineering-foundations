---
format_version: 1
id: engineering.foundation.composition-before-construction
kind: principle
category: architecture-and-design
severity: should
checks:
- enforcement: human-review
  effect: advisory
  verifiability: human-judgment
references:
- engineering.foundation.deterministic-first
- engineering.foundation.risk-based-prioritization
- engineering.foundation.sufficient-quality
- engineering.foundation.evidence-driven-evaluation
---
# F07 — Composition before construction

Version 0.2 — draft principle. Part of the [foundations catalog](00-engineering-foundations.md).

## Principle

Deliver the required capability by reusing and composing existing, proven solutions wherever sufficient. Introduce custom implementation only when demonstrated limitations justify its engineering and lifetime maintenance cost.

Make the feature work first; ownership of the underlying implementation is a separate decision.

## Why

System integration is engineering. Mature operating systems, command-line tools, databases, infrastructure services and execution environments already implement many difficult mechanisms. Their value includes established interfaces, failure behavior, security updates, operational experience and maintained implementations, not merely the algorithms they contain.

Using a working tool directly can be preferable to reimplementing its algorithm or even embedding a library. Integration still requires engineering judgment: individually reliable components do not automatically make a reliable composition.

F01 concerns whether a task needs intelligence or a deterministic mechanism. F07 concerns whether the required mechanism should be borrowed, composed, adapted, extended or constructed. F05 and F06 determine how much engineering effort is worthwhile; F04 requires evidence that the chosen composition actually works.

## Apply it

Use the following investigation order as a preference, not a mandatory five-stage process. Stop once a suitable solution is established.

| Approach | Meaning |
| --- | --- |
| Reuse | Use an existing working capability directly. |
| Compose | Connect existing capabilities using their supported interfaces. |
| Configure or adapt | Tailor configuration or introduce only the necessary translation or integration logic. |
| Extend | Add a demonstrated missing capability to an existing solution where permitted and maintainable. |
| Develop | Construct a new implementation only for a justified remaining gap. |

1. Start from the required outcome and operating constraints. Group related requirements for design when their interfaces, state or failure behavior interact; keep requirements individually traceable and verifiable.
2. Investigate existing products, commands, runtimes, protocols, services and maintained implementations before selecting a new algorithm. Include what is already installed or available. Do not force a particular number of candidates.
3. Compare feasible compositions by their actual guarantees, compatibility, authority model, dependency and licensing obligations, operational burden, upgrade path and total lifecycle cost.
4. Define the integration boundary: input and output contracts, identity, configuration, state ownership, error propagation, cancellation, recovery and responsibility for enforcement as applicable. Do not assume that two good components compose safely.
5. Demonstrate the required capability using real components in a proportionate safe proof of concept. Test important interface failures and interactions as well as normal behavior. Distinguish measured behavior from documentation and assumptions.
6. Write custom glue only for an identified gap. Choose configuration, shell, Python, Rust or another suitable mechanism according to the actual need, not a mandated language or fashionable abstraction.
7. If experience later establishes a worthwhile limitation, improve or replace the dependency without unnecessarily changing the surrounding contract. Reassess trade-offs when the environment changes.

## Examples

| Required capability | Initial integration candidate | Avoid by default |
| --- | --- | --- |
| Versioned source and history | Git | A custom version-control system |
| Transactional local records | SQLite, when suitable | A new database or transaction engine |
| Process resource limits | Existing OS resource controls | A replacement operating-system scheduler |
| Tool execution isolation | Existing permissions and sandbox mechanisms | A custom security boundary without justification |
| File discovery and search | Existing commands and libraries | Rewriting basic utilities |
| Flexible intelligent work | Model and tool interfaces | Encoding every reasoning step into fixed software |

Examples illustrate the selection method; they do not mandate particular products or establish their suitability in every environment.

## Do not confuse this with

- Blindly accepting upstream claims or assuming that a tool covers every requirement.
- Bypassing authorization, license obligations, security controls or necessary safety boundaries.
- Adding dependencies merely to reduce line count. Configuration, adapters and dependency maintenance have costs too.
- Treating an integration demonstration as production qualification.
- Refusing custom development even when existing solutions are demonstrably unsuitable.
- Copying an entire architecture when a small supported capability suffices.

## Evidence and review

This principle expresses an integration and delivery engineering decision: use established capabilities first and validate their composition against real requirements. No universal claim of lower cost or better performance is made. Relevant background includes the [Unix philosophy](https://www.faqs.org/docs/artu/ch01s06.html), and documented interfaces such as [Git](https://git-scm.com/docs), [SQLite](https://sqlite.org/docs.html) and [systemd](https://www.freedesktop.org/software/systemd/man/latest/). These are examples and reference starting points, not a new literature review.

Review question: what required capability is still missing after using the smallest suitable composition, and does owning a custom implementation create enough value to justify its full cost?
