---
id: 60
title: "HiGame Status / RO / Database 三横切"
source: local-archaeology
project_root: E:\HiProject\PerforceDev\UnrealEngine\Projects\HiGame
modules:
  - Plugins/LogicalChains/
  - Content/Script/actors/common/interactable/{base,RO}/
  - Content/Script/CommonScript/actors/Puzzle/GhostMechanism/
fetched: 2026-05-11
status: content-verified
covers_questions: [Q7, Q8, Q9 cross]
---

# Status / RO / Database 三横切

# 主题 A：Status 状态机 + LogicalSignal

## A.1 bNewStatusFW 新旧分水岭

`bNewStatusFW` 是声明在 BP_BaseItem 蓝图上的 boolean 字段，同样字段镜像在 BP_GroupActorSpawner 与 BPA_GhostBase 上 —— 即所有需要兼容新旧状态机的 Actor 都挂着这一个布尔。

**入口**：
- base_item UCS — `BaseItem:UserConstructionScript()` `self:SetAdvancedLogicalChains(self.bNewStatusFW)`
- BPA_CharacterBaseWithStatus UCS — :32-37 同样在 UCS 灌给 ItemStatusComponent

**默认值**：从 base_item.lua 路径推断，新写的 Actor（GhostMechanism、Trap 系列）默认 bNewStatusFW=true，老 BP_BaseItem 子类历史默认 false。

| 通道 | bNewStatusFW=false（旧） | bNewStatusFW=true（新） |
|---|---|---|
| 状态存储 | ItemStatusComponent.StatusFlowRaw (S_StatusFlowRaw) | ULogicalStateComponent.CurrentState (FLogicalStateName) |
| 状态枚举 | E_StatusFlowRaw（Appear/Sealed/InActive/Active/Complete/Destroy） | LogicalStates: TArray<FName>（蓝图配置任意） |
| 同步方式 | OnRep_StatusFlowRaw + Multicast RPC | OnRep_CurrentState + OnLogicalStateChanged |
| 调用入口 | BaseItem:Call_StatusFlowRaw → ItemStatusComponent:Call_StatusFlowRaw | BaseItem:SetAdvancedLogicalState → LogicalState:K2_SetState |

`Call_StatusFlowRaw`（base_item.lua:1622-1631）正是分水岭。`AllInOneEventRegister_*` 等三类任务挂钩同样按此 if 分两条独立链路。

## A.2 ItemStatusComponent

ItemStatusComponent **不是 C++ 类**，而是蓝图 ActorComponent（/Game/Blueprints/Components/ItemStatusComponent），UnLua 绑定到 `ServerScript.actors.common.components.item_status_component`。

关键 UPROPERTY：
- `bStatusFlowSealed: bool, replicated, RepNotify=OnRep_bStatusFlowSealed`
- `StatusFlowRaw: S_StatusFlowRaw, replicated, RepNotify=OnRep_StatusFlowRaw`
- `Event_ActorStatus: FMulticastScriptDelegate`
- `EventOnStatusMap: TMap<TSoftObjectPtr<BP_MissionEvent_AllInOne_C>, E_StatusFlowRaw>`

关键 Multicast：
- `Multicast_CallStatusFlow(eStatusFlow)` —— 旧框架状态广播
- `Multicast_Mission_CallStatusFlow(eStatusFlow)` —— 任务系统专用广播

`SetAdvancedLogicalChains(bool)` 是模式开关，运行时把 `bAdvancedLogicalChains` 字段写好。`BaseItem:GetAdvancedLogicalChains()` 读它。

## A.3 LogicalSignalGenerator / Receiver

C++ ActorComponent，定义在外挂插件 Plugins/LogicalChains/：
- `Plugins/LogicalChains/Source/LogicalChains/Public/Components/LogicalSignalGenerator.h` —— `class ULogicalSignalGenerator : public ULogicalSignalBaseComponent`
- `LogicalSignalReceiver.h` —— `class ULogicalSignalReceiver`
- `LogicalStateComponent.h` —— 状态本身的存储 + 复制
- `LogicalSignalGate.h` —— Gate 用于多输入 AND/OR 合流

