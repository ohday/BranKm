---
title: "NubisCloud Clipmap 调度 — Mip Ring / Sector 滚动 / Two-Pass"
type: Local code archaeology
fetched: 2026-05-11
project: higame-nubis-cloud
relevant_to: A3, B3
sources_drawn_from:
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisVolumes.cpp:152-172,316-330,887-1043,1049-1367
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisVolumes.h:51-100,104-113
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisVolumesLiveShadingPipeline.cpp:1438-1462,1602-1625,1810-1825,2540-2727
  - Engine/Source/Runtime/Engine/Public/NubisVolumeInterface.h:20-302,406-436
  - Engine/Source/Runtime/Engine/Private/Components/NubisVolumeComponent.cpp:75-100,665-720
  - Engine/Shaders/Private/NubisVolumes/NubisVolumesLightingCacheReseed.usf:25-65
  - Engine/Shaders/Private/NubisVolumes/NubisVolumesLiveShadingPipeline.usf:74-90,140-150,380-470
  - Engine/Shaders/Private/NubisVolumes/NubisVolumesTransmittanceVolumeUtils.ush:38-90
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Public/NubisClipmap.h:13-398
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Public/NubisClipmapSubsystem.h:1-99
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Private/NubisClipmapSubsystem.cpp:32-110
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Private/NubisClipmap.cpp:130-145,220-236,388-447,617-722,2434-2502,2560-2750
---

# NubisCloud Clipmap 调度 — Mip Ring / Sector 滚动 / Two-Pass

> 复用 raw#1 / raw#8 已验证事实(`HIGAME_ENABLE_NUBIS=1`、Mip 0-5 共 6 级、
> `SectorSize=256×256×64`、Sector 烘焙仅 Mip 3+),本笔记聚焦 **运行时调度**。

---

## A3-1 6 级 Mip 是否串行 — Two-Pass 调度的真实形态

`RenderNubisVolumes()` 在每 View / 每 VolumeMesh 内部串行展开
(`NubisVolumes.cpp:1077-1366`):

```
for ViewIndex                                      // 视图
  for VolumeIndex                                  // VolumeMesh (按相机距排序)
    ┌── Pass 1: LightingCache 远→近 ─────────────────────────────┐
    │   for (Level = LightingCacheLevelCount-1; Level >= 0; --L)  │
    │     1. AddReseedLightingCachePasses(Cfg, ParentCfg)         │
    │     2. RenderNubisClipmapLevel(..., bLightingCacheOnly=true)│
    │ ※ MaxLightingCacheLevels=5 → 实际只跑 L4..L0 (L5 不烘)      │
    │ ※ 忽略 r.NubisVolumes.DebugMipMask (cpp:1212-1215)          │
    └─────────────────────────────────────────────────────────────┘
    ┌── Pass 2: Scattering 近→远 ───────────────────────────────┐
    │   for (Level = 0; Level < ClipmapLevelCount; ++L)           │
    │     if (!(ScatterDebugMipMask & (1<<L))) continue;          │
    │     RenderNubisClipmapLevel(..., bLightingCacheOnly=false)  │
    │ ※ L0=Near (BlendMode=0), L1~L4=FarDither, L5=Octa(TODO)     │
    │ ※ Near→Near 通过 PrevNearLowResRadiance 链接 EarlyOut       │
    └─────────────────────────────────────────────────────────────┘
```

**结论:** 6 级 **完全串行**(单线程渲染线程,顺序 enqueue 到 RDG);
两趟 for 循环互不重叠,Pass1 全部完成才进入 Pass2。

### 为什么 LightingCache 远→近、Scattering 近→远?

| 趟 | 方向 | 原因 |
|---|---|---|
| Pass1 LCache | L4→L0 | **级联依赖**:Mip N 的 LCache CS 在 Local-Space ray-march 后,对超出本级 AABB 的远场段要采父级 Mip N+1 的 LCache 三线性查询光学深度 → 父级先生成。`ParentLightingCacheTexture` 来自 `PerLevelLightingCacheTextures[Level+1]`(`NubisVolumes.cpp:1232-1260`)。 |
| Pass2 Scatter | L0→L5 | **空间 EarlyOut + Composite blend-under**:Near L0 写 BlendMode=0(覆盖),L1+ 走 BlendMode=1(blend under),BilateralUpscale 把远 Mip 叠在近 Mip alpha 下方(`LiveShadingPipeline.cpp:2705`)。Near→Far 顺序保证近景先入 RT,远景被 NearDominanceBlend 压制(`NubisVolumes.cpp:174-200`)。 |

