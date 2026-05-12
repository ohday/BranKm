---
title: "NubisCloud Temporal + Bilateral Upscale + Visualize"
type: Local code archaeology
fetched: 2026-05-11
project: higame-nubis-cloud
relevant_to: A8, E1
sources_drawn_from:
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisRenderTargetViewStateData.h:23-289   # FNubisPerLevelDitherState
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisRenderTargetViewStateData.h:292-419  # FNubisRenderTargetViewStateData
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisVolumes.cpp:48-53                    # CVarNubisVolumesJitter
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisVolumes.cpp:177-200                  # CVar NearDominance
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisVolumes.cpp:507-510                  # ShouldJitter()
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisVolumes.cpp:561-662                  # FNubisRenderTargetViewStateData::Initialise
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisVolumes.cpp:1566-1808                # Visualize Pass + CVars
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisVolumesLiveShadingPipeline.cpp:957-1043  # FNubisBilateralUpscaleCS
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisVolumesLiveShadingPipeline.cpp:2658-2698  # PerLevelDitherState init
  - Engine/Shaders/Private/NubisVolumes/NubisVolumesReconstruct.usf:319-336                       # GetHistoryScreenPosition
  - Engine/Shaders/Private/NubisVolumes/NubisVolumesReconstruct.usf:368-540                       # Path A/B/C/D
  - Engine/Shaders/Private/NubisVolumes/NubisVolumesBilateralUpscale.usf:407-549                  # UpscaleCloud Mode 0
  - Engine/Shaders/Private/NubisVolumes/NubisVolumesBilateralUpscale.usf:551-678                  # UpscaleCloud4x4
  - Engine/Shaders/Private/NubisVolumes/NubisVolumesBilateralUpscale.usf:698-784                  # UpscaleFarCloud + Depth Occlusion
  - Engine/Shaders/Private/NubisVolumes/NubisVolumesBilateralUpscale.usf:786-856                  # NubisVolumesBilateralUpscaleCS + Far under Near
  - Engine/Shaders/Private/NubisVolumes/NubisVolumesVisualize.usf:9-175                           # Visualize 模式 + LUT
---

> **范围**：补全 raw#3 的 Reconstruct / BilateralUpscale 章节细节；正式答复 key_questions A8（ViewState 历史纹理 + Bilateral Upscale 数学）和 E1（Visualize 模式 / Jitter）。本笔记**不重复** raw#1 的总览与 raw#3 的 RDG 流程图，引用即可。

---

## A. ViewState 历史纹理结构 (A8)

### A.1 双层结构：Per-View Wrapper + Per-Level State

`FViewInfo::ViewState` 内嵌一个 **`FNubisRenderTargetViewStateData`** 实例（`NubisVolumeRenderTarget`），它进一步根据 Clipmap MipLevel 把 RT 拆分到 **`FNubisPerLevelDitherState`** 子结构里：

```
FViewInfo
  └─ ViewState
       └─ FNubisRenderTargetViewStateData NubisVolumeRenderTarget
            ├─ Reconstruct/Far Tracing/Combined RT[]            ← 全局共享（旧路径）
            ├─ Instability RT[2]                                 ← FSR2-style 双缓冲
            ├─ FrameId / NoiseFrameIndex / CurrentPixelOffset    ← Bayer 偏移帧计数
            ├─ CurrentRT (0/1)                                   ← 全局双缓冲 ping-pong 索引
            └─ TMap<int32, FNubisPerLevelDitherState> PerLevelDitherStates  ← Clipmap 多级分别独立
                  └─ Key = MipLevel (典型 2,3,4)
                       ├─ FrameIndex / CurrentPixelOffset / CurrentRT   ← 该 Level 自己的帧计数
                       ├─ DitherFrameCount / DownsampleFactor           ← Bayer Pattern 大小
                       ├─ ReconstructRT[2] + ReconstructRTDepth[2]     ← Per-Level 双缓冲
                       ├─ InstabilityRT[2]                              ← Per-Level FSR2
                       ├─ FarTracingRT + FarSecondaryTracingRT          ← 单缓冲（每帧重写）
                       └─ FarTracingRTDepth                             ← 单缓冲
```

