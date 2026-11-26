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

Version 0.1 — draft principle. Part of the [foundations catalog](00-engineering-foundations.md).

## Principle
Preserve the overall picture where reasoning is tightly coupled. Delegate bounded information gathering, not responsibility for understanding the complete objective.

## Why
Dividing work across agents introduces context-transfer costs and opportunities for conflicting assumptions. A faster parallel execution is not an improvement if its output is less coherent or requires expensive reconciliation.

## Apply it
- Keep the original objective, authorized scope, current subtask, relevant constraints and approved decisions visible and current.
- Use a main session for holistic design and tightly coupled implementation by default. Keep approved specifications available at their authoritative revisions.
- Delegate independently answerable research questions when the expected result can be compact: finding, measurement, source and limitation.
- Use separate main sessions when genuine independence is required, especially for test authoring. Independence and speed are different reasons for parallel work.
- Do not equate a separate conversation or directory with enforced information isolation. Restrict actual access where independence matters.
- Check existing decisions and local facts before re-asking the owner. Ask only when a consequential ambiguity cannot be resolved from available evidence.

| Task | Default arrangement |
| --- | --- |
| Explore | Main session; bounded assistance if useful |
| Research | Main session synthesizes independent findings |
| Design | Main session maintains the full picture |
| Plan | Main session coordinates dependencies |
| Build | Main session for tightly coupled code |
| Test | Independent test-author session and controlled execution |

## Do not confuse this with
- Never using subagents.
- Loading every historical document into every session.
- Sharing implementation context with a supposedly independent acceptance-test author.
- Continuing under obsolete instructions after an approved change.

## Evidence and review
Historical examples: MA03 (fragmentation and repeated vision corrections), MO159 (test and coding context concern), MA34 (stale language rule). The reported improvement from main-session coding is a qualitative observation; comparative measurement is still needed.

Review question: does the chosen execution arrangement preserve the knowledge required for correctness while isolating what must remain independent?
