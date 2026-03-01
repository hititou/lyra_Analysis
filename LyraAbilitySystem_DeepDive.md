# Lyra Gameplay Ability System (GAS) 深度代码分析

---

## 一、GAS 原理概述

### 1.1 什么是 Gameplay Ability System

Gameplay Ability System（GAS）是 Unreal Engine 内置的一套高度灵活的框架，用于管理角色的**技能（Ability）**、**属性（Attribute）**、**效果（GameplayEffect）**和**标签（GameplayTag）**。其核心组件包括：

| 组件 | 职责 |
|------|------|
| **AbilitySystemComponent (ASC)** | 技能系统的中枢，管理所有 Ability、Effect、Attribute |
| **GameplayAbility (GA)** | 定义一个具体的技能/行为 |
| **GameplayEffect (GE)** | 定义对属性的修改规则（伤害、治疗、Buff等） |
| **AttributeSet** | 定义可被 GE 修改的数值属性（生命值、攻击力等） |
| **GameplayTag** | 标签系统，用于标识状态、能力类型、事件等 |
| **GameplayCue** | 视觉/音频反馈系统 |
| **AbilityTask** | 异步任务节点，用于技能执行中的等待逻辑 |

### 1.2 GAS 数据流

```
[Input] → [ASC.TryActivateAbility] → [GA.CanActivateAbility] → [GA.ActivateAbility]
                                                                       ↓
                                                              [Apply GameplayEffect]
                                                                       ↓
                                                              [Modify AttributeSet]
                                                                       ↓
                                                              [Trigger GameplayCue]
```

---

## 二、Lyra GAS 整体架构

### 2.1 核心文件一览

```
Source/LyraGame/AbilitySystem/
├── LyraAbilitySystemComponent.h/cpp          // 自定义 ASC
├── LyraAbilitySet.h/cpp                       // 数据资产：批量授予 Ability/Effect/Attribute
├── LyraAbilitySystemGlobals.h/cpp             // 自定义 AllocGameplayEffectContext
├── LyraAbilitySourceInterface.h/cpp           // 伤害来源接口（距离/材质衰减）
├── LyraAbilityTagRelationshipMapping.h/cpp    // Tag 互斥关系映射表
├── LyraGameplayEffectContext.h/cpp            // 自定义 EffectContext（CartridgeID）
├── LyraGameplayCueManager.h/cpp               // 自定义 CueManager（延迟加载优化）
├── LyraGlobalAbilitySystem.h/cpp              // 全局 Ability/Effect 系统
├── LyraGameplayAbilityTargetData_SingleTargetHit.h/cpp
├── Abilities/
│   ├── LyraGameplayAbility.h/cpp              // 基类 Ability
│   ├── LyraGameplayAbility_Death.h/cpp        // 死亡技能
│   ├── LyraGameplayAbility_Jump.h/cpp         // 跳跃技能
│   ├── LyraGameplayAbility_Reset.h/cpp        // 重置技能
│   ├── LyraAbilityCost.h                       // 自定义消耗基类
│   ├── LyraAbilityCost_InventoryItem.h/cpp
│   ├── LyraAbilityCost_ItemTagStack.h/cpp
│   └── LyraAbilityCost_PlayerTagStack.h/cpp
├── Attributes/
│   ├── LyraAttributeSet.h/cpp                 // 属性集基类
│   ├── LyraHealthSet.h/cpp                    // 生命属性集
│   └── LyraCombatSet.h/cpp                    // 战斗属性集
├── Executions/
│   └── LyraDamageExecution.h/cpp              // 伤害计算执行
└── Phases/
    └── LyraGamePhaseAbility.h/cpp             // 游戏阶段技能
```

### 2.2 继承关系

```
UAbilitySystemComponent
└── ULyraAbilitySystemComponent                // 输入绑定、ActivationGroup、TagRelationship

UGameplayAbility
└── ULyraGameplayAbility                       // 基类：ActivationPolicy/Group、AdditionalCosts、CameraMode
    ├── ULyraGameplayAbility_Death             // 死亡处理（GameplayEvent 触发）
    ├── ULyraGameplayAbility_Jump              // 跳跃（输入触发）
    ├── ULyraGameplayAbility_Reset             // 重置（GameplayEvent 触发）
    ├── ULyraGamePhaseAbility                  // 游戏阶段管理
    └── ULyraGameplayAbility_FromEquipment     // 装备关联技能
        └── ULyraGameplayAbility_RangedWeapon  // 远程武器射击

UAttributeSet
└── ULyraAttributeSet                          // 属性集基类
    ├── ULyraHealthSet                         // Health, MaxHealth, Damage, Healing
    └── ULyraCombatSet                         // BaseDamage, BaseHeal

UAbilitySystemGlobals
└── ULyraAbilitySystemGlobals                  // 分配自定义 EffectContext

UGameplayCueManager
└── ULyraGameplayCueManager                    // 延迟加载 GameplayCue 优化

UWorldSubsystem
└── ULyraGlobalAbilitySystem                   // 全局 Ability/Effect 广播
```

---

## 三、ULyraAbilitySystemComponent 详解

### 3.1 类声明

```cpp
// LyraAbilitySystemComponent.h
UCLASS(MinimalAPI)
class ULyraAbilitySystemComponent : public UAbilitySystemComponent
{
    // Tag 关系映射表
    UPROPERTY()
    TObjectPtr<ULyraAbilityTagRelationshipMapping> TagRelationshipMapping;

    // 输入缓冲队列
    TArray<FGameplayAbilitySpecHandle> InputPressedSpecHandles;
    TArray<FGameplayAbilitySpecHandle> InputReleasedSpecHandles;
    TArray<FGameplayAbilitySpecHandle> InputHeldSpecHandles;

    // 激活组计数器
    int32 ActivationGroupCounts[(uint8)ELyraAbilityActivationGroup::MAX];
};
```

### 3.2 初始化流程（InitAbilityActorInfo）

当 Pawn 作为 Avatar 被分配时，ASC 执行以下初始化：

```cpp
void ULyraAbilitySystemComponent::InitAbilityActorInfo(AActor* InOwnerActor, AActor* InAvatarActor)
{
    const bool bHasNewPawnAvatar = Cast<APawn>(InAvatarActor) && (InAvatarActor != ActorInfo->AvatarActor);

    Super::InitAbilityActorInfo(InOwnerActor, InAvatarActor);

    if (bHasNewPawnAvatar)
    {
        // 1. 通知所有已授予的 Ability：新 Pawn 已分配
        for (const FGameplayAbilitySpec& AbilitySpec : ActivatableAbilities.Items)
        {
            ULyraGameplayAbility* LyraAbilityInstance = ...;
            LyraAbilityInstance->OnPawnAvatarSet();  // 触发蓝图事件 K2_OnPawnAvatarSet
        }

        // 2. 注册到全局 AbilitySystem（接收全局 Ability/Effect）
        if (ULyraGlobalAbilitySystem* GlobalAbilitySystem = UWorld::GetSubsystem<ULyraGlobalAbilitySystem>(GetWorld()))
        {
            GlobalAbilitySystem->RegisterASC(this);
        }

        // 3. 初始化 AnimInstance 与 ASC 的绑定
        if (ULyraAnimInstance* LyraAnimInst = Cast<ULyraAnimInstance>(ActorInfo->GetAnimInstance()))
        {
            LyraAnimInst->InitializeWithAbilitySystem(this);
        }

        // 4. 尝试激活 OnSpawn 策略的技能
        TryActivateAbilitiesOnSpawn();
    }
}
```

### 3.3 基于 InputTag 的输入系统

