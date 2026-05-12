---
id: npc-05
title: NPC 13 状态机 (server + client)
sources:
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_config.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_base.lua
  - Projects/HiGame/Content/Script/npc/states/server/state_mail_handlers.lua
  - Projects/HiGame/Content/Script/npc/states/server/way_point_data.lua
  - Projects/HiGame/Content/Script/npc/states/server/way_point_helper.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_root.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_normal.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_normal_move.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_move.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_dialogue.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_dialogue_group.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_stealth.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_paopao.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_teleport.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_destroy.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_change_head.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_changehead_move.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_charm.lua
  - Projects/HiGame/Content/Script/npc/states/server/npc_state_float.lua
  - Projects/HiGame/Content/Script/npc/states/client/npc_state_config.lua
  - Projects/HiGame/Content/Script/npc/states/client/npc_client_state_base.lua
  - Projects/HiGame/Content/Script/npc/states/client/state_mail_handlers.lua
  - Projects/HiGame/Content/Script/npc/states/client/npc_client_state_root.lua
  - Projects/HiGame/Content/Script/npc/states/client/npc_client_state_normal.lua
  - Projects/HiGame/Content/Script/npc/states/client/npc_client_state_move.lua
  - Projects/HiGame/Content/Script/npc/states/client/npc_client_state_dialogue.lua
  - Projects/HiGame/Content/Script/npc/states/client/npc_client_state_dialogue_group.lua
  - Projects/HiGame/Content/Script/npc/states/client/npc_client_state_stealth.lua
  - Projects/HiGame/Content/Script/npc/states/client/npc_client_state_paopao.lua
  - Projects/HiGame/Content/Script/npc/states/client/npc_client_state_change_head.lua
  - Projects/HiGame/Content/Script/npc/states/client/npc_client_state_charm.lua
updated: 2026-05-11
---

# npc-05 — NPC 13 状态机

## Sources
全部源自 `Projects/HiGame/Content/Script/npc/states/{server,client}/`，详见 frontmatter `sources` 列表。

## 1. 服务器状态树 (npc_state_config.lua)

`NpcStateConfig.get_npc_state_flow()` 手写 `StateFlow`，节点通过 `StateFlowNode:new(spec, flow)` 构造，再 `flow:add_state_flow_nodes(...)`。

### 1.1 节点表

| state_id | parent | is_selector | state_tag |
|---|---|---|---|
| `root` | `''` | false | `root` |
| `normal` | `root` | false | `root.normal` |
| `stealth` | `root` | false | `root.stealth` |
| `move` | `root` | false | `root.move` |
| `interact` | `root` | **true** | `root.interact` |
| `dialogue` | `interact` | false | `root.interact.dialogue` |
| `teleport` | `root` | false | `root.teleport` |
| `paopao` | `root` | false | `root.paopao` |
| `performance` | `root` | **true** | `root.performance` |
| `dialogue_group` | `performance` | false | `root.performance.dialogue_group` |
| `performance_changehead` | `performance` | false | `root.performance.change_head` |
| `change_head_move` | `performance_changehead` | false | `root.performance.change_head.move` |
| `charm` | `performance_changehead` | false | `root.performance.change_head.charm` |
| `float` | `performance_changehead` | false | `root.performance.change_head.float` |
| `destroy` | `root` | false | `root.destroy` |

> `root.child_state_id_list` 写为 `{'normal','stealth','interact','teleport','move, paopao','performance'}`（注意 `'move, paopao'` 是单字符串，看起来是手抖；`destroy` 未列入但作为 root 子节点存在）。

### 1.2 Transition 概要

```lua
-- dialogue / teleport / stealth: success → source(自取消)
-- move / change_head_move: success+fail → parent
-- charm: success → parent
```

`StateFlowConst.Enum_ReservedStateId` 提供 `source` (返回触发源 state) 与 `parent` (上跳父 state) 两个保留 ID。

## 2. 13 个服务器状态详表

