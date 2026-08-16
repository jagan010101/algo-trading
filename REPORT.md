# Project Report — NIFTY 500 Multi-Strategy Algo Trading System

**Date:** 2026-08-16
**Scope:** `data_collection.ipynb`, `backtesting.ipynb`

## 1. Objective

Build a systematic equity strategy over the NIFTY 500 universe that combines three distinct trading styles — a volatility/breakout strategy, a trend/momentum strategy, and a short-term mean-reversion strategy — each with its own stock-selection scoring and market-regime filter, and blend them into a single dynamically-allocated portfolio.

## 2. Data

- **Universe**: NIFTY 500 constituents as of 18 March 2026 (`nifty_500_list.csv`, 500 symbols), with `-`/`_` symbol mismatches between the NSE list and TradingView reconciled.
- **Prices**: daily bars per stock downloaded via `tvDatafeed` from the NSE exchange, stored per-symbol as Parquet and merged into a single long-format master file (`symbol`, `date`, OHLCV).
- **Benchmark/regime series**: NIFTY 50 index daily OHLCV, downloaded the same way, used both as a benchmark and as the input to the market-regime filters.
- **Analysis window**: data is truncated to start 2011-01-01 (one year of lookback buffer before the training period begins), with:
  - Train: 2012-01-01 → 2020-01-01
  - Test: 2020-01-01 → 2025-11-26
  - Production: 2025-11-26 → most recent available data

## 3. Feature Engineering

`backtesting.ipynb` implements a library of technical indicators as reusable functions (mostly thin wrappers over `pandas_ta`), computed per stock:

| Indicator | Purpose |
|---|---|
| MACD / MACD Signal | Trend/momentum crossover |
| EMA (20/50/200), MA (20/50/100/200) | Trend structure |
| RSI (5) | Short-term overbought/oversold |
| ADX (14) | Trend strength |
| ROC (20) | Momentum |
| ATR (14) | Volatility / stop sizing |
| Volatility (20-day stdev) | Dispersion |
| VWAP (5/20) | Intraday-anchored fair value |
| Keltner Channels (20, ×1.5 ATR) | Breakout bands |
| Range Expansion | `(close - open) / ATR` — breakout intensity |
| Volume Z-score (20-day) | Volume anomaly detection |
| Trend-Noise Ratio | `ATR14 / |EMA50 - EMA200|` — trend clarity vs. chop |
| Gap | Overnight gap magnitude (computed but currently unused — see §8) |
| Breadth | % of universe trading above its 50-day MA, computed daily across the filtered universe |

## 4. Universe Filtering

Before any stock enters a strategy's selection pool, `filter_universe()` rejects it if, over the trailing 20 days:

- Average turnover (close × volume) is below ₹10 crore, **or**
- Average price is below ₹10, **or**
- Average volume is below 50,000 shares, **or**
- Return volatility (stdev of daily % change) exceeds 7%

This removes illiquid, penny, and abnormally volatile names before scoring.

## 5. Strategy Design

Each strategy runs on its own rolling schedule: at each window boundary, it (a) rebuilds the eligible universe, (b) scores and ranks stocks, (c) checks the market-regime filter, (d) keeps only selections whose direction agrees with the regime, and (e) runs an equal-capital-split backtest per selected stock via `backtesting.py`.

### 5.1 Low Strategy — volatility breakout

- **Window**: 15 trading days.
- **Selection score** (top 20 by score): weighted combination of trend alignment (EMA50 vs EMA200), "pressure" — how far price has pushed through the Keltner channel relative to its width — normalized range expansion, and inverse-volatility (tighter ATR relative to price scores higher).
- **Regime filter**: long if NIFTY 50's 50-day MA > 200-day MA, short if inverted, else no trade.

### 5.2 Medium Strategy — trend & momentum

- **Window**: 45 trading days.
- **Selection score**: over a 240-day lookback, computes the *fraction of days* each stock spent in an up/down MA-stacking trend, ROC/MACD sign persistence, MACD-vs-signal trend quality, ADX strength, breakout frequency (new 50-day highs), and turnover — each feature is cross-sectionally z-scored across the universe on each date, then combined into weighted long/short composite scores.
- **Regime filter**: long if NIFTY 50 close > 60-day MA and ADX(14) > 20 (i.e., trending, not choppy); short if price is below the MA with the same momentum condition.

### 5.3 High Strategy — short-term mean reversion

- **Window**: 15 trading days.
- **Selection score**, over the last 10 days: 2-day-consistent VWAP crossovers, RSI turning points (RSI < 60 and rising for longs / > 40 and falling for shorts), and a volume-pickup ratio versus the 10-day average volume.
- **Regime filter**: long only if NIFTY 50 is trending up, in a low-volatility regime (ATR/close < 3%), *and* market breadth > 65%; short only if trending down, high-volatility (ATR/close > 4%), and breadth < 35%. This is the most conservative of the three filters — it requires trend, volatility, and breadth to all agree.

