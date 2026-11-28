# M02-A01 — Static PAF Model (Pure Rust)

Status: user-supplied reference implementation; not compiled or independently validated. Optional, not a mandatory methodology step.

```rust
#[derive(Debug, Clone)]
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

#[derive(Debug)]
pub struct PafReport {
    pub prevention: f64,
    pub appraisal: f64,
    pub internal_failure: f64,
    pub external_failure: f64,
    pub total_cost: f64,
    pub failure_ratio: f64,
    pub top_failure_drivers: Vec<(String, f64)>,
}

pub fn analyze_paf(expenses: &[Expense]) -> PafReport {
    let (mut prev, mut appr, mut int_f, mut ext_f) = (0.0, 0.0, 0.0, 0.0);
    let mut failure_items = Vec::new();

    for e in expenses {
        match e.category {
            CostCategory::Prevention => prev += e.amount,
            CostCategory::Appraisal => appr += e.amount,
            CostCategory::InternalFailure => {
                int_f += e.amount;
                failure_items.push((e.name.clone(), e.amount));
            }
            CostCategory::ExternalFailure => {
                ext_f += e.amount;
                failure_items.push((e.name.clone(), e.amount));
            }
        }
    }

    // Sort descending by dollar impact (Pareto)
    failure_items.sort_by(|a, b| b.1.total_cmp(&a.1));

    let total_cost = prev + appr + int_f + ext_f;
    let failure_ratio = if total_cost > 0.0 {
        (int_f + ext_f) / total_cost
    } else {
        0.0
    };

    PafReport {
        prevention: prev,
        appraisal: appr,
        internal_failure: int_f,
        external_failure: ext_f,
        total_cost,
        failure_ratio,
        top_failure_drivers: failure_items,
    }
}

fn main() {
    let logs = vec![
        Expense { name: "Staff Training".into(), category: CostCategory::Prevention, amount: 2_500.0 },
        Expense { name: "QA Audits".into(), category: CostCategory::Appraisal, amount: 1_800.0 },
        Expense { name: "Batch Rework".into(), category: CostCategory::InternalFailure, amount: 6_400.0 },
        Expense { name: "SLA Penalties".into(), category: CostCategory::ExternalFailure, amount: 14_200.0 },
    ];

    let report = analyze_paf(&logs);
    println!("Total Quality Cost: ${:.2}", report.total_cost);
    println!("Failure Cost Ratio: {:.1}%", report.failure_ratio * 100.0);
    println!("Top Failure Drivers: {:?}", report.top_failure_drivers);
}
```

## Adaptation notes

Use for descriptive quality-cost analysis when categorized expenses are available. Computation is linear accumulation plus sorting of failure items; data collection may dominate effort. This does not establish prospective intervention savings. Validate amounts and units; repeated driver names are not aggregated in this reference. No measured runtime is claimed.
