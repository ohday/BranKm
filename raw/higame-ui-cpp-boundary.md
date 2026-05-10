# HiGame UI C++ 类层次 + Lua/C++ 边界 + BlueprintCallable 接口清单

- **Type**: Local code archaeology
- **Fetched**: 2026-05-10
- **Project**: higame-ui-script
- **Relevant to**: Q11
- **Sources drawn from**:
  1. `Source/HiGame/Public/UI/UILogicSubSystem.h`
  2. `Source/HiGame/Public/UI/IngameLayerManagerWidget.h`
  3. `Source/HiGame/Public/UI/HiCursorWidget.h`
  4. `Source/HiGame/Public/UI/HiUserWidget.h`
  5. `Source/HiGame/Public/UI/HiActivableUserWidget.h`
  6. `Source/HiGame/Public/UI/HiActivableLayeredWidget.h`
  7. `Source/HiGame/Public/UI/UIBlueprintFunctionLibrary.h`
  8. `Source/HiGame/Public/UI/HiWidgetLayoutLibrary.h`
  9. `Source/HiGame/Public/UI/HiRenderTargetEraseLibrary.h`
  10. `Source/HiGame/Public/UI/UIExtensions/HiUIExtensionSubsystem.h`
  11. `Source/HiGame/Public/UI/UIExtensions/HiExtensionSlotWidget.h`
  12. `Source/HiGame/Public/UI/UIExtensions/HiActivatableWidgetInterface.h`
  13. `Source/HiGame/Public/UI/Cursor/HiUI_AssistMaster_Cursor_Base.h`
  14. `Source/HiGame/Public/UI/Gamepad/HiUI_GamepadMenu_Base.h`
  15. `Source/HiGame/Public/UI/MainHUD/HiUI_HUD_Track.h`
  16. `Source/HiGame/Public/UI/MainHUD/HiUI_MainHUD_QuickSlot.h`
  17. `Source/HiGame/Public/UI/HiListViewUserWidget.h`

---

## 1. C++ 类层次(从 UE 基类向下)

```
UWorldSubsystem
  └── UUILogicSubSystem (Abstract, Blueprintable)        ← Lua 继承入口

ULocalPlayerSubsystem
  ├── UHiUIExtensionSubsystem                            ← 拓展点管理
  └── UHiInputMasterSubsystem                            ← 输入总线(自研)

UUserWidget
  ├── UIngameLayerManagerWidget                           ← 根层容器(非分层)
  ├── UHiCursorWidget                                    ← 系统光标壳
  └── UHiUserWidget (Abstract)                           ← 所有业务 Widget 基类
        ├── UHiActivableUserWidget (Abstract)            ← 可激活 Widget
        │     ├── UHiActivableLayeredWidget              ← 层级面板(顶层 UI)
        │     └── UHiUI_GamepadMenu_Base                 ← 手柄菜单
        ├── UHiUI_AssistMaster_Cursor_Base               ← 角色光标(动画状态机)
        ├── UHiUI_HUD_Track                              ← 3D 追踪 HUD
        ├── UHiUI_MainHUD_QuickSlot                      ← 主 HUD 快捷栏
        └── UHiListViewUserWidget / EntryWidget          ← ListView 基础封装

UOverlay
  └── UHiExtensionSlotWidget                             ← 拓展槽(容器)

UBlueprintFunctionLibrary
  ├── UUIBlueprintFunctionLibrary                        ← 通用工具
  ├── UHiWidgetLayoutLibrary                             ← 布局工具
  └── UHiRenderTargetEraseLibrary                        ← RT 涂抹/覆盖率
```

## 2. 各模块职责

| 模块 | 文件 | 说明 |
|------|------|------|
| **UILogicSubSystem** | `UILogicSubSystem.h/cpp` | 仅客户端实例化的 WorldSubsystem; C++ 仅提供生命周期骨架,全部逻辑通过 `BlueprintImplementableEvent` 下沉到 Lua(`InitializeScript`/`PostInitializeScript`/`DeinitializeScript`/`OnWorldBeginPlayScript`) |
| **IngameLayerManager** | `IngameLayerManagerWidget.h/cpp` | 顶层 Widget 容器,**并非多 Layer 分层**;核心职责: 注册 Slate `InputPreProcessor` 拦截鼠标事件,广播输入方式/文化/前后台/全屏变化委托给 Lua;`OpenUIByName` 静态方法转调 `BP_OpenUIImpl` 交由 Lua 实现 |
| **UIExtensions** | `HiUIExtensionSubsystem.h`, `HiExtensionSlotWidget.h` | 自研拓展点(非 CommonUI UIExtensionSystem);DataTable 配置各 ExtensionSlot 的 WidgetClass(按平台 Desktop/Mobile/PS 区分);Subsystem 管理全局激活/反激活;`UHiExtensionSlotWidget` 是放在 UMG 蓝图中的 Overlay,按需异步加载子 Widget |
| **UINavigation** | `UINavigationConfig.h`, `HiInputMasterSubsystem.h` | **完全自研**;`UHiInputMasterSubsystem`(LocalPlayerSubsystem)统一管理键鼠/手柄/触屏输入,维护 UI 激活栈 `ActivatedUIArray`,判定 Top UI 并分发按键/摇杆;自定义 `FNavigationConfig` 覆盖 UE 原生方向导航;Handler 链: KeyAction / UINavigation / Block / KeyIcon / Cooldown |
| **Cursor** | `HiCursorWidget.h`, `HiUI_AssistMaster_Cursor_Base.h` | 双层设计: `UHiCursorWidget` 是系统级壳(硬件光标替换);`UHiUI_AssistMaster_Cursor_Base` 是角色化光标,带状态机(Moving/Click/ShortIdle/LongIdle)、眼球追踪、Hover Context 系统、动画 Config Map |
| **Gamepad** | `HiUI_GamepadMenu_Base.h` | 手柄专用菜单基类,通过 `UHiGamepadManager` 接收轴/按键;屏蔽默认 InputTypeChanged |
| **MainHUD** | `HiUI_HUD_Track.h` 等 | HUD 追踪(3D 屏幕坐标映射 + 椭圆边界夹持)、快捷栏(长按检测)等,均继承 `UHiUserWidget`;复杂计算在 C++,表现回调用 `BlueprintImplementableEvent` |

