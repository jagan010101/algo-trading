# Algo Trading — NIFTY 500 Multi-Strategy Backtesting System

A systematic, multi-strategy equity trading research project built around the NIFTY 500 universe. It downloads daily price history, engineers a set of technical indicators, scores and selects stocks under three distinct strategy styles (Low / Medium / High), filters trades against a market-regime signal, and backtests a capital-weighted "Master" allocation across all three.

> **Status:** work in progress. The most recent commit is titled *"Needs fine tuning"* — the training-set backtest runs end-to-end, but the test-set evaluation pipeline currently errors out (see [Known Issues](#known-issues) and `REPORT.md`).

## Repository Structure

| File | Purpose |
|---|---|
| `data_collection.ipynb` | Downloads ~20 years of daily OHLCV data for all NIFTY 500 constituents and the NIFTY 50 index from TradingView (via `tvDatafeed`), and merges per-stock files into master Parquet files. |
| `backtesting.ipynb` | The core notebook: technical indicators, the three strategies (Low/Medium/High), universe filtering, market-regime filters, the Master allocator, and train/test backtests with performance metrics. |
| `portfolio_optimization.ipynb` | A small, standalone exploration of a fixed two-stock portfolio (Reliance, HDFC Bank) — pulls prices via `yfinance` and computes a variance-covariance matrix. |
| `news.ipynb` | Standalone prototype for pulling financial news (Moneycontrol, ET Markets, NSE, Google News RSS) and scoring headline sentiment with FinBERT. Not wired into the backtest pipeline. |
| `stock_selection.ipynb` | Empty placeholder — not yet started. |
| `nifty_500_list.csv` | NIFTY 500 index constituents (company name, industry, symbol, ISIN), downloaded from the NSE website on 18 March 2026. |
| `data/Algo Trading Data.zip` | A packaged snapshot of the raw/master Parquet price data (untracked, large — see [Data](#data)). |

## Data

Price data is sourced from TradingView (`tvDatafeed`) for `data_collection.ipynb`, and separately from Yahoo Finance (`yfinance`) for the standalone `portfolio_optimization.ipynb`. Expected local layout (created by `data_collection.ipynb`, not committed to git):

```
data/
├── master_files/
│   ├── nifty_500_stocks.parquet   # all 500 stocks, long format (symbol, date, OHLCV)
│   └── nifty_50_index.parquet     # NIFTY 50 index OHLCV, used for regime filtering
└── nifty_500_stocks/
    └── <SYMBOL>.parquet           # one file per stock, ~5040 daily bars each
```

`data/Algo Trading Data.zip` contains a pre-downloaded snapshot of this layout — unzip it into `data/` to skip re-running the TradingView download.

## Strategy Overview

Three independently-scored sub-strategies, each rebalanced on its own window, plus a capital allocator on top:

- **Low** — volatility-breakout style. Scores stocks on Keltner Channel positioning, trend (EMA 50 vs 200), range expansion, and inverse volatility. 15-day rebalance window.
- **Medium** — trend/momentum style. Scores stocks on moving-average stacking, MACD/ROC/ADX momentum, breakout frequency, and noise, using z-scored composite factors. 45-day rebalance window.
- **High** — short-term mean-reversion style. Scores stocks on VWAP deviation, RSI turning points, and volume pickup over a 10-day lookback. 15-day rebalance window.

All three first pass a **liquidity/volatility universe filter** (turnover, price floor, volume floor, volatility ceiling), then a **market-regime filter** (built from NIFTY 50 trend/momentum/volatility/breadth) that only allows trades in the direction the regime supports.

A **Master strategy** allocates capital across Low/Medium/High every 45 days, weighted by each sub-strategy's trailing-period return (with a 5% minimum floor per leg), then re-normalized.

See `REPORT.md` for the full methodology, indicator definitions, and current results.

## Setup

No `requirements.txt` is currently checked in. Based on the imports used across the notebooks, you'll need:

```
pandas
polars
numpy
pandas_ta
backtesting          # backtesting.py — used for per-stock trade simulation
matplotlib
seaborn              # portfolio_optimization.ipynb only
tvDatafeed           # pip install --upgrade git+https://github.com/rongardF/tvdatafeed.git
yfinance             # portfolio_optimization.ipynb only
feedparser           # news.ipynb only
transformers         # news.ipynb only (FinBERT sentiment model)
```

## Running

1. **Collect data** — run `data_collection.ipynb` top to bottom (or unzip `data/Algo Trading Data.zip` into `data/` to skip this). Downloads are skipped automatically if the target Parquet files already exist.
2. **Run the backtest** — run `backtesting.ipynb` top to bottom. It loads the merged Parquet files, computes indicators, and runs the Low/Medium/High/Master strategies over the training window (2012–2020).
3. `portfolio_optimization.ipynb` and `news.ipynb` are standalone and can be run independently of the above.

## Known Issues

The pipeline is mid-refactor. Concrete gaps as of the last commit:

- **Missing definitions in `backtesting.ipynb`**: the notebook calls `Backtest(...)`, `Low_Strategy`, `Medium_Strategy`, `High_Strategy`, `plt`, and `traceback`, none of which are imported or defined in the notebook as saved. The actual per-stock entry/exit logic (the `Strategy` subclasses) is not present in this file.
- **Test-set evaluation errors out**: cell defining `test_low_results` references an undefined `test_low_equity` (`NameError`), which cascades into the final metrics comparison table.
- Several "Testing" section CAGR/drawdown cells still reference the **training** equity curve variable (`train_low_results`) instead of the corresponding test one — a copy/paste leftover.
- `get_low_signals()` contains an invalid expression (`df['ATR' - 1]`) and appears to be superseded by the (missing) `Low_Strategy` class; `get_medium_signals()` references an undefined `kc` variable. Both look like earlier drafts not currently wired into the pipeline.
- `stock_selection.ipynb` is an empty file.
- No `requirements.txt` / environment file is checked in.

Full detail and the training-set results that *did* run successfully are in `REPORT.md`.
