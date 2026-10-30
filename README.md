# CSI500 Stock Selection

A Python pipeline for predicting five-trading-day stock returns with XGBoost
and constructing a long-only portfolio from the highest-ranked predictions.

## Setup

Python 3.10 or later is recommended.

```bash
pip install -r requirements.txt
```

The repository includes a historical snapshot from January 2, 2025 through
April 21, 2026 under `data/`, so the model can run without downloading new data.
To fetch data from AkShare instead:

```bash
# Download history through today (may take tens of minutes).
python download_data.py --start 20250101

# Append newly available dates to the local data.
python download_data.py --update
```

## Train and predict

```bash
python baseline_xgboost.py
python baseline_xgboost.py --top-k 20 --out outputs/portfolio.csv
python baseline_xgboost.py --as-of 20260420 --out outputs/historical_portfolio.csv
```

Each run builds features, trains a new XGBoost regressor with early stopping,
prints validation rank IC, and predicts returns for the selected date. Training
and validation are split chronologically with a five-trading-day embargo.
The default validation window is ten trading days. `--as-of` must be a trading
date present in the data; by default, the latest available date is used.

The model selects up to `--top-k` stocks (default: 50). Portfolio weights are
proportional to descending prediction ranks and normalized to sum to one.
There is no fixed minimum number of holdings or per-stock weight cap.
The output CSV has two columns: `stock_code` and `weight`.

## Evaluate a portfolio

```bash
# Generate weights before the period being evaluated.
python baseline_xgboost.py --as-of 20260410 --out outputs/historical_portfolio.csv
python evaluate_portfolio.py outputs/historical_portfolio.csv --start 20260413 --end 20260417
```

The evaluator reports the fixed-weight portfolio return, CSI500 index return,
and excess return over the chosen date range. Entry uses the last close before
the start date, falling back to the first available open when necessary. Exit
uses the last available close within the range. Missing prices produce notes;
a stock with no observations in the range contributes zero return.

Use non-negative weights summing to one, with one row per stock. Price and
index data must cover the evaluation period. This is a single-period return
calculation, without transaction costs or execution modeling.

## Files

| File | Purpose |
| --- | --- |
| `baseline_xgboost.py` | Model training, validation, prediction, and portfolio construction |
| `features.py` | Technical indicators, cross-sectional ranks, and forward-return target |
| `download_data.py` | Download and update constituent prices and benchmark data |
| `evaluate_portfolio.py` | Compare a portfolio with the CSI500 index |
| `requirements.txt` | Python dependencies |
| `data/constituents.csv` | Constituent codes, names, and snapshot date |
| `data/prices.parquet` | Daily stock prices and trading activity |
| `data/index.parquet` | Daily CSI500 index prices |

## Features and data

The 14 model inputs cover 1/5/10/20/60-day returns, 20-day volatility,
volume z-score, average turnover, price relative to moving averages, RSI,
and daily cross-sectional ranks. The target is the five-trading-day forward
return on the adjusted close. Rows lacking complete features or targets are
excluded from training.

The bundled data is a historical snapshot, not a live feed. Run the downloader
to refresh it. Stock prices use forward adjustment (`qfq`); turnover is a
fraction and `pct_change` is in percent. The constituent file contains stocks
for which prices were successfully obtained, so coverage can be below 500.

Validation rank IC is a diagnostic, not a complete historical backtest. For
historical research, account for constituent changes, survivorship bias,
corporate-action adjustments, data availability, and trading costs.
