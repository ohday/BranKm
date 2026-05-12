---
id: npc-08
title: EventFlow Condition 与 Switch
sources:
  - Content/Script/npc/event_flow_condition/ef_condition_handler.lua
  - Content/Script/npc/event_flow_condition/ef_condition_registry.lua
  - Content/Script/npc/event_flow_condition/ef_condition_handlers/efcond_test_switch.lua
  - Content/Script/npc/event_flow_condition/ef_condition_handlers/efcond_home_capture_reward.lua
updated: 2026-05-11
---

# npc-08 — EventFlow Condition 与 Switch

## Sources

- `Projects/HiGame/Content/Script/npc/event_flow_condition/ef_condition_handler.lua`
- `Projects/HiGame/Content/Script/npc/event_flow_condition/ef_condition_registry.lua`
- `Projects/HiGame/Content/Script/npc/event_flow_condition/ef_condition_handlers/efcond_test_switch.lua`
- `Projects/HiGame/Content/Script/npc/event_flow_condition/ef_condition_handlers/efcond_home_capture_reward.lua`

## 1. EFCond 概念

EFCond（EventFlow Condition）是 NPC 事件流中分支选择逻辑的 Lua 端实现。
EventFlow 图上的 `EF_Switch_Condition` 节点本身只持有 UE 枚举字段
（`E_EventFlowConditionType`）以及若干 `exit_label_list`，运行时它必须把
"当前应选哪条出边" 这件事委派给一个 Lua handler——也就是 `EF_ConditionHandler`
的子类。每个 handler 只暴露一个语义：实现 `evaluate(_context_obj,
_condition_param)`，返回一个 1-based 的分支索引。

为了把 UE 枚举值映射到具体的 handler class，所有 handler 类在被 `require`
时通过 `EFConditionRegistry.create_handler_class(class_name)` 自注册到一张全局
`handler_class_mapping` 表中。注册键就是 `NpcConst.Enum_EventFlowConditionHandlerClsName`
里的字符串常量；查询时 `ef_condition_registry` 通过
`NpcUtils.__npc_evfl_cond_handler_cls_name_lookup(_enum_condition_type)` 把
UE 枚举值翻译为 class_name，再去 mapping 表里取出 handler 类。

## 2. ef_condition_handler 基类

文件：`ef_condition_handler.lua`（24 行）。

继承关系：

```lua
local EF_ConditionHandler = Kittens.class('EF_ConditionHandler', nil)
```

接口签名（verbatim）：

```lua
---@brief 条件判断方法，子类必须覆写此方法
---@param _context_obj any 上下文对象（由业务层决定具体类型，如 Actor 等）
---@param _condition_param string 额外参数
---@return number 分支索引（1-based），用于从 exit_label_list 中选择对应的分支
function EF_ConditionHandler:evaluate(_context_obj, _condition_param)
    Logger.error('event_flow_condition', 'EF_ConditionHandler.evaluate() must be overrided by child class')
    return 1
end
```

基类的 `evaluate` 实现是个保底：直接打 error 日志并返回 `1`。子类必须覆写。

**实例化与调用：** 注册表只保留 class（即 `Kittens.class` 的产物，相当于元类
表），并不在 require 阶段 new 出实例。从源码看 `evaluate` 是用 `:` 调用的方法
风格，但具体由 EF_Switch_Condition 节点 handler 在哪里完成 `:new()`/直接做
冒号调用（即 class 自身充当 self），需要参考 npc-07 中节点侧的代码。注册表
本身不做实例化，`evaluate` 的调用时机就是节点 fire 时。

## 3. ef_condition_registry 注册中心

文件：`ef_condition_registry.lua`（45 行）。

API 速查：

| API | 作用 |
|---|---|
| `EFConditionRegistry.handler_class_mapping` | `class_name -> handler 类` 的 table |
| `EFConditionRegistry.create_handler_class(_class_name)` | 创建子类（继承自 `EF_ConditionHandler`）并写入 mapping，返回类供调用方实现 `evaluate` |
| `EFConditionRegistry.get_handler_class_by_enum(_enum_condition_type)` | 拿 UE 枚举 `E_EventFlowConditionType` 的某个值，先经 `NpcUtils.__npc_evfl_cond_handler_cls_name_lookup` 翻译为 class_name，再到 mapping 取 handler；找不到打 error 返回 nil |

