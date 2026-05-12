---
id: npc-04
title: "Kittens — NodeHandle 与 NodeComponent"
sources:
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/node_handle/const.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/node_handle/node_component.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/node_handle/node_handle.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/node_handle/node_handle_mgr.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/node_handle/tick_group.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_flow/event_flow.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_flow/event_flow_const.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_flow/event_flow_context.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_flow/event_flow_exec_context.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_flow/event_flow_mgr.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_flow/event_flow_repr.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_flow/event_flow_repr_builder.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_flow/event_flow_node/event_flow_node.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_flow/event_flow_node/ef_action_node.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_flow/event_flow_node/ef_fork_node.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_flow/event_flow_node/ef_join_node.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_flow/event_flow_node/ef_subflow_node.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_flow/event_flow_node/ef_switch_node.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_system/event_system.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_system/event_system_interface.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/event_driven/event_system/event_system_locator.lua
updated: 2026-05-11
---

# npc-04 — Kittens NodeHandle 与 NodeComponent

## Sources

见 frontmatter sources 列表 (共21 个文件)。event_flow_state/ 和 node/ 子目录存在但为空。

## 1. NodeHandle 是什么

NodeHandle 是 Kittens 框架的 IoC 容器核心，类签名：

```lua
---@class NodeHandle
local NodeHandle = Kittens.class("NodeHandle")

function NodeHandle:initialize(_node, _mgr)
```

- 持有方：任何需要挂载 NodeComponent 的逻辑对象("node")通过 `NodeHandleMgr:bind_node(_node, _index_key)` 获取其 NodeHandle。
- `_index_key` 默认为 `Const.Default_Bind_Node_Index_Key = '__default'`；还有框架保留的 `Const.Cascading_Bind_Node_Index_Key = '__cascading'`。
- 获取方式：`NodeHandleMgr:get_binding_node_handle(_node, _index_key)` 返回已绑定的 handle。
- NPC 侧通过 get_server_logic_handle / get_client_logic_handle（定义在 NPC 逻辑层，非此文件）获取对应 NodeHandle 实例。

关键方法：add_component, remove_component, get_component, release, set_active, bind_node, unbind_node, set_exec_order_depencency, call_component_function。

## 2. NodeHandle vs UE Component 对比

| 维度 | UE UActorComponent | Kittens NodeHandle + NodeComponent |
|------|-------------------|------------------------------------|
| 语言层 | C++ / Blueprint | Lua (Kittens class 系统) |
| 注册方式 | Actor->CreateDefaultSubobject | `NodeHandle:add_component(_cls, _args, _key, _enable)` |
| 生命周期 | BeginPlay/EndPlay/Tick | on_init -> on_start -> on_enable -> on_update -> on_disable -> on_remove -> on_release |
| 跨服支持 | 需 Replicate | Lua 层直接区分 server/client handle，无引擎 replicate |
| 执行顺序 | TickGroup enum | 拓扑排序 (set_exec_order_depencency + DAG) |
| Tick 分散 | 无内建 | TickGroup scatter + time budget + priority |
| 激活/反激活 | SetActive | NodeHandle:set_active(bool) 级联通知所有 component |

## 3. NodeComponent

基类签名：

```lua
---@class NodeComponent
local NodeComponent = Kittens.class("NodeComponent")

function NodeComponent:initialize(_mgr, _args)
```

创建子类：`NodeComponent.create_node_component_class(_cls_name, _super_component_cls)`。

**add_component 签名**：

```lua
function NodeHandle:add_component(_component_cls, _component_args, _index_key, _enable)
```

- `_index_key`：Binding Key 机制，若 nil 则使用 component 实例本身作为 key。同一 handle 上 key 不可重复。
- `_enable`：默认 true，控制 component 添加后是否立即激活。

**Cascading Handle 机制**：`NodeComponent:become_cascading_handle()` 让组件自身成为一个 NodeHandle 的 bound node，其上可再挂子组件，激活/反激活与父组件联动。

**生命周期回调** (均可在子类 override)：on_init, on_start, on_enable, on_disable, on_remove, on_release, on_node_handle_bind, on_node_handle_unbind。

Tick 相关：若子类定义了 `on_update` 方法则自动被 NodeHandle 编入 update 链表；定义了 `on_late_update` 则编入 late_update 链表。

## 4. NodeHandleMgr

```lua
---@class NodeHandleMgr
local NodeHandleMgr = Kittens.class('NodeHandleMgr', nil)

function NodeHandleMgr:initialize(_kittens_inst)
```

- **非全局单例**：每个 _kittens_inst（Kittens 实例）拥有自己的 NodeHandleMgr。
- 创建时机：由 Kittens 实例创建并持有。
- 销毁时机：随 Kittens 实例销毁。通过 _unbind_node_handle 逐个清理绑定。
- `setup_signal(_signal_delegate)`：将 __update + __late_update 注册到外部 Delegate（通常是帧驱动信号）。
- 支持 SHARE_SCHEDULER_TIME_BUDGET_WITH_NODE_HANDLE 模式，tick 时调用 consume_logic_frame_time_budget 扣减共享时间预算。

