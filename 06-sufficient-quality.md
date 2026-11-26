---
format_version: 1
id: engineering.foundation.sufficient-quality
kind: principle
category: architecture-and-design
severity: should
checks:
- enforcement: human-review
  effect: advisory
  verifiability: human-judgment
references:
- engineering.foundation.risk-based-prioritization
- engineering.foundation.deterministic-first
---
# F06 — Sufficient quality over maximum quality

Version 0.1 — draft principle. Part of the [foundations catalog](00-engineering-foundations.md).

## Principle
Satisfy the primary objective and applicable mandatory assurance, then stop when further discretionary engineering is not worthwhile. Prefer explicit objective precedence and sufficient thresholds to asking the model to maximize every quality dimension at once.

## Objective hierarchy
Apply the following order. Lower priorities cannot silently override higher ones.

| Order | Objective | Treatment |
| --- | --- | --- |
| 1 | Mandatory constraints | Respect authority, scope and essential safety invariants |
| 2 | Primary objective | Deliver the actual requested result |
| 3 | Required assurance | Demonstrate the task-specific acceptance criteria |
| 4 | Additional risk reduction | Address credible consequential risks proportionately |
| 5 | Optional optimization | Only when expected benefit justifies cost and authorization allows |

This is a proposed lexicographic/satisficing-inspired decision policy, not a claim that all engineering decisions can be reduced to a strict mathematical optimization problem. When mandatory constraints conflict with the objective, preserve the constraints and ask for a decision rather than silently changing the goal.

## Apply it
- Establish the intended task and applicable assurance before evaluation. Separate mandatory, conditional and optional criteria. Use selected ISTQB testing levels and NIST SSDF practices as a menu, not a universal checklist.
- A PoC should demonstrate feasibility; a production feature needs stronger assurance; an operation affecting canonical source or valuable data needs protections even if it is a prototype.
- Review only the promised scope. A genuine defect must have evidence and an applicable criterion or credible material consequence. Label optional improvements as non-blocking. A newly discovered serious risk can be surfaced separately for authorized handling.
- Preserve correct work. Correct the affected units and rerun the relevant checks; do not automatically restart the full methodology or open another broad review round.
- Budget discretionary review effort. Compare the incremental expected value with time, tokens, maintenance burden and delayed delivery. Introduce extra reviewers or controls only when justified.

## Stop rule
The task is eligible for closure when the primary objective and required criteria have evidence, required authorization and integrity checks are satisfied, and no unresolved blocking issue remains under the approved policy. Disclose residual uncertainty and deferred defects proportionately. If mandatory assurance is missing, report the task as incomplete or inconclusive; do not call it accepted merely because the review budget ran out. If discretionary work has insufficient expected value, stop it rather than expanding the scope.

## Examples
| Task | Sufficient achievement | Unnecessary default |
| --- | --- | --- |
| Disposable PoC | Reproducible demonstration in intended safe test scope | Full production qualification |
| Small CLI | Required behavior, relevant invalid-input cases and data safety | Distributed-scale optimization |
| RAIL cleanup capability | Evidence for safe preservation, authorized deletion and recovery | Unrelated architectural redesign |

## Evidence and review
Historical examples: MA22 (heavy process for one-line changes), MA01 (repeated acceptance findings outside a stable scope), M14 (22 rounds on an unreachable test). These examples establish a reason to investigate stopping policies, not optimal numeric thresholds.

Review question: if the task is already sufficiently demonstrated, what concrete value would another iteration deliver?
