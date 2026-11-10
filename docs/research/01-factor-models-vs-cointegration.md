# Research Note 01: Currency-Factor Models vs. Multi-Pair Cointegration

**Status:** Initial literature review and research design  
**Date:** 10 November 2026  
**Decision status:** Provisional research recommendation only; no trading edge has been established  
**Scope:** Choose the first statistical representation to investigate for a multi-currency MT5 research system.

---

## 1. Executive conclusion

The first research baseline should be a **currency-factor model of synchronized FX returns**, with **cointegration treated as a separate, conditional strategy family** rather than assumed to be the same thing.

The two approaches answer different questions:

- A currency-factor model asks: *what common currency-level shocks explain the cross-section of pair returns, and which pairs exhibit unusual or potentially forecastable residual behaviour?*
- A cointegration model asks: *do particular non-stationary log-price series possess a stable stationary linear combination that can support an error-correction forecast?*

A factor residual is not automatically a mean-reverting spread. In a return-factor model, the residual is ordinarily an unexplained return innovation. It may be useful for conditional forecasting, anomaly detection, or risk decomposition, but a contrarian trade cannot be justified merely because a residual is large. Conversely, cointegration supplies a formal long-run equilibrium relation under its assumptions, but relationships may be unstable, sparse, costly to trade, or discovered by chance.

**Provisional research sequence:**

1. Build a quote-convention-safe currency return panel and estimate currency factors.
2. Test whether factor residuals have genuine, stable, out-of-sample predictive content after costs. Do not presume mean reversion.
3. Independently test a small, economically motivated set of cointegrating systems using log prices and rolling/walk-forward estimation.
4. Compare both families using the same data, execution assumptions, portfolio constraints, and untouched evaluation periods.
5. Combine them only if the combined model adds incremental out-of-sample value and does not merely duplicate exposures.

No live-trading recommendation follows from this note.

## 2. Research questions and hypotheses

### Primary questions

- Can cross-pair returns be decomposed into a small set of currency-level shocks plus pair-specific residuals?
- Do those residuals predict future returns, volatility, or liquidity after controlling for known currency exposures?
- Are there stable cointegrating relationships among selected FX log-price series?
- Do the two methods deliver distinct signals, or are apparent opportunities mostly shared currency exposure and algebraic cross-rate structure?
- Does any forecast survive realistic bid/ask execution, commission, swaps/financing, slippage, latency assumptions, and portfolio-level netting?

### Hypotheses to falsify

| ID | Hypothesis | What would count against it |
|---|---|---|
| H1 | Currency factors explain a meaningful share of contemporaneous pair-return covariance. | Unstable factors, weak explanatory power, or no improvement over simple baselines. |
| H2 | Factor-model residual features contain incremental conditional forecast information. | No stable improvement over zero-return and currency-factor-only forecasts after costs. |
| H3 | A subset of economically plausible pair baskets is cointegrated over useful horizons. | Rank/relationship instability, poor residual diagnostics, or failure to persist out of sample. |
| H4 | Error-correction forecasts can be monetized after executable costs. | Gross edge disappears at bid/ask, turnover, financing, or realistic fills. |
| H5 | Combining factors and cointegration improves portfolio outcomes. | No incremental out-of-sample benefit, higher tail risk, or unstable exposure concentration. |

A rejection is a useful research result. The system must not be optimized until one of these hypotheses appears to work.

## 3. Data and notation

Let (S_{i/j,t}) denote the number of units of quote currency (j) per one unit of base currency (i). Define the log mid-price:

[
p_{i/j,t} = log S_{i/j,t}^{mid}
]

and the log return over interval (t):

[
r_{i/j,t} = p_{i/j,t} - p_{i/j,t-1}.
]

For synchronized observations, the exact cross-rate identity is:

[
p_{i/k,t} = p_{i/j,t} + p_{j/k,t},
]

subject to consistent quote conventions and simultaneous prices. Thus:

[
r_{i/k,t} = r_{i/j,t} + r_{j/k,t}.
]

This identity creates deterministic dependencies among quoted pairs. A panel containing all possible crosses is not a collection of independent observations. Triangular relationships, stale quotes, asynchronous bar closes, and broker-specific prices can create misleading apparent residuals.

### Required data discipline

