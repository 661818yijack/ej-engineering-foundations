# M04 — Alternatives comparison

Status: collected references, not a mandatory methodology or approved implementation.

## Reference scope and variants

The supplied attachment, Alternatives Comparison.txt, contains two implementations of the same pair of decision engines:

| Engine ID | Approach |
| --- | --- |
| M04-A01 | MINIMAL_MCDA: min-max normalization and weighted-sum scoring. |
| M04-A02 | COMPLETE_MCDA: non-compensatory veto gate, SPOTIS-style distances, Shannon entropy weight fallback, VIKOR compromise scoring, and seeded Monte Carlo rank sensitivity (described as SMAA-2 in the source). |

Both implementations also contain an Auto router. Choose the implementation form independently of the analysis engine.

- [JSON CLI reference](json-cli.md): Cargo manifest, Rust source and JSON invocation example; uses serde and serde_json.
- [Standalone reference](standalone-rust.md): standard-library Rust source with example execution; includes the supplied three-round audit narrative.

The supplied text is split between these two files without modifying its code or examples. The first part of the attachment is the JSON CLI; the second is the standalone reference. Line endings were normalized to LF.

## Selection and adaptation

Apply the [engineering foundations](../../00-engineering-foundations.md): 守時 (adapt to evidence and present constraints), 節事 (least sufficient effort), 持盈 (protect required guarantees and preserve useful knowledge).

Use an already justified decision without running MCDA. For a simple reversible choice, a direct comparison of credible alternatives may suffice. Use weighted-sum scoring only when scores and weights are meaningful and hard requirements are enforced separately. Consider the advanced components independently when fixed reference bounds, competing criteria, a compromise set, or sensitivity to uncertain inputs could materially affect the decision. A veto gate can be used with simple scoring; it does not require VIKOR or Monte Carlo. No requirement to run all advanced modules, use the supplied automatic routing conditions, or build a CLI is established by collecting these references.

Before ranking, establish the intended outcome, hard constraints, measurement sources, units, and decision priorities. Keep measured scores, estimates, and unknown values distinguishable. Choose an effort limit appropriate to the consequence and value of further analysis. Stop when the decision is adequately supported or report a blocker.

## Limitations from source inspection

These are adaptation cautions, not changes to the supplied algorithms.

- Minimal mode ignores veto thresholds, fixed bounds and uncertainty; explicit Minimal selection can bypass hard constraints. Filter mandatory requirements first regardless of scoring mode.
- Missing scores silently default to zero, potentially favoring an alternative on a minimized criterion or permitting it through a veto. Validate complete finite scores, IDs, weights, bounds and configuration; treat missing required evidence as unknown or blocked.
- Minimal mode reports ClearWinner even for tied scores. Deterministic ID tie-breaking is not evidence of a superior alternative.
- Auto routes all cases with vetoes or bounds, missing weights, uncertainty, or at least five alternatives to the full pipeline. These are unvalidated design heuristics, not cost-benefit rules. Monte Carlo iterations need an explicit appropriate resource budget.
- Entropy weights represent variation among the supplied scores, not stakeholder importance. They must not silently replace the accepted intent or mandatory priorities.
- Fixed bounds can make SPOTIS distances less dependent on the candidate set, but VIKOR's alternative-dependent normalization and other stages prevent a blanket rank-reversal-immunity claim for the combined engine. Clamping can also hide differences outside supplied bounds.
- Monte Carlo samples use the original survivor set without rechecking vetoes; sampled violations of mandatory constraints are not filtered. Ties in simulated Q use input order rather than the baseline S/ID rule. Report model assumptions and distinguish simulated win frequencies from real-world probabilities. The supplied Robust/Sensitive/Fragile thresholds are illustrative, not validated universal criteria.
- Minimal scores are higher-is-better; complete Q scores are lower-is-better. Do not compare these raw values across engine modes.

## Evidence status

The source reports three review rounds covering mathematical correctness, numerical stability, and performance. This repository editor has not run those reviews, compiled the code, benchmarked runtime, or independently validated the named mathematical methods. No compiler logs, original review records, or benchmark data were supplied. The code is preserved as reference knowledge; it must be checked against actual requirements before implementation or decision use.