均以 `meta=(BlueprintSpawnableComponent)` 暴露给蓝图，**独立第三方插件**（"Copyright Space Raccoon Game Studio"）。

**注册流程**：
- Generator 持有 `TArray<FLogicalGeneratorSignal> GeneratorSignals`，每条 signal 配置 ActivationStates + Signals + bSendByDefault。`BindToStateComponent(true)` 把它挂到同 Actor 的 ULogicalStateComponent::OnStateChanged 多播上
- Receiver 持有 `TArray<FLogicalReceiveSignal> ReceiverSignals`，并维护 ActiveSignals，对外暴露 OnSignalReceived / OnSignalLost 多播代理
- 二者通过 LogicalChainsSubsystem 在场景中按层（Layer / GameplayTagContainer）配对
- EditorMode 下用 LogicalChainsEdMode* 一套图形工具配线

**bAutoRegister=false 的语义**：标准 UActorComponent::bAutoRegister 控制组件是否在 Actor RegisterAllComponents 阶段自动 OnRegister。GhostMechanism 注释明确：

> "蓝图层关闭 bAutoRegister 会破坏 UActorComponent 生命周期契约，导致 OnRegister 不跑 → FActiveGameplayEffectsContainer::Owner 未初始化 → null 解引用崩溃"

**结论**：LogicalSignal 这套组件需要 OnRegister 把自己注册到 Subsystem，**绝不能关 AutoRegister**；如果某个机关晚于 BeginPlay 才动态加 component，必须手动 RegisterComponent。

**信号能跨 Actor 吗**：能。Generator 发射 + Receiver 监听通过同一个 ULogicalChainsSubsystem 协调，FGameplayTagContainer Layer 是路由 key，与 Actor 实例无关。**跨 Server**：从代码上 LogicalChains 走 OnRep + Multicast，不直接跨 DDS Server；跨 Server 状态依然要靠 Aether 的 RO 机制。

## A.4 状态切换 API 速查

| API | 来源 | 在哪调 | 同步方式 |
|---|---|---|---|
| SetAdvancedLogicalChains(bool) | base_item.lua:543 (UCS) | UCS 双端 | 仅本地写 ItemStatusComponent.bAdvancedLogicalChains |
| Call_StatusFlowRaw(0, EnumKey, bNewStatusFW) | base_item.lua:1622 | Server | 分流：新→SetAdvancedLogicalState；旧→ItemStatusComponent.RepNotify |
| SetAdvancedLogicalState(NewStatusStr, bForce) | base_item.lua:588 | Server | LogicalState:K2_SetState → MARK_PROPERTY_DIRTY → OnRep |
| ServerSetAdvancedLogicalState | base_item.lua:572 | Server | 包了一层 Multicast_CallStatusFlow_New |
| Multicast_CallStatusFlow_New_RPC | base_item.lua:610 | Server→All | RPC 直推每端 SetAdvancedLogicalState |
| ClientSetAdvancedLogicalState | base_item.lua:577 | Client | 经 PlayerState.ItemAvatarComponent → 反向 RPC 到 Server |
| OnLogicalStateChanged | base_item.lua:615 | 两端 | C++ ULogicalStateComponent 多播 → BIE 派发 |
| BP_Mission_CallStatusFlow | base_item.lua:557 | 两端 | 派发 self["Status_<Name>"]() |
| ChangeStatue_Server(EnumKey) | BPA_GhostMechanism.lua:141 | Server | 同时调旧的 Call_StatusFlowRaw + 新的 ServerSetAdvancedLogicalState |

## A.5 状态机 ASCII 图

