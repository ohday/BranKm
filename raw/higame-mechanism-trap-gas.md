---
id: 59
title: "HiGame Trap 战斗陷阱与 GAS 集成"
source: local-archaeology
project_root: E:\HiProject\PerforceDev\UnrealEngine\Projects\HiGame
modules:
  - Content/Script/actors/common/trap/
  - Content/Script/CommonScript/actors/interactable/SwapHead/
  - Content/Script/actors/common/interactable/{FenNuBao,Inmate,PumpkinGame,BubbleTrap}/
fetched: 2026-05-11
status: content-verified
covers_questions: [Q15]
---

# Trap 战斗陷阱与 GAS 集成

## 一、Trap 在 Hi 项目里的定义

Hi 项目里"陷阱"并非统一基类层，而是按用途分流的**两条血脉 + 一个特殊群**：

### 1. 物理触发型 Trap（actors/common/trap/）
- **根类**：`TrapActorBase`（继承 common.actor）
- **核心模型**：Overlap 触发器 + InnerActors 列表 + Real_OnActorBeginOverlap/EndOverlap 钩子
- 不是采集物，没有 F 键交互、没有背包、没有 DataTable —— 玩家踩进去或外部 FlowGraph 激活就触发
- 典型成员：TrapActorBalloonSmall/Big、TrapActorBalloonSmallSpawnShowActor、TrapActorPlayAkAudio、TrapActorSafeArea

### 2. 采集物型 BattleTrap / SwapHeadTrap
- 位置：`Content/Script/CommonScript/actors/interactable/SwapHead/`
- **根类**：`BP_Interacted_PickedItem`（即 interactable 一族）
- 语义：换头采集物 —— 玩家走近、按交互键、Server_ReceiveDamage 在服务端触发 → 给玩家施加 Buff (GE) + 注册换头 + 走采集物消散流程

### 3. interactable 内的特殊小游戏 / 反派物件
- PumpkinGame（南瓜小游戏）、FenNuBao（愤怒煲爆炸物）、Inmate（囚徒）、BubbleTrap
- 继承 base_item 或 interacted_item 而非 TrapActorBase
- 行为模式都是"触发 → 危险化 → 对玩家施加 GE / 击飞 / 死亡"

### trap/ 与 interactable/ 边界对照

