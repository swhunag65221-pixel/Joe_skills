# VectorBT Indicators Reference

## Built-in Indicators

### Moving Averages

```python
import vectorbt as vbt

# Simple Moving Average
ma = vbt.MA.run(close, window=20, ewm=False)
ma.ma          # The MA values (Series/DataFrame)

# Exponential Moving Average
ema = vbt.MA.run(close, window=20, ewm=True)

# Multiple windows at once
ma = vbt.MA.run(close, window=[10, 20, 50])
# Returns DataFrame with 3 columns

# Crossover signals
fast_ma = vbt.MA.run(close, window=10)
slow_ma = vbt.MA.run(close, window=50)
entries = fast_ma.ma_crossed_above(slow_ma)
exits = fast_ma.ma_crossed_below(slow_ma)
```

### RSI (Relative Strength Index)

```python
rsi = vbt.RSI.run(close, window=14)
rsi.rsi        # RSI values

# Overbought/oversold signals
entries = rsi.rsi_below(30)   # Buy when RSI < 30
exits = rsi.rsi_above(70)     # Sell when RSI > 70

# Multiple windows
rsi = vbt.RSI.run(close, window=[7, 14, 21])
```

### Bollinger Bands

```python
bb = vbt.BBANDS.run(close, window=20, alpha=2.0)
bb.upper       # Upper band
bb.middle      # Middle band (SMA)
bb.lower       # Lower band
bb.bandwidth   # Bandwidth
bb.percent_b   # %B indicator

# Signals
entries = bb.close_below(bb.lower)   # Buy below lower band
exits = bb.close_above(bb.upper)     # Sell above upper band
```

### MACD

```python
macd = vbt.MACD.run(close, fast_window=12, slow_window=26, signal_window=9)
macd.macd          # MACD line
macd.signal        # Signal line
macd.hist          # Histogram

# Crossover signals
entries = macd.macd_crossed_above(macd.signal)
exits = macd.macd_crossed_below(macd.signal)
```

### ATR (Average True Range)

```python
atr = vbt.ATR.run(high, low, close, window=14)
atr.atr        # ATR values
```

### Stochastic Oscillator

```python
stoch = vbt.STOCH.run(high, low, close, k_window=14, d_window=3)
stoch.percent_k   # %K line
stoch.percent_d   # %D line
```

### OBV (On-Balance Volume)

```python
obv = vbt.OBV.run(close, volume)
obv.obv        # OBV values
```

## Generic Indicators

### Crossed Above / Below

```python
# Generic crossover detection (works with any two Series)
crossed = close.vbt.crossed_above(sma_200)   # Boolean Series
crossed = close.vbt.crossed_below(sma_200)
```

### Rolling Statistics

```python
# Rolling window operations
close.vbt.rolling_mean(window=20)
close.vbt.rolling_std(window=20)
close.vbt.rolling_min(window=20)
close.vbt.rolling_max(window=20)
```

## Custom Indicators with IndicatorFactory

### Basic Custom Indicator

```python
# Define a custom indicator
MyInd = vbt.IndicatorFactory(
    class_name="MyInd",
    short_name="myind",
    input_names=["close"],
    param_names=["window"],
    output_names=["output"]
).from_apply_func(
    lambda close, window: pd.Series(close).rolling(window).mean().values,
    window=20  # default value
)

# Use it
result = MyInd.run(close, window=20)
result.output  # Access the output
```

### With Numba Acceleration

```python
from numba import njit

@njit
def custom_calc_nb(close, window):
    """Numba-compiled custom indicator."""
    out = np.empty_like(close)
    for i in range(len(close)):
        if i < window - 1:
            out[i] = np.nan
        else:
            out[i] = np.mean(close[i - window + 1:i + 1])
    return out

MyFastInd = vbt.IndicatorFactory(
    class_name="MyFastInd",
    short_name="myfastind",
    input_names=["close"],
    param_names=["window"],
    output_names=["output"]
).from_apply_func(
    custom_calc_nb,
    window=20
)
```

### Multiple Outputs

```python
def calc_bands(close, window, num_std):
    ma = pd.Series(close).rolling(window).mean()
    std = pd.Series(close).rolling(window).std()
    upper = ma + num_std * std
    lower = ma - num_std * std
    return ma.values, upper.values, lower.values

MyBands = vbt.IndicatorFactory(
    class_name="MyBands",
    short_name="mybands",
    input_names=["close"],
    param_names=["window", "num_std"],
    output_names=["middle", "upper", "lower"]
).from_apply_func(
    calc_bands,
    window=20,
    num_std=2.0
)

result = MyBands.run(close, window=20, num_std=2.0)
result.middle   # Middle band
result.upper    # Upper band
result.lower    # Lower band
```

## TA-Lib Integration

```python
# If ta-lib is installed, use it via VectorBT
# pip install TA-Lib
import talib

# Use TA-Lib functions directly, then feed results to VectorBT
rsi = talib.RSI(close.values, timeperiod=14)
macd, signal, hist = talib.MACD(close.values)
```

## Indicator Plotting

```python
# Built-in indicators have plot methods
ma = vbt.MA.run(close, window=[20, 50])
ma.plot().show()

bb = vbt.BBANDS.run(close, window=20)
bb.plot().show()

rsi = vbt.RSI.run(close, window=14)
rsi.plot().show()

macd = vbt.MACD.run(close)
macd.plot().show()
```

## Important Notes

- All built-in indicators support **parameter arrays** for simultaneous multi-param testing
- Indicator outputs are **lazy** — computed only when accessed
- Use `.values` to get raw numpy arrays from indicator outputs
- Crossover methods (e.g., `ma_crossed_above`) return boolean Series suitable for `from_signals()`
- Custom indicators via `IndicatorFactory` inherit all broadcasting and plotting capabilities
