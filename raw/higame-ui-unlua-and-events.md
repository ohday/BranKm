# HiGame UnLua 绑定 + UI 事件原语 + UINotifier

- **Type**: Local code archaeology
- **Fetched**: 2026-05-10
- **Project**: higame-ui-script
- **Relevant to**: Q8, Q9
- **Sources drawn from**:
  1. `Config/DefaultUnLuaSettings.ini`(LuaModuleLocator 配置)
  2. `Content/Script/CommonScript/UnLua/HotReload.lua`
  3. `Content/Script/CommonScript/UnLua/InitEnv.lua`
  4. `Content/Script/CommonScript/Class.lua`(UnLua.Class = _G.Class L381-382)
  5. `Content/Script/ui/uiframework/ui_event/ui_wait_animation.lua`
  6. `Content/Script/ui/uiframework/ui_event/ui_wait_close_others.lua`
  7. `Content/Script/ui/uiframework/ui_event/ui_wait_close_lgui.lua`
  8. `Content/Script/ui/uiframework/ui_event/ui_wait_controller.lua`
  9. `Content/Script/ui/uiframework/ui_event/ui_wait_event_container.lua`
  10. `Content/Script/ui/uiframework/ui_event/ui_event_def.lua`
  11. `Content/Script/ui/uiframework/ui_notifier.lua`

---

## 1. UnLua 蓝图 ↔ Lua 路径绑定

### 配置(DefaultUnLuaSettings.ini)
```ini
ModuleLocatorClass=/Script/CoreUObject.Class'/Script/UnLua.LuaModuleLocator'
EnvLocatorClass=...LuaEnvLocator_HiGame   ; 自定义 CS 分离 EnvLocator
```

### 路径约定
WBP 实现 `IUnLuaInterface`,在 `GetClientModuleName` 中返回 Lua 模块路径:

```
WBP 路径:    /Game/UI/Mail/WBP_Mail_ComMain.uasset
对应 Lua:    Content/Script/ui/widget/Mail/ui_mainmail.lua
GetClientModuleName 返回: "ui.widget.Mail.ui_mainmail"
```

CS 分离模式: `GetModuleName` 留空,只填 `GetClientModuleName`。

### 子 Widget 自动绑定
蓝图中 `BindWidget` 标记的 UPROPERTY,在 Lua 中通过同名属性 `self.WidgetName` 直接访问,**无需声明**:

```lua
self.HiList_Consume      -- 蓝图中名为 HiList_Consume 的 ListView
self.WBP_Common_Number   -- 嵌套子 UserWidget
self.DX_In               -- UMG Animation
self.Canvas_Thoughts     -- CanvasPanel
self.WBP_Common_Number.Slider_Number  -- 多层嵌套用点号
```

### 类声明两种写法

```lua
-- 写法 A: 继承 UIWindowBase (全功能窗口,推荐)
local UIWindowBase = require('ui.uiframework.ui_window_base')
local M = UnLua.Class(UIWindowBase)

-- 写法 B: 无父类 (简单子控件 / ListItem)
local M = UnLua.Class()
```

文件末尾必须 `return M`。

> `UnLua.Class` 与 `_G.Class` 是同一个函数(`Class.lua` L381-382),完全等价。

## 2. 生命周期方法对应 UE/UMG

| Lua 方法 | UE 入口 | 备注 |
|---|---|---|
| `Construct()` / `OnConstruct()` | `UUserWidget::NativeConstruct` | 同一 hook 的两种命名,新代码用 `Construct` |
| `Destruct()` / `OnDestruct()` | `UUserWidget::NativeDestruct` | 必须解绑事件 |
| `Tick(MyGeometry, InDeltaTime)` | 引擎每帧 | 极少使用 |
| `OnShow()` / `OnHide()` | `UIWindowBase` 框架级 | 仅限 `UIWindowBase` 子类 |

## 3. 按钮 OnClicked 标准写法

```lua
function M:Construct()
    self.Button_Click.OnClicked:Add(self, self.OnClickHandler)
end

function M:Destruct()
    self.Button_Click.OnClicked:Remove(self, self.OnClickHandler)
end

function M:OnClickHandler()
    -- 业务逻辑
end
```

**关键规则**: `Destruct` 中必须 `Remove`,否则热更/GC 会泄漏。嵌套按钮同理: `self.WBP_xxx.OnClicked:Add(self, self.OnXxx)`。

## 4. 热更新机制

文件: `Content/Script/CommonScript/UnLua/HotReload.lua`

- 编辑器内保存 `.lua` 后**自动触发**(通过 `WatchScriptDirectory` 监听 mtime)
- 原理: 检测 `loaded_module_times` 时间戳变化 → `package.loaded` 置 nil → 重新 require → 逐函数替换 + upvalue 保留
- 远程 Hotfix: 通过服务器下发 Lua chunk(`reload_patches = {ModuleName, FunctionName, PatchCode}`)精确修复单个函数
- `InitEnv.lua` 中 `_G.UnLuaHotReload = UnLuaHotReload` 暴露给 C++ 调用

---

## 5. UI 事件原语(ui_event/)

| 类 | 文件 | 用途 |
|---|---|---|
| `UIWaitAnimation`        | `ui_wait_animation.lua`   | 等待 UMG Widget 动画播放完毕。`SetWaitAnimation(anim, bPlayWhenActive, startTime, playMode, speed)` |
| `UIWaitCloseOtherUI`     | `ui_wait_close_others.lua` | 等待指定 UI 全部关闭(监听 `UIHide` 通知) |
| `UIWaitCloseOtherLGUI`   | `ui_wait_close_lgui.lua`  | 等待 LGUI 关闭,主动 `CloseUIByName` 并等回调 |
| `UIWaitController`       | `ui_wait_controller.lua`  | 等待 PlayerController 有效(UI 开启时 Controller 尚未就绪) |
| `UIWaitEventContainer`   | `ui_wait_event_container.lua` | 容器组合,提供 `WaitAllEvents` / `WaitAnyEvent` 两种模式 |

### 用例
```lua
local container = UIWaitEventContainer.new()
local waitAnim = UIWaitAnimation.new(self)
waitAnim:SetWaitAnimation(self.ShowAnimation)
container:AddWaitEvent(waitAnim)
container:WaitAllEvents(self, self.OnShowAnimFinished, true)
```

`UIWindowBase` 内部就用 `WaitStartShowEvents / WaitShowEvents / WaitHideEvents` 三个 EventContainer 串起整个生命周期等待。

---

## 6. UINotifier — UI 事件总线

文件: `Content/Script/ui/uiframework/ui_notifier.lua`

```lua
-- 注册监听
UINotifier:BindNotification(NotificationName, UIObj, fnDelegate)
-- 取消单个
UINotifier:UnbindNotification(UIObj, fnDelegate)
-- 取消 UIObj 上所有监听 (Destruct 时建议)
UINotifier:UnbindAllNotification(UIObj)
-- 广播
UINotifier:UINotify(NotificationName, ...)
```

**标准事件名**(`ui_event_def.lua`): `UICreate` / `UIShow` / `UIAfterShow` / `UIHide` / `UIDestroy`。

### 实现细节
- 每个 `NotificationName` 对应一个 `ViewmodelField`
- `BindNotification` 用 `NotifyUI` 绑定方式将 UIObj 的 NotificationField 与之关联
- `UINotify` 调用该 Field 的 `BroadcastNotification(...)` 触发所有已绑定回调

### 用例
```lua
-- 监听 UI 关闭事件
UIManager.UINotifier:BindNotification('UIHide', self,
    function(ownerProxy, closedUI)
        -- closedUI 关闭时触发
    end)
```
