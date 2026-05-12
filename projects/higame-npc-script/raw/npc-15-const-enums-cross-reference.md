---
id: npc-15
title: npc_const.lua 全表枚举交叉索引
sources:
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/npc/npc_const.lua
updated: 2026-05-11
---

# npc-15 — npc_const.lua 全表枚举交叉索引

## Sources
- `E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/npc/npc_const.lua` (545 行)

## 0. 文件结构 (section 分组)

```lua
local Const = {}
-- Section 1: 基础开关 / 超时 / Debug
Const.Event_Flow_Context_Include_Key, Const.Debug_Mode, Const.LookAt_Enable,
  Const.Fade_In_Time_Out, Const.Performance_Time_Out, Const.Switches
-- Section 2: Significance 分级
Const.Significance_Distance_Table, Const.HUD_Significance_Name_Table,
  Const.Significance_Name_Table, Const.Enum_Significance_Category_Name, Const.Significance_Categories
-- Section 3: 状态 / 任务 / 交互枚举
Const.Enum_State, Const.Enum_GlobalTaskType, Const.Enum_BlockInteract_Reason,
  Const.Enum_Session, Const.Enum_Performance_Type, Const.Enum_Interact_Performance_Type
-- Section 4: Tag / Error / Mail / 资产类型
Const.MeshModel_Tag, Const.Accessary_Tag, Const.Enum_NpcEventFlowNodeClsName,
  Const.Enum_EventFlowConditionHandlerClsName, Const.Enum_Mail_Type,
  Const.Enum_EffectAssetType, Const.Enum_NpcTeleportAreaType, Const.Enum_NpcStateTag,
  Const.ChangeHead_Immunity_Tag, Const.Enum_NpcError
-- Section 5: NodeHandle 映射
Const.NodeCompBindingKey, Const.NodeHandleBindingKey
-- Section 6: 对话 / StateFlow 停止原因 / 转换原因 / 策略离开原因
Const.Enum_NpcInteractDialogueType, Const.Enum_State_Flow_Stop_Reason,
  Const.Enum_Reason, Const.Enum_State_Flow_Transition_Reason,
  Const.Enum_State_Task_Stop_Reason, Const.Enum_Strategy_Leave_Reason
-- Section 7: 隐藏/碰撞/寻路/移动枚举
Const.Enum_Npc_Hide_Reason, Const.Enum_Collision_Disable_Reason,
  Const.NpcEventFlowStateKey, Const.Enum_PathFollowingRequestResult,
  Const.Enum_PathFollowingResult, Const.Enum_MoveTaskResult
-- Section 8: 常用 EventFlow ID / Significance 分类
Const.NpcEntryPerformanceEventFlowId, Const.NpcEntryDestroyEventFlowId,
  Const.NpcCommonTrackingStand, Const.NpcCommonExcortStand,
  Const.NpcCustomDynamicFlowId, Const.NpcCustomDeadPerformFlowId,
  Const.Enum_PersistentPropertyName, Const.Enum_SignificanceCategory
-- Section 9: 数值/类型枚举 (NpcState/Klutz/BT)
Const.NpcState, Const.Enum_KlutzNpcType, Const.Enum_BehaviorTreeTaskType,
  Const.EBehaviorTreeType, Const.EStopDialogueReason, Const.EMoveWayType
-- Section 10: 动画槽 / 条件 Key / NPC 属性 / 相机 UI
Const.Npc_Anim_Slot_Name, Const.NpcEventFlowConditionKey,
  Const.NpcDialogueConditionKey, Const.NpcTopLogoConditionKey,
  Const.NpcProperty, Const.CameraCheckUIInfo
return Const
```

## 1. Enum_State (13 状态 id)

