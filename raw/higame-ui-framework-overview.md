# HiGame UI Lua 框架架构 — 入口/UIManager/uiframework vs lguiframework

- **Type**: Local code archaeology (synthesis from project code)
- **Fetched**: 2026-05-10
- **Project**: higame-ui-script
- **Relevant to**: Q1, Q2, Q4, Q6
- **Sources drawn from** (P4 path 在公司机器上 perforce workspace):
  1. `Content/Script/ui/uiframework/ui_manager.lua`
  2. `Content/Script/ui/uiframework/ui_window_base.lua`
  3. `Content/Script/ui/uiframework/ui_widget_base.lua`
  4. `Content/Script/ui/uiframework/ui_window_stack_manager.lua`
  5. `Content/Script/ui/uiframework/ui_window_container.lua`
  6. `Content/Script/ui/uiframework/ui_define.lua`
  7. `Content/Script/ui/uiframework/ui_depth_manager.lua`
  8. `Content/Script/ui/lguiframework/lgui_window_base.lua`
  9. `Content/Script/ui/lgui_manager.lua`
  10. `Content/Script/ui/ui_logic_subsystem.lua`
  11. `Content/Script/ui/ui_manager_launcher.lua`
  12. `Content/Script/ui/main_ui.lua`(已废弃确认)
  13. `Content/Script/ui/interactive_ui.lua`(已废弃确认)
  14. `Content/Script/CODEBUDDY.md`

---

## 1. 目录架构(三层 + UI 子树)

```
Content/Script/
├── CommonScript/        # 客户端 + 服务端共用 (Class, G, decorator, 基类)
├── ClientScript/        # 客户端专用 (UI 表现, 音频, 引导, 加载)
│   └── ui/              # 业务 UI 入口 (有些 widget 落在这里, 但占少数)
├── ServerScript/        # 服务端专用 (AI, 多世界, 天气)
└── ui/                  # ★ UI 框架与大部分 Widget 实际目录
    ├── uiframework/     # 2D UMG 核心: 基类 + MVVM + UIManager
    │   ├── mvvm/        # ViewModelBase/Field/Binder/Collection/WidgetProxy
    │   ├── ui_event/    # WaitAnimation/WaitCloseOthers/WaitController/EventContainer
    │   ├── cursor/      # 光标
    │   ├── ui_manager.lua
    │   ├── ui_window_base.lua
    │   ├── ui_widget_base.lua
    │   ├── ui_widget_listitem_base.lua
    │   ├── ui_define.lua
    │   ├── ui_window_stack_manager.lua
    │   ├── ui_window_container.lua
    │   ├── ui_depth_manager.lua
    │   ├── ui_notifier.lua
    │   ├── input_define.lua
    │   ├── ui_audio_state_manager.lua
    │   └── ...
    ├── lguiframework/   # 3D LGUI 核心 (Actor 形式的世界 UI)
    │   └── lgui_window_base.lua
    ├── widget/          # 53+ 业务子目录 (Mail/Dungeon/Office/Login/...)
    ├── viewmodel/       # 全局 ViewModel 注册与实现 (vm_define.lua)
    ├── components/      # 3D 世界 UI 组件 (mineral_hp_widget 等)
    ├── ui_common/
    ├── ui_logic_subsystem.lua    # UnLua 入口 (绑定 BP_UILogicSubsystem)
    ├── ui_manager_launcher.lua   # 启动序列
    ├── lgui_manager.lua          # 3D UI 加载器
    ├── ui_node_path_util.lua     # 编辑器调试工具
    ├── main_ui.lua               # ⚠️ 已废弃 (文件首行有作者注释)
    └── interactive_ui.lua        # ⚠️ 已废弃 (文件首行: 脚本及 WBP 无引用)
```

> **路径陷阱**: 业务 UI 大多数在 `Content/Script/ui/...`(无 ClientScript 前缀),少数在 `Content/Script/ClientScript/ui/...`。新写 UI 推荐落 `Content/Script/ui/widget/<模块>/`。

### 运行时环境判断

```lua
UnLua.IsSSInstanceClient()                                  -- 客户端
UE.UHiRuntimeEnvFunctionLibrary.IsSSInstanceGame()          -- DS 服务端
```

UI 层只在客户端跑。`UILogicSubSystem` 是 `UWorldSubsystem` 但仅客户端实例化。

---

## 2. UI 启动链路(Initialization Pipeline)

```
C++: 客户端启动 World
  └─→ UUILogicSubSystem::Initialize()      (Source/HiGame/Public/UI/UILogicSubSystem.h)
        └─→ BP_UILogicSubsystem 蓝图 (implements IUnLuaInterface)
              └─→ Lua: ui/ui_logic_subsystem.lua
                    └─→ UIManagerLauncher.DoInitUI()
                          ├─ UIDef:InitUIDef()              # 加载 UIInfo DataTable
                          ├─ UIDef:InitLGUIDef()            # 加载 LGUIInfo DataTable
                          ├─ ViewModelCollection:InitGlobalVM()  # 实例化所有全局 VM
                          └─ UIManager:InitManager()
```

> `UILogicSubSystem` C++ 类只是骨架,所有逻辑通过 `BlueprintImplementableEvent`(`InitializeScript` / `PostInitializeScript` / `DeinitializeScript` / `OnWorldBeginPlayScript`) 下沉到 Lua。

