---
id: 56
title: "HiGame Interactable 基类三件套与用例集"
source: local-archaeology
project_root: E:\HiProject\PerforceDev\UnrealEngine\Projects\HiGame
modules:
  - Content/Script/actors/common/interactable/
  - Content/Script/CommonScript/actors/interactable/
fetched: 2026-05-11
status: content-verified
covers_questions: [Q11, Q12, Q13]
---

# Interactable 基类与用例集

## 一、基类三件套

### base_item.lua

`BaseItem = Class(ActorBase)`，对应蓝图 `BP_BaseItem_C`。`---@class BaseItem: ActorCommon, BP_BaseItem_C`。

**核心字段**（Initialize 处 rawset）：

| 字段 | 语义 |
|---|---|
| CompFuncNames | `{{"IsVisible","SetVisibility"},{"GetCollisionEnabled","SetCollisionEnabled"}}` —— 反射对子，给 StatusFlow_Appear_StoreProperty 存盘组件可见性/碰撞 |
| CachedActorID | 缓存通过 MutableActorComponent:GetActorID() 解析的 ActorID（EditorID = "ActorID@GroupID"） |
| JsonObject | 从场景 JSON 反序列化的原始数据 |
| ActorIdList | `{ActorId=bIsReady}` 字典；MergeToActorIdList 合并外部 ID，CheckChildReady 全 ready 后置 bReady=true |
| ReadyActorIdList | UE TArray 的副本（蓝图属性，方便 RepNotify 同步） |
| CompIdList | `{CompName={ActorId=bReady}}` 子组件维度 ready 表 |
| CompFuncPropertys | StatusFlow_Appear 存盘前的可见性/碰撞快照 |
| MainActor | 主控 Actor 反向引用（多碎片 → 宝箱模式） |

**bNewStatusFW + SetAdvancedLogicalChains**（base_item.lua:543-555 + UCS:71）：

`bNewStatusFW` 是蓝图属性。`UserConstructionScript` 调 `SetAdvancedLogicalChains(self.bNewStatusFW)` 把布尔写进 `ItemStatusComponent.bAdvancedLogicalChains`。后续大量分支 `if self:GetAdvancedLogicalChains() then ... else ItemStatusComponent:Call_StatusFlowRaw(...) end`：
- **新框架** 走 `LogicalState:K2_SetState` / `OnLogicalStateChanged` / `BP_Mission_CallStatusFlow`
- **旧框架** 走 `ItemStatusComponent:Call_StatusFlowRaw`

**存盘**：`EntityPropertyMessageName = "BaseItem"`；K2_OnLoadFromDatabaseAllFinish 与 bHasSaved 标志位配合恢复状态。

### interacted_item.lua

`InteractedItem = Class(BaseItem)`，对应 `BP_InteractedItem_C`，新增 `InteractedItemComponent` 字段。`EntityPropertyMessageName = "InteractedItem"`。

**触发链路 QuerySystem → OnQueryActionStartDetected → OnBeginOverlapNew**：

```
ReceiveBeginPlay (Client only)
  └─ self.QuerySystem.OnQueryActionStartDetected:Add(self, OnBeginOverlapNew)
  └─ self.QuerySystem.OnQueryActionStopDetected :Add(self, OnEndOverlapNew)
OnBeginOverlapNew(QuerySystemData)
  └─ self.InteractedItemComponent:OnBeginOverlap(QuerySystemData, true)
  └─ self:OnBeginOverlap(nil, OtherActor, ...) -- 子类 Hook
```

QuerySystem 是 **Client-only**，HasAuthority Server 不订阅。旧版 BoxComponent.OnComponentBeginOverlap（如 SealedDoor、JiaSuHuan）走另一条路绕过 QuerySystem。

**ConditionMonitorComponent / GatherConditionIDList**：ReceivePostBeginPlay Client 等 OnClientReady 后调，把 self.ConditionIDList + sUIMulti[i].ConditionIDList 合并去重，注册到 PlayerState.ConditionMonitorComponent:Server_AddConditionID(IDArry)。