注册时机：每个 `efcond_*.lua` 文件最顶部都会 `require('npc.event_flow_condition.ef_condition_registry')`
然后立即调用 `EFConditionRegistry.create_handler_class(...)`。也就是说，**注册
是 require 副作用**，依赖 `npc_event_flow_module`（参考 npc-07）在引导阶段把所
有 handler 文件都 require 一遍。Logger 模块被 require 进来用于在 lookup 失败时
输出 `'EFConditionRegistry: no handler found for enum_condition_type: %s'`。

## 4. 2 个内置 EFCond

### EFCond_TestSwitch

文件：`ef_condition_handlers/efcond_test_switch.lua`（23 行）。

类创建方式（verbatim）：

```lua
local EFCond_TestSwitch = EFConditionRegistry.create_handler_class(
    NpcConst.Enum_EventFlowConditionHandlerClsName.EFCond_TestSwitch
)
```

注册键：`NpcConst.Enum_EventFlowConditionHandlerClsName.EFCond_TestSwitch`。

`evaluate` 行为：占位实现，无视参数，固定 `selected_index = 1` 并返回 `1`。
注释明确写了 `-- 占位，替换为实际逻辑`，意图是作为开发期的脚手架/最小可运行
样例。

### EFCond_HomeCaptureReward

文件：`ef_condition_handlers/efcond_home_capture_reward.lua`（40 行）。

注册键：`NpcConst.Enum_EventFlowConditionHandlerClsName.EFCond_HomeCaptureReward`。

`evaluate` 行为（基于 NPC actor 上下文）：

1. 通过 `UE.UHiUtilsFunctionLibrary.GetHostPlayerState(_npc_actor)` 拿到本机
   PlayerState；为 nil → warn 日志 + 返回 `1`。
2. 读 PlayerState 上的 `PlayerHomeCaptureComponent`；为 nil → warn 日志（注意
   日志正文写的是 `"FlyingCollectionNpc:init_listener PlayerHomeCaptureComponent is nil"`，
   疑似复制粘贴遗留）+ 返回 `1`。
3. 调 `PlayerHomeCaptureComponent:GetRewardListCount()`：若返回 truthy，则
   `reward_list_count > 0 and 2 or 1`——有奖励走分支 2，没奖励走分支 1。
4. 兜底返回 `1`。

约定：分支 1 = "无奖励/异常路径"，分支 2 = "有奖励路径"。这一约定是 EF 图侧
和 handler 侧的 implicit contract，需要图作者在 `exit_label_list` 里按相同顺序
排好出边。

## 5. EF_Switch_Condition vs EF_Switch_Random 差异

EF_Switch_Condition 与 EF_Switch_Random 都依赖 `exit_label_list` 的 1-based 索引
来选择分支，但触发与配置完全不同：

| 维度 | EF_Switch_Condition | EF_Switch_Random |
|---|---|---|
| 选择来源 | Lua handler 的 `evaluate` 返回值 | 节点内部按权重/uniform 随机 |
| 关键输入 | UE 枚举 `E_EventFlowConditionType` + `_condition_param` 字符串 | 概率/权重数组（节点字段） |
| 注册中心 | `EFConditionRegistry.handler_class_mapping` | 不需要 handler，节点自决 |
| 上下文 | `_context_obj`（业务层决定，NPC 场景下是 NPC actor） | 不依赖外部上下文 |
| 可测试性 | handler 可单测（mock actor） | 只能压统计 |

`_condition_param` 是字符串，handler 注释里举的例子是 "奖励ID"——也就是节点
作者可以把若干同类逻辑复用同一 handler，靠 `_condition_param` 区分子情况。
本仓库内置的两个 handler 暂时都没用到该字段。

## 6. 跨页关联

- **npc-07** — 详细描述 EF_Switch_Condition 节点本身如何在 npc_event_flow
  阶段拿到 handler、组装 `_context_obj`、再调 `evaluate`，以及 handler 文件
  是如何被 `npc_event_flow_module` 在 require 链上拉起来完成自注册的。
- **npc-15** — 列出 `NpcConst.Enum_EventFlowConditionHandlerClsName` 中
  所有 class_name 字符串常量，以及 UE 端 `E_EventFlowConditionType` 枚举与
  Lua class_name 之间的映射函数 `NpcUtils.__npc_evfl_cond_handler_cls_name_lookup`
  的实现位置。
