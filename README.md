// ============================================================
// DQSDv2-GPU Setup Script (Arithmetic + Code Exact Form)
// Creates: C:\Users\dillb_lzxy763\DQSDv2-GPU\
// ============================================================

let root = format!("{}/DQSDv2-GPU", std::env::var("HOME").unwrap());

// ─────────────────────────────────────────────────────────────
// CORE SPECTRAL EVOLUTION ARITHMETIC
// ─────────────────────────────────────────────────────────────

// Lie bracket:
// [Z,S]_k = Σ_j (Z_k·S_j − Z_j·S_k) κ_kj

#[inline(always)]
fn lie_bracket(
    z: &[f32; 16],
    s: &[f32; 16],
    kappa: &[f32; 256],
    k: usize,
) -> f32 {
    let mut acc = 0.0;

    for j in 0..16 {
        if j == k { continue; }

        let bracket =
            z[k] * s[j]
          - z[j] * s[k];

        acc += bracket * kappa[k * 16 + j];
    }

    acc
}

// Dissipative flow:
// Zₖ(t+1) = Zₖ + dt([Z,S]ₖ − λZₖ)

#[inline(always)]
fn evolve(
    zk: f32,
    coupling: f32,
    dt: f32,
    lambda: f32,
) -> f32 {
    zk + dt * (coupling - lambda * zk)
}

// EMA memory:
// S ← αS + (1−α)Z

#[inline(always)]
fn ema_memory(
    s: f32,
    z: f32,
    alpha: f32,
) -> f32 {
    alpha * s + (1.0 - alpha) * z
}

// Energy:
// E² = Σ Z²

#[inline(always)]
fn energy(z: &[f32; 16]) -> f32 {
    let mut e2 = 0.0;

    for v in z {
        e2 += *v * *v;
    }

    e2
}

// Stress:
// stress = ||S|| / ||Z||

#[inline(always)]
fn stress(
    z_energy: f32,
    s_energy: f32,
) -> f32 {
    (s_energy.sqrt()) / (z_energy.sqrt() + 1e-8)
}

// Entropy:
// H = −Σ p log(p)

#[inline(always)]
fn entropy(z: &[f32; 16], e2: f32) -> f32 {
    let mut h = 0.0;

    for v in z {
        let p = (*v * *v) / (e2 + 1e-8);

        if p > 1e-8 {
            h -= p * p.ln();
        }
    }

    h
}

// Temporal anti-ghosting gate:
// w = 1 / (1 + (stress / τ)^2)

#[inline(always)]
fn temporal_weight(
    stress: f32,
    tau: f32,
) -> f32 {
    1.0 / (1.0 + (stress / tau).powi(2))
}

// Frame generation confidence:
// stable manifold → confidence ≈ 1

#[inline(always)]
fn frame_gen_confidence(
    ghost: u8,
    stress: f32,
) -> f32 {
    let mut fg = 1.0;

    if ghost != 0 {
        fg *= 0.25;
    }

    if stress > 0.5 {
        fg *= (1.0 - stress).clamp(0.0, 1.0);
    }

    fg
}

// Containment:
// if ||Z||² > U² → reset manifold

#[inline(always)]
fn containment(
    z: &mut [f32; 16],
    e2: f32,
    umax: f32,
) -> bool {
    if e2 > umax * umax {
        for v in z.iter_mut() {
            *v = 0.0;
        }
        true
    } else {
        false
    }
}

// ─────────────────────────────────────────────────────────────
// GPU EXECUTION ORDER
// ─────────────────────────────────────────────────────────────
//
// 1. Lie-Bracket Kernel
// 2. EMA Memory Kernel
// 3. Containment + Diagnostics Kernel
// 4. Temporal / Frame Quality Kernel
//
// Deterministic ordering is mandatory.
// No hidden feedback paths permitted.
// Diagnostics remain observational only.
// ─────────────────────────────────────────────────────────────
