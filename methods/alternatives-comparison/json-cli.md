# M04 — MCDA JSON CLI reference

Status: supplied reference, not compiled or independently validated by this repository editor. Contains both MINIMAL_MCDA (M04-A01) and COMPLETE_MCDA (M04-A02). The supplied source text below is preserved, with line endings normalized to LF. See [selection and adaptation notes](README.md) before use.

---

Here is the minimal Cargo project with `serde` JSON stdin/stdout support. Disqualified alternatives now use `Option<f64>` for `primary_score` so JSON serialization never emits invalid `Infinity` floats, and optional fields use `#[serde(default)]` so AI agents can send minimal payloads.

## 1. `Cargo.toml`

```toml
[package]
name = "mcda-cli"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[profile.release]
opt-level = 3
lto = true
strip = true

```

---

## 2. `src/main.rs`

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::io::{self, Read};
use std::process;

const EPS: f64 = 1e-12;

// ============================================================================
// 1. JSON INPUT / OUTPUT CONTRACT
// ============================================================================

#[derive(Debug, Deserialize)]
pub struct DecisionRequest {
    pub alternatives: Vec<Alternative>,
    pub criteria: Vec<Criterion>,
    #[serde(default)]
    pub config: Config,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum Direction {
    Max,
    Min,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum EngineMode {
    Minimal,
    Complete,
    Auto,
}

#[derive(Debug, Clone, Deserialize)]
pub struct Criterion {
    pub name: String,
    pub direction: Direction,
    #[serde(default)]
    pub weight: f64,
    #[serde(default)]
    pub bounds: Option<(f64, f64)>,
    #[serde(default)]
    pub veto_threshold: Option<f64>,
    #[serde(default)]
    pub uncertainty_pct: f64,
}

#[derive(Debug, Clone, Deserialize)]
pub struct Alternative {
    pub id: String,
    pub scores: HashMap<String, f64>,
}

#[derive(Debug, Clone, Deserialize)]
#[serde(default)]
pub struct Config {
    pub mode: EngineMode,
    pub vikor_v: f64,
    pub mc_iterations: usize,
    pub weight_perturbation_pct: f64,
    pub rng_seed: u64,
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

#[derive(Debug, Clone, PartialEq, Eq, Serialize)]
#[serde(rename_all = "SCREAMING_SNAKE_CASE")]
pub enum DecisionStatus {
    ClearWinner,
    CompromiseSet,
    NoValidAlternatives,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize)]
#[serde(rename_all = "SCREAMING_SNAKE_CASE")]
pub enum StabilityVerdict {
    Robust,
    Sensitive,
    Fragile,
    NotApplicable,
}

#[derive(Debug, Clone, Serialize)]
pub struct RankedAlternative {
    pub id: String,
    pub rank: Option<usize>,
    pub primary_score: Option<f64>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub vikor_s: Option<f64>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub vikor_r: Option<f64>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub win_probability: Option<f64>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub top2_probability: Option<f64>,
    pub disqualified: bool,
    #[serde(skip_serializing_if = "Vec::is_empty")]
    pub veto_reasons: Vec<String>,
}

#[derive(Debug, Clone, Serialize)]
pub struct DecisionReport {
    pub engine_used: &'static str,
    pub status: DecisionStatus,
    pub winners: Vec<String>,
    pub stability_verdict: StabilityVerdict,
    pub effective_weights: HashMap<String, f64>,
    pub rankings: Vec<RankedAlternative>,
}

#[derive(Serialize)]
struct ErrorResponse {
    error: String,
}

// ============================================================================
// 2. CLI ENTRYPOINT (stdin -> JSON -> stdout)
// ============================================================================

fn main() {
    let mut input = String::new();
    if let Err(e) = io::stdin().read_to_string(&mut input) {
        fail(&format!("Failed to read stdin: {e}"));
    }

    if input.trim().is_empty() {
        fail("Empty stdin: expected JSON DecisionRequest payload");
    }

    let req: DecisionRequest = match serde_json::from_str(&input) {
        Ok(r) => r,
        Err(e) => fail(&format!("Invalid JSON input: {e}")),
    };

    let report = run_mcda(&req.alternatives, &req.criteria, &req.config);

    match serde_json::to_string_pretty(&report) {
        Ok(json) => println!("{json}"),
        Err(e) => fail(&format!("Failed to serialize output: {e}")),
    }
}

fn fail(msg: &str) -> ! {
    let err = serde_json::to_string(&ErrorResponse {
        error: msg.to_string(),
    })
    .unwrap_or_else(|_| format!(r#"{{"error":"{}"}}"#, msg));
    eprintln!("{err}");
    process::exit(1);
}

// ============================================================================
// 3. DETERMINISTIC PRNG (SplitMix64)
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

    #[inline]
    fn next_f64(&mut self) -> f64 {
        ((self.next_u64() >> 11) as f64) * (1.0 / ((1u64 << 53) as f64))
    }

    #[inline]
    fn uniform(&mut self, low: f64, high: f64) -> f64 {
        low + (high - low) * self.next_f64()
    }
}

// ============================================================================
// 4. AUTO-ROUTER & ENGINES
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

pub fn minimal_mcda(alternatives: &[Alternative], criteria: &[Criterion]) -> DecisionReport {
    if alternatives.is_empty() || criteria.is_empty() {
        return empty_report("MINIMAL_MCDA");
    }

    let n = criteria.len();
    let weights = normalize_weights(criteria);

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
                primary_score: Some(total),
                vikor_s: None,
                vikor_r: None,
                win_probability: None,
                top2_probability: None,
                disqualified: false,
                veto_reasons: vec![],
            }
        })
        .collect();

