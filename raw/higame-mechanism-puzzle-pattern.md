---
id: 57
title: "HiGame Puzzle 5 标准范式（DancingSofa / DouDing / GhostMechanism / HeadEye / SurveillanceBird）"
source: local-archaeology
project_root: E:\HiProject\PerforceDev\UnrealEngine\Projects\HiGame
modules: Content/Script/{Server,Client,Common}Script/actors/Puzzle/
fetched: 2026-05-11
status: content-verified
covers_questions: [Q6, Q7, Q8, Q9]
---

# Puzzle 5 标准范式

## 一、三层目录与继承链

### 文件矩阵

| 机关 | ServerScript | ClientScript | CommonScript | StateTree |
|------|---|---|---|---|
| DancingSofa | BPA_DancingSofa.lua | 同 | 同 | — |
| DouDing | BP_DouDingPuzzle.lua + Runner + TrackInteract | 同 | 同 | — |
| GhostMechanism | BPA_GhostMechanism.lua | 同 | 同 | STTask_SplinePatrol / MoveToSpline / FleeFromPlayer |
| HeadEye | BP_HeadEye.lua | 同 | 同 | STTask_Patrol / Spotted + STCond_PlayerDetected |
| SurveillanceBird | BP_SurveillanceBird.lua + BP_SurveillanceTV.lua | 同 | 同 | — |

### 继承链

```
ActorBase (actors.common.interactable.base.base_item)
  └── BPA_CharacterBaseWithStatus (actors.common.character)
        └── *Common (CommonScript)
              └── *Server (ServerScript, extends Common)
              └── *Client (ClientScript, extends Common)
```

每个机关 ServerScript / ClientScript 第一行 require：

```lua
local CommonBase = require("CommonScript.actors.Puzzle.<Mech>.<File>")
local <Mech>Server = Class(CommonBase)
```

蓝图侧通过 `GetServerModuleName / GetClientModuleName` 路由；`GetModuleName` 返回空。

### 三层职责矩阵

| 职责 | Common | Server | Client |
|---|:--:|:--:|:--:|
| Initialize 字段统一声明 | ✅ | (override 追加) | (override 追加) |
| Overlap 组件绑定 | ✅ | — | — |
| Overlap 回调 stub | ✅ (空) | ✅ override 写 bool | — |
| 配置校验 _InitConfig | ✅ | — | — |
| OnLogicalStateChanged 路由 | ✅ | — | — |
| ChangeStatue_Server | ✅ | — | — |
| Status_* skeleton | ✅ (log) | override 触发 AI/存盘/销毁 | override 触发视觉/音效/UI |
| CharacterMovement 调参 | — | ✅ | — |
| StateTreeComponent:StartLogic | — | ✅ | — |
| D4 fallback | — | ✅ | — |
| K2_OnSaveToDatabase 归一化 | — | ✅ | — |
| K2_OnLoadFromDatabaseAllFinish 重广播 | — | ✅ | — |
| HandleItemCharacterHitEvent | — | ✅ | — |
| GiveRewardToPlayer (ROUtils) | — | ✅ | — |
| Niagara 异步预加载 | — | — | ✅ |
| 灵压仪 SoulMeter 注册 | — | — | ✅ |
| Debug Tick 可视化 | — | — | ✅ |

## 二、Common 层职责

Common 层承载"两端都必须执行同一套语义、否则两端会失同步的逻辑"：

1. 字段集中声明（Initialize）
2. Overlap 组件绑定（BindDetectSphereOverlaps）
3. OnLogicalStateChanged → Status_xxx 路由
4. NSM/Spline 工具函数（SetupNSM / StartOrResumeNSM 等）—— 两端 Spline `Detach(KeepWorld)` 到 HLE JSON 配置同一世界坐标
5. 配置校验（_InitConfig 失败设 _bConfigFailed=true，两端跳过初始化）

不直接放 Server 的原因：Client 也需要执行其中很多逻辑（NSM 初始化、Overlap 绑定、状态路由），单独放 Server 会维护漂移 + Client 挂掉。

## 三、Server 层 ReceiveBeginPlay 套件

以 GhostMechanism Server `ReceiveBeginPlay`（:32-63）为标杆：

