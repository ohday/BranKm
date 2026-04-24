## [2026-04-24] explore | Topic defined

- Slug: `ai-swing-reminder-hkcn`
- 用户画像：中级量化水平，波段交易，A 股 + 港股双市场
- 核心需求：AI 帮我设计策略 → 阈值触发 → 推送提醒（A 股）/ 可选下单（港股）
- 承载层：OpenClaw（自托管 agent + IM 皮肤，已确认）
- 定义了 8 个 key questions，Q3（AI 帮构建策略的四路线对比）为核心
- 用户确认扩展范围：港股纳入合法下单 API 调研

## [2026-04-24] websearch | Research complete (+8 raw sources covering Q1-Q8)

所有 8 个 questions 均达到 "WHAT + WHY + HOW + DEPTH/TRADE/CROSS 至少其一" 的 complete 标准。

| Question | 完成 | 关键发现 |
|---|---|---|
| Q1 数据源对比 | ✅ | AkShare/Baostock 免费，Tushare 积分制，Futu 港股 LV2 免费 |
| Q2 回测框架 | ✅ | Qlib + Backtrader 双轨，vnpy 太重不选 |
| Q3 ★AI 四路线 | ✅ | 主推路线 A（LLM 规则）+ 辅路线 C（TradingAgents-CN） |
| Q4 开源仓库 | ✅ | Tier-1: TradingAgents-CN + Qlib + Backtrader + AkShare + Futu |
| Q5 信号分发 | ✅ | 轮询双 cron + 三级预警 + 多通道 |
| Q6 OpenClaw 承载 | ✅ | 香港单节点 VPS + SKILL.md + SQLite/Parquet/DuckDB |
| Q7 合规 + 港股 | ✅ | A 股纯提醒合规，港股 Futu/Longbridge 对比完整 |
| Q8 端到端模板 | ✅ | 7 步 pipeline，WF 验证配置 |

### Key insights

1. TradingAgents-CN 是全场最接近用户目标的现成仓库（A+HK+US / Docker / Qwen 优先）
2. 月开销 < ¥200 完全可达
3. 合规原则："对自己操作给出建议合法；代替别人需要持牌；让机器代替自己需要报备。本项目是第一种。"
4. 港股扩展首选 Futu（LV2 免费 + 社区最大），次选 Longbridge（无 daemon）
5. 路线 B/D 不适合中级 + 波段场景

## [2026-04-24] wiki | Synthesis complete — 10 pages created

### 页面清单

1. 项目总览与系统架构 —— 五层架构 + 三条核心决策
2. A股 + 港股数据源选型 —— Q1
3. 回测框架 Qlib 与 Backtrader 的分工 —— Q2
4. AI 构建策略的四条路线 —— Q3 核心
5. TradingAgents-CN 深度解析 —— Q3 落地
6. 开源仓库 Tier 清单 —— Q4
7. 信号分发工程架构 —— Q5
8. OpenClaw 承载方案 —— Q6
9. 合规红线与港股下单 API —— Q7
10. 端到端 Pipeline 模板 —— Q8

### 质量指标

- 总页面：10
- 总参考 raw 源：33（项目独占 8 个新增）
- 每页平均 Mermaid 图：5+
- 所有 key_question 均有专页回答
- 所有页面交叉引用 → index.md 闭合
- 所有 footnote 走 Obsidian wiki-link `[[raw-note]]` 格式

### 交付物

- `projects/ai-swing-reminder-hkcn/index.md`（总入口）
- `projects/ai-swing-reminder-hkcn/wiki/1..10.md`（10 个 wiki 页面）
- `projects/ai-swing-reminder-hkcn/.research-brief.yaml`（研究大纲）
- `raw/stock-data-sources-cn-hk.md` 等 8 个新增 synthesis 源
- `raw/sources.yaml` 追加 8 个注册
