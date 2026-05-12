---
id: npc-07
title: EventFlow — 27 Actions + 2 Switches (NPC 节点全表)
sources:
  - Content/Script/npc/event_flow/npc_event_flow_module.lua
  - Content/Script/npc/event_flow/npc_event_flow_context_mixin.lua
  - Content/Script/npc/event_flow/c_npc_event_flow_context_mixin.lua
  - Content/Script/npc/event_flow/s_npc_event_flow_context_mixin.lua
  - Content/Script/npc/event_flow/s_empty_npc_event_flow_context_mixin.lua
  - Content/Script/npc/event_flow/character_npc_event_flow_context_mixin.lua
  - Content/Script/npc/event_flow_nodes/ef_action_*.lua (27 files)
  - Content/Script/npc/event_flow_nodes/ef_switch_random.lua
  - Content/Script/npc/event_flow_nodes/ef_switch_condition.lua
updated: 2026-05-11
---

# npc-07 — EventFlow Action 节点全表

> 注：原任务标题写"28 Action"，但 `npc_event_flow_module.lua` 实际 require 26 个 action 文件，加上 `ef_action_speak_voice.lua`（存在文件但不在 manifest，由 mixin 接口提供）共 27 个。本文档以 manifest + speak_voice 的 27 个 action 为准。

## Sources

| 文件 | 角色 |
|---|---|
| `npc_event_flow_module.lua` | Action import manifest（一次性 require 全部节点 + 2 个 condition handler）|
| `npc_event_flow_context_mixin.lua` | 基类 mixin：add_or_remove_state_tag / play_monologue / show_bubble / start_float / stop_float / change_display_name / trigger_condition_event |
| `character_npc_event_flow_context_mixin.lua` | character 派生：play_anim_montage / play_dynamic_montage / show_or_hide / look_at_mode_change / teleport / speak_voice / turn_body_yaw / turn_body_target_location / turn_body_target_actor / stop_move / move_to_way_point / move_to_actor / play_fade / change_move_speed / move_along_spline / move_way_point_group / dialogue_group / preload_npc_anim_asset / play_effect / stop_effect / get_move_blocked_evfl_id |
| `c_npc_event_flow_context_mixin.lua` | 客户端：dispatch_mail 走 `ActiveObjectRef:local_dispatch_mail` |
| `s_npc_event_flow_context_mixin.lua` | 服务端：dispatch_mail 走 `ActiveObjectRef:send_mail`（character 派生）|
| `s_empty_npc_event_flow_context_mixin.lua` | 服务端 empty：dispatch_mail 也走 `send_mail`，但继承自纯 NpcEventFlowContextMixin（无 character 接口）|

## 1. EventFlow 概念

EventFlow 是 Kittens 框架（`kittens.event_driven.event_flow`）提供的「数据驱动协程图」运行时。每条 `*EventFlowId` 引用一行 DataTable（如 `DT_NpcCommonEventFlow`），加载后被 `EventFlowRepr` 反序列化为有向图，节点类型由 `Enum_NpcEventFlowNodeClsName` 注册（见 npc-15）。运行期 NPC 通过 mixin 上下文（`NpcConst.Event_Flow_Context_Include_Key`）暴露能力 —— 节点 `on_start(_cancel_token)` 拿到该 mixin 后，把 raw_args（DT 行字段）转成业务调用，最后 `self:complete(err, res)` 触发后继节点。

控制流核心 idiom：

```
local ef_context = self:get_context():get_mixin(NpcConst.Event_Flow_Context_Include_Key)
if not ef_context:isInstanceOf(<对应 Mixin>) then ... self:complete(); return end
local promise = ef_context:<some_call>(args, wait_complete)
if wait_complete then
    local err, _ = Kittens.await(promise:get_future(), _cancel_token)
    self:complete(err, nil)
else
    self:complete(nil, nil)
end
```

## 2. 4 类 Mixin 上下文对照

