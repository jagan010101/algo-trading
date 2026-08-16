# Algo Trading — NIFTY 500 Multi-Strategy Backtesting System

A systematic, multi-strategy equity trading research project built around the NIFTY 500 universe. It downloads daily price history, engineers a set of technical indicators, scores and selects stocks under three distinct strategy styles (Low / Medium / High), filters trades against a market-regime signal, and backtests a capital-weighted "Master" allocation across all three.

> **Status:** `backtesting.ipynb` runs end-to-end, top to bottom, with no errors. Train (2012–2020), Test (2020–2025) and Production (2025–present) results all compute successfully. See [Known Issues](#known-issues) and `REPORT.md` for remaining caveats and result quality notes.

## Repository Structure

| File | Purpose |
|---|---|
| `data_collection.ipynb` | Downloads ~20 years of daily OHLCV data for all NIFTY 500 constituents and the NIFTY 50 index from TradingView (via `tvDatafeed`), and merges per-stock files into master Parquet files. |
| `backtesting.ipynb` | The core notebook: technical indicators, the three strategies (Low/Medium/High), universe filtering, market-regime filters, the Master allocator, and Train/Test/Production backtests with performance metrics. |
| `nifty_500_list.csv` | NIFTY 500 index constituents (company name, industry, symbol, ISIN), downloaded from the NSE website on 18 March 2026. |
| `requirements.txt` | Pinned Python dependencies for both notebooks. |
| `data/` | Local price data (gitignored, not committed — see [Data](#data)). |

## Data

Price data is sourced from TradingView (`tvDatafeed`) via `data_collection.ipynb`. Expected local layout:

```
data/
├── master_files/
│   ├── nifty_500_stocks.parquet   # all 500 stocks, long format (symbol, date, OHLCV)
│   └── nifty_50_index.parquet     # NIFTY 50 index OHLCV, used for regime filtering
└── nifty_500_stocks/
    └── <SYMBOL>.parquet           # one file per stock
```

`data/` is gitignored — this layout is created locally by running `data_collection.ipynb`. A pre-downloaded snapshot may also be shared out-of-band (e.g. a zip of the `data/` folder) to skip re-running the TradingView download.

## Strategy Overview

Three independently-scored sub-strategies, each rebalanced on its own window, plus a capital allocator on top:

- **Low** — volatility-breakout style. Scores stocks on Keltner Channel positioning, trend (EMA 50 vs 200), range expansion, and inverse volatility. 15-day rebalance window.
- **Medium** — trend/momentum style. Scores stocks on moving-average stacking, MACD/ROC/ADX momentum, breakout frequency, and noise, using z-scored composite factors. 45-day rebalance window.
- **High** — short-term mean-reversion style. Scores stocks on VWAP deviation, RSI turning points, and volume pickup over a 10-day lookback. 15-day rebalance window.

All three first pass a **liquidity/volatility universe filter** (turnover, price floor, volume floor, volatility ceiling), then a **market-regime filter** (built from NIFTY 50 trend/momentum/volatility/breadth) that only allows trades in the direction the regime supports.

A **Master strategy** allocates capital across Low/Medium/High every 45 days, weighted by each sub-strategy's trailing-period return (with a 5% minimum floor per leg), then re-normalized.

See `REPORT.md` for the full methodology, indicator definitions, and current results.

## Setup

```
pip install -r requirements.txt
```

`tvDatafeed` (needed only for `data_collection.ipynb`) isn't on PyPI under a stable name — install it separately:

```
pip install --upgrade git+https://github.com/rongardF/tvdatafeed.git
```

## Running

1. **Collect data** — run `data_collection.ipynb` top to bottom. Downloads are skipped automatically if the target Parquet files already exist.
2. **Run the backtest** — run `backtesting.ipynb` top to bottom. It loads the merged Parquet files, computes indicators, and runs the Low/Medium/High/Master strategies over the Train (2012–2020), Test (2020–2025), and Production (2025–present) windows.

## Known Issues

- **`Gap` indicator has an unused operator-precedence bug**: `open - close.shift(1).abs() / close.shift(1)` evaluates as `open - (|close.shift(1)| / close.shift(1))` rather than the presumably intended `(open - close.shift(1)).abs() / close.shift(1)`. The `Gap` column is computed but not currently read by any strategy's scoring or filter logic, so this doesn't affect current results — but it should be fixed before `Gap` is used for anything.
- **Regime filters are frequently binding**, especially for the High strategy (trend + volatility + breadth must all agree). This suppresses trading activity in some windows, most visibly in the short Production window (Nov 2025–present) where the High and Medium legs show no realized trades.
- No automated tests — correctness relies on manual inspection of notebook output.

Full detail, the current results, and severity notes are in `REPORT.md`.
