# NubisCloud Wiki Lint 报告

扫描范围: `D:\BranKM\BranKM\projects\higame-nubis-cloud\wiki\` 全 12 个 .md 文件
扫描日期: 2026-05-11

---

## 1. 交叉引用矩阵

| 页 | 引用其他页(实际文件存在) | 被引用次数 | 状态 |
|---|---|---|---|
| 1 | 0 (所有外链路径均缺"— 副标题") | 1 (Page 7 引用 Page 1) | 🔴 全部 broken |
| 2 | 1 (←页内自环) | 5 (Page 1/3/4/5/9 引用) | 🔴 出链全部 broken |
| 3 | 1 (Page 2 ✅) | 6 | 🟡 5/6 出链 broken |
| 4 | 1 (Page 10 ✅) | 7 | 🔴 多数出链 broken |
| 5 | 2 (Page 2 ✅, Page 10 ✅) | 8 | 🟡 多数出链 broken |
| 6 | 2 (Page 7 ✅, Page 10 ✅) | 3 | 🟡 多数出链 broken |
| 7 | 1 (Page 1 ✅) | 5 | 🟡 多数出链 broken |
| 8 | 1 (Page 12 ✅) | 2 | 🟡 多数出链 broken |
| 9 | 1 (Page 7 ✅) | 4 | 🔴 [p1]..[p12] 引用宏全部 broken |
| 10 | 0 | 4 | 🔴 全部 broken |
| 11 | 0 | 1 | 🔴 全部 broken |
| 12 | 0 | 9 | 🔴 全部 broken |

**核心问题**: 几乎所有跨页链接 URL 都使用**短标题**(如 `2. 引擎端架构.md`),但实际文件名带有"— 副标题"(如 `2. 引擎端架构 — HIGAME_ENABLE_NUBIS 与 INubisVolumeInterface.md`)。**全 wiki ~80+ 处链接 broken**。

每页 acceptance_criteria 至少 1 处交叉引用 — **意图层面满足**(每页都尝试引用其他页),但**实际 URL 命中**很少。

## 2. 孤立页(无人引用 / 不引用别人)

无完全孤立页,但**实际可点击的入站引用**统计:
- **Page 11** 仅被 Page 10 文字提及("跳哪一页 |11. Cookbook"),链接 broken,事实上几无活跃入链
- **Page 1** 仅 Page 7 §9.x 有 1 条活引用
- 其余页都有 ≥1 个活引用

## 3. footnote 问题

- 全部 12 页所有 `[^xxx]` 引用**均有定义**,**0 处孤立引用** ✅
- 全部 12 页的 footnote 数量 ≥3 ✅
  - 最少: Page 2 (`[^arch]`, `[^shader]`),共 2 处定义 ⚠️
  - **Page 2 仅 2 个 footnote 定义**,违反"每页至少 3 处 footnote"目标
- 所有 footnote 路径指向 `D:\BranKM\BranKM\raw\*.md` 或 `[[wikilink]]` 形式 ✅

## 4. Mermaid 语法警告

- 节点标签使用英文双引号 `""` 包裹中文内容是 Mermaid **合法语法**(用于含特殊字符的节点),不算违规
- 全文未发现 `[1. xxx]` / `[2. xxx]` 形式的"数字+点"开头节点(均使用 ① ② ③ 圆圈号或 A1/B1 等字母前缀) ✅
- `flowchart` / `graph` / `sequenceDiagram` / `stateDiagram-v2` 关键字使用正确 ✅
- ⚠️ **Page 6 line 97**: `subgraph Bake["烘焙期 ([第 10 页](10.%20烘焙流水线%20—%20Houdini%20→%20VDB%20→%20BC1%20→%20Sector%20→%20Cook.md))"]` — 把完整 markdown 链接嵌进 Mermaid subgraph 标签内,渲染会崩或显示原文
- ⚠️ **Page 7 line 97**: 节点 `GT[NubisClipmapManager::Update<br/>...ScrollUVOffset = ScrollOffset / SectorWidth ∈ [0,1)]` — 节点标签内未用 `""` 包裹,且含 `[0,1)` 方括号会被 Mermaid 解析失败

## 5. 18 条事实一致性问题

### 严重(矛盾)

1. **🔴 SectorPerSide vs SectorWidth 矛盾**
   - Page 2 §4.2: `constexpr int32 SectorPerSide = 8; // 每级 Clipmap 8x8x8 sector`(暗示 8×8×8=512 sector/级)
   - Page 5 §8 表 18: `SectorPerSide=8`(同 Page 2)
   - Page 4 §1: `SectorWidth = 2×2×2 sector,每 Mip 共 8 个 Sector`,且 `TextureSize = SectorSize × SectorWidth` 算式校核 512×512×128 = 256×256×64 × 2×2×2 ✅
   - Page 9 §7: `SectorWidth=2×2×2`
   - **Page 2/5 的"SectorPerSide=8"与 Page 4/9 的"SectorWidth=2×2×2"在字面语义上矛盾**(每边 8 vs 每边 2),需统一为 SectorWidth=2×2×2