```
NpcEventFlowContextMixin (基)
├── S_EmptyNpcEventFlowContextMixin   ── server，仅基础接口（无 character 行为）
└── CharacterNpcEventFlowContextMixin (角色扩展)
    ├── C_NpcEventFlowContextMixin     ── client，dispatch_mail = local_dispatch_mail
    └── S_NpcEventFlowContextMixin     ── server，dispatch_mail = send_mail
```

| 文件 | 父类 | dispatch_mail 实现 | 适用场景 |
|---|---|---|---|
| `npc_event_flow_context_mixin.lua` | nil | 抽象（子类 override） | 基础能力：state tag / display name / monologue / bubble / float / trigger_condition_event |
| `character_npc_event_flow_context_mixin.lua` | NpcEventFlowContextMixin | 抽象 | 在基础上扩展 character 专属能力（动画/移动/语音/特效等） |
| `c_npc_event_flow_context_mixin.lua` | CharacterNpcEventFlowContextMixin | `local_dispatch_mail` | 客户端 NPC 跑 EventFlow（本地表演逻辑） |
| `s_npc_event_flow_context_mixin.lua` | CharacterNpcEventFlowContextMixin | `send_mail` | 服务端 NPC 跑 EventFlow（character 形态，主流程） |
| `s_empty_npc_event_flow_context_mixin.lua` | NpcEventFlowContextMixin | `send_mail` | 服务端 NPC 没有 character 形态（如纯逻辑/标记 NPC）只暴露基础接口 |

`receive_mode` 由 `wait_complete` 决定：true → `request_and_response`，false → `fire_and_forget`。Mail 以 `NpcConst.Enum_Mail_Type.*` 路由（见 npc-15）。

## 3. 27 Action 全表

> Mixin 列：N=NpcEventFlowContextMixin / C=CharacterNpcEventFlowContextMixin。Mail 列：实际 dispatch 的 `Enum_Mail_Type`，"—"表示无 mail（直接调用或仅修改本地状态）。

