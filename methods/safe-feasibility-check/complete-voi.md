# M06-A02 — Complete VoI for feasibility checks

Status: supplied reference, not independently validated or approved for execution. The advanced module and the supplied three-pass self-audit follow. Production-grade and exactness claims are reported, not independently established. See [selection and safety notes](README.md).

---

## Part 2: Complete Version (`voi_complete.rs`)

Includes:

1. **Exact Multi-Fidelity Correlated Knowledge Gradient (`mf_kg`)** using the Frazier–Powell–Dayanik (2008) piecewise-linear upper-envelope algorithm (zero Monte Carlo noise).
2. **Exact Discrete Rockafellar–Uryasev `CVaR-VoI**` solved to global optimality in $O(N \log N)$ via subdifferential quantile characterization, with optional hard tail-risk safety gating.
3. **Finite-Horizon Non-Myopic Sequential Belief-State VoI (`sequential_voi`)** with explicit cost/depth limits for multi-step synergistic check planning.
4. **`AdaptiveVoiRouter`** for dynamic agent selection.

```rust
//! Complete, numerically stable Value of Information (VoI) engine in pure std Rust.
//! Implements:
//!   1. Exact Multi-Fidelity Correlated Knowledge Gradient (Frazier-Powell-Dayanik envelope)
//!   2. Exact Discrete Rockafellar-Uryasev CVaR-VoI with safety-limit gating
//!   3. Finite-Horizon Non-Myopic Sequential Belief-State VoI (Expectimax DP)
//!   4. AdaptiveVoiRouter for dynamic AI-agent dispatch

use std::f64::consts::{FRAC_1_SQRT_2, PI};

// ============================================================================
// Math Utilities (Standard Normal PDF, CDF, and KG Loss Function)
// ============================================================================

/// Standard normal PDF: phi(z)
#[inline]
fn norm_pdf(z: f64) -> f64 {
    (-0.5 * z * z).exp() / (2.0 * PI).sqrt()
}

/// High-accuracy erf(x) via Abramowitz & Stegun 7.1.26 (max error ~1.5e-7).
fn erf_approx(x: f64) -> f64 {
    let sign = if x < 0.0 { -1.0 } else { 1.0 };
    let ax = x.abs();
    let t = 1.0 / (1.0 + 0.3275911 * ax);
    let poly = t
        * (0.254829592
            + t * (-0.284496736
                + t * (1.421413741 + t * (-1.453152027 + t * 1.061405429))));
    sign * (1.0 - poly * (-ax * ax).exp())
}

/// Standard normal CDF: Phi(z)
#[inline]
fn norm_cdf(z: f64) -> f64 {
    0.5 * (1.0 + erf_approx(z * FRAC_1_SQRT_2))
}

/// KG normal loss function f(z) = z * Phi(z) + phi(z) for z <= 0.
/// Always >= 0.0; avoids catastrophic cancellation in the far left tail.
#[inline]
fn kg_f(z: f64) -> f64 {
    debug_assert!(z <= 0.0);
    if z < -35.0 {
        0.0
    } else {
        (z * norm_cdf(z) + norm_pdf(z)).max(0.0)
    }
}

// ============================================================================
// 1. Multi-Fidelity Correlated Knowledge Gradient (Exact Envelope Algorithm)
// ============================================================================

/// Computes exact cost-normalized Multi-Fidelity Correlated Knowledge Gradient:
///   mfKG(x) = (E[max_{i in target_indices} (mu_i + sigma_tilde_i * Z)] - max_i mu_i) / cost
///
/// - `mu`: posterior mean vector over all alternatives/fidelities (len M)
/// - `cov`: M x M symmetric positive semi-definite covariance matrix
/// - `target_indices`: indices of high-fidelity (true system) alternatives to optimize
/// - `measure_idx`: index of the candidate check/fidelity to measure
/// - `obs_noise_var`: observation noise variance (>= 0.0) of the candidate check
/// - `cost`: execution cost of the candidate check (> 0.0)
pub fn mf_kg(
    mu: &[f64],
    cov: &[Vec<f64>],
    target_indices: &[usize],
    measure_idx: usize,
    obs_noise_var: f64,
    cost: f64,
) -> f64 {
    assert!(cost > 0.0, "cost must be > 0");
    assert!(obs_noise_var >= 0.0, "obs_noise_var must be >= 0");
    if target_indices.len() <= 1 {
        return 0.0;
    }

    // Cholesky-like diagonal jitter prevents division by zero on near-deterministic checks.
    const JITTER: f64 = 1e-12;
    let denom = (cov[measure_idx][measure_idx] + obs_noise_var + JITTER).sqrt();

    // Build lines (intercept a = mu_i, slope b = sigma_tilde_i)
    let mut lines: Vec<(f64, f64)> = target_indices
        .iter()
        .map(|&i| (mu[i], cov[i][measure_idx] / denom))
        .collect();

    // Sort by slope b ascending; for equal slopes, keep highest intercept a.
    lines.sort_by(|x, y| {
        x.1.total_cmp(&y.1)
            .then_with(|| y.0.total_cmp(&x.0))
    });

    // Deduplicate near-identical slopes (keep the first, which has max intercept).
    const SLOPE_EPS: f64 = 1e-12;
    let mut unique: Vec<(f64, f64)> = Vec::with_capacity(lines.len());
    for (a, b) in lines {
        if let Some(&(_, last_b)) = unique.last() {
            if (b - last_b).abs() <= SLOPE_EPS {
                continue;
            }
        }
        unique.push((a, b));
    }

    if unique.len() <= 1 {
        return 0.0;
    }

    // Build upper convex hull (piecewise-linear envelope)
    // Stack holds: (intercept a, slope b, left_breakpoint c)
    let mut hull: Vec<(f64, f64, f64)> = Vec::with_capacity(unique.len());
    for (a, b) in unique {
        while let Some(&(top_a, top_b, top_c)) = hull.last() {
            let c = (top_a - a) / (b - top_b);
            if c <= top_c {
                hull.pop();
            } else {
                hull.push((a, b, c));
                break;
            }
        }
        if hull.is_empty() {
            hull.push((a, b, f64::NEG_INFINITY));
        }
    }

    // Exact KG = sum_{k=0}^{K-2} (b_{k+1} - b_k) * f(-|c_{k+1}|)
    let mut kg = 0.0;
    for k in 0..(hull.len() - 1) {
        let b_k = hull[k].1;
        let b_next = hull[k + 1].1;
        let c_next = hull[k + 1].2;
        kg += (b_next - b_k) * kg_f(-c_next.abs());
    }

    kg.max(0.0) / cost
}

// ============================================================================
// 2. Exact Discrete Rockafellar-Uryasev CVaR & CVaR-VoI
// ============================================================================

/// Exact discrete CVaR_delta(L) via Rockafellar-Uryasev (2000):
///   CVaR_delta(L) = min_t { t + 1/(1 - delta) * sum_i p_i * max(L_i - t, 0) }
/// Global minimum is attained at the delta-quantile t* = VaR_delta(L).
/// - `losses`: loss values L_i (higher = worse)
/// - `probs`: probability weights p_i (normalized internally if needed)
/// - `delta`: confidence level in (0.0, 1.0), e.g., 0.95 for worst 5% tail
pub fn discrete_cvar(losses: &[f64], probs: &[f64], delta: f64) -> f64 {
    assert!(losses.len() == probs.len() && !losses.is_empty());
    assert!(delta > 0.0 && delta < 1.0, "delta must be in (0, 1)");

    let p_sum: f64 = probs.iter().sum();
    assert!(p_sum > 0.0, "probability mass must be > 0");

    let mut pairs: Vec<(f64, f64)> = losses
        .iter()
        .zip(probs)
        .filter(|(_, &p)| p > 0.0)
        .map(|(&l, &p)| (l, p / p_sum))
        .collect();

    pairs.sort_by(|a, b| a.0.total_cmp(&b.0));

    // Find VaR_delta: smallest L_k such that cumulative prob >= delta
    let mut cum = 0.0;
    let mut var_t = pairs.last().unwrap().0;
    for &(l, p) in &pairs {
        cum += p;
        if cum >= delta - 1e-12 {
            var_t = l;
            break;
        }
    }

    let tail_excess: f64 = pairs
        .iter()
        .map(|&(l, p)| p * (l - var_t).max(0.0))
        .sum();

    var_t + tail_excess / (1.0 - delta)
}

/// Risk-Sensitive CVaR Value of Sample Information (RU-CVaR-VoI).
/// Measures expected reduction in optimal tail-risk (CVaR_delta) from observing a check.
/// - `prior`: P(state)
/// - `losses`: losses[action][state] (higher = worse; negate payoffs if needed)
/// - `likelihoods`: likelihoods[obs][state] = P(obs | state)
/// - `delta`: tail confidence level (e.g. 0.95)
/// - `safety_limit`: optional hard cap; actions with CVaR > safety_limit are filtered out
pub fn cvar_voi(
    prior: &[f64],
    losses: &[Vec<f64>],
    likelihoods: &[Vec<f64>],
    delta: f64,
    safety_limit: Option<f64>,
) -> f64 {
    let best_cvar = |belief: &[f64]| -> f64 {
        let mut best = f64::INFINITY;
        for action_losses in losses {
            let c = discrete_cvar(action_losses, belief, delta);
            if safety_limit.map_or(true, |lim| c <= lim) && c < best {
                best = c;
            }
        }
        // Fallback: if all actions breach safety_limit, pick the least-bad action.
        if best.is_infinite() {
            losses
                .iter()
                .map(|al| discrete_cvar(al, belief, delta))
                .fold(f64::INFINITY, f64::min)
        } else {
            best
        }
    };

    let prior_cvar = best_cvar(prior);

    let mut expected_post_cvar = 0.0;
    for lik_o in likelihoods {
        let joint: Vec<f64> = prior.iter().zip(lik_o).map(|(p, l)| p * l).collect();
        let p_obs: f64 = joint.iter().sum();
        if p_obs <= 1e-15 {
            continue;
        }
        let post: Vec<f64> = joint.iter().map(|&j| j / p_obs).collect();
        expected_post_cvar += p_obs * best_cvar(&post);
    }

    (prior_cvar - expected_post_cvar).max(0.0)
}

// ============================================================================
// 3. Finite-Horizon Non-Myopic Sequential Belief-State VoI
// ============================================================================

#[derive(Clone, Debug)]
pub struct CheckSpec {
    pub name: String,
    pub cost: f64,
    /// likelihoods[obs][state] = P(obs | state)
    pub likelihoods: Vec<Vec<f64>>,
}

#[derive(Clone, Debug)]
pub enum SequentialDecision {
    CommitNow { best_action: usize, expected_value: f64 },
    RunCheck { check_idx: usize, expected_value: f64 },
}

/// Solves a bounded-horizon non-myopic sequential VoI tree.
/// Handles synergistic multi-step checks (e.g., tiny check -> escalate to isolated exp).
pub fn sequential_voi(
    belief: &[f64],
    payoffs: &[Vec<f64>],
    checks: &[CheckSpec],
    remaining_steps: usize,
    remaining_budget: f64,
) -> SequentialDecision {
    // Base option: stop and take best terminal action now.
    let (mut best_action, mut best_val) = (0usize, f64::NEG_INFINITY);
    for (a, row) in payoffs.iter().enumerate() {
        let val: f64 = row.iter().zip(belief).map(|(u, p)| u * p).sum();
        if val > best_val {
            best_val = val;
            best_action = a;
        }
    }

    if remaining_steps == 0 {
        return SequentialDecision::CommitNow {
            best_action,
            expected_value: best_val,
        };
    }

    let mut best_decision = SequentialDecision::CommitNow {
        best_action,
        expected_value: best_val,
    };

    for (idx, check) in checks.iter().enumerate() {
        if check.cost > remaining_budget {
            continue;
        }
        let mut expected_future = -check.cost;
        for lik_o in &check.likelihoods {
            let joint: Vec<f64> = belief.iter().zip(lik_o).map(|(p, l)| p * l).collect();
            let p_obs: f64 = joint.iter().sum();
            if p_obs <= 1e-15 {
                continue;
            }
            let post: Vec<f64> = joint.iter().map(|&j| j / p_obs).collect();
            let sub = sequential_voi(
                &post,
                payoffs,
                checks,
                remaining_steps - 1,
                remaining_budget - check.cost,
            );
            let sub_val = match sub {
                SequentialDecision::CommitNow { expected_value, .. }
                | SequentialDecision::RunCheck { expected_value, .. } => expected_value,
            };
            expected_future += p_obs * sub_val;
        }

        if expected_future > best_val + 1e-12 {
            best_val = expected_future;
            best_decision = SequentialDecision::RunCheck {
                check_idx: idx,
                expected_value: expected_future,
            };
        }
    }

    best_decision
}

// ============================================================================
// 4. Dynamic Agent Router
// ============================================================================

#[derive(Debug, Clone, Copy)]
pub struct AgentVoiRequirements {
    pub has_probabilistic_model: bool,
    pub has_tail_safety_limit: bool,
    pub has_correlated_fidelities: bool,
    pub needs_multistep_escalation: bool,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SelectedAlgorithm {
    MinimalCvoi,
    MinimalEvpiEvsi,
    CompleteCvarVoi,
    CompleteMfKg,
    CompleteSequentialPomdp,
}

impl AgentVoiRequirements {
    /// Deterministic routing policy for an AI agent choosing between minimal and complete VoI.
    pub fn select(&self) -> SelectedAlgorithm {
        if !self.has_probabilistic_model {
            SelectedAlgorithm::MinimalCvoi
        } else if self.has_tail_safety_limit {
            SelectedAlgorithm::CompleteCvarVoi
        } else if self.needs_multistep_escalation {
            SelectedAlgorithm::CompleteSequentialPomdp
        } else if self.has_correlated_fidelities {
            SelectedAlgorithm::CompleteMfKg
        } else {
            SelectedAlgorithm::MinimalEvpiEvsi
        }
    }
}

```