Lyra **不使用** UE 传统的 `InputID` 绑定，而是通过 **GameplayTag** 实现输入与 Ability 的映射。

#### 输入收集阶段

```cpp
void ULyraAbilitySystemComponent::AbilityInputTagPressed(const FGameplayTag& InputTag)
{
    if (InputTag.IsValid())
    {
        for (const FGameplayAbilitySpec& AbilitySpec : ActivatableAbilities.Items)
        {
            // 通过 DynamicSpecSourceTags 匹配 InputTag
            if (AbilitySpec.Ability && AbilitySpec.GetDynamicSpecSourceTags().HasTagExact(InputTag))
            {
                InputPressedSpecHandles.AddUnique(AbilitySpec.Handle);
                InputHeldSpecHandles.AddUnique(AbilitySpec.Handle);
            }
        }
    }
}
```

#### 输入处理阶段（每帧调用）

```cpp
void ULyraAbilitySystemComponent::ProcessAbilityInput(float DeltaTime, bool bGamePaused)
{
    // 如果被 TAG_Gameplay_AbilityInputBlocked 标签阻塞，清空所有输入
    if (HasMatchingGameplayTag(TAG_Gameplay_AbilityInputBlocked))
    {
        ClearAbilityInput();
        return;
    }

    static TArray<FGameplayAbilitySpecHandle> AbilitiesToActivate;
    AbilitiesToActivate.Reset();

    // 处理 WhileInputActive 策略：持续按住时激活
    for (const FGameplayAbilitySpecHandle& SpecHandle : InputHeldSpecHandles)
    {
        if (LyraAbilityCDO->GetActivationPolicy() == ELyraAbilityActivationPolicy::WhileInputActive)
            AbilitiesToActivate.AddUnique(AbilitySpec->Handle);
    }

    // 处理 OnInputTriggered 策略：按下时激活
    for (const FGameplayAbilitySpecHandle& SpecHandle : InputPressedSpecHandles)
    {
        if (AbilitySpec->IsActive())
            AbilitySpecInputPressed(*AbilitySpec);  // 传递输入事件给已激活技能
        else if (LyraAbilityCDO->GetActivationPolicy() == ELyraAbilityActivationPolicy::OnInputTriggered)
            AbilitiesToActivate.AddUnique(AbilitySpec->Handle);
    }

    // 批量激活技能
    for (const FGameplayAbilitySpecHandle& Handle : AbilitiesToActivate)
        TryActivateAbility(Handle);

    // 处理输入释放
    for (const FGameplayAbilitySpecHandle& SpecHandle : InputReleasedSpecHandles)
    {
        AbilitySpec->InputPressed = false;
        if (AbilitySpec->IsActive())
            AbilitySpecInputReleased(*AbilitySpec);
    }

    // 清空本帧缓冲
    InputPressedSpecHandles.Reset();
    InputReleasedSpecHandles.Reset();
}
```

**关键设计**：Lyra 不使用 `bReplicateInputDirectly`，而是通过 `InvokeReplicatedEvent` 使 `WaitInputPress`/`WaitInputRelease` AbilityTask 正常工作：

```cpp
void ULyraAbilitySystemComponent::AbilitySpecInputPressed(FGameplayAbilitySpec& Spec)
{
    Super::AbilitySpecInputPressed(Spec);
    if (Spec.IsActive())
    {
        InvokeReplicatedEvent(EAbilityGenericReplicatedEvent::InputPressed, Spec.Handle, OriginalPredictionKey);
    }
}
```

### 3.4 ActivationGroup 系统

Lyra 定义了三种激活组来管理技能之间的互斥关系：

```cpp
UENUM(BlueprintType)
enum class ELyraAbilityActivationGroup : uint8
{
    Independent,           // 独立运行，不影响其他技能
    Exclusive_Replaceable, // 互斥但可被替换（新技能激活时取消旧的）
    Exclusive_Blocking,    // 互斥且阻塞（激活时阻止其他互斥技能）
    MAX
};
```

激活/结束时的管理逻辑：

```cpp
void ULyraAbilitySystemComponent::AddAbilityToActivationGroup(ELyraAbilityActivationGroup Group, ULyraGameplayAbility* LyraAbility)
{
    ActivationGroupCounts[(uint8)Group]++;

    switch (Group)
    {
    case ELyraAbilityActivationGroup::Independent:
        break;  // 不取消任何技能

    case ELyraAbilityActivationGroup::Exclusive_Replaceable:
    case ELyraAbilityActivationGroup::Exclusive_Blocking:
        // 取消所有 Replaceable 组的技能
        CancelActivationGroupAbilities(ELyraAbilityActivationGroup::Exclusive_Replaceable, LyraAbility, false);
        break;
    }

    // 确保同时只有一个 Exclusive 技能在运行
    const int32 ExclusiveCount = ActivationGroupCounts[Exclusive_Replaceable] + ActivationGroupCounts[Exclusive_Blocking];
    ensure(ExclusiveCount <= 1);
}

bool ULyraAbilitySystemComponent::IsActivationGroupBlocked(ELyraAbilityActivationGroup Group) const
{
    switch (Group)
    {
    case Independent:          return false;  // 永不被阻塞
    case Exclusive_Replaceable:
    case Exclusive_Blocking:   return (ActivationGroupCounts[Exclusive_Blocking] > 0);
    }
}
```

### 3.5 Tag Relationship 扩展

Lyra 通过数据资产 `ULyraAbilityTagRelationshipMapping` 扩展了 GAS 原生的 Block/Cancel 机制：

```cpp
void ULyraAbilitySystemComponent::ApplyAbilityBlockAndCancelTags(
    const FGameplayTagContainer& AbilityTags, UGameplayAbility* RequestingAbility,
    bool bEnableBlockTags, const FGameplayTagContainer& BlockTags,
    bool bExecuteCancelTags, const FGameplayTagContainer& CancelTags)
{
    FGameplayTagContainer ModifiedBlockTags = BlockTags;
    FGameplayTagContainer ModifiedCancelTags = CancelTags;

    if (TagRelationshipMapping)
    {
        // 使用映射表扩展 block/cancel 标签
        TagRelationshipMapping->GetAbilityTagsToBlockAndCancel(AbilityTags, &ModifiedBlockTags, &ModifiedCancelTags);
    }

    Super::ApplyAbilityBlockAndCancelTags(AbilityTags, RequestingAbility, bEnableBlockTags, ModifiedBlockTags, bExecuteCancelTags, ModifiedCancelTags);
}
```

映射表数据结构：

```cpp
USTRUCT()
struct FLyraAbilityTagRelationship
{
    FGameplayTag AbilityTag;                  // 触发者标签
    FGameplayTagContainer AbilityTagsToBlock; // 被阻塞的技能标签
    FGameplayTagContainer AbilityTagsToCancel;// 被取消的技能标签
    FGameplayTagContainer ActivationRequiredTags;  // 额外激活需求
    FGameplayTagContainer ActivationBlockedTags;   // 额外激活阻塞
};
```

### 3.6 动态 Tag GameplayEffect

通过 GE 动态添加/移除 GameplayTag：

```cpp
void ULyraAbilitySystemComponent::AddDynamicTagGameplayEffect(const FGameplayTag& Tag)
{
    const TSubclassOf<UGameplayEffect> DynamicTagGE = ULyraAssetManager::GetSubclass(ULyraGameData::Get().DynamicTagGameplayEffect);
    const FGameplayEffectSpecHandle SpecHandle = MakeOutgoingSpec(DynamicTagGE, 1.0f, MakeEffectContext());
    Spec->DynamicGrantedTags.AddTag(Tag);
    ApplyGameplayEffectSpecToSelf(*Spec);
}
```