| # | ClassName | RegisteredName (Enum_NpcEventFlowNodeClsName.*) | Mixin | 关键输入字段 | Mail Type | 一句话场景 |
|---|---|---|---|---|---|---|
| 1 | EF_Action_AddOrRemoveStateTag | EF_Action_AddOrRemoveStateTag | N | StateAccessaryTag.TagName, IsAdd | — (server_logic_component:add/remove_state_accessary_tag) | 给 NPC 加/减 state tag（控制 collision/visibility/AI 等的副作用 tag） |
| 2 | EF_Action_ChangeDisplayName | EF_Action_ChangeDisplayName | N | bShowToplogo, DisplayName | Change_Display_Name (fire_and_forget) | 改头顶名字、显隐 toplogo |
| 3 | EF_Action_ChangeMoveSpeed | EF_Action_ChangeMoveSpeed | C | ENpcMoveGaitType, GaitSpeedRate | Change_Move_Speed (fire_and_forget) | 切换 walk/jog/run 步态及倍率 |
| 4 | EF_Action_Collision | EF_Action_Collision | N | bEnable | — (转写为 add_or_remove_state_tag('npc.collision.action', not bEnable)) | 关/恢复 NPC 碰撞（语义糖：bEnable=false→添 tag→关碰撞） |
| 5 | EF_Action_Dialogue_Group | EF_Action_Dialogue_Group | C | DialogueGroupId, DialogueGroupLoopCount | Start_Dialogue_Group (await) | 启动一组多人对白，等播放结束 |
| 6 | EF_Action_Effect | EF_Action_Effect | C | bIsPlay, ResourcesId, SocketName, EffectTag, Seconds, Location, Rotation, Scale, AssetType | Play_Effect / Stop_Effect (fire_and_forget) | 播/停 Niagara/StaticMesh/SkeletalMesh 特效（按 tag 幂等） |
| 7 | EF_Action_Fade | EF_Action_Fade | C | bFadeIn, Seconds, WaitComplete | Play_Fade | 黑屏淡入/淡出 |
| 8 | EF_Action_Float | EF_Action_Float | N | MaxHeight, Seconds, WaitComplete, IsAscending | Start_Float / Stop_Float | NPC 漂浮上升+悬停 / 触发下降 |
| 9 | EF_Action_LookAtModeChange | EF_Action_LookAtModeChange | C | ELookAtMode | Look_At_Mode_Change (fire_and_forget) | 切换头/身体的 LookAt 模式 |
| 10 | EF_Action_MoveAlongSpline | EF_Action_MoveAlongSpline | C | MoveAloneSplineId, WaitComplete | Move_Along_Spline | 沿 Spline Actor 移动；支持 pause→stop_move 保存 distance→resume 续走 |
| 11 | EF_Action_MoveAlongWayPointChain | EF_Action_MoveAlongWayPointChain | C | WayChainId, WaitComplete (use_dt_flow=true) | Move_Way_Point_Group | 沿 way point chain（DT）移动；pause→保存 group_id+point_index→resume |
| 12 | EF_Action_MoveToActor | EF_Action_MoveToActor | C | MoveSpeed, TargetActorID, AcceptanceRadius, bReachTestIncludesAgentRadius, MoveBlockedEvflID, OverrideWayPointEvflID, ReevaluationTimeInterval, WaitComplete | Move_To_Actor | 跟随某 Actor（玩家/NPC）；blocked 时切到 MoveBlockedEvflID 表演 |
| 13 | EF_Action_MoveToWayPoint | EF_Action_MoveToWayPoint | C | WayPointId, Location, Variable, MoveBlockedEvflID, OverrideWayPointEvflID, ReevaluationTimeInterval, WaitComplete, DynamicMontageKeyName, PropagateErrorOnFail | Move_To_Way_Point | 移动到指定 WayPoint；可由 EventFlow Variable 覆写终点；可同步 dynamic montage 数据让客户端进入 Move 状态时播放 |
| 14 | EF_Action_OccupySmartObject | EF_Action_OccupySmartObject | C | SmartObjectId, SmartObjectVariable, EOccupyMode, Variable, VariableYaw, WaitComplete | — (smart_object_warpper:claim/free) | 占用 SmartObject（way_point / server_smart_object_po），结果 transform/yaw 写回 EventFlow 变量；支持 Free 模式释放上一次占用 |
| 15 | EF_Action_PanicRun | EF_Action_PanicRun | C | Seconds, RandomRadius, AcceptanceRadius, MaxTurnAngle, DynamicMontageKeyName, MoveCount | Move_To_Way_Point (循环) | 在原点周围随机选点惊慌乱跑，约束最大转向角；按 duration 或 move_count 退出 |
| 16 | EF_Action_PlayAnimMontage | EF_Action_PlayAnimMontage | C | InPlayRate, WaitComplete, AnimAssetKeyName, Anim_Length, AnimMontage (SoftObjectPtr) | Play_Anim_Montage | 播放 AnimMontage（也可通过 flow_state 变量取 soft_ref） |
| 17 | EF_Action_PlayDynamicMontage | EF_Action_PlayDynamicMontage | C | SlotNodeName, BlendInTime, BlendOutTime, InPlayRate, LoopCount, BlendOutTriggerTime, WaitComplete, Anim_Length, AnimAssetKeyName, AnimSequence (SoftObjectPtr) | Play_Dynamic_Montage | 用 AnimSequence + slot 动态合成 Montage 播放 |
| 18 | EF_Action_PlayMonologue | EF_Action_PlayMonologue | N | MonologueId, MonologueKeyName, WaitComplete | Play_Monologue | NPC 头顶气泡式独白（id 可由 flow_state.MonologueKeyName 覆写） |
| 19 | EF_Action_PlayNpcDynamicMontage | EF_Action_PlayNpcDynamicMontage | C | KeyName (suffix), SlotNodeName, BlendInTime, BlendOutTime, InPlayRate, LoopCount, BlendOutTriggerTime, Anim_Length, WaitComplete | Play_Dynamic_Montage | 按 NPC body type 拼 prefix（如 `Man_P_Attack_01`），从 `DT_ResourceIndex_NPCAnim_Main` 取 AnimSoftPtr 后调用 play_dynamic_montage |
| 20 | EF_Action_PreloadNpcAnimAsset | EF_Action_PreloadNpcAnimAsset | C | KeyName (suffix) | Preload_Npc_Anim_Asset (fire_and_forget) | 预加载某 NPC 动画 soft asset，避免后续 Play 时卡顿 |
| 21 | EF_Action_ShowBubble | EF_Action_ShowBubble | N | BubbleId, bShowBubble, WaitComplete | Show_Bubble | 显示/隐藏头顶气泡 |
| 22 | EF_Action_ShowOrHide | EF_Action_ShowOrHide | C | bEnable, Seconds | Show_Or_Hide (fire_and_forget) | NPC mesh 显隐切换 |
| 23 | EF_Action_SpeakVoice | EF_Action_SpeakVoice | C | AudioId, WaitComplete | Speak_Voice | 播放配音（DTAudio），可 await |
| 24 | EF_Action_Teleport | EF_Action_Teleport | C | bUseLocation, TeleportPointId, TpLocation, TpRotator, WaitComplete | Teleport (fire_and_forget) | 瞬移到坐标或传送点；自动用 CapsuleHalfHeight 抬高 Z |
| 25 | EF_Action_TriggerConditionEvent | EF_Action_TriggerConditionEvent | N | ConditionActionType (E_NpcConditionActionType) | — (npc_actor:request_trigger_condition_event) | 主动触发条件事件，激活 condition 系统的某个动作类型 |
| 26 | EF_Action_TurnBody | EF_Action_TurnBody | C | ETurnType (Yaw / TargetLocation / TargetActor), Yaw, YawStateKey, TargetLocation, TargetActorID, TargetActorIdStateKey, WaitComplete | Turn_Body_Yaw / Turn_Body_Target_Location / Turn_Body_Target_Actor | NPC 转身：按 yaw 角 / 朝向某点 / 朝向某 Actor；yaw 与 actor_id 都可由 flow_state 覆写 |
| 27 | EF_Action_WaitSeconds | EF_Action_WaitSeconds | (any) | Seconds | — (Future.wait_for_seconds) | 等待 N 秒，pause 时挂起 → resume 后继续 |

