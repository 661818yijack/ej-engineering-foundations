# M06 — Safe feasibility check

Status: collected candidate references for selecting and sequencing feasibility checks; not an implemented or approved execution-safety mechanism.

## Reference scope

- [M06-A01: Minimal VoI](minimal-voi.md): constructed ordinal VoI, EVPI, EVSI/ENBS, and one-step Beta-Binomial net VoI.
- [M06-A02: Complete VoI](complete-voi.md): multi-fidelity knowledge gradient, discrete CVaR and conditional risk comparison, bounded-horizon sequential VoI, and a selection router. Includes the supplied three-pass self-audit.

Both reference modules are preserved from the supplied attachment with LF line endings. The source reports internal self-audit passes, not external subagent reviews. No independent compilation, benchmark, mathematical validation, or experiment was performed for this collection.

## Relationship to M05

[M05 — Value of Information](../value-of-information/README.md) concerns whether additional information is worth its cost. M06 concerns obtaining sufficient feasibility evidence within authorized safety and resource boundaries. These supplied algorithms mainly optimize information acquisition, not actual safe execution. Reuse M05 techniques where suitable; do not assume that two separate VoI engines are needed.

## Selection guidance (proposed adaptation)

Apply [engineering foundations](../../00-engineering-foundations.md): 守時 (use current evidence), 節事 (least sufficient effort), 持盈 (protect valid work, resources and required guarantees).

| Situation | Candidate approach |
| --- | --- |
| Existing evidence already answers the material feasibility question | NONE; reuse the evidence. |
| No defensible probabilities are available | Qualitative FLASH triage; CVOI may be used as an ordinal heuristic, not monetary expected value. |
| A small decision model with justified probabilities and payoffs exists | Consider EVPI/EVSI or ENBS; include the no-check option. |
| Repeated observations are conditionally independent Bernoulli trials with a justified Beta prior | Consider myopic Beta-Binomial VoI for one additional check. |
| Cheap and expensive observations are demonstrably correlated | Consider multi-fidelity knowledge gradient. |
| Credible tail-loss estimates matter | Consider CVaR as additional decision evidence, not an execution authorization or guarantee. |
| Multiple checks have significant combined information value | Consider finite-horizon sequential planning with explicit resource bounds. |

Use the smallest sufficient technique or combination. The supplied router is a candidate heuristic; it does not account for all interactions between requirements and is not a universal policy.

## Safe execution boundary (proposed adaptation; not supplied by the algorithms)

Before executing any check, identify the exact claim, relevant candidate and environment, required observable result, authorized read/write scope, resource limits, isolation, and stop/recovery conditions as applicable. Prefer existing evidence and non-invasive checks. Use a bounded disposable environment when necessary; do not deliberately expose the working PC, valuable data or active sessions to credible damage or interruption. Stop when the question is answered, an applicable limit is reached, or safety is uncertain. Preserve required assurance; if it cannot be obtained safely, report the blocker rather than treating the claim as demonstrated.

## Source-inspection cautions

- The purported hard CVaR safety gate explicitly falls back to the least-risk action when every action exceeds the limit. This is not a hard gate. If a constraint is mandatory and no action satisfies it, return no feasible action/blocked; do not permit a fallback to an unsafe action. CVaR itself measures modeled tail risk, not physical or operational safety.
- The router selects a single algorithm and prioritizes missing probabilistic data before tail-safety requirements; it does not establish whether a check is authorized or safely executable. Safety boundaries must apply regardless of route.
- EVPI/EVSI are conditional on a valid probabilistic payoff model; their cost comparison must not waive required evidence or safeguards. CVOI is a heuristic score. Beta-Binomial updates assume an appropriate model for repeated observations; repeated identical deterministic tests are not independent new evidence.
- Validate dimensions, finite values, nonnegative normalized probabilities, covariance assumptions, positive costs and parameter domains. Some functions use assertions, unchecked indexing or zip truncation; zero external dependencies does not imply safe input handling.
- The complete module uses an approximate normal CDF, slope tolerances and jitter, so numerical results are not literally exact. CVaR averaging over posterior branches is a conditional risk criterion, not necessarily a globally optimal ex-ante risk-sensitive policy.
- The sequential routine explores a branching tree without memoization or a wall-clock/memory bound; depth and modeled monetary budget alone do not bound real resource usage tightly enough for arbitrary inputs. It assumes repeatable conditionally independent observations and does not enforce per-check safety constraints.
- The Rust snippets are library-style modules without a main entry point; a runnable binary needs a caller/harness. No compilation or runtime claim is independently established here.

The original code is intentionally retained as reference knowledge. The proposed selection and safety guidance is separate from that source and must be adapted and verified against the actual task before use.