- Use bid, ask, and mid separately. Mid prices are suitable for many model estimates; executable P&L must use the appropriate bid/ask side.
- Align timestamps to a common UTC event/bar boundary; retain source timestamps and missingness flags.
- Record symbol contract specifications, quote precision, tick size/value, lot constraints, trading sessions, and broker suffix mappings.
- Treat missing and stale prices explicitly. Do not forward-fill across material gaps and then interpret the result as a tradable synchronous relationship.
- Preserve point-in-time data and model snapshots so each decision can be reconstructed.
- Begin with a liquid, predeclared G10 universe; expand only after the method survives validation.
- Do not mix broker feeds without a documented normalization and reconciliation procedure.

## 4. Approach A — currency-factor model

### 4.1 Cross-sectional return representation

Represent each currency as a node in a currency graph. For a pair (i/j), define an oriented incidence row with (+1) for the base currency and (-1) for the quote currency. Let (B) be the pair-by-currency incidence matrix and (f_t) the vector of latent currency returns.

A first-order model is:

[
r_t = B f_t + epsilon_t,
]

where:

- (r_t) is the vector of pair log returns;
- (f_t) is the vector of currency-level return shocks;
- (B) encodes each pair's base/quote orientation;
- (epsilon_t) is pair-specific residual return.

Because adding the same constant to every element of (f_t) leaves (Bf_t) unchanged, the currency factors are not uniquely identified without a normalization. Use a constraint such as (sum_i f_{i,t}=0), or drop a reference currency and document the interpretation. For a connected currency graph, the incidence matrix has rank at most (N-1).

Estimate (f_t) using a constrained least-squares solution, a robust estimator, or a regularized/state-space model. If using PCA, remember that PCA factors are statistical components, not automatically named economic currency factors. Compare PCA against the incidence-structure model rather than treating the labels as interchangeable.

### 4.2 What the residual does — and does not — mean

The residual return is:

[
hatepsilon_t = r_t - Bhat f_t.
]

This is a contemporaneous unexplained return component. It is **not** by definition a stationary price spread and does **not** imply a reversal opportunity. A large positive residual says that a pair rose more than the fitted currency-factor component during that interval; it does not say that it must fall next.

Potential research uses include:

1. **Conditional forecasting:** test whether lagged residuals, residual volatility, or residual interactions forecast future returns.
2. **Anomaly detection:** flag pair-specific moves that are unusual relative to the currency cross-section.
3. **Risk decomposition:** distinguish broad currency exposure from pair-specific risk.
4. **Regime conditioning:** test whether residual behaviour changes with volatility, liquidity, or market regime.

A candidate predictive specification might be:

[
r_{t+1} = Bhat f_{t+1|t} + Gamma z_t + u_{t+1},
]

where (z_t) contains only information available at time (t), such as lagged residual features and predeclared regime variables. The forecast (hat f_{t+1|t}) must itself be genuinely predictive; a contemporaneous factor estimate cannot be used as if it were known in advance.

An alternative dynamic specification can model the residual state:

[
epsilon_t = phi epsilon_{t-1} + eta_t,
]

but (|phi|<1), stability, and economically meaningful conditional forecasts must be established out of sample. Even if residual returns exhibit negative autocorrelation, the net tradable edge may be too small after costs.

### 4.3 Strengths

- Uses the shared currency structure across many pairs instead of treating every pair as unrelated.
- Naturally supports currency-level exposure aggregation and portfolio risk attribution.
- Can use a broad panel and can remain useful even when no stable cointegrating relation exists.
- Makes quote orientation and cross-rate dependencies explicit.
- Offers a natural bridge to dynamic factor models, Bayesian filtering, and regime-dependent covariance estimation.

### 4.4 Weaknesses and failure modes

- Factor estimation may be dominated by the selected universe, missingness, or one highly liquid currency.
- PCA components can rotate and change meaning across rolling windows.
- A factor model that explains covariance does not necessarily forecast returns.
- Residuals may mostly represent microstructure noise, asynchronous quotes, or broker-specific pricing.
- A return residual is not a cointegrating spread.
- Expanding the universe increases the number of candidate features and the risk of data snooping.

## 5. Approach B — multi-pair cointegration

### 5.1 Definition

Let (x_t) be a vector of log price levels whose individual components are plausibly integrated of order one, (I(1)). The components are cointegrated if there is a non-zero vector (eta) such that:

[
z_t = eta^	op x_t
]

is stationary, (I(0)).

For a two-series example:

