---
title: "HiGame NubisCloud 烘焙流水线"
type: Local code archaeology
fetched: 2026-05-11
project: higame-nubis-cloud
relevant_to: C1, C2, C3, C4, C5
sources_drawn_from:
  - Plugins/NubisCustom/Source/NubisEditorTools/Private/SNubisToolsPanel.cpp
  - Plugins/NubisCustom/Source/NubisEditorTools/Public/SNubisToolsPanel.h
  - Plugins/NubisCustom/Source/NubisEditorTools/Public/BP_NubisToolLibrary.h
  - Plugins/NubisCustom/Source/NubisCustom2/Public/NubisVDBDataAsset.h
  - Plugins/NubisCustom/Source/NubisCustom2/Public/NubisClipmapDataAsset.h
  - Plugins/NubisCustom/Source/SDFCompressEditor/Public/BC1ScalarSDFCompressor.h
  - Plugins/NubisCustom/Source/SDFCompressEditor/Private/BC1ScalarSDFCompressor.cpp
  - Plugins/NubisCustom/Source/SDFCompressEditor/Public/BC1ScalarEncoder.h
  - Plugins/NubisCustom/Source/NubisCustom/Public/NubisGenSDFComponent.h
  - Plugins/NubisCustom/Source/NubisCustom/Public/NubisDataAsset.h
  - Plugins/UnrealOmniverse/Source/TOmniversePythonActor/Public/
  - Engine/Source/Runtime/Engine/Public/NubisVolumeInterface.h
  - Engine/Source/Runtime/Engine/Classes/Engine/BC1ScalarVolumeTexture.h
---

# NubisCloud 烘焙流水线
## Houdini → OpenVDB → SDFCompress → NubisVDBDataAsset → 关卡放置 → Cook

---

## 1. 流水线 ASCII 总览

```
 [Houdini + lop_openvdb HDA]
         |
         | (通过 Omniverse Python Bridge)
         v
 [BP_NubisTOmniversePythonActor]  -- 在 UE Editor 内 spawn
         |  PythonAsset = lop_openvdb_higame (TUSDPythonAsset)
         |  SetupBluprint() -> OnOmniverseInitFinished()
         v
 [逐 HiCloud2Actor 串行烘焙]  -- SNubisToolsPanel::ProcessNextBakeTask()
         |  收集 Transform + HoudiniParams -> FNubisCloudSnapshot
         |  Omniverse 调用 Houdini 产出 SVT (OpenVDB -> UStaticSparseVolumeTexture)
         |  保存到 NubisVDBCache/Voxel{MipLevel}/{ActorLabel}_{GUID}.uasset
         v
 [UNubisVDBDataAsset]  -- NubisVDBCache_Config.uasset
         |  TMap<FGuid, FNubisCloudSnapshot> SavedClouds
         |  TMap<FSectorKey, TSoftObjectPtr<UVolumeTexture>> ExistingSectorTextures
         v
 [VDB Merge 状态机]  -- SNubisToolsPanel::StartAsyncVDBMerge()
         |  阶段: RouteSectorMip -> InitMip -> InitSector ->
         |         MergingSectorVDB -> WaitForStreaming -> SaveSector ->
         |         CleanupMip -> Finalize
         |  每 Sector 创建 256x256x64 RenderTarget, GPU Compute 合并
         v
 [Sector VolumeTexture]  -- NubisQuickCloud/Sectors/
         |  Mip{N}_Sector_{X}_{Y}_{Z}_Modeling.uasset (颜色/建模)
         |  Mip{N}_Sector_{X}_{Y}_{Z}_SDF.uasset      (SDF距离场, BC1压缩)
         v
 [UNubisClipmapDataAsset]  -- NubisQuickCloud/NubisClipmapData_{Zone}.uasset
         |  索引所有 Sector 软引用, 运行时由 NubisClipmapSubsystem 加载
         v
 [运行时 NubisClipmap]  -- 6级 Mip, 每级 2x2x2 Sector, BaseVoxel=1m
```

---

## 2. C1: Houdini Engine 与 Omniverse 桥

### lop_openvdb_higame — TUSDPythonAsset (非原生 HDA)