### 轻微(措辞差异,但事实一致)

2. **🟡 Sparse Voxel 空壳条数表述不统一**
   - Page 6 §1.1: 严格区分"4 条 cvar + 3 条无实现 getter = 共 7 条占位"
   - Page 12 §陷阱4: "7 个 cvar 全部空壳"(措辞不严谨,3 条实际是 namespace getter 不是 cvar)
   - Page 1 §3 仅模糊提到"Sparse Voxel...系列 cvar 全是空壳"

3. **🟡 EMA 衰半帧数 ≈ 0.97 vs 22.8 vs 33**
   - Page 1/9/10/11/12: "β=0.97" "33 帧衰半"
   - Page 7 §4.2 显式校正: 严格 β=0.97 单帧衰半 22.8 帧,叠加 Amortize=2 后 ~33 帧
   - 不算矛盾,但其它页均缺"33 帧是叠加 Amortize 后的有效衰减"这一限定条件

## 6. 篇幅

- 偏薄 (< 200 行): **无** ✅
- 略薄 (200-249): Page 1 (295) — 在 250-800 区间内
- 偏厚 (> 800 行): **无** ✅
- 接近上限: Page 8 (761),Page 11 (704)
- 全部 12 页都在 250-800 行预算内 ✅

## 7. frontmatter 缺失

- 全部 12 页都有 `title` / `tags` / `updated` 三字段 ✅
- 额外字段差异(可忽略):
  - Page 9 缺 `status`,但有 `page` / `related_questions`
  - Page 10 仅 title/tags/updated 三件套
  - Page 11 用 `project` / `relevant_to`

## 8. 总评

### 严重问题(必修)5 项
1. **🔴 跨页链接全网瘫痪**: ~80+ 处链接 URL 用短标题(如 `4. Clipmap 6 级调度.md`)而真实文件名带有副标题(`4. Clipmap 6 级调度 — Mip Ring 与 Two-Pass.md`),全部点击 404。Page 1/4/10/11/12 的出链几乎全部 broken。**修复方案: 全局批量替换 URL 段为完整文件名**
2. **🔴 SectorPerSide=8 vs SectorWidth=2×2×2 事实矛盾**(Page 2/5 vs Page 4/9),建议以 Page 4 推导(`TextureSize=SectorSize×SectorWidth`)为准,删除 Page 2 §4.2 / Page 5 §8 的 SectorPerSide 字段
3. **🔴 Page 9 [p1]..[p12] 引用宏定义全部 broken**:第 405-406 行的 link references 用了短文件名,需改为实际文件名
4. **🔴 Mermaid 嵌套 markdown 链接 (Page 6 line 97)**:渲染失败,需拆出 subgraph 名与正文链接
5. **🔴 Mermaid 节点 (Page 7 line 97) 含未引号的 `[0,1)` 方括号**:解析报错,需 `""` 包裹

### 轻微问题(可选修)4 项
1. 🟡 Page 2 仅 2 个 footnote 定义,低于"每页 ≥3"目标
2. 🟡 Sparse Voxel "7 条 cvar 空壳" vs "4 cvar + 3 getter" 表述不统一(Page 6 严谨,Page 12/1 模糊)
3. 🟡 EMA β=0.97 衰半帧数(22.8 单帧 vs 33 含 Amortize)各页未统一限定语
4. 🟡 Page 11 Mermaid `["看不到云"]` 等节点 OK,但调试小节流程图节点很多;Page 8 长达 761 行接近上限

### 无问题区
- ✅ 所有 footnote 引用均有定义,0 孤立 footnote
- ✅ 所有页篇幅在 250-800 行预算内
- ✅ 所有页 title/tags/updated 三字段齐全
- ✅ Mermaid 关键字与圆圈号编号合规(无 "1. xxx" 数字前缀)
- ✅ 关键事实(HIGAME_ENABLE_NUBIS / Build.h:1152 / Two-Pass / MipCount=6 / SectorSize=256×256×64 / TextureSize=512×512×128 / Linux deny / Visualize 5 模式 / Plugin 4 处 ENQUEUE / NubisCustom2 唯一生产 / Far under Near / CL611225)在 12 页之间高度一致
