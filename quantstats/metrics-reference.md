# QuantStats Metrics Reference

## All Available Metrics

All functions in `qs.stats` accept a pandas Series of **returns** (not prices).

### Return Metrics

```python
import quantstats as qs

# Compound Annual Growth Rate
qs.stats.cagr(returns, rf=0.0, compounded=True)

# Cumulative return
qs.stats.comp(returns)  # alias: compsum

# Total return over period
# Use: returns.sum() or qs.stats.comp(returns)

# Best/worst periods
qs.stats.best(returns, aggregate="daily")     # "daily", "weekly", "monthly", "yearly"
qs.stats.worst(returns, aggregate="daily")

# Average return
qs.stats.avg_return(returns)

# Average win/loss
qs.stats.avg_win(returns)
qs.stats.avg_loss(returns)

# Expected return (mean)
qs.stats.expected_return(returns, aggregate="daily")
```

### Risk Metrics

```python
# Volatility (annualized std)
qs.stats.volatility(returns, periods=252, annualize=True)

# Maximum drawdown
qs.stats.max_drawdown(returns)

# Value at Risk (parametric)
qs.stats.value_at_risk(returns, sigma=1, confidence=0.95)

# Conditional Value at Risk (Expected Shortfall)
qs.stats.conditional_value_at_risk(returns, sigma=1, confidence=0.95)

# Tail ratio
qs.stats.tail_ratio(returns)

# Skewness & Kurtosis
qs.stats.skew(returns)
qs.stats.kurtosis(returns)

# Payoff ratio (avg win / avg loss)
qs.stats.payoff_ratio(returns)

# Profit factor
qs.stats.profit_factor(returns)

# Profit ratio
qs.stats.profit_ratio(returns)

# Win rate
qs.stats.win_rate(returns)

# Ulcer Index
qs.stats.ulcer_index(returns)

# Ulcer Performance Index
qs.stats.ulcer_performance_index(returns, rf=0.0)
```

### Risk-Adjusted Return Ratios

```python
# Sharpe Ratio
qs.stats.sharpe(returns, rf=0.0, periods=252)

# Sortino Ratio
qs.stats.sortino(returns, rf=0.0, periods=252)

# Calmar Ratio (CAGR / max drawdown)
qs.stats.calmar(returns)

# Omega Ratio
qs.stats.omega(returns, rf=0.0, required_return=0.0)

# Gain/Loss Ratio
qs.stats.gain_to_pain_ratio(returns)

# Information Ratio (requires benchmark)
qs.stats.information_ratio(returns, benchmark)

# Treynor Ratio
qs.stats.treynor_ratio(returns, benchmark, rf=0.0)

# Risk of Ruin
qs.stats.risk_of_ruin(returns)

# Kelly Criterion
qs.stats.kelly_criterion(returns)

# Common Sense Ratio
qs.stats.common_sense_ratio(returns)

# CPC Index
qs.stats.cpc_index(returns)

# Recovery Factor
qs.stats.recovery_factor(returns)

# Risk-Return Ratio
qs.stats.risk_return_ratio(returns)
```

### Drawdown Metrics

```python
# Maximum drawdown
qs.stats.max_drawdown(returns)

# Drawdown details (returns DataFrame)
qs.stats.drawdown_details(returns)
# Columns: start, valley, end, days, max drawdown, 99% max drawdown

# Average drawdown
# Access via drawdown_details DataFrame

# Longest drawdown duration
# Access via drawdown_details DataFrame
```

### Streak & Distribution Metrics

```python
# Consecutive wins/losses
qs.stats.consecutive_wins(returns)
qs.stats.consecutive_losses(returns)

# Outlier metrics
qs.stats.outliers(returns, quantile=0.95)

# Distribution
qs.stats.skew(returns)
qs.stats.kurtosis(returns)
```

### Rolling Metrics

```python
# Rolling Sharpe
qs.stats.rolling_sharpe(returns, rf=0.0, rolling_period=126)

# Rolling Sortino
qs.stats.rolling_sortino(returns, rf=0.0, rolling_period=126)

# Rolling Volatility
qs.stats.rolling_volatility(returns, rolling_period=126)

# Rolling Beta (requires benchmark)
qs.stats.rolling_beta(returns, benchmark, rolling_period=126)

# Rolling Greeks
qs.stats.rolling_greeks(returns, benchmark, periods=252)
```

### Comparison Metrics (vs Benchmark)

```python
# Alpha & Beta
qs.stats.greeks(returns, benchmark, periods=252)
# Returns: namedtuple(alpha, beta)

# R-squared
qs.stats.r_squared(returns, benchmark)

# Information Ratio
qs.stats.information_ratio(returns, benchmark)

# Treynor Ratio
qs.stats.treynor_ratio(returns, benchmark, rf=0.0)
```

## Quick Full Summary

```python
# Get ALL metrics at once as a dictionary
qs.reports.metrics(returns, benchmark=bench, mode="full", display=True)

# Or as a DataFrame
metrics_df = qs.reports.metrics(returns, benchmark=bench, mode="full", display=False)
```

## Pandas Extension

```python
qs.extend_pandas()

# Now call metrics directly on Series
returns.sharpe()
returns.sortino()
returns.max_drawdown()
returns.cagr()
returns.calmar()
returns.volatility()
returns.win_rate()
returns.value_at_risk()
```

## Important Notes

- **All functions expect returns, not prices** — use `qs.utils.to_returns(prices)` to convert
- **`rf` parameter** is the risk-free rate as a decimal (e.g., 0.04 for 4% annual)
- **`periods` parameter** is trading days per year (252 for daily, 52 for weekly, 12 for monthly)
- **Benchmark** must be the same frequency and aligned dates as the returns Series
- **NaN handling** — most functions drop NaN automatically, but check edge cases
