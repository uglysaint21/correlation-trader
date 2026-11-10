# Research Note 05: Dynamic Currency-Factor Models — Mathematical Specification

**Status:** Mathematical research specification; empirical performance not yet tested  
**Date:** 10 November 2026  
**Depends on:** Research Notes 01–04 and Research Protocol 01  
**Scope:** Derive and specify an identifiable state-space model for latent G10 currency returns, its Kalman filter, uncertainty, missing-data behaviour, parameter estimation and falsifiable research tests.  
**Decision status:** Candidate model only. This note does not approve a trading signal or implementation.

---

## 1. Executive summary

This investigation develops a constrained dynamic factor model for synchronized FX pair returns. The currency-incidence structure is known from each instrument's verified base/quote convention; it is not a free loading matrix. The unknown object is the time-varying latent currency-return vector and its dynamics.

The recommended first model is:

\[
y_t = B_t C x_t + \varepsilon_t,
\qquad
x_t = A x_{t-1}+u_t,
\]

where:

- \(y_t\in\mathbb R^{M_t}\) is the vector of observed pair log returns at time \(t\);
- \(B_t\in\mathbb R^{M_t\times N}\) is the observed currency-incidence matrix;
- \(C\in\mathbb R^{N\times(N-1)}\) is a fixed orthonormal contrast basis satisfying \(C^\top\mathbf 1=0\);
- \(x_t\in\mathbb R^{N-1}\) are reduced-coordinate latent currency returns;
- \(A\) is a stable transition matrix;
- \(u_t\) is the factor-state innovation with covariance \(\Omega\);
- \(\varepsilon_t\) is the pair-specific observation residual, combining unexplained pair movement and quote/microstructure error at this stage.

The first empirical comparison should be deliberately parsimonious:

1. Cross-sectional constrained least squares at each timestamp.
2. Kalman model with \(A=0\) (no temporal predictability; filtering/risk-estimation benchmark).
3. Scalar persistence \(A=\rho I\).
4. Diagonal stable \(A\).
5. Full stable \(A\) only if the simpler models show a reproducible need.

The factor model's contemporaneous explanatory power is not evidence of forecasting ability. If \(A=0\) is best, the model may still be useful for risk decomposition and currency exposure measurement, but it supplies no linear forecast of future factor returns from the current factor state.

No parameters, trading thresholds or profitability claims are established here.

## 2. Exact target and conventions

### 2.1 Currency universe and quote convention

Let \(N=10\) and order the research currencies as:

\[
( AUD,CAD,CHF,EUR,GBP,JPY,NOK,NZD,SEK,USD ).
\]

For a spot pair \(i/j\), define \(S_{i/j,t}\) as units of quote currency \(j\) per one unit of base currency \(i\). Define:

\[
p_{i/j,t}=\log S_{i/j,t}^{mid},\qquad
r_{i/j,t}=p_{i/j,t}-p_{i/j,t-1}.
\]

A positive return means the base currency appreciated against the quote currency. Pair orientation is verified from contract metadata, not guessed from symbol spelling. Inverting a quote changes the sign of its log price and return.

The model is defined on synchronized mid-price returns. Mid prices are for statistical estimation; executable P&L must separately use the correct bid/ask side.

### 2.2 Incidence matrix

For an observed pair \(i/j\), its incidence row has +1 in the base currency column, -1 in the quote currency column and zero elsewhere. Stack the rows to obtain \(B_t\).

The structural identity is:

\[
B_t\mathbf 1_N=0.
\]

Consequently, currency factors are identified only up to a common additive level. For a graph with \(C_t\) connected components and no isolated vertices:

\[
\operatorname{rank}(B_t)=N-C_t.
\]

For a connected graph, the rank is \(N-1\). If the graph is disconnected, each component has its own unidentified common level; a single global factor vector cannot be fully identified from that timestamp.

### 2.3 A fixed contrast basis

Choose a deterministic orthonormal basis \(C\) for the subspace orthogonal to the common-level vector:

\[
C^\top\mathbf 1_N=0,\qquad C^\top C=I_{N-1}.
\]

