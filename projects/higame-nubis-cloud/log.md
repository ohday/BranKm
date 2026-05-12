## [2026-05-11] explore | Topic defined — HiGame NubisCloud Volumetric Cloud Pipeline

- 研究方法: **本地代码考古** (替代 km-websearch 的 web fetch)
- 数据源 (Phase 0 已踩点确认):
  - `Engine/Source/Runtime/Renderer/Private/NubisVolumes/` (4 文件, 5114 行)
  - `Engine/Shaders/Private/NubisVolumes/` (15 文件)
  - `Engine/Source/Runtime/Engine/Public/NubisVolumeInterface.h`
  - `Projects/HiGame/Plugins/NubisCustom/` (4 模块)
  - `Projects/HiGame/Content/EditorOnly/NubisTools/`
  - `Projects/HiGame/Content/Maps/<level>/NubisVDBCache, NubisQuickCloud`
  - `Projects/HiGame/Content/Material/(MaterialFunction|Volumetric)/Nubis`
  - `Projects/HiGame/Content/Developers/wenxiangzuo/`
- 关键宏: `HIGAME_ENABLE_NUBIS`
- key_questions: 22 条 (A1-A8, B1-B5, C1-C5, D1-D2, E1-E3)
- planned_raw_notes: 9 篇
- planned_wiki_pages: 12 页
- mode: new
- depth_target: source-code-level

## [2026-05-11] phase-1 | Code archaeology — 9 raw notes complete (3024 lines)

研究方法: local-code-archaeology。每篇笔记带 `相对路径:行号` 锚,推论标 [推测]。

### 落地清单
- raw#1 higame-nubis-engine-arch (255 行) — A1, A2
- raw#2 higame-nubis-clipmap-scheduling (282 行) — A3, B3
- raw#3 higame-nubis-rdg-passes (386 行) — A4, A5, A6, A7
- raw#4 higame-nubis-shader-permutations (341 行) — A4, A7, A8
- raw#5 higame-nubis-temporal-and-upscale (418 行) — A8, E1
- raw#6 higame-nubis-plugin-nubiscustom (241 行) — B1, B2, B5
- raw#7 higame-nubis-plugin-nubiscustom2 (315 行) — B3, B4
- raw#8 higame-nubis-bake-pipeline (317 行) — C1-C5
- raw#9 higame-nubis-debug-and-platforms (251 行) — D1, D2, E1-E3

### Phase 1 颠覆性发现 (修正了 Phase 0 的 8 项假设)

