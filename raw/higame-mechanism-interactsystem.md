---
id: 55
title: "HiGame InteractSystem C++ Module"
source: local-archaeology
project_root: E:\HiProject\PerforceDev\UnrealEngine\Projects\HiGame
module: Source/HiGame/{Public,Private}/InteractSystem/
fetched: 2026-05-11
status: content-verified
covers_questions: [Q1, Q2, Q3, Q4, Q5]
---

# HiGame InteractSystem 代码考古

## 文件清单

| 文件 | 一句话职责 |
|---|---|
| `InteractSystem.h` | 模块共享头：LogInteractSystem、TRACE_CHANNEL、EInteractItemState、EInteractAction、FInteractQueryParam、FInteractInfo |
| `InteractItemComponent.h/.cpp` | 可交互物件挂载点（USceneComponent 子类），状态机本体 |
| `InteractCharacterComponent.h/.cpp` | UInteractItemComponent 的子类，给 AHiCharacter 用（当前实现里 FindInteractInfo 被注释，仅留壳） |
| `InteractRangeInterface.h/.cpp` | IInteractRangeInterface：把 PrimitiveComponent 反向关联到 InteractItemComponent |
| `InteractRangeComponent.h/.cpp` | UInteractSphereRangeComponent / UInteractBoxRangeComponent 两种范围 collision |
| `InteractManagerComponent.h/.cpp` | 挂在 AHiPlayerController 上的本地客户端管理器 |
| `InteractWidget.h/.cpp` | 跟随世界坐标的 UMG，订阅 OnInteractActionDelayTime |
| `InteractWidgetUpdater.h/.cpp` | 单例 Actor (PerModuleDataObjects)，TG_PostUpdateWork Tick 中刷新所有 Widget |
| `CustomInteractExecutor.h/.cpp` | UObject Blueprintable 基类，IA_CustomExecutor 路径用 |
| `UInteractExecutorInterface.h/.cpp` | IInteractExecutorInterface + IInteractLevelExecutorInterface 两个接口 |

## 整体流程

```
[Item Actor 持有 UInteractItemComponent]
                │
InitializeComponent ▼
NewObject<UInteractSphereRangeComponent>
→ SetSphereRadius(max(Prompt,Focus))
→ SetCollisionProfileName("InteractRange")
→ AttachToComponent(this)

┌────────────────────────────────────────────────────────────────┐
│   AHiPlayerController.InteractManager (UInteractManagerComp)    │
│   BeginPlay → PushInputComponent → OnPossessEvent              │
│     └ Bind Capsule.OnComponentBeginOverlap                      │
│     └ ForceRefreshObserveComponents                             │
└─────────────────┬───────────────────────────────────────────────┘
                  │
   Pawn Capsule overlap with range collision (impl IInteractRangeInterface)
                  ▼
   OnPawnBeginOverlapWithInteractItem → ObserveComponents.Add(FObserveComponentInfo)
   OnPawnEndOverlapWithInteractItem   → bOutOfObserveRange = true (next tick remove)
                  │
                  ▼
   TickComponent ─────────────► UpdateObserveComponentsState()
                                Step1: 距离 + CanBeInteracted + CanBeFocused + 屏内
                                       ├ <= FocusRange & 屏内 → 加入 FocusCandidates
                                       ├ <= PromptRange       → IIS_Prompt
                                       └ else                  → IIS_None
                                Step2: 多候选时按 (Cam·Item) 角度 cos 选最大者
                                Step3: 命中者 IIS_Focus, 其它降为 Prompt/None
                                Step4: SetNewFocusingComponent

   Player presses InputActionName ─► OnInteractAction()
       ├ Type=EIAT_Immediate → DoRealAction(FocusingComponent)
       └ Type=EIAT_Delay     → InteractDelayTick = InteractDelayDuration 倒计时
                               <=0 → DoRealAction; 释放清零

   DoRealAction: Item->TryInteract(QueryParam)
                 → switch(InteractAction)：
                    IA_None                          → return false
                    IA_SendGameplayEventWithPayload  → ASBlueprintLibrary::SendGameplayEventToActor
                    IA_CustomExecutor                → Class.GetDefaultObject()->TryInteract
                    IA_InterfaceImplement            → IInteractExecutorInterface::Execute_TryInteract
                    其他 (IA_SetLocoState/IA_ActivateAbility/IA_InactToSublevelEvent) → switch 无 case
```

## 核心枚举