---

## A3-2 Mip Ring Crossover & EarlyExit — Near/Far 分工

| Cvar | 默认 | 作用 | 入口 |
|---|---|---|---|
| `r.NubisVolumes.MipRingEarlyExit` | **0**(关) | =1 时 Mip N 的 ray 进入 Mip(N-1) AABB 即截断;烘焙路由可能仅烘到粗 Mip → 默认关防漏厚云 | `NubisVolumes.cpp:155-163, :414-417` |
| `r.NubisVolumes.MipRingCrossoverCm` | **500.0 cm** | 环带交叠带宽,Mip N 截断内边界 = Mip(N-1) half-extent − CrossoverCm,做 lerp 过渡避免硬接缝 | `NubisVolumes.cpp:165-172, :419-422` |

**消费位置(只在 Pass 2 散射趟,模板相同):**
- `LiveShadingPipeline.cpp:1438-1445` Near 主 CS / `:1602-1620` Near EarlyOut CS(透传)/ `:1810-1818` FarDither 主 CS

```cpp
// LiveShadingPipeline.cpp:1441 / 1813 模板
const bool bHasPrevLevel = (PassParameters->CurrentMipLevel >= 1);
PassParameters->MipRingEarlyExitEnabled =
    (NubisVolumes::GetMipRingEarlyExit() && bHasPrevLevel) ? 1u : 0u;
PassParameters->MipRingCrossoverCm = NubisVolumes::GetMipRingCrossoverCm();
```

**Pass 1 LightingCache CS 不接收这两个参数** — 环带分工只对 Scattering 生效;
LCache 是被多级共享/级联采样的低频量,残缺会污染所有上层。

---

## A3-3 Sector 滚动 & LightingCache Reseed

### 核心式(`NubisClipmap.cpp:388-400, 655-670`)

```
ScrollOffset         = (ScrollOffset + SectorDelta) mod SectorWidth
ClipmapScrollUVOffset = ScrollOffset / SectorWidth   ∈ [0,1)³
ScrollOffsetSectors   = ScrollOffset                 // 整数副本,MipSelector 用
```

`Levels[i].VolumeTexture` 是 **Wrap 寻址 + 物理环形缓冲**;`MergeLoadedTexture`
通过 `RHICmdList.CopyTexture` 把新 sector 写入 ScrollOffset 决定的物理槽位
(`NubisClipmap.cpp:1647-1649`)。**Modeling/SDF Texture 不重建,只增量 Copy**。

### Logic UV ↔ Physical UV 换算(`NubisVolumes.h:107-112` + USF 验证)

```
读 LightingCache:    PhysicalUVW = frac(LogicUVW + LightingCacheScrollUVOffset + 1.0)
写 LCache (Reseed):  ChildLogicalUV = frac(ChildPhysUV - ChildClipmapScrollUVOffset + 1.0) USF:57
                     ParentPhysUV   = frac(ParentLogicUV + ParentClipmapScrollUVOffset + 1.0) USF:63
读 Modeling:         PhysicalUVW = frac(LogicUVW + ClipmapScrollUVOffset + 1.0) LiveShading.usf:388
```
+1.0 避免负数 frac;Wrap 硬件自动 frac,显式 frac 防止边界精度抖动。

### 为何需要 Reseed(LCache 不能直接 Copy)

LightingCache 是 CS 实时 trace 出来的光学深度场,分帧分摊(`AmortizeDivisor=2~6`,
8/27/64/216 帧填一遍)+ EMA(β=0.9~0.97)。Sector 滚动后旧物理槽位仍是 **旧 logic
含义** 下的光学深度;仅靠 Amortize+EMA 追赶 → 多帧色差/残影(`NubisVolumes.cpp:1241-1244`)。

解决:`AddReseedLightingCachePasses`(`NubisVolumes.cpp:947-1043`)+
`NubisVolumesLightingCacheReseed.usf`,在 LCache CS **之前** 对滚动换入的 voxel 区域
(`LightingCacheDirtyRegions`)从父级 Mip 三线性回填,无父级写 0。单帧最多 8 段
(三轴 ring-buffer wrap),常见 1~3 段。`r.Nubis.LightingCache.ReseedFromParent=1` 默认开。