---

## 四、ULyraGameplayAbility 基类详解

### 4.1 默认策略配置

```cpp
ULyraGameplayAbility::ULyraGameplayAbility(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    ReplicationPolicy = EGameplayAbilityReplicationPolicy::ReplicateNo;
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
    NetSecurityPolicy = EGameplayAbilityNetSecurityPolicy::ClientOrServer;

    ActivationPolicy = ELyraAbilityActivationPolicy::OnInputTriggered;
    ActivationGroup = ELyraAbilityActivationGroup::Independent;
}
```

**关键决策**：
- **InstancedPerActor**：每个 Actor 一个实例（非 NonInstanced，因为已弃用）
- **ReplicateNo**：不复制 Ability 本身，靠事件同步
- **LocalPredicted**：客户端预测执行

### 4.2 ActivationPolicy 枚举

```cpp
UENUM(BlueprintType)
enum class ELyraAbilityActivationPolicy : uint8
{
    OnInputTriggered,   // 输入按下时激活
    WhileInputActive,   // 输入持续时反复激活
    OnSpawn             // Avatar 分配时自动激活
};
```

OnSpawn 策略的实现：

```cpp
void ULyraGameplayAbility::TryActivateAbilityOnSpawn(const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilitySpec& Spec) const
{
    if (ActorInfo && !Spec.IsActive() && (ActivationPolicy == ELyraAbilityActivationPolicy::OnSpawn))
    {
        const bool bIsLocalExecution = (NetExecutionPolicy == LocalPredicted) || (NetExecutionPolicy == LocalOnly);
        const bool bIsServerExecution = (NetExecutionPolicy == ServerOnly) || (NetExecutionPolicy == ServerInitiated);

        const bool bClientShouldActivate = ActorInfo->IsLocallyControlled() && bIsLocalExecution;
        const bool bServerShouldActivate = ActorInfo->IsNetAuthority() && bIsServerExecution;

        if (bClientShouldActivate || bServerShouldActivate)
            ASC->TryActivateAbility(Spec.Handle);
    }
}
```

### 4.3 CanActivateAbility 扩展

```cpp
bool ULyraGameplayAbility::CanActivateAbility(...) const
{
    if (!Super::CanActivateAbility(...))
        return false;

    // 检查 ActivationGroup 是否被阻塞
    ULyraAbilitySystemComponent* LyraASC = CastChecked<ULyraAbilitySystemComponent>(ActorInfo->AbilitySystemComponent.Get());
    if (LyraASC->IsActivationGroupBlocked(ActivationGroup))
    {
        OptionalRelevantTags->AddTag(LyraGameplayTags::Ability_ActivateFail_ActivationGroup);
        return false;
    }
    return true;
}
```

### 4.4 DoesAbilitySatisfyTagRequirements 扩展

Lyra 重写了标签需求检查以支持：
1. **TagRelationship 扩展**：通过 ASC 的 `GetAdditionalActivationTagRequirements` 动态添加 Required/Blocked 标签
2. **死亡状态特殊反馈**：当因为死亡标签被阻塞时，返回专门的 `Ability_ActivateFail_IsDead` 标签

```cpp
bool ULyraGameplayAbility::DoesAbilitySatisfyTagRequirements(...) const
{
    // 从 TagRelationshipMapping 扩展激活条件
    AllRequiredTags = ActivationRequiredTags;
    AllBlockedTags = ActivationBlockedTags;
    if (LyraASC)
        LyraASC->GetAdditionalActivationTagRequirements(GetAssetTags(), AllRequiredTags, AllBlockedTags);

    // 检查死亡状态并提供特殊反馈
    if (AbilitySystemComponentTags.HasTag(LyraGameplayTags::Status_Death))
        OptionalRelevantTags->AddTag(LyraGameplayTags::Ability_ActivateFail_IsDead);

    // ... 标准的 Required/Blocked 检查 ...
}
```

### 4.5 自定义 Cost 系统

Lyra 在 GAS 原生 Cost（通过 CostGameplayEffect）之上，增加了**可插拔的 AdditionalCosts** 机制：

```cpp
// 基类 —— 通过 UObject 子类化实现策略模式
UCLASS(DefaultToInstanced, EditInlineNew, Abstract)
class ULyraAbilityCost : public UObject
{
    virtual bool CheckCost(const ULyraGameplayAbility* Ability, ...) const { return true; }
    virtual void ApplyCost(const ULyraGameplayAbility* Ability, ...) {}
    bool ShouldOnlyApplyCostOnHit() const { return bOnlyApplyCostOnHit; }

protected:
    UPROPERTY(EditAnywhere)
    bool bOnlyApplyCostOnHit = false;  // 仅命中时才消耗
};
```

使用方式：

```cpp
bool ULyraGameplayAbility::CheckCost(...) const
{
    if (!Super::CheckCost(...))
        return false;
    for (const TObjectPtr<ULyraAbilityCost>& AdditionalCost : AdditionalCosts)
    {
        if (AdditionalCost && !AdditionalCost->CheckCost(this, Handle, ActorInfo, OptionalRelevantTags))
            return false;
    }
    return true;
}

void ULyraGameplayAbility::ApplyCost(...) const
{
    Super::ApplyCost(...);
    for (const TObjectPtr<ULyraAbilityCost>& AdditionalCost : AdditionalCosts)
    {
        if (AdditionalCost)
        {
            if (AdditionalCost->ShouldOnlyApplyCostOnHit())
            {
                if (!bAbilityHitTarget) continue;  // 未命中则跳过
            }
            AdditionalCost->ApplyCost(this, Handle, ActorInfo, ActivationInfo);
        }
    }
}
```

### 4.6 EffectContext 与 AbilitySource

```cpp
FGameplayEffectContextHandle ULyraGameplayAbility::MakeEffectContext(...) const
{
    FGameplayEffectContextHandle ContextHandle = Super::MakeEffectContext(Handle, ActorInfo);
    FLyraGameplayEffectContext* EffectContext = FLyraGameplayEffectContext::ExtractEffectContext(ContextHandle);

    // 从 SourceObject（通常是 EquipmentInstance）获取 AbilitySource
    const ILyraAbilitySourceInterface* AbilitySource = nullptr;
    float SourceLevel = 0.0f;
    GetAbilitySource(Handle, ActorInfo, SourceLevel, AbilitySource, EffectCauser);

    EffectContext->SetAbilitySource(AbilitySource, SourceLevel);
    EffectContext->AddInstigator(Instigator, EffectCauser);
    return ContextHandle;
}
```

### 4.7 物理材质标签注入

```cpp
void ULyraGameplayAbility::ApplyAbilityTagsToGameplayEffectSpec(FGameplayEffectSpec& Spec, ...) const
{
    Super::ApplyAbilityTagsToGameplayEffectSpec(Spec, AbilitySpec);
    if (const FHitResult* HitResult = Spec.GetContext().GetHitResult())
    {
        if (const UPhysicalMaterialWithTags* PhysMatWithTags = Cast<const UPhysicalMaterialWithTags>(HitResult->PhysMaterial.Get()))
        {
            Spec.CapturedTargetTags.GetSpecTags().AppendTags(PhysMatWithTags->Tags);
        }
    }
}
```

### 4.8 CameraMode 控制

Ability 可以在激活期间设置/清除相机模式：

