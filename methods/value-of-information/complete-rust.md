# M05 — Complete Rust reference

Revision 2 — supplied reference material. Contains the advanced standalone module, supplied tests, and verification commands. The earlier reference remains in [Git history](https://github.com/661818yijack/ej-engineering-foundations/blob/0baf35e8214aca6359f268da1a0cd83f40efb8b7/methods/value-of-information/complete-rust.md). The supplied source text follows with LF line endings; implementation and audit claims are not independently established. See [selection and adaptation notes](README.md).

---

## 2. Complete Production Implementation (`voi_complete.rs`)

Designed for high-stakes, continuous, sequential, or risk-averse decisions. Includes a **Dynamic Strategy Router** for AI agents and a **`#[cfg(test)]` verification suite**.

```rust
// Compile & run:  rustc -O voi_complete.rs -o voi_complete && ./voi_complete
// Run unit tests: rustc --test -O voi_complete.rs -o voi_tests && ./voi_tests

use std::f64::consts::PI;

// ============================================================================
// 0. AGENT DYNAMIC ROUTER (Selects Algorithm Based on Problem Requirements)
// ============================================================================

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum VoIStrategy {
    GpMetamodelEvppi,
    MomentMatchingEvsi,
    CorrelatedKnowledgeGradient,
    InfluenceDiagramCara,
}

pub struct VoIRequirements {
    pub continuous_uncertainties: bool,
    pub optimize_sample_size: bool,
    pub sequential_correlated_tests: bool,
    pub risk_averse_or_causal_graph: bool,
}

/// Deterministic router an AI agent can call to pick the right VoI engine
pub fn select_strategy(req: &VoIRequirements) -> VoIStrategy {
    if req.risk_averse_or_causal_graph {
        VoIStrategy::InfluenceDiagramCara
    } else if req.sequential_correlated_tests {
        VoIStrategy::CorrelatedKnowledgeGradient
    } else if req.optimize_sample_size {
        VoIStrategy::MomentMatchingEvsi
    } else {
        VoIStrategy::GpMetamodelEvppi
    }
}

// ============================================================================
// 1. CORE MATH, PRNG & LINEAR ALGEBRA PRIMITIVES (Zero Dependencies)
// ============================================================================

pub struct Rng {
    state: u64,
}

impl Rng {
    pub fn new(seed: u64) -> Self {
        Self { state: if seed == 0 { 0x9E3779B97F4A7C15 } else { seed } }
    }

    fn next_u64(&mut self) -> u64 {
        let mut x = self.state;
        x ^= x >> 12;
        x ^= x << 25;
        x ^= x >> 27;
        self.state = x;
        x.wrapping_mul(0x2545F4914F6CDD1D)
    }

    pub fn uniform(&mut self) -> f64 {
        ((self.next_u64() >> 11) as f64 + 0.5) / ((1u64 << 53) as f64)
    }

    pub fn normal(&mut self) -> f64 {
        let u1 = self.uniform();
        let u2 = self.uniform();
        (-2.0 * u1.ln()).sqrt() * (2.0 * PI * u2).cos()
    }
}

fn norm_pdf(z: f64) -> f64 {
    (-0.5 * z * z).exp() / (2.0 * PI).sqrt()
}

fn norm_cdf(z: f64) -> f64 {
    let sign = if z < 0.0 { -1.0 } else { 1.0 };
    let x = z.abs() / std::f64::consts::SQRT_2;
    let t = 1.0 / (1.0 + 0.3275911 * x);
    let y = 1.0
        - (((((1.061405429 * t - 1.453152027) * t) + 1.421413741) * t - 0.284496736) * t
            + 0.254829592)
            * t
            * (-x * x).exp();
    0.5 * (1.0 + sign * y)
}

fn mean(xs: &[f64]) -> f64 {
    xs.iter().sum::<f64>() / xs.len().max(1) as f64
}

fn variance(xs: &[f64]) -> f64 {
    if xs.len() < 2 {
        return 0.0;
    }
    let m = mean(xs);
    xs.iter().map(|&x| (x - m).powi(2)).sum::<f64>() / (xs.len() as f64 - 1.0)
}

/// Cholesky decomposition L * L^T = A with adaptive diagonal jitter
fn cholesky_decompose(a: &[Vec<f64>]) -> Vec<Vec<f64>> {
    let n = a.len();
    let mut jitter = 0.0;
    for attempt in 0..8 {
        let mut l = vec![vec![0.0; n]; n];
        let mut ok = true;
        for i in 0..n {
            for j in 0..=i {
                let mut sum = a[i][j] + if i == j { jitter } else { 0.0 };
                for k in 0..j {
                    sum -= l[i][k] * l[j][k];
                }
                if i == j {
                    if sum <= 1e-12 {
                        ok = false;
                        break;
                    }
                    l[i][j] = sum.sqrt();
                } else {
                    l[i][j] = sum / l[j][j];
                }
            }
            if !ok {
                break;
            }
        }
        if ok {
            return l;
        }
        jitter = 10f64.powi(-9 + attempt);
    }
    panic!("Matrix not positive definite even after jitter escalation");
}

fn cholesky_solve(l: &[Vec<f64>], b: &[f64]) -> Vec<f64> {
    let n = l.len();
    let mut y = vec![0.0; n];
    for i in 0..n {
        let mut sum = b[i];
        for k in 0..i {
            sum -= l[i][k] * y[k];
        }
        y[i] = sum / l[i][i];
    }
    let mut x = vec![0.0; n];
    for i in (0..n).rev() {
        let mut sum = y[i];
        for k in (i + 1)..n {
            sum -= l[k][i] * x[k];
        }
        x[i] = sum / l[i][i];
    }
    x
}

// ============================================================================
// 2. ALGORITHM 1: MULTI-ACTION GP METAMODEL EVPPI (Strong, Oakley & Brennan)
// ============================================================================

fn matern52(x1: &[f64], x2: &[f64], lengthscale: f64, signal_var: f64) -> f64 {
    let dist_sq: f64 = x1.iter().zip(x2.iter()).map(|(a, b)| (a - b).powi(2)).sum();
    let r = dist_sq.sqrt() / lengthscale;
    let sqrt5_r = 5.0f64.sqrt() * r;
    signal_var * (1.0 + sqrt5_r + (5.0 / 3.0) * r * r) * (-sqrt5_r).exp()
}

/// Computes EVPPI(phi) for arbitrary A actions: `nb_matrix[action][sample]`
/// Returns (evppi, fitted_matrix[action][sample])
pub fn gp_metamodel_evppi(
    phi_samples: &[Vec<f64>],
    nb_matrix: &[Vec<f64>],
    lengthscale: f64,
    noise_ratio: f64,
) -> (f64, Vec<Vec<f64>>) {
    let s = phi_samples.len();
    let d = phi_samples[0].len();
    let n_actions = nb_matrix.len();

    // Standardize phi columns
    let mut phi_std = vec![vec![0.0; d]; s];
    for j in 0..d {
        let col: Vec<f64> = phi_samples.iter().map(|r| r[j]).collect();
        let m = mean(&col);
        let sd = variance(&col).sqrt().max(1e-9);
        for i in 0..s {
            phi_std[i][j] = (phi_samples[i][j] - m) / sd;
        }
    }

    // Build unit-variance base kernel matrix once (reused across all actions)
    let mut k_unit = vec![vec![0.0; s]; s];
    for i in 0..s {
        for j in 0..=i {
            let val = matern52(&phi_std[i], &phi_std[j], lengthscale, 1.0);
            k_unit[i][j] = val;
            k_unit[j][i] = val;
        }
    }

    let mut fitted = vec![vec![0.0; s]; n_actions];

    for a in 0..n_actions {
        let y = &nb_matrix[a];
        let y_mean = mean(y);
        let y_var = variance(y);
        if y_var < 1e-12 {
            fitted[a].fill(y_mean);
            continue;
        }

        let sig_var = y_var * (1.0 - noise_ratio);
        let noise_var = y_var * noise_ratio;
        let y_centered: Vec<f64> = y.iter().map(|&v| v - y_mean).collect();

        let mut k_noise = vec![vec![0.0; s]; s];
        for i in 0..s {
            for j in 0..=i {
                let v = k_unit[i][j] * sig_var + if i == j { noise_var } else { 0.0 };
                k_noise[i][j] = v;
                k_noise[j][i] = v;
            }
        }

        let chol = cholesky_decompose(&k_noise);
        let alpha = cholesky_solve(&chol, &y_centered);

        for i in 0..s {
            let mut pred = y_mean;
            for j in 0..s {
                pred += (k_unit[i][j] * sig_var) * alpha[j];
            }
            fitted[a][i] = pred;
        }
    }

    let v_prior = fitted
        .iter()
        .map(|row| mean(row))
        .fold(f64::NEG_INFINITY, f64::max);

    let v_perfect: f64 = (0..s)
        .map(|i| {
            fitted
                .iter()
                .map(|row| row[i])
                .fold(f64::NEG_INFINITY, f64::max)
        })
        .sum::<f64>()
        / s as f64;

    ((v_perfect - v_prior).max(0.0), fitted)
}

// ============================================================================
// 3. ALGORITHM 2: EXTENDED MOMENT-MATCHING EVSI (Optimal Sample Size n*)
// ============================================================================

pub struct EvsiOptimizationResult {
    pub optimal_n: usize,
    pub evsi_at_n: f64,
    pub enbs_at_n: f64,
}

pub fn moment_matching_evsi(
    g_hat_inb: &[f64],
    total_inb_var: f64,
    tau0: f64,
    obs_sd: f64,
    profit_per_unit: f64,
    psi_sd: f64,
    fixed_cost: f64,
    cost_per_sample: f64,
    n_min: usize,
    n_max: usize,
) -> EvsiOptimizationResult {
    let s = g_hat_inb.len();
    let eta_bar = mean(g_hat_inb);
    let sigma_phi_sq = variance(g_hat_inb).max(1e-12);

    let q_count = 50;
    let mut best = EvsiOptimizationResult {
        optimal_n: n_min,
        evsi_at_n: 0.0,
        enbs_at_n: f64::NEG_INFINITY,
    };

    for q in 0..q_count {
        let frac = (q as f64 + 0.5) / q_count as f64;
        let n_f = n_min as f64 + frac * (n_max - n_min) as f64;

        let post_phi_var = 1.0 / (1.0 / (tau0 * tau0) + n_f / (obs_sd * obs_sd));
        let post_inb_var = profit_per_unit.powi(2) * post_phi_var + psi_sd.powi(2);

        let r = ((total_inb_var - post_inb_var) / sigma_phi_sq)
            .clamp(0.0, 1.0)
            .sqrt();

        let sum_pos: f64 = g_hat_inb
            .iter()
            .map(|&eta| (eta_bar + r * (eta - eta_bar)).max(0.0))
            .sum();

        let evsi = (sum_pos / s as f64 - eta_bar.max(0.0)).max(0.0);
        let enbs = evsi - (fixed_cost + cost_per_sample * n_f);

        if enbs > best.enbs_at_n {
            best = EvsiOptimizationResult {
                optimal_n: n_f.round() as usize,
                evsi_at_n: evsi,
                enbs_at_n: enbs,
            };
        }
    }
    best
}

// ============================================================================
// 4. ALGORITHM 3: CORRELATED KNOWLEDGE GRADIENT (Frazier, Powell & Dayanik)
// ============================================================================

fn kg_f(z: f64) -> f64 {
    if z < -30.0 {
        0.0
    } else {
        z * norm_cdf(z) + norm_pdf(z)
    }
}

pub fn correlated_kg_single(mu: &[f64], sigma: &[Vec<f64>], x: usize, obs_var: f64) -> f64 {
    let m = mu.len();
    let denom = (sigma[x][x] + obs_var).sqrt();
    if denom < 1e-12 {
        return 0.0;
    }

    let mut lines: Vec<(f64, f64)> = (0..m).map(|i| (mu[i], sigma[i][x] / denom)).collect();
    lines.sort_by(|l1, l2| l1.1.total_cmp(&l2.1).then_with(|| l2.0.total_cmp(&l1.0)));

    let mut unique: Vec<(f64, f64)> = Vec::with_capacity(m);
    for (a, b) in lines {
        if let Some(&(_, prev_b)) = unique.last() {
            if (b - prev_b).abs() < 1e-12 {
                continue;
            }
        }
        unique.push((a, b));
    }
    if unique.len() <= 1 {
        return 0.0;
    }

    let mut hull_lines: Vec<(f64, f64)> = Vec::with_capacity(unique.len());
    let mut hull_c: Vec<f64> = Vec::with_capacity(unique.len());

    for (a_new, b_new) in unique {
        while hull_lines.len() >= 2 {
            let (a_top, b_top) = *hull_lines.last().unwrap();
            let c_new = (a_top - a_new) / (b_new - b_top);
            let c_prev = *hull_c.last().unwrap();
            if c_new <= c_prev {
                hull_lines.pop();
                hull_c.pop();
            } else {
                break;
            }
        }
        if let Some(&(a_top, b_top)) = hull_lines.last() {
            hull_c.push((a_top - a_new) / (b_new - b_top));
        }
        hull_lines.push((a_new, b_new));
    }

    let mut kg = 0.0;
    for k in 0..hull_c.len() {
        kg += (hull_lines[k + 1].1 - hull_lines[k].1) * kg_f(-hull_c[k].abs());
    }
    kg.max(0.0)
}

pub fn bayesian_mv_update(mu: &mut [f64], sigma: &mut [Vec<f64>], x: usize, y: f64, obs_var: f64) {
    let m = mu.len();
    let s_xx_r = sigma[x][x] + obs_var;
    let residual = y - mu[x];
    let col_x: Vec<f64> = (0..m).map(|i| sigma[i][x]).collect();

    for i in 0..m {
        mu[i] += (col_x[i] / s_xx_r) * residual;
    }
    for i in 0..m {
        for j in 0..m {
            sigma[i][j] -= (col_x[i] * col_x[j]) / s_xx_r;
        }
    }
}

// ============================================================================
// 5. ALGORITHM 4: EXACT INFLUENCE DIAGRAM VoI WITH CARA RISK AVERSION
// ============================================================================

fn cara_utility(wealth: f64, gamma: f64) -> f64 {
    if gamma.abs() < 1e-12 {
        wealth
    } else {
        (1.0 - (-gamma * wealth).exp()) / gamma
    }
}

pub struct InfluenceDiagram {
    pub p_state: Vec<f64>,
    pub p_test_given_state: Vec<Vec<f64>>,
    pub payoff: Vec<Vec<f64>>,
    pub gamma: f64,
}

impl InfluenceDiagram {
    pub fn meu_baseline(&self, wealth_shift: f64) -> f64 {
        self.payoff
            .iter()
            .map(|a_row| {
                self.p_state
                    .iter()
                    .enumerate()
                    .map(|(s, &ps)| ps * cara_utility(a_row[s] + wealth_shift, self.gamma))
                    .sum::<f64>()
            })
            .fold(f64::NEG_INFINITY, f64::max)
    }

    pub fn meu_with_test(&self, wealth_shift: f64) -> f64 {
        let n_states = self.p_state.len();
        let n_tests = self.p_test_given_state[0].len();
        let mut total_eu = 0.0;

        for t in 0..n_tests {
            let p_t: f64 = (0..n_states)
                .map(|s| self.p_state[s] * self.p_test_given_state[s][t])
                .sum();
            if p_t <= 1e-15 {
                continue;
            }
            let best_eu = self
                .payoff
                .iter()
                .map(|a_row| {
                    (0..n_states)
                        .map(|s| {
                            let post = (self.p_state[s] * self.p_test_given_state[s][t]) / p_t;
                            post * cara_utility(a_row[s] + wealth_shift, self.gamma)
                        })
                        .sum::<f64>()
                })
                .fold(f64::NEG_INFINITY, f64::max);
            total_eu += p_t * best_eu;
        }
        total_eu
    }

    /// Exact Buying Price of Information v* solving MEU_info(-v*) = MEU_base(0)
    pub fn buying_price_of_test(&self) -> f64 {
        let target = self.meu_baseline(0.0);
        if self.meu_with_test(0.0) <= target {
            return 0.0;
        }
        let mut lo = 0.0;
        let mut hi = 1.0;
        while self.meu_with_test(-hi) > target {
            hi *= 2.0;
        }
        for _ in 0..80 {
            let mid = 0.5 * (lo + hi);
            if self.meu_with_test(-mid) > target {
                lo = mid;
            } else {
                hi = mid;
            }
        }
        0.5 * (lo + hi)
    }
}

// ============================================================================
// 6. MAIN DEMO & GROUND-TRUTH UNIT TESTS
// ============================================================================

fn main() {
    let req = VoIRequirements {
        continuous_uncertainties: true,
        optimize_sample_size: false,
        sequential_correlated_tests: true,
        risk_averse_or_causal_graph: false,
    };
    println!("Agent Router Selected: {:?}\n", select_strategy(&req));

    // Demo: Correlated KG
    let mut mu = vec![10.0, 12.0, 11.5];
    let mut sigma = vec![
        vec![9.0, 5.0, 2.0],
        vec![5.0, 9.0, 4.0],
        vec![2.0, 4.0, 9.0],
    ];
    let kgs: Vec<f64> = (0..3).map(|x| correlated_kg_single(&mu, &sigma, x, 4.0)).collect();
    println!("cKG scores: {:?}", kgs);
    bayesian_mv_update(&mut mu, &mut sigma, 1, 14.0, 4.0);
    println!("Updated means: {:?}", mu);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_zero_info_test_yields_zero_buying_price() {
        let id = InfluenceDiagram {
            p_state: vec![0.4, 0.6],
            p_test_given_state: vec![vec![0.5, 0.5], vec![0.5, 0.5]], // Pure noise
            payoff: vec![vec![0.0, 0.0], vec![100_000.0, -50_000.0]],
            gamma: 1e-5,
        };
        assert!(id.buying_price_of_test() < 1e-6);
    }

    #[test]
    fn test_risk_neutral_cara_matches_exact_linear_evsi() {
        let id = InfluenceDiagram {
            p_state: vec![0.4, 0.6],
            p_test_given_state: vec![vec![0.85, 0.15], vec![0.20, 0.80]],
            payoff: vec![vec![0.0, 0.0], vec![100_000.0, -50_000.0]],
            gamma: 0.0, // Risk-neutral
        };
        // Exact hand-calculated EVSI = 18,000.0
        assert!((id.buying_price_of_test() - 18_000.0).abs() < 1e-4);
    }

    #[test]
    fn test_ckg_matches_monte_carlo_ground_truth() {
        let mu = vec![10.0, 12.0, 11.5];
        let sigma = vec![
            vec![9.0, 5.0, 2.0],
            vec![5.0, 9.0, 4.0],
            vec![2.0, 4.0, 9.0],
        ];
        let obs_var = 4.0;
        let exact_kg = correlated_kg_single(&mu, &sigma, 0, obs_var);

        // 100k Monte Carlo check for E[max_i (mu_i + sigma_tilde_i * Z)] - max_i mu_i
        let mut rng = Rng::new(777);
        let denom = (sigma[0][0] + obs_var).sqrt();
        let b: Vec<f64> = (0..3).map(|i| sigma[i][0] / denom).collect();
        let base = mu.iter().copied().fold(f64::NEG_INFINITY, f64::max);
        let n = 100_000;
        let mut sum = 0.0;
        for _ in 0..n {
            let z = rng.normal();
            let best = (0..3)
                .map(|i| mu[i] + b[i] * z)
                .fold(f64::NEG_INFINITY, f64::max);
            sum += best;
        }
        let mc_kg = (sum / n as f64) - base;
        assert!((exact_kg - mc_kg).abs() < 0.05, "exact={} vs mc={}", exact_kg, mc_kg);
    }
}

```

### Quick Verification Commands

```bash
# 1. Run Minimal Version
rustc -O voi_minimal.rs -o voi_minimal && ./voi_minimal

# 2. Run Complete Version & Ground-Truth Tests
rustc --test -O voi_complete.rs -o voi_tests && ./voi_tests

```
