---
id: npc-03
title: Kittens — StateFlow
sources:
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/state_flow/state_base.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/state_flow/state_flow.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/state_flow/state_flow_const.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/state_flow/state_flow_context.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/state_flow/state_flow_evaluator.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/state_flow/state_flow_exec_context.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/state_flow/state_flow_node.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/state_flow/state_flow_node_exec_context.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/state_flow/state_flow_transition.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/state_flow/state_flow_trigger.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/state_flow/state_flow_task/state_flow_task.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/state_flow/state_flow_task/sf_event_flow_task.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/state_flow/state_flow_task/sf_custom_task.lua
updated: 2026-05-11
---

# npc-03 — Kittens StateFlow

## Sources
见 frontmatter `sources` 字段——13 个文件,包含 `state_flow_task/` 子目录的 3 个文件。

## 1. StateFlow / StateFlowNode 类层次

| Class | Superclass | Role |
|-------|-----------|------|
| `StateFlow` | (none) | Runtime representation of a state flow config; owns node map + ancestor LUT |
| `StateFlowNode` | (none) | Static node descriptor (fields from spec_data); many-to-many with exec contexts |
| `StateFlowExecContext` | (none) | Runtime instance of a StateFlow; multiple may share same StateFlow |
| `StateFlowNodeExecContext` | (none) | Runtime representation of a single node within an exec context |
| `StateBase` | (none) | User-facing state class with `on_enter` / `on_exit` lifecycle |
| `StateFlowTransition` | (none) | Event-driven transition; owns trigger list |
| `StateFlowTrigger` | (none) | Single event-based trigger with conditions |
| `StateFlowEvaluator` | `NodeComponent` | Runtime data binding / pipeline evaluation |
| `StateFlowTask` | (none) | Base task; factory dispatches to `SF_EventFlowTask` or `SF_CustomTask` |
| `SF_EventFlowTask` | `StateFlowTask` | Wraps an EventFlow as a task |
| `SF_CustomTask` | `StateFlowTask` | Wraps an async function as a task |

## 2. StateFlowNode 字段

From `state_flow_node.lua` `initialize`:

```lua
self.__state_id = self.__spec_data.state_id
self.__is_selector = false        -- from spec_data.is_selector
self.__is_slot = false            -- from spec_data.is_slot
self.__parent_state = nil         -- lazy, resolved via spec_data.parent_state_id
self.__next_state = nil           -- set by parent's _get_child_states()
self.__child_states = nil         -- lazy, built from spec_data.child_state_id_list
self.__transition_spec_list       -- from spec_data.transition_spec_list
self.__task_spec_list             -- from spec_data.task_spec_list
```

Additional spec_data fields accessed elsewhere: `state_tag`, `accessary_tags`, `enter_condition_list`, `enter_condition_logic_operation`.

## 3. Transition 模型

`StateFlowTransition` spec_data fields: `target_state_id`, `trigger_list`, `condition_list`, `condition_type`.

```lua
---@class Enum_TransitionTriggerType
StateFlowConst.Enum_TransitionTriggerType = {
    state_completion_success = 'state_completion_success',
    state_completion_fail = 'state_completion_fail',
    sub_state_flow_return = 'sub_state_flow_return',
}
```

Built-in trigger types (上述三种) 由 `StateFlowNodeExecContext:_try_trigger_built_in_transition` 评估。Non-built-in triggers 走 `StateFlowTrigger` 的事件注册。

Transition evaluation: conditions checked via `StateFlowContext:check_conditions(condition_list, condition_type)`.

## 4. Trigger 类型

`StateFlowTrigger` registers an event listener on `spec_data.event_name` through `StateFlowContext:register_event_listener`. When triggered, it checks `spec_data.condition_list` with `spec_data.condition_logic_operation`.

Built-in triggers (`Enum_TransitionTriggerType`) bypass `StateFlowTrigger`; they are triggered programmatically by `StateFlowExecContext:_on_state_completed`.

## 5. is_selector vs is_slot 语义

**is_selector** — A selector state **cannot be a leaf**. During state selection (`__select_state`), if none of a selector's children pass enter conditions, the selector itself fails selection:

```lua
if not any_child_success and _root_state:_is_selector() then
    return false
end
```

A non-selector node can be selected as a leaf even if it has no eligible children.

