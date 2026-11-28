# M06-A01 — Minimal VoI for feasibility checks

Status: supplied reference, not independently validated or approved for execution. The original source introduction and minimal module follow. See [selection and safety notes](README.md).

---

Both Rust implementations below are **zero-dependency (`std` only)** so an AI agent can compile, test, and embed them immediately with `rustc` or `cargo` without crate-version friction.

* **Module 1 (`voi_minimal.rs`):** Fast triage (`CVOI`, `EVPI`, `EVSI`, `Myopic Beta-Binomial`). Use when latency, simplicity, or limited prior data matter most.
* **Module 2 (`voi_complete.rs`):** Production-grade (`mfKG` with exact Frazier–Powell–Dayanik piecewise-linear envelope, exact discrete Rockafellar–Uryasev `CVaR-VoI`, and finite-horizon non-myopic sequential belief-state DP), plus an `AdaptiveVoiRouter` that lets an AI agent pick the right tier dynamically.

---

## Part 1: Minimal Version (`voi_minimal.rs`)

```rust
//! Minimal Value of Information (VoI) toolkit for Safe Feasibility Checks.
//! Zero external dependencies. Designed for fast agent triage and stopping rules.

/// 1. Constructed VoI (CVOI): Ordinal heuristic when exact probabilities are unknown.
/// relevance: 0..=3, uncertainty: 0..=3, cost: >= 1.0
pub fn cvoi(relevance: f64, uncertainty: f64, cost: f64) -> f64 {
    assert!(cost > 0.0, "cost must be strictly positive");
    (relevance.max(0.0) * uncertainty.max(0.0)) / cost
}

/// 2. Expected Value of Perfect Information (EVPI).
/// Hard upper bound: if test_cost >= evpi(), never run the check.
/// - `prior`: P(state), must sum to ~1.0
/// - `payoffs`: payoffs[action][state]
pub fn evpi(prior: &[f64], payoffs: &[Vec<f64>]) -> f64 {
    let n_states = prior.len();
    assert!(n_states > 0 && !payoffs.is_empty());

    let prior_val = payoffs
        .iter()
        .map(|row| row.iter().zip(prior).map(|(u, p)| u * p).sum::<f64>())
        .fold(f64::NEG_INFINITY, f64::max);

    let perfect_val: f64 = (0..n_states)
        .map(|s| {
            let best_for_s = payoffs
                .iter()
                .map(|row| row[s])
                .fold(f64::NEG_INFINITY, f64::max);
            prior[s] * best_for_s
        })
        .sum();

    (perfect_val - prior_val).max(0.0)
}

/// 3. Expected Value of Sample Information (EVSI) & Expected Net Benefit of Sampling (ENBS).
/// - `likelihoods`: likelihoods[obs][state] = P(obs | state)
///   For each state s, sum_obs P(obs | s) should equal 1.0.
pub fn evsi(prior: &[f64], payoffs: &[Vec<f64>], likelihoods: &[Vec<f64>]) -> f64 {
    let prior_val = payoffs
        .iter()
        .map(|row| row.iter().zip(prior).map(|(u, p)| u * p).sum::<f64>())
        .fold(f64::NEG_INFINITY, f64::max);

    // Numerically stable joint form:
    // sum_o P(o) * max_a sum_s [P(o|s)P(s)/P(o)] * U(a,s)
    // == sum_o max_a sum_s P(o|s) * P(s) * U(a,s)
    // Avoids 0/0 division when P(o) == 0.
    let expected_post_val: f64 = likelihoods
        .iter()
        .map(|lik_o| {
            payoffs
                .iter()
                .map(|row| {
                    row.iter()
                        .zip(prior)
                        .zip(lik_o)
                        .map(|((u, p), l)| u * p * l)
                        .sum::<f64>()
                })
                .fold(f64::NEG_INFINITY, f64::max)
        })
        .sum();

    (expected_post_val - prior_val).max(0.0)
}

pub fn enbs(prior: &[f64], payoffs: &[Vec<f64>], likelihoods: &[Vec<f64>], cost: f64) -> f64 {
    evsi(prior, payoffs, likelihoods) - cost
}

/// 4. Myopic 1-Step Beta-Binomial VoI for iterative bounded checks.
/// Action 0 (Abort): fixed payoff `u_abort`.
/// Action 1 (Commit): `u_pass` if feasible, `u_fail` if infeasible.
/// Returns Net VoI of running 1 more Bernoulli check; stop when <= 0.0.
pub fn myopic_beta_binomial_net_voi(
    alpha: f64,
    beta: f64,
    u_abort: f64,
    u_pass: f64,
    u_fail: f64,
    step_cost: f64,
) -> f64 {
    assert!(alpha > 0.0 && beta > 0.0);
    let n = alpha + beta;
    let p = alpha / n;

    let commit_val = |prob: f64| prob * u_pass + (1.0 - prob) * u_fail;
    let prior_val = u_abort.max(commit_val(p));

    let p_after_pass = (alpha + 1.0) / (n + 1.0);
    let p_after_fail = alpha / (n + 1.0);

    let post_val = p * u_abort.max(commit_val(p_after_pass))
        + (1.0 - p) * u_abort.max(commit_val(p_after_fail));

    (post_val - prior_val).max(0.0) - step_cost
}

```

---

