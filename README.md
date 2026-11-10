# Correlation Trader

Research and development of a locally operated, multi-currency quantitative trading system connected to MetaTrader 5 (MT5).

The project investigates whether statistical relationships and dependencies across Forex instruments can be turned into **robust, cost-aware and independently validated trading opportunities**. The intended system will combine currency-factor modelling, statistical arbitrage, probabilistic inference, market-regime detection and portfolio-level risk management.

> **Project status: Research and strategy design.**
>
> This repository is not yet a live trading application. No strategy has been demonstrated profitable, no execution code is currently provided, and no live-trading use is supported. Research hypotheses must be specified, tested out of sample and approved before implementation.

## Objectives

The eventual system is intended to:

- Run locally on the user's computer, with MT5 providing broker connectivity, market data and order execution.
- Retrieve historical data and process live bid, ask and tick updates for multiple Forex symbols.
- Model shared currency drivers and pair-specific statistical relationships.
- Estimate conditional outcome probabilities, uncertainty, volatility and expected holding time.
- Adapt or disable strategies when relationships become unstable or market regimes are unsuitable.
- Make portfolio-aware decisions that aggregate exposure to the same underlying currencies.
- Account for spread, commission, swaps, slippage, latency, rejected orders and other execution constraints.
- Preserve reproducible research, model estimates, decisions, orders and performance records.

The primary objective is not to predict an exact future price. It is to identify situations where the estimated distribution of net trading outcomes offers a defensible positive expected value after costs and uncertainty.

## Research principles

The project follows a research-first workflow. Mathematical sophistication is not evidence of an edge.

1. **Evidence before complexity.** Start with interpretable baselines. Retain a complex model only if it adds measurable out-of-sample value.
2. **Relationships are hypotheses.** Correlation does not imply causation, predictability or cointegration. Relationships can change or disappear.
3. **Respect market structure.** Forex pairs share currencies and are algebraically linked. Do not mistake mechanical cross-rate relationships for independent alpha.
4. **No leakage.** Every feature, parameter estimate, universe selection and decision in a simulation must use information available at that historical time.
5. **Costs are part of the model.** Evaluate executable bid/ask prices and realistic transaction costs, not frictionless mid-price returns.
6. **Portfolio first.** Aggregate base- and quote-currency exposures across positions; do not manage each symbol in isolation.
7. **Uncertainty matters.** Account for parameter uncertainty, multiple testing, probability calibration and structural breaks.
8. **Fail safely.** Uncertain data, disconnected terminals, stale quotes, invalid orders or breached risk limits must prevent new exposure.
9. **Reproducibility.** Record data provenance, configuration, code version, model parameters, random seeds and experiment results.
10. **No profitability claims without evidence.** Backtests are research results, not guarantees of future performance.

## Research documentation

The first research note compares currency-factor return models with multi-pair cointegration, explains their distinct statistical meanings, and defines a controlled empirical study: [Research Note 01 — Currency-Factor Models vs. Multi-Pair Cointegration](docs/research/01-factor-models-vs-cointegration.md).

## Research scope

The techniques below are candidates for investigation, not a commitment to use all of them.

### 1. Currency factors and cross-pair structure

- Represent log exchange rates as differences between latent currency values, for example
  \[
  p_{A/B,t}=f_{A,t}-f_{B,t}+\epsilon_{A/B,t}.
  \]
- Use PCA, factor models and potentially dynamic factor models to estimate common currency drivers.
- Investigate whether residual pair movements contain information beyond broad currency factors.
- Respect cross-rate identities such as approximately
  \[
  EURJPY = EURUSD\times USDJPY,
  \]
  subject to quote conventions, timing and executable bid/ask prices.

### 2. Relationship discovery and statistical arbitrage

- Pearson and Spearman dependence, rolling correlations and cross-correlation / lead-lag analysis.
- Cointegration testing using Engle-Granger and Johansen methods.
- Error-correction models, spread construction and stationarity diagnostics.
- Multiple-testing controls and stability checks for a large candidate universe.
- Copula and tail-dependence models where they add demonstrable value.

Correlation is a screening and descriptive tool; it is not, by itself, a trading signal. Cointegration tests require careful treatment of integration order, deterministic terms, lag selection, sample length and repeated testing.

### 3. Spread and volatility dynamics

For a candidate two-instrument relationship, a starting spread specification may be

\[
s_t=\log(P_{1,t})-\beta_t\log(P_{2,t}),
\]

where \(\beta_t\) is an estimated hedge ratio. Candidate estimators include rolling regression and Kalman filtering. A normalized deviation can be defined as

\[
z_t=\frac{s_t-\hat\mu_t}{\hat\sigma_t}.
\]

