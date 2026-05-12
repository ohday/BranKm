---
id: npc-12
title: HiAIController + PathFollowing + Avoidance
sources:
  - Projects/HiGame/Source/HiGame/Public/HiAI/HiAIController.h
  - Projects/HiGame/Source/HiGame/Public/HiAI/HiAvoidanceManager.h
  - Projects/HiGame/Source/HiGame/Public/HiAI/HiDetourCrowdAIController.h
  - Projects/HiGame/Source/HiGame/Public/HiAI/HiPathFollowingComponent.h
  - Projects/HiGame/Source/HiGame/Private/HiAI/HiAIController.cpp
  - Projects/HiGame/Source/HiGame/Private/HiAI/HiAvoidanceManager.cpp
  - Projects/HiGame/Source/HiGame/Private/HiAI/HiDetourCrowdAIController.cpp
  - Projects/HiGame/Source/HiGame/Private/HiAI/HiPathFollowingComponent.cpp
updated: 2026-05-11
---

# npc-12 — HiAI Controller 与寻路系统

## Sources

见 frontmatter `sources` 字段(8 个文件:`HiAI/` 下 4 个 `.h` + 4 个 `.cpp`)。

## 1. HiAI 概念

`HiAI/` 目录是 HiGame 在 UE5 内置 AIController 体系之上的**轻量扩展层**:它继承
`AAIController`、`ADetourCrowdAIController`、`UPathFollowingComponent`、
`UAvoidanceManager` 这四个引擎类,并通过实现 `IHiProximityTickInterface`
把它们接入 HiGame 自研的 *Proximity Tick*(按距离玩家分级降频)子系统。它本身
不重写寻路算法,只是包装 BehaviorTree 启停接口、把 Avoidance 接口暴露给自定义
`UHiNavAvoidBoxComponent`,并提供 GameplayDebugger 类目用于运行时调试。

## 2. C++ 类层次

| 类名 | 继承 | 文件 | 一句话职责 |
|---|---|---|---|
| `AHiAIController` | `AAIController`, `IHiProximityTickInterface` | `HiAIController.h/.cpp` | NPC 默认 AIController,封装 BT 启停 + 调试输出 |
| `AHiDetourCrowdAIController` | `ADetourCrowdAIController`, `IHiProximityTickInterface` | `HiDetourCrowdAIController.h/.cpp` | 启用 DetourCrowd 的 AIController,只接 Proximity 节流 |
| `UHiPathFollowingComponent` | `UPathFollowingComponent`, `IHiProximityTickInterface` | `HiPathFollowingComponent.h/.cpp` | PathFollowing 的 Proximity Tick 包装 |
| `UHiAvoidanceManager` | `UAvoidanceManager` | `HiAvoidanceManager.h/.cpp` | 重写 RVO `GetAvoidanceVelocity_Internal`,支持 NavAvoidBox |
| `FHAIRepData` | (struct) | `HiAIController.h` | 调试器复制用的数据包,持有 `TArray<FString> DebugStrs` |
| `FGameplayDebuggerCategory_HiAI` | `FGameplayDebuggerCategory` | `HiAIController.h/.cpp` | GameplayDebugger 中的 "HiAI" 类目 |

## 3. AHiAIController

### 3.1 类声明

```cpp
UCLASS()
class HIGAME_API AHiAIController : public AAIController, public IHiProximityTickInterface
{
    GENERATED_BODY()
public:
    AHiAIController();
    UFUNCTION(BlueprintCallable, Category = "AI") void StopBehaviorTree();
    UFUNCTION(BlueprintCallable, Category = "AI") void PauseBehaviorTree(const FString& Reason);
    UFUNCTION(BlueprintCallable, Category = "AI") void ResumeBehaviorTree(const FString& Reason);
    UFUNCTION(BlueprintCallable, Category = "AI") FString GetAIDebugInfo() const;
    UFUNCTION(BlueprintCallable, Category = "AI") FString GetBTDebugInfo() const;

    void Tick(float DeltaTime) override;
    void BeginPlay() override;
    void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

    EHiProximityTickCategory GetProximityTickCategory() const override;
    void SetProximityTickCategory(EHiProximityTickCategory NewCategory, int NewTickRoundGap) override;
private:
    FHiProximityTickCtrl ProximityTickCtrl;
};
```

### 3.2 UPROPERTY 全表

`AHiAIController` 自身不声明任何 `UPROPERTY`,只有一个**私有非 UPROPERTY** 的
`FHiProximityTickCtrl ProximityTickCtrl`。其余成员(`BrainComponent` 等)继承自
`AAIController`。

### 3.3 UFUNCTION 全表

| 函数 | 反射标记 | 类别 |
|---|---|---|
| `StopBehaviorTree()` | `BlueprintCallable, Category = "AI"` | BT 控制 |
| `PauseBehaviorTree(const FString& Reason)` | `BlueprintCallable, Category = "AI"` | BT 控制 |
| `ResumeBehaviorTree(const FString& Reason)` | `BlueprintCallable, Category = "AI"` | BT 控制 |
| `GetAIDebugInfo() const` | `BlueprintCallable, Category = "AI"` | 调试 |
| `GetBTDebugInfo() const` | `BlueprintCallable, Category = "AI"` | 调试 |

