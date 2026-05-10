# HiGame UI 真实代码模板(弹窗 / 全屏面板 / ListItem / VM)+ 注册流程

- **Type**: Local code archaeology
- **Fetched**: 2026-05-10
- **Project**: higame-ui-script
- **Relevant to**: Q12
- **Sources drawn from** (项目内真实代码):
  1. `Content/Script/ui/widget/Login/WBP_Login_Popup_Device.lua`(简单弹窗)
  2. `Content/Script/ui/widget/Mail/ui_mainmail.lua`(全屏 + MVVM)
  3. `Content/Script/ui/widget/Mail/ui_mailitem.lua`(ListView 条目)
  4. `Content/Script/ui/viewmodel/vm_define.lua` 注册表
  5. `ClientScript/actors/Abyss/BPA_AbyssEntry.lua`(OpenUI 业务调用)
  6. `ClientScript/ui/widget/TradingPost/WBP_TradingPost_Main.lua`(带参 OpenUI)
  7. `common/item/ItemJumpToAccess.lua`(OpenUIByName)

---

## 1. AI 写新 UI 的标准流程

### 步骤 0: 决定 2D 还是 3D
- 全屏 / 弹窗 / HUD / 列表 → **2D UMG**(`UIWindowBase`)
- 角色头顶血条 / 世界标记 / 3D 菜单 → **3D LGUI**(`LGUIWindowBase`)

### 步骤 1: 创建 WBP
1. 蓝图继承自 `UHiActivableLayeredWidget` 或 `UHiUserWidget`(具体看用途)
2. WBP 实现 `IUnLuaInterface`
3. `GetClientModuleName` 返回 `"ui.widget.<模块>.<lua文件名>"`

### 步骤 2: 在 DataTable 中注册 UIInfo
打开 `/Game/CP0032305_GH/Blueprints/DT/UIInfo`,新增一行:
- `UIName` = `"UI_MyNewPanel"`
- `WidgetClassPath` = WBP 路径
- `UILayer` = `SystemLayer`(按需)
- `FullScreen` / `IsModal` / `EscClose` 等按需

### 步骤 3: 写 Lua 类
路径: `Content/Script/ui/widget/<模块>/ui_my_new_panel.lua`

```lua
local UIWindowBase = require('ui.uiframework.ui_window_base')
local UIManager    = require('ui.uiframework.ui_manager')
local ViewModelCollection = require('ui.uiframework.mvvm.viewmodel_collection')
local ViewModelBinder     = require('ui.uiframework.mvvm.viewmodel_binder')
local WidgetProxys        = require('ui.uiframework.mvvm.ui_widget_proxy')
local VMDef = require('ui.viewmodel.vm_define')

---@type WBP_MyNewPanel_C
local M = UnLua.Class(UIWindowBase)

function M:Construct()
    self.MyVM = ViewModelCollection:FindUniqueViewModel(
                    VMDef.UniqueVMInfo.MyVM.UniqueName)

    self.ListProxy = WidgetProxys:CreateWidgetProxy(self.ListView_Items)
    ViewModelBinder:BindViewModel(
        self.ListProxy.ListField, self.MyVM.ItemListField,
        ViewModelBinder.BindWayToWidget)

    self.Btn_Close.OnClicked:Add(self, self.OnCloseClick)
end

function M:UpdateParams(...)         -- 接收 OpenUI 透传参数
end

function M:OnShow()                  -- 显示动画完成后刷新数据
    self.MyVM:RefreshData()
end

function M:OnHide() end

function M:Destruct()
    ViewModelBinder:UnBindByUI(self, true)
    self.Btn_Close.OnClicked:Remove(self, self.OnCloseClick)
end

function M:OnCloseClick()
    UIManager:CloseUI(self, true)
end

return M
```

### 步骤 4(可选): 注册 ViewModel