An Ornstein-Uhlenbeck process is one candidate model for mean-reverting spread dynamics:

\[
dS_t=\theta(\mu-S_t)dt+\sigma dW_t.
\]

Its implied half-life, \(\ln(2)/\theta\), is a model-based diagnostic, not a guaranteed holding period. The OU assumptions and parameter stability must be tested rather than presumed.

GARCH-family models, Student-t innovations and other conditional-distribution models may be investigated for volatility and tail-risk estimation.

### 4. Probabilistic modelling and regimes

Potential methods include:

- Bayesian estimation and sequential updating.
- Hidden Markov Models and Markov-switching models.
- Change-point and structural-break detection.
- Gaussian and non-Gaussian conditional distributions, including Student-t and mixture models.
- Monte Carlo simulation of spread and portfolio paths.
- Extreme-value methods for tail-risk analysis.

The intended outputs are calibrated conditional probabilities and distributions, such as the probability of reaching a target before a stop within a specified horizon, expected adverse excursion, expected holding time and probability of relationship deterioration. Forecast probabilities must be evaluated for calibration and uncertainty.

### 5. Trade and portfolio decisions

A candidate trade should be evaluated on expected **net** outcome, not win probability alone. A simplified expression is

\[
EV=P_{TP}G-P_{SL}L-C,
\]

where \(G\) and \(L\) are the gross gain and loss under explicitly defined outcomes and \(C\) represents all relevant costs. A production decision would also need to account for other outcomes, path dependence, uncertainty and portfolio interactions.

Position sizing research may include volatility targeting, constrained portfolio optimization and conservative fractional-Kelly variants. All methods remain subject to hard limits on leverage, drawdown, per-currency exposure, concentration and operational risk.

## Candidate system architecture

The eventual application is expected to have clear boundaries between research, decision-making and execution. A Python-led design with an MT5 integration layer is a candidate; Python, MQL5 and hybrid alternatives will be evaluated before implementation.

| Component | Responsibility |
| --- | --- |
| MT5 adapter | Terminal connectivity, symbol metadata, historical rates, ticks, account and order interfaces |
| Market-data layer | Timestamp alignment, quote validation, resampling and persistent raw data |
| Feature and factor engine | Returns, currency factors, dependence estimates and candidate relationships |
| Statistical model layer | Cointegration, hedge ratios, spread dynamics, volatility and regime models |
| Probability and decision layer | Conditional outcome distributions, calibration and trade eligibility |
| Portfolio and risk engine | Currency-level exposure, sizing, limits and circuit breakers |
| Execution and reconciliation | Order validation, submission, fills, retries, position reconciliation and protective orders |
| Research and evaluation | Event-driven backtesting, walk-forward tests, diagnostics and reports |
| Local persistence | Versioned datasets, configurations, experiments, decisions and execution records |
| Monitoring interface | Health, model state, exposure, orders, risk and operational alerts |

**Execution must be independently guarded.** A model signal is only a proposal. Before any order is sent, the execution/risk boundary must validate data freshness, symbol trading conditions, account state, existing exposure, order size, stops and configured limits.

All core computation and storage are intended to remain local. External research sources may be consulted during development, but the trading runtime should not depend on a hosted modelling service.

## Backtesting and validation standard

A credible evaluation framework must be designed before a strategy is considered for implementation.

### Data and simulation requirements

- Use synchronized multi-symbol data with explicit timezone, session and missing-data handling.
- Preserve bid/ask information where available; do not assume mid-price fills.
- Model spread, commission, swaps, slippage, latency, partial fills, rejected orders and execution constraints.
- Prevent look-ahead bias in rolling windows, factor estimation, pair selection, hedge-ratio updates and regime classification.
- Simulate portfolio-level currency exposure and margin, not isolated per-pair P&L.
- Make assumptions and data limitations explicit, including broker-specific symbol specifications.

### Validation procedure

1. Establish simple, transparent baselines before testing complex models.
2. Separate training, validation and untouched test periods chronologically.
3. Use walk-forward evaluation with all selection and fitting repeated using only past data.
4. Control for multiple comparisons and selection bias when screening many pairs, windows and parameters.
5. Run parameter sensitivity and stability analysis across market periods and regimes.
6. Use block/bootstrap or Monte Carlo methods that preserve relevant dependence where appropriate.
7. Evaluate probability calibration, not just classification accuracy or win rate.
8. Compare gross and net results and report the impact of each cost assumption.
9. Record failed experiments and avoid repeatedly tuning against the final holdout.

### Metrics

Report, at minimum, net return, expectancy, Sharpe and Sortino ratios, maximum drawdown, profit factor, win/loss distribution, turnover, trade duration, exposure, leverage, tail losses and performance by symbol, currency, regime and time period. Include sample sizes and uncertainty intervals. No single metric is sufficient to establish robustness.

