---
id: npc-14
title: BehaviorTree 与战斗 NPC
sources:
  - Projects/HiGame/Content/Script/ServerScript/ai/BTCommon/BTTask_Base.lua
  - Projects/HiGame/Content/Script/ServerScript/ai/BTTask/BossPunk/BTTask_HI_CallMovePlatform.lua
  - Projects/HiGame/Content/Script/ServerScript/ai/BTTask/DonNavigation/BTTask_HI_OnGoalReached.lua
  - Projects/HiGame/Content/Script/ServerScript/ai/BTTask/DonNavigation/BTTask_HI_PickGoal.lua
  - Projects/HiGame/Content/Script/ServerScript/ai/BTTask/DonNavigation/BTTask_HI_RetryWithDelay.lua
  - Projects/HiGame/Content/Script/ServerScript/ai/BTDecorator/Boss/BTDecorator_HI_IsDogInTrap.lua
  - Projects/HiGame/Source/HiGame/Public/HiAI/BT/BTCompositeNode/HiBTCompositeNode_RandomWeight.h
  - Projects/HiGame/Source/HiGame/Private/HiAI/BT/BTCompositeNode/HiBTCompositeNode_RandomWeight.cpp
  - Projects/HiGame/Source/HiGame/Public/HiAI/BT/BTTask/HiBTTask_SetRandomWeight.h
updated: 2026-05-11
---

# npc-14 — BehaviorTree 与战斗 NPC

## Sources

- `Projects/HiGame/Content/Script/ServerScript/ai/BTCommon/BTTask_Base.lua`
- `Projects/HiGame/Content/Script/ServerScript/ai/BTTask/BossPunk/*.lua`
- `Projects/HiGame/Content/Script/ServerScript/ai/BTTask/DonNavigation/*.lua`
- `Projects/HiGame/Content/Script/ServerScript/ai/BTDecorator/Boss/*.lua`
- `Projects/HiGame/Content/Script/ServerScript/ai/BTService/*.lua`
- `Projects/HiGame/Source/HiGame/Public/HiAI/BT/BTCompositeNode/HiBTCompositeNode_RandomWeight.h`
- `Projects/HiGame/Source/HiGame/Private/HiAI/BT/BTCompositeNode/HiBTCompositeNode_RandomWeight.cpp`
- `Projects/HiGame/Source/HiGame/Public/HiAI/BT/BTTask/HiBTTask_SetRandomWeight.h`

> 注: 题目中给出的 `Content/Script/ai/BTTask/...` 路径在仓库中实际位于 `Content/Script/ServerScript/ai/...`,
> 即 BT Lua 节点是 **DDS 服务端脚本** (UnLua 服务器虚拟机),客户端没有镜像目录。
> `Source/HiGame/Public/BT/BTCompositeNode/` 目录存在但为空,真正的实现位于 `Source/HiGame/Public/HiAI/BT/BTCompositeNode/`。

## 1. 概念

HiGame 战斗向 NPC (Boss、AnimalAI、SummonedCreature、敌方 NPC 等) 走 **UE5 标准 BehaviorTree + Blackboard**
管线,而 **不是** 走剧情 NPC 的 `NpcActiveObject + Statesflow + NodeComponent` 那条路径 (见 npc-02 / npc-03)。
BT 的 **owner controller** 是 `AHiAIController` (见 npc-12),Pawn 上挂 `AIServerComponent` 与
`UtilsComponent` 暴露感知/状态查询接口给 BT 节点。

BT 的 Task / Decorator / Service 既可以是 C++ (`UBTTaskNode` / `UBTDecorator` 子类),
也可以是 **蓝图 + UnLua 绑定**:`BP_BT*_C` 蓝图实例化 lua class,在 `ReceiveExecuteAI / ReceiveAbortAI / ReceiveTickAI`
回调中转发到 lua 协程。HiGame 把 BT 子系统按 "流派 (faction)" 划分目录,典型两支:

- **BossPunk** — 关卡 boss 战相关的特化 task,例如召唤 / 平台机关 / 阶段切换。
- **DonNavigation** — 寻路向 task,源自社区 DonAINavigation 思想 (PickGoal / OnGoalReached / RetryWithDelay 三件套)。

