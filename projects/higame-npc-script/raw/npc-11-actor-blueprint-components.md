---
id: npc-11
title: Actor 级 BP 组件 + ChangeHead Attractor
sources:
  - Content/Script/actors/common/components/npc/new_npc_klutz_component.lua
  - Content/Script/actors/common/components/npc/npc_appearance.lua
  - Content/Script/actors/common/components/npc/npc_behavior_component.lua
  - Content/Script/actors/common/components/npc/npc_camera_component.lua
  - Content/Script/actors/common/components/npc/npc_hero_glue.lua
  - Content/Script/actors/common/components/npc/npc_intention_component.lua
  - Content/Script/actors/common/components/npc/npc_locomotion_component.lua
  - Content/Script/actors/common/components/npc/npc_mission_component.lua
  - Content/Script/actors/common/components/npc/npc_state_machine_component.lua
  - Content/Script/actors/common/components/npc/npc_time_control_component.lua
  - Content/Script/actors/common/changehead/npc_attractor.lua
  - Content/Script/actors/common/changehead/npc_attractor_active_object.lua
  - Content/Script/actors/common/changehead/npc_attractor_const.lua
updated: 2026-05-11
---

# npc-11 — Actor 级 BP 组件 + ChangeHead Attractor

## Sources

见 frontmatter `sources`:`actors/common/components/npc/` 下 9 个 BP 组件 lua(短文件如
`npc_appearance.lua` 24 行 / `npc_camera_component.lua` 55 行 / `npc_hero_glue.lua` 16 行 /
`npc_intention_component.lua` 47 行 / `npc_locomotion_component.lua` 43 行均全文读;长文件
如 `new_npc_klutz` / `npc_behavior` / `npc_mission` / `npc_state_machine` / `npc_time_control`
扫读首 80 行)+ `actors/common/changehead/` 下 3 个 Attractor lua(`npc_attractor_const.lua`
11 行全文,其余 80 行)。

## 1. 概念

`actors/common/components/npc/` 下的 lua 文件**不是 npc-06 描述的那 15 个 NodeComponent**,
而是 **UnLua 绑定的 Blueprint Component 脚本**:每个 lua 文件顶部都带
`---@type BP_xxx_C / xxx_C` 注解,关联到一个具体的 `UActorComponent` 蓝图(例如
`BP_NpcAppearance_C`、`NpcBehaviorComponent_C`、`BP_NpcStateMachineComponent_C` 等),
并通过 `Component(ComponentBase)` / `Component(AppearanceBase)` 工厂返回一个 lua table。
运行时 `self` 直接是该 BP 组件实例本身——可以访问 BP 暴露的字段(`self.actor`、`self.Mesh`、
`self.MovementData`、`self.Intention`、Multicast Delegate `self.EventOnXxx:Broadcast(...)`
等)、可以重写 `Initialize/ReceiveBeginPlay/ReceiveEndPlay/ReceiveTick` 这套 UnLua
生命周期 hook。这与 npc-06 描述的 Lua NodeComponent(继承 `npc_node_component_base`,
由 `add_component(BindingKey)` 挂在 NodeHandle 上,纯 lua 软件层)是**两套并行的组件体系**:
BP 组件挂在 Actor(典型如 `BPA_NPCBase_C`)的 ComponentTree 上、由蓝图直接 `attach`;
NodeComponent 挂在 NodeHandle 上、由 `npc_node_component_factory` 按 BindingKey 实例化
(见 npc-04 / npc-06)。

## 2. 9 + 1 + 3 = 13 个文件总表