> 来源：`NubisRenderTargetViewStateData.h:292-419` (Per-View) + `:23-289` (Per-Level)。

### A.2 RT 列表与作用

| RT | 双缓冲 | 格式 | 用途 |
|----|------|------|------|
| `ReconstructRT[0/1]` | ✅ ping-pong | `PF_FloatRGBA` | Reconstruct 输出（云预乘 RGB + α）；上一帧作为 history 被 Reproject 采样 |
| `ReconstructRTDepth[0/1]` | ✅ ping-pong | `PF_G16R16F` | (CloudFront km, Scene km)；reproject 时取 history 深度做 dis/occlusion 测试 |
| `InstabilityRT[0/1]` | ✅ ping-pong | `PF_R16F` | EMA 累积亮度抖动，用于软切换（启用 `NUBIS_LUMINANCE_INSTABILITY` 时使用，默认 0 关闭） |
| `FarTracingRT` | ❌ 单缓冲 | `PF_FloatRGBA` | 当前帧低分辨率 Trace 结果（rgb=辐射 × α） |
| `FarSecondaryTracingRT` | ❌ 单缓冲 | `PF_FloatRGBA` | MinMax 模式下的 Near 端截止结果（Path B 使用） |
| `FarTracingRTDepth` | ❌ 单缓冲 | `PF_FloatRGBA` | (CloudFront km, Scene km, MinDeviceZ, MaxDeviceZ) |
| `CombinedReconstructRT/Depth` | ❌ | `PF_FloatRGBA` | Bilateral Upscale 之前的多 Level 合并缓冲（仅旧路径使用） |

⚠️ 注意：项目中**没有** Radiance 与 Transmittance 的独立 RT；α 通道既是不透明度又是 1−Transmittance。LightingCache 是 3D RT（不在 ViewState 这条线，由 NubisVolume Component 持有）。

### A.3 双缓冲交换机制

```cpp
// NubisRenderTargetViewStateData.h:246-250 (Per-Level)
CurrentRT = 1 - CurrentRT;
const uint32 PreviousRT = 1 - CurrentRT;
bHistoryValid = !bCameraCut
              && ReconstructRT[PreviousRT].IsValid()
              && ReconstructRTDepth[PreviousRT].IsValid();
```

构造时 `CurrentRT = 1`（NubisVolumes.cpp:562），故第一帧会变 0、第二帧变 1，循环往复。`bHistoryValid` 是 Reconstruct CS Permutation `PERMUTATION_HISTORY_AVAILABLE` 的来源——首帧/相机切/分辨率改 → 走无 history 分支（`PathD`/直接双线性）。

```
帧 N 写 ReconstructRT[0]   ──────┐
                                  ▼
                          ┌──── Swap ────┐
                          ▼              │
帧 N+1 读 ReconstructRT[0] (history)     │
帧 N+1 写 ReconstructRT[1]   ────────────┘
                                  ▼
帧 N+2 读 ReconstructRT[1] (history)
帧 N+2 写 ReconstructRT[0]    ←── 循环 ──┘
```

### A.4 Per-Level 与 Per-View 的层级关系

- **Per-View** 字段（`NubisReconstructRT/Depth`、`NubisFarTracingRT*`、`NubisInstabilityRT`）是历史保留路径，仅一个 Level 时仍生效。
- **Per-Level** 是 Clipmap 多 MipLevel 引入的新机制：每个使用 Far Dither 策略的 Level（典型 2,3,4）通过 `GetOrCreatePerLevelDitherState(MipLevel)` 获取自己的双缓冲，**互不污染 history**。Near 策略（`!Config.bUseDither`）不进入这套，无 history。
- 生效路径在 `NubisVolumesLiveShadingPipeline.cpp:2659-2684`：
  ```cpp
  FNubisPerLevelDitherState& DitherState = NRT.GetOrCreatePerLevelDitherState(Config.MipLevel);
  DitherState.UpdateResolution(FullResolution, ReconstructDownsample, FarTracingDownsample);
  DitherState.Initialise(LevelDitherFrameCount, LevelTracingDownsampleFactor, 0.5f, bCameraCut);
  ```

---

