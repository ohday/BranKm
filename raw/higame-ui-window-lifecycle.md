# HiGame UIWindowBase 生命周期 + UIInfo 配置

- **Type**: Local code archaeology
- **Fetched**: 2026-05-10
- **Project**: higame-ui-script
- **Relevant to**: Q3, Q4, Q5
- **Sources drawn from**:
  1. `Content/Script/ui/uiframework/ui_window_base.lua`(完整源码,WindowState 枚举+生命周期实现)
  2. `Content/Script/ui/uiframework/ui_define.lua`(UIInfo 字段加载)
  3. `Content/Script/ui/uiframework/ui_widget_base.lua`(基类)
  4. `Content/Script/ui/uiframework/ui_widget_listitem_base.lua`
  5. UE DataTable: `/Game/CP0032305_GH/Blueprints/DT/UIInfo`
  6. `Content/Script/ui/lguiframework/lgui_window_base.lua`(2D/3D 共用相同生命周期方法名)

---

## 1. WindowState 枚举(ui_window_base.lua L27-34)

```lua
local WindowState = {
    Undefined  = -1,
    Collapsed  = 0,
    Showing    = 1,
    Showed     = 2,
    Collapsing = 3,
}
```

`UIWindowBase.CurrentWindowState` 在生命周期中流转。

## 2. 完整生命周期方法表(可覆写)

源码注释明确标注 "↓ 可覆写的生命周期函数,其他内部函数非必要禁止覆写 ↓":

| 方法 | 触发时机 | 写代码用途 |
|------|---------|----------|
| `OnCreate()`         | 首次创建,在 OnConstruct/Construct 之后 | 初始化一次性数据(全生命周期不变的) |
| `UpdateParams(...)`  | 每次 OpenUI 都触发,接收透传参数 | 接收打开参数 |
| `StartShow()`        | 显示动画前 | 准备显示 |
| `OnShow()`           | 显示动画完成 | **拉数据 / 播放生效逻辑** |
| `StartHide()`        | 关闭动画前 | 通知开始关闭 |
| `OnHide()`           | 完全隐藏后 | 资源回收准备 |
| `StartBackShow()`    | 被覆盖后重新激活前 | (非首次显示触发) |
| `OnBackShow()`       | 重新激活动画完成 | 刷新数据 |
| `OnDestroy()`        | 销毁时 | 资源销毁 |
| `RefreshPanel()`     | 其他界面关闭后,可见界面收到 | 数据刷新 |

**对应 UnLua/UMG 引擎层**:
- `Construct()` / `OnConstruct()` ← UMG `NativeConstruct`(项目里两种命名都存在,新代码倾向 `Construct`)
- `Destruct()` / `OnDestruct()` ← UMG `NativeDestruct`
- `Tick(MyGeometry, InDeltaTime)` ← 引擎每帧(极少用)

## 3. 生命周期分发机制 — DoOnCreate 递归

源码 L130 显示一个关键模式: 生命周期方法**递归到 `tbSubUserWidget`**:

```lua
function UIWindowBase:DoOnCreate(widget)
    if widget.OnCreate then
        widget:OnCreate()
    end
    if widget.tbSubUserWidget ~= nil and #widget.tbSubUserWidget > 0 then
        for _, subUserWidget in pairs(widget.tbSubUserWidget) do
            if subUserWidget.DoOnCreate then
                subUserWidget:DoOnCreate(subUserWidget)
            end
        end
    end
end
```

意义: 子 UserWidget 也能拿到完整生命周期。同样 `DoStartBackShow`、`DoOnShow` 等都按这个递归模式分发。

## 4. CallOnCreate 内部状态初始化

```lua
function UIWindowBase:CallOnCreate()
    self:SetVisibility(UE.ESlateVisibility.Collapsed)
    self.CurrentWindowState = WindowState.Collapsed

    self.WaitStartShowEvents = UIEventContainer.new()
    self.WaitShowEvents      = UIEventContainer.new()
    self.WaitHideEvents      = UIEventContainer.new()
    self.FocusReceivedHandlers = {}
    self.FocusLostHandlers     = {}
    self.tbDynamicSubWidget    = {}

    self:DoOnCreate(self)
    UIManager.UINotifier:UINotify(UIEventDef.UICreate, self)
end
```