| state_id | 类名 | state_tag | 关键 mail / 主要职责 |
|---|---|---|---|
| `root` | `NpcStateRoot` | `root` | 加载所有 Component；注册全部 stateless mail handler；`Teleport`/`Show_Or_Hide`/`Play_Exit_Performance`/`Start_PaoPao`/`Start_Dialogue_Group`/`Enter_Dialogue_Group`/`Sync_Dialogue_Group_Progress`/`PlayPerformanceChangeHead` 触发状态切换 |
| `normal` | `NpcStateNormal` | `root.normal` | copy root switcher；`Start_Dialogue`、`Move_To_*`、`Move_Along_Spline`、`Move_Way_Point_Group`、`Speak_Voice` |
| `move` | `NpcStateNormalMove` (extends `NpcStateMove`) | `root.move` | move 基础逻辑 + `Start_Dialogue` → 切到 `Dialogue` |
| `dialogue` | `NpcStateDialogue` | `root.interact.dialogue` | `Start_Dialogue`/`Stop_Dialogue_Interact`/`Disconnect_Dialogue`/`Wait_Dialogue_Interact_Perform_First_Phase`/`Teleport`/`Play_Exit_Performance`；维护 `__interacter_player_gids` |
| `stealth` | `NpcStateStealth` | `root.stealth` | 默认 handler `stash`；仅 `Show_Or_Hide`(is_show=true 退出) |
| `paopao` | `NpcStatePaoPao` | `root.paopao` | `on_enter` 设 `E_NpcPerformanceState.PaoPao`；`Stop_PaoPao` 回 source |
| `teleport` | `NpcStateTeleport` | `root.teleport` | 默认 handler stash；`Teleport` 调用 `teleport_component:teleport()` 后 `complete_state(true)` |
| `destroy` | `NpcStateDestroy` | `root.destroy` | 弃用 state（注释“强制移除 object 后无法收到信件”）；只接 `Play_Exit_Performance` |
| `dialogue_group` | `NpcStateDialogueGroup` | `root.performance.dialogue_group` | Owner/参与者两侧逻辑；`Start_Dialogue_Group`/`Stop_Dialogue_Group`/`Enter_Dialogue_Group`/`Exit_Dialogue_Group`/`Resume_Dialogue_Group`/`Start_Dialogue` |
| `performance_changehead` | `NpcStateChangeHead`（注意 class 名注册为 `'NpcStateNormal'`，是源码 bug） | `root.performance.change_head` | 大量 stateless copy + `PlayPerformanceChangeHead`/`StopPerformanceChangeHead`/`Move_*`/`Start_Float`/`Stop_Float`/`Notify_Float_Landed`；`on_enter` 进 `Performance` mission session |
| `change_head_move` | `NpcStateMoveChangeHead` (extends `NpcStateMove`) | `root.performance.change_head.move` | 继承 move + `PlayPerformanceChangeHead`/`StopPerformanceChangeHead`：先 `__handler_stop_move` 再 stash 回放 |
| `charm` | `NpcStateCharm` | `root.performance.change_head.charm` | `PlayPerformanceChangeHead`/`StopPerformanceChangeHead`/`StopCharmPerformance`/`Request_Charm_Config`；通过 `SwapHeadUtils.GetCharmPOConfigForActor` 读 DT 配置 |
| `float` | `NpcStateFloat` | `root.performance.change_head.float` | 自行 add `FloatComponent`；`Start_Float`/`Stop_Float`/`Notify_Float_Landed`；落地后 `transition_to_state(PerformanceChangeHead)` |

## 3. NpcStateRoot 详解

### 3.1 `initialize()` 加载的 Component

服务器 root 一次性 `add_component` 9 个：

```lua
handle:add_component(NpcTaskSchedulerComponent, ..., Task_Scheduler_Comp, true)
handle:add_component(NpcWidgetComponent,        ..., Widget_Component,    true)
handle:add_component(MonologueComponent,        ..., Monologue_Comp,      true)
handle:add_component(AnimComponent,             ..., Anim_Ctrl_Comp,      true)
handle:add_component(LookAtComponent,           ..., Look_At_Comp,        true)
handle:add_component(TeleportComponent,         ..., Teleport_Comp,       true)
handle:add_component(NpcFadeComponent,          ..., Fade_Comp,           true)
handle:add_component(MessageReceiverComponent,  ..., Message_Receiver_Comp, true)
handle:add_component(InteractComponent,         ..., Interact_Comp,       true)
handle:add_component(PerformComponent,          ..., Perform_Comp,        true)
```

