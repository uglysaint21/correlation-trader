# Research Note 02: G10 Currency Graph and Market-Data Contract

**Status:** Proposed research specification  
**Date:** 10 November 2026  
**Decision status:** Design baseline; broker-specific symbol availability and field semantics must be verified before implementation.

## 1. Purpose

This note defines the initial currency universe, graph representation, pair-orientation rules, time conventions, and canonical data contract. It is intended to prevent quote-direction errors, duplicated or algebraically redundant instruments, asynchronous observations, silent missing-data handling, and confusion between model prices and executable prices.

This is a specification, not an implementation. It does not assume the broker offers every theoretical cross or that all pairs are equally liquid or tradable.

## 2. Initial currency universe

Use the conventional G10 currency set as the initial research universe:

| Code | Currency |
|---|---|
| AUD | Australian dollar |
| CAD | Canadian dollar |
| CHF | Swiss franc |
| EUR | Euro |
| GBP | Pound sterling |
| JPY | Japanese yen |
| NOK | Norwegian krone |
| NZD | New Zealand dollar |
| SEK | Swedish krona |
| USD | US dollar |

Ten currencies imply at most 10 choose 2 = **45 distinct unordered currency relationships**, if all crosses are available. This is a theoretical graph, not a requirement to trade or even source all 45 pairs.

The initial broker-supported universe must be discovered from the connected MT5 terminal's actual symbol catalogue. Symbol names are not standardized across brokers: suffixes, prefixes, contract variants, and disabled symbols are possible. Store the broker symbol separately from the normalized currency pair.

A symbol may enter the research panel only when its contract metadata unambiguously identifies base and quote currencies; it is classified as spot FX; bid/ask and timestamps are available at the required resolution; history is sufficiently continuous; and it passes data-quality checks. Do not silently substitute CFDs, futures, synthetic indices, or non-deliverable products for spot FX.

Maintain a versioned universe manifest containing universe identifier, effective dates, normalized base/quote codes, exact broker symbol, broker/server identity, instrument type, contract metadata, inclusion/exclusion reason, first/last observation, mapping source, and verification timestamp. Historical membership must be point-in-time.

## 3. Currency graph

Represent currencies as vertices and observed currency pairs as directed edges. For a pair quoted as units of currency j per one unit of currency i:

\[
S_{i/j,t}=\frac{\text{units of }j}{1\text{ unit of }i}.
\]

Its log mid-price is p(i/j,t) = log(S mid). A positive return means the base currency appreciated against the quote currency.

For N currencies and M observed pairs, construct an incidence matrix B in R^(M x N). For each edge i/j, the row has +1 in the base-currency column, -1 in the quote-currency column, and zero elsewhere. The return model is:

\[
r_t=Bf_t+\epsilon_t.
\]

For a connected graph, rank(B) = N - 1. The common level of all currency factors is unidentified because B times the all-ones vector is zero. Apply a documented normalization, such as sum of factors equals zero. This makes factors relative currency returns, not absolute returns.

If the observed graph is disconnected, each connected component has its own unidentified common level. Do not fit one globally interpretable factor vector across disconnected components without explicitly handling that structure.

### Complete graph and independent basis

A complete graph of ten currencies contains 45 edges but only nine independent currency contrasts. The extra edges create redundant cycles. A spanning tree contains exactly N - 1 = 9 edges and connects all ten currencies.

Recommended practice:

- Keep every valid broker-supported pair in the raw market-data store.
- Use the full observed graph for consistency diagnostics and robust estimation.
- Use a documented spanning tree or rank-revealing basis when an independent coordinate representation is needed.
- Never treat all 45 pair returns as statistically independent observations.
- Track graph connectivity by timestamp/session; missing edges can change the estimable subspace.

The spanning-tree choice is a computational coordinate choice, not an economic ranking of pairs.

### Quote orientation and inversion

Never infer orientation from a ticker string alone. Verify the symbol's contract metadata and broker documentation.

If the broker quotes the inverse rate, then log price and return change sign. Bid/ask inversion must also swap sides:

\[
Bid_{j/i}=1/Ask_{i/j},\qquad Ask_{j/i}=1/Bid_{i/j}.
\]

Preserve original broker quotes and derive normalized views without overwriting source values. Inverting the mid while forgetting to swap bid and ask creates invalid spreads.

