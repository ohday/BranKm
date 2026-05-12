---
title: "NubisCloud 新路径插件 NubisCustom2 — 类层次与 GPU 资源"
type: Local code archaeology
fetched: 2026-05-11
project: higame-nubis-cloud
relevant_to: B3, B4
sources_drawn_from:
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/NubisCustom2.Build.cs:1-71
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Public/NubisCustom2.h:1-19
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Private/NubisCustom2.cpp:1-26
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Public/NubisClipmap.h:1-398
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Private/NubisClipmap.cpp:1-3122
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Public/NubisClipmapSubsystem.h:1-99
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Private/NubisClipmapSubsystem.cpp:1-358
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Public/NubisClipmapDataAsset.h:1-82
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Private/NubisClipmapDataAsset.cpp:1
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Public/NubisHiCloud2Actor.h:1-127
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Private/NubisHiCloud2Actor.cpp:1-181
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Public/NubisZone2Actor.h:1-217
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Private/NubisZone2Actor.cpp:1-340
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Public/NubisVDBDataAsset.h:1-205
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Private/NubisVDBDataAsset.cpp:1
  - Engine/Source/Runtime/Engine/Public/NubisVolumeInterface.h:127-413 (raw#1 已记录,此处仅引用)
---

## 0. 模块定位与依赖图

`NubisCustom` 插件下并存两个模块,新路径只用 `NubisCustom2`(老路径 `NubisCustom` 不在本笔记范围)。
模块 ID: `NUBISCUSTOM2_API`,Build 类型 `UseExplicitOrSharedPCHs`。

**Build.cs 依赖** (NubisCustom2.Build.cs:25-69):

| 类别 | 模块 | 用途 |
|---|---|---|
| Public | Core/CoreUObject/**Engine**/RHI | UObject + INubisVolumeInterface(Engine 端) + RHI 类型(`FRHITexture`、`RHIUpdateTexture3D`) |
| Public | GeometryCore/GeometryFramework | [推测] 烘焙阶段 mesh 处理,运行时不直接用 |
| Private | RenderCore/**Renderer** | `ENQUEUE_RENDER_COMMAND`、`FlushRenderingCommands` |
| Private | Niagara/NiagaraCore/NiagaraShader | [推测] 烘焙阶段 Niagara 模拟,运行时未发现使用点 |
| Private | Projects | `IPluginManager::FindPlugin("NubisCustom")` 取 Shaders 路径 |
| Private | DeveloperSettings/MeshConversion | [推测] 烘焙阶段 |
| Editor only | UnrealEd | Editor 工具(只在 `Target.Type == Editor` 注入) |

**StartupModule** (NubisCustom2.cpp:12-18) 只做 Shader 路径映射 `/Plugin/NubisCustom → <Plugin>/Shaders/`,没有 UCLASS 注册或 RT 资源初始化。Tick 入口在 `UNubisClipmapSubsystem`,见 §2。

## 1. UCLASS / USTRUCT 总览(去重 raw#8 已表)

| 类名 | 父类 | 文件:行号 | 角色 |
|---|---|---|---|
| `FNubisCustom2Module` | IModuleInterface | NubisCustom2.h:10 | StartupModule 仅注册 shader 路径 |
| `UNubisClipmapSubsystem` | UTickableWorldSubsystem | NubisClipmapSubsystem.h:30 | World 级 Tick 驱动器,持有 N 个 Manager |
| `NubisClipmap::NubisClipmapManager` | **FGCObject (非 UCLASS)** | NubisClipmap.h:81 | 单 Zone 的 Clipmap 状态机,唯一拥有 Per-Level VolumeTexture / MID / LightingCacheRT |
| `ANubisZone2Actor` | AActor | NubisZone2Actor.h:14 | 关卡放置体,持有 `UHeterogeneousUBSVolumeComponent` 子组件,转发材质参数 |
| `ANubisHiCloud2Actor` | AActor | NubisHiCloud2Actor.h:22 | **离散云朵**预览/烘焙 Actor,持有 `UStaticMeshComponent` 预览体 |
| `UNubisClipmapDataAsset` | UPrimaryDataAsset | NubisClipmapDataAsset.h:58 | 烘焙产物:Sector→Texture 软引用索引表 |
| `UNubisVDBDataAsset` | UPrimaryDataAsset | NubisVDBDataAsset.h:189 | 烘焙输入:云 Snapshot + Sector 重导出软引用 (raw#8 已展开) |
| `FNubisClipmapConfig` | (POD) | NubisClipmap.h:39 | per-Zone 运行时 Clipmap 配置 |
| `NubisClipmap::NubisLevelInfo` | (POD) | NubisClipmap.h:218 | 单 MIP Level 状态(VolumeTexture+SectorStates+ScrollOffset+ClipmapOrigin) |
| `NubisClipmap::NubisSectorState` | (POD) | NubisClipmap.h:201 | 单 Sector 状态机(LoadHandle/LoadState/bHasCloud) |
| `FNubisClipmapCacheElement` | USTRUCT | NubisClipmapDataAsset.h:9 | DataAsset 表项:Modeling+SDF 软引用 + MinSectorMipLevel |
| `FNubisClipmapSectorCache` | USTRUCT | NubisClipmapDataAsset.h:31 | 一个 MIP Level 下所有 Sector 的索引表 |
| `FNubisClipmapLevelCache` | USTRUCT | NubisClipmapDataAsset.h:46 | 整个 Zone 的全部 MIP 索引根 |
| `ECloudPresetType` | UENUM uint8 | NubisHiCloud2Actor.h:11 | 三档参数预设(MeshAttached/PureVisual/Custom) |
| `FSectorKey` | USTRUCT | NubisVDBDataAsset.h:9 | (raw#8) `(GridCoord, MipLevel)` 联合 key,带 `GetTypeHash` |
| `FNubisHoudiniParams` | USTRUCT | NubisVDBDataAsset.h:34 | (raw#8) 16 个 Houdini 浮点 |
| `FNubisCloudSnapshot` | USTRUCT | NubisVDBDataAsset.h:117 | (raw#8) Snapshot 主体 |

## 2. 类层次 / 持有关系(Mermaid)

```mermaid
graph TD
    UWorld[UWorld] --> Sub[UNubisClipmapSubsystem<br/>UTickableWorldSubsystem]
    Sub -->|TMap Zone→Manager<br/>非 UPROPERTY/手工管理| Mgr[NubisClipmapManager<br/>FGCObject 非 UCLASS]

    Sub -.UPROPERTY 防 GC.-> RegMap[TMap RegisteredZones]
    RegMap -.->Zone

    Mgr -->|Levels[2][MipCount]| LvlInfo[NubisLevelInfo<br/>VolumeTexture/ScrollOffset/SectorStates]
    LvlInfo --> Sec[NubisSectorState<br/>LoadHandle/LoadState/bHasCloud]
    Mgr -->|每级一个 MID| MID[UMaterialInstanceDynamic<br/>×MipCount]
    Mgr -->|每级一张 LightingCache RT| LCache[UTextureRenderTargetVolume<br/>×MipCount]
    Mgr -->|MipSelector Atlas<br/>PF_R8_UINT,SW.Z×MipCount 段| MS[UVolumeTexture]
    Mgr -->|EmptySectorTexture[2]| Empty[UVolumeTexture<br/>BC6H+DXT1]
    Mgr -.缓存指针.-> Zone

    Zone[ANubisZone2Actor : AActor] -->|RootComponent| Scene[USceneComponent]
    Zone -->|子 Component<br/>引擎端 UCLASS| HUBS[UHeterogeneousUBSVolumeComponent]
    Zone -->|EditAnywhere TObjectPtr| DA[UNubisClipmapDataAsset]

    HiCloud[ANubisHiCloud2Actor : AActor] -->|StaticMeshComponent| Preview[UStaticMeshComponent<br/>SetCanEverAffectNavigation=false]
    HiCloud -->|MID| HiMID[UMaterialInstanceDynamic]

    DA -->|TMap CacheSectorMap| LCache2[FNubisClipmapSectorCache]
    LCache2 -->|TMap CacheTextureMap| Elem[FNubisClipmapCacheElement<br/>SoftPtr Modeling+SDF<br/>MinSectorMipLevel]

    VDB[UNubisVDBDataAsset] -->|TMap SavedClouds| Snap[FNubisCloudSnapshot]
    VDB -->|TMap ExistingSectorTextures| SecTex[TSoftObjectPtr UVolumeTexture]
    Snap -->|TSoftObjectPtr| SVT[UStaticSparseVolumeTexture<br/>VDBFromHDA]

    Mgr -.SetClipmapData/SetMipSelectorVolume<br/>SetPerLevelLightingCacheRTs.-> HUBS
    HUBS -.SceneProxy 构造<br/>→ FNubisVolumeData<br/>raw#1.-> RT[Render Thread]
```

**关键事实**:
- `NubisClipmapManager` 是 **C++ 普通类继承 FGCObject**(NubisClipmap.h:81),不是 UCLASS。GC 引用通过 `AddReferencedObjects` 手动注册:`ClipmapMIDs`、`CacheDataAsset` (NubisClipmap.cpp:71-79)。`Levels[][].VolumeTexture`、`MipSelectorVolume`、`EmptySectorTexture` 都靠 **`AddToRoot`** 防 GC(grep 见后)。
- Subsystem 持有 Manager 用 **裸 `TMap<ANubisZone2Actor*, TUniquePtr<Manager>>`**(NubisClipmapSubsystem.h:90),配合并行的 `UPROPERTY` 占位 TMap (`RegisteredZones`,行 87)只为给 GC 看 Key 的 Actor 引用。

## 3. 生命周期时序(BeginPlay → Tick → EndPlay)

```
GT (Game Thread) Frame N:
─────────────────────────────────────────────────────────────────
ANubisZone2Actor::BeginPlay() (NubisZone2Actor.cpp:143-154)
  └─> UNubisClipmapSubsystem::Get(World)->RegisterZone(this, ClipmapDataAsset)

Subsystem::RegisterZone (NubisClipmapSubsystem.cpp:139-197)
  ├─> MakeUnique<Manager>
  ├─> Manager.ApplyConfig({ MipCount=Zone->ClipmapMipCount, TextureSize/SectorSize/BaseVoxelSize 来自 DataAsset })
  ├─> Manager.Initialize(DataAsset)
  │     ├─ for(Type∈{Modeling,SDF}) for(Level∈[0,MipCount))
  │     │     └─ Levels[Type][Level].VolumeTexture = CreateTransient(PhysicClipmapSize, BC6H/DXT1)
  │     │        ─ Rename → MakeUniqueObjectName,  AddToRoot
  │     │        ─ FillBC6HDefault / memzero,      UpdateResource()
  │     ├─ EmptySectorTexture[Type] = CreateTransient(SectorSize, …)
  │     ├─ StreamableManager.SetManagerName("NubisClipmapStreamableManager")
  │     ├─ CacheDataAsset = DataAsset
  │     └─ CreateMipSelectorVolume()
  │           ├─ CreateTransient(SectorWidth.X, SectorWidth.Y, SectorWidth.Z*MipCount, PF_R8_UINT)
  │           ├─ AddToRoot
  │           └─ ENQUEUE_RENDER_COMMAND(InitMipSelectorAtlas) → RHIUpdateTexture3D 整张填哨兵 0x0E
  ├─> Manager.InitClipmapLevels(Zone)  (NubisClipmap.cpp:2576-2753)
  │     ├─ ClipmapConfigs.SetNum(MipCount)
  │     ├─ for(Level) MID[Level] = UMaterialInstanceDynamic::Create(VolumeComponent.Material[0], Zone)
  │     │   └─ MID->SetTextureParameterValue("VoxelCloudModelingDataTexture_MipN", VolumeTexture)  (× 所有 MIP 交叉绑定)
  │     ├─ PerLevelLightingCacheRTs[Level] = NewObject<UTextureRenderTargetVolume>(VolumeComponent, RF_Transient)
  │     │   └─ Init(LightingCacheRes, PF_R16F); UpdateResourceImmediate(true)
  │     ├─ VolumeComponent->SetPerLevelLightingCacheRTs(PerLevelLightingCacheRTs)
  │     ├─ VolumeComponent->SetMipSelectorVolume(MipSelectorVolume)  ← ★ 必须先于 SetClipmapData
  │     └─ VolumeComponent->SetClipmapData(MipCount, ClipmapConfigs, ClipmapMIDs)
  │           └─ 内部 MarkRenderStateDirty → 下一帧 RT 重建 SceneProxy → 构造 FNubisVolumeData (raw#1)
  └─> ClipmapManagers.Add(Zone, MoveTemp(Mgr))

GT Frame N+k (Tick): UNubisClipmapSubsystem::Tick (NubisClipmapSubsystem.cpp:59-128)
  └─ CameraPos = GetCameraWorldPosition() (Editor: LevelViewportClient with priority A>B>C>D; Game: PC->GetPlayerViewPoint)
     for each (Zone, Mgr):
       Mgr->Update(CameraPos)  → §4 详述
     [Editor only] Mgr->DrawDebugBoundingBoxes/DrawDebugSectors

GT EndPlay / Destroyed: ANubisZone2Actor::EndPlay (NubisZone2Actor.cpp:156-164) / Destroyed:167-179
  └─> Subsystem::UnregisterZone(this) → Mgr->Shutdown() → 移除 TMap

UWorld::Deinitialize → Subsystem::Deinitialize (NubisClipmapSubsystem.cpp:38-53)
  └─ for each Mgr: Mgr->Shutdown(); ClipmapManagers.Empty()
```

**Editor 模式特殊路径** (NubisZone2Actor.cpp:67-88, 90-140):
- `OnConstruction` 在编辑器(`!IsGameWorld`)下 Unregister + Register 一次,所以拖动 Actor、属性面板任何重命名都会**重建 Manager + 重传 RT**。
- `PostEditChangeProperty` 区分两路:`ClipmapDataAsset / ClipmapMipCount` 改 → 完全重建;光照/PhaseG/材质参数改 → 仅 `UpdateMaterialParams()`(NubisZone2Actor.cpp:203-218)→ Manager.UpdateMaterialParams(Zone) 只刷 MID 标量/向量参数,不重建 GPU 资源。

## 4. Update() 频率与触发条件(NubisClipmap.cpp:220-735)

**频率**: 由 `UTickableWorldSubsystem::Tick` 驱动,**每帧** GT 调用一次。`IsTickableInEditor() = true`(NubisClipmapSubsystem.h:43)→ Editor 关闭 PIE 时也会 Tick。Subsystem 调度顺序:遍历整个 `ClipmapManagers` 表,挨个 `Manager->Update(CamPos)`。

**滚动判定** (NubisClipmap.cpp:259-310):
- `CameraSectorPos = RoundToInt(CamPos / SectorWorldSize)`,`SectorWorldSize = VoxelSize_cm × SectorSize_voxels`。
- **Z 轴各向异性死区** (CVar `r.Nubis.Clipmap.ZUpdateStretch`, 默认 -1 自动 = X/Z 比例):云层扁,Z 方向窗口窄,把 Z 判定拉长 N 倍 sector。`ForceBoundaryUpdate` 兜底:相机 Z 跑到窗口边缘外强制滚。
- 触发滚动的硬条件:`OldSectorPos != NewSectorPos`,逐 Mip 逐 Type 独立判定。首帧 `bFirstUpdate=true` 强制全量。
- Z stretch CVar 是 **CVarValue.GetValueOnAnyThread()**,GT 安全调用。

**多 Type 串行**: 每帧每 Mip 做 Modeling 和 SDF 两次相同的滚动判定(NubisClipmap.cpp:247),意味着同一级 ScrollOffset 在两 Type 之间是 **独立计算但天然一致**(同 CamPos + 同 SectorWorldSize → 同 CameraSectorPos)。LightingCache DirtyRegions 只在 `TypeIdx == 0`(Modeling) 趟里写一次(NubisClipmap.cpp:407)避免重复。

**6 级独立 Update**: `for (int i = 0; i < MipCount; i++)`,每级有自己的 `VoxelSize = 2^i × BaseVoxelSize × 100cm`、自己的 `SectorWorldSize`、自己的 `CameraSectorPos`(高级 mip SectorWorldSize 大,需要相机走更远才滚一次)。

**资源消耗端**:
- `EnqueueLoadRequest` (NubisClipmap.cpp:757-852):无数据 sector→`ClearSectorInClipmap`(走 `CopySectorToClipmap` from `EmptySectorTexture[Type]`,纯 GPU Copy 不限流);有数据 sector→入 `PendingLoadQueue` 带 `Priority = LevelIdx*100 + NormDistance[0,99]`。
- `ProcessPendingLoads` (NubisClipmap.cpp 引用 :630):每帧消费 `MaxLoadsPerFrame=8`(首帧 ×4=32),并发上限 `MaxConcurrentLoads=16`。
- `RequestTextureLoad`(NubisClipmap.cpp:964-1017):`StreamableManager.RequestAsyncLoad(SoftPath, Delegate)`,回调 `HandleTextureLoadComplete` → `MergeLoadedTexture` → `CopySectorToClipmap`(`ENQUEUE_RENDER_COMMAND(CopyClipmapSectorTexture)` → RT 上 `RHICmdList.CopyTexture`)。

## 5. 多 Zone 合并策略

```
ClipmapManagers : TMap<ANubisZone2Actor*, TUniquePtr<NubisClipmapManager>>
              N entries, **每个 Zone 一份 6 级 Clipmap + MID + LightingCacheRT**
```

**当前实现是 per-Zone 独立**(NubisClipmapSubsystem.cpp:139-197 的 `RegisterZone`):
- 没有跨 Zone 合并 atlas 的代码路径。`Initialize()` 直接为 (Type=2) × (MipCount=6) 共 12 张 VolumeTexture 走 `CreateTransient`。
- `UHeterogeneousUBSVolumeComponent` 也是 per-Zone 子组件(NubisZone2Actor.cpp:22),引擎端 SceneProxy 与 Component 1:1。
- 渲染端 `RenderNubisVolumes` 调度循环(raw#1 已记录)按 Scene 内所有 NubisVolumeProxy 各自一遍 RDG passes。

**Sector 全局对齐**: 同 mip 不同 Zone 的 SectorIndex 用相同的 `floor(WorldPos / SectorWorldSize)` 公式(NubisClipmap.cpp:782-793),理论上**坐标系是统一的**——但每个 Zone 自己的 ClipmapDataAsset 只索引自己烘焙范围内的 sector,**不存在两 Zone 同 mip 同 SectorKey** 这种合并需求 [推测]。
**[开放问题]** 如果两个 Zone 的 BoundingBox 物理重叠,运行时会出现 6+6=12 级独立 Clipmap 各自 raymarch,RT 端是否合成或先后覆盖,需查 raw#1 的 RenderNubisVolumes 调度细节(本笔记不展开)。

`HiCloud2Actor` 与 `Zone2Actor` 的关系:
- `Zone2Actor` **不持有** `HiCloud2Actor` 列表。后者是 Editor 工具(`ANubisHiCloud2Actor::ShowHiCloudMesh` 通过 `UGameplayStatics::GetAllActorsOfClass` 全图遍历,NubisZone2Actor.cpp:241-263)。
- HiCloud 是 **per-cloud 烘焙单元**,`FNubisCloudSnapshot.ActorUniqueID = FGuid` 在 `UNubisVDBDataAsset.SavedClouds` 里以 GUID 索引(raw#8)。多个 HiCloud → 一个 VDB DataAsset → 通过烘焙流水线 → `UNubisClipmapDataAsset.ClipmapLevelCacheMap` → 一个 Zone 消费,这是 N:1:1:1 关系。

## 6. NubisVDBDataAsset 字段表(B4 焦点,扩展 raw#8)

| 字段 | 类型 | 用途 | GT/RT 归属 | 是否驱动 GPU 资源 |
|---|---|---|---|---|
| `SavedClouds` | `TMap<FGuid, FNubisCloudSnapshot>` | 烘焙输入:每个 HiCloud 的 Snapshot 缓存 | 仅 GT(Editor 烘焙) | 否,运行时不消费 |
| `SavedClouds[i].CacheMipLevel` | int32 | Mesh 外接盒装入哪档 sector(粗粒度路由) | GT-Editor | 否 |
| `SavedClouds[i].SectorMipLevel` | int32 | SDF 实际厚度路由的 mip(`[3, MipCount-1]` 或 `INDEX_NONE`) | GT-Editor → 写入 ClipmapDataAsset.MipLevels | 间接驱动 sector 入哪张 Clipmap VolumeTexture |
| `SavedClouds[i].WorldMeshBounds` | FBox | 该云的 mesh AABB | GT-Editor | 否 |
| `SavedClouds[i].GlobalLocation/Scale/Rotation` | FVector/FVector/FRotator | Snapshot 时刻 transform | GT-Editor | 否 |
| `SavedClouds[i].ActorUniqueID` | FGuid | HiCloud Actor 唯一 ID,= TMap key | GT-Editor | 否 |
| `SavedClouds[i].HoudiniParams` | FNubisHoudiniParams (16 floats) | 烘焙参数,用于脏判定(operator==,KINDA_SMALL_NUMBER) | GT-Editor | 否,纯 CPU 侧脏判定 |
| `SavedClouds[i].VDBFromHDA` | TSoftObjectPtr<**UStaticSparseVolumeTexture**> | Houdini HDA 输出的 SVT 资产 | GT-Editor 软引用,Editor 烘焙时 LoadSync → SVT pages 上传 GPU | 烘焙时驱动一次性 GPU 上传 |
| `ExistingSectorTextures` | `TMap<FSectorKey, TSoftObjectPtr<UVolumeTexture>>` | 烘焙阶段去重表(已存在 sector RT 复用) | GT-Editor | 否,运行时未消费(运行时走 `UNubisClipmapDataAsset` 的 CacheTextureMap) |
| `NubisGUID` | FString | 该 DataAsset 唯一标识(用于跨实例校验) | GT-Editor | 否 |

**关键发现 — VDB DataAsset 不参与运行时**:
- `NubisVDBDataAsset.cpp` 全文只一行 `#include`,**没有 PostLoad/Serialize 自定义**。
- `Reset()` 是**唯一的成员函数**(NubisVDBDataAsset.h:203),由 Editor 烘焙工具调用清空 SavedClouds。
- 运行时 GPU 上传链 **完全不走 VDBDataAsset**;走的是 `UNubisClipmapDataAsset.ClipmapLevelCacheMap.CacheSectorMap[Level].CacheTextureMap[SectorIdx].CachedSectorColorTexture`(已经是烘焙后压成 BC6H/DXT1 的 `TSoftObjectPtr<UVolumeTexture>`)。
- 因此 **B4 的"运行时 Sector 加载时机"答案** = `Subsystem::Tick → Manager::Update → EnqueueLoadRequest → ProcessPendingLoads → StreamableManager.RequestAsyncLoad`,**完全 lazy**,只对当前 Clipmap 窗口内 sector 做异步流式,**不存在 BeginPlay 全加载**。

## 7. GPU 资源生命周期 — Plugin 内 RT 触达点

| RT 操作 | 文件:行 | Plugin 自己做 vs 走 Engine | 用途 |
|---|---|---|---|
| `ENQUEUE_RENDER_COMMAND(SetNubisVolumeTextureDebugName)` | NubisClipmap.cpp:1220 | Plugin 自己做 | RHI debug 名(截帧用) |
| `ENQUEUE_RENDER_COMMAND(InitMipSelectorAtlas)` + `RHIUpdateTexture3D` 全张哨兵 | :1344-1357 | Plugin 自己做 | MipSelector Atlas 初始填 0x0E,**唯一一处对 RT atlas 的批量上传** |
| `ENQUEUE_RENDER_COMMAND(UpdateMipSelectorEntry)` + `RHIUpdateTexture3D(1×1×1)` | :1689-1700 | Plugin 自己做 | sector 上线/下线时单字节增量更新 atlas |
| `ENQUEUE_RENDER_COMMAND(CopyClipmapSectorTexture)` + `RHICmdList.CopyTexture` | :2085-2100 | Plugin 自己做 | sector RT(BC6H/DXT1) → Clipmap VolumeTexture 环形位置 |
| `LoadedTexture->UpdateResource()` (BC6H mip strip 后) | :2399 | 走 Engine `UTexture::UpdateResource` | Mip strip 后重建 GPU 资源 |
| `MipSelectorVolume->UpdateResource()` | :1326 | 走 Engine | atlas 创建后初始 GPU 资源(BulkData→GPU,但不保证落地,所以再来一次 `RHIUpdateTexture3D` 强制) |
| `NewTexture->UpdateResource()` (Clipmap VolumeTexture 创建) | :1211 | 走 Engine | per-Mip Clipmap VolumeTexture 初始 GPU |
| `VolumeComponent->MarkRenderStateDirty()` | :201, :712 | 调 Engine 端 Component | SceneProxy 重建触发 |
| `VolumeComponent->SetClipmapData()` | :711, :2745 | 调 Engine 端 Component | 推送 ClipmapConfigs+MID 数组 |
| `VolumeComponent->SetMipSelectorVolume()` | :2744 | 调 Engine 端 Component | atlas 句柄推送(Component 内部 ENQUEUE 同步 RHI 到 FNubisVolumeData,raw#1) |
| `VolumeComponent->SetPerLevelLightingCacheRTs()` | :2703 | 调 Engine 端 Component | 6 张 LightingCache RT 推送 |
| `VolumeComponent->SyncClipmapScrollToProxy_RenderThread()` | :720 | 调 Engine 端 Component | 当帧把新 ScrollUVOffset 即时推 SceneProxy(避免下一帧重建期间 UV 错位) |
| `FlushRenderingCommands()` | :207 | 调 Engine | Shutdown 期间等渲染线程把旧 SceneProxy 处理完再 ReleaseVolumeTextures |
| `BeginInitResource` | (无) | — | grep 0 命中,Plugin 不直接管 FRenderResource |

**结论 (B4)**:
1. **Plugin 自己持有 GPU 资源**:Per-Mip Clipmap VolumeTexture(2×6 张)、MipSelector Atlas(1 张)、EmptySectorTexture(2 张)、PerLevelLightingCacheRT(6 张)。这些都是 `UVolumeTexture`/`UTextureRenderTargetVolume`,通过 `UpdateResource()`(走 Engine 标准路径)上 GPU,生命周期靠 `AddToRoot` + 析构函数 `RemoveFromRoot`。
2. **Plugin 自己做 RT 操作**:仅限 `RHIUpdateTexture3D`(MipSelector 单字节/全张) 和 `RHICmdList.CopyTexture`(sector 写入 Clipmap)。**不直接接 RDG**——RDG Pass 在 Engine 端 `NubisVolumesLiveShadingPipeline.cpp` (raw#3),Plugin 只把 `UVolumeTexture::Resource->TextureRHI` 通过 Component → SceneProxy → `FNubisVolumeData::*RHI` 字段传过去。
3. **跨模块边界**:`UHeterogeneousUBSVolumeComponent::Set*` 系列(`SetClipmapData/SetMipSelectorVolume/SetPerLevelLightingCacheRTs/SyncClipmapScrollToProxy_RenderThread`)是 Plugin → Engine 唯一的写入接口。读取(SceneProxy 构造时)由 Engine 自己负责。

## 8. GPU 资源 → Shader Parameter 绑定表

| GPU 资源 | Plugin 端持有者 | 推送方式 | RT 端落点 | Shader 绑定 |
|---|---|---|---|---|
| `Levels[Modeling][L].VolumeTexture` (PF_BC6H, TextureSize) | NubisClipmapManager | `MID->SetTextureParameterValue("VoxelCloudModelingDataTexture_MipL", T)` (NubisClipmap.cpp:2657) + `_Mip0..5` 全交叉绑定 | MID → SceneProxy 材质 | Material 节点 `VoxelCloudModelingDataTexture_MipN`,`SwitchByInt(CurrentRenderMipLevel)` 选档 |
| `Levels[SDF][L].VolumeTexture` (PF_DXT1, TextureSize) | 同上 | `VoxelCloudSDFTexture_MipL`(:2662) | 同上 | Material 节点 `VoxelCloudSDFTexture_MipN` |
| `MipSelectorVolume` (PF_R8_UINT, SW.X×SW.Y×SW.Z×MipCount) | 同上 | `VolumeComponent->SetMipSelectorVolume(MS)` (:2744) | `FNubisVolumeData::MipSelectorTextureRHI` (raw#1) | RDG `MipSelectorTexture` (`Texture3D<uint>`) — raw#3 已表 |
| `PerLevelLightingCacheRTs[L]` (PF_R16F, LightingCacheRes) | 同上 | `VolumeComponent->SetPerLevelLightingCacheRTs(arr)` (:2703) | (raw#3 推断为 RDG 双缓冲源端) | LightingCache CS 写入,RayMarch 采样 |
| `EmptySectorTexture[Type]` (PF_BC6H/DXT1, SectorSize) | 同上 | 不上传到 SceneProxy,仅作 `RHICmdList.CopyTexture` 源 | RT(本帧 Copy 源) | 不绑定到 Shader,仅 Copy Pass 用 |
| `ClipmapMIDs[L]` 标量参数 `CurrentRenderMipLevel` | 同上 | MID Set | SceneProxy 材质 | Material 节点输入 |
| `ClipmapMIDs[L]` 向量参数 `ClipmapScrollUVOffset` | 同上(NubisClipmap.cpp:675) | MID Set | 同上 | Material 节点 + raw#3 三端镜像之一 |

**与 raw#1/3 的边界划分**:
- `FNubisClipmapLevelRenderConfig` 结构体本身在 Engine 端 `NubisVolumeInterface.h`(raw#1),Plugin 通过 `SetClipmapData` 把数组推过去——**Plugin 不直接在 RDG SHADER_PARAMETER 里绑任何东西**。
- raw#3 的 PassParameter 表已经记录了 `MipSelectorTexture / NubisSectorSize / NubisClipmapOriginSectorIdxArray[6]` 等;本笔记的贡献是补全 **Plugin 哪个 UObject 字段对应 RDG 哪个绑定**。

## 9. 关键 grep 验证(Plugin 内是否完全走 Engine 转发)

**`ENQUEUE_RENDER_COMMAND` 在 Plugin 内的总数 = 4**(NubisClipmap.cpp:1220, 1344, 1689, 2085 — 1220 设 debug name、1344 init atlas、1689 单字节更新、2085 sector copy)。**没有一处直接构造 RDG Builder 或 SHADER_PARAMETER 结构**。所有 RDG Pass 都在 Engine 端 `NubisVolumesLiveShadingPipeline.cpp`(raw#3)。结论:**Plugin 与 RDG 解耦,只在 RHI 层操作纹理资源,RDG 端通过 Engine 端 Component → SceneProxy → FNubisVolumeData 隔层取 RHI 句柄**。

**`BeginInitResource` 在 Plugin 内的命中数 = 0**:Plugin 不自定义 `FRenderResource`,所有 GPU 资源都是标准 `UVolumeTexture`/`UTextureRenderTargetVolume`,生命周期由 Engine `UTexture::UpdateResource` 接管。

**`RequestAsyncLoad` 命中 = 1**(NubisClipmap.cpp:1008):每个 sector 独立 RequestAsyncLoad,回调发到 GT,在 GT 上 `UpdateResource` + Enqueue CopyTexture。**没有 LoadSynchronous**,即运行时不会同步阻塞 GT 加载。

**`UHeterogeneousUBSVolumeComponent` 在 Plugin 内出现 = 5 处**(NubisClipmap.cpp:195, 706, 2576;NubisClipmap.h:10 fwd;NubisZone2Actor.h:214/cpp:22)。**Zone Actor 直接持有此 Component 作为 DefaultSubobject**,验证了"Actor 持有 Engine 端 Component 间接使用 INubisVolumeInterface"(raw#1 前提)。

## 10. Shutdown 安全顺序(NubisClipmap.cpp:173-218)

```
1. PendingLoadQueue.Empty(); ActiveLoadCount = 0
2. StrippedAssets.Empty()  // 否则下次 Initialize 老路径仍占用
3. for(Mip,Type) CancelTextureLoad
4. VolumeComponent->ClipmapLevelCount = 0; ClipmapConfigs.Empty(); ClipmapMaterialProxies.Empty()
5. VolumeComponent->MarkRenderStateDirty()  // SceneProxy 重建后 Clipmap 循环不进
6. FlushRenderingCommands()  // 等旧 SceneProxy 死透
7. ReleaseVolumeTextures(); ReleaseMipSelectorVolume()  // RemoveFromRoot
8. ClipmapConfigs.Empty(); ClipmapMIDs.Empty(); CachedZone = null; CacheDataAsset = null
```

注释里点明**关键陷阱**:如果不先 Step 4-6 就直接 Step 7,旧 SceneProxy 在 RT 上仍持有对已 RemoveFromRoot 的 VolumeTexture 的 RHI 引用,**会出现 use-after-free**。这给出了 RT 资源生命周期的硬约束:**Plugin 释放任何 GPU 资源前必须先把 SceneProxy 这条引用链切断 + Flush**。

## 11. 结合 NubisClipmapDataAsset 的运行时消费链

```
DataAsset (Persistent) :
  ClipmapLevelCacheMap.CacheSectorMap[Level].CacheTextureMap[SectorIdx]
    = FNubisClipmapCacheElement {
        CachedSectorColorTexture: TSoftObjectPtr<UVolumeTexture>,  // BC6H
        CachedSectorSDFTexture:   TSoftObjectPtr<UVolumeTexture>,  // DXT1
        MinSectorMipLevel:        int32                            // 用于 MipSelector entry
      }

运行时 (Update Tick):
  EnqueueLoadRequest(Type, Level, SectorIdx) :
    Element = DataAsset.Find(Level, SectorIdx)
    if (!Element || soft.IsNull) → ClearSectorInClipmap (CopyTexture from EmptySector)
    else → PendingLoadQueue.Add({SoftPath, Priority})
  ProcessPendingLoads (limited by MaxLoadsPerFrame) :
    StreamableManager.RequestAsyncLoad(SoftPath, Delegate)
  Delegate fires (GT) :
    HandleTextureLoadComplete → MipsRemoveAt(1, …) (BC6H 兼容修复) → UpdateResource()
    → MergeLoadedTexture → CopySectorToClipmap (ENQUEUE_RENDER_COMMAND CopyTexture)
    → 若 Modeling+SDF 都 InGPU → PropagateMipSelectorChange → UpdateMipSelectorEntry (RHIUpdateTexture3D 1×1×1)
```

**MinSectorMipLevel 的运行时落点** (NubisClipmapDataAsset.h:27):烘焙时记录"该 sector 命中的最细 mip",运行时 `RecomputeMipSelectorEntry`(NubisClipmap.cpp:1379)读取,写到 MipSelector Atlas 的 bit1-3,供 RDG raymarch 跳跃(raw#3 已表)。

## 12. 开放问题

1. **多 Zone 物理重叠时的渲染顺序**:Plugin 端确认每 Zone 独立维护 6 级 Clipmap + Component,但 RT 端 `RenderNubisVolumes` 调度循环对 N 个 NubisVolumeProxy 是合成还是先后覆盖,**未在本笔记 grep 范围**。需查 raw#1 第 N 节 RenderNubisVolumes。
2. **`PerLevelLightingCacheRTs` 的 Per-Frame 双缓冲**:NubisClipmap.cpp:2685-2697 创建是单张 `UTextureRenderTargetVolume`,但 raw#1 提到 Engine 端 `FNubisRenderTargetViewStateData` 双缓冲。两者是同一份还是 Plugin 端 RT 给 Engine 端复制成双缓冲?[推测] 后者,但需查 NubisVolumeComponent.cpp 的 `SetPerLevelLightingCacheRTs`。
3. **`StrippedAssets` 集合的全局性**:目前 per-Manager 维护(NubisClipmap.h:359)。如果两个 Zone 共享同一个烘焙 Asset(理论上应该不会,但 DataAsset 可手动赋值),Strip 会做两次。
4. **`ANubisHiCloud2Actor.bUseCollision/bShowMesh` 的运行时意义**:这些字段在烘焙阶段决定 mesh 行为,运行时 `BeginPlay()` 内为空(NubisHiCloud2Actor.cpp:57-60),也就是说 **HiCloud Actor 在运行时是死的**——只有 PreviewCloudStaticMesh 还在场景里(可能影响碰撞/导航)。是否应该在 Shipping 中剔除?需查 raw#8 烘焙后是否 spawn HiCloud Actor 到正式关卡。
5. **`Niagara*` 模块在 Build.cs 的依赖**:运行时 grep 0 命中 Niagara API。**可能仅烘焙阶段使用**(raw#8 说 Houdini → SVT → 烘焙),需要进一步在烘焙工具(Editor 模块)源码中确认。
