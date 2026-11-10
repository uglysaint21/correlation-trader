# Research Note 04: Integrated Quantitative Strategy Research Agenda

**Status:** Research programme specification; no strategy selected or validated  
**Date:** 10 November 2026  
**Decision status:** Proposed research agenda, not an approved trading specification  
**Depends on:** Research Notes 01–03  
**Purpose:** Consolidate and sharpen the mathematical, statistical, strategy, portfolio, execution and validation work required before selecting an implementable strategy.

---

## 1. Executive recommendation

The system should not be designed as a single correlation indicator or a large monolithic prediction model. It should be researched as a collection of separately testable layers:

1. A quote-safe currency graph and synchronized data panel.
2. A constrained currency-factor model for contemporaneous risk decomposition.
3. Separate predictive models for future currency and pair returns.
4. Independent relative-value / cointegration and momentum strategy families.
5. A probabilistic regime and uncertainty layer that may restrict strategy eligibility.
6. A currency-aware portfolio construction and risk layer.
7. An execution model based on the broker's actual bid/ask quotes, contract terms and fills.
8. A pre-registered, leakage-resistant evaluation process.

The first strategy candidates should be **currency-factor conditional forecasting**, **cointegration/error-correction mean reversion**, and **cross-sectional currency momentum**. They must be evaluated independently before any ensemble is considered.

The most important design principle is separation of roles: a model that explains returns is not necessarily a model that forecasts returns; a stationary spread is not necessarily profitable; a high-confidence forecast is not necessarily executable; and a profitable isolated trade is not necessarily desirable at portfolio level.

No profitability claim, trade rule, parameter, or live deployment is approved by this note.

## 2. Corrections and refinements to the current research direction

### 2.1 Keep the incidence structure fixed; let the latent factors evolve

For a correctly mapped spot-FX pair, the +1/-1 currency incidence row is determined by its base and quote currencies. It is not an estimated loading that should drift over time.

The core structural representation remains:

\[
r_t = B_t f_t + \epsilon_t.
\]

Here, the observed subgraph can change as quotes become missing or stale, so \(B_t\) may vary with the eligible instrument set. But the orientation of each valid edge is fixed by contract metadata. The latent currency-return state \(f_t\), its transition dynamics and the residual covariance may evolve.

Time-varying loadings belong to alternative empirical factor models, such as PCA or unconstrained latent-factor benchmarks. They should not be confused with the known currency-incidence matrix.

### 2.2 A factor model is initially a risk/decomposition model, not an alpha model

The contemporaneous estimate \(\hat f_t\) explains the cross-section at time \(t\). It is not available before that cross-section has occurred and cannot be used as a forecast of the same interval.

Predictive work must forecast future factors using information available at the decision time:

\[
f_{t+h|t}=E[f_{t+h}\mid\mathcal F_t],
\]

then evaluate whether the forecast and any residual features improve future pair-return forecasts over simple baselines.

A large contemporaneous residual is an anomaly or unexplained return, not an automatic reversal signal.

### 2.3 FX graph redundancy must be handled explicitly

For a graph with \(N\) currencies, \(M\) observed edges and \(C\) connected components, the incidence rank is \(N-C\), assuming isolated nodes are excluded. The residual subspace has dimension \(M-(N-C)\).

For a connected ten-currency graph with all 45 possible pairs, the rank is nine and there are 36 redundant edge directions. These directions encode cycle consistency information; they are not 36 independent opportunities for alpha.

Exact cross-rate identities mean that many apparent pair relationships are mechanically related. A strategy must distinguish executable relative-value opportunities from algebraic redundancy, asynchronous quotes, stale prices and broker-feed inconsistencies.

### 2.4 Do not assume that a dynamic model is better

Kalman filters, HMMs, DCC-GARCH, copulas, neural networks and other flexible methods are candidate extensions. Every extension must beat a simpler baseline on a predeclared out-of-sample metric after costs, while preserving calibration and parameter stability. Complexity without incremental evidence is a reason to reject the extension.