Represent the normalized currency factors as:

\[
f_t=Cx_t.
\]

This enforces \(\mathbf 1_N^\top f_t=0\) exactly and removes the redundant state coordinate. The basis is a coordinate convention, not an economic ranking of currencies. A different orthonormal basis \(C^*=CU\), with orthogonal \(U\), must produce the same fitted pair returns after transforming state coordinates accordingly.

A deterministic Helmert contrast basis is suitable. For column \(j=1,\ldots,N-1\), set its first \(j\) entries to \(1/\sqrt{j(j+1)}\), its \((j+1)\)-th entry to \(-j/\sqrt{j(j+1)}\), and all later entries to zero.

### 2.4 Observation equation

Define:

\[
H_t=B_tC.
\]

Then:

\[
y_t=H_tx_t+\varepsilon_t.
\]

This is a measurement/decomposition equation for the current cross-section of pair returns. It says the observed pair returns are explained by normalized currency returns plus a residual component. It does not assert that residuals are stationary, mean-reverting or tradable.

For a connected graph, \(\operatorname{rank}(H_t)=N-1\). For a disconnected or sparse graph, the observation may not identify all \(N-1\) state coordinates at that timestamp.

---

## 3. Dynamic state equation

### 3.1 Model the factor as a return, not a price level

In this note, \(x_t\) represents the reduced coordinates of **currency log returns over one sampling interval**. It is not a currency price level or a cumulative currency index.

The baseline transition is:

\[
x_t=A x_{t-1}+u_t,
\qquad
E[u_t]=0,
\qquad
\operatorname{Cov}(u_t)=\Omega.
\]

For a stationary return-state model, require the spectral radius:

\[
\rho(A)<1.
\]

This is a modelling hypothesis to test, not a constraint to hide a unit root. If evidence suggests nonstationary latent levels, formulate a separate level-state model rather than silently replacing the return transition with a random walk.

### 3.2 Nested transition models

Use a complexity ladder:

| Model | Transition | Interpretation |
|---|---|---|
| M0 | \(A=0\) | No serial predictability in factor returns; contemporaneous decomposition baseline |
| M1 | \(A=\rho I\) | Common scalar persistence across contrast coordinates |
| M2 | \(A=\operatorname{diag}(\rho_1,\ldots,\rho_{N-1})\) | Coordinate-specific persistence; basis-sensitive, so interpret cautiously |
| M3 | General stable \(A\) | Cross-coordinate predictive dynamics; highest estimation burden |

M2 is not invariant to rotations of the contrast basis. It is a useful restricted benchmark only if the chosen basis is fixed and disclosed. M3 is coordinate-invariant when all parameters are transformed consistently, but it has many parameters and must be regularized and validated.

A more interpretable alternative to diagonal dynamics is a scalar transition plus a structured low-rank interaction term. It should be introduced only if a named empirical deficiency motivates it.

### 3.3 Innovation and observation assumptions

The Gaussian baseline is:

\[
u_t\sim\mathcal N(0,\Omega),\qquad
\varepsilon_t\sim\mathcal N(0,R_t),\qquad
E[u_t\varepsilon_s^\top]=0.
\]

These assumptions define the initial likelihood and filter. They are not claims that FX returns are Gaussian or homoskedastic.

- \(\Omega\) describes new variation in latent currency-return factors.
- \(R_t\) describes contemporaneous pair-specific residual and observation uncertainty.
- The baseline assumes serially uncorrelated state and observation innovations and independence between them.

Test residual autocorrelation, conditional heteroskedasticity, tail behaviour and cross-pair dependence. If the observation residual is serially correlated, the simple filter is misspecified. Do not interpret that failure as evidence of a mean-reverting trade.

A Student-t innovation or volatility-state extension is a later sensitivity model, not the starting point.

---

## 4. Static estimator as a mathematical benchmark

At a timestamp with full column rank \(H_t\), and a positive-definite observation covariance \(R_t\), generalized least squares gives:

\[
\hat x_t^{GLS}
=
(H_t^\top R_t^{-1}H_t)^{-1}
H_t^\top R_t^{-1}y_t.
\]

