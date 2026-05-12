---
title: "NubisCloud 引擎魔改入口 + 跨模块 API + GT↔RT 边界"
type: Local code archaeology
fetched: 2026-05-11
project: higame-nubis-cloud
relevant_to: A1, A2
sources_drawn_from:
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisVolumes.h:1-159
  - Engine/Source/Runtime/Engine/Public/NubisVolumeInterface.h:1-438
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisVolumes.cpp:1-1418
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisRenderTargetViewStateData.h:1-422
  - Engine/Source/Runtime/Renderer/Private/NubisVolumes/NubisVolumesLiveShadingPipeline.cpp:1-60,2540-2726
  - Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp:3160-3179,3648-3653
  - Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.h:1059-1065
  - Engine/Source/Runtime/Renderer/Private/SceneRendering.h:1253-1255,1491-1494
  - Engine/Source/Runtime/Renderer/Private/SceneVisibility.cpp:2736-2744
  - Engine/Source/Runtime/Renderer/Private/SceneVisibilityPrivate.h:758-759
  - Engine/Source/Runtime/Engine/Private/Components/NubisVolumeComponent.cpp:1-730
  - Engine/Source/Runtime/Engine/Classes/Components/NubisVolumeComponent.h:1-172
  - Engine/Source/Runtime/Engine/Public/PrimitiveSceneProxy.h:996-1003,1566-1569
  - Engine/Source/Runtime/Core/Public/Misc/Build.h:1152-1154
  - Projects/HiGame/Plugins/NubisCustom/Source/NubisCustom2/Public/NubisZone2Actor.h:1-80
---

## 1. HIGAME_ENABLE_NUBIS 宏与编译开关

**事实**: 宏在 `Engine/Source/Runtime/Core/Public/Misc/Build.h:1152-1153` 中定义,硬编码为 `1`,无条件编译:

```cpp
// Build.h:1152-1153
#ifndef HIGAME_ENABLE_NUBIS
#define HIGAME_ENABLE_NUBIS 1
#endif
```

**推论**: 该宏不受任何 `.Build.cs` / `.Target.cs` 控制,无法通过工程配置关闭。所有 Target 默认启用。如需裁剪,须手动修改 Build.h 或在 `-D` 编译参数中覆盖。

## 2. 引擎魔改文件完整清单

### 2.1 Renderer 模块 (4 文件 + 散布改动)

| 文件 | 角色 |
|------|------|
| `Renderer/Private/NubisVolumes/NubisVolumes.h` (159行) | External/Internal API 声明 |
| `Renderer/Private/NubisVolumes/NubisVolumes.cpp` (1809行) | CVar、Should*/DoesPlatform*、RenderNubisVolumes 调度循环、CompositeNubisVolumes |
| `Renderer/Private/NubisVolumes/NubisVolumesLiveShadingPipeline.cpp` (2727行) | RenderNubisClipmapLevel、Near/Far/Octahedral 管线 |
| `Renderer/Private/NubisVolumes/NubisRenderTargetViewStateData.h` (422行) | Per-View/Per-Level 双缓冲 RT 状态、Bayer Dither 逻辑 |

散布改动 (引擎已有文件中 `#if HIGAME_ENABLE_NUBIS` 插入):
- `DeferredShadingRenderer.h:1059-1065` — 声明 RenderNubisVolumes / CompositeNubisVolumes
- `DeferredShadingRenderer.cpp:3160-3179` — 渲染插入点 (VolumetricFog 之后、VolumetricCloud 之前)
- `DeferredShadingRenderer.cpp:3648-3653` — Composite 插入点 (Translucent 之后)
- `SceneRendering.h:1253-1255` — FViewInfo 增加 NubisVolumesMeshBatches
- `SceneRendering.h:1491-1494` — FViewInfo 增加 NubisVolumeRadiance / NubisVolumeDepth
- `SceneVisibility.cpp:2736-2744` — MeshBatch 收集: IsNubisVolume → NubisVolumesMeshBatches
- `SceneVisibilityPrivate.h:758-759` — 可见性线程局部 NubisVolumesMeshBatches

### 2.2 Engine 模块 (Public Interface + Component)

