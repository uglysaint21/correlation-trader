# Research Protocol 02: Dynamic Currency-Factor Model Estimation and Forecast Evaluation

**Status:** Proposed preregistration; execution not yet authorized by available broker data  
**Date:** 10 November 2026  
**Depends on:** Research Note 05 and Research Protocol 01  
**Purpose:** Define the empirical comparison of static constrained GLS and dynamic Kalman currency-factor models before examining predictive performance.

---

## 1. Objective

Determine whether a constrained state-space/Kalman model improves on static currency-factor estimation and whether any estimated factor dynamics provide incremental out-of-sample predictive information.

This protocol separates three possible outcomes:

1. **Descriptive success:** factors provide a stable contemporaneous decomposition.
2. **Filtering success:** dynamic filtering improves estimation or uncertainty quantification relative to static GLS.
3. **Forecasting success:** filtered factors predict future returns better than simple baselines.

Only the third outcome can support a directional factor-based strategy hypothesis. None alone establishes profitability.

## 2. Required data and frozen manifest

Use the actual FXPesa MT5 account and terminal catalogue, not only the public product list.

Freeze:
- Exact broker/server identity (non-secret).
- MT5 terminal and Python package versions.
- Exact symbols, verified base/quote currencies and contract specifications.
- Tick/bar history start and end dates by symbol.
- Sampling frequency and UTC boundary.
- Bid, ask, mid construction and quote-age threshold.
- Market closures, rollover, holiday and missing-observation policy.
- Universe inclusion/exclusion reasons.
- Dataset hashes, retrieval timestamp and transformation version.
- Any changes to the universe over time.

If symbol mappings, timestamps or price fields are ambiguous, stop and repair the data contract before estimation.

## 3. Pre-analysis graph checks

For each timestamp, record:
- Number of observed edges and active currencies.
- Connected-component count.
- \(\operatorname{rank}(B_t)\) and \(\operatorname{rank}(B_tC)\).
- Fraction of timestamps with all nine contrast directions identified.
- Time spent in disconnected/partially observed states.
- Quote age, stale quote counts and triangle residuals.
- Residual graph-cycle diagnostics.

Do not impute missing pair returns to force connectivity. For partially observed timestamps, use the corresponding rows of \(H_t\) and covariance submatrix. Report when the state is only partially observed.

## 4. Models to compare

All models use the same training windows, observed panel, sampling schedule and evaluation timestamps.

| ID | Model | Purpose |
|---|---|---|
| S0 | Per-timestamp equal-weight constrained OLS | Simple descriptive benchmark |
| S1 | Per-timestamp constrained GLS with diagonal/shrinkage R | Static uncertainty-aware benchmark |
| K0 | Kalman with A = 0 | No-persistence dynamic filtering baseline |
| K1 | Kalman with A = rho I | Scalar persistence |
| K2 | Kalman with diagonal A | Coordinate-specific persistence sensitivity |
| K3 | Kalman with full stable A and regularization | High-complexity challenger; only if justified |
| P0 | PCA benchmark fitted within each training window | Unconstrained statistical benchmark |

PCA factors must not be relabelled as named currency factors. K2 is basis-dependent; report the fixed contrast basis and test sensitivity to alternative orthonormal bases. K3 should be deferred if the data do not support stable estimation.

## 5. Parameter estimation

- Estimate \(A,\Omega,R\) only within training data.
- Use innovation maximum likelihood or EM, with initialization and convergence criteria recorded.
- Enforce positive-semidefinite process covariance, positive-definite observation covariance and stable transition matrix where the stationary-return model is used.
- Begin with diagonal/shrinkage covariance structures; do not estimate unrestricted 37-by-37 observation covariance without a justified sample-size and regularization analysis.
- If using robust or Student-t variants, treat them as separate predeclared sensitivity experiments.
- Refit only at scheduled walk-forward boundaries.
- For out-of-sample state features, use the forward filter; never use smoothed states.

## 6. Evaluation questions and metrics

### A. Reconstruction and factor estimation

Report:
- Pair-return reconstruction error.
- Cross-sectional explained sum-of-squares with denominator specified.
- Factor estimate and covariance stability.
- Sensitivity to pair removal, graph changes and synchronization thresholds.
- Basis invariance of fitted pair returns.
- Residual distribution, autocorrelation, cross-pair dependence and conditional heteroskedasticity.