**sUIMulti**：BP 端 `TArray<F_UIMulti>`，每项含 ConditionIDList。同一交互物可挂多套 UI（如电话亭：有卡 → 拨号；无卡 → 独白）。

**核心入口分发**：`TriggerInteractedItem / Server_ReceiveDamage / DoClientInteractAction / DoServerInteractEndAction / DoClientInteractActionWithLocation` 全部转发给 InteractedItemComponent，bNeedHandleReconnected 时缓存 InteractPlayerStateGid 用于断线重连。

### placeable_item.lua

`PlaceableItem = Class(InteractedItem)`，对应 `BP_PlaceableItem`。`EntityPropertyMessageName = "PlaceableItem"`。

**可放置物件字段**：

| 字段 | 语义 |
|---|---|
| PlaceableParam.bSlotPlaced | 优先用 Slot（台座、插座）放置 |
| PlaceableParam.SlotCheckRadius | Slot 检测球半径 |
| PlaceableParam.SlotActorClass | 合法 Slot 的 SoftClass |
| PlaceableParam.bSlotOnly | Slot 找不到时是否回退到 Trace 平面放置 |
| PlaceableParam.bAttachToTarget | 放置后是否 K2_AttachToActor 到目标 |
| PlaceableParam.bPlaceableInRange | 限制只能在原始位置周围 PlaceableRange 内放置 |
| OriginalLocation/Rotation | 服务端记录的初始 Transform |
| bPlaced/bPickedUp | 放置/拾取标志 |
| AttachedTargetID | Attach 父对象的 ActorID，重连后仍能 reattach |

**核心算法**：TryGetPlaceablePosition → GetSlotPlaceLocationAndRotation（Slot 路径）或 GetTracedPlaceLocationAndRotation（射线检测路径，前向射线 + 多个 Checker SceneComponent 纵向射线 + PartsTraceCheckFunction 同水平面校验 + CheckPartsConnectable 连通性）。

### base_ghost.lua

`BaseGhost = Class(editor_character)`，**不在 BaseItem 链路上**。Ghost 是 DDS 架构下"代理 Actor"概念。

**字段**：
- `BP_LogicalSignalGenerator/Receiver`：新状态框架下"逻辑信号生成器/接收器"组件，bNewStatusFW=true 时才 RegisterComponent
- `NPCMoveViaPoint`：路点移动组件
- `EntityPropertyMessageName = "BPA_GhostBase"`

**存盘**：仅走 `ItemStatusComponent:Call_StatusFlowRaw` 旧框架（不分新旧）。

### ui_interact.lua

UMG Widget 的 Lua 基类，仅 27 行：Construct → BindDelegate 把 btn_interact OnPressed/OnReleased 绑到 OnPressed_ButtonInteract / OnReleased_ButtonInteract（默认空实现）。本基类不直接调 InteractSystem。

## 继承关系图

```
ActorCommon (CommonScript/actors/actor_common.lua)
 ├── Actor (Client/Server actors/actor.lua)
 │    ├── BaseItem (BP_BaseItem_C, EntityPropertyMessageName="BaseItem")
 │    │     └── InteractedItem (BP_InteractedItem_C, "InteractedItem")
 │    │           ├── PlaceableItem (BP_PlaceableItem, "PlaceableItem")
 │    │           ├── automatic_door_base / base_door
 │    │           ├── elevatorBase ── elevator (EntityPropertyMessageName="ElevatorBase")
 │    │           │     └─ foodfactoryelevator/elevatorSplineImpl
 │    │           │              └── BBLElevator
 │    │           ├── BP_MachineDoor
 │    │           ├── BP_Album / BP_TelephoneBooth
 │    │           ├── note (ABT_Note)
 │    │           ├── BP_Interacted_Bottle
 │    │           ├── BP_Interacted_PickedItem
 │    │           │     └── seal_fragment (PumpkinGame)
 │    │           ├── chest/base/base_chest
 │    │           │     ├── middle_chest ── higher_chest
 │    │           │     └── multi_chest
 │    │           └── DreamRebuild ...
 │    │     ├── BP_SealedDoor (跳过 InteractedItem)
 │    │     ├── JiaSuHuan ── BP_AccelerationRing
 │    │     ├── faithhourglass
 │    │     └── pumpkin_game_manager
 │    └── editor_character
 │          └── BaseGhost (EntityPropertyMessageName="BPA_GhostBase")
 │                └── 各种 Ghost (BPA_*)
```