> 注：`ef_action_speak_voice.lua` 文件存在但**未**在 `npc_event_flow_module.lua` 的 require 列表中。猜测正在迁移或被某个上层 require；mixin `speak_voice` 接口仍可被其他节点/外部调用。

## 4. 2 Switch 全表

| ClassName | RegisteredName | Mixin | 输入 | 行为 |
|---|---|---|---|---|
| EF_Switch_Random | EF_Switch_Random | (any) | (无 raw arg) | `math.random(1, #exit_label_list)` 选一条出口；无出口则 complete(nil)。父类 `EF_SwitchNode` 在 `_add_successor_node` 时收集 exit_label。 |
| EF_Switch_Condition | EF_Switch_Condition | N | ConditionType (E_EventFlowConditionType 枚举值), ConditionParam (string) | 按 ConditionType 从 `EFConditionRegistry.get_handler_class_by_enum` 查处理器，调 `handler:evaluate(npc_actor, condition_param) → 1-based index`，按 index 从 `exit_label_list` 选出口；handler 缺失或 ConditionType 为 nil → 兜底走第一条分支。pcall 包裹 evaluate 防止 handler 抛异常。 |

Condition handler 注册见 npc-08；当前 `npc_event_flow_module.lua` 末尾还会 require：

- `npc.event_flow_condition.ef_condition_handlers.efcond_test_switch`
- `npc.event_flow_condition.ef_condition_handlers.efcond_home_capture_reward`

