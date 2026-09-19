# Kalman Pairs Trading (Paper-First)

A GitHub-ready research and paper-trading project for U.S. equity pairs. The strategy is based on Jia Yu (2023): it screens pairs for cointegration, updates the hedge ratio online with a one-dimensional Kalman filter, and generates trading signals from a rolling, look-ahead-safe z-score.

> For research purposes only. This project is not financial advice. It sends no orders by default. The Alpaca integration is restricted to `paper=True`; observe paper performance, execution quality, and short-borrow constraints for an extended period before considering any further use.

## Real-Data Research Update — September 19, 2026

The updated EWA/EWC candidate made **+0.59% after modeled costs in January–September 18, 2026**
(**+0.45% with doubled costs**, six round trips). This is a positive historical test period,
**not evidence of a consistently profitable strategy**: validation in 2020–2022 returned
−0.67%, and the previously observed 2023–2025 audit returned −0.91%.
The predefined allocation rule therefore selected **$0**, not a deployment allocation.

All returns below are cumulative account returns on a $100,000 account with 20% gross
pair exposure. Each evaluation period starts flat and ends with costed liquidation.

| Period | Role | Base costs | Doubled costs | Entries |
|---|---|---:|---:|---:|
| 2016–2019 | Training | +2.00% | +1.01% | 35 |
| 2020–2022 | Validation | −0.67% | −1.63% | 36 |
| 2023–2025 | Previously observed audit | −0.91% | −1.75% | 33 |
| 2026 through September 18 | Newly reserved historical test | +0.59% | +0.45% | 6 |

The search covered 384 predefined Kalman/rolling-OLS configurations and eight allocation
levels. Using the same upgraded ledger and costs, the original thresholds lost 4.30%
over 2020–2025; the selected diagnostic candidate lost 1.57%. This comparison is not
a reproduction of the older adjusted-price baseline, because accounting and sizing changed.

- [Full research report / 完整回测报告](reports/2026-09-19/REPORT.md)
- [Frozen candidate and allocation decision](reports/2026-09-19/frozen_selection.json)
- [Protocol and selection criteria](reports/2026-09-19/protocol.json)
- [Machine-readable evaluation](reports/2026-09-19/evaluation.json)
- [All training candidates](reports/2026-09-19/training_search.csv)
- [2026 base-cost trades](reports/2026-09-19/fresh_holdout_base/trades.csv)
- [Reproduction and artifact provenance](reports/2026-09-19/README.md)

The research engine uses actual closing prices, separate dividend cash flows, lagged
share hedge ratios, whole shares, minimum commissions, slippage and calendar-day borrow
charges. It rejects datasets containing stock splits. The cost assumptions are $0.005/share
with a $1 minimum per leg/order, 2 bp slippage, a 0.3 bp sell-side fee reserve and 3% annual
short borrow. These are research assumptions, not an exact broker or historical fee quote.
Borrow availability, asynchronous fills, market impact and market-beta neutrality remain
unverified. The candidate is **not wired into the paper-order command**.

### Reproduce the Real-Data Experiments

```bash
pip install -e '.[data,dev]'
pairs-trader download --symbols EWA EWC --start 2015-01-01 --end 2026-01-01 --output output/real_api/market_data
python scripts/evaluate_real_data.py
python scripts/optimize_pairs.py select
pairs-trader download --symbols EWA EWC --start 2015-01-01 --end 2026-09-19 --output output/optimization/market_data
python scripts/optimize_pairs.py evaluate
```

The `select` step freezes parameters before the fresh historical test is downloaded.
2020–2025 had already been inspected for the original baseline and is not claimed to be
untouched. Yahoo Finance is accessed through the unofficial `yfinance` research interface;
provider revisions can change downloaded history. Raw market-data downloads and cache
databases stay local; this repository publishes derived results and provenance hashes.
`pairs-trader backtest --trade-start YYYY-MM-DD` can separately warm up on preceding data
without including that warm-up period in performance statistics.

## Paper Alignment and Engineering Improvements

