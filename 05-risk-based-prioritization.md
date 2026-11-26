---
format_version: 1
id: engineering.foundation.risk-based-prioritization
kind: principle
category: architecture-and-design
severity: should
checks:
- enforcement: human-review
  effect: advisory
  verifiability: human-judgment
references:
- engineering.foundation.sufficient-quality
- engineering.foundation.evidence-driven-evaluation
---
# F05 — Risk-based prioritization

Version 0.1 — draft principle. Part of the [foundations catalog](00-engineering-foundations.md).

## Principle
Spend engineering effort first on credible failures with meaningful consequences and worthwhile opportunities for risk reduction. Do not maximize defect count, test count, process depth or assurance without considering value.

## Why
Engineering effort is finite. Every additional review, hook, control and test has development, execution, maintenance and opportunity costs. Reducing a frequent consequential failure is generally more valuable than pursuing a speculative, negligible edge case at great expense.

## Apply it
Assess the intended environment, likely exposure, consequences, existing safeguards, reversibility, uncertainty, mitigation cost and expected effectiveness. Use qualitative categories (low, medium, high, unknown) when data is inadequate. Where justified by data and consistent units, estimate expected loss over a defined time horizon as probability times consequence; compare reduction in expected loss with the full cost of the control over the same horizon. Avoid invented probabilities and false precision.

| Risk scenario | Default treatment for discretionary work |
| --- | --- |
| Frequent and consequential | Prioritize; seek simple prevention |
| Frequent but low impact | Automate cheaply or batch corrections |
| Rare but severe or irreversible | Investigate credible scenarios; retain appropriate safeguards |
| Rare and negligible, costly to address | Normally defer with a reason |
| Unknown likelihood and serious impact | Obtain evidence or apply proportionate conservative protection |

Mandatory safety, security, authorization and agreed acceptance constraints are not discretionary trade-offs. If satisfying them is infeasible, escalate or renegotiate the task; do not silently waive them. A cheap guard may be worthwhile even for uncommon failures. A recurring minor issue may accumulate into material cost.

## Decision procedure
1. Identify the affected objective and credible failure scenario.
2. Establish consequence and exposure with available evidence; mark unknowns.
3. Identify existing protections and possible lower-cost alternatives.
4. Select prevention, detection, recovery, explicit deferral or escalation.
5. For deferred material risks, record why, who owns the decision, and the trigger for reconsideration. Do not turn every trivial preference into a risk record.

## Do not confuse this with
- Ignoring severe risks merely because they are rare.
- Treating historical incident counts as probabilities without exposure denominators.
- Automatically prioritizing P1–P4 without considering likelihood, cumulative effects and cost.
- Creating a full risk-management process for every one-line change.

## Evidence and review
Historical examples: MA147 (51 agents launched in one review), MA153 (10.8 million tokens and 10.6 hours on harness work), D11 (test data reached a real store). These observations motivate proportionality but do not quantify mitigation effectiveness. Source inspiration: ISTQB risk-based testing and NIST SSDF risk-based adoption.

Review question: what risk reduction does the proposed work buy, and is there a cheaper sufficient alternative?