VM 类: `Content/Script/ui/viewmodel/my_vm.lua`
```lua
local ViewModelBaseClass = require('ui.uiframework.mvvm.viewmodel_base')
local M = Class(ViewModelBaseClass)

function M:ctor()
    Super(M).ctor(self)
    self.ItemListField  = self:CreateVMArrayField({})
    self.SomeValueField = self:CreateVMField(0)
end

function M:RefreshData()
    -- 拉数据,更新 Field
end

return M
```

VM 注册到 `Content/Script/ui/viewmodel/vm_define.lua`:
```lua
VMDef.UniqueVMInfo.MyVM = {
    UniqueName = 'MyVM',
    ViewModelClassPath = 'ui.viewmodel.my_vm',
}
```

### 步骤 5: 业务代码中打开

```lua
local UIDef     = require('ui.uiframework.ui_define')
local UIManager = require('ui.uiframework.ui_manager')

-- 推荐: 通过 UIInfo 引用打开
UIManager:OpenUI(UIDef.UIInfo.UI_MyNewPanel)

-- 带回调
UIManager:OpenUI(UIDef.UIInfo.UI_MyNewPanel, function(panel)
    panel:SomeAfterOpenAction()
end)

-- 同步打开 + 透传参数 (会被 UpdateParams 收到)
UIManager:OpenUI(UIDef.UIInfo.UI_MyNewPanel, nil, true, arg1, arg2)

-- 通过名字 (动态打开场景)
UIManager:OpenUIByName("UI_MyNewPanel", nil, nil, arg1)
```

---

## 2. 真实代码模板 A — 简单弹窗

`Content/Script/ui/widget/Login/WBP_Login_Popup_Device.lua`:

```lua
local UIWindowBase = require('ui.uiframework.ui_window_base')
local UIManager    = require('ui.uiframework.ui_manager')

local M = Class(UIWindowBase)

function M:OnConstruct()
    self.CommitCallBacks = {}
    self.WBP_Common_SquareIconButton_01:BindBtnClick(self, self.OnCopyEvent)
    self.WBP_Common_Popup_Small:SetKeyCalloutKeys(true,
        "ESC_KEYBOARD_RETURN_AND_CONFIRM", nil, KeyInfoData)
    self.WBP_Common_Popup_Small:RegisterKeySelectedCallBacks(self, self.OnKeySelected)
end

function M:StartShow()
    self.WBP_Common_Popup_Small:PlayInAnim()
end

function M:CloseMyself()
    UIManager:CloseUI(self, true)
end

return M
```

## 3. 真实代码模板 B — 全屏面板 + MVVM

`Content/Script/ui/widget/Mail/ui_mainmail.lua`:

```lua
local UIWindowBase  = require('ui.uiframework.ui_window_base')
local WidgetProxys  = require('ui.uiframework.mvvm.ui_widget_proxy')
local ViewModelBinder     = require('ui.uiframework.mvvm.viewmodel_binder')
local ViewModelCollection = require('ui.uiframework.mvvm.viewmodel_collection')

local M = UnLua.Class(UIWindowBase)

function M:Construct()
    self.MailList   = WidgetProxys:CreateWidgetProxy(self.ListView_Mail)
    self.SelectMail = self:CreateUserWidgetField(self.ClickMail)
    self.MailVM     = ViewModelCollection:FindUniqueViewModel(
                          VMDef.UniqueVMInfo.MailVM.UniqueName)

    ViewModelBinder:BindViewModel(self.SelectMail,
                                  self.MailVM.SelectInfo,
                                  ViewModelBinder.BindWayToWidget)
    ViewModelBinder:BindViewModel(self.MailList.ListField,
                                  self.MailVM.MailListField,
                                  ViewModelBinder.BindWayToWidget)

    self.WBP_ComBtn_Delete.OnClicked:Add(self, DeleteCurMail)
end

function M:OnShow()
    self.MailVM:SetOwner(self)
end

function M:Destruct()
    ViewModelBinder:UnBindByUI(self, true)
    self.WBP_ComBtn_Delete.OnClicked:Remove(self, DeleteCurMail)
end

return M
```

