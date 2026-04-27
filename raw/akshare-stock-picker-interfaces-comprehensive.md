# AkShare 选股数据接口全集（北向 / 龙虎榜 / 行业资金流 / 港股）

- **Type**: Synthesis (aggregated from multiple sources)
- **Fetched**: 2026-04-28
- **Project**: ai-stock-picker-hkcn
- **Relevant to**: KQ5
- **Sources drawn from**:
  1. [AkShare 沪深港通历史数据（知乎）](https://zhuanlan.zhihu.com/p/692666826)
  2. [AkShare 板块排行（腾讯云）](https://cloud.tencent.com/developer/article/1745040)
  3. [AkShare 个股龙虎榜（知乎）](https://zhuanlan.zhihu.com/p/482691225)
  4. [AkShare 行业历史资金流（知乎）](https://zhuanlan.zhihu.com/p/22607644143)
  5. [AkShare 资金流接口优化（GitCode）](https://blog.atomgit.com/fe86c96f2068921d311617b11969118a.html)
  6. [AKShare 官方文档](https://akshare.akfamily.xyz/)
  7. [富途牛牛港股关键财务指标](https://www.futunn.com/stock/80388-HK/financials-key-indicators)

---

## 一、北向资金（资金面核心）

| 接口 | 功能 | 参数 | skill 用途 |
|------|------|------|----------|
| `stock_hsgt_hist_em` | 沪深港通历史数据（成交净买额、累计净买额） | `symbol="北向资金"` | 长期资金流趋势 |
| `stock_hsgt_north_net_flow_in` | 北向资金净流入 | `indicator="沪股通"/"深股通"/"北上"` | 日度净流入 |
| `stock_hsgt_fund_min_em` | 日内分时（万元） | `symbol="北向资金"` | 盘中情绪 |
| **`stock_hsgt_board_rank_em`** | **北向增持行业/概念/地域排行** | `symbol="北向资金增持行业板块排行"`, `indicator="今日"` | **★ KQ2 行业选择核心** |

---

## 二、龙虎榜（A 股特色资金面）

| 接口 | 功能 | 参数 |
|------|------|------|
| `stock_lhb_detail_em` | 东财龙虎榜详情（21 字段，含上榜后 N 日涨跌幅） | `start_date`, `end_date` |
| `stock_lhb_jgmmtj_em` | 机构买卖统计（机构数、净买额） | `start_date`, `end_date` |
| `stock_lhb_stock_detail_em` | 个股龙虎榜营业部明细 | `symbol`, `date`, `flag="买入"/"卖出"` |
| `stock_sina_lhb_ggtj` | 个股最近 5/10/30/60 天上榜统计 | `recent_day="5"` |

---

## 三、行业资金流

| 接口 | 功能 | 参数 |
|------|------|------|
| **`stock_board_industry_fund_flow_rank_em`** | **行业板块资金流向排名**（主力/超大/大/中/小单） | 无参 |
| `stock_sector_fund_flow_hist` | 指定行业历史资金流 | `symbol="电源设备"` |
| `stock_sector_fund_flow_summary` | 板块内所有个股资金流（需 AkShare 1.15.23+） | `symbol`, `indicator="今日"` |

---

## 四、港股核心（选股 skill 必备）

| 接口 | 功能 | 备注 |
|------|------|------|
| `stock_hk_spot_em` | 全港股实时行情（含 PE/PB） | **全市场扫描入口** |
| `stock_hk_hist` | 个股日/周/月 K | 分复权 |
| `stock_fundamental(symbol="00700")` | 单只港股财务指标（PE/PB/ROE） | 腾讯示例 |
| `stock_financial_hk_analysis_indicator_em` | 港股财务分析指标 | 东财数据 |

### 港股字段识别（列名差异）

不同接口中 PE 可能是 "市盈率" / "PE(TTM)" / "PE(静态)"，skill 必须做字段归一化。

---

## 五、基本面（A 股）

| 维度 | AkShare 接口 | 备注 |
|------|------------|------|
| 财务指标 | `stock_financial_analysis_indicator` | 年度财务指标 |
| 三大表 | `stock_financial_abstract` + `stock_financial_*_em` | 利润/资产/现金流 |
| 分红 | `stock_dividend_cninfo` | 分红历史 |
| 股东 | `stock_gdfx_holding_detail_em` | 前十大股东 |
| 质押 | `stock_gpzy_pledge_ratio_em` | 控股股东质押率 |
| 商誉 | `stock_sy_em` | 商誉披露 |

---

## 六、一致预期（分析师研报）

- `stock_report_disclosure`：研报披露
- Tushare `report_rc`：一致预期（需付费）
- 东财研报接口：免费但抓取易被反爬

---

## 七、宏观景气

| 接口 | 数据 |
|------|------|
| `macro_china_pmi_yearly` | 中国 PMI（先行景气） |
| `macro_china_cpi_yearly` | CPI |
| `macro_china_ppi_yearly` | PPI |
| `macro_china_fx_reserves_yearly` | 外汇储备 |
| `macro_china_money_supply` | M0/M1/M2 |
| `stock_industry_category_cninfo` | 行业分类 |

---

## 八、特色辅助（对 skill 价值高）

| 接口 | 用途 |
|------|------|
| `stock_gszl_em` | 公司概念涉及 |
| `stock_profile_cninfo` | 公司概况（主营业务） |
| `stock_notice_report` | 公告查询（硬否决证监会立案等） |
| `stock_comment_em` | 千股千评（东财市场情绪分数） |

---

## 九、选股主推数据链（推荐组合）

```
┌─────────────────────────────────────────────────┐
│                  A 股数据层                      │
├─────────────────────────────────────────────────┤
│ 行业景气 (KQ2) │ stock_hsgt_board_rank_em       │
│                │ stock_board_industry_fund_flow │
│                │ macro_china_pmi_yearly         │
│─────────────────────────────────────────────────│
│ 基本面 (KQ3)   │ stock_financial_analysis_ind   │
│                │ stock_dividend_cninfo          │
│                │ stock_sy_em / stock_gpzy_*     │
│─────────────────────────────────────────────────│
│ 技术/量价      │ stock_zh_a_hist (日K)          │
│                │ → 自行算 Alpha158 / MA/RSI      │
│─────────────────────────────────────────────────│
│ 资金面 (特色)  │ stock_hsgt_hist_em (北向)      │
│                │ stock_lhb_detail_em (龙虎榜)    │
│                │ stock_individual_fund_flow     │
│─────────────────────────────────────────────────│
│ 风险 (KQ8)     │ stock_notice_report (立案查询) │
│                │ stock_zh_a_spot_em (ST/价格)   │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│                  港股数据层                      │
├─────────────────────────────────────────────────┤
│ 全市场扫描     │ stock_hk_spot_em               │
│ 个股行情       │ stock_hk_hist                  │
│ 基本面         │ stock_financial_hk_analysis_*  │
│ 南下资金       │ stock_hsgt_components (南下成分)│
│ 卖空比例       │ 需自抓港交所 Short Sell 页面    │
└─────────────────────────────────────────────────┘
```

---

## 十、AkShare 使用注意事项

1. **数据延迟**：实时数据 ~15 秒延迟，不适合高频
2. **单位差异**：% / 亿 / 万元 混用，需归一化
3. **接口重命名**：老接口 `stock_em_*` 正迁移到 `stock_*_em`，建议固定版本
4. **依赖上游**：AkShare 是东财/新浪/同花顺的爬虫包装，上游改版时会坏
5. **限流**：短时间内大量调用会被上游网站 ban IP，建议本地缓存 + 夜间批量

---

## 十一、对选股 skill 的落地建议

1. **核心数据组合**：AkShare（免费）+ Tushare Pro 200/年（一致预期、北向历史）
2. **港股全量扫描**：AkShare + 富途 OpenAPI（LV2 免费）
3. **本地缓存策略**：所有财务数据日度缓存 SQLite + Parquet；日 K 线增量更新；北向/龙虎榜按交易日全量
4. **反爬防御**：多次调用间加 1-3 秒 sleep；关键数据多源验证（东财 + 新浪交叉）
