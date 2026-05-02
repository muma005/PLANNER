

## CORE PHILOSOPHY

**The evaluation system exists to prevent self-deception.**

Most competition failures happen not because the method is bad, but because the builder convinced themselves it was good using weak tests. This system makes that impossible.

Three principles:

1. **Assume failure until proven otherwise.** Every test starts with verdict = FAIL. The system must earn PASS.
2. **No partial credit on critical checks.** Either row alignment is preserved or it isn't. Either predictions depend on Z or they don't. Binary gates.
3. **Quantitative thresholds set BEFORE seeing results.** We don't get to move the goalposts after running the system.

---

## EVALUATION ARCHITECTURE

The system has three tiers, run sequentially. Failing any gate in Tier 1 means the system is fundamentally broken — don't even look at Tier 2.

### TIER 1: STRUCTURAL VALIDITY (Binary Gates)

These are pass/fail. No gradations. If any gate fails, the entire reconstruction is **REJECTED**.

**Gate 1.1 — Shape Conformance**
- X_hat.shape must equal (N_test, D) exactly
- D must match X_train.shape[1]
- Threshold: exact match, zero tolerance

**Gate 1.2 — Finite Values**
- Every element of X_hat must be finite (not NaN, not Inf, not -Inf)
- Threshold: zero NaN/Inf allowed. Even one is a FAIL.

**Gate 1.3 — Permutation Equivariance (Row Alignment)**
- Generate 5 random permutations of Z_test
- For each permutation π: compute X_hat_π = reconstruct(Z_test[π])
- Check: X_hat_π[i, :] == X_hat[π[i], :] for all i, all features
- Threshold: max absolute difference < 1e-10 across all 5 permutations
- This is the HARDEST structural test. If the pipeline uses any global operation (sorting, batch normalization, ranking), this gate catches it.

**Gate 1.4 — Z-Dependence (Permutation Test)**
- Shuffle Z_test randomly (breaking the Z→row correspondence)
- Compute X_hat_shuffled = reconstruct(Z_test_shuffled)
- Compute divergence = mean L2 distance between X_hat and X_hat_shuffled per row
- Threshold: divergence > 1e-6
- If predictions don't change when Z is shuffled, the system is outputting constants. FAIL.

**Gate 1.5 — Non-Degeneracy**
- For each feature j: compute var(X_hat[:, j])
- Count features where var < 1e-10
- Threshold: at most 50% of features can be degenerate
- If more than half the features are constant predictions, the system has collapsed. FAIL.

**Gate 1.6 — Determinism**
- Run reconstruct() twice with identical inputs and seed
- Check: outputs are bit-for-bit identical
- Threshold: max absolute difference = 0.0 exactly

---

### TIER 2: QUANTITATIVE PERFORMANCE (Scored Gates)

These produce numerical scores. Each has a HARD threshold (must pass) and a QUALITY threshold (indicates strong performance).

**Gate 2.1 — Holdout SRMSE**

The most important single number.

*Protocol:*
- Split training data: 80% train, 20% holdout (deterministic split by index)
- Train reconstruction on 80%
- Predict holdout from holdout Z values
- Compute SRMSE: mean across features of (RMSE_j / std_j)

*Thresholds:*

| SRMSE | Verdict |
|---|---|
| > 1.0 | **FAIL** — worse than predicting the mean. System is actively harmful. |
| 0.95 – 1.0 | **MARGINAL FAIL** — statistically indistinguishable from constant prediction. |
| 0.8 – 0.95 | **WEAK PASS** — system extracts minimal signal. Likely insufficient for competition. |
| 0.5 – 0.8 | **PASS** — meaningful reconstruction. Competitive depending on other entries. |
| 0.3 – 0.5 | **STRONG PASS** — substantial reconstruction. Likely competitive. |
| < 0.3 | **EXCELLENT** — high-fidelity reconstruction. Top contender. |

*Hard threshold:* SRMSE < 0.95. Above this, the system has FAILED.

**Gate 2.2 — Feature-Level SRMSE Distribution**

Global SRMSE can hide feature-level disasters.

*Protocol:*
- Compute SRMSE_j for each feature j on the holdout
- Compute: fraction of features with SRMSE_j > 0.99 ("dead features")
- Compute: fraction of features with SRMSE_j < 0.5 ("well-reconstructed")

*Thresholds:*

| Metric | Hard Threshold | Quality Threshold |
|---|---|---|
| Dead features (SRMSE > 0.99) | < 80% | < 30% |
| Well-reconstructed (SRMSE < 0.5) | > 5% (at least something works) | > 50% |

**Gate 2.3 — Baseline Superiority Test**

The system must provably beat trivial baselines.

*Baselines:*
1. **Mean baseline:** X_hat = tile(mean(X_train), N_test) — predict the training mean for every row
2. **Noise baseline:** X_hat = random samples from N(mean(X_train), std(X_train)) per feature — distribution matching
3. **Linear baseline:** X_hat[:, j] = a_j * Z + b_j — simple linear regression per feature