Its conditional estimation covariance under the stated linear model is:

\[
\operatorname{Var}(\hat x_t^{GLS}-x_t)
=(H_t^\top R_t^{-1}H_t)^{-1}.
\]

If \(R_t=\sigma^2 I\), this reduces to ordinary least squares up to the common scale.

This is the per-timestamp benchmark against which the dynamic filter must be compared. If \(H_t\) is rank deficient, the displayed inverse does not exist. Do not hide the problem with an arbitrary inverse or silently fill missing observations. Use a valid partial-observation filter, report the unobserved directions, or restrict the estimand to identifiable connected components.

The corresponding currency-factor estimate and covariance are:

\[
\hat f_t=C\hat x_t,
\qquad
\Sigma_{f,t}=C\operatorname{Var}(\hat x_t)C^\top.
\]

The covariance in the original ten-currency coordinates is necessarily singular in the common-level direction. That is correct: absolute common currency level is unidentified.

---

## 5. Kalman filter derivation

Let \(m_{t|t}=E[x_t\mid\mathcal F_t]\) and \(P_{t|t}=\operatorname{Var}(x_t\mid\mathcal F_t)\), where \(\mathcal F_t\) contains only observations available through time \(t\).

### 5.1 Prediction

\[
m_{t|t-1}=A m_{t-1|t-1},
\]

\[
P_{t|t-1}=A P_{t-1|t-1}A^\top+\Omega.
\]

### 5.2 Innovation

For the observed rows at time \(t\):

\[
v_t=y_t-H_tm_{t|t-1},
\]

\[
S_t=H_tP_{t|t-1}H_t^\top+R_t.
\]

The innovation \(v_t\) is the new information in the observed pair returns after accounting for the prior state estimate. Under the correctly specified Gaussian model it has conditional covariance \(S_t\).

### 5.3 Update

\[
K_t=P_{t|t-1}H_t^\top S_t^{-1},
\]

\[
m_{t|t}=m_{t|t-1}+K_tv_t.
\]

For numerical stability, use the Joseph covariance update:

\[
P_{t|t}=(I-K_tH_t)P_{t|t-1}(I-K_tH_t)^\top
+K_tR_tK_t^\top.
\]

In implementation, solve linear systems using a Cholesky factorization of \(S_t\) when positive definite; do not explicitly form \(S_t^{-1}\). Check symmetry and positive semidefiniteness after updates.

### 5.4 Likelihood

The Gaussian innovation log-likelihood contribution is:

\[
\ell_t=-\frac12\left[
m_t\log(2\pi)+\log|S_t|+
v_t^\top S_t^{-1}v_t
\right],
\]

where \(m_t\) is the number of observed pair returns at timestamp \(t\). The sample likelihood is the sum over eligible timestamps, with initialization and missingness handled consistently.

This gives a principled route to estimate \(A,\Omega\), and parameters of \(R_t\) using maximum likelihood or EM. Parameter fitting must occur within each training window only.

### 5.5 Filtered versus smoothed estimates

- **Filtered:** \(m_{t|t}=E[x_t\mid\mathcal F_t]\), using data only through time \(t\). This is the admissible state estimate for a historical decision at time \(t\).
- **Predicted:** \(m_{t|t-1}\), using information only through \(t-1\).
- **Smoothed:** \(m_{t|T}=E[x_t\mid\mathcal F_T]\), for \(T>t\), using future observations. This may be used for training-window parameter estimation and retrospective diagnostics, but never as a historical live-time feature or trading decision.

A backtest that uses smoothed states at each historical timestamp leaks future information.

---

## 6. Missing observations, graph connectivity and information

At each timestamp, construct \(B_t\), \(y_t\), and \(H_t\) from only valid, sufficiently fresh quotes. If some pairs are missing, select the matching rows of the observation matrix and the matching submatrix of \(R_t\). Do not replace missing returns with zero or forward-fill stale quotes as though they were contemporaneous.