---

## 3. Research Route A — Dynamic currency-factor estimation

### 3.1 Constrained state-space model

The currency factors are only identified up to a common level. Rather than repeatedly imposing an unconstrained state vector, define an orthonormal basis \(Q\in\mathbb R^{N\times(N-1)}\) for the subspace orthogonal to the all-ones vector:

\[
Q^\top\mathbf 1=0,\qquad Q^\top Q=I.
\]

Represent the factors as \(f_t=Qg_t\). A baseline state-space model is:

\[
r_t=B_tQg_t+\epsilon_t,\qquad
g_t=F g_{t-1}+u_t.
\]

With:

\[
\epsilon_t\sim(0,R_t),\qquad u_t\sim(0,Q_u).
\]

This allows the Kalman filter to estimate a time-varying latent currency-return state while keeping the currency-level normalization explicit. The initial benchmark should use a simple stable transition such as a zero-mean or low-order autoregressive process; richer dynamics require evidence.

For disconnected observed graphs, construct a separate identifying basis for each connected component. Do not compare factor levels across components as though they shared a common numeraire.

### 3.2 Estimator ladder

Compare in this order:

1. Constrained equal-weight least squares at each timestamp.
2. Weighted least squares using predeclared, observable data-quality or measurement-variance weights.
3. Robust regression as a sensitivity analysis for outliers.
4. A constrained state-space/Kalman filter.
5. PCA or other unconstrained factor models as statistical benchmarks, not as interchangeable currency factors.

Use the full eligible graph for diagnostics and estimation, and a deterministic spanning-tree basis for independent-coordinate checks. Compare reconstructed fitted returns across equivalent bases.

### 3.3 Factor uncertainty and covariance

Store factor estimates with their posterior/estimation covariance, not just point values:

\[
\hat g_{t|t},\quad P_{t|t},\quad
\hat f_{t|t}=Q\hat g_{t|t},\quad
\Sigma_{f,t}=QP_{t|t}Q^\top.
\]

Use uncertainty when ranking signals and sizing risk. A large estimated currency divergence with high uncertainty should not automatically outrank a smaller but more precisely estimated divergence.

For covariance estimation, benchmark sample covariance against shrinkage estimators. Consider DCC-GARCH only if the dimension, data quantity and numerical stability support it. The 10-currency factor space is much smaller than a 37–45 pair space and should be the natural first place to investigate multivariate volatility.

### 3.4 Required tests

- Factor reconstruction and invariance under spanning-tree/basis changes.
- Rank and connectivity diagnostics at every timestamp.
- State and covariance stability across sessions and rolling windows.
- Innovation whiteness and residual dependence diagnostics.
- Filtered versus smoothed estimates: **only filtered estimates using information through time t are admissible for live-like decisions**.
- Incremental forecast value against zero-return, historical-mean and simple autoregressive baselines.
- Sensitivity to missing edges, quote-age thresholds, weighting and sampling frequency.

**Route A go/no-go:** the factor model must be reproducible, numerically stable and useful either as a risk representation or as a forecast input. Contemporaneous explanatory power alone does not qualify it as a trading signal.

---

## 4. Research Route B — Cointegration and relative-value dynamics

### 4.1 Correct sequence of tests

For every candidate system:

1. Define the exact log-price series, quote orientation and economically motivated candidate universe before testing.
2. Test integration order and inspect deterministic components; do not apply standard I(1) cointegration procedures blindly to series with incompatible orders.
3. Specify lag selection and deterministic terms before looking at trading performance.
4. Apply Engle-Granger for a predeclared bivariate hypothesis or Johansen for a multivariate system.
5. Diagnose residual stationarity, serial correlation, parameter stability and structural breaks.
6. Repeat the entire selection process inside each training window only.
7. Control the family of tests across all candidate pairs, baskets, lags and specifications.

Do not search every possible combination, retain the best p-values, and then report those same p-values as confirmatory evidence.

