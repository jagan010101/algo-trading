# Project Report — NIFTY 500 Multi-Strategy Algo Trading System

**Date:** 2026-08-05
**Scope:** `data_collection.ipynb`, `backtesting.ipynb`, `portfolio_optimization.ipynb`, `news.ipynb`

## 1. Objective

Build a systematic equity strategy over the NIFTY 500 universe that combines three distinct trading styles — a volatility/breakout strategy, a trend/momentum strategy, and a short-term mean-reversion strategy — each with its own stock-selection scoring and market-regime filter, and blend them into a single dynamically-allocated portfolio.

## 2. Data

- **Universe**: NIFTY 500 constituents as of 18 March 2026 (`nifty_500_list.csv`, 500 symbols), with `-`/`_` symbol mismatches between the NSE list and TradingView reconciled.
- **Prices**: ~5,040 daily bars per stock (≈20 years) downloaded via `tvDatafeed` from the NSE exchange, stored per-symbol as Parquet and merged into a single long-format master file (`symbol`, `date`, OHLCV).
- **Benchmark/regime series**: NIFTY 50 index daily OHLCV, downloaded the same way, used both as a benchmark and as the input to the market-regime filters.
- **Analysis window**: data is truncated to start 2011-01-01 (one year of lookback buffer before the training period begins), with:
  - Train: 2012-01-01 → 2020-01-01
  - Test: 2020-01-01 → 2025-11-26

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
| Gap | Overnight gap magnitude |
| Breadth | % of universe trading above its 50-day MA, computed daily across the filtered universe |

A second, vectorized version of most of these (`compute_features`, using Polars lazy expressions) also exists in the notebook; it is not currently called by the strategy pipeline and appears to be a parallel/experimental implementation.

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

For each rebalance window, per selected stock: a `backtesting.py` `Backtest` is run over the lookback + trade window with `exclusive_orders=True`, capital split evenly across the stocks selected in that window. The window's portfolio return is the equal-weighted average of the per-stock returns, compounded into a running cash balance. If no stocks survive the universe filter, or the regime filter blocks all directions, or all per-stock backtests fail, the window carries cash forward unchanged (visible in the run logs as frequent "Carrying cash forward" messages, especially through 2020–2025 where the High and Medium regime filters are frequently unsatisfied).

## 7. Results (Training Set, 2012–2020)

These are the only results that executed successfully end-to-end in the current notebook:

| Strategy | CAGR | Max Drawdown |
|---|---|---|
| Master (blended) | 26.94% | -3.8% |
| Low | 1.78% | -7.7% |
| Medium | 76.06% | -1.2% |
| High | 36.7% | -9.9% |

Initial capital: ₹3,00,00,000 for the Master run (₹1,00,00,000 per leg, matching the individual leg runs).

**Read with caution:** these are in-sample (training-period) numbers only — no out-of-sample validation currently completes (see §8). The Medium strategy's training CAGR (76%) alongside a very shallow drawdown (-1.2%) is a strong outlier relative to the other two legs and warrants scrutiny (position sizing, look-ahead risk in the 240-day-lookback composite score, and the 45-day window length are the first places to check) before being trusted.

## 8. Known Issues / Limitations

These are concrete defects observed while reading the notebook, not speculative:

1. **The strategy classes referenced at runtime are not defined in this notebook.** `backtest_low_strategy`, `backtest_medium_strategy`, and `backtest_high_strategy` all instantiate `Backtest(df_bt_run, Low_Strategy, ...)` / `Medium_Strategy` / `High_Strategy` and call `bt.run(direction=direction)`, but none of `Backtest` (the `backtesting.py` class), `Low_Strategy`, `Medium_Strategy`, or `High_Strategy` are imported or defined anywhere in `backtesting.ipynb`. Likewise `plt` (matplotlib) and `traceback` are used without being imported. The notebook only runs because these names exist in the live kernel's namespace from a prior/external session — the file itself is not self-contained. (`get_low_signals()`, `get_medium_signals()`, `get_high_signals()` look like earlier, non-class-based drafts of the same logic and are not called by the pipeline; `get_low_signals` also contains an invalid expression, `df['ATR' - 1]`, and `get_medium_signals` references an undefined `kc`.)
2. **Test-set evaluation is broken.** The cell converting the Low strategy's test-period equity curve does `test_low_results = pd.DataFrame(test_low_equity, ...)`, but the preceding cell assigned the raw result to `test_low_results` (not `test_low_equity`) — a variable-name mismatch that raises `NameError` and prevents the Low strategy's test results, and the final comparison table (`yearly_metrics_df`), from being produced.
3. **Copy/paste leftovers in the "Testing" section.** The CAGR/max-drawdown print cells under Master, Low, Medium, and High in the *Testing* section still read from `train_low_results` in their source text rather than the corresponding test variable, so any numbers they print are not reliable test-period figures.
4. **`Gap` indicator has an operator-precedence bug**: `open - close.shift(1).abs() / close.shift(1)` evaluates as `open - (|close.shift(1)| / close.shift(1))` rather than the presumably intended `(open - close.shift(1)).abs() / close.shift(1)`.
5. **`stock_selection.ipynb` is empty** — planned but not started.
6. **No dependency manifest** (`requirements.txt` / `pyproject.toml`) is checked in; see `README.md` for the inferred package list.
7. **Regime filters are frequently binding**, especially for the High strategy (trend + volatility + breadth must all agree) — the test-period run logs show long stretches of "no trade" windows through 2020–2025, which will suppress realized returns/activity regardless of the bugs above.

## 9. Recommended Next Steps

1. Add the missing `Strategy` subclass definitions (or import them from wherever they currently live) so `backtesting.ipynb` is runnable standalone, top to bottom, in a fresh kernel.
2. Fix the `test_low_equity`/`test_low_results` naming bug and the four train/test variable mix-ups in the Testing section, then re-run to get real out-of-sample numbers.
3. Investigate the Medium strategy's outsized training CAGR for look-ahead bias or sizing errors before trusting it.
4. Fix the `Gap` formula precedence bug, and decide whether to keep or remove the vestigial `get_*_signals()` functions and the unused Polars `compute_features()` path.
5. Pin dependencies in a `requirements.txt` (`pandas`, `polars`, `numpy`, `pandas_ta`, `backtesting`, `matplotlib`, `tvDatafeed`, plus `yfinance`/`seaborn` for `portfolio_optimization.ipynb` and `feedparser`/`transformers` for `news.ipynb`).
6. Decide the role of `news.ipynb` (FinBERT headline sentiment) and `portfolio_optimization.ipynb` (variance-covariance analysis) relative to the core pipeline — currently both are disconnected prototypes.
