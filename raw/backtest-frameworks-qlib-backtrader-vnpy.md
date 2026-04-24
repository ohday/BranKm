# 回测框架对比：Qlib / Backtrader / vn.py / Zipline-cn / bt

- **Type**: Synthesis
- **Fetched**: 2026-04-24
- **Project**: ai-swing-reminder-hkcn
- **Relevant to**: Q2
- **Sources drawn from**:
  1. [Microsoft Qlib GitHub README](https://github.com/microsoft/qlib)
  2. [Backtrader 官方站点](https://www.backtrader.com/)
  3. [VeighNa / vn.py GitHub README](https://github.com/vnpy/vnpy)

---

## 1. Microsoft Qlib（ML 量化一体化平台）[1]

### 定位
完整 ML pipeline：数据处理 → 因子构建 → 模型训练 → 回测评估 → 订单执行。41.2k star，微软维护，学术气质重。

### 内置模型（Zoo）

| 类别 | 模型 |
|---|---|
| GBDT | XGBoost (KDD16), LightGBM (NIPS17), CatBoost (NIPS18) |
| NN | MLP, LSTM, GRU, ALSTM (IJCAI17), GATs |
| 频域 | SFM (KDD17) |
| Transformer | Transformer (NeurIPS17), Localformer |
| 时序 | TCN, TRA (KDD21) |
| TensorFlow | TFT (仅 Py 3.6~3.7) |
| 自适应/图 | TabNet (AAAI19), DoubleEnsemble (ICDM20), ADARNN, ADD, IGMTF, HIST |
| 2023 新 | KRNN, Sandwich |
| RL（订单执行） | PPO, OPDS, TWAP |

**没有 HMM**（常见误解）。

### Alpha158 / Alpha360

- 定义在 `qlib/contrib/data/handler.py`
- **Alpha158** = 158 因子（动量/波动率/量价）
- **Alpha360** = 360 因子，更宽
- 官方同时支持 US Market 和 China Market（矩阵标 √）
- **Alpha158 已被 vn.py 4.0 参考复刻**到 `vnpy.alpha` 模块

### 数据格式（核心性能差异来源）

自有 `.bin` 二进制 + Expression Cache (+E) + Dataset Cache (+D)。**800 股 × 2007-2020 日频 × 14 因子** 的任务耗时对比：

| 方案 | 1 CPU | 64 CPU |
|---|---|---|
| HDF5 | 184.4s | 8.8s |
| MySQL | 365.3s | — |
| MongoDB | 253.6s | — |
| InfluxDB | 368.2s | — |
| Qlib -E -D | 147.0s | — |
| Qlib +E -D | 47.6s | — |
| **Qlib +E +D** | **7.4s** | **4.2s** |

比 HDF5 单 CPU 快 25×，比 MongoDB 快 34×。

### Qlib-RL（2022 年 11 月 release）

场景：**订单执行**。策略：TWAP / PPO (IJCAI20) / OPDS (AAAI21)。NestedExecutor 允许多层嵌套决策（如：组合管理策略 + 订单执行策略联合优化）。

### A 股支持（开箱即用）

```bash
python -m qlib.cli.data qlib_data --target_dir ~/.qlib/qlib_data/cn_data --region cn
```
数据源是 `scripts/data_collector` 的 Yahoo Finance 爬虫，社区有替代源。

### 港股支持（需自行适配！）

**README 未列 HK region**。Alpha158/360 支持矩阵只标 US + China。方案：
1. 自己用 CSV 准备数据
2. 走 "Converting CSV Format into Qlib Format" 流程
3. 或扩展 `data_collector` 爬虫脚本

**这是本研究场景的一个核心 friction point**。

### 优劣
- ✅ ML pipeline 完整，20+ 模型开箱；因子缓存性能碾压通用 DB；论文复现方便
- ❌ 实盘连接弱（偏研究）；自定义交易逻辑不如 Backtrader；YAML workflow 学习成本

## 2. Backtrader（传统事件驱动回测，自定义之王）[2]

### 架构
- **Cerebro** 中心引擎
- **Strategy**（class-based，实现 `next()` 方法）
- **Indicator**（自定义或 TA-Lib 集成）
- **DataFeed**（CSV / pandas / Yahoo / IB Live / Oanda / TradingView / ccxt / NorgateData / MQL5）
- **Broker**（含 slippage / cheat-on-open / volume fill）
- **Analyzer**（PyFolio 集成 / VWR 等）

### 订单类型
一般单 / Target Orders / **OCO**（二选一）/ **Bracket Orders**（主单+止盈+止损）/ **StopTrail**（追踪止损）/ Future-Spot 对冲补偿。

### 维护状态（警示 ⚠️）
- Copyright 2015-2024 Daniel Rodriguez
- **最近博客 2019 年**
- GitHub 最近 commit ~2023
- **项目进入维护停滞状态**，但代码量大 + 文档完善 + 社区仍活跃

### 中国数据适配
**官方零提及** akshare/tushare。适配是社区做的：
```python
import akshare as ak
import pandas as pd
import backtrader as bt

df = ak.stock_zh_a_hist(symbol="600519", period="daily",
                        start_date="20200101", end_date="20240101", adjust="qfq")
df = df.rename(columns={"日期":"date","开盘":"open","最高":"high",
                         "最低":"low","收盘":"close","成交量":"volume"})
df["date"] = pd.to_datetime(df["date"])
df.set_index("date", inplace=True)

data = bt.feeds.PandasData(dataname=df)
cerebro = bt.Cerebro()
cerebro.adddata(data)
```

### 适用场景
- 规则派波段（"20日均线回踩 + 放量 2 倍"）—— **直观**
- 多时间周期 / 多资产组合 —— 原生支持
- 需要控制下单细节（如分批开仓、动态止损）—— 灵活
- **不适合** 纯 ML 策略 / 大规模因子研究

## 3. vn.py / VeighNa 4.3.0（中国量化实盘事实标准）[3]

### 定位
**重实盘，偏中国本土期货/A 股柜台**。39.8k star，66 releases，机构用户多（私募/证券/期货公司）。

### Gateway 支持

**国内**:
- **CTP / CTP Mini** —— 期货/期权（中国期货事实柜台）
- **XTP**（中泰）—— A 股 + ETF 期权
- **Tora**（华鑫奇点） / **UFT**（恒生）/ **Femas** / **Esunny** / **HTS** / **SEC**（顶点）

**海外**:
- Interactive Brokers —— 全球证券/期货/期权
- Esunny 9.0 外盘 / Direct Access

**❌ 无 Futu / Tiger / Longbridge 等港股散户 Gateway**。

### 回测模块
- CTA Backtester（GUI + 参数优化）
- Portfolio Strategy（多合约历史回测）
- 偏 **bar 级**（tick 级执行控制），适合 CTA 策略

### `vnpy.alpha` 模块（4.0 新，ML 研究）
> 明确声明"受 Qlib 启发"
- **Dataset**: 表达式引擎 + Alpha 158 因子（复刻 Qlib）
- **Model**: Lasso / LightGBM / MLP 标准化模板
- **Strategy**: 基于 ML 信号的 cross-sectional / time-series 策略
- **Lab**: 数据管理 + 训练 + 信号生成 + 回测可视化

### Python 版本
Python 3.10~3.13（推荐 3.13）。Windows/Linux/macOS。

### 适用场景
- 期货 CTA 实盘（非本项目目标）
- 机构级稳定性要求
- **对本项目偏重**（散户、A+HK、纯提醒 + 港股下单）—— **过重**

## 4. Zipline（原 Quantopian，社区维护）

Zipline 原由 Quantopian 开发，2020 年 Quantopian 关停后社区接手为 `zipline-reloaded`。
- 事件驱动 bar 级回测
- 美股生态为主，中国数据需自行对接
- Pipeline API 适合多因子研究
- **对中国市场几乎无生态** —— 不推荐本项目主用

## 5. bt（Philippe Morissette）

轻量级策略回测：重在资产配置 / 组合再平衡。
- API: `Strategy(...) + Algo(...).run()`
- 适合：月度调仓、配置型策略
- **不适合**：高频规则择时、复杂订单管理

---

## 对比矩阵

| 维度 | **Qlib** | **Backtrader** | **vn.py** | Zipline | bt |
|---|---|---|---|---|---|
| 活跃度 | ✅ 微软持续维护 | ⚠️ 2019 后停滞 | ✅ 2025 年 4.3 | 社区 | 低 |
| ML 集成 | ✅ 内置 20+ | ❌（手动） | 🟡 4.0 新增 alpha | ❌ | ❌ |
| 因子引擎 | ✅ Alpha158/360 | ❌ | 🟡 Alpha158 复刻 | 🟡 Pipeline | ❌ |
| 数据性能 | ✅ .bin 秒杀 | 🟡 pandas | 🟡 | 🟡 | 🟡 |
| 自定义交易逻辑 | 🟡 中等 | ✅ 极灵活 | ✅ 灵活 | 🟡 | ❌ |
| A 股原生 | ✅ cn region | ❌ 自适配 | ✅ XTP | ❌ | ❌ |
| **港股原生** | **❌** | **❌** | **❌** | **❌** | **❌** |
| 实盘对接 | 🟡 弱 | 🟡 IB/Oanda | ✅ 国内柜台 | 🟡 弱 | ❌ |
| 中文社区 | ✅ | 🟡 | ✅ | ❌ | ❌ |
| 入门成本 | 🟡 YAML + 表达式 | 🟢 低 | 🟡 中 | 🟡 中 | 🟢 低 |
| 波段场景适配 | ✅ 日频研究 | ✅ 规则派 | 🟡 偏 CTA | 🟡 | 🟢 |

---

## 本项目推荐路径（中级 + 波段 + A+HK + 纯提醒 + 港股下单）

### 双轨推荐

```
┌────────────────────────────────────────────────────────┐
│  主干：Qlib                                            │
│  - A 股日频历史回测 + Alpha158 因子基线                │
│  - LightGBM / LSTM 作为 AI 辅助模型                    │
│  - ML 路线（Q3 路线 B/D）的首选承载                    │
│                                                        │
│  辅助：Backtrader                                      │
│  - 规则派波段（Q3 路线 A）的直观实现                   │
│  - "20日均线 + 放量"这类自然语言规则直接映射           │
│  - 港股策略的承载（自己喂 AkShare/Futu 数据）          │
│  - OCO / Bracket / StopTrail 等订单语义本地模拟        │
│                                                        │
│  不要选：vn.py                                         │
│  - 太重，偏期货柜台；散户提醒场景用不上                │
│  - 如未来扩展到 A 股 CTA 期货再考虑                    │
│                                                        │
│  不要选：Zipline / bt                                  │
│  - 中国市场生态空白                                    │
└────────────────────────────────────────────────────────┘
```

### 决策节点

| 如果你的策略是... | 选 |
|---|---|
| "Alpha 因子排名 + Top-N 持仓" 这类多因子选股 | **Qlib** |
| "MA / MACD / RSI 交叉 + 量价确认" 规则派 | **Backtrader** |
| "LightGBM 对全 A 排序，每周调仓" | **Qlib** |
| "单只港股的分时突破 + 止盈止损" | **Backtrader**（自喂 Futu 数据） |
| "跨 A+HK 配对交易" | Backtrader（Qlib 港股要自适配） |
| 未来要做 A 股期货 CTA 实盘 | vn.py |

### 港股痛点

- Qlib / vn.py / Backtrader / Zipline 都**没有原生港股数据层**
- 解决方案：AkShare / Futu OpenAPI 抓数据 → CSV/pandas → 喂入 Backtrader
- 或用 Qlib 的 CSV → .bin 转换脚本构建本地港股 bin 库

### 学习路径（中级用户）

1. Qlib 15 分钟跑通 `qrun examples/benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml` baseline
2. 读 `workflow_by_code.ipynb` 理解 handler / model / strategy 解耦
3. 同时在 Backtrader 实现一个纯规则策略（MA 金叉 + 放量）
4. 两者对比后决定主干走哪条
