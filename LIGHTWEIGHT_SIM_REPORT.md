# 轻量级 A 股实盘模拟系统推荐方案

针对您“轻量级”、“实盘模拟”、“Mac 环境”的需求，Qlib 确实显得过于厚重且配置繁琐。

经过调研，我为您筛选了目前最主流、最轻量且适合 Mac 的 **两套** 解决方案：

1.  **DIY 极简循环系统 (推荐)**：基于 `Akshare` (数据) + `MyTT` (逻辑) 的纯 Python 脚本。
    -   **优点**: 代码完全可控，零依赖，Mac 完美运行，极其轻量。
    -   **缺点**: 需要自己写一个 `while True` 循环。
2.  **EasyQuant (进阶)**：基于 `EasyTrader` (交易) + `EasyQuotation` (行情) 的事件驱动框架。
    -   **优点**: 框架结构清晰，内置了雪球 (Xueqiu) 模拟盘接口。
    -   **缺点**: 依赖雪球网页版登录，可能需要处理验证码或 Cookie 过期问题。

---

## 方案一：DIY 极简循环系统 (最推荐)

这是目前 Python 量化圈最流行的“轻量级”玩法。不需要安装任何复杂的框架，只需要一个 Python 脚本。

### 核心架构
-   **行情源**: `Akshare` (免费、开源、数据全)。
-   **指标库**: `MyTT` (直接复制 `MyTT.py` 文件即可)。
-   **交易执行**: 模拟阶段只需要打印 Log 或写入 CSV 文件，甚至可以推送到微信/钉钉。

### 代码实现 (Copy & Run)

您可以直接在 Mac 上运行以下代码，它已经是一个完整的 **“低吸/打板监控系统”**：

```python
import time
import pandas as pd
import akshare as ak
from MyTT import *  # 假设您已下载 MyTT.py
from datetime import datetime

# ================= 配置区域 =================
# 监控股票列表 (示例：贵州茅台, 宁德时代)
STOCK_LIST = ['600519', '300750']
# 策略参数
MA_PERIOD = 20    # 均线周期
DROP_THRESHOLD = -0.05 # 低吸阈值 (跌幅超过5%)

def log_trade(action, code, price, msg):
    """模拟交易日志"""
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    print(f"[{timestamp}] {action} {code} @ {price}: {msg}")
    # 这里可以扩展：写入 csv，发送邮件/微信通知

def strategy_logic(code, df):
    """
    核心策略逻辑 (基于 MyTT)
    df 包含: open, close, high, low, volume
    """
    # 提取数据序列
    CLOSE = df['close'].values

    # 1. 计算指标 (使用 MyTT)
    ma20 = MA(CLOSE, MA_PERIOD)
    current_price = CLOSE[-1]
    last_ma20 = ma20[-1]

    # 2. 策略判断
    # 示例 A: 低吸策略 (价格低于 MA20 且 跌幅较大)
    daily_return = (current_price - df['close'].values[-2]) / df['close'].values[-2]

    if current_price < last_ma20 and daily_return < DROP_THRESHOLD:
        return "BUY", f"低吸信号触发! 跌幅: {daily_return:.2%}, 现价: {current_price}"

    # 示例 B: 打板策略 (价格触及涨停)
    # 简单模拟: 涨幅 > 9.5%
    if daily_return > 0.095:
        return "BUY", f"打板信号触发! 涨幅: {daily_return:.2%}, 现价: {current_price}"

    return None, None

def run_simulation():
    print("启动轻量级模拟交易系统 (Ctrl+C 停止)...")
    while True:
        try:
            # 1. 获取全市场实时行情 (速度快，一次请求)
            # 注意: akshare 接口可能会更新，这里演示核心逻辑
            df_all = ak.stock_zh_a_spot_em()

            # 2. 遍历监控股票
            for code in STOCK_LIST:
                # 从全市场数据中筛选当前股票
                row = df_all[df_all['代码'] == code]
                if row.empty:
                    continue

                price = row['最新价'].values[0]
                pct_chg = row['涨跌幅'].values[0]

                # 简单策略: 涨幅超过 9% 报警
                if pct_chg > 9.0:
                    log_trade("ALERT", code, price, f"即将涨停! 涨幅: {pct_chg}%")

                # 如果需要历史K线计算 MA20 (低吸策略需要历史数据)
                # 可以在这里通过 ak.stock_zh_a_hist() 获取日线数据
                # df_hist = ak.stock_zh_a_hist(symbol=code, period="daily", adjust="qfq")
                # action, msg = strategy_logic(code, df_hist)
                # if action:
                #     log_trade(action, code, price, msg)

            # 3. 休息 N 秒 (避免请求过频)
            time.sleep(3)

        except Exception as e:
            print(f"Error: {e}")
            time.sleep(10)

if __name__ == "__main__":
    run_simulation()
```

---

## 方案二：EasyQuant (框架式)

如果您希望有一个更“正规”的框架来管理资金、持仓和订单，**EasyQuant** 是不二之选。

### 核心优势
- **自带引擎**: 不需要自己写 `while True`。
- **雪球模拟**: 内置了对接雪球组合 (Snowball Cube) 的功能，可以进行真实的“模拟下单”，并在雪球 App 上看到收益曲线。

### Mac 使用指南
1.  **安装**:
    ```bash
    pip install easyquant
    ```
2.  **配置雪球账号**:
    - 需要在 `xq.json` 中配置您的雪球账号信息 (cookies)。
    - *注意*: 获取 cookies 可能需要手动抓包，略有门槛。
3.  **编写策略**:
    在 `strategies/` 目录下新建 `MyStrategy.py`:
    ```python
    from easyquant import StrategyTemplate

    class Strategy(StrategyTemplate):
        name = '我的打板策略'

        def strategy(self, event):
            # event.data 是实时推送的行情字典
            stock = event.data['000001']
            if stock.now > stock.open * 1.095:
                # 调用 easytrader 的买入接口
                self.user.buy('000001', price=stock.now, amount=100)
    ```
4.  **运行**: `python main.py`

---

## 总结建议

- **最推荐 (DIY)**: **方案一 (Akshare + MyTT)**。
    - 原因：**绝对轻量**，代码就在您眼皮底下，想改什么逻辑都行。对于“低吸”和“打板”这种强逻辑策略，自己控制循环比用框架更灵活。Mac 上运行毫无压力。
- **备选 (EasyQuant)**: **方案二**。
    - 原因：如果您一定要看到“模拟盘账户”的资金变动曲线，且不愿意自己写 `cash = cash - cost` 的逻辑，可以用它对接雪球。

**下一步建议**:
建议您先复制 **方案一** 的代码，在本地跑起来。看着控制台不断打印行情和信号，您会对量化系统的运作有更直观的理解。等逻辑成熟了，再考虑接入 EasyQuant 或实盘接口。
