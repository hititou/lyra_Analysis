# Lyra 角色系统 (Character System) 深度代码分析

## 目录

1. [系统架构总览](#1-系统架构总览)
2. [ULyraPawnData — 角色数据资产](#2-ulyrapawndata--角色数据资产)
3. [ALyraCharacter — 角色基类](#3-alyracharacter--角色基类)
4. [ULyraPawnExtensionComponent — 初始化协调器](#4-ulyrapawnextensioncomponent--初始化协调器)
5. [ULyraHeroComponent — 英雄组件](#5-ulyraherocomponent--英雄组件)
6. [ULyraHealthComponent — 生命组件](#6-ulyrahealthcomponent--生命组件)
7. [ULyraCharacterMovementComponent — 移动组件](#7-ulyracharactermovementcomponent--移动组件)
8. [ALyraCharacterWithAbilities — 自含ASC角色](#8-alyracharacterwithabilities--自含asc角色)
9. [ALyraPawn — 通用Pawn基类](#9-alyrapawn--通用pawn基类)
10. [外观系统 (Cosmetics)](#10-外观系统-cosmetics)
11. [初始化状态机详解](#11-初始化状态机详解)
12. [网络复制机制](#12-网络复制机制)
13. [完整架构图](#13-完整架构图)
14. [核心设计模式总结](#14-核心设计模式总结)
15. [文件索引](#15-文件索引)

---

## 1. 系统架构总览

Lyra的角色系统采用**组件化架构**，将传统的巨型Character拆分为多个职责单一的组件，通过`IGameFrameworkInitStateInterface`状态机协调初始化顺序。

### 1.1 核心设计理念

- **数据驱动**：`ULyraPawnData`定义角色的能力、输入、相机等配置
- **组件化**：功能分散到`PawnExtensionComponent`/`HeroComponent`/`HealthComponent`等
- **状态机协调**：通过4阶段InitState保证异步/网络环境下的正确初始化顺序
- **Modular Actor**：继承自`AModularCharacter`，支持GameFeature动态注入组件
- **ASC外置**：ASC默认挂在PlayerState上（跨Pawn持久化），而非Character自身

### 1.2 类层次结构

```
AModularCharacter (ModularGameplayActors插件)
  └── ALyraCharacter (核心角色基类)
        ├── 实现: IAbilitySystemInterface
        ├── 实现: IGameplayCueInterface
        ├── 实现: IGameplayTagAssetInterface
        ├── 实现: ILyraTeamAgentInterface
        ├── 组件: ULyraPawnExtensionComponent (初始化协调)
        ├── 组件: ULyraHealthComponent (生命/死亡)
        ├── 组件: ULyraCameraComponent (相机)
        ├── 组件: ULyraCharacterMovementComponent (移动)
        └── ALyraCharacterWithAbilities (自含ASC变体)

AModularPawn (ModularGameplayActors插件)
  └── ALyraPawn (通用Pawn基类，用于载具等非Character情况)
        └── 实现: ILyraTeamAgentInterface
```

### 1.3 组件职责分工

| 组件 | 核心职责 |
|------|----------|
| `ULyraPawnExtensionComponent` | 初始化协调、PawnData管理、ASC桥接 |
| `ULyraHeroComponent` | 玩家输入绑定、相机模式决策 |
| `ULyraHealthComponent` | 生命值管理、死亡状态机 |
| `ULyraCharacterMovementComponent` | 移动扩展（GAS集成、地面信息查询） |
| `ULyraCameraComponent` | 第三人称相机控制 |
| `ULyraPawnComponent_CharacterParts` | 外观部件管理（Cosmetics） |
| `ULyraControllerComponent_CharacterParts` | 控制器侧外观配置 |

---

## 2. ULyraPawnData — 角色数据资产

**文件**: `Source/LyraGame/Character/LyraPawnData.h/.cpp`

`ULyraPawnData`是一个不可变的数据资产（`UPrimaryDataAsset`），集中定义角色的所有配置。Experience通过`DefaultPawnData`引用它。

### 2.1 数据结构

```cpp
UCLASS(BlueprintType, Const, Meta = (DisplayName = "Lyra Pawn Data"))
class ULyraPawnData : public UPrimaryDataAsset
{
    // 要实例化的Pawn类（通常派生自ALyraPawn或ALyraCharacter）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Pawn")
    TSubclassOf<APawn> PawnClass;

    // 授予此Pawn的能力集
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Abilities")
    TArray<TObjectPtr<ULyraAbilitySet>> AbilitySets;

    // 能力标签关系映射（定义能力间的阻断/取消关系）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Abilities")
    TObjectPtr<ULyraAbilityTagRelationshipMapping> TagRelationshipMapping;

    // 输入配置（InputAction到GameplayTag的映射）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Input")
    TObjectPtr<ULyraInputConfig> InputConfig;

    // 默认相机模式
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Camera")
    TSubclassOf<ULyraCameraMode> DefaultCameraMode;
};
```

### 2.2 数据流向

```
Experience Definition
  └── DefaultPawnData: ULyraPawnData
        ├── PawnClass → GameMode::GetDefaultPawnClassForController 使用
        ├── AbilitySets → HeroComponent 在 DataInitialized 阶段授予
        ├── TagRelationshipMapping → PawnExtComp::InitializeAbilitySystem 设置到ASC
        ├── InputConfig → HeroComponent::InitializePlayerInput 绑定
        └── DefaultCameraMode → HeroComponent::DetermineCameraMode 查询
```

---

## 3. ALyraCharacter — 角色基类

**文件**: `Source/LyraGame/Character/LyraCharacter.h/.cpp`

### 3.1 构造函数 — 组件创建和默认参数

```cpp
ALyraCharacter::ALyraCharacter(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer.SetDefaultSubobjectClass<ULyraCharacterMovementComponent>(
        ACharacter::CharacterMovementComponentName))
{
    // 关闭Tick，角色不需要每帧Tick
    PrimaryActorTick.bCanEverTick = false;
    PrimaryActorTick.bStartWithTickEnabled = false;

    // 大范围网络裁剪距离
    SetNetCullDistanceSquared(900000000.0f);

    // 碰撞体配置
    UCapsuleComponent* CapsuleComp = GetCapsuleComponent();
    CapsuleComp->InitCapsuleSize(40.0f, 90.0f);
    CapsuleComp->SetCollisionProfileName("LyraPawnCapsule");

    // 骨骼网格体配置
    USkeletalMeshComponent* MeshComp = GetMesh();
    MeshComp->SetRelativeRotation(FRotator(0.0f, -90.0f, 0.0f)); // Y-forward → X-forward
    MeshComp->SetCollisionProfileName("LyraPawnMesh");

    // 移动组件默认参数
    ULyraCharacterMovementComponent* LyraMoveComp = CastChecked<ULyraCharacterMovementComponent>(GetCharacterMovement());
    LyraMoveComp->GravityScale = 1.0f;
    LyraMoveComp->MaxAcceleration = 2400.0f;
    LyraMoveComp->BrakingFriction = 6.0f;
    LyraMoveComp->GroundFriction = 8.0f;
    LyraMoveComp->BrakingDecelerationWalking = 1400.0f;
    LyraMoveComp->bUseControllerDesiredRotation = false;
    LyraMoveComp->bOrientRotationToMovement = false;
    LyraMoveComp->RotationRate = FRotator(0.0f, 720.0f, 0.0f);
    LyraMoveComp->GetNavAgentPropertiesRef().bCanCrouch = true;
    LyraMoveComp->SetCrouchedHalfHeight(65.0f);

    // 创建核心组件
    PawnExtComponent = CreateDefaultSubobject<ULyraPawnExtensionComponent>(TEXT("PawnExtensionComponent"));
    PawnExtComponent->OnAbilitySystemInitialized_RegisterAndCall(
        FSimpleMulticastDelegate::FDelegate::CreateUObject(this, &ThisClass::OnAbilitySystemInitialized));
    PawnExtComponent->OnAbilitySystemUninitialized_Register(
        FSimpleMulticastDelegate::FDelegate::CreateUObject(this, &ThisClass::OnAbilitySystemUninitialized));

    HealthComponent = CreateDefaultSubobject<ULyraHealthComponent>(TEXT("HealthComponent"));
    HealthComponent->OnDeathStarted.AddDynamic(this, &ThisClass::OnDeathStarted);
    HealthComponent->OnDeathFinished.AddDynamic(this, &ThisClass::OnDeathFinished);

    CameraComponent = CreateDefaultSubobject<ULyraCameraComponent>(TEXT("CameraComponent"));
    CameraComponent->SetRelativeLocation(FVector(-300.0f, 0.0f, 75.0f));

    // 旋转控制
    bUseControllerRotationPitch = false;
    bUseControllerRotationYaw = true;    // Yaw跟随控制器
    bUseControllerRotationRoll = false;

    BaseEyeHeight = 80.0f;
    CrouchedEyeHeight = 50.0f;
}
```

**设计要点**：
- `PrimaryActorTick`关闭 — 角色不需要每帧Tick，所有逻辑由组件或GAS驱动
- 通过`SetDefaultSubobjectClass`替换默认移动组件为`ULyraCharacterMovementComponent`
- 构造函数中立即注册ASC初始化/反初始化委托，使用`RegisterAndCall`模式

### 3.2 IAbilitySystemInterface 实现 — ASC外置模式

```cpp
UAbilitySystemComponent* ALyraCharacter::GetAbilitySystemComponent() const
{
    // ASC不在Character上，而是通过PawnExtensionComponent获取
    // PawnExtensionComponent缓存了对PlayerState上ASC的引用
    if (PawnExtComponent == nullptr)
    {
        return nullptr;
    }
    return PawnExtComponent->GetLyraAbilitySystemComponent();
}
```

**关键设计**：ASC默认挂在`ALyraPlayerState`上，而非Character。这意味着：
- 角色死亡重生后，能力和属性不会丢失
- Buff/Debuff状态跨Pawn保持
- 通过`PawnExtensionComponent::InitializeAbilitySystem()`将ASC与当前Pawn关联

### 3.3 ASC初始化回调

```cpp
void ALyraCharacter::OnAbilitySystemInitialized()
{
    ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent();
    check(LyraASC);

    // 将HealthComponent与ASC关联
    HealthComponent->InitializeWithAbilitySystem(LyraASC);

    // 初始化GameplayTag（清理旧Tag，设置当前移动模式Tag）
    InitializeGameplayTags();
}

void ALyraCharacter::OnAbilitySystemUninitialized()
{
    HealthComponent->UninitializeFromAbilitySystem();
}
```

### 3.4 控制器变更与团队系统

```cpp
void ALyraCharacter::PossessedBy(AController* NewController)
{
    const FGenericTeamId OldTeamID = MyTeamID;
    Super::PossessedBy(NewController);

    // 通知PawnExtComponent控制器变更
    PawnExtComponent->HandleControllerChanged();

    // 从控制器获取团队ID
    if (ILyraTeamAgentInterface* ControllerAsTeamProvider = Cast<ILyraTeamAgentInterface>(NewController))
    {
        MyTeamID = ControllerAsTeamProvider->GetGenericTeamId();
        ControllerAsTeamProvider->GetTeamChangedDelegateChecked().AddDynamic(
            this, &ThisClass::OnControllerChangedTeam);
    }
    ConditionalBroadcastTeamChanged(this, OldTeamID, MyTeamID);
}

void ALyraCharacter::UnPossessed()
{
    AController* const OldController = GetController();
    const FGenericTeamId OldTeamID = MyTeamID;

    // 取消监听旧控制器的团队变更
    if (ILyraTeamAgentInterface* ControllerAsTeamProvider = Cast<ILyraTeamAgentInterface>(OldController))
    {
        ControllerAsTeamProvider->GetTeamChangedDelegateChecked().RemoveAll(this);
    }

    Super::UnPossessed();
    PawnExtComponent->HandleControllerChanged();

    // 取消控制后重置团队ID
    MyTeamID = DetermineNewTeamAfterPossessionEnds(OldTeamID); // 默认返回NoTeam
    ConditionalBroadcastTeamChanged(this, OldTeamID, MyTeamID);
}
```

### 3.5 移动模式与GameplayTag同步

```cpp
void ALyraCharacter::OnMovementModeChanged(EMovementMode PrevMovementMode, uint8 PreviousCustomMode)
{
    Super::OnMovementModeChanged(PrevMovementMode, PreviousCustomMode);

    ULyraCharacterMovementComponent* LyraMoveComp = CastChecked<ULyraCharacterMovementComponent>(GetCharacterMovement());

    // 移除旧移动模式的Tag
    SetMovementModeTag(PrevMovementMode, PreviousCustomMode, false);
    // 添加新移动模式的Tag
    SetMovementModeTag(LyraMoveComp->MovementMode, LyraMoveComp->CustomMovementMode, true);
}

void ALyraCharacter::SetMovementModeTag(EMovementMode MovementMode, uint8 CustomMovementMode, bool bTagEnabled)
{
    if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
    {
        const FGameplayTag* MovementModeTag = nullptr;
        if (MovementMode == MOVE_Custom)
        {
            MovementModeTag = LyraGameplayTags::CustomMovementModeTagMap.Find(CustomMovementMode);
        }
        else
        {
            MovementModeTag = LyraGameplayTags::MovementModeTagMap.Find(MovementMode);
        }

        if (MovementModeTag && MovementModeTag->IsValid())
        {
            LyraASC->SetLooseGameplayTagCount(*MovementModeTag, (bTagEnabled ? 1 : 0));
        }
    }
}
```

**用途**：GAS中的能力可以通过Tag查询当前移动状态（Walking/Falling/Swimming等），实现移动状态条件判断。

### 3.6 蹲伏 — Tag同步与跳跃修改

```cpp
void ALyraCharacter::OnStartCrouch(float HalfHeightAdjust, float ScaledHalfHeightAdjust)
{
    if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
    {
        LyraASC->SetLooseGameplayTagCount(LyraGameplayTags::Status_Crouching, 1);
    }
    Super::OnStartCrouch(HalfHeightAdjust, ScaledHalfHeightAdjust);
}

// 移除蹲伏检查，允许蹲伏时跳跃
bool ALyraCharacter::CanJumpInternal_Implementation() const
{
    return JumpIsAllowedInternal(); // 不检查IsCrouched
}
```

### 3.7 死亡流程

```cpp
void ALyraCharacter::OnDeathStarted(AActor*)
{
    DisableMovementAndCollision();
}

void ALyraCharacter::OnDeathFinished(AActor*)
{
    // 延迟到下一帧销毁，避免在回调中直接销毁
    GetWorld()->GetTimerManager().SetTimerForNextTick(this, &ThisClass::DestroyDueToDeath);
}

void ALyraCharacter::DisableMovementAndCollision()
{
    if (GetController())
    {
        GetController()->SetIgnoreMoveInput(true);
    }

    UCapsuleComponent* CapsuleComp = GetCapsuleComponent();
    CapsuleComp->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    CapsuleComp->SetCollisionResponseToAllChannels(ECR_Ignore);

    ULyraCharacterMovementComponent* LyraMoveComp = CastChecked<ULyraCharacterMovementComponent>(GetCharacterMovement());
    LyraMoveComp->StopMovementImmediately();
    LyraMoveComp->DisableMovement();
}

void ALyraCharacter::UninitAndDestroy()
{
    if (GetLocalRole() == ROLE_Authority)
    {
        DetachFromControllerPendingDestroy();
        SetLifeSpan(0.1f);
    }

    // 仅当我们仍是ASC的Avatar时才反初始化ASC
    if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
    {
        if (LyraASC->GetAvatarActor() == this)
        {
            PawnExtComponent->UninitializeAbilitySystem();
        }
    }

    SetActorHiddenInGame(true);
}
```

### 3.8 GameplayTag接口代理

```cpp
void ALyraCharacter::GetOwnedGameplayTags(FGameplayTagContainer& TagContainer) const
{
    // 将所有Tag查询代理到ASC
    if (const ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
    {
        LyraASC->GetOwnedGameplayTags(TagContainer);
    }
}

bool ALyraCharacter::HasMatchingGameplayTag(FGameplayTag TagToCheck) const
{
    if (const ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
    {
        return LyraASC->HasMatchingGameplayTag(TagToCheck);
    }
    return false;
}
// HasAllMatchingGameplayTags / HasAnyMatchingGameplayTags 同理
```

### 3.9 SignificanceManager 集成

```cpp
void ALyraCharacter::BeginPlay()
{
    Super::BeginPlay();

    // 非DedicatedServer上注册到SignificanceManager
    const bool bRegisterWithSignificanceManager = !IsNetMode(NM_DedicatedServer);
    if (bRegisterWithSignificanceManager)
    {
        if (ULyraSignificanceManager* SignificanceManager = 
            USignificanceManager::Get<ULyraSignificanceManager>(GetWorld()))
        {
            // @TODO: SignificanceManager->RegisterObject(this, SignificanceType);
        }
    }
}
```

---

## 4. ULyraPawnExtensionComponent — 初始化协调器

**文件**: `Source/LyraGame/Character/LyraPawnExtensionComponent.h/.cpp`

这是Lyra角色系统的**核心协调组件**，负责管理PawnData、协调所有组件的初始化顺序、桥接ASC。

### 4.1 InitState 状态机

实现`IGameFrameworkInitStateInterface`，定义4阶段状态链：

```
Spawned → DataAvailable → DataInitialized → GameplayReady
```

```cpp
const FName ULyraPawnExtensionComponent::NAME_ActorFeatureName("PawnExtension");

void ULyraPawnExtensionComponent::CheckDefaultInitialization()
{
    // 先尝试推进依赖的其他Feature
    CheckDefaultInitializationForImplementers();

    // 状态链定义
    static const TArray<FGameplayTag> StateChain = {
        LyraGameplayTags::InitState_Spawned,
        LyraGameplayTags::InitState_DataAvailable,
        LyraGameplayTags::InitState_DataInitialized,
        LyraGameplayTags::InitState_GameplayReady
    };

    // 尝试沿状态链推进
    ContinueInitStateChain(StateChain);
}
```

### 4.2 状态转换条件

```cpp
bool ULyraPawnExtensionComponent::CanChangeInitState(UGameFrameworkComponentManager* Manager,
    FGameplayTag CurrentState, FGameplayTag DesiredState) const
{
    APawn* Pawn = GetPawn<APawn>();

    if (!CurrentState.IsValid() && DesiredState == InitState_Spawned)
    {
        // 只要有有效Pawn即可 → Spawned
        return (Pawn != nullptr);
    }
    if (CurrentState == InitState_Spawned && DesiredState == InitState_DataAvailable)
    {
        // 需要PawnData已设置
        if (!PawnData) return false;

        // 权威端或本地控制端需要Controller
        const bool bHasAuthority = Pawn->HasAuthority();
        const bool bIsLocallyControlled = Pawn->IsLocallyControlled();
        if (bHasAuthority || bIsLocallyControlled)
        {
            if (!GetController<AController>()) return false;
        }
        return true;
    }
    else if (CurrentState == InitState_DataAvailable && DesiredState == InitState_DataInitialized)
    {
        // 等待所有Feature组件都达到DataAvailable
        return Manager->HaveAllFeaturesReachedInitState(Pawn, InitState_DataAvailable);
    }
    else if (CurrentState == InitState_DataInitialized && DesiredState == InitState_GameplayReady)
    {
        return true;
    }
    return false;
}
```

### 4.3 监听其他Feature的状态变化

```cpp
void ULyraPawnExtensionComponent::BeginPlay()
{
    Super::BeginPlay();

    // 监听所有Feature的状态变化（NAME_None = 所有Feature）
    BindOnActorInitStateChanged(NAME_None, FGameplayTag(), false);

    // 标记自己已Spawned
    ensure(TryToChangeInitState(LyraGameplayTags::InitState_Spawned));
    CheckDefaultInitialization();
}

void ULyraPawnExtensionComponent::OnActorInitStateChanged(const FActorInitStateChangedParams& Params)
{
    // 当其他Feature达到DataAvailable时，尝试推进自身状态
    if (Params.FeatureName != NAME_ActorFeatureName)
    {
        if (Params.FeatureState == LyraGameplayTags::InitState_DataAvailable)
        {
            CheckDefaultInitialization();
        }
    }
}
```

### 4.4 PawnData设置与复制

```cpp
void ULyraPawnExtensionComponent::SetPawnData(const ULyraPawnData* InPawnData)
{
    check(InPawnData);
    APawn* Pawn = GetPawnChecked<APawn>();

    // 仅Authority可设置
    if (Pawn->GetLocalRole() != ROLE_Authority) return;

    // 只能设置一次
    if (PawnData)
    {
        UE_LOG(LogLyra, Error, TEXT("Trying to set PawnData on pawn that already has valid PawnData."));
        return;
    }

    PawnData = InPawnData;
    Pawn->ForceNetUpdate();
    CheckDefaultInitialization(); // 推进状态机
}

// 客户端收到复制的PawnData
void ULyraPawnExtensionComponent::OnRep_PawnData()
{
    CheckDefaultInitialization();
}
```

### 4.5 ASC桥接

```cpp
void ULyraPawnExtensionComponent::InitializeAbilitySystem(
    ULyraAbilitySystemComponent* InASC, AActor* InOwnerActor)
{
    check(InASC && InOwnerActor);

    if (AbilitySystemComponent == InASC) return;
    if (AbilitySystemComponent) UninitializeAbilitySystem();

    APawn* Pawn = GetPawnChecked<APawn>();
    AActor* ExistingAvatar = InASC->GetAvatarActor();

    // 踢掉旧的Avatar（客户端延迟场景：新Pawn先到，旧Pawn未清理）
    if ((ExistingAvatar != nullptr) && (ExistingAvatar != Pawn))
    {
        ensure(!ExistingAvatar->HasAuthority());
        if (ULyraPawnExtensionComponent* OtherExtComp = FindPawnExtensionComponent(ExistingAvatar))
        {
            OtherExtComp->UninitializeAbilitySystem();
        }
    }

    AbilitySystemComponent = InASC;
    AbilitySystemComponent->InitAbilityActorInfo(InOwnerActor, Pawn);

    // 设置Tag关系映射
    if (ensure(PawnData))
    {
        InASC->SetTagRelationshipMapping(PawnData->TagRelationshipMapping);
    }

    OnAbilitySystemInitialized.Broadcast();
}

void ULyraPawnExtensionComponent::UninitializeAbilitySystem()
{
    if (!AbilitySystemComponent) return;

    // 仅当我们仍是Avatar时才清理
    if (AbilitySystemComponent->GetAvatarActor() == GetOwner())
    {
        FGameplayTagContainer AbilityTypesToIgnore;
        AbilityTypesToIgnore.AddTag(LyraGameplayTags::Ability_Behavior_SurvivesDeath);

        AbilitySystemComponent->CancelAbilities(nullptr, &AbilityTypesToIgnore);
        AbilitySystemComponent->ClearAbilityInput();
        AbilitySystemComponent->RemoveAllGameplayCues();

        if (AbilitySystemComponent->GetOwnerActor() != nullptr)
        {
            AbilitySystemComponent->SetAvatarActor(nullptr);
        }
        else
        {
            AbilitySystemComponent->ClearActorInfo();
        }

        OnAbilitySystemUninitialized.Broadcast();
    }

    AbilitySystemComponent = nullptr;
}
```

**关键细节**：
- 反初始化时，标记了`Ability_Behavior_SurvivesDeath`的能力**不会**被取消
- 先检查当前Avatar是否是自己，避免重复清理

### 4.6 RegisterAndCall 模式

```cpp
void ULyraPawnExtensionComponent::OnAbilitySystemInitialized_RegisterAndCall(
    FSimpleMulticastDelegate::FDelegate Delegate)
{
    if (!OnAbilitySystemInitialized.IsBoundToObject(Delegate.GetUObject()))
    {
        OnAbilitySystemInitialized.Add(Delegate);
    }

    // 如果ASC已初始化，立即执行回调（解决注册时序问题）
    if (AbilitySystemComponent)
    {
        Delegate.Execute();
    }
}
```

---

## 5. ULyraHeroComponent — 英雄组件

**文件**: `Source/LyraGame/Character/LyraHeroComponent.h/.cpp`

处理**玩家特有**的逻辑：输入绑定、相机模式决策。AI Bot也可使用（模拟玩家行为）。

### 5.1 InitState 集成

```cpp
const FName ULyraHeroComponent::NAME_ActorFeatureName("Hero");
const FName ULyraHeroComponent::NAME_BindInputsNow("BindInputsNow");

void ULyraHeroComponent::BeginPlay()
{
    Super::BeginPlay();

    // 监听PawnExtension组件的状态变化
    BindOnActorInitStateChanged(ULyraPawnExtensionComponent::NAME_ActorFeatureName, FGameplayTag(), false);

    ensure(TryToChangeInitState(LyraGameplayTags::InitState_Spawned));
    CheckDefaultInitialization();
}
```

### 5.2 状态转换条件

```cpp
bool ULyraHeroComponent::CanChangeInitState(...) const
{
    // Spawned → DataAvailable 的额外条件：
    if (CurrentState == InitState_Spawned && DesiredState == InitState_DataAvailable)
    {
        // 需要PlayerState
        if (!GetPlayerState<ALyraPlayerState>()) return false;

        // 非SimulatedProxy需要Controller与PlayerState正确配对
        if (Pawn->GetLocalRole() != ROLE_SimulatedProxy)
        {
            AController* Controller = GetController<AController>();
            const bool bHasControllerPairedWithPS = (Controller != nullptr) &&
                (Controller->PlayerState != nullptr) &&
                (Controller->PlayerState->GetOwner() == Controller);
            if (!bHasControllerPairedWithPS) return false;
        }

        // 本地玩家需要InputComponent和LocalPlayer
        const bool bIsLocallyControlled = Pawn->IsLocallyControlled();
        const bool bIsBot = Pawn->IsBotControlled();
        if (bIsLocallyControlled && !bIsBot)
        {
            ALyraPlayerController* LyraPC = GetController<ALyraPlayerController>();
            if (!Pawn->InputComponent || !LyraPC || !LyraPC->GetLocalPlayer())
                return false;
        }

        return true;
    }

    // DataAvailable → DataInitialized：需要PawnExtension已到达DataInitialized
    if (CurrentState == InitState_DataAvailable && DesiredState == InitState_DataInitialized)
    {
        ALyraPlayerState* LyraPS = GetPlayerState<ALyraPlayerState>();
        return LyraPS && Manager->HasFeatureReachedInitState(Pawn,
            ULyraPawnExtensionComponent::NAME_ActorFeatureName,
            LyraGameplayTags::InitState_DataInitialized);
    }
}
```

### 5.3 DataInitialized — 核心初始化

```cpp
void ULyraHeroComponent::HandleChangeInitState(UGameFrameworkComponentManager* Manager,
    FGameplayTag CurrentState, FGameplayTag DesiredState)
{
    if (CurrentState == InitState_DataAvailable && DesiredState == InitState_DataInitialized)
    {
        APawn* Pawn = GetPawn<APawn>();
        ALyraPlayerState* LyraPS = GetPlayerState<ALyraPlayerState>();

        if (ULyraPawnExtensionComponent* PawnExtComp = ULyraPawnExtensionComponent::FindPawnExtensionComponent(Pawn))
        {
            PawnData = PawnExtComp->GetPawnData<ULyraPawnData>();

            // 核心：将PlayerState的ASC与当前Pawn关联
            PawnExtComp->InitializeAbilitySystem(LyraPS->GetLyraAbilitySystemComponent(), LyraPS);
        }

        // 初始化输入
        if (ALyraPlayerController* LyraPC = GetController<ALyraPlayerController>())
        {
            if (Pawn->InputComponent != nullptr)
            {
                InitializePlayerInput(Pawn->InputComponent);
            }
        }

        // 绑定相机模式决策
        if (PawnData)
        {
            if (ULyraCameraComponent* CameraComponent = ULyraCameraComponent::FindCameraComponent(Pawn))
            {
                CameraComponent->DetermineCameraModeDelegate.BindUObject(
                    this, &ThisClass::DetermineCameraMode);
            }
        }
    }
}
```

### 5.4 输入初始化

```cpp
void ULyraHeroComponent::InitializePlayerInput(UInputComponent* PlayerInputComponent)
{
    const APawn* Pawn = GetPawn<APawn>();
    const APlayerController* PC = GetController<APlayerController>();
    const ULyraLocalPlayer* LP = Cast<ULyraLocalPlayer>(PC->GetLocalPlayer());
    UEnhancedInputLocalPlayerSubsystem* Subsystem = LP->GetSubsystem<UEnhancedInputLocalPlayerSubsystem>();

    Subsystem->ClearAllMappings();

    if (const ULyraPawnData* PawnData = PawnExtComp->GetPawnData<ULyraPawnData>())
    {
        if (const ULyraInputConfig* InputConfig = PawnData->InputConfig)
        {
            // 添加默认InputMappingContext
            for (const FInputMappingContextAndPriority& Mapping : DefaultInputMappings)
            {
                if (UInputMappingContext* IMC = Mapping.InputMapping.LoadSynchronous())
                {
                    if (Mapping.bRegisterWithSettings)
                    {
                        if (UEnhancedInputUserSettings* Settings = Subsystem->GetUserSettings())
                        {
                            Settings->RegisterInputMappingContext(IMC);
                        }
                        Subsystem->AddMappingContext(IMC, Mapping.Priority);
                    }
                }
            }

            ULyraInputComponent* LyraIC = Cast<ULyraInputComponent>(PlayerInputComponent);
            if (ensure(LyraIC))
            {
                LyraIC->AddInputMappings(InputConfig, Subsystem);

                // 绑定能力输入（GameplayTag驱动）
                TArray<uint32> BindHandles;
                LyraIC->BindAbilityActions(InputConfig, this,
                    &ThisClass::Input_AbilityInputTagPressed,
                    &ThisClass::Input_AbilityInputTagReleased,
                    BindHandles);

                // 绑定原生输入（移动/视角/蹲伏/自动跑）
                LyraIC->BindNativeAction(InputConfig, InputTag_Move, ETriggerEvent::Triggered,
                    this, &ThisClass::Input_Move, false);
                LyraIC->BindNativeAction(InputConfig, InputTag_Look_Mouse, ETriggerEvent::Triggered,
                    this, &ThisClass::Input_LookMouse, false);
                LyraIC->BindNativeAction(InputConfig, InputTag_Look_Stick, ETriggerEvent::Triggered,
                    this, &ThisClass::Input_LookStick, false);
                LyraIC->BindNativeAction(InputConfig, InputTag_Crouch, ETriggerEvent::Triggered,
                    this, &ThisClass::Input_Crouch, false);
                LyraIC->BindNativeAction(InputConfig, InputTag_AutoRun, ETriggerEvent::Triggered,
                    this, &ThisClass::Input_AutoRun, false);
            }
        }
    }

    bReadyToBindInputs = true;

    // 发送扩展事件，通知GameFeature可以绑定额外输入了
    UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(
        const_cast<APlayerController*>(PC), NAME_BindInputsNow);
    UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(
        const_cast<APawn*>(Pawn), NAME_BindInputsNow);
}
```

### 5.5 输入处理函数

```cpp
void ULyraHeroComponent::Input_Move(const FInputActionValue& InputActionValue)
{
    APawn* Pawn = GetPawn<APawn>();
    AController* Controller = Pawn ? Pawn->GetController() : nullptr;

    // 移动输入取消自动跑
    if (ALyraPlayerController* LyraController = Cast<ALyraPlayerController>(Controller))
    {
        LyraController->SetIsAutoRunning(false);
    }

    if (Controller)
    {
        const FVector2D Value = InputActionValue.Get<FVector2D>();
        const FRotator MovementRotation(0.0f, Controller->GetControlRotation().Yaw, 0.0f);

        // 基于控制器Yaw方向的相对移动
        if (Value.X != 0.0f)
        {
            const FVector MovementDirection = MovementRotation.RotateVector(FVector::RightVector);
            Pawn->AddMovementInput(MovementDirection, Value.X);
        }
        if (Value.Y != 0.0f)
        {
            const FVector MovementDirection = MovementRotation.RotateVector(FVector::ForwardVector);
            Pawn->AddMovementInput(MovementDirection, Value.Y);
        }
    }
}

void ULyraHeroComponent::Input_LookMouse(const FInputActionValue& InputActionValue)
{
    // 鼠标：直接添加Yaw/Pitch
    APawn* Pawn = GetPawn<APawn>();
    const FVector2D Value = InputActionValue.Get<FVector2D>();
    if (Value.X != 0.0f) Pawn->AddControllerYawInput(Value.X);
    if (Value.Y != 0.0f) Pawn->AddControllerPitchInput(Value.Y);
}

void ULyraHeroComponent::Input_LookStick(const FInputActionValue& InputActionValue)
{
    // 手柄：乘以速率和DeltaTime
    APawn* Pawn = GetPawn<APawn>();
    const FVector2D Value = InputActionValue.Get<FVector2D>();
    const UWorld* World = GetWorld();
    if (Value.X != 0.0f)
        Pawn->AddControllerYawInput(Value.X * LookYawRate * World->GetDeltaSeconds());
    if (Value.Y != 0.0f)
        Pawn->AddControllerPitchInput(Value.Y * LookPitchRate * World->GetDeltaSeconds());
}

// 能力输入：转发到ASC
void ULyraHeroComponent::Input_AbilityInputTagPressed(FGameplayTag InputTag)
{
    if (const APawn* Pawn = GetPawn<APawn>())
    {
        if (const ULyraPawnExtensionComponent* PawnExtComp = 
            ULyraPawnExtensionComponent::FindPawnExtensionComponent(Pawn))
        {
            if (ULyraAbilitySystemComponent* LyraASC = PawnExtComp->GetLyraAbilitySystemComponent())
            {
                LyraASC->AbilityInputTagPressed(InputTag);
            }
        }
    }
}
```

### 5.6 相机模式决策

```cpp
TSubclassOf<ULyraCameraMode> ULyraHeroComponent::DetermineCameraMode() const
{
    // 优先级：能力相机 > PawnData默认相机
    if (AbilityCameraMode)
    {
        return AbilityCameraMode;
    }

    const APawn* Pawn = GetPawn<APawn>();
    if (ULyraPawnExtensionComponent* PawnExtComp = ULyraPawnExtensionComponent::FindPawnExtensionComponent(Pawn))
    {
        if (const ULyraPawnData* PawnData = PawnExtComp->GetPawnData<ULyraPawnData>())
        {
            return PawnData->DefaultCameraMode;
        }
    }
    return nullptr;
}

void ULyraHeroComponent::SetAbilityCameraMode(TSubclassOf<ULyraCameraMode> CameraMode,
    const FGameplayAbilitySpecHandle& OwningSpecHandle)
{
    if (CameraMode)
    {
        AbilityCameraMode = CameraMode;
        AbilityCameraModeOwningSpecHandle = OwningSpecHandle;
    }
}

void ULyraHeroComponent::ClearAbilityCameraMode(const FGameplayAbilitySpecHandle& OwningSpecHandle)
{
    // 仅清除由同一能力设置的相机模式
    if (AbilityCameraModeOwningSpecHandle == OwningSpecHandle)
    {
        AbilityCameraMode = nullptr;
        AbilityCameraModeOwningSpecHandle = FGameplayAbilitySpecHandle();
    }
}
```

---

## 6. ULyraHealthComponent — 生命组件

**文件**: `Source/LyraGame/Character/LyraHealthComponent.h/.cpp`

### 6.1 死亡状态机

```cpp
UENUM(BlueprintType)
enum class ELyraDeathState : uint8
{
    NotDead = 0,
    DeathStarted,    // 死亡开始（播放动画、禁用碰撞）
    DeathFinished    // 死亡完成（准备销毁）
};
```

### 6.2 与ASC的绑定

```cpp
void ULyraHealthComponent::InitializeWithAbilitySystem(ULyraAbilitySystemComponent* InASC)
{
    AbilitySystemComponent = InASC;

    // 从ASC获取HealthSet（属性集）
    HealthSet = AbilitySystemComponent->GetSet<ULyraHealthSet>();

    // 监听属性变化
    HealthSet->OnHealthChanged.AddUObject(this, &ThisClass::HandleHealthChanged);
    HealthSet->OnMaxHealthChanged.AddUObject(this, &ThisClass::HandleMaxHealthChanged);
    HealthSet->OnOutOfHealth.AddUObject(this, &ThisClass::HandleOutOfHealth);

    // 初始化血量为最大值
    AbilitySystemComponent->SetNumericAttributeBase(
        ULyraHealthSet::GetHealthAttribute(), HealthSet->GetMaxHealth());

    ClearGameplayTags();

    // 广播初始值
    OnHealthChanged.Broadcast(this, HealthSet->GetHealth(), HealthSet->GetHealth(), nullptr);
    OnMaxHealthChanged.Broadcast(this, HealthSet->GetHealth(), HealthSet->GetHealth(), nullptr);
}
```

### 6.3 血量耗尽处理

```cpp
void ULyraHealthComponent::HandleOutOfHealth(AActor* DamageInstigator, AActor* DamageCauser,
    const FGameplayEffectSpec* DamageEffectSpec, float DamageMagnitude, float OldValue, float NewValue)
{
#if WITH_SERVER_CODE
    if (AbilitySystemComponent && DamageEffectSpec)
    {
        // 发送 GameplayEvent.Death 事件到ASC
        // 这将触发死亡GameplayAbility
        FGameplayEventData Payload;
        Payload.EventTag = LyraGameplayTags::GameplayEvent_Death;
        Payload.Instigator = DamageInstigator;
        Payload.Target = AbilitySystemComponent->GetAvatarActor();
        Payload.OptionalObject = DamageEffectSpec->Def;
        Payload.ContextHandle = DamageEffectSpec->GetEffectContext();
        Payload.InstigatorTags = *DamageEffectSpec->CapturedSourceTags.GetAggregatedTags();
        Payload.TargetTags = *DamageEffectSpec->CapturedTargetTags.GetAggregatedTags();
        Payload.EventMagnitude = DamageMagnitude;

        FScopedPredictionWindow NewScopedWindow(AbilitySystemComponent, true);
        AbilitySystemComponent->HandleGameplayEvent(Payload.EventTag, &Payload);

        // 发送标准化消息（Lyra.Elimination.Message）
        // 供HUD、统计等系统监听
        FLyraVerbMessage Message;
        Message.Verb = TAG_Lyra_Elimination_Message;
        Message.Instigator = DamageInstigator;
        Message.Target = ULyraVerbMessageHelpers::GetPlayerStateFromObject(
            AbilitySystemComponent->GetAvatarActor());

        UGameplayMessageSubsystem& MessageSystem = UGameplayMessageSubsystem::Get(GetWorld());
        MessageSystem.BroadcastMessage(Message.Verb, Message);
    }
#endif
}
```

### 6.4 死亡状态转换

```cpp
void ULyraHealthComponent::StartDeath()
{
    if (DeathState != ELyraDeathState::NotDead) return;

    DeathState = ELyraDeathState::DeathStarted;

    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->SetLooseGameplayTagCount(LyraGameplayTags::Status_Death_Dying, 1);
    }

    AActor* Owner = GetOwner();
    OnDeathStarted.Broadcast(Owner);
    Owner->ForceNetUpdate();
}

void ULyraHealthComponent::FinishDeath()
{
    if (DeathState != ELyraDeathState::DeathStarted) return;

    DeathState = ELyraDeathState::DeathFinished;

    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->SetLooseGameplayTagCount(LyraGameplayTags::Status_Death_Dead, 1);
    }

    AActor* Owner = GetOwner();
    OnDeathFinished.Broadcast(Owner);
    Owner->ForceNetUpdate();
}
```

### 6.5 网络复制与死亡状态同步

```cpp
void ULyraHealthComponent::OnRep_DeathState(ELyraDeathState OldDeathState)
{
    const ELyraDeathState NewDeathState = DeathState;

    // 暂时回退状态，因为StartDeath/FinishDeath会推进它
    DeathState = OldDeathState;

    // 防止服务器状态回退
    if (OldDeathState > NewDeathState)
    {
        UE_LOG(LogLyra, Warning, TEXT("Predicted past server death state [%d] -> [%d]"),
            (uint8)OldDeathState, (uint8)NewDeathState);
        return;
    }

    // 按顺序执行死亡流程
    if (OldDeathState == ELyraDeathState::NotDead)
    {
        if (NewDeathState == ELyraDeathState::DeathStarted)
        {
            StartDeath();
        }
        else if (NewDeathState == ELyraDeathState::DeathFinished)
        {
            StartDeath();
            FinishDeath();
        }
    }
    else if (OldDeathState == ELyraDeathState::DeathStarted)
    {
        if (NewDeathState == ELyraDeathState::DeathFinished)
        {
            FinishDeath();
        }
    }
}
```

### 6.6 自毁机制

```cpp
void ULyraHealthComponent::DamageSelfDestruct(bool bFellOutOfWorld)
{
    if ((DeathState == ELyraDeathState::NotDead) && AbilitySystemComponent)
    {
        // 使用GAS的SetByCaller伤害GE
        const TSubclassOf<UGameplayEffect> DamageGE = 
            ULyraAssetManager::GetSubclass(ULyraGameData::Get().DamageGameplayEffect_SetByCaller);

        FGameplayEffectSpecHandle SpecHandle = AbilitySystemComponent->MakeOutgoingSpec(
            DamageGE, 1.0f, AbilitySystemComponent->MakeEffectContext());
        FGameplayEffectSpec* Spec = SpecHandle.Data.Get();

        Spec->AddDynamicAssetTag(TAG_Gameplay_DamageSelfDestruct);
        if (bFellOutOfWorld)
        {
            Spec->AddDynamicAssetTag(TAG_Gameplay_FellOutOfWorld);
        }

        // 施加等于最大血量的伤害
        Spec->SetSetByCallerMagnitude(LyraGameplayTags::SetByCaller_Damage, GetMaxHealth());
        AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*Spec);
    }
}
```

---

## 7. ULyraCharacterMovementComponent — 移动组件

**文件**: `Source/LyraGame/Character/LyraCharacterMovementComponent.h/.cpp`

### 7.1 GAS集成 — Tag驱动的移动控制

```cpp
UE_DEFINE_GAMEPLAY_TAG(TAG_Gameplay_MovementStopped, "Gameplay.MovementStopped");

FRotator ULyraCharacterMovementComponent::GetDeltaRotation(float DeltaTime) const
{
    // 如果有MovementStopped Tag，禁止旋转
    if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(GetOwner()))
    {
        if (ASC->HasMatchingGameplayTag(TAG_Gameplay_MovementStopped))
        {
            return FRotator(0, 0, 0);
        }
    }
    return Super::GetDeltaRotation(DeltaTime);
}

float ULyraCharacterMovementComponent::GetMaxSpeed() const
{
    // 如果有MovementStopped Tag，速度为0
    if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(GetOwner()))
    {
        if (ASC->HasMatchingGameplayTag(TAG_Gameplay_MovementStopped))
        {
            return 0;
        }
    }
    return Super::GetMaxSpeed();
}
```

### 7.2 地面信息查询（帧缓存）

```cpp
struct FLyraCharacterGroundInfo
{
    uint64 LastUpdateFrame;         // 上次更新的帧号
    FHitResult GroundHitResult;     // 地面射线检测结果
    float GroundDistance;           // 距离地面的距离
};

const FLyraCharacterGroundInfo& ULyraCharacterMovementComponent::GetGroundInfo()
{
    // 帧缓存：同一帧内不重复计算
    if (!CharacterOwner || (GFrameCounter == CachedGroundInfo.LastUpdateFrame))
    {
        return CachedGroundInfo;
    }

    if (MovementMode == MOVE_Walking)
    {
        // 行走模式直接使用CurrentFloor
        CachedGroundInfo.GroundHitResult = CurrentFloor.HitResult;
        CachedGroundInfo.GroundDistance = 0.0f;
    }
    else
    {
        // 非行走模式执行向下射线检测
        const float CapsuleHalfHeight = CharacterOwner->GetCapsuleComponent()->GetUnscaledCapsuleHalfHeight();
        const FVector TraceStart(GetActorLocation());
        const FVector TraceEnd(TraceStart.X, TraceStart.Y, TraceStart.Z - GroundTraceDistance - CapsuleHalfHeight);

        FHitResult HitResult;
        GetWorld()->LineTraceSingleByChannel(HitResult, TraceStart, TraceEnd, CollisionChannel, QueryParams);

        CachedGroundInfo.GroundHitResult = HitResult;
        CachedGroundInfo.GroundDistance = HitResult.bBlockingHit ?
            FMath::Max((HitResult.Distance - CapsuleHalfHeight), 0.0f) : GroundTraceDistance;
    }

    CachedGroundInfo.LastUpdateFrame = GFrameCounter;
    return CachedGroundInfo;
}
```

### 7.3 复制加速度处理

```cpp
void ULyraCharacterMovementComponent::SimulateMovement(float DeltaTime)
{
    if (bHasReplicatedAcceleration)
    {
        // 保留复制的加速度值，不被SimulateMovement覆盖
        const FVector OriginalAcceleration = Acceleration;
        Super::SimulateMovement(DeltaTime);
        Acceleration = OriginalAcceleration;
    }
    else
    {
        Super::SimulateMovement(DeltaTime);
    }
}

void ULyraCharacterMovementComponent::SetReplicatedAcceleration(const FVector& InAcceleration)
{
    bHasReplicatedAcceleration = true;
    Acceleration = InAcceleration;
}
```

### 7.4 蹲伏跳跃

```cpp
bool ULyraCharacterMovementComponent::CanAttemptJump() const
{
    // 移除蹲伏检查（与LyraCharacter::CanJumpInternal_Implementation配合）
    return IsJumpAllowed() &&
        (IsMovingOnGround() || IsFalling());
}
```

---

## 8. ALyraCharacterWithAbilities — 自含ASC角色

**文件**: `Source/LyraGame/Character/LyraCharacterWithAbilities.h/.cpp`

标准的`ALyraCharacter`从PlayerState获取ASC。`ALyraCharacterWithAbilities`则在角色自身创建ASC，用于**AI角色**或**非玩家控制的角色**。

### 8.1 ASC自包含

```cpp
UCLASS(Blueprintable)
class ALyraCharacterWithAbilities : public ALyraCharacter
{
    // ASC在Character上而非PlayerState上
    UPROPERTY(VisibleAnywhere, Category = "Lyra|PlayerState")
    TObjectPtr<ULyraAbilitySystemComponent> AbilitySystemComponent;

    // 属性集也在Character上
    UPROPERTY()
    TObjectPtr<const ULyraHealthSet> HealthSet;
    UPROPERTY()
    TObjectPtr<const ULyraCombatSet> CombatSet;
};

ALyraCharacterWithAbilities::ALyraCharacterWithAbilities(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    AbilitySystemComponent = ObjectInitializer.CreateDefaultSubobject<ULyraAbilitySystemComponent>(
        this, TEXT("AbilitySystemComponent"));
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);

    // 属性集作为子对象创建，会被ASC::InitializeComponent自动检测
    HealthSet = CreateDefaultSubobject<ULyraHealthSet>(TEXT("HealthSet"));
    CombatSet = CreateDefaultSubobject<ULyraCombatSet>(TEXT("CombatSet"));

    SetNetUpdateFrequency(100.0f);
}

void ALyraCharacterWithAbilities::PostInitializeComponents()
{
    Super::PostInitializeComponents();

    // Owner和Avatar都是自身
    AbilitySystemComponent->InitAbilityActorInfo(this, this);
}

UAbilitySystemComponent* ALyraCharacterWithAbilities::GetAbilitySystemComponent() const
{
    // 覆盖基类的ASC获取，直接返回自身的ASC
    return AbilitySystemComponent;
}
```

**使用场景**：
- AI NPC（无PlayerState）
- 临时可控载具
- 环境互动物体

---

## 9. ALyraPawn — 通用Pawn基类

**文件**: `Source/LyraGame/Character/LyraPawn.h/.cpp`

`ALyraPawn`继承自`AModularPawn`，用于不需要Character移动/碰撞体的场景（如载具、可控设备等）。

### 9.1 类定义

```cpp
UCLASS()
class ALyraPawn : public AModularPawn, public ILyraTeamAgentInterface
{
    // 与ALyraCharacter相同的团队系统实现
    UPROPERTY(ReplicatedUsing = OnRep_MyTeamID)
    FGenericTeamId MyTeamID;

    UPROPERTY()
    FOnLyraTeamIndexChangedDelegate OnTeamChangedDelegate;
};
```

### 9.2 团队系统实现

`ALyraPawn`的`PossessedBy`/`UnPossessed`/`SetGenericTeamId`实现与`ALyraCharacter`**完全相同**——从Controller获取TeamID，监听团队变更委托，权威端才能修改TeamID。

**与ALyraCharacter的区别**：
- 没有`PawnExtensionComponent`/`HealthComponent`/`CameraComponent`
- 没有ASC集成
- 没有移动相关逻辑
- 纯粹提供团队接口和Modular Actor支持

---

## 10. 外观系统 (Cosmetics)

**文件目录**: `Source/LyraGame/Cosmetics/`

Lyra的外观系统采用**Controller-Pawn双组件架构**，将外观配置（Controller侧）与外观实例化（Pawn侧）分离。

### 10.1 数据类型

**FLyraCharacterPart** — 单个外观部件描述：

```cpp
USTRUCT(BlueprintType)
struct FLyraCharacterPart
{
    // 要生成的部件Actor类
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSubclassOf<AActor> PartClass;

    // 附着的Socket名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FName SocketName;

    // 碰撞模式
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    ECharacterCustomizationCollisionMode CollisionMode = ECharacterCustomizationCollisionMode::NoCollision;

    // 比较两个Part是否等价（忽略碰撞模式）
    static bool AreEquivalentParts(const FLyraCharacterPart& A, const FLyraCharacterPart& B)
    {
        return (A.PartClass == B.PartClass) && (A.SocketName == B.SocketName);
    }
};
```

**碰撞模式**：

```cpp
UENUM()
enum class ECharacterCustomizationCollisionMode : uint8
{
    NoCollision,                  // 禁用碰撞（默认）
    UseCollisionFromCharacterPart // 使用部件自身碰撞设置
};
```

### 10.2 ULyraPawnComponent_CharacterParts — Pawn侧外观管理

**文件**: `Source/LyraGame/Cosmetics/LyraPawnComponent_CharacterParts.h/.cpp`

挂在Pawn上，管理实际的外观Actor生成和销毁。使用**FastArraySerializer**实现高效网络复制。

#### 复制列表结构

```cpp
// 单个应用的部件条目
USTRUCT()
struct FLyraAppliedCharacterPartEntry : public FFastArraySerializerItem
{
    UPROPERTY()
    FLyraCharacterPart Part;           // 部件描述

    UPROPERTY(NotReplicated)
    int32 PartHandle = INDEX_NONE;      // 服务器端Handle

    UPROPERTY(NotReplicated)
    TObjectPtr<UChildActorComponent> SpawnedComponent = nullptr; // 客户端实例
};

// 复制的部件列表
USTRUCT(BlueprintType)
struct FLyraCharacterPartList : public FFastArraySerializer
{
    UPROPERTY()
    TArray<FLyraAppliedCharacterPartEntry> Entries;

    TObjectPtr<ULyraPawnComponent_CharacterParts> OwnerComponent;
    int32 PartHandleCounter = 0;
};
```

#### 网络复制回调

```cpp
void FLyraCharacterPartList::PostReplicatedAdd(const TArrayView<int32> AddedIndices, int32 FinalSize)
{
    bool bCreatedAnyActors = false;
    for (int32 Index : AddedIndices)
    {
        FLyraAppliedCharacterPartEntry& Entry = Entries[Index];
        bCreatedAnyActors |= SpawnActorForEntry(Entry);
    }
    if (bCreatedAnyActors && ensure(OwnerComponent))
    {
        OwnerComponent->BroadcastChanged();
    }
}

void FLyraCharacterPartList::PreReplicatedRemove(const TArrayView<int32> RemovedIndices, int32 FinalSize)
{
    bool bDestroyedAnyActors = false;
    for (int32 Index : RemovedIndices)
    {
        FLyraAppliedCharacterPartEntry& Entry = Entries[Index];
        bDestroyedAnyActors |= DestroyActorForEntry(Entry);
    }
    if (bDestroyedAnyActors && ensure(OwnerComponent))
    {
        OwnerComponent->BroadcastChanged();
    }
}
```

#### Actor生成

```cpp
bool FLyraCharacterPartList::SpawnActorForEntry(FLyraAppliedCharacterPartEntry& Entry)
{
    bool bCreatedAnyActors = false;

    // 非DedicatedServer才生成视觉Actor
    if (ensure(OwnerComponent) && !OwnerComponent->IsNetMode(NM_DedicatedServer))
    {
        if (Entry.Part.PartClass != nullptr)
        {
            if (USceneComponent* ComponentToAttachTo = OwnerComponent->GetSceneComponentToAttachTo())
            {
                // 使用ChildActorComponent方式生成
                UChildActorComponent* PartComponent = NewObject<UChildActorComponent>(
                    OwnerComponent->GetOwner());
                PartComponent->SetupAttachment(ComponentToAttachTo, Entry.Part.SocketName);
                PartComponent->SetChildActorClass(Entry.Part.PartClass);
                PartComponent->RegisterComponent();

                if (AActor* SpawnedActor = PartComponent->GetChildActor())
                {
                    // 碰撞模式处理
                    if (Entry.Part.CollisionMode == ECharacterCustomizationCollisionMode::NoCollision)
                    {
                        SpawnedActor->SetActorEnableCollision(false);
                    }

                    // 设置Tick依赖
                    if (USceneComponent* SpawnedRootComponent = SpawnedActor->GetRootComponent())
                    {
                        SpawnedRootComponent->AddTickPrerequisiteComponent(ComponentToAttachTo);
                    }
                }

                Entry.SpawnedComponent = PartComponent;
                bCreatedAnyActors = true;
            }
        }
    }
    return bCreatedAnyActors;
}
```

#### Body Style 切换

```cpp
void ULyraPawnComponent_CharacterParts::BroadcastChanged()
{
    const bool bReinitPose = true;

    if (USkeletalMeshComponent* MeshComponent = GetParentMeshComponent())
    {
        // 收集所有部件的Cosmetic Tag
        const FGameplayTagContainer MergedTags = GetCombinedTags(FGameplayTag());

        // 基于Tag选择最佳Body Style网格体
        USkeletalMesh* DesiredMesh = BodyMeshes.SelectBestBodyStyle(MergedTags);
        MeshComponent->SetSkeletalMesh(DesiredMesh, bReinitPose);

        // 应用强制Physics Asset
        if (UPhysicsAsset* PhysicsAsset = BodyMeshes.ForcedPhysicsAsset)
        {
            MeshComponent->SetPhysicsAsset(PhysicsAsset, bReinitPose);
        }
    }

    // 通知观察者（如团队着色系统）
    OnCharacterPartsChanged.Broadcast(this);
}
```

#### GameplayTag收集

```cpp
FGameplayTagContainer FLyraCharacterPartList::CollectCombinedTags() const
{
    FGameplayTagContainer Result;
    for (const FLyraAppliedCharacterPartEntry& Entry : Entries)
    {
        if (Entry.SpawnedComponent != nullptr)
        {
            // 从部件Actor的IGameplayTagAssetInterface获取Tag
            if (IGameplayTagAssetInterface* TagInterface = 
                Cast<IGameplayTagAssetInterface>(Entry.SpawnedComponent->GetChildActor()))
            {
                TagInterface->GetOwnedGameplayTags(Result);
            }
        }
    }
    return Result;
}
```

### 10.3 ULyraControllerComponent_CharacterParts — Controller侧外观配置

**文件**: `Source/LyraGame/Cosmetics/LyraControllerComponent_CharacterParts.h/.cpp`

挂在Controller上，管理外观配置和Pawn切换时的自动迁移。

```cpp
UCLASS(meta = (BlueprintSpawnableComponent))
class ULyraControllerComponent_CharacterParts : public UControllerComponent
{
    // 外观部件列表
    UPROPERTY(EditAnywhere, Category=Cosmetics)
    TArray<FLyraControllerCharacterPartEntry> CharacterParts;
};
```

#### Pawn切换自动迁移

```cpp
void ULyraControllerComponent_CharacterParts::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority())
    {
        if (AController* OwningController = GetController<AController>())
        {
            // 监听Pawn变化
            OwningController->OnPossessedPawnChanged.AddDynamic(
                this, &ThisClass::OnPossessedPawnChanged);

            if (APawn* ControlledPawn = GetPawn<APawn>())
            {
                OnPossessedPawnChanged(nullptr, ControlledPawn);
            }
        }
        ApplyDeveloperSettings();
    }
}

void ULyraControllerComponent_CharacterParts::OnPossessedPawnChanged(APawn* OldPawn, APawn* NewPawn)
{
    // 从旧Pawn移除所有外观部件
    if (ULyraPawnComponent_CharacterParts* OldCustomizer = 
        OldPawn ? OldPawn->FindComponentByClass<ULyraPawnComponent_CharacterParts>() : nullptr)
    {
        for (FLyraControllerCharacterPartEntry& Entry : CharacterParts)
        {
            OldCustomizer->RemoveCharacterPart(Entry.Handle);
            Entry.Handle.Reset();
        }
    }

    // 添加到新Pawn
    if (ULyraPawnComponent_CharacterParts* NewCustomizer = 
        NewPawn ? NewPawn->FindComponentByClass<ULyraPawnComponent_CharacterParts>() : nullptr)
    {
        for (FLyraControllerCharacterPartEntry& Entry : CharacterParts)
        {
            if (!Entry.Handle.IsValid() && Entry.Source != ECharacterPartSource::NaturalSuppressedViaCheat)
            {
                Entry.Handle = NewCustomizer->AddCharacterPart(Entry.Part);
            }
        }
    }
}
```

#### 来源追踪

```cpp
enum class ECharacterPartSource : uint8
{
    Natural,                          // 正常来源
    NaturalSuppressedViaCheat,        // 被作弊命令抑制
    AppliedViaDeveloperSettingsCheat,  // 开发者设置作弊
    AppliedViaCheatManager             // 控制台作弊命令
};
```

### 10.4 外观动画类型选择

**文件**: `Source/LyraGame/Cosmetics/LyraCosmeticAnimationTypes.h/.cpp`

#### 动画层选择

```cpp
USTRUCT(BlueprintType)
struct FLyraAnimLayerSelectionSet
{
    // 有序规则列表，第一个匹配的生效
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<FLyraAnimLayerSelectionEntry> LayerRules;

    // 无匹配时的默认层
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSubclassOf<UAnimInstance> DefaultLayer;

    TSubclassOf<UAnimInstance> SelectBestLayer(const FGameplayTagContainer& CosmeticTags) const
    {
        for (const FLyraAnimLayerSelectionEntry& Rule : LayerRules)
        {
            // 要求部件Tag包含规则的所有RequiredTags
            if ((Rule.Layer != nullptr) && CosmeticTags.HasAll(Rule.RequiredTags))
            {
                return Rule.Layer;
            }
        }
        return DefaultLayer;
    }
};
```

#### Body Style选择

```cpp
USTRUCT(BlueprintType)
struct FLyraAnimBodyStyleSelectionSet
{
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<FLyraAnimBodyStyleSelectionEntry> MeshRules;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TObjectPtr<USkeletalMesh> DefaultMesh = nullptr;

    UPROPERTY(EditAnywhere)
    TObjectPtr<UPhysicsAsset> ForcedPhysicsAsset = nullptr;

    USkeletalMesh* SelectBestBodyStyle(const FGameplayTagContainer& CosmeticTags) const
    {
        for (const FLyraAnimBodyStyleSelectionEntry& Rule : MeshRules)
        {
            if ((Rule.Mesh != nullptr) && CosmeticTags.HasAll(Rule.RequiredTags))
            {
                return Rule.Mesh;
            }
        }
        return DefaultMesh;
    }
};
```

### 10.5 开发者作弊系统

```cpp
// ULyraCosmeticDeveloperSettings — 编辑器配置
UCLASS(config=EditorPerProjectUserSettings)
class ULyraCosmeticDeveloperSettings : public UDeveloperSettingsBackedByCVars
{
    // 作弊用的外观部件列表
    UPROPERTY(Transient, EditAnywhere)
    TArray<FLyraCharacterPart> CheatCosmeticCharacterParts;

    // 作弊模式：替换/追加
    UPROPERTY(Transient, EditAnywhere)
    ECosmeticCheatMode CheatMode; // ReplaceParts 或 AddParts
};

// ULyraCosmeticCheats — 控制台命令扩展
UCLASS(NotBlueprintable)
class ULyraCosmeticCheats final : public UCheatManagerExtension
{
    UFUNCTION(Exec, BlueprintAuthorityOnly)
    void AddCharacterPart(const FString& AssetName, bool bSuppressNaturalParts = true);

    UFUNCTION(Exec, BlueprintAuthorityOnly)
    void ReplaceCharacterPart(const FString& AssetName, bool bSuppressNaturalParts = true);

    UFUNCTION(Exec, BlueprintAuthorityOnly)
    void ClearCharacterPartOverrides();
};
```

---

## 11. 初始化状态机详解

### 11.1 4阶段状态链

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  [无状态] ──→ Spawned ──→ DataAvailable ──→ DataInitialized ──→ GameplayReady  │
│                                                                         │
│  触发条件:                                                               │
│  BeginPlay     PawnData设置    所有Feature达到      无额外条件            │
│  Pawn有效      Controller就绪  DataAvailable        直接通过              │
│                               (聚合等待)                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### 11.2 多Feature协调

PawnExtensionComponent作为**协调者**，监听所有Feature的状态变化：

```
┌────────────────────────────────────────────────────────────────────┐
│ PawnExtensionComponent (Feature: "PawnExtension")                  │
│                                                                    │
│ BeginPlay:                                                         │
│   BindOnActorInitStateChanged(NAME_None, ...) // 监听所有Feature  │
│   TryToChangeInitState(Spawned)                                    │
│   CheckDefaultInitialization()                                     │
│                                                                    │
│ OnActorInitStateChanged:                                           │
│   当其他Feature达到DataAvailable → CheckDefaultInitialization()    │
│                                                                    │
│ CanChangeInitState(DataAvailable → DataInitialized):               │
│   return Manager->HaveAllFeaturesReachedInitState(                 │
│       Pawn, DataAvailable); // 等待所有Feature                     │
│                                                                    │
│ CheckDefaultInitialization:                                        │
│   CheckDefaultInitializationForImplementers() // 先推进依赖        │
│   ContinueInitStateChain(StateChain)          // 再推进自身        │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ HeroComponent (Feature: "Hero")                                    │
│                                                                    │
│ BeginPlay:                                                         │
│   BindOnActorInitStateChanged("PawnExtension", ...) // 只监听Ext  │
│   TryToChangeInitState(Spawned)                                    │
│   CheckDefaultInitialization()                                     │
│                                                                    │
│ OnActorInitStateChanged:                                           │
│   当PawnExtension达到DataInitialized → CheckDefaultInitialization()│
│                                                                    │
│ CanChangeInitState(DataAvailable → DataInitialized):               │
│   return PlayerState存在 &&                                        │
│       PawnExtension已达到DataInitialized                           │
│                                                                    │
│ HandleChangeInitState(DataAvailable → DataInitialized):            │
│   → InitializeAbilitySystem (ASC ↔ Pawn桥接)                     │
│   → InitializePlayerInput (输入绑定)                               │
│   → DetermineCameraMode绑定                                       │
└────────────────────────────────────────────────────────────────────┘
```

### 11.3 时序图

```
Server:                                Client:
────────                               ────────
GameMode::HandleStartingNewPlayer()
  → SpawnDefaultPawnAtTransform()
    → ALyraCharacter 构造
      ├── PawnExtComp 创建
      ├── HealthComp 创建
      └── CameraComp 创建

  → RestartPlayer()                    OnRep_PlayerState()
    → PossessedBy()                      → PawnExtComp->HandlePlayerStateReplicated()
      → PawnExtComp->HandleControllerChanged()    → CheckDefaultInitialization()
      → MyTeamID 更新
                                       OnRep_Controller()
  → SetPawnData(PawnData)                → PawnExtComp->HandleControllerChanged()
    → PawnData 复制到Client                → CheckDefaultInitialization()

  → PawnExtComp::BeginPlay()           OnRep_PawnData()
    → Spawned                            → CheckDefaultInitialization()
    → CheckDefaultInitialization()
      → DataAvailable (PawnData+Controller就绪)
                                       PawnExtComp::BeginPlay()
  → HeroComp::BeginPlay()               → Spawned → DataAvailable
    → Spawned
    → CheckDefaultInitialization()     HeroComp::BeginPlay()
      → DataAvailable                    → Spawned → DataAvailable
        (等待PawnExtension)

  → PawnExtComp: 所有Feature DataAvailable
    → DataInitialized

  → HeroComp: PawnExtension DataInitialized
    → DataInitialized
      → InitializeAbilitySystem()
      → InitializePlayerInput()

  → PawnExtComp → GameplayReady
  → HeroComp → GameplayReady
```

---

## 12. 网络复制机制

### 12.1 压缩加速度复制

`ALyraCharacter`使用自定义压缩格式复制加速度，大幅减少带宽：

```cpp
USTRUCT()
struct FLyraReplicatedAcceleration
{
    UPROPERTY()
    uint8 AccelXYRadians = 0;     // XY方向角 [0, 2π] → [0, 255]

    UPROPERTY()
    uint8 AccelXYMagnitude = 0;   // XY大小 [0, MaxAccel] → [0, 255]

    UPROPERTY()
    int8 AccelZ = 0;              // Z分量 [-MaxAccel, MaxAccel] → [-127, 127]
};
// 总计 3 字节 vs 原始FVector的 12 字节

// 编码（服务器PreReplication）
void ALyraCharacter::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
    const double MaxAccel = MovementComponent->MaxAcceleration;
    const FVector CurrentAccel = MovementComponent->GetCurrentAcceleration();
    double AccelXYRadians, AccelXYMagnitude;
    FMath::CartesianToPolar(CurrentAccel.X, CurrentAccel.Y, AccelXYMagnitude, AccelXYRadians);

    ReplicatedAcceleration.AccelXYRadians   = FMath::FloorToInt((AccelXYRadians / TWO_PI) * 255.0);
    ReplicatedAcceleration.AccelXYMagnitude = FMath::FloorToInt((AccelXYMagnitude / MaxAccel) * 255.0);
    ReplicatedAcceleration.AccelZ           = FMath::FloorToInt((CurrentAccel.Z / MaxAccel) * 127.0);
}

// 解码（客户端OnRep）
void ALyraCharacter::OnRep_ReplicatedAcceleration()
{
    const double MaxAccel = LyraMovementComponent->MaxAcceleration;
    const double AccelXYMagnitude = double(ReplicatedAcceleration.AccelXYMagnitude) * MaxAccel / 255.0;
    const double AccelXYRadians   = double(ReplicatedAcceleration.AccelXYRadians) * TWO_PI / 255.0;

    FVector UnpackedAcceleration(FVector::ZeroVector);
    FMath::PolarToCartesian(AccelXYMagnitude, AccelXYRadians, UnpackedAcceleration.X, UnpackedAcceleration.Y);
    UnpackedAcceleration.Z = double(ReplicatedAcceleration.AccelZ) * MaxAccel / 127.0;

    LyraMovementComponent->SetReplicatedAcceleration(UnpackedAcceleration);
}
```

### 12.2 FastShared 移动复制

当标准属性复制被跳过时，使用不可靠多播发送移动更新：

```cpp
USTRUCT()
struct FSharedRepMovement
{
    FRepMovement RepMovement;          // 位置/旋转/速度
    float RepTimeStamp = 0.0f;         // 时间戳
    uint8 RepMovementMode = 0;         // 移动模式
    bool bProxyIsJumpForceApplied;     // 跳跃力
    bool bIsCrouched;                  // 蹲伏状态
};

bool ALyraCharacter::UpdateSharedReplication()
{
    if (GetLocalRole() == ROLE_Authority)
    {
        FSharedRepMovement SharedMovement;
        if (SharedMovement.FillForCharacter(this))
        {
            // 仅在数据变化时发送
            if (!SharedMovement.Equals(LastSharedReplication, this))
            {
                LastSharedReplication = SharedMovement;
                SetReplicatedMovementMode(SharedMovement.RepMovementMode);
                FastSharedReplication(SharedMovement); // NetMulticast, unreliable
            }
            return true;
        }
    }
    return false;
}
```

### 12.3 复制属性总结

| 属性 | 复制条件 | 说明 |
|------|----------|------|
| `ReplicatedAcceleration` | `COND_SimulatedOnly` | 压缩加速度，仅模拟代理 |
| `MyTeamID` | 始终 | 团队ID |
| `PawnData` (PawnExtComp) | 始终 | Pawn配置数据 |
| `DeathState` (HealthComp) | 始终 | 死亡状态 |
| `CharacterPartList` (CosmeticsComp) | 始终 | 外观部件列表 (FastArray) |

---

## 13. 完整架构图

```
┌─────────────── ALyraCharacter (AModularCharacter) ─────────────────────────────┐
│                                                                                 │
│  接口: IAbilitySystemInterface, IGameplayCueInterface,                         │
│        IGameplayTagAssetInterface, ILyraTeamAgentInterface                     │
│                                                                                 │
│  ┌── ULyraPawnExtensionComponent (初始化协调器) ────────────────────────────┐  │
│  │  Feature: "PawnExtension"                                                │  │
│  │  ├── PawnData: ULyraPawnData (Replicated)                               │  │
│  │  │     ├── PawnClass                                                     │  │
│  │  │     ├── AbilitySets[]                                                 │  │
│  │  │     ├── TagRelationshipMapping                                        │  │
│  │  │     ├── InputConfig                                                   │  │
│  │  │     └── DefaultCameraMode                                             │  │
│  │  ├── AbilitySystemComponent (缓存指针, 指向PlayerState的ASC)            │  │
│  │  ├── InitState状态机: Spawned→DataAvailable→DataInitialized→GameplayReady│  │
│  │  ├── 监听所有Feature状态变化, 聚合等待                                   │  │
│  │  ├── InitializeAbilitySystem(ASC, OwnerActor)                           │  │
│  │  │     → ASC.InitAbilityActorInfo(PlayerState, Pawn)                     │  │
│  │  │     → ASC.SetTagRelationshipMapping(PawnData)                         │  │
│  │  │     → Broadcast OnAbilitySystemInitialized                            │  │
│  │  └── UninitializeAbilitySystem()                                        │  │
│  │        → CancelAbilities(除SurvivesDeath外)                             │  │
│  │        → ClearAbilityInput + RemoveAllGameplayCues                      │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ┌── ULyraHeroComponent (玩家特有) ────────────────────────────────────────┐  │
│  │  Feature: "Hero"                                                         │  │
│  │  ├── DefaultInputMappings[]                                              │  │
│  │  ├── AbilityCameraMode / AbilityCameraModeOwningSpecHandle               │  │
│  │  ├── DataInitialized阶段:                                               │  │
│  │  │     → PawnExtComp.InitializeAbilitySystem(PS.ASC, PS)                │  │
│  │  │     → InitializePlayerInput()                                         │  │
│  │  │       ├── 添加DefaultInputMappings到EnhancedInput                    │  │
│  │  │       ├── BindAbilityActions(InputConfig)                            │  │
│  │  │       ├── BindNativeAction(Move/Look/Crouch/AutoRun)                 │  │
│  │  │       └── Send NAME_BindInputsNow 事件                              │  │
│  │  │     → CameraComponent.DetermineCameraModeDelegate.Bind(this)         │  │
│  │  ├── Input_Move: Controller Yaw相对方向移动                             │  │
│  │  ├── Input_LookMouse: 直接Yaw/Pitch                                    │  │
│  │  ├── Input_LookStick: 速率×DeltaTime                                   │  │
│  │  ├── Input_AbilityInputTag: 转发到ASC                                  │  │
│  │  └── DetermineCameraMode: Ability相机 > PawnData默认相机               │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ┌── ULyraHealthComponent ─────────────────────────────────────────────────┐  │
│  │  ├── DeathState (Replicated): NotDead→DeathStarted→DeathFinished        │  │
│  │  ├── InitializeWithAbilitySystem(ASC)                                   │  │
│  │  │     → 获取HealthSet, 监听OnHealthChanged/OnOutOfHealth              │  │
│  │  │     → 初始化血量 = MaxHealth                                         │  │
│  │  ├── HandleOutOfHealth:                                                 │  │
│  │  │     → 发送GameplayEvent.Death到ASC                                  │  │
│  │  │     → 广播Lyra.Elimination.Message                                  │  │
│  │  ├── StartDeath: Tag(Status_Death_Dying) + OnDeathStarted委托          │  │
│  │  ├── FinishDeath: Tag(Status_Death_Dead) + OnDeathFinished委托         │  │
│  │  └── DamageSelfDestruct: 施加MaxHealth伤害的GE                         │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ┌── ULyraCharacterMovementComponent ──────────────────────────────────────┐  │
│  │  ├── GetMaxSpeed: Gameplay.MovementStopped Tag → 0                      │  │
│  │  ├── GetDeltaRotation: Gameplay.MovementStopped Tag → 无旋转            │  │
│  │  ├── GetGroundInfo: 帧缓存的地面射线检测                                │  │
│  │  ├── SimulateMovement: 保留复制的加速度                                 │  │
│  │  └── CanAttemptJump: 允许蹲伏时跳跃                                    │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ┌── ULyraCameraComponent (详见相机系统文档) ──────────────────────────────┐  │
│  │  └── DetermineCameraModeDelegate → HeroComponent                        │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  角色生命周期:                                                                 │
│  ├── PossessedBy: PawnExtComp.HandleControllerChanged() + TeamID同步        │
│  ├── UnPossessed: PawnExtComp.HandleControllerChanged() + TeamID重置        │
│  ├── OnRep_Controller: PawnExtComp.HandleControllerChanged()                │
│  ├── OnRep_PlayerState: PawnExtComp.HandlePlayerStateReplicated()           │
│  ├── OnAbilitySystemInitialized: HealthComp.InitializeWithAbilitySystem()   │
│  ├── OnMovementModeChanged: 更新MovementMode GameplayTag                   │
│  ├── OnDeathStarted: DisableMovementAndCollision()                          │
│  ├── OnDeathFinished → DestroyDueToDeath → UninitAndDestroy                │
│  └── FellOutOfWorld: HealthComp.DamageSelfDestruct(true)                   │
│                                                                                 │
│  网络复制:                                                                     │
│  ├── ReplicatedAcceleration (3字节压缩, COND_SimulatedOnly)                │
│  ├── MyTeamID (始终复制)                                                    │
│  └── FastSharedReplication (不可靠多播, 跳帧时的移动更新)                   │
│                                                                                 │
├─────────────── 外观系统 (Cosmetics) ───────────────────────────────────────────┤
│                                                                                 │
│  Controller 侧:                           Pawn 侧:                            │
│  ULyraControllerComponent_CharacterParts   ULyraPawnComponent_CharacterParts   │
│    ├── CharacterParts[] (配置)               ├── CharacterPartList (FastArray) │
│    ├── OnPossessedPawnChanged               │     ├── Entries[]                │
│    │     → 旧Pawn移除 + 新Pawn添加         │     ├── SpawnActorForEntry()     │
│    ├── AddCharacterPart(Part)               │     └── DestroyActorForEntry()   │
│    │     → PawnCustomizer.AddCharacterPart()├── BodyMeshes (Tag→Mesh规则)     │
│    └── ApplyDeveloperSettings()             └── BroadcastChanged()             │
│          (PIE中的Cosmetic作弊)                   → SelectBestBodyStyle()       │
│                                                  → OnCharacterPartsChanged     │
│                                                                                 │
│  选择规则: FLyraAnimBodyStyleSelectionSet                                      │
│    MeshRules[]: RequiredTags → SkeletalMesh (第一个匹配生效)                  │
│    DefaultMesh: 无匹配时的默认网格                                             │
│                                                                                 │
│  选择规则: FLyraAnimLayerSelectionSet                                          │
│    LayerRules[]: RequiredTags → AnimInstance Layer                             │
│    DefaultLayer: 无匹配时的默认动画层                                          │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 14. 核心设计模式总结

### 14.1 ASC外置模式 (PlayerState-Owned ASC)

**核心思想**：ASC放在PlayerState上，Character作为Avatar。

```
ALyraPlayerState
  └── ULyraAbilitySystemComponent (Owner)
        ├── OwnerActor = PlayerState
        └── AvatarActor = ALyraCharacter (可变)

ALyraCharacter 死亡 → 新 ALyraCharacter 生成
  → PawnExtComp.InitializeAbilitySystem(PS.ASC, PS)
    → ASC.InitAbilityActorInfo(PS, NewCharacter)
    → 能力/属性/Buff 全部保留
```

**优势**：
- 死亡重生不丢失能力和状态
- 角色切换（如载具）时保持一致的能力系统
- 观战者可以查看被观战者的能力状态

**例外**：`ALyraCharacterWithAbilities`将ASC放在Character自身上，用于AI/非玩家角色。

### 14.2 InitState 状态机协调模式

**核心思想**：多组件通过统一状态机协调异步初始化。

| 阶段 | 条件 | 执行内容 |
|------|------|----------|
| `Spawned` | Pawn有效 | BeginPlay中标记 |
| `DataAvailable` | PawnData就绪 + Controller就绪 | 数据已到位 |
| `DataInitialized` | **所有**Feature都DataAvailable | ASC初始化、输入绑定、相机绑定 |
| `GameplayReady` | 无额外条件 | 通知外部系统（GameFeature等） |

**优势**：
- 解决网络异步（PawnData复制、Controller复制的时序不确定）
- 解决组件间依赖（HeroComponent等待PawnExtension先完成）
- 支持GameFeature动态添加的组件也参与状态协调

### 14.3 组件化拆分模式

**核心思想**：Character类本身极薄，功能分散到各组件。

```
ALyraCharacter (薄壳层)
  ├── 转发生命周期事件到PawnExtComp
  ├── 代理GAS接口到PawnExtComp.ASC
  ├── 监听死亡委托
  └── 管理团队ID复制

功能全部在组件中:
  PawnExtComp → 初始化协调 + ASC桥接
  HeroComp    → 输入 + 相机
  HealthComp  → 生命 + 死亡
  MoveComp    → 移动扩展
  CosmeticsComp → 外观
```

**优势**：
- 各组件可独立复用（HeroComponent可放在任何Pawn上）
- GameFeature可通过`GameFrameworkComponentManager`动态添加组件
- 单元测试更容易

### 14.4 Controller-Pawn 外观双组件模式

**核心思想**：外观配置在Controller侧持久化，实例化在Pawn侧执行。

```
Controller (持久)                  Pawn (临时)
ULyraControllerComponent_CharacterParts    ULyraPawnComponent_CharacterParts
  ├── CharacterParts[]                       ├── CharacterPartList (FastArray)
  │   (配置+来源追踪)                         │   (网络复制+Actor生成)
  │                                           │
  └── OnPossessedPawnChanged ────────────────→│ AddCharacterPart / RemoveCharacterPart
        旧Pawn移除, 新Pawn添加
```

**优势**：
- 外观配置跨Pawn保持
- 支持开发者作弊覆盖（来源追踪）
- Pawn侧使用FastArray高效复制

### 14.5 Tag驱动行为模式

整个角色系统大量使用GameplayTag驱动行为：

| Tag | 功能 |
|-----|------|
| `Movement.Mode.Walking/Falling/...` | 移动模式同步到GAS |
| `Status_Crouching` | 蹲伏状态Tag |
| `Status_Death_Dying` | 死亡开始Tag |
| `Status_Death_Dead` | 死亡完成Tag |
| `Gameplay.MovementStopped` | 停止移动（速度归0，禁止旋转） |
| `Ability_Behavior_SurvivesDeath` | 标记此能力在角色死亡时不取消 |
| `GameplayEvent_Death` | 死亡GAS事件 |
| `SetByCaller_Damage` | 自毁伤害值 |

### 14.6 压缩复制优化

- **加速度**: 3字节极坐标压缩（vs FVector 12字节，节省75%）
- **移动**: FastShared不可靠多播，仅在数据变化时发送
- **外观**: FastArraySerializer增量复制
- **死亡状态**: 枚举复制 + OnRep确保状态机顺序正确

---

## 15. 文件索引

| 目录/文件 | 核心职责 |
|-----------|----------|
| **角色核心** | |
| `Source/LyraGame/Character/LyraCharacter.h/.cpp` | 角色基类（组件创建、事件转发、团队、网络） |
| `Source/LyraGame/Character/LyraPawnExtensionComponent.h/.cpp` | 初始化协调器（PawnData、ASC桥接、InitState） |
| `Source/LyraGame/Character/LyraHeroComponent.h/.cpp` | 英雄组件（输入绑定、相机决策、能力输入转发） |
| `Source/LyraGame/Character/LyraHealthComponent.h/.cpp` | 生命组件（属性监听、死亡状态机、消息广播） |
| `Source/LyraGame/Character/LyraCharacterMovementComponent.h/.cpp` | 移动组件（GAS Tag集成、地面信息、加速度复制） |
| `Source/LyraGame/Character/LyraPawnData.h/.cpp` | Pawn数据资产（能力、输入、相机配置） |
| `Source/LyraGame/Character/LyraCharacterWithAbilities.h/.cpp` | 自含ASC角色变体（AI/NPC用） |
| `Source/LyraGame/Character/LyraPawn.h/.cpp` | 通用Pawn基类（非Character场景） |
| **外观系统** | |
| `Source/LyraGame/Cosmetics/LyraCharacterPartTypes.h` | 外观部件数据类型定义 |
| `Source/LyraGame/Cosmetics/LyraPawnComponent_CharacterParts.h/.cpp` | Pawn侧外观管理（FastArray复制、Actor生成） |
| `Source/LyraGame/Cosmetics/LyraControllerComponent_CharacterParts.h/.cpp` | Controller侧外观配置（Pawn迁移、来源追踪） |
| `Source/LyraGame/Cosmetics/LyraCosmeticAnimationTypes.h/.cpp` | 外观动画/Body Style选择规则 |
| `Source/LyraGame/Cosmetics/LyraCosmeticDeveloperSettings.h/.cpp` | 开发者外观作弊设置 |
| `Source/LyraGame/Cosmetics/LyraCosmeticCheats.h/.cpp` | 控制台外观作弊命令 |
| **基础设施** | |
| `Plugins/ModularGameplayActors/Source/.../ModularCharacter.h/.cpp` | Modular Character基类 |
| `Plugins/ModularGameplayActors/Source/.../ModularPawn.h/.cpp` | Modular Pawn基类 |