| Lua 文件 | `---@type` 注解 / 类 | 一句话职责 | 关联的 NodeComponent / C++ 侧 |
|---|---|---|---|
| `new_npc_klutz_component.lua` | `BP_NPCKlutzComponent_C` | "红绿灯关卡" Klutz NPC: 5 态 NPCState + BT + 黑板初始化 | 与 `npc/npc_klutz_task.lua` 任务侧配合 |
| `npc_appearance.lua` | `BP_NpcAppearance_C` (extends `AppearanceBase`) | 监听 `Mesh.OnAnimInitialized`,空 hook 由子类/BP 填 | 与 `s/c_npc_anim_ctrl_component` 平行 |
| `npc_behavior_component.lua` | `NpcBehaviorComponent_C` | 蒙太奇监测 / `IdleMontage` 自播 / Billboard SelfTalking / 互动 UI | 与 `s/c_npc_perform_component` 形态相近 |
| `npc_camera_component.lua` | `BP_NPC_CameraComponent_C` | NPC 交互特写相机 `SetCameraPos→CreateNpcCamera` | 不同于 C++ `HiDialogueCameraControllerComp` |
| `npc_hero_glue.lua` | (无注解,几乎空文件) | 占位 stub:require 工具后空返回 | — |
| `npc_intention_component.lua` | `BP_NpcIntentionComponent_C` | NPC 意图 3 态 (`Normal/Mission/Battle`) + OnRep | 同名陷阱:与 npc-06 BindingKey `npc_intention` 不是同一物件 |
| `npc_locomotion_component.lua` | `BP_NpcLocomotionComponent_C` (extends `AppearanceBase`) | 把 BP `FWalk/Run/SprintSpeed` 落到 `FHiMovementSettings` | C++ `HiNPCMovementComponent` / `HiNpcLocomotionAppearance` 上层 |
| `npc_mission_component.lua` | `BP_NpcMissionComponent_C` | 跟踪/护送任务 + AIPerception Sight + Waypoint 到达 | `mission/npc_interact_item` 模块 |
| `npc_state_machine_component.lua` | `BP_NpcStateMachineComponent_C` | 服务端 AI 4 态 SM (`Idle/Walk/Dialogue/Stress`) 全连通 | 不同于 NpcActiveObject (npc-02/05) 的 mailbox SM |
| `npc_time_control_component.lua` | `NpcTimeControlComponent_C` | 跟随 `WorldTimeComponent.Hour` 切换 `time_<H>` action list | 与 `s_npc_task_scheduler_component` 互补 |
| `changehead/npc_attractor.lua` | `BP_NPCAttractor` (extends `BP_PO_Base`) | 独立 Actor:吸引 MassAI Crowd + 触发 Kittens NPC 表演 | — |
| `changehead/npc_attractor_active_object.lua` | `NpcAttractorActiveObject : ActiveObject` | Attractor 1:1 Kittens AO + MailSwitcher | 与 `NpcActiveObject` 模式同构 (npc-02) |
| `changehead/npc_attractor_const.lua` | (纯常量) | `Enum_Mail_Type = {POClaim, POOccupy, POClaimAndOccupy, POFree}` | — |

> 注:此目录还存在 `dialogue_component.lua`、`visibility_management_component.lua` 两个
> 文件,本笔记任务范围聚焦上面 9 个 NPC BP 组件,后两者留给 npc-13 / npc-14 处理。

## 3. 各组件详解

### 3.1 `new_npc_klutz_component` ⇒ `BP_NPCKlutzComponent_C`
"红绿灯/踉跄关卡"Klutz mini-game 用。模块内本地枚举:

```lua
local NPCState = { MOVING_TO_SAFE=1, HIDING=2, DEAD=3, IDLE=4, RUSH_TO_COVER=5 }
```

`Initialize(...)` 调 `Super.Initialize`,新建 `CurrentRoundData / NPCState=IDLE / Config`,
`InitializeMovementVars` 准备 `CurrentSpeed / TargetLocation / CurrentCover / bIsSafe /
InitialTransform / PathPoints={} / CurrentPathIndex=1`。`GameStart(Restart)` 直接调 BP
函数 `ExecuteBehaviorTree(Restart)` 启动行为树。`OnNewRound(roundConfig)` 把 `RoundConfig`
存到 `self.Config.RoundConfig`,首次保存 `InitialTransform = self.NPC:GetTransform()`,然后
`UpdateBalckboardInfo(GreenLightSpeed, EndZone:K2_GetActorLocation(), SafeBoxTag, true)`
写入黑板,`UpdateLightState(1, GreenLightSpeed, GreenTime, YellowTime, RedTime)` 启动红绿
灯节奏。`LoadMoveParams()` 默认 `GreenLightSpeed=400, YellowLightSpeed=500`(注释中保留
`Acceleration/Deceleration/TurnSpeed/RandomFactor` 备选)。— 与 `npc/npc_klutz_task.lua`
(NodeHandle 任务侧)是**两层**:此处驱动单只 NPC,后者管理整轮任务。

