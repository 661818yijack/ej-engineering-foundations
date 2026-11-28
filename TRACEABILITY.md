# Foundation traceability

Version 0.3. This record explains the basis of this rewrite. It is not a new runtime policy or a requirement to create a traceability file for every task.

## Source register

| Source | What it establishes | Limit |
| --- | --- | --- |
| U01 | EJ's discussion: establish coherent intent before WHY, WHAT and HOW; avoid individually reasonable plans pulling in different directions. | An owner preference and design requirement, not a measured effectiveness claim. |
| U02 | EJ's discussion: focus discretionary work on recurring, evidence-supported problems, actual failure cost and the minimum needed intervention. | The discussion does not supply exposure denominators or comparative failure rates. |
| U03 | EJ's discussion: a solo engineer needs to consider what can be omitted and the full burden of a solution. | No universal monetary threshold or optimization formula was agreed. |
| U04 | EJ's discussion: existing open-source tools can be reused, composed or adapted; useful Rust integration reportedly reduced coding-agent prompt and return lengths. | Reported experience; the tools and savings were not independently measured for this rewrite. |
| U05 | EJ's discussion: technological change can justify a different model, algorithm or smaller harness while preserving the purpose. | The possible future harness reduction is a scenario, not a forecast supported by measurements. |
| U06 | EJ's discussion: evidence collection should be repeatable without risking the working PC, WSL2, data or sessions. | A constraint on investigation; no destructive reproduction is required to establish it. |
| U07 | EJ's rewrite request: use the three selected ideas, explain each principle in 3–6 points, add useful diagrams, remove unrelated detail, review with subagents for three rounds, and measure ambiguity, conflict and traceability. | This specifies the present documentation task, not a universal three-review rule. |
| H01 | [國語·越語下](https://zh.wikisource.org/wiki/國語/卷21), first section: speeches attributed to 范蠡 discuss 持盈、定傾、節事 and 守時. | Historical inspiration only. 守時、節事、持盈 is this project's selection and engineering adaptation. |

U01–U07 summarize the owner discussion supplied to this rewrite, recorded on 2026-11-28. They are identified as conversation requirements because no public transcript is linked. The short summaries are the basis used here; they are not presented as independently audited research.

The following sources are immutable files from the repository baseline at commit `9d9eca3d8453cc57158170a905791a2efef9c0ae`:

| Source | Baseline |
| --- | --- |
| B00 | [Foundation index](https://github.com/661818yijack/ej-engineering-foundations/blob/9d9eca3d8453cc57158170a905791a2efef9c0ae/00-engineering-foundations.md) |
| B01 | [F01: deterministic first](https://github.com/661818yijack/ej-engineering-foundations/blob/9d9eca3d8453cc57158170a905791a2efef9c0ae/01-deterministic-first.md) |
| B02 | [F02: context continuity](https://github.com/661818yijack/ej-engineering-foundations/blob/9d9eca3d8453cc57158170a905791a2efef9c0ae/02-context-continuity.md) |
| B03 | [F03: independent verification](https://github.com/661818yijack/ej-engineering-foundations/blob/9d9eca3d8453cc57158170a905791a2efef9c0ae/03-independent-verification.md) |
| B04 | [F04: evidence-driven evaluation](https://github.com/661818yijack/ej-engineering-foundations/blob/9d9eca3d8453cc57158170a905791a2efef9c0ae/04-evidence-driven-evaluation.md) |
| B05 | [F05: risk-based prioritization](https://github.com/661818yijack/ej-engineering-foundations/blob/9d9eca3d8453cc57158170a905791a2efef9c0ae/05-risk-based-prioritization.md) |
| B06 | [F06: sufficient quality](https://github.com/661818yijack/ej-engineering-foundations/blob/9d9eca3d8453cc57158170a905791a2efef9c0ae/06-sufficient-quality.md) |
| B07 | [F07: composition before construction](https://github.com/661818yijack/ej-engineering-foundations/blob/9d9eca3d8453cc57158170a905791a2efef9c0ae/07-composition-before-construction.md) |

The baseline documents are sources for prior engineering decisions. Their historical incident examples remain historical reports; this rewrite does not claim to have re-examined the original traces. The old bibliography remains available in B00 rather than being presented as a new literature review.

## Clause mappings

The measured inventory is the 57 numbered explanations: 15 under S01–S03 and 42 under F01–F07. Each row below maps one explanation to the source of its rule. Mapping coverage means the inventory has references; it does not prove the rule's effectiveness or that every possible interpretation is resolved.

### S01

| Clause | Basis | Connection |
| --- | --- | --- |
| S01.1 | U01, B02 | Keep one intent when alternatives optimize different goals. |
| S01.2 | U02, B04 | Identify evidence and its limits before using it. |
| S01.3 | U04, U05, B07 | Avoid selecting the solution before establishing the need. |
| S01.4 | U05, B02, B07 | Revisit affected decisions when conditions change. |
| S01.5 | U05, B03, B06 | Keep forecasts separate from evidence for retiring a safeguard. |

### S02

| Clause | Basis | Connection |
| --- | --- | --- |
| S02.1 | U03, U04, B07 | Consider omission and reuse before justified construction. |
| S02.2 | U02, U06, B05 | Prioritize recurring problems without requiring severe damage to recur. |
| S02.3 | U03, U04, B05 | Compare the total burden, including model usage and maintenance. |
| S02.4 | U02, U06, B05, B06 | Bound investigation by the decision it can change. |
| S02.5 | U03, B06 | Stop discretionary effort without waiving required assurance. |

### S03

| Clause | Basis | Connection |
| --- | --- | --- |
| S03.1 | U06, B03, B05 | Preserve the working environment during evidence collection. |
| S03.2 | B04, B06 | Repair the affected part and retain valid achievements. |
| S03.3 | U01, B02 | Maintain intent, current decisions and existing authority. |
| S03.4 | B01, B03, B04, B07 | Bind verification to the actual property and composition. |
| S03.5 | U03, U05, B06, B07 | Retain useful guarantees rather than obsolete mechanisms. |

H01 supports the historical terms used in S01–S03, not the modern engineering actions. Their operational content is grounded in the discussion and prior foundations above.

### F01

| Clause | Basis | Connection |
| --- | --- | --- |
| F01.1 | B01 | Identify the intended property and evidence. |
| F01.2 | B01, B07 | Prefer a sufficient existing deterministic mechanism. |
| F01.3 | B01 | Use judgment for interpretation and deterministic work for specified operations. |
| F01.4 | B01, B03 | Validate the oracle, input and exercised path. |
| F01.5 | B01, B04 | Separate observed results from narrative. |
| F01.6 | B01, B05 | Justify enforcement while protecting high-consequence actions. |

### F02

| Clause | Basis | Connection |
| --- | --- | --- |
| F02.1 | U01, B02 | Keep the full objective and task context visible. |
| F02.2 | B02 | Keep authoritative decisions current without loading all history. |
| F02.3 | B02 | Retain tightly coupled reasoning and bound separable work. |
| F02.4 | B02, U07 | Delegate bounded questions with evidence and limitations. |
| F02.5 | B02, B03 | Distinguish another session from controlled independence. |
| F02.6 | B02 | Reuse known facts, decisions and authorization. |

### F03

| Clause | Basis | Connection |
| --- | --- | --- |
| F03.1 | B03 | Derive the verdict from accepted criteria. |
| F03.2 | B02, B03 | Match independence to required assurance and disclose actual isolation limits. |
| F03.3 | B03 | Identify the suite and justify expectation changes. |
| F03.4 | B03, B04 | Exercise and identify the actual candidate. |
| F03.5 | U06, B03 | Use targeted verifier controls without dangerous reproduction. |
| F03.6 | B03, B06 | Recheck affected guarantees and report evidence gaps. |

### F04

| Clause | Basis | Connection |
| --- | --- | --- |
| F04.1 | B04 | Evaluate at the relevant unit and state scope. |
| F04.2 | U04, B04 | Distinguish reports, observations, inference and unknowns. |
| F04.3 | B04 | Bind conclusions to evidence while avoiding unnecessary duplication. |
| F04.4 | B04, B06 | Separate defects, uncertainty, suggestions and acceptance. |
| F04.5 | U02, U07, B04 | Count only what the denominator and review scope support. |
| F04.6 | B04, B06 | Preserve correct work and disclose limits. |

### F05

| Clause | Basis | Connection |
| --- | --- | --- |
| F05.1 | U02, U03, B05 | Analyze the actual consequence of leaving the problem alone. |
| F05.2 | U02, U06, B04, B05 | Establish recurrence safely without invented likelihood. |
| F05.3 | U03, U04, B05 | Compare prevention, detection, recovery and omission. |
| F05.4 | U03, B05 | Compare lifetime costs with defensible units and uncertainty. |
| F05.5 | U06, B05 | Preserve required safeguards without converting speculation into scope. |
| F05.6 | B05 | Record only material deferrals and revisit triggers. |

### F06

| Clause | Basis | Connection |
| --- | --- | --- |
| F06.1 | B06 | Establish sufficient criteria using existing context. |
| F06.2 | B00, B06 | Preserve constraints, objective and required assurance together. |
| F06.3 | B03, B06 | Select assurance appropriate to the actual scope and consequence. |
| F06.4 | U03, U07, B06 | Budget review and keep optional findings non-blocking. |
| F06.5 | B04, B06 | Correct affected work without restarting every phase. |
| F06.6 | B06 | Report whether the task is complete, incomplete or blocked. |

### F07

| Clause | Basis | Connection |
| --- | --- | --- |
| F07.1 | U01, B07 | Begin with traceable required outcomes. |
| F07.2 | U04, B07 | Consider available capabilities without a forced survey. |
| F07.3 | U03, B07 | Compare real ownership costs and obligations. |
| F07.4 | B07 | Define the integration boundaries that matter. |
| F07.5 | U06, B03, B07 | Verify the actual composition proportionately and safely. |
| F07.6 | U04, U05, B07 | Build the justified gap and revisit it when warranted. |

## Shared material and migration

| Material | Basis or disposition |
| --- | --- |
| Intent and INTENT/WHY/WHAT/HOW table | U01–U06; B00 objective and authority; B07 outcome before solution. |
| Definitions, precedence and unknown/conflict handling | B00, B02, B05, B06; U07 asks for explicit agent behavior. |
| Diagrams | Summaries of the adjacent clauses and shared decision rules; they add no separate requirements. |
| Completion states | Operational wording of B06's completion, missing-assurance and blocked outcomes. |
| Worked examples | Illustrations of stated rules. The tool-composition experience is reported; the Git recovery case is conditional on what is actually copied. |
| Existing F01–F07 identifiers and paths | Preserved for existing consumers. F06's shortened title keeps the same sufficient-quality meaning. |
| Old P1–P4 proposal | Removed from current guidance: it was a proposed defect taxonomy, not required for these decisions. Defects retain their criterion, consequence and evidence under F04/F05. |
| Old MAESTRO/RAIL component mapping | Removed from the general catalog: it was a project-specific example and is not required to apply the foundations. |
| Historical incident lists and background bibliography | Preserved at the immutable baseline links; not restated as newly verified evidence. |
| Review record | U07; applies to this rewrite only. |

There is one governing layer and one application layer. Existing principles have been clarified and connected to the selected intent; they have not been replaced by historical quotations.