```lua
function GhostMechanismServer:ReceiveBeginPlay()
    self.Super.ReceiveBeginPlay(self)              -- 调 Common
    self:CacheSplineData()                         -- Spline 路点数缓存
    self.CharacterMovement.MaxWalkSpeed              = self.PatrolSpeed or 200.0
    self.CharacterMovement.bOrientRotationToMovement = true
    self.CharacterMovement.MaxAcceleration            = 600.0
    self.CharacterMovement.BrakingDecelerationWalking = 400.0
    self.CharacterMovement.RotationRate               = UE.FRotator(0, 360, 0)
    self.StateTreeComponent:StartLogic()
    self._initFallbackHandle = G.TimerManager:SetTimer(
        {self, self.CheckAndForceInitialState}, FALLBACK_DELAY, false)
end
```

**Overlap 感知 override**（:72-99）：Common 绑定事件，Server override 判断 OtherActor.PlayerState 与 bCanFlee，再写 bPlayerInInner/Outer。两个 bool 由 StateTree Cond 节点查询决定 Patrol→Flee 转移。

## 四、Client 层

Client 做"看得见的事"。GhostMechanism Client：

| 子系统 | 函数 |
|---|---|
| Niagara 异步预加载 | PreloadNiagaraAssets (Kittens async loader) |
| 视觉激活/停止 | ActivateVisuals / DeactivateVisuals |
| 消散特效 | PlayDissipateEffect |
| 灵压仪 (SoulMeter) 注册 | RegisterSoulMeterListener (GameEventBus) |
| Spline 路径可视化 | ShowSplinePath (Niagara DI Custom_Spline) |
| Reward Mesh + 动画 | ShowRewardMeshAndAnim |
| Debug 可视化 + 瞬移检测 | ReceiveTick |

Client 端**不写状态、不存盘、不发奖励**——所有结果从 Server Multicast 或 OnLogicalStateChanged 流入。

## 五、Status 状态机

`Enum.E_StatusFlowRaw`：Appear / InActive / Active / Complete / Destroy（DancingSofa 多 Sealed）。

```
                       D4 fallback (0.5s 未收到 DB 初始化)
                                  │
                                  ▼
Spawn ─► (LoadDB) ─► Appear ──[Server: ChangeStatue_Server(Active)]──► Active
                       │                                                  │
                       │                                              HitEvent
                       │                                                  ▼
                       │                                              Complete
                       │                                                  │
                       │                                              TriggerDestroy timer
                       │                                                  ▼
                       └────── (LoadDB Destroy) ──► K2_DestroyActor◄──┘
```

**Common:OnLogicalStateChanged** 是 dispatcher：`if StateName == "Appear" then ChangeStatue_Server(Active) elseif "Active" then Status_Active() ...`。**Appear 是一次性自驱状态**：进 Appear 立刻调 ChangeStatue_Server(Active)，不停留。

**AutoActivate** 来自 D4 fallback：T+0.5s 未初始化则 ChangeStatue_Server(Appear)。

## 六、bNewStatusFW + LogicalChain

### 老旧分水岭

- **旧框架**：ItemStatusComponent.StatusFlowRaw（属性复制）→ OnRep → BP_Mission_CallStatusFlow
- **新框架**：BP_LogicalSignalGenerator/Receiver + ULogicalStateComponent。`SetAdvancedLogicalChains(true)` 把状态写入路由到 LogicalState:K2_SetState，触发 OnLogicalStateChanged + Multicast_CallStatusFlow_New

### bAutoRegister=false 是坑（Sprint 11 注释）

> 蓝图层关闭 bAutoRegister 会破坏 UActorComponent 生命周期契约 → OnRegister 不跑 → FActiveGameplayEffectsContainer::Owner / AbilityActorInfo 未初始化 → Client-predicted hit 在 UTargetTagRequirementsGameplayEffectComponent::CanGameplayEffectApply 解引用 nullptr 崩溃 @ 0x190。

**正解**：保持默认 AutoRegister=true，运行时 K2_DestroyComponent 清理 ASC（幽灵走 ItemCharacterDamageReceiver 不走 GAS 伤害链路）。DestroyComponent 是纯本地操作，不走 Replication，Server/Client 各 destroy 各自实例。

### bNewStatusFW suppress-restore（SurveillanceBird 技巧）

```lua
local savedNewStatusFW = self.bNewStatusFW
self.bNewStatusFW = false
Super.ReceiveBeginPlay(self)   -- 让基类按"老框架"路径初始化
self.bNewStatusFW = savedNewStatusFW
if savedNewStatusFW and self.ItemStatusComponent then
    self.ItemStatusComponent:SetAdvancedLogicalChains(true)
end
```

动机：基类 ReceiveBeginPlay 在 bNewStatusFW=true 路径会立刻按 LogicalChain 推 Appear，但鸟需要在 Appear 之前先 SetActorHiddenInGame(true)。临时关闭新框架开关让基类走老路径（不立即推状态），再恢复。