### 3.2 `npc_appearance` ⇒ `BP_NpcAppearance_C`
仅 24 行,继承 `actors.common.components.common_appearance` 的 `AppearanceBase`。
`ReceiveBeginPlay` 调 `OnAnimInitialized()` 一次后再监听 `self.actor.Mesh.OnAnimInitialized:Add(self, self.OnAnimInitialized)`。`OnAnimInitialized()` 自身函数体为空——子类或 BP override
才有实质行为。还 require 了 `state_conflict_data` 但本文件未使用。意味着 NPC 走通用外观体系
(换装/挂件/蒙皮),具体实现下沉到父类 `common_appearance`(本笔记不展开)。

### 3.3 `npc_behavior_component` ⇒ `NpcBehaviorComponent_C`
非 GAS 行为壳,require `mission.npc_interact_item`、`interact_base_data`、`npc_base_data`、
`blueprint_const`。`AnyMontagePlaying()` 三处检查:`AnimInstance:Montage_IsPlaying(nil)` /
`Mesh:GetLinkedAnimGraphInstanceByTag("Locomotion"):Cast(UE.UHiLocomotionAnimInstance)` /
Tag `"Template"` 同 cast。`ReceiveBeginPlay` 服务端时 `BillboardComponent:EnableSelfTalking()`;
客户端检查 `AnimMode == AnimationBlueprint` 且无蒙太奇时自播 `IdleMontage`。状态字段
`bShowingInteractUI / DelayResumeInteractUITimer / bIsInSing`。**不是 BehaviorTree 组件**——
BT 由 npc-14 `HiAIController` 通道驱动,本组件管理动画与互动 UI 的可见态。

### 3.4 `npc_camera_component` ⇒ `BP_NPC_CameraComponent_C`
仅 55 行。唯一公开 API `SetCameraPos(Actor, Position)`:把 Position 偏移
`-GameConstData.NPC_INTERACT_CAMERA_DELTA_Z.IntValue`(Z 下移),交给 BP 函数
`CreateNpcCamera(targrt_pos, Actor)`。文件其余代码全部注释掉了 — 历史上有
`LGUIUpdaterClass + ApplyCustomViewUpdater + DisableInput` 流程及配套 `RemoveNpcCamera`,
目前线上保留极简包装。**与 C++ `HiDialogueCameraControllerComp` 不同层**:那是对话期间
的全局相机控制器(c++ ActorComponent),此处只是 NPC 自己的"相机预设挂点"。

### 3.5 `npc_hero_glue`(无 BP 注解,文件几乎空)
最小文件:require `componentbase`/`component`/`utils`/`state_conflict_data`/`SkillUtils` 与
`InputModes`、`CustomMovementModes` 之后,`Component(ComponentBase)` 写完就**直接 `return`**,
没有任何方法实现。是为"NPC ↔ Hero 粘合"预留的扩展点(命名暗示 Avatar 与 NPC 间互动事件
路由),目前是 stub。

### 3.6 `npc_intention_component` ⇒ `BP_NpcIntentionComponent_C`
NPC "意图" 三态枚举(`Enum.BPE_NpcIntentionType.{Normal, Mission, Battle}`)轻量状态机:
查询接口 `InNormal / InMission / InBattle`;服务器接口 `ChangeIntention(NewIntension)` 写
本地 + `EventOnIntentionChange:Broadcast`;`OnRep_Intention` 客户端复制后再次 `Broadcast`,
前端订阅一致。注意 npc-06 列表里 BindingKey `npc_intention` 也叫这个名 — 是同名异源
(BP 组件 vs Lua NodeComponent),**不应混淆**:本组件在 BP ComponentTree 里,NodeComponent
在 NodeHandle 上。比 `npc_state_machine_component` 维度更高。

### 3.7 `npc_locomotion_component` ⇒ `BP_NpcLocomotionComponent_C`
**奇特点**:实现继承 `AppearanceBase`(与 `npc_appearance.lua` 同基类),职责却是移动。
`ReceiveBeginPlay` 调 `SetNPCSpeedSettings()`:从 BP 暴露字段 `FWalkSpeed/FRunSpeed/FSprintSpeed`
(均为 lua-visible 可在 BP 调速,值 ≠ 0 视为有效),拼装 `UE.FHiMovementSettings`,改后
赋给 `self.MovementData`(BP 复制属性)。再调 `OnAnimInitialized()` 并监听同名 Mesh 事件。
**与 C++ `HiNPCMovementComponent` 不同层**:C++ 那个是 `UCharacterMovementComponent`
派生类、提供真实物理移动;本组件只是把 BP 编辑器里设置的速度参数注入到 `MovementData`
这个 struct,供 C++ 上层读取。