### DirtyRegions 填充(`NubisClipmap.cpp:407-447`)

```
if (TypeIdx == 0 /* Modeling 趟 */) {
    LevelConfig.LightingCacheDirtyRegions.Reset();
    bFullReseed = (|DeltaInt| >= SectorWidth) || bFirstUpdate;
    if (bFullReseed)  AddRegion(0, CacheRes);                 // 整级
    else for axis ∈ {X,Y,Z}: EmitSlabOnAxis(axis, ...)        // 三轴 slab,环形拆段
}
```
每帧 `Update()` 开头先 `LightingCacheDirtyRegions.Reset()`(`:233-236`);
`bHadLightingCacheDirtyLastFrame` 标志位强制再同步空 regions 一次,
避免渲染端残留 dirty 每帧覆盖 trace 结果(`:344-347, 725-731`)。

### Sector 滚动 → Reseed 流程图

```
摄像机 SectorDelta = (1,0,0)  (向 +X 走过 1 个 sector,SectorWidth=2×2×2,Mip3 视角)

旧 ScrollOffset=(0,0,0)              新 ScrollOffset=(1,0,0)
┌──────┬──────┐                       ┌──────┬──────┐
│S(0,0)│S(1,0)│  CopyTexture →        │S(2,0)│S(1,0)│  ← S(2,0) 覆盖 phys00
│phys00│phys10│  把 S(2,0) 写入       │phys00│phys10│
├──────┼──────┤  旧 phys00 槽位       ├──────┼──────┤
│S(0,1)│S(1,1)│                       │S(0,1)│S(1,1)│
└──────┴──────┘                       └──────┴──────┘

LightingCache(同 phys 坐标,但内容是上帧 trace 的旧 S(0,0) 光学深度):
phys00 用新 ScrollUVOffset 解读 → 映射到 logic S(2,0) → 视觉错位
→ DirtyRegions 标 phys00 列对应 voxel
→ AddReseedLightingCachePasses 从 Mip4(父级)三线性回填 → Amortize+EMA 收敛
```

---

## B3-1 NubisClipmapSubsystem 生命周期 + Tick

`NubisClipmapSubsystem.h:30` 继承 **`UTickableWorldSubsystem`**(World 级,
非 GameInstance 级)→ 跨 World 不共享 Manager。`IsTickableInEditor()=true`。

```
Initialize(...)         (cpp:32) 仅打 Log,不创建 Manager
Deinitialize()          (cpp:38) Manager->Shutdown(),Empty
Tick(DeltaTime)         (cpp:59) for (Zone, Manager):
                                   CameraPos = GetCameraWorldPosition()
                                     // Game: PlayerCamera, Editor: ViewportClient
                                   if Zone->bFreezeClipmapOrigin:
                                     Manager->Update(FrozenCameraPositions[Zone])
                                   else
                                     Manager->Update(CameraPos)

注册路径:
  ANubisZone2Actor::BeginPlay → RegisterZone   → new Manager → Initialize(DataAsset)
  ANubisZone2Actor::EndPlay   → UnregisterZone → Manager->Shutdown → TUniquePtr 析构
```

`Manager::Update(CameraPos)` 每渲染帧一次:
1. `LightingCacheDirtyRegions.Reset()` 全清
2. 对每 (TextureType, Level) 计算 `OldSectorPos / NewSectorPos`
3. 滚动则更新 `ScrollOffset` / 填 `DirtyRegions` / 入队增量加载
4. `ProcessPendingLoads(bIsFirstFrame)` — `MaxLoadsPerFrame=8` 限流
5. 任意级变化 OR 上帧带 dirty:重算 `ScrollUVOffset / ScrollOffsetSectors / Parent*` →
   `SetClipmapData` + 当帧 `SyncClipmapScrollToProxy_RenderThread()`
6. 维护 `bHadLightingCacheDirtyLastFrame`

---

## B3-2 FNubisClipmapLevelRenderConfig 完整字段表

来源:`NubisVolumeInterface.h:82-242`