**is_slot** — A slot state is used to **dynamically bind a sub StateFlow**. When entered, it delegates to the bound `StateFlowExecContext`:

```lua
if self:__is_slot_state() then
    local slot_flow_exec = self.__state_flow_exec_context:_get_slot_state_flow_exec(...)
    slot_flow_exec:_enter_cache_states()
end
```

Slot states cannot have a `StateBase` class attached. Slot transitions use `sub_state_flow_return` trigger type.

## 6. ExecContext / NodeExecContext

**StateFlowExecContext** — Created by `StateFlow:create_instance(_context, _flow_args)`. Holds:
- `__state_flow` (shared StateFlow reference)
- `__context` (StateFlowContext — dependency injection bridge)
- `__state_exec_mappings` (state_id -> NodeExecContext)
- `__curr_state` (leaf active NodeExecContext)
- `__sub_flow_exec_contexts` (slot_state_id -> child StateFlowExecContext)
- `__global_tasks`, `__active_tasks`, `__evaluators`

**StateFlowNodeExecContext** — Created lazily by `_state_node_to_state_exec`. Holds:
- `__state_node` (the StateFlowNode)
- `__state_instance` (StateBase subclass, created on first enter)
- `__tasks`, `__critical_tasks` (active tasks)
- `__transitions` (active StateFlowTransition instances, set up on enter, torn down on exit)
- `__active` flag

Lifecycle: `_enter(_source_state)` -> setup tags, fire tasks, setup transitions, enter StateBase. `_exit(_reason)` -> remove tags, stop tasks, teardown transitions, exit StateBase.

## 7. ReservedStateId 枚举

```lua
StateFlowConst.Enum_ReservedStateId = {
    root = 'root',       -- root node of any StateFlow
    parent = 'parent',   -- resolved to parent_state_node (or Attached_Slot_State for root of sub-flow)
    next = 'next',       -- next sibling in child_state_id_list order
    source = 'source',   -- leaf state at transition time (NOT the trigger-owning state)
}
```

Resolution logic in `StateFlowNodeExecContext:_find_state_node_by_id`:
- `parent` -> parent node, or `StateFlowConst.Attached_Slot_State` if root of slot-bound flow
- `next` -> `_get_next_state()` (linked-list built from `child_state_id_list` order)
- `source` -> the `__transition_source_state` captured at enter time

## 8. StateBase

Minimal base class (`state_base.lua`):

```lua
function StateBase:initialize(_state_exec, _args)
function StateBase:get_state_flow_exec_context()
function StateBase:complete_state(_success)  -- triggers transition evaluation
function StateBase:on_enter()
function StateBase:on_exit(_reason)
```

User subclasses override `on_enter` / `on_exit`. Call `self:complete_state(true/false)` to signal task/state completion.

## 9. Evaluator

`StateFlowEvaluator` extends `NodeComponent` (Kittens component system). Provides runtime data bindings for all nodes within a flow. Created during `StateFlowExecContext:__setup_evaluators()` from `state_flow:_get_evaluator_spec_list()`.

Interface: `on_init`, `on_enable`, `on_disable`, `get_data`. Attached to `__exec_context_handle` (a `NodeHandle`). Supports dependency ordering between evaluators via `set_exec_order_depencency`.

## 10. 跨页关联

- **npc-05** uses StateFlow to configure NPC's 13-state behaviour tree. Each NPC state (root/normal/move/dialogue/...) maps to a `StateFlowNode` with transitions driven by `state_completion_success/fail` and custom event triggers.
- `StateFlowContext` is the DI bridge — NPC system implements it (via `NpcActiveObject:include(StateFlowContext)`) to supply condition functions, event listeners, custom task functions, and tag container (GameplayTag-based state query).
- `SF_CustomTask` and `SF_EventFlowTask` are the two execution primitives used by NPC state actions.
- `Enum_StateFlowTaskType`: `event_flow` | `custom`.
- `Enum_ConditionLogicOperation`: `all` | `any`.
- Error constants: `Error_Reason_State_Flow_Node_Exited`, `Error_Reason_State_Flow_Node_Failed`, `Error_Pause_Custom_Task`, `Error_Reason_Stop_Global_Task`.
- See **npc-04** for NodeHandle + NodeComponent (StateFlowEvaluator subclasses NodeComponent).