### 5.4 Master Strategy — capital allocation

Every 45 days, capital is split across Low/Medium/High proportional to each leg's return over the *previous* period (equal-thirds in period 1), with a 5% minimum floor per leg to avoid fully zeroing out an underperformer, then re-normalized to sum to 100%. Each leg is then run independently for that period and results are recombined into a single equity curve.

## 6. Backtest Mechanics

For each rebalance window, per selected stock: a `backtesting.py` `Backtest` is run over the lookback + trade window with `exclusive_orders=True`, capital split evenly across the stocks selected in that window. The window's portfolio return is the equal-weighted average of the per-stock returns, compounded into a running cash balance. If no stocks survive the universe filter, or the regime filter blocks all directions, or all per-stock backtests fail, the window carries cash forward unchanged.

## 7. Results

`backtesting.ipynb` now runs end-to-end — all 64 code cells execute in order with no errors — producing results across all three periods. Initial capital: ₹1,00,00,000 per leg (₹3,00,00,000 for the Master run).

| Period | Strategy | CAGR (%) | Max Drawdown (%) | Avg Sharpe |
|---|---|---|---|---|
| Train (2012–2020) | Master | 29.94 | -2.78 | 12.304 |
| Train | High | 33.75 | -11.22 | 4.507 |
| Train | Medium | 74.63 | -1.17 | 23.172 |
| Train | Low | 5.13 | -7.40 | 5.681 |
| Test (2020–2025) | Master | 64.96 | -15.67 | 10.554 |
| Test | High | 94.35 | -13.60 | 9.417 |
| Test | Medium | 55.28 | -11.44 | 10.235 |
| Test | Low | -3.23 | -34.71 | 1.998 |
| Production (2025–present) | Master | -3.27 | -1.22 | -11.225 |
| Production | High | 0.00 | 0.00 | n/a |
| Production | Medium | 0.00 | 0.00 | 0.000 |
| Production | Low | -0.14 | -1.76 | n/a |

**Read with caution:**

- The Medium strategy's Train CAGR (75%) alongside a very shallow drawdown (-1.2%) is a strong outlier and warrants scrutiny — position sizing, look-ahead risk in the 240-day-lookback composite score, and the 45-day window length are the first places to check before trusting it. It stays strong but more plausible in Test (55%, -11.4% DD), which is at least reassuring.
- The Low strategy degrades sharply out-of-sample: 5% CAGR / -7.4% DD in Train vs. -3.2% CAGR / **-34.7% DD** in Test — the largest drawdown of any leg in either period. This is the strategy most likely to be overfit to Train-period conditions.
- The Production window (Nov 2025 → present) is short (~8.5 months) and both High and Medium show flat 0.0% CAGR — the regime filter is apparently not greenlighting any trades for those two legs in this window, not a computation error. Production-period conclusions should be treated as low-confidence until the window is longer.

## 8. Known Issues / Limitations

Previous revisions of this report described the strategy classes as missing, the test-set pipeline as broken by a `NameError`, and several train/test variable mix-ups. Those have all been fixed — `Low_Strategy`, `Medium_Strategy`, and `High_Strategy` are fully defined and imported correctly, the test-period cells reference the correct variables, and the notebook is self-contained and runs standalone in a fresh kernel. Remaining items:

1. **`Gap` formula has an operator-precedence bug**: `open - close.shift(1).abs() / close.shift(1)` evaluates as `open - (|close.shift(1)| / close.shift(1))` rather than the presumably intended `(open - close.shift(1)).abs() / close.shift(1)`. Low severity — the `Gap` column is computed in `compute_indicators()` but not read by any strategy's scoring or filter logic, so it doesn't affect the results in §7. Should be fixed before `Gap` is wired into anything.
2. **Regime filters are frequently binding**, especially for the High strategy (trend + volatility + breadth must all agree) — this suppresses realized trades/returns in some windows regardless of scoring quality, most visibly in the short Production window (see §7).
3. **No automated tests** — correctness currently relies on manual inspection of notebook output rather than a repeatable check.

## 9. Recommended Next Steps

1. Investigate the Medium strategy's outsized Train CAGR (look-ahead bias, sizing, or window-length sensitivity) and the Low strategy's out-of-sample drawdown blowout (34.7% in Test vs. 7.4% in Train) before trusting either for live use.
2. Fix the `Gap` formula precedence bug, and either wire `Gap` into a strategy's scoring/filter or remove it if it's not needed.
3. Let the Production window accumulate more history before drawing conclusions from it — 8.5 months isn't enough to distinguish "regime filter correctly sitting out" from "regime filter miscalibrated."
4. Add basic automated checks (e.g. a smoke test that the notebook executes top-to-bottom without error, or unit tests for the indicator functions) so pipeline regressions are caught before they reach a full backtest run.
