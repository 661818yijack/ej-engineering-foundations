# M02-A02 — TEW-DNN + Live Cost Deduction

Status: user-supplied reference implementation; not compiled or independently validated. Optional advanced reference, not a mandatory methodology step. Burn version and API compatibility are unverified.

```toml
[dependencies]
burn = { version = "0.22", features = ["flex", "autodiff"] }
```

```rust
use burn::module::Module;
use burn::nn::{Linear, LinearConfig, Relu};
use burn::tensor::{activation::sigmoid, Device, Tensor};

// --- Module 1: Data Analysis (TEW-DNN) ---
#[derive(Module, Debug)]
pub struct TewDnn {
    fc1: Linear,
    fc2: Linear,
    out: Linear,
    relu: Relu,
}

impl TewDnn {
    pub fn new(input_dim: usize, device: &Device) -> Self {
        Self {
            fc1: LinearConfig::new(input_dim, 32).init(device),
            fc2: LinearConfig::new(32, 16).init(device),
            out: LinearConfig::new(16, 1).init(device),
            relu: Relu::new(),
        }
    }

    pub fn forward(&self, x: Tensor<2>) -> Tensor<2> {
        let x = self.relu.forward(self.fc1.forward(x));
        let x = self.relu.forward(self.fc2.forward(x));
        sigmoid(self.out.forward(x)) // Predicted defect probability in (0, 1)
    }
}

/// Asymmetric Type-Error-Weighted Binary Cross-Entropy Loss:
/// `p` in [0.0, 1.0] controls the penalty trade-off:
///   - Higher `p`: heavier penalty on Type II errors (missed defects / false negatives)
///   - Lower `p`: heavier penalty on Type I errors (false alarms / false positives)
pub fn tew_loss(preds: Tensor<2>, targets: Tensor<2>, p: f32) -> Tensor<1> {
    let eps = 1e-7_f32;
    let preds = preds.clamp(eps, 1.0 - eps);
    let ones = Tensor::ones_like(&targets);

    // Type II term (target = 1): -p * y * ln(y_hat)
    let fn_loss = targets.clone() * preds.clone().log() * (-p);
    // Type I term (target = 0): -(1 - p) * (1 - y) * ln(1 - y_hat)
    let fp_loss = (ones.clone() - targets) * (ones - preds).log() * (-(1.0 - p));

    (fn_loss + fp_loss).mean()
}

// --- Module 2: Live Cost Function Deduction ---
#[derive(Debug, Clone, Copy)]
pub struct LiveCostParams {
    pub benefit_dd: f64, // Cost saved per caught defect (negative cost)
    pub cost_nd: f64,    // Cost per false alarm (unnecessary rework/inspection)
    pub cost_dn: f64,    // Cost per missed defect (field failure/penalty)
}

impl LiveCostParams {
    pub fn total_cost(&self, x_dd: usize, x_nd: usize, x_dn: usize) -> f64 {
        -(x_dd as f64 * self.benefit_dd)
            + (x_nd as f64 * self.cost_nd)
            + (x_dn as f64 * self.cost_dn)
    }
}

// --- Module 3: Cost Analysis & Robustness Selector ---
pub struct CandidateEval {
    pub weight_p: f32,
    pub mean_cost: f64,
    pub std_cost: f64,
    pub worst_case_cost: f64,
}

/// Selects the optimal model weight `p*` using risk-adjusted cost:
/// Score = Mean_Cost + risk_aversion * Std_Cost
pub fn select_optimal_p(candidates: &[CandidateEval], risk_aversion: f64) -> Option<f32> {
    candidates
        .iter()
        .min_by(|a, b| {
            let score_a = a.mean_cost + risk_aversion * a.std_cost;
            let score_b = b.mean_cost + risk_aversion * b.std_cost;
            score_a.total_cmp(&score_b)
        })
        .map(|best| best.weight_p)
}

fn main() {
    let device = Device::flex().autodiff();
    let model = TewDnn::new(4, &device);

    // Dummy batch: 8 samples, 4 sensor features
    let x = Tensor::<2>::ones([8, 4], &device);
    let y = Tensor::<2>::from_floats(
        [[1.0], [0.0], [1.0], [0.0], [0.0], [1.0], [0.0], [0.0]],
        &device,
    );

    // Forward + asymmetric TEW backward pass (p = 0.8 prioritizes catching defects)
    let preds = model.forward(x);
    let loss = tew_loss(preds, y, 0.8);
    let _grads = loss.backward();

    // Live supply-chain cost evaluation
    let live_costs = LiveCostParams {
        benefit_dd: 120.0,
        cost_nd: 35.0,
        cost_dn: 950.0,
    };
    let batch_cost = live_costs.total_cost(3, 1, 0);
    println!("Live Batch Cost: ${:.2}", batch_cost);
}
```

## Adaptation notes

The three modules can be considered independently. Live cost deduction and risk-adjusted comparison do not inherently need a neural network. The example only demonstrates a backward pass, not a complete training, validation or deployment pipeline. The selector does not use worst_case_cost. Use predictive methods only when a relevant prediction problem, adequate data, validation and expected economic benefit justify the effort. End-to-end investigation may take hours or days; this is not a measured runtime.
