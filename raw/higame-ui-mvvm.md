# HiGame MVVM(ViewModel + Field + Binder + WidgetProxy + Collection)

- **Type**: Local code archaeology
- **Fetched**: 2026-05-10
- **Project**: higame-ui-script
- **Relevant to**: Q7
- **Sources drawn from**:
  1. `Content/Script/ui/uiframework/mvvm/viewmodel_base.lua`
  2. `Content/Script/ui/uiframework/mvvm/viewmodel_field.lua`
  3. `Content/Script/ui/uiframework/mvvm/viewmodel_binder.lua` (BindWay 常量定义在 L155-158)
  4. `Content/Script/ui/uiframework/mvvm/viewmodel_collection.lua`
  5. `Content/Script/ui/uiframework/mvvm/ui_widget_proxy.lua`
  6. `Content/Script/ui/uiframework/ui_widget_base.lua`(`CreateUserWidgetField` 定义在 L146)
  7. `Content/Script/ui/viewmodel/vm_define.lua`(注册表)
  8. 真实样例: `Content/Script/ui/widget/Mail/ui_mainmail.lua`

---

## 1. ViewModelBase 核心方法

```lua
-- 创建单值字段(可绑定)
self.PlayerName = self:CreateVMField(InFieldValue)        -- 任意初始值

-- 创建数组字段(驱动 ListView/TileView)
self.ItemListField = self:CreateVMArrayField(InFieldValue) -- 通常是 {}
```

返回值分别为 `ViewmodelField` / `ViewmodelFieldArray`。

## 2. ViewmodelField 读写与广播

```lua
local val = vmField:GetFieldValue()

-- 写入 (值变化时自动广播给所有已绑定的 WidgetField)
vmField:SetFieldValue(newVal)
-- 内部逻辑:
--   if newVal ~= oldVal then
--       self.FieldValue = newVal
--       self:BroadcastValueChanged()
--   end
```

支持自定义读写: `SetOverrideGetter(fn)` / `SetOverrideSetter(fn)`。

## 3. Widget 侧创建可绑定 Field

**方式 A**: 自定义 Setter/Getter — `ui_widget_base.lua` L146
```lua
function UIWidgetBase:CreateUserWidgetField(fnSetter, fnGetter)
-- 返回 UIWidgetField, UsedByUserWidget = self
```

用例:
```lua
self.SelectMail = self:CreateUserWidgetField(self.OnSelectMailChanged)
```

**方式 B**: WidgetProxy 自动适配标准控件
```lua
local WidgetProxys = require('ui.uiframework.mvvm.ui_widget_proxy')
local proxy = WidgetProxys:CreateWidgetProxy(self.TextBlock_Name)
-- proxy.TextField    UTextBlock
-- proxy.ListField    UListView
-- proxy.CheckedField UCheckBox
-- proxy.ValueField   USlider
```

## 4. BindViewModel 三种 BindWay

`viewmodel_binder.lua` L155-158:

```lua
ViewModelBinder:BindViewModel(widgetField, vmField, BindWay)
```

| 常量 | 值 | 数据流向 | 语义 |
|---|---|---|---|
| `BindWayToWidget` | 1 | VM → Widget | 立即用 VM 值初始化 Widget;VM 变化自动调 Setter 推送 |
| `BindWayToVM`     | 2 | Widget → VM | 立即用 Widget 值初始化 VM;Slider/CheckBox 交互时回写 VM |
| `NotifyUI`        | 3 | VM → UI 回调 | 不传值,触发 `Field_Notifier` 回调,纯事件 |

**解绑接口** (Destruct 时必须调):
```lua
ViewModelBinder:UnBindByUI(self, true)
ViewModelBinder:UnBindByViewModel(vm)
ViewModelBinder:UnBindByWidgetField(field)
```

## 5. ListField 驱动 ListView 标准做法

```lua
-- VM 侧
self.MailListField = self:CreateVMArrayField({})
function VM:Add(item) self.MailListField:AddItem(item) end
function VM:Remove(id)
    self.MailListField:RemoveItemIf(function(f)
        return f:GetFieldValue().ID == id
    end)
end

-- Widget 侧
self.MailList = WidgetProxys:CreateWidgetProxy(self.ListView_Mail)
ViewModelBinder:BindViewModel(
    self.MailList.ListField,
    self.MailVM.MailListField,
    ViewModelBinder.BindWayToWidget)
```

`UListViewProxy:SetListItems` 内部为每条数据创建 `UObject(ItemObjClass)`,设置 `obj.ItemValue = data`,然后调用 `BP_SetListItems`。每个条目 Widget 在 `OnListItemObjectSet(ListItemObject)` 中接收数据。

支持 OpCode `'AddItem'` / `'RemoveItem'` 的**增量操作**(不要每次全量 SetItems,会卡)。

## 6. ViewModelCollection — 全局 VM 注册中心

VM 在 `ui/viewmodel/vm_define.lua` 集中声明,启动时 `ViewModelCollection:InitGlobalVM()` 全部实例化:

```lua
-- vm_define.lua
VMDef.UniqueVMInfo.MyNewVM = {
    UniqueName = 'MyNewVM',
    ViewModelClassPath = 'ui.viewmodel.my_new_vm',
}
```

Widget 通过单例引用获取(从而**实现跨 UI 通信**):

```lua
self.MyVM = ViewModelCollection:FindUniqueViewModel(VMDef.UniqueVMInfo.MyNewVM.UniqueName)
```

## 7. 完整数据流(Mail 真实案例)

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

function M:Destruct()
    ViewModelBinder:UnBindByUI(self, true)
    self.WBP_ComBtn_Delete.OnClicked:Remove(self, DeleteCurMail)
end

return M
```