`lop_openvdb_higame.uasset` (16KB) 是 **TUSDPythonAsset** 类型,不是原生 HoudiniEngine HDA。
证据来自 SNubisToolsPanel.cpp:1913:

```cpp
FString PythonAssetPath = TEXT(
  "/Script/TUSDEditor.TUSDPythonAsset'/Game/EditorOnly/NubisTools/lop_openvdb_higame.lop_openvdb_higame'");
```

它通过 **UnrealOmniverse 插件** 的 `ATOmniversePythonActor` 框架运作:
- `BP_NubisTOmniversePythonActor.uasset` (402KB) 是蓝图子类
- 工具面板 Spawn 此 Actor -> 设置 PythonAsset -> 调用 `SetupBluprint()` -> 等 `OnOmniverseInitFinished` 回调
- Omniverse Python Bridge 连接到远程 Houdini Session,执行 LOP 网络
- Houdini 输出 OpenVDB -> Omniverse 传回 UE -> 落地为 `UStaticSparseVolumeTexture`

### Omniverse Python Actor 用途

`ATOmniversePythonActor` (来自 Plugins/UnrealOmniverse) 封装了:
- `UTUSDPythonAsset` 引用 (Python 脚本定义)
- `TArray<UTOmniverseInput>` 输入绑定
- `ETOmniverseHoudiniAssetController` 支持 ACTORS / ASSETS 两种控制模式
- SopFilter 可过滤 Mesh/BasisCurves/HoudiniFieldAsset/PointInstancer 类型

用途: 充当 UE Editor 与 Houdini (通过 Omniverse Nucleus) 之间的 **双向数据桥**。

---

## 3. C2: 资产格式与 SDFCompress 工具

### NubisVDBDataAsset (新格式, NubisCustom2 模块)

| 字段 | 类型 | 用途 |
|------|------|------|
| `SavedClouds` | `TMap<FGuid, FNubisCloudSnapshot>` | 每朵云的完整快照 |
| `ExistingSectorTextures` | `TMap<FSectorKey, TSoftObjectPtr<UVolumeTexture>>` | Sector 纹理软引用 |
| `NubisGUID` | `FString` | 缓存唯一标识 |

FNubisCloudSnapshot 包含:
- `CacheMipLevel` — Mesh 外接盒决定的 Voxel 目录 (烘焙精度)
- `SectorMipLevel` — SDF 厚度路由的 Mip 档 (合并阶段)
- `WorldMeshBounds`, `GlobalLocation/Scale/Rotation` — Transform
- `FNubisHoudiniParams` — 16 个 Houdini 浮点参数 (SDFOffset, DensityScale 等)
- `VDBFromHDA` — `TSoftObjectPtr<UStaticSparseVolumeTexture>` (延迟加载)

### NubisDataAsset (旧格式, NubisCustom 模块)

旧格式使用 `TMap<FString, FNubisCacheData>` 以 ActorUniqueID 字符串为 key,
VDB 引用为 `TObjectPtr<USparseVolumeTexture>` (硬引用)。新格式改为 Guid key + 软引用。

### Sector VolumeTexture 产物

每个 Sector 产出两张纹理:
1. **Modeling** — 颜色/建模数据, `TC_Default` 压缩
2. **SDF** — 距离场, 使用自定义 **BC1 标量编码器** 压缩 (4 bpp)

### SDFCompressEditor 模块

**入口函数**:
- `Nubis::SDFCompress::CompressR16VolumeToBC1()` — 高层 API
- `Nubis::SDFCompress::BC1Scalar::BuildTexture3D()` — 底层编码
- `UBP_NubisToolLibrary::SaveRenderTargetSubRegionToBC1VolumeTexture()` — 蓝图可调用

**压缩算法**: 来自 Guerrilla Games (Horizon Forbidden West) 的 BCn 技巧:
- 16-bit scalar -> BC1 (DXT1) 编码, 用 `dot(RGB, float3(0.96414679, 0.03518212, 0.00067109))` 解码
- 保证 GPU 采样值 **<= 源值** (undershoot constraint), raymarch 不穿透
- 最大误差 ~2e-5 (~1/50000)
- 支持 PF_R16F / PF_G16 / PF_FloatRGBA / PF_R32_FLOAT 源格式

