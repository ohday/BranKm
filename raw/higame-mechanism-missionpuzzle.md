---
id: 58
title: "HiGame MissionPuzzle Subsystem 全链路"
source: local-archaeology
project_root: E:\HiProject\PerforceDev\UnrealEngine\Projects\HiGame
modules:
  - Plugins/HiMissionPuzzle/Source/HiMissionPuzzle/Public/
  - Content/Script/{Server,Client}Script/mission/{puzzle,mission_puzzle}/
  - Content/Script/{Server,Client,Common}Script/actors/MissionPuzzle/
fetched: 2026-05-11
status: content-verified
covers_questions: [Q14]
---

# MissionPuzzle Subsystem 全链路

## 一、C++ 类层次（独立插件）

注意：MissionPuzzle 不在 `Source/HiGame/`，而在 **独立插件 `Plugins/HiMissionPuzzle/`**。模块名 HIMISSIONPUZZLE_API。

### UHiMissionPuzzleSubsystem
- **类型**：`UWorldSubsystem`（NOT GameInstanceSubsystem）。文件 `HiMissionPuzzleSubsystem.h:217`
- **静态访问**：`Get(WorldContextObject)` / `World->GetSubsystem<UHiMissionPuzzleSubsystem>()`
- **核心存储**：`TMap<FString, FHiMissionPuzzleSpec> PuzzleEntries`（line 775），EntryID → Spec
- **主要 UFUNCTION**：
  - `RegisterPuzzleBySettings(Settings, Params) → FString EntryID`（255，新主路径）
  - `RegisterAndStartPuzzleBySettings`（307，**Lua 端默认入口**）
  - `EndPuzzle(EntryID, Result, bAutoUnregister)`（352）
  - `EndPuzzleByPuzzleID(PuzzleID, Result, PlayerID, bAutoUnregister)`（366，**mission_node_end_puzzle 入口**）
  - `EndCurrentPuzzle / PausePuzzle / ResumePuzzle / UnregisterPuzzle / AddProgress`
  - 查询：`GetPuzzleState / GetPuzzleInstance / IsPuzzleRunning / FindEntryIDs / GetAllPuzzleStatus / GetAllRunningSnapshots / GetSnapshotByEntryID`
  - 客户端便捷：`GetLocalPlayerRunningSnapshots / LocalPlayerRequestEndPuzzle` 等
  - Reconnect：`ApplySnapshot(Snapshot)`（580）

### UHiMissionPuzzleComponent
- **挂载位置**：`UActorComponent` on `APlayerState`
- **职责**：DS 端订阅 Subsystem 委托 → PullSnapshotFromSubsystem → 写入 ReplicatedSnapshots TArray 进行属性复制；客户端 OnRep_Snapshots → PushSnapshotToSubsystem（→ ApplySnapshot）
- **关键 RPC**：
  - Server RPC：Server_RequestPausePuzzle / Server_RequestEndPuzzle / Server_RequestUnregisterPuzzle
  - Client RPC：`Client_NotifyPuzzleEnded(EntryID, PuzzleID, Result)`（**带外可靠通道**）
- **复制属性**：`UPROPERTY(ReplicatedUsing=OnRep_Snapshots) TArray<FHiPuzzleSnapshot> ReplicatedSnapshots`

### UHiMissionPuzzleConfig
- **类型**：`UPrimaryDataAsset`（不是 DataTable Row Type），`UCLASS(Abstract)`
- **现状**：在 RegisterPuzzleBySettings 推出后，`Register*(UHiMissionPuzzleConfig*)` API 全部 DeprecatedFunction。新世界由 Lua 端 mission_puzzle_config 数据表 + FHiMissionPuzzleSettings flat struct 驱动

### 关键数据类型 (HiMissionPuzzleTypes.h)
- `EHiMissionPuzzleState`：None / Registered / Running / Paused / Ended
- `EHiMissionPuzzleResult`：Succeeded / Failed / Cancelled
- `EHiMissionPuzzleSyncPolicy`：ServerOnly=0 / ClientOnly=1 / ServerAndClient=2
- `EHiMissionPuzzleClientInstanceState`：NotSpawned / LoadingClass / Spawned / Terminated
- `FHiMissionPuzzleSpec`：subsystem 内部 spec（**类比 GAS 的 FGameplayAbilitySpec**）
- `FHiMissionPuzzleSettings`：flat data struct
- `FHiPuzzleSnapshot`：复制单元，**operator== 故意排除 RemainingTime**

## 二、关键 Delegate（5 组、每组 Native + Dynamic）

