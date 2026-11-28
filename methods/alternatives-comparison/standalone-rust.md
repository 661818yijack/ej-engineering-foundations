# M04 — Standalone MCDA Rust reference

Status: supplied reference, not compiled or independently validated by this repository editor. Contains both MINIMAL_MCDA (M04-A01) and COMPLETE_MCDA (M04-A02). Audit and performance statements below are supplied claims, not verified results. The source text is preserved with LF line endings. See [selection and adaptation notes](README.md) before use.

---

Below is a **zero-dependency, single-file Rust implementation** (`mcda_engine.rs`) containing `MINIMAL_MCDA`, `COMPLETE_MCDA` (Veto Gate, SPOTIS Fixed-Bounds, Shannon Entropy Weight Fallback, VIKOR + Stability Proofs, and SMAA-2 Monte Carlo), and the dynamic `Auto` router.

It compiles directly with `rustc -O mcda_engine.rs` (no `Cargo.toml` or external crates required) and uses a deterministic `SplitMix64` PRNG so Monte Carlo results are 100% reproducible across runs.

---

## 3-Round Verification Audit Summary

Before finalizing the code, the implementation was audited across three passes:

* **Round 1 — Mathematical & Algorithmic Correctness:**
* *SPOTIS Clamping Bug Caught:* If a raw score exceeds the ideal bound (e.g., a cost lower than `bounds.0`), taking $\vert{}x - x^*\vert{}$ without clamping first would *penalize* a better-than-ideal option. **Fix:** Clamp `x` into `[lo, hi]` before computing distance to $x^*$.
* *VIKOR Compromise Set Rule:* Verified the exact Opricovic & Tzeng (2004) branching logic: if **C1 fails**, include all $A^{(k)}$ where $Q(A^{(k)}) - Q(A^{(1)}) < \frac{1}{m-1}$; if **C1 passes but C2 fails**, return $\{A^{(1)}, A^{(2)}\}$.
* *SMAA-2 Consistency:* Ensured Stage 5 evaluates perturbed inputs using the same SPOTIS + VIKOR $Q_i$ objective as Stage 4 rather than switching to a mismatched linear sum.


* **Round 2 — Numerical Stability & Edge Cases:**
* *Zero-Division Guards:* Added `EPSILON = 1e-12` guards for flat criteria ($\max == \min$), identical $S_i$/$R_i$ spreads ($S^- == S^*$), and $m < 2$ surviving alternatives after Veto filtering.
* *Shannon Entropy Underflow:* Added an $\epsilon$ shift to $p_{ij}$ before $\ln(p_{ij})$ and guarded $\ln(m)$ when $m \le 1$.
* *Float NaN / Tie-Breaking:* Replaced raw `.partial_cmp().unwrap()` with `total_cmp` plus deterministic `id` tie-breaking so sorting can never panic.


* **Round 3 — Rust Idioms & Performance:**
* *Zero-Allocation Hot Loop:* Pre-allocated scratch buffers (`s_buf`, `r_buf`, `q_buf`, `w_buf`, `idx_buf`) outside the 10,000-iteration SMAA-2 loop so the Monte Carlo engine runs in a few milliseconds on a standard PC.



---

## Complete Rust Implementation (`mcda_engine.rs`)