## EntityPropertyMessageName 用途

字符串常量，**不在 base_item.lua 内部使用**。DDS Entity 框架（C++ 侧）用来路由"属性同步消息"的标识：当 RealActor ↔ Ghost ↔ RO 之间 RPC/属性广播时，C++ 用此名字把消息打散到对应类别处理器上。子类只需重写常量即可在不改路由代码下加新 Entity 类型。

## 二、用例集 A — 门、电梯、平台

### Door/automatic_door_base.lua（自动门）
- 继承：`BaseItem`（直接，不经 InteractedItem——靠 Box 触发器自动开门，无需点 E）
- 关键字段：Timeline/Duration、Door_Left/Right、DoorType ∈ {SlidingDoor, SwingDoor}、bMoving / bControlledByFlow / CloseTime
- 触发：AllChildReadyClient → BindTriggerActor 把 TriggerActor.OnActorBeginOverlap 接到 OnBeginOverlap；玩家进入 → 计算 Forward·(DoorLocation-PlayerLocation)，PlayTimeline + Call_StatusFlowRawToServer(Active)
- 完成：Timeline 走完 OnMoveFinish → CloseTime 定时器；离开触发器 → ReverseTimeline + Inactive
- 存盘：BaseItem 链（旧框架）

### Door/base_door.lua（通用基础门）
- 继承：`InteractedItem`
- 实现：仅 `local M = Class(ActorBase); return M`，**纯继承无 Lua 实现**。所有逻辑在 BP_BaseDoor_C 蓝图

### MachineDoor/BP_MachineDoor.lua（机械锁门）
- 继承：`InteractedItem`
- 关键字段：Box、MachineLockInit、bClaimed、MoveLocation
- 触发：ReceiveBeginPlay Server 调 MachineLockInit 强制 SetInteractable(UnInteractable)；钥匙解锁后才走 InteractedItemComponent 标准 E 键链

### SealedDoor/BP_SealedDoor.lua（密封门）
- 继承：`BaseItem`（不被点击）
- 关键字段：Box、width/height（运行时 SetBoxExtent(width,height,curZ)）
- 触发：状态机驱动（任务/区域触发把它从 Sealed → Active）；走 BaseItem.StatusFlow_Appear

### MiaoQuDoor / GrottoFluvialDoor
- 目录存在但**无 .lua 文件**——纯蓝图实现

### elevator/elevatorBase.lua + elevator.lua（标准电梯）
- 继承：均 `InteractedItem`
- EntityPropertyMessageName = "ElevatorBase"
- 关键字段：ElevatorID、offsetZ=-100、Spline、SwitchArray、CurIndex/TargetIndex、Event_AfterDoAction
- 触发：Multicast_ReceiveDamage_RPC（玩家点击/伤害即触发）
- 任务对接：AllInOneEventRegister_ElevatorReachIndex / AllInOneAction_ElevatorContinueMove

### BBLElevator/BP_BBLElevator.lua
- 继承：foodfactoryelevator.elevatorSplineImpl → 间接 InteractedItem
- 关键字段：NiagaraLocation、ElevatorStatus = {Idle=0, Rise=1, Descend=2}（Lua local table）

### foodfactoryelevator/（5 文件）
- elevator.lua / elevator_switch.lua / elevatorMontage.lua + elevatorMontageImpl.lua / elevatorSplineImpl.lua
- Impl 文件是策略模式的"运动方式实现层"

