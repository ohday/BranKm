# 开源仓库深度清单（波段 + A/HK + 中级）

- **Type**: Synthesis
- **Fetched**: 2026-04-24
- **Project**: ai-swing-reminder-hkcn
- **Relevant to**: Q4
- **Sources drawn from**:
  1. [Microsoft Qlib](https://github.com/microsoft/qlib)
  2. [FinRL](https://github.com/AI4Finance-Foundation/FinRL)
  3. [FinGPT](https://github.com/AI4Finance-Foundation/FinGPT)
  4. [TradingAgents](https://github.com/TauricResearch/TradingAgents)
  5. [TradingAgents-CN](https://github.com/hsliuping/TradingAgents-CN)
  6. [FinMem](https://github.com/pipiku915/FinMem-LLM-StockTrading)
  7. [ai-hedge-fund (virattt)](https://github.com/virattt/ai-hedge-fund)
  8. [vnpy](https://github.com/vnpy/vnpy)
  9. [awesome-quant 中文版 (thuquant)](https://github.com/thuquant/awesome-quant)
  10. [easytrader (shidenggui)](https://github.com/shidenggui/easytrader)
  11. [Backtrader](https://www.backtrader.com/)
  12. [AkShare](https://github.com/akfamily/akshare)

---

## Tier-1（必 fork / 主干）

### 1. **TradingAgents-CN**（多 Agent LLM 研究员 · 中国本地化）
- **Stars**: 24.6k / **Fork**: 5.2k
- **License**: Apache 2.0（v2.0 闭源预警）
- **契合度**: ★★★★★ —— A 股 + HK + 美股，Qwen/DeepSeek 优先，Docker 部署
- **用法**: 作为"研究员 agent"每周末/盘前对 watchlist 出多角度研报
- **改造点**: 加一层 cron + 阈值 → Telegram（路线 C 不是信号源，而是二次确认器）

### 2. **Microsoft Qlib**（ML 量化一体化）
- **Stars**: 41.2k
- **License**: MIT
- **契合度**: ★★★★☆ —— A 股开箱，港股需自适配 CSV→bin
- **用法**: Alpha158 因子库 + LightGBM 基线 + 回测报告
- **配套**: `workflow_by_code.ipynb` 是中级入门最佳教程

### 3. **Backtrader**（规则派波段的自定义之王）
- **Stars**: 14k+
- **License**: GPL-3
- **契合度**: ★★★★☆ —— 维护停滞但代码完整 + 订单语义丰富
- **用法**: 规则派波段（"20日均线回踩 + 放量 2 倍"类策略）
- **注意**: 自己拼 AkShare 数据 → pandas → PandasData feed

### 4. **AkShare**（A+HK+宏观一站式数据）
- **Stars**: 18.5k
- **License**: MIT
- **活跃**: 178 releases，2026-04 仍在更新
- **契合度**: ★★★★★ —— 免费 + 覆盖广 + 港股覆盖
- **用法**: 所有非实时 A/HK 数据的首选源

### 5. **Futu OpenAPI (futu-api)**
- **官方**: [富途 OpenAPI](https://openapi.futunn.com/)
- **License**: 官方 SDK（闭源，但可白嫖 LV2）
- **契合度**: ★★★★★ —— 港股实时行情 + 下单
- **注意**: 需运行 OpenD 本地 gateway（Windows/Mac/Linux）

## Tier-2（强烈参考 / 按需 fork）

### 6. **Qlib-RL / FinRL**（路线 B 的两个入口）
- Qlib-RL 是 Qlib 的 RL 模块（TWAP/PPO/OPDS），**偏订单执行**
- FinRL 42k+ star，**偏策略级 RL**（DDPG/PPO/A2C/TD3/SAC）
- 多中国数据源支持：AkShare / Baostock / JoinQuant / Tushare / RiceQuant
- **本项目优先级**: 波段策略不需要 RL，先验证路线 A 再考虑

### 7. **ai-hedge-fund (virattt)**
- **Stars**: 57.2k（MIT）
- **契合度**: ★★★☆☆ —— 美股硬编码，A 股改造成本大
- **价值**: 13 投资人人格 prompt 设计值得**抄过来**用到 TradingAgents-CN 里

### 8. **FinGPT**（NLP 增强 tool，不是主策略）
- **License**: MIT
- **契合度**: ★★★☆☆ —— 新闻情感打标用
- **推荐模型**: `fingpt-mt_qwen-7b_lora`（中文 + 轻量）
- **用法**: 作为 OpenClaw 的一个 sentiment skill，调 HuggingFace Inference Endpoint

### 9. **Longbridge OpenAPI**
- **契合度**: ★★★★☆ —— 港股下单 + 无本地 daemon（比 Futu 轻）
- **对比 Futu 的优势**: Python SDK 设计更现代，环境变量配置简洁
- **对比 Futu 的劣势**: 社区小，Windows GUI 体验弱

### 10. **Baostock**（A 股免费历史数据兜底）
- **License**: BSD
- **契合度**: ★★★★☆（仅 A 股）
- **价值**: Tushare 免费档不够用、AkShare 不稳时的稳定兜底

### 11. **QuantAxis / RQAlpha / pyalgotrade-cn**（thuquant awesome 列出）
- **契合度**: ★★☆☆☆ —— 活跃度低于 Qlib
- **价值**: 研究用；生产优先 Qlib+Backtrader

## Tier-3（了解即可）

### 12. **vnpy**
- 39.8k star，4.3.0 (2025-12)
- 定位：**中国期货/A 股实盘柜台**（CTP / XTP / Tora / UFT）
- **本项目不适合**：散户场景 + 纯提醒，vnpy 太重
- **未来扩展**: 如果做 A 股 CTA 期货实盘，再进

### 13. **easytrader / easyquotation (shidenggui)**
- **Stars**: easytrader 9.6k
- **机制**: 模拟同花顺 GUI 操作 + miniQMT 封装 + 雪球组合
- **法律灰区**: 未在 README 讨论，但社区知道**不要用 GUI 模拟去大资金交易**（券商可能封号）
- **正路**: 用其 `miniQMT` 封装（真 API，前提是券商给你开 QMT 权限）
- **本项目评估**: 纯提醒不需要下单，**可忽略**；未来扩展到 A 股实盘时，走 QMT 正路

### 14. **Qbot (Charmve)**
- AI 量化系统，Qlib + RL 一个"大杂烩"
- 未 deeply verified；**参考而不依赖**

### 15. **FinMem / FinAgent**（学术代码）
- 实验性质 > 生产价值
- 适合读论文理解架构，不直接用

### 16. **Alpha-GPT 1.0/2.0**
- **无公开代码** —— 只能读论文（arXiv:2308.00016, 2402.09746）参考思路
- 2025 EMNLP demo track

---

## A 股实盘 API 现状（Q7 前置）

### QMT / miniQMT（迅投）[来自搜索]
- **支持券商**: 国信 / 东财 / 华泰 / 中信 / 国泰君安 等多数头部
- **资金门槛**: **50 万**（部分券商放宽到 30 万）
- **API**: 免费（开了权限即可用）
- **SDK**: `xtquant`（Python 3.6~3.12），需本地运行 MiniQMT 客户端
- **功能**: XtData（行情） + XtTrader（下单，支持推送）
- **本项目**: 超出 50 万门槛才有意义；符合"纯提醒"用户画像的话不需要

### Ptrade（恒生电子）
- **资金门槛**: **100 万**
- **语言**: Python / VBA
- **形态**: 可视化 + 后端
- **本项目**: 门槛高，非首选

### 机构柜台（恒生 O32 / 金证 / 华锐）
- 门槛 500 万 + 私募资质 + 系统接入认证
- **本项目完全不在考虑范围**

### easytrader GUI 模拟
- **灰色地带**: 理论可用，实操有封号风险（券商检测）
- **不推荐**作为生产路径

---

## 港股实盘 API 三家对比（Q7 核心）

| 维度 | **Futu (富途)** | **Longbridge (长桥)** | **Tiger (老虎)** |
|---|---|---|---|
| Python SDK | `futu-api` | `longport` | `tigeropen` |
| 本地 daemon | ✅ 需 OpenD | ❌ 不需 | ❌ 不需 |
| 市场 | HK/US/CN-A/SG/JP/AU | HK/US/CN-A/SG | HK/US/SG/AU/CN-A |
| 港股 LV2 | **✅ 中国大陆个人免费** | 需付费 | 需付费 |
| 美股 LV1 | 部分免费 | 免费 tier | 付费 |
| 订单类型 | 市/限/止损/条件/特殊 HK | 市/限/到价/追踪止损 | MKT/LMT/STP/STP_LMT/Trailing |
| 行情限频 | ~30/s | 1s≤10 次 + 并发≤5，SDK 内置 | ~60-120/min |
| 交易限频 | 宽松 | 30s≤30 次，间隔 ≥0.02s，**SDK 不内置** | 中等 |
| 错误处理 | 自定义 | 返回 301606 | HTTP 标准 |
| 社区 | ✅ 最大 | 🟡 中等 | 🟡 中等 |
| 架构 | TCP via OpenD | REST + WebSocket | REST + WebSocket |
| 下单 API 手续费 | ✅ 无额外费 | ✅ 无额外费 | ✅ 无额外费 |

### 本项目推荐

**首选 Futu**（港股下单 + LV2 免费，杀手锏）。Longbridge 作备选（SDK 更现代、无 daemon 更适合 OpenClaw 承载）。Tiger 如果你已经用它做美股/新加坡可以，否则没必要多开一家。

---

## 仓库选型决策表

| 我的角色是... | Tier-1 组合 |
|---|---|
| 刚起步 | **AkShare + Backtrader + Claude (路线 A)** |
| 想要多角度研报 | **+ TradingAgents-CN (路线 C 辅助)** |
| 想做多因子选股 | **+ Qlib + Alpha158 (路线 D)** |
| 港股要实盘下单 | **+ Futu OpenAPI** |
| 港股下单且偏好轻量架构 | **+ Longbridge** |
| A 股上了 50 万想实盘 | **+ xtquant (QMT)** |

## 注意事项

1. **TradingAgents-CN v2.0 将闭源** —— 锁定在 v1.x 的最后一个稳定版，或 fork 后自己维护
2. **Backtrader 2019 后停滞** —— 已够用但不要期待新功能
3. **Qlib 港股自适配**：要写一个 CSV 爬虫（AkShare 港股日线 → CSV → qlib bin 转换脚本）
4. **Futu OpenD 部署**：Windows 有 GUI 版，Linux 纯 CLI，**需 7×24 在线**——对 OpenClaw VPS 是硬需求
5. **LangGraph 依赖**：TradingAgents / ai-hedge-fund 都用 LangGraph，注意 LangChain 版本漂移