### 4.2 Avoid mechanical cointegration discoveries

FX cross rates are algebraically connected. For example, under consistent simultaneous quotes:

\[
\log(EURJPY)=\log(EURUSD)+\log(USDJPY).
\]

A linear relation created by this identity is not a discovered economic equilibrium or independent trading edge. Build candidate systems from an independent coordinate representation where possible, document the economic interpretation of each relation, and compare any proposed spread with the exact synthetic replication implied by the currency graph.

A cointegrating vector is only a statistical relation. To trade it, translate the vector into actual broker instruments and signed currency exposures, then compute the executable bid/ask cost of establishing, maintaining and unwinding the portfolio.

### 4.3 Spread dynamics

For a spread that passes the cointegration and stability tests, compare:

- Discrete-time AR(1) and AR(p) models.
- Error-correction models.
- OU dynamics where the sampling assumptions justify a continuous-time approximation.
- State-space/Kalman hedge-ratio models as an adaptive alternative.
- Heavy-tailed or heteroskedastic innovations.

For regular interval observations, an AR(1) spread model is:

\[
S_t-\mu=\phi(S_{t-1}-\mu)+\eta_t.
\]

For \(0<\phi<1\), the implied half-life in observations is:

\[
h_{1/2}=\frac{\log(0.5)}{\log(\phi)}.
\]

For an OU model sampled at elapsed interval \(\Delta\), the exact conditional mean mapping is:

\[
\phi(\Delta)=e^{-\kappa\Delta}.
\]

The half-life in clock time is \(\log(2)/\kappa\). Irregular time gaps must use elapsed time, not an assumed constant bar count. If fitted dynamics do not support a meaningful positive-persistence half-life, do not force one onto the process.

### 4.4 Required tests

- Cointegration rank and residual tests with correct deterministic terms.
- Break and rolling-window stability.
- Hedge-ratio uncertainty and turnover induced by re-estimation.
- Forecast calibration for future spread changes.
- First-passage and expected-time-to-target estimates.
- Net profitability under executable bid/ask prices and realistic financing.
- Comparison against a no-trade baseline and simple spread rules.
- False-discovery control over the full search process.

**Route B go/no-go:** a candidate must show stable out-of-sample error-correction behaviour and positive net value after implementation costs. A low p-value or stationary in-sample residual is insufficient.

---

## 5. Research Route C — Cross-sectional currency momentum

This is a deliberately different strategy family from mean reversion.

Estimate currency-level returns over predeclared lookback horizons, rank currencies cross-sectionally, and test whether relative strength persists over a future holding horizon.

A basic signal could be:

\[
m_{i,t}^{(L)}=\sum_{j=1}^{L}w_j f_{i,t-j},
\]

where weights and lookback horizon are frozen in advance. A currency score should be normalized for volatility and estimation uncertainty, and currency scores should respect the factor normalization.

Research questions:

- Does currency-level relative strength persist at any tested horizon?
- Does it survive controls for USD dominance and common market exposure?
- Is it distinct from simple time-series momentum?
- Does it behave differently by session and volatility regime?
- Does a portfolio constructed from currency ranks produce lower redundant exposure than independently ranking every pair?

Use a small predeclared horizon grid and nested training-window selection. Compare against zero, equal-weight and simple trend baselines. Do not use future returns to decide which lookback is reported.

**Route C go/no-go:** require stable incremental out-of-sample forecasting and executable net returns; a ranking that merely reconstructs contemporaneous factor movements is not enough.

---

## 6. Regime detection and model uncertainty

Regime models should describe uncertainty and conditional behaviour, not serve as decorative labels.

### Candidate models

- Markov-switching autoregressive models.
- Hidden Markov Models.
- Hidden semi-Markov models where state duration matters.
- Bayesian online change-point detection.
- Simple volatility/liquidity state rules as transparent baselines.

Start with a small state count and observable features such as realized volatility, spread/quote quality, factor covariance and residual dispersion. Add macroeconomic event data only if a reliable, timestamped, point-in-time source is available.