| 文件 | 角色 |
|------|------|
| `Engine/Public/NubisVolumeInterface.h` (438行) | NubisDefaults 常量、FNubisClipmapLevelRenderConfig、INubisVolumeInterface 纯虚接口、FNubisVolumeData 默认实现 |
| `Engine/Private/NubisVolumeInterface.cpp` (13行) | 空文件 (纯虚接口无默认实现) |
| `Engine/Classes/Components/NubisVolumeComponent.h` (172行) | UHeterogeneousUBSVolumeComponent / ANubisVolume Actor |
| `Engine/Private/Components/NubisVolumeComponent.cpp` (~730行) | FHeterogeneousUBSVolumeSceneProxy、GT→RT 同步逻辑 |

散布改动:
- `PrimitiveSceneProxy.h:996-1003,1566-1569` — bIsNubisVolume 标志位 + IsNubisVolume() 访问器
- `PrimitiveSceneProxy.cpp:583` — 构造初始化
- `MaterialShared.h` (3处) / `Material.h` / `MaterialCompiler.h` / `HLSLMaterialTranslator.h/.cpp` — bIsUsedWithNubisVolumes 管线支持
- `MaterialExpressionNubisAdvancedMaterialOutput.h` / `MaterialExpressionNubisAdvancedMaterialInput.h` — Nubis 专用材质节点

### 2.3 Shader (15 文件,不是预估的 14)

Engine/Shaders/Private/NubisVolumes/ 目录下:

| .usf (Compute/Pixel Shader 入口) | .ush (Include 工具) |
|---|---|
| NubisVolumesLiveShadingPipeline.usf | NubisVolumesLiveShadingUtils.ush |
| NubisVolumesFarDitherScattering.usf | NubisVolumesLightingUtils.ush |
| NubisVolumesNearScattering.usf | NubisVolumesTracingUtils.ush |
| NubisVolumesBilateralUpscale.usf | NubisVolumesRayMarchingFar.ush |
| NubisVolumesReconstruct.usf | NubisVolumesRayMarchingNear.ush |
| NubisVolumesSceneComposite.usf | NubisVolumesRayMarchingUtils.ush |
| NubisVolumesVisualize.usf | NubisVolumesTransmittanceVolumeUtils.ush |
| NubisVolumesLightingCacheReseed.usf | |

### 2.4 Plugin 模块 (NubisCustom2)

| 文件 | 角色 |
|------|------|
| `Plugins/NubisCustom/Source/NubisCustom2/Public/NubisZone2Actor.h` | ANubisZone2Actor — GT 侧 Clipmap 管理 Actor |
| `Plugins/NubisCustom/Source/NubisCustom2/Public/NubisClipmap.h` | NubisClipmapManager — GT 侧 Clipmap 加载/滚动调度 |
| `Plugins/NubisCustom/Source/NubisCustom2/Public/NubisClipmapSubsystem.h` | World Subsystem 管理 |

## 3. 文件间调用关系

```
DeferredShadingRenderer.cpp
  |-- [#if HIGAME_ENABLE_NUBIS]
  |   |-- InitNubisRenderTargetForViews()           // NubisVolumes.cpp
  |   |-- RenderNubisVolumes()                       // NubisVolumes.cpp:1049
  |   |     |-- for each View.NubisVolumesMeshBatches
  |   |     |     |-- INubisVolumeInterface* (from MeshBatch.UserData)
  |   |     |     |-- for Level 4→0: RenderNubisClipmapLevel(bLightingCacheOnly=true)
  |   |     |     |-- for Level 0→5: RenderNubisClipmapLevel(bLightingCacheOnly=false)
  |   |     |           |-- Near:  RenderNearCloudWithLiveShading()
  |   |     |           |-- Far:   RenderTemporalSingleScatteringWithLiveShading()
  |   |     |           |-- Octa:  [TODO] RenderOctahedralScattering()
  |   |-- CompositeNubisVolumes()                    // NubisVolumes.cpp:1421
```

## 4. GT → RT 边界: 数据同步机制

### 4.1 ASCII 流程图

