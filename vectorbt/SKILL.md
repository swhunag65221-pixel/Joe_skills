---
name: vectorbt
description: Comprehensive guide for VectorBT (vectorbt) — a vectorized backtesting library for Python. Use when working with vectorized portfolio simulation, from_signals, from_orders, parameter optimization, technical indicators via VectorBT, or when the user mentions vectorbt, vbt, vectorized backtest, or portfolio simulation. Includes data downloading, portfolio construction, indicator factories, optimization, and plotting.
compatibility: Requires Python 3.8+ and numpy/pandas. Install via pip install vectorbt.
---

# VectorBT — Vectorized Backtesting Framework

## Execution Philosophy

When a user asks for a VectorBT backtest: **execute the code and show results**. Do not provide code snippets for the user to run manually.

## Prerequisites

```bash
# Install vectorbt and dependencies
uv pip install --system vectorbt python-dotenv yfinance 2>/dev/null || pip install vectorbt python-dotenv yfinance

# Or use uv run for zero-setup execution
uv run --with vectorbt --with yfinance --with python-dotenv python3 script.py
```

## Quick Start

```python
import vectorbt as vbt
import numpy as np

# Download data
price = vbt.YFData.download("AAPL", start="2020-01-01", end="2024-01-01").get("Close")

# Create signals from moving average crossover
fast_ma = vbt.MA.run(price, window=10)
slow_ma = vbt.MA.run(price, window=50)

entries = fast_ma.ma_crossed_above(slow_ma)
exits = fast_ma.ma_crossed_below(slow_ma)

# Run backtest
pf = vbt.Portfolio.from_signals(price, entries, exits, init_cash=100000)

# Show results
print(pf.stats())
pf.plot().show()
```

## Architecture Overview

VectorBT is built on **vectorized operations** using NumPy and Numba JIT compilation. Unlike event-driven frameworks, it processes entire time series at once, enabling:

- **Massive parameter sweeps** — test thousands of parameter combinations simultaneously
- **Fast execution** — Numba-compiled core functions
- **Broadcasting** — automatic parameter combination via NumPy broadcasting

### Core Modules

| Module | Purpose | Reference File |
|--------|---------|----------------|
| `vbt.YFData` / `vbt.BinanceData` | Data downloading | `data-reference.md` |
| `vbt.Portfolio` | Portfolio simulation | `portfolio-reference.md` |
| `vbt.MA`, `vbt.RSI`, `vbt.BBANDS` | Technical indicators | `indicators-reference.md` |
| `vbt.IndicatorFactory` | Custom indicator creation | `indicators-reference.md` |
| Parameter arrays + broadcasting | Optimization | `optimization-reference.md` |

### Key Design Principle: Broadcasting

```python
# Test multiple MA windows simultaneously
fast_ma = vbt.MA.run(price, window=[10, 20, 30])  # 3 variants
slow_ma = vbt.MA.run(price, window=[50, 100])      # 2 variants

# Cross-product: 3 x 2 = 6 combinations automatically
entries = fast_ma.ma_crossed_above(slow_ma)
pf = vbt.Portfolio.from_signals(price, entries, exits)
print(pf.total_return())  # Returns for all 6 combos
```

## Common Workflow

1. **Download data** — `vbt.YFData.download()` or provide your own pandas DataFrame
2. **Compute indicators** — `vbt.MA.run()`, `vbt.RSI.run()`, etc.
3. **Generate signals** — Boolean entries/exits from indicator crossovers or conditions
4. **Simulate portfolio** — `vbt.Portfolio.from_signals()` or `from_orders()`
5. **Analyze results** — `.stats()`, `.total_return()`, `.plot()`
6. **Optimize** — Pass arrays of parameters, use broadcasting

## Reference Files

| File | Purpose |
|------|---------|
| `SKILL.md` | This file — overview & quick start |
| `data-reference.md` | Data downloading & management |
| `portfolio-reference.md` | Portfolio construction & simulation |
| `indicators-reference.md` | Built-in & custom indicators |
| `optimization-reference.md` | Parameter optimization & grid search |
| `best-practices.md` | Performance tips, anti-patterns, common pitfalls |