| 键 | 字符串值 | 对应 state 文件(server) |
|---|---|---|
| `Dialogue` | `'dialogue'` | `npc_state_dialogue.lua` |
| `Stealth` | `'stealth'` | `npc_state_stealth.lua` |
| `Teleport` | `'teleport'` | `npc_state_teleport.lua` |
| `Cutscene` | `'cutscene'` | (未对应独立文件) |
| `Move` | `'move'` | `npc_state_normal_move.lua` |
| `PaoPao` | `'paopao'` | `npc_state_paopao.lua` |
| `DialogueGroup` | `'dialogue_group'` | `npc_state_dialogue_group.lua` |
| `PerformanceChangeHead` | `'performance_changehead'` | `npc_state_change_head.lua` |
| `ChangeHeadMove` | `'change_head_move'` | `npc_state_changehead_move.lua` |
| `Charm` | `'charm'` | `npc_state_charm.lua` |
| `Float` | `'float'` | `npc_state_float.lua` |
| `Destroy` | `'destroy'` | `npc_state_destroy.lua` |

备注:`Const.Enum_State` 明确列 12 个(外加 `Cutscene` 共 13);另有内部 `normal` / `root` 状态在 `npc_state_config.lua` 的 node 声明中但未暴露到此枚举。

## 2. Enum_Mail_Type (60+ 类型 — 按业务簇重组)

**动画簇**:`Play_Anim_Montage`、`Play_Dynamic_Montage`、`Stop_Montage_In_Slot`、`Play_Mission_Monologue`、`Play_Monologue`、`Preload_Npc_Anim_Asset`

**对话簇**:`Start_Dialogue`、`Stop_Dialogue_Interact`、`Wait_Dialogue_Interact_Perform_First_Phase`、`Start_Dialogue_Interact_Perform`、`Disconnect_Dialogue`、`Start_Dialogue_Group`、`Stop_Dialogue_Group`、`Enter_Dialogue_Group`、`Exit_Dialogue_Group`、`Sync_Dialogue_Group_Progress`、`Resume_Dialogue_Group`

**表演簇**:`Start_Cutscene`、`Stop_Cutscene`、`Play_Exit_Performance`、`PlayPerformanceChangeHead`、`StopPerformanceChangeHead`、`StopCharmPerformance`、`Request_Charm_Config`、`Start_PaoPao`、`Stop_PaoPao`、`Show_Bubble`、`Start_Float`、`Stop_Float`、`Notify_Float_Landed`、`Play_Effect`、`Stop_Effect`

**移动簇**:`Teleport`、`Mission_Teleport`、`Move_To_Way_Point`、`Move_To_Actor`、`Stop_Move`、`Change_Move_Speed`、`Move_Along_Spline`、`Move_Way_Point_Group`

**淡入淡出/隐藏**:`Show_Or_Hide`、`Play_Fade`

**任务会话**:`Enter_Mission_Session`、`Exit_Mission_Session`、`Set_Exclusive_Interact`、`Start_Tracking_Task`、`Start_Escort_Task`、`Unload`

**其他**:`Change_Display_Name`、`Speak_Voice`、`Turn_Body_Yaw`、`Turn_Body_Target_Location`、`Turn_Body_Target_Actor`、`Look_At_Mode_Change`、`Player_Touch_NPC`

## 3. Enum_NpcEventFlowNodeClsName (EF 节点类名,与 npc-07 对齐)

30 个类名(28 Action + 2 Switch):`EF_Action_ShowOrHide`、`EF_Action_PlayAnimMontage`、`EF_Action_PlayDynamicMontage`、`EF_Action_PlayNpcDynamicMontage`、`EF_Action_ShowBubble`、`EF_Action_PlayMonologue`、`EF_Action_TurnBody`、`EF_Action_WaitSeconds`、`EF_Action_Teleport`、`EF_Action_AddOrRemoveStateTag`、`EF_Action_LookAtModeChange`、`EF_Action_ChangeDisplayName`、`EF_Action_SpeakVoice`、`EF_Action_MoveToWayPoint`、`EF_Action_MoveAlongWayPointChain`、`EF_Action_MoveToActor`、`EF_Action_CallStatusFlowRaw`、`EF_Action_CallAnimalBehaviorStateFlow`、`EF_Switch_Random`、`EF_Switch_Condition`、`EF_Action_Fade`、`EF_Action_ChangeMoveSpeed`、`EF_Action_MoveAlongSpline`、`EF_Action_Dialogue_Group`、`EF_Action_OccupySmartObject`、`EF_Action_PreloadNpcAnimAsset`、`EF_Action_Float`、`EF_Action_Effect`、`EF_Action_PanicRun`、`EF_Action_TriggerConditionEvent`、`EF_Action_Collision`

