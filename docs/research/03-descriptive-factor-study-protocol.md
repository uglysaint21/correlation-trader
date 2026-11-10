# Research Protocol 01: Descriptive G10 Panel and Factor Diagnostics

**Status:** Proposed preregistration template  
**Date:** 10 November 2026  
**Purpose:** Define the first empirical study before inspecting performance results. This protocol is descriptive and diagnostic, not a strategy backtest.

## 1. Objective

Establish whether the broker's available FX data can support a coherent, synchronized G10 currency panel and whether a currency-incidence factor model provides a stable description of cross-pair return variation.

The study asks:
1. Is the observed broker instrument graph sufficiently connected and correctly mapped?
2. Are quotes and bars synchronized and reliable enough for cross-sectional analysis?
3. Can currency factors be estimated consistently across the observed graph?
4. What statistical structure remains in pair-specific residual returns?

Do not optimize entries, exits, thresholds, holding periods, or position sizes.

## 2. Scope

Included: instrument catalogue, G10 spot-FX mappings, bid/ask/mid data, UTC synchronization, graph connectivity, incidence-matrix algebra, descriptive factor fit, residual diagnostics, and stability analysis.

Excluded: live orders, strategy optimization, profitability claims, leverage decisions, portfolio sizing, machine-learning feature selection, and use of final test data for model choice.

The ten-currency G10 definition is a research target, not a guarantee of full broker coverage.

## 3. Required dataset manifest

Freeze and version:
- Broker and MT5 server identity (non-secret identifiers only).
- MT5 terminal and Python package versions.
- Retrieval timestamp and historical date range.
- Exact broker symbol catalogue and contract metadata snapshots.
- Normalized base/quote mapping and verification method.
- Instrument inclusion/exclusion list and reasons.
- Timeframe(s) and UTC boundary schedule.
- Quote synchronization rule and maximum quote age.
- Treatment of market closures, rollover, holidays, and missing observations.
- Price source (bid/ask/mid) and transformation version.
- Data-quality thresholds and excluded intervals.

If an item is unknown, record it as unknown rather than silently supplying a default.

## 4. Pre-analysis checks

### Universe and graph
- Confirm each instrument is spot FX.
- Verify base/quote metadata against broker contract specifications.
- Build the directed graph and incidence matrix.
- Report nodes, edges, connected components, degree per currency, and matrix rank.
- Compare observed coverage with the 45 possible unordered G10 pairs.
- Identify reverse duplicates and inconsistent mappings.
- Select a deterministic spanning tree and record its construction rule.

### Quote and bar integrity

Report by instrument and session:
- Missing intervals and duplicate timestamps.
- Bid/ask invalidity and crossed quotes.
- Median, quantiles and extremes of absolute and relative spread.
- Quote-age distribution and stale-observation frequency.
- Bar gaps and incomplete-bar frequency.
- Tick/bar history coverage and terminal-history limits.
- Weekend, rollover, holiday and maintenance behaviour.

### Synchronization sensitivity

Run panel construction under a small, predeclared set of quote-age tolerances appropriate to the sampling interval. Select tolerances based on timestamp granularity and market activity, not on which produces the best factor fit.

For each tolerance, report:
- Percentage of target timestamps with a fully connected graph.
- Percentage with at least one missing edge.
- Connected-component count and rank distribution.
- Pairwise quote-age quantiles.
- Triangle-residual distribution on eligible triples.
- Number and proportion of excluded observations.

Do not fill missing values simply to make the graph complete.

## 5. Factor estimation

At each eligible timestamp t, estimate:

\[
\hat f_t=\arg\min_f (r_t-B_t f)^\top W_t(r_t-B_t f)
\]

subject to:

\[
\mathbf{1}^\top f=0.
\]

Here B(t) is built from pairs actually observed at t, and W(t) is a predeclared weighting matrix. The baseline starts with equal weights on valid pair returns. Any spread/quality weighting is a separate sensitivity experiment and must be justified independently.

If the observed graph is disconnected, estimate factors by connected component and apply a component-specific normalization. Do not compare component factor levels as if they shared one global numeraire.

Compare:
1. **Structured incidence OLS:** the constrained model above.
2. **Robust sensitivity estimator:** a documented robust loss/outlier-resistant procedure, only after the OLS baseline is frozen.
3. **PCA benchmark:** fit PCA within analysis windows only; report component rotations and explain why PCA components are not named currency factors.

### Algebraic controls

- Verify B times the all-ones vector equals zero.
- Verify rank(B) = N - C for a graph with C connected components, considering only currencies incident to observed edges.
- Verify every edge has one +1 and one -1.
- Verify reversing an edge reverses its return and incidence row.
- Verify fitted pair returns and residuals are consistent under equivalent spanning-tree bases.

## 6. Descriptive statistics

Report by pair, currency, session, and rolling window.

### Factor fit
- Cross-sectional R-squared or explained sum-of-squares with denominator stated.
- Residual sum-of-squares and distribution.
- Factor covariance and correlation.
- Factor volatility and contribution to pair variance.
- Observation count and effective graph rank.

Do not present cross-sectional R-squared as a time-series forecasting R-squared.

### Residual behaviour
- Mean, standard deviation, skewness, kurtosis and quantiles.
- Autocorrelation at predeclared lags.
- Cross-pair residual covariance/correlation.
- Conditional heteroskedasticity diagnostics.
- Residual dependence by session and volatility bucket.
- Sensitivity to quote-age threshold, universe composition and price sampling.

These are diagnostic statistics. They do not establish a tradable forecast.

### Stability
- Rolling factor covariance and graph changes.
- Pairwise residual variance and covariance drift.
- Subsample comparisons by calendar period and session.
- Stability under removal of one pair or one currency edge.
- Stability under alternative predeclared sampling and synchronization choices.

## 7. Null comparisons

Use transparent baselines:
- Zero currency-factor return forecast.
- Historical mean return estimated strictly on the analysis window.
- Pairwise sample covariance without factor structure.
- Simple descriptive market-basket benchmark where meaningful.

Do not choose the estimator because it has the best apparent future-return performance. Predictive tests require a separate protocol and untouched evaluation period.

## 8. Deliverables

The research report must contain:
1. Frozen dataset manifest and universe mapping.
2. Graph edge list and diagram/table.
3. Incidence matrix definition, rank, connected components and spanning tree.
4. Data-quality report by instrument and session.
5. Synchronization and triangle-consistency diagnostics.
6. Factor-estimation method and normalization.
7. Factor-fit, residual and stability results.
8. Sensitivity analysis and limitations.
9. Explicit go/no-go decision for predictive research.
10. Reproducibility details sufficient to reconstruct every table.

Report all exclusions and failed diagnostics.

## 9. Go/no-go criteria

Proceed to predictive research only if:
- Instrument base/quote mappings are verified.
- Graph structure and incidence rank behave as expected.
- UTC timestamps and synchronization rules are unambiguous.
- Data gaps and stale observations are measurable and controlled.
- Triangle inconsistencies are explainable or bounded by documented data-quality rules.
- Factor estimates are reproducible under equivalent graph bases.
- Factors and residuals can be reconstructed from immutable raw data.

If these conditions fail, repair the data foundation first. A sophisticated model cannot compensate for inconsistent input data.

## Research log

| Date | Entry |
|---|---|
| 2026-11-10 | Initial descriptive panel protocol drafted. No empirical results or trading claims recorded. |