### TeleportElevator / JumpItem
- 目录**无 .lua 文件**

### AccelerationRing/BP_AccelerationRing.lua + JiaSuHuan.lua（加速环）
- 继承：`AccelerationRing = Class(JiaSuHuan)`；`JiaSuHuan = Class(BaseItem)`
- 关键字段：Box、InHiCharacter、BufferVelocity
- 触发：Box.OnComponentBeginOverlap → OnActorEnter，强转 HiCharacter，再访问 DodgeComponent / GlideComponent / InteractionComponent

## 三、用例集 B — 容器、物品、机械谜题

### chest/base 三档（middle_chest / higher_chest / multi_chest）
- 继承链：HigherChest → MiddleChest → base_chest；MultiChest → base_chest
- base_chest 双套：Client/Server 各自继承 CommonScript.actors.common.interactable.chest.base.base_chest
- 关键字段：VATPlayer、Niagara_Open、NS_GoldenLight、VATAnimIndex
- 触发：MultiChest:DoInteractShow 播 VAT + AkAudio + Niagara + Client_RemoveInitationScreenUI
- 奖励：Server 通过 RewardUtils + ChestContentUtil 发放

### CombinationLock —— 目录**无 .lua 文件**，纯蓝图

### bottle/BP_Interacted_Bottle.lua（摇瓶子）
- 继承：`InteractedItem`
- 关键字段：Bottle_Used、SceneCaptureComponent2D、LEAVE_DISTANCE=600、TIMER_ANIM=1.27、TIMER_CARD=4.0、CardID=180102
- 玩法：玩家持续 E 摇晃，距离 > LEAVE_DISTANCE 中断
- 完成：摇够时间发奖（CardID=180102）+ Bottle_Used=true

### note/note.lua（字条 ABT_Note）
- 继承：`InteractedItem`
- 关键字段：isGet、BookCaptureComponent2D（字条内容渲染）、Sk_AbtNote + ABPNote 动画
- 玩法：交互打开 NoteWidget UMG，关闭 + 进背包

### Album/BP_Album.lua（相册）
- 继承：`InteractedItem`
- 玩法：DoServerInteractAction → Server_OnInteractItemFinished 通知任务系统；DoClientInteractActionWithLocation Client 打开相册 UI

### telephone/BP_TelephoneBooth.lua（电话亭）
- 继承：`InteractedItem`
- NO_CARD_MONOLOGUE_ID = 1009
- 玩法：DoClientInteractActionWithLocation 检查 ItemUtil.GetAllPhoneCards：有卡 → 打开 UI_InteractionTelephone；无卡 → MonoLogueUtils.GenerateMonologueData(1009) → HudMessageCenterVM
- **sUIMulti 典型用例**：两套 UI 按持卡条件切换

### Tombstone / WarChessboard —— **无 .lua 文件**，纯蓝图

### PumpkinGame/pumpkin_game_manager.lua + seal_fragment.lua（南瓜小游戏）
- 继承：`PumpkinGameManager = Class(BaseItem)`（管理器）；`SealFragment = Class(BP_Interacted_PickedItem)`（碎片）
- BP 字段：NPCActorIDs / FragmentActorIDs / BehaviorTreeAsset / PuzzleID / CatchRadius / MoveSpeed / ViewAngleThreshold
- 启动模式：① 正式：MissionFlow execute_puzzle → Manager 监听 OnPuzzleStarted → StartGame；② 调试 PuzzleID=0：AllChildReadyServer 后延迟自启动
- seal_fragment 反模式：复用 BP 但**完全覆写交互**——拾取后只隐藏 + 禁用交互 + 广播 Server_OnInteractItemFinished，**不进背包不销毁不 UnRegisterRO**

### gravityitem/gravity_item.lua（重力道具）
- 继承：`grapple_point` 子类
- 关键字段：AOIIndex（RepNotify）、NowIndex、bProcessAOI、GravityVectors、NowGravity / PreGravity、GravityModule
- 玩法：OnRep_AOIIndex → ProcessAOI 选向量 → 计算旋转 → GravityModule:K2_SetRelativeTransform