### LogicalSignal 链路

```
[Server BP] LogicalSignalGenerator:Activate()
        │
        ▼
[Server] ULogicalStateComponent::SetCurrentState
        │   (MARK_PROPERTY_DIRTY + Multicast)
        ▼
[Server+Client] OnLogicalStateChanged event
        │
        ▼ (Common dispatcher)
   Status_xxx() → BP_LogicalSignalReceiver:OnReceive
```

`ChangeStatue_Server`（Common 层封装）：先调旧框架 `Call_StatusFlowRaw`，再调新框架 `ServerSetAdvancedLogicalState`。两套同时写是过渡期兼容设计。

## 七、Overlap 双层感知

机关用 InnerDetectSphere / OuterDetectSphere 两个 SphereComponent，对应 bPlayerInInner / bPlayerInOuter。

**已确认使用**：GhostMechanism（bCanFlee 模式）；SurveillanceBird/TV 用 Box；HeadEye 用 ScanSphere（单层 + XVisionEye 真实视锥）。

**为什么 Overlap 替代 Tick**：
1. CPU 成本：每帧 Distance 是 O(N²)；Overlap 用 PhysX/Chaos 空间索引 O(log N)
2. 状态原子性：bool 是边沿触发，天然适合状态机入口
3. AOI 自动剔除：Sphere 在玩家 AOI 外不发事件
4. 碰撞通道运行时修正：Common 层统一 SetCollisionResponseToChannel(ECC_Pawn, ECR_Overlap)

bCanFlee=false 时 Overlap 回调直接 return，机关退化为"纯巡逻+挨打就死"。

## 八、StateTree 集成

| 机关 | StateTree | ItemStatusComponent |
|---|:---:|:---:|
| DancingSofa | ❌ | ✅ |
| DouDing | ❌ | ✅ |
| GhostMechanism | ✅ Patrol/Flee/Return | ✅ |
| HeadEye | ✅ Patrol/Spotted | ✅ |
| SurveillanceBird | ❌ | ✅ |

外层是 Item 全局生命周期（Appear/Active/Complete/Destroy），内层是 Active 阶段 AI 行为子树。

### GhostMechanism 三个 STTask
- **STTask_SplinePatrol**：EnterState `actor:EndFreeMovement()` + `StartOrResumeNSM(speed)` —— NSM 接管沿 Spline 巡逻
- **STTask_FleeFromPlayer**：EnterState `actor:BeginFreeMovement()` + `SetMovementSpeed(fleeSpeed)` —— CMC 接管 + 每帧 SmoothAddMovementInput(awayDir)
- **STTask_MoveToSpline**：Flee 完成后回归 Spline 的过渡态

### 启动方式

`StateTreeComponent:StartLogic()` **仅 Server**，蓝图开关 `bStartLogicAutomatically=false`：(a) 必须等 NSM/Spline/CMC 配置完成；(b) Client 不跑 AI。HeadEye 更晚——Status_Appear 1s 延迟后才 StartLogic，给客户端预热视觉。

## 九、CharacterMovement 平滑动力学

| 字段 | 引擎默认 | Puzzle 推荐 | 动机 |
|---|---|---|---|
| MaxWalkSpeed | 600 | PatrolSpeed (200) | 慢速观察 |
| MaxAcceleration | 2048 | **600** | 速度渐变避免方向突变"瞬移感" |
| BrakingDecelerationWalking | 2048 | **400** | 制动平滑 |
| RotationRate | (0,540,0) | **(0,360,0)** | Yaw 360°/s 平滑转向 |
| bOrientRotationToMovement | false | **true** | 朝向跟着移动方向 |
| bUseControllerDesiredRotation | true | **false** | 关掉让上面那条生效 |

**NSM 与 CMC 切换**：从 NSM 巡逻切到 Flee 自由移动时，必须 `BeginFreeMovement` 保留惯性（显式拷 prevVelocity 再恢复，避免 ForceStop 清零造成"急刹"）。

**方向插值**：SmoothAddMovementInput 用指数 lerp `alpha = 1 - exp(-DIR_LERP_SPEED * dt)`，不是硬切目标方向。

## 十、D4 Fallback (FALLBACK_DELAY = 0.5)

### 解决什么

正式地图 Entity 系统会在 BeginPlay 后短时间内调用 K2_OnLoadFromDatabaseAllFinish。但下列场景不触发：
1. Developers 测试地图（无 Entity 数据库）
2. HLE 配置缺失（Actor 没有 EditorID）
3. Entity 子系统晚于 Actor 启动