```
 Game Thread (GT)                            Render Thread (RT)
 ================                            ===================
 ANubisZone2Actor::Tick()
   |
   v
 NubisClipmapManager::Update()
   |-- 计算新 ClipmapCenter (Sector 对齐)
   |-- 计算 ScrollUVOffset / DirtyRegions
   |-- 填充 FNubisClipmapLevelRenderConfig[]
   |
   |-- Component->SyncClipmapScrollToProxy_RenderThread()
   |     |
   |     v ......ENQUEUE_RENDER_COMMAND......> SceneProxy->UpdateClipmapScrollUVOffsets_RenderThread()
   |                                              写入 NubisVolumeData.ClipmapConfigs[]
   |                                              (同帧同步,与 CopyTexture 时序一致)
   |
   |-- Component->SetMipSelectorVolume()
         |
         v ......ENQUEUE_RENDER_COMMAND......> SceneProxy->SetMipSelectorTextureRHI_RenderThread()

 (首次 / 配置变更时)
 NubisClipmapManager::InitClipmapLevels()
   |-- Component->SetClipmapData() → MarkRenderStateDirty()
   |     下一帧触发 CreateSceneProxy()
   |       构造函数中:
   |         拷贝 ClipmapConfigs / ClipmapMaterialProxies
   |         多个 ENQUEUE_RENDER_COMMAND:
   |           → RT 写 NubisTransmittanceCacheRenderTarget
   |           → RT 写 PerLevelLightingCacheRTs[]
   |           → RT 写 MipSelectorTextureRHI
   |
   |                                           SceneVisibility (RT)
   |                                             收集 NubisVolumesMeshBatches
   |                                               GetDynamicMeshElements()
   |                                                 BatchElement.UserData = &NubisVolumeData
   |
   |                                           RenderNubisVolumes (RT)
   |                                             NubisVolume = (INubisVolumeInterface*)UserData
   |                                             读 ClipmapConfigs / MaterialProxies / RT
```

### 4.2 关键线程归属

| 接口方法 | 线程 | 证据 |
|----------|------|------|
| `INubisVolumeInterface::GetClipmapLevelConfig()` | RT | 通过 MeshBatch.UserData 在渲染帧访问, NubisVolumes.cpp:1219 |
| `INubisVolumeInterface::GetClipmapMaterialProxy()` | RT | 同上, NubisVolumes.cpp:1221 |
| `INubisVolumeInterface::GetPerLevelLightingCacheRT()` | RT | NubisVolumes.cpp:1189 |
| `INubisVolumeInterface::GetMipSelectorTextureRHI()` | RT | [推测] 仅在 PassParameters 绑定时读取 |
| `UHBSVolumeComponent::SetClipmapData()` | GT | 调用 MarkRenderStateDirty(), NubisVolumeComponent.cpp:670 |
| `UHBSVolumeComponent::SyncClipmapScrollToProxy_RenderThread()` | GT→RT | ENQUEUE_RENDER_COMMAND, NubisVolumeComponent.cpp:687 |

### 4.3 FNubisClipmapLevelRenderConfig 生命周期

1. **填充 (GT)**: `NubisClipmapManager::InitClipmapLevels()` 构建完整 Config 数组,写入 Component->ClipmapConfigs。每帧 `Update()` 更新 scroll 相关字段。
2. **过渡 (GT→RT)**: 两条路径:
   - **首次/配置变更**: `MarkRenderStateDirty()` → 下一帧 `CreateSceneProxy()` 构造函数拷贝全量 Config → ENQUEUE 同步 RT 资源引用
   - **每帧 scroll**: `SyncClipmapScrollToProxy_RenderThread()` → ENQUEUE 同帧增量更新 ScrollUVOffset / WorldBoundsOrigin / DirtyRegions (NubisVolumeComponent.cpp:82-96)
3. **消费 (RT)**: `RenderNubisVolumes()` 通过 `NubisVolume->GetClipmapLevelConfig(Level)` (NubisVolumes.cpp:1219) 逐级读取,驱动两趟调度循环。

## 5. 两趟调度循环详解

`RenderNubisVolumes` (NubisVolumes.cpp:1203-1360) 内执行:

**第一趟 — LightingCache 生成 (远→近: Level 4→0)**:
- 从最远 Level 开始,确保父级 LightingCache 先于子级生成
- 每级先执行 `AddReseedLightingCachePasses()` 从父级回填 Dirty 区域
- 然后调用 `RenderNubisClipmapLevel(bLightingCacheOnly=true)` 仅写 LightingCache UAV
- 忽略 DebugMipMask — 全级烘焙保证级联采样数据完整

**第二趟 — Scattering 渲染 (近→远: Level 0→5)**:
- 受 `CVarNubisDebugMipMask` 位掩码控制
- 根据 Config.bUseDither / bUseOctahedral 路由到不同管线
- Near 管线: CompositeBlendMode=0 (Level 0 覆盖) / =1 (后续 blend-under)
- Far Dither 管线: 使用 Per-Level DitherState 独立帧计数和 Bayer Pattern

## 6. RenderNubisClipmapLevel 管线路由