## 4. 真实代码模板 C — ListView 条目

`Content/Script/ui/widget/Mail/ui_mailitem.lua`:

```lua
local UIWindowBase = require('ui.uiframework.ui_window_base')
local M = UnLua.Class(UIWindowBase)

function M:OnListItemObjectSet(ListItemObject)
    self.Info  = ListItemObject.ItemValue.FieldValue
    self.ID    = ListItemObject.ItemValue.FieldValue.ID
    self.Title = ListItemObject.ItemValue.FieldValue.Title
    RefreshShow(self)
end

function M:Construct()
    self.WBP_Btn_MailList.OnClicked:Add(self, Click)
    self.WBP_Btn_MailList.OnHovered:Add(self, OnHoverItem)
end

function M:Destruct()
    self.WBP_Btn_MailList.OnClicked:Remove(self, Click)
    self.WBP_Btn_MailList.OnHovered:Remove(self, OnHoverItem)
end

return M
```

---

## 5. 真实业务代码中的 OpenUI 用法

| 调用形式 | 文件 |
|---------|------|
| `UIManager:OpenUI(UIDef.UIInfo.UI_Abyss_Shelter)` | `ClientScript/actors/Abyss/BPA_AbyssEntry.lua` |
| `UIManager:OpenUI(UIDef.UIInfo.UI_Common_SecondTextConfirm, function(SureUI) ... end)` | `ui/widget/Mail/ui_mainmail.lua` |
| `UIManager:OpenUI(UIDef.UIInfo.UI_Exchange_Filter, nil, nil, VMName)` | `ClientScript/ui/widget/TradingPost/WBP_TradingPost_Main.lua` |
| `UIManager:OpenUIByName(UI_NAME)` | `ClientScript/ui/viewmodel/ld_head_swap_vm.lua` |
| `UIManager:OpenUIByName(NavigationData.details, nil, nil, ...)` | `common/item/ItemJumpToAccess.lua` |

签名: `UIManager:OpenUI(UIInfo, CallBack, Sync, ...)` / `UIManager:OpenUIByName(UIName, CallBack, Sync, ...)`

---

## 6. 常见陷阱 / 反模式

| 陷阱 | 后果 | 正确做法 |
|---|---|---|
| 在 `Destruct` 没解绑事件/VM | 热更/GC 泄漏,旧 self 残留 | 配套 `OnClicked:Remove`、`UnBindByUI` |
| 用 `main_ui.lua` / `interactive_ui.lua` 当模板 | 已废弃,代码死 | 用 `ui/widget/Mail/ui_mainmail.lua` |
| 绕过 UIManager 直接 `AddToViewport` | 不入栈,ESC 关不掉,层级错乱 | `UIManager:OpenUI(UIDef.UIInfo.UI_xxx)` |
| 跨 UI 通过全局变量传数据 | 时序问题,VM 不刷新 | 共享 `ViewModelCollection:FindUniqueViewModel` |
| 在 `Construct` 拉网络数据 | UI 还没显示就触发,可能时序错 | 在 `OnShow` 拉,或交给 VM 自己拉 |
| 监听 KeyCode 而非 Action | 设备切换/重映射不工作 | 用 `RegisterActionDelegate` + `InputDef.DefaultUIAction` |
| ListView 数据全量刷新 | 性能差,UI 闪烁 | 用 `AddItem` / `RemoveItemIf` 增量 OpCode |
| 写 Tick 做 UI 动画 | 帧率耦合,GC 抖动 | 用 UMG `WidgetAnimation` + `UIWaitAnimation` |
| 在 DS(服务端)写 UI 代码 | UI 只在客户端实例化,代码不会执行 | UI 一律走 ClientScript / `ui/` 目录 |
| WBP 没 implements UnLua 接口 | Lua 类不会绑定,所有方法不触发 | WBP 实现 `IUnLuaInterface` 并填 `GetClientModuleName` |
| 多个 UI 共用一个 UIName | UIManager 状态冲突 | UIName 必须唯一 |
