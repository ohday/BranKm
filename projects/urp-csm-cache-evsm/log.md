## [2026-06-28] explore | Topic defined

- 主题：URP14/PC 阴影优化 —— CSM 缓存 + Variance/EVSM 软阴影
- VSM 澄清：指 Variance/EVSM（矩滤波软阴影），非 UE5 Virtual Shadow Maps
- 平台 PC/主机；URP14 旧 ScriptableRenderPass + RTHandle
- 范围：缓存为主 + 质量（EVSM 软阴影）补充
- 双视角：第三人称过肩 + 2.5D 建造
- 7 个关键问题已定义；核心难题 = EVSM moment 不可 min 合并的三者组合管线

## [2026-06-28] explore | Scope refined
- 核心问题主攻方向定为：静/动 EVSM 分开存、光照阶段合并（先合并 depth 作对照基准）
- 屏幕空间接触阴影：用户明确不纳入，移入排除项

## [2026-06-28] websearch | 深度研究完成

**方法**：4 个并行研究流（A=CSM缓存机制Q1-3 / B=EVSM+合并Q4-5 / C=开源盘点Q6 / D=双视角Q7），共 ~150 次工具调用 + 主 agent 1 轮补搜确认。WebSearch 工具全程 400 不可用，全部检索改用 WebFetch + DuckDuckGo HTML。

**产出**：raw/ 新增 4 篇 synthesis 笔记 + sources.yaml（24 个独立来源，去重后远低于 30 上限）。

**完备性检查（WHAT/WHY/HOW/DEPTH/TRADE/CROSS）**：
- Q1 静/动分离+min-depth+持久RTHandle ✅ 完备（HDRP+Insomniac+Unreal 三方交叉）
- Q2 卷动+texel snap ✅ 完备（Insomniac+Hoppe clipmap+Microsoft+Tardif）
- Q3 时间错峰+脏区失效 ✅ 完备（社区实测+Unreal VSM page+HDRP API）
- Q4 Variance/EVSM 机理 ✅ 完备深入（GPU Gems3+MJP+ESM+MSM 一手论文/代码）
- Q5 ★核心★ 静/动 EVSM 合并 ✅ 达标但 CROSS 弱——**补搜确认业界无先例，为原创工程设计**，已诚实标注；WHAT+WHY+HOW+DEPTH+TRADE 满足
- Q6 开源盘点 ✅ 完备（结论：URP14 无即插即用成品，给出四块拼装路径）
- Q7 双视角调参 ✅ 完备（Microsoft+PSSM+Catlike+VSM+HDRP，含工程推断✱标注）

**关键发现**：① 静/动合并主流走 "blit 静态深度+LEQUAL 渲动态" 而非 receiver-min；② EVSM 必须 fp32(c≤42)；③ **静/动 EVSM moment 合并无任何文献先例**，需在可见性空间 min/乘合并（本项目原创）；④ 开源界零成品，最现实是 Catlike/URP-pass + radishface + HDRP 范式 + TheRealMJP EVSM 四块自拼。

**主要 Gaps**：原始 VSM(Donnelly2006)/LVSM(2008) 全文未直取（经 GPU Gems3 Lauritzen 本人执笔间接覆盖）；CryEngine GSM 一手缺失；具体城建游戏(Cities/Anno)阴影方案无一手；Q5 合并的"并排乘/前后min"与多处工程参数为合理推导，落地需实测。

## [2026-06-28] wiki | URP14 CSM 缓存 + Variance/EVSM — Synthesis Complete
- Pages created: 9（1 总览 + Q1-Q7 各一页 + 实现路线图）
- Sources used: 24 个独立来源（归入 4 篇 raw synthesis 笔记，sources.yaml id 1–4）
- 可视化：每个 ## 章节均配 Mermaid 图，遵守圈号编号/引号规则
- 核心页：第 6 页「静动 EVSM 合并」明确标注为原创工程设计（无文献先例）