**输出类型**: `UBC1ScalarVolumeTexture` (引擎级自定义子类, Override Serialize/PostLoad/UpdateResource, 阻止 UE 重压缩)

**依赖**: 纯 CPU 编码, 无外部压缩库依赖 (Build.cs 仅依赖 Core/RHI/RenderCore + Editor 模块)

---

## 4. C3: NubisVDBCache vs NubisQuickCloud

### 目录结构差异

| 目录 | 内容 | 阶段 |
|------|------|------|
| `NubisVDBCache/` | 单朵云 SVT 资产 + Config DataAsset | 阶段一: Houdini 烘焙 |
| `NubisQuickCloud/` | Sector VolumeTexture + ClipmapDataAsset | 阶段二: VDB 合并 |

NubisVDBCache 是**中间产物缓存**: 每朵 HiCloud2Actor 的 OpenVDB 数据按 MipLevel 存入 `Voxel{N}/` 子目录。
NubisQuickCloud 是**最终运行时产物**: 所有 VDB 合并后的 Clipmap Sector 切片。

### 资产量级对比

| 关卡 | VDBCache | QuickCloud | Sector 数 |
|------|----------|------------|-----------|
| LV_KTD | 1.2 GB | 148 MB | 88 |
| LV_ZC_YLD | 4.7 GB | 383 MB | 402 |
| LV_FGD_01 | 4 KB (几乎空) | 34 MB | 0 (旧格式) |
| LV_WD_FAHN | 44 KB | 1.9 MB | 0 (旧格式) |

### QuickCloud 旧格式 vs 新 Sector 格式

**旧格式** (LV_FGD_01, LV_WD_FAHN): 按 Zone 整体生成单张 VolumeTexture
- `NubisModelingDataBP_NubisZone_C_{N}.uasset`
- `NubisSDFDataBP_NubisZone_C_{N}.uasset`

**新 Sector 格式** (LV_KTD, LV_ZC_YLD): 按 Sector 切割
- `Sectors/Mip{N}_Sector_{X}_{Y}_{Z}_Modeling.uasset`
- `Sectors/Mip{N}_Sector_{X}_{Y}_{Z}_SDF.uasset`
- `NubisClipmapData_{Zone}.uasset` 索引所有 Sector

**LV_KTD 额外有按 Mip 整体的旧产物** (`NubisModelingData_NubisZone2_Mip{0-5}.uasset`) 与新 Sector 并存。

### WeatherSystem/Maps/NubisQuickCloud

仅 2 个文件 (9.4 MB), 旧格式 (`NubisModelingData...` + `NubisSDFData...`), Zone 编号 `_C_1`。
推断为**全局天气系统的默认云层**, 与关卡级 QuickCloud 独立 — 关卡云覆盖或叠加天气系统云。

---

## 5. C4: NubisGenSDFComponent — 旧版运行时/编辑器内 SDF 生成

`UNubisGenSDFComponent` 是 **Editor 内 GPU SDF 生成组件** (旁支,非主流水线):

- `GenerateSDF()` / `GenerateSDF_CPU()` — 从 StaticMesh 生成 SDF (GPU Compute / CPU fallback)
- `GenerateNVDB()` — 生成 NVDB (Nubis VDB)
- `SaveSDF()` — 将 RenderTarget 保存为 VolumeTexture

使用 GPU Compute Shader 实现:
- `FInitializeCS` -> `FSplatTriangleDistancesUnsignedCS` -> Flood Fill
- 依赖 `Implicit/SweepingMeshSDF.h` / `Spatial/MeshAABBTree3.h` (几何处理库)

**属性**: CallInEditor — 仅编辑器可调用, **非运行时**。

### SDFCompressEditor 压缩目标

SDFCompressEditor **仅压缩 SDF VolumeTexture** (R16F/G16 -> BC1):
- API 命名: `CompressR16TextureToBC1` / `CompressR16VolumeToBC1`
- 对 Modeling 纹理不适用 (Modeling 用标准 TC_Default 压缩)
- 实际调用点在 `BP_NubisToolLibrary::SaveRenderTargetSubRegionToBC1VolumeTexture()`

