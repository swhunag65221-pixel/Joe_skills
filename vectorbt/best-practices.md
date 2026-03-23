# VectorBT Best Practices

## Critical Anti-Patterns

### Never Use For-Loops for Parameter Sweeps

```python
# BAD — slow, defeats the purpose of VectorBT
for window in range(10, 100):
    ma = vbt.MA.run(close, window=window)
    # ...

# GOOD — vectorized, orders of magnitude faster
ma = vbt.MA.run(close, window=np.arange(10, 100))
```

### Never Confuse Returns with Prices

```python
# BAD — feeding prices where returns are expected
pf.returns()  # These ARE returns, not prices

# GOOD — understand what each method returns
returns = pf.returns()           # Percent returns
cum_returns = pf.cumulative_returns()  # Cumulative returns
values = pf.value()              # Portfolio dollar value
```

### Never Ignore Lookahead Bias

```python
# BAD — using future data in signals
signal = close > close.shift(-1)  # FUTURE data leak!

# GOOD — only use past/present data
signal = close > close.shift(1)   # Compare to PREVIOUS day
```

### Never Skip Transaction Costs

```python
# BAD — unrealistic backtest
pf = vbt.Portfolio.from_signals(close, entries, exits)

# GOOD — include fees and slippage
pf = vbt.Portfolio.from_signals(
    close, entries, exits,
    fees=0.001,       # 0.1% commission
    slippage=0.001,   # 0.1% slippage
)
```

### Never Optimize Without Validation

```python
# BAD — optimize and report on same data
pf = vbt.Portfolio.from_signals(close, entries, exits)
best = pf.sharpe_ratio().idxmax()  # This is in-sample!

# GOOD — walk-forward or train/test split
train = close["2020":"2022"]
test = close["2023":"2024"]
# Optimize on train, validate on test
```

## Performance Guidelines

### Data Management

```python
# Cache expensive downloads
data = vbt.YFData.download("AAPL", start="2015-01-01")
data.to_hdf("cache/aapl.h5")

# Reuse cached data
data = vbt.YFData.from_hdf("cache/aapl.h5")
```

### Memory Management

```python
# For large parameter sweeps, process in chunks
import gc

chunk_size = 100
all_results = []

for i in range(0, len(param_combos), chunk_size):
    chunk = param_combos[i:i+chunk_size]
    # ... run backtest on chunk ...
    all_results.append(chunk_results)
    gc.collect()  # Free memory between chunks
```

### Use Numba for Custom Logic

```python
from numba import njit

@njit
def custom_signal_nb(close, window, threshold):
    """Numba-compiled signal generator — runs 10-100x faster."""
    n = len(close)
    entries = np.full(n, False)
    exits = np.full(n, False)

    for i in range(window, n):
        ret = (close[i] - close[i - window]) / close[i - window]
        if ret < -threshold:
            entries[i] = True
        elif ret > threshold:
            exits[i] = True

    return entries, exits
```

## Common Patterns

### Strategy Comparison

```python
# Compare multiple strategies side-by-side
strategies = {}

# Strategy 1: MA Crossover
fast_ma = vbt.MA.run(close, window=20)
slow_ma = vbt.MA.run(close, window=50)
e1 = fast_ma.ma_crossed_above(slow_ma)
x1 = fast_ma.ma_crossed_below(slow_ma)
strategies["MA Cross"] = vbt.Portfolio.from_signals(close, e1, x1, fees=0.001)

# Strategy 2: RSI Mean Reversion
rsi = vbt.RSI.run(close, window=14)
e2 = rsi.rsi_below(30)
x2 = rsi.rsi_above(70)
strategies["RSI"] = vbt.Portfolio.from_signals(close, e2, x2, fees=0.001)

# Strategy 3: Buy & Hold
strategies["B&H"] = vbt.Portfolio.from_holding(close, fees=0.001)

# Compare
for name, pf in strategies.items():
    print(f"{name}: Return={pf.total_return():.2%}, "
          f"Sharpe={pf.sharpe_ratio():.2f}, "
          f"MaxDD={pf.max_drawdown():.2%}")
```

### Integration with QuantStats

```python
import quantstats as qs

# Get returns from VectorBT portfolio
returns = pf.returns()

# Generate QuantStats report
qs.reports.html(returns, benchmark="SPY", output="report.html")
```

### Integration with FinLab (台股)

```python
from dotenv import load_dotenv
load_dotenv()
from finlab import data

# Get Taiwan stock data
close = data.get("price:收盤價")

# Pick a single stock for VectorBT analysis
stock_close = close["2330"]  # TSMC

# Use VectorBT for backtesting
fast_ma = vbt.MA.run(stock_close, window=20)
slow_ma = vbt.MA.run(stock_close, window=60)
entries = fast_ma.ma_crossed_above(slow_ma)
exits = fast_ma.ma_crossed_below(slow_ma)
pf = vbt.Portfolio.from_signals(stock_close, entries, exits, fees=0.001425)
print(pf.stats())
```

## Debugging Tips

1. **Check signal counts**: `entries.sum()`, `exits.sum()` — zero signals = no trades
2. **Inspect trades**: `pf.trades.records_readable` — see actual trade entries/exits
3. **Verify data alignment**: Ensure entries/exits index matches price index
4. **Check for NaN**: `close.isna().sum()` — NaN values can cause silent issues
5. **Print stats**: Always call `pf.stats()` to verify the backtest ran correctly

## Frequency & Resampling

```python
# Specify correct frequency for accurate annualized metrics
pf = vbt.Portfolio.from_signals(close, entries, exits, freq="1D")  # Daily
pf = vbt.Portfolio.from_signals(close, entries, exits, freq="1H")  # Hourly
pf = vbt.Portfolio.from_signals(close, entries, exits, freq="5T")  # 5-minute

# Without freq, Sharpe ratio and other annualized metrics will be wrong!
```
