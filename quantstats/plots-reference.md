# QuantStats Plots Reference

## All Plot Functions

All functions in `qs.plots` accept a pandas Series of **returns** and return a matplotlib figure.

### Snapshot (Overview)

```python
import quantstats as qs

# Multi-panel overview: cumulative returns, drawdown, monthly returns, return distribution
qs.plots.snapshot(
    returns,
    title="Strategy Overview",
    grayscale=False,
    figsize=(10, 8),
    savefig="snapshot.png",     # Optional: save to file
    show=True,                  # Display plot
)
```

### Cumulative Returns

```python
# Strategy equity curve
qs.plots.returns(
    returns,
    benchmark=bench,
    cumulative=True,
    savefig="cumulative.png",
)

# Log scale cumulative returns
qs.plots.log_returns(
    returns,
    benchmark=bench,
    savefig="log_returns.png",
)
```

### Drawdown

```python
# Drawdown chart
qs.plots.drawdown(returns, savefig="drawdown.png")

# Top drawdown periods highlighted
qs.plots.drawdowns_periods(returns, savefig="dd_periods.png")
```

### Monthly & Yearly Returns

```python
# Monthly returns heatmap
qs.plots.monthly_heatmap(
    returns,
    compounded=True,
    savefig="monthly_heatmap.png",
)

# Yearly returns bar chart
qs.plots.yearly_returns(
    returns,
    benchmark=bench,
    savefig="yearly.png",
)

# EOY (End of Year) returns
qs.plots.eoy_returns(returns, savefig="eoy.png")
```

### Distribution

```python
# Daily returns distribution histogram
qs.plots.histogram(
    returns,
    compounded=True,
    savefig="histogram.png",
)

# Return quantiles (box plot by timeframe)
qs.plots.return_quantiles(returns, savefig="quantiles.png")

# Daily returns scatter/strip
qs.plots.daily_returns(returns, savefig="daily.png")
```

### Rolling Metrics

```python
# Rolling Sharpe ratio
qs.plots.rolling_sharpe(
    returns,
    rf=0.0,
    rolling_period=126,     # 6-month rolling window
    savefig="rolling_sharpe.png",
)

# Rolling Sortino ratio
qs.plots.rolling_sortino(
    returns,
    rf=0.0,
    rolling_period=126,
    savefig="rolling_sortino.png",
)

# Rolling Volatility
qs.plots.rolling_volatility(
    returns,
    rolling_period=126,
    savefig="rolling_vol.png",
)

# Rolling Beta (requires benchmark)
qs.plots.rolling_beta(
    returns,
    benchmark=bench,
    rolling_period=126,
    savefig="rolling_beta.png",
)
```

### Comparison Plots (vs Benchmark)

```python
# Strategy vs Benchmark cumulative returns
qs.plots.returns(returns, benchmark=bench, savefig="comparison.png")

# Relative performance (strategy / benchmark)
# Calculate manually:
# relative = (1 + returns).cumprod() / (1 + bench).cumprod()
```

## Common Parameters

All plot functions share these parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `returns` | Series | Daily returns (required) |
| `benchmark` | Series/str | Benchmark returns or ticker symbol |
| `grayscale` | bool | Black & white mode (default: False) |
| `figsize` | tuple | Figure size as (width, height) |
| `savefig` | str | Save plot to file path |
| `show` | bool | Display plot (default: True) |
| `title` | str | Custom plot title |

## Saving Plots

```python
# Save to file (any format matplotlib supports)
qs.plots.snapshot(returns, savefig="report.png")
qs.plots.snapshot(returns, savefig="report.pdf")
qs.plots.snapshot(returns, savefig="report.svg")

# Get matplotlib figure for custom modification
import matplotlib.pyplot as plt
fig = qs.plots.snapshot(returns, show=False)
# ... customize fig ...
fig.savefig("custom.png", dpi=300, bbox_inches="tight")
plt.close(fig)
```

## Pandas Extension Plots

```python
qs.extend_pandas()

# Call plot methods directly on returns Series
returns.plot_snapshot()
returns.plot_drawdown()
returns.plot_monthly_heatmap()
returns.plot_yearly_returns()
returns.plot_histogram()
returns.plot_rolling_sharpe()
returns.plot_rolling_sortino()
returns.plot_rolling_volatility()
returns.plot_rolling_beta(benchmark=bench)
```

## Important Notes

- **Backend**: QuantStats uses matplotlib. For headless servers, set `matplotlib.use("Agg")` before importing
- **Save before show**: If using `savefig`, the file is saved before `show=True` displays it
- **Timezone**: Strip timezone from index if you get errors: `returns.index = returns.index.tz_localize(None)`
- **Resolution**: For publication quality, save as SVG or use `dpi=300` with PNG
- **Headless execution**: Use `show=False` when running in scripts; use `savefig` to save output