- Core model: `spread = price_y - beta_t * price_x`, with `beta_t` updated by the Kalman filter as new prices arrive.
- Signals: short the spread when the z-score crosses above `+entry_z`, go long when it crosses below `-entry_z`, and exit when it reverts to `exit_z`.
- Pair selection: includes a Johansen trace test and requires both price series to pass an I(1) check first.
- Look-ahead protection: a signal generated on one bar is executed on the next; rolling means and standard deviations use only information available through the previous bar.
- Trading realism: models fees, slippage, maximum holding periods, stop-loss z-scores, per-pair capital limits, data freshness, whole-share quantities, and rollback behavior when paired orders fail.
- Risks outside the paper's model: live borrow availability, dividends, corporate actions, partial fills, and market impact still require broker-side monitoring.

## Run in 30 Seconds (No Account or Network Required)

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e .
python -m pairs_trading.cli demo --output output/demo
```

This creates `equity.csv`, `trades.csv`, `signals.csv`, and `summary.json`. You can also provide your own adjusted daily-price CSV:

```bash
pairs-trader backtest --csv data/prices.csv --x EWA --y EWC --output output/ewa_ewc
```

The CSV must have the columns `date,EWA,EWC`, with dates in ascending order and adjusted closing prices in each symbol column.

Run the paper's Johansen pair-selection procedure on a universe CSV:

```bash
pairs-trader select --csv data/universe.csv
```

## Configure Alpaca Paper Trading

```bash
pip install -e '.[alpaca]'
cp .env.example .env
# Keep credentials in your local environment and never commit .env
export APCA_API_KEY_ID='...'
export APCA_API_SECRET_KEY='...'
export ALLOW_PAPER_ORDERS=true
pairs-trader paper --x EWA --y EWC --lookback-days 400
```

The `paper` command performs one decision cycle, making it suitable for GitHub Actions, cron, or another scheduler after the U.S. market closes. By default, the first run records signals only; simulated orders are submitted only when `ALLOW_PAPER_ORDERS=true` is explicitly enabled. Before opening a position, the command checks both existing legs to prevent duplicate entries. If it detects an orphaned leg, it stops and requests manual intervention. The integration follows the official SDK pattern with `TradingClient(..., paper=True)` and obtains market data separately through `StockHistoricalDataClient`.

## Backtesting and Pre-Deployment Checklist

```bash
python -m unittest discover -s tests -v
```

1. Perform walk-forward validation on periods that were not used for parameter tuning; do not rely solely on the paper's sample.
2. Confirm that both series remain I(1) and cointegrated under current market conditions. Stop opening positions when the relationship breaks down.
3. Verify that the Alpaca paper account permits shorting both symbols, and monitor whether both legs of each paired order fill.
4. Keep `ALLOW_PAPER_ORDERS=false` while reviewing logs, then enable simulated orders deliberately.
5. This repository does not provide a live-trading switch. Any live deployment requires a separate code and risk review.

## Project Structure

```text
src/pairs_trading/  Strategy, backtesting, data, and broker adapter
tests/              Network-free unit tests
config/default.json Default risk and signal parameters
.github/workflows/  Continuous integration
```

## Reference

Yu, J. (2023). "Cointegration Approach for the Pair Trading Based on the Kalman Filter." In V. Escudero et al. (Eds.), *Proceedings of the 2022 2nd International Conference on Business Administration and Data Science (BADS 2022)*, Atlantis Highlights in Computer Sciences, Vol. 11, pp. 633-642. Atlantis Press. [https://doi.org/10.2991/978-94-6463-102-9_66](https://doi.org/10.2991/978-94-6463-102-9_66)

The paper is available under the [Creative Commons Attribution-NonCommercial 4.0 International License](https://creativecommons.org/licenses/by-nc/4.0/). This project is an independent software implementation and is not affiliated with the paper's author or publisher. It is not a reproduction of the paper's source code; where the paper is ambiguous, the project uses conservative, testable engineering definitions.