[
z_t = log S_{A/B,t} - alpha - eta log S_{C/D,t}.
]

The estimated equilibrium error (z_t) may be modelled dynamically. A simple error-correction representation is:

[
Delta x_t = Pi x_{t-1} + sum_{k=1}^{p-1} Gamma_k Delta x_{t-k} + e_t,
qquad Pi = alphaeta^	op.
]

The rank of (Pi) determines the number of independent cointegrating relations in the system. Johansen's framework estimates and tests the cointegration rank in a multivariate system; Engle-Granger provides a classic two-step framework for a single relation.

### 5.2 A spread is not automatically a trade

A stationary spread does not guarantee a profitable mean-reversion strategy. A trading rule needs an explicit conditional distribution for future spread changes, a forecast horizon, executable entry/exit prices, and a risk model.

For a scalar spread with an approximate AR(1) model:

[
z_t - mu = phi(z_{t-1}-mu) + eta_t,
]

mean reversion requires a stable (|phi|<1) under the fitted specification. If (0<phi<1), the approximate half-life is:

[
h_{1/2} = rac{log(0.5)}{log(phi)}.
]

This half-life is a model-derived diagnostic, not a guaranteed holding period. It is unreliable if parameters drift, the spread is nonlinear, or the fitted dynamics are misspecified. For (phileq0), the usual positive-persistence half-life interpretation is not appropriate.

### 5.3 Strengths

- Provides a formal statistical concept of a stationary long-run combination of non-stationary levels.
- Error-correction models can produce a direct, interpretable forecast of adjustment dynamics.
- A multi-variate system can identify more than one independent equilibrium relation.
- Offers a clear set of statistical diagnostics and testable assumptions.

### 5.4 Weaknesses and FX-specific complications

- Cointegration may be regime-dependent and may disappear after structural breaks.
- Pair selection across many combinations creates severe multiple-testing and selection-bias risk.
- Estimated hedge ratios can be unstable and sensitive to sample window and deterministic terms.
- FX crosses have algebraic triangular identities. Including redundant crosses can create rank deficiency or apparently impressive but economically empty relations.
- A mathematically stationary spread may be too slow, too volatile, or too expensive to trade.
- Standard asymptotic test assumptions may be poor in finite samples with heteroskedasticity, heavy tails, serial dependence, and structural breaks.
- A cointegrating vector may imply awkward or infeasible legs, mismatched contract sizes, or concentrated currency exposure.

### 5.5 Avoiding algebraic redundancy

Do not indiscriminately feed every available FX cross into a Johansen system. For (N) currencies, choose a connected spanning set of (N-1) independent exchange-rate coordinates as a starting basis, or explicitly model the rank and constraints induced by the full graph. The exact basis is a coordinate choice, not an economic claim that those pairs are uniquely important.

When a candidate cointegrating basket is expressed in traded pairs, translate it back to net currency exposures. Reject apparent diversification that is only duplicated currency exposure or a triangular identity.

## 6. Direct comparison

| Dimension | Currency-factor returns | Multi-pair cointegration |
|---|---|---|
| Primary object | Synchronized pair returns | Log price levels |
| Main question | Shared currency shocks and residual return behaviour | Stationary linear combinations of non-stationary series |
| Typical output | Currency shocks, fitted returns, residual innovations | Cointegrating vectors, equilibrium errors, error-correction dynamics |
| Mean reversion implied? | No | Stationarity of the fitted combination, not profitability |
| Universe | Broad connected panel | Smaller selected system or independent basis |
| Main benefit | Cross-sectional structure and portfolio attribution | Explicit long-run equilibrium hypothesis |
| Main risk | Confusing explanation with prediction | Unstable relation and multiple testing |
| FX-specific issue | Quote orientation and asynchronous cross-rates | Triangular identities and redundant crosses |
| Natural role | First baseline, risk model, candidate conditional forecaster | Separate candidate alpha model where evidence supports it |

These are complementary rather than mutually exclusive. A factor model can describe common movements and exposures around a cointegration strategy. But the combination should be tested incrementally; do not add factors, HMMs, GARCH, copulas, or Bayesian layers simply because they are available.

## 7. Proposed empirical study

### 7.1 Freeze the experiment before looking at results

Write down and version:

- Currency universe and inclusion/exclusion rules.
- Broker/feed and exact symbol mapping.
- Bar frequency and timestamp convention.
- Training, validation, and final untouched test periods.
- Candidate model families and maximum parameter/search budget.
- Cost assumptions and stress scenarios.
- Portfolio constraints and performance metrics.
- Statistical tests, stability criteria, and rejection rules.

Record every attempted configuration, not just the winner.

### 7.2 Common data set

Start with a liquid G10 subset supported by the broker's history. Use one consistent feed and a common timestamp grid. Store bid, ask, mid, spread, and missingness. Start at a modest intraday or hourly frequency only after checking the feed's timestamp and quote quality; do not select the frequency based on the best backtest.

Do not assume all symbols are tradable at all times. Exclude or separately model rollover, weekends, holidays, market closures, and illiquid sessions.

### 7.3 Experiment A — factor model

1. Construct the pair-by-currency incidence matrix (B) from quote metadata.
2. Verify cross-rate sign conventions using hand-checked triangular identities.
3. Estimate constrained currency returns on each timestamp using only observed pairs and a documented missing-data rule.
4. Compare the structured least-squares model with a small PCA benchmark.
5. Measure in-sample and rolling out-of-sample explained covariance/variance, factor stability, and residual distribution.
6. Test predeclared lagged residual features against a zero-return forecast and a currency-factor-only forecast.
7. Evaluate calibration and conditional forecast value, not only directional accuracy.
8. Convert forecasts into a conservative portfolio only after the forecasting experiment is frozen.

**Key null:** the residual features add no predictive information beyond the baseline.

### 7.4 Experiment B — cointegration

1. Select candidate systems using predeclared economic or graph criteria, not exhaustive winner-picking.
2. Check integration order and deterministic components; do not apply standard (I(1)) cointegration tests blindly to arbitrary series.
3. Use Engle-Granger for a small two-series hypothesis and Johansen for genuinely multivariate systems.
4. Select lag length and deterministic terms within training data only.
5. Check residual stationarity with appropriate critical values, residual autocorrelation, heteroskedasticity, and parameter stability.
6. Re-estimate only on scheduled training windows and evaluate the relation in the next unseen window.
7. Record how often each relationship survives, its coefficient drift, spread volatility, half-life uncertainty, and break frequency.
8. Build a trade simulation only for relations that pass predeclared stability gates.

**Key null:** candidate cointegration is absent, unstable, or not monetizable after costs.

### 7.5 Walk-forward and leakage controls

- Use chronological walk-forward evaluation. Random shuffling is inappropriate for the primary time-series test.
- Keep a final untouched period that is not used for feature, model, or threshold selection.
- Fit scaling, PCA, hedge ratios, factor loadings, hyperparameters, and regime labels on training data only.
- Apply a purge/embargo around boundaries when labels or holding periods overlap.
- Ensure signals are computed using information available before the order could be sent.
- Simulate bid/ask entry and exit, commissions, swaps/financing, slippage, rejected orders, partial fills, and realistic execution delay.
- Stress wider spreads, adverse slippage, delayed execution, missing bars, and gaps.
- Aggregate P&L by currency exposure, not only by pair, and include concurrent positions in portfolio risk.

### 7.6 Metrics

Report at minimum:

- Forecast: out-of-sample (R^2) relative to a declared benchmark, rank/information coefficient where suitable, directional calibration, log score or Brier score for probabilistic forecasts.
- Trading: net return, volatility, Sharpe/Sortino with caveats, maximum drawdown, tail loss/expected shortfall, turnover, average holding period, exposure, cost as a fraction of gross edge, and capacity sensitivity.
- Stability: performance by year/session/currency/regime, parameter drift, signal frequency, relation survival rate, and performance degradation from train to test.
- Statistical reliability: confidence intervals using dependence-aware resampling, multiple-testing adjustment, and probability-of-backtest-overfitting analysis where the candidate search is large enough.

Do not use a single Sharpe ratio as the acceptance criterion. Report the number of configurations tried and include failed experiments.

## 8. Decision gates

The following are proposed research gates, not universal constants. Exact thresholds must be set before the test period is inspected.

