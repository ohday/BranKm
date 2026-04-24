# OpenClaw 承载方案：Skill + VPS + 存储选型

- **Type**: Synthesis
- **Fetched**: 2026-04-24
- **Project**: ai-swing-reminder-hkcn
- **Relevant to**: Q6
- **Sources drawn from**:
  1. [OpenClaw 自定义 Skill 开发教程（RSS 智能日报）](https://2048ai.net/69a28da454b52172bc5e1758.html)
  2. [OpenClaw 完全实战指南 CSDN](https://blog.csdn.net/quyixiao/article/details/160382493)
  3. [ClawHub Skills 指南 15 选 devpress](https://devpress.csdn.net/v1/article/detail/158419661)
  4. [OpenClaw 定时任务全攻略 阿里云](https://developer.aliyun.com/article/1718611)
  5. [喝水提醒助手开源项目（51CTO）](https://blog.51cto.com/passjava/14552278)
  6. [Hummingbot 心跳监控 Skill](https://skillsmd.top/skills/9db0dfe3baa304ef3f5b6694bff259d5)
  7. [OpenClaw 主站](https://openclaw.ai/)
  8. 阿里云/腾讯云香港节点评测（综合搜索）
  9. DuckDB/Parquet/SQLite Python 对比（综合搜索）

---

## 一、OpenClaw Skill 机制核心 [1][2][3]

### Skill 目录结构

```
~/.openclaw/skills/
└── ai-swing-reminder/
    ├── SKILL.md               # 唯一必需文件
    ├── scripts/               # 可执行 Python / Shell（执行，不读入 context）
    │   ├── daily_signal.py
    │   ├── backtest_strategy.py
    │   └── notify.py
    ├── references/            # 按需被 agent 读入 context 的文档
    │   ├── watchlist.md
    │   └── strategy-params.yaml
    └── assets/                # 不读入 context（模板、静态数据）
        └── report-template.md
```

### SKILL.md 格式

```markdown
---
name: ai-swing-reminder
description: 每日盘后扫描 watchlist，对波段策略触发的股票发送提醒。触发词：盘后信号、每日提醒、扫描策略
version: 0.1.0
user-invocable: true
metadata: {"openclaw":{"requires":{"bins":["python3"],"env":["TUSHARE_TOKEN","TELEGRAM_BOT_TOKEN","WECOM_WEBHOOK"]},"primaryEnv":"TUSHARE_TOKEN"}}
---

# AI 波段提醒 Skill

## 功能
每日 15:10（A 股）/ 16:10（港股）执行波段策略扫描，满足阈值的股票通过 Telegram + 企业微信推送提醒。

## 步骤
1. 读 `{baseDir}/references/watchlist.md` 获取 A 股 + 港股代码
2. 调 `python3 {baseDir}/scripts/daily_signal.py --watchlist {baseDir}/references/watchlist.md --market $MARKET`
3. 脚本会把触发信号写 `/tmp/openclaw-signals-YYYY-MM-DD.json`
4. 调 `python3 {baseDir}/scripts/notify.py --input /tmp/openclaw-signals-YYYY-MM-DD.json`
5. 汇报：总扫描数、触发数、推送渠道

## 错误处理
- 数据源限频：Tushare 500/min → 每股间隔 0.15s
- 推送失败：本地 SQLite 重试队列
```

### Frontmatter 字段清单

| 字段 | 用途 |
|---|---|
| `name` | 唯一标识 + `/name` 斜杠命令 |
| `description` | **触发匹配文本**（用户消息关键词） |
| `version` | Semver，ClawHub 发布必需 |
| `user-invocable` | 是否暴露斜杠命令（默认 true） |
| `disable-model-invocation` | 禁止 LLM 上下文注入 |
| `metadata.openclaw.requires.bins` | 需要的 PATH 二进制 |
| `metadata.openclaw.requires.env` | 必需环境变量（缺失则 skill 不加载） |
| `metadata.openclaw.primaryEnv` | 关联 `skills.entries.<name>.apiKey` 的快捷方式 |

### Skill 加载优先级

1. `<workspace>/skills/`（最高，项目专属）
2. `~/.openclaw/skills/`（中等，跨 agent 共享）
3. Bundled skills（最低，随安装）

还有 `skills.load.extraDirs` 支持团队共享。

### 脚本执行方式

脚本不是"注册"的，SKILL.md 用 `{baseDir}/scripts/xxx.py` 路径引用。`chmod +x`。Skill 在 `openclaw.json` 的 `skills.entries` 下配置：

```json
{
  "skills": {
    "entries": {
      "ai-swing-reminder": {
        "enabled": true,
        "env": {
          "TUSHARE_TOKEN": "your-token",
          "TELEGRAM_BOT_TOKEN": "123:ABC",
          "WECOM_WEBHOOK": "https://qyapi.weixin.qq.com/..."
        }
      }
    }
  }
}
```

### 输入/输出模型

- **输入**: 自然语言 chat 消息匹配 `description` → agent 读 SKILL.md → 按指令执行；或 `/ai-swing-reminder` 直接斜杠命令
- **输出**: 脚本 stdout（agent 读 JSON）+ 写文件（中间结果 `/tmp/xxx.json`，最终报告 `~/openclaw-output/digests/xxx.md`） + 汇报 chat（Markdown 文本）

**没有特殊 return 机制** —— SKILL.md 明文写让 agent 汇报什么。

### Secrets 管理

1. **env in `openclaw.json`** 下 `skills.entries.<name>.env` —— 运行时注入
2. **`.env` 文件** —— agent 读
3. `requires.env` gating —— 必需变量缺失则 skill 不加载

---

## 二、Cron 定时任务 [4]

### Cron 引擎

OpenClaw 内置 **cron-job** skill（需手动启用）:
```bash
claw skill enable cron-job
# 或 clawhub install cron-job
```

### Cron 表达式

标准 5 段: `分钟 小时 日期 月份 星期`
Web UI 支持 Quartz 7 段: `0 0 3 * * ?`

| 场景 | Cron | 说明 |
|---|---|---|
| A 股盘后 | `10 15 * * 1-5` | 每周一到周五 15:10 |
| 港股盘后 | `10 16 * * 1-5` | 16:10 |
| 周末研报 | `0 10 * * 6` | 周六 10:00 |
| 每日心跳 | `59 23 * * *` | 每晚 23:59 |

### 添加任务（CLI）

```bash
openclaw cron add \
  --name "a-share-daily-scan" \
  --cron "10 15 * * 1-5" \
  --channel wecom,telegram \
  --message "/ai-swing-reminder market=a-share"
```

### 任务存储
- 配置: `~/.openclaw/cron/jobs.json`
- 日志: `~/.openclaw/cron/runs/<jobId>.jsonl`（排错）

### 独立会话 vs 主会话
- **独立会话**: 无上下文污染，跑脚本型任务首选
- **主会话**: 共享历史记录，适合"提醒我昨天说了什么"

### 时区
按目标时区配。对中国股市业务，OpenClaw VPS 服务器时区建议设为 `Asia/Shanghai`。

---

## 三、数据持久化选型 [9]

### 需求
- 历史日 K（约 5000 只 A 股 × 10 年 = ~18M 行）
- 触发日志（SQLite 最合适）
- 回测结果缓存（Parquet 列存效率高）
- 策略状态（SQLite KV）

### 三选一对比

| 维度 | SQLite | Parquet (pyarrow) | DuckDB |
|---|---|---|---|
| 安装 | Python 自带 | `pip install pyarrow` | `pip install duckdb` |
| 架构 | 行式，文件级 | 列式，文件级 | 列式，进程内 |
| 事务 | ✅ ACID | ❌（静态文件） | ✅ |
| 写入频率 | ✅ 频繁 | ❌ append 需 rewrite | ✅ |
| 分析性能（日 K 聚合） | 🟡 | ✅✅ | ✅✅✅ |
| 时序窗口/rolling | 🟡 3.45 开始有 | 依赖 Python | ✅ 原生 |
| 文件压缩 | 中等 | **最优** | 用 Parquet 外部表 |
| 适合规模 | GB 级 | TB 级 | TB 级 |

### 推荐架构

```
SQLite：
  ├── watchlist.db        # 监控列表
  ├── signals.db          # 触发日志 + 去重/冷却
  ├── strategy_state.db   # 策略状态 KV
  └── openclaw_jobs.db    # 任务执行历史

Parquet（磁盘）：
  └── data/
      ├── a_share_daily/year=2025/part-01.parquet
      ├── hk_daily/year=2025/part-01.parquet
      └── factor_cache/strategy_id=ma20/year=2025/...

DuckDB（分析入口）：
  # 查询时用 DuckDB 挂载 Parquet
  duckdb.sql("SELECT * FROM read_parquet('data/a_share_daily/*.parquet') WHERE symbol='600519'")
```

**核心理念**: 事务/状态用 SQLite，历史数据用 Parquet，分析查询用 DuckDB 读 Parquet。

---

## 四、VPS 选型（关键！）[8]

### 需求分解

| 任务 | 需要的 IP 地域 |
|---|---|
| Tushare / AkShare 爬 A 股 | **中国大陆**（爬东财 / 新浪，海外有墙风险） |
| Baostock | 中国大陆（服务器在国内） |
| Futu OpenAPI（港股） | 香港 / 海外（富途香港服务器，大陆访问稍延迟） |
| Longbridge OpenAPI | 香港 / 新加坡 |
| Telegram Bot API | **海外**（大陆被墙） |
| OpenAI / Anthropic / Claude API | **海外** |
| 国产 LLM（DeepSeek / Qwen） | 中国大陆或海外（都可达） |
| 企业微信 API | 中国大陆（海外可达但可能延迟） |

### 单节点 vs 双节点

**方案 A：单海外节点（最简）**
- 香港 VPS（腾讯云/阿里云/Vultr HK）
- 大陆数据爬取走代理回国（比如 Tushare token 下从境外仍可用，AkShare 爬东财延迟增大）
- Telegram / OpenAI / Futu 都顺
- **月成本**: ¥50~100（腾讯云轻量 4核4G 香港节点 $12起/月）

**方案 B：双节点（推荐生产）**
- **节点 1 大陆**: Tushare / AkShare / Baostock 数据爬取 + DuckDB 本地分析
- **节点 2 香港**: OpenClaw + Futu OpenAPI + Telegram + LLM API
- 两节点用 Tailscale / WireGuard 打通，节点 1 每晚同步数据到节点 2
- **月成本**: ¥30（大陆轻量）+ ¥50（香港轻量） = ¥80

### 云服务商对比

| 云 | 香港节点 | 大陆→香港延迟 | 香港→Futu 延迟 | 入门价 |
|---|---|---|---|---|
| **腾讯云轻量香港** | ✅ | 20-40ms | **1-3ms**（富途合作） | $12/月起 |
| **阿里云香港** | ✅ | 30-50ms（CN2） | 5-10ms | $15/月起 |
| Vultr HK | ✅ | 50-80ms | 5-15ms | $6/月 |
| Digital Ocean SG | ✅ | 80-120ms | 10-20ms | $6/月 |

**推荐**: 腾讯云轻量香港节点（富途 OpenAPI 延迟最低的公有云）。

### OpenD 部署注意

Futu OpenD **必须 7×24 在线** —— 部署到 VPS 的 Linux 版（非 Docker 官方镜像，但 systemd service 可做）。

```bash
# OpenD 启动示例（Ubuntu）
cd OpenD_8.0.3306_Linux
./OpenD --login_account=xxx --login_pwd=xxx --lang=en --console_log=all
```

配合 systemd:
```ini
[Unit]
Description=Futu OpenD
After=network.target

[Service]
Type=simple
User=openclaw
ExecStart=/opt/OpenD/OpenD --login_account=...
Restart=always

[Install]
WantedBy=multi-user.target
```

### 大陆 VPS 注意事项

- **备案**: 大陆 VPS 必须实名备案（ICP），但**只跑 API 不开网页服务**的话可以免备案（看各家策略，阿里云轻量大陆节点支持免备案运行程序）
- **网络限制**: 部分云（腾讯云大陆）的轻量服务器**禁止访问境外某些端口** —— 跑 Tushare 没问题，跑 Telegram 会连不上
- **合规**: 跑金融策略不涉及公网 web 服务，一般无合规问题

---

## 五、OpenClaw 承载架构（本项目推荐）

```
┌───────────────────────────────────────────────────────────┐
│  腾讯云香港轻量 4核 4G（$12/月）                           │
│                                                           │
│  OpenClaw 主进程                                          │
│    ├── Skills:                                            │
│    │     ├── ai-swing-reminder (核心, 本项目)             │
│    │     ├── cron-job (内置, 已启用)                      │
│    │     ├── tradingagents-cn-wrapper (周末研报)          │
│    │     └── brave-search (可选新闻搜索)                  │
│    │                                                      │
│    ├── Cron jobs:                                         │
│    │     ├── 08:50 mon-fri: 盘前观察                     │
│    │     ├── 15:10 mon-fri: A 股盘后扫描                 │
│    │     ├── 16:10 mon-fri: 港股盘后扫描                 │
│    │     └── 10:00 sat:     周末研报 (TradingAgents-CN)  │
│    │                                                      │
│    ├── IM 出口: Telegram + 企业微信                       │
│    │                                                      │
│    └── 存储:                                              │
│          ├── SQLite: signals/state/cooldown              │
│          ├── Parquet: 历史 K 线（每日增量）               │
│          └── DuckDB: 盘后分析查询                         │
│                                                           │
│  Sidecar 进程                                             │
│    ├── Futu OpenD (systemd, 7x24)                        │
│    └── TradingAgents-CN Docker (docker compose)          │
│                                                           │
│  数据源出口:                                              │
│    ├── AkShare / Tushare / Baostock (via proxy 回大陆)   │
│    ├── Futu OpenAPI (香港本地)                            │
│    ├── Longbridge OpenAPI (可选)                          │
│    ├── Telegram Bot API (海外直连)                        │
│    ├── DeepSeek API (海外 API 大陆都可达)                │
│    └── 企业微信群 webhook (大陆直连)                      │
└───────────────────────────────────────────────────────────┘
```

### 为什么不用双节点

初期单节点足够：Tushare Token 从香港 IP 照样能用，AkShare 爬东财延迟可接受（日频任务不在乎 500ms）。**只有当月成交量 > 100 笔或策略 > 10 个且 intraday 化后**，再拆双节点。

### 为什么不 Dockerize 所有东西

- OpenClaw 本身推荐 npm 装
- Futu OpenD **不建议 Docker 跑**（GUI 登录会话复杂）
- TradingAgents-CN 已经 Docker 化，保持独立 compose
- 数据脚本纯 Python，直接 venv 跑

---

## 六、关键踩坑清单

1. **OpenD 会话掉线**: Futu 服务器每日定时维护，OpenD 可能断连 → systemd `Restart=always` 必不可少，且策略代码要处理重连
2. **Cron 静默失败**: 一定要配 `~/.openclaw/cron/runs/<jobId>.jsonl` 日志巡检，心跳任务（23:59）检测今日 cron 成功率
3. **Skill 改动热加载**: 开 `skills.load.watch`，否则要重启 OpenClaw
4. **时区**: OpenClaw VPS 时区 `Asia/Shanghai`；Cron 用 UTC 可能误触
5. **chmod +x**: Python 脚本忘了可执行权限 → 跑不起来
6. **超长 Markdown 推送**: 企业微信 4096 字节限制，Telegram 4096 字符，超长要拆
7. **单次 token 过量**: LLM skill 不要把"跑全市场 4000 股分析"这种巨作业塞一个 session
8. **VPS 突发流量**: 腾讯云轻量有月带宽配额（通常 1 Mbps 或 10 Mbps 峰值），大量 Futu L2 订阅要注意
