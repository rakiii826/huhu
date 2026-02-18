# Microsoft Qlib 深度解析与 Mac 操作指南

本报告在上一份调研报告的基础上，针对您对 **Microsoft Qlib** 的兴趣，进行了更深度的技术解析和 Mac 环境下的实操验证。

---

## 第一部分：Qlib 策略与数学原理深度解析

Qlib 不仅仅是一个回测框架，它本质上是一个 **"AI 算子工厂" (AI Operator Factory)**。它的核心理念是将量化投资视为一个标准的机器学习监督学习问题 (Supervised Learning)。

### 1. Alpha158/Alpha360 因子库：AI 的燃料

Qlib 并不直接使用传统的 "RSI > 80" 这种规则，而是通过**表达式引擎 (Expression Engine)** 生成海量因子，供模型训练。

#### 核心逻辑
- **基础数据**: `Open`, `High`, `Low`, `Close`, `Volume`
- **算子 (Operators)**:
  - `Ref(x, d)`: 获取 d 天前的数据（时间平移）。
  - `Mean(x, d)`: d 天的移动平均。
  - `Std(x, d)`: d 天的标准差。
  - `Slope(x, d)`: d 天的线性回归斜率（趋势强度）。
  - `Rsquare(x, d)`: d 天的线性回归 R²（趋势稳定性）。
  - `Rank(x, d)`: x 在过去 d 天中的百分比排名。

#### 示例：一个动量反转因子的诞生
Qlib 可能会生成这样一个因子：
`Ref($close, 0) / Mean($close, 20) - 1`
*数学含义*: 当前价格相对于 20 日均线的偏离度（乖离率 Bias）。AI 模型会学习这个偏离度与未来收益率的关系。如果模型发现 Bias < -10% 时未来收益率很高，它就学会了 **"低吸"**。

### 2. 模型训练：LightGBM 预测未来收益

在 Qlib 中，策略不是由人写死的规则，而是由模型预测的 **Score (分数)** 决定的。

- **输入 (X)**: Alpha158 生成的 158 个因子值。
- **目标 (Y)**: 未来 N 日的收益率。通常定义为：
  $$ Y = \frac{Ref(Close, -2)}{Ref(Close, -1)} - 1 $$
  *(注意：这里使用了 Ref(..., -2) 表示后天，即 T+2 的开盘/收盘，符合 A 股 T+1 交易规则的实际收益)*
- **模型**: 使用 LightGBM (GBDT) 对 X 和 Y 进行拟合。
- **输出**: 预测分数 $\hat{y}$。分数越高，模型认为该股票未来涨幅越大。

### 3. 组合策略：TopK Dropout

这是 Qlib 最经典的策略，非常适合 **"价值/多因子"** 风格。

- **每日持仓**: 持有预测分数 $\hat{y}$ 最高的 Top $K$ 只股票（如 Top 50）。
- **换仓逻辑**:
  - 每天计算所有股票的新分数。
  - 如果持仓股的排名掉出 Top $K + n\_drop$（缓冲区），则卖出。
  - 如果新股票进入 Top $K$，则买入。
  - **缓冲区设计**: 防止频繁换仓（减少交易成本）。

---

## 第二部分：A 股特色适配 (涨跌停与 T+1)

Qlib 在回测引擎 (`Exchange`) 中原生支持 A 股规则。

### 1. 涨跌停限制 (Limit Threshold)
在配置文件中，你可以看到：
```yaml
exchange_kwargs:
  limit_threshold: 0.095  # 9.5% 限制（考虑误差，实际对标 10%）
  deal_price: close
```
- **买入限制**: 如果当天 `High` 涨停（或 `Close` 涨停），Qlib 会判断无法买入。
- **卖出限制**: 如果当天 `Low` 跌停，Qlib 会判断无法卖出。
- **数学逻辑**:
  $$ \text{CanTrade} = \text{True} \quad \text{if} \quad | \frac{Price_{t}}{Price_{t-1}} - 1 | < \text{Threshold} $$

### 2. T+1 交易
Qlib 的回测引擎默认按 **日 (Day)** 撮合。
- **买入**: T 日买入。
- **卖出**: T+1 日及以后才能卖出（只要策略不在当天同时生成买卖信号，自然符合 T+1）。

---

## 第三部分：Mac 环境操作指南 (M1/M2/M3 & Intel)

在 Mac 上运行 Qlib 需要特别注意 **编译环境** 和 **LightGBM** 的依赖。

### 1. 环境准备 (Prerequisites)

打开终端 (Terminal)，执行以下步骤：

**步骤 A: 安装 Xcode 命令行工具** (用于编译 C++ 扩展)
```bash
xcode-select --install
```

