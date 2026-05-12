---
id: npc-01
title: NPC 全栈拓扑与启动链
sources:
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/actors/common/NPC.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/actors/common/BaseNPC.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/npc/npc_active_object.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/npc/npc_const.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Source/HiGame/Public/Npc/NpcSettings.h
updated: 2026-05-11
---

# npc-01 — NPC 全栈拓扑与启动链

## Sources
- `E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/actors/common/NPC.lua`
- `E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/actors/common/BaseNPC.lua`
- `E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/npc/npc_active_object.lua`
- `E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/npc/npc_const.lua`
- `E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Source/HiGame/Public/Npc/NpcSettings.h`

## 1. 蓝图绑定层 (BPA_NPCBase_C / BP_BaseNPC_C)

### NPC.lua -> BPA_NPCBase_C

文件头部的 `@type` 注解声明了蓝图绑定目标:

```lua
---@type BPA_NPCBase_C
require "UnLua"
```

`NPC` 类继承自 `actors.common.interactable.base.editor_character`(即 `ActorBase`),通过 `Class(ActorBase)` 创建。

关键 UnLua 钩子:
- `ReceiveBeginPlay` — 调用 `Super(NPC).ReceiveBeginPlay(self)`,然后处理 `AudioLoop` 逻辑(仅客户端 `self:IsClient()` 时开启 AudioTick 定时器)。
- `ReceivePostBeginPlay` — 通过 `self:SendMessage("ReceivePostBeginPlay")` 广播消息(Kittens 组件消息机制)。
- `ReceiveEndPlay` — 清理 AudioTickHandle 定时器后调用 super。

静态属性:
- `NPC.EntityServiceName = ""`
- `NPC.EntityPropertyMessageName = "Npc"`

### BaseNPC.lua -> BP_BaseNPC_C

```lua
---@type BP_BaseNPC_C
require "UnLua"
```

`BaseNPC` 类继承自 `common.actor`(即 `Actor`),是一个极简骨架。仅覆写了 `ReceiveBeginPlay`,调用 super。其余 hook(EndPlay / Tick / AnyDamage / Overlap)均注释掉。

### 两个蓝图类的关系

| Lua 文件 | 绑定蓝图 | 基类链 |
|---|---|---|
| `NPC.lua` | `BPA_NPCBase_C` | `editor_character` -> (interactable chain) |
| `BaseNPC.lua` | `BP_BaseNPC_C` | `common.actor` (Actor) |

`NPC.lua` 是功能完备的 NPC 载体(含 DialogueComponent、NpcBehaviorComponent、BillboardComponent、VisibilityManagementComponent 等蓝图组件引用),`BaseNPC.lua` 是轻量壳。

## 2. NpcActiveObject 与 NpcActor 关系

文件 `npc_active_object.lua` 首行注释明确声明:

```lua
---NpcActiveObject is 1:1 with NpcActor
```

### 类继承

```lua
local NpcActiveObject = Kittens.class('NpcActiveObject', ActiveObject):include(StateFlowContext)
```

`NpcActiveObject` 继承 `Kittens.ActiveObject.ActiveObject`,并 mixin 了 `Kittens.StateFlow.StateFlowContext` 接口。

### 1:1 映射建立

在 `initialize` 中,传入的 `_context` 参数即为 NpcActor:

```lua
function NpcActiveObject:initialize(_active_object_mgr, _active_object_id, _context, _location, _effect_handlers)
    NpcActiveObject.super.initialize(self, _active_object_mgr, _active_object_id, _context, _location, _effect_handlers)
    self.__npc_actor = _context
    self.__handle = self.__npc_actor:get_server_logic_handle()
    self.__state_component = self.__handle:get_component(NpcConst.NodeCompBindingKey.State_Comp)
    -- ...
end
```

- `self.__npc_actor` 持有 NpcActor 引用(即 C++ 侧 NPC Actor 对象)。
- 生命周期终结在 `_on_npc_actor_end_play()`,其中停止 `__state_flow` 并置空 `__event_flow_context`。

### Server 侧定位

`NpcActiveObject` 只获取 `get_server_logic_handle()`,不碰 client handle。这意味着 `NpcActiveObject` 运行在服务器侧。客户端侧的 NPC 逻辑由 `NpcClientStateRoot` 等 client-only 类驱动(详见 npc-05)。

### StateFlowContext 接口实现

`NpcActiveObject` 实现以下 StateFlowContext 方法,均代理到 `__state_component`(UE 侧 NPCStateComponent):
- `impl_add_tag(_tag)` -> `self.__state_component:add_state_tag(_tag)`
- `impl_remove_tag(_tag)` -> `self.__state_component:remove_state_tag(_tag)`
- `impl_has_tag(_tag, _exact_match)` -> `self.__state_component:has_state_tag(_tag, _exact_match)`
- `impl_has_any_tags(...)` / `impl_has_all_tags(...)`
- `get_context_object()` -> 返回 `self.__npc_actor`

## 3. StateFlow 实例化时机

StateFlow 在 `NpcActiveObject:initialize()` 中同步创建并启动,紧跟在 handle 和 state_component 获取之后:

```lua
self.__state_flow = NpcStateConfig.get_npc_state_flow():create_instance(self, {})
self.__state_component:_set_state_machine(self.__state_flow)
self.__state_flow:start()
```

时序:
1. `NpcStateConfig.get_npc_state_flow()` 返回状态流定义(详见 npc-05 `npc_state_config.lua`)。
2. `:create_instance(self, {})` 以 `NpcActiveObject` 自身为 context 创建实例。
3. `__state_component:_set_state_machine(self.__state_flow)` 将状态机注册到 UE 侧 NPCStateComponent,建立 Lua <-> C++ 双向握手。
4. `self.__state_flow:start()` 立即启动状态机。

停止时机在 `_on_npc_actor_end_play()`:

```lua
self.__state_flow:stop(NpcConst.Enum_State_Flow_Stop_Reason.Actor_EndPlay)
self.__state_flow = nil
```

`Enum_State_Flow_Stop_Reason.Actor_EndPlay` 定义在 `npc_const.lua` 中:`Error:new('npc actor end play')`。

## 4. NodeHandle 获取链

### Server Handle

在 `NpcActiveObject:initialize()` 中:

```lua
self.__handle = self.__npc_actor:get_server_logic_handle()
```

`get_server_logic_handle()` 是 NpcActor(蓝图/C++ 侧)暴露的方法。返回的 handle 用于查询绑定组件。

### NodeCompBindingKey 全表

`npc_const.lua` 中 `Const.NodeCompBindingKey` 定义了所有通过 handle 查找组件的 key:

| Key 常量 | 字符串值 |
|---|---|
| `Anim_Ctrl_Comp` | `'anim_ctrl_comp'` |
| `Look_At_Comp` | `'look_at_comp'` |
| `Fade_Comp` | `'fade_component'` |
| `Interact_Comp` | `'npc_interact_comp'` |
| `Visibility_Comp` | `'visibility_comp'` |
| `Significance_Comp` | `'significance_comp'` |
| `Task_Scheduler_Comp` | `'task_scheduler_comp'` |
| `Widget_Component` | `'npc_widget_component'` |
| `Monologue_Comp` | `'monologue_component'` |
| `Teleport_Comp` | `'teleport_component'` |
| `State_Comp` | `'state_component'` |
| `Message_Receiver_Comp` | `'message_receiver_comp'` |
| `Server_Logic_Node_Component` | `'server_logic_node_comp'` |
| `Client_Logic_Node_Component` | `'client_logic_node_comp'` |
| `Dialogue_Group_Comp` | `'dialogue_group_comp'` |
| `Float_Comp` | `'float_component'` |
| `Perform_Comp` | `'perform_component'` |

### NodeHandleBindingKey

```lua
Const.NodeHandleBindingKey = {
    npc_active_object = 'npc_active_object'
}
```

这是反向绑定:handle 上绑定 `NpcActiveObject` 自身,供其他系统通过 handle 查找 active object。

## 5. NpcSettings.h 暴露的 UCLASS 配置

```cpp
UCLASS(config = Game, defaultconfig, meta = (DisplayName = "Npc"))
class HIGAME_API UNpcSettings : public UDeveloperSettings
{
    GENERATED_UCLASS_BODY()

public:
    static bool EnableServerAnimation();
    static bool EnableServerLoadSkinAnim();

public:
    UPROPERTY(Config, EditAnywhere, Category = Npc)
    uint8 bEnableServerAnimation : 1;

    UPROPERTY(Config, EditAnywhere, Category = Npc)
    uint8 bEnableServerLoadSkinAnim : 1;
};
```

- 继承自 `UDeveloperSettings`,通过 `config = Game` 存储在 `DefaultGame.ini`。
- 两个位域 UPROPERTY:`bEnableServerAnimation` 和 `bEnableServerLoadSkinAnim`。
- 两个静态方法 `EnableServerAnimation()` / `EnableServerLoadSkinAnim()` 供全局查询。
- 这些配置控制服务器侧是否播放动画和加载皮肤动画资源。具体消费点在 C++ NPC Actor 或动画组件中(详见 npc-13)。

## 6. 跨页关联线索

- **npc-05** (npc_state_config.lua): `NpcStateConfig.get_npc_state_flow()` 的状态流定义、`NpcStateConfig.StateClassMapping` 状态类映射表。
- **npc-05** (states/server/*): 各 `Const.Enum_State` 对应的状态实现(Dialogue, Stealth, Teleport, Move 等 13 个)。
- **npc-07** (event_flow): `npc_event_flow_module` 和 `Const.Enum_NpcEventFlowNodeClsName` 中列举的 30+ EventFlow action/switch 节点。
- **npc-06** (components): `Client_Logic_Node_Component` 的客户端侧对称结构。
- **npc-15** (npc_const): `Const.Significance_Distance_Table` 和 `Const.Significance_Categories` 定义的 LOD 分级策略。
- **npc-02/03/04** (Kittens framework): `ActiveObject`、`StateFlowContext`、`EventFlowContext` 框架层接口定义。
