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

Version 0.1 — draft principle. Part of the [foundations catalog](00-engineering-foundations.md).

## Principle
Separate the production of a result from the evidence used to judge it whenever shared assumptions or consequential defects create meaningful risk. Design acceptance tests from the authorized specification, not from the implementation's internal logic.

## Why
A builder and its tests can share the same misunderstanding. A green test suite can check copied formulas, fake implementations, empty datasets or code paths that never execute. Independence reduces some correlated errors, but does not by itself establish test adequacy.

## Apply it
1. Establish the task's expected behavior, boundaries and applicable assurance before judging the candidate.
2. For consequential acceptance testing, provide the test author with the approved design, requirements and public interfaces. Exclude the builder's implementation and conversation during independent test authoring where practical and required.
3. Freeze the acceptance suite or record its exact version. Changes require traceable justification and revalidation, not edits solely to turn red into green.
4. Run the suite against the exact candidate in a suitable isolated environment. Collect the actual test outcomes, execution scope and revision identity.
5. Use targeted negative controls, known-good controls, mutation or fault injection when a guarantee's test adequacy matters. Check the real path, not a stand-in.
6. Recheck affected guarantees after corrections. Add white-box tests and reviews where useful; they complement rather than replace independent acceptance tests.

| Assurance need | Suitable response |
| --- | --- |
| Small disposable experiment | A direct reproducible check may suffice |
| User-facing feature | Independent acceptance criteria and representative execution |
| Resource deletion or sensitive state | Enforced boundaries, negative tests and recovery checks |
| Changed verifier | Targeted calibration against known correct and defective cases |

## Do not confuse this with
- Requiring several LLM reviewers for every change.
- Insisting on different model families as a universal rule; measure correlated errors where independence matters.
- Treating model agreement as truth.
- Running broad mutation suites on every trivial edit.
- Declaring a suite adequate merely because all tests pass.

## Evidence and review
Historical examples: M6 (20 tests green without target execution), M7 (copied formula), J5 (46 green tests while jobs built nothing), MA19 (planted defect survived frozen tests). Selected source foundations: ISTQB CTFL testing levels and techniques; Barr et al., The Oracle Problem in Software Testing.

Review question: could the check still pass if the guarantee it claims to verify were deliberately broken?
