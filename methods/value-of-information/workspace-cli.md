# M05 — JSON CLI workspace reference

Revision 2 — supplied reference material. Contains the proposed Cargo workspace, minimal library, CLI and build/invocation examples. The complete library must be assembled from the standalone advanced module as described in the supplied text; this is not a separately built project. The supplied source text follows with LF line endings; implementation and audit claims are not independently established. See [selection and adaptation notes](README.md).

---

This Cargo workspace keeps `voi-minimal` and `voi-complete` as **zero-dependency library crates** while isolating JSON serialization inside a thin `voi-cli` binary. An AI agent can pipe a single JSON request into `stdin` and read a structured JSON response from `stdout`.

## Workspace Layout

```text
voi-workspace/
├── Cargo.toml
└── crates/
    ├── voi-minimal/
    │   ├── Cargo.toml
    │   └── src/lib.rs
    ├── voi-complete/
    │   ├── Cargo.toml
    │   └── src/lib.rs
    └── voi-cli/
        ├── Cargo.toml
        └── src/main.rs

```

---

## 1. Root Workspace (`Cargo.toml`)

```toml
[workspace]
resolver = "2"
members = [
    "crates/voi-minimal",
    "crates/voi-complete",
    "crates/voi-cli",
]

[workspace.package]
edition = "2021"
version = "0.1.0"

```

---

## 2. Minimal Library Crate (`crates/voi-minimal/`)

**`crates/voi-minimal/Cargo.toml`**

```toml
[package]
name = "voi-minimal"
version.workspace = true
edition.workspace = true

```

**`crates/voi-minimal/src/lib.rs`**

```rust
/// 1. EVPI: Expected Value of Perfect Information (Upper Bound)
pub fn evpi(probs: &[f64], payoffs: &[Vec<f64>]) -> f64 {
    if probs.is_empty() || payoffs.is_empty() {
        return 0.0;
    }
    let n_states = probs.len();

    let v_prior = payoffs
        .iter()
        .map(|a_row| a_row.iter().zip(probs).map(|(u, p)| u * p).sum::<f64>())
        .fold(f64::NEG_INFINITY, f64::max);

    let v_perfect: f64 = (0..n_states)
        .map(|s| {
            let best_in_s = payoffs
                .iter()
                .map(|a_row| a_row[s])
                .fold(f64::NEG_INFINITY, f64::max);
            probs[s] * best_in_s
        })
        .sum();

    (v_perfect - v_prior).max(0.0)
}

/// 2. EVSI: Expected Value of Sample Information (Noisy Test)
pub fn evsi(probs: &[f64], payoffs: &[Vec<f64>], likelihood: &[Vec<f64>]) -> f64 {
    if probs.is_empty() || payoffs.is_empty() || likelihood.is_empty() {
        return 0.0;
    }
    let n_states = probs.len();
    let n_signals = likelihood[0].len();

    let v_prior = payoffs
        .iter()
        .map(|a_row| a_row.iter().zip(probs).map(|(u, p)| u * p).sum::<f64>())
        .fold(f64::NEG_INFINITY, f64::max);

    let v_sample: f64 = (0..n_signals)
        .map(|y| {
            payoffs
                .iter()
                .map(|a_row| {
                    (0..n_states)
                        .map(|s| probs[s] * likelihood[s][y] * a_row[s])
                        .sum::<f64>()
                })
                .fold(f64::NEG_INFINITY, f64::max)
        })
        .sum();

    (v_sample - v_prior).max(0.0)
}

/// 3. CVOI: Constructed Value of Information (Fast Heuristic Triage)
pub fn cvoi(relevance: u8, uncertainty: u8, reducibility: u8) -> u16 {
    let r = relevance.min(3) as u16;
    let u = uncertainty.min(3) as u16;
    let red = reducibility.clamp(1, 3) as u16;
    r * u * red
}

/// 4. Monte Carlo VoI (Simulation-Based EVPI)
pub fn monte_carlo_evpi(action_payoffs: &[Vec<f64>]) -> f64 {
    if action_payoffs.is_empty() || action_payoffs[0].is_empty() {
        return 0.0;
    }
    let n = action_payoffs[0].len();

    let v_prior = action_payoffs
        .iter()
        .map(|row| row.iter().sum::<f64>() / n as f64)
        .fold(f64::NEG_INFINITY, f64::max);

    let v_perfect: f64 = (0..n)
        .map(|i| {
            action_payoffs
                .iter()
                .map(|row| row[i])
                .fold(f64::NEG_INFINITY, f64::max)
        })
        .sum::<f64>()
        / n as f64;

    (v_perfect - v_prior).max(0.0)
}

```