## B. Reconstruct.usf 时序重建（A8）

### B.1 重投影 UV 计算

```hlsl
// NubisVolumesReconstruct.usf:416-428
float TracingVolumetricSampleDepthKm =
    SafeLoadTracingVolumetricDepthTexture(int2(PixelPos) / NubisFarTracingRTDownsampleFactor)
        .CloudFrontDepthFromViewKm;
float DeviceZ = ConvertToDeviceZ(TracingVolumetricSampleDepthKm * KILOMETER_TO_CENTIMETER);

float4 CurrClip   = float4(ScreenPosition, DeviceZ, 1);
float4 PrevClip   = mul(CurrClip, View.ClipToPrevClip);   // 不含 TAA jitter
float2 PrevScreen = PrevClip.xy / PrevClip.w;
float2 PrevUVs    = ScreenPosToViewportUV(ScreenPosition - (ScreenPosition - PrevScreen));
```

```
当前帧 NDC + 云前端 DeviceZ
        │
        │ × View.ClipToPrevClip (无 jitter)
        ▼
  PrevClip → 透视除  →  PrevScreen
        │
        ▼
  Velocity = ScreenPos − PrevScreen
        │
        ▼
  PrevScreenUVs = ScreenPos − Velocity
        │
        ▼
  ┌───  bValidPreviousUVs = all(0 ≤ UVs ≤ 1)
  │
  ├── 有效 → 采样 PreviousFrameVolumetricTexture / Depth / Instability
  └── 无效 → Path D（屏幕外回退，直接双线性当前帧 Tracing）
```

> **关键决定**：用云前端深度（不是场景深度）做 reproject——这样移动云团能跟着自身速度对齐，避免远处云片被场景物体的 motion vector 错误带走。

### B.2 路径选择骨架

| 路径 | 入选条件 | 行为 | Debug 颜色 |
|-----|---------|------|---------|
| A | `bUseNewSample` (子像素 Bayer 命中) | `lerp(NewRGBA, HistoryRGBA, BlenderFactor)` + 8-邻域 AABB clip + DepthBlend smoothstep | 红 |
| B | `!bUseNewSample` & MinMax DepthDiff > Threshold | Primary/Secondary 沿 SceneDepth lerp + GatherRed 边缘检测 | 橙/黄 |
| C1/C2 | DepthDeltaCm 越过 ±0.5×Threshold | smoothstep 渐进丢弃 history | 绿/青 |
| C3 | DepthDelta 在阈值内 | 直接采用 history | 蓝 |
| D | `!bValidPreviousUVs` | 无 history → 当前帧双线性 | 品红 |

`bUseNewSample` 的子像素判定（reconstruct 的核心）：

```hlsl
// NubisVolumesReconstruct.usf:373-383
int2 IntPixelPosDownsample = IntPixelPos / NubisFarTracingRTDownsampleFactor;
const int XSub = IntPixelPos.x - IntPixelPosDownsample.x * NubisFarTracingRTDownsampleFactor;
const int YSub = IntPixelPos.y - IntPixelPosDownsample.y * NubisFarTracingRTDownsampleFactor;
const bool bUseNewSample = (XSub == CurrentTracingPixelOffset.x)
                        && (YSub == CurrentTracingPixelOffset.y);
```

→ 同 Bayer 周期内每个 Reconstruct 像素恰好一帧被命中、其他帧走 Path B/C 上采样。

---

## C. Bilateral Upscale 数学 (A8)

### C.1 模式概览

`UPSCALE_MODE` 是 .usf 内 `#define`（不再走 permutation，2026 年改造），4 个模式：

| 模式 | 名称 | 核 | 备注 |
|-----|------|---|------|
| 0 | 原始 2×2 | 2×2 邻域 | 阈值 `MaxDepth4Diff > 0.1×PixelDist` 切换 bilinear / depth-weighted |
| 1 | 4×4 高斯 + 双边 | 4×4 | σ_spatial=1.0 tex, σ_depth=2% dist |
| 2 | 直接 1/2 → 全分辨率 bilateral | 4×4 | 固定 2× downsample, σ_spatial=0.7, σ_depth=1.5% |
| 3 | 深度二值测试 + 多采样平均 | 2×2 | 累加通过 `PixelDist > CloudFront−bias` 的 4 个样本 |