**关键陷阱**: 自定义子类**不要**覆写 `CallOnCreate`,只覆写 `OnCreate`。

## 5. BeginShow / 等待动画 / WaitStartShowComplete

`BeginShow` 在 `Collapsed` 或 `Collapsing` 时进入。流程:
1. 调 `UE.UHiInputMasterSubsystem.GetFromWidget(self):OnLayeredUIActivated(self)` 入栈
2. 非首次显示调 `DoStartBackShow(self)`
3. `WaitHideEvents:StopWait()`,如果还在 Collapsing 立即 `ClearWaitEvent`(让出与 Show 对应的 Hide)
4. 检测 `StartMiniGameGuide`(新手引导优先)
5. `ShowingUI()` → 添加 WaitController + WaitStartShowEvents → 等所有事件结束 → `WaitStartShowComplete`

## 6. UIInfo 配置完整字段(DataTable: `/Game/CP0032305_GH/Blueprints/DT/UIInfo`)

新 UI **必须**在这个 DataTable 中注册一行,运行时通过 `GH_FunctionLibrary.GetUIInfoContent` 读取并由 `ui_define.lua:InitUIDef()` 组装。

| 字段 | 类型 | 说明 |
|------|------|------|
| `UIName`           | string | UI 唯一标识(如 `"UI_Mail"`) |
| `WidgetClassPath`  | ObjectPath | WBP 蓝图路径(运行时自动拼 `_C` 后缀) |
| `UILayer`          | Enum_UILayerNew | 实际层级(SceneLayer/SystemLayer/GuideLayer/LoadingLayer/TopLayer/TipsLayer/AlertLayer/HUDLayer) |
| `UILayerIdent`     | Enum_UILayer    | 旧层级枚举(兼容) |
| `FullScreen`       | bool | 是否全屏(触发自动隐藏下层) |
| `IsModal`          | bool | 是否模态 |
| `EscClose`         | bool | ESC 是否关闭 |
| `Mouse`            | Enum_UIMouseVisibility | 鼠标行为(Show/Hide/NoEffect) |
| `IMC`              | bool | 是否使用 IMC_UI 输入映射 |
| `DefaultOpen`      | bool | 是否随游戏自动打开 |
| `ForceOpen`        | bool | 强制打开 |
| `DontDestroy`      | bool | 关闭时不销毁(下次打开复用实例) |
| `Group`            | table | UI 分组(Communication/Sequence 等) |
| `OpenAkEvent`      | string | 打开 Wwise 事件 |
| `CloseAkEvent`     | string | 关闭 Wwise 事件 |
| `AudioStateType`   | Enum_EUIAudioStateType | 音频状态类型 |

3D 界面则注册到 `LGUIInfo` DataTable。

## 7. WindowState 状态机(可视化)

状态字段 `self.CurrentWindowState`,转换由 `BeginShow / ShowingUI / OnShow / BeginHide / OnHide` 等内部方法驱动。

```
            [Undefined]
                │
                │ CallOnCreate
                ▼
         [Collapsed]
            │   ▲
   BeginShow│   │ OnHide 完成
            ▼   │
         [Showing]            ─── 动画播放中 + WaitStartShowEvents
            │
            │ WaitStartShowComplete
            ▼
         [Showed]
            │
            │ BeginHide
            ▼
         [Collapsing]         ─── 动画播放中 + WaitHideEvents
            │
            │ 完成
            ▼
         [Collapsed]
            │
            │ DoOnDestroy + bDestroyUIAfterHiding=true
            ▼
         (实例移除)
```

特殊点: 若 `Collapsing` 期间 `BeginShow`,会调 `WaitHideEvents:ClearWaitEvent`,使后续 OnShow 与对应的 OnHide 配对(防止状态错乱)。