    ranked.sort_by(|a, b| {
        b.primary_score
            .unwrap_or(f64::NEG_INFINITY)
            .total_cmp(&a.primary_score.unwrap_or(f64::NEG_INFINITY))
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

    // Stage 1: Non-Compensatory Veto Gate
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
                primary_score: None,
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

    let mut raw = vec![0.0; m * n];
    for (i, a) in survivors.iter().enumerate() {
        for (j, c) in criteria.iter().enumerate() {
            raw[i * n + j] = *a.scores.get(&c.name).unwrap_or(&0.0);
        }
    }

    // Stage 2: SPOTIS Fixed-Bounds Normalization
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

    let mut dist = vec![0.0; m * n];
    compute_spotis_distances(&raw, m, n, criteria, &bounds, &mut dist);

    // Stage 3: User Weights or Shannon Entropy Fallback
    let has_user_weights = criteria.iter().any(|c| c.weight > EPS);
    let weights = if has_user_weights {
        normalize_weights(criteria)
    } else {
        shannon_entropy_weights(&dist, m, n)
    };

    // Stage 4: VIKOR Compromise Scoring + Stability Proofs
    let mut s_vals = vec![0.0; m];
    let mut r_vals = vec![0.0; m];
    let mut q_vals = vec![0.0; m];
    compute_vikor(&dist, m, n, &weights, v, &mut s_vals, &mut r_vals, &mut q_vals);

    let mut order: Vec<usize> = (0..m).collect();
    order.sort_by(|&i, &j| {
        q_vals[i]
            .total_cmp(&q_vals[j])
            .then_with(|| s_vals[i].total_cmp(&s_vals[j]))
            .then_with(|| survivors[i].id.cmp(&survivors[j].id))
    });

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
            (
                DecisionStatus::CompromiseSet,
                vec![survivors[best_idx].id.clone(), survivors[second_idx].id.clone()],
            )
        }
    };

    // Stage 5: SMAA-2 Stochastic Robustness
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

    let mut rankings: Vec<RankedAlternative> = order
        .iter()
        .enumerate()
        .map(|(rank_idx, &i)| RankedAlternative {
            id: survivors[i].id.clone(),
            rank: Some(rank_idx + 1),
            primary_score: Some(q_vals[i]),
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
// 5. MATHEMATICAL KERNELS
// ============================================================================

fn normalize_weights(criteria: &[Criterion]) -> Vec<f64> {
    let sum: f64 = criteria.iter().map(|c| c.weight.max(0.0)).sum();
    if sum <= EPS {
        vec![1.0 / (criteria.len() as f64); criteria.len()]
    } else {
        criteria.iter().map(|c| c.weight.max(0.0) / sum).collect()
    }
}

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
                let clamped = raw[idx].clamp(lo, hi);
                out_dist[idx] = (clamped - ideal).abs() / span;
            }
        }
    }
}

fn shannon_entropy_weights(dist: &[f64], m: usize, n: usize) -> Vec<f64> {
    if m <= 1 {
        return vec![1.0 / (n as f64); n];
    }
    let k = 1.0 / (m as f64).ln();
    let mut div = vec![0.0; n];

    for j in 0..n {
        let mut col_sum = 0.0;
        for i in 0..m {
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
        effective_weights: HashMap::new(),
        rankings: vec![],
    }
}

```

---

## 3. Build & Invoke via `stdin` / `stdout`

```bash
cargo build --release

```

Pipe a JSON request directly into the binary:

```bash
./target/release/mcda-cli << 'EOF'
{
  "alternatives": [
    { "id": "Vendor_A", "scores": { "cost": 180, "latency_ms": 45, "reliability": 0.992 } },
    { "id": "Vendor_B", "scores": { "cost": 310, "latency_ms": 18, "reliability": 0.998 } },
    { "id": "Vendor_C", "scores": { "cost": 90,  "latency_ms": 80, "reliability": 0.930 } }
  ],
  "criteria": [
    { "name": "cost",        "direction": "min", "weight": 0.40, "bounds": [50, 500], "veto_threshold": 400, "uncertainty_pct": 0.05 },
    { "name": "latency_ms",  "direction": "min", "weight": 0.35, "bounds": [10, 200], "uncertainty_pct": 0.10 },
    { "name": "reliability", "direction": "max", "weight": 0.25, "bounds": [0.90, 1.00], "veto_threshold": 0.95, "uncertainty_pct": 0.01 }
  ]
}
EOF

```

