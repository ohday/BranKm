---
id: npc-13
title: C++ NPC Component 套件
sources:
  - Projects/HiGame/Source/HiGame/Public/Component/HiNpcSklMeshChangeComponent.h
  - Projects/HiGame/Source/HiGame/Public/Component/HiActorCollisionPushComponent.h
  - Projects/HiGame/Source/HiGame/Public/Component/HiNPCEnablePhysicsBoundCounterComponent.h
  - Projects/HiGame/Source/HiGame/Public/Component/HiNPCEditorManagerComponet.h
  - Projects/HiGame/Source/HiGame/Public/Component/HiNPCMovementComponent.h
  - Projects/HiGame/Source/HiGame/Public/Component/HiNpcLocomotionAppearance.h
  - Projects/HiGame/Source/HiGame/Public/Component/HiDialogueCameraControllerComp.h
  - Projects/HiGame/Source/HiGame/Public/Characters/Animation/HiNpcAnimInstance.h
updated: 2026-05-11
---

# npc-13 — C++ NPC Component 套件

## Sources

- `Projects/HiGame/Source/HiGame/Public/Component/HiNpcSklMeshChangeComponent.h`
- `Projects/HiGame/Source/HiGame/Public/Component/HiActorCollisionPushComponent.h`
- `Projects/HiGame/Source/HiGame/Public/Component/HiNPCEnablePhysicsBoundCounterComponent.h`
- `Projects/HiGame/Source/HiGame/Public/Component/HiNPCEditorManagerComponet.h`
- `Projects/HiGame/Source/HiGame/Public/Component/HiNPCMovementComponent.h`
- `Projects/HiGame/Source/HiGame/Public/Component/HiNpcLocomotionAppearance.h`
- `Projects/HiGame/Source/HiGame/Public/Component/HiDialogueCameraControllerComp.h`
- `Projects/HiGame/Source/HiGame/Public/Characters/Animation/HiNpcAnimInstance.h`

## 1. 概念

这些类是 NPC Actor 通过 UE Blueprint AddComponent 挂载的 C++ `UActorComponent`/`UCharacterMovementComponent`/`UAnimInstance` 派生类,位于 `Projects/HiGame/Source/HiGame/Public/Component/`(以及 `Public/Characters/Animation/`)。与 npc-06 Lua NodeComponent 不同,这一层是 UE-side 组件,通过 `UPROPERTY` 暴露字段、走 `Replicated`/`NetMulticast`/`Client` RPC 复制,通过 `UFUNCTION(BlueprintCallable)` 暴露给蓝图与 UnLua;Lua 侧 `npc_camera_component`/`npc_locomotion_component`/`anim_ctrl_comp`(npc-11) 是其高层包装。

## 2. 7 + 1 (AnimInstance) 组件总表

| 类名 | 父类 | 文件 | 一句话职责 |
|---|---|---|---|
| `UHiNpcSklMeshChangeComponent` | `UActorComponent` | `HiNpcSklMeshChangeComponent.h` | 通过 JSON 描述驱动 NPC 头/身换装与妆容、骨骼网格 merge |
| `UHiActorCollisionPushComponent` | `UActorComponent` | `HiActorCollisionPushComponent.h` | 角色之间(玩家/怪物/NPC)碰撞推挤逻辑 |
| `UHiNPCEnablePhysicsBoundCounterComponent` | `UActorComponent` | `HiNPCEnablePhysicsBoundCounterComponent.h` | 引用计数式开关多个子骨骼网格 `bComponentUseFixedSkelBounds` 类优化 |
| `UHiNPCEditorManagerComponet` | `UActorComponent` | `HiNPCEditorManagerComponet.h` | 编辑器侧 NPC 捏脸/换装/存读 JSON 工具 |
| `UHiNPCMovementComponent` | `UHiAIMovementComponent`(派生自 `UCharacterMovementComponent`) | `HiNPCMovementComponent.h` | NPC 角色移动组件,仅暴露 `SetMaxWalkSpeed` |
| `UHiNpcLocomotionAppearance` | `UHiLocomotionComponent` | `HiNpcLocomotionAppearance.h` | NPC Locomotion(步态/旋转模式/Ragdoll)外观 |
| `UHiDialogueCameraControllerComp` | `UActorComponent` | `HiDialogueCameraControllerComp.h` | 对话开始/结束时镜头控制 + Blend |
| `UHiNpcAnimInstance` | `UHiAnimInstance` | `Characters/Animation/HiNpcAnimInstance.h` | NPC 动画蓝图基类(Gait/Grounded/Slot Montage) |