For a hidden state \(z_t\), the admissible live-time quantity is the filtered posterior:

\[
P(z_t=k\mid X_{1:t}).
\]

The smoothed posterior \(P(z_t=k\mid X_{1:T})\), for \(T>t\), uses future observations and is not admissible as a historical trading feature.

### Regime research protocol

- Compare against no regime filter.
- Evaluate state stability, posterior entropy and transition frequency.
- Measure out-of-sample conditional calibration and strategy performance by regime.
- Penalize excessive state switching and parameter sensitivity.
- Test whether the regime layer adds value after costs rather than merely reducing trade count.
- Define an explicit uncertain-state/no-trade policy.

A regime model must not be allowed to retrospectively relabel history using future data in the backtest.

---

## 7. Probabilistic forecasting and trade decisions

### 7.1 Forecast distributions, not just directions

For each strategy and horizon, estimate a conditional distribution:

\[
p(R_{t,t+h}\mid\mathcal F_t).
\]

Candidate approaches should progress from simple to complex:

1. Historical/empirical conditional distributions.
2. Linear or autoregressive forecasts with robust standard errors.
3. Student-t innovations.
4. GARCH-family conditional volatility.
5. Bayesian parameter uncertainty.
6. Multivariate volatility or copula models only where tail dependence or joint-risk modelling adds demonstrable value.

Forecast evaluation should include proper scoring rules such as log score and CRPS where appropriate, alongside calibration curves, probability integral transform diagnostics, interval coverage and tail-event calibration. Classification accuracy alone is not a sufficient measure.

### 7.2 Trade outcome probabilities

For a proposed direction, size and execution plan, estimate quantities such as:

- Probability of target before stop within horizon H.
- Expected net P&L and its uncertainty.
- Expected adverse and favourable excursion.
- Probability of gap-through or stop slippage.
- Expected holding time and probability of time-out.
- Probability that the statistical relationship deteriorates before exit.

These should be derived from a path-dependent model or simulation where stop/target order matters. A terminal-return distribution alone does not determine which barrier is hit first.

### 7.3 Decision rule

A trade should be eligible only if its estimated net utility is positive and sufficiently robust under parameter and execution uncertainty:

\[
E[U(\Pi_{net})\mid\mathcal F_t]
>
U_{no\ trade},
\]

subject to portfolio and operational constraints.

Do not use a fixed probability threshold without accounting for payoff asymmetry, costs, tail losses, model uncertainty and opportunity cost. A 70% win probability can be unattractive if the average loss is much larger than the average gain.

### 7.4 Entry, exit and optimal stopping

Compare simple fixed rules with state-dependent rules. Research first-passage-time distributions and optimal stopping only after the spread/return process is credibly specified. Include maximum holding time, invalidation conditions, relationship-break exits and time-out outcomes. Do not equate OU half-life with an optimal holding period.

---

## 8. Portfolio construction and risk

### 8.1 Currency exposure, not symbol count

Let \(q_t\) be signed base-currency notionals for the selected instruments, expressed in a consistent valuation currency and with quote-currency cashflows converted consistently. Then a first-order currency exposure representation is:

\[
e_t=B_t^\top q_t.
\]

This expression is only meaningful when \(q_t\) has been converted from MT5 lots/contracts into comparable currency notionals. Raw lot counts across different pairs are not interchangeable. Contract size, account-currency conversion, price, tick value and broker-specific specifications must be applied.

For nonlinear or large positions, compute exact account-currency P&L sensitivities rather than relying solely on the linear approximation.

### 8.2 Portfolio objectives and constraints

Candidate optimizers:

- Minimum variance and constrained mean-variance.
- Volatility targeting.
- Risk budgeting / risk parity.
- CVaR or expected-shortfall minimization.
- Robust optimization under forecast/covariance uncertainty.
- Fractional Kelly only as a later benchmark with conservative caps.

A candidate objective is:

\[
\max_q\quad \hat\mu^\top q
-\lambda\,CVaR_\alpha(L(q))
- C(q,\Delta q),
\]

subject to hard limits on gross/net leverage, currency concentration, pair concentration, margin, turnover, liquidity, drawdown and instrument eligibility.

Costs must be charged on changes in positions, not only on final exposure. Optimizers should be tested for sensitivity to small changes in expected returns and covariance estimates. If minor estimation changes produce extreme position changes, prefer shrinkage, robust constraints or no trade.

### 8.3 Risk controls

Research and specify:
- Per-currency and aggregate exposure caps.
- Gross/net leverage and margin buffers.
- Maximum portfolio and daily loss.
- Drawdown-based de-risking and hard shutdown.
- Correlation/concentration stress tests.
- Gap and tail scenarios.
- Data staleness, spread widening and execution failure controls.
- Limits on simultaneous correlated strategy bets.

Risk limits are hard constraints, not parameters optimized for historical profitability.

---

## 9. Execution and broker-specific economics

All research P&L must be computed from executable prices and account-currency cashflows.

At minimum, model:

- Long entry at ask and long exit at bid; short entry at bid and short exit at ask.
- Variable spread by instrument, session, volatility and event conditions.
- Commission, swap/financing and triple-swap rules where applicable.
- Slippage distribution conditional on order type, size, volatility and quote state.
- Latency, partial fills, requotes, rejections and stop execution uncertainty.
- Contract size, lot step, minimum/maximum volume, tick size/value and margin rules.
- Account-currency conversion and realized cashflow reconciliation.

A simplified net-return decomposition is:

\[
\Pi_{net}=\Pi_{gross}
-C_{spread}-C_{commission}-C_{slippage}
-C_{financing}-C_{other}.
\]

Avoid double-counting spread if gross P&L already uses executable bid/ask fills. Build a single, explicit fill and cost ledger.

Use FXPesa's actual MT5 symbol catalogue, contract specifications and observed quote/fill data. Public instrument lists are not proof of live-account tradability, historical tick coverage, or realized execution quality.

---

## 10. Validation, multiple testing and research governance

This layer is mandatory for every route.

### 10.1 Freeze the research process

Before a strategy experiment, record:
- Hypothesis and rationale.
- Universe and point-in-time inclusion rule.
- Sampling interval and forecast horizon.
- Features and their availability timestamps.
- Model and parameter search space.
- Baselines and evaluation metrics.
- Cost and fill assumptions.
- Train/validation/test dates.
- Acceptance/rejection criteria.
- Number of attempted variants, including failed experiments.

Keep an experiment registry. The count of attempted variants is needed to interpret selection-adjusted performance statistics.

### 10.2 Time-series validation

Use chronological walk-forward evaluation as the primary deployment simulation. For labels whose outcome windows overlap, purge training observations whose label intervals overlap the test interval and apply an embargo where justified by the dependence structure.

Combinatorial purged cross-validation and probability-of-backtest-overfitting analysis may be useful diagnostics, but they do not replace a final untouched chronological holdout. The exact split method must match the deployment question and the label horizon. Purging cannot repair leakage already embedded in features or data preprocessing.

All transformations, universe selection, factor estimation, feature selection, hedge-ratio estimation, regime fitting, calibration and parameter tuning must be performed inside the training portion of each fold.

### 10.3 Multiple comparisons and selection bias

Use a documented strategy for the full hypothesis family:
- Family-wise error control for confirmatory claims where false positives are especially costly.
- False-discovery-rate procedures for exploratory screening, with dependence assumptions stated.
- Bootstrap or block-bootstrap inference that respects time dependence.
- Deflated Sharpe Ratio / Probabilistic Sharpe Ratio with a defensible estimate of the number of effective trials.
- Probability of Backtest Overfitting as a complementary diagnostic.
- Reality Check / SPA-style comparisons where a family of competing rules is evaluated.