| 字段 | 类型 | 默认 | 含义 / 填充端 |
|---|---|---|---|
| `MipLevel` | int32 | 0 | Level 0~5 / Mgr:2441 |
| `bUseDither` | bool | false | Far Dither 管线开关 / Mgr:2459-2497 |
| `bUseOctahedral` | bool | false | Mip5 Octahedral 管线 / Mgr:2494 |
| `WorldBoundsOrigin` | FVector3f | 0 | ClipmapCenter(Sector 对齐世界中心)/ Mgr:650 |
| `LocalBoundsExtent` | FVector3f | `NubisDefaults::GetLocalBoundsExtent()` | 所有 Level 共享 / Mgr:2447 |
| `ClipmapScrollUVOffset` | FVector3f | 0 | = ScrollOffset / SectorWidth / Mgr:655 |
| `ClipmapOriginSectorIdx` | FIntVector | 0 | 窗口左下角 Global Sector Idx / Mgr:666 |
| `ScrollOffsetSectors` | FIntVector | 0 | 整数副本(MipSelector)/ Mgr:670 |
| `TransmittanceVoxelResolution` | FIntVector | (512,512,128) | Modeling 体素 / Mgr:2450 |
| `ScatteringCacheVoxelResolution` | FIntVector | (256,256,64) | LCache=SectorSize / Mgr:2453 |
| `DitherFrameCount` | int32 | 1 | 1=Near/2/3/4/0=Octa / Mgr:2461-2497 |
| `DownsampleFactor` | int32 | 2 | Reconstruct RT 屏幕降采 / Mgr:2462-2496 |
| `TracingDownsampleFactor` | int32 | 2 | Tracing/Reconstruct 比(Bayer 大小)/ Mgr:2463-2497 |
| `MaxTraceDistance` | float (cm) | 0 | 本级 AABB 对角线长度上界 / Mgr:2503+ |
| `PerLevelMinStepSize / StepCoefficient / MaxStepCount` | float / float / int32 | -1 | 步长参数,-1 回落全局 cvar |
| `LightingCacheAmortizeDivisor` | int32 | 2 | N → 8/27/64/216 帧填一遍 |
| `LightingCacheEmaHistory` | float | 0.9 | EMA β,L0=0.9 → L4=0.97 |
| `bHasParentLightingCache` | bool | false | 是否有 Mip+1 父级 / Mgr:692,698 |
| `ParentWorldBoundsOrigin / ParentClipmapScrollUVOffset` | FVector3f / FVector3f | 0 / 0 | 父级 Center / ScrollUV / Mgr:693-694 |
| `LightingCacheDirtyRegions` | TArray<FDirtyRegion, TInlineAlloc<8>> | empty | 滚动换入 voxel AABB / Mgr 每帧:407-447 |

**已废弃(注释保留):** `bHasPreviousLevel / Prev*`(Previous-Level Bounds Cull,
`:218-222`);`MinTraceDistance`(被 RayTraceBoundsExtent 替代,`:161-163`)。

---

## B3-3 数据流时序图(GT ↔ RT)

```
GT (Tick)                                  RT (RDG)
─────────────────────────────              ───────────────
ANubisZone2Actor::BeginPlay
  └─ Subsystem::RegisterZone → new Manager → Manager::Initialize(DataAsset)
       └─ CreateSingleVolumeTexture × N / CreateMipSelectorVolume

Manager::InitClipmapLevels(Zone)
  └─ Build ClipmapConfigs[6] / MIDs[6] / PerLevelLightingCacheRTs[6]
  └─ VolComp->SetPerLevelLightingCacheRTs   →→ MarkRenderStateDirty
  └─ VolComp->SetMipSelectorVolume          →→ ENQUEUE_RENDER_COMMAND
  └─ VolComp->SetClipmapData(...)           →→ MarkRenderStateDirty
                                                ↓ (次帧) SceneProxy 重建 → FNubisVolumeData 拷贝

【每帧】 Subsystem::Tick → Manager::Update(CameraPos)
  ├─ for(L,Type): 检测 SectorDelta
  ├─ if 滚动: 更新 ScrollOffset / 入队 PendingLoad / 填 DirtyRegions
  ├─ ProcessPendingLoads → StreamableManager.RequestAsyncLoad
  ├─ (异步回调) HandleTextureLoadComplete → MergeLoadedTexture
  │    └─ ENQUEUE_RENDER_COMMAND(CopyTexture) →→ RHICmdList.CopyTexture
  ├─ if 任意级变化 OR 上帧 dirty:
  │    ├─ 重算 ScrollUVOffset / ScrollOffsetSectors / Parent*
  │    ├─ MID->SetVectorParameterValue(...) (材质参数)
  │    ├─ VolComp->SetClipmapData(...)               →→ MarkRenderStateDirty (次帧重建)
  │    └─ VolComp->SyncClipmapScrollToProxy_RenderThread()
  │         └─ ENQUEUE_RENDER_COMMAND →→ Proxy::UpdateClipmapScrollUVOffsets_RenderThread
  │             当帧覆写 WorldBoundsOrigin / ScrollUVOffset / OriginSectorIdx /
  │             ScrollOffsetSectors / Parent* / DirtyRegions / bHasParent
  │             (NubisVolumeComponent.cpp:82-96)
  └─ bHadLightingCacheDirtyLastFrame = (any DirtyRegions)
                                                ↓
                               【RenderNubisVolumes】Pass1 远→近 reseed + LCache CS
                                                ↓
                               【RenderNubisVolumes】Pass2 近→远 Scattering
```