### C.2 关键深度夹紧（防天空 NaN）

```hlsl
// NubisVolumesBilateralUpscale.usf:499-501（Mode 0），亦见 4x4 :610-612
float MaxNeighborDepthKm = max(max(SceneDepth4Km.x, .y), max(.z, .w));
PixelDistanceFromViewKm = min(PixelDistanceFromViewKm, MaxNeighborDepthKm);
```

天空像素 DeviceZ 极小，`ConvertFromDeviceZ` 返回极大值，会把所有 `Depth4Diff` 推爆 → 全权重 0 → NaN/黑斑。**夹到 4 邻域最大深度后，至少 1 个邻域权重正常。**

### C.3 反距离权重融合（Mode 0）

```hlsl
// NubisVolumesBilateralUpscale.usf:511-529
const float WeightMultiplier   = 1000.0f;
const float ThresholdToBilinear = PixelDistanceFromViewKm * 0.1;  // 10% 距离

if (MaxDepth4Diff > ThresholdToBilinear) {
    float4 Depth4Diff = abs(PixelDistanceFromViewKm - SceneDepth4Km);
    float4 weights = 1.0f / (Depth4Diff * WeightMultiplier + 1.0f);
    weights /= dot(weights, 1.0.xxxx);     // 归一化
    DataAcc = w.x*RGBT0 + w.y*RGBT1 + w.z*RGBT2 + w.w*RGBT3;
} else {
    DataAcc = bilinear sample at offset center;
}
```

### C.4 "Near Dominance Blend" 公式（核心）

⚠️ **重要发现**：Near Dominance 的 cbuffer 已声明（`FNubisBilateralUpscaleCS::FParameters` 含 `NearDominanceEnabled/StartAlpha/EndAlpha`），CPU 端两处 Pass 也都赋值了（`NubisVolumesLiveShadingPipeline.cpp:2161-2163, 2376-2378`），**但 .usf 文件中没有任何引用这些参数的 HLSL 代码**。Shader 当前的 `Far under Near` 公式只用基本 transmittance：

```hlsl
// NubisVolumesBilateralUpscale.usf:842-851 (CompositeBlendMode > 0)
float4 ExistingNear = RWCombinedVolumetricTexture[PixelCoord];
float ExistingNearTransmittance = saturate(1.0 - ExistingNear.a);
BlendedResult.rgb = ExistingNear.rgb + FarReconstructColor.rgb * ExistingNearTransmittance;
BlendedResult.a   = ExistingNear.a   + FarReconstructColor.a   * ExistingNearTransmittance;
```

CVar 注释（`NubisVolumes.cpp:177-200`）描述了**期望**的语义：

```
开启时：当 Near.a ∈ [Start, End] 应有进一步 smoothstep 压制 Far：
  α  = saturate( (Near.a - Start) / (End - Start) )       // 0..1
  k  = 1 − α                                              // Far 衰减系数
  Far'.rgb = Far.rgb * k * NearTransmittance              // 在原 Far × T 上再乘 k
  Result.rgb = Near.rgb + Far'.rgb
默认 Start=0.3, End=0.6 → Near.a≥0.6 时 Far 完全被压制
```

**结论 [推测+证据]**：参数已在 cbuffer/CPU 端就位，但 `BilateralUpscale.usf` 似乎漏写或被回滚——这是一个 **未完成的功能开关**。Grep `Engine/Shaders` 确认没有任何 `Dominance|StartAlpha|EndAlpha` 出现在 shader 端（输出为空）。

### C.5 全分辨率 Depth Occlusion（边缘安全网）

```hlsl
// NubisVolumesBilateralUpscale.usf:758-779 (UpscaleFarCloud)
float TransitionPercent = lerp(0.002, 0.008, saturate((DownsampleScale - 2.0) / 2.0));
float TransitionWidthKm = max(0.001, CloudFrontDepthKm * TransitionPercent);

float OcclusionFactor = smoothstep(
    CloudFrontDepthKm - TransitionWidthKm,
    CloudFrontDepthKm,
    PixelDistanceFromViewKm);  // PixelDist >= CloudFront → 1（可见）
                              // PixelDist <  CloudFront → 0（被前面的不透明物体遮挡）
NubisRt0 *= OcclusionFactor;   // 衰减 RGB+α
```