No single adjustment is a universal cure. P-values from cointegration tests, Sharpe ratios from strategy selection, and repeated feature/model searches require different treatment.

### 10.4 Robustness and uncertainty

Report:
- Net performance with uncertainty intervals.
- Performance by chronological test fold, session, pair, currency and regime.
- Drawdown and tail-loss distributions.
- Sensitivity to spreads, commissions, swaps and slippage.
- Sensitivity to sampling frequency, synchronization tolerance and missingness.
- Parameter and model stability.
- Trade count, turnover, capacity proxies and exposure.
- Calibration and forecast-score diagnostics.
- Ablation tests showing the incremental contribution of each model layer.

A strategy must not be accepted because it wins on one aggregate metric or one historical episode.

### 10.5 Minimum progression gates

**Gate 0 — Data validity:** verified symbols, quote orientation, UTC alignment, coverage, executable bid/ask and reproducible raw data.

**Gate 1 — Statistical validity:** model assumptions and diagnostics are acceptable; results survive stability and multiplicity checks.

**Gate 2 — Predictive validity:** forecasts beat simple baselines on untouched chronological data and are adequately calibrated.

**Gate 3 — Economic validity:** net expected value remains positive under realistic costs and conservative stress assumptions.

**Gate 4 — Portfolio validity:** the strategy improves portfolio-level outcomes without unacceptable currency concentration, drawdown or tail risk.

**Gate 5 — Operational validity:** demo/paper execution, reconciliation, kill switches, logging and failure handling pass a separate review.

Passing a gate permits further research, not automatic live deployment.

---

## 11. Comparative experiment design

The first predictive comparison should use a common data manifest, identical execution assumptions, the same chronological splits and the same portfolio constraints.

| Candidate | Primary signal | Essential null/baseline | Key failure test |
|---|---|---|---|
| A. Currency-factor forecast | Forecasted latent currency returns and conditional pair residual features | Zero return; historical mean; simple AR | No incremental forecast value after factor/risk controls |
| B. Cointegration / error correction | Forecasted adjustment in a preselected stationary combination | No-trade; simple AR spread | Breaks, false discoveries, synthetic cross-rate redundancy |
| C. Currency momentum | Lagged relative currency strength | Equal-weight; zero forecast; simple time-series trend | Signal vanishes out of sample or is only USD exposure |
| D. Regime-conditioned variants | A–C gated/weighted by filtered regime probabilities | Each strategy without regime filter | Regime layer adds no net value or is unstable |
| E. Combined portfolio | Diversified allocation across independently validated strategies | Best standalone strategy and fixed-weight combination | No incremental risk-adjusted benefit or hidden duplicated exposure |

Do not optimize all strategies jointly at the outset. First estimate each family's standalone contribution. Only then test whether a combined portfolio improves out-of-sample utility.

---

## 12. Research order and deliverables

The existing Notes 01–03 remain the foundation. The next work should be split into bounded, auditable documents and protocols:

| Order | Proposed document | Main deliverable |
|---|---|---|
| 04A | Dynamic Currency-Factor Models | Constrained state-space specification, estimator ladder and diagnostic protocol |
| 04B | Cointegration and Error-Correction Research | Candidate construction, integration/cointegration tests, break tests and spread dynamics |
| 04C | Currency Momentum Research | Predeclared cross-sectional and time-series momentum hypotheses |
| 04D | Regime and Probabilistic Forecasting | Filtered regime-state protocol, predictive distributions and calibration tests |
| 04E | Portfolio and Execution Economics | Currency-notional mapping, constraints, fill/cost model and stress tests |
| 04F | Validation and Experiment Governance | Experiment registry, chronological splits, purging, multiplicity and acceptance gates |

The first empirical work remains the descriptive data/factor study in Research Protocol 01. Do not skip it to begin strategy optimization.

### Priority order