## 4. NodeCompBindingKey (17 键)

(与 npc-01 §4 全表一致,此处不复列)

## 5. Significance 系列 (与 npc-13 对齐)

```lua
Const.Significance_Distance_Table = {
    node_handle_high = 3000,        movement_high = 5000,       skeletal_high = 5000,
    node_handle_middle = 5000,      movement_middle = 8000,     skeletal_middle = 8000,
    node_handle_low = 999999,       movement_low = 999999,      skeletal_low = 999999,
    masked_material_apply = 1000,
}

Const.HUD_Significance_Name_Table = {
    name = 'name',
    happiness = 'happiness',
    bubble = 'bubble',
    track_icon = 'track_icon',
}

Const.Enum_Significance_Category_Name = {
    skeletal, node_handle, movement, name, happiness, bubble, track_icon, masked_material
}  -- 8 类

Const.Enum_SignificanceCategory = {  -- 注意与上面不同!
    Npc_Default_Tickable = 0,
    Npc_Widget_Component = 1,
    Item_Actor = 2,
}
```

## 6. 小枚举速查

```lua
Const.Enum_GlobalTaskType = {MoveToWayPoint, MoveToWayChain, MoveAlongSpline}

Const.Enum_BlockInteract_Reason = {Mission, Fading, Mission_Session, ChangeHead}

Const.Enum_Session = {Mission, Sequence, Performance='performance'}

Const.Enum_Performance_Type = {Entrance=1, Exit=2}

Const.Enum_Interact_Performance_Type = {Direct=1, Environment=2}  -- Direct 可打断, Environment 不被打断

Const.MeshModel_Tag = {Root='mesh_model', Fading='mesh_model.fading'}

Const.Accessary_Tag = {
    Tracking  = 'npc.accessary.tracking',
    Following = 'npc.accessary.following',
}

Const.Enum_EffectAssetType = {Niagara, StaticMesh, SkeletalMesh}

Const.Enum_NpcTeleportAreaType = {main_world, office}

Const.Enum_NpcStateTag = {speaking, stealth, interact_dialogue}

Const.ChangeHead_Immunity_Tag = {
    All         = 'changehead.immunity',
    Environment = 'changehead.immunity.environment',
    Direct      = 'changehead.immunity.direct',
}

Const.Enum_NpcInteractDialogueType = {Default=1, Title=2}

Const.Enum_Npc_Hide_Reason = {
    Unknown=1, Enter_Stealth_State=2, Interact_UI_Event=3, Player_Camera=4,
    Soul_Meter=5, Sequence=6, MiniGame=7, Init=8,
}

Const.Enum_Collision_Disable_Reason = {
    Visibility=1, Fading=2, EventFlow=3, StateTag=4,
}

Const.Enum_PathFollowingRequestResult = {Failed=0, AlreadyAtGoal=1, RequestSuccessful=2}

Const.Enum_PathFollowingResult = {
    Success=0, Blocked=1, OffPath=2, Aborted=3, Invalid=4,
}

Const.Enum_MoveTaskResult = {Success=0, Failed=1, Paused=2}

Const.NpcState = {None=0, Standing=1, Moving=2}  -- Same with UE NpcState enum

Const.Enum_KlutzNpcType = {
    Node=0, StartKlutz=1, StartRound=2, UpdateLight=3,
    UpdateSafe=4, RestartBT=5, Dead=6, Complete=7,
}

Const.Enum_BehaviorTreeTaskType = {
    None=0, Move=1, PlayAnimation=2, Dead=3,
    ShowBubble=4, StopBT=5, FollowPlayer=6,
}

Const.EBehaviorTreeType = {Base=1, Klutz=2, ChaseGame=3}

Const.EStopDialogueReason = {None=0, Normal=1, Sequence=2, InvalidInteract=3}

Const.EMoveWayType = {None=1, Actor=2, Location=3, Spline=4, Group=5, FollowActor=6}

Const.Npc_Anim_Slot_Name = {
    AllBody, FaceSlot, DefaultSlot, Performance,
    Mission_Flow_Body='AllBody', Mission_Flow_Face='FaceSlot',
    Event_Flow_Body='AllBody', Skyrail_Body='AllBody', Npc_Actor_Body='AllBody',
}

Const.NpcProperty = {
    DisplayName, DisplayIdentity, bShowToplogo,
    MissionSetEventFlowId, MissionSetName, MissionSetIdentity,
}

Const.CameraCheckUIInfo = {UI_Store_Content_Main, UI_PurchaseList}
```