| 名 | 签名 | 广播者 | 监听者 |
|---|---|---|---|
| OnPuzzleRegistered | (EntryID, PuzzleID) | Subsystem::SetPuzzleState→Registered | DebugWidget |
| OnPuzzleStarted | (EntryID, PuzzleID) | SetPuzzleState→Running、ApplySnapshot Running 重放 | MissionPuzzlePlayerComponent (打开 UI) |
| OnPuzzleProgressChanged | (EntryID, PuzzleID, NewCount, TargetCount) | Subsystem::BroadcastProgressChanged | mission_node_execute_puzzle (同步到 MissionTargetProgress) |
| OnPuzzleEnded | (EntryID, PuzzleID, EHiMissionPuzzleResult Result) | EndPuzzle / Client_NotifyPuzzleEnded RPC / ApplySnapshot Ended 重放（去重） | mission_node_execute_puzzle (流转 Pin)、PlayerComponent (关闭 UI)、path_chasing_puzzle |
| OnPuzzleUnregistered | (EntryID, PuzzleID) | UnregisterPuzzle | PlayerComponent 兜底关 UI |

每个事件还发到 `UHiGameplayMessageSubsystem` 命名通道 `Puzzle.Message.Registered/Started/...`。

## 三、Reconnect 快照恢复

### GetLocalPlayerRunningSnapshots
```cpp
static TArray<FHiPuzzleSnapshot> GetLocalPlayerRunningSnapshots(const UObject* WorldContextObject);
```

UI Widget 在 Construct() / 组件 BeginPlay() 调一次。**为什么**：OnPuzzleStarted 是事件型，UI 创建之前事件已派发完毕；快照接口提供"补一次初始态"，配合事件订阅形成 **冷启动 + 增量** 双通道。

### FHiPuzzleSnapshot 字段
EntryID(FString) / PuzzleID(int32) / State / CurrentCount / TargetCount / RemainingTime(float, -1=无倒计时) / TimeLimit / EndResult / OwnerPlayerID / SyncTargets(TArray) / PuzzleInstanceClass(FString classpath) / SyncPolicy。

### 重连完整流
1. 玩家断线，DS 继续推进 puzzle，timer/counter 仍在 server 跑
2. PlayerState 自动复制 → 新连接接收到 ReplicatedSnapshots 全量数组
3. OnRep_Snapshots → PushSnapshotToSubsystem → ApplySnapshot
4. ApplySnapshot 按 State 分支处理：Running→注册并触发 OnPuzzleRegistered + OnPuzzleStarted；Ended→触发 OnPuzzleEnded
5. UI Lua 同时通过 GetLocalPlayerRunningSnapshots 主动查一次冷启动数据

## 四、文件名陷阱：mission_puzzle_subsystem.lua ≠ Puzzle Subsystem

`ServerScript/mission/puzzle/mission_puzzle_subsystem.lua` 的代码内部 Class 名是 **`MissionTrackingManager_S`**（line 28）—— 实际是 **MissionTrackingManager 实现**，不是 PuzzleSubsystem 的 Lua 镜像。文件路径与内容不一致（命名遗留）。它管理 FS_Mission_TrackingInstanceData 任务追踪光柱（光柱、跨区域 DungeonID、Actor 跟随），与 Puzzle 子系统**无直接关系**。

Puzzle "Server 侧 manager"职责完全沉到 C++ Subsystem 中，Lua 不再做 manager。

## 五、mission_node_execute_puzzle vs mission_node_end_puzzle

| 维度 | execute_puzzle | end_puzzle |
|---|---|---|
| 定位 | **启动节点**：注册并运行，等结果 | **强制结束节点**：mission 流程主动叫停 |
| 输出引脚 | SuccessPin / FailedPin | 单个 Out |
| 关键 API | Subsystem:RegisterAndStartPuzzleBySettings | Subsystem:EndPuzzleByPuzzleID(PuzzleId, Result, nil, true) |
| 监听委托 | OnPuzzleEnded + OnPuzzleProgressChanged | 无（同步触发完即走） |
| 桥接 | _SyncProgressToMission → MissionSystemComp:UpdateMissionTargetProgress | 无 |
| 中断行为 | K2_Cleanup → EndPuzzle(EntryID, Cancelled, true) | 无 |
| 恢复行为 | OnNodeResume → 重新 _StartPuzzle | — |

## 六、mission_pathchasing_puzzle vs actors/MissionPuzzle/path_chasing_puzzle

**两个完全不同概念**：

