---
title: Strategy Development Guide
---

# Strategy Development Guide

> **Audience**: Strategy developers who know Python but are not familiar with our system internals.
>
> You only need to write a single `.py` file, push it to a Git repository, and use the Web UI to backtest and deploy. This guide tells you everything you need to know.

---

## Table of Contents

1. [Quick Start](#1-quick-start)
2. [Architecture Overview](#2-architecture-overview)
3. [Strategy Interface (StrategyBase)](#3-strategy-interface-strategybase)
4. [Data Models](#4-data-models)
5. [Built-in Indicators Library](#5-built-in-indicators-library)
6. [Position & State Management](#6-position--state-management)
7. [Vectorized Strategy (Advanced)](#7-vectorized-strategy-advanced)
8. [How to Backtest](#8-how-to-backtest)
9. [Parameter Sweep (Optimization)](#9-parameter-sweep-optimization)
10. [Publishing Your Strategy](#10-publishing-your-strategy)
11. [Complete Examples](#11-complete-examples)
12. [Risk Engine Rules](#12-risk-engine-rules)
13. [FAQ & Common Pitfalls](#13-faq--common-pitfalls)
14. [API Quick Reference](#14-api-quick-reference)

---

## 1. Quick Start

### Minimal Strategy in 30 Lines

```python
"""My first strategy: buy when RSI < 30, sell when RSI > 70."""

from core.enums import OrderSide, OrderType
from core.models import Bar, Signal
from strategies.base import StrategyBase


class RsiMeanReversionStrategy(StrategyBase):
    """RSI mean-reversion strategy."""

    author = "your_name"
    parameters = ["rsi_period", "oversold", "overbought"]
    variables = ["current_rsi"]

    def __init__(self, symbols=None, rsi_period=14, oversold=30, overbought=70):
        super().__init__(name="rsi_mean_reversion", symbols=symbols or ["AAPL"])
        self.rsi_period = rsi_period
        self.oversold = oversold
        self.overbought = overbought
        self.current_rsi = 0.0
        self._in_position: set[str] = set()

    def on_trade(self, symbol, side, price, volume):
        if str(side) == "BUY":
            self._in_position.add(symbol)
        elif str(side) == "SELL":
            self._in_position.discard(symbol)

    def on_bar(self, bar: Bar) -> list[Signal]:
        bars = self._bars.get(bar.symbol, [])
        if len(bars) < self.rsi_period + 1:
            return []

        ind = self.indicators_for(bar.symbol)
        rsi = ind.rsi(self.rsi_period)
        self.current_rsi = float(rsi[-1])

        if self.current_rsi < self.oversold and bar.symbol not in self._in_position:
            return [Signal(
                symbol=bar.symbol,
                side=OrderSide.BUY,
                strength=0.8,
                reason=f"RSI oversold: {self.current_rsi:.1f}",
            )]

        if self.current_rsi > self.overbought and bar.symbol in self._in_position:
            return [Signal(
                symbol=bar.symbol,
                side=OrderSide.SELL,
                strength=0.7,
                reason=f"RSI overbought: {self.current_rsi:.1f}",
            )]

        return []
```

Save this as `rsi_mean_reversion.py`, push to your Git repo, and run a backtest from the Web UI. That's it.

---

## 2. Architecture Overview

```
                        Your responsibility
                    ┌─────────────────────────┐
                    │     Strategy (.py)       │
                    │  on_bar() → [Signal]     │
                    └──────────┬──────────────┘
                               │ Signal
                    ───────────┼─────────── System boundary ──
                               ▼
                    ┌──────────────────────┐
                    │   Risk Engine        │ ← Position limits, drawdown checks
                    ├──────────────────────┤
                    │   Order Matching     │ ← Slippage + commission model
                    ├──────────────────────┤
                    │   Broker Gateway     │ ← Paper trading / live execution
                    └──────────────────────┘
```

**Key principle**: Your strategy only produces `Signal` objects. It never directly places orders, accesses broker APIs, or fetches market data. The system feeds you K-line data via `on_bar()`, and you return signals. Everything else is handled automatically.

---

## 3. Strategy Interface (StrategyBase)

Every strategy must inherit from `StrategyBase` and implement `on_bar()`.

### Class Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `author` | `str` | Your name/handle |
| `parameters` | `list[str]` | Names of tunable parameters (shown in UI) |
| `variables` | `list[str]` | Names of runtime state variables (shown in UI) |
| `timeframe` | `str` | Default K-line period: `"1m"`, `"5m"`, `"15m"`, `"1h"`, `"1d"` |

### Constructor: `__init__(self, symbols, **params)`

```python
def __init__(self, symbols=None, my_param=20):
    super().__init__(name="my_strategy", symbols=symbols or ["SPY"])
    self.my_param = my_param              # Tunable parameter
    self._in_position: set[str] = set()   # Internal tracking
```

**Rules:**
- First argument must be `symbols` (list of stock tickers).
- Call `super().__init__(name=..., symbols=...)` — `name` must be unique, use `snake_case`.
- Store all tunable parameters as instance attributes.

### Required Method: `on_bar(self, bar: Bar) -> list[Signal]`

Called once for every K-line bar. This is where your logic lives.

```python
def on_bar(self, bar: Bar) -> list[Signal]:
    """Called on each new bar. Return a list of Signals (can be empty)."""
    # bar.symbol  — e.g. "AAPL"
    # bar.open / bar.high / bar.low / bar.close — Decimal
    # bar.volume  — Decimal
    # bar.timestamp — datetime

    # Access bar history for this symbol:
    bars = self._bars[bar.symbol]  # list[Bar], up to 500 most recent

    # Your logic here...
    return []  # Return [] if no signal this bar
```

### Optional Methods

| Method | When Called | Use Case |
|--------|------------|----------|
| `on_init()` | Before first bar | Pre-load indicators, warm up state |
| `on_start()` | After init, before bars | Enable trading flag |
| `on_stop()` | After last bar | Cleanup, flush state |
| `on_trade(symbol, side, price, volume)` | After an order fills | Track your position state |
| `on_tick(tick: Tick) -> list[Signal]` | On each tick (live only) | Tick-driven strategies |
| `on_order(order_id, status)` | Order status change | Track pending orders |

### `on_trade` — Position Tracking

The system does NOT give you a "current position" object. You must track positions yourself via `on_trade()`:

```python
def on_trade(self, symbol, side, price, volume):
    """Called when your signal results in a filled order."""
    if str(side) == "BUY":
        self._in_position.add(symbol)
    elif str(side) == "SELL":
        self._in_position.discard(symbol)
```

This is critical: without `on_trade()`, your strategy won't know if it's already in a position and may send duplicate buy signals.

---

## 4. Data Models

### Bar (K-line)

```python
class Bar:
    symbol: str           # "AAPL", "SPY", etc.
    timeframe: str        # "1m", "5m", "15m", "1h", "1d"
    timestamp: datetime   # Bar close time (UTC)
    open: Decimal
    high: Decimal
    low: Decimal
    close: Decimal
    volume: Decimal
    source: str = "polygon"  # Data source identifier
```

**Accessing values**: Bar fields are `Decimal` for precision. For math, cast to `float`:
```python
price = float(bar.close)
```

### Signal (Your Output)

```python
class Signal:
    symbol: str                      # Which stock
    side: OrderSide                  # OrderSide.BUY or OrderSide.SELL
    strength: float                  # 0.0 to 1.0 (signal confidence)
    order_type: OrderType = MARKET   # MARKET, LIMIT, STOP, etc.
    limit_price: Decimal | None      # Required for LIMIT orders
    quantity: Decimal | None         # Shares (None = auto-size by system)
    reason: str = ""                 # Human-readable explanation (shown in UI)
    timestamp: datetime | None       # Optional, auto-set if omitted
```

**Quantity**: If you leave `quantity=None`, the system auto-sizes the position using `position_size_ratio` (default 2% of portfolio equity). You can also specify exact shares:

```python
Signal(symbol="AAPL", side=OrderSide.BUY, strength=0.9, quantity=Decimal("100"))
```

### OrderSide / OrderType

```python
from core.enums import OrderSide, OrderType

OrderSide.BUY
OrderSide.SELL

OrderType.MARKET        # Fill at next bar's open price
OrderType.LIMIT         # Fill only if price reaches limit_price
OrderType.STOP          # Trigger when price hits stop_price
OrderType.STOP_LIMIT
```

### Tick (Live Trading Only)

```python
class Tick:
    symbol: str
    timestamp: datetime
    price: Decimal
    size: Decimal | None
    exchange: str | None             # Exchange identifier
    source: str = "polygon"          # Data source identifier
```

---

## 5. Built-in Indicators Library

Access indicators via `self.indicators_for(symbol)`. Returns an `Indicators` object computed from the symbol's bar history. Results are cached per bar count — no performance penalty for repeated calls.

```python
def on_bar(self, bar: Bar) -> list[Signal]:
    ind = self.indicators_for(bar.symbol)

    sma_20 = ind.sma(20)        # numpy array, same length as bar history
    current_sma = sma_20[-1]    # Latest value (float or NaN)
```

### Available Indicators

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `sma` | `sma(period)` | `np.ndarray` | Simple Moving Average |
| `ema` | `ema(period)` | `np.ndarray` | Exponential Moving Average |
| `rsi` | `rsi(period=14)` | `np.ndarray` | Relative Strength Index (0-100) |
| `macd` | `macd(fast=12, slow=26, signal=9)` | `(macd, signal, hist)` | MACD — returns 3 arrays |
| `atr` | `atr(period=14)` | `np.ndarray` | Average True Range |
| `bbands` | `bbands(period=20, nbdev=2.0)` | `(upper, mid, lower)` | Bollinger Bands — returns 3 arrays |
| `obv` | `obv()` | `np.ndarray` | On-Balance Volume |
| `volume_sma` | `volume_sma(period)` | `np.ndarray` | Volume Simple Moving Average |

**All indicators return numpy arrays** with the same length as the bar history. Early values are `NaN` when there isn't enough data for the lookback period. Always check for NaN:

```python
import numpy as np

rsi = ind.rsi(14)
if np.isnan(rsi[-1]):
    return []  # Not enough data yet
```

### ATR Shortcut

The base class also provides a standalone ATR helper:

```python
atr = self._calculate_atr(bars, period=14)  # Returns a single float
```

### Using Raw Bar Data

You can always compute your own indicators from the bar history:

```python
bars = self._bars[bar.symbol]  # list[Bar], up to 500 bars

closes = [float(b.close) for b in bars]
highest_20 = max(float(b.high) for b in bars[-20:])
avg_volume = sum(float(b.volume) for b in bars[-20:]) / 20

# VWAP calculation
recent = bars[-20:]
total_vol = sum(float(b.volume) for b in recent)
if total_vol > 0:
    vwap = sum(float(b.close) * float(b.volume) for b in recent) / total_vol
```

### Third-Party Libraries

The backtest environment has these pre-installed:

| Library | Version | Use Case |
|---------|---------|----------|
| `numpy` | latest | Numerical computing |
| `pandas` | latest | DataFrames, time series |
| `talib` | 0.4.x | 150+ technical indicators (C library, fast) |
| `scipy` | latest | Statistical functions |
| `scikit-learn` | latest | Machine learning |

You can import them directly in your strategy:

```python
import numpy as np
import pandas as pd
import talib

def on_bar(self, bar):
    closes = np.array([float(b.close) for b in self._bars[bar.symbol]])
    upper, mid, lower = talib.BBANDS(closes, timeperiod=20)
    stoch_k, stoch_d = talib.STOCH(
        np.array([float(b.high) for b in self._bars[bar.symbol]]),
        np.array([float(b.low) for b in self._bars[bar.symbol]]),
        closes,
    )
    ...
```

---

## 6. Position & State Management

### Bar History

The system automatically maintains bar history per symbol:

```python
self._bars[symbol]  # list[Bar], max 500 bars, oldest first
```

You never need to store bars manually. The system calls `feed_bar()` which appends to `self._bars` and then calls your `on_bar()`.

### Tracking Open Positions

**You are responsible for tracking your own positions.** The simplest pattern:

```python
def __init__(self, symbols=None):
    super().__init__(name="my_strategy", symbols=symbols or ["AAPL"])
    self._in_position: set[str] = set()

def on_trade(self, symbol, side, price, volume):
    if str(side) == "BUY":
        self._in_position.add(symbol)
    elif str(side) == "SELL":
        self._in_position.discard(symbol)

def on_bar(self, bar):
    if bar.symbol in self._in_position:
        # Already holding — check for exit
        ...
    else:
        # Not holding — check for entry
        ...
```

For more precise tracking (quantity, average cost):

```python
def __init__(self, symbols=None):
    super().__init__(name="my_strategy", symbols=symbols or ["AAPL"])
    self._positions: dict[str, dict] = {}  # {symbol: {"qty": float, "avg_cost": float}}

def on_trade(self, symbol, side, price, volume):
    price, volume = float(price), float(volume)
    pos = self._positions.get(symbol, {"qty": 0, "avg_cost": 0})

    if str(side) == "BUY":
        total_cost = pos["avg_cost"] * pos["qty"] + price * volume
        pos["qty"] += volume
        pos["avg_cost"] = total_cost / pos["qty"] if pos["qty"] > 0 else 0
    elif str(side) == "SELL":
        pos["qty"] -= volume
        if pos["qty"] <= 0:
            pos = {"qty": 0, "avg_cost": 0}

    self._positions[symbol] = pos
```

---

## 7. Vectorized Strategy (Advanced)

For parameter optimization, you can also write a **vectorized** version that operates on entire price series using pandas/numpy. This runs 100-1000x faster than bar-by-bar iteration.

```python
from strategies.vectorized import VectorizedStrategy, sma, ema, rsi, atr, bollinger_bands

class MyStrategyVectorized(VectorizedStrategy):
    name = "my_strategy"  # Must match your StrategyBase name
    param_schema = {
        "fast_period": (5, 50),      # (min, max) — for UI display
        "slow_period": (20, 200),
        "rsi_threshold": (20, 40),
    }

    def generate_signals(self, prices, volumes, **params):
        """
        Args:
            prices: pd.DataFrame — close prices (index=datetime, columns=symbols)
            volumes: pd.DataFrame — volume (same shape)
            **params: parameter values from the sweep grid

        Returns:
            (entries, exits) — boolean DataFrames, same shape as prices
        """
        fast = sma(prices, params["fast_period"])
        slow = sma(prices, params["slow_period"])
        r = rsi(prices, 14)

        entries = (fast > slow) & (r < params["rsi_threshold"])
        exits = fast < slow

        return entries.fillna(False), exits.fillna(False)
```

### Vectorized Indicator Helpers

These are pandas-native versions, available by importing from `strategies.vectorized`:

| Function | Signature | Description |
|----------|-----------|-------------|
| `sma(series, period)` | `pd.DataFrame → pd.DataFrame` | Simple Moving Average |
| `ema(series, period)` | `pd.DataFrame → pd.DataFrame` | Exponential Moving Average |
| `rsi(series, period)` | `pd.DataFrame → pd.DataFrame` | RSI (0-100) |
| `atr(high, low, close, period)` | `pd.DataFrame → pd.DataFrame` | Average True Range |
| `bollinger_bands(series, period, std_dev)` | `→ (upper, mid, lower)` | Bollinger Bands |

---

## 8. How to Backtest

### Method 1: Web UI (Recommended)

1. Go to **Backtest** page in the Web UI.
2. Select your strategy (or enter Git repo URL for external strategies).
3. Configure:
   - **Symbols**: e.g. `AAPL, MSFT, SPY`
   - **Date range**: e.g. `2023-01-01` to `2024-01-01`
   - **Timeframe**: `5m` (recommended for day-trading strategies)
   - **Initial cash**: default `$100,000`
   - **Parameters**: override strategy defaults
4. Click **Run Backtest**.
5. Results show: equity curve chart, trade list, performance metrics.

For **external strategies** (your own Git repo):
- Enter your repo URL (HTTPS or SSH)
- Optionally specify branch, commit, and file path
- The system will clone your repo, load the strategy, and run the backtest

### Method 2: REST API

```bash
# Single backtest
curl -X POST https://your-server/api/backtest \
  -H "Content-Type: application/json" \
  -d '{
    "strategy": "rsi_mean_reversion",
    "symbols": ["AAPL", "MSFT"],
    "start": "2023-01-01",
    "end": "2024-01-01",
    "timeframe": "5m",
    "initial_cash": 100000,
    "params": {"rsi_period": 14, "oversold": 25},
    "git_repo": "https://github.com/your-name/strategies.git",
    "git_branch": "main",
    "strategy_path": "rsi_mean_reversion.py"
  }'

# Response: {"task_id": "a1b2c3d4"}

# Poll for results
curl https://your-server/api/backtest/a1b2c3d4
```

### Method 3: WebSocket (Real-time Progress)

```javascript
const ws = new WebSocket("wss://your-server/ws/backtest/a1b2c3d4");
ws.onmessage = (e) => {
    const data = JSON.parse(e.data);
    console.log(`Progress: ${data.progress}%, Status: ${data.status}`);
    if (data.status === "done") {
        console.log("Report:", data.report);
    }
};
```

### Backtest Report Metrics

| Metric | Description |
|--------|-------------|
| **Total Return** | `(final_equity - initial_cash) / initial_cash` |
| **Annualized Return** | `(1 + total_return)^(252/trading_days) - 1` |
| **Max Drawdown** | Largest peak-to-trough decline |
| **Sharpe Ratio** | Risk-adjusted return: `mean(daily_return) / std(daily_return) * sqrt(252)` |
| **Win Rate** | % of closed trades with positive PnL |
| **Profit/Loss Ratio** | Average win / Average loss |
| **Total Trades** | Number of completed round-trip trades |
| **Avg Holding Period** | Average days per position |
| **Max Daily Loss** | Worst single-day return |
| **Alpha** | Excess return vs SPY benchmark (annualized) |
| **Beta** | Correlation with SPY benchmark |

### Backtest Matching Rules

| Rule | Detail |
|------|--------|
| **Market orders** | Filled at next bar's `open` price |
| **Limit orders** | Filled if next bar's `high/low` reaches the limit price |
| **Slippage** | Default 2 bps (0.02%) per trade |
| **Commission** | Default $0.005 per share |
| **Position sizing** | 2% of portfolio equity per signal (when `quantity=None`) |

### Data Sources (Automatic Fallback)

1. **PostgreSQL** — stored historical bars (fastest)
2. **Massive.com S3** — cloud flat files
3. **Demo data** — synthetic random walk (for testing only)

The system tries each source in order and uses whatever is available.

---

## 9. Parameter Sweep (Optimization)

### Vectorized Sweep (Fast)

For strategies with a `VectorizedStrategy` counterpart. Uses numpy — 100-1000x faster.

```bash
curl -X POST https://your-server/api/backtest/sweep \
  -H "Content-Type: application/json" \
  -d '{
    "strategy": "momentum_breakout",
    "symbols": ["SPY", "QQQ"],
    "start": "2023-01-01",
    "end": "2024-01-01",
    "timeframe": "5m",
    "param_grid": {
      "breakout_window": [10, 15, 20, 30],
      "volume_multiplier": [1.0, 1.5, 2.0],
      "short_ma_period": [5, 10, 20]
    }
  }'
```

**Limits**: max 500 parameter combinations, max 3-year date range.

### Parallel Sweep (Complex Strategies)

For strategies that can't be vectorized (stateful, ML-based, multi-factor). Runs bar-by-bar backtests across CPU cores using multiprocessing.

```bash
curl -X POST https://your-server/api/backtest/sweep/parallel \
  -H "Content-Type: application/json" \
  -d '{
    "strategy": "rsi_mean_reversion",
    "symbols": ["AAPL"],
    "start": "2023-01-01",
    "end": "2024-01-01",
    "timeframe": "5m",
    "param_grid": {
      "rsi_period": [10, 14, 20],
      "oversold": [20, 25, 30],
      "overbought": [70, 75, 80]
    },
    "git_repo": "https://github.com/your-name/strategies.git",
    "strategy_path": "rsi_mean_reversion.py"
  }'
```

### Sweep Response

```json
{
  "total_combinations": 36,
  "elapsed_seconds": 12.5,
  "results": [
    {
      "params": {"breakout_window": 20, "volume_multiplier": 1.5, "short_ma_period": 10},
      "total_return": 0.1523,
      "sharpe_ratio": 1.85,
      "max_drawdown": -0.0412,
      "win_rate": 0.62,
      "total_trades": 48
    },
    ...
  ],
  "best": {
    "params": {"breakout_window": 20, "volume_multiplier": 1.5, "short_ma_period": 10},
    "sharpe_ratio": 1.85
  }
}
```

---

## 10. Publishing Your Strategy

### Repository Structure

```
your-strategies-repo/
├── my_strategy.py              ← Strategy file (any name)
├── multi_factor_alpha.py       ← Another strategy
└── README.md                   ← Optional
```

Or organized in a `strategies/` subdirectory:

```
your-strategies-repo/
├── strategies/
│   ├── my_strategy.py
│   └── another_strategy.py
└── README.md
```

### Naming Convention

- **File name**: `snake_case.py` (e.g. `rsi_mean_reversion.py`)
- **Class name**: `PascalCaseStrategy` (e.g. `RsiMeanReversionStrategy`)
- **Strategy name** (in `__init__`): `snake_case` matching the file (e.g. `"rsi_mean_reversion"`)

The system auto-discovers strategies by scanning for `StrategyBase` subclasses. One class per file.

### Push to Git

```bash
git add rsi_mean_reversion.py
git commit -m "Add RSI mean reversion strategy"
git push origin main
```

### Use in Web UI

In the Backtest page:
1. Set **Git Repository** to `https://github.com/your-name/strategies.git`
2. Set **Branch** to `main` (or any branch/tag)
3. Set **Strategy File** to `rsi_mean_reversion.py` (relative path)
4. Or leave Strategy File empty — the system will scan `strategies/` directory for your class

### Dependencies

Your strategy can use any library pre-installed in the backtest environment:

```
numpy, pandas, scipy, scikit-learn, talib, pydantic, statsmodels
```

If you need additional packages, contact the system administrator to add them to the base image.

---

## 11. Complete Examples

### Example 1: Dual Moving Average Crossover

Classic trend-following strategy: buy when fast MA crosses above slow MA, sell on reverse cross.

```python
"""Dual Moving Average Crossover Strategy."""

from core.enums import OrderSide
from core.models import Bar, Signal
from strategies.base import StrategyBase


class DualMaCrossoverStrategy(StrategyBase):
    """Buy on golden cross (fast MA > slow MA), sell on death cross."""

    author = "example"
    parameters = ["fast_period", "slow_period"]
    variables = ["fast_ma", "slow_ma"]
    timeframe = "1d"

    def __init__(self, symbols=None, fast_period=10, slow_period=30):
        super().__init__(name="dual_ma_crossover", symbols=symbols or ["SPY"])
        self.fast_period = fast_period
        self.slow_period = slow_period
        self.fast_ma = 0.0
        self.slow_ma = 0.0
        self._in_position: set[str] = set()
        self._prev_fast_above: dict[str, bool | None] = {}

    def on_trade(self, symbol, side, price, volume):
        if str(side) == "BUY":
            self._in_position.add(symbol)
        elif str(side) == "SELL":
            self._in_position.discard(symbol)

    def on_bar(self, bar: Bar) -> list[Signal]:
        bars = self._bars.get(bar.symbol, [])
        if len(bars) < self.slow_period:
            return []

        ind = self.indicators_for(bar.symbol)
        fast = ind.sma(self.fast_period)
        slow = ind.sma(self.slow_period)

        self.fast_ma = float(fast[-1])
        self.slow_ma = float(slow[-1])

        fast_above = self.fast_ma > self.slow_ma
        prev = self._prev_fast_above.get(bar.symbol)
        self._prev_fast_above[bar.symbol] = fast_above

        if prev is None:
            return []

        # Golden cross: fast crosses above slow
        if fast_above and not prev and bar.symbol not in self._in_position:
            return [Signal(
                symbol=bar.symbol,
                side=OrderSide.BUY,
                strength=0.7,
                reason=f"Golden cross: SMA{self.fast_period}={self.fast_ma:.2f} > SMA{self.slow_period}={self.slow_ma:.2f}",
            )]

        # Death cross: fast crosses below slow
        if not fast_above and prev and bar.symbol in self._in_position:
            return [Signal(
                symbol=bar.symbol,
                side=OrderSide.SELL,
                strength=0.7,
                reason=f"Death cross: SMA{self.fast_period}={self.fast_ma:.2f} < SMA{self.slow_period}={self.slow_ma:.2f}",
            )]

        return []
```

### Example 2: Bollinger Bands Mean Reversion

Buy at lower band, sell at upper band.

```python
"""Bollinger Bands Mean Reversion Strategy."""

import numpy as np

from core.enums import OrderSide, OrderType
from core.models import Bar, Signal
from strategies.base import StrategyBase


class BollingerMeanReversionStrategy(StrategyBase):
    """Buy at lower Bollinger Band, sell at upper band."""

    author = "example"
    parameters = ["bb_period", "bb_std", "rsi_filter"]
    timeframe = "15m"

    def __init__(self, symbols=None, bb_period=20, bb_std=2.0, rsi_filter=True):
        super().__init__(
            name="bollinger_mean_reversion",
            symbols=symbols or ["AAPL", "MSFT", "GOOGL"],
        )
        self.bb_period = bb_period
        self.bb_std = bb_std
        self.rsi_filter = rsi_filter
        self._in_position: set[str] = set()

    def on_trade(self, symbol, side, price, volume):
        if str(side) == "BUY":
            self._in_position.add(symbol)
        elif str(side) == "SELL":
            self._in_position.discard(symbol)

    def on_bar(self, bar: Bar) -> list[Signal]:
        bars = self._bars.get(bar.symbol, [])
        if len(bars) < self.bb_period + 5:
            return []

        ind = self.indicators_for(bar.symbol)
        upper, mid, lower = ind.bbands(self.bb_period, self.bb_std)

        price = float(bar.close)
        upper_val = float(upper[-1])
        lower_val = float(lower[-1])

        if np.isnan(upper_val) or np.isnan(lower_val):
            return []

        # Optional RSI filter: only buy when RSI < 40
        if self.rsi_filter:
            rsi = ind.rsi(14)
            rsi_val = float(rsi[-1])
            if np.isnan(rsi_val):
                return []
        else:
            rsi_val = 50

        # Buy: price touches lower band + RSI filter
        if price <= lower_val and bar.symbol not in self._in_position:
            if not self.rsi_filter or rsi_val < 40:
                return [Signal(
                    symbol=bar.symbol,
                    side=OrderSide.BUY,
                    strength=0.8,
                    reason=f"Price {price:.2f} at lower BB {lower_val:.2f}, RSI={rsi_val:.0f}",
                )]

        # Sell: price reaches upper band
        if price >= upper_val and bar.symbol in self._in_position:
            return [Signal(
                symbol=bar.symbol,
                side=OrderSide.SELL,
                strength=0.7,
                reason=f"Price {price:.2f} at upper BB {upper_val:.2f}",
            )]

        return []
```

### Example 3: Multi-Timeframe Regime + Stock Selection

More complex strategy: use SPY regime detection to decide offense/defense, then pick individual stocks.

```python
"""Multi-Factor Regime Rotation Strategy.

Uses SPY as a regime indicator:
- Offensive regime (uptrend): buy high-momentum tech stocks
- Defensive regime (downtrend): rotate to bonds/gold

Only trades on regime transitions to minimize churn.
"""

from decimal import Decimal

from core.enums import OrderSide
from core.models import Bar, Signal
from strategies.base import StrategyBase


OFFENSIVE = ["QQQ", "NVDA", "META", "AMZN"]
DEFENSIVE = ["TLT", "GLD"]
ALL_SYMBOLS = ["SPY"] + OFFENSIVE + DEFENSIVE


class RegimeRotationStrategy(StrategyBase):
    """Regime-aware rotation between growth and safe-haven assets."""

    author = "example"
    parameters = ["fast_ma", "slow_ma", "atr_period", "vol_threshold"]
    variables = ["regime"]
    timeframe = "1d"

    def __init__(
        self,
        symbols=None,
        fast_ma=20,
        slow_ma=50,
        atr_period=14,
        vol_threshold=1.1,
    ):
        super().__init__(name="regime_rotation", symbols=symbols or ALL_SYMBOLS)
        self.fast_ma = fast_ma
        self.slow_ma = slow_ma
        self.atr_period = atr_period
        self.vol_threshold = vol_threshold
        self.regime = "neutral"
        self._holdings: set[str] = set()

    def on_trade(self, symbol, side, price, volume):
        if str(side) == "BUY":
            self._holdings.add(symbol)
        elif str(side) == "SELL":
            self._holdings.discard(symbol)

    def on_bar(self, bar: Bar) -> list[Signal]:
        # Only evaluate regime on SPY bars
        if bar.symbol != "SPY":
            return []

        spy_bars = self._bars.get("SPY", [])
        if len(spy_bars) < self.slow_ma + self.atr_period:
            return []

        # Detect regime
        ind = self.indicators_for("SPY")
        fast = float(ind.sma(self.fast_ma)[-1])
        slow = float(ind.sma(self.slow_ma)[-1])

        if fast > slow:
            new_regime = "offensive"
        else:
            atr_now = float(ind.atr(self.atr_period)[-1])
            atr_prev = float(ind.atr(self.atr_period)[-self.atr_period])
            if atr_prev > 0 and atr_now > atr_prev * self.vol_threshold:
                new_regime = "defensive"
            else:
                new_regime = "neutral"

        if new_regime == self.regime:
            return []

        # Regime changed — generate rotation signals
        signals = []
        old_regime = self.regime
        self.regime = new_regime

        if new_regime == "offensive":
            # Sell defensive, buy offensive
            for s in DEFENSIVE:
                if s in self._holdings:
                    signals.append(Signal(symbol=s, side=OrderSide.SELL, strength=0.8,
                                         reason=f"Regime → offensive: exit {s}"))
            for s in OFFENSIVE:
                if s not in self._holdings:
                    signals.append(Signal(symbol=s, side=OrderSide.BUY, strength=0.8,
                                         reason=f"Regime → offensive: enter {s}"))

        elif new_regime == "defensive":
            # Sell offensive, buy defensive
            for s in OFFENSIVE:
                if s in self._holdings:
                    signals.append(Signal(symbol=s, side=OrderSide.SELL, strength=0.8,
                                         reason=f"Regime → defensive: exit {s}"))
            for s in DEFENSIVE:
                if s not in self._holdings:
                    signals.append(Signal(symbol=s, side=OrderSide.BUY, strength=0.8,
                                         reason=f"Regime → defensive: enter {s}"))

        else:  # neutral
            # Close all
            for s in list(self._holdings):
                signals.append(Signal(symbol=s, side=OrderSide.SELL, strength=0.6,
                                     reason=f"Regime → neutral: close {s}"))

        return signals
```

### Example 4: Vectorized Strategy with Parameter Sweep

```python
"""Vectorized version of Bollinger Bands strategy for fast parameter sweep."""

import pandas as pd

from strategies.vectorized import VectorizedStrategy, bollinger_bands, rsi


class BollingerMeanReversionVectorized(VectorizedStrategy):
    name = "bollinger_mean_reversion"
    param_schema = {
        "bb_period": (10, 40),
        "bb_std": (1.5, 3.0),
    }

    def generate_signals(self, prices, volumes, **params):
        period = params.get("bb_period", 20)
        std = params.get("bb_std", 2.0)

        upper, mid, lower = bollinger_bands(prices, period, std)
        r = rsi(prices, 14)

        entries = (prices <= lower) & (r < 40)
        exits = prices >= upper

        return entries.fillna(False), exits.fillna(False)
```

Sweep this strategy:

```bash
curl -X POST https://your-server/api/backtest/sweep \
  -d '{
    "strategy": "bollinger_mean_reversion",
    "symbols": ["AAPL", "MSFT"],
    "start": "2023-01-01",
    "end": "2024-01-01",
    "param_grid": {
      "bb_period": [10, 15, 20, 25, 30],
      "bb_std": [1.5, 2.0, 2.5, 3.0]
    }
  }'
```

---

## 12. Risk Engine Rules

All signals pass through the Risk Engine before execution. The following rules are checked automatically:

| Rule | Default | Description |
|------|---------|-------------|
| Max single order value | $20,000 | Single order cannot exceed this dollar amount |
| Max symbol weight | 20% | Single stock cannot exceed 20% of portfolio |
| Max total equity weight | 80% | Total invested cannot exceed 80% of portfolio |
| Min cash ratio | 10% | Must keep at least 10% in cash |
| Allow margin | No | Margin trading disabled |
| Allow short | No | Short selling disabled |
| Max total loss | 3% | Daily loss limit (resets daily) |
| Max drawdown | 8% | Strategy halted if drawdown exceeds this |
| Price deviation | 1% | Order rejected if price moved >1% since signal |
| Duplicate order window | 60s | Same symbol+side+quantity blocked within 60 seconds |
| Trading hours | US market hours | Orders only during 9:30-16:00 ET (backtest ignores this) |

**For backtesting**: trading hours check is disabled. All other rules apply.

If the Risk Engine rejects your signal, it's silently dropped (logged as `debug`). If your strategy produces signals but you see zero trades in the backtest report, the Risk Engine may be blocking them. Common causes:
- Signal quantity is too large (exceeds max order value)
- Too many signals for the same symbol (duplicate order window)
- Portfolio already at max equity weight

---

## 13. FAQ & Common Pitfalls

### Q: My strategy runs but produces zero trades. Why?

**A**: Most common causes:
1. **Not enough bar history** — your `on_bar()` returns `[]` because the `if len(bars) < N` guard hasn't been met. Check that your backtest date range is long enough.
2. **Risk Engine blocking** — your signals are rejected. Try increasing `initial_cash` or reducing position size.
3. **Missing `on_trade()`** — without tracking positions, your strategy keeps sending BUY signals which the Risk Engine blocks as duplicate positions.

### Q: Can I access multiple symbols' data in one `on_bar()` call?

**A**: Yes! `self._bars` contains history for ALL symbols your strategy is subscribed to:

```python
def on_bar(self, bar: Bar) -> list[Signal]:
    # bar is the current symbol's new bar
    spy_bars = self._bars.get("SPY", [])  # Access any symbol's history
    aapl_bars = self._bars.get("AAPL", [])

    # Calculate cross-symbol spread
    if spy_bars and aapl_bars:
        spy_price = float(spy_bars[-1].close)
        aapl_price = float(aapl_bars[-1].close)
        ratio = aapl_price / spy_price
```

### Q: How is `strength` used?

**A**: The `strength` field (0.0-1.0) currently serves as metadata/documentation. It does not affect order sizing or priority. Future risk engine versions may use it for dynamic sizing.

### Q: Can I use limit orders?

**A**: Yes:

```python
Signal(
    symbol="AAPL",
    side=OrderSide.BUY,
    strength=0.8,
    order_type=OrderType.LIMIT,
    limit_price=Decimal("150.00"),  # Only fill at $150 or better
    reason="Limit buy at support level",
)
```

In backtesting, limit orders fill if the next bar's high/low reaches the limit price.

### Q: What's the maximum bar history available?

**A**: 500 bars per symbol (configurable via `_MAX_BAR_HISTORY`). For a 5-minute timeframe, that's about 6 trading days. For daily bars, that's ~2 years.

### Q: Can I store state between bars?

**A**: Yes, use instance attributes:

```python
def __init__(self, symbols=None):
    super().__init__(name="my_strategy", symbols=symbols or ["SPY"])
    self._last_signal_time = {}    # Track when we last signaled
    self._entry_prices = {}        # Track entry prices
    self._trade_count = 0          # Count trades
```

### Q: How do I test locally before pushing?

**A**: You can run a backtest directly from the API without a Git repo. Or test your strategy class in a Python script:

```python
from strategies.base import StrategyBase
from core.models import Bar, Signal
from datetime import datetime
from decimal import Decimal

# Import your strategy
from my_strategy import MyStrategy

strategy = MyStrategy(symbols=["AAPL"])
strategy.on_init()
strategy.on_start()

# Create a test bar
bar = Bar(
    symbol="AAPL", timeframe="5m",
    timestamp=datetime.now(),
    open=Decimal("150"), high=Decimal("151"),
    low=Decimal("149"), close=Decimal("150.5"),
    volume=Decimal("1000000"),
)

signals = strategy.feed_bar(bar)
print(f"Signals: {signals}")
```

---

## 14. API Quick Reference

### StrategyBase (you inherit from this)

```python
class StrategyBase(ABC):
    # --- Override these ---
    author: str = ""
    parameters: list[str] = []
    variables: list[str] = []
    timeframe: str = "1m"

    def __init__(self, name: str, symbols: list[str]): ...

    @abstractmethod
    def on_bar(self, bar: Bar) -> list[Signal]: ...

    # --- Optional overrides ---
    def on_init(self) -> None: ...
    def on_start(self) -> None: ...
    def on_stop(self) -> None: ...
    def on_trade(self, symbol, side, price, volume) -> None: ...
    def on_tick(self, tick: Tick) -> list[Signal]: ...
    def on_order(self, order_id, status) -> None: ...

    # --- Provided by base class ---
    self._bars: dict[str, list[Bar]]                        # Bar history per symbol
    def indicators_for(self, symbol: str) -> Indicators: ...  # Technical indicators
    @staticmethod
    def _calculate_atr(bars, period) -> float: ...           # ATR shortcut
    def get_parameters(self) -> dict: ...                    # Serialize params
    def get_variables(self) -> dict: ...                     # Serialize state
```

### Signal

```python
Signal(
    symbol="AAPL",          # Required
    side=OrderSide.BUY,     # Required: BUY or SELL
    strength=0.8,           # Required: 0.0 - 1.0
    order_type=OrderType.MARKET,  # Optional, default MARKET
    limit_price=None,       # Required for LIMIT orders
    quantity=None,           # Optional, None = auto-size
    reason="",              # Optional, shown in UI
)
```

### Indicators

```python
ind = self.indicators_for("AAPL")

ind.sma(period) -> np.ndarray
ind.ema(period) -> np.ndarray
ind.rsi(period=14) -> np.ndarray
ind.macd(fast=12, slow=26, signal=9) -> (macd, signal, hist)
ind.atr(period=14) -> np.ndarray
ind.bbands(period=20, nbdev=2.0) -> (upper, mid, lower)
ind.obv() -> np.ndarray
ind.volume_sma(period) -> np.ndarray
```

### Backtest API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/backtest` | Start a backtest (returns `task_id`) |
| `GET` | `/api/backtest/{task_id}` | Get status & results |
| `GET` | `/api/backtest` | List all backtests |
| `POST` | `/api/backtest/sweep` | Vectorized parameter sweep |
| `POST` | `/api/backtest/sweep/parallel` | Bar-by-bar parameter sweep |
| `POST` | `/api/backtest/cache/clear` | Clear backtest data cache |
| `WS` | `/ws/backtest/{task_id}` | Real-time progress stream |

---

## Checklist: Before You Publish

- [ ] Strategy inherits from `StrategyBase`
- [ ] `__init__` calls `super().__init__(name=..., symbols=...)`
- [ ] `on_bar()` is implemented and returns `list[Signal]`
- [ ] `on_trade()` is implemented for position tracking
- [ ] Parameters are stored as instance attributes
- [ ] `parameters` class attribute lists all tunable params
- [ ] Tested with at least one successful backtest
- [ ] No hard-coded file paths or network calls
- [ ] No side effects (don't print to stdout, don't write files)

---

## 15. AI-Generated Strategies

The AI Quant Assistant can write, validate, backtest, and promote strategies through natural language conversation.

### How It Works

1. **Describe** your strategy idea to the AI assistant (entry/exit logic, symbols, timeframe)
2. The AI calls `get_strategy_guide` to load templates and indicator API reference
3. The AI writes a `StrategyBase` subclass and calls `create_strategy` to validate and save it
4. You ask the AI to backtest it — it calls `run_backtest` with the AI strategy ID
5. Iterate on parameters or logic based on backtest results
6. When satisfied, ask the AI to `promote_strategy` — it copies to production after re-validation

### Validation Pipeline

AI-generated code goes through multi-layer validation before execution:

| Layer | Check | Blocks |
|-------|-------|--------|
| AST Import | Static analysis of import statements | `os`, `subprocess`, `sys`, `socket`, `importlib`, etc. |
| AST Builtins | Static analysis of function calls | `exec()`, `eval()`, `__import__()`, `open()`, `compile()` |
| Restricted Exec | Limited `__builtins__` at module load | Runtime bypass attempts |
| Subclass Check | Must be a `StrategyBase` subclass | Arbitrary code masquerading as strategy |
| Instantiation | Must instantiate with `symbols=["TEST"]` | Broken constructors |

### Restrictions for AI Strategies

Your AI-generated strategy code **cannot**:

- Import blocked modules (`os`, `subprocess`, `sys`, `shutil`, `socket`, `http`, `requests`, `urllib`, `pathlib`, `ctypes`, `multiprocessing`, `threading`, `signal`, `importlib`, etc.)
- Call blocked builtins (`exec`, `eval`, `compile`, `__import__`, `open`, `breakpoint`, `input`)
- Perform network I/O or file system operations
- Use threading or multiprocessing

Your strategy code **can**:

- Import from `strategies.base`, `core.models`, `core.enums`
- Use all built-in indicators via `self.indicators_for(symbol)`
- Use standard library math, statistics, collections, dataclasses
- Track state via instance attributes

### Storage

```
ai_strategies/
  {user_id}/
    {strategy_name}.py    # Source code
```

Database table `ai_strategies` stores metadata, status (`draft` → `tested` → `promoted`), and up to 10 backtest results per strategy.

---

*Last updated: 2026-06-01*
