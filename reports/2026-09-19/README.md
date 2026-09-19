# EWA/EWC Research Snapshot — 2026-09-19

This snapshot records a cost-aware strategy search and its historical evaluation.
The 2026 candidate earned approximately 0.59% after modeled costs, but failed the
earlier validation/audit requirements. **The selected allocation remains zero.**
These are simulated historical trades, not broker fills or forward paper-trading results.

## Contents

- [Full report (Chinese)](REPORT.md): results, assumptions, sizing and limitations.
- [Evaluation JSON](evaluation.json): all metrics and the allocation decision.
- [Protocol](protocol.json): dates, parameter count, cost assumptions and acceptance criteria.
- [Frozen selection](frozen_selection.json): parameters chosen before downloading 2026 data.
- [Training search](training_search.csv): every one of the 384 configurations.
- [Validation candidates](validation_candidates.csv): the 12 training finalists.
- [Sizing](sizing.csv): all eight gross-allocation levels, including cash.
- Each stage directory contains `equity.csv`, `trades.csv` and `summary.json`.
- `fresh_holdout_base/` and `fresh_holdout_stress/` contain the six 2026 round trips.
- `comparison_*` contains the old-threshold/new-candidate comparison under identical accounting.
- `original_baseline_*` preserves the first, simpler adjusted-price backtest.
- [Provenance](provenance.json): runtime versions and SHA256 hashes of source, results and raw inputs.
- [Data metadata](data_metadata.json): provider, retrieval time and downloaded coverage.

The loader metadata describes the downloader's adjusted-price CSV. The upgraded
research engine instead reads raw `Close`, `Dividends`, `Capital Gains` and
`Stock Splits` fields from the two source files. It builds a forward total-return
index for signals and separately books dividends against actual-price holdings.

## Reproduction

Run from the repository root:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e '.[data,dev]'
pairs-trader download --symbols EWA EWC --start 2015-01-01 --end 2026-01-01 --output output/real_api/market_data
python scripts/evaluate_real_data.py
python scripts/optimize_pairs.py select
pairs-trader download --symbols EWA EWC --start 2015-01-01 --end 2026-09-19 --output output/optimization/market_data
python scripts/optimize_pairs.py evaluate
ruff check src tests scripts
pytest
```

The scripts write new runs to `output/`, leaving this dated snapshot unchanged.
Yahoo may revise historical data, so reruns need not be bit-identical. Compare input
hashes and package versions before comparing results. Raw market-data files and
download caches are not distributed in this snapshot; fetch them from the provider.

For the original run, parameter selection preceded the 2026 download. The 2020–2025
baseline had already been inspected, and is not represented as untouched data.
Repeating the commands now does not create a new unseen test. Each period starts
with $100,000, no open position, and ends with costed liquidation. The 20% candidate
allocation shown in the report is diagnostic; it does not override the zero-allocation decision.

The candidate has not been connected to the Alpaca paper-order command. Actual
borrow availability/rates, asynchronous fills, market impact and market factor
neutrality still require validation. The 5% drawdown limit is a historical selection
constraint, not a promise about future losses.
