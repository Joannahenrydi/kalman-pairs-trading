# Kalman Pairs Trading (Paper-First)

A GitHub-ready research and paper-trading project for U.S. equity pairs. The strategy is based on Jia Yu (2023): it screens pairs for cointegration, updates the hedge ratio online with a one-dimensional Kalman filter, and generates trading signals from a rolling, look-ahead-safe z-score.

> For research purposes only. This project is not financial advice. It sends no orders by default. The Alpaca integration is restricted to `paper=True`; observe paper performance, execution quality, and short-borrow constraints for an extended period before considering any further use.

## Latest: Dynamic Signals and Hedge Allocation

The dynamic research version updates a linear forecast monthly, evaluates cost-aware entry
thresholds daily, and computes a constrained minimum-variance dollar hedge. It fixes share
quantities at the signal close for next-close execution and charges for rebalancing.

**Continuous 2020–September 18, 2026 result, after modeled costs:**

| Version | Account return | Maximum drawdown | Entries | Mean gross exposure |
|---|---:|---:|---:|---:|
| Dynamic candidate, 20% entry cap | +0.010% | 0.212% | 18 | $510 |

The approximately $9.74 profit on $100,000 is economically negligible. The small drawdown
reflects infrequent trading and low actual exposure. The dynamic candidate has
**zero 2026 entries**, failed the training/validation acceptance criteria, and retains a
**zero supported allocation**. All periods have been observed during research; none is
claimed to be a new untouched test for this version.

As of September 18, the conditional risk-minimizing gross-dollar split is **41.48% EWA /
58.52% EWC**, or approximately **1.4844 EWA shares per EWC share**, with opposite signs.
This is a notional allocation, not a margin requirement. The current signal is below the
cost threshold, so the diagnostic desired position is zero.

![Dynamic performance](reports/2026-09-19/performance.png)

- [Dynamic report and methodology](reports/2026-09-19/REPORT.md)
- [Performance images and data index](reports/2026-09-19/README.md)
- [Download backtest data and images](reports/2026-09-19/backtest_bundle.zip)
- [Daily dynamic signals and hedge ratios](reports/2026-09-19/dynamic_signals.csv)

```bash
pip install -e '.[data,charts,dev]'
pairs-trader download --symbols EWA EWC --start 2015-01-01 --end 2026-09-19 --output output/optimization/market_data
python scripts/evaluate_dynamic.py
```

This writes figures, signals, fits, trades and a ZIP bundle to `output/dynamic/`.
The 72-configuration search is exploratory: 48 general ridge candidates were followed by
24 symmetric Kalman-z reversion candidates. The report documents this iterative research process.
The updated model remains separate from paper-order submission.

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
