# Multi-Asset Momentum Strategy

A fully systematic quantitative trading strategy built on diversified time-series momentum across 8 asset classes, with volatility targeting and rigorous out-of-sample validation.

This project started as a question: can a rules-based, theory-motivated strategy generate consistent risk-adjusted returns without fitting parameters to historical data? The code, results, and research paper here are an honest attempt to answer that.

---

## What this strategy does

This strategy is rooted in a well-documented phenomenon in financial economics: **momentum** — the tendency of assets that have been trending upward to continue doing so over medium-term horizons (3–12 months), and vice versa.

Rather than concentrating risk in a single market, the strategy spreads exposure across **8 uncorrelated asset classes** simultaneously — US equities, international equities, emerging markets, short and long-term government bonds, investment-grade credit, gold, and commodities. At any given time, it sizes into assets that are in uptrends and sizes out of assets in downtrends, with position sizes continuously adjusted to keep overall portfolio volatility stable regardless of market conditions.

The central question driving the project: *does this edge persist out-of-sample, or does it disappear the moment you stop looking at the data that produced it?*

---

## How it works

### 1. Signal generation
For each of the 8 assets, momentum is measured across three lookback windows simultaneously — approximately 3 months (63 trading days), 6 months (126 days), and 12 months (252 days). The signal is the sign-average across all three lookbacks: if 2 of 3 lookbacks are positive, the asset receives a positive signal. Blending multiple lookbacks reduces sensitivity to any single window choice and produces a more stable signal with fewer false reversals.

### 2. Volatility-targeted position sizing
Momentum signals are not traded at equal weight. Each asset's position is scaled by the inverse of its recent realised volatility (63-day trailing window), so that every asset contributes roughly equal *risk* to the portfolio rather than equal *capital*. Total portfolio volatility is then scaled to a 10% annualised target — positions grow in calm markets and shrink automatically when volatility rises.

### 3. Risk controls
Several hard limits keep the sizing disciplined:
- Gross exposure is capped at 1.5× (no excessive leverage)
- A drawdown control mechanism reduces exposure progressively as losses deepen, resuming normal sizing as the portfolio recovers
- All signals are executed with a one-day lag (signal computed at close of day *t*, traded at open of day *t+1*), eliminating any lookahead bias
- Transaction costs (commission + slippage) are charged on every rebalance

### 4. Parameters are fixed, not optimised
Every parameter — lookback windows, volatility target, drawdown thresholds — was set based on economic reasoning before running any backtest, and never adjusted afterward. The goal was to test whether a theoretically grounded strategy holds up, not to find parameters that happened to fit historical data.

---

## Results (2007–2024 backtest)

| Metric | Strategy | Benchmark (SPY buy & hold) |
|---|---|---|
| CAGR | 4.77% | 10.93% |
| Volatility (ann.) | 10.76% | — |
| Sharpe ratio | 0.30 | 0.52 |
| Max drawdown | -19.05% | -55.19% |
| Beta | -0.09 | 1.0 |
| Profit factor | 1.85 | — |

The strategy underperforms the benchmark on raw returns — and the validation gate in the code flags this explicitly with `[FAIL] Sharpe >= 1.0`. That result is presented as-is rather than explained away.

The more interesting finding is the **near-zero beta (-0.09)** and a maximum drawdown of -19% versus the benchmark's -55%. A strategy that loses less than a third as much during crashes, while maintaining a positive profit factor (1.85) and positive expectancy per trade ($5,310), has genuine portfolio utility as a diversifier even without beating a bull-market equity index outright. Whether that tradeoff is worthwhile is a question the paper explores in more depth.

### Walk-forward out-of-sample validation

To test whether results are a backtest artefact, the strategy was validated across three independent out-of-sample periods covering meaningfully different market regimes:

| Test period | Market regime | OOS CAGR | OOS Sharpe | OOS Max DD |
|---|---|---|---|---|
| 2012–2014 | Post-GFC recovery | 2.96% | 0.154 | -6.44% |
| 2017–2019 | Low-vol bull market | 2.87% | 0.150 | -9.76% |
| 2022–2024 | Rate hike cycle | 4.55% | 0.267 | -16.92% |

**Sharpe is positive across 100% of out-of-sample folds.** The strategy performs weakest during the low-volatility bull market of 2017–2019 — consistent with what the academic literature would predict for trend-following strategies in choppy, directionless conditions — and strongest during the 2022–2024 rate cycle, where cross-asset divergence created strong persistent trends. The pattern of results matches theory rather than contradicting it.

### Parameter sensitivity

Sharpe ratios remain stable as key parameters are varied, which suggests results are not dependent on a specific lucky choice of inputs:

| `target_portfolio_vol` | Sharpe | CAGR | Max DD |
|---|---|---|---|
| 6% | 0.31 | 4.37% | -13.33% |
| 8% | 0.31 | 4.79% | -16.78% |
| **10% (default)** | **0.30** | **4.77%** | **-19.05%** |
| 12% | 0.27 | 4.51% | -20.10% |
| 15% | 0.25 | 4.31% | -21.45% |

---

## Repository structure

```
├── systematic_trend_strategy.py     # Core library: config, signals, risk, backtester, metrics, walk-forward
├── run_strategy.py                  # Runner: loads data, executes backtest, prints full tearsheet
├── multi_asset_momentum_paper.docx  # Research paper: theoretical motivation and full methodology
└── trading strategy backtest results.png  # Equity curve and drawdown chart
```

### Running it

```bash
pip install numpy pandas yfinance

# Live download — fetches real ETF price history via yfinance
python run_strategy.py

# Offline demo with synthetic data, no internet needed
python run_strategy.py --synthetic

# Your own adjusted-close CSV (date index + one column per ticker)
python run_strategy.py --csv prices.csv
```

The runner prints a full performance report, walk-forward results, validation gate, and parameter sensitivity table.

### Paper trading hook

`run_strategy.py` exposes a `latest_target_weights()` function that returns the strategy's current target allocations as of the last available price bar. These weights can be passed directly to a broker API (Alpaca, IBKR) to run the strategy live with the same code path as the backtest.

---

## Research paper

The full theoretical motivation — covering the academic evidence for time-series momentum, the rationale for multi-asset diversification, inverse-volatility sizing, and an honest discussion of the strategy's limits — is in [`multi_asset_momentum_paper.docx`](./multi_asset_momentum_paper.docx).

---

## Limitations and honest caveats

- **Survivorship bias**: the ETF universe only includes instruments that exist today. ETFs that were delisted or merged are not represented, which likely biases results slightly upward.
- **Capacity**: the strategy assumes any trade size can be executed at modelled costs. At large capital levels this assumption breaks down.
- **Regime dependence**: trend-following is known to struggle in choppy, mean-reverting markets. The 2017–2019 out-of-sample period reflects this and is not glossed over.
- **Backtests are not promises.** The walk-forward validation shows the edge is real and stable across regimes, but future performance depends on markets continuing to exhibit trending behaviour at medium-term horizons — which is not guaranteed.

---

*Built as an independent research project. Not financial advice.*