状态卡 0 → 受击/交互被 `Status != Active` 拒绝 → 机关变"哑巴"。

### Fallback 流程

```
ReceiveBeginPlay
    │
    ├─ SetTimer(0.5s) ── ► CheckAndForceInitialState()
    │                          │
    │                          ├─ if _bStateInitialized then return
    │                          └─ else ChangeStatue_Server(Appear)
    │
    └─ (并行：Entity 系统正常 < 0.5s 调 K2_OnLoadFromDatabaseAllFinish)
            └─ self._bStateInitialized = true (Common:OnLogicalStateChanged)
```

`_bStateInitialized` 在 OnLogicalStateChanged 第一次触发时置 true，D4 timer 自动判别要不要兜底。

## 十一、存盘恢复

### Save 归一化（Complete → Destroy）

```lua
function GhostMechanismServer:K2_OnSaveToDatabase()
    if raw == Enum.E_StatusFlowRaw.Complete then
        self.ItemStatusComponent.StatusFlowRaw.eStatusFlowRaw = Enum.E_StatusFlowRaw.Destroy
    end
end
```

**动机**：Complete 是 1.5s TriggerDestroy 的过渡态。玩家可能在 1.5s 窗口内断网/离开 AOI 导致存档锁在 Complete。下次入场会重播 Status_Complete（消散特效 + TriggerDestroy timer），既视觉重复又状态污染。

注意：直接改字段而**不走** `Call_StatusFlowRaw`，避免触发 OnItemStatusCondition 任务链副作用。

### Load 重广播

```lua
function GhostMechanismServer:K2_OnLoadFromDatabaseAllFinish()
    if self.ItemStatusComponent and self.ItemStatusComponent:GetStatusFlowRaw() ~= 0 then
        self.bHasSaved = true
    end
    self.Super.K2_OnLoadFromDatabaseAllFinish(self)
    self._bStateInitialized = true

    if not self.bHasSaved then return end
    local raw = self.ItemStatusComponent:GetStatusFlowRaw()

    -- 已死亡：立即销毁
    if raw == Destroy or raw == Complete then
        self:K2_DestroyActor()
        return
    end

    -- 仍存活：force broadcast Multicast 重广播
    self:ServerSetAdvancedLogicalState(stateName, true)
end
```

**两个分支语义**：
- **Destroy 直销毁**：K2_OnLoadFromDatabaseAllFinish 比 Initial Replication 还早（注释 "ReceiveBeginPlay 前 ~1ms"），Client 还没 Spawn → Server K2_DestroyActor → Client 永远不会收到 Spawn → 没有"幽灵闪现 + 重复消散特效"
- **Multicast 重广播**：bug "传送返回幽灵消失"的根因——ULogicalStateComponent::SetCurrentState 只在被调用时 MARK_PROPERTY_DIRTY，SaveGame 反序列化直接改字段不走该函数 → Push Model 不标脏 → Client OnRep_CurrentState 不触发。修法：`ServerSetAdvancedLogicalState(state, true)` 强制 force flag → 绕过 SetCurrentState 同值去重 → MARK_PROPERTY_DIRTY + 广播 → 两端 OnLogicalStateChanged 正常触发

DancingSofa 还多一个 `_ForceSaveBeforeDestroy`：在 K2_DestroyActor 前显式 `EntityComponent:K2_SaveActor()`，因为 EndPlay(Destroyed) 走的是 ActorSaveType_Destroy，IsSaveTypeNeedWriteBackProperties 返回 false → 属性不写回 → 重新登录沙发复活。

## 十二、HandleItemCharacterHitEvent

### 调用链

```
玩家攻击 → 武器 CalcComponent trace 命中
    → ItemCharacterDamageReceiver (前置：ReceiveBeginPlay 内 SetActive(true))
        → 蓝图事件路由
            → HandleItemCharacterHitEvent(Instigator, Causer, Hit, KnockInfoData)
```

```lua
function GhostMechanismServer:HandleItemCharacterHitEvent(Instigator, Causer, Hit, KnockInfoData)
    local status = self.ItemStatusComponent:GetStatusFlowRaw()
    if status ~= Enum.E_StatusFlowRaw.Active then return end  -- 非 Active 拒绝
    self:ChangeStatue_Server(Enum.E_StatusFlowRaw.Complete)
    self:GiveRewardToPlayer(Instigator)
end
```

**前置条件两条**：
1. ItemCharacterDamageReceiver:SetActive(true) 激活
2. CapsuleComponent 把 SkillDamaged (ECC_GameTraceChannel5) 设为 Overlap