```
                                    AutoActivate / Trigger
         ┌──────────┐  Sapwn2Appear   ┌──────────┐  Appear2InActive  ┌────────────┐
 Spawn ──► [Spawn]  ─────────────────► [Appear]  ────────────────────► [InActive] │
         └──────────┘                 └────┬─────┘                    └─────┬──────┘
                                           │ Appear2Sealed                   │ InActive2Active
                                           ▼                                 ▼
                                       ┌──────────┐                    ┌──────────┐
                                       │ [Sealed] │ Sealed2InActive    │ [Active] ├─┐
                                       └────┬─────┘  ─────────────────►└────┬─────┘ │ Active2Active (loop)
                                            │                               │
                                            └─── ► [InActive]               ▼
                                                                       Active2Complete
                                                                            │
                                                                            ▼
                                                                       ┌────────────┐
                                                            ┌──────────┤ [Complete] │
                                          归一化 (SaveDB)    │          └────┬───────┘
                                          Complete→Destroy │               │ Complete2Destroy (~3s)
                                                          ▼               ▼
                                                     ┌──────────┐    [DestroyActor]
                                                     │[Destroy] ├───────►(K2_DestroyActor)
                                                     └──────────┘

      ▲  D4 fallback (Server only, T+0.5s)
      │  if K2_OnLoadFromDatabaseAllFinish 未触发 / EditorID 缺失
      └── CheckAndForceInitialState() → ChangeStatue_Server(Appear)
```

## A.6 新旧状态机辨别

- **已迁 (bNewStatusFW=true，走 LogicalState)**：GhostMechanism、ChairSeat、SwapHeadTrap、SwapHeadBattleTrap、Destructible 系、TextNarrative、Letter
- **仍是旧 (bNewStatusFW=false，走 ItemStatusComponent.StatusFlowRaw)**：base_item 系列下未显式开 bNewStatusFW 的历史 Actor
- **辨别速查**：grep `OnLogicalStateChanged` 命中的就是新框架；只有 `Multicast_CallStatusFlow` 的就是旧。`BaseItem:GetAdvancedLogicalChains()` 运行时给答案

# 主题 B：RO 复制对象

## B.1 RO 与 UE Replication 的差别

RO = **ReplicationObject**，是 Hi 项目（DDS / Aether 框架）下针对"轻量逻辑实例"的自研 UObject 复制方案。

UE 自带 Replication 是 **AActor 维度** —— 每个被复制的实例必须是 AActor，要承担 PhysicsScene、Components、Tick、Channel 等开销。Hi 的 Aether-DDS 多 Server 拓扑里有以下痛点：
- **跨 Server 漫游 (ghost-real)**：Actor 在 Server A 是 real，Server B 是 ghost，UE 原生 Channel 不能直接迁
- **轻量数据**：椅子、采集物、信件、文字提示这种"几个属性 + 状态"的对象，做成 Actor 浪费 CPU/带宽
- **数据驱动构造**：HLE EditorJson 配置驱动，Actor 不应在内存常驻，需要可在 Client 端按需 Construct() 出代理对象

RO 是 UObject (URO_*)，由 `MutableActorSubsystem.GetRO(ActorID)` 注册/查询；Server 端是逻辑权威，Client 端按 AOI 动态 Construct/Deconstruct，状态走 RepNotify (OnRep_StatusFlowRaw / OnRep_bStatusFlowSealed)，但 Actor 的"表现层"另起 BP_*_Actor，由 RO 关联。

## B.2 RO 文件清单（actors/common/interactable/RO/）

| 路径 | 类名 | 用途 |
|---|---|---|
| BP_Interacted_GainItem_RO.lua | GainItem_RO | 可拾取/获得物的通用基类 |
| BP_Interacted_PickedItem_RO.lua | PickedItem_RO | 采集物（Pick=直接捡） |
| BP_Interacted_FarmPickedItem_RO.lua | FarmPicked RO | 农田采集物 |
| BP_Destructible_RO.lua | Destructible RO | 可破坏物（轻量） |
| BP_DestructibleActor_RO.lua | DestructibleActor RO | 可破坏物（带 Actor 表现） |
| BP_BaseChest_RO.lua | BaseChest RO | 宝箱基类 |
| BP_TakeawayBox_RO.lua | TakeawayBox RO | 外卖盒 |
| BP_ChairSeatBase_RO.lua | ChairSeatBase RO | 椅子座位 |
| BP_Letter_RO.lua | Letter RO | 信件 |
| BP_TextNarrative_RO.lua | TextNarrative RO | 文本叙事 |
| BP_ShapeTrigger_RO.lua | ShapeTrigger RO | 区域形状触发器 |
| BP_ChangeModel_RO.lua | ChangeModel RO | 换模型 |
| BP_NsSpawner_RO.lua | NsSpawner RO | Niagara 特效生成器 RO |
| SwapHead/BP_SwapHeadTrap_RO.lua | SwapHeadTrap RO | 换头陷阱（普通） |
| SwapHead/BP_SwapHeadBattleTrap_RO.lua | SwapHeadBattleTrap RO | 换头陷阱（战斗） |