```cpp
void ULyraGameplayAbility::SetCameraMode(TSubclassOf<ULyraCameraMode> CameraMode)
{
    if (ULyraHeroComponent* HeroComponent = GetHeroComponentFromActorInfo())
    {
        HeroComponent->SetAbilityCameraMode(CameraMode, CurrentSpecHandle);
        ActiveCameraMode = CameraMode;
    }
}

void ULyraGameplayAbility::EndAbility(...)
{
    ClearCameraMode();  // 技能结束时自动清除相机模式
    Super::EndAbility(...);
}
```

### 4.9 失败反馈机制

技能激活失败时通过 `GameplayMessageSubsystem` 广播反馈：

```cpp
void ULyraGameplayAbility::NativeOnAbilityFailedToActivate(const FGameplayTagContainer& FailedReason) const
{
    for (FGameplayTag Reason : FailedReason)
    {
        // 文本反馈
        if (const FText* pMessage = FailureTagToUserFacingMessages.Find(Reason))
        {
            FLyraAbilitySimpleFailureMessage Message;
            Message.UserFacingReason = *pMessage;
            MessageSystem.BroadcastMessage(TAG_ABILITY_SIMPLE_FAILURE_MESSAGE, Message);
        }

        // 动画 Montage 反馈
        if (UAnimMontage* pMontage = FailureTagToAnimMontage.FindRef(Reason))
        {
            FLyraAbilityMontageFailureMessage Message;
            Message.FailureMontage = pMontage;
            MessageSystem.BroadcastMessage(TAG_ABILITY_PLAY_MONTAGE_FAILURE_MESSAGE, Message);
        }
    }
}
```

---

## 五、ULyraAbilitySet — 批量授予系统

### 5.1 数据结构

`ULyraAbilitySet` 是一个 `UPrimaryDataAsset`，将 Ability、Effect、AttributeSet 打包在一起：

```cpp
UCLASS(BlueprintType, Const)
class ULyraAbilitySet : public UPrimaryDataAsset
{
    // 要授予的 Ability 列表
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay Abilities")
    TArray<FLyraAbilitySet_GameplayAbility> GrantedGameplayAbilities;

    // 要授予的 Effect 列表
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay Effects")
    TArray<FLyraAbilitySet_GameplayEffect> GrantedGameplayEffects;

    // 要授予的 AttributeSet 列表
    UPROPERTY(EditDefaultsOnly, Category = "Attribute Sets")
    TArray<FLyraAbilitySet_AttributeSet> GrantedAttributes;
};
```

每个 Ability 条目包含 InputTag 绑定：

```cpp
USTRUCT(BlueprintType)
struct FLyraAbilitySet_GameplayAbility
{
    TSubclassOf<ULyraGameplayAbility> Ability;
    int32 AbilityLevel = 1;
    FGameplayTag InputTag;  // 绑定的输入标签（如 InputTag.Ability1）
};
```

### 5.2 授予流程

```cpp
void ULyraAbilitySet::GiveToAbilitySystem(ULyraAbilitySystemComponent* LyraASC,
    FLyraAbilitySet_GrantedHandles* OutGrantedHandles, UObject* SourceObject) const
{
    if (!LyraASC->IsOwnerActorAuthoritative())
        return;  // 只在权威端授予

    // 1. 授予 AttributeSet（必须先于 Ability/Effect，因为它们可能依赖属性）
    for (const FLyraAbilitySet_AttributeSet& SetToGrant : GrantedAttributes)
    {
        UAttributeSet* NewSet = NewObject<UAttributeSet>(LyraASC->GetOwner(), SetToGrant.AttributeSet);
        LyraASC->AddAttributeSetSubobject(NewSet);
        OutGrantedHandles->AddAttributeSet(NewSet);
    }

    // 2. 授予 Ability（带 InputTag 绑定）
    for (const FLyraAbilitySet_GameplayAbility& AbilityToGrant : GrantedGameplayAbilities)
    {
        FGameplayAbilitySpec AbilitySpec(AbilityCDO, AbilityToGrant.AbilityLevel);
        AbilitySpec.SourceObject = SourceObject;
        AbilitySpec.GetDynamicSpecSourceTags().AddTag(AbilityToGrant.InputTag);  // ← 关键！InputTag 存储在 DynamicSpecSourceTags 中
        OutGrantedHandles->AddAbilitySpecHandle(LyraASC->GiveAbility(AbilitySpec));
    }

    // 3. 授予 GameplayEffect
    for (const FLyraAbilitySet_GameplayEffect& EffectToGrant : GrantedGameplayEffects)
    {
        const UGameplayEffect* GE = EffectToGrant.GameplayEffect->GetDefaultObject<UGameplayEffect>();
        OutGrantedHandles->AddGameplayEffectHandle(LyraASC->ApplyGameplayEffectToSelf(GE, EffectToGrant.EffectLevel, LyraASC->MakeEffectContext()));
    }
}
```

### 5.3 回收流程

```cpp
void FLyraAbilitySet_GrantedHandles::TakeFromAbilitySystem(ULyraAbilitySystemComponent* LyraASC)
{
    if (!LyraASC->IsOwnerActorAuthoritative()) return;

    for (const FGameplayAbilitySpecHandle& Handle : AbilitySpecHandles)
        LyraASC->ClearAbility(Handle);
    for (const FActiveGameplayEffectHandle& Handle : GameplayEffectHandles)
        LyraASC->RemoveActiveGameplayEffect(Handle);
    for (UAttributeSet* Set : GrantedAttributeSets)
        LyraASC->RemoveSpawnedAttribute(Set);

    AbilitySpecHandles.Reset();
    GameplayEffectHandles.Reset();
    GrantedAttributeSets.Reset();
}
```

---

## 六、属性系统详解

### 6.1 ULyraHealthSet — 生命属性集

#### 属性定义

| 属性 | 类型 | 复制 | 说明 |
|------|------|------|------|
| `Health` | Stateful | `ReplicatedUsing` | 当前生命值，隐藏于 Modifier（仅 Execution 可修改） |
| `MaxHealth` | Stateful | `ReplicatedUsing` | 最大生命值 |
| `Damage` | Meta | 不复制 | 输入伤害值（映射到 -Health） |
| `Healing` | Meta | 不复制 | 输入治疗值（映射到 +Health） |

#### Damage/Healing 的元属性模式

```cpp
void ULyraHealthSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    if (Data.EvaluatedData.Attribute == GetDamageAttribute())
    {
        // 发送伤害消息
        if (Data.EvaluatedData.Magnitude > 0.0f)
        {
            FLyraVerbMessage Message;
            Message.Verb = TAG_Lyra_Damage_Message;
            Message.Magnitude = Data.EvaluatedData.Magnitude;
            MessageSystem.BroadcastMessage(Message.Verb, Message);
        }

        // 将 Damage 转换为 -Health
        SetHealth(FMath::Clamp(GetHealth() - GetDamage(), MinimumHealth, GetMaxHealth()));
        SetDamage(0.0f);  // 重置元属性
    }
    else if (Data.EvaluatedData.Attribute == GetHealingAttribute())
    {
        // 将 Healing 转换为 +Health
        SetHealth(FMath::Clamp(GetHealth() + GetHealing(), MinimumHealth, GetMaxHealth()));
        SetHealing(0.0f);
    }

    // 触发事件委托
    if (GetHealth() != HealthBeforeAttributeChange)
        OnHealthChanged.Broadcast(Instigator, Causer, &Data.EffectSpec, ...);
    if ((GetHealth() <= 0.0f) && !bOutOfHealth)
        OnOutOfHealth.Broadcast(Instigator, Causer, &Data.EffectSpec, ...);
}
```

#### 伤害免疫机制

