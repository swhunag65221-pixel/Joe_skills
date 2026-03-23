# VectorBT Optimization Reference

## Parameter Broadcasting (Core Mechanism)

VectorBT's optimization is based on NumPy broadcasting — pass arrays of parameters and all combinations are tested simultaneously.

```python
import vectorbt as vbt
import numpy as np

close = vbt.YFData.download("AAPL", start="2020-01-01").get("Close")

# Test many MA windows at once
fast_windows = np.arange(5, 50, 5)    # [5, 10, 15, ..., 45]
slow_windows = np.arange(50, 200, 10)  # [50, 60, 70, ..., 190]

fast_ma = vbt.MA.run(close, window=fast_windows, short_name="fast")
slow_ma = vbt.MA.run(close, window=slow_windows, short_name="slow")

# All cross-combinations generated automatically
entries = fast_ma.ma_crossed_above(slow_ma)
exits = fast_ma.ma_crossed_below(slow_ma)

pf = vbt.Portfolio.from_signals(close, entries, exits)

# Results for ALL parameter combinations
returns = pf.total_return()
sharpes = pf.sharpe_ratio()

# Find best combination
best_idx = sharpes.idxmax()
print(f"Best params: fast={best_idx[0]}, slow={best_idx[1]}")
print(f"Sharpe: {sharpes[best_idx]:.2f}")
print(f"Return: {returns[best_idx]:.2%}")
```

## Heatmap Visualization

```python
# Reshape results into 2D heatmap
returns_2d = returns.unstack()

import plotly.express as px
fig = px.imshow(
    returns_2d.values,
    x=[str(c) for c in returns_2d.columns],
    y=[str(i) for i in returns_2d.index],
    labels=dict(x="Slow Window", y="Fast Window", color="Total Return"),
    color_continuous_scale="RdYlGn",
    title="MA Crossover Parameter Heatmap"
)
fig.show()

# Or use VectorBT's built-in heatmap
returns.vbt.heatmap().show()
```

## RSI Parameter Optimization

```python
# Test RSI window + overbought/oversold thresholds
rsi_windows = np.arange(7, 28, 7)          # [7, 14, 21]
entry_thresholds = np.arange(20, 40, 5)    # [20, 25, 30, 35]
exit_thresholds = np.arange(65, 85, 5)     # [65, 70, 75, 80]

rsi = vbt.RSI.run(close, window=rsi_windows)

# For multi-dimensional parameter sweeps, build manually
results = {}
for rsi_w in rsi_windows:
    for entry_t in entry_thresholds:
        for exit_t in exit_thresholds:
            r = vbt.RSI.run(close, window=rsi_w)
            entries = r.rsi_below(entry_t)
            exits = r.rsi_above(exit_t)
            pf = vbt.Portfolio.from_signals(close, entries, exits)
            results[(rsi_w, entry_t, exit_t)] = {
                "return": pf.total_return(),
                "sharpe": pf.sharpe_ratio(),
                "max_dd": pf.max_drawdown(),
                "trades": pf.total_trades(),
            }

results_df = pd.DataFrame(results).T
results_df.index.names = ["rsi_window", "entry_thresh", "exit_thresh"]
print(results_df.sort_values("sharpe", ascending=False).head(10))
```

## Walk-Forward Optimization

Split data into in-sample (train) and out-of-sample (test) periods.

```python
# Rolling split
(in_price, in_indexes), (out_price, out_indexes) = close.vbt.rolling_split(
    n=5,             # 5 folds
    window_len=504,  # 2 years in-sample
    set_lens=(252,), # 1 year out-of-sample
    left_to_right=False
)

# Optimize on each in-sample period
best_params = []
for i in range(len(in_price)):
    in_close = in_price[i]

    fast_ma = vbt.MA.run(in_close, window=np.arange(5, 50, 5))
    slow_ma = vbt.MA.run(in_close, window=np.arange(50, 200, 10))

    entries = fast_ma.ma_crossed_above(slow_ma)
    exits = fast_ma.ma_crossed_below(slow_ma)

    pf = vbt.Portfolio.from_signals(in_close, entries, exits)
    best_idx = pf.sharpe_ratio().idxmax()
    best_params.append(best_idx)

# Test on out-of-sample periods
oos_returns = []
for i in range(len(out_price)):
    out_close = out_price[i]
    fast_w, slow_w = best_params[i]

    fast_ma = vbt.MA.run(out_close, window=fast_w)
    slow_ma = vbt.MA.run(out_close, window=slow_w)

    entries = fast_ma.ma_crossed_above(slow_ma)
    exits = fast_ma.ma_crossed_below(slow_ma)

    pf = vbt.Portfolio.from_signals(out_close, entries, exits)
    oos_returns.append(pf.total_return())

print(f"Average OOS Return: {np.mean(oos_returns):.2%}")
```

## Monte Carlo Simulation

```python
# Random signal generation for statistical significance testing
n_tests = 1000
random_sharpes = []

for seed in range(n_tests):
    pf = vbt.Portfolio.from_random_signals(
        close, n=20, seed=seed, init_cash=100000
    )
    random_sharpes.append(pf.sharpe_ratio())

random_sharpes = np.array(random_sharpes)

# Compare strategy Sharpe against random distribution
strategy_sharpe = strategy_pf.sharpe_ratio()
percentile = (random_sharpes < strategy_sharpe).mean() * 100
print(f"Strategy Sharpe: {strategy_sharpe:.2f}")
print(f"Percentile vs Random: {percentile:.1f}%")
```

## Multi-Symbol Optimization

```python
# Test same strategy across multiple symbols
symbols = ["AAPL", "GOOGL", "MSFT", "AMZN", "META"]
data = vbt.YFData.download(symbols, start="2020-01-01")
close = data.get("Close")

# Each column is a symbol — signals broadcast automatically
fast_ma = vbt.MA.run(close, window=20)
slow_ma = vbt.MA.run(close, window=50)

entries = fast_ma.ma_crossed_above(slow_ma)
exits = fast_ma.ma_crossed_below(slow_ma)

pf = vbt.Portfolio.from_signals(close, entries, exits)

# Per-symbol stats
for symbol in symbols:
    print(f"{symbol}: Return={pf[symbol].total_return():.2%}, "
          f"Sharpe={pf[symbol].sharpe_ratio():.2f}")
```

## Performance Tips

- **Use broadcasting** instead of for-loops whenever possible — orders of magnitude faster
- **Limit parameter ranges** — don't test 10,000+ combinations unless necessary
- **Use Numba** for custom indicators in optimization loops
- **Cache data** — download once, reuse across optimization runs
- **Profile first** — use `%%timeit` to identify bottlenecks before optimizing

## Overfitting Warning

Parameter optimization is prone to overfitting. Always:

1. **Use walk-forward validation** — never optimize and test on the same data
2. **Check parameter stability** — good parameters should be in a "plateau", not a spike
3. **Run Monte Carlo** — compare your strategy against random signals
4. **Penalize complexity** — prefer simpler strategies with fewer parameters
5. **Test robustness** — small parameter changes should not drastically change results
