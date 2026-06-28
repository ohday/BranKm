# URP CSM 缓存机制：静/动分离、卷动更新、时间错峰与脏区失效

- **Type**: Synthesis (aggregated from multiple sources)
- **Fetched**: 2026-06-28
- **Project**: urp-csm-cache-evsm
- **Relevant to**: Q1（静/动分离 + min-depth 合并 + URP14 持久化 RTHandle）、Q2（卷动更新 + texel snapping）、Q3（时间错峰 + 脏区失效）
- **Sources drawn from**:
  1. [Insomniac Games — "CSM Scrolling" (SIGGRAPH 2012, Acton/Day/Hastings)](https://advances.realtimerendering.com/s2012/insomniac/Acton-CSM_Scrolling(Siggraph2012).pdf)
  2. [Unity — Shadows in HDRP (GitHub Documentation~)](https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.render-pipelines.high-definition/Documentation~/Shadows-in-HDRP.md)
  3. [Unity — Control shadow resolution and quality (HDRP 17.1)](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@17.1/manual/Shadows-in-HDRP.html)
  4. [Unity Discussions — Mixing OnDemand Shadows with Fully Dynamic Objects](https://discussions.unity.com/t/mixing-ondemand-shadows-with-fully-dynamic-objects-in-shadow-map/1643904)
  5. [Unity Discussions — HDRP Cached Shadows are good](https://discussions.unity.com/t/hdrp-cached-shadows-are-good/1673820)
  6. [Unity Discussions — (URP 13.1.8) Proper RTHandle usage in a Renderer Feature](https://discussions.unity.com/t/urp-13-1-8-proper-rthandle-usage-in-a-renderer-feature/895405)
  7. [Unreal Engine — Virtual Shadow Maps](https://dev.epicgames.com/documentation/en-us/unreal-engine/virtual-shadow-maps-in-unreal-engine)
  8. [NVIDIA GPU Gems 2, Ch.2 — Terrain Rendering Using GPU-Based Geometry Clipmaps (Hoppe/Asirvatham)](https://developer.nvidia.com/gpugems/gpugems2/part-i-geometric-complexity/chapter-2-terrain-rendering-using-gpu-based-geometry)
  9. [Microsoft — Common Techniques to Improve Shadow Depth Maps](https://learn.microsoft.com/en-us/windows/win32/dxtecharts/common-techniques-to-improve-shadow-depth-maps)
  10. [Alex Tardif — Shadow Mapping](https://alextardif.com/shadowmapping.html)
  11. [Microsoft — Cascaded Shadow Maps](https://learn.microsoft.com/en-us/windows/win32/dxtecharts/cascaded-shadow-maps)
  12. [Unity Discussions — Does cached shadowmap update when the camera moves or rotates?](https://discussions.unity.com/t/does-cached-shadowmap-update-when-the-camera-moves-or-rotates/907516)
  13. [Unity — Set a shadow update mode (HDRP 16.0)](https://docs.unity.cn/Packages/com.unity.render-pipelines.high-definition@16.0/manual/shadow-update-mode.html)

> 下文 [n] 引用对应上面编号（注：本笔记沿用研究流内部编号，与 sources 列表逐条对应）。

---

# Q1 — 静/动分离缓存：静态阴影图渲一次，动态每帧叠加

## 机理详述

### 标准合并做法：blit 静态深度 → 在其上渲动态（不是 receiver 端 min）

业界两条主流路线，**实际产品里占主导的是第二条**：

1. **两张 RT，receiver 端取 min**：静态投射体渲进 `staticSM`，动态投射体每帧渲进 `dynamicSM`，光照 shader 里对同一个 light-space UV 各采样一次，取「更近的遮挡者」即 `shadow = min(shadowTest(staticSM), shadowTest(dynamicSM))`。代价是接收端两次采样 + 两张常驻 RT。[1][2]

2. **单张 dynamic atlas，先 blit 静态再在其上渲动态（HDRP / Insomniac 都走这条）**：把缓存的静态阴影图 **blit（拷贝深度）到本帧的 dynamic atlas**，然后用标准深度测试把动态投射体直接渲到这张已含静态深度的 RT 上。Unity 官方原文：「If you set up a shadow to be mixed cached, HDRP performs a blit from the cached shadow map to the dynamic atlas」，且「dynamic shadow casters into their respective shadow maps each frame」「both are composed together」。[3][4][5]

   **这一步本质上就是 `min(depth)`**：blit 后 RT 里是静态遮挡者深度；动态投射体用 `LEQUAL`/`LESS` 深度测试渲入，只有当动态投射体比已有静态深度**更近**时才覆写——即每个 texel 最终保留 `min(static_depth, dynamic_depth)`，最近遮挡者胜出。深度测试硬件天然实现了 min，不需要在 receiver 端做两次采样。这是为什么路线 2 在接收端只采一次、更省。Insomniac 把同一动作描述为「Copy map to use as final shadow map for current frame」→「Render non-static geometry into final shadow map for frame」。[1][6]

   代价（Unity 明确点出）：blit 本身有运行时开销，且「a single shadow map requires space in both atlases」——静态缓存图和 dynamic atlas 各占一份显存，内存翻倍；另外每个动态物体在它出现的每张阴影图里都增加一个 draw call。[3][7]

   HDRP 开启路径（可直接照搬到 URP 自实现的概念模型）：Light → Shadows → 勾 **Always draw dynamic**；所有要缓存的 Renderer 勾 **Static Shadow Caster**；方向光额外要在 HDRP Asset 里开 **Allow Mixed Cached Shadows**。[3][4]

### URP14 下持久化 RTHandle（跨帧持有、不被帧末释放）

URP13/14 旧 `ScriptableRenderPass` 路线（非 RenderGraph）持有跨帧 RT 的正确模式：[8]

- **RTHandle 字段挂在 Pass 实例上，Pass 由 Feature 在 `Create()` 里 new 一次并复用** → 字段自然跨帧存活。这是「持久化」的根：不是靠特殊 API，而是靠对象生命周期。`cmd.GetTemporaryRT`/`ReleaseTemporaryRT` 那套**不适用**于 RTHandle。[8]
- **分配 API 二选一**：
  - 尺寸固定且生命周期内不变（缓存阴影图正是这种）→ 一次性 `RTHandles.Alloc(...)`，避免每帧检查开销。
  - 尺寸可能随屏幕变 → `RenderingUtils.ReAllocateIfNeeded(ref handle, ...)`，仅当 descriptor 变化才重分配，否则保留现有 handle。可安全放在 `OnCameraSetup` 里调。[8]
- **绝不在 `OnCameraCleanup` 里 Release**——这正是原帖 VRAM 泄漏/被错误释放的根因。借来的 handle（如 `cameraColorTargetHandle`）只置 null 解引用，**不 Release**（你不拥有它）。[8]
- **释放时机**：自有 handle 在 `Feature.Dispose(bool)` → 转发到 `Pass.Dispose()` 里 `handle?.Release()`。这个点在管线重载 / domain reload / 退出时触发，不是每帧。[8]
- **坑**：`ReAllocateIfNeeded` 只会**放大**不会缩小（RTHandle 系统保留历史最大尺寸）；要缩小必须显式 `Release()` 后重分配。色彩与深度不能共用一个 RTHandle（`depthBufferBits=0`）。[8]

### 静态缓存何时失效（invalidation 触发条件）

跨来源高度一致：[6][9][3]

- **光方向/形状变化** → Insomniac 直接列「light direction and shape relatively stable」为前提假设；Unreal VSM 文档更绝对：「Any movement or rotation of a light invalidates **all** cached pages for that light」——方向光只要转一下，整张缓存全失效、全重渲。[6][9]
- **相机移动 / FOV 变化** → Insomniac 把这两条列为 cache「Invalid if…」。这是 Q2 卷动要解决的核心。[6]
- **静态几何增删/移动** → Insomniac「'Static' geometry moves」失效；Unreal「Shadow-casting geometry moving / added / removed invalidates any pages that overlap its bounding box from the light's perspective」——只失效**与其包围盒在光视角下重叠的页**，不是整张（这正是 Q3 脏区失效的基础）。[6][9]
- **「static」的定义带时间滞后**：Insomniac 定义 static = 「not moved for t time (e.g. 5 seconds)」——物体静止超过阈值时间才被收编进静态缓存，避免刚停下就重渲。Unreal 对应的是 per-primitive `Shadow Cache Invalidation Behavior` 枚举：Auto / Always / Rigid / **Static**（Static = Rigid + 抑制 transform 变化失效，只有 Add/Remove 才失效）。[6][9]
- Unreal 还列出会**每帧失效**的「永不缓存」类：WPO/PDO 材质、骨骼动画、tessellation——这些动态量在 render 前不可知，缓存必须假设要重画。[9]

---

# Q2 — 卷动更新 + texel snapping

## 机理详述

### 卷动（scrolling / toroidal）——只重渲新露出的边条

核心来自 Insomniac Games SIGGRAPH 2012「CSM Scrolling」(Mike Day / Mike Acton / Al Hastings)，这是该技术最权威的一手演讲。流程：[1]

```
每帧（相机缓慢移动时）:
  1. Scroll cached map  —— 按相机在光空间的位移，平移缓存阴影图
  2. Render into exposed edges —— 只把新露出边条里的静态几何渲进去
  3. 渲新晋 static 几何进缓存区
  4. Copy map → 作为本帧 final shadow map
  5. Render non-static (动态) 几何进 final（= Q1 的合并）
```

相机运动是 **3D**，要拆成两类处理：[1]

- **Lateral scrolling（垂直于光线的平移）**：直接把缓存图按「相机在光坐标系下的 delta」做 **UV 平移**，纯点采样（point sampling）查上一帧 texel 即可。新滚入区域用 clamp-to-border、color = 1.0（即「无遮挡/全亮」）。
- **Depth scrolling（平行于光线的平移）**：除了平移，还要**把所有历史深度值整体偏移**（offset all previous depths by delta-camera-depth-in-light-frame）。Gotchas：near plane 处要 **clamp 到 0.0**；far plane 处 1.0 = buffer clear 值会和真实远点冲突，需特殊处理。[1]

**新露出区域的重渲方式**：把滚入的边条切成 **slabs（细长 OBB）**，凡是 bounding volume 与某 slab 重叠的静态几何就渲进去。演讲强调：相对视野，几何越「粗」（tile 越大）重叠卷积越多——所以用方形几何 tile 能让重叠最少。最终效果：**约 70% 的 static 几何不必重渲**，每张图 512×512（PS3/360）。[1]

**环形寻址（toroidal addressing）的精确数学**——Insomniac 演讲把它类比「2D bitmap scrolling」但没贴模运算公式；其底层机制的权威出处是 Hoppe 的 GPU-Based Geometry Clipmaps（GPU Gems 2 Ch.2），阴影卷动与之同构：[8]

- 固定 n×n 纹理缓存一个**移动的空间窗口**，「the clipmap window is accessed **toroidally**, that is, with **2D wraparound addressing**」。[8]
- 不去**平移已有 texel**（太贵），而是**移动窗口的逻辑原点**，texel 地址对纹理维度取模（modulo）回绕。「we use **wraparound addressing** to position the clipmap window within the texture image」。[8]
- 相机移动后只需更新一条 **L 形** 新露出带；在环形寻址下，这个 L 形通常**变成 + 形**（因为带子跨越纹理边界回绕），用 **2 个 quad** 渲完（跨边界时需 3~4 个 quad）。这正是 Insomniac「render into exposed edges」的纹理空间实现。[8]
- 采样时 `uv = gridPos * (1/w, 1/h) + originInTexture`，回绕交给采样器的 wrap 寻址模式。粗级别窗口内相对运动指数级减小，**粗级别极少需要更新**——这条直接支撑 Q3「远级联更新更稀」。[8]

### 光空间 texel snapping —— 缓存与防抖的前提

**为什么是前提**：相机一动，光投影矩阵每帧重算，produce 亚 texel 抖动；缓存图若不 snap，复用时静/动接缝处会 shimmer。snap 把投影原点**量化到光空间纹理栅格的整数倍**，使 texel 栅格只能整 texel 跳，从而（a）边缘不再 shimmer，（b）卷动时上一帧 texel 能 1:1 对齐复用。[9][10]

**经典公式（Microsoft「Common Techniques to Improve Shadow Depth Maps」，对方向光）**：把构成正交投影 X/Y 边界的 min/max 各 round 到 texel 大小整数倍——divide / floor / multiply：[9]

```c
worldUnitsPerTexel = cascadeBound / shadowBufferSize;   // 一个 texel 覆盖的世界距离
orthoMin /= worldUnitsPerTexel;  orthoMin = floor(orthoMin);  orthoMin *= worldUnitsPerTexel;
orthoMax /= worldUnitsPerTexel;  orthoMax = floor(orthoMax);  orthoMax *= worldUnitsPerTexel;
```

关键约束：**投影尺寸必须每帧恒定**，snap 才有意义——所以要么用「fit to scene」（按视锥最大尺寸 pad，尺寸恒定），要么按**包围球**拟合（见下）。纹理还要在宽高各**多 1 像素**，防 snap 后采样越界。[9]

**旋转不变的现代写法（Alex Tardif，含完整代码）**——过肩视角必须用这个，因为它对相机 yaw/pitch 恒定：[10]

```c
// 用包围球而非视锥角的 AABB —— 球的直径与相机朝向无关，投影范围旋转不变
worldUnitsPerTexel = (sphereRadius * 2.0) / shadowMapResolution;  // 直径 / 分辨率

lightPos  = frustumCenter - lightDir * sphereRadius;
lightView = LookAtLH(lightPos, frustumCenter, up);

centerLS  = TransformCoord(frustumCenter, lightView);             // 球心进光空间
centerLS.x = floor(centerLS.x / worldUnitsPerTexel) * worldUnitsPerTexel;   // 吸附 X
centerLS.y = floor(centerLS.y / worldUnitsPerTexel) * worldUnitsPerTexel;   // 吸附 Y
// z 不 snap，深度范围由 sphereRadius 或 pancaking 给

L = centerLS.x - sphereRadius;  R = centerLS.x + sphereRadius;
B = centerLS.y - sphereRadius;  T = centerLS.y + sphereRadius;
lightProj = OrthoOffCenterLH(L, R, B, T, centerLS.z - sphereRadius, centerLS.z + sphereRadius);
```

为什么用球不用紧致视锥拟合：紧致 AABB 的尺寸**随相机旋转变化**→ texel 世界尺寸抖动 → shimmer；球直径恒定，代价是浪费部分纹理空间（Tardif 明确承认 trade-off）。逐像素级联选择也因此从 box test 简化为对各级联直径的**距离判定**。[10]（Microsoft「fit to scene vs fit to cascade」给的是同一动机的另一种表述：fit to cascade 投影会随视锥朝向胀缩，fit to scene 按最大尺寸 pad 来消除该 artifact。[11]）

### 相机旋转（过肩）相比纯平移的额外失效

- 纯**平移**只需卷动 + 整 texel snap，缓存大部分可复用。[1]
- **旋转**会改变光投影的拟合范围：若用紧致视锥拟合，投影尺寸变 → texel 世界尺寸变 → snap 失效、缓存对不齐 → 抖动。**所以包围球拟合（旋转不变）是过肩视角缓存的硬前提**。[10][11]
- 即便如此，**光自身的旋转**（如昼夜太阳转动）会让 Insomniac/Unreal 直接判定整张缓存失效——光转动 ≠ 相机转动，前者无解只能整体重渲，后者可被球拟合 + 卷动吸收。[1][9]

---

# Q3 — 时间错峰 + 脏区失效

## 机理详述

### 远级联错帧更新策略

实践者一致方案（Unity Discussions 多位 + Unreal）：[7][12][1]

- **轮转（round-robin）：每帧只更新一个级联**。实测配置：更新锁 60Hz，每个 tick 轮一个 cascade，「the **furthest cascade is actually updated even less frequently**」，主观「perfectly stable and accurate」，小幅相机移动/旋转下无抖动、无 desync。[7]
- **为什么远级联可以更稀**：远级联覆盖**更大世界区域**，同样的相机位移在它上面产生的像素级变化按比例**更小**——所以降频肉眼不可见。Hoppe 的 clipmap 结论同构印证：「the relative motion of the viewer within the windows **decreases exponentially at coarser levels**, so coarse levels seldom require updating」。这就是「远级联每 K 帧更新因为它覆盖大、texel 稳定」的机理依据，不是拍脑袋。[7][8]
- **触发策略三选一/可组合**：
  1. **固定每 K 帧**（轮转计数器，近级联 K=1、中 K=2、远 K=4…，按覆盖范围递增）；[7]
  2. **按相机移动量阈值**——「the frustum of shadow maps can be **staggered and locked in place until the camera moves a sufficient distance**」；Unity HDRP 暴露两个具体阈值属性：`cachedShadowTranslationUpdateThreshold`（平移）、`cachedShadowAngleUpdateThreshold`（旋转角度），超阈值才重渲；[12][13]
  3. **按距离**——级联越远更新越稀（即上面 K 随级联序号递增）。[7]
- **近级联 vs 远级联频率**：近级联（含/紧贴角色脚下）必须**每帧**——「dynamic objects must update every frame, anything less immediately causes visible jitters and de-sync」；远级联可降到每 2/4/N 帧。**注意：这里的「每帧」指动态投射体那一层（Q1 的 dynamic atlas），静态缓存层才是被错峰的对象**。[7]
- **API 级支持（直接可用）**：HDRP `RequestSubShadowMapRendering(shadowIndex)` 可**只刷新单个 sub-shadow（即单个级联）**，配合 `Update Mode = OnDemand` 实现你自己的轮转调度器；整光刷新用 `RequestShadowMapRendering()`。`onDemandShadowRenderOnPlacement=false` 可抑制首帧自动渲染，完全交给你的调度。[13]

### 局部失效 / 脏区重渲（2.5D 放置/拆除建筑）

- **基础机制**：增删一个静态投射体时，**只失效与其包围盒在光视角下重叠的区域/页**，不是整张缓存。Unreal VSM 原文：「Shadow-casting geometry moving, being added, or removed invalidates **any pages that overlap its bounding box from the light's perspective**」——VSM 把阴影图切成 page，逐 page 失效，天然就是脏区粒度。[9]
- **VSM 的可视化/诊断（实现脏区时照抄思路）**：`r.Shadow.Virtual.Stats` 里 **Invalidated** 计数（上一帧被动态几何作废的已缓存页，理想静态页失效数应≈0）、**Merged**（static+dynamic 拷贝因一方未缓存而合并的页）；`ShowFlag.VisualizeShadowCasters` 显示哪些物体/包围盒在驱动失效——**包围盒要尽量紧**，低光照角度下被拉长的包围盒会造成大量无谓失效（2.5D 斜俯视角尤其要注意）。[9]
- **在 atlas/单图缓存里做脏区**：HDRP 的对应物是 `DefragAtlas()`（把所有 atlas 图标脏，光可见时重渲）与 per-light 重渲请求；细到「只重渲被改的那一小块」时，标准做法是**用 scissor rect / 局部 viewport 把失效矩形圈出，只在该矩形内重渲受影响的静态投射体**，其余 texel 保留缓存——这与 Insomniac「render into exposed edges（slabs）」是同一种「只画一个轴对齐矩形区域」的手法，只不过触发源从「相机卷动」换成「建筑增删的包围盒投影矩形」。[3][1]
  - 具体步骤（综合）：① 取被增删建筑的世界包围盒 → ② 投影到该级联的光空间，求其 texel 空间 AABB（dirty rect）→ ③ 设 scissor/viewport 为该 rect → ④ 清该 rect → ⑤ 重渲 bounding volume 与该 rect 重叠的所有静态投射体（拆除时只需清+重渲邻近静态体，放置时还含新建筑本身）→ ⑥ 其余区域缓存不动。[1][9]

---

## Gaps（研究流 A 自报）

1. **Receiver 端 `min(两张 SM)` 的逐字代码/一手出处缺失**：路线 1（两张 RT、shader 里取 min）能从机理推断并由 Insomniac/HDRP 的 blit 路线交叉印证「最近遮挡者胜出」，但**没找到贴出 `min(shadowA, shadowB)` receiver 代码的权威单页**——业界主流（HDRP/Insomniac）都走 blit-then-render 而非 receiver min，所以路线 1 仅单一推断、未一手验证。
2. **HDRP blit 是否严格为 `CopyDepthBuffer`（深度拷贝）未被官方逐字确认**：官方只说「performs a blit」，没写明深度拷贝还是其它。「blit 深度 + LEQUAL 渲动态 = min」是据深度测试语义的合理推断，需读 HDRP `HDShadowManager`/`HDCachedShadowManager` 源码确认。
3. **CryEngine `e_ShadowsCache` / GSM 一手细节全缺**：docs.cryengine.com 全部 302 跳转到 SPA 落地页，抓不到正文。该主题经典实现之一，但无可引用一手来源，本报告未纳入。
4. **远级联「每 K 帧」的 K 具体取值无权威基准**：轮转方案有实践者证言 + clipmap 指数级理论支撑，但「近 K=1 / 中 K=2 / 远 K=4」是综合推断的工程默认值，需在目标项目实测标定。
5. **Insomniac depth-scroll 的 near/far clamp 精确数值/着色器代码缺失**：演讲只给了提示性 gotcha，无完整公式。
6. **2.5D 脏区 scissor-rect 局部重渲为综合做法**：步骤①~⑥是把 Insomniac「render into exposed edges」与 Unreal「per-page bounding-box 失效」两条一手机理**合成**的实现路径，逻辑自洽但属跨源综合，未一手验证。
