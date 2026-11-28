---
format_version: 1
id: engineering.foundation.evidence-driven-evaluation
kind: principle
category: testing-and-quality
severity: should
checks:
- enforcement: human-review
  effect: advisory
  verifiability: human-judgment
references:
- engineering.foundation.independent-verification
- engineering.foundation.risk-based-prioritization
---
# F04 — Evidence-driven evaluation

Version 0.3. Part of the [foundations catalog](00-engineering-foundations.md).

## Principle

Evaluate the actual result at the level of the claim or requirement. Report demonstrated achievements, confirmed defects and unknowns separately, with enough evidence to trace each conclusion.

**Connection:** 守時 needs relevant evidence; 持盈 preserves valid achievements while directing corrections.

## Explanation and action

1. **F04.1 — Name the evaluated unit.** Identify the behavior, requirement, test, action or artifact section being judged. State what was examined and what was outside the review; a verdict about one part is not a verdict about everything.
2. **F04.2 — Label the basis of a claim.** Use observed for directly inspected results, reported for another person's account, inferred for reasoning from evidence, and unknown when the necessary evidence is missing. A reported success can inform investigation without becoming an independently measured result.
3. **F04.3 — Bind evidence to its subject.** Reference the input, candidate or document revision, relevant environment and check or source. Include a reproducible command or procedure when repeatability matters. Link to existing evidence instead of duplicating large logs or creating a new report for every small action.
4. **F04.4 — Separate findings and acceptance.** Record a confirmed defect against its location, applicable criterion and evidence. Mark a suspected defect as unresolved. Report missing required evidence and optional improvements separately; neither a suggestion nor an evaluator failure automatically proves a worker defect.
5. **F04.5 — Count what the data supports.** Use event counts when exposure is unknown. Estimate a rate only with a relevant denominator, time window and treatment of duplicates. Keep dimensions separate: passing tests, requirement coverage, unresolved conflicts and review scope are different measurements.
6. **F04.6 — Preserve valid conclusions with their limits.** Retain achievements still supported after a change and correct the affected claims. Do not present missing evidence as proof of failure, or a limited pass as proof of all quality dimensions. Report uncertainty that changes the decision.

## Reporting examples

| Available evidence | Supported statement | Unsupported extension |
| --- | --- | --- |
| A failure-only incident collection | These failure examples occurred in the collected records. | The records establish an overall failure rate. |
| EJ reports lower prompt and return lengths from tool composition | Reported benefit worth considering. | Independently measured token savings of an invented amount. |
| Every document clause has a source mapping | The mappings cover the stated clause inventory. | The wording is unambiguous for every possible agent. |
| Targeted tests pass | The tested cases passed on the identified candidate. | All requirements and operational risks are resolved. |

The historical foundation catalog contains incident examples, but this rewrite does not newly verify their primary artifacts or infer quantitative effectiveness from them.

**Review question:** can another reader identify what is known, how it is known, and where the conclusion stops?

**Basis:** [F04 traceability](TRACEABILITY.md#f04).