For a connected ten-currency graph, the state has nine identifiable contrast directions. If the observed graph has \(C_t>1\) connected components, the instantaneous observation rank is \(10-C_t\), leaving \(C_t-1\) cross-component contrast directions unobserved at that timestamp.

The Kalman prediction may carry prior information into a partially observed timestamp, but it cannot manufacture current observations for an unobserved direction. Report the rank of \(H_t\), posterior uncertainty by direction and time spent disconnected. A strategy layer must be able to abstain when uncertainty is too high.

### 6.1 FXPesa published-universe algebra check

The 37 G10 pairs previously transcribed from FXPesa's public product list form a connected graph on the ten target currencies. An exact rational-matrix calculation gives:

\[
B\in\mathbb R^{37\times10},\qquad
\operatorname{rank}(B)=9.
\]

Therefore the left-nullspace/cycle-space dimension is:

\[
37-9=28.
\]

This is a graph-algebra result for that stated 37-edge list, not confirmation that all 37 symbols are enabled on the user's account or present in MT5 history. The 28 dimensions describe redundant cycle directions in the observed edge space; they are not 28 independent trading opportunities.

The calculation used exact integer incidence entries and SymPy 1.14.0. It verified \(B\mathbf1=0\), rank 9, and an orthonormal ten-by-nine Helmert contrast basis \(C\) with \(C^\top C=I\), \(C^\top\mathbf1=0\), and \(\operatorname{rank}(BC)=9\). This is a finite algebraic check for the listed graph only.

---

## 7. Observation covariance and residual interpretation

This is one of the most consequential modelling choices.

The residual:

\[
\varepsilon_t=y_t-H_tx_t
\]

can contain several distinct sources:

1. Genuine pair-specific market movement not represented by the currency-factor structure.
2. Asynchronous or stale quotes.
3. Bid/ask bounce and microstructure effects.
4. Broker-specific price construction, rounding and feed noise.
5. Mapping or data errors.

These sources must not be casually treated as one economic signal. Data-quality filtering should remove or flag known defects before model estimation. The remaining residual is still not automatically forecastable or mean-reverting.

### 7.1 Covariance ladder

Use a predeclared estimator ladder:

- R0: diagonal \(R_t\), with pair-specific variances estimated on training data.
- R1: diagonal plus shrinkage toward a common scale.
- R2: structured covariance estimated from residuals with shrinkage.
- R3: a cycle-space representation plus diagonal quote-noise floor, only if diagnostics show material residual dependence and enough data exist.

An unrestricted 37-by-37 covariance has 703 distinct entries before time variation. Do not estimate it freely by default. Use shrinkage or structure and report sensitivity.

The observation covariance must be positive definite for the standard innovation likelihood. If a proposed covariance is singular or ill-conditioned, use an explicitly regularized model and document the regularization.

### 7.2 Optional residual-state extension — deferred

If residual diagnostics show persistent pair-specific structure, a later model could use a basis \(D_t\) for the residual/cycle subspace:

\[
y_t=H_tx_t+D_th_t+v_t,
\qquad
h_t=G h_{t-1}+\xi_t.
\]

This is not part of the baseline. It increases the state dimension, creates additional identification and regularization questions, and risks absorbing quote errors into apparently predictive latent states. It should be attempted only after a named residual pattern survives data-quality controls and a frozen out-of-sample test.

---

## 8. Parameter estimation and regularization

### 8.1 Training procedure

Within each chronological training window:

1. Construct the eligible synchronized panel using the frozen data rules.
2. Fix the currency ordering and contrast basis.
3. Initialize \(A,\Omega,R\) from transparent estimates or declared defaults.
4. Estimate parameters by maximizing the innovation likelihood or by EM.
5. Enforce covariance positive semidefiniteness and transition stability.
6. Inspect likelihood convergence, parameter sensitivity and residual diagnostics.
7. Freeze parameters for the next out-of-sample evaluation segment, unless the walk-forward design explicitly schedules refitting.

Smoothing may be used internally by EM on the training window. At each out-of-sample decision timestamp, run the forward filter only.

### 8.2 Complexity control