- `ServerScript/mission/mission_puzzle/mission_pathchasing_puzzle.lua` —— **UHiMissionPuzzle 子类的 Lua 绑定**（对应 BP_PathChasingPuzzle_C），是 cfg.puzzle_class 指向的 PuzzleInstance 类。它实现 OnPuzzleStart / OnPuzzleEnd 接口，是 subsystem RegisterAndStart 时 spawn 出来的逻辑实例
- `actors/MissionPuzzle/path_chasing_puzzle.lua` (Server/Client/Common) —— **场景 Actor**（Spline + Niagara 路径机关）

**关系**：mission_pathchasing_puzzle 是 puzzle 系统注册的"runtime ability"，作为**桥**控制场景 Actor path_chasing_puzzle 进入 Active 状态；场景 Actor 用 OnPuzzleEnded 委托回 puzzle 系统报结果。

## 七、PathChasing 玩法

1. Status_Active 触发 → 玩家被 EnterState(State_ForbidMove) + ForceLocklnWalk
2. 倒计时（默认 3.5s）UI 弹出 → 倒计时结束放开
3. Server：发 mail 给怪物 NPC `Move_To_Way_Point` 跟随玩家（MonsterMoveSpeed=200）
4. Client 200ms tick：路径偏移 / 怪物追上 / 终点到达检测；20ms tick：UpdatePathProgress 让 Niagara 路径在玩家身后逐渐隐藏
5. Niagara 路径靠 `Custom_Spline` DataInterface 绑定 self 的 SplineComponent，按 LerpAlpha 切割可视段
6. 失败/成功 → FinishPuzzle(Result) → Server OnPuzzleEnded 委托回调 → EndPuzzleByPuzzleID

## 八、MissionPuzzleUtils API

| 函数 | 签名 | 用途 |
|---|---|---|
| GetPuzzleConfig | (PuzzleID:string|number) → table | 读 common.data.mission_puzzle_config.data[PuzzleID] |
| GetPuzzleConfigField | (PuzzleID, FieldName, Default) | 读单字段含默认值 |
| RegisterAndStartPuzzle | (WorldContext, PuzzleID, Params) → EntryID | 一站式启动；内部组 FHiMissionPuzzleSettings + FHiPuzzleStartParams 调 Subsystem |

## 九、mission_puzzle_config DataTable

`require("common.data.mission_puzzle_config").data` —— Lua 表（不是 .uasset DataTable）。

| 字段 | 类型 | 用途 |
|---|---|---|
| time_limit | float | → Settings.CountdownDuration |
| target_count | int | → Settings.GoalCount |
| puzzle_class | string (BP path) | → Settings.PuzzleInstanceClass |
| sync_policy | int (0/1/2) | → Settings.SyncPolicy |
| start_with_countdown | bool | 客户端 UI 是否走 UI_Challenge_Progress 倒计时 |
| countdown | float | utils 旧字段名（与 time_limit 重复） |

**PuzzleID 命名约定**：C++ 侧统一为 int32（FHiMissionPuzzleSettings::PuzzleID、FHiPuzzleSnapshot::PuzzleID）；mission_node 路径强类型 int32；DataTable 主键也是数字 ID。`UHiMissionPuzzleConfig::PuzzleID` 是 FString —— 那是被 deprecated 的老路径。

## 十、Client UI 联动

`MissionPuzzlePlayerComponent` 继承 `Component(ComponentBase)`，蓝图基类 `BP_MissionPuzzlePlayerComponent_C`。

ReceiveBeginPlay：
1. Subsystem.OnPuzzleStarted:Add(self.OnPuzzleStarted)
2. Subsystem.OnPuzzleEnded:Add(self._OnPuzzleEnded)
3. Subsystem.OnPuzzleUnregistered:Add(self._OnPuzzleUnregistered)
4. GetLocalPlayerRunningSnapshots 重放运行中的快照

OnPuzzleStarted(EntryID, PuzzleID)：
- Subsystem:GetSnapshotByEntryID(EntryID) 取快照
- PuzzleUtils.GetPuzzleConfig(tonumber(PuzzleID)) 取配置
- 若 cfg.start_with_countdown 则先 UIManager:OpenUI(UIDef.UIInfo.UI_Challenge_Progress, ..., CountDownInfo) 倒计时
- 回调中 OpenTipUI(REFLECTLINE_MSG_GAME_START) + OpenTimerUI(cfg.time_limit)