### 3.4 关键方法签名与实现要点

构造函数仅设置 `bAttachToPawn = true`(挂到 Pawn 上)。

```cpp
AHiAIController::AHiAIController() { bAttachToPawn = true; }
```

BT 三件套(`StopBehaviorTree` / `PauseBehaviorTree` / `ResumeBehaviorTree`)
共同模式:把 `BrainComponent` 强转为 `UBehaviorTreeComponent`,如果为空则
新建 `BTComponent` 并替换 `BrainComponent`,然后调用 `StopTree(Safe)` /
`PauseLogic(Reason)` / `ResumeLogic(Reason)`。

`GetAIDebugInfo()` 输出格式:`Behavior:[Running|Paused|Inactive], Tree:[...] Active task:[...]`,
内部走 `BTComp->DescribeActiveTasks()` / `DescribeActiveTrees()`。
`GetBTDebugInfo()` 直接 `BrainComponent->GetDebugInfoString()`。

### 3.5 Proximity Tick 集成

`Tick()` 把 `DeltaTime` 交给 `ProximityTickCtrl.Tick()`,返回值 `<=0` 则跳过本帧
`Super::Tick`,达到按距离降频的目的。`BeginPlay` 在 server 上向
`UHiProximityTickSubsystem` 注册自己,`EndPlay` 反注册。

```cpp
void AHiAIController::Tick(float DeltaTime)
{
    DeltaTime = ProximityTickCtrl.Tick(DeltaTime);
    if (DeltaTime <= 0) return;
    Super::Tick(DeltaTime);
}
```

## 4. AHiDetourCrowdAIController

```cpp
UCLASS()
class HIGAME_API AHiDetourCrowdAIController : public ADetourCrowdAIController, public IHiProximityTickInterface
{
    GENERATED_BODY()
public:
    AHiDetourCrowdAIController(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());
    void Tick(float DeltaTime) override;
    void BeginPlay() override;
    void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
    EHiProximityTickCategory GetProximityTickCategory() const override;
    void SetProximityTickCategory(EHiProximityTickCategory NewCategory, int NewTickRoundGap) override;
private:
    FHiProximityTickCtrl ProximityTickCtrl;
};
```

差异点:
- 父类不同 — 继承的是引擎 `ADetourCrowdAIController`,自动接入 DetourCrowd
  群体寻路(由 `UCrowdManager` 调度)。
- 没有 `StopBehaviorTree` / `PauseBehaviorTree` 等接口 — 这个 AIController
  目前只用于"群体生物/避让密集"场景,BT 控制走父类。
- `Tick / BeginPlay / EndPlay` 和 `AHiAIController` 完全同构,只接 Proximity Tick。

## 5. UHiPathFollowingComponent 状态机

```cpp
UCLASS()
class HIGAME_API UHiPathFollowingComponent : public UPathFollowingComponent, public IHiProximityTickInterface
{
    GENERATED_UCLASS_BODY()
    void TickComponent(float DeltaTime, enum ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
    void BeginPlay() override;
    EHiProximityTickCategory GetProximityTickCategory() const override;
    void SetProximityTickCategory(EHiProximityTickCategory NewCategory, int NewTickRoundGap) override;
private:
    FHiProximityTickCtrl ProximityTickCtrl;
};
```

### 5.1 状态枚举

`UHiPathFollowingComponent` **没有**自定义状态机,直接复用父类
`UPathFollowingComponent` 提供的 `EPathFollowingStatus`(Idle / Waiting /
Paused / Moving)和 `EPathFollowingResult` / `EPathFollowingRequestResult`。

### 5.2 关键回调

派生层只重写两个生命周期回调:
- `TickComponent` — 套上 `ProximityTickCtrl.Tick`,通过则透传给父类。
- `BeginPlay` — 调 `Super`,然后 `ProximityTickCtrl.Reset()`。

`OnPathFinished` / `RequestMove` / `AbortMove` / `OnLanded` 等业务回调**全部
继承父类**,没有 HiGame 侧的覆写或额外处理。

### 5.3 与 npc_const 的对应

Lua 侧 `npc_const.Enum_PathFollowingResult` 与
`npc_const.Enum_PathFollowingRequestResult` 是对引擎枚举
`EPathFollowingResult::Type` / `EPathFollowingRequestResult::Type` 的镜像
(具体值映射在 npc-15 中展开)。`UHiPathFollowingComponent` 不改变这些枚举。

## 6. UHiAvoidanceManager