下采样系数越大，过渡带越宽（1/2→0.2%, 1/4→0.8%）以补偿低分辨率纹素覆盖更多全分辨率像素时的边缘锯齿。

---

## D. Jitter 与 Far Dither 管线 (A8)

### D.1 主开关

```cpp
// NubisVolumes.cpp:48-53, 507-510
static TAutoConsoleVariable<int32> CVarNubisVolumesJitter(
    TEXT("r.NubisVolumes.Jitter"), 1,
    TEXT("Enables jitter when ray marching (Default = 1)"),
    ECVF_RenderThreadSafe);

bool ShouldJitter() { return CVarNubisVolumesJitter.GetValueOnRenderThread() != 0; }
```

> 默认开启。`ShouldJitter()` 主要用于光线步进起点的随机化（向 ray-marching CS 注入 sub-step jitter）。

### D.2 Bayer Pattern（空间抖动）

`FNubisPerLevelDitherState::Initialise()` 内的查表（`NubisRenderTargetViewStateData.h:225-289`）：

| DownsampleFactor | Pattern 大小 | 周期 | 来源 |
|---|---|---|---|
| 2 | 2×2 = 4 帧 | `OrderDithering2x2[4] = {0,2,3,1}` | Bayer-2 |
| 4 | 4×4 = 16 帧 | `OrderDithering4x4[16] = {0,8,2,10, 12,4,14,6, 3,11,1,9, 15,7,13,5}` | Bayer-4 |
| 8 | 8×8 = 64 帧 | 完整 8×8 Bayer 矩阵 | Bayer-8 |
| 其他 | 线性扫描 | `(idx % N, idx / N)` | fallback |

每帧把 `FrameIndex++`、查表得到 `LocalFrameId`、还原成 `(x, y) ∈ [0,N)`，写入 `CurrentPixelOffset`。

```
DownsampleFactor=2:
  帧 N   → FrameId 0 → Order[0]=0 → offset (0,0)
  帧 N+1 → FrameId 1 → Order[1]=2 → offset (0,1)
  帧 N+2 → FrameId 2 → Order[2]=3 → offset (1,1)
  帧 N+3 → FrameId 3 → Order[3]=1 → offset (1,0)
  帧 N+4 → 回到 (0,0) ……
```

### D.3 Jitter ↔ Reproject 耦合

注入路径：`CurrentPixelOffset → CB(CurrentTracingPixelOffset) → Reconstruct.usf` 用于判定 `bUseNewSample`。**Reproject 不需要补偿 Bayer 偏移**——因为：

1. Reconstruct 用云前端深度做重投影（与 Bayer 偏移无关）；
2. AABB Color Clip 通过邻域包围盒夹紧，自然吸收 jitter 引入的小偏差；
3. `View.ClipToPrevClip` 不含 jitter，保证当前/历史在同一非抖动空间对齐。

### D.4 Per-Level 帧计数初始化时机

```cpp
// FNubisPerLevelDitherState::Initialise (.h:231-289)
if (bCameraCut) { FrameIndex=0; CurrentPixelOffset=0; CurrentRT=0; bHistoryValid=false; }
CurrentRT = 1 - CurrentRT;                              // 双缓冲交换
bHistoryValid = !bCameraCut && Reconstruct[Prev].IsValid() && ReconstructDepth[Prev].IsValid();
FrameIndex++;
uint32 FrameId = FrameIndex % (DownsampleFactor * DownsampleFactor);
// ... 查 Bayer 表得 CurrentPixelOffset
```

每个 Level 维护**自己的 FrameIndex**，因此不同 Level 即使同时启用 Far Dither，Bayer 命中点也彼此独立——避免多 Level 帧抖动同步导致整屏闪烁。

---

## E. Visualize 模式 (E1)

### E.1 完整模式枚举