其余流派目录还包括 `Animal/`、`NPC/`、`SummonedCreature/`、`Boss/` (decorator),以及无前缀的通用 BTTask / BTDecorator / BTService。

## 2. BT Lua Task / Decorator / Service 全表

> 仅列举 `BossPunk/`、`DonNavigation/`、`BTDecorator/Boss/` 与 BTCommon 基类(题目要求范围),
> 其余流派在表末以小计形式概括。

| 文件 | 流派目录 | 类型 | 绑定 BP (推断命名) | 一句话职责 |
|---|---|---|---|---|
| `BTCommon/BTTask_Base.lua` | BTCommon | Task 基类 | (lua 基类,被各 BTTask 继承) | 统一 `Execute / Tick / Abort` 协程模板,管理 `RegisterBTSwitchCB` / `CanBreak` / `OnFinish` |
| `BTTask/BossPunk/BTTask_HI_CallMovePlatform.lua` | BossPunk | Task | `BP_BTTask_HI_CallMovePlatform_C` | 通过 `GameAPI.SpawnActor` 召出 `Pawn.MovePlatformActorClass` 移动平台,并 `SendMessage("OnMovePlatformActorCreate", ...)` |
| `BTTask/DonNavigation/BTTask_HI_PickGoal.lua` | DonNavigation | Task | `BP_BTTask_HI_PickGoal_C` | 从 Pawn 的 `GetGoalActor()` 取下一个目标 transform,通过 `UBTFunctionLibrary.SetBlackboardValueAsVector` 写入 BB 的 `GoalKey` |
| `BTTask/DonNavigation/BTTask_HI_OnGoalReached.lua` | DonNavigation | Task | `BP_BTTask_HI_OnGoalReached_C` | 调用 Pawn 的 `OnGoalReached()` 取等待时间 → `utils.DoDelay` 延时后 `FinishExecute(true)` |
| `BTTask/DonNavigation/BTTask_HI_RetryWithDelay.lua` | DonNavigation | Task | `BP_BTTask_HI_RetryWithDelay_C` | `utils.DoDelay(Pawn, self.DefaultWaitTime, ...)` 后无条件返回 success,等价于"等一拍再重试外层节点" |
| `BTDecorator/Boss/BTDecorator_HI_IsDogInTrap.lua` | Boss(decorator) | Decorator | `BP_BTDecorator_HI_IsDogInTrap_C` | `PerformConditionCheck` 直接读 `Pawn.UtilsComponent.DogInTrap` 布尔位作为分支条件 |

**其它流派小计** (位于 `BTTask/`、`BTDecorator/`、`BTService/` 根或同级子目录):

- `BTTask/Animal/*` — 6 个动物 AI task (`AnimalAlertAnim`, `AnimalBehavior_CalcNextState`, `AnimalBehavior_Initialization`, `AnimalEscape`, `AnimalFly`, `AnimalMoveViaWaypoint`, `AnimalStroll`)。
- `BTTask/NPC/*` — 6 个剧情/敌方 NPC task (`NPCChasePlayer`, `NPCDead`, `NPCFreeze`, `NPCMoveTo`, `NPCPlayAnimation`, `NPCShowBubble`)。
- `BTTask/SummonedCreature/*` — 召唤生物 task (`HISC_MoveToMaster`, `HISC_MoveToMasterTarget`)。
- `BTTask/BTTask_HI_*` — 通用 70+ 个根级 task,覆盖移动 (`MoveToTarget`/`MoveToLocation`/`SprintToLocation`/`TeleportTo`)、技能 (`TryActiveAbility*`/`SmartTryActiveAbility`/`TryCounterattackAbility`)、动画 (`PlayAnim`/`PlaySequence`/`PlayIdleAnimation`/`PlayEnterBattleAnim`)、感知 (`ObserveTarget`/`ObserveConfig`/`AddObserveIndex`/`SetFocus*`)、Buff/GE (`AddBuffAround`/`RemoveBuffByID`/`RemoveGameplayEffect_C`)、巡逻 (`WayPointBehavior`/`WayPointMovingToNext`)、镜像怪物 (`MirrorMonster`/`MirrorPatrol`/`MirrorPresent`)、调试 (`Debug`/`TestMoving`) 等。
- `BTDecorator/*` (根级) — 40+ 个通用 decorator,覆盖距离/能量/血量/韧性/技能 CD/陷阱判定/方位扇区/概率/翻转开关 等条件。
- `BTDecorator/NPC/*` — `BTDecorator_HI_IsInPlayerView.lua`。
- `BTDecorator/SummonedCreature/*` — `BTDecorator_HISC_IsMasterInBattle.lua`。
- `BTService/*` — 12 个 service:`CheckAbilityHitTarget`、`EnterAlertState`、`PrepareToCapture`、`SetBuildingsCanBlast`、`SetCounter`、`SetDesiredRotationMode`、`SetMoveType`、`SetRandomWeight`、`SetSpeed`、`SetSpeedByConfig`、`WaitAction_Dist`、`WaitAction_Energy`。

