# HiGame UI 输入系统(键鼠/手柄/ESC/KeyAction/KeyIcon)+ UI 激活栈

- **Type**: Local code archaeology
- **Fetched**: 2026-05-10
- **Project**: higame-ui-script
- **Relevant to**: Q10
- **Sources drawn from**:
  1. `Source/HiGame/Public/UINavigation/UINavigationConfig.h`
  2. `Source/HiGame/Public/UINavigation/Input/*.h`(KeyAction/UINavigation/Block/KeyIcon/Cooldown 5 个 handler)
  3. `Source/HiGame/Public/UI/Gamepad/HiUI_GamepadMenu_Base.h`
  4. `Source/HiGame/Public/UI/HiInputMasterSubsystem.h`(`ActivatedUIArray` / `OnLayeredUIActivated` / `EvaluateTopUI` / `IsTopUI` / `FOnLayeredUIActivationChangedEvent` / `EHiInputMasterUIHandleKeyPolicy`)
  5. `Source/HiGame/Public/UI/IngameLayerManagerWidget.h`(`OnInputMethodChanged`)
  6. `Source/HiGame/Public/UI/HiInputKeyActionWidget.h`
  7. `Source/HiGame/Public/UI/HiKeyIconWidget.h`
  8. `Source/HiGame/Public/UI/HiInputDataAsset.h`
  9. `Content/Script/ui/uiframework/input_define.lua`(`DefaultUIAction` 表 + `ActionEvent` 枚举)
  10. `Content/Script/ui/uiframework/ui_window_base.lua`(`OnLayeredUIActivated` 调用)
  11. UE DataTable: `DT_GamepadIcon` / `DT_PCKeyIcon` / `DT_InputKeysExcludedLayeredWidgets` / `DT_InputKeysExcludedLayer`

---

## 1. Lua 监听按键 / 手柄按钮

**机制**: 通过 EnhancedInput Action 名注册回调,**不直接监听 KeyCode**。

### 配置文件
`Content/Script/ui/uiframework/input_define.lua`:
- `DefaultUIAction` 表定义所有 UI 可用 Action(映射到 `IA_UI_*` InputAction 资产)
- `ActionEvent` 枚举: `Triggered / Started / Completed / Canceled`

### 注册 API(在 ui_manager.lua)
```lua
UIManager:RegisterActionDelegate(UIObj, fnDelegate, ActionName, ActionEventType)
UIManager:UnRegisterActionDelegate(UIObj, ActionName, ActionEventType)
```

### 示例: 监听手柄 Y 按钮
```lua
local InputDef = require('ui.uiframework.input_define')

UIManager:RegisterActionDelegate(self, self.OnYButtonPressed,
    InputDef.DefaultUIAction.Button_Top,           -- IA_UI_Button_Top
    InputDef.ActionEvent.Started)

-- Destruct 必须反注册
UIManager:UnRegisterActionDelegate(self,
    InputDef.DefaultUIAction.Button_Top,
    InputDef.ActionEvent.Started)
```

## 2. ESC / Back 关闭

不需手写监听,设 `UIInfo.EscClose=true` 即可。底层流程:

```
C++  UHiInputMasterSubsystem::HandleInputKey
  └─→ 检测 CanHandleBack
        └─→ Lua  UIWindowBase:CanEscClose()  (查 UIInfo.EscClose)
              └─→ Lua  UIManager:CloseTopUIByEscClose()
                       (从栈顶找第一个 CanEscClose()=true 的关闭)
```

如需自定义 Back 逻辑,覆写:
```lua
function MyWindow:OnReturn()
    -- 自定义 Back 行为
end
```

## 3. UI 激活栈与 TopUI 判定

### C++ 端(`UHiInputMasterSubsystem`)
- `ActivatedUIArray: TArray<TWeakObjectPtr<UUserWidget>>` — 有序栈
- `OnLayeredUIActivated(UUserWidget*)` / `OnLayeredUIDeactivated(UUserWidget*)` — 入栈/出栈
- `EvaluateTopUI()` 计算 `CurrentTopUIID`
- `IsTopUI()` / `IsTopUIById()` 判定栈顶
- `FOnLayeredUIActivationChangedEvent` 广播栈变化