```cpp
bool ULyraHealthSet::PreGameplayEffectExecute(FGameplayEffectModCallbackData& Data)
{
    if (Data.EvaluatedData.Attribute == GetDamageAttribute())
    {
        if (Data.EvaluatedData.Magnitude > 0.0f)
        {
            const bool bIsSelfDestruct = Data.EffectSpec.GetDynamicAssetTags().HasTagExact(TAG_Gameplay_DamageSelfDestruct);

            // 伤害免疫检查
            if (Data.Target.HasMatchingGameplayTag(TAG_Gameplay_DamageImmunity) && !bIsSelfDestruct)
            {
                Data.EvaluatedData.Magnitude = 0.0f;
                return false;
            }

            // GodMode 作弊检查
            if (Data.Target.HasMatchingGameplayTag(LyraGameplayTags::Cheat_GodMode) && !bIsSelfDestruct)
            {
                Data.EvaluatedData.Magnitude = 0.0f;
                return false;
            }
        }
    }
    return true;
}
```

### 6.2 ULyraCombatSet — 战斗属性集

```cpp
UCLASS(BlueprintType)
class ULyraCombatSet : public ULyraAttributeSet
{
    // 基础伤害值（Source 捕获，传递给 DamageExecution）
    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_BaseDamage, Category = "Lyra|Combat")
    FGameplayAttributeData BaseDamage;

    // 基础治疗值
    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_BaseHeal, Category = "Lyra|Combat")
    FGameplayAttributeData BaseHeal;
};
```

**复制策略**：`COND_OwnerOnly` —— 战斗属性只复制给拥有者，减少带宽。

---

## 七、伤害系统完整链路

### 7.1 ULyraDamageExecution — 伤害计算

```cpp
ULyraDamageExecution::ULyraDamageExecution()
{
    // 捕获 Source 的 BaseDamage 属性
    RelevantAttributesToCapture.Add(FGameplayEffectAttributeCaptureDefinition(
        ULyraCombatSet::GetBaseDamageAttribute(),
        EGameplayEffectAttributeCaptureSource::Source,
        true  // Snapshot
    ));
}
```

执行逻辑：

```cpp
void ULyraDamageExecution::Execute_Implementation(...) const
{
    // 1. 获取基础伤害值（从 Source 的 CombatSet.BaseDamage 捕获）
    float BaseDamage = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().BaseDamageDef, EvaluateParameters, BaseDamage);

    // 2. 阵营检查（友军不造成伤害）
    float DamageInteractionAllowedMultiplier = 0.0f;
    ULyraTeamSubsystem* TeamSubsystem = HitActor->GetWorld()->GetSubsystem<ULyraTeamSubsystem>();
    DamageInteractionAllowedMultiplier = TeamSubsystem->CanCauseDamage(EffectCauser, HitActor) ? 1.0 : 0.0;

    // 3. 距离衰减
    float DistanceAttenuation = 1.0f;
    if (const ILyraAbilitySourceInterface* AbilitySource = TypedContext->GetAbilitySource())
        DistanceAttenuation = AbilitySource->GetDistanceAttenuation(Distance, SourceTags, TargetTags);

    // 4. 物理材质衰减（如头部倍率、护甲减免等）
    float PhysicalMaterialAttenuation = 1.0f;
    if (const UPhysicalMaterial* PhysMat = TypedContext->GetPhysicalMaterial())
        PhysicalMaterialAttenuation = AbilitySource->GetPhysicalMaterialAttenuation(PhysMat, SourceTags, TargetTags);

    // 5. 最终伤害计算
    const float DamageDone = FMath::Max(
        BaseDamage * DistanceAttenuation * PhysicalMaterialAttenuation * DamageInteractionAllowedMultiplier,
        0.0f);

    // 6. 输出到 HealthSet.Damage 元属性
    if (DamageDone > 0.0f)
        OutExecutionOutput.AddOutputModifier(
            FGameplayModifierEvaluatedData(ULyraHealthSet::GetDamageAttribute(), EGameplayModOp::Additive, DamageDone));
}
```

### 7.2 伤害完整时序图

```
[武器射击 Ability] → PerformLocalTargeting() → 收集 HitResult
    ↓
StartRangedWeaponTargeting() → 构建 TargetData → OnTargetDataReadyCallback()
    ↓
CommitAbility() → 检查 Cost → ApplyCost()
    ↓
OnRangedWeaponTargetDataReady() (蓝图) → 应用 DamageGameplayEffect
    ↓
[DamageGameplayEffect] → Execute ULyraDamageExecution
    ↓                         ↓ 捕获 Source.BaseDamage
    ↓                         ↓ 计算距离/材质衰减
    ↓                         ↓ 检查队伍关系
    ↓                         → 输出到 Target.HealthSet.Damage
    ↓
ULyraHealthSet::PreGameplayEffectExecute()
    ↓ 检查免疫/GodMode
ULyraHealthSet::PostGameplayEffectExecute()
    ↓ Damage → -Health
    ↓ 广播 Lyra.Damage.Message
    ↓
if (Health <= 0) → OnOutOfHealth 委托
    ↓
ULyraHealthComponent::HandleOutOfHealth()
    ↓ 发送 GameplayEvent.Death
    ↓
ULyraGameplayAbility_Death (由 GameplayEvent 触发)
    ↓ CancelAbilities (除 SurvivesDeath)
    ↓ SetCanBeCanceled(false)
    ↓ ChangeActivationGroup(Exclusive_Blocking)
    ↓ StartDeath() → HealthComponent.StartDeath()
    ↓               → 设置 Status.Death.Dying 标签
    ↓ FinishDeath() → HealthComponent.FinishDeath()
                    → 设置 Status.Death.Dead 标签
```

---

## 八、具体 Ability 子类分析

### 8.1 ULyraGameplayAbility_Death

**触发机制**：通过 CDO 构造函数注册 GameplayEvent Trigger：

```cpp
ULyraGameplayAbility_Death::ULyraGameplayAbility_Death(...)
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerInitiated;

    if (HasAnyFlags(RF_ClassDefaultObject))
    {
        FAbilityTriggerData TriggerData;
        TriggerData.TriggerTag = LyraGameplayTags::GameplayEvent_Death;
        TriggerData.TriggerSource = EGameplayAbilityTriggerSource::GameplayEvent;
        AbilityTriggers.Add(TriggerData);
    }
}
```

**激活逻辑**：

```cpp
void ULyraGameplayAbility_Death::ActivateAbility(...)
{
    // 取消所有非 SurvivesDeath 的技能
    FGameplayTagContainer AbilityTypesToIgnore;
    AbilityTypesToIgnore.AddTag(LyraGameplayTags::Ability_Behavior_SurvivesDeath);
    LyraASC->CancelAbilities(nullptr, &AbilityTypesToIgnore, this);

    // 设为不可取消 + Exclusive_Blocking
    SetCanBeCanceled(false);
    ChangeActivationGroup(ELyraAbilityActivationGroup::Exclusive_Blocking);

    if (bAutoStartDeath)
        StartDeath();

    Super::ActivateAbility(...);
}
```

### 8.2 ULyraGameplayAbility_Jump

```cpp
ULyraGameplayAbility_Jump::ULyraGameplayAbility_Jump(...)
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
}

bool ULyraGameplayAbility_Jump::CanActivateAbility(...) const
{
    const ALyraCharacter* LyraCharacter = Cast<ALyraCharacter>(ActorInfo->AvatarActor.Get());
    if (!LyraCharacter || !LyraCharacter->CanJump())
        return false;
    return Super::CanActivateAbility(...);
}
```

### 8.3 ULyraGameplayAbility_Reset