### EInteractItemState (InteractSystem.h:11-18)
| 值 | 名 | 切换条件 | 谁触发 |
|---|---|---|---|
| 0 | IIS_None | !CanBeInteracted 或距离 > PromptRange，或离开屏幕，或 bOutOfObserveRange | Manager Tick / ForceRefresh / QuitInteract |
| 1 | IIS_Prompt | CanBeInteracted && 距离 ≤ PromptRange && 屏内 | Manager Step1/Step3 |
| 2 | IIS_Focus | CanBeInteracted && CanBeFocused && 距离 ≤ FocusRange && 屏内 && 是当前最佳候选 | Manager Step3 |
| 3 | IIS_Interacting | TryInteract 成功 / ForceSetCurrentInteractingItem | DoRealAction / 外部强制 |

切换入口唯一：`UInteractItemComponent::InternalSetInteractState(NewState)`（InteractItemComponent.cpp:119-157）。

### EInteractAction (InteractSystem.h:29-50)
| 值 | 名 | 行为 | 实现状态 |
|---|---|---|---|
| 0 | IA_None | TryInteract 直接 false | 已实现 |
| 1 | IA_SetLocoState | 给 Pawn 设 LocomotionState | switch 无 case，回落 false |
| 2 | IA_ActivateAbility | TryActivateAbilityByClass | switch 无 case |
| 3 | IA_SendGameplayEventWithPayload | SendGameplayEventToActor，Payload.OptionalObject=Owner，OptionalObject2=this | 已实现 |
| 4 | IA_InterfaceImplement | Owner 实现 UInteractExecutorInterface | 已实现 |
| 5 | IA_InactToSublevelEvent | 关卡子关卡事件 | 未实现 |
| 99 | IA_CustomExecutor | CDO 调 CustomInteractExecutorClass.TryInteract | 已实现 |

`IA_CustomExecutor=99` 走 **CDO 调用**：`CustomInteractExecutorClass->GetDefaultObject<UCustomInteractExecutor>()->TryInteract(...)`，所以 Executor 不能持有运行期实例状态。

## USTRUCT

`FInteractQueryParam` (InteractSystem.h:20-27)：仅 1 字段 `TObjectPtr<APawn> InitiatePawn`。

`FInteractInfo` (InteractSystem.h:52-65)：3 字段 `Type / Range / AutoInteract`。**全模块仅在 InteractCharacterComponent.cpp 中以局部变量出现一次，且周边 FindInteractInfo 调用全部被注释**——预留未生效。

## 5 大组件 API 速查

### UInteractItemComponent (USceneComponent 子类)
- 关键 UPROPERTY：`PromptRange=500 / FocusRange=200 / InputActionName="InteractAction1" / InteractWidgetClass / InteractAction=IA_None / InteractActionType=EIAT_Immediate / InteractState`
- 关键方法：`TryInteract(FInteractQueryParam)` / `CanBeInteracted/CanBeFocused` / `InitializeRangeCollision`（默认创建 Sphere，profile=InteractRange）/ `SetObserver` / `InternalSetInteractState` / `QuitInteract`
- 关键 delegate：`FOnInteractStateChanged OnInteractStateChanged(Old,New)` BlueprintAssignable
- 限制：`bCanEverTick=false`，由 Manager 通过 `PassivelyTick` 推动

### UInteractCharacterComponent
- 给 AHiCharacter 用，`UpdateFocusRange()` 核心被注释，**当前 NPC 互动是死的**

### UInteractRangeComponent / IInteractRangeInterface
- 两种实现：Sphere / Box，**没有 Capsule**
- `InteractRange` collision profile 在 DefaultEngine.ini 中**未找到**，是隐式依赖

### UInteractManagerComponent (UActorComponent)
- 挂在 AHiPlayerController，`BeginPlay` 显式 `if (OwnerController->IsLocalController())` 包住，**仅本地客户端**
- 焦点切换：屏幕中心方向 cos 最大者胜出（ItemVec 用 PawnLocation 而非 CamLoc）
- 关键 BlueprintCallable：`GetFocusingActor/Component / GetMainPawnCurrentInteractingActor/Component (static, WorldContext) / ForceSetCurrentInteractingItem / QuitInteract`
- 关键 delegate：`OnInteractActionDelayTime / OnInteractActionDelaySuccess`

### UInteractWidget + AInteractWidgetUpdater
- Widget 由 C++ 在 Item BeginPlay 中 CreateWidget；Updater 是隐式单例，TG_PostUpdateWork 每帧刷新所有 Widget 屏幕坐标
- **UpdateRate：每帧**，无可配置降频
- 屏幕外用 `ProjectPointOntoScreenSide` 投到屏幕边沿

### UCustomInteractExecutor + IInteractExecutorInterface
- 写自定义 Executor：蓝图继承 CustomInteractExecutor，重写 `TryInteract` BlueprintNativeEvent
- 或走 InterfaceImplement 路径：让 Item Owner 实现 UInteractExecutorInterface
- `IInteractLevelExecutorInterface` 是孤立的另一套，未连线

