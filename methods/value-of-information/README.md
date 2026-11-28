# M05 — Value of Information (VoI)

Status: collected reference knowledge, not a mandatory methodology step or an approved implementation.

## Reference

[Complete Rust reference](complete-rust.md) preserves the supplied std-only implementation, its three reported adversarial review passes, synthetic demonstration and run instructions. Unlike earlier method families, this attachment provides four algorithms in one reference rather than separate minimal and advanced implementations.

| Algorithm ID | Technique | Decision supported |
| --- | --- | --- |
| M05-A01 | GP metamodel EVPPI | Estimate the expected value of perfect information about selected uncertain parameters, for the modeled two-action decision. |
| M05-A02 | Extended moment-matching EVSI | Estimate the value of a proposed sample and compare candidate sample sizes by expected net benefit of sampling. |
| M05-A03 | Correlated knowledge gradient | Estimate the one-step value of observing a candidate when beliefs about alternatives are correlated. |
| M05-A04 | Influence diagram with CARA utility | Estimate the maximum buying price of information under the modeled probabilities, payoffs and risk preference. |

## Selection and adaptation

Apply [engineering foundations](../../00-engineering-foundations.md): 守時 (use current evidence), 節事 (research only when worthwhile), 持盈 (protect resources and required guarantees).

A practical FLASH adaptation proposed for our methodology, not part of the supplied source:

1. Name the remaining uncertainty and the decision it could change.
2. If the answer would not materially change a decision, stop discretionary research.
3. If it could, compare the plausible benefit of learning with the full cost of a bounded investigation, including delay.
4. Investigate only within authorized safety and effort limits; reassess with the resulting evidence. If material required evidence is unavailable, report the blocker rather than declaring the requirement satisfied.

Use NONE when existing evidence already supports the decision. A small decision table or qualitative comparison can suffice. Use a mathematical module only if its assumptions, probabilities, payoffs, units and data can be justified and the additional accuracy is decision-relevant. Do not run all four modules automatically. Do not convert unsupported subjective estimates into apparently precise facts. No fixed time quota or runtime guarantee is established by collecting these references.

## Limitations from source inspection

These cautions do not modify the supplied code.

- The source is a synthetic example, not a validated production decision pipeline. Validate finite values, dimensions, distributions, covariance matrices, cost parameters and resource bounds before reuse.
- Dense GP kernel construction uses quadratic memory in the sample count and the Cholesky factorization uses cubic work. An in-sample fitted metamodel and a manually supplied noise ratio do not establish accurate EVPPI estimates or eliminate fitting bias. The jitter routine explicitly panics after exhausting its attempts, contrary to the supplied claim that it never panics.
- The EVSI search evaluates a fixed grid of fractional sample sizes and rounds only the selected size. Reevaluate admissible integer sample sizes and include the no-study baseline; a negative ENBS does not justify sampling. The unused mu0 input and the assumed conjugate normal model should be examined before adaptation.
- The cKG example selects the largest gross information value and prints net value after subtracting a common test cost; it does not stop when net value is nonpositive. Compare net value and include the option of no further measurement. Numerical approximation and near-equal slope handling warrant targeted tests.
- The CARA implementation assumes well-formed probability tables and payoffs. Bound the bisection bracketing loop, handle invalid and non-finite inputs, and check for numerical saturation. Its buying price is conditional on the assumed model, not a guarantee of real-world benefit.
- Preserve mandatory safety, authorization and acceptance constraints regardless of an estimated economic value.

## Evidence status

The supplied author explicitly states that no separate external subagents were used; three structured adversarial review passes are reported. The repository editor checked the attachment text and recorded source-level limitations, but has not compiled, benchmarked, independently reproduced those review passes or verified the named mathematical methods and Rust compatibility. Treat the timing and compatibility claims as unverified.
