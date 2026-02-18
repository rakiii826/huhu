# A股优质开源量化交易项目深度调研报告

本报告针对您开发量化模拟交易系统的需求，深入调研了 GitHub 上最优质、受欢迎且适合 A 股市场的 Python 量化项目。调研重点涵盖了**打板/低吸策略（技术面）**、**价值投资/AI Alpha（基本面/多因子）**以及**高性能交易/回测框架**。

以下是精选的 4 个顶级项目，它们分别代表了不同的技术流派和应用场景。

---

## 1. Microsoft Qlib (AI/价值投资/多因子流派)

**项目地址**: [https://github.com/microsoft/qlib](https://github.com/microsoft/qlib)
**核心定位**: 面向 AI 的量化投资平台，涵盖数据、模型训练、回测全流程。
**推荐理由**: 业界公认的 AI 量化标杆，特别适合**价值投资**和**多因子选股**策略。

### 核心策略与数学原理
Qlib 的核心在于通过机器学习模型（如 LightGBM, LSTM, Transformer）预测股票未来的收益率（Alpha），从而构建投资组合。
- **Alpha158 / Alpha360**: 内置了 158 个（或 360 个）经典的量化因子，包括量价因子（如 `ROC`, `RSI`, `MA`）和技术指标。
- **模型预测**: 使用机器学习算法 $f(x)$ 拟合 $y$（未来 N 日收益率）。
  $$ \hat{y} = f(Factors) $$
- **TopK Dropout 策略**:
  - **逻辑**: 每日根据模型预测的分数（Score）对全市场股票排名。
  - **买入**: 持有分数最高的 Top K 只股票（例如 Top 50）。
  - **卖出**: 每日卖出排名跌出 Top K 的股票，买入新进入 Top K 的股票。
  - **换仓控制**: 引入 `n_drop` 参数限制每日换仓数量，降低交易成本。

### 代码结构分析
- **`qlib/contrib/model/`**: 包含大量 SOTA 模型（LightGBM, GATs, TRA 等）。
- **`qlib/contrib/strategy/signal_strategy.py`**: 实现了 `TopkDropoutStrategy` 和 `EnhancedIndexingStrategy`（指数增强）。
- **`examples/benchmarks/`**: 提供了完整的训练和回测配置（YAML 文件），这是学习的最佳入口。

### A股适配性
- **涨跌停处理**: 回测引擎原生支持 A 股涨跌停限制（`limit_threshold`），无法在涨停价买入或跌停价卖出。
- **交易成本**: 支持设置印花税和佣金（`open_cost`, `close_cost`）。
- **数据**: 提供 A 股历史数据下载脚本（`scripts/get_data.py`），数据格式为高效的二进制文件。

---

## 2. MyTT (打板/低吸/技术指标流派)

**项目地址**: [https://github.com/mpquant/MyTT](https://github.com/mpquant/MyTT)
**核心定位**: 将通达信（TDX）、同花顺、文华财经等指标公式移植到 Python。
**推荐理由**: **打板**和**低吸**策略高度依赖特定的技术指标（如 MACD, KDJ, RSI, 均线金叉等）。MyTT 是实现这些策略逻辑的**数学基石**。

### 核心策略与数学原理
MyTT 并非一个完整的交易系统，而是一个单文件库（Library），它让你能在 Python 中一行代码实现复杂的指标计算。
- **MACD (指数平滑异同移动平均线)**:
  $$ DIF = EMA(Close, 12) - EMA(Close, 26) $$
  $$ DEA = EMA(DIF, 9) $$
  $$ MACD = (DIF - DEA) \times 2 $$
- **KDJ (随机指标)**:
  $$ RSV = \frac{Close - LLV(Low, 9)}{HHV(High, 9) - LLV(Low, 9)} \times 100 $$
  $$ K = EMA(RSV, 3), D = EMA(K, 3), J = 3K - 2D $$
- **打板/低吸逻辑实现**:
  - **涨停判断**: `C / REF(C, 1) > 1.095`
  - **连板天数**: `BARSLASTCOUNT(C >= REF(C, 1) * 1.095)`
  - **均线金叉**: `CROSS(MA(C, 5), MA(C, 10))`

### 代码结构分析
- **`MyTT.py`**: 核心文件，仅需复制此文件即可使用。包含了 `RD`, `RET`, `ABS`, `MA`, `EMA`, `SMA`, `MACD`, `RSI`, `BOLL` 等 100+ 个函数。
- **使用方式**:
  ```python
  from MyTT import *
  CLOSE = df.close.values
  OPEN = df.open.values
  # 计算 MACD
  DIF, DEA, MACD_VAL = MACD(CLOSE)
  # 判断是否金叉
  gold_cross = CROSS(DIF, DEA)
  ```

### A股适配性
- **完全对标**: 所有的函数实现都经过校对，与通达信/同花顺软件结果一致，非常适合习惯传统软件公式的 A 股交易者。

---

## 3. EasyQuant (实盘交易/策略执行流派)

**项目地址**: [https://github.com/shidenggui/easyquant](https://github.com/shidenggui/easyquant)
**核心定位**: 针对个人投资者的股票量化框架，支持实盘和模拟交易。
**推荐理由**: 解决了“策略写好了怎么下单”的问题。非常适合**个人开发者**构建自己的**打板/低吸自动交易系统**。

### 核心架构
- **事件驱动引擎**: 类似于 `vnpy`，但更轻量。通过 `MainEngine` 接收行情推送（Tick 数据），触发 `strategy` 函数。
- **行情源**: 支持新浪（Sina）、集思录等免费实时行情源（1秒推送）。
- **交易接口**: 内置 `easytrader`，支持华泰、银河、佣金宝等券商的实盘接口（基于 Web/客户端模拟），以及雪球组合模拟盘。

### 策略实现逻辑
用户只需继承 `StrategyTemplate` 并实现 `strategy` 方法：
```python
class MyStrategy(StrategyTemplate):
    def strategy(self, event):
        # event.data 包含最新的行情快照 (ask/bid/price)
        stock_code = '000001'
        price = event.data[stock_code]['now']

        # 结合 MyTT 实现逻辑:
        # 1. 维护一个历史数据列表 history_data
        # 2. 将最新 price 加入 history_data
        # 3. 使用 MyTT 计算指标
        # 4. 下单
        if condition_met:
            self.user.buy(stock_code, price=price, amount=100)
```

### A股适配性
- **实盘友好**: 专为 A 股散户设计，解决了获取实时行情和自动下单的痛点。
- **即插即用**: 配置好券商账号（如 `ht.json`）即可开始交易。

---

## 4. WonderTrader (高性能/专业机构流派)

**项目地址**: [https://github.com/wondertrader/wtpy](https://github.com/wondertrader/wtpy)
**核心定位**: C++ 核心的高性能量化交易平台，提供 Python 接口（`wtpy`）。
**推荐理由**: 如果您追求**回测速度**（C++核心）和**系统稳定性**，或者未来考虑高频/多资产套利，这是一个比 EasyQuant 更专业的选择。

### 核心策略与数学原理
采用经典的 CTA（Commodity Trading Advisor）策略架构，但同样适用于股票。
- **Dual Thrust 策略 (示例)**:
  - 经典的区间突破策略。
  - 计算前 N 日的 `Range = Max(HH-LC, HC-LL)`。
  - 上轨 = `Open + K1 * Range`，下轨 = `Open - K2 * Range`。
  - 突破上轨做多，突破下轨做空（股票则平仓）。
- **执行逻辑**: `on_calculate` (K线闭合时触发) 或 `on_tick` (Tick 到来时触发)。

### 代码结构分析
- **`wtpy/`**: Python 适配层。
- **`demos/cta_stk_bt/`**: A 股 CTA 策略回测的示例。
- **`demos/Strategies/`**: 包含 `DualThrust`, `R-Breaker` 等经典策略实现。

### A股适配性
- **复权处理**: 完美支持 A 股的前复权/后复权（代码后缀 `+` 或 `-`）。
- **多市场支持**: 除了股票，还支持期货、ETF，方便进行跨市场套利研究。
- **高性能**: C++ 底层保证了回测效率，适合大规模参数优化。

---

## 总结与建议

根据您的需求，我建议采用**组合式**的开发路径：

1.  **策略逻辑核心**: 使用 **MyTT** 作为您的“数学库”。无论是打板还是低吸，您都需要计算均线、MACD、RSI 等指标，MyTT 是最高效的工具。
2.  **交易系统框架**:
    - 如果偏向**实盘/个人模拟**且策略逻辑相对简单（如条件单），使用 **EasyQuant**。它上手最快，直接对接券商。
    - 如果偏向**复杂回测/高性能**或**多因子选股**，使用 **Qlib**（因子挖掘）或 **WonderTrader**（CTA/日内策略）。
3.  **策略复现**:
    - **打板策略**: 参考 `EasyQuant` 的架构，利用 `MyTT` 判断 `C/REF(C,1)>1.095` 且 `High==Close`。
    - **低吸策略**: 参考 `WonderTrader` 的 `DualThrust` 或 `R-Breaker` 逻辑，利用 `MyTT` 寻找超卖信号（如 `RSI<20`）。
    - **价值投资**: 深入研究 `Qlib` 的 `Alpha158` 因子集，学习如何清洗财务数据和构建多因子模型。

这个组合能最大程度地利用开源社区的优质成果，避免重复造轮子。
