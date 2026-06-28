# 双视角阴影调参：过肩第三人称 vs 2.5D 建造

- **Type**: Synthesis (aggregated from multiple sources)
- **Fetched**: 2026-06-28
- **Project**: urp-csm-cache-evsm
- **Relevant to**: Q7（两套相机下 cascade 配置/缓存命中率/失效频率/局部失效/视角切换）
- **Sources drawn from**:
  1. [Microsoft — Cascaded Shadow Maps](https://learn.microsoft.com/en-us/windows/win32/dxtecharts/cascaded-shadow-maps)
  2. [NVIDIA GPU Gems 3, Ch.10 — Parallel-Split Shadow Maps (PSSM)](https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-10-parallel-split-shadow-maps-programmable-gpus)
  3. [Microsoft — Common Techniques to Improve Shadow Depth Maps](https://learn.microsoft.com/en-us/windows/win32/dxtecharts/common-techniques-to-improve-shadow-depth-maps)
  4. [Unity Discussions — Does cached shadowmap update when the camera moves or rotates?](https://discussions.unity.com/t/does-cached-shadowmap-update-when-the-camera-moves-or-rotates/907516)
  5. [Unity Discussions — HDRP Cached Shadows are good](https://discussions.unity.com/t/hdrp-cached-shadows-are-good/1673820)
  6. [Epic Developer Community — CSM Caching and Objects Culling Questions](https://forums.unrealengine.com/t/csm-caching-and-objects-culling-questions/2684008)
  7. [Alex Tardif — Shadow Mapping](https://alextardif.com/shadowmapping.html)
  8. [Catlike Coding — Directional Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/)
  9. [Unreal Engine — Virtual Shadow Maps](https://dev.epicgames.com/documentation/en-us/unreal-engine/virtual-shadow-maps-in-unreal-engine)
  10. [Unity — Shadows in HDRP (14.0)](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@14.0/manual/Shadows-in-HDRP.html)
  11. [Unity — URP Asset (cascade/bias 参数)](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@14.0/manual/universalrp-asset.html)

> 适用：Unity 2022 LTS / URP 14。URP 14 内置阴影系统本身**不提供 cached/scrolling shadow**，下面的"缓存/失效"策略需靠**自定义 ScriptableRenderPass / 自绘 shadow RT**落地；URP 原生只能调级联划分、分辨率、距离衰减、Conservative Enclosing Sphere。每条结论标注"官方/文献支撑" vs "工程推断 ✱"。

---

## 机理总览（两类相机为什么要分开调）

CSM 核心是解决**透视走样（perspective aliasing）**：近处像素密、远处像素疏，单张图无法 1:1 映射到 shadow texel，按视距切多级、近级高密 [1][2]。两个前提决定差异：

1. **透视走样严重程度取决于相机 Z 动态范围与朝向**。Microsoft 明确：贴地相机需要的级联划分与高空/俯视机位完全不同；几何"挤在视锥一小段"（俯视、飞行模拟）时**需要的级联更少**；Z 动态范围大才需要多级、且应把更多 split 堆在近处 [1]。→ 过肩=贴地大动态范围→多级偏 log；2.5D 俯视=几何集中、动态范围小→少级偏 uniform。
2. **缓存收益取决于"光的正交投影是否逐帧稳定"**。shimmer 根因是相机一动，拟合视锥的光投影矩阵就变（尺寸变 + 位置移），shadow texel 逐帧错位 [3][7]。要稳定就得**保持投影尺寸恒定 + 按 texel 大小取整对齐（texel snapping）** [3][7]。相机越稳，缓存命中越高——这正是 2.5D 慢速/固定俯角相机的天然优势，而过肩相机的平移+旋转持续打破对齐 [5]。

---

## 过肩视角调参

视距中等、贴近角色、近景动态密集、相机持续平移+旋转。本质"贴地大 Z 动态范围 + 高失效频率"的经典动作游戏场景。

**Cascade 数量与划分**
- **4 级**，分割偏 **logarithmic**（λ 往 1 方向调，比默认 0.5 更偏 log）。机理：贴地相机近处透视走样变化最快，把更多 split 堆在近处 [1]；PSSM 论文指出纯 log 会让近级过窄、近处过采样，所以用 practical scheme（log/uniform 加权）而非纯 log，λ=0.5 是起点，近景吃重往 1 调 [2]。
- URP/SRP 默认 split 比例 0.1 / 0.25 / 0.5（占 shadow distance 的分数）是偏 log 的合理起点 [8]；过肩可把第一刀再往前压（如 0.06–0.08）让贴身近景更密。
- Shadow Distance 取**中等**（例如 40–80m）。距离越短，近级单位面积 texel 越密——对"看重近景质量"直接有效 [8]。

**分辨率分配**
- 近级（Cascade 0/1）**2048**，远级（2/3）可降到 1024。URP 按 tile 切同一张 atlas，所以"分配"通过 split 比例 + 总分辨率间接实现：近级覆盖世界范围小、texel 更密。`world-space texel size = 2 * cullingSphere.radius / tileSize` [8]——缩小近级 culling sphere（压近 split）等价于提升近级有效分辨率。
- 开 **PCF**（URP soft shadows）配合 normal bias；URP normal bias 已按 cascade texelSize × √2 缩放以覆盖对角最坏情况 [8]。

**缓存与失效（高失效场景）**
- 过肩相机平移+旋转持续改变视锥朝向与位置，directional cached shadow 几乎每帧都要更新——HDRP/社区一致：方向光缓存在 FPS/TPS 下"基本每帧重算，优化被抵消"，XR/持续运动下"方向光缓存基本不可能" [5][4]。**所以过肩不要指望靠静态缓存省方向光的全部开销**。
- 务实做法：**静动分离 + 远级降频**，而非整图缓存。
  - 动态投射者（角色、可动物）**每帧重绘**——少于每帧会 jitter/de-sync [5][6]。
  - 远级（Cascade 2/3）用**交错更新**：社区实测"60Hz 每周期只更新一个 cascade，最远级更新更稀"，正常移动下"看起来完全稳定" [5]——即近级每帧、中级每 2–3 帧、最远级每 4+ 帧。这是过肩唯一可靠的缓存式省法。
- **必做 texel snapping**：把光正交投影 min/max 按 `worldUnitsPerTexel`（= cascadeBound / bufferSize）floor 取整 + 保持投影尺寸恒定（按视锥最大尺寸 padding，即 Microsoft "fit to scene"）[3]。过肩相机会旋转，"fit to cascade"会让投影随朝向胀缩抖动，**过肩应选 fit-to-scene/stable-fit**（牺牲一点分辨率换稳定）[1][3]。URP 14 内置无 "Stable Fit" 开关（社区一直在要 [5]），自绘 pass 时务必自己实现 snapping。

---

## 2.5D 建造视角调参

相机拉远、近似正交/固定俯角、移动慢且稳定；玩家频繁放置/拆除建筑（局部几何频繁变更）。本质"几何集中、Z 动态范围小、缓存命中天然极高、但有局部脏区"的 RTS/城建场景。

**Cascade 数量、划分、正交是否还需要多级**
- **关键机理判断（部分工程推断 ✱）**：CSM 存在的理由是透视走样；正交投影下视野内 texel 密度本就**均匀**，没有"近密远疏"的透视梯度。Microsoft 明确：几何集中/俯视时"多级收益很小，3 级甚至更少就够"，Z 动态范围低"几乎没有多级的好处" [1]。
  - **取向**：2.5D 用 **1–2 级**即可，分割偏 **uniform**（λ 往 0 调）——没有近处透视梯度需要补偿，uniform 在远场覆盖更均匀 [2]。极端固定俯角可退化为**单张大正交图**覆盖整个可见建造区 ✱。
  - 注意 URP 级联是**球形 culling sphere**，正交方块投影"紧贴球并多覆盖一圈周边空间" [8]——单级时这个 padding 浪费相对小；级数越多球间重叠浪费越大，对几何集中的俯视场景更不划算，故少级正确。开 **Conservative Enclosing Sphere** 减少级联重叠的冗余 static caster [11][8]。
- 分辨率：单张图要覆盖大范围，2.5D 反而要**更大总分辨率**（2048–4096）维持大范围均匀 texel 密度；但因为更新稀，预算给得起。

**缓存命中率与失效频率（缓存收益最高的一类）**
- 2.5D 是缓存收益最高的视角：相机慢、固定俯角，光投影逐帧基本不变→cached map 命中率极高。社区共识："当前缓存实现更适合相机几乎不动的场景——比如 top-down 游戏" [5]。
- **失效触发应收窄到**：① 相机移动超过阈值距离（"frustums can be locked until the camera moves beyond a threshold distance" [4]）；② 光方向变化（昼夜/天气）——**一次性失效全部缓存**，是性能尖峰，应避免连续改方向，或只改光的颜色/强度而非方向以保住缓存 [4]；③ 局部建筑增删（见下）。
- **更新预算**：静态缓存平时 0 重绘；只在阈值跨越/光转向/脏区时付费。可设"相机平移累计 > N×texel 才触发一次 snap-重渲" [3][4]。

**2.5D 建造的局部失效（dirty region，Q7 核心）**
URP 14 内置阴影**没有** page/dirty-region 机制，但 UE5 VSM 给出可直接借鉴的工程范式：
- **机理**：VSM 把 shadow map 切成 128×128 的 **page**，缓存默认开；**只有与"发生变化的物体包围盒（从光方向投影）相交的 page 被失效**"，而非整图重绘 [9]。"移动、新增、删除"都只失效"包围盒在光视角下覆盖的那些 page" [9]——这**正是放置/拆除建筑只让被影响区域重渲**的标准做法。
- **静动双层缓存**：每个 page 存"两份深度"（静态层 + 动态层），动态物体移动时**保留静态层**，只重渲动态层再叠加 [9]。建造场景：已落成建筑进静态层缓存，正在拖拽预览/施工动画的建筑进动态层。
- **增删始终失效该 footprint**：UE 文档明确"add/remove 无论 invalidation behavior 都会失效"该投影 footprint，但**邻近 page 和其它光的 page 仍保持缓存** [9]。
- **工程要点**：① 包围盒要**紧**——松包围盒 + 浅光照角会让投影 footprint 拉长，造成大量不必要的 page 失效 [9]。2.5D 太阳压低（黄昏俯角）时尤其要小心，建筑越高、光越斜，脏区拖得越长。② per-primitive 的 `Shadow Cache Invalidation Behavior`（Auto/Always/Rigid/Static）可把"有 WPO 但实际不动"的物体钉为不失效 [9]——城建里随风摆动的旗帜/树可设 Rigid。
- URP 落地建议（工程推断 ✱）：自绘一张覆盖建造区的正交 depth/moment RT，维护一个 tile 脏标记表；放置/拆除时只把建筑 AABB 沿光方向投影到的 tile 标脏、下帧只重渲这些 tile 的静态层。把 VSM 的 page-dirty 思路降维到 URP 单图的等价实现。

---

## 视角切换

**两套独立缓存还是共享**
- **应使用两套独立的级联配置 + 独立缓存**（工程推断 ✱，但有强机理支撑）。两种相机的 split 比例、级数、分辨率、stable-fit 策略都不同（过肩 4 级 log / fit-to-scene；2.5D 1–2 级 uniform / 单大图），共享一套必然两边都不优。directional cascade 本质 **view-dependent** [10]——相机参数一换，整套 cascade 几何就变，旧缓存对新视角无效，所以"共享同一份缓存"在切换瞬间必然全失效，不如直接维护两份。
- HDRP 文档佐证：directional cascade 缓存"始终随视角变化"，官方建议方向光缓存配 **OnDemand + 频繁 `RequestShadowMapRendering`** [10]——切换相机时正是要主动触发一次完整重渲。

**避免切换瞬间的弹跳/闪烁**
- 切换瞬间 shadow distance / cascade split / 投影尺寸全变，会引发 ① 阴影锐度突变、② texel 网格重对齐导致整体跳动。规避：
  1. **切换帧强制完整重渲新配置**（OnDemand 触发），不要让旧缓存残留一帧——否则会闪一帧错位阴影 [10]。
  2. **过渡几帧插值**：在切换的若干帧内对 shadow distance / split 比例 / 分辨率做插值过渡，而非一帧硬切（工程推断 ✱，机理上等同把"投影尺寸突变"摊到多帧，避免 texel 重对齐的可见跳变 [3][7]）。
  3. **两套缓存都常驻、切换时切指针**：若显存允许，过肩与 2.5D 各保留一份 cached atlas，切换只换采样目标 + 一次重对齐，避免从零重建整图的卡顿（推断 ✱）。
  4. 切换若伴随**光方向不变**则保住静态层缓存，只重算 cascade 几何；务必不要在切换同时改光方向，否则叠加全量失效 [4][9]。
- **HDRP mixed-cache 成本提醒**：若用静动混合缓存，HDRP 每帧从 cached map blit 到 dynamic atlas，有运行时 blit 开销且**两个 atlas 各占一份显存** [10]；维护两套缓存时要把双倍显存预算算进去。

---

## Gaps（研究流 D 自报）

- **文献/官方逐字支撑**：级联划分与相机朝向关系、interval/map 选级、fit-to-scene vs fit-to-cascade、texel snapping 机制 [1][3]；PSSM split 公式与 λ=0.5 [2]；URP 4 级/默认 0.1/0.25/0.5/球形 culling/texelSize 公式 [8]；VSM page 级脏失效/双深度层/增删失效语义 [9]；HDRP ShadowUpdateMode/静动 blit 成本/directional view-dependent + OnDemand 建议 [10]；方向光缓存随相机失效、阈值锁 frustum [4]；UE 移动 CSM scroll+overlap-throttle [6]；交错"每周期一 cascade、最远级更稀"是社区**实测**而非官方文档 [5]。
- **工程常识推断（✱）**：① "正交固定俯视用 1–2 级甚至单张大图、偏 uniform"——机理由 [1][2] 强支撑，但"单张大图最优"未在某官方文档对正交相机逐字断言；② 切换时多帧插值 split/distance 防跳变；③ 维护两套常驻缓存、切换切指针；④ 把 VSM page-dirty 降维成 URP 自绘 RT + tile 脏表的具体实现。落地需自测。
- **未交叉验证 / 未取一手**：
  - **具体游戏（Cities Skylines / Anno / 等距 RTS）阴影方案**未取到一手资料——多次检索无命中，本文未据此下任何结论，"top-down 适合缓存"仅引社区共识 [5] 与机理 [1]。
  - **Insomniac SIGGRAPH 2012 "CSM Scrolling"** 原始 PDF 在本流无法解码（研究流 A/B 已成功获取，见对应笔记）。
  - **WebSearch 工具全程不可用**（持续 400），全部检索改用 WebFetch 直取 + DuckDuckGo HTML，可能漏部分一手来源（尤其具体游戏案例）。
  - URP 14 是否原生支持任何形式 cascade 缓存/scroll：依据 [5] 社区抱怨推断**否**，未取到 URP 官方明确"不支持"声明，属反向推断。
