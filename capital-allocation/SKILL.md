---
name: capital-allocation
description: Guide for portfolio capital allocation, position sizing, and risk budgeting. Use when working with Kelly Criterion, position sizing, MDD targets, Monte Carlo risk simulation, or when the user mentions capital allocation, 資金配置, 部位大小, risk budget, Kelly, Half-Kelly, drawdown target, or portfolio allocation. Includes linear scaling, precise simulation, and Monte Carlo confidence methods.
compatibility: Requires Python 3.8+ with numpy, pandas, quantstats, scipy.
---

# Capital Allocation & Position Sizing

## Execution Philosophy

When a user asks about capital allocation: **run the calculation and show results**. Don't just explain theory — compute the exact allocation percentage for their specific strategy and risk tolerance.

## Prerequisites

```bash
# Core packages (usually already installed)
uv pip install --system quantstats numpy pandas scipy yfinance
```

## Quick Start

```python
import numpy as np
import quantstats as qs

# Given a strategy's daily returns Series
# Find allocation for target MDD < 10%

strategy_mdd = abs(qs.stats.max_drawdown(strategy_returns))
target_mdd = 0.10

# Simple linear estimate
allocation = target_mdd / strategy_mdd
print(f"Allocate {allocation*100:.0f}% to strategy, {(1-allocation)*100:.0f}% to cash")
```

## Five Methods for Position Sizing

### Method 1: Linear Scaling (Simplest)

```python
allocation = target_mdd / actual_mdd
```

Fast but assumes drawdown scales linearly. Slightly optimistic.

### Method 2: Precise Simulation

```python
daily_cash = (1 + 0.04) ** (1/252) - 1  # 4% annual cash rate

results = []
for pct in range(5, 101, 5):
    alloc = pct / 100
    blended = strategy_returns * alloc + daily_cash * (1 - alloc)
    results.append({
        'allocation': pct,
        'cagr': qs.stats.cagr(blended),
        'sharpe': qs.stats.sharpe(blended),
        'mdd': qs.stats.max_drawdown(blended),
        'calmar': qs.stats.calmar(blended),
    })

# Find the maximum allocation where MDD stays within target
max_alloc = max(r['allocation'] for r in results if abs(r['mdd']) <= target_mdd)
```

Most accurate for historical data, but only reflects one path.

### Method 3: Binary Search for Exact Target

```python
lo, hi = 0.01, 1.0
for _ in range(50):
    mid = (lo + hi) / 2
    blended = strategy_returns * mid + daily_cash * (1 - mid)
    if abs(qs.stats.max_drawdown(blended)) > target_mdd:
        hi = mid
    else:
        lo = mid
exact_allocation = (lo + hi) / 2
```

Finds the precise allocation where MDD equals the target.

### Method 4: Monte Carlo with Block Bootstrap

```python
n_sims = 1000
block_size = 63  # ~3 months, preserves serial correlation
strategy_arr = strategy_returns.values
n_days = len(strategy_arr)

mc_mdds = []
for _ in range(n_sims):
    n_blocks = n_days // block_size + 1
    starts = np.random.randint(0, n_days - block_size, size=n_blocks)
    boot = np.concatenate([strategy_arr[s:s+block_size] for s in starts])[:n_days]

    blended = boot * allocation + daily_cash * (1 - allocation)
    equity = np.cumprod(1 + blended)
    peak = np.maximum.accumulate(equity)
    dd = (equity - peak) / peak
    mc_mdds.append(dd.min())

mc_mdds = np.array(mc_mdds)
prob_exceed = (mc_mdds < -target_mdd).mean()
print(f"Probability of exceeding -{target_mdd*100}% MDD: {prob_exceed*100:.1f}%")
```

The gold standard — gives probability-based confidence intervals.

### Method 5: Kelly Criterion

```python
# Discrete Kelly (from trade-level stats)
win_rate = len(wins) / total_trades
avg_win = wins.mean()
avg_loss = abs(losses.mean())
kelly = (win_rate * avg_win - (1 - win_rate) * avg_loss) / avg_win
half_kelly = kelly / 2  # More conservative, widely recommended

# Continuous Kelly (from daily returns)
daily_kelly = qs.stats.kelly_criterion(strategy_returns)
```

Kelly maximizes long-term geometric growth but assumes you know the true distribution. **Always use Half-Kelly or less** in practice.

## Decision Framework

```
Target MDD ──► Method 2 (Precise Simulation) ──► Get "backtest allocation"
                                                          │
                                                          ▼
                                              Method 4 (Monte Carlo)
                                                          │
                                              ┌───────────┴───────────┐
                                              ▼                       ▼
                                     P(exceed) < 5%          P(exceed) > 5%
                                     Use this allocation     Reduce allocation
                                                             until P < 5%
```

## Typical Results Table

For a strategy with MDD = -26.4% (like Ralph V3 + VOO/MA200):

| Target MDD | Linear | Precise | MC 95% | Half-Kelly |
|------------|--------|---------|--------|------------|
| ≤ 5% | 19% | 20% | ~13% | — |
| ≤ 10% | 38% | 37% | ~24% | 25% |
| ≤ 15% | 57% | 50% | ~35% | — |
| ≤ 20% | 76% | 70% | ~50% | — |
| ≤ 25% | 95% | 90% | ~65% | — |

**Key insight**: The MC 95% allocation is typically 60-70% of the backtest-precise allocation. This is the safety margin for out-of-sample degradation.

## Cash Alternatives for Non-Allocated Capital

| Vehicle | Expected Return | Risk | Use When |
|---------|----------------|------|----------|
| Money Market / T-Bills | ~4-5% | Near zero | Default safe choice |
| Short-term bonds (SHY) | ~3-4% | Very low | Slightly more yield |
| VOO (S&P 500) | ~10-14% | Moderate | If comfortable with equity risk |
| VOO with MA200 filter | ~10-14% (bear-filtered) | Lower | Best risk-adjusted idle |

## Common Pitfalls

- **Backtest MDD is minimum, not typical** — future MDD will likely be worse
- **Kelly overestimates** — assumes perfect knowledge of return distribution
- **Correlation during crises** — cash alternatives may correlate with strategy during crashes
- **Rebalancing frequency** — how often do you adjust the allocation? Monthly is typical
- **Tax implications** — frequent switching between strategy and cash creates taxable events

## Integration with Specific Strategies

### Ralph V3 + VOO/MA200

```python
# Recommended: 24-25% allocation
# This gives: CAGR ~13%, MDD < 10%, Sharpe ~1.8
# The remaining 75% in money market at 4%
```

### General Formula

```python
# For any strategy with known MDD:
conservative_alloc = 0.10 / abs(strategy_mdd) * 0.65  # 65% safety factor from MC
# This gives ~95% confidence that realized MDD < 10%
```