### MarkRenderStateDirty vs 增量 ENQUEUE_RENDER_COMMAND 分工

| 路径 | 触发字段 | 时延 |
|---|---|---|
| `MarkRenderStateDirty()` (Component re-create) | `ClipmapConfigs` 数组结构、`PerLevelLightingCacheRenderTargets`、首次 `SetClipmapData`、`MaterialProxies` | **次帧**(SceneProxy 重建) |
| `ENQUEUE_RENDER_COMMAND` (`SyncClipmapScrollToProxy_RenderThread`) | `WorldBoundsOrigin / ClipmapScrollUVOffset / ScrollOffsetSectors / ClipmapOriginSectorIdx / Parent* / LightingCacheDirtyRegions / bHasParentLightingCache` | **当帧**(与 CopyTexture 同 FIFO) |
| `ENQUEUE_RENDER_COMMAND` (`SetMipSelectorVolume`) | `MipSelectorTextureRHI` | **当帧** |

设计意图(`NubisVolumeComponent.cpp:75-81` 注释):ScrollOffset 当帧 `MergeLoadedTexture`
已经把新 sector 写入新物理槽位;若仅靠 `MarkRenderStateDirty` 次帧重建,中间一帧
SceneProxy 用旧 ScrollUVOffset → UV 错一格 sector → 系统性采样错位。所以 enqueue 渲染
指令与 `CopyTexture` 共享同一 FIFO 队列保证时序。

---

## 开放问题

1. `bUseOctahedral` 路径在 `LiveShadingPipeline.cpp:2649-2653` 是 TODO。Mip5 实际是 **不渲染** 还是回退?需要 PIE 截帧确认。
2. Pass2 `PrevNearLowResRadiance` EarlyOut 链:[推测] `OutNearLowResRadiance` 仅 Near 分支赋值,FarDither 不写出 → 链路天然在 Near→Far 转换处断。需在 `RenderNearCloudWithLiveShading` 内部确认。
3. `LightingCacheReseedFromParent` 在最高 Level (L4) 无父级时写 0,但 EMA β=0.97 → ~33 帧才衰半。是否会导致 L4 Sector 滚动后长时间残留?是否需要 force-clear 而非 force-zero?
4. `MipRingEarlyExit` 默认 0,但 `MipRingCrossoverCm=500` 始终生效。[推测] CrossoverCm 单独存在意味着 shader 内即使 EarlyExit 关闭也用 Crossover 做 lerp 过渡,需 USF 端确认。
5. `bFreezeClipmapOrigin` 是 Editor-only,但 `IsTickableInEditor()=true` 会在无 PIE 时也 Tick。无相机 Editor Viewport 下 `GetCameraWorldPosition()` 行为?
6. raw#8 提到 Sector 烘焙仅 Mip 3+。但 `LightingCacheDirtyRegions` 填充对所有 Level 都跑(仅 `TypeIdx==0` 去重)。Mip 0-2 的 DirtyRegions 是否始终为空?需交叉验证 `Levels[0~2].SectorStates` 是否始终空。