```hlsl
// NubisVolumesVisualize.usf:9-14
#define NUBIS_VIS_MODE_RADIANCE         1   // 显示云预乘 radiance（HDR Reinhard tonemap）
#define NUBIS_VIS_MODE_DEPTH            2   // 用 Turbo-like LUT 着色 CloudFront 深度
#define NUBIS_VIS_MODE_LIGHTING_CACHE   3   // 3D LightingCache 切片或 tile 网格 (Heat LUT)
#define NUBIS_VIS_MODE_FAR_TRACING      4   // 低分辨率 dither tracing 输出
#define NUBIS_VIS_MODE_RECONSTRUCT      5   // Reconstruct 后结果
```

| ID | 模式 | 输入 RT | LUT |
|----|------|---------|-----|
| 1 | Radiance | `View.NubisVolumeRadiance` | `rgb / (1+rgb)` Reinhard |
| 2 | Depth | `View.NubisVolumeDepth` | TurboColormap(t = `Depth.r * DepthScale`) |
| 3 | LightingCache | 3D RT (Per-Level via `r.Nubis.Visualize.ClipmapLevel`) | HeatColormap + 可选 tile 网格 |
| 4 | Far Tracing | ViewState 的 `FarTracingRT`（或 Per-Level 的） | `rgb / (1+rgb)` |
| 5 | Reconstruct | ViewState 的 `Reconstruct[Cur]` | `rgb / (1+rgb)` |
| default | unknown | — | 品红警示 |

⚠️ 任务清单中提到的 SparseVoxel / RayMarchSteps / DitherPattern / FrameJitter / Transmittance 等模式**当前 .usf 没有实现**——只有 5 个基础模式 + 默认品红。Visualize 系统的更多功能依赖于已注释的 `NubisVisualizationData.h`（CL611225 调试 RT 改动），目前在仓库中已不存在。

### E.2 触发开关

无单一 "r.Nubis.Visualize" CVar；通过两条路径进入：

```cpp
// NubisVolumes.cpp:1645-1660
bool ShouldVisualizeNubis(const FViewInfo& View) {
#if WITH_DEBUG_VIEW_MODES
    if (View.Family && View.Family->EngineShowFlags.VisualizeNubis) return true;
#endif
    FNubisVisualizationData& NubisVis = GetNubisVisualizationData();
    NubisVis.Update(NAME_None);            // 让 NubisVisualizationData 自己读 CVar
    return NubisVis.IsActive();
}
```

→ 一是 Editor 的 `EngineShowFlags.VisualizeNubis`（视图模式菜单），二是 `FNubisVisualizationData` 内部维护的 ViewMode 名 +  CVar（具体 CVar 名要看 `NubisVisualizationData.h`，但该文件已注释/不存在）。

辅助 CVar（`NubisVolumes.cpp:1566-1592`）：
- `r.Nubis.Visualize.ClipmapLevel`（int，默认 0）— 选哪一级 LightingCache
- `r.Nubis.Visualize.LightingCacheSlice`（int，默认 0）— 单切片视图的 Z slice
- `r.Nubis.Visualize.LightingCacheTiled`（int，默认 0）— 1=网格平铺所有切片
- `r.Nubis.Visualize.DepthScale`（float，默认 1e-5）— 深度归一化系数

### E.3 Reconstruct.usf 内置的 DebugResolveMode

独立于 Visualize Pass，Reconstruct CS 自己有一组路径染色（仅供 RDG 替换调试）：

| Mode | 含义 |
|------|------|
| 0 | 正常输出 |
| 1 | 路径染色：A=红 / B=黄 / C1=绿 / C2=青 / C3=蓝 / D=品红 |
| 2 | Path B 深度三通道 (Near, Far, SceneHalf) |
| 3 | edgeFactor / LerpFactor |
| 4 | Secondary RT |
| 5 | Primary RT |
| 6 | edgeFactor 灰度 + 边缘红色叠加 |
| 7 | Luminance Instability（仅 NUBIS_LUMINANCE_INSTABILITY=1） |

注：`DebugResolveMode` 由 C++ 端始终设 0（CL611225 注释），实际运行不进入这些分支，这些代码相当于 dead-but-kept 调试基础设施。

---

## F. 关键参数与 CVar 速查（A8 + E1）