## 3. UHiNPCMovementComponent

最薄的一个组件,只对外暴露一个写访问器:

```cpp
UCLASS()
class HIGAME_API UHiNPCMovementComponent : public UHiAIMovementComponent
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintCallable, Category = "Gameplay|Character|Npc")
    void SetMaxWalkSpeed(float UpdateMaxWalkSpeed);
};
```

继承链: `UHiNPCMovementComponent` → `UHiAIMovementComponent`(在 `Component/HiAIMovementComponent.h`) → 最终落到 `UCharacterMovementComponent`。
所有寻路、RVO、Avoidance 等都在父类里;Npc 层只需要外部以 cm/s 设置最大行走速度。

## 4. UHiNpcSklMeshChangeComponent

负责 NPC 角色形象的"运行期 + 编辑器期"换装与妆容,基于 JSON 文件描述。

关键 USTRUCT/委托:

```cpp
USTRUCT(BlueprintType)
struct FMakeUpInfo {
    UPROPERTY(EditAnywhere, BlueprintReadWrite) TMap<MakeupInfoType, float> InfoMap;
    UPROPERTY(EditAnywhere, BlueprintReadWrite) FString TexturePath;
    UPROPERTY(EditAnywhere, BlueprintReadWrite) FLinearColor Color;
};
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FNPCFiledLoaded);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FNPCMeshLoadFinished, const bool, ParamValue);
```

蓝图 API:`ChangeNpcMeshByFile`、`ChangeNpcMeshByTwoFiles`、`InitNPCFiles`、
`GetFloatValue` / `GetColorValue` / `GetPathValue`、`ModifyMakeUpInfo(bool IsEditor=false)`、
`InitializeNPCData`(protected 但 BlueprintCallable);委托 `OnNPCFilesLoaded`、`OnNPCMeshLoadFinished` 用于异步通知。

骨骼合并核心 API(配合 Editor Util):

```cpp
static FSoftObjectPath CreateMergedSkeletalMesh(
    FString InMergedSkeletonName, FString InMergedSkeletalMeshName,
    const TArray<FSoftObjectPath>& InRequiredSkeletalMeshAssets,
    FString InFaceFilePathAsInfo, FString InBodyFilePathAsInfo,
    bool bCheckoutAsset, FString& OutFailureReason);
UFUNCTION(BlueprintCallable)
static FString CreateMergedSkeletalMesh_EditorUtil(FString InFaceFilePath, FString InBodyFilePath);
```

异步:`FTask_ReadNPCJsonFile` 跑非 GameThread IO,`FTask_InitializeNPCDataImpl` 回 GameThread 应用结果,通过 `friend` 修改 `FaceFileJsonContainer`/`BodyFileJsonContainer`。配套 `UHiNPCSkeletalMeshMergeSourceInfo : UAssetUserData` 把 RequiredAssetList、FaceFilePath、BodyFilePath、GenerateVersion 写到生成资产;`UHiNpcSklMeshChangeComponentSettings : UDeveloperSettings` 存 `SkeletonAssetPathCommonPrefix`/`MergedMeshPackageFolder`/`MergedMeshLODSettings`/`MergedSkeletonMapping`。

## 5. UHiNpcLocomotionAppearance

NPC 专用的 Locomotion 外观组件,继承自 `UHiLocomotionComponent`(同插件公共基类)。

```cpp
UCLASS(BlueprintType)
class HIGAME_API UHiNpcLocomotionAppearance : public UHiLocomotionComponent
{
    UHiNpcLocomotionAppearance(const FObjectInitializer& ObjectInitializer);
    virtual void InitializeComponent() override;
    virtual void TickLocomotion(float DeltaTime) override;
    virtual void RagdollStart() override;
    virtual void RagdollEnd() override;
    virtual EHiGait GetActualGait(EHiGait AllowedGait) const;
};
```

