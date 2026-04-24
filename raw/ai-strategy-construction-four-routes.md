# AI 帮构建策略：四路线对比（LLM 规则 / RL / Multi-Agent / LLM 因子挖掘）

- **Type**: Synthesis
- **Fetched**: 2026-04-24
- **Project**: ai-swing-reminder-hkcn
- **Relevant to**: Q3（核心问题）
- **Sources drawn from**:
  1. [TradingAgents GitHub (TauricResearch/TradingAgents)](https://github.com/TauricResearch/TradingAgents)
  2. [TradingAgents 论文 arXiv:2412.20138](https://arxiv.org/abs/2412.20138)
  3. [TradingAgents-CN (hsliuping)](https://github.com/hsliuping/TradingAgents-CN)
  4. [FinRL GitHub](https://github.com/AI4Finance-Foundation/FinRL)
  5. [FinGPT GitHub](https://github.com/AI4Finance-Foundation/FinGPT)
  6. [FinMem-LLM-StockTrading GitHub](https://github.com/pipiku915/FinMem-LLM-StockTrading)
  7. [FinMem 论文 arXiv:2311.13743](https://arxiv.org/abs/2311.13743)
  8. [Alpha-GPT 论文 arXiv:2308.00016](https://arxiv.org/abs/2308.00016)
  9. [Alpha-GPT 2.0 论文 arXiv:2402.09746](https://arxiv.org/abs/2402.09746)
  10. [ai-hedge-fund (virattt)](https://github.com/virattt/ai-hedge-fund)
  11. [TradingAgents-CN 部署教程 CSDN](https://blog.csdn.net/benshu_001/article/details/149693611)

---

## 路线 A：LLM 对话式生成规则/因子代码（pair-quant）

### 机制

开发者用自然语言描述策略意图 → LLM（Claude / GPT / DeepSeek）生成因子表达式、择时规则、回测代码 → 人工审核 + 执行 → 读回测结果反馈给 LLM → 参数微调 / 规则迭代。

**不是"LLM 自主做决策"，而是"LLM 当编程助手 + 金融顾问"。**

### 工作流
```
自然语言偏好 → prompt 模板 → LLM 生成因子/规则代码
  → 人工 review → 放入 Qlib/Backtrader 回测 → 读报告
  → prompt 追问 "夏普只有 0.8，怎么优化？" → LLM 给建议
  → 迭代直到收敛 → 部署
```

### 代表实践
- 无专用开源框架（太通用了），但 **Cursor / Claude Code / ChatGPT + Qlib/Backtrader** 组合是事实标准。
- 提示词工程是关键：见 Q8。

### 数据需求
- 标准 OHLCV + 基本面（AkShare / Baostock 够用）
- LLM token（¥10-50/次迭代）

### 算力
- 本地 CPU 即可，LLM 调 API

### 过拟合风险
- **中**：人在环里能识别明显过拟合，但参数 grid search 仍有风险。
- 用 walk-forward validation 控制。

### 适合用户
- ✅ **中级量化 + 波段 + 规则派 + 少量策略**
- ✅ 本项目场景的**首选路线**
- ❌ 大规模因子研究（用路线 D）

---

## 路线 B：强化学习（FinRL / Qlib-RL）[4]

### 机制

Market Environments + DRL Agents + Financial Applications 三层。Agent 观察 state (OHLCV + 技术指标)，输出 action (买卖权重)，环境返回 reward (回报)。

### 核心 repo：FinRL（AI4Finance Foundation）

**环境** (`finrl/meta/`):
- `env_stock_trading`
- `env_portfolio_allocation`
- `env_cryptocurrency_trading`
- `env_high_frequency_trading`

**默认 State 特征**:
```
OHLCV + adjusted close + [macd, boll_ub, boll_lb, rsi_30, dx_30,
                          close_30_sma, close_60_sma] + VIX + turbulence index
```

**Action space**: 主要连续（DDPG/TD3/SAC），也有 A2C/PPO 兼容离散。

**算法**: A2C / DDPG / PPO / TD3 / SAC（Stable Baselines 3 / ElegantRL / RLlib 三种 wrapper）。

**中国数据源支持**（关键）:

| Source | 覆盖 |
|---|---|
| AkShare | CN Securities, 2015-至今, 1day |
| Baostock | CN Securities, 1990-12-19 至今, 5min |
| JoinQuant | CN Securities, 2005-至今, 1min |
| RiceQuant | CN Securities, 2005-至今, 1ms |
| Tushare | CN Securities A股, 至今, 1min |

所以 FinRL 能**直接跑 A 股**，无需额外工作。

### 关键论文
- NeurIPS 2018 Workshop: "Practical deep RL approach for stock trading"
- ACM ICAIF 2020: "Ensemble strategy"
- arXiv:2011.09607 (NeurIPS 2020 DeepRL): FinRL 论文
- NeurIPS 2022: "FinRL-Meta: Market Environments and Benchmarks"
- 2024 Springer: "Dynamic Datasets..."
- FinRL-X arXiv:2603.21330 (PAKDD 2026)

### 算力
- Stable Baselines 3 支持 CPU/GPU
- CPU 训练 1 个 env 大约 2-6 小时（日频，5-10 年数据）

### 过拟合风险
- **高**：RL 最大痛点。DDPG 在 A 股容易过拟合；PPO 相对稳。
- 必须用 in-sample / out-of-sample 严格划分 + paper trading。

### 适合用户
- ✅ 有 ML/RL 基础、愿意调参
- ✅ 组合优化、多资产配置场景
- ❌ 规则派波段（overkill）
- ⚠️ **本项目不推荐作为主干**：波段 + 纯提醒不需要 RL 的复杂性。但可以作为进阶路线。

---

## 路线 C：Multi-Agent LLM 研究员（TradingAgents / FinMem / FinAgent）

### TradingAgents（本路线代表作）[1][2]

**论文**: arXiv:2412.20138 (Yijia Xiao, Edward Sun, Di Luo, Wei Wang, 2025, TauricResearch)

**架构（像个真实投行）**:

```
分析师团队（4 人）:
  Fundamentals Analyst   ──┐
  Sentiment Analyst      ──┤  → 报告
  News Analyst           ──┤
  Technical Analyst      ──┘

研究员团队（辩论）:
  Bullish Researcher  ⇆  Bearish Researcher
  (structured debates, max_debate_rounds 可配置)

交易员:
  Trader Agent → 综合报告决定 timing + magnitude

风控 + 组合经理:
  Risk Management Team → Portfolio Manager → approve/reject
  → Simulated Exchange
```

**通信**: LangGraph 编排；Bull/Bear 研究员辩论是核心；记忆/反思机制未在 README 详述（在论文里）。

**LLM 支持**: OpenAI / Google / Anthropic / xAI / DeepSeek / Qwen (DashScope) / Zhipu / OpenRouter / Ollama（本地）+ 企业版 Azure OpenAI / AWS Bedrock。

**单次分析成本估算**:
- 4 分析师 + 2 研究员 × N 轮辩论 + 交易员 + 风控 + 组合经理 = **10-20+ LLM calls**
- GPT-4 级别：**$0.50 ~ $5+/次/股票**
- Ollama 本地：零 API 成本，需 GPU

**市场覆盖**: 示例用 NVDA，数据源 **Alpha Vantage（偏美股）**。A 股/HK 需要自适配。

### ★ TradingAgents-CN（中国版，本项目重点）[3][11]

**GitHub**: `hsliuping/TradingAgents-CN` —— 24.6k star, 5.2k fork（截至 2026-04）

**核心改造**:

| 改造点 | 细节 |
|---|---|
| 数据源 | Tushare (主) + AkShare (备) + BaoStock + 多级 fallback：`stock_bid_ask_em → stock_zh_a_spot → stock_zh_a_spot_em → stock_zh_a_hist` |
| LLM | Qwen / DeepSeek 优先；OpenAI / Google / Anthropic / AiHubMix / 阿里百炼 |
| 市场 | A 股 + 港股 + 美股 |
| 报告导出 | Markdown / Word / PDF |
| 部署 | Docker Compose 多服务（FastAPI + Vue3 + MongoDB + Redis + Nginx），多架构（amd64 + arm64 支持树莓派/Apple Silicon），5 分钟启动 |
| 限制 | ❌ 不含实盘；❌ 不集成 Futu/券商 API；❌ v2.0 不再开源（盗版原因） |

**分析师角色**（README 提及）:
- Market Analyst（技术指标）
- Fundamentals Analyst（PE/PB，含 Qwen LLM 重建逻辑）
- （推断还有）News / Risk / Bull / Bear

**应用示例**:
- 海康威视（002415）：多空倾向度 90% / 风险评分 40%

**本项目评估**:
- ✅ **最接近用户目标的现成仓库**（A 股 + 港股 + 多 agent + 中文 + 国产 LLM）
- ✅ Docker 部署契合 OpenClaw 承载
- ⚠️ 只出报告，**不自动触发提醒**—— 需要自己加个 cron + 阈值 + Telegram 层
- ⚠️ v2.0 闭源 —— fork 需锁定 v1.x 或自己维护
- ⚠️ 单次分析 LLM 成本较高（10-20 次 call） —— 对波段（周频调仓）可接受

### FinMem（论文级代表）[6][7]

**论文**: arXiv:2311.13743

**三模块**: Profiling（人格）/ Memory（分层）/ Decision-making。

**记忆分层**（论文细节，repo 未列举完整）:
- Working memory
- Short-term
- Mid-term
- Long-term
- Critical：可调 cognitive span 超人类上限

**实验**:
- 默认 Ticker: **TSLA**（2022-06-30 ~ 2022-10-11）
- 其他 ticker 如 NFLX 在论文里
- LLM：GPT-4 / HuggingFace TGI / Gemini
- Embedding：`text-embedding-ada-002` (OpenAI)

**Character**（`character.png`）:
- 基于风险态度的 prompt 分层（risk-seeking vs risk-averse）
- 具体 prompt 在 `config/config.toml`

**模式**: `train`（填充记忆）+ `test`（决策）两步。

**A 股适配**: 可行但工作量大（数据管道 + 中文新闻 + T+1/涨跌停）。

### FinAgent
与 FinMem / TradingAgents 同生态。未在本次 fetch 中深入。

### ai-hedge-fund (virattt, 57.2k star) [10]

**19 个 agent**:

| 13 投资人人格 | 功能 |
|---|---|
| Buffett / Munger / Graham / Ackman / Burry / Lynch / Fisher / Druckenmiller / Damodaran / Wood / Pabrai / Taleb / Jhunjhunwala | 每人一种投资哲学 |

| 6 功能 agent | 功能 |
|---|---|
| Valuation / Sentiment / Fundamentals / Technicals / Risk Manager / Portfolio Manager | 模块化信号融合 |

**数据源**: Financial Datasets API（美股为主）。**A 股/HK 完全不支持**，需 fork 重写数据层。

**LLM**: OpenAI / Anthropic / Groq / DeepSeek / Ollama。

**本项目评估**:
- ❌ 数据层硬编码美股，改造成本 > TradingAgents-CN
- ✅ 投资人人格 prompt 设计值得借鉴

### 数据需求
新闻 API（东财 / 新浪 / 财联社 / Twitter / Reddit）+ 基本面 + 行情。

### 算力
低（纯调 LLM）；token 成本是主要支出。

### 过拟合风险
**低**（无训练）；但 LLM 幻觉是风险。

### 适合用户
- ✅ 有研报式分析需求、愿意付 LLM 费
- ✅ 波段 + 周/月调仓 + 中级用户：**TradingAgents-CN 是准开箱方案**
- ❌ 高频 / 分钟级

---

## 路线 D：LLM 因子挖掘（Alpha-GPT / QuantAgent）

### Alpha-GPT 1.0 [8]

**团队**: HKUST / IDEA (Saizhuo Wang, Hang Yuan, Leon Zhou, Lionel M. Ni, Heung-Yeung Shum, Jian Guo)

**场景**: EMNLP 2025 System Demonstration Track

**机制**: 人机协作的 alpha 挖掘
- 研究员用自然语言描述"想法"
- Alpha-GPT 通过 prompt 工程理解意图
- 输出创造性、有效的 alpha 表达式
- 对传统遗传编程（gplearn / AutoAlpha）的替代

**痛点**: **无公开代码** / 无 IC/ICIR/Sharpe 数字 / 市场未知。

### Alpha-GPT 2.0 [9]

**arXiv**: 2402.09746

**扩展**: 从"alpha 挖掘"延伸到"quantitative investment pipeline 全流程"。
Human-in-the-Loop 贯穿始终。

**痛点**: 仍无公开代码（"Draft. Work in progress."），无具体数字。

### FinGPT（严格说不是因子挖掘，是 LLM 微调）[5]

**定位**: 不是 BloombergGPT 那种预训练，而是 **LoRA 轻量微调** 开源 LLM。
- BloombergGPT 重训成本 ~$3M
- FinGPT 单次微调 **< $300**

**任务**（NLP 为主，**不是因子挖掘！**）:
- 情感分析（FinGPT v3.3 在 FPB 数据集上 F1=0.882 > GPT-4 的 0.833）
- 股价预测（FinGPT-Forecaster，输入 ticker + date + 新闻 → 下周方向）
- 关系抽取 / 实体识别 / 财经 Q&A
- RAG 增强分析

**开源模型**（HuggingFace `FinGPT/*`）:

| 模型 | 基座 | 任务 |
|---|---|---|
| fingpt-sentiment_llama2-13b_lora | Llama2-13B | 情感 |
| fingpt-forecaster_dow30_llama2-7b_lora | Llama2-7B | 预测 |
| fingpt-mt_llama2-7b_lora | Llama2-7B | 多任务 |
| fingpt-mt_falcon-7b_lora | Falcon-7B | 多任务 |
| fingpt-mt_chatglm2-6b_lora | ChatGLM2-6B | 多任务（中文） |
| fingpt-mt_qwen-7b_lora | Qwen-7B | 多任务（中文） |
| fingpt-mt_bloom-7b1_lora | Bloom-7B1 | 多任务 |
| fingpt-mt_mpt-7b_lora | MPT-7B | 多任务 |

**训练数据**:
- fingpt-sentiment-train: 76.8K
- fingpt-finred: 27.6K（关系抽取）
- fingpt-headline: 82.2K
- fingpt-ner: 511
- fingpt-fiqa_qa: 17.1K
- fingpt-fineval: 1.06K（**中文多选**）

**中国 A 股微调**:
- FinGPT V1 明确微调 ChatGLM2 / Llama2 for Chinese Market
- 有 fingpt-fineval 中文数据集
- **但没有 A 股票面情感专门模型**（需自己训）

**本项目评估**:
- ✅ 作为新闻/研报情感打标的**辅助 tool**（不是主策略）
- ✅ 中文金融情感可用 fingpt-mt_qwen-7b
- ❌ 不是"帮我构建策略"的核心路线（是 NLP 工具）

### 数据需求
大量历史因子值（OHLCV + 衍生）

### 算力
中等（遗传搜索 + LLM 评估）

### 过拟合风险
**极高** —— 因子挖掘本质就是在超大搜索空间里找"正样本"，bonferroni 类问题。必须用严格的 IC/ICIR/Rank IC 跨样本验证。

### 适合用户
- ✅ 高级量化（有专业 alpha 研究背景）
- ❌ **本项目用户（中级 + 规则派波段）不推荐**

---

## 四路线对比矩阵

| 维度 | A: LLM 规则 | B: RL | C: Multi-Agent | D: LLM 因子 |
|---|---|---|---|---|
| 代表工具 | Claude + Qlib/Backtrader | FinRL / Qlib-RL | **TradingAgents-CN** / FinMem | Alpha-GPT / FinGPT |
| 成熟度 | ✅ 即装即用 | ✅ 成熟 repo | ✅ CN 版本 24.6k star | ❌ 大多无代码 |
| A 股/HK 开箱 | ⚠️ 自接数据 | ✅ 多源 | ✅ CN 版全覆盖 | ⚠️ 自接 |
| 数据需求 | OHLCV + 基本面 | OHLCV + 技术指标 | OHLCV + 新闻 + 研报 | 海量历史因子 |
| 算力 | 🟢 CPU | 🟡 CPU 训几小时 | 🟢 纯 API 调用 | 🟡 中等 |
| LLM token 成本 | 低（调试期） | 零 | 高（持续 10-20 call/次） | 中（搜索期高） |
| 过拟合风险 | 中 | **高** | 低 | **极高** |
| 策略直觉可解释 | ✅ 高 | ❌ 黑箱 | ✅ 有辩论记录 | 🟡 因子式 |
| 波段适配 | ✅ | ⚠️ 复杂度不匹配 | ✅ | 🟡 |
| 中级用户可驾驭 | ✅ | ⚠️ 需 RL 基础 | ✅ | ❌ |

---

## 本项目路线推荐（用户画像：中级 + 波段 + A+HK + 纯提醒 + 港股可下单）

### 主推：路线 A + 路线 C 混合

**主干（路线 A，LLM 规则派）**:
```
用户自然语言描述 →
  Claude 生成 Backtrader 策略代码 →
  本地回测 →
  Claude 解读报告 + 建议调参 →
  迭代收敛 →
  包装成 OpenClaw skill →
  cron 每日跑 →
  触发 → Telegram
```

**增强（路线 C，Multi-Agent 研究员）**:
- 把 TradingAgents-CN 当"盘中研究助手"：每周末对候选股运行一次，出一份多角度研报
- 不依赖它做决策，但用它的 Bull/Bear 辩论作为"第二意见"
- Qwen/DeepSeek 做 LLM 后端控制成本（¥1-5/次/股）

### 次选（探索期再考虑）

**路线 B (FinRL)**: 如果路线 A 的规则派策略遇到天花板，且愿意投入 2-3 周学 RL，可以在 FinRL 上试组合优化。

**路线 D**: 除非你想做多因子量化选股，否则不考虑。

### 不推荐的组合

- **直接上 FinMem / ai-hedge-fund（美股版）**：数据层改造成本 > 自己基于 TradingAgents-CN 定制
- **想用 Alpha-GPT**：无开源代码，只能读论文参考思路

---

## 路线对比的关键 insight

1. **"AI 帮我构建策略"在今天有两种截然不同的含义**：
   - A/C：LLM 当"顾问/分析师/程序员"，人在环里做决策 → **适合中级用户**
   - B/D：AI 端到端自主训练策略 → **需要量化工程师背景**

2. **TradingAgents-CN 是本领域 2025-2026 最新 state-of-art**：
   - 它不是"学术玩具"，24.6k star 证明产品化程度
   - 但它**不做触发通知**—— 所以在本项目架构里是"研报生成器"而非"信号源"

3. **真正的信号源**要靠路线 A 手写的量化规则**：用 LLM 帮你写代码，但触发逻辑是硬规则。这样可解释 + 可调试 + 成本可控。

4. **路线 C 的成本陷阱**: 如果天天跑 TradingAgents 对全市场 4000 只股票做分析，一天 $2000 LLM 费。所以 C 要**限定在 watchlist（20-50 只）+ 低频（周/月）**。
