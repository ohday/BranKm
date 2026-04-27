## [2026-04-28] explore | Topic defined

Topic: AI 辅助选股助手（A 股 + 港股，Socratic 引导 + Top-down 候选池）
Slug: ai-stock-picker-hkcn
Audience: 新手可懂 + 能与 skill 对话
Mode: new

Key questions defined: 8
  - KQ1 选股方法论总览（五流派 × Top-down/Bottom-up）
  - KQ2 ★ 行业选择：怎么判断当前最值得关注的行业
  - KQ3 个股 12 维度体系 + 周期权重矩阵
  - KQ4 ★ A 股 vs 港股特色因子对比
  - KQ5 数据实现地图（12 维 × A/HK × 接口）
  - KQ6 ★ Socratic 引导问卷设计（stock-profile.yaml）
  - KQ7 候选池输出格式（分层池 / 打分卡 / 研报式）
  - KQ8 避坑清单与硬性否决项

Reused existing raw sources (from ai-swing-reminder-hkcn):
  - id 26: A+HK 数据源全景（直接复用）
  - id 27: 回测框架对比（取 Qlib Alpha158 部分）
  - id 28: AI 策略构建四路线（取路线 D + 路线 A 的 prompt 模式）
  - id 29: 开源仓库 Tier List（取 TradingAgents-CN 选股研报参考）
  - id 32: 合规红线（取风险提示文案部分）

Decisions made during explore:
  - skill 形态: Socratic 引导 + 候选池（不是打分卡，不是研报单只）
  - 与 ai-swing-reminder-hkcn: 独立项目，共享 raw/ 但不联动
  - 流派倾向: 用户无倾向，由 skill 通过问卷挖掘；主路径 Top-down
  - 持有周期: 三周期并行（长/中/波段），skill 运行时让用户选
  - 深度目标: 新手可懂 + 能与 skill 对话
  - 研究核心创新: KQ2 + KQ4 + KQ6（上一项目未覆盖）

## [2026-04-28] websearch | Research complete — 14 new raw sources, 8/8 questions covered

### 新增 raw sources（id 34-47）

| KQ | sources | 代表文件 |
|----|---------|----------|
| KQ1 流派总览 | 36, 43 | five-investment-styles-canslim-magic-formula.md |
| KQ2 ★行业选择 | 34, 38, 44 | china-merrill-clock + industry-prosperity-three-dim |
| KQ3 12 维度 | 35, 36, 46 | 12-dimension-stock-picking-framework.md |
| KQ4 ★A/HK 特色 | 38, 39, 40 | a-share-market-mechanics + hk-market-specifics |
| KQ5 数据落地 | 35, 41, 46 | akshare-stock-picker-interfaces-comprehensive.md |
| KQ6 ★Socratic 问卷 | 37, 47 | socratic-questionnaire-stock-profile-design.md |
| KQ7 输出格式 | 42 | stock-pool-tier-design-core-watch-candidate.md |
| KQ8 避坑清单 | 39, 45 | stock-picker-hard-veto-and-soft-warnings.md |

### 复用已有 raw（ID 26-33 from ai-swing-reminder-hkcn）
- ID 26 数据源全景 → KQ5 数据落地的 A/HK 数据源总表
- ID 27 回测框架 → KQ3 Qlib Alpha158 使用语境
- ID 28 AI 四路线 → KQ1 流派的 AI 实现（路线 A/D）
- ID 29 开源仓库 Tier → KQ7 输出格式（TradingAgents-CN 研报参考）

### 完整性核对

| KQ | WHAT | WHY | HOW | DEPTH | TRADE | CROSS | LINK | 状态 |
|----|------|-----|-----|-------|-------|-------|------|------|
| 1 | ✅ | ✅ | ✅ | ✅ (CANSLIM 7 要素 + Magic Formula 公式) | ✅ | ✅ (Fama-French 对照) | ✅ | complete |
| 2 | ✅ | ✅ | ✅ | ✅ (华宝三维 + 美林时钟 73% 准确率) | ✅ | ✅ | ✅ | complete |
| 3 | ✅ | ✅ | ✅ | ✅ (Alpha158 153 个公式 + 12 维度完整) | ✅ | ✅ | ✅ | complete |
| 4 | ✅ | ✅ | ✅ | ✅ (涨跌停板/ST/退市/T+0/卖空 20% 上限) | ✅ | ✅ | ✅ | complete |
| 5 | ✅ | ✅ | ✅ | ✅ (AkShare 接口级详细) | ✅ | ✅ | ✅ | complete |
| 6 | ✅ | ✅ | ✅ | ✅ (15 题 Cold Start + 完整 yaml schema) | ✅ | ✅ (C1-C5 对标) | ✅ | complete |
| 7 | ✅ | ✅ | ✅ | ✅ (3 层池数量 + 仓位分配 + 升降级规则) | ✅ | ✅ | ✅ | complete |
| 8 | ✅ | ✅ | ✅ | ✅ (A 股 10 条 + 港股 9 条硬否决 + M-Score 公式) | ✅ | ✅ | ✅ | complete |

**8/8 questions complete**. 总 raw sources 消耗 14 个（id 34-47）+ 5 个复用（26-29,32）= 19 个素材。

### 关键发现亮点

- 美林时钟**中国改良版** 73% 准确率 vs 传统版 40% —— 直接套美股套路在 A 股会错
- **Alpha158 = 153 个因子，不是 158**，且完全不含基本面
- 华宝三维景气度用**二阶导数**捕捉边际拐点（营收/净利/ROE 同比增速的环比增长率）
- 港股**卖空比例 20% 上限**是监管硬规则，达到即暂停申报
- **min(Capacity, Tolerance)** 是 C1-C5 方法论的核心原则，skill 问卷必须两维分测
- Beneish M-Score **TATA 对 A 股最敏感**；阈值需本土化至 -1.89 ~ -2.0

## [2026-04-28] wiki | ai-stock-picker-hkcn — Synthesis Complete
- Pages created: 12
- Sources used: 19 (14 new + 5 reused)
- Parts: A 心智模型 (1-3) / B 机制层 (4-7) / C 实施层 (8-11) / D 综合 (12)
- 每页含 Mermaid 可视化 + 脚注引用 raw note（Obsidian wiki-link 格式）