*Protocol:*
- Compute holdout SRMSE for each baseline
- Compute improvement = (baseline_SRMSE - system_SRMSE) / baseline_SRMSE

*Thresholds:*

| Comparison | Hard Threshold | Quality Threshold |
|---|---|---|
| vs. Mean baseline | improvement > 0% (must beat it) | improvement > 20% |
| vs. Noise baseline | improvement > 0% | improvement > 30% |
| vs. Linear baseline | improvement ≥ 0% (at least tie) | improvement > 10% |

If the system can't beat predicting the mean, it's worthless. If it can't beat simple linear regression, the adaptive complexity machinery adds nothing.

**Gate 2.4 — Per-Row Error Distribution**

Good average performance can mask catastrophic per-row failures.

*Protocol:*
- Compute per-row reconstruction error: e_i = ||X_hat[i] - X[i]|| / ||X[i]|| (relative error per row)
- Compute percentiles: P50, P90, P95, P99, max

*Thresholds:*

| Percentile | Hard Threshold | Quality Threshold |
|---|---|---|
| P95 | < 5.0 (no catastrophic rows) | < 2.0 |
| P99 | < 10.0 | < 3.0 |
| Max | < 50.0 | < 5.0 |

If the P99 error is > 10x average, something is structurally wrong — likely extrapolation blow-up or model instability on edge cases.

---

### TIER 3: ROBUSTNESS & GENERALIZATION (Stress Tests)

These test whether the system will survive on unseen data. They simulate adversarial conditions.

**Test 3.1 — Cross-Validation Stability**

*Protocol:*
- Run 5-fold CV on the full training data (not the holdout split — fresh folds)
- Compute SRMSE on each fold
- Compute: mean, std, max across folds

*Threshold:*
- std(fold_SRMSEs) / mean(fold_SRMSEs) < 0.3 (coefficient of variation)
- If performance varies wildly across folds, the system is unstable. FAIL.

**Test 3.2 — Noise Injection Resilience**

*Protocol:*
- Add Gaussian noise to Z_test: Z_noisy = Z_test + N(0, σ_noise) where σ_noise = 0.1 × std(Z_train)
- Reconstruct from Z_noisy
- Compute SRMSE degradation = SRMSE_noisy - SRMSE_clean