```rust
use std::collections::HashMap;

const EPS: f64 = 1e-12;

// ============================================================================
// 1. SHARED DATA CONTRACT (Unified Input / Output for AI Agents)
// ============================================================================

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Direction {
    Max, // Higher is better (e.g., reliability, throughput)
    Min, // Lower is better (e.g., cost, latency)
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum EngineMode {
    Minimal,
    Complete,
    Auto,
}

#[derive(Debug, Clone)]
pub struct Criterion {
    pub name: String,
    pub direction: Direction,
    pub weight: f64,                   // Set all to 0.0 to trigger Entropy weights
    pub bounds: Option<(f64, f64)>,    // Fixed (min, max) for SPOTIS rank-reversal immunity
    pub veto_threshold: Option<f64>,   // Hard deal-breaker limit
    pub uncertainty_pct: f64,          // e.g., 0.05 for +/- 5% measurement noise
}

#[derive(Debug, Clone)]
pub struct Alternative {
    pub id: String,
    pub scores: HashMap<String, f64>,
}

#[derive(Debug, Clone)]
pub struct Config {
    pub mode: EngineMode,
    pub vikor_v: f64,                  // Strategy weight [0.0..1.0], default 0.5
    pub mc_iterations: usize,          // SMAA-2 Monte Carlo trials, default 10_000
    pub weight_perturbation_pct: f64,  // e.g., 0.20 for +/- 20% weight sensitivity
    pub rng_seed: u64,                 // Deterministic seed for reproducibility
}

impl Default for Config {
    fn default() -> Self {
        Self {
            mode: EngineMode::Auto,
            vikor_v: 0.5,
            mc_iterations: 10_000,
            weight_perturbation_pct: 0.20,
            rng_seed: 42,
        }
    }
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum DecisionStatus {
    ClearWinner,
    CompromiseSet,
    NoValidAlternatives,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum StabilityVerdict {
    Robust,    // Win probability > 75%
    Sensitive, // Win probability 50% - 75%
    Fragile,   // Win probability < 50%
    NotApplicable,
}

#[derive(Debug, Clone)]
pub struct RankedAlternative {
    pub id: String,
    pub rank: Option<usize>,           // None if disqualified
    pub primary_score: f64,            // WSM (higher=better) or VIKOR Q (lower=better)
    pub vikor_s: Option<f64>,          // Group utility (lower=better)
    pub vikor_r: Option<f64>,          // Max regret (lower=better)
    pub win_probability: Option<f64>,  // SMAA-2 rank #1 probability
    pub top2_probability: Option<f64>, // SMAA-2 top-2 probability
    pub disqualified: bool,
    pub veto_reasons: Vec<String>,
}

#[derive(Debug, Clone)]
pub struct DecisionReport {
    pub engine_used: &'static str,
    pub status: DecisionStatus,
    pub winners: Vec<String>,
    pub stability_verdict: StabilityVerdict,
    pub effective_weights: Vec<(String, f64)>,
    pub rankings: Vec<RankedAlternative>,
}

// ============================================================================
// 2. DETERMINISTIC PRNG (SplitMix64 — Zero External Dependencies)
// ============================================================================

struct SplitMix64 {
    state: u64,
}

impl SplitMix64 {
    fn new(seed: u64) -> Self {
        Self { state: seed }
    }

    #[inline]
    fn next_u64(&mut self) -> u64 {
        self.state = self.state.wrapping_add(0x9E37_79B9_7F4A_7C15);
        let mut z = self.state;
        z = (z ^ (z >> 30)).wrapping_mul(0xBF58_476D_1CE4_E5B9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94D0_49BB_1331_11EB);
        z ^ (z >> 31)
    }

    /// Uniform float in [0.0, 1.0)
    #[inline]
    fn next_f64(&mut self) -> f64 {
        // Use top 53 bits for full IEEE-754 f64 mantissa precision
        ((self.next_u64() >> 11) as f64) * (1.0 / ((1u64 << 53) as f64))
    }

    /// Uniform float in [low, high)
    #[inline]
    fn uniform(&mut self, low: f64, high: f64) -> f64 {
        low + (high - low) * self.next_f64()
    }
}

// ============================================================================
// 3. AUTO-ROUTER (Dynamic Selection for AI Agents)
// ============================================================================

pub fn run_mcda(
    alternatives: &[Alternative],
    criteria: &[Criterion],
    config: &Config,
) -> DecisionReport {
    let use_complete = match config.mode {
        EngineMode::Minimal => false,
        EngineMode::Complete => true,
        EngineMode::Auto => {
            let has_veto = criteria.iter().any(|c| c.veto_threshold.is_some());
            let has_bounds = criteria.iter().any(|c| c.bounds.is_some());
            let has_uncertainty = criteria.iter().any(|c| c.uncertainty_pct > EPS);
            let missing_weights = criteria.iter().all(|c| c.weight <= EPS);
            has_veto || has_bounds || has_uncertainty || missing_weights || alternatives.len() >= 5
        }
    };

    if use_complete {
        complete_mcda(alternatives, criteria, config)
    } else {
        minimal_mcda(alternatives, criteria)
    }
}

// ============================================================================
// 4. VERSION 1: MINIMAL_MCDA (Min-Max Normalized WSM)
// ============================================================================

pub fn minimal_mcda(alternatives: &[Alternative], criteria: &[Criterion]) -> DecisionReport {
    if alternatives.is_empty() || criteria.is_empty() {
        return empty_report("MINIMAL_MCDA");
    }

    let n = criteria.len();
    let weights = normalize_weights(criteria);

    // Precompute min/max per criterion
    let mut bounds = Vec::with_capacity(n);
    for c in criteria {
        let mut lo = f64::INFINITY;
        let mut hi = f64::NEG_INFINITY;
        for a in alternatives {
            let val = *a.scores.get(&c.name).unwrap_or(&0.0);
            if val < lo { lo = val; }
            if val > hi { hi = val; }
        }
        bounds.push((lo, hi));
    }

    let mut ranked: Vec<RankedAlternative> = alternatives
        .iter()
        .map(|a| {
            let mut total = 0.0;
            for (j, c) in criteria.iter().enumerate() {
                let val = *a.scores.get(&c.name).unwrap_or(&0.0);
                let (lo, hi) = bounds[j];
                let norm = if (hi - lo).abs() < EPS {
                    1.0
                } else {
                    match c.direction {
                        Direction::Max => (val - lo) / (hi - lo),
                        Direction::Min => (hi - val) / (hi - lo),
                    }
                };
                total += weights[j] * norm;
            }
            RankedAlternative {
                id: a.id.clone(),
                rank: None,
                primary_score: total,
                vikor_s: None,
                vikor_r: None,
                win_probability: None,
                top2_probability: None,
                disqualified: false,
                veto_reasons: vec![],
            }
        })
        .collect();

    // Sort descending by WSM score, tie-break by id
    ranked.sort_by(|a, b| {
        b.primary_score
            .total_cmp(&a.primary_score)
            .then_with(|| a.id.cmp(&b.id))
    });

    for (idx, item) in ranked.iter_mut().enumerate() {
        item.rank = Some(idx + 1);
    }

    let winners = vec![ranked[0].id.clone()];
    let effective_weights = criteria
        .iter()
        .zip(weights.iter())
        .map(|(c, &w)| (c.name.clone(), w))
        .collect();

    DecisionReport {
        engine_used: "MINIMAL_MCDA",
        status: DecisionStatus::ClearWinner,
        winners,
        stability_verdict: StabilityVerdict::NotApplicable,
        effective_weights,
        rankings: ranked,
    }
}

// ============================================================================
// 5. VERSION 2: COMPLETE_MCDA (Veto + SPOTIS + Entropy + VIKOR + SMAA-2)
// ============================================================================

pub fn complete_mcda(
    alternatives: &[Alternative],
    criteria: &[Criterion],
    config: &Config,
) -> DecisionReport {
    if alternatives.is_empty() || criteria.is_empty() {
        return empty_report("COMPLETE_MCDA");
    }

    let n = criteria.len();
    let v = config.vikor_v.clamp(0.0, 1.0);

    // --- STAGE 1: Non-Compensatory Veto Gate (ELECTRE III style) ---
    let mut survivors: Vec<&Alternative> = Vec::new();
    let mut disqualified_list: Vec<RankedAlternative> = Vec::new();

    for a in alternatives {
        let mut reasons = Vec::new();
        for c in criteria {
            if let Some(veto) = c.veto_threshold {
                let val = *a.scores.get(&c.name).unwrap_or(&0.0);
                let violated = match c.direction {
                    Direction::Min => val > veto,
                    Direction::Max => val < veto,
                };
                if violated {
                    reasons.push(format!("{}={:.4} violated veto {:.4}", c.name, val, veto));
                }
            }
        }
        if reasons.is_empty() {
            survivors.push(a);
        } else {
            disqualified_list.push(RankedAlternative {
                id: a.id.clone(),
                rank: None,
                primary_score: f64::INFINITY,
                vikor_s: None,
                vikor_r: None,
                win_probability: Some(0.0),
                top2_probability: Some(0.0),
                disqualified: true,
                veto_reasons: reasons,
            });
        }
    }

    let m = survivors.len();
    if m == 0 {
        return DecisionReport {
            engine_used: "COMPLETE_MCDA",
            status: DecisionStatus::NoValidAlternatives,
            winners: vec![],
            stability_verdict: StabilityVerdict::NotApplicable,
            effective_weights: criteria.iter().map(|c| (c.name.clone(), c.weight)).collect(),
            rankings: disqualified_list,
        };
    }

    // Build flat raw matrix [m * n] for cache-friendly access
    let mut raw = vec![0.0; m * n];
    for (i, a) in survivors.iter().enumerate() {
        for (j, c) in criteria.iter().enumerate() {
            raw[i * n + j] = *a.scores.get(&c.name).unwrap_or(&0.0);
        }
    }

    // --- STAGE 2: Rank-Reversal-Free Bounds (SPOTIS) ---
    let mut bounds = Vec::with_capacity(n);
    for (j, c) in criteria.iter().enumerate() {
        if let Some((lo, hi)) = c.bounds {
            bounds.push((lo.min(hi), lo.max(hi)));
        } else {
            let mut lo = f64::INFINITY;
            let mut hi = f64::NEG_INFINITY;
            for i in 0..m {
                let val = raw[i * n + j];
                if val < lo { lo = val; }
                if val > hi { hi = val; }
            }
            bounds.push((lo, hi));
        }
    }

    // Compute SPOTIS normalized distances d_ij in [0, 1] (0 = ideal, 1 = worst)
    let mut dist = vec![0.0; m * n];
    compute_spotis_distances(&raw, m, n, criteria, &bounds, &mut dist);

    // --- STAGE 3: Weights (User-provided or Shannon Entropy Fallback) ---
    let has_user_weights = criteria.iter().any(|c| c.weight > EPS);
    let weights = if has_user_weights {
        normalize_weights(criteria)
    } else {
        shannon_entropy_weights(&dist, m, n)
    };

    // --- STAGE 4: VIKOR Compromise Scoring + Stability Proofs ---
    let mut s_vals = vec![0.0; m];
    let mut r_vals = vec![0.0; m];
    let mut q_vals = vec![0.0; m];
    compute_vikor(&dist, m, n, &weights, v, &mut s_vals, &mut r_vals, &mut q_vals);

    // Sort indices by Q ascending (lower is better), tie-break by S, then id
    let mut order: Vec<usize> = (0..m).collect();
    order.sort_by(|&i, &j| {
        q_vals[i]
            .total_cmp(&q_vals[j])
            .then_with(|| s_vals[i].total_cmp(&s_vals[j]))
            .then_with(|| survivors[i].id.cmp(&survivors[j].id))
    });

    // VIKOR Stability Checks (C1: Acceptable Advantage, C2: Acceptable Stability)
    let (status, winners) = if m == 1 {
        (DecisionStatus::ClearWinner, vec![survivors[order[0]].id.clone()])
    } else {
        let best_idx = order[0];
        let second_idx = order[1];
        let dq = 1.0 / ((m - 1) as f64);

        let c1_pass = (q_vals[second_idx] - q_vals[best_idx]) >= dq - EPS;

        let min_s = s_vals.iter().copied().fold(f64::INFINITY, f64::min);
        let min_r = r_vals.iter().copied().fold(f64::INFINITY, f64::min);
        let c2_pass = (s_vals[best_idx] - min_s).abs() <= EPS
            || (r_vals[best_idx] - min_r).abs() <= EPS;

        if c1_pass && c2_pass {
            (DecisionStatus::ClearWinner, vec![survivors[best_idx].id.clone()])
        } else if !c1_pass {
            let compromise: Vec<String> = order
                .iter()
                .filter(|&&idx| (q_vals[idx] - q_vals[best_idx]) < dq - EPS)
                .map(|&idx| survivors[idx].id.clone())
                .collect();
            (DecisionStatus::CompromiseSet, compromise)
        } else {
            // C1 passed, C2 failed -> top 2 form the compromise set
            (
                DecisionStatus::CompromiseSet,
                vec![survivors[best_idx].id.clone(), survivors[second_idx].id.clone()],
            )
        }
    };

    // --- STAGE 5: SMAA-2 Stochastic Robustness (Monte Carlo) ---
    let mut win_counts = vec![0usize; m];
    let mut top2_counts = vec![0usize; m];
    let iters = config.mc_iterations.max(1);

    if m == 1 {
        win_counts[0] = iters;
        top2_counts[0] = iters;
    } else {
        let mut rng = SplitMix64::new(config.rng_seed);
        let mut p_raw = vec![0.0; m * n];
        let mut p_dist = vec![0.0; m * n];
        let mut p_weights = vec![0.0; n];
        let mut p_s = vec![0.0; m];
        let mut p_r = vec![0.0; m];
        let mut p_q = vec![0.0; m];

        for _ in 0..iters {
            // Perturb weights
            let mut w_sum = 0.0;
            for j in 0..n {
                let factor = rng.uniform(
                    (1.0 - config.weight_perturbation_pct).max(0.01),
                    1.0 + config.weight_perturbation_pct,
                );
                let w = (weights[j] * factor).max(EPS);
                p_weights[j] = w;
                w_sum += w;
            }
            for w in &mut p_weights {
                *w /= w_sum;
            }

            // Perturb raw scores by each criterion's uncertainty_pct
            for i in 0..m {
                for (j, c) in criteria.iter().enumerate() {
                    let base = raw[i * n + j];
                    if c.uncertainty_pct > EPS {
                        let noise = rng.uniform(1.0 - c.uncertainty_pct, 1.0 + c.uncertainty_pct);
                        p_raw[i * n + j] = base * noise;
                    } else {
                        p_raw[i * n + j] = base;
                    }
                }
            }

            compute_spotis_distances(&p_raw, m, n, criteria, &bounds, &mut p_dist);
            compute_vikor(&p_dist, m, n, &p_weights, v, &mut p_s, &mut p_r, &mut p_q);

            // Find top 2 (lowest Q) in O(m) without full sort
            let mut first = 0;
            let mut second = 1;
            if p_q[second] < p_q[first] {
                std::mem::swap(&mut first, &mut second);
            }
            for i in 2..m {
                if p_q[i] < p_q[first] {
                    second = first;
                    first = i;
                } else if p_q[i] < p_q[second] {
                    second = i;
                }
            }
            win_counts[first] += 1;
            top2_counts[first] += 1;
            top2_counts[second] += 1;
        }
    }

    let top_win_prob = (win_counts[order[0]] as f64) / (iters as f64);
    let stability_verdict = if top_win_prob > 0.75 {
        StabilityVerdict::Robust
    } else if top_win_prob >= 0.50 {
        StabilityVerdict::Sensitive
    } else {
        StabilityVerdict::Fragile
    };

    // Assemble final rankings
    let mut rankings: Vec<RankedAlternative> = order
        .iter()
        .enumerate()
        .map(|(rank_idx, &i)| RankedAlternative {
            id: survivors[i].id.clone(),
            rank: Some(rank_idx + 1),
            primary_score: q_vals[i],
            vikor_s: Some(s_vals[i]),
            vikor_r: Some(r_vals[i]),
            win_probability: Some((win_counts[i] as f64) / (iters as f64)),
            top2_probability: Some((top2_counts[i] as f64) / (iters as f64)),
            disqualified: false,
            veto_reasons: vec![],
        })
        .collect();

    rankings.extend(disqualified_list);

    let effective_weights = criteria
        .iter()
        .zip(weights.iter())
        .map(|(c, &w)| (c.name.clone(), w))
        .collect();

    DecisionReport {
        engine_used: "COMPLETE_MCDA",
        status,
        winners,
        stability_verdict,
        effective_weights,
        rankings,
    }
}

// ============================================================================
// 6. CORE MATHEMATICAL KERNELS
// ============================================================================

fn normalize_weights(criteria: &[Criterion]) -> Vec<f64> {
    let sum: f64 = criteria.iter().map(|c| c.weight.max(0.0)).sum();
    if sum <= EPS {
        vec![1.0 / (criteria.len() as f64); criteria.len()]
    } else {
        criteria.iter().map(|c| c.weight.max(0.0) / sum).collect()
    }
}

/// SPOTIS normalized distance to ideal point: 0.0 = ideal, 1.0 = anti-ideal
fn compute_spotis_distances(
    raw: &[f64],
    m: usize,
    n: usize,
    criteria: &[Criterion],
    bounds: &[(f64, f64)],
    out_dist: &mut [f64],
) {
    for j in 0..n {
        let (lo, hi) = bounds[j];
        let span = hi - lo;
        let ideal = match criteria[j].direction {
            Direction::Max => hi,
            Direction::Min => lo,
        };
        for i in 0..m {
            let idx = i * n + j;
            if span.abs() < EPS {
                out_dist[idx] = 0.0;
            } else {
                // Clamp to [lo, hi] so better-than-bound values get 0.0 distance
                let clamped = raw[idx].clamp(lo, hi);
                out_dist[idx] = (clamped - ideal).abs() / span;
            }
        }
    }
}

/// Shannon Entropy objective weights from SPOTIS distances
fn shannon_entropy_weights(dist: &[f64], m: usize, n: usize) -> Vec<f64> {
    if m <= 1 {
        return vec![1.0 / (n as f64); n];
    }
    let k = 1.0 / (m as f64).ln();
    let mut div = vec![0.0; n];

    for j in 0..n {
        let mut col_sum = 0.0;
        for i in 0..m {
            // Convert distance [0..1] to benefit [0..1] + EPS to prevent ln(0)
            col_sum += (1.0 - dist[i * n + j]) + EPS;
        }
        let mut entropy = 0.0;
        for i in 0..m {
            let p = ((1.0 - dist[i * n + j]) + EPS) / col_sum;
            entropy -= p * p.ln();
        }
        let e_j = (k * entropy).clamp(0.0, 1.0);
        div[j] = 1.0 - e_j;
    }

    let div_sum: f64 = div.iter().sum();
    if div_sum <= EPS {
        vec![1.0 / (n as f64); n]
    } else {
        div.into_iter().map(|d| d / div_sum).collect()
    }
}

/// VIKOR S (L1), R (L_inf), and Q (Compromise Index)
fn compute_vikor(
    dist: &[f64],
    m: usize,
    n: usize,
    weights: &[f64],
    v: f64,
    out_s: &mut [f64],
    out_r: &mut [f64],
    out_q: &mut [f64],
) {
    let mut s_star = f64::INFINITY;
    let mut s_minus = f64::NEG_INFINITY;
    let mut r_star = f64::INFINITY;
    let mut r_minus = f64::NEG_INFINITY;

    for i in 0..m {
        let mut s = 0.0;
        let mut r = 0.0f64;
        for j in 0..n {
            let wd = weights[j] * dist[i * n + j];
            s += wd;
            if wd > r {
                r = wd;
            }
        }
        out_s[i] = s;
        out_r[i] = r;
        if s < s_star { s_star = s; }
        if s > s_minus { s_minus = s; }
        if r < r_star { r_star = r; }
        if r > r_minus { r_minus = r; }
    }

    let s_span = s_minus - s_star;
    let r_span = r_minus - r_star;

    for i in 0..m {
        let s_term = if s_span.abs() < EPS { 0.0 } else { (out_s[i] - s_star) / s_span };
        let r_term = if r_span.abs() < EPS { 0.0 } else { (out_r[i] - r_star) / r_span };
        out_q[i] = v * s_term + (1.0 - v) * r_term;
    }
}

fn empty_report(engine_used: &'static str) -> DecisionReport {
    DecisionReport {
        engine_used,
        status: DecisionStatus::NoValidAlternatives,
        winners: vec![],
        stability_verdict: StabilityVerdict::NotApplicable,
        effective_weights: vec![],
        rankings: vec![],
    }
}

// ============================================================================
// 7. EXAMPLE EXECUTION
// ============================================================================

fn main() {
    let criteria = vec![
        Criterion {
            name: "cost_usd".into(),
            direction: Direction::Min,
            weight: 0.40,
            bounds: Some((50.0, 500.0)),
            veto_threshold: Some(400.0), // Hard budget cap
            uncertainty_pct: 0.05,
        },
        Criterion {
            name: "latency_ms".into(),
            direction: Direction::Min,
            weight: 0.35,
            bounds: Some((10.0, 200.0)),
            veto_threshold: None,
            uncertainty_pct: 0.10,
        },
        Criterion {
            name: "reliability".into(),
            direction: Direction::Max,
            weight: 0.25,
            bounds: Some((0.90, 1.00)),
            veto_threshold: Some(0.95), // Hard SLA floor
            uncertainty_pct: 0.01,
        },
    ];

    let mk_alt = |id: &str, cost: f64, lat: f64, rel: f64| Alternative {
        id: id.into(),
        scores: HashMap::from([
            ("cost_usd".into(), cost),
            ("latency_ms".into(), lat),
            ("reliability".into(), rel),
        ]),
    };

    let alternatives = vec![
        mk_alt("Vendor_A_Balanced", 180.0, 45.0, 0.992),
        mk_alt("Vendor_B_Fast", 310.0, 18.0, 0.998),
        mk_alt("Vendor_C_Cheap_Unreliable", 90.0, 80.0, 0.930), // Fails reliability veto
        mk_alt("Vendor_D_Overbudget", 450.0, 12.0, 0.999),      // Fails cost veto
    ];

    // 1. Run Minimal Mode
    let min_report = run_mcda(
        &alternatives,
        &criteria,
        &Config { mode: EngineMode::Minimal, ..Default::default() },
    );
    println!("=== {} ===", min_report.engine_used);
    for r in &min_report.rankings {
        println!("Rank {:?} | {:26} | WSM={:.4}", r.rank, r.id, r.primary_score);
    }

    // 2. Run Auto Mode (routes to COMPLETE_MCDA because vetoes/bounds/uncertainty exist)
    let full_report = run_mcda(&alternatives, &criteria, &Config::default());
    println!(
        "\n=== {} | Status: {:?} | Stability: {:?} | Winners: {:?} ===",
        full_report.engine_used, full_report.status, full_report.stability_verdict, full_report.winners
    );
    for r in &full_report.rankings {
        if r.disqualified {
            println!("DISQUALIFIED | {:26} | Reasons: {:?}", r.id, r.veto_reasons);
        } else {
            println!(
                "Rank {:?} | {:26} | Q={:.4} S={:.4} R={:.4} | WinProb={:.1}% Top2={:.1}%",
                r.rank,
                r.id,
                r.primary_score,
                r.vikor_s.unwrap(),
                r.vikor_r.unwrap(),
                r.win_probability.unwrap() * 100.0,
                r.top2_probability.unwrap() * 100.0
            );
        }
    }
}

```