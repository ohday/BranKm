# 合规红线 + 港股下单 API 实战

- **Type**: Synthesis
- **Fetched**: 2026-04-24
- **Project**: ai-swing-reminder-hkcn
- **Relevant to**: Q7
- **Sources drawn from**:
  1. [证监会《证券市场程序化交易管理规定（试行）》 官方原文](https://www.csrc.gov.cn/csrc/c100028/c7480577/content.shtml)
  2. [国务院公报 证监会公告 2024 年 8 号](https://www.gov.cn/gongbao/2024/issue_11426/202406/content_6959680.html)
  3. [证监会：划定程序化交易监控"红线" 新华每日电讯](https://mrdx.cn/h5/mrdx/content/20240711/Articel05006GN.htm)
  4. [沪深北交易所实施细则 央广网](https://www.cnr.cn/jingji/gundong/20240610/t20240610_526737217.shtml)
  5. [上交所关于股票程序化交易报告工作的通知 百度百科](https://baike.baidu.com/item/%E5%85%B3%E4%BA%8E%E8%82%A1%E7%A5%A8%E7%A8%8B%E5%BA%8F%E5%8C%96%E4%BA%A4%E6%98%93%E6%8A%A5%E5%91%8A%E5%B7%A5%E4%BD%9C%E6%9C%89%E5%85%B3%E4%BA%8B%E9%A1%B9%E7%9A%84%E9%80%9A%E7%9F%A5/63403434)
  6. [财经网 证监会新规解读](https://www.financialnews.com.cn/2024-05/15/content_292641.html)
  7. [Longbridge OpenAPI 官方文档](https://open.longportapp.cn/zh-CN/docs/index)
  8. [Futu OpenAPI 官方文档](https://openapi.futunn.com/)
  9. [Tiger OpenAPI PyPI + GitHub](https://github.com/tigerfintech/openapi-python-sdk)

---

## 一、A 股程序化交易合规总览（2024-10-08 起实施）[1][2]

### 法规时间线

- **2023-09-01**: 上交所《关于股票程序化交易报告工作有关事项的通知》—— 首次明确报告制
- **2024-05-15**: 证监会发布《证券市场程序化交易管理规定（试行）》[1]
- **2024-06-10**: 沪深北交易所制定实施细则 [4]
- **2024-07-11**: 证监会划定异常交易"红线" [3]
- **2024-10-08**: 正式实施

### 定义（关键理解）

**程序化交易** = 通过计算机程序自动生成或执行交易指令的行为

**适用对象**:
- 机构投资者（私募、QFII、券商自营、公募、保险）
- **个人投资者** —— 规则同等适用（这是很多人忽视的点）

### 高频交易认定标准

满足以下任一即为高频：
- 申报速率 **≥ 300 笔/秒**
- 单日申报 **≥ 20000 笔**

→ 需额外报告服务器位置、系统测试、应急方案。本项目**日频波段触不到这个线**，所以**不属于高频**。

### "先报告后交易"原则

程序化交易投资者需通过证券公司报备：
- 账户信息（资金规模、杠杆）
- 交易策略（大类描述）
- 软件技术参数
- 变更需及时更新

**未报告/虚报** → 限制交易或处罚。

### 异常交易红线（重点）[3]

禁止行为：
1. **虚假挂单**：频繁申报后撤单（撤单率过高）
2. **操纵市场**：拉抬打压 / 影响收盘价 / 制造"秒板"
3. **算法共振**：短时间大额成交造成市场异常波动

---

## 二、A 股"纯提醒不下单"的合规路径（本项目）

### 为什么推荐纯提醒

| 路径 | 合规状态 |
|---|---|
| AI 分析 + 推送提醒，人工手动下单 | ✅ 不属于程序化交易（提醒不是交易指令） |
| AI 触发 + easytrader 模拟 GUI 下单 | 🟡 **同时违反两条**：① 程序化未报备；② 券商反欺诈系统可能封号 |
| 人工确认 + API 自动下单（QMT） | ✅ 但需向券商报备（按程序化交易处理） |
| QMT 自动化下单 + 日频波段 | ✅ 低频不算高频，报备后合法 |
| QMT 高频（> 20000 笔/日） | 🟡 高频类，额外报备 + 可能更高收费 |

### 本项目路径（完全合规）

```
AI 策略扫描 → 触发阈值 → 生成"提醒消息" → 推送给自然人 →
  自然人接到消息 → 自主决策 → 打开券商 APP 手动下单 →
  成交后可选择反馈给 AI 做复盘
```

**核心原理**: 系统输出的是"分析报告/提醒"，不是"交易指令"。不触发程序化交易监管。**上交所明确：提供投资建议、咨询服务不属于程序化交易。**

### A 股"自动执行"进阶路径（未来扩展）

如果你持仓账户 ≥ 50 万，可以开通券商的 **miniQMT** 权限（`xtquant`）：
- 向券商报备策略大类（如"日频趋势跟踪"）
- 签署程序化交易风险揭示书
- OpenClaw 通过 `xtquant` 下单 → 合规

**当前版本不推荐**：用户画像是中级 + 提醒，提醒足矣；且要避免被当成程序化交易需要报备的复杂度。

### easytrader 为什么不推荐

- 模拟 GUI 键鼠 → 本质是程序化交易但未走报备通道 → 违反 2024 新规
- 券商反欺诈系统会识别（异常 click 模式、窗口标题匹配）→ 封账号
- 新规后头部券商已经加强检测

---

## 三、港股下单 API 对比（本项目可选扩展）

### 三家全维度对比

| 维度 | **Futu OpenAPI** | **Longbridge** | **Tiger OpenAPI** |
|---|---|---|---|
| **账户开户** | 富途香港（线上 30 min 可开） | 长桥 BVI / 香港 | 老虎香港 / 新加坡 |
| **API 申请** | 默认开通，需二次验证 | 后台 1 键开通，发 access token | 后台申请 + 上传 RSA 公钥 |
| **交易密码** | API 独立交易密码（非账户登录密码） | access token 体系 | RSA 签名，无密码 |
| **开户门槛** | HKD 10000（最新要求） | 无最低（建议 HKD 10000+） | USD 3000 / HKD 25000 |
| **港股佣金** | 0.03% 最低 3 HKD | 0.03% 最低 3 HKD | 0.03% 最低 7 HKD |
| **港股实时** | **LV2 免费**（大陆个人） | LV1 免费，LV2 付费 | LV1 免费，LV2 付费 |
| **订单类型** | 市/限/止损/条件/特殊HK | 市/限/到价Stop/追踪止损 | MKT/LMT/STP/STP_LMT/Trailing |
| **Python SDK** | `futu-api` | `longport` | `tigeropen` |
| **本地 daemon** | ✅ 需 **OpenD** | ❌ | ❌ |
| **架构** | TCP via OpenD | REST + WebSocket | REST + WebSocket |
| **限频（交易）** | 宽松（~30/s） | 30s ≤ 30 次，间隔 ≥ 0.02s | 中等 |
| **限频（行情）** | ~30/s | 1s ≤ 10 次，并发 ≤ 5 | 60-120/min |
| **SDK 内置限流** | 🟡 部分 | ✅ 行情内置，❌ 交易 | 🟡 部分 |
| **社区/文档** | ✅ 最强 | 🟡 | 🟡 |
| **监管** | 香港 SFC Type 1/4/9 | 香港 SFC Type 1/2/4/9 | 香港/新加坡/美国多牌照 |

### 订单类型细节

**Futu OpenAPI** 支持的 HK 特殊单：
- 竞价限价单（AuctionLimit）
- 增强限价单（EnhancedLimit）
- 特别限价单（SpecialLimit，仅 FOK）

**Longbridge** 订单（代码示例）[7]:
```python
from longbridge.openapi import TradeContext, Config, OrderSide, OrderType, TimeInForceType

config = Config.from_env()
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

# 到价触发的限价单
ctx.submit_order(
    symbol="0700.HK",
    order_type=OrderType.Limit,
    side=OrderSide.Buy,
    submitted_price=310.0,
    submitted_quantity=100,
    time_in_force=TimeInForceType.Day,
    trigger_price=310.0,
    trigger_at="last"
)
```

**Longbridge 限流踩坑** [7]:
- 超限返回错误码 **301606**
- **SDK 未内置交易限流**，必须自己 `time.sleep(0.05)` 或队列
- 多子账户独立计频

### 本项目推荐

**首选：Futu OpenAPI**
- 杀手锏：**港股 LV2 免费**，一年省 ~HKD 2400
- 社区最大，遇到问题好 debug
- OpenD 需要 7×24 在线（可系统服务化）

**次选：Longbridge**
- 无本地 daemon，对 OpenClaw 部署更友好
- SDK 设计现代，Python 生态体验好
- 适合你的 OpenClaw VPS **架构更简洁**的偏好

**不首选 Tiger**
- 除非你已经有 Tiger 美股账号
- 港股费用比 Futu 贵（最低佣金 7 vs 3）

---

## 四、半自动工作流（提醒 → 人工确认 → API 下单）

### 推荐流程（港股扩展）

```
OpenClaw cron 15:10 → AI 策略扫描 →
  触发信号 → 生成 Markdown:
    "📈 0700.HK 腾讯控股 突破信号
     当前价 315.4 (+2.1%)
     建议单：买入 100 股 @ 315 限价
     仓位 8%（当前组合）
     止损：310
     [回复 /confirm_BUY_0700_100_315 下单]
     [回复 /ignore 忽略]"
  → Telegram 推送 →
  人工看到消息 →

  [选项 A] 手动打开富途 APP 下单（无任何自动化）
  [选项 B] 在 Telegram 回复 /confirm_* →
          OpenClaw 解析 → 调 Futu OpenAPI →
          下单 → 回报成交 "
```

### 为什么一定要"人工确认"这一步

1. **LLM 幻觉兜底**：AI 可能因为新闻误判或因子故障给出错误信号
2. **价格漂移**: 盘后信号到开盘价格可能差 2-3%，要当日复核
3. **资金管理复核**: 人工看当前组合决定是否要开这笔
4. **合规心理**: "不是机器人自己下单，是我主动同意的"
5. **心理健康**: 长期 all-auto 会陷入"算法迷信"，亏钱难反思

### confirm_* 指令解析

```python
# OpenClaw skill 入口
@skill_handler("confirm_.*")
def on_confirm(cmd: str, ctx: ChatContext):
    # "/confirm_BUY_0700.HK_100_315" → action=BUY, symbol="0700.HK", qty=100, price=315
    parts = cmd[9:].split("_")
    action, symbol, qty, price = parts
    # 有效期检查（提醒 30 分钟内有效）
    signal = load_signal_by_symbol(symbol)
    if (now() - signal.ts).minutes > 30:
        return "⏱️ 信号已过期（30 分钟），请重新扫描"
    # 二次确认（防误触）
    ctx.reply(f"⚠️ 确认 {action} {qty} {symbol} @ {price}？回复 YES 提交")
    if ctx.await_reply(60) != "YES":
        return "已取消"
    # 走 Futu OpenAPI
    order_id = futu_client.place_order(...)
    return f"✅ 订单已提交 {order_id}"
```

---

## 五、合规避坑 checklist（中国大陆投资者）

### 纯提醒模式（本项目 default）
- [x] 不需要任何监管报备
- [x] 不算程序化交易
- [x] 不触发任何新规

### 港股 API 下单（本项目扩展）
- [x] 港股市场监管归属 SFC，不受 A 股新规约束
- [x] 富途/长桥/老虎作为持牌券商，API 操作合法
- [ ] 注意：**大陆居民海外证券账户** 需按规定申报个税（全球征税）
- [ ] 注意：**资金出境** 限制（5 万美元/年购汇额度）
- [ ] 注意：以美股为主的账户，SEC 监管和税表（W-8BEN）

### A 股实盘进阶（未启用）
- [ ] ≥ 50 万净资产开通 miniQMT
- [ ] 向券商提交报备表（策略大类 + 参数）
- [ ] 签署程序化交易风险揭示书
- [ ] 不做高频（≤ 300 笔/秒 + ≤ 20000 笔/日）
- [ ] 避免虚假挂单、算法共振
- [ ] 关注交易所异常交易红线更新

### 完全不要做
- ❌ easytrader 模拟 GUI 键鼠下单 A 股
- ❌ 未报备的程序化大资金策略
- ❌ A 股新规下的高频（>300 笔/秒）
- ❌ 写开放到公网的"AI 荐股" SaaS（未持牌提供证券投资咨询服务违法）

---

## 六、合规一句话总结

> "**对自己操作给出建议**从古至今都合法。**代替别人做决定**需要持牌。**让机器代替自己做决定**需要报备。本项目是第一种，所以只推提醒。"