Do not call cross-sectional fit a time-series forecast score.

### B. Filtering quality

Compare static GLS and Kalman filtered estimates using:
- Innovation log-likelihood on held-out observations.
- Innovation mean, variance, autocorrelation and cross-correlation.
- Calibration of stated factor uncertainty where an appropriate proxy/benchmark is available.
- Stability under missing edges and quote-age changes.

Because latent factors are not directly observed, do not claim absolute factor estimation error without an independently justified proxy or simulation study. A better likelihood alone is not sufficient.

### C. Forecasting

Forecast future pair returns using:
- Zero-return benchmark.
- Historical mean estimated in training only.
- Simple pairwise AR baseline.
- Static-factor baseline with no predictive state dynamics.
- Kalman factor forecast from each candidate transition model.

Evaluate:
- Out-of-sample log score / CRPS for predictive distributions where available.
- Mean squared error and MAE as secondary point-forecast metrics.
- Directional metrics only as supplementary diagnostics.
- Calibration and forecast uncertainty.
- Results by pair, currency, session, volatility bucket and chronological fold.

All forecasts must use information available at the decision timestamp. Evaluate both one-step and a small, predeclared set of economically relevant horizons. Avoid tuning horizon selection against the final holdout.

## 7. Walk-forward design and leakage controls

- Use chronological train/validation/test partitions.
- Re-estimate preprocessing, missing-data rules if learned, covariance, factor dynamics, shrinkage, and any model-selection choice within the training window.
- Select hyperparameters on validation data only.
- Keep a final untouched chronological holdout.
- If target returns overlap across forecast horizons, purge training labels whose outcome intervals overlap test intervals and apply an embargo appropriate to the dependence structure.
- Never use full-sample PCA, full-sample standardization, smoothed states or future-selected universes in historical decisions.
- Record every tested model and failed variant for multiple-testing assessment.

The final holdout must not be revisited for model tuning.

## 8. Robustness and sensitivity

Predeclare a small grid for:
- Sampling interval.
- Quote-age/synchronization tolerance.
- Training-window length.
- Refit cadence.
- Diagonal versus shrinkage observation covariance.
- Static versus dynamic model complexity.
- Sessions and calendar subperiods.

Stress data gaps, missing edges, isolated stale symbols and abrupt changes in residual variance. Test whether model parameters and forecasts are dominated by a small subset of pairs or by USD exposure.

## 9. Go/no-go gates

**Gate A — Data:** actual symbol mappings and historical coverage verified; timestamps, quote orientation and bid/ask data pass Protocol 01.

**Gate B — Algebra:** graph rank and contrast basis checks pass; fitted returns are invariant under equivalent basis transformations.

**Gate C — Numerical:** filter likelihood, covariance updates and missing-observation handling are stable; no hidden singular-matrix workarounds.

**Gate D — Statistical:** innovations and parameter estimates are sufficiently stable for the stated model; complexity improves held-out diagnostics over simpler alternatives.

**Gate E — Predictive:** factor forecasts beat zero/mean/AR baselines on untouched chronological data with calibrated uncertainty and robust fold performance.

**Gate F — Economic:** only after Gate E, evaluate strategy construction with executable bid/ask prices, commissions, financing, slippage, portfolio constraints and multiple-testing adjustments.

Failure at a gate means stop, revise the data/model specification or record a no-go result. It does not justify widening the parameter search until something appears profitable.

## 10. Required report

1. Frozen data manifest and symbol audit.
2. Graph rank/connectivity timeline.
3. Model equations, basis and covariance parameterization.
4. Estimation method, initialization, convergence and software versions.
5. Static versus dynamic comparison.
6. Filtered-state and uncertainty diagnostics.
7. Out-of-sample forecasts and baselines.
8. Robustness and multiple-testing disclosure.
9. Gate-by-gate decision and unresolved assumptions.
10. Reproducibility details and all exclusions.

## 11. Current status

No FXPesa MT5 data has been analysed in this protocol. No model parameters have been estimated, no forecast has been evaluated and no strategy profitability claim has been made.

## Research log

| Date | Entry |
|---|---|
| 2026-11-10 | Preregistered the empirical comparison of static constrained factors and dynamic Kalman models. Pending actual FXPesa MT5 symbol/history audit and frozen dataset manifest. |