## 3. Lua BTTask 实现模式 (示例)

`BTCommon/BTTask_Base.lua` 是所有战斗 lua BTTask 的 **协程模板基类**,绑定到 UE 的 `ReceiveExecuteAI` /
`ReceiveAbortAI` / `ReceiveTickAI` 三个 lua 重载点(继承自蓝图基类 `BTTask_BlueprintBase`)。

**关键模式**:

```lua
function BTTask_Base:ReceiveExecuteAI(Controller, Pawn)
    xpcall(function(...)
        Pawn:SendMessage("RegisterBTSwitchCB", self, self.OnSwitch)
        local ret = self:Execute(Controller, Pawn)
        if ret == ai_utils.BTTask_Succeeded then
            self:TaskFinish(Controller, Pawn, true)
        elseif ret == ai_utils.BTTask_Failed then
            self:TaskFinish(Controller, Pawn, false)
        end
    end, function(err) ... end)
end
```

子类只需实现 `Execute(Controller, Pawn)`,**返回三种状态**:

- `ai_utils.BTTask_Succeeded` → 立即 `FinishExecute(true)`。
- `ai_utils.BTTask_Failed` → 立即 `FinishExecute(false)`。
- 不返回值 → 进入 **延迟完成模式**,task 自己择机调用 `self:FinishExecute(bool)` (例如 `BTTask_HI_OnGoalReached`
  中的 `utils.DoDelay(Pawn, WaitTime, function() self:FinishExecute(true) end)`)。

`ReceiveTickAI` 走相同 xpcall 包装,先 `CanBreak` 检查 `PawnAIServerComponent.bCanBreakCurBTNode`,
满足则 `OnBreak` + `FinishExecute(true)` + `SendMessage("SetCurBTNodeBreak", false)` 主动让位。
否则调 `self:Tick(Controller, Pawn, DeltaSeconds, PawnAIServerComponent)`,语义同 `Execute` 三态返回。

`OnSwitch / OnBreak / OnAbortAI / OnFinish` 是给子类的 hook 钩子,基类提供空实现。
统一异常处理: 任何 `xpcall` 抛出都打印 PawnDisplayName + `debug.traceback()` 并以 `false` 结束。

**示例 1 — 同步立即返回 (BossPunk/CallMovePlatform)**:

```lua
function BTTask_CallMovePlatform:Execute(Controller, Pawn)
    if Pawn.MovePlatformActor ~= nil then
        return ai_utils.BTTask_Failed
    end
    local MovePlatformActor = GameAPI.SpawnActor(Pawn:GetWorld(), Pawn.MovePlatformActorClass,
        Pawn:GetTransform(), UE.FActorSpawnParameters(), {})
    Pawn:SendMessage("OnMovePlatformActorCreate", MovePlatformActor)
    return ai_utils.BTTask_Succeeded
end
```

**示例 2 — 延迟完成 (DonNavigation/OnGoalReached)**:

```lua
function BTTask_HI_OnGoalReached:Execute(Controller, Pawn)
    if Pawn and Pawn.OnGoalReached then
        local WaitTime, _ = Pawn:OnGoalReached()
        utils.DoDelay(Pawn, WaitTime, function()
            self:FinishExecute(true)
        end)
    else
        self:FinishExecute(false)
    end
end
```