| Gate | Required evidence | Failure action |
|---|---|---|
| Data integrity | Synchronized timestamps, correct quote orientation, documented missingness, reproducible prices | Stop; repair data pipeline |
| Statistical validity | Assumptions checked, stable estimates, diagnostics reported | Reject or narrow model |
| Predictive value | Incremental out-of-sample forecast value over simple baselines | Reject alpha claim |
| Economic value | Positive net expectancy under conservative executable costs | Reject trade strategy |
| Robustness | Survives time splits, reasonable parameter perturbations, and cost stress | Reject or classify as fragile |
| Portfolio value | Improves portfolio after currency aggregation and tail-risk constraints | Do not add to portfolio |
| Operational safety | Deterministic signals, bounded risk, reconciliation and fail-safe design | No live execution |

## 9. Provisional recommendation

**Prioritize the currency-factor representation as the first shared research foundation.** It makes the currency graph explicit, reduces redundant pairwise thinking, and creates a useful common risk/exposure language for every later strategy. Its first deliverable is an explanatory and diagnostic model, not a contrarian trading rule.

In parallel, run a deliberately small cointegration study on independent, economically defensible candidate systems. Promote cointegration only when the relation is stable across rolling training windows and yields net out-of-sample value after costs.

Do not yet select OU thresholds, Z-score entries, fixed holding periods, HMM filters, or Kelly sizing. Those depend on the forecast distribution and evidence that have not yet been established. Position sizing should eventually use conservative, calibrated forecasts and portfolio-level currency exposures, with hard risk limits independent of model confidence.

### Next research task

Specify the data schema and construct a quote-convention-safe G10 currency graph/pair panel. Then run a descriptive study (no strategy optimization) of factor fit, factor stability, residual dependence, triangular consistency, and data quality. This provides the shared data foundation for both candidate model families.

## 10. Literature and source notes

1. **Aloosh, A. & Bekaert, G. (2019; published 2022), “Currency Factors.”** NBER Working Paper 25449. Motivates modelling currency-basket comovement and reports a structured factor model with clustering, commodity, and world factors. It is a useful research starting point, not evidence that the same factors predict returns in this broker's data. [NBER paper](https://www.nber.org/papers/w25449).
2. **Engle, R. F. & Granger, C. W. J. (1987), “Co-integration and Error Correction: Representation, Estimation, and Testing.”** *Econometrica*, 55(2), 251–276. Foundational representation linking cointegration and error-correction models. [Bibliographic record and abstract](https://ideas.repec.org/a/ecm/emetrp/v55y1987i2p251-76.html).
3. **Johansen, S. (1988), “Statistical Analysis of Cointegration Vectors.”** *Journal of Economic Dynamics and Control*, 12(2–3), 231–254. Develops maximum-likelihood estimation and rank testing for multivariate cointegrating vectors. [DOI record](https://doi.org/10.1016/0165-1889(88)90041-3).
4. **Chaboud, A., Rime, D. & Sushko, V. (2023), “The Foreign Exchange Market.”** BIS Working Paper 1094. Provides institutional and market-structure context for the OTC FX market, electronic trading, intermediation, and execution frictions. [BIS publication](https://www.bis.org/publ/work1094.htm).
5. **Bailey, D. H., Borwein, J., López de Prado, M. & Zhu, Q. J. (2017), “The Probability of Backtest Overfitting.”** *Journal of Computational Finance*. Introduces combinatorially symmetric cross-validation as a framework for assessing investment backtest overfitting. [Author paper record](https://escholarship.org/uc/item/4w1110bb).
6. **Bailey, D. H. & López de Prado, M. (2014), “The Deflated Sharpe Ratio: Correcting for Selection Bias, Backtest Overfitting and Non-Normality.”** *The Journal of Portfolio Management*, 40(5), 94–107. Discusses Sharpe-ratio inflation from selection bias and non-normal returns. [DOI record](https://doi.org/10.2139/ssrn.2460551).

### Interpretation cautions

- These sources establish concepts and motivate methods; they do not validate this project's data, broker, frequency, or strategy.
- The currency-factor paper studies currency comovement and asset-pricing structure. Its reported explanatory results must not be reinterpreted as an executable trading edge.
- Cointegration tests have assumptions and finite-sample limitations. Test selection, deterministic terms, structural breaks, and repeated search all matter.
- PBO/DSR are safeguards, not magic corrections. They do not repair look-ahead bias, bad data, unrealistic fills, or an unrepresentative test period.

---

## Research log

| Date | Entry |
|---|---|
| 2026-11-10 | Initial comparison note created. Provisional priority: currency-factor return panel as shared foundation; cointegration as a parallel, separately validated strategy family. |
