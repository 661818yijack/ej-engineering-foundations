---
format_version: 1
id: engineering.foundation.deterministic-first
kind: principle
category: correctness-and-safety
severity: should
checks:
- enforcement: human-review
  effect: advisory
  verifiability: human-judgment
references:
- engineering.foundation.evidence-driven-evaluation
- engineering.foundation.risk-based-prioritization
---
# F01 — Deterministic first

Version 0.1 — draft principle. Part of the [foundations catalog](00-engineering-foundations.md).

## Principle
Use the simplest reliable deterministic mechanism for work whose inputs, rules and outcomes can be specified and observed. Use AI where interpretation, ambiguity, synthesis or judgment actually requires intelligence.

## Why
An LLM should not have to remember to generate a timestamp, count records, check a Git revision or verify a command's exit status. Those tasks have cheaper and more repeatable mechanisms. But deterministic execution is not a guarantee of correctness: an incorrect rule, incomplete oracle, stale input or faulty implementation can produce a repeatable wrong result.

## Apply it
1. Identify the decision or operation and the evidence needed to establish it.
2. If an existing tool or small deterministic function can solve it correctly, reuse that mechanism.
3. If the goal or interpretation is unclear, use the model to reason about the problem; validate its proposed check against the intended behavior before relying on it.
4. Keep trustworthy state outside the model's narrative. Record actual observations, not merely intentions or success claims.
5. Start with clear methodology instructions for low-risk behavior. Add validation or enforcement when recurring failures, economics or safety justify it. For irreversible or high-consequence actions, enforce the necessary safety boundary from the beginning.

| Situation | Preferred approach |
| --- | --- |
| Record a timestamp | Read the clock programmatically |
| Verify the target revision | Compare Git identities |
| Count items in a structured dataset | Compute from the data |
| Interpret an ambiguous product goal | Model reasoning, with human input when necessary |
| Choose a technically suitable architecture | Evidence-grounded reasoning and trade-offs |

## Do not confuse this with
- Replacing all reasoning with scripts.
- Writing a custom tool where a reliable existing command suffices.
- Adding a hook merely because a model once made a harmless slip.
- Trusting deterministic results without checking the oracle or input identity.

## Evidence and review
Historical examples: M1 (claimed edit not applied), MO157 (time typed rather than read), G4 (SQL and Python paths disagree). These are examples, not estimates of failure probability.

Review question: is there a simpler reliable mechanism, and does it actually verify the promised property?