UI 路径常量：
- UI_ImportantTips —— 屏幕中央提示语
- UI_TimerDisplay (`WBP_HUD_TimerDisplay_C`) —— HUD 倒计时
- UI_Challenge_Progress —— 挑战开始的环形倒计时

## 十一、任务驱动全链路

```
[Server / DS]                                  [Client]
============                                   =========

mission_node_execute_puzzle:K2_ExecuteInput
    │
    │  PuzzleUtils.GetPuzzleConfig(PuzzleId)
    ▼
build FHiMissionPuzzleSettings
    │
    │  Subsystem:RegisterAndStartPuzzleBySettings
    ▼
UHiMissionPuzzleSubsystem (UWorldSubsystem on DS)
    │  RegisterPuzzle_Internal → EntryID = FGuid::NewGuid()
    │  insert FHiMissionPuzzleSpec into PuzzleEntries[EntryID]
    │  SetPuzzleState(Registered) → broadcast OnPuzzleRegistered*
    │  StartPuzzle_Internal → NewObject<UHiMissionPuzzle>(InstanceClass)
    │  SetPuzzleState(Running) → broadcast OnPuzzleStarted*
    │  start FTimerHandle CountdownTimerHandle
    ▼
UHiMissionPuzzleComponent (PlayerState, DS)
    │  OnSubsystemPuzzleStarted → PullSnapshotFromSubsystem
    │  fill FHiPuzzleSnapshot
    │  write to ReplicatedSnapshots
    │
    │  ───── UE Property Replication ──────────►
    │                                   ▼
    │                               OnRep_Snapshots → PushSnapshotToSubsystem
    │                                   ▼
    │                               Subsystem::ApplySnapshot
    │                                   │  if SyncPolicy in {ClientOnly, ServerAndClient}:
    │                                   │     BeginAsyncSpawnClientInstance
    │                                   │  SetPuzzleState(Running)
    │                                   │  fire OnPuzzleStarted
    │                                   ▼
    │                               MissionPuzzlePlayerComponent.OnPuzzleStarted
    │                                   │  GetSnapshotByEntryID
    │                                   │  PuzzleUtils.GetPuzzleConfig
    │                                   │  UIManager:OpenUI(UI_Challenge_Progress
    │                                   │                  → UI_TimerDisplay
    │                                   │                  → UI_ImportantTips)
    │                                   ▼
    │                               [玩家在 UI / 场景 Actor 中操作]
    │  ◄──── reliable RPC ─────────────-┤
    ▼                                   │
PathChasingPuzzle.Status_Complete       │
    PuzzleSubsystem:EndPuzzleByPuzzleID(PuzzleID, Result)
    ▼
Subsystem::EndPuzzle
    │  SetPuzzleState(Ended) → broadcast OnPuzzleEnded*
    │  invalidate timer, destroy Instance
    │  if (bAutoUnregister) UnregisterPuzzle
    │  SendEndedRPCToTargets → Component->Client_NotifyPuzzleEnded (RELIABLE RPC)
    │  PullSnapshotFromSubsystem (Component) → Replicated array updated
    │  ─── Replication path  ──►  OnRep_Snapshots → ApplySnapshot
    │  ─── Reliable RPC path ──►  Component::Client_NotifyPuzzleEnded
    │                              → Subsystem::NotifyPuzzleEndedOnClient
    │                                  (DEDUP: ignore if already Ended)
    │                                  fire OnPuzzleEnded once
    │                                  ▼
    │                              PlayerComponent._OnPuzzleEnded
    │                                  CloseTimerUI / OpenTipUI(REFLECTLINE_MSG_*)
    │  DS 侧 mission_node_execute_puzzle._OnPuzzleEnded
    │     TriggerOutput(SuccessPin / FailedPin)
    ▼
[Mission flow 继续推进]
```

## 十二、PuzzleID vs EntryID

| 维度 | EntryID | PuzzleID |
|---|---|---|
| 类型 | FString（FGuid::NewGuid().ToString()） | int32 |
| 生成 | 服务器在 RegisterPuzzle_Internal 生成 | 配置表主键，puzzle 类型标识 |
| 唯一性 | 单次运行唯一 | 类型唯一（同 PuzzleID 可多 EntryID 并发） |
| Map Key | 是（PuzzleEntries） | 否（FindEntryIDs 反查） |
| Lua 行为 | mission_node 缓存 self._EntryID，回调过滤 if EntryID ~= self._EntryID then return | 配置查找的 key |

**为什么需要 EntryID**：同一 PuzzleID 在多人 / 多 mission 中可能并发存在，单靠 PuzzleID 无法区分某次具体 run；EntryID 是稳定 GUID。