*Threshold:*
- Degradation < 0.2 (a little noise shouldn't destroy reconstruction)
- If degradation > 0.5, the system is fragile and will fail on real evaluation data with any measurement noise.

**Test 3.3 — Extrapolation Survival**

*Protocol:*
- Identify holdout rows where Z is in the top/bottom 10% of training Z range (edge cases)
- Compute SRMSE on these edge rows only
- Compare to SRMSE on middle 80% rows

*Threshold:*
- Edge SRMSE < 2 × Middle SRMSE
- If edge-case performance is drastically worse, the system extrapolates badly.

**Test 3.4 — Synthetic Ground Truth Test**

This is the GOLD STANDARD test — the only test where we know the true answer.

*Protocol:*
- Create 3 synthetic datasets with known forward maps:

  *Synthetic A — Linear:* X ~ N(0, Σ) with correlated features, Z = Xw + ε. We know X, so we can compute true SRMSE.

  *Synthetic B — Nonlinear monotonic:* Z = tanh(Xw) + ε. Invertible in principle.

  *Synthetic C — Lossy:* Z = X₁ + X₂ (sum of two features, 3+ features total). Provably non-invertible.

- Run reconstruct() on each
- Compute SRMSE against known X

*Expected results:*
- Synthetic A: SRMSE should be low (< 0.7). If not, the system can't even handle the easiest case.
- Synthetic B: SRMSE should be moderate (< 0.85). Nonlinear model should activate.
- Synthetic C: SRMSE should be moderate-high. System should correctly identify unpredictable features and predict conditional means.

*Threshold:*
- Synthetic A SRMSE < 0.7: **HARD requirement.** If a known-linear problem can't be solved, the pipeline is broken.

**Test 3.5 — Feature Correlation Preservation**

*Protocol:*
- Compute correlation matrix of X_train: C_train = corr(X_train)
- Compute correlation matrix of X_hat (on holdout): C_hat = corr(X_hat)
- Compute Frobenius distance: ||C_train - C_hat||_F / D

*Threshold:*
- Distance < 0.5 per feature (normalized)
- The reconstruction should preserve feature relationships, not just marginals

---

## COMPOSITE SCORING

After all gates and tests, compute a single summary:

```
FINAL VERDICT LOGIC:

IF any Tier 1 gate FAILS:
    VERDICT = "REJECTED — structural failure"
    STOP. Do not compute further.

IF holdout SRMSE > 0.95:
    VERDICT = "REJECTED — no reconstruction signal"
    STOP.

IF system does not beat mean baseline:
    VERDICT = "REJECTED — worse than trivial"
    STOP.

IF synthetic A SRMSE > 0.7:
    VERDICT = "REJECTED — can't solve known-easy problem"
    STOP.

# If we get here, system has basic validity.
# Now score quality.

quality_score = 0

# SRMSE contribution (0-40 points)
IF holdout SRMSE < 0.3:    quality_score += 40
ELIF holdout SRMSE < 0.5:  quality_score += 30
ELIF holdout SRMSE < 0.7:  quality_score += 20
ELIF holdout SRMSE < 0.85: quality_score += 10
ELSE:                       quality_score += 5

# Baseline superiority (0-20 points)
IF beats linear by > 10%:  quality_score += 20
ELIF beats linear:          quality_score += 10
ELSE:                       quality_score += 0

# Feature coverage (0-15 points)
well_reconstructed_frac = fraction with SRMSE_j < 0.5
quality_score += int(15 * well_reconstructed_frac)

# Robustness (0-15 points)
IF CV stability < 0.15:    quality_score += 15
ELIF CV stability < 0.3:   quality_score += 10
ELSE:                       quality_score += 0

# Noise resilience (0-10 points)
IF noise_degradation < 0.1:  quality_score += 10
ELIF noise_degradation < 0.2: quality_score += 5
ELSE:                          quality_score += 0

# Final verdict
IF quality_score >= 80:
    VERDICT = "EXCELLENT — high confidence in competition viability"
ELIF quality_score >= 60:
    VERDICT = "STRONG — competitive, minor improvements possible"
ELIF quality_score >= 40:
    VERDICT = "ADEQUATE — functional but vulnerable"
ELIF quality_score >= 20:
    VERDICT = "WEAK — substantial improvement needed"
ELSE:
    VERDICT = "POOR — rethink approach"
```

---

## REPORTING FORMAT

The evaluation system outputs a structured report:

```
================================================================
        INVERSE RECONSTRUCTION EVALUATION REPORT
================================================================

TIER 1: STRUCTURAL VALIDITY
  [PASS] Shape conformance:       (1000, 12) == (1000, 12)
  [PASS] Finite values:           0 NaN, 0 Inf
  [PASS] Permutation equivariance: max_diff = 0.0
  [PASS] Z-dependence:            divergence = 0.0342
  [PASS] Non-degeneracy:          2/12 degenerate features (16.7%)
  [PASS] Determinism:             bit-identical across 2 runs

  ► TIER 1 VERDICT: ALL GATES PASSED

TIER 2: QUANTITATIVE PERFORMANCE
  Holdout SRMSE:          0.6234  [PASS — meaningful reconstruction]
  Feature breakdown:
    Dead (>0.99):         1/12 (8.3%)   [PASS]
    Well-reconstructed:   7/12 (58.3%)  [QUALITY PASS]
  Baseline comparison:
    vs Mean:              -34.2%  improvement  [PASS]
    vs Noise:             -41.7%  improvement  [PASS]
    vs Linear:            -8.3%   improvement  [PASS]
  Per-row error:
    P50=0.41  P90=1.12  P95=1.54  P99=2.31  Max=3.87

  ► TIER 2 VERDICT: PASSED (SRMSE in competitive range)

TIER 3: ROBUSTNESS
  CV stability:           std/mean = 0.089  [STABLE]
  Noise resilience:       degradation = 0.073  [RESILIENT]
  Extrapolation:          edge/middle = 1.43  [ACCEPTABLE]
  Synthetic A (linear):   SRMSE = 0.312  [PASS]
  Synthetic B (nonlinear): SRMSE = 0.587  [PASS]
  Synthetic C (lossy):    SRMSE = 0.823  [EXPECTED]
  Correlation preservation: 0.218/feature  [ACCEPTABLE]

  ► TIER 3 VERDICT: ROBUST

================================================================
QUALITY SCORE:        72 / 100
FINAL VERDICT:        STRONG
================================================================
Feature-level detail:
  Feature 0:  r=0.89  MI=0.34  model=poly_2   SRMSE=0.21  ██░░░
  Feature 1:  r=0.76  MI=0.28  model=ridge     SRMSE=0.38  ███░░
  Feature 2:  r=0.82  MI=0.31  model=poly_3    SRMSE=0.29  ██░░░
  ...
  Feature 11: r=0.03  MI=0.01  model=z_const   SRMSE=0.99  █████
================================================================
```

---

## WHAT THIS SYSTEM CATCHES THAT WEAK EVALUATION MISSES

| Self-deception pattern | How this system catches it |
|---|---|
| "My SRMSE looks good" but it's just predicting the mean | Gate 2.3 — must beat mean baseline |
| "It works on training data" but overfits | Gate 2.1 — holdout-only evaluation |
| "Average error is low" but some rows are catastrophic | Gate 2.4 — P95/P99 error bounds |
| "Predictions look reasonable" but don't actually use Z | Gate 1.4 — permutation test |
| "It works on this dataset" but won't generalize | Test 3.4 — synthetic ground truth on different forward maps |
| "Features look correlated" but the correlation is fabricated | Test 3.5 — correlation preservation check |
| "Model selection chose complex models" but they don't help | Gate 2.3 — must beat simple linear baseline |
| "The system runs" but produces different results each time | Gate 1.6 — determinism check |

---

Ready to implement this as a self-contained evaluation module? Once we have data, this is the first thing that runs — before we celebrate anything.
