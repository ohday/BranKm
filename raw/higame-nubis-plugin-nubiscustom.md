---
title: "NubisCloud 老路径插件 NubisCustom + Editor Tools + SDFCompress"
type: Local code archaeology
fetched: 2026-05-11
project: higame-nubis-cloud
relevant_to: B1, B2, B5
sources_drawn_from:
  - Projects/HiGame/Plugins/NubisCustom/NubisCustom.uplugin:17-49
  - Projects/HiGame/Plugins/NubisCustom/Config/DefaultNubisCustom.ini:1-2
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom/NubisCustom.Build.cs:25-52
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom/Public/NubisCustom.h:10-18
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom/Public/NubisCustomSubsystem.h:13-36
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom/Private/NubisCustomSubsystem.cpp:32-72
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom/Public/NubisHiCloudActor.h:9-41
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom/Private/NubisHiCloudActor.cpp:1-42
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom/Public/NubisZoneActor.h:9-31
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom/Private/NubisZoneActor.cpp:1-33
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom/Public/NubisGenSDFComponent.h:11-131
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom/Private/NubisGenSDFComponent.cpp:30-35
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom/Public/NubisDataAsset.h:8-114
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisEditorTools/NubisEditorTools.Build.cs:27-94
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisEditorTools/Private/NubisEditorTools.cpp:17-115
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisEditorTools/Private/NubisToolsCommands.cpp:7-10
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisEditorTools/Private/NubisManagerCommands.cpp:9-19
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisEditorTools/Private/NubisActorFilter.cpp:22-37
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisEditorTools/Private/NubisZoneEditorSubsystem.cpp:9-85
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisEditorTools/Public/SNubisToolsPanel.h:23-540
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisEditorTools/Private/SNubisToolsPanel.cpp:164-405
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisEditorTools/Private/SNubisToolsPanel.cpp:505-625
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisEditorTools/Public/BP_NubisToolLibrary.h:113-402
  - Projects/HiGame/Plugins/NubisCustom/Source/SDFCompressEditor/SDFCompressEditor.Build.cs:27-73
  - Projects/HiGame/Plugins/NubisCustom/Source/SDFCompressEditor/Public/BC1ScalarEncoder.h:24-87
  - Projects/HiGame/Plugins/NubisCustom/Source/SDFCompressEditor/Public/BC1ScalarSDFCompressor.h:11-134
  - Projects/HiGame/Plugins/NubisCustom/Source/SDFCompressEditor/Private/SDFCompressEditorModule.cpp:7-19
---

# NubisCloud 老路径插件 NubisCustom + Editor Tools + SDFCompress

> 继续 raw#1/#7/#8。本笔记只覆盖 **老路径 NubisCustom 模块**、**NubisEditorTools 模块**、**SDFCompressEditor 模块**，不再重复 raw#7 已覆盖的 NubisCustom2 或 raw#8 已覆盖的烘焙流水线。

## 1 插件 4 模块一览

`NubisCustom.uplugin` 只声明了 4 个模块，全部 `LoadingPhase=PostConfigInit`,全部 `PlatformDenyList=["Linux"]`,外加 2 个显式启用的依赖插件 `Niagara` / `BlueprintMaterialTextureNodes` (NubisCustom.uplugin:17-60)。

```mermaid
flowchart TB
  subgraph RuntimePhase[LoadingPhase=PostConfigInit | PlatformDenyList=Linux]
    Old["NubisCustom (Runtime)<br/>老路径<br/>4 个 UCLASS + SDF Compute Shader"]:::old
    New["NubisCustom2 (Runtime)<br/>新路径 raw#7 覆盖"]:::new
    EdTools["NubisEditorTools (Editor)<br/>Slate Panel + BP Library + Outliner Filter<br/>EditorSubsystem"]:::edtool
    SDFEd["SDFCompressEditor (Editor)<br/>BC1 标量编码器 raw#8"]:::sdf
  end

  EdTools -->|PrivateDep| Old
  EdTools -->|PrivateDep| New
  EdTools -->|PrivateDep| SDFEd
  EdTools -->|PrivateDep| HGE[HiGameEditorEntry]
  SDFEd -.独立.-> X((无依赖<br/>到老/新 Runtime))
  Old -.无依赖到新路径.-> New

  classDef old fill:#e8d3ff,stroke:#8e4fbf,color:#000;
  classDef new fill:#d3ffd8,stroke:#4fbf63,color:#000;
  classDef edtool fill:#fff4c2,stroke:#d4ae2a,color:#000;
  classDef sdf fill:#d3ecff,stroke:#4f8ebf,color:#000;
```