## 5. 典型 Action 实现模式

### 5.1 `EF_Action_PlayAnimMontage`（带 await）

```lua
local EF_Action_PlayAnimMontage = EventFlowRepr.create_event_flow_node_class(
    NpcConst.Enum_NpcEventFlowNodeClsName.EF_Action_PlayAnimMontage,
    Kittens.EventFlow.EF_ActionNode)

function EF_Action_PlayAnimMontage:fetch_raw_args(_raw_args)
    self.__args.in_play_rate = _raw_args.InPlayRate
    self.__args.wait_complete = _raw_args.WaitComplete
    self.__args.anim_asset_key_name = _raw_args.AnimAssetKeyName
    self.__args.anim_length = _raw_args.Anim_Length
    -- soft_obj_ptr → asset_path → soft_obj_path → soft_obj_ref 三步转换
    local path = Kittens.AssetUtils.get_asset_path_by_soft_obj_ptr(_raw_args.AnimMontage)
    local soft_path = Kittens.AssetUtils.get_soft_obj_path_by_asset_path(path)
    self.__args.anim_montage_soft_object_reference =
        Kittens.AssetUtils.get_soft_obj_ref_by_soft_obj_path(soft_path)
end

function EF_Action_PlayAnimMontage:on_start(_cancel_token)
    local ef_context = self:get_context():get_mixin(NpcConst.Event_Flow_Context_Include_Key)
    -- 类型守卫：必须在 character mixin 中
    if not ef_context:isInstanceOf(CharacterNpcEventFlowContextMixin) then ... end

    -- 资产降级：raw_args 取不到则从 flow_state 变量取
    local soft_ref = args.anim_montage_soft_object_reference
    if Kittens.AssetUtils.get_asset_path_by_soft_obj_ptr(soft_ref) == '' then
        soft_ref = self:get_event_flow_state():get_value(args.anim_asset_key_name)
    end

    local promise = ef_context:play_anim_montage(in_play_rate, soft_ref, wait_complete, anim_length)
    -- mixin 内部 dispatch_mail(Play_Anim_Montage, payload, receive_mode)
    if wait_complete then
        local err, _ = Kittens.await(promise:get_future(), _cancel_token)
        self:complete(err, nil)
    else
        self:complete(nil, nil)
    end
end
```

要点：

- raw_args → __args 字段重命名（驼峰→蛇形）由 `fetch_raw_args` 完成
- 软引用三段转换：`soft_obj_ptr → asset_path → soft_obj_path → soft_obj_ref`
- 资产二级降级：直接配置缺失 → 退到 EventFlowState 变量
- 同步/异步分支：根据 wait_complete 选 await 或立即 complete

### 5.2 `EF_Action_OccupySmartObject`（与 SmartObject 握手）

```lua
function EF_Action_OccupySmartObject:on_start(_cancel_token)
    -- 1. Free 模式：直接释放上一次占用的 warpper
    if occupy_mode == Enum.E_SmartObjectOccupyMode.Free then
        self:__free_previous_warpper(npc_actor, actor_id, _cancel_token)
        self:complete(nil, nil); return
    end

    -- 2. SmartObjectId 来源：raw_args 直接给 / 或从 flow_state 变量读
    local id = smart_object_variable_value or args.smart_object_id

    -- 3. 通过 SmartObjectInterface 拿对应的 warpper（way_point / server_smart_object_po 已 require 触发自注册）
    local warpper = SmartObjectInterface.get_smart_object_warpper(id)

    -- 4. 调 warpper:claim(...) → Promise，await 拿 transform 数据
    -- 5. 把结果 transform/yaw 写回 EventFlow 变量（Variable / VariableYaw）
    -- 6. 持有 warpper 引用挂到 npc_actor.occupy_warpper 供下次 Free
end
```

要点：

