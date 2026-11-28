# M02 — Failure-cost analysis

Two optional reference implementations (revision 2, 2026-11-28):
- [M02-A01: Static PAF](static-paf.md) — descriptive quality-cost calculation.
- [M02-A02: TEW-DNN and live cost deduction](tew-dnn-live-cost.md) — cost-sensitive prediction and comparison.

The [previous revision](https://github.com/661818yijack/ej-engineering-foundations/tree/1b1a4340069e5775dff88a67155b653162f492b1/methods/failure-cost-analysis) remains available in Git history.

## Supplied selection rubric

| Requirement signal | Select blueprint | Key trigger |
| --- | --- | --- |
| Accounting logs only, no ML pipeline, fast audit | V1: Static PAF | Zero dependencies, deterministic, reported sub-millisecond calculation |
| Labeled sensor/event data, asymmetric error costs, shifting prices | V2: TEW-DNN | Autodiff training, p-sweep, live drift trigger |

V1 and V2 are the source's labels for the basic and advanced blueprints, not successive versions of one algorithm. The stated timing is not independently measured. The supplied code includes single-candidate training and selection utilities, not a complete p-sweep pipeline.

## Supplied three-round audit

The attached reference reports that three structured reviews were performed. These are reported results, not reviews executed or witnessed by this repository editor.

| Round | Reported findings and changes |
| --- | --- |
| 1 — Mathematical and algorithmic correctness | Aggregate recurring failure names in a BTreeMap; clamp predicted probabilities and validate p in (0, 1); use n-1 in variance estimation and add a worst-case cost ceiling. |
| 2 — Rust safety and Burn 0.22.1 API accuracy | Reported verification of runtime Device and optimizer APIs; replace panicking unwrap calls in core logic with typed errors and use total_cmp for sorting. |
| 3 — AI agent composability | Separate model training, cost deduction, and drift/selection mechanisms to support adaptation. |

No original review artifacts, compiler output, benchmark measurements or test logs were supplied. The visible code supports inspection of the stated changes, but not a claim of independently demonstrated correctness or API compatibility. See the adaptation notes in each file for remaining limitations.

## Engineering adaptation

Apply [engineering foundations](../../00-engineering-foundations.md): 守時 (current evidence and capabilities), 節事 (least sufficient effort), 持盈 (preserve useful knowledge and required guarantees).

The references are not mandatory stages or interchangeable algorithms. Select, simplify, combine or skip techniques according to the specific intent, unresolved decision, available data, risk and authorized effort. Start with existing evidence; bound further investigation by its decision value. Static PAF may be quick when data is ready; predictive analysis can require substantial data preparation and validation. No fixed time quota or runtime benchmark is established. Preserve the supplied reference code separately from adaptation notes. Do not infer that either implementation is production-ready.