### 表 1 · 4 模块清单

| Name | Type | LoadingPhase | Platform Deny | PublicDep (精简) | PrivateDep (精简) | 行号 |
|---|---|---|---|---|---|---|
| NubisCustom | Runtime | PostConfigInit | Linux | Core/CoreUObject/Engine/RHI/GeometryCore/GeometryFramework | RenderCore/Renderer/Niagara*/MeshConversion/DeveloperSettings | NubisCustom.Build.cs:25-52 |
| NubisCustom2 | Runtime | PostConfigInit | Linux | 同上 | 同上 (Editor 加 UnrealEd) | NubisCustom2.Build.cs:25-69 |
| NubisEditorTools | Editor | PostConfigInit | Linux | 同上 Public | **+NubisCustom +NubisCustom2 +SDFCompressEditor +HiGameEditorEntry +InteractiveToolsFramework** + UnrealEd/Slate/UMG/SceneOutliner/LevelEditor/Blutility/AssetTools/EditorSubsystem/SourceControl 等 | NubisEditorTools.Build.cs:27-94 |
| SDFCompressEditor | Editor | PostConfigInit | Linux | Core/CoreUObject/Engine/RHI/RenderCore | Projects/Slate/SlateCore/InputCore + Editor: UnrealEd/AssetTools/ToolMenus/ContentBrowser/ImageWrapper | SDFCompressEditor.Build.cs:27-73 |

**关键观察**

- NubisCustom2 **不依赖** NubisCustom — 新路径完全独立,老路径只是被编译保留 (NubisCustom2.Build.cs:25-52 里没有 `"NubisCustom"`)。
- NubisEditorTools 是**唯一的汇聚点**,同时依赖老 + 新 + SDF 三者,承担"迁移老 Actor 到新 Actor"、"烘焙、Sector 切块、BC1 压缩、刷新 Clipmap"等整套工作流。
- SDFCompressEditor **完全独立**,不依赖 NubisCustom / NubisCustom2,只依赖 Core/Engine/RHI 以及 Editor 基础模块,输出是一张标准 `PF_DXT1` `UVolumeTexture` (SDFCompressEditorModule.cpp:9-12 "Editor-only module. Owns the BC1 scalar encoder… no Runtime module")。
- `LoadingPhase=PostConfigInit` [推测]: UE 文档里这是 `Default` 之前的一个早期阶段,在 ini 配置已加载、但 Default 普通 Runtime 模块尚未加载前,用来允许插件注册全局 Shader 源码目录 (`AddShaderSourceDirectoryMapping`) ——见 NubisCustom.cpp:16-17 虽被注释掉但保留了 shader 路径提示,NubisEditorTools.cpp:25-27 也有同款注释。
- `PlatformDenyList=Linux` 全平台排除: uplugin 内**没有任何注释说明原因**。[推测] 原因: (1) HiGameServer=Linux 是 DS,无渲染需求,体积云整条管线毫无意义; (2) Editor 本身 HiGame 项目只跑 Win64 (见根 `CLAUDE.md`); (3) 插件大量依赖 Niagara / `UHeterogeneousVolumeComponent` / BC1 编码 / Slate 这些 Linux DS 上或者没有意义、或者维护成本很高的 API。

## 2 老路径 UCLASS/USTRUCT 全景

### 表 2 · NubisCustom 模块的类层次

