# M05 — Value of Information (VoI)

Revision 2 (2026-11-28). Status: collected reference knowledge, not a mandatory methodology step, compiled tool, or approved implementation.

## References

- [Minimal Rust reference](minimal-rust.md): four lightweight techniques, synthetic demonstration, and the supplied audit log for both standalone modules.
- [Complete Rust reference](complete-rust.md): revised advanced module, strategy router, demonstration, and three supplied unit tests.
- [JSON CLI workspace reference](workspace-cli.md): proposed workspace separating two std-only libraries from a thin serde/serde_json CLI. The complete library is specified by reference to the standalone source, not supplied as an additional full source file.

The two attachments are preserved as reference text. The standalone attachment is split at its complete-module heading; the workspace attachment is retained together. The [previous advanced revision](https://github.com/661818yijack/ej-engineering-foundations/blob/0baf35e8214aca6359f268da1a0cd83f40efb8b7/methods/value-of-information/complete-rust.md) remains in Git history.

## Available techniques

The existing advanced algorithm IDs remain stable. Additional IDs are assigned to the minimal techniques for reference, not as mandatory stages.

| Algorithm ID | Technique | Decision supported |
| --- | --- | --- |
| M05-A05 | Discrete EVPI | Upper bound on value of perfect information for a modeled decision. |
| M05-A06 | Discrete EVSI | Value of a noisy observation under the modeled likelihoods and payoffs. |
| M05-A07 | Constructed VoI (CVOI) | Ordinal triage by relevance, uncertainty and reducibility. |
| M05-A08 | Monte Carlo EVPI | Sample-average estimate of perfect information value from aligned action-payoff scenarios. |
| M05-A01 | GP metamodel EVPPI | Estimate expected value of perfect information about selected parameters; revised code fits arbitrary numbers of actions. |
| M05-A02 | Extended moment-matching EVSI | Estimate sample information value and net benefit over a sample-size grid; retains a two-action incremental-net-benefit model. |
| M05-A03 | Correlated knowledge gradient | Estimate one-step observation value when beliefs about alternatives are correlated. |
| M05-A04 | Influence diagram with CARA utility | Estimate maximum buying price of information given modeled probabilities, payoffs and risk preference. |

The advanced router and JSON operations expose individual techniques. They do not require executing the entire suite.

## Selection and adaptation

Apply [engineering foundations](../../00-engineering-foundations.md): 守時 (use current evidence), 節事 (research only when worthwhile), 持盈 (protect resources and required guarantees).

A practical FLASH adaptation proposed for our methodology, not part of the supplied source:

1. Name the remaining uncertainty and the decision it could change.
2. If the answer would not materially change a decision, stop discretionary research.
3. If it could, compare plausible benefit of learning with the full cost of a bounded investigation, including delay.
4. Investigate only within authorized safety and effort limits; reassess with the resulting evidence. If material required evidence is unavailable, report the blocker rather than declaring the requirement satisfied.

Use NONE when existing evidence suffices. Use qualitative triage when defensible probabilities and payoffs are unavailable, minimal arithmetic for small well-specified decisions, and individual advanced techniques only when their assumptions and decision value justify the effort. CVOI is ordinal, not monetary; the revised score includes reducibility but not investigation cost. Compare effort separately and do not invent estimates. The supplied strategy router selects only advanced methods and defaults to GP even with all flags false; it does not implement NONE/FLASH or an effort budget, and continuous_uncertainties is not consulted. A decision may require a combination of techniques, not the router's first match.

The JSON interface is a reusable integration reference. It does not establish that a CLI, three crates, or a new engine is necessary. Use [M06](../safe-feasibility-check/README.md) when deciding how to obtain feasibility evidence safely. Reuse common VoI mathematics rather than create duplicate engines, subject to validating the differing contracts.

## Limitations from source inspection

These cautions do not modify the supplied code.

- **Input contract:** M05's discrete EVSI uses likelihood[state][signal]; M06's minimal EVSI uses likelihood[observation][state]. Normalize or explicitly transpose with validated dimensions before sharing data. The revised M05 and M06 CVOI scores also have different definitions and domains; they are not interchangeable.
- Validate nonempty and rectangular matrices, complete finite values, nonnegative normalized probabilities, positive variance/scale assumptions, cost units, dimensions and actual resource limits. Empty inputs sometimes return zero, and indexing, zip truncation and nonfinite arithmetic can hide or cause failures. A structured JSON response alone does not ensure input validity.
- Dense GP construction uses quadratic memory; Cholesky factorization is cubic work per factorization. The manually supplied noise ratio and in-sample fit do not establish calibrated EVPPI estimates. Jitter still ends in an explicit panic on exhaustion. Multi-action output does not make the two-action moment-matching model multi-action.
- EVSI optimization evaluates 50 fractional grid points and rounds the selected size without recalculating the corresponding metrics. Reevaluate admissible integers, endpoints and the no-study option. The CLI provides a positive-ENBS flag, not a guarantee of profitable investigation.
- The CLI adds a positive-net-gain measurement flag for correlated KG, with common test cost. Compare differing measurement costs appropriately, validate covariances and update denominators, and handle nonfinite or invalid beliefs.
- CARA depends on valid probability tables and payoffs. Bound the price-bracketing loop and address saturation/nonfinite arithmetic before use. Its buying price is conditional on the model.
- The supplied tests target a pure-noise test, risk-neutral EVSI and one seeded Monte Carlo comparison of cKG. They do not establish general correctness or malformed-input safety. Preserve mandatory constraints and do not use economic calculations to authorize unsafe checks or waive required evidence.

## Evidence status

The author reports three internal adversarial self-review passes, not external subagents, and claims clean compilation. The current repository editor inspected the supplied sources and checked their integrity; the Rust compiler is unavailable in this environment, so neither code nor tests nor the proposed workspace was compiled or executed here. No independent benchmark, verification of named mathematical methods, or reproduction of the supplied review was performed. No Cargo project or dependencies were installed by this collection.
