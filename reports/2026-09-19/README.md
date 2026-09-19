# Current Backtest: Dynamic Signals and Hedge Allocation

This directory contains only the current dynamic strategy's research results.
The continuous January 2020–September 18, 2026 diagnostic run returned **+0.010%**
with **0.212% maximum drawdown**, 18 entries and approximately **$510 average gross
exposure** on $100,000. It failed the acceptance tests and made **no trades in 2026**.
The supported allocation remains zero.

![Performance, drawdown, signals and hedge allocation](performance.png)

![Validation, sizing and annual performance](validation_and_sizing.png)

Both performance curves use this version. Doubled costs rerun the entry filters,
so trade counts can change; the report also shows doubled costs on identical orders.
All historical periods have been observed during research. No fresh untouched test
or stable trading edge is claimed.

## Files

- [Full report (Chinese)](REPORT.md)
- [Download images and backtest data](backtest_bundle.zip)
- [Machine-readable evaluation](evaluation.json)
- [Research protocol](protocol.json)
- [Daily forecasts and hedge weights](dynamic_signals.csv)
- [Monthly model fits and label-maturity dates](monthly_model_fits.csv)
- [72-configuration search](training_search.csv)
- [Validation candidates](validation_candidates.csv)
- [Sizing analysis](sizing.csv)
- [Continuous equity](continuous_dynamic/equity.csv)
- [Continuous order events](continuous_dynamic/trades.csv)
- [Source and input hashes](provenance.json)
- [Provider metadata](data_metadata.json)
- SVG: [performance](performance.svg), [validation and sizing](validation_and_sizing.svg)

Each scenario directory contains `equity.csv`, `trades.csv` and `summary.json`.
The equal-dollar ablation changes only this version's hedge construction and retrains
its forecast with unchanged hyperparameters; it is not an earlier strategy version.

## Units and Reproduction

`equity`, `gross`, `net`, `commission` and `slippage` are USD. `qx` and `qy` are
signed EWA/EWC shares; negative means short. `prediction` and `entry_threshold`
are horizon returns per gross dollar. `weight_x` and `weight_y` divide gross
notional dollars, not broker margin or cash requirements. `signal_date` fixes
share quantities; `date` is the simulated fill date. Zero-trade files retain headers.

Use the main README commands to fetch data, install the chart dependencies and run
`python scripts/evaluate_dynamic.py`. New local runs are written to `output/dynamic/`.
Current data revisions may prevent bit-for-bit reproduction; compare provider
input hashes and package versions. No raw market-data downloads or cache databases
are distributed, and the research script submits no paper orders.