> 第 4 个参数 `true` 表示 binding key 持久化 (见 `npc-04 node_handle`)。

### 3.2 mail_switcher 注册

**Stateless（从 `StateMailHandlers` 仓库 `copy_handler_item`）**：`Play_Anim_Montage`、`Play_Dynamic_Montage`、`Stop_Montage_In_Slot`、`Turn_Body_Yaw`、`Turn_Body_Target_Location`、`Turn_Body_Target_Actor`、`Look_At_Mode_Change`、`Play_Monologue`、`Play_Mission_Monologue`、`Show_Bubble`、`Change_Display_Name`、`Play_Fade`、`Change_Move_Speed`、`Mission_Teleport`、`Enter_Mission_Session`、`Exit_Mission_Session`、`Set_Exclusive_Interact`、`Start_Tracking_Task`、`Start_Escort_Task`、`Preload_Npc_Anim_Asset`、`Play_Effect`、`Stop_Effect`、`Player_Touch_NPC` (合计 23 个)。

**Stateful（直接 `case` 注入 root 自有方法，依赖状态切换）**：

```lua
self.__mail_switcher:case(Teleport,             self.__handler_teleport)             -- stash + transition Teleport
self.__mail_switcher:case(Show_Or_Hide,         self.__handler_show_or_hide)         -- not show → Stealth
self.__mail_switcher:case(Play_Exit_Performance,self.__handler_play_exit_performance)-- stash + Destroy
self.__mail_switcher:case(Start_PaoPao,         self.__handler_start_paopao)         -- transition PaoPao
self.__mail_switcher:case(Start_Dialogue_Group, self.__handler_start_dialogue_group) -- stash + DialogueGroup
self.__mail_switcher:case(Enter_Dialogue_Group, self.__handler_enter_dialogue_group) -- stash + DialogueGroup
self.__mail_switcher:case(Sync_Dialogue_Group_Progress, ...)                          -- 写 Server_Logic_Comp data
self.__mail_switcher:case(PlayPerformanceChangeHead, self.__handle_play_performance_change_head) -- 检查 mission session 后 stash + ChangeHead
-- + 4 个 'client_send_test_*' / 'server_send_*' 测试 handler
```

> Stateful handler 之所以留在 root 而非仓库：它们依赖 `self.__active_object:stash(_mail)` + `self:transition_to_state(...)`，需绑定到具体 state 实例。

## 4. state_mail_handlers 共享仓库

`Projects/HiGame/Content/Script/npc/states/server/state_mail_handlers.lua` 把所有「不依赖状态切换」的处理器打成一张表。每个 handler 签名固定：`function(_state, _mail) -> Promise|Error|nil`。

| 区域 | Stateless handler |
|---|---|
| Animation | `__handler_play_anim_montage`、`__handler_play_dynamic_montage`、`__handler_stop_montage_in_slot` |
| Turn body | `__handler_turn_body_yaw`、`__handler_turn_body_target_location`、`__handler_turn_body_target_actor` |
| Look at | `__handler_look_at_mode_change` |
| Monologue | `__handler_play_monologue` (兼 `Play_Monologue` 与 `Play_Mission_Monologue`) |
| Widget | `__handler_show_bubble`(含类型校验 → 防 RPC `Text Lua is not allowed` 崩溃)、`__handler_change_display_name` |
| Fade | `__handler_play_fade` |
| Move speed | `__handler_change_move_speed` |
| Teleport | `__handler_mission_teleport` |
| Mission session | `__handler_enter_mission_session`、`__handler_exit_mission_session` |
| Interact | `__handler_set_exclusive_interact` |
| Task | `__handler_start_tracking_task`、`__handler_start_escort_task` |
| Anim asset | `__handler_preload_npc_anim_asset` |
| Effect | `__handler_play_effect`、`__handler_stop_effect` |
| Player touch | `__handler_player_touch_npc` (NPC 上 spawn `USphereComponent` 触发器,promise.on_settled 清理) |

仓库末尾把 `Enum_Mail_Type.X → handler` 全部塞入 `StateMailHandlers[mail_type] = handler_fn`，供 `MailSwitcher:copy_handler_item(repo, mail_type)` 按需拷贝。