注意: `Execute` 没有 `return`,因此基类不会立即 finish;`utils.DoDelay` 回调里手动 finish。这是 lua BT 协程模式的典型用法。

## 4. Lua BTDecorator 实现模式 (示例)

Decorator 比 Task 简单,因为它只回答 "条件成立否",没有协程。基类直接是 `Class()`,
重载 `PerformConditionCheck(Controller)` 返回 bool。

`BTDecorator/Boss/BTDecorator_HI_IsDogInTrap.lua` 完整内容:

```lua
require "UnLua"
local G = require("G")

local BTDecorator_IsDogInTrap = Class()

function BTDecorator_IsDogInTrap:PerformConditionCheck(Controller)
    local Pawn = Controller:GetInstigator()
    return Pawn.UtilsComponent.DogInTrap
end

return BTDecorator_IsDogInTrap
```

可以看到:

- Decorator 没有继承 `BTTask_Base` (那是 Task 协程基类),直接 `Class()` 起手。
- 通过 `Controller:GetInstigator()` 拿到 Pawn,而不是 `BTComponent:GetAIOwner():GetPawn()` 这条更长的链路。
- 直接读 `Pawn.UtilsComponent.<flag>` 这种 **预聚合状态位**,即把 EQS / sensing 的结果先在 C++ 侧
  存到 `UtilsComponent` 的 bool 上 (避免每次 BT tick 重做 trace)。

## 5. C++ BTCompositeNode 子集

> `Source/HiGame/Public/BT/BTCompositeNode/` 与 `Source/HiGame/Private/BT/BTCompositeNode/` 这两个目录虽然存在,
> 但 **没有任何头/源文件**。HiGame 真正的自定义 BT C++ 节点全部集中在 `Source/HiGame/Public/HiAI/BT/` 下。

| 头文件 | 类 | 简介 |
|---|---|---|
| `HiAI/BT/BTCompositeNode/HiBTCompositeNode_RandomWeight.h` | `UHiBTCompositeNode_RandomWeight : UBTCompositeNode` | 按权重随机选一个子分支执行的 composite 节点。子分支失败后从剩余分支重新加权抽取,直到全部失败才回到父节点 |
| `HiAI/BT/BTTask/HiBTTask_SetRandomWeight.h` | `UHiBTTask_SetRandomWeight : UBTTaskNode` | 配套 task: 把指定 NodeName 的 RandomWeight 节点的覆盖权重以 JSON 写入 Blackboard `ExtraRandomWeight` 键,实现 per-AI 实例化的权重微调 |

`UHiBTCompositeNode_RandomWeight` 实现要点 (`HiBTCompositeNode_RandomWeight.cpp`):

- 重写 `GetNextChildHandler(SearchData, PrevChild, LastResult)`,默认返回 `BTSpecialChild::ReturnToParent`(成功即退出)。
- 失败重选: 当 `PrevChild == NotInitialized` (首次进入) 或上次结果非 Succeeded 时,把 `PrevChild`
  加入私有的 `FailedChildren` 列表;若列表已等于 `GetChildrenNum()`,直接 `ReturnToParent` (全部失败放弃)。
- 权重来源: 先尝试从 Blackboard 字符串键 `ExtraRandomWeight` 反序列化 JSON,
  通过 `RootObject->TryGetArrayField(NodeName, WeightArray)` 按 NodeName 查表;若不存在,则用 CDO 配置的 `WeightPerBranch`。
- 已失败子节点的权重清零,避免再次抽中 (`Weights[i] = 0`)。
- 用前缀和 `WeightSumPerBranch` 在 `[0, totalWeight]` 上做一次 `FMath::RandRange`,落入哪段就选那条分支。

`UPROPERTY` 字段:`TArray<int32> WeightPerBranch` (EditInstanceOnly)。
私有成员 `FailedChildren` 是 `new` 出来的 `TArray<int32>*`,在析构里 `Empty()` + `delete`。

## 6. BT Service / BTCommon 目录

