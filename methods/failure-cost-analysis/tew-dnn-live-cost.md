# M02-A02 — TEW-DNN + Live Cost Deduction & Drift Trigger

Revision 2 — supplied reference module, code preserved as submitted on 2026-11-28. Optional advanced reference, not a mandatory methodology step. The earlier version remains in [Git history](https://github.com/661818yijack/ej-engineering-foundations/blob/1b1a4340069e5775dff88a67155b653162f492b1/methods/failure-cost-analysis/tew-dnn-live-cost.md).

## Reference dependency

```toml
[dependencies]
burn = { version = "0.22.1", features = ["flex", "optim"] }

```

## Reference code

```rust
use burn::module::Module;
use burn::nn::{Linear, LinearConfig, Relu};
use burn::optim::{AdamWConfig, GradientsParams, Optimizer};
use burn::tensor::{activation::sigmoid, Device, Tensor};

#[derive(Debug, Clone, PartialEq)]
pub enum FcaError {
    InvalidWeight(f32),
    InvalidCostParams,
    EmptyCandidates,
    NoFeasibleCandidate,
}

// ============================================================================
// MODULE 1: Data Analysis — Type-Error-Weighted Deep Neural Network (TEW-DNN)
// ============================================================================

#[derive(Module, Debug)]
pub struct TewDnn {
    fc1: Linear,
    fc2: Linear,
    out: Linear,
    relu: Relu,
}

impl TewDnn {
    pub fn new(input_dim: usize, hidden_dim: usize, device: &Device) -> Self {
        Self {
            fc1: LinearConfig::new(input_dim, hidden_dim).init(device),
            fc2: LinearConfig::new(hidden_dim, hidden_dim / 2).init(device),
            out: LinearConfig::new(hidden_dim / 2, 1).init(device),
            relu: Relu::new(),
        }
    }

    /// Returns defect probabilities in (0, 1) with shape [batch, 1].
    pub fn forward(&self, x: Tensor<2>) -> Tensor<2> {
        let x = self.relu.forward(self.fc1.forward(x));
        let x = self.relu.forward(self.fc2.forward(x));
        sigmoid(self.out.forward(x))
    }
}

/// Asymmetric Type-Error-Weighted BCE Loss:
///   L_TEW(p) = -mean[ p * y * ln(y_hat) + (1 - p) * (1 - y) * ln(1 - y_hat) ]
/// Higher `p` penalizes missed defects (Type II / FN) more heavily.
pub fn tew_loss(preds: Tensor<2>, targets: Tensor<2>, p: f32) -> Result<Tensor<1>, FcaError> {
    if !(0.0 < p && p < 1.0) || !p.is_finite() {
        return Err(FcaError::InvalidWeight(p));
    }
    let eps = 1e-7_f32;
    let preds = preds.clamp(eps, 1.0 - eps);
    let ones = Tensor::ones_like(&targets);

    let fn_term = targets.clone() * preds.clone().log() * (-p);
    let fp_term = (ones.clone() - targets) * (ones - preds).log() * (-(1.0 - p));
    Ok((fn_term + fp_term).mean())
}

/// Trains a single candidate TEW-DNN for a specific error-weight `p`.
pub fn train_candidate(
    input_dim: usize,
    hidden_dim: usize,
    train_x: &Tensor<2>,
    train_y: &Tensor<2>,
    p: f32,
    epochs: usize,
    lr: f64,
    device: &Device,
) -> Result<TewDnn, FcaError> {
    if !(0.0 < p && p < 1.0) {
        return Err(FcaError::InvalidWeight(p));
    }
    let mut model = TewDnn::new(input_dim, hidden_dim, device);
    let mut optim = AdamWConfig::new().with_weight_decay(1e-4).init();

    for _ in 0..epochs {
        let preds = model.forward(train_x.clone());
        let loss = tew_loss(preds, train_y.clone(), p)?;
        let grads = loss.backward();
        let grads_params = GradientsParams::from_grads(grads, &model);
        model = optim.step(lr, model, grads_params);
    }
    Ok(model)
}

// ============================================================================
// MODULE 2: Live Cost Function Deduction
// ============================================================================

#[derive(Debug, Clone, Copy)]
pub struct LiveCostParams {
    pub benefit_dd: f64, // Cost saved per caught defect (TP)
    pub cost_nd: f64,    // Cost per false alarm (FP)
    pub cost_dn: f64,    // Cost per missed defect (FN)
}

#[derive(Debug, Clone, Copy, Default)]
pub struct ConfusionCounts {
    pub dd: usize, // True Positives  (Defect detected)
    pub nd: usize, // False Positives (False alarm)
    pub dn: usize, // False Negatives (Missed defect)
}

impl LiveCostParams {
    pub fn validate(&self) -> Result<(), FcaError> {
        let valid = [self.benefit_dd, self.cost_nd, self.cost_dn]
            .iter()
            .all(|v| v.is_finite() && *v >= 0.0);
        if valid { Ok(()) } else { Err(FcaError::InvalidCostParams) }
    }

    /// f_total = -benefit_dd * x_DD + cost_nd * x_ND + cost_dn * x_DN
    pub fn total_cost(&self, c: ConfusionCounts) -> f64 {
        -(c.dd as f64 * self.benefit_dd)
            + (c.nd as f64 * self.cost_nd)
            + (c.dn as f64 * self.cost_dn)
    }

    /// Evaluates confusion counts from probability predictions and binary targets.
    pub fn evaluate_batch(&self, probs: &[f32], targets: &[f32], threshold: f32) -> f64 {
        let mut counts = ConfusionCounts::default();
        for (&prob, &y) in probs.iter().zip(targets.iter()) {
            let pred_defect = prob >= threshold;
            let actual_defect = y >= 0.5;
            match (actual_defect, pred_defect) {
                (true, true) => counts.dd += 1,
                (false, true) => counts.nd += 1,
                (true, false) => counts.dn += 1,
                (false, false) => {}
            }
        }
        self.total_cost(counts)
    }
}

// ============================================================================
// MODULE 3: Cost Analysis, Robustness Selection & Live Drift Trigger
// ============================================================================

#[derive(Debug, Clone)]
pub struct CandidateEval {
    pub weight_p: f32,
    pub mean_cost: f64,
    pub std_cost: f64,
    pub worst_case_cost: f64,
}

impl CandidateEval {
    /// Computes mean, unbiased sample std dev, and worst-case cost across folds/batches.
    pub fn from_fold_costs(weight_p: f32, fold_costs: &[f64]) -> Result<Self, FcaError> {
        if fold_costs.is_empty() {
            return Err(FcaError::EmptyCandidates);
        }
        let n = fold_costs.len() as f64;
        let mean_cost = fold_costs.iter().sum::<f64>() / n;
        let std_cost = if fold_costs.len() > 1 {
            let var = fold_costs
                .iter()
                .map(|c| (c - mean_cost).powi(2))
                .sum::<f64>()
                / (n - 1.0);
            var.sqrt()
        } else {
            0.0
        };
        let worst_case_cost = fold_costs
            .iter()
            .copied()
            .max_by(f64::total_cmp)
            .unwrap_or(mean_cost);

        Ok(Self {
            weight_p,
            mean_cost,
            std_cost,
            worst_case_cost,
        })
    }
}

/// Selects optimal `p*` minimizing risk-adjusted cost: `mean + risk_aversion * std`,
/// subject to an optional hard worst-case budget ceiling `max_worst_case`.
pub fn select_optimal_p(
    candidates: &[CandidateEval],
    risk_aversion: f64,
    max_worst_case: Option<f64>,
) -> Result<CandidateEval, FcaError> {
    if candidates.is_empty() {
        return Err(FcaError::EmptyCandidates);
    }
    candidates
        .iter()
        .filter(|c| max_worst_case.map_or(true, |cap| c.worst_case_cost <= cap))
        .min_by(|a, b| {
            let score_a = a.mean_cost + risk_aversion * a.std_cost;
            let score_b = b.mean_cost + risk_aversion * b.std_cost;
            score_a.total_cmp(&score_b)
        })
        .cloned()
        .ok_or(FcaError::NoFeasibleCandidate)
}

/// Conditional trigger: returns `true` if any live supply-chain cost parameter
/// drifted by more than `drift_threshold` (e.g., 0.15 = 15%) from baseline.
pub fn should_reselect_model(
    baseline: LiveCostParams,
    current: LiveCostParams,
    drift_threshold: f64,
) -> bool {
    let rel_drift = |base: f64, curr: f64| -> f64 {
        (curr - base).abs() / base.abs().max(1e-9)
    };
    rel_drift(baseline.benefit_dd, current.benefit_dd) >= drift_threshold
        || rel_drift(baseline.cost_nd, current.cost_nd) >= drift_threshold
        || rel_drift(baseline.cost_dn, current.cost_dn) >= drift_threshold
}

```

## Adaptation notes

The new reference includes a training loop for one candidate, typed errors, cost-parameter validation, fold statistics, a worst-observed-cost ceiling and a relative cost-drift trigger. The live cost and risk-comparison functions can be used without a neural network. A full p-sweep, held-out evaluation pipeline and deployment are not supplied. The cost ceiling filters observed fold maxima; it does not guarantee future worst-case cost. Validate input lengths before evaluate_batch (zip silently truncates mismatched inputs), validate finite values and thresholds, and actually invoke cost validation before using the costs. The n-1 denominator estimates sample variance without bias under standard assumptions; its square root is not generally an unbiased estimator of standard deviation. Burn API compatibility and the supplied audit claims have not been independently verified here. No compilation, training or benchmark was performed.