protected 钩子:`OnRotationModeChanged(EHiRotationMode)`、`OnSpeedScaleChanged(float)`、`UpdateCharacterMovement()`、`UpdateCharacterRotation(float)`(override)。公共配置:`bEnableCustomizedRotationWithRootMotion`、`DefaultActorRotationSpeed = 3.0f`。

与 npc-11 Lua `npc_locomotion_component` 关系:Lua 通过 UnLua 调用 `GetActualGait`、读 `DefaultActorRotationSpeed`,本组件持有 `MyCharacterMovementComponent : UHiCharacterMovementComponent` 强引用并在 `TickLocomotion` 内同步移动参数。

## 6. UHiDialogueCameraControllerComp

```cpp
UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "NPC Dialogue Camera Move")
void StartDialogueWithNPC(AActor* NPC);
UFUNCTION(BlueprintCallable, BlueprintNativeEvent, ...)
void EndDialogueWithNPC();
UFUNCTION(BlueprintCallable, BlueprintNativeEvent, ...)
void OnBlendOutCompleted();
UFUNCTION(BlueprintCallable, ...)
UHiVisionerView_Dialogue* FindDialogueView() const;
```

参数:`MaxPitch/MinPitch/BestPitch`、`SpringArmLengthCurve : UCurveFloat*`、`MinYawConstraintCurve`、`BlendInParam : FAlphaBlend = 1.f`、`BlendOutParam : FAlphaBlend = 1.f`、`TargetOffsetRate = 0.5f`。`UpdateViewTarget(FVector PivotLocation)` 与 `DetectCollision(...)` 在 Tick 中持续运算视点。内部 `FDialogueCameraConfig` 保留 Yaw 约束(被注释,运行期改用 `MinYawConstraintCurve`)。

与 npc-11 Lua `npc_camera_component` 关系:Lua 在对话节点起始/结束时调用 `StartDialogueWithNPC`/`EndDialogueWithNPC`,具体相机插值由 `FindDialogueView()` 找到的 `UHiVisionerView_Dialogue` 执行。

## 7. UHiActorCollisionPushComponent

NPC vs 玩家、玩家 vs 怪物之间的"被推开"判定,运行在服务器(`ApplyPushToMonster`) + 本地(`ApplyPushToPlayer`)。

```cpp
UFUNCTION(BlueprintCallable)
void AddPushActor(FHitResult hit, AActor* OtherCharacter, UCapsuleComponent* MonsterCapsule);
UFUNCTION(BlueprintCallable)
void RemovePushActor(AActor* OtherCharacter);
UFUNCTION(BlueprintCallable)
void PushHitActor(FHitResult hit, UHiActorCollisionPushComponent* OtherActorCP,
    UHiCharacterMovementComponent* OtherCharacterMovement, float DeltaTime,
    bool bIsHit = true, UCapsuleComponent* MonsterCapsule = nullptr);
UFUNCTION(NetMulticast, Reliable)
void SetActorBufferVel(FVector CalBufferVel, const FString& ActorName, AActor* InActor);
UFUNCTION(Client, Reliable)
void SetActorBufferVelForClient(FVector CalBufferVel, const FString& ActorName, AActor* InActor);
```

权重:`CollisionWeight`、`CanBePushWeight`、`PushOtherWeight` 三档,加上 `bIsNpcOrMonster` 区分对象。速度:`MinSpeed=100`、`MaxSpeed=200`、`FlyPushOtherSpeed=500` + `static const float HitTime / HitMonsterTime / Acceleration / Acceleration2`。保底校验定时器:`ValidationInterval=0.5s`、`DistanceToleranceMultiplier=1.5`、`DistanceBufferCm=20`,周期遍历 `PushActorInfoMap`/`PushHitActorInfoMap` 清理 `EndOverlap` 未触发导致的残留。

## 8. UHiNPCEnablePhysicsBoundCounterComponent

