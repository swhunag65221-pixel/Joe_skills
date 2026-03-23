---
name: quantstats
description: Comprehensive guide for QuantStats — a Python library for portfolio analytics and performance metrics. Use when working with performance reports, tearsheets, Sharpe ratio, drawdown analysis, monthly returns heatmaps, or when the user mentions quantstats, qs, performance report, tearsheet, portfolio analytics, or risk metrics. Includes HTML reports, 50+ metrics, and publication-quality plots.
compatibility: Requires Python 3.7+ and pandas. Install via pip install quantstats.
---

# QuantStats — Portfolio Analytics & Reporting

## Execution Philosophy

When a user asks for a performance report or metrics: **execute the code and show results**. Generate HTML reports and open them. Print metrics directly. Do not provide code snippets for the user to run manually.

## Prerequisites

```bash
# Install quantstats
uv pip install --system quantstats python-dotenv yfinance 2>/dev/null || pip install quantstats python-dotenv yfinance

# Or use uv run for zero-setup execution
uv run --with quantstats --with yfinance --with python-dotenv python3 script.py
```

## Quick Start

```python
import quantstats as qs

# Extend pandas with QuantStats methods
qs.extend_pandas()

# Download returns data (returns, not prices!)
stock = qs.utils.download_returns("AAPL", period="5y")
bench = qs.utils.download_returns("SPY", period="5y")

# Generate full HTML report
qs.reports.html(stock, benchmark=bench, output="report.html", title="AAPL Analysis")

# Or get individual metrics
print(f"CAGR: {qs.stats.cagr(stock):.2%}")
print(f"Sharpe: {qs.stats.sharpe(stock):.2f}")
print(f"Max Drawdown: {qs.stats.max_drawdown(stock):.2%}")
```

## Architecture Overview

QuantStats has three core modules:

| Module | Purpose | Reference File |
|--------|---------|----------------|
| `qs.stats` | 50+ performance & risk metrics | `metrics-reference.md` |
| `qs.plots` | Publication-quality visualizations | `plots-reference.md` |
| `qs.reports` | Pre-built report templates (HTML, console) | `reports-reference.md` |
| `qs.utils` | Data downloading & preparation helpers | This file |

### Key Design Principle: Returns-Based

QuantStats works with **returns** (percent change), not prices. Most functions expect a pandas Series of daily returns.

```python
# Convert prices to returns
import pandas as pd
prices = pd.read_csv("prices.csv", index_col=0, parse_dates=True)["Close"]
returns = prices.pct_change().dropna()

# Or download returns directly
returns = qs.utils.download_returns("AAPL")
```

## Utility Functions

```python
# Download returns (wrapper around yfinance)
returns = qs.utils.download_returns("AAPL", period="5y")
returns = qs.utils.download_returns("AAPL", start="2020-01-01", end="2024-01-01")

# Convert prices to returns
returns = qs.utils.to_returns(prices)

# Convert returns to prices (for cumulative plots)
prices = qs.utils.to_prices(returns, base=1.0)

# Make returns index timezone-naive (common fix)
returns.index = returns.index.tz_localize(None)
```

## Pandas Extension

```python
qs.extend_pandas()

# Now use directly on pandas Series
returns = prices.pct_change().dropna()
returns.sharpe()
returns.max_drawdown()
returns.monthly_returns()
returns.plot_snapshot()
```

## Common Workflow

1. **Prepare returns** — Download or convert prices to returns Series
2. **Quick metrics** — `qs.stats.sharpe()`, `qs.stats.cagr()`, etc.
3. **Generate report** — `qs.reports.html()` for comprehensive analysis
4. **Custom plots** — `qs.plots.snapshot()`, `qs.plots.drawdown()`, etc.

## Reference Files

| File | Purpose |
|------|---------|
| `SKILL.md` | This file — overview & quick start |
| `metrics-reference.md` | All 50+ performance & risk metrics |
| `reports-reference.md` | HTML reports, tearsheets, console output |
| `plots-reference.md` | All visualization functions |
| `best-practices.md` | Common pitfalls, tips, integration patterns |