## 7. Enum_NpcError (30+ 错误类型)

全部用 `Error:new('string')` 创建,可通过 `==` 比较。关键错误分类:

**动画/播放错误**:`Play_Anim_Montage_Error`、`Play_Dynamic_Montage_Error`、`Load_Anim_Asset_Error`
**组件缺失**:`Missing_Widget_Component`、`Missing_Monologue_component`、`Fail_To_Load_Mesh_Asset_Missing_Component`
**移动失败**:`Fail_To_Move_Stopped_By_Mail`、`Move_To_Invalid_Way_Point_Data`、`Move_To_Invalid_Way_Point`、`Move_To_Another_Way_Point`、`Fail_To_Move_To_Missing_Dependencies`、`Fail_To_Send_Move_To_Request`、`Fail_To_Send_Move_To_Request_Nil_ReceiveMoveCompleted`、`Fail_To_Move_To_Way_Point`、`Fail_To_Move_To_Way_Point_Blocked`、`Fail_To_Move_To_Way_Point_Cannot_Claim`、`Fail_To_Move_To_Way_Point_Cannot_Occupy`、`Move_To_Invalid_Way_Point_Group`
**护送/追踪**:`Excort_Time_out`(注意拼写)、`Tracking_Discovered_Player`、`Tracking_Lost_Npc`
**生命周期**:`Cancel_Caused_By_Npc_End_Play`
**被拒绝**:`Rejected_In_Mission_Session`、`Rejected_Interact_Performance_Priority`、`Rejected_Changehead_Immunity`、`Rejected_Same_Performance`
**参数错误**:`Show_Bubble_Invalid_Param`

```lua
Const.Enum_State_Flow_Stop_Reason = {
    Actor_EndPlay = Error:new('npc actor end play'),
    Global_Task_Condition_Completed = Error:new('Global_Task_Condition_Completed'),
}

Const.Enum_Reason = {
    AsyncBeforeDestroyActor, FadeInTimeOut, PerformanceTimeOut,
}

Const.Enum_State_Flow_Transition_Reason = {
    Player_Playing_Dialogue_Flow, Start_Dialogue_Interact_With_Player,
    Player_Interact_Component_Null, Player_Enter_Interact_Repeatedly,
    Player_Interact_Count_Invalid, Dialogue_Group_Mail_Exit,
}

Const.Enum_State_Task_Stop_Reason = {
    Exit_State, State_Dialogue_Reentrant_When_Exiting,
}

Const.Enum_Strategy_Leave_Reason = {
    Replaced,    -- 被新的表演/策略替换
    Stop,        -- 正常停止表演
    Exit_State,  -- 宿主状态退出时保底清理
}
```

## 8. NpcEventFlowStateKey (EF 上下文状态键)