### Lua 端(`ui_window_base.lua`)
```lua
-- BeginShow 时入栈
UE.UHiInputMasterSubsystem.GetFromWidget(self):OnLayeredUIActivated(self)
-- BeginHide / BeginDestroy / CloseMyself 时出栈
UE.UHiInputMasterSubsystem.GetFromWidget(self):OnLayeredUIDeactivated(self)
```

**KeyAction 分发仅对栈顶生效**: `FHiInputKeyActionHandler::HandleInputKeyForUI(TopUIId, Key, Event)` 根据 `IsTopUIById` 过滤。

## 4. 输入设备切换通知

### C++ 流程
- `FHiInputMasterInputPreProcessor::GetInputType(Key)` 判断设备类型
- `UHiInputMasterSubsystem::SetInputType()` → 广播 `FInputTypeChangedEvent OnInputTypeChangedDelegate`
- 所有 `FHiInputHandlerBase` 子类的 `OnInputTypeChanged(Old, New)` 被调用
- `FHiKeyIconHandler::OnInputTypeChanged` 刷新已注册的 `UHiKeyIconWidget` 列表

### Lua 订阅
```lua
UIManager.IngameLayerManager.OnInputMethodChanged:Add(self, self.OnInputMethodChanged)

function self:OnInputMethodChanged(NewInputType)
    -- NewInputType: UE.ECommonInputType.MouseAndKeyboard / Gamepad
    -- 用途: 切换提示图标、显隐手柄光标等
end
```

## 5. KeyMap / KeyIcon / KeyAction 封装

| 层 | 类/文件 | 职责 |
|---|---|---|
| C++ Widget | `UHiInputKeyActionWidget`  | 绑定 `GamepadKey` + `PCKey`,自动切换显示,按键触发 `CallClick()` |
| C++ Widget | `UHiKeyIconWidget`         | 纯图标展示,按设备类型自动切 Gamepad/Keyboard Slot |
| C++ Handler | `FHiKeyIconHandler`       | 查 `DT_GamepadIcon` / `DT_PCKeyIcon` DataTable 获图标 |
| C++ DataAsset | `UHiInputDataAsset`     | 持有 `DT_GamepadIcon`(按 PS4/PS5/XBox 分列)、`DT_PCKeyIcon` |
| Lua 配置  | `input_define.DefaultUIAction` | Action 名 → IA 资产名映射 |

**配置按键提示无需写 Lua**: 在 WBP 中放置 `UHiInputKeyActionWidget`,设置 `GamepadKey`/`PCKey`,引擎自动:
1. 通过 `FHiKeyIconHandler` 查表获取图标
2. 设备切换时自动调 `OnInputTypeChanged` 切换 Slot 可见性
3. 按键按下时自动找关联 Button 执行 `CallClick`

## 6. Tip/Toast vs 模态 UI 输入屏蔽

### 策略枚举(`EHiInputMasterUIHandleKeyPolicy`)

| 策略 | 含义 |
|---|---|
| `Ignore`   | 透传按键到下层 |
| `Accept`   | 接收按键 |
| `Reject`   | 阻塞按键(不传递) |
| `LowLevel` | 接收但最低优先级 |

### 配置方式
- DataTable `DT_InputKeysExcludedLayeredWidgets`: 按 UI 蓝图名配置 `KeyPolicy + BackPolicy`
- DataTable `DT_InputKeysExcludedLayer`: 按 Layer 名配置
- 运行时: `SetLayeredUIKeyPolicy(UI, Policy)` / `SetLayeredUIBackPolicy(UI, Policy)`

### 实际差异
- **Tip/Toast**: 不调 `OnLayeredUIActivated`(不入栈),或入栈但 `KeyPolicy=Ignore` — 按键穿透到下层
- **模态 UI**: 入栈 + `KeyPolicy=Accept/Reject` — 阻断下层输入;`BackPolicy=Accept` 允许 ESC 关闭
- **输入全屏蔽**: `FHiInputBlockHandler::DisableAllInput(Reason)` 引用计数式阻断

### Lua 侧额外机制
`bIgnoreAnyViewportWidgets=true` 的 `KeyActionWidget` **无视栈顶判断**,始终响应(适用于全局快捷键)。

## 7. Handler 链(C++ 侧 5 个)

按顺序处理: **KeyAction → UINavigation → Block → KeyIcon → Cooldown**。每个 Handler 实现 `OnInputTypeChanged` 与 `HandleInputKey`。
