---
format_version: 1
id: engineering.foundation.independent-verification
kind: principle
category: testing-and-quality
severity: should
checks:
- enforcement: human-review
  effect: advisory
  verifiability: human-judgment
references:
- engineering.foundation.evidence-driven-evaluation
- engineering.foundation.context-continuity
---
# F03 — Independent verification

Version 0.3. Part of the [foundations catalog](00-engineering-foundations.md).

## Principle

Judge results against the authorized requirement and actual behavior. Introduce independent test authorship or verification when shared assumptions or consequential defects make it necessary.

**Connection:** 持盈 protects demonstrated guarantees; 節事 keeps verification proportionate to the decision.

## Explanation and action

1. **F03.1 — Establish the criterion before the verdict.** Use the accepted behavior, boundaries and required assurance to decide what success means. Deriving expected results only from the candidate's implementation can repeat its mistake.
2. **F03.2 — Match independence to the need.** A direct check may suffice for a small disposable change. When the task requires independent acceptance testing, provide the specification and public interfaces while excluding the builder's implementation and discussion during test authoring. If that access separation cannot be achieved, disclose the limitation; do not claim independence.
3. **F03.3 — Identify the acceptance suite when one is used.** Freeze or record its exact version. Change an expectation only for a traceable requirement or oracle correction, and revalidate the affected tests. A failing candidate alone is not a reason to weaken its test.
4. **F03.4 — Exercise the actual candidate.** Record the candidate, relevant environment, check version and actual outcomes. Ensure the test reaches the promised path; a stand-in, empty dataset or unexecuted job cannot establish that path's behavior.
5. **F03.5 — Check the verifier when its adequacy is in question.** Use a targeted known-correct or known-defective case to establish whether a check can distinguish the required behavior. Use mutation or fault injection only when justified and safely bounded; never recreate damaging conditions on the working machine.
6. **F03.6 — Recheck affected guarantees.** After a correction, run checks for the changed behavior and its relevant interactions. Preserve still-valid evidence, distinguish unrun checks, and report missing required assurance. Broaden verification only for a concrete remaining risk or required gate.

## Selecting verification

| Decision | Evidence to seek |
| --- | --- |
| Is this disposable experiment feasible? | A safe direct demonstration of the claimed behavior. |
| Does this feature meet its accepted interface? | Representative execution against specification-derived criteria. |
| Does a consequential operation respect its boundary? | Evidence for that boundary, relevant negative cases and required recovery behavior. |
| Can this changed verifier detect the defect? | A targeted control with a known expected outcome. |

Different model families and more reviewers are options, not universal requirements. Agreement between models does not establish correctness. A documentation review does not measure how agents will behave after reading the document.

**Review question:** could this check pass while the guarantee it claims to verify is broken?

**Basis:** [F03 traceability](TRACEABILITY.md#f03).

