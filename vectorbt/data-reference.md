# VectorBT Data Reference

## Data Downloading

### YFinance Data

```python
import vectorbt as vbt

# Single symbol
data = vbt.YFData.download("AAPL", start="2020-01-01", end="2024-01-01")

# Multiple symbols
data = vbt.YFData.download(["AAPL", "GOOGL", "MSFT"], start="2020-01-01")

# Access OHLCV columns
close = data.get("Close")    # DataFrame (multi-symbol) or Series (single)
open_ = data.get("Open")
high = data.get("High")
low = data.get("Low")
volume = data.get("Volume")

# With specific interval
data = vbt.YFData.download("AAPL", start="2023-01-01", interval="1h")
# Intervals: 1m, 2m, 5m, 15m, 30m, 60m, 90m, 1h, 1d, 5d, 1wk, 1mo, 3mo

# With period instead of date range
data = vbt.YFData.download("AAPL", period="1y")
# Periods: 1d, 5d, 1mo, 3mo, 6mo, 1y, 2y, 5y, 10y, ytd, max
```

### Binance Data

```python
# Crypto data from Binance
data = vbt.BinanceData.download(
    "BTCUSDT",
    start="2022-01-01",
    end="2024-01-01",
    interval="1d"
)
# Intervals: 1m, 3m, 5m, 15m, 30m, 1h, 2h, 4h, 6h, 8h, 12h, 1d, 3d, 1w, 1M

close = data.get("Close")
```

### Using Your Own Data

```python
import pandas as pd

# From CSV
df = pd.read_csv("data.csv", index_col=0, parse_dates=True)
close = df["Close"]

# From FinLab (台股)
from dotenv import load_dotenv
load_dotenv()
from finlab import data as finlab_data
close = finlab_data.get("price:收盤價")

# VectorBT works directly with pandas Series/DataFrame
pf = vbt.Portfolio.from_signals(close, entries, exits)
```

### Data Caching

```python
# Save downloaded data locally
data = vbt.YFData.download("AAPL", start="2020-01-01")
data.to_hdf("aapl_data.h5")

# Load cached data
data = vbt.YFData.from_hdf("aapl_data.h5")
```

## Data Utilities

```python
# Resample OHLCV data
close_weekly = close.vbt.resample_apply("W", vbt.nb.last_reduce_nb)

# Split data for walk-forward analysis
(in_price, in_indexes), (out_price, out_indexes) = close.vbt.rolling_split(
    n=5,           # number of splits
    window_len=252, # in-sample window
    set_lens=(126,), # out-of-sample window
    left_to_right=False
)

# Generate date ranges
index = vbt.date_range("2020-01-01", "2024-01-01", freq="B")  # business days
```

## Important Notes

- **Timezone handling**: YFinance returns timezone-aware data. If mixing with other sources, align timezones or use `.tz_localize(None)`.
- **Missing data**: VectorBT handles NaN values in most operations, but check for gaps in intraday data.
- **Multi-symbol DataFrames**: When downloading multiple symbols, `data.get("Close")` returns a DataFrame with symbols as columns. All VectorBT operations broadcast correctly across columns.