## C++ ↔ Lua / Blueprint 边界

| 函数 | 修饰 | 类 |
|---|---|---|
| OnInteractStateChanged | BlueprintAssignable | UInteractItemComponent |
| TryInteract | UFUNCTION | UInteractItemComponent |
| GetFocusingActor/Component | BlueprintPure | UInteractManagerComponent |
| GetMainPawnCurrentInteractingActor (static) | BlueprintCallable WorldContext | UInteractManagerComponent |
| ForceSetCurrentInteractingItem / QuitInteract | BlueprintCallable | UInteractManagerComponent |
| OnInteractActionDelayTime/Success | BlueprintAssignable | UInteractManagerComponent |
| K2_OnInteractStateChanged(Old,New) | BlueprintImplementableEvent | UInteractWidget |
| CanBeInteracted/Focused/TryInteract/QuitInteract | BlueprintNativeEvent | IInteractExecutorInterface |

## 与 GAS / UI / Mission 的耦合

- **GAS**：仅 `IA_SendGameplayEventWithPayload` 真路由到 GAS（SendGameplayEventToActor + Payload）
- **IA_ActivateAbility**：声明字段存在，但 switch 无 case，**未实现**
- **UI**：通过 `InteractWidgetClass` UPROPERTY；状态广播链 Item.OnInteractStateChanged → Widget.OnInteractStateChanged → K2_OnInteractStateChanged
- **Mission**：本模块**没有**直接调用 MissionPuzzle/Subsystem；Mission 侧通过订阅 GameplayEvent Tag

## Server / Client / Standalone

| 组件 | Client | Server | 说明 |
|---|---|---|---|
| UInteractManagerComponent | ✅ Local | ❌ | IsLocalController 守卫 |
| UInteractItemComponent | ✅ | ✅ | 不区分；Widget 创建用 GetFirstPlayerController()，DS 上跳过 |
| UInteractWidget | ✅ Only | ❌ | UMG |
| AInteractWidgetUpdater | ✅ Only | ❌ | World->IsGameWorld() 守卫 |

## 常见陷阱

1. **TRACE_CHANNEL_INTERACT_FOCUS_TEST 宏定义但全模块零引用** —— 整个交互依赖的是 Capsule overlap，不是 LineTrace
2. **InteractRange collision profile 隐式依赖** —— DefaultEngine.ini 中未定义 profile，Sphere 用 fallback 可能不与 Pawn Capsule overlap
3. **IA_SetLocoState / IA_ActivateAbility / IA_InactToSublevelEvent 未实现** —— 蓝图配了会"按下没反应"
4. **IA_CustomExecutor 走 CDO** —— Executor 写成员变量会跨 Item 串味
5. **Manager 只有本地客户端跑** —— 服务器逻辑必须走 GAS 或 RPC
6. **InteractWidget 必须配 WidgetSize_WidgetSpace** —— 没填会贴左上角
7. **Widget 注销路径** —— Destroy() 不真销毁，只 SetVisibility(Hidden)+RemoveFromParent
8. **InteractCharacterComponent 当前是死的** —— UpdateFocusRange 核心代码全注释
9. **bOutOfObserveRange 延迟一帧** —— 立即清需 ForceRefreshObserveComponents
10. **同名 InputActionName 隐藏行为** —— 切换焦点时若 Old.InputActionName == New.InputActionName 则不重新 Bind
11. **GetFocusingActor 无空保护** —— FocusingComponent 可能为 null，蓝图调用前必须判空
12. **ForceSetCurrentInteractingItem 不触发 TryInteract / Action** —— 只强设 IIS_Interacting + ForceRefresh
13. **OnInteractActionDelayTime 双重防重绑** —— SetObserver 中 IsBound() 反逻辑：已绑过任何监听，新 Widget 收不到 Delay 进度

## 关键代码位置

- InteractSystem.h:6-66 — 枚举/结构体集中
- InteractItemComponent.cpp:107-117 — InitializeRangeCollision
- InteractItemComponent.cpp:119-157 — InternalSetInteractState
- InteractItemComponent.cpp:170-228 — TryInteract switch
- InteractManagerComponent.cpp:135-166 — Manager BeginPlay + IsLocalController gating
- InteractManagerComponent.cpp:193-327 — UpdateObserveComponentsState 算法主体
- InteractManagerComponent.cpp:280-377 — 焦点选择 cos 算法
- InteractManagerComponent.cpp:554-583 — OnInteractAction 入口
- InteractWidgetUpdater.cpp:36-79 — Updater Tick (TG_PostUpdateWork)
- DefaultEngine.ini:179-264 — Collision Profiles（注意 GameTraceChannel5 实际名为 SkillDamaged）