The state dimension is nine. A fully unrestricted transition matrix has 81 coefficients; a full process covariance has 45 distinct entries. The observation covariance for 37 pairs has 703 distinct entries. These are not necessarily impossible to estimate with a large dataset, but they are not justified as the first model.

Start with:
- \(A=0\), then \(A=\rho I\);
- diagonal or shrinkage \(\Omega\);
- diagonal or structured shrinkage \(R\);
- a fixed, documented sampling frequency and training window.

Only add parameters when the simpler model leaves a specific, repeatable diagnostic failure and the extension improves held-out performance.

### 8.3 Model selection

Use training/validation data for model choice. Compare:
- Innovation log-likelihood and information criteria as in-sample fit diagnostics.
- Out-of-sample predictive log score / CRPS where distribution forecasts are available.
- Calibration and residual diagnostics.
- Stability across chronological folds.
- Incremental value against simple baselines.

A better likelihood or lower in-sample residual sum of squares is not enough to establish a useful forecast.

---

## 9. Forecasting interpretation

The filtered state estimates the current interval's latent currency return after observing the current pair panel. Under the transition model, the next interval's factor-return forecast is:

\[
E[x_{t+1}\mid\mathcal F_t]=A m_{t|t}.
\]

Thus:

\[
E[f_{t+1}\mid\mathcal F_t]=CA m_{t|t},
\qquad
E[y_{t+1}\mid\mathcal F_t]=B_{t+1}CA m_{t|t}.
\]

For a constant future graph, the \(h\)-step factor forecast is \(A^h m_{t|t}\). A cumulative multi-period return forecast must account for cross-horizon state covariance; it is not correct to multiply a one-period variance by the horizon unless the relevant dependence assumptions justify that simplification.

**Critical implication:** if \(A=0\), then the next-period conditional mean forecast is zero even though the model may estimate current currency factors accurately. In that case the model can still support risk attribution, normalization and exposure aggregation, but it has not established a directional alpha signal.

Any trading test must compare the dynamic factor forecast against zero-return, historical-mean and simple autoregressive benchmarks using identical chronological splits and execution-cost assumptions.

---

## 10. Research hypotheses and falsification criteria

| ID | Hypothesis | Evidence required | Falsifier / no-go |
|---|---|---|---|
| DF1 | The constrained factor state is identifiable and numerically stable on eligible connected panels. | Rank checks, basis-invariance checks, stable filtering and reproducible reconstruction. | Rank defects, unstable estimates or dependence on arbitrary basis/edge choices. |
| DF2 | Dynamic filtering improves state estimation over static GLS. | Lower held-out estimation error against a defined proxy/diagnostic, improved innovation behaviour and robust sensitivity. | No repeatable improvement over static GLS or M0. |
| DF3 | Factor returns exhibit stable exploitable temporal dependence. | Better out-of-sample predictive score than zero/mean/AR baselines across folds. | A=0 is preferred, persistence changes sign/size across folds, or forecast gains vanish out of sample. |
| DF4 | Factor uncertainty is informative for risk control. | Calibrated uncertainty and lower error/less exposure in high-uncertainty cases without arbitrary tuning. | Poor calibration or no incremental value over simple volatility/data-quality filters. |
| DF5 | The residual model captures non-factor variation without mistaking feed defects for alpha. | Residual diagnostics robust to quote-age, synchronization and universe sensitivities. | Residuals are dominated by asynchronous quotes, stale data or unstable instrument-specific artifacts. |

A positive result on DF1 or DF2 supports a better factor estimator, not a trading edge. Only DF3 plus subsequent cost-aware economic validation could support a predictive strategy hypothesis.

---

## 11. Limitations and failure modes

- Currency-factor returns are relative and normalized; they are not absolute currency prices.
- The factor model can explain contemporaneous covariance without forecasting.
- The 37 published-pair universe is not yet verified against the actual MT5 account.
- A changing observed graph changes instantaneous information and can make state uncertainty time-varying.
- Diagonal observation noise may overstate information when pair residuals are correlated.
- Full covariance and full transition models can overfit, particularly when refit frequently.
- Gaussian innovations may understate tail risk.
- Sampling and synchronization choices can dominate apparent high-frequency residual structure.
- Kalman filtering does not fix poor data, wrong symbol mappings or nonstationary model structure.
- A state estimate with small variance is not necessarily economically predictive.
- Broker mid-price consistency does not establish executable arbitrage or net profitability.