### Triangular consistency

For synchronized quotes:

\[
S_{i/k}=S_{i/j}S_{j/k},\qquad
p_{i/k}=p_{i/j}+p_{j/k}.
\]

Define a log-price cycle residual:

\[
c_{i,j,k,t}=p_{i/k,t}-p_{i/j,t}-p_{j/k,t}.
\]

With synchronous frictionless quotes this is zero. Real broker data can differ because of bid/ask sides, timestamp offsets, stale updates, rounding, and feed construction. Treat this as a data-quality diagnostic, not a trading signal. Executable triangular checks must use side-consistent bid/ask conversion and include all costs. A mid-price cycle residual is not proof of an executable arbitrage.

## 4. Time model and synchronization

### Canonical time

All persisted timestamps must be UTC. Use timezone-aware UTC values in the application and an unambiguous UTC representation in storage. Preserve original source timestamp and precision. Do not store naive local timestamps and later guess their timezone.

MetaQuotes documents that MT5 tick and bar times are stored in UTC and that UTC datetimes should be supplied to range-copy functions. It also notes that available bar history is limited by the terminal's chart-history settings. Official references:
- [copy_ticks_range](https://www.mql5.com/en/docs/python_metatrader5/mt5copyticksrange_py)
- [copy_rates_range](https://www.mql5.com/en/docs/python_metatrader5/mt5copyratesrange_py)

### Tick records

Retain source event time in UTC at the highest available precision; millisecond timestamp where supplied; bid, ask, last, volume and volume-real fields; tick flags; broker/server and instrument identity; local ingestion time in UTC; and retrieval batch identifier.

Do not assume every FX tick has a meaningful last-traded price or exchange volume. OTC FX quote updates are not equivalent to centralized exchange trade prints. Tick volume is a platform activity/update measure, not consolidated global FX traded volume.

### Bars

Define bars using UTC interval boundaries and explicit interval semantics. Store bar open time, timeframe, OHLC source fields, bid/ask fields if available, tick volume, reported spread, real volume, observation count, source batch and quality flags.

MT5 standard rates output includes time, OHLC, tick volume, spread and real volume. It does not by itself provide a full bid/ask OHLC history for every bar. Do not pretend one bar spread field reconstructs the executable bid/ask path through the bar. For realistic intrabar execution, tick data or a separately validated bid/ask bar source is required.

### Synchronization policy

For each model interval:

1. Define one UTC boundary schedule for the whole panel.
2. Choose a price observation rule, such as last valid quote at or before the boundary, with a maximum permitted age.
3. Require freshness and market-open checks.
4. Store observation age and synchronization diagnostics.
5. Mark absent or stale observations as missing; do not silently forward-fill.
6. Estimate factors using the observed subgraph and record connectivity and rank.
7. Exclude or separately classify intervals where the graph is disconnected or materially incomplete.

A common bar label does not guarantee synchronous underlying quotes. Sensitivity to synchronization tolerance must be measured.

Store session state sufficient to distinguish normal trading, rollover/maintenance, weekend closure, holidays, market reopen, and exceptional outages. Do not assume Forex is continuously liquid or that broker availability equals market-wide liquidity.

## 5. Canonical data contract

Use layered storage. Preserve raw broker observations immutably; normalized records and analytical features are derived products and must be rebuildable.

### Instrument mapping

One effective-dated record per broker instrument mapping:

| Field | Meaning |
|---|---|
| instrument_id | Stable internal identifier |
| broker_id, server_id | Broker and MT5 server identity |
| broker_symbol | Exact MT5 symbol including suffix/prefix |
| instrument_type | Spot FX / CFD / other |
| base_currency, quote_currency | Normalized currency codes |
| digits, point, trade_tick_size | Price precision metadata |
| contract_size | Contract units reported by broker |
| volume_min, volume_max, volume_step | Tradable volume constraints |
| trade_mode | Broker-reported availability |
| swap_long, swap_short, swap_mode | Carry metadata with effective time |
| effective_from_utc, effective_to_utc | Mapping validity |
| metadata_observed_at_utc | Retrieval timestamp |

Metadata can change. Store snapshots or effective-dated versions.

### Tick quote

One row per source tick/update:

| Field | Meaning |
|---|---|
| broker_id, server_id, instrument_id | Composite instrument identity |
| event_time_utc, event_time_msc | Source event time and precision |
| ingested_at_utc | Local receipt time |
| bid, ask, last | Raw broker quote fields |
| volume, volume_real | Source volume fields |
| flags | Raw tick flags |
| source_batch_id | Retrieval provenance |
| quality_flags | Parsed validation flags |

Timestamp alone may not uniquely identify multiple updates. Use source sequence/flags where available and detect duplicates explicitly.

### Bar

One row per instrument and interval:

| Field | Meaning |
|---|---|
| instrument_id, timeframe | Instrument and interval |
| bar_open_time_utc | UTC interval start |
| open, high, low, close | Source OHLC with documented price basis |
| tick_volume | Source activity count |
| reported_spread | Source field with units documented |
| real_volume | Source field; do not assume meaningful for OTC FX |
| source_batch_id | Retrieval provenance |
| is_complete | Whether the interval has ended |
| quality_flags | Gaps, duplicates, invalid OHLC, stale source |

Unique key should include instrument, timeframe, bar open time, and broker/source identity.

### Synchronized quote

One row per instrument per target timestamp and synchronization-policy version:

| Field | Meaning |
|---|---|
| instrument_id, target_time_utc | Instrument and common panel time |
| source_event_time_utc | Actual quote time selected |
| quote_age_ms | Target/source time difference |
| bid, ask, mid | Selected quote values |
| relative_spread | (ask - bid) / mid |
| is_fresh, is_market_open | Eligibility flags |
| sync_policy_version | Reproducibility |
| quality_flags | Stale, crossed, zero, missing |

Define mid = (bid + ask)/2 only when both sides are positive and ask >= bid. A crossed quote is flagged, not silently repaired.

### Pair return

One row per instrument and return interval:

| Field | Meaning |
|---|---|
| instrument_id, interval_end_utc | Pair and interval end |
| interval_seconds | Actual elapsed interval |
| log_mid_return | Difference of log mid prices |
| simple_mid_return | Optional simple return |
| start_quote_age_ms, end_quote_age_ms | Endpoint freshness |
| is_valid | Explicit eligibility |
| quality_flags | Missing interval, gap, stale |

Never calculate across an unmarked weekend, outage, or missing interval as if it were an ordinary one-period return. Retain elapsed time and gap labels.

### Currency-factor snapshot

One row per timestamp, model version and currency:

| Field | Meaning |
|---|---|
| timestamp_utc, model_version | Time and exact model specification |
| currency | Currency code |
| factor_return | Estimated relative currency return |
| standard_error | If estimator supports it |
| n_pairs_used | Observed incident edges |
| graph_component_id | Connected component |
| design_rank | Observed design matrix rank |

Store fitted pair returns and pair residuals separately under the same model version. Never overwrite raw pair returns with model residuals.

### Data-quality event

Append-only record containing event type, broker/symbol, affected time range, detection time, severity, measured values, threshold/version and disposition. Preserve rejected observations and their exclusion reasons.

## 6. Price and return definitions

For valid bid b and ask a:

\[
m_t=(a_t+b_t)/2,\qquad q_t=(a_t-b_t)/m_t.
\]

Here q is the relative quoted spread. Also retain absolute spread and broker point/tick conventions.

Mid log return is:

\[
r^{mid}_t=\log(m_t)-\log(m_{t-1}).
\]

Mid returns are a modelling convention, not executable P&L. A long normally enters at ask and exits at bid; a short normally enters at bid and exits at ask. Cash P&L also depends on contract size, account-currency conversion, lot size, commissions, swaps, and broker rules.

For an interval with no valid quote at either endpoint, return is missing. For a large elapsed gap, preserve the full elapsed time and flag the interval. Do not divide a multi-hour/weekend return by a nominal bar duration without a separate model.

## 7. Data validation invariants

### Instrument and graph
- Base and quote are distinct supported currency codes.
- No instrument maps to itself or conflicting base/quote metadata in one effective interval.
- Reverse duplicates are detected and not treated as independent instruments.
- Graph components and matrix rank are calculated per timestamp/panel.
- A connected N-currency graph has incidence rank N - 1.
- Each incidence row has exactly one +1, one -1, and zeros elsewhere.

### Quotes and bars
- Bid and ask are positive and ask >= bid.
- OHLC satisfies low <= min(open, close), high >= max(open, close), and low <= high.
- Prices respect tick-size/precision rules within documented tolerance.
- Duplicate timestamps and out-of-order records are detected.
- Quote-age and spread outliers are flagged, not automatically winsorized.

### Time and cross-rates
- Persisted timestamps are UTC.
- Bars align to the declared timeframe schedule; incomplete bars are distinguished.
- Missing intervals, weekend gaps and maintenance are classified.
- Cross-pair synchronization age is measured for every panel row.
- Triangle residuals are measured only on sufficiently synchronized observations.
- Bid/ask triangular checks use correct sides and all transaction costs.
- No apparent arbitrage is accepted from mid-price identities alone.

## 8. Storage and reproducibility

The user's local machine is the intended storage and compute environment. Database/file format remains an architecture decision. The chosen storage must support immutable raw partitions, deterministic derived-data rebuilds, versioned schemas and transformations, UTC timestamps, duplicate detection, compression, incremental ingestion, provenance, backups and integrity checks.

Each dataset should have a manifest recording broker/server, symbol mappings, retrieval range, timeframe, terminal/package version, retrieval time, missingness, transformations, and content/checksum identifiers where practical.

Never store account passwords, investor passwords, API secrets, or terminal credentials in the repository or data extracts.

## 9. Initial empirical work plan

**A. Catalogue and coverage audit**
- Enumerate MT5 symbols visible to the terminal.
- Verify exact broker symbols and contract metadata.
- Map valid spot-FX instruments to normalized base/quote edges.
- Report missing theoretical edges, reverse duplicates, disconnected currencies and ambiguity.

**B. Graph algebra**
- Build B from verified metadata, not ticker-string heuristics.
- Verify rank and connected components.
- Verify sign changes under pair inversion.
- Verify cross-rate log identities on hand-checked triples.
- Compare spanning-tree and full-graph representations.

**C. Data quality**
- Retrieve a representative sample across active and inactive sessions.
- Measure quote age, missingness, duplicate rate, spread distribution and bar gaps.
- Measure triangle residuals before and after synchronization filtering.
- Test sensitivity to UTC boundary and stale-quote threshold.
- Confirm the broker-specific meaning of the MT5 bar spread field.

**D. Descriptive factor study**
- Estimate constrained currency factors at each eligible timestamp.
- Compare full-graph least squares with a spanning-tree representation.
- Measure explained cross-sectional variation, residual covariance, residual autocorrelation and factor stability.
- Repeat across sessions and rolling windows.
- Treat results as descriptive, not predictive.

## 10. Broker-specific open decisions

Generic documentation cannot resolve:
1. Which G10 pairs are exposed and tradable?
2. Are historical MT5 bars based on bid, ask, or another price convention?
3. What are the units and semantics of the bar spread field?
4. How far back are tick and bar histories available?
5. Does tick history include both bid and ask changes at adequate frequency?
6. What quote-age tolerance is defensible per timeframe?
7. What are rollover, weekend, holiday and maintenance schedules?
8. What contract sizes, lot steps, tick values, swap modes and account-currency conversions apply?

These must be measured from the connected terminal and broker contract specifications before implementation.

## 11. References

- MetaQuotes, [copy_ticks_range — Python Integration](https://www.mql5.com/en/docs/python_metatrader5/mt5copyticksrange_py).
- MetaQuotes, [copy_rates_range — Python Integration](https://www.mql5.com/en/docs/python_metatrader5/mt5copyratesrange_py).
- BIS, [Triennial Central Bank Survey of FX and OTC derivatives markets](https://www.bis.org/publications/triennial-central-bank-survey-foreign-exchange-and-otc-derivatives-markets-2022). The BIS reported the US dollar was on one side of 88% of FX trades in April 2022. This supports including USD in the initial universe, but does not establish broker-specific liquidity.
- BIS Data Portal, [FX turnover by currency, 1992–2025](https://data.bis.org/topics/DER/tables-and-dashboards/BIS%2CDER_D11_3%2C1.0).

## Research log

| Date | Entry |
|---|---|
| 2026-11-10 | Defined ten-currency G10 target, 45 possible unordered edges, incidence rank and spanning-tree basis. Specified layered data contract, UTC synchronization and initial diagnostics. |