### 3.8 `npc_mission_component` ⇒ `BP_NpcMissionComponent_C`
任务驱动 NPC 行为(跟踪/护送)。模块内 `MissionNPCOperateType = {StartNPCMove=1,
StopNPCTracking=2, StopNPCEscort=3, ReSetNPC=4}`。`Initialize` 初始化
`StartNPCTracking/StartNPCEscort/IsMoving=false`、`bHasBeenInitialized=false`。
`ReceiveBeginPlay` 拿 `self.NPC = self:GetOwner()`,借 `bHasBeenInitialized` 跨 BeginPlay
持久化标记,如已初始化则发 `ReSetNPC`(空 Param JSON)给 BP 函数 `CallNPCMission`。
`HandleSight(Actor, Stimulus: FAIStimulus)`:订阅 AISense_Sight,`StartNPCEscort &&
!EscortDetected && bSuccessfullySensed` 时遍历 `Actor.Tags` 找 `Configs.AvatarTag`,命中
`StartEscortNPC()` + `EscortDetected=true` 去重。`HandleEvent_ReachWayPointIndex`:跟踪模式
下到达 `TrackingConfig.TargetIndex` 时 `Multicast_StopNPCTracking(true)`。注释还提到
AIPerception 单一检测半径限制(TODO 多 Sense_Sight 拆分)。

### 3.9 `npc_state_machine_component` ⇒ `BP_NpcStateMachineComponent_C`
**仅在 server 上跑**(`if self:GetOwner():IsServer() then ...`)。基于 `common.utils.base_state_machine`。
4 状态:`Enum.BPE_NpcAiState.{Idle, Walk, Dialogue, Stress}`(注意:与 npc-05 的
`NpcStateComponent` 主状态枚举体系**不同**)。转移**全连通**(任意两态可互换);
`SetStartState(Idle)` + `Start()`。三 hook `HandleStateEnter/Leave/Update` 分别
`EventOnStateEnter/Leave/Update:Broadcast(StateName)`,让 BP 侧订阅。`ChangeToPrevState()`
走 `StateMachine:GetPrevState()`;`ChangeState(StateName)` 直接转。`ReceiveEndPlay` 服务端
时 `StateMachine:Destroy()`。**与 npc-05 `NpcStateComponent` 关系**:不是同一物件。
NpcStateComponent 是 Lua 主状态机(挂在 `server_logic_handle` 上,主导
`npc_lifecycle/idle/move/dialogue/stress/...`);此处 BP 组件是更轻量的"AI 子状态机"——
4 态 BPE 枚举,广播给前端做表现切换。两者**平行存在**,在业务层通过事件订阅桥接。

### 3.10 `npc_time_control_component` ⇒ `NpcTimeControlComponent_C`
仅 server 生效。`ReceiveBeginPlay` 读 `NpcBaseData[GetNpcId()].Time_Control` 拿 `TimeControlId`
(无则 return),通过 `UE.UGameplayStatics.GetGameState(self).WorldTimeComponent:GetHourOfDay()`
拿当前小时,调 `GetStartActionList(TimeControlId, Hour)` 在 `NpcTimeControlData[TimeControlId]`
表里查 `time_<Hour>` 字段;查不到就向前回环 `(CurHour - 1 + 24) % 24`,直到找到或转完一圈
返回 nil。命中则 `Actions = UE.TArray(1)` 装 `ActionId`,`EventOnChangeActions:Broadcast(Actions)`
派出去给业务层(EventFlow 等)消费。再订阅
`WorldTimeComponent.EventOnWorldTimeHourChanged:Add(self, HandleHourChange)` 持续触发。

## 4. ChangeHead Attractor 系统