## 3. C++ 与 Lua 边界

### 必须 C++ (理由)
- **Slate `InputPreProcessor` 注册分发** — Lua 无法触达 `FSlateApplication` 层
- **自定义 `FNavigationConfig` 覆盖** — 引擎框架级 Hook
- **`UHiInputMasterSubsystem` 输入总线** — 每帧 Tick 高频轴值累积、Focus 追踪
- **`UHiUI_AssistMaster_Cursor_Base` 眼球插值/动画序列队列** — 每帧 Tick 性能敏感
- **RT 降采样链(`HiRenderTargetEraseLibrary`)** — 涉及 GPU Readback 和 RHI 线程安全
- **HUD 3D 追踪坐标椭圆夹持算法(`HiUI_HUD_Track`)**

### 应写 Lua
- `UILogicSubSystem` 的 Initialize/Deinitialize/OnWorldBeginPlay 具体逻辑
- `IngameLayerManagerWidget.BP_OpenUIImpl` — UI 打开路由
- `HiActivableLayeredWidget` 的 `BP_OnOpen` / `BP_HandleClose` / `BP_Setup` / `BP_Cleanup`
- 所有 `OnActivated` / `OnDeactivated` 表现层响应
- ListView 条目的 `RefreshWidgets` / `OnClicked`
- 输入类型切换后的 UI 适配(`BP_OnInputTypeChanged`)
- 光标状态变化响应(`OnCursorStateChanged`)

## 4. 高频 BlueprintCallable 接口(Lua 直接 `self:Xxx()` 或 `UE.UXxx.Yyy()` 调)

### `UHiUserWidget`(基类方法,所有 Lua Widget 可用)

| 方法 | 作用 |
|---|---|
| `K2_GetPlayerController/State/Character` | 获取游戏对象 |
| `K2_SetImageFromResourcePicKey` / `K2_SetImageFromSoftObject` | 异步图片加载 |
| `K2_SetWidgetVisible` / `K2_SetWidgetVisibility` | 可见性 |
| `K2_SetWidgetText` / `K2_SetWidgetTextFromStringTable` | 文本设置 |
| `PlayAnimationAndWaitForFinished` | Latent 动画播放 |
| `PlayAnimationLoop` | 循环动画 |
| `SetTimerOnce` | 定时回调 |
| `K2_GetWidgetAnimationByName` | 按名获取动画 |
| `K2_GetWidgetFromName` | 按名获取子控件 |
| `K2_GetOwnerLayeredWidget` | 获取所属层级 Widget |
| `AsyncLoadObject` | 异步加载资源 |

### `UHiActivableUserWidget`

| 方法 | 作用 |
|---|---|
| `K2_RequestActivateExtension` / `K2_RequestDeactivateExtension` | 拓展点开关 |
| `K2_DoBackAction` | 返回操作 |
| `K2_PlayActivateAnimation` / `K2_PlayDeactivateAnimation` | |

### `UUIBlueprintFunctionLibrary`(静态)

| 方法 | 作用 |
|---|---|
| `GetWidgetScreenPosition` | Widget 屏幕坐标 |
| `SetFocus` / `HasFocus` | 焦点控制 |
| `GetDefaultInputDeviceType` | 当前输入设备 |
| `TextLengthMeasure` | 文本像素长度 |
| `DrawUIToRenderTarget` | Widget → RT |
| `GlobalUpdateTypefaceToFarh` / `GlobalResetTypefaceFromFarh` | 法恩语切换 |

### `UHiWidgetLayoutLibrary`

| 方法 | 作用 |
|---|---|
| `DuplicateWidget` | 克隆 Widget |
| `GetWidgetFromName` | 按名查子控件 |
| `GetAllWidgets` | 获取所有子控件 |
| `GetEntryWidgetFromItem` | ListView Item → Entry |

### `UHiInputMasterSubsystem`

| 方法 | 作用 |
|---|---|
| `SetInputMode` / `GetCurrentInputMode` | 输入模式切换 |
| `SetMouseCursorVisible` | 光标显隐 |
| `OnLayeredUIActivated` / `OnLayeredUIDeactivated` | 注册/注销层级 UI |
| `IsWidgetReachable` | 导航可达检测 |
| `SimulateKeyToKeysActions` | 模拟按键 |

### `UHiRenderTargetEraseLibrary`
- `DrawMIDToRT` + `ComputeAlphaRatioByDownsample` + `InitDownsampleResources` — 涂抹进度

## 5. UnLua 暴露模式总结

C++ 通过两种 UFUNCTION 宏暴露给 Lua:

1. **BlueprintImplementableEvent** — Lua 端覆写实现(如 `BP_OnOpen`、`OnCursorStateChanged`)
2. **BlueprintCallable** — Lua 端直接调用 C++ 能力(如 `K2_SetWidgetText`)

`UUILogicSubSystem` 标记为 `Abstract + Blueprintable`,意味着项目中存在一个 BP 子类绑定了 UnLua 脚本,由 Lua 实现整个 UI 管理器逻辑。Widget 层面,Lua 继承 `UHiActivableLayeredWidget` 的 BP 子类实现具体面板。