A strategy should progress to paper trading only after pre-defined out-of-sample acceptance criteria are met. Live deployment, if ever considered, requires a separate operational and risk review.

## Development roadmap

Work proceeds through explicit gates. Implementation of the autonomous trading system is intentionally deferred.

| Stage | Deliverable | Exit condition |
| --- | --- | --- |
| 1. Quantitative research | Literature review and comparison of candidate methods | Assumptions, limitations and promising hypotheses documented |
| 2. Strategy formulation | Pair/factor selection, signal, entry/exit, holding-time and sizing logic | One coherent strategy proposal reviewed |
| 3. Mathematical specification | Equations, estimators, parameters and decision rules | Rules are precise enough to implement and test |
| 4. Validation design | Data requirements, backtest protocol and acceptance criteria | Leakage, costs and selection bias controls specified |
| 5. System architecture | Local MT5, data, modelling, risk and execution boundaries | Architecture and operational failure modes reviewed |
| 6. Implementation | Incremental research tooling, backtester and eventually MT5 integration | Each increment tested and reviewed; no live trading by default |

### Immediate research priorities

1. Establish a reliable representation of currency returns and shared currency factors.
2. Compare factor residuals with pairwise cointegration opportunities.
3. Specify robust relationship discovery, stability and multiple-testing controls.
4. Evaluate static, rolling and Kalman-filter hedge-ratio estimators.
5. Test OU-style spread models against simpler empirical alternatives.
6. Define probability targets and calibration procedures for target-before-stop outcomes.
7. Evaluate regime filters only against a baseline without a regime filter.
8. Design a cost-aware, portfolio-level validation protocol.

## Repository status and planned structure

This repository is currently at its initial documentation stage. The README and license are present; the research, strategy, backtesting and MT5 application have not yet been implemented.

As decisions are approved, the repository may evolve toward a structure such as:

\`\`\`
correlation-trader/
├── README.md
├── LICENSE
├── docs/
│   ├── research/
│   ├── strategy/
│   ├── validation/
│   └── architecture/
├── config/
├── data/                 # local data; do not commit private or broker data
├── research/
├── src/
│   ├── data/
│   ├── factors/
│   ├── relationships/
│   ├── models/
│   ├── signals/
│   ├── portfolio/
│   ├── risk/
│   ├── execution/
│   └── monitoring/
├── tests/
└── scripts/
\`\`\`

This is a proposed layout, not a claim that these directories or modules already exist. The final package boundaries and technology choices will follow the approved strategy and architecture. Local datasets, credentials, terminal configuration and account information must never be committed.

## Safety and scope

- Research and backtesting are the current scope.
- No live order execution should be enabled by default in future implementations.
- Never commit passwords, API keys, broker credentials, account identifiers, private market data or local terminal files.
- Use a demo account and explicit operator approval for any future execution testing.
- Include kill switches, maximum-loss controls, stale-data checks, connection-loss handling and order/position reconciliation before any autonomous operation.
- Trading involves substantial risk. Historical or simulated performance does not guarantee future results.

## Selected foundational references

These are starting points for the literature review, not endorsements of any specific trading rule.

- Engle, R. F. & Granger, C. W. J. (1987). [Co-integration and Error Correction: Representation, Estimation, and Testing](https://doi.org/10.2307/1913236). *Econometrica*.
- Johansen, S. (1988). [Statistical Analysis of Cointegration Vectors](https://doi.org/10.1016/0165-1889(88)90041-3). *Journal of Economic Dynamics and Control*.
- Bailey, D. H., Borwein, J. M., López de Prado, M. & Zhu, Q. J. (2017). [The Probability of Backtest Overfitting](https://escholarship.org/uc/item/4w1110bb).
- Hamilton, J. D. (1989). [A New Approach to the Economic Analysis of Nonstationary Time Series and the Business Cycle](https://doi.org/10.1257/aer.79.2.357). *Econometrica*.
- Research on currency factors, dynamic factor models, market microstructure and realistic execution costs will be added as the research questions are narrowed.

## Contributing and decision process

The project is developed incrementally. Research proposals should state:

- The hypothesis and economic/statistical rationale.
- Required data and sampling frequency.
- Estimation method and assumptions.
- Baseline and competing approaches.
- Leakage, multiple-testing and transaction-cost controls.
- Out-of-sample acceptance criteria.
- Known failure modes and operational implications.

A technique is not adopted merely because it is sophisticated or performs well in one historical sample. Strategy and architecture decisions should be documented before implementation.

---

**Current next step:** complete the quantitative research and choose a testable baseline strategy. No trading code should be written until the strategy, mathematical specification and validation plan have been reviewed and approved.