## 十三、为什么需要带外 RPC Client_NotifyPuzzleEnded

当 DS 调 EndPuzzle(..., bAutoUnregister=true)，ReplicatedSnapshots 在同一复制窗口内被写两次（先 Ended、再 remove），UE 属性复制只发最后一次值，**Ended 的中间帧被吞**，client 只看到 entry 消失而错过 OnPuzzleEnded。带外 reliable RPC 保证 Ended 一定能到达；Subsystem 端用"已 Ended 则 no-op"做幂等去重。

## 十四、跨区域追踪 (FS_CrossRegionalTracking)

跨区域字段在 puzzle 系统中**没有直接出现**——它属于姊妹系统 MissionTracking。结构 `S_CrossRegionalTracking { DungeonID:int, CrossRegionalTrackingTag:string }`。

**Puzzle 子系统对 DDS 多 Server 的处理**：`SyncTargets` + `OwnerPlayerID` 是核心。每个 Snapshot 携带 SyncTargets:TArray，PullSnapshotFromSubsystem 在 DS 端按 OwnerPlayerID ∈ SyncTargets 过滤进入 ReplicatedSnapshots；这样多 World、多 Server 实例下同一 PuzzleEntries 不会跨域复制。EntryID 全局唯一保证跨 Server 迁移时不冲突。

## 十五、与 actors/Puzzle/ 区分

| 维度 | actors/Puzzle/（5 具体机关） | mission/puzzle + actors/MissionPuzzle/ |
|---|---|---|
| 定位 | 场景物件 Actor | 任务驱动子系统 |
| 触发入口 | StatusFlow（Active/Sealed 由 Interact / Mission 推） | mission_node_execute_puzzle 主动 RegisterPuzzle |
| 核心 C++ | 各自蓝图 + InteractItem 基类 | UHiMissionPuzzleSubsystem (Plugin) + Component |
| 数据 | 单 Actor 蓝图属性 | mission_puzzle_config Lua 表 + FHiMissionPuzzleSettings |
| 网络模型 | Actor RPC + Status 复制 | EntryID/Snapshot TArray + 带外 RPC + ApplySnapshot |
| 重连 | Actor 重生由场景管理 | TArray 复制 + GetLocalPlayerRunningSnapshots |
| 是否 spawn 实例 | 永远是 Actor | UHiMissionPuzzle 内存对象 (NewObject，不是 AActor) |
| 用例 | DancingSofa / DouDing / Ghost / HeadEye / Bird | PathChasing 等 "跟随路径不被怪物追上" 小游戏 |

## 关键代码位置

- Plugins/HiMissionPuzzle/.../HiMissionPuzzleSubsystem.h:217 — UWorldSubsystem
- HiMissionPuzzleSubsystem.h:16-69 — 5 组 Multicast Delegate
- HiMissionPuzzleSubsystem.h:255 — RegisterPuzzleBySettings
- HiMissionPuzzleSubsystem.h:307 — RegisterAndStartPuzzleBySettings
- HiMissionPuzzleSubsystem.h:366 — EndPuzzleByPuzzleID
- HiMissionPuzzleSubsystem.h:509 — GetLocalPlayerRunningSnapshots
- HiMissionPuzzleSubsystem.h:580 — ApplySnapshot
- HiMissionPuzzleSubsystem.h:684-688 — SendEndedRPCToTargets
- HiMissionPuzzleComponent.h:18-57 — PlayerState 挂载注释
- HiMissionPuzzleComponent.h:140 — Client_NotifyPuzzleEnded RPC 设计
- HiMissionPuzzleTypes.h:16-71 — 4 个枚举
- HiMissionPuzzleTypes.h:301-412 — FHiPuzzleSnapshot
- HiMissionPuzzleTypes.h:425-497 — FHiMissionPuzzleSettings
- mission/puzzle/mission_puzzle_utils.lua:25-108 — Utils API
- ServerScript/mission/mission_node/mission_node_execute_puzzle.lua:24-151 — execute 节点全流程
- ServerScript/mission/mission_node/mission_node_end_puzzle.lua:22-42 — end 节点
- ServerScript/mission/mission_puzzle/mission_pathchasing_puzzle.lua:11-29 — UHiMissionPuzzle 子类 Lua 绑定
- ClientScript/mission/puzzle/mission_puzzle_player_component.lua:32-187 — 客户端组件
- ServerScript/mission/puzzle/mission_puzzle_subsystem.lua:28 — **Class 名 MissionTrackingManager_S（文件名误导）**
