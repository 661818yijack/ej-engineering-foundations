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

Version 0.1 — draft principle. Part of the [foundations catalog](00-engineering-foundations.md).

## Principle
Evaluate the actual outcome, preserving demonstrated achievements, localized defects and uncertainty separately. A deliverable is not a single Boolean value, and a completion claim is not completion evidence.

## Why
A candidate can contain 98 valid tests and two defective tests. It can demonstrate useful functionality while still lacking integration or security evidence. A single pass/fail label obscures progress; an average percentage obscures critical defects. The acceptance decision is a separate policy applied to the quality profile.

## Apply it
- Evaluate at the appropriate unit: claim, requirement, test case, behavior, action or artifact section. Record which units were evaluated and which remain unknown.
- Bind evidence to the intended input, candidate, environment, evaluator/check version and relevant authority. Distinguish observed facts, inferences and missing evidence.
- Record positive achievements and applicable assurance: feasibility (PoC), implemented capability, component testing, component integration, system/E2E testing, external integration and acceptance. Select security, reliability, performance and recovery assurance according to actual needs; they are not mandatory stages for every task.
- Record each confirmed defect with location, mistake type, severity, affected criterion and evidence. Distinguish a suspected defect from a confirmed one. Preserve valid work and target the correction.
- Keep severity and treatment priority separate. Proposed P1–P4 defect levels: P1 critical impact; P2 major impact; P3 limited but real impact; P4 low-impact violation. Optional improvements are not defects. Evidence confidence is separate from severity.
- Use counts with meaningful denominators and describe what the measurements establish. A passing test count is not a requirements-coverage score. Do not average incompatible dimensions by default.
- An inconclusive result or an evaluator failure is not automatically a worker defect. Report what is missing and do not blindly retry the entire task.

## Illustrative evaluation
A builder submits 100 tests. Of the 100 independently examined, 98 have correct expectations, one has a P2 wrong expectation and one has a P3 non-discriminating assertion. Three additional required behaviors have no verified coverage. These are separate findings, not 95/100 or a universal quality score. The 98 valid tests should be preserved if still relevant to the corrected revision. Acceptance depends on the required assurance, remaining risk and explicit policy.

## Do not confuse this with
- Unbounded search for all possible flaws.
- Treating untested work as proven or automatically wrong.
- Using P1–P4 as the only input to treatment priority.
- Applying production assurance to a disposable PoC without a task-specific reason.
- Blocking on preferences or unapproved criteria.

## Evidence and review
Historical examples: M8 (claimed tests completed while waiting), X15 (reported benchmark value differs from source), MA55 (unstable reviewer verdicts), XB32 (green tests missed bypasses). The supplied mistake corpus is failure-only and cannot establish general success or failure probabilities.

Review question: does the report make clear what is demonstrated, what is defective, what remains uncertain, and how each conclusion was established?