## 5. TickGroup

```lua
---@class TickGroup
local TickGroup = Kittens.class('TickGroup', nil)
```

核心调度策略：

1. **Scatter（分帧）**：`scatter(_frame_cnt)` 将所有 tickable 分散在 N 帧内轮转执行 (tickable_per_frame = tickable_cnt / scatter_frame_cnt)。
2. **Time Budget（时间预算）**：`set_time_budget(_budget_ms)` 每帧超时后截断剩余 tickable。
3. **Priority（优先级）**：`set_priority_fn(_priority_fn, _sort_interval)` 每隔 N 帧按 priority 降序重排列表，高优先级先执行。
4. **Lazy Purge（惰性清理）**：unregister 仅标记 dirty；下一帧 tick 开始前双指针紧缩数组 O(n)。

NodeHandleMgr 持有 __default_update_tick_group 和 __tick_groups[key] 字典。通过 `alloc_tick_group(_group_key)` 创建命名 TickGroup，scatter_tick_group / register_handle_to_tick_group 配置。NPC 场景中 Significance 系统会基于距离调用 scatter 和 set_priority_fn 动态调整 NPC tick 频率。

## 6. EventFlow

EventFlow 是异步状态机，节点组成有向图，运行在 Kittens AsyncRoutine 协程上。

**EventFlowRepr**：共享的流程图结构定义（可被多个 EventFlow 实例复用）。通过 `EventFlowRepr.build_and_cache_event_flow_repr(_repr_key, _json_repr_data)` 从 JSON 反序列化，或 EventFlowReprBuilder 程序化构建。

**EventFlowContext**：依赖注入容器，持有 __context_obj（通常为 NPC 实例），支持 mixin 注入 (include_mixin)。

**EventFlowExecContext**：单次执行上下文，包装当前执行的 EventFlow + EventFlowContext + 当前节点。通过 metatable hook 将未找到的方法调用代理到当前 __flow_node——这使得 EF_ActionNode:on_start(self, _cancel_token) 中的 self 实际是 ExecContext。

**EventFlowNode 类型层级**：

| 节点类 | 继承 | 职责 |
|--------|------|------|
| EventFlowNode | 基类 | _eval / _get_next_node |
| EF_ActionNode | EventFlowNode | 执行单个 Action（NPC 的28 个 Action 均继承此类），完成时调用 complete 推进流 |
| EF_SwitchNode | EventFlowNode | 条件分支，按 exit_label 选后继 |
| EF_ForkNode | EventFlowNode | 并行分叉（ForkMode: all / all_settled / any / race；CancelMode: detach / exclusive） |
| EF_JoinNode | EventFlowNode | 汇合节点，接收 ForkNode 的完成信号 |
| EF_SubflowNode | EventFlowNode | 嵌套子流程，可设 reevaluate_on_resumed |

**EventFlowMgr**：管理活跃 EventFlow 实例的生命周期，创建/启动/停止流程。支持 EventFlow 池化复用 (ENABLE_EVENT_FLOW_POOL)。

**EventFlowConst.Enum_EventFlowErrorReason**：Paused, Stopped, Reevaluation_Event_Flow_Node, Stopped_By_Reevaluate_On_Resumed_Subflow_Node_Paused, Stopped_By_Subflow_Node_Stopped_During_Paused, Fork_Exclusive, Fatal_Error_Trace。

## 7. EventSystem

**EventSystemInterface** (纯接口)：

```lua
function EventSystemInterface:register_listener(_event_name, _listener, _handler)
function EventSystemInterface:register_listener_once(_event_name, _listener, _handler)
function EventSystemInterface:unregister_listener(_event_name, _listener, _handler)
```

**EventSystem** 继承自 EventSystemInterface，实现发布-订阅总线。当前实现体方法为空壳（方法体无逻辑），说明项目中实际通过外部 EventSystem 实现（如 C++ 侧或 HiGame 自定义实现）注入。提供 `broadcast_event(_event_name, ...)` 广播。

**EventSystemLocator**：Service Locator 模式，按 _register_key 注册/获取不同的 EventSystemInterface 实例。允许同一进程内存在多套事件总线。

EventSystem 与 EventFlow 关系：EventSystem 是扁平的发布-订阅通道；EventFlow 是有向图状态机。NPC Action 节点内部可通过 EventSystem 监听外部事件（如玩家进入范围），再通过 complete 推进 EventFlow 执行。

## 8. 跨页关联

- **npc-06**（NPC Component 体系）使用 NodeComponent.create_node_component_class 派生 NPC 专用组件（如 NPCBehaviorComponent），挂载到 NPC 的 server/client NodeHandle 上。
- **npc-07**（NPC EventFlow 28 Actions）每个 Action 均为 EF_ActionNode 子类，通过 EventFlowRepr.create_event_flow_node_class 注册；DT 配置通过 EventFlowReprBuilder 或 JSON repr 构建流程图。