引用计数器,管理多个 `USkeletalMeshComponent` 的"是否启用物理 Bound"。

```cpp
UCLASS(BlueprintType, meta = (BlueprintSpawnableComponent))
class HIGAME_API UHiNPCEnablePhysicsBoundCounterComponent : public UActorComponent
{
public:
    UFUNCTION(BlueprintCallable)
    void SetEnabled(bool bNewState);
private:
    int EnabledCount = 0;
    UPROPERTY(EditAnywhere)
    TArray<FName> ChildComponentNames;
    UPROPERTY()
    TArray<TObjectPtr<USkeletalMeshComponent>> ChildComponents;
    void OnRegister() override;
    void UpdateState();
};
```

`OnRegister` 解析 `ChildComponentNames` 拿到对应的 `USkeletalMeshComponent` 指针,`SetEnabled` 走 +/- 引用计数后调用 `UpdateState()` 切换。
这是为 LOD/Significance 优化使用的薄壳组件,与 npc-15 中的 Significance 重要性分级配合。

## 9. UHiNPCEditorManagerComponet

NPC 捏脸+服装管理的"编辑器/数据驱动"接入点(注意类名拼写 `Componet`)。
核心枚举(也被 npc-13 上面 `FMakeUpInfo` 引用):

```cpp
UENUM(BlueprintType)
enum class NPCModel : uint8 { Woman, Man, LittleBoy, LittleGirl, OldMan, OldWoman, Boy, Girl };
UENUM(BlueprintType)
enum class NPCMakeup : uint8 { Eyeshadow, Eyebrow, Eyeliner, LipMakeUp, Hair, End };
UENUM(BlueprintType)
enum class MakeupInfoType : uint8 { HueShift, HueSaturation, Transparency, Brightness,
    Specular, SpecularMask, Intensity, End };
```

蓝图 API 抽样:`ChangeModel(NPCModel)`、`ChangeAllBody/ChangeUpperBody/ChangeLowerBody/ChangeFace/ChangeHair(FString AssertPath)`、`ChangeMakeup(NPCMakeup, FString)`、`ModifyMakeup(NPCMakeup, FVector Color, float Lightness, float Saturability, float Gradation)`、`ModifyFaceValue(FString, float)` / `GetFaceValue(FString)`、`SaveNPCInfo / LoadNPCInfo / SavcFaceKeyValueInfo / LoadFaceKeyValueInfo / SavcFaceMesh`、`CheckFaceSaveToMesh()`(返回 -1/1)、
`InitNPCInfo(NPCModel)`、`CanModifyFace()`。
状态字段:`UpperBodySkin/LowerBodySkin/FaceSkin/HairSkin`、4 张默认 `DefaultModel*Map`、`FaceTargetMesh`(简化骨骼用于 bake 捏脸)、
`NewFaceMesh`(捏脸保存结果)。

## 10. UHiNpcAnimInstance

NPC 专用的 AnimInstance 基类:

```cpp
UCLASS()
class HIGAME_API UHiNpcAnimInstance : public UHiAnimInstance {
public:
    virtual void NativeBeginPlay() override;
    virtual void NativeUpdateAnimation(float DeltaSeconds) override;
    UFUNCTION(BlueprintCallable, Category = "Animation|Npc")
    void SetMoveGait(EHiGait MoveGait);
    UPROPERTY(...) EHiGait LogicGait = EHiGait::Idle;
    UPROPERTY(...) FHiAnimGroundedConfig GroundedConfig;
    UPROPERTY(...) FHiAnimGraphGrounded Grounded;
    UPROPERTY(BlueprintReadOnly, ...) bool bCopyFromFaceMesh;
    UPROPERTY(BlueprintReadOnly, ...) TObjectPtr<USkeletalMeshComponent> CopyFaceSource;
    UFUNCTION(BlueprintCallable, ...) UAnimMontage* PlaySlotAnimationAsDynamicMontageNotStopOthers(...);
private:
    float CharacterMoveSpeed = 0.0f;
    TObjectPtr<UHiNPCMovementComponent> HiNPCMovementComponent;
    EHiGait MoveGait = EHiGait::Walking;
    EHiGait GetLogicGait() const;
    float CalculateStandingPlayRate() const;
};
```