| 类名 | 父类 | 文件:行号 | 角色 | 是否"还活着" |
|---|---|---|---|---|
| `FNubisCustomModule` | `IModuleInterface` | NubisCustom.h:10 | 壳,StartupModule 里只有注释掉的 AddShaderSourceDirectoryMapping | 活 (DEFINE_LOG_CATEGORY(LogNubis)) |
| `UNubisCustomSubsystem` | `UTickableWorldSubsystem` | NubisCustomSubsystem.h:14 | `Tick` 里用 `TActorRange` 扫全世界的 `ANubisHiCloudActor`/`ANubisZoneActor`,`#if WITH_EDITOR` 里对每个调 `BP_CustomUpdate()` | **只在 Editor 活**,运行时 Tick 空转 (cpp:40-56) |
| `ANubisHiCloudActor` | `AActor` | NubisHiCloudActor.h:10 | 3 个 `BlueprintImplementableEvent`: `BP_GetVolumeName` / `BP_LoadAndAttachMesh` / `BP_CustomUpdate`;`#include "Components/HeterogeneousVolumeComponent.h"` 但 cpp 里没有实际 New/Attach 任何 Component | **近乎纯蓝图壳**,C++ 侧不挂 HeterogeneousVolumeComponent |
| `ANubisZoneActor` | `AActor` | NubisZoneActor.h:10 | 只有 `BP_CustomUpdate` 一个 BIE | **纯蓝图壳** |
| `UNubisGenSDFComponent` | `UPrimitiveComponent` | NubisGenSDFComponent.h:19 | **老路径里唯一有实质内容的类**。1738 行 cpp,注册了 10+ 个 `FGlobalShader` (FInitializeCS/FSplatTriangleDistances*CS/FFinalizeCS/FLinearFloodStepCS/FJumpFloodInitializeCS/FJumpFloodStepCS/FJumpFloodFinalizeCS/FGenNVDB2CS 等),导出 4 个 `CallInEditor` 函数: `GenerateSDF` / `GenerateSDF_CPU` / `GenerateNVDB` / `SaveSDF` / `DebugDrawNoiseType` | **工具链仍在使用**: /Plugin/NubisCustom/Private/ 下 18 个 .usf (GenSDFInitialize/Finalize/LinearFloodStep/JumpFloodStep/GenNVDB 等) 被这里的 Compute Shader 引用 |
| `FNubisCacheData` | `USTRUCT` | NubisDataAsset.h:9 | 老 DataAsset 的单条记录:GlobalScale/Rotation/Location + `USparseVolumeTexture* VDBFromHDA` + ActorUniqueID + 20 个 Profile/Density/DetailType 参数。**注意**:SDFRange 参数在 UNubisGenSDFComponent 里,这里没有 | 还在编译,但被 raw#7 的 `FNubisCloudSnapshot` + `UNubisVDBDataAsset` 替代 |
| `UNubisDataAsset` | `UPrimaryDataAsset` | NubisDataAsset.h:68 | 字段 `TMap<FString, FNubisCacheData> CacheMap`、`int CacheVoxelSize=8`、`FVector VolumeSizeMeter=(4096,1024,4096)` + `SaveCache` / `LoadCache` / `HasValidateCache` 3 个 BlueprintCallable | 还在编译,被新路径 `UNubisVDBDataAsset` 替代 |
| `FloodMode` | `UENUM(BlueprintType)` | NubisGenSDFComponent.h:12 | `Linear=0 / Jump=1`,给 UNubisGenSDFComponent 的 `mFloodMode` 选 GPU jump-flood 还是 linear-flood | 活 |

### 老路径的实际数据流

从 NubisHiCloudActor.cpp:1-42 与 NubisCustomSubsystem.cpp:32-56 可以断定:

1. **ANubisHiCloudActor/ANubisZoneActor 不走 INubisVolumeInterface**。cpp 里没有任何 `CreateSceneProxy` 覆写、没有 `RegisterComponent` 挂 `UHeterogeneousVolumeComponent`、没有 `GetSceneProxy`、没有实现接口。虽然头文件 include 了 `Components/HeterogeneousVolumeComponent.h`,但 `ANubisHiCloudActor::ANubisHiCloudActor()` 构造函数里只设置 `PrimaryActorTick.bCanEverTick = true`。
2. **老路径唯一的"渲染"逻辑留在蓝图里**: `BP_LoadAndAttachMesh`、`BP_GetVolumeName` 都是 `BlueprintImplementableEvent`,Actor 的蓝图子类 (e.g. `BP_NubisHiCloudActor`,需在内容目录确认) 才会真正 SpawnComponent/挂 `UHeterogeneousVolumeComponent`/赋材质。C++ 侧完全没有 ghost 过 INubisVolumeInterface。
3. **SceneProxy 路径**: 当蓝图挂上 `UHeterogeneousVolumeComponent` 后,由引擎端 UHeterogeneousVolumeComponent 自己实现 `CreateSceneProxy`,再走 **raw#1 已描述的** `INubisVolumeInterface`(不是从这里老 Actor 来的)。
4. **GenSDF 组件是工具用途**:`UNubisGenSDFComponent` 是独立的 UComponent,在编辑器里挂到任何 StaticMeshComponent Actor 上就能把 Mesh 烧成 SDF VolumeTexture (`GenerateSDF` / `GenerateSDF_CPU`),或者生成 NVDB (`GenerateNVDB`) 并写入 `UTextureRenderTargetVolume` / `UVolumeTexture`。但它**不直接参与运行时 Nubis 渲染管线**,是烘焙前中间工具。
5. `UNubisCustomSubsystem::Tick` 在 `#if WITH_EDITOR` 里对所有 `ANubisHiCloud/ZoneActor` 调 `BP_CustomUpdate` (Subsystem.cpp:40-56);而 Actor 自己的 `Tick` 里也在 `#if WITH_EDITOR` 里调同一个 `BP_CustomUpdate` (HiCloudActor.cpp:34-37)。也就是**编辑器里会被调两次**(一次由 Subsystem 扫描,一次由 Actor Tick),Shipping 里完全不跑,仅为蓝图提供 hook。

