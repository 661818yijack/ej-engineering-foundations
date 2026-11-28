---
format_version: 1
id: engineering.foundation.context-continuity
kind: principle
category: architecture-and-design
severity: should
checks:
- enforcement: human-review
  effect: advisory
  verifiability: human-judgment
references:
- engineering.foundation.independent-verification
- engineering.foundation.sufficient-quality
---
# F02 — Context continuity

Version 0.3. Part of the [foundations catalog](00-engineering-foundations.md).

## Principle

Keep responsibility for the complete objective in the main session. Delegate bounded work when its result can be integrated without losing the decisions needed for correctness.

**Connection:** 持盈 preserves shared understanding; 守時 updates it when the authorized direction changes.

## Explanation and action

1. **F02.1 — Keep the goal visible.** Maintain the main goal, current task, scope, constraints and accepted decisions. This prevents independently reasonable subtasks from pulling the project in incompatible directions.
2. **F02.2 — Use one authoritative version.** Refer to the current design or requirement and identify its revision when differences could affect the work. Mark superseded decisions and update their dependents; do not load all history by default.
3. **F02.3 — Keep coupled reasoning together.** Use the main session for overall design and tightly coupled implementation. Parallelize separable work only when the interface, required context and expected return are clear enough to reconcile.
4. **F02.4 — Bound delegated work.** Give each assistant a specific question, relevant sources, authorized actions and expected output. Require findings, evidence and limitations. The main session reconciles the result against the full objective.
5. **F02.5 — Separate review from proven independence.** Another conversation can review a draft, but it does not prove independent test authorship. When independence is required, control the test author's actual access to implementation and builder discussion as described in F03.
6. **F02.6 — Reuse decisions and authority.** Check supplied context and available project facts before asking the owner. Ask only for a missing decision that materially changes the work; preserve authorization already given.

## Examples

| Work | Arrangement |
| --- | --- |
| Choose a coherent design | Main session integrates intent, constraints and evidence. |
| Check one source or count records | Bounded delegated investigation with a compact return. |
| Review this document | A reviewer can see the document; the review is not a behavioral experiment. |
| Author independent acceptance tests | Use specification and public interfaces with the required access separation. |

**Review question:** who maintains the complete intent, and does each delegated result preserve the context needed to use it?

**Basis:** [F02 traceability](TRACEABILITY.md#f02).