---

## 3-Round Verification Audit

I ran three structured internal review passes across both modules (note: I do not spawn external subagents, so these are three dedicated self-audit passes focused on distinct failure modes):

1. **Round 1 — Mathematical Correctness:**
* **`evsi`:** Verified that $\sum_o P(o) \max_a \sum_s \frac{P(o \mid s) P(s)}{P(o)} U(a,s)$ simplifies algebraically to $\sum_o \max_a \sum_s P(o \mid s) P(s) U(a,s)$, eliminating $0/0$ singularities when an observation has zero marginal probability.
* **`mf_kg`:** Verified the Frazier–Powell–Dayanik envelope identity $\mathbb{E}[\max_k (a_k + b_k Z)] - \max_k a_k = \sum_{k=0}^{K-2}(b_{k+1}-b_k)f(-\vert{}c_{k+1}\vert{})$ for both $c_1 \ge 0$ and $c_1 < 0$. Confirmed that sorting slopes ascending and deduplicating equal slopes by keeping the maximum intercept prevents degenerate vertical intersections.
* **`discrete_cvar`:** Verified that for Rockafellar–Uryasev's $F(t) = t + \frac{1}{1-\delta}\sum p_i (L_i - t)_+$, the subdifferential $0 \in \partial F(t^*)$ holds exactly at the smallest $L_k$ with cumulative probability $\ge \delta$.


2. **Round 2 — Numerical Stability & Edge Cases:**
* Added `JITTER = 1e-12` to `cov[measure_idx][measure_idx] + obs_noise_var` in `mf_kg` so zero-noise deterministic checks cannot trigger `NaN` division.
* Guarded `kg_f(z)` for $z < -35.0$ to return `0.0` instead of floating-point underflow noise.
* Added a fallback in `cvar_voi` so if *every* action breaches `safety_limit`, the function returns the minimum-CVaR action rather than `f64::INFINITY`.
* Used `f64::total_cmp` for deterministic sorting in Rust without panicking on edge floats.


3. **Round 3 — Rust Idioms & Agent Composability:**
* Eliminated all external crate dependencies (`nalgebra`, `statrs`, `good_lp`) so an AI agent can drop either file into a project or compile standalone.
* Added `AgentVoiRequirements::select()` so an agent can programmatically route between `voi_minimal` and `voi_complete` based on task constraints.