```cpp
void ULyraGameplayAbility_Reset::ActivateAbility(...)
{
    // 取消所有技能（除 SurvivesDeath）
    LyraASC->CancelAbilities(nullptr, &AbilityTypesToIgnore, this);
    SetCanBeCanceled(false);

    // 执行角色重置
    if (ALyraCharacter* LyraChar = Cast<ALyraCharacter>(CurrentActorInfo->AvatarActor.Get()))
        LyraChar->Reset();

    // 广播重置消息
    FLyraPlayerResetMessage Message;
    Message.OwnerPlayerState = CurrentActorInfo->OwnerActor.Get();
    MessageSystem.BroadcastMessage(LyraGameplayTags::GameplayEvent_Reset, Message);

    // 立即结束
    EndAbility(...);
}
```

### 8.4 ULyraGameplayAbility_FromEquipment

通过 `AbilitySpec.SourceObject` 获取关联的装备和物品：

```cpp
ULyraEquipmentInstance* ULyraGameplayAbility_FromEquipment::GetAssociatedEquipment() const
{
    if (FGameplayAbilitySpec* Spec = GetCurrentAbilitySpec())
        return Cast<ULyraEquipmentInstance>(Spec->SourceObject.Get());
    return nullptr;
}

ULyraInventoryItemInstance* ULyraGameplayAbility_FromEquipment::GetAssociatedItem() const
{
    if (ULyraEquipmentInstance* Equipment = GetAssociatedEquipment())
        return Cast<ULyraInventoryItemInstance>(Equipment->GetInstigator());
    return nullptr;
}
```

### 8.5 ULyraGameplayAbility_RangedWeapon

远程武器技能是最复杂的 Ability 子类，实现了完整的射击系统。

#### 激活流程

```cpp
void ULyraGameplayAbility_RangedWeapon::ActivateAbility(...)
{
    // 绑定 TargetData 回调
    OnTargetDataReadyCallbackDelegateHandle = MyAbilityComponent->AbilityTargetDataSetDelegate(
        CurrentSpecHandle, CurrentActivationInfo.GetActivationPredictionKey()
    ).AddUObject(this, &ThisClass::OnTargetDataReadyCallback);

    // 更新射击时间（用于散布计算）
    WeaponData->UpdateFiringTime();
    Super::ActivateAbility(...);
}
```

#### 瞄准与射线检测

```cpp
void ULyraGameplayAbility_RangedWeapon::PerformLocalTargeting(OUT TArray<FHitResult>& OutHits)
{
    // 从相机方向获取瞄准 Transform
    const FTransform TargetTransform = GetTargetingTransform(AvatarPawn, ELyraAbilityTargetingSource::CameraTowardsFocus);
    InputData.AimDir = TargetTransform.GetUnitAxis(EAxis::X);
    InputData.StartTrace = TargetTransform.GetTranslation();
    InputData.EndAim = InputData.StartTrace + InputData.AimDir * WeaponData->GetMaxDamageRange();

    TraceBulletsInCartridge(InputData, OutHits);
}
```

#### 子弹散布模型（正态分布）

```cpp
FVector VRandConeNormalDistribution(const FVector& Dir, const float ConeHalfAngleRad, const float Exponent)
{
    const float FromCenter = FMath::Pow(FMath::FRand(), Exponent);  // 指数分布
    const float AngleFromCenter = FromCenter * ConeHalfAngleDegrees;
    const float AngleAround = FMath::FRand() * 360.0f;
    // ... 四元数旋转 ...
}
```

#### 双重射线策略（先 Ray 后 Sweep）

```cpp
FHitResult DoSingleBulletTrace(...)
{
    // 第一次：精确射线检测
    Impact = WeaponTrace(StartTrace, EndTrace, 0.0f, bIsSimulated, OutHits);

    // 如果没有命中 Pawn 且支持 SweepRadius
    if (FindFirstPawnHitResult(OutHits) == INDEX_NONE && SweepRadius > 0.0f)
    {
        // 第二次：球形扫描检测（更宽容的命中判定）
        Impact = WeaponTrace(StartTrace, EndTrace, SweepRadius, bIsSimulated, SweepHits);
    }
}
```

#### 网络同步

```cpp
void ULyraGameplayAbility_RangedWeapon::OnTargetDataReadyCallback(...)
{
    // 客户端通知服务器 TargetData
    const bool bShouldNotifyServer = CurrentActorInfo->IsLocallyControlled() && !CurrentActorInfo->IsNetAuthority();
    if (bShouldNotifyServer)
    {
        MyAbilityComponent->CallServerSetReplicatedTargetData(
            CurrentSpecHandle, CurrentActivationInfo.GetActivationPredictionKey(),
            LocalTargetDataHandle, ApplicationTag, MyAbilityComponent->ScopedPredictionKey);
    }

    // 服务器确认命中标记
    if (Controller->GetLocalRole() == ROLE_Authority)
    {
        WeaponStateComponent->ClientConfirmTargetData(LocalTargetDataHandle.UniqueId, bIsTargetDataValid, HitReplaces);
    }

    // 提交技能（消耗弹药等）并添加散布
    if (CommitAbility(...))
    {
        WeaponData->AddSpread();
        OnRangedWeaponTargetDataReady(LocalTargetDataHandle);  // 蓝图处理命中
    }
}
```

### 8.6 ULyraGamePhaseAbility

游戏阶段技能用于管理比赛流程（如等待开始 → 正式比赛 → 突然死亡）：

```cpp
ULyraGamePhaseAbility::ULyraGamePhaseAbility(...)
{
    ReplicationPolicy = EGameplayAbilityReplicationPolicy::ReplicateNo;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerInitiated;
    NetSecurityPolicy = EGameplayAbilityNetSecurityPolicy::ServerOnly;
}

void ULyraGamePhaseAbility::ActivateAbility(...)
{
    if (ActorInfo->IsNetAuthority())
    {
        ULyraGamePhaseSubsystem* PhaseSubsystem = UWorld::GetSubsystem<ULyraGamePhaseSubsystem>(World);
        PhaseSubsystem->OnBeginPhase(this, Handle);  // 通知阶段子系统
    }
    Super::ActivateAbility(...);
}

void ULyraGamePhaseAbility::EndAbility(...)
{
    if (ActorInfo->IsNetAuthority())
        PhaseSubsystem->OnEndPhase(this, Handle);
    Super::EndAbility(...);
}
```

阶段以嵌套 Tag 形式组织（如 `GamePhase.Playing.NormalPlay`），切换子阶段时只取消同级和子级，不影响父级。

---

## 九、自定义 GameplayEffectContext

### 9.1 FLyraGameplayEffectContext

```cpp
USTRUCT()
struct FLyraGameplayEffectContext : public FGameplayEffectContext
{
    // 子弹所属的弹药批次 ID（同一次射击的多颗子弹共享）
    UPROPERTY()
    int32 CartridgeID = -1;

    // 技能来源对象（实现 ILyraAbilitySourceInterface 接口）
    UPROPERTY()
    TWeakObjectPtr<const UObject> AbilitySourceObject;
};
```

### 9.2 ULyraAbilitySystemGlobals

```cpp
FGameplayEffectContext* ULyraAbilitySystemGlobals::AllocGameplayEffectContext() const
{
    return new FLyraGameplayEffectContext();  // 全局替换 EffectContext 类型
}
```

配置入口（在 `DefaultGame.ini`）：
```ini
[/Script/GameplayAbilities.AbilitySystemGlobals]
AbilitySystemGlobalsClassName=/Script/LyraGame.LyraAbilitySystemGlobals
```

---