- 「类型注册表」模式：`require('npc.smart_objects.smart_object_warpper.smart_object_way_point')` 等通过 require 副作用注册到 SmartObjectInterface
- 占用结果通过 EventFlowState 双向桥接到后续节点（如 MoveToWayPoint 通过 Variable 取占用点的 transform）
- 释放时机：下一次 Occupy 触发 `__free_previous_warpper`，或显式 EOccupyMode=Free 节点

## 6. DataTable 200001-200009 行名注解（DT_NpcCommonEventFlow）

`npc_const.lua` 中已知的预留 EventFlowId（行 ID）：

| ID | 字段名 (NpcConst) | 含义 |
|---|---|---|
| 200001 | NpcEntryPerformanceEventFlowId | 入场表演（NPC spawn 时跑） |
| 200002 | NpcEntryDestroyEventFlowId | 销毁前表演（NPC despawn 前跑） |
| 200006 | NpcCommonTrackingStand | tracking 模式下的待机站立 flow |
| 200007 | NpcCommonExcortStand | escort 模式下的待机站立 flow |
| 200008 | NpcCustomDynamicFlowId | 自定义动态 flow 入口（运行时由业务决定具体 flow） |
| 200009 | NpcCustomDeadPerformFlowId | 自定义死亡表演 flow |

> 200003-200005 在 npc_const 中暂未发现命名常量，可能是预留 ID 或在子模块中定义。

## 7. 跨页关联

- npc-08：Condition Switch 注册中心（`EFConditionRegistry`、`E_EventFlowConditionType` → handler 类映射、`efcond_test_switch` / `efcond_home_capture_reward` 实现）
- npc-09：OccupySmartObject 与 SmartObject 系统（`smart_object_way_point`、`server_smart_object_po`、claim/free 协议、target_transform/target_yaw 解析）
- npc-15：枚举反查表
  - `Enum_NpcEventFlowNodeClsName.*` ↔ 27 + 2 节点 ClassName
  - `Enum_Mail_Type.*` ↔ Play_Anim_Montage / Play_Dynamic_Montage / Show_Bubble / Play_Monologue / Show_Or_Hide / Look_At_Mode_Change / Teleport / Speak_Voice / Turn_Body_Yaw / Turn_Body_Target_Location / Turn_Body_Target_Actor / Stop_Move / Move_To_Way_Point / Move_To_Actor / Play_Fade / Change_Move_Speed / Move_Along_Spline / Move_Way_Point_Group / Start_Dialogue_Group / Preload_Npc_Anim_Asset / Play_Effect / Stop_Effect / Change_Display_Name / Start_Float / Stop_Float
- npc-05/06：NPC Actor 与 ServerLogicComponent（`add_state_accessary_tag` / `remove_state_accessary_tag` / `request_trigger_condition_event` 的 C++ 侧实现）
- npc-10：mixin 上下文挂载点（`NpcConst.Event_Flow_Context_Include_Key`）与 NPC 生命周期中绑定 mixin 的时机

## 8. 节点能力分类总览

按职能把 27 个 action 分组，便于策划/程序快速定位：

| 类别 | 节点 |
|---|---|
| 状态/属性 | EF_Action_AddOrRemoveStateTag, EF_Action_Collision, EF_Action_ChangeDisplayName, EF_Action_LookAtModeChange, EF_Action_ChangeMoveSpeed, EF_Action_ShowOrHide |
| 动画/蒙太奇 | EF_Action_PlayAnimMontage, EF_Action_PlayDynamicMontage, EF_Action_PlayNpcDynamicMontage, EF_Action_PreloadNpcAnimAsset |
| 移动 | EF_Action_MoveToWayPoint, EF_Action_MoveToActor, EF_Action_MoveAlongSpline, EF_Action_MoveAlongWayPointChain, EF_Action_PanicRun, EF_Action_OccupySmartObject, EF_Action_TurnBody, EF_Action_Teleport |
| 表演/UI | EF_Action_PlayMonologue, EF_Action_ShowBubble, EF_Action_Dialogue_Group, EF_Action_SpeakVoice, EF_Action_Effect, EF_Action_Fade, EF_Action_Float |
| 控制流/事件 | EF_Action_WaitSeconds, EF_Action_TriggerConditionEvent + EF_Switch_Random, EF_Switch_Condition |