```lua
self.__mail_switcher:copy_handler_item(StateMailHandlers,
    NpcConst.Enum_Mail_Type.Play_Anim_Montage)  -- 等同 case(Play_Anim_Montage, repo[Play_Anim_Montage])
```

> `NpcStateRoot`、`NpcStateChangeHead`、`NpcStateCharm`、`NpcStateFloat`、`NpcStateMoveChangeHead` 都通过 `copy_handler_item` 共享同一份实现，避免每个 state 重写一次 anim/turn/effect 逻辑。

## 5. 客户端状态机镜像

### 5.1 `npc_state_config.lua` (client)

只手写 9 个映射，**没有显式构造 StateFlow**，仅返回 `StateClassMapping`：`root / normal / stealth / dialogue / move / paopao / dialogue_group / change_head / change_head_move(=NpcClientStateMove) / charm`。

### 5.2 客户端缺失的状态（相对服务器）

服务器有但客户端无独立类的：`teleport`、`destroy`、`performance_changehead.float` (无独立 client 类)、`interact` (selector)、`performance` (selector)。其中 `change_head_move` 复用了 `NpcClientStateMove`。

### 5.3 `NpcClientStateRoot:initialize` 加什么 Component

```lua
self.widget_comp     = handle:add_component(NpcWidgetComponent,    ..., Widget_Component, false)
-- 非空 NPC 才加：
self.anim_comp       = handle:add_component(AnimComponent,         ..., Anim_Ctrl_Comp,   false)
self.look_at_comp    = handle:add_component(LookAtComponent,       ..., Look_At_Comp,     false)
self.monologue_comp  = handle:add_component(NodeMonologueComponent,..., Monologue_Comp,   false)
self.teleport_comp   = handle:add_component(NodeTeleportComponent, ..., Teleport_Comp,    false)
self.fade_comp       = handle:add_component(NodeFadeComponent,     ..., Fade_Comp,        false)
self.perform_comp    = handle:add_component(NodePerformComponent,  ..., Perform_Comp,     false)
-- 全 NPC 都加：
self.visibility_comp = handle:add_component(VisibilityComponent,   ..., Visibility_Comp,  false)
-- 默认音乐广场 npc 用专用 interact comp，否则用通用
self.interact_comp   = handle:add_component(NpcInteractMusicPlazaComponent | NpcInteractComponent, ..., Interact_Comp, ?)
```

随后 `__async_init_state` 在 `npc_high_scheduler` 中按顺序 `try_start` 每个 component 并 `Kittens.yield()`，直到 `end_play_cancel_token` 被取消。

### 5.4 客户端 `state_mail_handlers` 与服务器的差异

仓库小很多，仅 9 个 handler：`Turn_Body_{Yaw,Target_Actor,Target_Location}`、`Play_Anim_Montage`、`Play_Dynamic_Montage`、`Show_Bubble`、`Change_Display_Name`、`Teleport` (兜底版,直接 `K2_SetActorLocationAndRotation`)、`Preload_Npc_Anim_Asset`、`Player_Touch_NPC`。所有 handler 签名 `(_state NpcClientStateBase, _mail Mail)`。客户端没有 monologue/effect/fade/task/mission_session 等复杂业务，因为它们都由 server 推动并 RPC 下发。

## 6. 双端状态同步机制

- **没有 state replication**：双端各自跑自己的 `StateFlow`。状态同步通过 **Mail 流转 + RPC + 蓝图 Replicated 属性** 三条路径：
  - **Mail**：服务器 stateful handler 处理后通常会 `send_mail` 给客户端 `mail_dispatcher`（参见 `npc-02`）。例如 `Show_Bubble`、`Change_Display_Name`、`Turn_Body_*`、`Play_*_Montage` 双端都有同名 handler。
  - **Multicast RPC**：例如 dialogue_group 用 `Multicast_ResumeDialogueGroup`（见 `npc_state_dialogue_group.lua` line 51）。
  - **Replicated UProperty**：例如 `MoveDynamicMontageData` 蓝图变量被服务器写入，客户端 `NpcClientStateMove:on_enter` 读取后 `request_play_dynamic_montage`。