## 十、ULyraHealthComponent — 生命值管理组件

### 10.1 职责

`ULyraHealthComponent` 是 Actor 层面的生命值管理器，**不直接操作属性**，而是**监听 HealthSet 的委托**：

```cpp
void ULyraHealthComponent::InitializeWithAbilitySystem(ULyraAbilitySystemComponent* InASC)
{
    AbilitySystemComponent = InASC;
    HealthSet = AbilitySystemComponent->GetSet<ULyraHealthSet>();

    // 注册委托
    HealthSet->OnHealthChanged.AddUObject(this, &ThisClass::HandleHealthChanged);
    HealthSet->OnMaxHealthChanged.AddUObject(this, &ThisClass::HandleMaxHealthChanged);
    HealthSet->OnOutOfHealth.AddUObject(this, &ThisClass::HandleOutOfHealth);

    // 初始化生命值为最大值
    AbilitySystemComponent->SetNumericAttributeBase(ULyraHealthSet::GetHealthAttribute(), HealthSet->GetMaxHealth());
}
```

### 10.2 死亡流程

```cpp
void ULyraHealthComponent::HandleOutOfHealth(...)
{
    // 通过 GameplayEvent 触发死亡 Ability
    FGameplayEventData Payload;
    Payload.EventTag = LyraGameplayTags::GameplayEvent_Death;
    Payload.Instigator = DamageInstigator;
    Payload.Target = AbilitySystemComponent->GetAvatarActor();
    AbilitySystemComponent->HandleGameplayEvent(Payload.EventTag, &Payload);

    // 广播击杀消息
    FLyraVerbMessage Message;
    Message.Verb = TAG_Lyra_Elimination_Message;
    MessageSystem.BroadcastMessage(Message.Verb, Message);
}

void ULyraHealthComponent::StartDeath()
{
    DeathState = ELyraDeathState::DeathStarted;
    AbilitySystemComponent->SetLooseGameplayTagCount(LyraGameplayTags::Status_Death_Dying, 1);
    OnDeathStarted.Broadcast(Owner);
}

void ULyraHealthComponent::FinishDeath()
{
    DeathState = ELyraDeathState::DeathFinished;
    AbilitySystemComponent->SetLooseGameplayTagCount(LyraGameplayTags::Status_Death_Dead, 1);
    OnDeathFinished.Broadcast(Owner);
}
```

### 10.3 自毁机制

```cpp
void ULyraHealthComponent::DamageSelfDestruct(bool bFellOutOfWorld)
{
    const TSubclassOf<UGameplayEffect> DamageGE = ULyraAssetManager::GetSubclass(ULyraGameData::Get().DamageGameplayEffect_SetByCaller);
    FGameplayEffectSpecHandle SpecHandle = AbilitySystemComponent->MakeOutgoingSpec(DamageGE, 1.0f, AbilitySystemComponent->MakeEffectContext());

    Spec->AddDynamicAssetTag(TAG_Gameplay_DamageSelfDestruct);
    if (bFellOutOfWorld)
        Spec->AddDynamicAssetTag(TAG_Gameplay_FellOutOfWorld);

    Spec->SetSetByCallerMagnitude(LyraGameplayTags::SetByCaller_Damage, GetMaxHealth());
    AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*Spec);
}
```

---

## 十一、ULyraGlobalAbilitySystem — 全局 Ability/Effect

### 11.1 设计模式

`ULyraGlobalAbilitySystem` 是一个 `UWorldSubsystem`，维护所有已注册 ASC 的列表，支持全局广播：

```cpp
UCLASS()
class ULyraGlobalAbilitySystem : public UWorldSubsystem
{
    // 全局应用的 Ability（如全局被动）
    TMap<TSubclassOf<UGameplayAbility>, FGlobalAppliedAbilityList> AppliedAbilities;

    // 全局应用的 Effect（如全局 Buff/Debuff）
    TMap<TSubclassOf<UGameplayEffect>, FGlobalAppliedEffectList> AppliedEffects;

    // 所有已注册的 ASC
    TArray<TObjectPtr<ULyraAbilitySystemComponent>> RegisteredASCs;
};
```

### 11.2 注册/注销

```cpp
void ULyraGlobalAbilitySystem::RegisterASC(ULyraAbilitySystemComponent* ASC)
{
    // 为新 ASC 应用所有现有全局 Ability/Effect
    for (auto& Entry : AppliedAbilities)
        Entry.Value.AddToASC(Entry.Key, ASC);
    for (auto& Entry : AppliedEffects)
        Entry.Value.AddToASC(Entry.Key, ASC);
    RegisteredASCs.AddUnique(ASC);
}
```

### 11.3 全局广播

```cpp
void ULyraGlobalAbilitySystem::ApplyAbilityToAll(TSubclassOf<UGameplayAbility> Ability)
{
    if (Ability.Get() && !AppliedAbilities.Contains(Ability))
    {
        FGlobalAppliedAbilityList& Entry = AppliedAbilities.Add(Ability);
        for (ULyraAbilitySystemComponent* ASC : RegisteredASCs)
            Entry.AddToASC(Ability, ASC);
    }
}
```

---

## 十二、ULyraGameplayCueManager — 延迟加载优化

### 12.1 三种加载模式

```cpp
enum class ELyraEditorLoadMode
{
    LoadUpfront,                            // 启动时全部加载（编辑器默认）
    PreloadAsCuesAreReferenced_GameOnly,    // 运行时按引用预加载（仅游戏）
    PreloadAsCuesAreReferenced              // 运行时按引用预加载（含编辑器）
};
```

### 12.2 延迟加载流程

```
[GameplayTag 被加载] → OnGameplayTagLoaded()
                         ↓ (线程安全收集)
                    [GameThread] ProcessLoadedTags()
                         ↓
                    ProcessTagToPreload()
                         ↓ 查找 CueSet 中的数据
                    StreamableManager.RequestAsyncLoad()
                         ↓
                    OnPreloadCueComplete()
                         ↓
                    RegisterPreloadedCue() → PreloadedCues / AlwaysLoadedCues
```

### 12.3 关键策略

```cpp
bool ULyraGameplayCueManager::ShouldSyncLoadMissingGameplayCues() const
{
    return false;  // 绝不同步加载缺失的 Cue（避免卡顿）
}

bool ULyraGameplayCueManager::ShouldAsyncLoadMissingGameplayCues() const
{
    return true;   // 缺失时异步加载
}

bool ULyraGameplayCueManager::ShouldDelayLoadGameplayCues() const
{
    return !IsRunningDedicatedServer() && bClientDelayLoadGameplayCues;
    // 专用服务器不需要加载 Cue
}
```

---

## 十三、GameplayTag 系统

### 13.1 Lyra 定义的核心 Tag