### 4.1 `npc_attractor.lua` ⇒ `BP_NPCAttractor`(extends `BP_PO_Base`)
**独立 Actor**,继承 `BP_PO_Base`(自带 PO/SmartObject 能力),与具体机关解耦
(`@AUTHOR dougzhang @DATE 2026-03-31`)。两条职责:**客户端 / Standalone** 通过 PO 或
Direct 模式吸引 MassAI Crowd NPC 视觉表现;**DS 侧 / Standalone** 通过
`MutableActorSubSystem` 查找附近 Kittens NPC,触发 EventFlow 表演。关键常量与默认配置:

```lua
local AttractMode = { PO = "PO", Direct = "Direct", Both = "Both" }
local DEFAULT_CONFIG = {
    Mode             = "PO",
    SearchRadius     = 5000.0,
    MaxDuration      = 10.0,
    TickInterval     = 0.033,
    ArriveRadius     = 200.0,
    MoveSpeed        = 800.0,        -- MassAI NPC 移动速度 (cm/s)
    SpreadRadius     = 200.0,
    MaxCount         = 0,
    AutoDestroy      = false,
    NpcAttractRadius = NpcAttractorUtils.DEFAULT_NPC_ATTRACT_RADIUS,
    PODefinitionPath = nil,          -- nil 用 SO_Attractor_NoSpawn
    PODuration       = nil,          -- nil 用 POLifecycleManager.DEFAULT_DURATION
}
```

依赖:`NpcAttractorUtils` / `POLifecycleManager` / `SwapHeadSlotUtils`(均在
`CommonScript.actors.interactable.SwapHead.Utils`)。本文件用 `Class()` 定义类
(npc-01 风格的 BP 绑定),`UserConstructionScript` 仅打日志。

网络辅助:`KSL = UE.UKismetSystemLibrary`、`CanRunMassAI(actor) = not KSL.IsDedicatedServer(actor)`、
`CanRunKittens(actor) = true`(由 `AttractNearbyKittensNPC` 内部 nil-check 兜底)、
`IsValidClaimHandle = UE.USmartObjectBlueprintFunctionLibrary.IsValidSmartObjectClaimHandle`
(值类型 struct,**不能用 nil/truthiness 判断**)。

### 4.2 `npc_attractor_active_object.lua`(`NpcAttractorActiveObject : ActiveObject`)
**1:1 ActiveObject**,与 `BP_NPCAttractor` 一一对应,模式与 NpcActiveObject(npc-02)
完全一致:`Kittens.class('NpcAttractorActiveObject', ActiveObject)`。`initialize` 保存
`__owner_actor=_context`、`__claim_handles={}`(npc_actor_id → ClaimHandle 表);新建
`MailSwitcher` 注册 4 个 case → `_handler_*`(`POClaim/POOccupy/POClaimAndOccupy/POFree`),
`become(__mail_switcher, true)` 立即激活。`get_owner_actor()` 返回 `__owner_actor`。
`_get_slot_or_actor_transform(attractor, npc_key, claim_handle, npc_z)` 优先链
**NavPointWorldTransform > SlotTransform > ActorTransform**:`POComp:GetClaimedSlotNavPointWorldTransform`
→ `POComp:GetClaimedSlotTransform` → 兜底 `attractor:GetTransform()`。通过 MailSwitcher
与外部(机关 / 头能力)解耦:消息发到 AO,AO 根据 mail type 路由到 owner Actor 上的 PO 操作。

### 4.3 `npc_attractor_const.lua`(11 行)

```lua
local Const = {}
Const.Enum_Mail_Type = {
    POClaim = 'POClaim',
    POOccupy = 'POOccupy',
    POClaimAndOccupy = 'POClaimAndOccupy',
    POFree = 'POFree',
}
return Const
```

对应 4 个 SmartObject Slot 操作语义:单纯申领 / 占据 / 申领并占据 / 释放。供
`npc_attractor.lua` 与 `npc_attractor_active_object.lua` 共享 Mail Type 字符串常量,
避免拼写漂移。

## 5. 解决 Open Question Q2:Actor 级 BP 组件 vs Lua NodeComponent

