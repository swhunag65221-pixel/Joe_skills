# VectorBT Portfolio Reference

## Portfolio.from_signals()

The most common way to create a portfolio — from boolean entry/exit signals.

```python
pf = vbt.Portfolio.from_signals(
    close,                    # Price series (Series or DataFrame)
    entries,                  # Boolean Series/DataFrame — True = enter long
    exits,                    # Boolean Series/DataFrame — True = exit long
    short_entries=None,       # Boolean — True = enter short
    short_exits=None,         # Boolean — True = exit short
    init_cash=100000,         # Initial capital
    size=np.inf,              # Order size (np.inf = use all available cash)
    size_type="amount",       # "amount", "value", "percent"
    fees=0.001,               # Commission rate (0.1%)
    slippage=0.001,           # Slippage rate (0.1%)
    freq="1D",                # Data frequency
    direction="both",         # "longonly", "shortonly", "both"
    accumulate=False,         # Allow multiple entries before exit
    upon_opposite_entry="close",  # "close", "closereduce", "reverse"
    sl_stop=None,             # Stop-loss (e.g., 0.05 for 5%)
    tp_stop=None,             # Take-profit (e.g., 0.10 for 10%)
    ts_stop=None,             # Trailing stop (e.g., 0.03 for 3%)
    delta_format="percent",   # "percent" or "absolute" for stops
)
```

### Size Types

```python
# Fixed number of shares
pf = vbt.Portfolio.from_signals(close, entries, exits, size=100, size_type="amount")

# Fixed dollar value
pf = vbt.Portfolio.from_signals(close, entries, exits, size=10000, size_type="value")

# Percent of portfolio
pf = vbt.Portfolio.from_signals(close, entries, exits, size=0.5, size_type="percent")

# All available cash (default)
pf = vbt.Portfolio.from_signals(close, entries, exits, size=np.inf)
```

### Stop-Loss & Take-Profit

```python
# 5% stop-loss, 10% take-profit
pf = vbt.Portfolio.from_signals(
    close, entries, exits,
    sl_stop=0.05,
    tp_stop=0.10,
)

# Trailing stop of 3%
pf = vbt.Portfolio.from_signals(
    close, entries, exits,
    ts_stop=0.03,
)
```

## Portfolio.from_orders()

Lower-level API — specify exact order sizes at each timestep.

```python
pf = vbt.Portfolio.from_orders(
    close,                    # Price series
    size,                     # Order size at each step (positive=buy, negative=sell)
    size_type="amount",       # "amount", "value", "targetamount", "targetvalue", "targetpercent"
    init_cash=100000,
    fees=0.001,
    slippage=0.001,
    freq="1D",
)
```

### Target-Based Ordering

```python
# Rebalance to target percentages
target_pct = pd.DataFrame(...)  # Target allocation at each step
pf = vbt.Portfolio.from_orders(
    close, target_pct,
    size_type="targetpercent",
    init_cash=100000,
)
```

## Portfolio.from_holding()

Simplest — buy and hold from start to end.

```python
pf = vbt.Portfolio.from_holding(close, init_cash=100000, fees=0.001)
```

## Portfolio.from_random_signals()

Generate random entry/exit signals for Monte Carlo analysis.

```python
pf = vbt.Portfolio.from_random_signals(
    close,
    n=10,           # Number of entry signals
    seed=42,        # Random seed for reproducibility
    init_cash=100000,
)
```

## Portfolio Statistics & Metrics

### Summary Stats

```python
# Full stats summary (returns pd.Series)
print(pf.stats())

# Key individual metrics
pf.total_return()        # Total return (decimal)
pf.annual_return()       # Annualized return
pf.sharpe_ratio()        # Annualized Sharpe ratio
pf.sortino_ratio()       # Sortino ratio
pf.max_drawdown()        # Maximum drawdown (decimal)
pf.calmar_ratio()        # Calmar ratio
pf.omega_ratio()         # Omega ratio
pf.total_profit()        # Absolute profit in currency
pf.total_trades()        # Number of trades
pf.win_rate()            # Win rate (decimal)
pf.profit_factor()       # Gross profit / gross loss
pf.expectancy()          # Average profit per trade
pf.value()               # Portfolio value series
pf.returns()             # Portfolio returns series
pf.cumulative_returns()  # Cumulative returns series
pf.drawdown()            # Drawdown series
pf.max_drawdown()        # Maximum drawdown
pf.underwater()          # Underwater equity curve
```

### Trade Analysis

```python
# Access individual trades
trades = pf.trades
trades.records_readable  # DataFrame of all trades
trades.pnl.sum()         # Total P&L
trades.return_.mean()    # Average trade return
trades.duration.mean()   # Average trade duration
trades.win_rate()        # Win rate
trades.count()           # Number of trades

# Entry & exit trades separately
pf.entry_trades.records_readable
pf.exit_trades.records_readable
```

### Orders & Positions

```python
# Order records
pf.orders.records_readable

# Position records
pf.positions.records_readable

# Cash & value time series
pf.cash()           # Cash series
pf.value()          # Total portfolio value series
pf.asset_value()    # Asset value (excluding cash)
```

## Plotting

```python
# Full portfolio plot (equity curve, drawdown, trades)
fig = pf.plot()
fig.show()

# Individual plots
pf.plot_cum_returns().show()
pf.plot_drawdowns().show()
pf.plot_underwater().show()
pf.plot_trades().show()

# Save plot
fig = pf.plot()
fig.write_image("portfolio.png")
fig.write_html("portfolio.html")
```

## Multi-Column Portfolios

When using parameter arrays (broadcasting), Portfolio returns multi-column results:

```python
# Multiple parameter combinations
fast = [10, 20, 30]
slow = [50, 100]

# ... generate signals for all combos ...
pf = vbt.Portfolio.from_signals(close, entries, exits)

# Results for all combinations
print(pf.total_return())    # Series with multi-index
print(pf.sharpe_ratio())    # One value per combination

# Best combination
best_idx = pf.sharpe_ratio().idxmax()
print(f"Best params: {best_idx}")
print(pf.stats(column=best_idx))

# Plot best
pf[best_idx].plot().show()
```

## Important Notes

- `size=np.inf` means use all available cash — this is the default for from_signals
- Fees and slippage are specified as fractions (0.001 = 0.1%), not percentages
- `init_cash=100000` is the default; always specify for reproducibility
- Stop-loss/take-profit values are fractions (0.05 = 5%)
- `direction="longonly"` prevents short selling; use "both" for long/short strategies
