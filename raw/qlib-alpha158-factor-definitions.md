# Qlib Alpha158 因子完整定义（源码级）

- **URL**: https://raw.githubusercontent.com/microsoft/qlib/main/qlib/contrib/data/loader.py
- **Fetched**: 2026-04-28
- **Project**: ai-stock-picker-hkcn
- **Relevant to**: KQ3, KQ5

---

## 一、总览

Alpha158 由 `Alpha158DL` 类构造，默认配置 `{"kbar":{}, "price":{"windows":[0],"feature":["OPEN","HIGH","LOW","VWAP"]}, "rolling":{}}`，产出 **9 (kbar) + 4 (price) + 0 (volume, 默认不含) + 140 (rolling, 28 类 × 5 窗口) = 153** 个特征。158 是包含可选扩展（如加上 volume、额外 price 窗口）时的总数。

Label 默认定义：`Ref($close, -2)/Ref($close, -1) - 1` — 即 **T+1 日收盘到 T+2 日收盘的 1 日前瞻收益**。Alpha158vwap 变种用 VWAP 代替。

---

## 二、KBAR 因子（9 个，形态类）

| 因子 | 定义 | 直觉 |
|------|------|------|
| KMID | `($close-$open)/$open` | 实体涨跌幅 |
| KLEN | `($high-$low)/$open` | 总振幅 |
| KMID2 | `($close-$open)/($high-$low+1e-12)` | 实体占振幅比 |
| KUP | `($high-Greater($open,$close))/$open` | 上影相对开盘 |
| KUP2 | `($high-Greater($open,$close))/($high-$low+1e-12)` | 上影占振幅比 |
| KLOW | `(Less($open,$close)-$low)/$open` | 下影相对开盘 |
| KLOW2 | `(Less($open,$close)-$low)/($high-$low+1e-12)` | 下影占振幅比 |
| KSFT | `(2*$close-$high-$low)/$open` | 收盘偏振幅中点（位置） |
| KSFT2 | `(2*$close-$high-$low)/($high-$low+1e-12)` | 收盘偏振幅中点占比 |

---

## 三、PRICE 因子（默认 4 个）

默认 `windows=[0]`，`feature=["OPEN","HIGH","LOW","VWAP"]`。

- 若 d != 0：`Ref($<field>, d)/$close`
- 若 d == 0：`$<field>/$close`

默认输出：`OPEN0 = $open/$close`，`HIGH0 = $high/$close`，`LOW0 = $low/$close`，`VWAP0 = $vwap/$close`。

---

## 四、VOLUME 因子（默认不含）

若启用，规则：d != 0 用 `Ref($volume,d)/($volume+1e-12)`；d == 0 用 `$volume/($volume+1e-12)`。

---

## 五、ROLLING 因子（28 类 × 5 窗口 = 140）

默认窗口 `[5, 10, 20, 30, 60]`，每类 × 5 窗口。

### 5.1 价格变化率与均线

| 因子 | 定义 | 用途 |
|------|------|------|
| ROC{d} | `Ref($close,d)/$close` | d 日前价 / 当前价 → d 日反向涨幅 |
| MA{d} | `Mean($close,d)/$close` | 均线偏离 |
| STD{d} | `Std($close,d)/$close` | d 日波动率 |

### 5.2 趋势回归

| 因子 | 定义 | 用途 |
|------|------|------|
| BETA{d} | `Slope($close,d)/$close` | 线性拟合斜率 → 趋势强度 |
| RSQR{d} | `Rsquare($close,d)` | 线性拟合 R² → 趋势一致性 |
| RESI{d} | `Resi($close,d)/$close` | 线性拟合残差 → 偏离趋势 |

### 5.3 极值与分位

| 因子 | 定义 |
|------|------|
| MAX{d} | `Max($high,d)/$close` |
| MIN{d} | `Min($low,d)/$close` |
| QTLU{d} | `Quantile($close,d,0.8)/$close` |
| QTLD{d} | `Quantile($close,d,0.2)/$close` |
| RANK{d} | `Rank($close,d)` |
| RSV{d} | `($close-Min($low,d))/(Max($high,d)-Min($low,d)+1e-12)` (相当于 KDJ 的 RSV) |

