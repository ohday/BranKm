# 策略 → 信号 → 提醒：工程架构与推送渠道对比

- **Type**: Synthesis
- **Fetched**: 2026-04-24
- **Project**: ai-swing-reminder-hkcn
- **Relevant to**: Q5
- **Sources drawn from**:
  1. [Telegram Bot API 官方文档](https://core.telegram.org/bots/api)
  2. [Bark GitHub (Finb/Bark)](https://github.com/Finb/Bark)
  3. [Server酱 官网 sct.ftqq.com](https://sct.ftqq.com/)
  4. [企业微信群机器人避坑指南 CSDN](https://blog.csdn.net/weixin_29009401/article/details/148544017)
  5. [企业微信群机器人 API 指南 CSDN](https://blog.csdn.net/2501_94198109/article/details/156989571)

---

## 一、轮询 vs 事件驱动的选择

### 轮询模式（推荐本项目）
```
cron: 每日 15:10（A 股收盘后）
  → 拉日 K →
  → 计算因子/信号 →
  → 阈值判断 →
  → 推送
```

**优势**:
- 简单，debug 容易
- 日频波段场景下足够实时（收盘 10 分钟后出结论，第二天早盘前完全可以看到）
- 数据源限频友好（一次性批量拉）
- OpenClaw cron skill 天然支持

**劣势**:
- 分钟级信号会延迟
- 盘中意外事件（停牌/公告/突发新闻）漏报

### 事件驱动模式（本项目不必要）
WebSocket 订阅实时行情 → tick 更新触发因子重算 → 立即推。
- 只适合日内/高频，不适合波段

### 本项目选择

**双层 cron 轮询**:
- **盘前 08:50**: 检查昨夜公告 + 北向资金 + 美股走势 → "早盘观察"
- **盘后 15:10**: 跑收盘数据 → 硬信号（如"突破 20 日高"）→ "盘后提醒"
- （可选） **周末 10:00**: TradingAgents-CN 对 watchlist 跑多 agent 研报

---

## 二、信号处理的工程模式

### 1. 信号去重（防止同一信号一天触发多次）

```python
class SignalDeduper:
    """基于 (signal_id, date) 的幂等层"""
    def __init__(self, db_path):
        self.conn = sqlite3.connect(db_path)
        self.conn.execute('''CREATE TABLE IF NOT EXISTS signals
            (sig_id TEXT, date TEXT, PRIMARY KEY (sig_id, date))''')

    def should_send(self, sig_id: str, date: str) -> bool:
        cur = self.conn.execute(
            'SELECT 1 FROM signals WHERE sig_id=? AND date=?',
            (sig_id, date))
        if cur.fetchone():
            return False
        self.conn.execute('INSERT INTO signals VALUES (?, ?)', (sig_id, date))
        self.conn.commit()
        return True

# sig_id 例："MA20_breakout_600519_20261010"
```

### 2. 防抖（冷却期）

同一标的同一类信号，N 日内只触发一次：

```python
def in_cooldown(symbol: str, signal_type: str, cooldown_days: int = 5) -> bool:
    last_trigger = get_last_trigger(symbol, signal_type)  # from SQLite
    if last_trigger and (today - last_trigger).days < cooldown_days:
        return True
    return False
```

### 3. 多源确认（置信度）

信号达到 K 个才推送：

```python
def confirm_signal(symbol: str, signals: dict) -> (bool, int):
    """signals = {'ma_cross': True, 'volume_surge': True, 'rsi_oversold': False, ...}"""
    score = sum(1 for v in signals.values() if v)
    return score >= 2, score
```

### 4. 分级提醒

```
Level 1 (弱): 单因子触发 → Telegram 普通 push
Level 2 (中): 2 因子一致 + RSI 配合 → Bark + Telegram
Level 3 (强): 3+ 因子一致 + 基本面支持 + TradingAgents-CN 多 agent 辩论确认 → 企业微信 + Telegram + Bark 重要通知
```

Bark 支持 `level=critical` / `level=timeSensitive`，iOS 会**穿透勿扰模式**。

---

## 三、推送渠道全维度对比

### 海外 IM（Telegram）[1]

**API**:
```
POST https://api.telegram.org/bot<TOKEN>/sendMessage
{
  "chat_id": -1001234567890,
  "text": "*600519* 贵州茅台触发突破信号\n价格: 1850 (+2.3%)\n建议仓位 <10%",
  "parse_mode": "MarkdownV2"
}
```

**Token 获取**: 找 `@BotFather` `/newbot`。Token 格式 `123456:ABC-DEF...`。

**Bot 创建**: 需要自己先 `/start` 一次 bot，获取 chat_id。

**限频**: 官方 API 文档未直接列出；实践经验：
- 同一 chat 每秒 1 条
- 全局每秒 30 条
- 群里每分钟 20 条

**Parse mode**: `Markdown` / `MarkdownV2` / `HTML`

**推荐库**:
- `python-telegram-bot` —— 最成熟
- `aiogram` —— 纯 async
- `pyrogram` —— MTProto 底层（bot + user 账号）

**本项目适配**: ✅ 首选海外渠道。无需备案，全球可达。**但中国大陆用户需要代理访问 Telegram**。

### 国内 IM（企业微信群机器人）[4][5]

**API**:
```
POST https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=XXX
{
  "msgtype": "markdown",
  "markdown": {
    "content": "### 盘后信号提醒\n**600519** 贵州茅台\n- 突破 20 日高\n- 成交量放大 2 倍"
  }
}
```

**创建**: 企业微信群 → "群机器人" → 添加 → 获取 webhook key

**限频**（**重要**）:
- **每个机器人每分钟 20 条**（触发返回错误码 45009）
- Markdown 最长 4096 字节（UTF-8）
- 文本最长 2048 字节

**避坑**:
- `<font color="info">` 等颜色标签受限
- 硬编码 webhook URL 是安全风险，走环境变量
- 支持 IP 白名单（企业微信后台配）
- 建议指数退避重试：
  ```python
  def send_with_retry(url, data, max_retries=3):
      for i in range(max_retries):
          r = requests.post(url, json=data)
          if r.status_code == 200 and r.json().get('errcode') == 0:
              return True
          time.sleep(2 ** i)
      return False
  ```

**本项目适配**: ✅ 国内首选，免代理，企业微信手机端 push 及时。**但需要有企业微信（可免费注册企业账号）**。

### 国内移动（Server 酱）[3]

**推送目的地**: 个人微信 / 企业微信 / 钉钉群 / 飞书群 / 手机客户端
**API**:
```
POST https://sctapi.ftqq.com/<SendKey>.send
{"title": "...", "desp": "..."}
```

**层级**:
- 免费版：每月若干次（具体看当时政策，~500/月）
- Turbo 版（付费）：扩展通道

**限频**: 个人微信通道大约每小时数十条。

**本项目适配**: ✅ 作为"个人微信 push"最方便的路径（个人微信不开放 bot API）。**建议用 Turbo 版 + 企业微信通道作为兜底**。

### 国内移动（PushPlus）
类似 Server 酱。支持微信/邮件/钉钉/飞书/Webhook。免费额度更宽松（每日 100+）。

### iOS 专属（Bark）[2]

**架构**: iOS app + 后端（官方公共 `api.day.app` 或 self-host）

**API**:
```
curl https://api.day.app/<KEY>/<TITLE>/<BODY>
```

**特色**:
- **穿透勿扰**: `level=critical`
- **时效通知**: `level=timeSensitive`
- **重复响铃 30 秒**: `sound=call` 或 `call=1` 参数
- **自定义图标、分组、加密推送**（`ciphertext`）

**Self-host**: `bark-server` 开源（Docker 可部署），需自建 APNs 证书链接

**本项目适配**: ✅ iOS 用户的"重要信号"专用通道。对波段三级预警里的 L3 最合适。

### 其他（扫过）

| 渠道 | 成本 | 限频 | 本项目评分 |
|---|---|---|---|
| 钉钉群机器人 | 免费 | 每分钟 20 | ★★★☆ 国内候补 |
| 飞书群机器人 | 免费 | 宽松 | ★★★☆ 团队场景 |
| Discord webhook | 免费 | 每 5s 5 条 | ★★☆☆ 海外候补 |
| Slack webhook | 免费 | 每秒 1 条 | ★★☆☆ 企业海外 |
| SMTP 邮件 | 个人邮箱 | 宽松 | ★★☆☆ 归档用 |
| 短信网关（阿里云/腾讯云） | ¥0.045/条 | 需签名审核 | ★★☆☆ 关键兜底 |
| iMessage / WhatsApp | 需借 OpenClaw 绑定 | 宽松 | ★★★☆ OpenClaw 原生 |

---

## 四、推送渠道矩阵（按用户画像）

| 用户场景 | 主 | 副 | 重要信号专用 |
|---|---|---|---|
| 中国大陆，iOS 手机 | **企业微信机器人** | Server 酱 | **Bark Critical** |
| 中国大陆，Android | **企业微信** | PushPlus / Server 酱 | Server 酱 +钉钉 |
| 大陆 + 海外 | **Telegram** (代理) + **企业微信** | Server 酱 | Bark |
| OpenClaw 已接 IM | **OpenClaw 原生 WhatsApp/iMessage** | Telegram | Bark |

---

## 五、生产级信号分发代码骨架

```python
# notifier.py
import os, time, sqlite3, requests
from datetime import datetime, date
from dataclasses import dataclass
from typing import Literal

@dataclass
class Signal:
    symbol: str          # "600519.SH" / "0700.HK"
    signal_id: str       # "ma20_breakout" / "volume_surge_2x" / ...
    level: Literal[1, 2, 3]
    title: str
    body: str            # Markdown
    score: int           # 多源确认 score
    extra: dict = None   # 附加 (price, vol, etc.)

class Notifier:
    def __init__(self, config: dict, db_path: str):
        self.cfg = config
        self.db = sqlite3.connect(db_path)
        self._init_db()

    def _init_db(self):
        self.db.execute('''CREATE TABLE IF NOT EXISTS sent_signals
            (sig_id TEXT, symbol TEXT, date TEXT, level INT, ts TEXT,
             PRIMARY KEY (sig_id, symbol, date))''')

    def _dedup_ok(self, sig: Signal) -> bool:
        today = date.today().isoformat()
        row = self.db.execute(
            "SELECT 1 FROM sent_signals WHERE sig_id=? AND symbol=? AND date=?",
            (sig.signal_id, sig.symbol, today)).fetchone()
        return row is None

    def _cooldown_ok(self, sig: Signal, days: int = 5) -> bool:
        row = self.db.execute(
            "SELECT MAX(date) FROM sent_signals WHERE sig_id=? AND symbol=?",
            (sig.signal_id, sig.symbol)).fetchone()
        if not row[0]: return True
        last = datetime.strptime(row[0], "%Y-%m-%d").date()
        return (date.today() - last).days >= days

    def _mark_sent(self, sig: Signal):
        self.db.execute(
            "INSERT OR REPLACE INTO sent_signals VALUES (?,?,?,?,?)",
            (sig.signal_id, sig.symbol, date.today().isoformat(),
             sig.level, datetime.now().isoformat()))
        self.db.commit()

    # --- 渠道 ---
    def _telegram(self, sig: Signal):
        token = self.cfg['telegram']['token']
        chat_id = self.cfg['telegram']['chat_id']
        requests.post(
            f"https://api.telegram.org/bot{token}/sendMessage",
            json={"chat_id": chat_id,
                  "text": f"*{sig.title}*\n{sig.body}",
                  "parse_mode": "Markdown"}, timeout=10)

    def _wecom(self, sig: Signal):
        url = self.cfg['wecom']['webhook']
        for attempt in range(3):
            r = requests.post(url, json={
                "msgtype": "markdown",
                "markdown": {"content": f"### {sig.title}\n{sig.body}"}},
                timeout=10)
            if r.status_code == 200 and r.json().get('errcode') == 0:
                return
            time.sleep(2 ** attempt)

    def _bark_critical(self, sig: Signal):
        key = self.cfg['bark']['key']
        requests.get(f"https://api.day.app/{key}/{sig.title}/{sig.body}",
                     params={"level": "critical" if sig.level == 3 else "timeSensitive",
                             "sound": "call" if sig.level == 3 else "default"},
                     timeout=10)

    # --- 主入口 ---
    def dispatch(self, sig: Signal, force: bool = False):
        if not force:
            if not self._dedup_ok(sig) or not self._cooldown_ok(sig):
                return False

        # 分级路由
        if sig.level >= 1:
            self._wecom(sig)      # 所有级别国内
        if sig.level >= 2:
            self._telegram(sig)   # 中强信号加海外
        if sig.level >= 3:
            self._bark_critical(sig)  # 强信号穿透勿扰

        self._mark_sent(sig)
        return True
```

---

## 六、故障兜底设计

```
主通道失败 → 退避重试 3 次 → 切备用通道 → 再失败 → 写入 SQLite 待处理队列
  → 次日 OpenClaw 上线时补发（或降级邮件）
```

健康检查：每天 23:59 发一条"心跳" 到所有渠道，如某渠道 48h 无反馈 → 下次 cron 触发 fallback。