- `Content/Script/ServerScript/ai/BTService/` 共 12 个 service lua,**非空**(见 §2 末尾小计)。
- `Content/Script/ServerScript/ai/BTCommon/` 仅有一个文件 `BTTask_Base.lua`,即 §3 的协程基类。
- `Content/Script/ai/` 下题目所写的根级 `BTTask/`、`BTDecorator/`、`BTService/`、`BTCommon/` 子目录
  在仓库实际为 **空目录** (3 月 13 日创建后未 check in 任何 lua),所有内容都在 `ServerScript/ai/` 镜像里。
  这与 HiGame 把 AI 决策放在 DDS server 侧、客户端只跑表现的架构一致。

## 7. 与 HiAIController 的握手 (npc-12)

- BT 的 `AIOwner` = `AHiAIController` (npc-12 第 1 节);它持有 `BehaviorTreeComponent` 与 `BlackboardComponent`,
  并把 Pawn 注入为 `Controller:GetInstigator()` 返回值。
- BTTask lua 几乎都通过 `Pawn:SendMessage("...", ...)` 与 Pawn 上的 `UtilsComponent` / `AIServerComponent` 通信,
  而不是直接调 controller 方法。这把 BT 节点变成纯"消息发出者",controller 不会被业务塞满。
- Blackboard 典型键 (从代码反推):
  - `GoalKey` (Vector) — DonNavigation 的当前导航目标。
  - `ExtraRandomWeight` (String, JSON) — RandomWeight composite 的权重覆盖。
  - 其它常见键还包括 TargetActor / TargetLocation / FocalPoint / Counter 等 (与各 BTTask_HI_Set* 命名对应)。
- AIServerComponent 暴露 `bCanBreakCurBTNode` 给 `BTTask_Base:CanBreak`,允许外部 (受击/换阶段)
  请求当前节点提前结束,通过 `SetCurBTNodeBreak(false)` 复位。

## 8. 与 ActiveObject NPC 路径的分野

战斗 NPC **不走** `NpcActiveObject + StateFlow + NodeComponent`(npc-02 / npc-03)那套数据驱动状态机,
而是 **AHiAIController + BehaviorTree + Blackboard** + lua BTTask 协程基类。两条路径职责清晰分开,
互不重叠,但通过 `NpcBehaviorTreeTask` (npc-10) 这一 ActiveObject 节点可以在剧情流程里包一个 BT 子树作为子任务执行。

| 维度 | 剧情 NPC (npc-02/03) | 战斗 NPC (本页) |
|---|---|---|
| 决策框架 | NpcActiveObject + StatesFlow (节点链) | UE BehaviorTree + Blackboard |
| Controller | 默认 AIController / 业务 Controller | `AHiAIController` |
| 节点扩展点 | NodeComponent (lua) `OnNodeEnter / Tick / Exit` | BTTask/BTDecorator/BTService (lua + BP) |
| 基类 | `NpcActiveObject` (lua) | `BTTask_Base` (lua,见 §3) |
| 跨节点状态 | 节点参数表 + State Variable | Blackboard Key (Vector/Float/String/Object) |
| 中断协议 | StopReason 枚举(npc-15) | `bCanBreakCurBTNode` + `OnSwitch / OnBreak / OnAbortAI` hooks |
| 典型用例 | NPC 寒暄/送任务/巡逻/收菜 | Boss 多阶段战斗、动物 AI、召唤物、敌方 NPC 战斗 |
| C++ 自定义节点 | 一批 NodeComponent 子类 (`NodeComponentXXX`) | `UHiBTCompositeNode_RandomWeight` + `UHiBTTask_SetRandomWeight` 等少量 |

## 9. 跨页关联

- **npc-12 (HiAIController)** — BT 的 owner,挂 BehaviorTreeComponent / BlackboardComponent,
  `Controller:GetInstigator()` 是 lua 节点拿 Pawn 的标准入口。
- **npc-10 (NpcBehaviorTreeTask)** — 在 ActiveObject 路径中包装一棵 BT 子树作为剧情节点的执行单元。
- **npc-15 (Enum)** — `Enum_BehaviorTreeTaskType` / `EBehaviorTreeType` 与 `EStopDialogueReason`
  为 BT 与剧情流之间的协议常量(BT 任务类型分类、对话被 BT 强制中断的原因码等)。
