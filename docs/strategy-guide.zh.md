---
title: 策略开发指南
---

# 策略开发指南

> **读者**：了解 Python 语法，但不熟悉本系统内部实现的策略开发者。
>
> 你只需编写一个 `.py` 文件，推送到 Git 仓库，然后通过 Web UI 回测和部署。本指南将告诉你一切所需信息。

---

## 目录

1. [快速开始](#1-快速开始)
2. [架构概览](#2-架构概览)
3. [策略接口 (StrategyBase)](#3-策略接口-strategybase)
4. [数据模型](#4-数据模型)
5. [内置指标库](#5-内置指标库)
6. [仓位与状态管理](#6-仓位与状态管理)
7. [向量化策略（进阶）](#7-向量化策略进阶)
8. [如何回测](#8-如何回测)
9. [参数扫描（优化）](#9-参数扫描优化)
10. [发布策略](#10-发布策略)
11. [完整示例](#11-完整示例)
12. [风控引擎规则](#12-风控引擎规则)
13. [常见问题与陷阱](#13-常见问题与陷阱)
14. [API 快速参考](#14-api-快速参考)

---

## 1. 快速开始

### 30 行最小策略

```python
"""我的第一个策略：RSI < 30 买入，RSI > 70 卖出。"""

from core.enums import OrderSide, OrderType
from core.models import Bar, Signal
from strategies.base import StrategyBase


class RsiMeanReversionStrategy(StrategyBase):
    """RSI 均值回归策略。"""

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
                reason=f"RSI 超卖: {self.current_rsi:.1f}",
            )]

        if self.current_rsi > self.overbought and bar.symbol in self._in_position:
            return [Signal(
                symbol=bar.symbol,
                side=OrderSide.SELL,
                strength=0.7,
                reason=f"RSI 超买: {self.current_rsi:.1f}",
            )]

        return []
```

保存为 `rsi_mean_reversion.py`，推送到你的 Git 仓库，然后在 Web UI 中运行回测。就这么简单。

---

## 2. 架构概览

```
                        你的职责
                    ┌─────────────────────────┐
                    │     策略 (.py)           │
                    │  on_bar() → [Signal]     │
                    └──────────┬──────────────┘
                               │ Signal
                    ───────────┼─────────── 系统边界 ──
                               ▼
                    ┌──────────────────────┐
                    │   风控引擎            │ ← 仓位限制、回撤检查
                    ├──────────────────────┤
                    │   订单撮合            │ ← 滑点 + 手续费模型
                    ├──────────────────────┤
                    │   券商网关            │ ← 模拟交易 / 实盘执行
                    └──────────────────────┘
```

**核心原则**：你的策略只产生 `Signal` 对象。它不直接下单、不访问券商 API、不获取行情数据。系统通过 `on_bar()` 向你推送 K 线数据，你返回信号。其余一切由系统自动处理。

---

## 3. 策略接口 (StrategyBase)

每个策略必须继承 `StrategyBase` 并实现 `on_bar()`。

### 类属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `author` | `str` | 你的名字/昵称 |
| `parameters` | `list[str]` | 可调参数名称（在 UI 中显示） |
| `variables` | `list[str]` | 运行时状态变量名称（在 UI 中显示） |
| `timeframe` | `str` | 默认 K 线周期：`"1m"`, `"5m"`, `"15m"`, `"1h"`, `"1d"` |

### 构造函数：`__init__(self, symbols, **params)`

```python
def __init__(self, symbols=None, my_param=20):
    super().__init__(name="my_strategy", symbols=symbols or ["SPY"])
    self.my_param = my_param              # 可调参数
    self._in_position: set[str] = set()   # 内部状态跟踪
```

**规则：**
- 第一个参数必须是 `symbols`（股票代码列表）。
- 调用 `super().__init__(name=..., symbols=...)` — `name` 必须唯一，使用 `snake_case`。
- 所有可调参数存储为实例属性。

### 必须实现的方法：`on_bar(self, bar: Bar) -> list[Signal]`

每根 K 线调用一次。你的逻辑写在这里。

```python
def on_bar(self, bar: Bar) -> list[Signal]:
    """每根新 K 线调用。返回 Signal 列表（可以为空）。"""
    # bar.symbol  — 例如 "AAPL"
    # bar.open / bar.high / bar.low / bar.close — Decimal
    # bar.volume  — Decimal
    # bar.timestamp — datetime

    # 访问该股票的 K 线历史：
    bars = self._bars[bar.symbol]  # list[Bar]，最多 500 根最新 K 线

    # 你的逻辑...
    return []  # 本根 K 线无信号则返回 []
```

### 可选方法

| 方法 | 调用时机 | 用途 |
|------|----------|------|
| `on_init()` | 第一根 K 线之前 | 预加载指标，预热状态 |
| `on_start()` | 初始化后，K 线之前 | 启用交易标志 |
| `on_stop()` | 最后一根 K 线之后 | 清理，保存状态 |
| `on_trade(symbol, side, price, volume)` | 订单成交后 | 跟踪持仓状态 |
| `on_tick(tick: Tick) -> list[Signal]` | 每个 tick（仅实盘） | tick 驱动策略 |
| `on_order(order_id, status)` | 订单状态变更 | 跟踪挂单 |

### `on_trade` — 仓位跟踪

系统不会给你一个"当前持仓"对象。你必须通过 `on_trade()` 自己跟踪持仓：

```python
def on_trade(self, symbol, side, price, volume):
    """当你的信号产生成交时调用。"""
    if str(side) == "BUY":
        self._in_position.add(symbol)
    elif str(side) == "SELL":
        self._in_position.discard(symbol)
```

这非常关键：没有 `on_trade()`，你的策略不知道自己是否已经持仓，可能发送重复买入信号。

---

## 4. 数据模型

### Bar（K 线）

```python
class Bar:
    symbol: str           # "AAPL", "SPY" 等
    timeframe: str        # "1m", "5m", "15m", "1h", "1d"
    timestamp: datetime   # K 线收盘时间 (UTC)
    open: Decimal
    high: Decimal
    low: Decimal
    close: Decimal
    volume: Decimal
    source: str = "polygon"  # 数据源标识
```

**访问数值**：Bar 字段是 `Decimal` 精确类型。做数学运算时转为 `float`：
```python
price = float(bar.close)
```

### Signal（你的输出）

```python
class Signal:
    symbol: str                      # 哪只股票
    side: OrderSide                  # OrderSide.BUY 或 OrderSide.SELL
    strength: float                  # 0.0 到 1.0（信号置信度）
    order_type: OrderType = MARKET   # MARKET, LIMIT, STOP 等
    limit_price: Decimal | None      # LIMIT 订单必填
    quantity: Decimal | None         # 股数（None = 系统自动计算仓位）
    reason: str = ""                 # 可读说明（在 UI 中显示）
    timestamp: datetime | None       # 可选，未设置时自动填充
```

**数量**：如果 `quantity=None`，系统使用 `position_size_ratio`（默认为组合权益的 2%）自动计算仓位。你也可以指定精确股数：

```python
Signal(symbol="AAPL", side=OrderSide.BUY, strength=0.9, quantity=Decimal("100"))
```

### OrderSide / OrderType

```python
from core.enums import OrderSide, OrderType

OrderSide.BUY
OrderSide.SELL

OrderType.MARKET        # 以下一根 K 线的开盘价成交
OrderType.LIMIT         # 仅当价格达到 limit_price 时成交
OrderType.STOP          # 当价格触及 stop_price 时触发
OrderType.STOP_LIMIT
```

### Tick（仅实盘）

```python
class Tick:
    symbol: str
    timestamp: datetime
    price: Decimal
    size: Decimal | None
    exchange: str | None             # 交易所标识
    source: str = "polygon"          # 数据源标识
```

---

## 5. 内置指标库

通过 `self.indicators_for(symbol)` 访问指标。返回一个 `Indicators` 对象，从该股票的 K 线历史计算。结果按 K 线数量缓存——重复调用无性能损失。

```python
def on_bar(self, bar: Bar) -> list[Signal]:
    ind = self.indicators_for(bar.symbol)

    sma_20 = ind.sma(20)        # numpy 数组，长度与 K 线历史相同
    current_sma = sma_20[-1]    # 最新值（float 或 NaN）
```

### 可用指标

| 方法 | 签名 | 返回值 | 说明 |
|------|------|--------|------|
| `sma` | `sma(period)` | `np.ndarray` | 简单移动平均 |
| `ema` | `ema(period)` | `np.ndarray` | 指数移动平均 |
| `rsi` | `rsi(period=14)` | `np.ndarray` | 相对强弱指数 (0-100) |
| `macd` | `macd(fast=12, slow=26, signal=9)` | `(macd, signal, hist)` | MACD — 返回 3 个数组 |
| `atr` | `atr(period=14)` | `np.ndarray` | 平均真实波幅 |
| `bbands` | `bbands(period=20, nbdev=2.0)` | `(upper, mid, lower)` | 布林带 — 返回 3 个数组 |
| `obv` | `obv()` | `np.ndarray` | 能量潮 |
| `volume_sma` | `volume_sma(period)` | `np.ndarray` | 成交量简单移动平均 |

**所有指标返回 numpy 数组**，长度与 K 线历史相同。回溯期不够时前面的值为 `NaN`。务必检查：

```python
import numpy as np

rsi = ind.rsi(14)
if np.isnan(rsi[-1]):
    return []  # 数据不够
```

### ATR 快捷方法

基类还提供了一个独立的 ATR 辅助函数：

```python
atr = self._calculate_atr(bars, period=14)  # 返回单个 float
```

### 使用原始 K 线数据

你随时可以从 K 线历史自行计算指标：

```python
bars = self._bars[bar.symbol]  # list[Bar]，最多 500 根

closes = [float(b.close) for b in bars]
highest_20 = max(float(b.high) for b in bars[-20:])
avg_volume = sum(float(b.volume) for b in bars[-20:]) / 20

# VWAP 计算
recent = bars[-20:]
total_vol = sum(float(b.volume) for b in recent)
if total_vol > 0:
    vwap = sum(float(b.close) * float(b.volume) for b in recent) / total_vol
```

### 第三方库

回测环境预装了以下库：

| 库 | 版本 | 用途 |
|---|------|------|
| `numpy` | 最新 | 数值计算 |
| `pandas` | 最新 | DataFrame，时间序列 |
| `talib` | 0.4.x | 150+ 技术指标（C 库，高性能） |
| `scipy` | 最新 | 统计函数 |
| `scikit-learn` | 最新 | 机器学习 |

你可以在策略中直接导入：

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

## 6. 仓位与状态管理

### K 线历史

系统自动维护每个股票的 K 线历史：

```python
self._bars[symbol]  # list[Bar]，最多 500 根，最旧在前
```

你无需手动存储 K 线。系统调用 `feed_bar()` 追加到 `self._bars`，然后调用你的 `on_bar()`。

### 跟踪持仓

**你负责跟踪自己的持仓。** 最简单的模式：

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
        # 已持仓 — 检查退出条件
        ...
    else:
        # 未持仓 — 检查入场条件
        ...
```

更精确的跟踪（数量、均价）：

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

## 7. 向量化策略（进阶）

对于参数优化，你可以编写一个**向量化**版本，使用 pandas/numpy 操作整个价格序列。运行速度比逐 bar 迭代快 100-1000 倍。

```python
from strategies.vectorized import VectorizedStrategy, sma, ema, rsi, atr, bollinger_bands

class MyStrategyVectorized(VectorizedStrategy):
    name = "my_strategy"  # 必须与你的 StrategyBase 名称匹配
    param_schema = {
        "fast_period": (5, 50),      # (最小值, 最大值) — UI 中显示
        "slow_period": (20, 200),
        "rsi_threshold": (20, 40),
    }

    def generate_signals(self, prices, volumes, **params):
        """
        参数：
            prices: pd.DataFrame — 收盘价（index=datetime, columns=symbols）
            volumes: pd.DataFrame — 成交量（同形状）
            **params: 参数扫描网格中的参数值

        返回：
            (entries, exits) — 布尔 DataFrame，与 prices 同形状
        """
        fast = sma(prices, params["fast_period"])
        slow = sma(prices, params["slow_period"])
        r = rsi(prices, 14)

        entries = (fast > slow) & (r < params["rsi_threshold"])
        exits = fast < slow

        return entries.fillna(False), exits.fillna(False)
```

### 向量化指标辅助函数

这些是 pandas 原生版本，从 `strategies.vectorized` 导入：

| 函数 | 签名 | 说明 |
|------|------|------|
| `sma(series, period)` | `pd.DataFrame → pd.DataFrame` | 简单移动平均 |
| `ema(series, period)` | `pd.DataFrame → pd.DataFrame` | 指数移动平均 |
| `rsi(series, period)` | `pd.DataFrame → pd.DataFrame` | RSI (0-100) |
| `atr(high, low, close, period)` | `pd.DataFrame → pd.DataFrame` | 平均真实波幅 |
| `bollinger_bands(series, period, std_dev)` | `→ (upper, mid, lower)` | 布林带 |

---

## 8. 如何回测

### 方法 1：Web UI（推荐）

1. 进入 Web UI 的 **回测** 页面。
2. 选择策略（或输入 Git 仓库 URL 加载外部策略）。
3. 配置：
   - **股票代码**：如 `AAPL, MSFT, SPY`
   - **日期范围**：如 `2023-01-01` 到 `2024-01-01`
   - **K 线周期**：`5m`（推荐日内策略使用）
   - **初始资金**：默认 `$100,000`
   - **参数**：覆盖策略默认值
4. 点击 **运行回测**。
5. 结果展示：权益曲线图、交易列表、绩效指标。

**外部策略**（你自己的 Git 仓库）：
- 输入仓库 URL（HTTPS 或 SSH）
- 可选指定分支、commit、文件路径
- 系统会克隆仓库、加载策略并运行回测

### 方法 2：REST API

```bash
# 单次回测
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

# 响应: {"task_id": "a1b2c3d4"}

# 轮询结果
curl https://your-server/api/backtest/a1b2c3d4
```

### 方法 3：WebSocket（实时进度）

```javascript
const ws = new WebSocket("wss://your-server/ws/backtest/a1b2c3d4");
ws.onmessage = (e) => {
    const data = JSON.parse(e.data);
    console.log(`进度: ${data.progress}%, 状态: ${data.status}`);
    if (data.status === "done") {
        console.log("报告:", data.report);
    }
};
```

### 回测报告指标

| 指标 | 说明 |
|------|------|
| **总收益率** | `(最终权益 - 初始资金) / 初始资金` |
| **年化收益率** | `(1 + 总收益率)^(252/交易天数) - 1` |
| **最大回撤** | 最大峰值到谷底跌幅 |
| **夏普比率** | 风险调整收益：`mean(日收益率) / std(日收益率) * sqrt(252)` |
| **胜率** | 正收益平仓交易的百分比 |
| **盈亏比** | 平均盈利 / 平均亏损 |
| **总交易数** | 完成的往返交易次数 |
| **平均持仓时间** | 每笔持仓的平均天数 |
| **最大单日亏损** | 最差单日收益率 |
| **Alpha** | 相对 SPY 基准的超额年化收益 |
| **Beta** | 与 SPY 基准的相关性 |

### 回测撮合规则

| 规则 | 详情 |
|------|------|
| **市价单** | 以下一根 K 线的 `open` 价成交 |
| **限价单** | 下一根 K 线的 `high/low` 达到限价时成交 |
| **滑点** | 默认 2 个基点 (0.02%) |
| **手续费** | 默认每股 $0.005 |
| **仓位大小** | 组合权益的 2%（当 `quantity=None` 时） |

### 数据来源（自动回退）

1. **PostgreSQL** — 已存储的历史 K 线（最快）
2. **Massive.com S3** — 云端平面文件
3. **Demo 数据** — 合成随机游走（仅测试用）

系统按顺序尝试每个数据源，使用可用的那个。

---

## 9. 参数扫描（优化）

### 向量化扫描（快速）

适用于有 `VectorizedStrategy` 对应版本的策略。使用 numpy — 比逐 bar 快 100-1000 倍。

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

**限制**：最多 500 个参数组合，最长 3 年日期范围。

### 并行扫描（复杂策略）

适用于无法向量化的策略（有状态、基于 ML、多因子）。使用多进程在 CPU 核心上并行运行逐 bar 回测。

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

### 扫描响应

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
    }
  ],
  "best": {
    "params": {"breakout_window": 20, "volume_multiplier": 1.5, "short_ma_period": 10},
    "sharpe_ratio": 1.85
  }
}
```

---

## 10. 发布策略

### 仓库结构

```
your-strategies-repo/
├── my_strategy.py              ← 策略文件（任意文件名）
├── multi_factor_alpha.py       ← 另一个策略
└── README.md                   ← 可选
```

或者放在 `strategies/` 子目录中：

```
your-strategies-repo/
├── strategies/
│   ├── my_strategy.py
│   └── another_strategy.py
└── README.md
```

### 命名规范

- **文件名**：`snake_case.py`（如 `rsi_mean_reversion.py`）
- **类名**：`PascalCaseStrategy`（如 `RsiMeanReversionStrategy`）
- **策略名**（`__init__` 中）：`snake_case`，与文件名对应（如 `"rsi_mean_reversion"`）

系统通过扫描 `StrategyBase` 子类自动发现策略。每个文件一个类。

### 推送到 Git

```bash
git add rsi_mean_reversion.py
git commit -m "Add RSI mean reversion strategy"
git push origin main
```

### 在 Web UI 中使用

在回测页面：
1. 设置 **Git 仓库** 为 `https://github.com/your-name/strategies.git`
2. 设置 **分支** 为 `main`（或任意分支/标签）
3. 设置 **策略文件** 为 `rsi_mean_reversion.py`（相对路径）
4. 或留空策略文件 — 系统会扫描 `strategies/` 目录查找你的类

### 依赖

你的策略可以使用回测环境中预装的任何库：

```
numpy, pandas, scipy, scikit-learn, talib, pydantic, statsmodels
```

如需额外包，联系系统管理员将其添加到基础镜像中。

---

## 11. 完整示例

### 示例 1：双均线交叉策略

经典趋势跟踪策略：快速均线上穿慢速均线时买入，下穿时卖出。

```python
"""双均线交叉策略。"""

from core.enums import OrderSide
from core.models import Bar, Signal
from strategies.base import StrategyBase


class DualMaCrossoverStrategy(StrategyBase):
    """金叉买入（快 MA > 慢 MA），死叉卖出。"""

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

        # 金叉：快线上穿慢线
        if fast_above and not prev and bar.symbol not in self._in_position:
            return [Signal(
                symbol=bar.symbol,
                side=OrderSide.BUY,
                strength=0.7,
                reason=f"金叉: SMA{self.fast_period}={self.fast_ma:.2f} > SMA{self.slow_period}={self.slow_ma:.2f}",
            )]

        # 死叉：快线下穿慢线
        if not fast_above and prev and bar.symbol in self._in_position:
            return [Signal(
                symbol=bar.symbol,
                side=OrderSide.SELL,
                strength=0.7,
                reason=f"死叉: SMA{self.fast_period}={self.fast_ma:.2f} < SMA{self.slow_period}={self.slow_ma:.2f}",
            )]

        return []
```

### 示例 2：布林带均值回归

下轨买入，上轨卖出。

```python
"""布林带均值回归策略。"""

import numpy as np

from core.enums import OrderSide, OrderType
from core.models import Bar, Signal
from strategies.base import StrategyBase


class BollingerMeanReversionStrategy(StrategyBase):
    """在布林带下轨买入，上轨卖出。"""

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

        # 可选 RSI 过滤：仅当 RSI < 40 时买入
        if self.rsi_filter:
            rsi = ind.rsi(14)
            rsi_val = float(rsi[-1])
            if np.isnan(rsi_val):
                return []
        else:
            rsi_val = 50

        # 买入：价格触及下轨 + RSI 过滤
        if price <= lower_val and bar.symbol not in self._in_position:
            if not self.rsi_filter or rsi_val < 40:
                return [Signal(
                    symbol=bar.symbol,
                    side=OrderSide.BUY,
                    strength=0.8,
                    reason=f"价格 {price:.2f} 触及下轨 {lower_val:.2f}, RSI={rsi_val:.0f}",
                )]

        # 卖出：价格达到上轨
        if price >= upper_val and bar.symbol in self._in_position:
            return [Signal(
                symbol=bar.symbol,
                side=OrderSide.SELL,
                strength=0.7,
                reason=f"价格 {price:.2f} 达到上轨 {upper_val:.2f}",
            )]

        return []
```

### 示例 3：多时间框架 Regime + 选股

更复杂的策略：用 SPY regime 检测决定攻防，然后选择个股。

```python
"""多因子 Regime 轮动策略。

用 SPY 作为 regime 指标：
- 进攻 regime（上升趋势）：买入高动量科技股
- 防御 regime（下降趋势）：轮动到债券/黄金

仅在 regime 切换时交易以减少换手。
"""

from decimal import Decimal

from core.enums import OrderSide
from core.models import Bar, Signal
from strategies.base import StrategyBase


OFFENSIVE = ["QQQ", "NVDA", "META", "AMZN"]
DEFENSIVE = ["TLT", "GLD"]
ALL_SYMBOLS = ["SPY"] + OFFENSIVE + DEFENSIVE


class RegimeRotationStrategy(StrategyBase):
    """Regime 感知的成长与避险资产轮动。"""

    author = "example"
    parameters = ["fast_ma", "slow_ma", "atr_period", "vol_threshold"]
    variables = ["regime"]
    timeframe = "1d"

    def __init__(self, symbols=None, fast_ma=20, slow_ma=50, atr_period=14, vol_threshold=1.1):
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
        # 仅在 SPY K 线上评估 regime
        if bar.symbol != "SPY":
            return []

        spy_bars = self._bars.get("SPY", [])
        if len(spy_bars) < self.slow_ma + self.atr_period:
            return []

        # 检测 regime
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

        # Regime 变化 — 生成轮动信号
        signals = []
        self.regime = new_regime

        if new_regime == "offensive":
            for s in DEFENSIVE:
                if s in self._holdings:
                    signals.append(Signal(symbol=s, side=OrderSide.SELL, strength=0.8,
                                         reason=f"Regime → 进攻: 退出 {s}"))
            for s in OFFENSIVE:
                if s not in self._holdings:
                    signals.append(Signal(symbol=s, side=OrderSide.BUY, strength=0.8,
                                         reason=f"Regime → 进攻: 买入 {s}"))

        elif new_regime == "defensive":
            for s in OFFENSIVE:
                if s in self._holdings:
                    signals.append(Signal(symbol=s, side=OrderSide.SELL, strength=0.8,
                                         reason=f"Regime → 防御: 退出 {s}"))
            for s in DEFENSIVE:
                if s not in self._holdings:
                    signals.append(Signal(symbol=s, side=OrderSide.BUY, strength=0.8,
                                         reason=f"Regime → 防御: 买入 {s}"))

        else:  # neutral
            for s in list(self._holdings):
                signals.append(Signal(symbol=s, side=OrderSide.SELL, strength=0.6,
                                     reason=f"Regime → 中性: 平仓 {s}"))

        return signals
```

### 示例 4：向量化策略 + 参数扫描

```python
"""布林带策略的向量化版本，用于快速参数扫描。"""

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

使用此策略进行参数扫描：

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

## 12. 风控引擎规则

所有信号在执行前通过风控引擎。以下规则自动检查：

| 规则 | 默认值 | 说明 |
|------|--------|------|
| 最大单笔金额 | $20,000 | 单笔订单不得超过此金额 |
| 单票最大仓位 | 20% | 单只股票不得超过组合的 20% |
| 总股票仓位上限 | 80% | 总投资不得超过组合的 80% |
| 最小现金比例 | 10% | 至少保留 10% 现金 |
| 允许融资 | 否 | 禁止杠杆交易 |
| 允许做空 | 否 | 禁止卖空 |
| 最大总亏损 | 3% | 日亏损限制（每日重置） |
| 最大回撤 | 8% | 回撤超过此值则暂停策略 |
| 价格偏离 | 1% | 信号发出后价格变动 >1% 则拒绝 |
| 重复下单窗口 | 60 秒 | 同一股票+方向+数量在 60 秒内被阻止 |
| 交易时间 | 美股交易时间 | 仅在 9:30-16:00 ET 下单（回测忽略此规则） |

**回测时**：交易时间检查被禁用。其他所有规则生效。

如果风控引擎拒绝你的信号，它会被静默丢弃（记录为 `debug` 日志）。如果你的策略产生了信号但回测报告中交易为零，可能是风控引擎阻止了它们。常见原因：
- 信号数量太大（超过最大单笔金额）
- 同一股票信号太多（重复下单窗口）
- 组合已达到最大股票仓位上限

---

## 13. 常见问题与陷阱

### 问：策略运行了但产生零笔交易，为什么？

**答**：最常见的原因：
1. **K 线历史不够** — 你的 `on_bar()` 因为 `if len(bars) < N` 的判断而返回 `[]`。确保回测日期范围足够长。
2. **风控引擎阻止** — 你的信号被拒绝。尝试增加 `initial_cash` 或减少仓位大小。
3. **缺少 `on_trade()`** — 不跟踪持仓时，策略持续发送 BUY 信号，被风控引擎作为重复持仓阻止。

### 问：能在一次 `on_bar()` 调用中访问多只股票的数据吗？

**答**：可以！`self._bars` 包含你策略订阅的所有股票的历史：

```python
def on_bar(self, bar: Bar) -> list[Signal]:
    # bar 是当前股票的新 K 线
    spy_bars = self._bars.get("SPY", [])  # 访问任何股票的历史
    aapl_bars = self._bars.get("AAPL", [])

    # 计算跨股票价差
    if spy_bars and aapl_bars:
        spy_price = float(spy_bars[-1].close)
        aapl_price = float(aapl_bars[-1].close)
        ratio = aapl_price / spy_price
```

### 问：`strength` 是如何使用的？

**答**：`strength` 字段 (0.0-1.0) 目前仅作为元数据/文档。它不影响订单大小或优先级。未来版本的风控引擎可能用它做动态仓位管理。

### 问：能用限价单吗？

**答**：可以：

```python
Signal(
    symbol="AAPL",
    side=OrderSide.BUY,
    strength=0.8,
    order_type=OrderType.LIMIT,
    limit_price=Decimal("150.00"),  # 仅在 $150 或更低价成交
    reason="在支撑位限价买入",
)
```

回测中，限价单在下一根 K 线的 high/low 达到限价时成交。

### 问：最大可用 K 线历史是多少？

**答**：每只股票 500 根 K 线（可通过 `_MAX_BAR_HISTORY` 配置）。5 分钟周期约为 6 个交易日。日线约为 2 年。

### 问：能在 K 线之间保存状态吗？

**答**：可以，使用实例属性：

```python
def __init__(self, symbols=None):
    super().__init__(name="my_strategy", symbols=symbols or ["SPY"])
    self._last_signal_time = {}    # 跟踪上次发信号的时间
    self._entry_prices = {}        # 跟踪入场价格
    self._trade_count = 0          # 计数交易次数
```

### 问：推送之前如何本地测试？

**答**：你可以直接通过 API 运行回测而不需要 Git 仓库。或者在 Python 脚本中测试你的策略类：

```python
from strategies.base import StrategyBase
from core.models import Bar, Signal
from datetime import datetime
from decimal import Decimal

# 导入你的策略
from my_strategy import MyStrategy

strategy = MyStrategy(symbols=["AAPL"])
strategy.on_init()
strategy.on_start()

# 创建测试 K 线
bar = Bar(
    symbol="AAPL", timeframe="5m",
    timestamp=datetime.now(),
    open=Decimal("150"), high=Decimal("151"),
    low=Decimal("149"), close=Decimal("150.5"),
    volume=Decimal("1000000"),
)

signals = strategy.feed_bar(bar)
print(f"信号: {signals}")
```

---

## 14. API 快速参考

### StrategyBase（你继承的基类）

```python
class StrategyBase(ABC):
    # --- 需要覆盖 ---
    author: str = ""
    parameters: list[str] = []
    variables: list[str] = []
    timeframe: str = "1m"

    def __init__(self, name: str, symbols: list[str]): ...

    @abstractmethod
    def on_bar(self, bar: Bar) -> list[Signal]: ...

    # --- 可选覆盖 ---
    def on_init(self) -> None: ...
    def on_start(self) -> None: ...
    def on_stop(self) -> None: ...
    def on_trade(self, symbol, side, price, volume) -> None: ...
    def on_tick(self, tick: Tick) -> list[Signal]: ...
    def on_order(self, order_id, status) -> None: ...

    # --- 基类提供 ---
    self._bars: dict[str, list[Bar]]                        # 每只股票的 K 线历史
    def indicators_for(self, symbol: str) -> Indicators: ...  # 技术指标
    @staticmethod
    def _calculate_atr(bars, period) -> float: ...           # ATR 快捷方法
    def get_parameters(self) -> dict: ...                    # 序列化参数
    def get_variables(self) -> dict: ...                     # 序列化状态
```

### Signal

```python
Signal(
    symbol="AAPL",          # 必填
    side=OrderSide.BUY,     # 必填：BUY 或 SELL
    strength=0.8,           # 必填：0.0 - 1.0
    order_type=OrderType.MARKET,  # 可选，默认 MARKET
    limit_price=None,       # 限价单必填
    quantity=None,           # 可选，None = 自动计算仓位
    reason="",              # 可选，在 UI 中显示
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

### 回测 API 端点

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/api/backtest` | 启动回测（返回 `task_id`） |
| `GET` | `/api/backtest/{task_id}` | 获取状态和结果 |
| `GET` | `/api/backtest` | 列出所有回测 |
| `POST` | `/api/backtest/sweep` | 向量化参数扫描 |
| `POST` | `/api/backtest/sweep/parallel` | 逐 bar 参数扫描 |
| `POST` | `/api/backtest/cache/clear` | 清理回测数据缓存 |
| `WS` | `/ws/backtest/{task_id}` | 实时进度推送 |

---

## 发布前检查清单

- [ ] 策略继承自 `StrategyBase`
- [ ] `__init__` 调用了 `super().__init__(name=..., symbols=...)`
- [ ] `on_bar()` 已实现并返回 `list[Signal]`
- [ ] `on_trade()` 已实现用于仓位跟踪
- [ ] 参数存储为实例属性
- [ ] `parameters` 类属性列出了所有可调参数
- [ ] 至少通过了一次成功的回测
- [ ] 无硬编码文件路径或网络调用
- [ ] 无副作用（不打印到 stdout，不写文件）

---

## 15. AI 生成策略

AI 量化助手可以通过自然语言对话来编写、验证、回测和发布策略。

### 工作流程

1. 向 AI 助手**描述**你的策略想法（入场/出场逻辑、标的、时间框架）
2. AI 调用 `get_strategy_guide` 加载模板和指标 API 参考
3. AI 编写 `StrategyBase` 子类并调用 `create_strategy` 验证保存
4. 你要求 AI 回测 — 它调用 `run_backtest` 并分析结果
5. 根据回测结果迭代优化参数或逻辑
6. 满意后要求 AI `promote_strategy` — 重新验证后复制到生产目录

### 验证流程

AI 生成的代码在执行前经过多层验证：

| 层级 | 检查内容 | 拦截对象 |
|------|---------|---------|
| AST Import 检查 | 静态分析 import 语句 | `os`, `subprocess`, `sys`, `socket`, `importlib` 等 |
| AST 内置函数检查 | 静态分析函数调用 | `exec()`, `eval()`, `__import__()`, `open()`, `compile()` |
| 受限执行 | 模块加载时限制 `__builtins__` | 运行时绕过尝试 |
| 子类检查 | 必须是 `StrategyBase` 子类 | 伪装为策略的任意代码 |
| 实例化检查 | 必须能用 `symbols=["TEST"]` 实例化 | 构造函数错误 |

### AI 策略限制

AI 生成的策略代码**不能**：

- 导入被阻止的模块（`os`, `subprocess`, `sys`, `shutil`, `socket`, `http`, `requests`, `urllib`, `pathlib`, `ctypes`, `multiprocessing`, `threading`, `signal`, `importlib` 等）
- 调用被阻止的内置函数（`exec`, `eval`, `compile`, `__import__`, `open`, `breakpoint`, `input`）
- 执行网络 I/O 或文件系统操作
- 使用多线程或多进程

AI 生成的策略代码**可以**：

- 从 `strategies.base`, `core.models`, `core.enums` 导入
- 通过 `self.indicators_for(symbol)` 使用所有内置指标
- 使用标准库 math, statistics, collections, dataclasses
- 通过实例属性跟踪状态

### 存储结构

```
ai_strategies/
  {user_id}/
    {strategy_name}.py    # 源代码
```

数据库表 `ai_strategies` 存储元数据、状态（`draft` → `tested` → `promoted`）以及每个策略最多 10 条回测结果。

---

*最后更新: 2026-06-01*
