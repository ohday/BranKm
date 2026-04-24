# 端到端闭环：从自然语言偏好 → 上线提醒 的完整 pipeline

- **Type**: Synthesis
- **Fetched**: 2026-04-24
- **Project**: ai-swing-reminder-hkcn
- **Relevant to**: Q8
- **Sources drawn from**:
  1. [QuantInsti Walk-Forward Optimization 教程](https://blog.quantinsti.com/walk-forward-optimization-introduction/)
  2. [Qlib workflow_by_code 教程](https://github.com/microsoft/qlib/tree/main/examples/tutorial)
  3. [TradingAgents-CN 部署教程](https://blog.csdn.net/benshu_001/article/details/149693611)
  4. [OpenClaw Skill 开发教程](https://2048ai.net/69a28da454b52172bc5e1758.html)
  5. 前 7 个问题的综合产出

---

## 总览 Pipeline（7 步）

```
Step 1: 自然语言偏好描述
  ↓
Step 2: Claude 翻译为因子/规则代码
  ↓
Step 3: Qlib / Backtrader 回测 + Claude 解读
  ↓
Step 4: 样本外 / Walk-Forward 验证
  ↓
Step 5: 封装 OpenClaw skill + Cron
  ↓
Step 6: 港股可选：提醒 → Telegram 确认 → Futu 下单
  ↓
Step 7: 持续复盘 + Claude 微调
```

---

## Step 1：自然语言偏好描述（用户主导）

### Prompt 模板

```
我是中级量化用户，想做波段策略（持仓 3-20 日）。

【市场】A 股（300 股沪深 300 成分） + 港股（恒生科技 50）
【风格】低波动 + 趋势跟随
【偏好规则】
  - 20 日均线上方
  - 过去 5 日内首次站上 60 日均线
  - 成交量较 20 日均量放大 1.5 倍以上
  - 当日涨幅 < 7%（避免追涨）
  - RSI(14) 在 40-70 之间（不超买）
【风控】
  - 单票仓位 ≤ 10%
  - 组合最多持仓 8 只
  - 止损 -7% 硬规则
  - 止盈 +15% 或 20 日回吐到 10 日均线
【交易频率】
  - 最多每周换 2 次
  - 每笔持仓 ≥ 3 日
【目标指标】
  - 夏普 > 1.2
  - 最大回撤 < 15%
  - 年化换手率 < 400%

请基于以上偏好，生成：
1. Backtrader 策略类代码
2. 因子计算代码
3. 5 年回测脚本（用 AkShare 数据）
4. 报告输出格式
```

### 关键要点

- **量化到数字**: "低波动" → "过去 60 日波动率 < 25%"。不量化的偏好 LLM 会发挥得不好。
- **规则排列组合**: 规则之间用 `AND` / `OR` 明确
- **风控先于信号**: 先定清楚风控边界，再谈触发条件
- **目标指标**: Claude 会在调优时用这些作锚点

---

## Step 2：Claude 生成因子/规则代码

### Claude 输出的典型结构

```python
# strategy_v1.py
import backtrader as bt

class SwingTrendStrategy(bt.Strategy):
    params = (
        ('sma_short', 20),    # 短均线
        ('sma_long', 60),     # 长均线
        ('volume_multiplier', 1.5),
        ('rsi_period', 14),
        ('rsi_low', 40),
        ('rsi_high', 70),
        ('max_positions', 8),
        ('position_size', 0.10),
        ('stop_loss', 0.07),
        ('take_profit', 0.15),
    )

    def __init__(self):
        self.sma_short = {d: bt.indicators.SMA(d.close, period=self.p.sma_short)
                          for d in self.datas}
        self.sma_long = {d: bt.indicators.SMA(d.close, period=self.p.sma_long)
                         for d in self.datas}
        self.rsi = {d: bt.indicators.RSI(d.close, period=self.p.rsi_period)
                    for d in self.datas}
        self.vol_avg = {d: bt.indicators.SMA(d.volume, period=20) for d in self.datas}
        self.cross = {d: bt.indicators.CrossOver(d.close, self.sma_long[d])
                      for d in self.datas}

    def next(self):
        held = len([d for d in self.datas if self.getposition(d).size > 0])

        for d in self.datas:
            pos = self.getposition(d)
            price = d.close[0]

            # ---- 开仓 ----
            if not pos.size and held < self.p.max_positions:
                cond_trend = price > self.sma_short[d][0]
                cond_breakout = self.cross[d][0] > 0  # 今日首次站上长均线
                cond_volume = d.volume[0] > self.p.volume_multiplier * self.vol_avg[d][0]
                cond_rsi = self.p.rsi_low <= self.rsi[d][0] <= self.p.rsi_high
                cond_no_chase = d.close[0] / d.close[-1] - 1 < 0.07

                if all([cond_trend, cond_breakout, cond_volume, cond_rsi, cond_no_chase]):
                    cash_per_trade = self.broker.get_value() * self.p.position_size
                    qty = int(cash_per_trade / price / 100) * 100  # A 股 100 股为一手
                    if qty > 0:
                        self.buy(data=d, size=qty)

            # ---- 止损 / 止盈 ----
            elif pos.size > 0:
                cost = pos.price
                pnl_pct = price / cost - 1
                if pnl_pct <= -self.p.stop_loss:
                    self.sell(data=d, size=pos.size)
                elif pnl_pct >= self.p.take_profit:
                    self.sell(data=d, size=pos.size)
                elif price < self.sma_short[d][0]:  # 跌破短均线止盈
                    self.sell(data=d, size=pos.size)
```

### Review Checklist（人工 review LLM 输出）

- [ ] **规则逻辑与意图一致**（最容易错：>/>=/</<= 边界）
- [ ] **无未来函数**（用当前 bar 的收盘判断下一 bar 开盘买入？）
- [ ] **仓位管理**：`size` 是股数（必须整百股） vs 金额
- [ ] **交易成本**: 佣金 ~ 0.03%、印花税（A 股 0.1%，卖方）、过户费、港股印花税 0.1%
- [ ] **复权**: 用前复权价（`adjust="qfq"`）
- [ ] **无前视偏差**: 今日信号只能用今日及之前的数据

---

## Step 3：回测 + Claude 解读

### 回测脚本

```python
# backtest.py
import backtrader as bt
import akshare as ak
import pandas as pd
from strategy_v1 import SwingTrendStrategy

def load_data(symbol, start, end):
    df = ak.stock_zh_a_hist(symbol=symbol, period="daily",
                             start_date=start, end_date=end, adjust="qfq")
    df = df.rename(columns={"日期":"date","开盘":"open","最高":"high",
                             "最低":"low","收盘":"close","成交量":"volume"})
    df["date"] = pd.to_datetime(df["date"])
    df.set_index("date", inplace=True)
    return df[["open","high","low","close","volume"]]

cerebro = bt.Cerebro()
cerebro.addstrategy(SwingTrendStrategy)

# 沪深 300 样本
for sym in CSI300_SAMPLE_LIST:  # 简单版：取 30 只流动好的
    df = load_data(sym, "20200101", "20251231")
    cerebro.adddata(bt.feeds.PandasData(dataname=df), name=sym)

cerebro.broker.setcash(1_000_000)
cerebro.broker.setcommission(commission=0.0003)  # 万三

# Analyzers
cerebro.addanalyzer(bt.analyzers.SharpeRatio, _name='sharpe', riskfreerate=0.03)
cerebro.addanalyzer(bt.analyzers.DrawDown, _name='dd')
cerebro.addanalyzer(bt.analyzers.Returns, _name='ret')
cerebro.addanalyzer(bt.analyzers.TradeAnalyzer, _name='trades')

results = cerebro.run()
strat = results[0]
print(f"Sharpe: {strat.analyzers.sharpe.get_analysis()['sharperatio']:.3f}")
print(f"Max DD: {strat.analyzers.dd.get_analysis()['max']['drawdown']:.2f}%")
print(f"Total Return: {(cerebro.broker.get_value() / 1e6 - 1) * 100:.2f}%")
```

### 将回测报告交给 Claude 解读

```
【回测报告】
- 时间范围：2020-01 到 2025-12（5 年）
- 标的：沪深 300 样本 30 只
- 总收益：68.4%
- 年化：10.9%
- 夏普：1.35 ✅
- 最大回撤：-14.8% ✅
- 胜率：52%
- 盈亏比：1.8
- 换手率：年 520% ⚠️ 超标
- 月度胜率：62%
- 最差单月：-8.2% (2022-04)

请分析：
1. 哪些 metric 超预期 / 不达标
2. 换手率超标的根本原因（调哪个参数能降）
3. -8.2% 那个月发生了什么（2022-04 上海疫情，市场剧烈下跌）
4. 参数敏感性：把 RSI 上限从 70 调到 65，模型会怎样？
5. 推荐 v2 版本的 3 个改进方向
```

Claude 的解读会基于统计学 + 市场常识给出具体建议，比如：
- "换手率高 → RSI 区间过松 → 频繁进出"
- "2022-04 -8.2% 看起来是系统性风险，建议加大盘趋势 filter（如沪深 300 低于 200 日均线则暂停开仓）"

### 参数敏感性分析

```python
# param_sweep.py
for sma_long in [50, 60, 70, 80]:
    for rsi_high in [65, 70, 75]:
        # ... 跑一次 backtest，收集 Sharpe/DD/Turnover
```

画成 heatmap 给 Claude 看，它会识别"参数平原" vs "参数峭壁"（峭壁意味着过拟合）。

---

## Step 4：样本外 + Walk-Forward 验证 [1]

### Walk-Forward 配置

```
训练窗（IS）: 5 年
测试窗（OOS）: 1 年
步长: 1 年

Cycle 1: train 2015-2019 → test 2020
Cycle 2: train 2016-2020 → test 2021
Cycle 3: train 2017-2021 → test 2022
Cycle 4: train 2018-2022 → test 2023
Cycle 5: train 2019-2023 → test 2024
Cycle 6: train 2020-2024 → test 2025
```

### Python 实现

```python
def walk_forward(data, train_years=5, test_years=1, step_years=1):
    results = []
    years = sorted(data.index.year.unique())
    start = years[0]
    end = years[-1]
    current = start

    while current + train_years + test_years - 1 <= end:
        train_end = current + train_years - 1
        test_start = train_end + 1
        test_end = test_start + test_years - 1

        in_sample = data[data.index.year.between(current, train_end)]
        oos = data[data.index.year.between(test_start, test_end)]

        best_params = grid_search(in_sample)
        oos_perf = run_backtest(oos, best_params)

        results.append({
            'train': f'{current}-{train_end}',
            'test': f'{test_start}-{test_end}',
            'params': best_params,
            'oos_sharpe': oos_perf['sharpe'],
            'oos_ret': oos_perf['return']
        })
        current += step_years

    return pd.DataFrame(results)
```

### 验证合格线

- [ ] 所有 OOS 周期 Sharpe > 0
- [ ] OOS 平均 Sharpe > IS 平均 Sharpe × 0.5（≥ 50% 保留率）
- [ ] 最差 OOS 周期回撤 < 20%
- [ ] Params 在相邻 cycle 之间连续（不跳跃 —— 跳跃意味着过拟合）

### 常见踩坑 [1]

1. **Look-Ahead Bias**: 全局标准化用了未来数据（e.g., `df.close.std()` 跨全量）→ 必须滚动计算
2. **Window Selection Bias**: 试了很多 train_years 组合选最好的 → 是更高级的过拟合
3. **Regime Lag**: WFO 对牛熊转换反应慢 → 要加大盘 regime filter
4. **Compute Cost**: ML 策略 WFO 成本高 → 日频策略多数还 OK
5. **WFO 过拟合 WFO 自己**: 反复调 WFO 配置直到漂亮 —— 先定好配置再看结果

---

## Step 5：封装 OpenClaw Skill

### 目录结构

```
~/.openclaw/skills/ai-swing-reminder/
├── SKILL.md
├── scripts/
│   ├── daily_scan.py           # Step 3 回测代码改造为单日扫描
│   ├── backtester.py           # Step 3 用于按需回测
│   ├── notify.py               # Step 5 推送
│   └── walk_forward.py         # Step 4 周末自动 WF 验证
├── references/
│   ├── watchlist.md            # 监控池
│   └── strategy-params.yaml    # 当前生产参数
└── assets/
    └── signal-template.md      # 推送模板
```

### SKILL.md

```yaml
---
name: ai-swing-reminder
description: 扫描 watchlist 的波段信号，触发时推送提醒。关键词：盘后扫描、每日信号、波段提醒
version: 0.1.0
metadata: {"openclaw":{"requires":{"bins":["python3"],"env":["AKSHARE_OK","WECOM_WEBHOOK","TELEGRAM_BOT_TOKEN","TELEGRAM_CHAT_ID"]}}}
---

# 波段提醒 Skill

## 用户调用方式
- `/ai-swing-reminder` → 立即扫描当前日
- `/ai-swing-reminder backtest --symbol 600519` → 对单股票跑回测
- `/ai-swing-reminder update-watchlist --add 300750` → 维护池
- Cron: 每日 15:10 / 16:10 自动运行

## 步骤（扫描模式）
1. 读 watchlist: `cat {baseDir}/references/watchlist.md`
2. 扫描: `python3 {baseDir}/scripts/daily_scan.py --watchlist {baseDir}/references/watchlist.md --params {baseDir}/references/strategy-params.yaml --output /tmp/signals-$(date +%Y%m%d).json`
3. 推送: `python3 {baseDir}/scripts/notify.py --input /tmp/signals-$(date +%Y%m%d).json --channels wecom,telegram`
4. 汇报：扫描总数 / 触发数 / 推送渠道 / 耗时

## 错误处理
- 数据源失败：AkShare 失败则 fallback Baostock（A 股）/ Futu OpenAPI（港股）
- 企业微信推送失败：走 Server 酱 fallback
- 所有失败写 `/tmp/openclaw-errors-$(date +%Y%m%d).log`，23:59 心跳检查
```

### Cron 注册

```bash
openclaw cron add --name "a-share-daily-scan" \
  --cron "10 15 * * 1-5" \
  --message "/ai-swing-reminder"

openclaw cron add --name "hk-daily-scan" \
  --cron "10 16 * * 1-5" \
  --message "/ai-swing-reminder market=hk"

openclaw cron add --name "weekend-research" \
  --cron "0 10 * * 6" \
  --message "/tradingagents-cn-watchlist-review"

openclaw cron add --name "weekly-walk-forward" \
  --cron "0 14 * * 0" \
  --message "/ai-swing-reminder walk-forward-check"

openclaw cron add --name "daily-heartbeat" \
  --cron "59 23 * * *" \
  --message "/heartbeat"
```

---

## Step 6：港股可选：提醒 → 确认 → 下单

### 设计原则

- **提醒永远是无状态的**（不依赖用户确认）
- **下单永远是有状态 + 交互式的**
- **双向交互走 Telegram**（IM 用户体验最好）

### 下单 Flow

```
15:10 scan → 信号生成 "0700.HK 突破" →
  notify.py 发送 Markdown：
    📈 *0700.HK* 腾讯控股 突破信号 L2
    当前价 315.4 (+2.1%)
    建议：买入 100 股 @ 315 限价
    仓位 8%（当前组合 45%）
    止损 310 / 止盈 340
    [⚠️ 30 分钟内有效]
    ──────────────
    /tc_BUY_0700_100_315
    /tc_IGNORE
  →

Telegram 推送 →
用户看到 →

[场景 A]: 直接打开富途 App 手动下单（无自动化）
[场景 B]: 回复 /tc_BUY_0700_100_315 →
  OpenClaw 接收 → 匹配 signal_id →
  二次确认："⚠️ 确认买入 100 股 0700.HK @ 315？回复 YES"
  用户: YES →
  调 Futu OpenAPI place_order →
  回报："✅ Order ID 202604241510001 已提交"
  5-10 秒后拉成交："✅ 部分成交 100@315.20" 或 "⏳ 挂单中"
```

### 注意事项

- **有效期**: 30 分钟是经验值（波段 + 港股盘时）
- **二次确认**: 避免误点
- **持仓检查**: 下单前拉最新组合，防"以为空仓但实际还持着昨天的单"
- **金额校验**: 现金足 + 港股 100 股最小交易单位
- **异常兜底**: Futu API 下单超时 → 立即查订单状态 → 状态未知时警报人工

---

## Step 7：持续复盘 + Claude 微调

### 周报机制

每周五 20:00，Claude 接管：

```
本周交易记录：
  ├── 触发信号 15 次
  ├── 推送 15 条 → 我手动下单 8 单
  ├── 盈利 5 单（最大 +12%）/ 亏损 3 单（最大 -8%）
  ├── 平均持有 4 天
  ├── 胜率 62.5%
  └── 本周收益 +3.2%

请分析：
1. 哪类信号质量最差（按 signal_id 分组看胜率）
2. 我漏单的 7 次事后看结果怎么样（overfitting 测试）
3. 是否有新的规律（比如周一信号总亏、尾盘信号总赢）
4. 下周 watchlist 建议调整
5. 策略参数是否该微调
```

### 月度 Re-calibration

```
月度跑一次 walk-forward，如果最近一个 OOS 周期：
  - Sharpe < 0.5 × 历史平均 → 策略进入低谷期
  - 考虑：降低信号阈值 / 缩小仓位 / 临时停机
  - 绝不：在历史 OOS 差的时候"硬扛" / 加杠杆
```

### Claude 的持续角色

| 频率 | 任务 | Prompt 模板 |
|---|---|---|
| 每日 | 解读当日触发信号 | "今日触发 X 单，帮我判断优先级" |
| 每周 | 交易复盘 | "本周交易记录（附 JSON），分析" |
| 每月 | 参数再校准 | "最新 WF 结果，是否调参" |
| 每季 | 市场风格分析 | "这个季度风格切换，策略还适用吗" |
| 年度 | 策略大迭代 | "这一年策略 vs 大盘，是否要 v2" |

---

## 每一 step 避坑清单

### Step 1 偏好描述
- ❌ "我想买涨的" → 太模糊
- ❌ "抄底" → 没有定义"底"
- ✅ 量化到数字 + 规则 + 风控

### Step 2 代码生成
- ❌ 直接抄 LLM 代码不 review
- ❌ 忽略复权、交易成本、T+1
- ✅ 对着 checklist 逐项过

### Step 3 回测
- ❌ 只看 Sharpe，不看最大回撤 / 换手 / 月度表现
- ❌ 单次回测就拍板
- ✅ 参数敏感性 + Claude 多角度解读

### Step 4 验证
- ❌ 过拟合 WFO 本身（反复调 WF 配置）
- ❌ 忽略 regime 切换
- ✅ 先定死 WF 配置

### Step 5 上线
- ❌ 上来就跑全市场
- ❌ 无心跳无巡检
- ✅ watchlist 从小池（30 只）起步

### Step 6 港股下单
- ❌ 一键全自动
- ❌ 忽略订单状态查询
- ✅ 永远人工确认

### Step 7 复盘
- ❌ 亏了就加杠杆
- ❌ 赚了就盲目扩仓位
- ✅ 让 Claude 当"冷静的参谋"

---

## 交付物总览

当这 7 步完成，你会拥有：

1. **一个 OpenClaw skill**（`ai-swing-reminder`）
2. **4 个 Cron 定时任务**（A 盘后 / 港盘后 / 周末研报 / 每日心跳）
3. **一套策略代码**（Backtrader / Qlib）+ 测试用例
4. **一份 WF 验证报告**（6 个 OOS 周期）
5. **推送通道配置**（企业微信 + Telegram + Bark）
6. **可选：Futu 下单 skill**（`/tc_*` 指令）
7. **周报 / 月度复盘** Claude prompt 模板

**月开销估算**:
- 腾讯云香港 VPS: $12/月
- Tushare Pro 2000 积分（可选）: ¥17/月
- LLM API（DeepSeek 为主，少量 Claude）: ¥50-100/月
- 合计: **< ¥200/月**

符合 <¥500/月 的约束。