"非 Active 拒绝"是关键防御——存盘恢复未完成 / Complete 1.5s 窗口内的击中都被拒绝，避免重复发奖、重复 TriggerDestroy。

## 十三、RO 模式（仅奖励发放）

GhostMechanism / DancingSofa / HeadEye 都用 `BP_Interacted_PickedItem_ROUtils`，**仅用于击杀奖励**：

```lua
local ROUtils = require("actors.common.interactable.RO.Utils.BP_Interacted_PickedItem_ROUtils")
self.ROUtils = ROUtils.new(self)
self.ROUtils:AddItemToPlayerBag(PlayerPawn, dropID)
```

**为什么用 RO 而不是直接 Multicast**：奖励发放有强一致性需求（防丢、防重、防作弊），需要 Server 权威 + Client 可预测的 RO 模型。

## 十四、5 个机关玩法速览

### DancingSofa（多轮 QTE）
玩家走近沙发按 F 坐下，沙发抽搐扭动弹出多轮方向 QTE。每轮要求按对一串方向键序列（base4 编码）。全胜 → 击飞 + 奖励；失败超过 QTEMaxFailCount → 失败击飞。怪谈头难度系数 (1.0/1.2/0.8) 动态调整序列长度和时长。

### DouDing（多块拼图）
基类 `BPA_PuzzleBase` 定义 N 块拼图嵌入机制：Server `RefreshPuzzlePiece` 按 PuzzlePieceDataList Spawn N 个 PuzzlePiece Actor 并 Attach；玩家与 PuzzlePiece 交互 → EmbedPuzzlePiece 标记 → 全嵌入触发 Status_Complete → 播 LevelSequence。

### GhostMechanism（巡逻 + 击杀奖励）
幽灵 NPC 沿 Spline 巡逻；玩家进感知圈触发 Flee 远离；玩家命中 → 一击毙命 + 消散特效 + DropID。借助灵压仪 SoulMeter 显示 Spline 路径辅助追击。

### HeadEye（巡逻 + 视觉检测）
会移动的"眼睛"NPC，沿 Spline 巡逻 + 服务端 Tick 转身朝向移动方向；XVisionEye Client 端做真实视觉检测，盯到玩家时身体转向播 Spotted 状态；命中可击杀。bQuiet 模式完全沉默不工作。

### SurveillanceBird（双 Actor 配合）
监控鸟在场景中飞行（默认隐形需电视唤醒）；电视屏幕（BP_SurveillanceTV）作为玩家"摄像头"，Box Overlap 检测玩家进入 → 电视亮起显示鸟视角 RenderTarget。鸟被击落 → Complete + 奖励。**关键技巧**：bNewStatusFW suppress-restore + 双端关闭重力和 MaxWalkSpeed 防止"碰撞关闭后穿地坠落"。

## 关键代码位置

- CommonScript/.../GhostMechanism/BPA_GhostMechanism.lua:39-64 — Initialize 字段集中
- CommonScript/.../GhostMechanism:75-84 — DestroyASC + bAutoRegister 注释
- CommonScript/.../GhostMechanism:141-149 — ChangeStatue_Server 双框架双写
- CommonScript/.../GhostMechanism:151-164 — OnLogicalStateChanged dispatcher
- CommonScript/.../GhostMechanism:275-311 — SetupNSM Detach(KeepWorld)
- ServerScript/.../GhostMechanism:24 — FALLBACK_DELAY = 0.5
- ServerScript/.../GhostMechanism:32-63 — Server ReceiveBeginPlay 标杆
- ServerScript/.../GhostMechanism:185-240 — K2_OnLoadFromDatabaseAllFinish 重广播 vs 直销毁
- ServerScript/.../GhostMechanism:248-257 — K2_OnSaveToDatabase 归一化
- ServerScript/.../GhostMechanism:264-276 — CheckAndForceInitialState D4 fallback
- ServerScript/.../GhostMechanism/StateTree/STTask_SplinePatrol.lua:15-36
- ServerScript/.../HeadEye:54 — DamageReceiver SetActive
- ServerScript/.../HeadEye:112-135 — Status_Appear 1s 延迟 → StateTree.StartLogic
- CommonScript/.../SurveillanceBird:40-62 — bNewStatusFW suppress-restore
- ServerScript/.../DancingSofa:107-121 — _ForceSaveBeforeDestroy
- CommonScript/actors/Puzzle/base/BPA_PuzzleBase.lua:67-90 — RefreshPuzzlePiece
