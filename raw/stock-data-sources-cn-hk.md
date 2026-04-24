# A 股 + 港股数据源全景对比（AkShare / Tushare Pro / Baostock / Futu / Longbridge）

- **Type**: Synthesis (aggregated from multiple sources)
- **Fetched**: 2026-04-24
- **Project**: ai-swing-reminder-hkcn
- **Relevant to**: Q1
- **Sources drawn from**:
  1. [AkShare GitHub README (akfamily/akshare)](https://github.com/akfamily/akshare)
  2. [Tushare Pro 积分频次对应表 doc_id=290](https://tushare.pro/document/1?doc_id=290)
  3. [Tushare 社区综合信息（知乎/百度百家号）— 2025 积分与港股权限](https://baijiahao.baidu.com/s?id=1829031498578754242)
  4. [Tushare Pro 使用教程（Shimo）](https://shimo.im/docs/PGWdY6Q8xDKpvDXG)
  5. [Futu OpenAPI 官方文档 - Intro](https://openapi.futunn.com/futu-api-doc/en/intro/intro.html)
  6. [Futu OpenAPI 港股 LV2 免费说明（凤凰财经）](https://finance.ifeng.com/a/20161012/14932722_0.shtml)
  7. [港美股行情数据使用合规性分析 - 雪球](https://xueqiu.com/6083412391/233356058)
  8. [Longbridge OpenAPI 文档（官方）](https://open.longportapp.cn/zh-CN/docs/index)
  9. [Longbridge 通用问题（限频规则）](https://open.longbridge.com/zh-CN/docs/qa/general)
  10. Baostock 项目公开文档（社区整理）

---

## 1. AkShare（开源免费，A 股 + 港股 + 期货 + 宏观全覆盖）[1]

- **License**: MIT, 完全开源免费，无需注册/token。
- **覆盖广度**: 30+ 第三方数据源的 Python 包装器（东方财富、新浪财经、上交所/深交所/北交所、金十、和讯、SGX 新加坡交易所、中金所/上期所/大商所/郑商所/能源中心）。港股/A 股/美股/期货/期权/债券/宏观/另类数据都覆盖。
- **版本**: 截至 2026-04，826 commits / 178 releases（最新 v1.18.57，2026-04-23）/ 18.5k star / 3.1k fork。活跃度极高。
- **典型 A 股调用**:
  ```python
  ak.stock_zh_a_hist(symbol="000001", period="daily",
                     start_date="20170301", end_date="20231022",
                     adjust="qfq")  # "" 不复权, "qfq" 前复权, "hfq" 后复权
  # 返回 11 列：日期/开盘/收盘/最高/最低/振幅/涨跌幅/涨跌额/换手率/成交量/成交额
  ```
- **限制**: 官方 README 明确声明 "Based on some uncontrollable factors, some data interfaces in AKShare may be removed" —— 依赖上游网站，网站改版会破坏接口，**稳定性风险是最大痛点**。
- **Python 要求**: 3.9+ 64-bit。阿里云镜像安装:
  ```bash
  pip install akshare -i http://mirrors.aliyun.com/pypi/simple/ --upgrade
  ```

## 2. Tushare Pro（积分制，商业化最成熟）[2][3][4]

### 积分获取方式

| 方式 | 积分 |
|---|---|
| 新用户注册 | 100 |
| 完善资料 | 20 |
| 学生/教师认证（绑定手机号 + 加入高校 QQ 群 819132361 / 1108906234） | **2000（免费）** |
| 捐助 ¥200（加入 QQ 群 621499723 / 870052128） | 2000 |
| 捐助 ¥500 | 5000 |
| 捐助 ¥1000 | 10000 |

### 积分等级与权限（官方表 `doc_id=290`）[2]

| 积分 | 每分钟调用 | 每日上限 | 访问范围 | 价格/年 |
|---|---|---|---|---|
| 120 | 50 | 8,000 | **仅不复权日线**，其它接口不可用 | ¥0（免费） |
| 2000+ | 200 | 10 万/API | 按各 API 文档声明的积分门槛 | ¥200 |
| 5000+ | 500 | 常规数据无限 | 同上，范围更广 | ¥500 |
| 10000+ | 500 | 常规无限；"特色数据"300/分钟 | 加送：盈利预测、每日筹码分布、胜率、券商金股 | ¥1000 |
| 15000+ | 500 | 特色数据无总量限制 | 独占高级数据 | ¥1500 |

### 独立权限接口（积分不够的单独买）

| 数据 | 价格 | 限频 |
|---|---|---|
| A 股历史分钟（1/5/15/30/60m） | ¥2000/年 | 500/min, 8000 行/次 |
| A 股实时分钟 | ¥1000/月 | 500/min, 300 股/次 |
| A 股实时日线 | ¥200/月 | 50/min |
| **港股日线 + 复权** | **¥1000/年** | 500/min, 6000 行/次 |
| **港股分钟（2015 起）** | **¥2000/年** | 500/min, 8000 行/次 |
| **港股财务（2000 起）** | **¥500/年** | 500/min, 10000 行/次 |
| **港股实时日线** | **¥1000/月** | 50/min, 全市场 |
| 期货/期权历史分钟 | ¥2000/年 | 500/min |

**企业用户 10 倍个人价**。捐助后转阿里云充值，**不支持退款**。[2]

### Python SDK
```python
pip install tushare -i https://pypi.tuna.tsinghua.edu.cn/simple
import tushare as ts
ts.set_token('你的token')  # 个人中心 → 获取 Token
pro = ts.pro_api()
df = pro.daily(ts_code='000001.SZ', start_date='20230101', end_date='20231231')
```

## 3. Baostock（完全免费，A 股 Only）[10]

- **License**: BSD, 完全开源，无 token，无账号，无限频（合理使用）。
- **安装/登录**:
  ```python
  import baostock as bs
  lg = bs.login()   # 匿名
  # ... 查询 ...
  bs.logout()
  ```
- **核心函数**: `query_history_k_data_plus` / `query_all_stock` / `query_stock_basic` / `query_stock_industry`（中信行业）/ `query_trade_dates` / 五张财务表（`query_profit_data` / `query_balance_data` / `query_cash_flow_data` / `query_growth_data` / `query_operation_data`）+ DuPont。
- **K 线字段**: `date, code, open, high, low, close, preclose, volume, amount, turn, tradestatus, pctChg, peTTM, pbMRQ, psTTM, pcfNcfTTM, isST`（选字段做字符串参数传入）。
- **复权**: `"1"`=后复权 / `"2"`=前复权 / `"3"`=不复权。历史最早 2006 年。
- **覆盖**: ✅ A 股沪深（日/周/月 K + 5/15/30/60min）/ ✅ 基础财务 / ✅ 中信行业 / ✅ 交易日历；**❌ 港股 / ❌ 美股 / ❌ 期货期权**。
- **痛点**: 服务器偶尔宕机；盘后数据晚上才更新（不适合实时）；维护节奏慢。

## 4. Futu OpenAPI（富途，港股首选，免费 LV2）[5][6][7]

### 架构（关键！）

双进程：
- **OpenD** 本地 daemon（TCP gateway，必须先装并用 Futu 账号密码登录），支持 Windows / macOS / CentOS / Ubuntu。
- **SDK** (`futu-api`) 通过 TCP 与 OpenD 通信。语言无关（Python / Java / C# / C++ / JavaScript）。

### 行情权限（核心优势）

| 行情 | 延迟 | 费用 |
|---|---|---|
| BMP | 延迟 15min, 手动刷新 | 免费 |
| LV1 | 实时，最佳买卖 1 档 | 中国大陆个人**免费** |
| **LV2** | **实时 10 档 + 经纪队列 + 逐笔** | **中国大陆个人免费**（富途每年自费 1000 万港币承担成本） |

对比同行：耀才 ~200 港币/月、致富 ~868 港币/月（豁免需佣金达标）。**Futu 是港股量化的行情成本洼地。**

### 交易能力

| 市场 | 行情 | 交易（真实） |
|---|---|---|
| HK 股票/ETF/权证/牛熊证/窝轮 | ✅ | ✅ FUTU HK |
| HK 期权（含指数期权） | ✅ | ✅ |
| HK 期货 | ✅ | ✅ |
| 美股股票/ETF/期权/期货 | ✅ | ✅（Moomoo US/SG/CA） |
| A 股（沪深港通标的） | ✅ | ✅（港股通） |
| A 股（非港股通） | ✅ | ❌（无真实交易通道） |

### 声明指标

- "最快下单 0.0014s"
- "通过 Futu API 交易无附加费"
- 直连交易所

### 注意

Intro 页未列订单类型细节、限频数字、SDK 具体函数（在 Quick Start / Trade API 子页）。**Python**:
```bash
pip install futu-api
```

## 5. Longbridge OpenAPI（长桥，港美股交易 API 新锐）[8][9]

### 市场与订单

| 市场 | 覆盖 |
|---|---|
| 港股 | 股票/ETF/权证/牛熊证 |
| 美股 | 股票/ETF/期权 |
| A 股 | 部分（港股通） |

### 订单类型

- **市价单** (Market Order)
- **限价单** (Limit Order)
- **条件单 / 到价单** (Stop Order) —— 触发价 + 限价/市价执行
- **追踪止损** (Trailing Stop) —— 触发价随市场动态调整

### 代码示例（关键）

```python
from longbridge.openapi import TradeContext, Config, OrderSide, OrderType, TimeInForceType

config = Config.from_env()      # 读 LONGPORT_APP_KEY / LONGPORT_APP_SECRET / LONGPORT_ACCESS_TOKEN
ctx = TradeContext(config)

# 限价买 100 股腾讯 @ 300 HKD
ctx.submit_order(
    symbol="0700.HK",
    order_type=OrderType.Limit,
    side=OrderSide.Buy,
    submitted_price=300.0,
    submitted_quantity=100,
    time_in_force=TimeInForceType.Day,
)

# 条件单（到价 310 触发限价买）
ctx.submit_order(
    symbol="0700.HK", order_type=OrderType.Limit, side=OrderSide.Buy,
    submitted_price=310.0, submitted_quantity=100,
    time_in_force=TimeInForceType.Day,
    trigger_price=310.0, trigger_at="last"
)
```

### Rate Limit（非常严，必须自己做限流）

**TradeContext（交易）**:
- 30 秒累计 ≤ 30 次
- 两次调用间隔 ≥ 0.02s
- 多账户独立计频
- 超限返回错误码 `301606`
- **SDK 未内置交易限频，开发者需自己 `time.sleep(0.05)` 或队列**

**QuoteContext（行情）**:
- 1 秒 ≤ 10 次
- 并发数 ≤ 5
- **SDK 已内置行情限频逻辑**（自动处理）

### 认证
环境变量三元组：`LONGPORT_APP_KEY` + `LONGPORT_APP_SECRET` + `LONGPORT_ACCESS_TOKEN`。Token 需要定期刷新。无本地 daemon（相比 Futu 更轻量）。

---

## 按用途选源决策表（生产建议）

| 用途 | 首选 | 备选 | 理由 |
|---|---|---|---|
| A 股日线历史回测（5~10 年） | **Baostock** | AkShare | 免费无 token，字段够用，稳定 |
| A 股基本面/财务/行业 | **AkShare** | Tushare Pro ¥200 档 | AkShare 覆盖广；Tushare 更结构化 |
| A 股北向/龙虎榜/分钟线 | **Tushare Pro ¥200~500** | AkShare（稳定性略差） | 商业化数据质量更高 |
| **港股日线历史** | **AkShare** | Tushare Pro ¥1000/年 | AkShare 免费够用；重度用户买 Tushare |
| **港股实时行情** | **Futu OpenAPI** | Longbridge | LV2 免费是杀手锏 |
| 港股实盘下单 | **Futu OpenAPI** 或 **Longbridge** | Tiger | 看账户生态偏好 |
| 新闻/研报 | AkShare（东财/新浪） | Tushare Pro ¥500+ | 开源免费够 60% 用户 |

## 主推组合（本项目目标场景：波段 + A+HK 双市场 + 中级开发者 + 月开销 < ¥500）

```
┌──────────────────────────────────────────────┐
│  回测/历史数据层：                            │
│    Baostock (A 股日线 + 财务, 免费)          │
│    + AkShare (A 股辅助字段 + 港股日线, 免费) │
│                                              │
│  实时行情 + 港股实盘层：                     │
│    Futu OpenAPI (港股 LV2 免费 + 下单 + 提醒) │
│                                              │
│  可选增强（¥200~500/年）：                    │
│    Tushare Pro 2000~5000 积分                │
│    → 解锁北向资金 / 龙虎榜 / 分钟 K 等       │
└──────────────────────────────────────────────┘
```

**月开销**: ¥0 起步，¥17~42/月（Tushare Pro 年费分摊）增强。完全满足 <¥500/月 的约束。

## 稳定性排名（从高到低）

1. **Longbridge / Futu** —— 商业化 API，SLA 清晰
2. **Tushare Pro 付费档** —— 专用数据库，积分限频但接口稳
3. **Baostock** —— 单服务器，偶尔宕机但规模可控
4. **Tushare 免费档** —— 120 积分限制严，生产不要依赖
5. **AkShare** —— 最脆弱（依赖 30+ 上游站点），但覆盖最广

## 部署约束

- **大陆 VPS（阿里云/腾讯云）**：AkShare 爬东财/新浪通畅；Tushare 官方走 CDN 稳；Futu OpenD 可运行但要能连富途服务器。
- **海外 VPS（香港/新加坡）**：Futu/Longbridge 原生更稳；AkShare 爬国内站可能被墙，要走代理；Baostock 实测一般可连。
- **双节点推荐**：A 股任务 → 大陆节点；港股实盘 → 香港节点；两节点用 VPN/Tailscale 打通。