## 9. Pause / Resume 协议

带 await 的 action（动画/移动/对话）必须支持 EventFlow 的 pause / resume 语义。从 `ef_action_move_along_spline` 的实现归纳出标准模板：

```
while true do
    local err, res = Kittens.await(promise:get_future(), _cancel_token)
    if err == nil then self:complete(nil, res); return end
    if err == EventFlowConst.Enum_EventFlowErrorReason.Fatal_Error_Trace then
        self:complete(err, nil); return
    elseif err == EventFlowConst.Enum_EventFlowErrorReason.Paused then
        -- 1) 主动停止当前异步任务（保存进度，如 spline 的 current_distance）
        local stop_promise = ef_context:stop_move(true)
        local _, stop_res = Kittens.await(stop_promise:get_future())
        local saved_progress = stop_res:unfold().current_distance or 0
        -- 2) 等 resume 信号
        local resume_err, updated_token = self:await_for_resume()
        if resume_err ~= nil then self:complete(resume_err, nil); return end
        -- 3) 用保存的进度重启异步任务
        _cancel_token = updated_token
        promise = ef_context:move_along_spline(spline_id, wait_complete, saved_progress)
    else
        if _cancel_token:is_cancellation_requested() then ef_context:stop_move() end
        self:complete(err, nil); return
    end
end
```

变种：

- `ef_action_move_along_way_point_chain` 保存 `move_group_id + move_point_index` 而非 distance
- `ef_action_wait_seconds` 直接用 `Future.wait_for_seconds`，pause 时仅 `await_for_resume`，无需保存进度
- `ef_action_dialogue_group` 当前注释掉了 Paused 分支处理（TODO），仅响应 Fatal_Error_Trace

## 10. EventFlow Variable 桥接惯例

多个节点通过 EventFlowState 的 key/value 系统在节点之间传值：

| 输出节点 | 写入的 variable | 消费节点 |
|---|---|---|
| EF_Action_OccupySmartObject | Variable (transform), VariableYaw | EF_Action_MoveToWayPoint (Variable 读 transform.Translation 作终点), EF_Action_TurnBody (yaw_state_key) |
| (外部业务) | MonologueKeyName | EF_Action_PlayMonologue |
| (外部业务) | AnimAssetKeyName | EF_Action_PlayAnimMontage / EF_Action_PlayDynamicMontage (soft_obj_ref 降级) |
| (外部业务) | TargetActorIdStateKey, YawStateKey | EF_Action_TurnBody |

接口：`self:get_event_flow_state():get_value(key)` / `:set_value(key, value)`。Mixin 端不感知 EventFlowState，所有桥接在节点 on_start 内完成。

## 11. 动态 Montage Key 规则

`ef_action_play_npc_dynamic_montage` 与 `ef_action_preload_npc_anim_asset` 共用一套 key 规则：

```
prefix = NpcUtils.get_prefix_body_type(npc_actor)   -- 如 "Man" / "Woman" / "Child"
key_name = prefix .. '_' .. suffix                  -- DT 行 ID
row = DataTableUtils.GetDataTableRow(AssetRef.DT_ResourceIndex_NPCAnim_Main, key_name)
soft_ref = (path → soft_path → soft_obj_ref) row.AnimSoftPtr
```

策划只需配 suffix（如 `P_Attack_01`），运行时自动按 NPC body type 拼前缀。`EF_Action_MoveToWayPoint` 与 `EF_Action_PanicRun` 也通过 `DynamicMontageKeyName` 字段把 key 透传给客户端 Move 状态机使用。