1. Complete FXPesa MT5 symbol, history, bid/ask and contract audit.
2. Execute the descriptive G10 panel and factor diagnostics protocol.
3. Specify and compare static constrained factors with a constrained Kalman state-space model.
4. Specify one carefully controlled cointegration study and one currency-momentum study.
5. Establish the common predictive-validation and execution-cost framework before optimizing any entry or exit.
6. Test regime filters only as incremental overlays against unfiltered baselines.
7. Design portfolio allocation after standalone signals have demonstrated independent predictive value.
8. Only after the above gates, select one strategy for a precise mathematical specification and architecture review.

---

## 13. Explicit non-goals for the current phase

Defer the following until the simpler research programme establishes a credible baseline:

- Deep neural networks and transformer models.
- Reinforcement learning and policy optimization.
- Large unconstrained copula systems.
- High-dimensional multivariate GARCH across every broker pair.
- Automated online model selection without a frozen evaluation protocol.
- Adaptive strategy switching based on in-sample performance.
- Live trading or autonomous order execution.

These methods are not rejected forever. They are deferred because their complexity, data demands and overfitting risk are not justified before baseline evidence exists.

---

## 14. Selected literature and source notes

These references motivate candidate methods; none establishes that the proposed FXPesa strategy will be profitable.

- Engle, R. F. & Granger, C. W. J. (1987). [Co-integration and Error Correction: Representation, Estimation, and Testing](https://doi.org/10.2307/1913236). *Econometrica*. Foundational cointegration/error-correction framework.
- Johansen, S. (1988). [Statistical Analysis of Cointegration Vectors](https://doi.org/10.1016/0165-1889(88)90041-3). *Journal of Economic Dynamics and Control*. Multivariate cointegration rank and vector estimation.
- Hamilton, J. D. (1989). [A New Approach to the Economic Analysis of Nonstationary Time Series and the Business Cycle](https://ideas.repec.org/a/ecm/emetrp/v57y1989i2p357-84.html). *Econometrica*. Regime-switching state inference.
- Bollerslev, T. (1986). [Generalized Autoregressive Conditional Heteroskedasticity](https://doi.org/10.1016/0304-4076(86)90063-1). *Journal of Econometrics*. Conditional variance modelling.
- Engle, R. (2002). [Dynamic Conditional Correlation](https://doi.org/10.1198/073500102288618487). *Journal of Business & Economic Statistics*. Parsimonious time-varying correlation modelling.
- Benjamini, Y. & Hochberg, Y. (1995). [Controlling the False Discovery Rate: A Practical and Powerful Approach to Multiple Testing](https://doi.org/10.1111/j.2517-6161.1995.tb02031.x). *JRSS B*. Multiple-hypothesis screening; dependence assumptions must be checked.
- Bailey, D. H., Borwein, J. M., López de Prado, M. & Zhu, Q. J. (2017). [The Probability of Backtest Overfitting](https://doi.org/10.21314/JCF.2016.322). *Journal of Computational Finance*. CSCV/PBO framework and limitations.
- Bailey, D. H. & López de Prado, M. (2014). [The Deflated Sharpe Ratio](https://doi.org/10.3905/jpm.2014.40.5.094). *The Journal of Portfolio Management*. Selection bias and non-normality adjustments.
- Gatev, E., Goetzmann, W. N. & Rouwenhorst, K. G. (2006). [Pairs Trading: Performance of a Relative-Value Arbitrage Rule](https://doi.org/10.1093/rfs/hhj020). *Review of Financial Studies*. Historical equity-pairs evidence; not direct evidence for OTC FX or this broker.
- Bank for International Settlements (2025). [OTC foreign exchange turnover in April 2025](https://www.bis.org/publications/202509-commentary-otc-derivatives). Market-wide context only; it does not establish FXPesa liquidity, spreads or execution quality.

## Research log

| Date | Entry |
|---|---|
| 2026-11-10 | Consolidated the research programme; clarified fixed incidence loadings versus dynamic latent factors, filtered versus smoothed regime estimates, graph-cycle redundancy, executable portfolio exposures, and validation gates. No empirical results or strategy profitability claims recorded. |