- **状态切换信号**：客户端 state 由 server 通过特定 mail（如 `Start_Dialogue_Interact_Perform`、`Show_Or_Hide`）+ npc_actor 上的事件驱动；客户端没有 `transition_to_state` 调用。

`NpcStateBase.on_enter` 通过 `self.__active_object:become(self.__mail_switcher, true)` 切换 MailSwitcher，并 `unstash_all()`；`NpcClientStateBase.on_enter` 通过 `self.__npc_ref:set_mail_filter` / `set_mail_dispatcher` 切换 filter+dispatcher，并 `unstash_all_dispatch_mails()`。

## 7. way_point_data / way_point_helper

`way_point_data.lua` 定义 `MovePointDataBase` 抽象基类 + 5 个具体类型（按 `NpcConst.EMoveWayType`）：

| 类 | EMoveWayType | 用途 | get_move_task |
|---|---|---|---|
| `MovePointDataActor` | `Actor` | 通过 `way_point_id` 找 WayPoint Actor | `npc_move_point_task` |
| `MovePointDataLocation` | `Location` | 直接给 `move_to_location` (FVector) | `npc_move_point_task` |
| `MoveAlongSpline` | `Spline` | 沿 spline actor 移动 | `npc_move_spline_task` |
| `MoveWayPointGroup` | `Group` | 一组 way point + 起始 index | `npc_move_point_group_task` |
| `MovePointDataFollowActor` | `FollowActor` | 跟随玩家/Actor (含 catch_radius/move_speed) | `npc_move_follow_player_task` |

每个类实现：`get_type / get_value / get_data_id / get_move_task / try_get_fetch_evfl_id / is_valid / async_claim_way_point / async_occupy_way_point / free_way_point`。Actor/Group 类型支持 `claim/occupy/free` 互斥占用，Location/Spline/Follow 三种是空操作。

`way_point_helper.lua` 仅 3 个函数：
- `create_way_point_data(_args)`：按 `move_way_type` 分发到 5 个具体类 (附带兼容旧字段自动推断)
- `equal(a, b)`：`type+value` 匹配
- `is_valid(ctx, data)`：转发到 `data:is_valid(ctx)`

`NpcStateMove:__handler_move` 接受 `Move_To_Way_Point/Move_Along_Spline/Move_Way_Point_Group` 等 mail，调用 `WayPointHelper.create_way_point_data(payload)` 得到统一的 `MovePointDataBase` 子类对象，再调用 `:get_move_task()` 得到 `NpcMoveTaskBase`，最后 fire 为 `SF_CustomTask`。

## 8. 跨页关联

- **npc-04 (node_handle)**：root state `add_component` 的第 4 参 `true`/`false` 表示 binding key 是否持久；客户端 root 的异步 `try_start` 模式见 `npc-04` 解释。
- **npc-06 (components)**：root state 一次性加载 9 个 server / 7~8 个 client component；`NpcStateFloat` 自行加 `FloatComponent`；client `dialogue_group` state 按需加 `C_NpcDialogueGroupComponent`。
- **npc-15 (npc_const)**：所有 `NpcConst.Enum_State` (Teleport/Stealth/PaoPao/Dialogue/DialogueGroup/PerformanceChangeHead/Destroy/Move) 与 `NpcConst.Enum_Mail_Type.*` (60+ 类型) 是状态机切换/邮件路由的字符串/枚举常量来源。
- **npc-10 (custom_task)**：`NpcStateMove` 通过 `WayPointData:get_move_task()` 获得 `npc_move_*_task`，再 `state_exec:fire_task` 挂到 state_flow；`NpcStateChangeHead` 通过 `__skill_strategy_classes` 挂 `NpcChangeheadCharmStrategy/NpcChangeheadEnchantedStrategy/NpcChangeheadIronBallStrategy`。
- **npc-02 (active_object)**：`MailSwitcher:case` / `:copy` / `:copy_handler_item` 是状态机切 handler 的核心机制；`__active_object:stash` + `unstash_all()` 实现状态间消息保留。
- **npc-03 (state_flow)**：`StateFlowConst.Enum_ReservedStateId.{source,parent}` 是 transition target 的特殊值；`is_selector=true` 节点（`interact`、`performance`）由 selector 调度子节点。