**结论**:老路径在 C++ 层只是一个"Actor 壳 + 烘焙工具组件" 。所有实际渲染效果都靠蓝图/美术资产完成,无任何 ghost-real、无任何 Sector/Clipmap 感知。新路径 `NubisCustom2/ANubisHiCloud2Actor + ANubisZone2Actor + UNubisClipmapSubsystem + UNubisVDBDataAsset + UNubisClipmapDataAsset` (raw#7) 是完整的重写。

## 3 NubisEditorTools 模块

### 模块入口 `FNubisEditorToolsModule` (NubisEditorTools.cpp:22-115)

StartupModule 要做三件事:
1. `FNubisManagerCommands::Register()` + `FNubisToolsCommands::Register()` 注册 2 组 UICommand (ToolsCommands.cpp:7-10, ManagerCommands.cpp:16-19)。
2. `FGlobalTabmanager::Get()->RegisterNomadTabSpawner("NubisToolsPanel", …)`,Tab 里塞一个 `SNew(SNubisToolsPanel)` (NubisEditorTools.cpp:36-40, 90-96)。
3. `FCoreDelegates::OnAllModuleLoadingPhasesComplete` 回调里:
   - 通过 `FHiGameEditorEntryModule::AddMenuCommand` 把两个按钮注入 HiGame 自己的菜单 (NubisEditorTools.cpp:69-83)。
   - `FLevelEditorModule::AddCustomFilterToOutliner` 注册 `FOutlineFilter_NubisMeshActor` 到 HiGame 分类 (NubisEditorTools.cpp:85-87)。

其中 `NubisManagerButtonClicked` 会 `LoadObject<UWidgetBlueprint>(…/WBP_NubisEditorUtility)` 然后调用 `GEditor->GetEditorSubsystem<UEditorUtilitySubsystem>()->SpawnAndRegisterTab` 起一个 EUW (NubisEditorTools.cpp:104-113),即所谓"旧 Nubis Manager"入口;`NubisToolsButtonClicked` 则起 Slate 原生 `SNubisToolsPanel`。两者共存,EUW 老工具仍保留在 `/Game/EditorOnly/NubisTools/WBP_NubisEditorUtility`。

### `FOutlineFilter_NubisMeshActor` (NubisActorFilter.cpp:22-37)

Scene Outliner 过滤器,红色图标,过滤 **4 种** Actor 同时通过:`ANubisHiCloudActor` / `ANubisZoneActor` / `ANubisHiCloud2Actor` / `ANubisZone2Actor`。实现极简,`PassesFilter` 里就是 4 个 `IsA<...>()` 的 OR。

### `UNubisZoneEditorSubsystem : UEditorSubsystem` (NubisZoneEditorSubsystem.cpp:9-85)

- Initialize 里 `FTSTicker::GetCoreTicker().AddTicker(...)` 注册每帧 Tick。
- Tick 里 `GEditor->GetEditorWorldContext().World()`,`TActorIterator<ANubisZone2Actor>` 找所有新路径 Zone,调它们的 `EditorTick(DeltaTime)`。
- **仅伺候新路径 ANubisZone2Actor**,老路径 `ANubisZoneActor` 不参与 (证据: `FindAllNubisZoneActors` 里 Iterator 的模板参数固定为 `ANubisZone2Actor*` ——NubisZoneEditorSubsystem.cpp:66-83)。

### `SNubisToolsPanel` (SNubisToolsPanel.h:135,.cpp 5254 行)

Slate Widget,构造函数在 Construct 里一次性布局;头文件的面板顶层注释 (SNubisToolsPanel.h:83-134) 已经把整个烘焙管线的两阶段状态机画出来了,见 raw#8。

### 表 3 · SNubisToolsPanel 按钮清单 → 回调 → 用途

按出现顺序 (SNubisToolsPanel.cpp:164-405):

| 按钮文案 | 回调 | 作用 |
|---|---|---|
| ⚠️ 迁移旧Actor到新Actor | `OnMigrateOldActorToNew` (cpp:505-…) | `TActorIterator<ANubisHiCloudActor>` 扫老 Actor,对每个 SpawnActor<ANubisHiCloud2Actor>,同名同 Transform,复制匹配字段,删除旧,用 `GEditor->BeginTransaction` 包事务 |
| 查找或创建NubisZone | `OnAddOrCreateNubisZone` | 确保当前 Level 里有且仅有一个 `ANubisZone2Actor` |
| 显示NubisZone边界框 | `OnToggleDebugBoundingBox` | 切 ANubisZone2Actor 的 DebugBoundingBox |
| 冻结Clipmap原点 | `OnToggleFreezeClipmap` | 固定 Clipmap 中心,用于调试 |
| 添加HiCloud | `OnAddHiCloud` | SpawnActor<ANubisHiCloud2Actor> |
| 选中新创建的HiCloud | `OnSelectLastHiCloud` | 恢复上一步的选择状态 |
| 显示/隐藏 HiCloudMesh | `OnShowHiCloudMesh` / `OnHideHiCloudMesh` | 切 Actor 内部 StaticMesh 可见性 |
| 显示/隐藏 Nubis云 | `OnShowNubisCloud` / `OnHideNubisCloud` | 切 HeterogeneousVolumeComponent 可见性 |
| 启用/禁用/切换 HiCloud碰撞 | `OnEnableHiCloudCollision` / `OnDisableHiCloudCollision` / `OnToggleHiCloudCollision` | 3 个按钮操作碰撞 |
| **烘焙HiCloud** | `OnBakeHiCloud` | 驱动 BakeQueue + Omniverse 蓝图串行烘焙。见 raw#8 阶段一 |
| 停止烘焙 | `OnStopBakeClicked` | 保存已有产物并退出状态机 |
| 生成Mipmap（Modeling Average / SDF Min） | `OnGenerateMipmaps` | 对已存在的 Mip Modeling/SDF VolumeTexture 重新生成 mipmap 链 |
| 🔄 刷新 Clipmap | `OnRefreshClipmap` | 对 `UNubisClipmapSubsystem` 做 Unregister+Register,强制 ClipmapManager 重建 Sector 软引用 |

面板底部还有一个 **"📋 输出日志"** SListView + 清空按钮,所有按钮回调都会 `AddLog()` 到这里。

### 表 4 · `FNubisToolsCommands` / `FNubisManagerCommands` 注册

| UICommand | Context | DisplayName | 注入位置 |
|---|---|---|---|
| `OpenNubisToolsPanel` | "NubisTools" | "Nubis Tools" | HiGameEditorEntry 菜单 (NubisEditorTools.cpp:77-82) |
| `OpenNubisManager` | "NubisManager" | "NubisManager" | HiGameEditorEntry 菜单 (NubisEditorTools.cpp:69-74),点击后 load EUW `WBP_NubisEditorUtility` |

### `UBP_NubisToolLibrary` BlueprintCallable 清单

BP_NubisToolLibrary.h 里共 **25+** 个 `UFUNCTION(BlueprintCallable)`,按功能分组:

| 分类 | 函数 | 用途 |
|---|---|---|
| NubisHoudini | `CalcMeshWorldBounds` | 遍历 LOD0 顶点在世界空间算精确 Bounds |
| NubisHoudini | `ForceReconstructActor` | `InActor->RerunConstructionScripts()` |
| SourceControl | `PrepareAssetForSave` / `PrepareAssetByPath` / `PrepareAssetByAbsFilePath` | P4 Checkout / MarkForAdd / MakeWritable 兜底 |
| SourceControl | `IsAssetCheckedOut` / `IsAssetCheckedOutByOther` / `AddAssetToChangelist` / `MakeAssetWritable` | 细粒度状态查询 |
| Editor UI | `ShowEditorNotification` | FNotificationManager 通知 |
| Editor UI | `GetOrCreateVDBCacheConfig` | 加载/创建 `UNubisDataAsset` 配置 **(传入的是老 UNubisDataAsset)** |
| Editor UI | `GetHiCloudVDBPath` | 给**老** `ANubisHiCloudActor` 算 VDB 存路径 (DisplayName+GUID);`GetHiCloud2VDBPath` 已被注释掉 |
| NubisTools | `CreateVolumeTextureFromTextureArray` | 把一堆 UTexture2D 合成一张 UVolumeTexture |
| NubisTools | `GenerateNubisModelingData` | 用 MergeVDB 的 CS 把多个 SVT 合成 Modeling+SDF VolumeTexture |
| NubisTools | `GenerateNubisModelingDataFromMesh` | 从 mesh 版本 (GenSDFSplatTriangleDistances.usf) |
| NubisTools | `GetCurrentWorldSaveDirectory` | 当前关卡所在内容目录 |
| NubisTools | `GenerateNubisClipmap` | 从 Actor+DataAsset 生成 Clipmap 数据 |
| NubisTools | `SetPropertiesForOmniverse` / `SetPropertySetFloatValue` / `GetPropertySetFromActor` / `GetPropertySetFromActorWithName` / `GetOmniverseBridgeFromActor` | 与 Omniverse+Houdini 蓝图 Actor 对话设参 (raw#8 阶段一入口) |
| CloudFixer | `GenerateClipmapFromVDB` | 从 `UNubisVDBDataAsset` 生成 `UNubisClipmapDataAsset` |
| NubisTools\|SDFCompress | `SaveRenderTargetSubRegionToBC1VolumeTexture` | **唯一真正暴露给 BP 的 BC1 压缩入口**,内部调 SDFCompressEditor 的 `CompressR16VolumeToBC1` (见 raw#8) |
| NubisTools\|ThicknessMip | `ComputeRecommendedMipForVDB` / `ComputeAndCacheSectorMipForDataAsset` | 按 SDF 最小值推导 VDB 该烘到哪一档 Sector Mip,写回 `FNubisCloudSnapshot::SectorMipLevel` |

## 4 SDFCompressEditor 简介

依赖面 (SDFCompressEditor.Build.cs:27-73):

- Public: Core / CoreUObject / Engine / RHI / **RenderCore**
- Private: Projects / Slate / SlateCore / InputCore
- Editor-only: UnrealEd / AssetTools / EditorSubsystem / EditorFramework / EditorStyle / ToolMenus / ContentBrowser / ContentBrowserData / **ImageWrapper**
- **不依赖** NubisCustom / NubisCustom2,因此压缩算法完全脱离 Nubis 业务,只关心 16bit 标量 → BC1 编码。

### 入口文件一览 (Public/)

| 文件 | 内容要点 |
|---|---|
| `SDFCompressEditorModule.h` | 普通 `IModuleInterface`,StartupModule/ShutdownModule 空实现 (cpp:7-17 注释 "no Runtime module…output is a standard UVolumeTexture with pixel format PF_DXT1") |
| `BC1ScalarEncoder.h` | 命名空间 `Nubis::SDFCompress::BC1Scalar`,对外 `SDFCOMPRESSEDITOR_API` 3 个函数:`GetDefaultUnpackDot()` / `BuildBlock(16 floats, UnpackDot) → uint64` / `BuildTexture2D` / `BuildTexture3D`。Horizon Forbidden West 最优 dot 常量 = `(0.96414679, 0.03518212, 0.00067109)`,强序 R>G>B (见 raw#8) |
| `BC1ScalarSDFCompressor.h` | 命名空间 `Nubis::SDFCompress`,对外:`FSDFCompressOptions` (SourceMin/Max/bAutoDetectRange/UnpackDot/bParallel/bClampOutOfRange)、`FSDFCompressResult`、`CompressR16TextureToBC1(UTexture2D*)` / `CompressR16VolumeToBC1(UVolumeTexture*)`、底层 `CreateBC1ScalarTexture2D` / `CreateBC1ScalarVolumeTexture` (Block Array → UTexture) |

**暴露形态**:

- **不注册任何 Slate UI / ToolMenu / Factory** (StartupModule 空实现),也没有 UObject / BlueprintCallable ——整套 API 是纯 C++ namespace,只能从依赖它的其它 Editor 模块(当前即 NubisEditorTools)里调用。
- 唯一暴露给 BP 的入口是 `UBP_NubisToolLibrary::SaveRenderTargetSubRegionToBC1VolumeTexture` (BP_NubisToolLibrary.h:269-282),它拼好参数后调 `CompressR16VolumeToBC1`。

## 5 从老到新的迁移证据

- NubisEditorTools 的 "迁移旧 Actor 到新 Actor" 按钮 (SNubisToolsPanel.cpp:505-625) 会显式 spawn `ANubisHiCloud2Actor::StaticClass()` 并拷贝匹配属性,这是**从工具层把老资产转成新资产的官方路径**。
- `UNubisZoneEditorSubsystem::FindAllNubisZoneActors` 只找 `ANubisZone2Actor`;老 `ANubisZoneActor` 没有对应 Editor tick。
- 但 `FOutlineFilter_NubisMeshActor` 仍同时认 4 种 Actor —— 说明项目内**还有未迁移的老 Actor 存在**,Outliner 过滤器需要照顾到。
- `UBP_NubisToolLibrary::GetHiCloud2VDBPath` 被整段注释掉 (BP_NubisToolLibrary.cpp:1048-1060),只留老的 `GetHiCloudVDBPath` —— [推测] 说明"VDB 路径统一放到 HiCloud2 的 FNubisCloudSnapshot.VDBPath 字段里由 Snapshot 自己管",老函数还保留给可能仍在调它的蓝图向后兼容。
- `UBP_NubisToolLibrary::AutoMigrateAllMatchingProperties` 同样被整段注释掉 (BP_NubisToolLibrary.cpp:2791-…),但其逻辑已经内联进 `SNubisToolsPanel::OnMigrateOldActorToNew`。

## 6 开放问题

1. **老 `ANubisHiCloudActor` 今天还在场景里吗?**  
   `FOutlineFilter_NubisMeshActor` 同时过滤 4 个类、迁移按钮仍在,提示至少还有历史场景未迁。需要在 P4 里搜 `Content/**/*.umap` / `External*`,确认实际数量。无法靠本次考古得出数字。

2. **`UNubisGenSDFComponent` 被新路径复用吗?**  
   `NubisEditorTools.Build.cs` 把 `NubisCustom` 写进 PrivateDep,但 grep 老模块内 `UNubisGenSDFComponent` 的外部引用只在**老模块自己的 cpp** 里。`UBP_NubisToolLibrary` 里 `GenerateNubisModelingDataFromMesh` 是重新注册的 FGlobalShader (MergeMeshModelDatas.usf) + 自己的 CS,不走 UNubisGenSDFComponent。
   → [推测] `UNubisGenSDFComponent` 仍保留给美术在蓝图里单独用 (e.g. "给任意 StaticMesh 生成一张 SDF VolumeTexture"),但**不属于 Nubis 运行时渲染栈的任何一部分**。

3. **PlatformDenyList=Linux 是否误伤 Editor Linux?**  
   HiGame 不官方支持 Linux Editor,但 UE5 原生支持;若将来接 Linux Editor,这 4 个模块会全部缺失。
   [推测] 维护成本与项目优先级决定了这个 Deny,短期不会变。

4. **LoadingPhase=PostConfigInit 必要吗?**  
   当前 StartupModule 里的 `AddShaderSourceDirectoryMapping` 全部被注释掉 (两处:NubisCustom.cpp:16-17 与 NubisEditorTools.cpp:25-27),看起来不再需要这么早。真正的 shader 路径映射 [推测] 被移到了更早的地方(例如 HiGame 主模块或 NubisCustom2),或者引擎通过 `Plugins` → `Shaders/Private` 目录名约定自动挂载。保留 `PostConfigInit` 只是最初编写时的惯性,理论上降到 `Default` 不会影响当前逻辑。

5. **SDFCompressEditor 的 `Tests/` 目录**  
   `SDFCompressEditor/Private/Tests` 存在,但本笔记未深入;raw#8 聚焦算法正确性,如未来要扩展 BC1 压缩算法可以从这里入手。

---

**Cross-Reference**: raw#1 (Engine 端 NubisVolumes 入口) · raw#7 (NubisCustom2 新路径全景) · raw#8 (烘焙流水线与 BC1 压缩算法细节)。本笔记只补足 **老路径插件外壳 + Editor Tools 面板/BP 库 + SDFCompress 模块依赖关系**,不再重复其它笔记已有内容。