### 5.4 高低点位置

| 因子 | 定义 |
|------|------|
| IMAX{d} | `IdxMax($high,d)/d` — 最高价距今位置 / d |
| IMIN{d} | `IdxMin($low,d)/d` — 最低价距今位置 / d |
| IMXD{d} | `(IdxMax($high,d)-IdxMin($low,d))/d` — 高低点时间差 |

### 5.5 量价关系

| 因子 | 定义 |
|------|------|
| CORR{d} | `Corr($close, Log($volume+1), d)` — 价量相关 |
| CORD{d} | `Corr($close/Ref($close,1), Log($volume/Ref($volume,1)+1), d)` — 涨幅-成交量变化率相关 |

### 5.6 涨跌天数统计

| 因子 | 定义 |
|------|------|
| CNTP{d} | `Mean($close>Ref($close,1), d)` — 上涨天数占比 |
| CNTN{d} | `Mean($close<Ref($close,1), d)` — 下跌天数占比 |
| CNTD{d} | CNTP - CNTN |

### 5.7 涨跌幅累计

| 因子 | 定义 |
|------|------|
| SUMP{d} | `Sum(Greater($close-Ref($close,1),0),d) / (Sum(Abs($close-Ref($close,1)),d)+1e-12)` |
| SUMN{d} | `Sum(Greater(Ref($close,1)-$close,0),d) / (分母同上)` |
| SUMD{d} | (SUMP - SUMN) 形式：分子 `Sum(Greater($close-Ref($close,1),0),d)-Sum(Greater(Ref($close,1)-$close,0),d)`，分母同上 |

### 5.8 成交量衍生

| 因子 | 定义 |
|------|------|
| VMA{d} | `Mean($volume,d)/($volume+1e-12)` — 均量偏离 |
| VSTD{d} | `Std($volume,d)/($volume+1e-12)` — 量波动 |
| WVMA{d} | `Std(\|pctchg\|*$volume,d) / (Mean(\|pctchg\|*$volume,d)+1e-12)` — 波动加权量的波动 |
| VSUMP{d} | 放量占比（放量/量变化绝对和） |
| VSUMN{d} | 缩量占比 |
| VSUMD{d} | 放量 - 缩量 |

---

## 六、处理链

- **Infer processors (推断时)**：ProcessInf → ZScoreNorm → Fillna
- **Learn processors (训练时)**：DropnaLabel → CSZScoreNorm(label, 截面 Z-Score)

截面 Z-Score 的意义：Alpha158 本质是**截面选股因子**——每日对全市场横切面做归一化，rank 最高的被选出。这和波段择时（按时间序列预测涨跌方向）是不同范式。

---

## 七、选股 vs 择时的用法

- **选股（本项目用途）**：用 Alpha158 + XGBoost/LightGBM 训练每日截面 Top-N 排序模型，选出评分最高的 N 只。
- **择时**：Alpha158 中 MA/ROC/RSI 类可单独做时序信号，但 Qlib 本身不主推这种用法。

---

## 八、与 Alpha360 的区别

Alpha360 是**原始特征派**：把过去 60 天的 OHLCV 直接作为 360 维特征（60 × 6），让模型自己学；特征更"原始"但需要更大模型与数据。Alpha158 是**工程派**：已经把 trader 经验做成 158 个公式。

---

## 九、对本项目（选股 skill）的意义

1. **现成的技术面因子库**：KBAR 9 + Rolling 140 可直接用于 skill 的"技术面 + 量价"维度打分
2. **需自行扩展**：Alpha158 **完全不含基本面**（PE/PB/ROE/营收增速），也不含 A 股特色（北向/龙虎榜）。skill 要做全维选股必须在 Alpha158 之上补基本面与资金面
3. **港股需自己做 CSV→bin**：Alpha158 原生只接 CN/US 市场，港股数据需手工适配
4. **IC 阈值经验值**：社区共识 `abs(IC) > 0.05` 作为因子有效性下限，ICIR > 0.5 更稳健
