# 开源方案盘点：URP 阴影缓存 / 静动分离 / VSM-EVSM

- **Type**: Synthesis (aggregated from multiple sources)
- **Fetched**: 2026-06-28
- **Project**: urp-csm-cache-evsm
- **Relevant to**: Q6（现有开源方案盘点、评测、推荐、拼装路径）
- **Sources drawn from**:
  1. [aivclab/CachedShadowMaps](https://github.com/aivclab/CachedShadowMaps)
  2. [danix2d/VSMShadows](https://github.com/danix2d/VSMShadows)
  3. [arcsearoc/VarianceShadowMaps](https://github.com/arcsearoc/VarianceShadowMaps)
  4. [Kabinet0/Shadow-Mapping](https://github.com/Kabinet0/Shadow-Mapping)
  5. [radishface/OnDemandShadowMapUpdate](https://github.com/radishface/OnDemandShadowMapUpdate)
  6. [Catlike Coding — Directional Shadows (Custom SRP)](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/)
  7. [raptoravis/custom-srp-04-directional-shadows](https://github.com/raptoravis/custom-srp-04-directional-shadows)
  8. [cinight/CustomSRP](https://github.com/cinight/CustomSRP)
  9. [Unity — Shadows in HDRP (14.0)](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@14.0/manual/Shadows-in-HDRP.html)
  10. [Unity — HDCachedShadowManager API](https://docs.unity.cn/Packages/com.unity.render-pipelines.high-definition@13.1/api/UnityEngine.Rendering.HighDefinition.HDCachedShadowManager.html)
  11. [TheRealMJP/Shadows](https://github.com/TheRealMJP/Shadows)
  12. [Unity-Technologies/Graphics (URP MainLightShadowCasterPass)](https://github.com/Unity-Technologies/Graphics)
  13. [Unity Discussions — URP Cached shadows (Static Shadow Caster broken)](https://discussions.unity.com/t/urp-cached-shadows/895662)
  14. [Unity Discussions — Mixing OnDemand Shadows with Dynamic Objects](https://discussions.unity.com/t/mixing-ondemand-shadows-with-fully-dynamic-objects-in-shadow-map/1643904)
  15. [Peter Stefcek — Cached Shadowmaps for URP (YouTube)](https://www.youtube.com/watch?v=aROejzpUEkc)

---

## 开源方案清单

### A 类：URP CSM 缓存 / 静动分离 / scrolling

#### A1. aivclab/CachedShadowMaps
- **作者/URL**：aivclab（Christian Heider，源自已发售游戏 *Lightmatter*）— https://github.com/aivclab/CachedShadowMaps
- **star / 更新**：83 star，12 forks，Apache-2.0。仅 12 个 commit，1 个 release，最近更新日期**未核实**（提交量极低、CI 用已停用的 `.travis.yml`，判断已基本停更）。
- **Unity/URP 版本**：README 未声明版本。**关键**：明确针对 **Built-in 管线的内置光照系统**（"caching shadow maps on unity's built in lighting system"），**不是 URP/SRP**。
- **覆盖技术点**：只有"shadow map 有效期内缓存复用 + light context 变脏时自动重算"。**没有**级联缓存、静动分离、scrolling、时间错峰中的任何一个。
- **质量评估**：有完整 UPM 包结构、有测试目录，工程化规范尚可；但技术深度浅、与 URP 范式不符。**对 URP14 几乎无直接复用价值**，最多看其"dirty 检测触发重算"的思路。

#### A2. Peter Stefcek "Cached Shadowmaps for URP"（无开源源码）
- **URL**：YouTube https://www.youtube.com/watch?v=aROejzpUEkc ；LinkedIn activity-7254384777137467392
- **状态**：这是**最贴合需求**的演示——自述"custom render feature，缓存 shadowmap 同时为动态物体维持 dynamic shadowmap"（即静动分离）。但**多轮检索确认没有公开 GitHub 仓库**。只能当**思路参考/可行性佐证**，无法当底座。

> URP 原生的 **Static Shadow Caster 勾选项**（2022.2+ MeshRenderer 上）社区实测**当前是空壳、不起作用**（Unity 论坛 895662 明确说"doesn't do anything at the moment"）——指望官方实现不行。

**结论：URP 上 CSM 缓存 / 静动分离 / scrolling 的可用开源成品基本不存在。**

### B 类：Unity Variance / EVSM 实现

#### B1. danix2d/VSMShadows
- https://github.com/danix2d/VSMShadows | **0 star**，3 commit，更新日期未核实，无 license。
- URP，用 **Render Objects feature + 自定义 depth shader** 实现 **VSM**（非 EVSM）。实验性个人项目，无文档/截图/CI。可作为"URP 下如何把深度渲成 moment 纹理"的最小样例参考，不可直接用。

#### B2. arcsearoc/VarianceShadowMaps
- https://github.com/arcsearoc/VarianceShadowMaps | **1 star**，3 commit，更新日期未核实。
- **VSM**（非 EVSM）。RG 浮点纹理存 depth/depth²，Chebyshev，含 light bleeding reduction、min variance、depth bias 等可调参，面向**方向光**。双语 README。自述"research and educational"，结构清晰（2 shader + 3 C# 脚本），无测试/CI。**B 类里对方向光 VSM 最直接的参考**，但仍教学级。

#### B3. Kabinet0/Shadow-Mapping
- https://github.com/Kabinet0/Shadow-Mapping | **1 star**，8 commit。
- URP 上的 **VSM**，**仅支持点/聚光（punctual），不支持方向光**——作者自己注明"方向光要做好看需要 EVSM"（但没实现）。亮点是 **moving-average box blur compute shader**（含 cubemap 跨边过滤）。自述"experimental / thoroughly unfinished"。**对方向光不适用**，compute blur 思路可借鉴。

**结论：Unity 上没有现成的 EVSM 实现（URP 或自定义 SRP 都没有），只有几个零散的教学级 VSM 实现。**

### C 类：可作参考的非 URP / 底座 / 官方源码

#### C1. Unity HDRP 官方 cached shadow maps（最重要的机制参考）
- 文档：https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@14.0/manual/Shadows-in-HDRP.html ；API：`HDCachedShadowManager`
- 方向光缓存支持程度：
  - 三种 Update 模式：**Every Frame / OnEnable / OnDemand**。OnDemand 通过 `RequestShadowMapRendering` 显式触发。
  - **方向光不用独立 cached atlas**，而与普通方向光**共用同一 atlas**（punctual/area 才有专用 cached atlas）。
  - **级联是 view-dependent**：相机移动/转向会在级联边缘和相机周围产生错误阴影，官方建议"相机移动时频繁调 `RequestShadowMapRendering`"。
  - **静动混合**：方向光需开 `Allow Mixed Cached Shadows`，再在灯上开 `Always draw dynamic`、把静态物标 `Static Shadow Caster`；HDRP 每帧把静态缓存 **blit 进 dynamic atlas** 再画动态 caster。**这正是"静动分离 + 合并"的官方参考实现范式**。
  - 坑：`RequestShadowMapRendering` **是 light 级、不是 per-cascade**；tessellation 等 view-dependent 效果会让缓存几何与主相机不一致。

#### C2. radishface/OnDemandShadowMapUpdate（HDRP，补足 per-cascade）
- https://github.com/radishface/OnDemandShadowMapUpdate | **21 star**，3 forks，**release 1.2.0 / 2024-02-15**，GPL-3.0，Unity 2021.3.33f1+。
- 在 HDRP OnDemand 之上做**更细粒度控制**——支持 **per-cascade 刷新**（用 `RequestSubShadowMapRendering`，弥补 C1 的 light 级局限）、相机移动驱动方向光刷新、按秒/按帧调度（**时间错峰**的直接示例）。
- 注意：GPL-3.0（传染性 license），HDRP，**思路可借鉴、代码不能直接搬进 URP 商业工程**。

#### C3. Catlike Coding "Directional Shadows"（自定义 SRP，最佳实现底座）
- https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/ ；伴随仓库 raptoravis/custom-srp-04-directional-shadows（MIT-0）
- 版本：Unity 2019.2 起，已升级到 2022.3.5f1（**与 2022 LTS 对齐**）。
- 机制覆盖：`_DirectionalShadowAtlas` 单 atlas（最多 4 光 × 4 级联 = 16 tile）、`ComputeDirectionalShadowMatricesAndCullingPrimitives`、cascade culling spheres、PCF（2×2/3×3/5×5/7×7，用 Core RP `ShadowSamplingTent.hlsl`）、cascade blend（hard/soft/dither）、distance fade、slope-scale + normal bias + near plane offset。
- **可作底座的原因**：`Shadows` 类自带 Setup/Render/Cleanup 生命周期且自包含；`ReserveDirectionalShadows` 已把"哪些光要阴影"与"何时渲染"解耦，天然适配"只重画脏光/脏级联"；tile/atlas 管理（`SetTileViewport`/`ConvertToAtlasMatrix`）已封装；per-tile shadow matrix 已存数组可跨帧持久化。
- **改造成本**：要把 `GetTemporaryRT`（每帧释放）换成**持久 RenderTexture**、tile 槽位**稳定分配**、加 **per-light/per-cascade dirty 跟踪**。**最现实的"从零搭可缓存方向光阴影"底座**。

#### C4. cinight/CustomSRP（自定义 SRP 大全，机制参考）
- https://github.com/cinight/CustomSRP | **646 star**，78 forks，149 commit，活跃（声明支持 6000.3.11f1+，老版本在其他分支）。
- 含 `SRP0603_RealtimeShadowDirectional` 等场景，覆盖 RTHandle、RenderPass、compute、callbacks 等基础设施样例。**没有现成级联缓存**，但理解"在自定义 pass 里管理 RTHandle/atlas"的高质量参考库。

#### C5. TheRealMJP/Shadows（EVSM/MSM 的权威移植源）
- https://github.com/TheRealMJP/Shadows | **~1k star**，81 forks，MIT，最新 release 2024-04-07。配套博客 therealmjp.github.io/posts/shadow-maps/
- 平台：**DirectX 11 / C++ + HLSL，不是 Unity**。实现 CSM、Stabilized CSM、SDSM 自动级联拟合、各种 PCF、**VSM、EVSM、MSM**（博客含 EVSM4 正负项讨论）。
- **业界做 EVSM/MSM 移植到 Unity 的事实标准参考**。moment 计算、exponential warp、filtering 的 HLSL 数学可直接搬；需重写主机侧（cascade setup、render pass 换成 SRP CommandBuffer）并处理 reversed-Z / 深度精度约定。

#### C6. Unity 官方 URP 源码 `MainLightShadowCasterPass`
- https://github.com/Unity-Technologies/Graphics（`Packages/com.unity.render-pipelines.universal/Runtime/Passes/MainLightShadowCasterPass.cs` 及 cascade 相关结构）
- 你要改造的就是这条 pass。URP 用**方向光独立 atlas**（与 punctual 分开），cascade 切分、`SetupMainLightShadowReceiverConstants` 都在这里。**这是旧 ScriptableRenderPass + RTHandle 路线的直接改造对象**，务必拿对应 URP14 tag 源码对照（14.x 还是 RTHandle、未上 RenderGraph，符合约束）。

---

## 对比与推荐表

| 方案 | 维护状态 | URP14 兼容 | 覆盖技术点 | 性能 | 上手/改造成本 | 定位 |
|---|---|---|---|---|---|---|
| aivclab/CachedShadowMaps | 停更(未核实) | ❌ Built-in 管线 | 仅 dirty 重算缓存 | 中 | 低用价值 | **仅参考**（dirty 思路） |
| Peter Stefcek URP cached | 无源码 | — | 静动分离(演示) | 未知 | 不可得 | **仅参考**（可行性佐证） |
| danix2d/VSMShadows | 实验/停 | ⚠️ URP 但极简 | VSM | 中 | 中 | 仅参考（moment 渲染样例） |
| arcsearoc/VarianceShadowMaps | 实验/停 | ⚠️ 需适配 | VSM(方向光) | 中 | 中 | **VSM 改造起点候选** |
| Kabinet0/Shadow-Mapping | 未完成 | ⚠️ URP punctual | VSM + compute blur | 中 | 中 | 仅参考(blur 思路) |
| HDRP cached（官方） | ✅ 官方维护 | ❌ HDRP | 级联缓存/OnDemand/静动混合 blit | 高 | 不可移植但范式权威 | **机制参考(必读)** |
| radishface/OnDemandShadowMapUpdate | ✅ 2024 | ❌ HDRP/GPL | per-cascade + 时间错峰 | 高 | GPL 限制 | **机制参考**(错峰/分级刷新) |
| **Catlike Directional Shadows** | ✅ 教程在维护 | ✅ 2022.3 自定义SRP | CSM/PCF/blend/bias 全套 | 高 | 中(需加缓存层) | **改造底座(首选)** |
| cinight/CustomSRP | ✅ 活跃 646★ | ✅ | RTHandle/RenderPass 基建 | — | 低(参考) | 基建参考 |
| **TheRealMJP/Shadows** | ✅ 2024 | ❌ D3D11 | EVSM/MSM 权威 | 高 | 高(移植 HLSL) | **EVSM 移植源(首选)** |
| **URP MainLightShadowCasterPass** | ✅ 官方 | ✅ 你要改的就是它 | URP 原生 CSM | 高 | 高 | **改造对象** |

**文字推荐**：
- **直接能用的**：没有。开源界没有 URP14 上即插即用的 CSM 缓存或 EVSM 成品。
- **改造底座**：(1) **走自定义/扩展 SRP** → Catlike Directional Shadows（C3，MIT-0，机制最透、版本对齐、atlas/tile/matrix 都已模块化）。(2) **走改 URP 内置 pass（更贴合"旧 ScriptableRenderPass + RTHandle"约束）** → 直接改 `MainLightShadowCasterPass`（C6），把临时 atlas 换持久 RTHandle、加 per-cascade dirty + scrolling。
- **机制必读参考**：HDRP cached（C1，静动 blit 合并范式）+ radishface（C2，per-cascade 刷新 + 时间错峰现成调度逻辑）+ TheRealMJP（C5，EVSM/MSM 数学）。
- **避坑**：
  1. **别指望 URP 的 Static Shadow Caster 勾选项**——2022.2+ 上是空壳，社区实测无效。
  2. **方向光级联是 view-dependent**——缓存后相机一动级联边缘就崩，必须配 scrolling（卷动 texel-snap）或相机移动阈值触发重画。
  3. **radishface 是 GPL-3.0**——只读思路，别拷代码进闭源工程。
  4. **HDRP `RequestShadowMapRendering` 是 light 级**，per-cascade 要用 `RequestSubShadowMapRendering`。
  5. **TheRealMJP 是 D3D11**——移植要处理 reversed-Z / 深度精度 / sampler 约定差异。
  6. VSM/EVSM 的 **light bleeding**：B 类几个 repo 都只到 VSM，漏光严重；要软阴影质量必须上 EVSM（指数 warp 抑制漏光），这块只能自己从 TheRealMJP 移植。

---

## 拼装路径

**"EVSM + 缓存组合"在开源界没有任何成品**（多轮交叉检索确认：HDRP cached 系统不含 VSM/EVSM；几个 Unity VSM repo 都不带缓存；TheRealMJP 带 EVSM 但不带缓存且非 Unity）。最现实的拼装——**缓存层 + 滤波层（EVSM）两段拼**：

1. **缓存底座**：以 **Catlike Directional Shadows（C3）** 的 `Shadows` 类为骨架，或直接在 **URP `MainLightShadowCasterPass`（C6）** 上改——核心改三处：① `GetTemporaryRT` → 持久 RTHandle；② 引入稳定 tile 槽位 + per-light/per-cascade **dirty 标记**；③ 加 **scrolling**（按级联 texel size 做 world-space snap，相机平移时卷动而非全重画）。
2. **时间错峰 + per-cascade 刷新调度**：照搬 **radishface（C2）** 的调度思路（per-cascade 轮转、按秒/帧分摊；近级联高频、远级联低频）——读逻辑、自己用 URP API 重写（避开 GPL）。
3. **静动分离合并范式**：照搬 **HDRP cached（C1）** 的 "静态缓存 blit 进 dynamic atlas，每帧只补画动态 caster" 做法。
4. **EVSM 软阴影滤波**：把 **TheRealMJP/Shadows（C5）** 的 EVSM HLSL（moment 生成 + exponential warp + Chebyshev + LBR）移植成 URP shader；moment 纹理用 **RGBA float32**（EVSM 必须 fp32），对其做大核 blur（box/Gaussian，可参考 Kabinet0 的 compute blur）。**关键拼接点**：EVSM 存 moment 而非 depth，所以"静态缓存"缓存的是**静态 moment 纹理**，与动态 moment 的合并**不能像普通 depth 那样简单 blit/min**——需按本项目 Q5 方案在**可见性空间** min/乘合并，这一步无任何现成实现，是整个组合里最需要自研验证的环节。

简言之：**Catlike/URP-pass 出缓存与 scrolling，radishface 出错峰调度，HDRP 出静动合并范式，TheRealMJP 出 EVSM 数学**——四块拼，无一可整体复用。

---

## Gaps（研究流 C 自报）

- **URP 上的 CSM 缓存/静动分离/scrolling 开源成品：没找到。** 最贴近的 Peter Stefcek 演示无公开源码；aivclab 那个是 Built-in 管线。
- **任何 Unity（URP 或自定义 SRP）的 EVSM 实现：没找到。** 只有 3 个教学级 VSM repo（其中 2 个 0–1 star）。EVSM 唯一权威源是非 Unity 的 TheRealMJP（D3D11）。（补充：主 agent 另查到 LVSM 实现 szaboeme/layered_variance_shadow_maps 与 EVSM 实现 martincap.io，均非 Unity/无缓存。）
- **"EVSM + 缓存"组合：开源界完全不存在**，需按上面拼装路径自研，尤其 moment 空间的静动合并语义无先例可循。
- **多个小 repo 的精确 last-update 日期 / Unity 版本未核实**：aivclab/danix2d/arcsearoc/Kabinet0/raptoravis 的日期、部分 Unity 版本均**未核实**。star 数为抓取当下值。
- **未深入读取的潜在线索**：StressLevelZero/Custom-URP（生产级 URP fork）、Echoes of Somewhere 博客（URP custom shadow 实战，2023 后转 shader graph）——值得后续单独深挖。
- WebSearch 工具本轮全程报 schema 错误，检索改用 DuckDuckGo HTML via WebFetch；`gh` CLI 本机未安装，未能用 GitHub API 精确核对 star/日期。