---

## 6. 资产清单表

### EditorOnly/NubisTools/ (工具资产)

| 资产名 | 大小 | 类型推断 |
|--------|------|----------|
| `lop_openvdb_higame.uasset` | 16 KB | TUSDPythonAsset (Houdini LOP 定义) |
| `BP_NubisTOmniversePythonActor.uasset` | 402 KB | Blueprint (Omniverse 桥 Actor) |
| `WBP_NubisEditorUtility.uasset` | 1.3 MB | Editor Utility Widget (工具面板) |

### Developers/wenxiangzuo/CustomNubisVolume/ (开发者测试, 406 MB)

| 资产/目录 | 大小 | 备注 |
|-----------|------|------|
| `ParkouringCloud.uasset` | 8.1 MB | 跑酷云 VDB |
| `DebugQuickCloud/DebugNVDBData.uasset` | 31.6 MB | 调试 NVDB 数据 |
| `DebugQuickCloud/DebugSDFData.uasset` | 30.9 MB | 调试 SDF 数据 |
| `NubisQuickCloud/NubisSDFData*.uasset` | 14-21 MB 每个 | Zone SDF (旧格式, 4 个) |
| `NubisQuickCloud/NubisModelingData*.uasset` | 1.8-6.6 MB | Zone Modeling (旧格式, 4 个) |
| `TestVDBMesh/CloudShape_Group.uasset` | 90.8 MB | VDB 网格化测试 |
| `VDBs/Altocumulus*.uasset` | 3.9 MB 每个 | 高积云 VDB 模板 |
| `VDBs/cirrostratus.uasset` | 7.9 MB | 卷层云 VDB 模板 |

### Developers/zshanzong/NubisVDBCache/ (旧版, 1.1 MB)

Voxel0-Voxel6, 共 ~130 个 `NubisHiCloud2Actor{N}_{GUID}.uasset` (~4.3 KB 每个)。
注: 文件极小 (~4KB), 推断为**软引用 / 空壳资产**, 实际 VDB 数据可能已被清理。

### Clipmap 默认参数 (NubisDefaults)

| 参数 | 值 | 说明 |
|------|-----|------|
| MipCount | 6 | Mip 0-5 |
| BaseVoxelSize | 1m | Mip0 体素 = 1m, Mip5 = 32m |
| TextureSize | 512x512x128 | 全 Clipmap 体素数 |
| SectorSize | 256x256x64 | 单 Sector 体素数 |
| SectorWidth | 2x2x2 | 每 Mip 的 Sector 网格 |

---

## 7. C5: 8 步 Cookbook — 给 LV_NEW 添加云