CommonScript 端 RO 基类（`CommonScript/actors/common/interactable/base/RO/`）：
- actor_proxy_ro.lua —— RO 与 Actor 代理基础
- base_item_ro.lua —— Item RO 基类，承载 OnRep_StatusFlowRaw / Call_StatusFlowRaw
- interacted_item_ro.lua —— 可交互 RO，继承 base_item_ro

ROUtils（`actors/common/interactable/RO/Utils/`）：BaseROUtils.lua / BP_Interacted_GainItem_ROUtils.lua / BP_Interacted_PickedItem_ROUtils.lua / BP_MineralBase_ROUtils.lua / chair_seat_utils.lua

## B.3 RO 注册与生命周期

```
[Server 启动]
  └─► HLE 加载 EditorJson → MutableActorSubsystem 解析 ActorID
        └─► 按需 Construct RO（URO_* UObject 实例）
              ├─► ro:Construct() 调用业务初始化
              ├─► RO 注册到 MutableActorSubsystem 的 RO 表
              └─► 关联 GroupActor / OwningActor

[Client 进入 AOI]
  └─► 收到 RO 同步 → Client 侧按需 Construct → 触发 ROActorReady
        └─► 状态变化通过 OnRep_StatusFlowRaw → Call_StatusFlowRaw(true) 派发表现

[Actor 端]
  └─► BaseItem:ListenROSpawnOrDestroy(ActorID, Listener, bSpawnOrDestroy)
        ├─► MutableActorSubSystem:GetRO(ActorID)
        └─► CheckChildReady() —— 把 RO 实例与 Actor 实例绑定

[Server Destroy]
  └─► RO:Deconstruct(ROContainer) 调业务清理
```

## B.4 Multicast 重广播 + Destroy 直销毁

**场景**：K2_OnLoadFromDatabaseAllFinish 阶段（ReceiveBeginPlay 前 ~1ms），Actor 尚未 Initial Replication。GhostMechanism 注释把动机讲得非常透：

> "C++ ULogicalStateComponent::SetCurrentState 只在被调用时 MARK_PROPERTY_DIRTY，SaveGame 反序列化直接改字段不走该函数 → Push Model 不标脏 → 初始 Replication 给 Client 的 CurrentState 值与默认空值相同 → OnRep_CurrentState 不触发。"

**为什么不能用 RepNotify**：因为 SaveGame 反序列化是把字段直接拷回，没经 SetCurrentState 这条路径，UE 5.5 的 Push Model 不会标脏，初始 Replication 把"恢复后的值"和"默认值"打包发给 Client 时被判等抑制 → 客户端 OnRep 没机会触发 → Status_xxx 蓝图回调不被调用 → 客户端表现卡死。

**修法**：
- bHasSaved=true 时，**主动调** `ServerSetAdvancedLogicalState(state, true)`，bForce=true 绕过 C++ SetCurrentState 同值去重，强制 MARK_PROPERTY_DIRTY + 主动 Multicast
- 已死亡（Destroy / Complete）的特例：直接 K2_DestroyActor()，不走 Multicast —— Actor 还没初始 Replication，Client 从未 Spawn → 直接销毁 = 无副作用

## B.5 C++ RO 基类

未在 Source/HiGame 目录下命中 `class.*RO` 或 ReplicationObject 直接定义 —— Hi 自研 RO 基类位于 **DDS/Aether 框架插件**（不在 HiGame Source 内，而在外部 Plugins / Engine 私有分支）。运行时通过 MutableActorSubsystem 暴露：`GetRO(ActorID)` / `GetActor(ActorID)` / `MarkPropertyDirty(self, "StatusFlowRaw")` 等接口证明 RO 是 UObject 体系下的 PushModel 复制对象。