**步骤 B: 安装 Homebrew** (Mac 的包管理器，如果已安装可跳过)
*(参考 brew.sh 官网安装)*

**步骤 C: 安装基础依赖库**
LightGBM 需要 OpenMP 支持，PyTables 需要 HDF5。
```bash
brew install libomp
brew install hdf5
```

**步骤 D: 安装 Python 环境 (推荐使用 Miniforge 或 Conda)**
强烈建议使用 Python 3.8, 3.9 或 3.10 (3.12 可能会有兼容性问题)。
```bash
conda create -n qlib_env python=3.9
conda activate qlib_env
```

### 2. 安装 Qlib

**推荐方式: 源码安装** (最稳妥，避免二进制包不兼容)

```bash
# 1. 克隆代码
git clone https://github.com/microsoft/qlib.git
cd qlib

# 2. 安装 Python 依赖 (先安装 numpy, cython 以确保编译顺利)
pip install numpy cython

# 3. 安装 Qlib
# 注意：在 Mac 上安装 LightGBM 经常需要指定 OpenMP 路径，但 qlib 会自动处理大部分依赖
pip install .
```

*如果在安装过程中遇到 LightGBM 报错，请单独安装:*
```bash
pip install lightgbm --install-option=--nomp
# 或者
export CXX=clang++
export CC=clang
pip install lightgbm
```

### 3. 数据准备 (A 股数据)

Qlib 提供了一键下载脚本。A 股数据 (1min/1day) 来源于网络爬虫整理。

```bash
# 下载并初始化 A 股数据 (约几 GB)
# target_dir 是数据存放目录，建议放在 ~/.qlib/qlib_data/cn_data
python scripts/get_data.py qlib_data --target_dir ~/.qlib/qlib_data/cn_data --region cn
```

### 4. 运行第一个 Demo

我们需要运行一个完整的 "数据 -> 训练 -> 回测" 流程。

1.  进入示例目录：
    ```bash
    cd examples
    ```
2.  运行 LightGBM 工作流 (Alpha158)：
    ```bash
    qrun benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml
    ```
    *注意：你需要修改 yaml 文件中的 `provider_uri` 指向你刚才下载的数据目录 `~/.qlib/qlib_data/cn_data`。*

3.  **查看结果**:
    运行结束后，屏幕会打印：
    - **IC (Information Coefficient)**: 预测值与真实值的相关性。
    - **Backtest Return**: 回测年化收益率。
    - **Analysis**: 超额收益 (Alpha) 曲线。

---

## 第四部分：如何实现您的自定义策略？

您提到想做 **"低吸"** 和 **"打板"**。虽然 Qlib 擅长 Alpha，但也可以通过自定义因子来实现。

### 实现 "低吸" 策略 (Mean Reversion)
**核心思路**: 寻找短期跌幅大但长期趋势向上的股票。

1.  **定义因子**:
    在 `qlib/contrib/data/handler.py` 中添加自定义因子：
    - `RSI`: 使用 Qlib 表达式实现 (参考 `Alpha158` 中的 `SUMP`, `SUMN`)。
    - `Bias`: `(Close - MA(20)) / MA(20)`
2.  **训练模型**:
    让 LightGBM 学习这些反转因子。
3.  **修改策略 (TopkDropoutStrategy)**:
    在配置 YAML 中，将 `method_buy` 设置为 `bottom` (买入预测分数最低的，假设它是反转模型) 或者保持 `top` (如果模型预测的是反转后的涨幅)。

### 实现 "打板" 监控
Qlib 的回测是 **日级别** 的，无法精确模拟 "盘中触板" 的微秒级操作。
**建议**:
- 使用 Qlib 做 **选股 (Screening)**: 每天收盘后，预测第二天最可能涨停的股票池。
- 将 Qlib 选出的 Top 50 股票列表，导出给 **EasyQuant**。
- 让 **EasyQuant** 在盘中监控这 50 只股票，一旦价格触及涨停价，立即下单。

---

## 总结

**Microsoft Qlib** 是一个强大的 AI 投研平台。
- **优点**: 工业级代码质量，AI 训练流程完善，A 股数据支持好。
- **缺点**: 对 "盘中高频/打板" 支持较弱（回测粒度不够细）。
- **Mac 适配**: 只要解决 `libomp` (OpenMP) 和 `hdf5` 依赖，在 Mac (M1/M2) 上运行非常流畅，训练速度也很快。

建议您先在 Mac 上跑通 `Alpha158` 的 Demo，理解了 "因子 -> 模型 -> 策略" 的流程后，再尝试修改因子来实现您的低吸逻辑。