| CVar | 默认 | 作用 | 来源 |
|------|------|------|------|
| `r.NubisVolumes.Jitter` | 1 | Ray-marching 起点 jitter 总开关 | NubisVolumes.cpp:48 |
| `r.NubisVolumes.NearDominanceBlend` | 1 | 近景 α 压制 Far 贡献开关 | NubisVolumes.cpp:177 |
| `r.NubisVolumes.NearDominanceStartAlpha` | 0.3 | 开始压制阈值 | NubisVolumes.cpp:186 |
| `r.NubisVolumes.NearDominanceEndAlpha` | 0.6 | 完全压制阈值 | NubisVolumes.cpp:194 |
| `r.Nubis.Visualize.ClipmapLevel` | 0 | LightingCache 可视化的 Level | NubisVolumes.cpp:1566 |
| `r.Nubis.Visualize.LightingCacheSlice` | 0 | LightingCache Z 切片 | NubisVolumes.cpp:1573 |
| `r.Nubis.Visualize.LightingCacheTiled` | 0 | 1=切片网格平铺 | NubisVolumes.cpp:1580 |
| `r.Nubis.Visualize.DepthScale` | 1e-5 | 深度归一化 | NubisVolumes.cpp:1587 |

| 着色器宏 | 默认 | 作用 |
|---------|------|------|
| `NUBIS_LUMINANCE_INSTABILITY` | **0** | Reconstruct.usf:108，开启 EMA 软过渡防闪 |
| `UPSCALE_MODE` | **0** | BilateralUpscale.usf:80，0=2×2 / 1=4×4 高斯 / 2=直接 1/2→全 / 3=深度二值 |
| `USE_YCOCG` | 1 | Reconstruct AABB 在 YCoCg 空间裁剪 |
| `PERMUTATION_HISTORY_AVAILABLE` | 由 C++ 注入 | 首帧/相机切走 1 (无 history) 分支 |

---

## G. 开放问题

1. **Near Dominance Blend 实际是否生效？** cbuffer/CPU 已就位，但 `BilateralUpscale.usf` 没有任何引用 `NearDominanceEnabled/StartAlpha/EndAlpha` 的代码。需要：
   - (a) 找历史 Perforce CL 看 shader 端是否曾有实现并被回滚；
   - (b) 在已运行场景中测 Near.a ≥ 0.6 时 Far 是否真的被进一步压制；
   - (c) 评估是否要把 CVar 注释中描述的公式补回到 :843-851 处。
2. **Visualize 模式被 CL611225 砍剩 5 个**，原版 `NubisVisualizationData.h` 还存在于历史 CL 中吗？里面是否定义了 SparseVoxel / RayMarchSteps / DitherPattern / FrameJitter 等模式？
3. **Per-Level Dither 与 Per-View 状态共存**：当前 cpp 同时维护 `NubisVolumeRT.Initialise()`（Per-View）与 `PerLevelDitherStates`（Per-Level）。两套是同时调用还是互斥？哪个 Level 走 Per-View 路径？
4. **DownsampleFactor=8 的 Bayer-8 表**已经写好（64 帧周期），但实际项目里有没有任何 Config 把 Tracing 降采样设到 8？过长周期会让相机微动产生明显残影。
5. **`Initialise` 的 `bHistoryValid` 与 RT Resolution-Reset 顺序**：cpp:2674-2684 注释说"先 UpdateResolution 再 Initialise"避免不一致，但 `Initialise` 内部 :622-628 还会再次释放 `NubisReconstructRT[CurrentRT]`。这个二次释放的边界条件是否覆盖动态分辨率缩放？
6. **Instability EMA 默认关闭**（`NUBIS_LUMINANCE_INSTABILITY=0`），意味着当前 ship 配置走的是硬切换。开启的性能开销注释说 `~5~10 ALU + 0.5 MB`，但是否在中端 GPU 可承担？
7. **Per-Level RT 占用统计**：Clipmap 4 级时 ReconstructRT[2]+InstabilityRT[2]+FarTracingRT 等加起来显存量级？`GetGPUSizeBytes` 只统计 Per-View，不含 PerLevelDitherStates → 显存监控有盲区。
