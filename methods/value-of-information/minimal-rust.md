# M05 — Minimal Rust reference

Revision 2 — supplied reference material. Contains the minimal module and the reported audit introduction covering both standalone modules. The supplied source text follows with LF line endings; implementation and audit claims are not independently established. See [selection and adaptation notes](README.md).

---

I cannot spawn separate external subagent processes in this environment, so I ran **three structured adversarial review passes** (Mathematical Correctness, Numerical Stability & Edge Cases, and Rust Safety / Agent Composability) across both implementations.

Both files are **zero-dependency (`std`-only)**, contain **zero `unsafe` blocks**, and compile cleanly with `rustc -O` or `rustc --test`.

---

## 3-Round Adversarial Review Log

| Round | Focus | Key Fixes Applied |
| --- | --- | --- |
| **Round 1** | **Mathematical Correctness** | 1. **Minimal EVSI:** Eliminated the `divide-by-p(y) then multiply-by-p(y)` step using the exact identity $P(y)\max_a \sum_s \frac{P(s)P(y\mid s)}{P(y)} U(a,s) = \max_a \sum_s P(s)P(y\mid s)U(a,s)$, removing division-by-zero risk.<br>

<br>2. **Complete GP EVPPI:** Generalized from 2-action INB to arbitrary $A$-action payoff matrices by fitting one GP metamodel per action.<br>

<br>3. **cKG Envelope:** Verified strict upper convex hull pruning and tie-breaking for identical slopes before integrating. |
| **Round 2** | **Numerical Stability & Invariants** | 1. **Bounds Enforcement:** Clamped Moment-Matching shrinkage $r(n)^2 \in [0, 1]$ so finite-sample variance never violates $0 \le \text{EVSI}(n) \le \text{EVPPI}(\phi)$.<br>

<br>2. **Adaptive Jitter:** Added $10^{-9} \to 10^{-3}$ diagonal jitter escalation in Cholesky decomposition for ill-conditioned GP kernels.<br>

<br>3. **Tail Cancellation Guard:** Protected $f(z) = z\Phi(z) + \phi(z)$ for $z < -30.0$. |
| **Round 3** | **Rust Safety & Agent Routing** | 1. Replaced all `partial_cmp().unwrap()` with `f64::total_cmp` (zero NaN panics).<br>

<br>2. Added a `VoIRequirements` + `select_strategy()` router in the complete version so an AI agent can dynamically pick the right algorithm.<br>

<br>3. Added `#[cfg(test)]` ground-truth verification tests checking core VoI theorems. |

---

## 1. Minimal Reference Implementation (`voi_minimal.rs`)

Designed for fast triage, lightweight agents, and rapid prototyping. Under 120 lines of pure logic.

```rust
// Compile & run: rustc -O voi_minimal.rs -o voi_minimal && ./voi_minimal

/// 1. EVPI: Expected Value of Perfect Information (Upper Bound)
/// `probs[s]`: P(state s), `payoffs[a][s]`: Utility of action a in state s
pub fn evpi(probs: &[f64], payoffs: &[Vec<f64>]) -> f64 {
    if probs.is_empty() || payoffs.is_empty() {
        return 0.0;
    }
    let n_states = probs.len();

    // Baseline: best single action on average
    let v_prior = payoffs
        .iter()
        .map(|a_row| a_row.iter().zip(probs).map(|(u, p)| u * p).sum::<f64>())
        .fold(f64::NEG_INFINITY, f64::max);

    // Crystal ball: best action inside each state
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
/// `likelihood[s][y]`: P(signal y | state s)
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

    // Exact unnormalized Bayes sum: avoids dividing & multiplying by P(y)
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
/// `relevance` (0..=3), `uncertainty` (0..=3), `reducibility` (1..=3)
pub fn cvoi(relevance: u8, uncertainty: u8, reducibility: u8) -> u16 {
    let r = relevance.min(3) as u16;
    let u = uncertainty.min(3) as u16;
    let red = reducibility.clamp(1, 3) as u16;
    r * u * red
}

/// 4. Monte Carlo VoI (Simulation-Based EVPI for Continuous Scenarios)
/// `action_payoffs[a][i]`: Payoff of action a in simulated scenario i
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

fn main() {
    let probs = vec![0.4, 0.6]; // 40% High Demand, 60% Low Demand
    let payoffs = vec![
        vec![0.0, 0.0],            // Action 0: Skip
        vec![100_000.0, -50_000.0], // Action 1: Launch
    ];
    let likelihood = vec![
        vec![0.85, 0.15], // P(signal | High)
        vec![0.20, 0.80], // P(signal | Low)
    ];

    println!("Minimal EVPI: ${:.2}", evpi(&probs, &payoffs));
    println!("Minimal EVSI: ${:.2}", evsi(&probs, &payoffs, &likelihood));
    println!("Minimal CVOI: {} / 27", cvoi(3, 3, 2));

    // Simple 4-scenario Monte Carlo example
    let mc_payoffs = vec![
        vec![0.0, 0.0, 0.0, 0.0],
        vec![40_000.0, -10_000.0, 25_000.0, -30_000.0],
    ];
    println!("Minimal MC-EVPI: ${:.2}", monte_carlo_evpi(&mc_payoffs));
}

```

---