| Tag | 说明 |
|-----|------|
| `Ability.ActivateFail.IsDead` | 因为死亡而激活失败 |
| `Ability.ActivateFail.Cooldown` | 因为冷却而激活失败 |
| `Ability.ActivateFail.Cost` | 因为消耗不足而激活失败 |
| `Ability.ActivateFail.TagsBlocked` | 因为标签被阻塞而激活失败 |
| `Ability.ActivateFail.TagsMissing` | 因为缺少标签而激活失败 |
| `Ability.ActivateFail.Networking` | 因为网络条件而激活失败 |
| `Ability.ActivateFail.ActivationGroup` | 因为激活组冲突而激活失败 |
| `Ability.Behavior.SurvivesDeath` | 拥有此 Tag 的技能在死亡时不被取消 |
| `Ability.Weapon.NoFiring` | 阻止武器发射 |
| `Gameplay.AbilityInputBlocked` | 阻止所有输入型技能激活 |
| `Gameplay.Damage` | 伤害事件 Tag |
| `Gameplay.DamageImmunity` | 伤害免疫 |
| `Gameplay.Damage.SelfDestruct` | 自毁伤害（无视免疫） |
| `Gameplay.Damage.FellOutOfWorld` | 掉出世界 |
| `Status.Death` | 死亡状态根 Tag |
| `Status.Death.Dying` | 正在死亡 |
| `Status.Death.Dead` | 已死亡 |
| `Status.Crouching` | 蹲下状态 |
| `GameplayEvent.Death` | 死亡事件（触发 Death Ability） |
| `GameplayEvent.Reset` | 重置完成事件 |
| `GameplayEvent.RequestReset` | 请求重置事件（触发 Reset Ability） |
| `SetByCaller.Damage` | SetByCaller 伤害数值 |
| `SetByCaller.Heal` | SetByCaller 治疗数值 |
| `InputTag.Move` | 移动输入 |
| `InputTag.Look.Mouse` | 鼠标观察 |
| `InputTag.Look.Stick` | 手柄摇杆观察 |
| `InputTag.Crouch` | 蹲下输入 |
| `Movement.Mode.*` | 移动模式标签映射 |
| `Cheat.GodMode` | 上帝模式作弊 |
| `Cheat.UnlimitedHealth` | 无限生命作弊 |

### 13.2 MovementMode → Tag 映射

```cpp
const TMap<uint8, FGameplayTag> MovementModeTagMap =
{
    { MOVE_Walking,    Movement_Mode_Walking },
    { MOVE_NavWalking, Movement_Mode_NavWalking },
    { MOVE_Falling,    Movement_Mode_Falling },
    { MOVE_Swimming,   Movement_Mode_Swimming },
    { MOVE_Flying,     Movement_Mode_Flying },
    { MOVE_Custom,     Movement_Mode_Custom }
};
```

---

## 十四、完整系统时序图

### 14.1 角色初始化 → 技能就绪

```
[GameMode.InitGame]
    ↓
[Experience Loading] → ExperienceManager.OnExperienceLoaded
    ↓
[HeroComponent.OnDataInitialized]
    ↓
[PawnExtensionComponent] → InitializeAbilitySystem()
    ↓
    ├── 创建 ULyraAbilitySystemComponent (在 PlayerState 上)
    ├── ASC.InitAbilityActorInfo(PlayerState, Pawn)
    │     ├── 通知所有 Ability: OnPawnAvatarSet()
    │     ├── GlobalAbilitySystem.RegisterASC(this)
    │     ├── AnimInstance.InitializeWithAbilitySystem()
    │     └── TryActivateAbilitiesOnSpawn()
    ├── HealthComponent.InitializeWithAbilitySystem(ASC)
    └── AbilitySet.GiveToAbilitySystem(ASC)
          ├── 授予 AttributeSet (HealthSet, CombatSet)
          ├── 授予 Ability (Death, Jump, RangedWeapon, ...)
          │     └── 每个 Ability.InputTag → DynamicSpecSourceTags
          └── 授予 GameplayEffect (初始 Buff 等)
```

### 14.2 输入 → 技能激活

```
[Enhanced Input Action 触发]
    ↓
[LyraHeroComponent] → ASC.AbilityInputTagPressed(InputTag)
    ↓                    匹配 DynamicSpecSourceTags
    ↓                    → InputPressedSpecHandles / InputHeldSpecHandles
    ↓
[PlayerController.PostProcessInput]
    ↓
ASC.ProcessAbilityInput(DeltaTime, bGamePaused)
    ├── 检查 TAG_Gameplay_AbilityInputBlocked
    ├── 处理 WhileInputActive 持续激活
    ├── 处理 OnInputTriggered 按下激活
    ├── TryActivateAbility() × N
    └── 处理 InputReleased
```

### 14.3 伤害 → 死亡

```
[RangedWeapon.OnTargetDataReady] → 应用 DamageGameplayEffect
    ↓
[DamageExecution.Execute]
    ├── 捕获 Source.CombatSet.BaseDamage
    ├── 距离衰减 × 材质衰减 × 阵营检查
    └── 输出 → Target.HealthSet.Damage

[HealthSet.PreGameplayEffectExecute]
    ├── 检查 DamageImmunity
    └── 检查 GodMode

[HealthSet.PostGameplayEffectExecute]
    ├── Damage → SetHealth(Health - Damage)
    ├── 广播 OnHealthChanged
    └── if (Health <= 0) → 广播 OnOutOfHealth

[HealthComponent.HandleOutOfHealth]
    └── HandleGameplayEvent(GameplayEvent.Death)

[Death Ability 被触发]
    ├── CancelAbilities (除 SurvivesDeath)
    ├── Exclusive_Blocking
    ├── StartDeath → Status.Death.Dying
    └── FinishDeath → Status.Death.Dead
```

---

## 十五、设计模式总结

| 模式 | 实现 | 说明 |
|------|------|------|
| **组合模式** | AbilitySet | 将 Ability + Effect + AttributeSet 打包为可复用数据资产 |
| **策略模式** | ActivationPolicy / ActivationGroup | 通过枚举控制激活行为 |
| **策略模式** | ULyraAbilityCost | 可插拔的消耗计算策略 |
| **观察者模式** | FLyraAttributeEvent / HealthComponent 委托 | 属性变化通知 |
| **元属性模式** | Damage/Healing → Health | 输入→转换→输出的间接修改方式 |
| **命令模式** | GameplayEvent 触发 Ability | Death/Reset 通过事件触发而非直接调用 |
| **中介者模式** | GlobalAbilitySystem | 全局 Ability/Effect 广播中枢 |
| **数据驱动** | TagRelationshipMapping | 技能互斥关系通过数据资产配置 |
| **标签匹配** | InputTag → DynamicSpecSourceTags | 输入与技能的解耦绑定 |
| **双重检测** | Ray + Sweep 策略 | 先精确后宽容的射击命中判定 |
| **客户端预测** | LocalPredicted + TargetData 复制 | 射击预测 + 服务器验证 |
| **延迟加载** | GameplayCueManager | 按引用异步预加载 GameplayCue |

---

## 十六、关键设计决策分析

### 16.1 为什么 ASC 在 PlayerState 上而不是 Pawn 上

- PlayerState 在 Pawn 重生时不销毁，保留技能、效果、属性的连续性
- ASC 通过 `InitAbilityActorInfo(PlayerState, NewPawn)` 切换 Avatar
- 全局 Effect（如比赛级 Buff）不会因 Pawn 销毁而丢失

### 16.2 为什么使用 InputTag 而不是 InputID

- **解耦**：Ability 不需要知道具体的输入键位，只需要声明关注哪个 InputTag
- **灵活**：同一个 InputTag 可以映射到不同的物理输入（键盘/手柄）
- **动态绑定**：通过 `DynamicSpecSourceTags` 实现运行时绑定，支持装备切换

### 16.3 Meta Attribute 的设计意图

`Damage` 和 `Healing` 作为"元属性"（Meta Attribute）存在：
- 不被网络复制（节省带宽）
- 在 `PostGameplayEffectExecute` 中消费并转换为 Health 变化
- 允许在转换前进行拦截（免疫、护盾等）
- 支持广播伤害消息而不直接暴露 Health 修改

### 16.4 AbilitySet 的数据驱动优势

- 编辑器中可视化配置每个角色/装备的技能组合
- 支持运行时动态授予/回收（装备切换、状态变化）
- 保留所有 Handle 用于精确回收，避免内存泄漏