```cpp
UCLASS(Blueprintable)
class HIGAME_API UHiAvoidanceManager : public UAvoidanceManager
{
    GENERATED_UCLASS_BODY()
protected:
    FVector GetAvoidanceVelocity_Internal(const FNavAvoidanceData& inAvoidanceData,
                                          float DeltaTime, int32* inIgnoreThisUID) override;
public:
    int RegisterAvoidanceBoxComponent(UHiNavAvoidBoxComponent* NavAvoidanceBox);
};
```

### 6.1 算法

继承自引擎 `UAvoidanceManager`(RVO 风格)。重写
`GetAvoidanceVelocity_Internal` — `.cpp` 内部会:

1. 检查 `bSystemActive` 和 `DeltaTime > 0`,无效则返回原速度。
2. `MaxSpeed = ReturnVelocity.Size2D()`,若 `< 0.01` 直接 push forward。
3. 调用文件级辅助函数 `HiAvoidCones(AllCones, BasePosition, DesiredPosition,
   NumConesToTest)` — 这是一个**速度空间锥形规避**算法:每个动态障碍生成一对
   `FVelocityAvoidanceCone::ConePlane[2]` 平面,逐锥裁剪 `CurrentPosition`,
   被完全卡住时返回 `BasePosition`。
4. 调用 `HiAvoidsNavEdges(OrgLocation, TestVelocity, NavEdges, MaxZDiff)` —
   通过 `FNavEdgeSegment` 数组判断目标速度是否会穿越导航网边缘,使用 2D 叉积
   解直线相交。

### 6.2 与 NavAvoidBox 的协作

新增公开接口 `RegisterAvoidanceBoxComponent(UHiNavAvoidBoxComponent*)` —
让自定义的 `UHiNavAvoidBoxComponent`(声明在 `Component/HiNavAvoidComponent.h`)
作为静态规避盒子注入进 RVO 流程,弥补 `UAvoidanceManager` 默认只支持
"圆柱体 Agent" 不支持长方形障碍的问题。

### 6.3 与 PathFollowing 的协作

`UHiAvoidanceManager` 是 *被动调用方*:`UCharacterMovementComponent`
(或派生 `UHiNPCMovementComponent`,见 npc-13)在每帧速度更新时调用
`GetAvoidanceVelocity` → 走到 `_Internal`。`UHiPathFollowingComponent`
本身**不直接调用** Avoidance,它只把"沿路径前进的期望速度"喂给 MovementComp,
后者再让 AvoidanceManager 修正,最终速度才落到角色。

## 7. 与 NPC 系统的耦合点

- `NpcActiveObject`(Lua,见 npc-02)**不直接持有** `AHiAIController`。它持有
  `UE.NpcActor`(C++ Actor)的引用,通过 `npc_actor:GetController()` 获取
  `AHiAIController`(或 `AHiDetourCrowdAIController`)。
- 移动 mail(`Move_To_Way_Point` / `Move_To_Actor`,见 npc-09)在 Lua 状态里
  最终会调用 `AIController->MoveToLocation` / `MoveToActor`(继承自
  `AAIController`),引擎内部 → 创建 `FAIMoveRequest` → 走
  `UHiPathFollowingComponent::RequestMove`(实际使用父类 `RequestMove`)→
  完成时回调 `OnRequestFinished` 派发 `EPathFollowingResult`,这个枚举值
  通过桥接 mail 反馈回 Lua 状态机(npc_const.Enum_PathFollowingResult)。
- BT 启停 mail / 动作:`Action_Pause_BT` / `Action_Resume_BT`
  最终落到 `AHiAIController::PauseBehaviorTree` / `ResumeBehaviorTree`。
- `AHiCharacter::CollectAIDebugData` / `DrawAIDebugData` 是
  `FGameplayDebuggerCategory_HiAI` 的数据源,意味着 NPC Character 持有自己的
  调试字符串集合,被 GameplayDebugger 同步到客户端。

## 8. 跨页关联

- **npc-13** — `UHiNPCMovementComponent` 是 PathFollowing 真正的"执行者",
  与本页 `UHiPathFollowingComponent` 协作:PathFollowing 算"去哪里",
  MovementComponent 算"怎么走到那"。
- **npc-14** — BT(BehaviorTree)的 owner 一律是 `AHiAIController`;
  本页的 `StopBehaviorTree` / `PauseBehaviorTree` / `ResumeBehaviorTree`
  是 BT 生命周期的入口。
- **npc-15** — Lua 侧 `npc_const.Enum_PathFollowingResult` 与
  `npc_const.Enum_PathFollowingRequestResult` 即引擎枚举的镜像,本页
  `UHiPathFollowingComponent` 是产生这些枚举值的源头。
- **npc-09** — Smart Object/WayPoint 系统通过 mail 触发 `MoveToLocation`,
  从而进入本页描述的 Controller + PathFollowing + Avoidance 链路。
- **Proximity Tick** — 本页四个类全部接入 `IHiProximityTickInterface` /
  `FHiProximityTickCtrl` / `UHiProximityTickSubsystem`,这是 HiGame 控制
  AI 服务器开销的统一节流框架,值得另起页面单独整理。