| 维度 | actors/common/trap/ | interactable/.../*Trap*、PumpkinGame/FenNuBao |
|---|---|---|
| 父类 | common.actor | BP_Interacted_PickedItem / base_item / interacted_item |
| 触发方式 | OnActorBeginOverlap | 交互键 (F)、爆炸半径 Sphere Overlap |
| 与背包 | 无 | 有（PickedItem 流程） |
| 状态机 | 自定义 bActive + Real_On* | E_StatusFlowRaw |
| 复制对象 | 不走 RO | 走 RO（actors/common/interactable/RO/SwapHead/*_RO.lua） |
| 主要消费者 | LD 关卡触发 | MissionPuzzle / 主线 / 采集 |

### SwapHead 哪些是"陷阱"

`CommonScript/actors/interactable/SwapHead/` 中：
- **真陷阱采集物**：`BP_SwapHeadBattleTrap.lua`（战斗换头）、`BP_SwapHeadTrap_C.lua`（LD 换头）
- **道具靶子**：`BP_BalloonTarget_*` / `BP_OilTarget_*` / `BP_CandleTarget_*` / `BP_SpotlightTarget_*`（被换头后角色用技能命中），不是 trap 本身

### trap 与 puzzle 的区分

trap 通常 **单点 / 即时 / 对玩家产生作用**；puzzle 通常有 **进度 (PuzzleID) + Mission 节点对接**。`pumpkin_game_manager.lua` 兼具两者特征 —— 是 **Puzzle 包装的 Trap 组合**。

## 二、Trap 通用模型

```
                          ┌────────────────────────────────────────┐
                          │  外部触发源:                            │
                          │  - FlowGraph (ActiveByFlowGraph)        │
                          │  - 玩家碰撞 (OnActorBeginOverlap)       │
                          │  - F 键交互 (Server_ReceiveDamage)      │
                          │  - Hit Event (Event_ActorHit)           │
                          └──────────────────┬─────────────────────┘
                                             │
   [Spawn]───SetbActive(false)───>[Standby/Inactive]
                                             │ activated
                                             ▼
                          ┌────────────────────────────────────────┐
                          │ [Activated/Active]                     │
                          │ - SetActorHiddenInGame(false)          │
                          │ - 播 Niagara/AK 音效                   │
                          │ - SpawnActor(ShowActor)                │
                          │ - 进入 E_StatusFlowRaw.Active          │
                          └──────────┬───────────────┬─────────────┘
                                     │ delay         │ overlap with player
                                     ▼               ▼
              [BoomAndBreak/SphereOverlap]    [Server_ReceiveDamage]
                                     │               │
                                     ▼               ▼
              SkillUtils.HandleHitEvent     ASC:BP_ApplyGameplayEffect
              ASC:MakeOutgoingSpec               ToSelf(BuffGEClass)
              ASC:BP_ApplyGameplayEffect         (走 buff_component)
                 SpecToSelf(GE_Damage_Boom)
                                     │               │
                                     ▼               ▼
                          [Effect On Player]  GameplayCue.SwapHead.* 触发表现
                                     │
                                     ▼ FlyAnimationOver / OnBoomFinished / Status_Complete
                          [Deactivated/Complete]
                                     │
                                     ▼ bAutoDestroy
                          [Destroy → K2_DestroyActor]
```

## 三、与 GAS 的耦合（三种触发方式实测对照）

| 方式 | 触发方法 | 适用场景 | 实例 | 优劣 |
|---|---|---|---|---|
| **A. SendMessage("AddBuffByID") → BuffComponent → ApplyGameplayEffectToSelf** | `MainActor:SendMessage("AddBuffByID", BuffID, 1, nil, bPureClient)` → buff_component.lua:546 `ASC:BP_ApplyGameplayEffectToSelf(BuffGEClass, Level, Context)` | 持续型 Debuff/换头 Buff，**配置驱动**（DataTable 列 CombatBuffID） | BP_SwapHeadBattleTrap:Server_ReceiveDamage、BP_SwapHeadTrap_C 通过 LDComp:ForceEquipHead 间接走 AddBuffByID | 优：配置化、自动 RPC; 劣：不能即时携带 Magnitude |
| **B. ASC:MakeOutgoingSpec + AssignTagSetByCallerMagnitude + ApplyGameplayEffectSpecToSelf** | `MakeOutgoingSpec(GEClass, 1, Ctx)` → `AssignTagSetByCallerMagnitude(Spec, "SetByCaller.Modifier", self.Damage)` → `BP_ApplyGameplayEffectSpecToSelf(Spec)` | **携带运行时数值的瞬时伤害** | BPA_GdObject_Prop_Fennubao:Server_HitTargetActor (line 198–200) | 优：可运行时设伤害值; 劣：直接耦合具体 GEClass |
| **C. SendMessage("HandleHitEvent", FGameplayEventData)** | `gas_utils:HandleHitEvent` 构造 FGameplayEventData，挂 KnockInfo 到 OptionalObject，TargetActor:SendMessage("HandleHitEvent", HitPayload) | **击飞/受击**链路，命中后让目标走"被击飞" Ability | BPA_GdObject_Prop_Fennubao、BP_HeadEye:HandleItemCharacterHitEvent | 优：解耦受击表现、由 Target 端决定怎么响应; 劣：约定式（依赖 Tag 触发链） |

## 四、EGASAbilityInputID 与 trap

`Source/HiGame/Public/HiGame.h:18-30` 仅枚举 8 项：None / Dodge / DodgeFwd / DodgeLeft / DodgeRight / Attack / Combo / Block / Kick。**全部是玩家主动输入类技能** —— 陷阱不直接走 InputID 路径。陷阱触发的 Ability/GE 全部通过：
1. BuffID 配置驱动
2. GameplayEvent (HandleHitEvent)
3. Tag 触发的 Ability（如换头 Tag 进入战斗状态）

## 五、Damage / 受击事件链

```
BP_ItemCharacterReceiveDamageComponent:OnHitEvent(Instigator, Causer, Hit, KnockInfoData)
    ↓ (反射)
self.actor:HandleItemCharacterHitEvent(...)
    ↓
具体 Actor (BP_HeadEye / BPA_Character_PlatinumCake / BPA_GhostMechanism) 实现该方法
    → 改 StatusFlow / 给奖励 / 反击
```

`BPA_CharacterBaseWithStatus` 是 pumpkin_npc 等 Trap NPC 的基类 —— 它本身没有 trap 受击逻辑，受击逻辑通过附加 BP_ItemCharacterReceiveDamageComponent 组件挂上去（**组件化注入而非继承注入**）。

## 六、GameplayTag 速查

实证可 grep:
- `GameplayCue.SwapHead`（父 Tag，互斥检查）—— SwapHeadUtils.lua:301
- `SetByCaller.Modifier`（动态伤害值）—— Fennubao:199
- `Event.Hit.KnockBack.SuperHeavy`（击飞类型）—— gas_utils.lua:19
- `OwnerTagQuery`（蓝图属性，测换头是否已存在）—— BP_SwapHeadBattleTrap:128
- `SwapHeadTag`、`SkillUtils.GetInBattleTag()`、`SkillUtils.GetBattleStarSkillActiveTag()`

## 七、Cue / VFX

陷阱命中表现走两条路：

- **GameplayCue 路**：换头表现走 GameplayCue.SwapHead.*（互斥/恢复在 SwapHeadUtils.SwitchHeadVisual 集中）。GC 由 GE 自动触发。BP_SwapHeadBattleTrap:_PrewarmCombatGC 客户端异步预热 GC 蓝图
- **直接生成路**：愤怒煲（NiagaraEffectSetActive、PlayBoomOnMulticast）、Balloon（PlayAnimSeq、SetActorHiddenInGame 切换）、HiAudioFunctionLibrary.PlayAKAudio("Scn_Item_PickUp", RoActor)

## 八、典型样例深读

### 8.1 SwapHead BattleTrap（战斗换头陷阱）
- 文件：BP_SwapHeadBattleTrap.lua + BP_SwapHeadTrap_C.lua + Utils/SwapHeadUtils.lua + Utils/POLifecycleManager.lua
- **玩法**：玩家进入战斗状态后，靠近 BattleTrap（地图上的怪谈头模型），按 F 交互 → 玩家"换头"（挂战斗 Buff + 替换 SkeletalMesh）→ 用换好的怪谈头释放特定怪物技能
- **GAS 触发链**：
  1. DataTable 行 CombatBuffID → 缓存到 self._CombatBuffID
  2. 服务端 Server_ReceiveDamage：先用 OwnerTagQuery 与 MainActor.AbilitySystemComponent:MatchGameplayTagQuery 互斥
  3. HeadComp:RegisterPlayerHead(Combat, TemplateID)
  4. MainActor:SendMessage("AddBuffByID", BuffID, 1, nil, bPureClient) → buff_component.lua:546 → ASC:BP_ApplyGameplayEffectToSelf
  5. GE 触发 GameplayCue.SwapHead.* → SwapHeadUtils.SwitchHeadVisual

### 8.2 TrapActorBalloonSmallSpawnShowActor
- **玩法**：地面气球 —— FlowGraph 调 ActiveByFlowGraph 激活；激活时：1) SetActorHiddenInGame(false) 显形；2) TrySpawnShowActor 在 ShowActorSpawnPoint 处生成"展示 Actor"（如气球+飞行动画）；3) TryStiffnessBoss —— 对当前 Boss 调 Boss:SendMessage("BSM_EnterStiffness")
- **生命周期**：FlyAnimationOver 收尾（AkSwitch 复位、bActive=false、销毁 ShowActor）
- **GAS 触发**：自身**不直接对玩家施 GE** —— 是 *对 Boss 的反向 trap*（"气球击中 Boss" Trap-as-tool 范式）

### 8.3 Inmate（囚徒）
- 继承 interacted_item，只在 bLock=false && StatusFlowRaw==InActive && bFakeTriggered=false 时才允许 OnBeginOverlap
- BP_Mission_CallStatusFlow 接 Appear/Destroy 两态
- **GAS 触发**：本类不直接走 GAS —— 它是物件层；陷阱效果由"囚徒释放后 NPC 行为树"完成（间接陷阱）
- **存档**：Multicast_Dissolve_RPC 直接 K2_DestroyActor，**不走 RO**（单次任务对象，不需跨周目复活）

### 8.4 PumpkinGame Manager（关卡型 trap 组合）
- 文件：pumpkin_game_manager.lua、pumpkin_npc.lua、seal_fragment.lua、exit_portal.lua
- **玩法**：典型 N 个 NPC 陷阱 + N 碎片采集 + 1 出口 的组合 puzzle
- NPC 追玩家：CatchRadius / MoveSpeed / ViewAngleThreshold 由蓝图属性配置。玩家进视野角→冻结追逐，离开→恢复
- Manager 双模式：Manager 模式（Move_To_Way_Point 邮件 + 自定义 Timer）/ BT 模式（黑板写入 + 事件）
- **Puzzle 集成**：__listen_puzzle_start 订阅 UHiMissionPuzzleSubsystem.OnPuzzleStarted/OnPuzzleEnded
- **GAS 触发**：南瓜 NPC 抓到玩家 → 自身 ASC 发起技能（继承自 BPA_CharacterBaseWithStatus，是带 ASC 的角色基类）；碎片是采集物。Manager 自身不调用 GAS

### 8.5 FenNuBao（愤怒煲，最纯粹的"伤害陷阱"）
- 文件：BPA_GdObject_Prop_Fennubao.lua
- **玩法**：玩家踩进 TriggerSphere（默认 200cm）或被 Hit → 1s 描边 → 1.5s 后 BoomAndBreak 球形 Overlap 找所有 Pawn → 对每个 Pawn 调 Server_HitTargetActor 击飞+伤害 → 2.5s 后 SetComplete
- **GAS 触发链（最干净的 GE 直送范式）**：
  ```
  gas_utils:HandleHitEvent(self, TargetActor, HitResult)  -- 击飞（路径 C）
  TargetActor.AbilitySystemComponent
      :MakeOutgoingSpec(EffectClass, 1, FGameplayEffectContextHandle())
      ⇣
  AssignTagSetByCallerMagnitude(Spec, "SetByCaller.Modifier", self.Damage)
      ⇣
  ASC:BP_ApplyGameplayEffectSpecToSelf(GESpecHandle)         -- 伤害（路径 B）
  ```
- **状态机**：E_StatusFlowRaw 完整六态全用上 —— Status_Active → PlayBoomOnMulticast，Status_Complete → OnBoomFinished，bAutoDestroy=false 时回到 Appear 形成复活循环

## 九、PO LifecycleManager（PlayerOwned NPC 吸引器生命周期）

文件：CommonScript/actors/interactable/SwapHead/Utils/POLifecycleManager.lua

**PO = "NPC 吸引器" Actor (BP_NPCAttractor)** —— 是换头机关 / MassAI F 交互 / ed_runtime 换头三场景共用的 *动态生成、吸引附近 NPC 进 Slot 表演* 的容器 Actor。"PlayerOwned"指其归属玩家上下文。

**为何需要管理器**：PO 必须有明确生命周期 —— 持续 N 秒后过期（DEFAULT_DURATION=10s）但*不能立即销毁*（NPC 还在表演中），需要事件驱动 + Tick 兜底。

**API 速查**：
- Start(TimerOwner) / Stop() —— 由 LDHeadSwapComponent 在头效果激活/失效时启停
- RegisterPO(actor, duration) —— 生成后注册
- MarkReadyToDestroy(actor) —— Attractor 检测到 OccupiedSlotCount==0 时调
- _OnTick —— 0.5s 一次扫描；过期 → actor:MarkExpired() 占满空闲 Slot；ready_to_destroy → 立即销毁；超时 5s 兜底 → SetReleaseNpcsOnDestroy(true) 强销
- SpawnPOAtLocation —— 统一生成入口：地面投影 LineTrace、ULQTMassPOSubsystem:CreateDynamicPOFromActorClass 优先、回退 World:SpawnActor

## 十、RO 在 trap 中的使用

- 目录：actors/common/interactable/RO/SwapHead/{BP_SwapHeadBattleTrap_RO.lua, BP_SwapHeadTrap_RO.lua}
- **核心差异**：RO 版继承 BP_Interacted_PickedItem_RO 而非 BP_Interacted_PickedItem。RO 是 Replication Object —— Hi 项目的"轻量级跨周目持久对象"，可大量批量管理而不占 Actor 数
- **生命周期入口**：用 Construct 而非 ReceiveBeginPlay
- **完成流程**：Status_Complete 改为先调 VisibilityManagementComponent:ControlMaterialParam 做溶解、再 UnRegisterRO（防父类 GainItem_RO.Status_Complete 的瞬间消失感）
- **GAS 行为完全一致**：RO 与非 RO 共享同一 GAS 链，**代码复制/粘贴而非抽公共**（重复维护成本）

## 十一、DDS 跨 Server 行为

- 战斗 Buff 走 `bPureClient = SkillUtils.IsSinglePlayerGame(self)` 判定。单机模式下 GE 走纯本地路径不发 RPC
- PO 通过 ULQTMassPOSubsystem 子系统创建 —— 所有 trap 衍生 PO 经统一子系统跟踪，跨区域时由子系统 GC
- Trap 状态用 OnRep_bActive，标准 UE Replication 字段，跨 Server 时由 DDS Routing 层转发

## 十二、常见反模式与陷阱

1. **Tick 检测受击** —— 项目主流是组件化注入 + OnHitEvent → HandleItemCharacterHitEvent 反射调用，避免 Tick 轮询
2. **Server_ReceiveDamage 缺校验** —— BP_SwapHeadBattleTrap:DoClientInteractAction 显式做客户端前置校验，避免无效 RPC
3. **GE 不带 SetByCaller** —— 愤怒煲示范了正确做法。新 Trap 想做"动态伤害"必须用 SetByCaller，不能改 GEClass
4. **MetaData 透传 lightuserdata 崩溃** —— BP_SwapHeadTrap_C 显式判 `type(MetaData) ~= "string"` 兜底（rapidjson 崩）
5. **GC OnRemove 不触发** —— SwapHeadUtils.RegisterGCCleanup / RunGCCleanup 补丁
6. **PO 卡死销毁** —— POLifecycleManager 用"事件驱动 + 5s 超时强制销毁"双保险
7. **PuzzleStartedEvent 未触发** —— PumpkinGameManager.K2_OnLoadFromDatabaseAllFinish 10s 兜底 Timer
8. **AttachTo 后位置错位** —— POLifecycleManager._ProjectToGround 显式 LineTrace 把根从胶囊体中心拉到地面

## 关键代码位置

- actors/common/trap/TrapActorBase.lua:21-29 — TrapActorBase OnActorBeginOverlap
- actors/common/trap/TrapActorBalloonSmallSpawnShowActor.lua:47-53 — ActiveByFlowGraph
- CommonScript/actors/interactable/SwapHead/BP_SwapHeadBattleTrap.lua:128-147 — Server_ReceiveDamage GAS 链
- CommonScript/.../SwapHead/Utils/POLifecycleManager.lua:48-205 — PO Start/Stop + _OnTick
- CommonScript/.../SwapHead/Utils/POLifecycleManager.lua:225-402 — SpawnPOAtLocation + _ProjectToGround
- CommonScript/.../SwapHead/Utils/SwapHeadUtils.lua:296-358 — ShouldSkipGCRemove + SwitchHeadVisual
- actors/common/interactable/FenNuBao/BPA_GdObject_Prop_Fennubao.lua:150-202 — BoomAndBreak + Server_HitTargetActor 范式
- common/utils/gas_utils.lua:15-43 — HandleHitEvent
- CommonScript/actors/components/buff_component.lua:540-546 — BuffComponent ASC:BP_ApplyGameplayEffectToSelf
- actors/common/components/BP_ItemCharacterReceiveDamageComponent.lua:24-28 — OnHitEvent 反射
- CommonScript/actors/interactable/Inmate/BP_InmateItemBase.lua:18-22 — Inmate 三条件门
- actors/common/interactable/PumpkinGame/pumpkin_game_manager.lua:79-119 — PuzzleSubsystem 监听
- Source/HiGame/Public/HiGame.h:18-30 — EGASAbilityInputID