| 维度 | Actor 级 BP 组件(本页 9 个) | Lua NodeComponent(npc-06 的 15 个) |
|---|---|---|
| 物理形态 | UE 蓝图 `UActorComponent`(BP 类) | 纯 Lua 表(挂在 `NodeHandle.components`) |
| 绑定方式 | UnLua `---@type BP_Xxx_C` + 蓝图 attach | `node:add_component(BindingKey, ...)` |
| 注册粒度 | 蓝图编辑器里挂在 `BPA_NPCBase_C` 上 | 数据驱动(BindingKey 字符串注册) |
| 生命周期 | UE Actor 引擎驱动(`Initialize/ReceiveBeginPlay/EndPlay/Tick`) | NodeHandle 调度(由 npc-04 统一驱动 `init/start/stop/tick`) |
| 网络复制 | 走蓝图属性 + OnRep + Multicast Delegate | Server/Client 双侧分别实现(`s_*` / `c_*`),靠 npc-04 通道 |
| BP 可见性 | **能**(就在 NPC Actor ComponentTree 上) | **不能**(纯 Lua 软件层) |
| 数量 | 9 个(含 1 stub) + Attractor 子系统(3 个) | 15 组件 / 17 BindingKey |
| 典型用例 | 动画/移动/相机/任务/状态机/时间控制等 BP-friendly 功能 | Dialogue/Voice/LookAt/Fade/Teleport/Float 等"软件层"NPC 行为单元 |

**关系总结**:两套并行系统,不互相替代。BP 组件偏向"Actor 物理骨架 + 引擎绑定";
NodeComponent 偏向"lua 行为编排 + Kittens AO 状态流"。**典型桥接路径**:
`npc_state_machine_component`(BP 组件,server 4 态 AI SM)广播 `EventOnStateEnter` →
业务层接住 → 调用 `NpcStateComponent`(npc-05 Lua 主状态机)上的 push/pop。**同名陷阱**:
`npc_intention_component` 既出现在 BP 组件目录(`BP_NpcIntentionComponent_C`)又出现在
npc-06 BindingKey `npc_intention`,**不是同一物件**。NpcActiveObject 取
`server_logic_handle`(npc-05)拿到的是 NodeComponent 树(`s_npc_state_component` 上挂
的 handle),与本页 BP 组件**没有 owner/handle 直接耦合**。Attractor 子系统在两者之外另起
一支:独立 Actor(`BP_NPCAttractor`)+ 自带 1:1 Kittens AO,仅通过 EventFlow 与 Kittens
NPC AO 间接交互。

## 6. 跨页关联

- **npc-02 / npc-05** — NpcActiveObject 模型;`npc_attractor_active_object` 用同样的
  `Kittens.class('X', ActiveObject) + MailSwitcher` 模式;`server_logic_handle` 来自
  NodeComponent 而非本页 BP 组件
- **npc-04** — NodeHandle 与 BindingKey 体系,对照本页 BP 组件的"Actor-Component"挂载方式
- **npc-06** — 15 个 Lua NodeComponent 矩阵,与本页 9 个 BP 组件互为表里;两组同名陷阱
  (`npc_intention`、`npc_klutz_task`)需小心区分
- **npc-07 / npc-08** — `npc_attractor:Activate` 通过 `EventFlowId` 驱动 Kittens NPC 表演
- **npc-09** — Attractor 复用 `LQTMassPO` + `SmartObjectClaimHandle` + Slot Transform 体系
- **npc-12**(待写) — Attractor 在 ChangeHead 表演里的角色(PO/Direct/Both 三模式)
- **npc-13**(待写) — C++ 侧 NPC 组件:`HiNPCMovementComponent` /
  `HiNpcLocomotionAppearance` / `HiDialogueCameraControllerComp` /
  `UHiLocomotionAnimInstance`;`npc_locomotion_component` 把 BP 速度落到 `FHiMovementSettings`,
  `npc_camera_component` 是简化 lua 版相机
- **npc-14**(待写) — `npc_behavior_component` 不是 BT(BT 在 `HiAIController` 通道),
  但 `new_npc_klutz_component:ExecuteBehaviorTree` 是 BT 入口;`npc_state_machine_component`
  4 态 AI SM 是另一条 AI 控制流

## 7. 待确认 / TODO

- `npc_hero_glue.lua` 几乎为空,是否在别处通过 hot-add method 注入?需 Grep 确认
- `npc_camera_component:CreateNpcCamera` 在 BP 侧的实现位置
- `BP_NPCAttractor` 完整生命周期与 `POLifecycleManager` 细节留给 npc-12 展开
- 目录还存 `dialogue_component.lua`、`visibility_management_component.lua`,本任务范围外