---

## 3. Complete Library Crate (`crates/voi-complete/`)

**`crates/voi-complete/Cargo.toml`**

```toml
[package]
name = "voi-complete"
version.workspace = true
edition.workspace = true

```

**`crates/voi-complete/src/lib.rs`**
Move the library code from `voi_complete.rs` (Sections 0–5 and the `#[cfg(test)]` module, omitting `fn main()`) directly into `crates/voi-complete/src/lib.rs`. Ensure `VoIStrategy`, `VoIRequirements`, `select_strategy`, `gp_metamodel_evppi`, `EvsiOptimizationResult`, `moment_matching_evsi`, `correlated_kg_single`, `bayesian_mv_update`, and `InfluenceDiagram` are all marked `pub`.

---

## 4. JSON Stdin/Stdout CLI Crate (`crates/voi-cli/`)

**`crates/voi-cli/Cargo.toml`**

```toml
[package]
name = "voi-cli"
version.workspace = true
edition.workspace = true

[dependencies]
voi-minimal = { path = "../voi-minimal" }
voi-complete = { path = "../voi-complete" }
serde = { version = "1", features = ["derive"] }
serde_json = "1"

```

**`crates/voi-cli/src/main.rs`**

```rust
use serde::{Deserialize, Serialize};
use serde_json::json;
use std::io::{self, Read};
use std::process;
use voi_complete::{
    bayesian_mv_update, correlated_kg_single, gp_metamodel_evppi, moment_matching_evsi,
    select_strategy, InfluenceDiagram, VoIRequirements, VoIStrategy,
};

#[derive(Deserialize)]
#[serde(tag = "op", rename_all = "snake_case")]
enum ToolRequest {
    // --- Agent Strategy Router ---
    Route {
        continuous_uncertainties: bool,
        optimize_sample_size: bool,
        sequential_correlated_tests: bool,
        risk_averse_or_causal_graph: bool,
    },

    // --- Minimal Suite ---
    Evpi {
        probs: Vec<f64>,
        payoffs: Vec<Vec<f64>>,
    },
    Evsi {
        probs: Vec<f64>,
        payoffs: Vec<Vec<f64>>,
        likelihood: Vec<Vec<f64>>,
    },
    Cvoi {
        relevance: u8,
        uncertainty: u8,
        #[serde(default = "default_reducibility")]
        reducibility: u8,
    },
    MonteCarloEvpi {
        action_payoffs: Vec<Vec<f64>>,
    },

    // --- Complete Suite ---
    GpEvppi {
        phi_samples: Vec<Vec<f64>>,
        nb_matrix: Vec<Vec<f64>>,
        #[serde(default = "default_lengthscale")]
        lengthscale: f64,
        #[serde(default = "default_noise_ratio")]
        noise_ratio: f64,
    },
    MomentMatchingEvsi {
        g_hat_inb: Vec<f64>,
        total_inb_var: f64,
        tau0: f64,
        obs_sd: f64,
        profit_per_unit: f64,
        psi_sd: f64,
        fixed_cost: f64,
        cost_per_sample: f64,
        n_min: usize,
        n_max: usize,
    },
    CorrelatedKg {
        mu: Vec<f64>,
        sigma: Vec<Vec<f64>>,
        obs_var: f64,
        #[serde(default)]
        test_cost: f64,
    },
    CorrelatedKgUpdate {
        mu: Vec<f64>,
        sigma: Vec<Vec<f64>>,
        measured_index: usize,
        observed_value: f64,
        obs_var: f64,
    },
    InfluenceDiagram {
        p_state: Vec<f64>,
        p_test_given_state: Vec<Vec<f64>>,
        payoff: Vec<Vec<f64>>,
        #[serde(default)]
        gamma: f64,
    },
}

fn default_reducibility() -> u8 { 1 }
fn default_lengthscale() -> f64 { 1.5 }
fn default_noise_ratio() -> f64 { 0.25 }

#[derive(Serialize)]
struct ToolResponse {
    ok: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    result: Option<serde_json::Value>,
    #[serde(skip_serializing_if = "Option::is_none")]
    error: Option<String>,
}

fn strategy_name(s: VoIStrategy) -> &'static str {
    match s {
        VoIStrategy::GpMetamodelEvppi => "gp_evppi",
        VoIStrategy::MomentMatchingEvsi => "moment_matching_evsi",
        VoIStrategy::CorrelatedKnowledgeGradient => "correlated_kg",
        VoIStrategy::InfluenceDiagramCara => "influence_diagram",
    }
}

fn handle(req: ToolRequest) -> Result<serde_json::Value, String> {
    match req {
        ToolRequest::Route {
            continuous_uncertainties,
            optimize_sample_size,
            sequential_correlated_tests,
            risk_averse_or_causal_graph,
        } => {
            let s = select_strategy(&VoIRequirements {
                continuous_uncertainties,
                optimize_sample_size,
                sequential_correlated_tests,
                risk_averse_or_causal_graph,
            });
            Ok(json!({ "recommended_op": strategy_name(s) }))
        }
        ToolRequest::Evpi { probs, payoffs } => {
            Ok(json!({ "evpi": voi_minimal::evpi(&probs, &payoffs) }))
        }
        ToolRequest::Evsi { probs, payoffs, likelihood } => {
            Ok(json!({ "evsi": voi_minimal::evsi(&probs, &payoffs, &likelihood) }))
        }
        ToolRequest::Cvoi { relevance, uncertainty, reducibility } => {
            Ok(json!({ "cvoi": voi_minimal::cvoi(relevance, uncertainty, reducibility), "max_score": 27 }))
        }
        ToolRequest::MonteCarloEvpi { action_payoffs } => {
            Ok(json!({ "evpi": voi_minimal::monte_carlo_evpi(&action_payoffs) }))
        }
        ToolRequest::GpEvppi { phi_samples, nb_matrix, lengthscale, noise_ratio } => {
            if phi_samples.is_empty() || nb_matrix.is_empty() {
                return Err("phi_samples and nb_matrix must be non-empty".into());
            }
            let (evppi, fitted) = gp_metamodel_evppi(&phi_samples, &nb_matrix, lengthscale, noise_ratio);
            Ok(json!({ "evppi": evppi, "fitted": fitted }))
        }
        ToolRequest::MomentMatchingEvsi {
            g_hat_inb, total_inb_var, tau0, obs_sd, profit_per_unit,
            psi_sd, fixed_cost, cost_per_sample, n_min, n_max,
        } => {
            if g_hat_inb.len() < 2 || n_min > n_max {
                return Err("invalid g_hat_inb length or n_min > n_max".into());
            }
            let res = moment_matching_evsi(
                &g_hat_inb, total_inb_var, tau0, obs_sd, profit_per_unit,
                psi_sd, fixed_cost, cost_per_sample, n_min, n_max,
            );
            Ok(json!({
                "optimal_n": res.optimal_n,
                "evsi": res.evsi_at_n,
                "enbs": res.enbs_at_n,
                "should_run_study": res.enbs_at_n > 0.0
            }))
        }
        ToolRequest::CorrelatedKg { mu, sigma, obs_var, test_cost } => {
            if mu.is_empty() || sigma.len() != mu.len() {
                return Err("mu and sigma dimensions must match and be non-empty".into());
            }
            let scores: Vec<f64> = (0..mu.len())
                .map(|x| correlated_kg_single(&mu, &sigma, x, obs_var))
                .collect();
            let (best_idx, &best_kg) = scores
                .iter()
                .enumerate()
                .max_by(|a, b| a.1.total_cmp(b.1))
                .ok_or("empty scores")?;
            let net_gain = best_kg - test_cost;
            Ok(json!({
                "kg_scores": scores,
                "best_option": best_idx,
                "best_kg": best_kg,
                "net_gain": net_gain,
                "should_measure": net_gain > 0.0
            }))
        }
        ToolRequest::CorrelatedKgUpdate { mut mu, mut sigma, measured_index, observed_value, obs_var } => {
            if measured_index >= mu.len() || sigma.len() != mu.len() {
                return Err("measured_index out of bounds or dimension mismatch".into());
            }
            bayesian_mv_update(&mut mu, &mut sigma, measured_index, observed_value, obs_var);
            Ok(json!({ "mu": mu, "sigma": sigma }))
        }
        ToolRequest::InfluenceDiagram { p_state, p_test_given_state, payoff, gamma } => {
            if p_state.is_empty() || p_test_given_state.len() != p_state.len() || payoff.is_empty() {
                return Err("invalid InfluenceDiagram dimensions".into());
            }
            let id = InfluenceDiagram { p_state, p_test_given_state, payoff, gamma };
            let buying_price = id.buying_price_of_test();
            Ok(json!({
                "buying_price": buying_price,
                "meu_baseline": id.meu_baseline(0.0),
                "meu_with_test": id.meu_with_test(0.0)
            }))
        }
    }
}

fn main() {
    let mut input = String::new();
    if let Err(e) = io::stdin().read_to_string(&mut input) {
        let resp = ToolResponse { ok: false, result: None, error: Some(format!("stdin read error: {e}")) };
        println!("{}", serde_json::to_string(&resp).unwrap());
        process::exit(1);
    }

    let response = match serde_json::from_str::<ToolRequest>(&input) {
        Ok(req) => match handle(req) {
            Ok(val) => ToolResponse { ok: true, result: Some(val), error: None },
            Err(err) => ToolResponse { ok: false, result: None, error: Some(err) },
        },
        Err(e) => ToolResponse { ok: false, result: None, error: Some(format!("invalid JSON request: {e}")) },
    };

    println!("{}", serde_json::to_string(&response).unwrap());
    if !response.ok {
        process::exit(1);
    }
}

```