```
步骤 1: 打开关卡, 放置 NubisZone2Actor
  - 在 LV_NEW 关卡中放置一个 ANubisZone2Actor
  - 设置 ClipmapMipCount=6 (默认)
  - 工具面板按钮: "添加或替换 NubisZone"

步骤 2: 放置 HiCloud2Actor (云朵代理 Mesh)
  - 在 NubisZone 范围内放置 ANubisHiCloud2Actor
  - 为每个 Actor 指定 PreviewCloudStaticMesh (SM_Cloud 网格)
  - 调整 Transform (Location/Scale/Rotation) 控制云的位置和大小
  - 调整 HoudiniParams (SDFOffset, DensityScale 等) 控制云的形态

步骤 3: 启动 Omniverse + Houdini 连接
  - 确保本地 Houdini 已启动并连接到 Omniverse Nucleus
  - 工具面板自动 Spawn BP_NubisTOmniversePythonActor
  - 加载 lop_openvdb_higame TUSDPythonAsset
  - 等待 OnOmniverseInitFinished 回调

步骤 4: 一键烘焙 (工具面板 "烘焙HiCloud" 按钮)
  - SNubisToolsPanel::OnBakeHiCloud() 触发
  - 自动创建 NubisVDBCache_Config.uasset (UNubisVDBDataAsset)
  - 逐 Actor 串行: 收集 Snapshot -> Omniverse 调 Houdini -> SVT 落盘
  - 产物: Content/Maps/LV_NEW/NubisVDBCache/Voxel{N}/{ActorLabel}_{GUID}.uasset

步骤 5: VDB 合并 (自动, 烘焙完成后 2 秒触发)
  - PerformVDBMerge() -> StartAsyncVDBMerge()
  - 状态机逐 Mip(3-5) 逐 Sector: 加载 SVT -> GPU Compute 合并 -> 保存 VolumeTexture
  - SDF 纹理使用 BC1 压缩 (SaveRenderTargetSubRegionToBC1VolumeTexture)
  - 产物: Content/Maps/LV_NEW/NubisQuickCloud/Sectors/Mip{N}_Sector_{X}_{Y}_{Z}_{Type}.uasset

步骤 6: ClipmapDataAsset 生成 (合并阶段自动)
  - UNubisClipmapDataAsset 记录所有 Sector 的软引用
  - 产物: Content/Maps/LV_NEW/NubisQuickCloud/NubisClipmapData_{ZoneLabel}.uasset

步骤 7: 刷新 Clipmap (自动 + 可手动)
  - Finalize 阶段自动调用 RefreshClipmapForZone()
  - NubisClipmapSubsystem Unregister + Register, 重建运行时 Clipmap
  - 工具面板 "刷新 Clipmap" 按钮可手动触发

步骤 8: Cook 验证  [推测]
  - 标准 UE Cook 流程, NubisQuickCloud/ 下资产被 Cook
  - NubisVDBCache/ 为 EditorOnly 中间产物, 不随 Cook 打包 [推测]
  - BC1ScalarVolumeTexture 的 Serialize override 确保 BC1 blocks 原样写入 cooked asset
  - 运行时 NubisClipmapSubsystem 按需加载 Sector 纹理
```

---

## 8. 开放问题

1. **Houdini 内部黑盒**: `lop_openvdb_higame` 的 Python 脚本内容无法从 .uasset 读取。LOP 网络具体如何从 Mesh 生成 OpenVDB? 使用了哪些 Houdini 节点 (VDB From Polygons? Pyro Solver?)?

2. **Omniverse 通信协议**: UE -> Omniverse -> Houdini 的数据流是通过 USD Stage 还是 Nucleus Live Session? 延迟和稳定性如何?

3. **NubisVDBCache 是否 EditorOnly**: Cache 目录下的 SVT 资产在 Cook 时是否被排除? 代码中未见显式 EditorOnly 标记, 需确认 Cook 配置。

4. **Mip 0-2 为何不产出 Sector**: SNubisToolsPanel.h 注释 "只为 Mip3+ 创建 VolumeTexture", 但 LV_KTD 的 Sector 全部从 Mip3 起。Mip 0-2 的精度数据是否完全不使用?

5. **旧格式迁移**: LV_FGD_01 / LV_WD_FAHN 仍使用旧的 per-Zone VolumeTexture 格式, 工具面板有 "迁移旧Actor数据到新Actor" 按钮 (OnMigrateOldActorToNew), 但迁移是否已完成?

6. **BC1 SDF 压缩的 SourceMin/Max**: 合并阶段使用 bAutoDetectRange=true 还是手动指定? 运行时 Shader 如何获取 remap 参数?

7. **WeatherSystem 全局云层**: WeatherSystem/Maps/NubisQuickCloud 与关卡级 QuickCloud 如何叠加? 是通过不同 NubisZone2Actor 还是独立 Clipmap?

8. **Developers/zshanzong 资产极小 (~4KB)**: 这些 NubisHiCloud2Actor*.uasset 是空壳还是仅含软引用? 实际 VDB 数据存储在何处?

9. **VDB Mesh 化实验**: wenxiangzuo 目录下的 `TestVDBMesh/` (CloudShape_*_mesh.uasset) 暗示有 VDB -> StaticMesh 转换实验路径, 但未在主流水线中使用。目的是什么? 远景云 LOD?

10. **SubVoxelGrid 超采样**: MergeSingleVDBToRenderTarget 支持 SubVoxelGrid 参数 (典型值 = 2^MipLevel), 实际烘焙时使用的值? 对质量/性能的影响?