---

## 12. Literature review and relevance

The literature supports investigating common currency factors and dynamic estimation, but it also gives strong reasons to avoid assuming predictability.

1. **Greenaway-McGrevy, Mark, Sul and Wu (2018), “Identifying Exchange Rate Common Factors,” International Economic Review.** Their work identifies dollar and euro common factors in a broad exchange-rate panel and motivates multilateral rather than purely bilateral modelling. Their data, frequency and factor identification differ from the incidence-constrained FXPesa intraday model proposed here. [NBER Working Paper 23726](https://www.nber.org/papers/w23726) · [Published article DOI](https://doi.org/10.1111/iere.12334).

2. **Aloosh and Bekaert (2022), “Currency Factors,” Management Science.** Their G10 currency-basket analysis reports clustered comovement, including dollar and European blocks, and additional commodity/world factors. This motivates testing structured cross-currency dependence, but their currency-basket factors are not the same object as the incidence-constrained latent currency returns here. [NBER Working Paper 25449](https://www.nber.org/papers/w25449).

3. **Ahmed, Liu and Valente (2016), “Can currency-based risk factors help forecast exchange rates?”, International Journal of Forecasting.** Their reported monthly out-of-sample results largely fail to outperform a random-walk-with-drift benchmark. This is important negative evidence against assuming that factor structure automatically produces useful forecasts. [Author repository record](https://livrepository.liverpool.ac.uk/3045882/) · [DOI](https://doi.org/10.1016/j.ijforecast.2015.01.010).

4. **Date and Maunthrooa (2024), “Modelling and Forecasting of Exchange Rate Pairs Using the Kalman Filter,” Journal of Forecasting.** They formulate joint exchange-rate prediction in state-space form and report results for 18 rates. This is a methodological precedent, not evidence that our different constrained G10 model will work. [Published article DOI](https://doi.org/10.1002/for.3217) · [Brunel repository record](https://bura.brunel.ac.uk/handle/2438/30113).

5. **Engel, Mark and West (2012), “Factor Model Forecasts of Exchange Rates.”** They study forecasts from cross-sectional exchange-rate factors and idiosyncratic deviations, with results varying across historical subsamples and horizons. This reinforces the need for regime/time stability and simple benchmark comparisons. [NBER Working Paper 18382](https://www.nber.org/papers/w18382).

These are empirical and methodological references. Abstract/landing-page descriptions were used for this scoped review; full empirical tables and all specification details have not been independently reanalysed here. No cited paper validates the proposed FXPesa strategy.

---

## 13. Decision and next research step

**Current outcome:** mathematically specified candidate model; no empirical validation.

**What is established in this note:**
- The normalized state can be represented in nine coordinates for a connected ten-currency graph.
- The state-space and Kalman recursions are specified.
- The missing-data and graph-rank implications are explicit.
- A 37-edge graph transcribed from the published FXPesa G10 list has exact incidence rank nine and cycle-space dimension 28.
- The model's forecasting claim is separated from contemporaneous factor estimation.

**What remains unresolved:**
- Actual MT5 symbol mapping, coverage and synchronization.
- Empirical estimates of \(A,\Omega,R\).
- Whether Kalman filtering improves on static GLS.
- Whether any factor dynamics forecast future returns out of sample.
- Whether forecasts survive execution costs.

**Next step:** execute Research Protocol 02 on a frozen FXPesa historical panel, after the actual MT5 instrument audit. Do not proceed to strategy optimization until the data and descriptive-factor gates in Research Protocol 01 pass.

## Research log

| Date | Entry |
|---|---|
| 2026-11-10 | Derived a nine-dimensional constrained state-space specification, Kalman recursions, likelihood, missing-graph implications and a nested estimator ladder. Exact incidence-rank calculation performed for the 37-pair public-list graph; no broker-account data or predictive model fit used. |