```cpp
// NubisVolumesLiveShadingPipeline.cpp:2649-2722
if (Config.bUseOctahedral)     → [TODO] Octahedral 管线
else if (Config.bUseDither)    → RenderTemporalSingleScatteringWithLiveShading()
else                           → RenderNearCloudWithLiveShading()
```

Far 管线在 `NubisVolumesLiveShadingPipeline.cpp:2658-2698` 初始化 Per-Level Dither 状态:
- 从 `View.ViewState->NubisVolumeRenderTarget` 获取 `FNubisPerLevelDitherState`
- UpdateResolution → Initialise (Bayer 查表、双缓冲交换) → 渲染

## 7. FNubisVolumeData: RT 侧的数据载体

`FNubisVolumeData` (NubisVolumeInterface.h:306-436) 继承 `INubisVolumeInterface` + `FOneFrameResource`:
- 持有 SceneProxy 指针、ClipmapConfigs 数组、ClipmapMaterialProxies 数组
- 持有 PerLevelLightingCacheRTs[] 和 MipSelectorTextureRHI
- 在 `GetDynamicMeshElements` 中通过 `BatchElement.UserData = &NubisVolumeData` 传递给渲染帧
- RT 侧 `RenderNubisVolumes` 从 `Mesh->Elements[0].UserData` 强转获取

## 8. 渲染集成点在 Deferred Pipeline 中的位置

```
DeferredShadingRenderer::Render()
  ...
  ComputeVolumetricFog()
  RenderHeterogeneousVolumes()
  ─── RenderNubisVolumes() ───        // DeferredShadingRenderer.cpp:3160
  FlushSetupQueue()
  RenderVolumetricCloud()
  ...
  RenderTranslucency()
  CompositeHeterogeneousVolumes()
  ─── CompositeNubisVolumes() ───     // DeferredShadingRenderer.cpp:3648
  AddResolveSceneColorPass()
```

**推论**: Nubis 渲染在 VolumetricFog 之后、VolumetricCloud 之前。Composite 在半透明之后,与 HeterogeneousVolumes 对齐。

## 9. 平台支持判定

```cpp
// NubisVolumes.cpp:359-365
bool DoesPlatformSupportNubisVolumes(EShaderPlatform Platform)
{
    return IsFeatureLevelSupported(Platform, ERHIFeatureLevel::SM5)
        && !IsForwardShadingEnabled(Platform);
}
```

要求 SM5 + Deferred Shading。Forward 渲染路径不支持。移动端 (ES3.1) 不支持。

## 10. 开放问题

1. **Shader 14 还是 15?** 任务描述提到 "Shader 14 文件",实际 glob 扫到 15 个文件 (8 usf + 7 ush)。需确认是否有遗漏或多算。
2. **引擎魔改标记**: 搜索 `HiGame Begin` + `Nubis` 组合在 Engine/Source 下无匹配。NubisVolumes 代码中未使用 `// HiGame Begin/End` 标记,而是统一以 `#if HIGAME_ENABLE_NUBIS` 包裹整个文件或代码块。需确认团队是否有其他标记规范。
3. **INubisVolumeInterface 在 Plugin 侧的实现**: `NubisCustom/Source` 目录下未直接引用 `INubisVolumeInterface`,ANubisZone2Actor 通过 `UHeterogeneousUBSVolumeComponent` (Engine 侧) 间接使用。Plugin 侧是否有独立的 SceneProxy 子类需进一步确认。
4. **Octahedral 管线状态**: `NubisVolumesLiveShadingPipeline.cpp:2651` 标注 `// Phase 3 实现:八面体映射 // TODO`,该管线尚未实装。
5. **NubisVolumeInterface.cpp 的存在意义**: 该文件仅含 `#include` 和注释 "Clipmap 接口已改为纯虚函数,不再需要默认实现",疑似历史遗留。
6. **ClipmapConfigs 的线程安全**: 每帧通过 `ENQUEUE_RENDER_COMMAND` 增量同步 scroll 字段,但 `MarkRenderStateDirty` 触发的 SceneProxy 重建是异步的 (下一帧)。是否存在重建帧与 scroll 帧的竞态窗口? [推测] MarkRenderStateDirty 仅在 InitClipmapLevels (低频) 调用,scroll 走 ENQUEUE 路径不触发重建,应无竞态。
7. **Editor 模块 DumpMaterialInfo.cpp:264** 也引用了 HIGAME_ENABLE_NUBIS,可能是材质导出工具对 Nubis 材质的特殊处理,需确认。