---

## 3. UIManager 单例数据结构与核心 API

文件: `ui/uiframework/ui_manager.lua`

**数据**:
- `tbCreatingUI{}` / `tbCreatedUI{}` / `tbDestroyingUI{}` — 三态实例字典
- `uiStackManager` — 8 层 UI 栈
- `uiDepthMgr` — ZOrder 管理
- `LGUIManager` — 3D 世界 UI(延迟加载)
- `UINotifier` — UI 事件总线
- `CursorManager` — 光标

**核心 API**:
```lua
UIManager:OpenUI(UIInfo, Callback, Sync, ...)       -- 推荐: 通过 UIInfo 引用打开
UIManager:OpenUIByName(UIName, Callback, Sync, ...) -- 通过名字 (动态场景)
UIManager:CloseUI(UIInstance, bImmediate)
UIManager:CloseUIByName(UIName)
UIManager:ReturnUI(Instance)              -- ESC 返回
UIManager:GetUIInstance(UIName)
UIManager:GetTopUI() / UIManager:GetTopLGUI()
UIManager:RegisterActionDelegate(...)     -- 注册输入 Action
```

---

## 4. uiframework(2D UMG)vs lguiframework(3D LGUI)

**关键发现**: 两者**不是新旧替代,而是两种渲染模式**,**共用同一个 UIManager**,通过 `UIInfo.IsLGUI` 标志区分流程。

| 维度 | uiframework (2D UMG) | lguiframework (3D LGUI) |
|------|----------------------|------------------------|
| 基类 | `UIWindowBase` ← `UIWidgetBase` | `LGUIWindowBase` ← `ComponentBase` |
| 渲染 | UMG Widget, AddToViewport | 3D 场景 Actor prefab |
| 隐藏方式 | `SetVisibility(Collapsed)` | `SetActorScale3D(0,0,0)` + `SetActorHiddenInGame(true)` |
| 配置表 | `UIDef.UIInfo["UI_xxx"]` | `UIDef.LGUIInfo["LGUI_xxx"]` |
| 生命周期方法名 | OnCreate/UpdateParams/StartShow/OnShow/.../OnDestroy | 完全相同(在 LGUIWindowBase 上独立实现) |

---

## 5. UIWindowContainer 与 8 层 UI 栈

`ui_window_container` 维护 UI 实例的状态机: `Loading → Loaded → Destroy`,持有 `UIInfo, UIInstance, Promise, Args`。

`ui_window_stack_manager` 维护 8 层独立栈,优先级从高到低:

| Layer 枚举 | 用途 |
|-----------|------|
| `TopLayer`     | 水印等永久 UI |
| `AlertLayer`   | 弹窗 / 模态确认 |
| `LoadingLayer` | Loading 界面 |
| `TipsLayer`    | 飘字 / 获得提示 |
| `GuideLayer`   | 新手引导遮罩 |
| `SystemLayer`  | 全屏系统界面 (背包、角色面板) |
| `SceneLayer`   | 常驻非全屏 (任务追踪、HUD 信息) |
| `HUDLayer`     | 主界面 HUD |

**自动隐藏规则**: 全屏界面(`UIInfo.FullScreen=true`)打开时,自动隐藏下层所有已打开界面;关闭时恢复到下一个全屏界面为止。

**模态机制**: `UIInfo.IsModal=true` 时在 `IngameLayerManager` 中插入 `UIMask` 阻断下层输入。

---

## 6. 已废弃文件与现存入口

| 文件 | 状态 | 证据 |
|------|------|------|
| `ui/main_ui.lua`         | ⚠️ 已废弃 | 文件首行注释: "已废弃,在已废弃的 ui_manager 中有调用 by shiniingliu" |
| `ui/interactive_ui.lua`  | ⚠️ 已废弃 | 文件首行: "已废弃,脚本以及绑定的 WBP 无引用 by shiniingliu" |
| `ui/ui_logic_subsystem.lua` | ✅ 仍在使用 | 作为 UI 系统入口,通过 `UIManagerLauncher.DoInitUI` 启动整个 UI 框架 |

---

## 7. 关键文件索引

### 框架核心
- `ui/uiframework/ui_manager.lua`
- `ui/uiframework/ui_window_base.lua`
- `ui/uiframework/ui_widget_base.lua`
- `ui/uiframework/ui_widget_listitem_base.lua`
- `ui/uiframework/ui_define.lua`
- `ui/uiframework/ui_window_stack_manager.lua`
- `ui/uiframework/ui_window_container.lua`
- `ui/uiframework/ui_depth_manager.lua`
- `ui/uiframework/ui_notifier.lua`
- `ui/uiframework/input_define.lua`
- `ui/uiframework/ui_audio_state_manager.lua`

### 3D LGUI
- `ui/lguiframework/lgui_window_base.lua`
- `ui/lgui_manager.lua`

### 入口
- `ui/ui_logic_subsystem.lua`
- `ui/ui_manager_launcher.lua`

### ViewModel
- `ui/viewmodel/vm_define.lua`(项目层)
- `Content/CP0032305_GH/Script/viewmodel/vm_define.lua`(GH 框架层)

### UE DataTable(资源)
- `/Game/CP0032305_GH/Blueprints/DT/UIInfo`
- `/Game/CP0032305_GH/Blueprints/DT/LGUIInfo`
