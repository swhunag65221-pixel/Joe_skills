# QuantStats Reports Reference

## HTML Report

The flagship feature — generates a comprehensive HTML tearsheet.

```python
import quantstats as qs

# Full HTML report with benchmark comparison
qs.reports.html(
    returns,                    # pandas Series of returns
    benchmark=bench_returns,    # Optional benchmark returns Series
    output="report.html",       # Output file path
    title="My Strategy",        # Report title
    rf=0.0,                     # Risk-free rate (decimal)
    grayscale=False,            # Grayscale mode for printing
    compounded=True,            # Use compounded returns
    periods_per_year=252,       # Trading days per year
    match_dates=True,           # Align strategy and benchmark dates
    download_filename="report.html",  # Download button filename
)
```

### HTML Report Sections

The generated report includes:
1. **Summary** — CAGR, Sharpe, Sortino, Max Drawdown, Volatility
2. **Monthly Returns Heatmap** — Color-coded monthly returns table
3. **Cumulative Returns** — Strategy vs benchmark equity curve
4. **End of Year Returns** — Bar chart of annual returns
5. **Drawdown Periods** — Top 5 worst drawdowns with dates
6. **Rolling Sharpe** — 6-month rolling Sharpe ratio
7. **Rolling Volatility** — 6-month rolling volatility
8. **Rolling Beta** — Rolling beta vs benchmark (if provided)
9. **Monthly Distribution** — Histogram of monthly returns
10. **Return Quantiles** — Box plot of return distribution
11. **Worst Drawdowns** — Table of worst drawdown periods

## Console Reports

```python
# Full report to console (stdout)
qs.reports.full(returns, benchmark=bench)

# Basic report (key metrics only)
qs.reports.basic(returns, benchmark=bench)
```

## Metrics Table

```python
# Display metrics as a formatted table
qs.reports.metrics(
    returns,
    benchmark=bench,
    mode="full",         # "full", "basic"
    sep=True,            # Separator lines
    display=True,        # Print to stdout
    compounded=True,
    periods_per_year=252,
    rf=0.0,
)

# Get metrics as DataFrame (for programmatic access)
metrics_df = qs.reports.metrics(
    returns,
    benchmark=bench,
    mode="full",
    display=False,       # Don't print, return DataFrame
)
```

### Metrics Table Output Example

```
                           Strategy    Benchmark
─────────────────────────────────────────────────
Start Period               2020-01-02  2020-01-02
End Period                 2024-12-31  2024-12-31
Risk-Free Rate             0.0%        0.0%
─────────────────────────────────────────────────
CAGR                       18.42%      12.15%
Sharpe                     1.24        0.89
Sortino                    1.87        1.21
Max Drawdown               -15.3%      -23.1%
Volatility (ann.)          14.8%       18.2%
...
```

## Generating Reports from Different Data Sources

### From VectorBT Portfolio

```python
import vectorbt as vbt
import quantstats as qs

# Run VectorBT backtest
pf = vbt.Portfolio.from_signals(close, entries, exits)

# Extract returns for QuantStats
strategy_returns = pf.returns()
strategy_returns.index = strategy_returns.index.tz_localize(None)  # Fix timezone

# Generate report
qs.reports.html(strategy_returns, benchmark="SPY", output="vbt_report.html")
```

### From FinLab Backtest

```python
from finlab.backtest import sim

# Run FinLab backtest
report = sim(position, resample="M", upload=False)

# Get returns
strategy_returns = report.daily_return  # or calculate from equity curve

# Generate QuantStats report
qs.reports.html(strategy_returns, output="finlab_report.html", title="FinLab Strategy")
```

### From Raw Price Data

```python
import yfinance as yf

# Download prices
prices = yf.download("AAPL", start="2020-01-01")["Close"]
returns = prices.pct_change().dropna()

bench_prices = yf.download("SPY", start="2020-01-01")["Close"]
bench_returns = bench_prices.pct_change().dropna()

# Align dates
common_dates = returns.index.intersection(bench_returns.index)
returns = returns[common_dates]
bench_returns = bench_returns[common_dates]

qs.reports.html(returns, benchmark=bench_returns, output="report.html")
```

### Using Ticker Symbols as Benchmark

```python
# QuantStats can download benchmark data automatically (via string symbol)
qs.reports.html(returns, benchmark="SPY", output="report.html")

# This downloads SPY returns internally and aligns dates
```

## Important Notes

- **`output` parameter** — if provided, saves HTML to file; if omitted, returns HTML string
- **Benchmark as string** — pass a ticker symbol (e.g., "SPY") and QuantStats downloads it automatically
- **Benchmark as Series** — pass pre-computed returns Series for custom benchmarks
- **Timezone issues** — if you get timezone errors, strip timezone: `returns.index = returns.index.tz_localize(None)`
- **Date alignment** — use `match_dates=True` (default) to align strategy and benchmark dates
- **Large datasets** — HTML reports with 10+ years of daily data may be slow to generate