## B.6 RO 与 DDS 跨 Server 迁移

DDS Aether 框架下 RO 是逻辑权威，Actor 是表现代理。当玩家从 Server A 走到 Server B：
- RO **EditorID + 业务字段**通过 Aether 的 ghost-real 机制随玩家 / AOI 迁移
- Actor 在 Server B 按 RO 重新 Spawn 表现层
- `Server_RPCByGhost` 解决 ghost 对应的 Server-to-Server RPC，避免 ghost 端调到本地空函数
- `SendMessageWhenActorNotExist` 在 Actor ghost 都不存在时由 MutableActorComponent 走"消息总线"投递

# 主题 C：存盘 / 恢复 / D4 fallback

## C.1 OnSaveToDatabase 流程

**触发时机**：DDS 框架定时/事件触发持久化（玩家离开 AOI、DS 关服、跨服迁移点等）。

**写入字段**：ItemStatusComponent.StatusFlowRaw 是核心；ULogicalStateComponent.CurrentState / PreviousState 标了 SaveGame 也参与序列化。

**典型实现**（base_item.lua:2009-2012）：
```lua
function BaseItem:K2_OnSaveToDatabase()
    self:LogInfo("zsf", "BaseItem:K2_OnSaveToDatabase %s", self:GetName())
    self.bHasSaved = true
end
```

GhostMechanism 在此基础上加"归一化"步骤（C.4）。

## C.2 OnLoadFromDatabaseAllFinish 流程

**为什么 AllFinish**：DDS 加载流程是分片 + 异步（多个组件、多个 Promise，async 加载素材），单个 K2_OnLoadFromDatabase 只代表"我这个 Actor 的字段反序列化完了"，但素材/RO 关联未必齐。AllFinish 是"所有相关分片都完成"的一次回调。

**流程**（base_item.lua:1989-2003）：
```
K2_OnLoadFromDatabaseAllFinish
  ├─ if bHasSaved:
  │     新框架 → SetAdvancedLogicalState(LogicalState:K2_GetCurrentState(), force=true)
  │     旧框架 → ItemStatusComponent:Call_StatusFlowRaw(0, GetStatusFlowRaw(), bNewStatusFW)
  └─ else (从未存过盘):
        SubSystem:SetInfoSceneItemStatus(self, ActorID)  -- 触发首次状态初始化为 Appear
```

GhostMechanism Server 在此基础上做了"已死亡直销毁"分支 + "过渡态强制重广播"分支。

## C.3 D4 fallback 详解

**D4 含义**：代码注释仅写 "D4 Server fallback" 没有展开。**从上下文推测，D4 可能是 GhostMechanism 这个机关的"防御等级 4 级"或"第 4 类边界条件 (Defense #4)"**，是该机关在 sprint 修 bug 时用的内部编号，专指"Entity 加载链路完全失败"的兜底场景。如需准确含义需查 sprint 内部文档。

**fallback 解决什么问题**：
1. Actor 不是 Entity 管理（无 EditorID）—— K2_OnLoadFromDatabaseAllFinish 永远不会被调
2. Entity 加载链路异常 —— DDS 没把这个 Actor 派发为"AllFinish 回调对象"
3. 只有 ReceiveBeginPlay 被调，状态卡在 0 → 受击逻辑被 `if status ~= Active` 拒掉

**0.5s 延迟动机**：给 Entity 加载链路充足时间走完。0.5s 是经验值：足够长以容下正常加载（绝大多数 < 100ms），又足够短让玩家不会觉察到延迟。

**触发条件**：ReceiveBeginPlay 启 timer，T+0.5s 时检查 `_bStateInitialized` 标志（在 OnLogicalStateChanged 与 K2_OnLoadFromDatabaseAllFinish 中被置 true）。如果还是 false，认为 Entity 链路掉线，手动 ChangeStatue_Server(Appear)。

## C.4 CheckAndForceInitialState

