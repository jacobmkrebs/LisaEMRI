# Blind EMRI Parameter Estimation Pipeline

A morphology-based, waveform-agnostic pipeline for recovering the intrinsic parameters of Extreme Mass Ratio Inspirals (EMRIs) from LISA time-frequency data — **upstream of and independent from** matched-filter or MCMC follow-up.

## Overview

EMRIs — compact objects (~5–60 M☉) spiraling into massive Kerr black holes (~10⁵–10⁷ M☉) — are primary science targets for the Laser Interferometer Space Antenna (LISA). Detecting and characterizing them requires parameter estimation across a vast prior volume. This pipeline provides a fast, non-parametric initial estimate of (M, a, μ, e₀) by treating the EMRI signal as a geometric object in the wavelet domain and inverting the Kerr geodesic equations directly.

**Author:** Jacob Krebs

---

## Method

The pipeline operates in eight stages:

| Stage | Description |
|-------|-------------|
| 0 | Waveform generation via FEW (`Pn5AAKWaveform`), trajectory integration (`EMRIInspiral/PN5`) |
| 0.5 | TDI channel extraction (A, E) |
| 1 | WDM time-frequency transform (A+E combined power) |
| 2a | t=0 WDM spectrum → 2fΦ seed for Viterbi tracker |
| 2b | Multi-window FFT (3×3-day segments) → chirp-corrected fR(t=0) by linear extrapolation |
| 3 | Viterbi HMM track extraction → fΦ(t) |
| 4 | Savitzky-Golay smoothing of Viterbi track |
| 5–8 | Parameter recovery via Kerr geodesic inversion + PN5 flux matching (blind path) |

### Core recovery logic

**Spacetime parameters (M, a):** Nelder-Mead minimization of fR residuals. At each trial (M, a), the Kerr geodesic is inverted — `ΩΦ(a, p, e, x) / (2π M_s) = fΦ_obs` — to find p(t), then fR is predicted and compared to the chirp-corrected fR measurement.

**Eccentricity (e₀):** Brentq scan over e ∈ [0.01, 0.69] matching fR at the inferred p₀.

**Secondary mass (μ):** Trajectory-derived `dp/dt` matched to PN5 flux:
```
μ = median( ṗ_traj × M_s / F_PN5(p, e; M_rec, μ=1) ) × M_rec
```

---

## Key design choices

- **fR chirp correction**: Three non-overlapping 3-day FFT windows measure fR at t = 1.5, 4.5, 7.5 days. A linear fit extrapolates to t = 0, correcting the systematic bias from chirp smearing over longer windows. This was the dominant source of error for fast inspirals (high μ or low M).

- **Trajectory-derived ṗ for μ**: Uses the smooth PN5 trajectory `p_t` (not the Viterbi-inverted p, which is noise-limited at WDM resolution) for the `dp/dt` estimate. The `robust_pdot` helper selects between Savitzky-Golay derivative (fast inspirals) and linear regression slope (slow inspirals) adaptively.

- **Fixed p₀/M = 9M**: All grid points use the same dimensionless semi-latus rectum, placing every source at the same orbital geometry (~4.4M above the separatrix for a=0.5, e₀=0.23). This decouples the initial frequency from M, allowing a clean 2D (M, μ) performance characterization.

---

## Performance (zero-noise, 1-year observation)

**2D grid:** M ∈ {3×10⁵, 10⁶, 3×10⁶, 10⁷} M☉ × μ ∈ {5, 15, 30, 60} M☉, a = 0.5, e₀ = 0.23, p₀ = 9M.

| Parameter | Typical error | Notes |
|-----------|--------------|-------|
| M | < 2.5% | Consistent across full M–μ grid |
| a | ~3–4.5% | Uniform systematic; limited by WDM fΦ resolution |
| μ | < 5% | M=3×10⁵, μ=60 SKIP (waveform < 15 days) |
| e₀ | < 0.001% | Essentially exact from fR/fΦ scan |

All errors are zero-noise (best-case bounds). Noise injection and SNR-dependent characterization are planned next steps.

---

## Dependencies

```
few          # Fast EMRI Waveforms (Pn5AAKWaveform, EMRIInspiral, PN5, get_fundamental_frequencies)
WDMWaveletTransforms   # transform_wavelet_time
numpy, scipy, matplotlib
```

Install FEW: https://bhptoolkit.org/FastEMRIWaveforms/

---

## Files

| File | Description |
|------|-------------|
| `EmriBlindPipeline.ipynb` | Main pipeline notebook (Stages 0–8 + 2D grid) |
| `emri_2d_grid_optA.png` | 2D (M, μ) error summary plots |
| `emri_pipeline_summary.png` | Single-injection trajectory-assisted vs blind comparison |

---

## Status

Research prototype. Zero-noise performance validated across a 2D (M, μ) grid. Intended as an upstream complement to segmented MCMC follow-up pipelines. Not yet validated with injected LISA noise.

---

## References

- Babak et al. (2017) — EMRI population models for LISA
- Drasco & Hughes (2006) — Kerr geodesic frequencies
- Chua et al. (2021) — Fast EMRI Waveforms (FEW)
- Cornish & Littenberg (2020) — WDM wavelet transforms for LISA data analysis
