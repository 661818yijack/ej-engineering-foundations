# M02-A01 — Static PAF Model (Pure Rust)

Revision 2 — supplied reference module, code preserved as submitted on 2026-11-28. Optional, not a mandatory methodology step. The earlier version remains in [Git history](https://github.com/661818yijack/ej-engineering-foundations/blob/1b1a4340069e5775dff88a67155b653162f492b1/methods/failure-cost-analysis/static-paf.md).

## Reference code

```rust
use std::collections::BTreeMap;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum CostCategory {
    Prevention,
    Appraisal,
    InternalFailure,
    ExternalFailure,
}

#[derive(Debug, Clone)]
pub struct Expense {
    pub name: String,
    pub category: CostCategory,
    pub amount: f64,
}

#[derive(Debug, Clone, PartialEq)]
pub enum PafError {
    InvalidAmount { name: String, amount: f64 },
}

#[derive(Debug, Clone)]
pub struct PafReport {
    pub prevention: f64,
    pub appraisal: f64,
    pub internal_failure: f64,
    pub external_failure: f64,
    pub total_cost: f64,
    pub failure_ratio: f64,
    /// Aggregated failure drivers sorted descending by dollar impact (Pareto).
    pub pareto_drivers: Vec<(String, f64)>,
}

pub fn analyze_paf(expenses: &[Expense]) -> Result<PafReport, PafError> {
    let (mut prev, mut appr, mut int_f, mut ext_f) = (0.0, 0.0, 0.0, 0.0);
    let mut failure_totals: BTreeMap<String, f64> = BTreeMap::new();

    for e in expenses {
        if !e.amount.is_finite() || e.amount < 0.0 {
            return Err(PafError::InvalidAmount {
                name: e.name.clone(),
                amount: e.amount,
            });
        }
        match e.category {
            CostCategory::Prevention => prev += e.amount,
            CostCategory::Appraisal => appr += e.amount,
            CostCategory::InternalFailure => {
                int_f += e.amount;
                *failure_totals.entry(e.name.clone()).or_insert(0.0) += e.amount;
            }
            CostCategory::ExternalFailure => {
                ext_f += e.amount;
                *failure_totals.entry(e.name.clone()).or_insert(0.0) += e.amount;
            }
        }
    }

    let mut pareto_drivers: Vec<(String, f64)> = failure_totals.into_iter().collect();
    pareto_drivers.sort_by(|a, b| b.1.total_cmp(&a.1));

    let total_cost = prev + appr + int_f + ext_f;
    let failure_ratio = if total_cost > 0.0 {
        (int_f + ext_f) / total_cost
    } else {
        0.0
    };

    Ok(PafReport {
        prevention: prev,
        appraisal: appr,
        internal_failure: int_f,
        external_failure: ext_f,
        total_cost,
        failure_ratio,
        pareto_drivers,
    })
}

```

## Adaptation notes

The revised code aggregates failure drivers by name before ranking and rejects individual negative or non-finite expenses using a typed error. The algorithm is descriptive, not a prospective savings model. Define units, accounting window and aggregation identity before comparing costs; accumulation can still overflow with extreme finite values. The supplied note claims sub-millisecond execution, but supplies no benchmark conditions or results. This revision has not been compiled or benchmarked here.
