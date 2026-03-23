# QuantStats Best Practices

## Critical Anti-Patterns

### Never Pass Prices Instead of Returns

```python
# BAD — prices are not returns! All metrics will be wrong
qs.stats.sharpe(prices)

# GOOD — convert to returns first
returns = prices.pct_change().dropna()
qs.stats.sharpe(returns)

# Or use the utility
returns = qs.utils.to_returns(prices)
```

This is the #1 most common mistake. QuantStats functions silently accept any numeric Series — they don't validate whether input is returns or prices.

### Never Ignore Timezone Mismatches

```python
# BAD — mixed timezones cause alignment errors
returns = yf.download("AAPL")["Close"].pct_change()  # tz-aware
bench = other_source_returns  # tz-naive
qs.reports.html(returns, benchmark=bench)  # May crash or misalign

# GOOD — normalize timezones
returns.index = returns.index.tz_localize(None)
bench.index = bench.index.tz_localize(None)
qs.reports.html(returns, benchmark=bench)
```

### Never Compare Misaligned Date Ranges

```python
# BAD — different date ranges distort comparison
strategy = returns["2020":"2024"]
benchmark = bench["2018":"2024"]  # Different start!
qs.reports.html(strategy, benchmark=benchmark)

# GOOD — align dates explicitly
common = strategy.index.intersection(benchmark.index)
qs.reports.html(strategy[common], benchmark=benchmark[common])

# Or use match_dates=True (default in html report)
qs.reports.html(strategy, benchmark=benchmark, match_dates=True)
```

### Never Forget to Drop NaN

```python
# BAD — NaN from pct_change() can distort metrics
returns = prices.pct_change()  # First row is NaN
qs.stats.sharpe(returns)       # NaN included

# GOOD — always dropna
returns = prices.pct_change().dropna()
qs.stats.sharpe(returns)
```

### Never Assume Annual Risk-Free Rate

```python
# BAD — using raw annual rate
qs.stats.sharpe(returns, rf=0.04)  # This is treated as DAILY rf

# GOOD — QuantStats rf parameter is annualized rate (it converts internally)
# Actually, qs.stats.sharpe() rf IS annual rate, but verify per function
# When in doubt, check the source or set rf=0.0 for simplicity
```

## Common Patterns

### Complete Workflow

```python
import quantstats as qs
import pandas as pd

# 1. Prepare data
prices = pd.read_csv("prices.csv", index_col=0, parse_dates=True)["Close"]
returns = prices.pct_change().dropna()

# 2. Strip timezone if needed
returns.index = returns.index.tz_localize(None)

# 3. Quick metrics check
print(f"CAGR: {qs.stats.cagr(returns):.2%}")
print(f"Sharpe: {qs.stats.sharpe(returns):.2f}")
print(f"Max DD: {qs.stats.max_drawdown(returns):.2%}")
print(f"Volatility: {qs.stats.volatility(returns):.2%}")

# 4. Full report
qs.reports.html(returns, benchmark="SPY", output="report.html", title="My Strategy")
```

### Multi-Strategy Comparison

```python
# Compare multiple strategies
strategies = {
    "Strategy A": returns_a,
    "Strategy B": returns_b,
    "Strategy C": returns_c,
}

# Metrics comparison table
comparison = pd.DataFrame({
    name: {
        "CAGR": qs.stats.cagr(r),
        "Sharpe": qs.stats.sharpe(r),
        "Sortino": qs.stats.sortino(r),
        "Max DD": qs.stats.max_drawdown(r),
        "Volatility": qs.stats.volatility(r),
        "Calmar": qs.stats.calmar(r),
        "Win Rate": qs.stats.win_rate(r),
    }
    for name, r in strategies.items()
})
print(comparison.T.to_string(float_format="{:.4f}".format))

# Individual reports
for name, r in strategies.items():
    qs.reports.html(r, output=f"{name}_report.html", title=name)
```

### Integration with VectorBT

```python
import vectorbt as vbt
import quantstats as qs

# Run VectorBT backtest
pf = vbt.Portfolio.from_signals(close, entries, exits, fees=0.001)

# Get returns
returns = pf.returns()
returns.index = returns.index.tz_localize(None)

# VectorBT stats (quick)
print(pf.stats())

# QuantStats report (comprehensive)
qs.reports.html(returns, benchmark="SPY", output="full_report.html")
```

### Integration with FinLab (台股)

```python
from finlab.backtest import sim
import quantstats as qs

# Run FinLab backtest
report = sim(position, resample="M", upload=False)

# Extract daily returns from equity curve
equity = report.equity
returns = equity.pct_change().dropna()

# Taiwan market benchmark: 0050 (元大台灣50)
bench_close = data.get("price:收盤價")["0050"]
bench_returns = bench_close.pct_change().dropna()

# Align dates
common = returns.index.intersection(bench_returns.index)
qs.reports.html(
    returns[common],
    benchmark=bench_returns[common],
    output="tw_strategy_report.html",
    title="台股策略分析"
)
```

### Headless Server Execution

```python
import matplotlib
matplotlib.use("Agg")  # MUST be before any other matplotlib import
import quantstats as qs

# Generate report without display
qs.reports.html(returns, output="report.html")

# Save plots without showing
qs.plots.snapshot(returns, savefig="snapshot.png", show=False)
```

## Performance Tips

- **Large datasets**: HTML reports with 20+ years of data can be slow. Consider filtering to relevant periods.
- **Multiple reports**: Batch generate reports in a loop, using `show=False` to avoid display overhead.
- **Memory**: For many strategies, process one at a time and clear plots with `plt.close("all")`.

## Metric Interpretation Guide

| Metric | Good | Great | Warning |
|--------|------|-------|---------|
| Sharpe | > 1.0 | > 2.0 | < 0.5 |
| Sortino | > 1.5 | > 3.0 | < 0.5 |
| Max Drawdown | > -20% | > -10% | < -40% |
| Calmar | > 1.0 | > 3.0 | < 0.5 |
| Win Rate | > 50% | > 60% | < 40% |
| CAGR | > 10% | > 20% | < 5% |

These are rough guidelines for equity strategies. Adjust for your asset class and strategy type.