---

## 5. Build, Test & Agent Usage Examples

```bash
# Build release binary and run all unit tests across the workspace
cargo test --workspace --release
cargo build --release -p voi-cli

# 1. Ask the router which algorithm to use
echo '{"op":"route","continuous_uncertainties":true,"optimize_sample_size":false,"sequential_correlated_tests":true,"risk_averse_or_causal_graph":false}' \
  | ./target/release/voi-cli

# 2. Run Minimal EVSI
echo '{"op":"evsi","probs":[0.4,0.6],"payoffs":[[0,0],[100000,-50000]],"likelihood":[[0.85,0.15],[0.2,0.8]]}' \
  | ./target/release/voi-cli

# 3. Run Correlated Knowledge Gradient
echo '{"op":"correlated_kg","mu":[10.0,12.0,11.5],"sigma":[[9,5,2],[5,9,4],[2,4,9]],"obs_var":4.0,"test_cost":0.25}' \
  | ./target/release/voi-cli

# 4. Run Risk-Averse Influence Diagram (CARA)
echo '{"op":"influence_diagram","p_state":[0.4,0.6],"p_test_given_state":[[0.85,0.15],[0.2,0.8]],"payoff":[[0,0],[100000,-50000]],"gamma":0.00001}' \
  | ./target/release/voi-cli

```