`NativeUpdateAnimation` 每帧从 `HiNPCMovementComponent` 拉取速度并算 `CharacterMoveSpeed`、`LogicGait`,
`PlaySlotAnimationAsDynamicMontageNotStopOthers` 是变体 PlaySlotAnimation,不打断已有 Montage,常被 Lua `anim_ctrl_comp` 调用。

## 11. Replication 策略统计

- `UHiActorCollisionPushComponent` 重写 `GetLifetimeReplicatedProps`,且使用 `NetMulticast, Reliable`(`SetActorBufferVel`)+ `Client, Reliable`(`SetActorBufferVelForClient`)做推挤速度同步;权威由 `IsServer()` 私有方法决定。
- 其他组件(`SklMeshChange`、`LocomotionAppearance`、`DialogueCameraController`、`PhysicsBoundCounter`、`EditorManager`、`NPCMovement`)在头文件中**未声明**任何 `UPROPERTY(Replicated)`/`UFUNCTION(Server/Client/NetMulticast)`,均为本地组件;复制由其 OwnerActor 或父类(`UCharacterMovementComponent`、`UHiLocomotionComponent`)自身机制承担。
- `UHiNpcSklMeshChangeComponent` 通过两个 `BlueprintAssignable` 委托(`OnNPCFilesLoaded`、`OnNPCMeshLoadFinished`)广播状态,而非 RPC。

## 12. 与 Lua 侧桥接点

抽样 5–10 个最常被 UnLua 调用的入口:

1. `UHiNpcSklMeshChangeComponent::ChangeNpcMeshByTwoFiles(FacePath, BodyPath)` — Lua 侧外形切换。
2. `UHiNpcSklMeshChangeComponent::OnNPCMeshLoadFinished` — Lua 通过 `:Add` 监听完成事件。
3. `UHiNPCMovementComponent::SetMaxWalkSpeed(float)` — locomotion node 调用切速度。
4. `UHiDialogueCameraControllerComp::StartDialogueWithNPC(AActor* NPC)` / `EndDialogueWithNPC()` — `npc_camera_component`(npc-11) 入口。
5. `UHiActorCollisionPushComponent::AddPushActor / RemovePushActor / PushHitActor` — Lua 蓝图层在 OnHit/OnBeginOverlap 调用。
6. `UHiNpcAnimInstance::SetMoveGait(EHiGait)` 与 `PlaySlotAnimationAsDynamicMontageNotStopOthers` — Lua `anim_ctrl_comp` 调用。
7. `UHiNPCEnablePhysicsBoundCounterComponent::SetEnabled(bool)` — Significance/AOI 切换时 +/- 计数。
8. `UHiNPCEditorManagerComponet::ChangeMakeup` / `ModifyMakeup` / `SaveNPCInfo` / `LoadNPCInfo` — 编辑器工具或数据驱动加载。
9. `UHiNpcLocomotionAppearance::GetActualGait(EHiGait AllowedGait)` — Lua 侧拿当前 gait。
10. `UHiNpcSklMeshChangeComponent::CreateMergedSkeletalMesh_EditorUtil(...)` — 编辑器流水线烘焙合并骨骼网格。

## 13. 跨页关联

- npc-11 (Lua actor 组件 — `npc_camera_component` ↔ `UHiDialogueCameraControllerComp`;`npc_locomotion_component` ↔ `UHiNpcLocomotionAppearance` + `UHiNPCMovementComponent`)
- npc-12 (`UHiNPCMovementComponent` ↔ PathFollowing/Aether — 父类 `UHiAIMovementComponent` 是接入处)
- npc-06 (Lua NodeComponent 的 `anim_ctrl_comp`、`look_at` 通过 UnLua 调用 `UHiNpcAnimInstance::SetMoveGait`、Slot Montage API)
- npc-15 (Significance 分级会驱动 `UHiNPCEnablePhysicsBoundCounterComponent::SetEnabled` 与 `UHiNpcSklMeshChangeComponent` 的 LOD 切换)