```lua
Const.NpcEventFlowStateKey = {
    Dialogue_Interact_Target_Actor_Id, Dialogue_Interact_Exit_Yaw,
    Montage_Entry_Performance_Asset = 'entry_performance_anim',
    Montage_Destroy_Performance_Asset = 'destroy_performance_anim',
    MonologueAlertKeyName = 'AlertId',   TurnBodyKeyName = 'AlertTargetId',
    MonologueLostKeyName = 'LostId',     Dynamic_Montage_Alert_Asset = 'AlertMontage',
    MonologueUnsafeKeyName = 'UnsafeId', Dynamic_Montage_Unsafe_Asset = 'UnsafeMontage',
    CustomDynamicMontage = 'CustomDynamicMontage',
    SmartObjectId, SmartObjectType, POTransform, OriginTransform, OriginYaw,
}
```

## 9. 常用 EventFlow ID (DataTable `DT_NpcCommonEventFlow` 行号)

```lua
Const.NpcEntryPerformanceEventFlowId = 200001   -- 入场表演
Const.NpcEntryDestroyEventFlowId    = 200002   -- 销毁/出场表演
Const.NpcCommonTrackingStand        = 200006   -- tracking 任务站立 flow
Const.NpcCommonExcortStand          = 200007   -- escort 任务站立 flow (拼写保留)
Const.NpcCustomDynamicFlowId        = 200008   -- 自定义动态 flow
Const.NpcCustomDeadPerformFlowId    = 200009   -- 自定义死亡表演
```

## 10. 反查索引 (给其他 raw 笔记用)

| 主题 | 所在 const 表 | 引用 raw |
|---|---|---|
| 13 状态 id | `Enum_State` | npc-05 |
| 60+ mail 类型 | `Enum_Mail_Type` | npc-05, npc-06, npc-07 |
| 30 EF 节点类名 | `Enum_NpcEventFlowNodeClsName` | npc-07 |
| 2 条件处理器类名 | `Enum_EventFlowConditionHandlerClsName` | npc-08 |
| 17 组件键 | `NodeCompBindingKey` | npc-06 |
| 3 级 Significance 距离 | `Significance_Distance_Table` | npc-13 |
| 8 种隐藏原因 | `Enum_Npc_Hide_Reason` | npc-06(visibility) |
| 4 种碰撞禁用原因 | `Enum_Collision_Disable_Reason` | npc-06(interact) |
| 寻路结果 | `Enum_PathFollowingRequestResult`, `Enum_PathFollowingResult`, `Enum_MoveTaskResult` | npc-12 |
| Klutz 7 状态 | `Enum_KlutzNpcType` | npc-10 |
| BT 7 任务类型 | `Enum_BehaviorTreeTaskType` + `EBehaviorTreeType` | npc-10, npc-14 |
| 移动方式 | `EMoveWayType` | npc-10(move 子目录) |
| 动画槽 | `Npc_Anim_Slot_Name` | npc-06(anim_ctrl) |
| Immunity Tag | `ChangeHead_Immunity_Tag` | npc-11, npc-10(changehead) |

## 11. 跨页关联

- **npc-05**: `Enum_State`, `Enum_Mail_Type`, `NodeCompBindingKey`
- **npc-06**: `NodeCompBindingKey`, `Enum_Npc_Hide_Reason`, `Enum_Collision_Disable_Reason`, `Npc_Anim_Slot_Name`
- **npc-07**: `Enum_NpcEventFlowNodeClsName`, `NpcEventFlowStateKey`, DataTable 200001-200009
- **npc-08**: `Enum_EventFlowConditionHandlerClsName`
- **npc-09**: `NpcEventFlowStateKey.SmartObjectId / SmartObjectType / POTransform`
- **npc-10**: `Enum_KlutzNpcType`, `Enum_BehaviorTreeTaskType`, `EBehaviorTreeType`, `EMoveWayType`
- **npc-11**: `ChangeHead_Immunity_Tag`, `Enum_Interact_Performance_Type`
- **npc-12**: `Enum_PathFollowingRequestResult`, `Enum_PathFollowingResult`, `Enum_MoveTaskResult`
- **npc-13**: `Significance_Distance_Table`, `Enum_SignificanceCategory`
- **npc-14**: `EBehaviorTreeType`, `Enum_BehaviorTreeTaskType`