1. **HIGAME_ENABLE_NUBIS 硬编码=1 在 Build.h:1152** (不是 .Build.cs 控制)
2. **Shader 数 = 15** (Phase 0 错记 14)
3. **Sparse Voxel 7 cvar 全是空壳** — Pipeline.cpp + Shader 双端 0 处使用 (raw#3 + raw#4 双端独立验证),实际通过 6 档 SVT + MipSelector Atlas + Sector Slab-Skip 等价
4. **Hardware Ray Tracing 完全未接通** — cvar 存在但 0 调用,代码注释 `[CL611225 调试RT]` 暗示历史回滚
5. **Bilateral Near Dominance 未实装** — cbuffer/cvar 已声明但 BilateralUpscale.usf 0 处引用 → 实际是 Reconstruct 5 路 + Bilateral 4 mode + Far under Near
6. **NubisCustom 老路径几乎全是蓝图实验遗骸** — C++ 仅 BlueprintImplementableEvent,唯一活着的是 UNubisGenSDFComponent (1738 行 SDF 烘焙工具,被新路径间接复用)
7. **Plugin 直接调用 ENQUEUE_RENDER_COMMAND + RHIUpdateTexture3D 4 处,不触 RDG** (Phase 0 假设是通过 RDG)
8. **NubisVDBDataAsset 运行时 Plugin 内零消费** — 只烘焙期 EditorTools 用

### 跨笔记三角验证

| 主张 | 来源 | 验证状态 |
|------|------|--------|
| Sparse Voxel cvar 空壳 | raw#3 + raw#4 | ✅ 双端确认 |
| Hardware RT 0 调用 | raw#3 (cpp) + raw#4 (shader) | ✅ 双端确认 |
| CL611225 回滚 | raw#1 + raw#3 + raw#5 | ✅ 三处证据 |
| Reconstruct Path 数 | raw#5 (4 路) vs raw#4 (5 路) | ⚠️ 冲突 → wiki 第 8 页采 5 路 |
| Plugin 直接 ENQUEUE | raw#7 | ✅ 实证 |

### 完整性自检
- WHAT: ✅ (每篇首段一句话定义)
- WHY: ✅ (BC1 undershoot / EMA β=0.97 / Two-Pass 方向 / Plugin 不触 RDG 的动机均说明)
- HOW: ✅ (每篇含 ASCII / Mermaid + 关键代码摘录 + 文件:行号锚)
- DEPTH: ✅ (源码级)
- TRADE-OFFS: ✅ (老/新路径并存原因 / Sparse Voxel 工程权衡 / Linux deny 原因)
- CROSS: ✅ (跨笔记三角验证至少 5 处)
- LINK: ✅ (9 篇互相引用)

## [2026-05-11] phase-2 | Wiki synthesis — 12 pages complete (5590 lines)

派发策略:6 个 + 6 个分两批,全部 general-purpose,带 18 条已知事实 + raw 来源 + 12 页大纲作为统一前提。

### 落地清单
- wiki#1 总览 (295 行)
- wiki#2 引擎架构 (317 行)
- wiki#3 GT↔RT 时序 (480 行)
- wiki#4 Clipmap 6 级调度 (532 行)
- wiki#5 RDG Pass DAG (405 行)
- wiki#6 ★ MipSelector + Slab-Skip (376 行) — 推翻 Phase 0 的 Sparse Voxel 章
- wiki#7 LightingCache (476 行)
- wiki#8 ★ Reconstruct + Bilateral (761 行,最长) — 主题修正 Near Dominance
- wiki#9 ★ NubisCustom 插件 (405 行) — 主题修正 老/新路径
- wiki#10 烘焙流水线 (370 行)
- wiki#11 关卡 Cookbook (704 行)
- wiki#12 ★ 调试/性能/平台/陷阱 (469 行) — 加 CL611225 / 标记缺失等 12 项陷阱

### 大纲修正项 (用户审核通过)
- 第 6 页: "Sparse Voxel 两级" → "MipSelector + Sector Slab-Skip 等价方案 (Sparse Voxel cvar 空壳)"
- 第 8 页: "Bilateral + Near Dominance Blend" → "Reconstruct 5 路 + Bilateral 4 mode + Far under Near (Near Dominance 未实装)"
- 第 9 页: "老/新路径并存" → "新路径唯一; 老路径是蓝图遗骸 + 复用 SDF 工具"
- 第 12 页: 加节 "CL611225 回滚" + "HiGame Begin/End 标记缺失"

### Agent 执行问题与解决
- Wave 1 (Phase 1 初期) 用 Explore (read-only),其中 3 个 agent 在 15:18-15:19 卡死 ~107 分钟,全部重派为 general-purpose
- 双触发现象: 多个 agent 完成后 system 重发 task-notification,触发 agent 重新校验 — 已确认是无害事件 (后写覆盖前写,留存的总是更高质量版本)
- general-purpose 平均 5-15 分钟,Explore read-only 多次卡死,**结论: 落盘类任务一律用 general-purpose**

### 自检通过项
- ✅ 22 / 22 key_questions 全覆盖
- ✅ 每页含 Mermaid 或 ASCII 图
- ✅ 重大主张挂 footnote 引用 raw
- ✅ 12 页全部至少 1 处交叉引用
- ✅ 18 条已知事实表全员一致

### 待 lint agent 检查项
- 交叉引用矩阵
- footnote 完整性
- Mermaid 语法 sanity
- 18 条事实页间一致性
- 篇幅与 frontmatter

## [2026-05-11] phase-2-followup | Index + Log update + Lint + Publish plan

- index.md 已重写,反映 Phase 1+2 真实结果(8 项颠覆性发现明示,12 页知识地图含 ★ 标记修正页)
- log.md (本文件) 已记录完整 Phase 0/1/2 节点
- Lint agent 已派出 (general-purpose),报告将落到 `lint-report.md`
- Publish plan 待生成 (iWiki/km 发布)
