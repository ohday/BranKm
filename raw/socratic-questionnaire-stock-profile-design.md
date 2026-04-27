# Socratic 引导问卷设计（stock-profile.yaml）

- **Type**: Synthesis (aggregated from multiple sources)
- **Fetched**: 2026-04-28
- **Project**: ai-stock-picker-hkcn
- **Relevant to**: KQ6
- **Sources drawn from**:
  1. [Betterment 引导问卷](https://www.betterment.com) (via Seeking Alpha review)
  2. [Wealthfront 引导问卷](https://www.wealthfront.com) (via Forbes / CGAA review)
  3. [证监会《证券期货投资者适当性管理办法》C1-C5](https://www.csrc.gov.cn/)
  4. [晨星投资风格箱 晨星中国](https://www.morningstar.cn/article/AR00000882)
  5. [Kahneman & Tversky Prospect Theory (行为金融基础)](https://psycnet.apa.org/record/1979-24396-001)
  6. [Forbes: Evaluating Client Needs - Regulators vs Robo-Advisers](https://www.forbes.com/sites/marcgerstein/2016/01/27/evaluating-client-needs-regulators-1-robo-advisers-0/)

---

## 一、stock-profile.yaml 完整字段设计

```yaml
# ========================================
# Stock Profile v1.0
# 由 Cold Start 问卷生成；每 N 次推荐后由行为校准更新
# ========================================

# ── 基础信息（必填） ──
user:
  first_use_date: 2026-04-28
  last_update: 2026-04-28
  version: 1.0

# ── 风险承受能力 (对标证监会 C1-C5) ──
risk:
  level: C3                # C1 保守/C2 谨慎/C3 稳健/C4 积极/C5 激进
  capacity: C4             # 客观承受力（资产/年龄/收入）
  tolerance: C3            # 主观偏好（情绪/经验）
  final: C3                # 取 min(capacity, tolerance)
  max_drawdown_tolerance: 0.20   # 最大可接受回撤 20%

# ── 投资目标 ──
goal:
  primary: "steady_growth" # capital_preservation/steady_growth/aggressive_growth/income
  horizon: medium          # short_term(<1y) / medium(1-3y) / long_term(3y+)
  liquidity_need: low      # 流动性需求
  target_return_annual: 0.15

# ── 流派倾向（推断得分，非用户自选） ──
style:
  value: 0.6               # 0-1
  growth: 0.7
  quality: 0.8
  dividend: 0.3
  momentum: 0.4
  # —— 基于上述推断 ——
  dominant: ["quality", "growth"]
  weighting_profile: "blended_growth"

# ── 持有周期偏好 ──
horizon_preference:
  long_term: 0.6   # 核心池占比
  mid_term: 0.3
  swing: 0.1

# ── 市场偏好 ──
market:
  a_share: 0.7
  hk: 0.3
  exclude_boards: []       # e.g., [北交所]
  exclude_industries: []   # 黑名单
  prefer_industries: []    # 白名单

# ── 排除项（硬性过滤）──
exclusions:
  moral:                   # e.g., 烟草/博彩
    - tobacco
  industry_blacklist: []
  stock_blacklist: []      # 具体股票代码

# ── 偏好风格（辅助） ──
preferences:
  market_cap: ["large", "mid"]  # large/mid/small
  st_tolerance: false
  new_listing_max_age_days: 180
  min_market_cap_hkd: 5e9
  min_market_cap_cny: 5e9

# ── 行为特征（用于校准）──
behavior:
  loss_aversion: 2.5       # 损失厌恶系数（典型 2-3）
  overconfidence: low      # low/mid/high
  anchor_on_entry: true    # 是否被入场价锚定
  reaction_to_20pct_drop: "hold"  # panic_sell/hold/add

# ── 输出偏好 ──
output:
  pool_size_core: 5
  pool_size_watch: 15
  pool_size_candidate: 30
  show_target_price: false       # 目标价（新手建议关闭）
  show_stop_loss: true
  detail_level: "newbie"         # newbie/intermediate/expert
  preferred_language: "zh-CN"

# ── skill 交互偏好 ──
interaction:
  weekly_review: true
  socratic_frequency: "monthly"  # 多久重新问卷
  alert_channel: "none"          # 本 skill 不做提醒
```

---

## 二、Cold Start 问卷（首次使用，15 题）

### 第一部分：基础画像（5 题）

**Q1**（投资经验）
> 你投资股票的时间是？
> - 🌱 从未投资过
> - 🌿 少于 1 年
> - 🌳 1-3 年
> - 🌲 3 年以上

**Q2**（资金规模）
> 你准备投入股市的资金占你总资产的比例？
> - 一小部分（< 10%，娱乐金）
> - 相当部分（10-30%，投资金）
> - 大部分（30-60%，重要投资）
> - 几乎全部（> 60%，all-in）

**Q3**（投资目标）
> 你投资股票的主要目的是？
> - 跑赢存款就行（保值）
> - 希望有稳定现金流（比如每年 5% 分红）
> - 追求资产长期增值（年化 10-20% 能接受）
> - 想博取高回报（愿意承担翻倍和腰斩两种可能）

**Q4**（持有期限）
> 你准备把这笔钱投入多久？
> - 几个月内就要用
> - 1-3 年
> - 3-5 年
> - 5 年以上

**Q5**（既有知识）
> 下列哪些名词你能用一句话解释？（多选）
> ☐ PE / 市盈率
> ☐ ROE / 净资产收益率
> ☐ 均线 / MA
> ☐ PEG
> ☐ 自由现金流
> ☐ 龙虎榜
> ☐ 港股通 / 沪股通
> ☐ IC / IR

### 第二部分：风险承受（4 题）—— 场景化

**Q6**（最大亏损容忍）
> 假设你投入了 10 万元，**一个月后账户只剩 8 万**（亏损 20%）。你会：
> - 😱 立即全部卖出（我受不了）
> - 😐 继续持有但不再加仓
> - 🤔 仔细分析，若逻辑没变可能加仓
> - 😎 毫不动摇，这正是加仓机会

**Q7**（长期回撤）
> 假设你投入的股票未来 3 年期间，**有一次跌了 30%**，但 3 年整体赚了 50%。你会：
> - 完全受不了 30% 的过程
> - 痛苦但能坚持
> - 习惯就好
> - 这是正常波动

**Q8**（浮亏 vs 浮盈）
> 你发现你的一只股票 **从 100 元涨到 130 元又跌回 110 元**，你会：
> - 懊恼没在 130 卖
> - 感觉还行，毕竟赚着
> - 关注未来走势，不纠结过去
> - 这是锚定效应我知道

**Q9**（错失恐惧 FOMO）
> **某热门股票你没买，然后它涨了 3 倍**，你会：
> - 追进去（可能被高位套牢）
> - 等回调再说
> - 继续看自己的
> - 理性分析是否还有继续上涨逻辑

### 第三部分：偏好（4 题）

**Q10**（流派倾向 - 情景选择）
> 在下面 3 只假想的股票中，你最想买哪只？
> - A. PE 10 倍，ROE 12%，股息率 5%，过去 5 年零增长（蓝筹稳定派）
> - B. PE 40 倍，ROE 25%，股息率 0%，过去 3 年营收翻了 3 倍（高成长派）
> - C. PE 25 倍，ROE 20%，股息率 2%，过去 5 年复合增长 20%（质量派）

**Q11**（市场偏好）
> 你对 A 股和港股的偏好是？
> - 只想买 A 股（语言/信息熟悉）
> - A 股为主，偶尔港股
> - A + HK 均衡
> - 港股为主（估值更便宜）

**Q12**（排除项）
> 你有没有明确不想碰的行业？（多选）
> ☐ 白酒
> ☐ 房地产
> ☐ 银行
> ☐ 煤炭/钢铁/有色
> ☐ 烟草/博彩
> ☐ 军工
> ☐ 互联网平台
> ☐ 没有特别排除的

**Q13**（市值偏好）
> 你更偏好？
> - 只碰大白马（市值 > 1000 亿）
> - 大+中盘（市值 > 200 亿）
> - 不限市值，只看基本面
> - 喜欢小盘股弹性

### 第四部分：输出偏好（2 题）

**Q14**（推荐数量）
> 你希望 skill 每次推荐多少只股票？
> - 3-5 只（聚焦深度）
> - 5-10 只
> - 10-20 只（多样化）

**Q15**（解释详细度）
> 你希望每只股票的推荐理由？
> - 一句话够了（不想看长报告）
> - 三五句话解释（核心理由 + 风险）
> - 一篇研报（数据详实，带历史对比）

---

## 三、日常精简版问卷（5 题）

首次使用后每月触发一次：

1. 最近你的风险偏好有变化吗？（更保守/不变/更激进）
2. 本月行情你最想买的行业是哪个？（或"跟 skill 推荐"）
3. 过去 1 个月你对 skill 的哪条推荐 **印象最深**（好或不好）？
4. 你对持有期望是？（短线变多/不变/长线变多）
5. 本周有想加入黑名单的股票或行业吗？

---

## 四、行为校准机制

skill 不只听用户的嘴（问卷），也看用户的行动（反馈）。每次推荐后收集：

| 用户动作 | profile 更新 |
|---------|------|
| 点"加入核心池"某票 | 该票 style 向量 → 加权到用户 `style.*` |
| 点"拒绝推荐"且给理由 | 若理由是"估值太贵" → 用户 `value` 权重 +0.05 |
| 点"这只我已买" | 记录实际持仓，与推荐对比 |
| 连续 3 次忽略某行业推荐 | 该行业加入 `exclude_industries` |
| 用户主动搜索某行业 | `prefer_industries` +0.1 |

### 双维度判定（C1-C5 方法论）

最终风险等级 = min(Capacity, Tolerance)

- **Capacity 计算**（客观）：
  - 年龄（< 30 + 1 级，> 60 -1 级）
  - 投资占总资产比例（< 10% + 1 级，> 60% -1 级）
  - 投资经验（> 3 年 + 1 级）
- **Tolerance 计算**（主观）：
  - Q6/Q7/Q8/Q9 加权得分

---

## 五、关键问题设计原则（合规 + 反偏差）

1. **场景化优于直接问**："20% 亏损怎么办" > "你风险偏好是什么"
2. **不要诱导答案**：选项顺序打乱，不要从"保守"到"激进"单调排列
3. **避免"权威答案"**：不要让用户觉得某个选项是"正确的"
4. **合规**：必须包含"投资有风险，本 skill 仅供学习"
5. **非 KYC 金融**：本 skill 非持牌机构，问卷只驱动"推荐"而非"销售产品"
6. **隐私**：问卷结果仅本地存储（stock-profile.yaml）

---

## 六、Robo-Advisor 借鉴（Betterment / Wealthfront）

### Betterment 简约派（8 题）
- How much to invest
- Why invest (retirement/savings)
- When to cash out
- What would you do if portfolio lost 10% in a month
- Employment status
- Target allocations

### Wealthfront 人口统计派（10 题）
- Age（强影响风险画像）
- Primary reason for investing
- What are you looking for（diversification/tax/...）
- Annual after-tax income
- Total cash + liquid investments
- Risk tolerance via scenarios

### 对本 skill 的取舍

- 不问资产总额（隐私）
- **强化情景化问题**（Q6-Q9 四题）
- 本 skill **不做组合配置**，只做选股 → 问题更聚焦个股偏好
- **不问年龄**（避免偏见 "年轻 = 激进"）

---

## 七、与 12 维度 + 5 流派的联动

```
stock-profile.yaml
      │
      ├── style.* → 激活 12 维度权重（本文档第二章）
      │
      ├── horizon_preference → 激活周期×维度矩阵
      │
      ├── market + exclusions → 股票池过滤
      │
      ├── behavior.loss_aversion → 
      │         调整 D11 风险维度权重
      │
      └── output.* → 调整输出格式与详细度
```