### faithhourglass/faithhourglass.lua（信仰沙漏）
- 继承：`BaseItem`
- 关键字段：faithclockcount、faithclocksActors、is_finished
- 玩法：InitFaithClockActors 读 JsonObject.sourcepath，扫目录下所有 *.json 收集"信仰钟"Actor；玩家推动各 faithclock 后 hourglass 汇总判定

### dreamrebuild/dream_rebuild.lua（梦境重建）
- 继承：`InteractedItem`
- 关键字段：DestDissolveValue、BuildingsConfig（DataTable Row Name）、AssetsTable、DISSOLVE_UPDATE_INTERVAL=0.01
- 玩法：Client 异步加载 DT_DreamRebuildBuilding 行的 Mesh + Materials，交互后调动态材质参数 Dissolve 从 0 → 1 渐变出现
- **完全 Client 表现**，Server 不参与

### TimedChallenges/timed_challenges_logic_ui.lua（限时挑战 UI 控制器）
- 普通 Lua 类（不是 Actor），通过 BPA_TimedChallenges 持有
- BPA_TimedChallenges 状态变化 callback 进来，按状态切换 HUD（开/关 ChallengesUI 与 ActorTrack）

## 四、横切观察

### 命名约定

- **BP_xxx.lua** = 1:1 对应蓝图 BP_xxx_C 的 Lua 实现脚本
- **BPA_xxx.lua** = "Blueprint Actor"，多用于较独立、自带管理性质的复合 Actor（BPA_SourceTower / BPA_TimedChallenges / BPA_GhostBase）
- **base_xxx.lua** = 基类/抽象类
- **纯小写 lua（如 note.lua）** = 早期/非严格命名规范

### RO 子目录约定

- 13 个 BP_*_RO.lua + SwapHead/ + Utils/
- 每个能被 Group Actor 管理、需要跨 Server↔Client 同步状态的可交互物，都对应一个 BP_xxx_RO.lua
- RO 类继承链与 Actor 主类继承链平行
- RO 上的 OnRep_* 回调需要"找回真实 Actor"再转发：`GetGroupActor():GetROActor(ActorId)` → 写值 → 调真 Actor 的 OnRep_*

### Utils 子目录约定

`actors/common/interactable/RO/Utils/` 文件：BaseROUtils.lua / BP_Interacted_GainItem_ROUtils.lua / BP_Interacted_PickedItem_ROUtils.lua / BP_MineralBase_ROUtils.lua / chair_seat_utils.lua

- BaseROUtils.lua 用 `_G.Class()` 定义普通 Lua 类（不是 UClass），ctor(owner) 接收 RO 作为 owner
- 模式：**组合优于继承**——RO 类持有 ROUtils 实例，把可复用工具方法（库存查询、Json 解析、奖励计算）放到 ROUtils

类似的还有 SwapHead/Utils/POLifecycleManager.lua（PlayerOwned 生命周期管理）。

### 常见反模式

1. **bNewStatusFW 双状态框架并存** —— 每个状态切换函数都要 if/else 双分支，分支翻倍
2. **复用 BP 但旁路父类销毁链** —— seal_fragment 借 BP_Interacted_PickedItem 但不进背包不销毁不 UnRegisterRO
3. **MergeToActorIdList / CheckChildReady 强依赖 EditorID** —— 跨地图、过期 ID 让 ready 永远不 fire
4. **QuerySystem 仅 Client 订阅 vs Box.OnComponentBeginOverlap 双端** —— 混用导致 Server 收不到点击
5. **OnPlayerStateReady / DoOnClientReady / OnReconnected / OnRep_InteractPlayerStateGid 四套时机** —— 少调一个就在断线重连后看不到 UI
6. **EditorID 解析手写 split** —— `utils.StrSplit(self.EditorID, "@")` 散落多处