`BPA_GhostMechanism.lua:264-276`：
- 清掉自身 timer handle
- 看 _bStateInitialized，已初始化跳过
- 检查 EditorID 是否合法（"" / "0" / 0 都判非 Entity）
- 强行 `ChangeStatue_Server(Enum.E_StatusFlowRaw.Appear)` —— 同时调旧 Call_StatusFlowRaw + 新 ServerSetAdvancedLogicalState

## C.5 存盘归一化原则

**Complete → Destroy（GhostMechanism Server :248-257）**：
- **设计动机**：Status_Complete 是过渡态 —— 进入后 1.5s 就会调 TriggerDestroy → Destroy
- 玩家若在这 1.5s 窗口内断网/AOI 离开/DS 切服，存盘字段 eStatusFlowRaw=Complete 被序列化
- 下次入场若不归一化，会在 Common:Status_Complete 重播消散特效 + 再起一次 TriggerDestroy timer，造成"幽灵复活又消散"
- 修法：序列化**前**直接改字段 `self.ItemStatusComponent.StatusFlowRaw.eStatusFlowRaw = Destroy`，**不走** Call_StatusFlowRaw —— 那样会触发 OnItemStatusCondition 任务链副作用，污染 Mission 系统

**过渡态不入档**：原则推广 —— 任何"中间动画态 / 倒计时态"在 Save 前都应归一到稳定终态。

## C.6 重连恢复 vs Reconnect 快照

- **机关恢复**：走 K2_OnSaveToDatabase / K2_OnLoadFromDatabaseAllFinish，**Actor 维度**，每个机关自己存盘自己复盘
- **Mission/Puzzle 进度恢复**：走 MissionPuzzleSubsystem.GetLocalPlayerRunningSnapshots，**玩家维度**，跟踪"任务在哪一步"的状态机快照

二者**不是同一套**但会协同：玩家断线重连时，先由 MissionPuzzle 恢复任务节点状态，再由各 Actor 的 OnLoadFromDatabaseAllFinish 把每个机关自己的物理/动画状态复位。

## C.7 DDS 跨 Server 存盘

跨 Server 时，Actor 在 Server A K2_OnSaveToDatabase 把状态写入数据库；Server B 加载该 ActorID 时，DDS 把 EditorJson + 存档数据合并喂给新 Spawn 的 Actor，再触发 K2_OnLoadFromDatabaseAllFinish。RO 视角下，RO 是逻辑权威，存档与 RO 字段直接对应；Actor 在 Server B 是"按 RO 状态重建表现"，因此 Multicast 重广播修复对跨 Server 同样有效。

## 关键代码位置

- base_item.lua:71-73 — UCS / SetAdvancedLogicalChains
- base_item.lua:543-555 — Get/SetAdvancedLogicalChains
- base_item.lua:557-636 — BP_Mission_CallStatusFlow / Server/ClientSetAdvancedLogicalState / OnLogicalStateChanged
- base_item.lua:1622-1631 — Call_StatusFlowRaw 新旧分流
- base_item.lua:1989-2012 — K2_OnLoadFromDatabaseAllFinish / K2_OnSaveToDatabase 基类
- base_item.lua:2391-2412 — GetGroupActor / GetROContainer_Client / GetRO
- BPA_CharacterBaseWithStatus.lua:32-37 — 角色侧 SetAdvancedLogicalChains UCS
- BPA_GhostMechanism (Common):141-164 — ChangeStatue_Server / OnLogicalStateChanged 派发
- BPA_GhostMechanism (Server):24 — FALLBACK_DELAY = 0.5
- BPA_GhostMechanism (Server):185-276 — Load 重广播 / Save 归一化 / D4 fallback
- CommonScript/.../base/RO/base_item_ro.lua:25-75 — OnRep_StatusFlowRaw / Call_StatusFlowRaw / MarkPropertyDirty
- actors/common/interactable/RO/BP_Interacted_PickedItem_RO.lua:19-45 — Construct / Server_ReceiveDamage
- Plugins/LogicalChains/Source/LogicalChains/Public/Components/LogicalStateComponent.h:85-167
- Plugins/LogicalChains/.../LogicalSignalGenerator.h:24-73
- Plugins/LogicalChains/.../LogicalSignalReceiver.h
