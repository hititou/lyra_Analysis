# Lyra 模块化基类与 GameFeature 动态注入机制 — 代码级深度分析

## 一、概述

Lyra 项目的核心设计理念是**"所有核心 Gameplay 类继承自 Modular 基类，通过 GameFeature 插件动态注入功能"**。这套机制依赖三大支柱：

1. **Modular 基类体系** — 引擎提供的 `ModularGameplay` / `ModularGameplayActors` 插件中的基类
2. **GameFrameworkComponentManager** — 引擎级的组件管理器，支持动态组件注入和扩展事件
3. **GameFeatureAction** — 定义在 GameFeature 插件激活/停用时需要执行的操作

---

## 二、Modular 基类继承关系

### 2.1 Lyra 核心类与 Modular 基类的继承链

```
┌─────────────────────────────────────────────────────────────────────┐
│                          UE5 引擎基类                                │
├─────────────────────────────────────────────────────────────────────┤
│  AGameModeBase          AGameStateBase        ACharacter            │
│  APlayerController      APlayerState                                │
└────────┬────────────────┬──────────────────────┬────────────────────┘
         │                │                      │
┌────────▼────────┐ ┌─────▼──────────┐  ┌───────▼───────────────┐
│ ModularGameplay │ │ ModularGameplay│  │ ModularGameplayActors │
│    插件基类      │ │    插件基类     │  │      插件基类          │
├─────────────────┤ ├────────────────┤  ├───────────────────────┤
│AModularGameMode │ │AModularGameSta │  │AModularCharacter      │
│       Base      │ │    teBase      │  │AModularPlayerState    │
│                 │ │                │  │AModularPlayerController│
└────────┬────────┘ └───────┬────────┘  └──────┬────────────────┘
         │                  │                   │
   ┌─────▼──────┐   ┌──────▼───────┐   ┌──────▼──────────┐
   │CommonGame  │   │              │   │CommonGame       │
   │  插件基类   │   │              │   │  插件基类        │
   ├────────────┤   │              │   ├─────────────────┤
   │            │   │              │   │ACommonPlayer    │
   │            │   │              │   │   Controller    │
   └─────┬──────┘   └──────┬───────┘   └──────┬──────────┘
         │                 │                   │
  ┌──────▼───────┐  ┌──────▼────────┐  ┌──────▼──────────────┐
  │ Lyra 核心类   │  │ Lyra 核心类    │  │ Lyra 核心类          │
  ├──────────────┤  ├───────────────┤  ├─────────────────────┤
  │ALyraGameMode │  │ALyraGameState │  │ALyraCharacter       │
  │              │  │               │  │ALyraPlayerController│
  │              │  │               │  │ALyraPlayerState     │
  └──────────────┘  └───────────────┘  └─────────────────────┘
```

### 2.2 代码中的继承声明

**ALyraCharacter** — 继承自 `AModularCharacter`：

```cpp
// Source/LyraGame/Character/LyraCharacter.h
UCLASS(MinimalAPI, Config = Game)
class ALyraCharacter : public AModularCharacter,
                       public IAbilitySystemInterface,
                       public IGameplayCueInterface,
                       public IGameplayTagAssetInterface,
                       public ILyraTeamAgentInterface
{
    GENERATED_BODY()
    // ...
};
```

**ALyraGameMode** — 继承自 `AModularGameModeBase`：

```cpp
// Source/LyraGame/GameModes/LyraGameMode.h
UCLASS(MinimalAPI, Config = Game)
class ALyraGameMode : public AModularGameModeBase
{
    GENERATED_BODY()
    // ...
};
```

**ALyraGameState** — 继承自 `AModularGameStateBase`：

```cpp
// Source/LyraGame/GameModes/LyraGameState.h
UCLASS(MinimalAPI, Config = Game)
class ALyraGameState : public AModularGameStateBase, public IAbilitySystemInterface
{
    GENERATED_BODY()
    // ...
};
```

**ALyraPlayerState** — 继承自 `AModularPlayerState`：

```cpp
// Source/LyraGame/Player/LyraPlayerState.h
UCLASS(MinimalAPI, Config = Game)
class ALyraPlayerState : public AModularPlayerState,
                         public IAbilitySystemInterface,
                         public ILyraTeamAgentInterface
{
    GENERATED_BODY()
    // ...
};
```

**ALyraPlayerController** — 继承自 `ACommonPlayerController`（间接继承 `AModularPlayerController`）：

```cpp
// Source/LyraGame/Player/LyraPlayerController.h
UCLASS(MinimalAPI, Config = Game)
class ALyraPlayerController : public ACommonPlayerController,
                              public ILyraCameraAssistInterface,
                              public ILyraTeamAgentInterface
{
    GENERATED_BODY()
    // ...
};
```

### 2.3 Modular 基类的核心能力

Modular 基类（如 `AModularCharacter`）的本质是在关键生命周期回调中，通过 `UGameFrameworkComponentManager` 广播**扩展事件**，使外部系统能够监听 Actor 的创建/销毁等事件并动态注入组件或功能。

引擎中 `AModularCharacter` 的关键实现（概念伪代码）：

```cpp
// 引擎 ModularGameplayActors 插件中的 AModularCharacter
void AModularCharacter::PreInitializeComponents()
{
    Super::PreInitializeComponents();
    // 向 GameFrameworkComponentManager 广播：本 Actor 正在初始化
    UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(
        this, UGameFrameworkComponentManager::NAME_GameActorReady);
}
```

这使得任何注册了对该 Actor 类型的扩展处理器的 `GameFeatureAction` 都能收到通知并执行操作（如注入组件、授予技能等）。

---

## 三、GameFrameworkComponentManager — 核心中枢

`UGameFrameworkComponentManager` 是整个动态注入机制的**中枢调度器**，它提供两个核心 API：

### 3.1 AddExtensionHandler — 注册扩展处理器

```cpp
// 引擎 API
TSharedPtr<FComponentRequestHandle> AddExtensionHandler(
    const TSoftClassPtr<AActor>& ReceiverClass,
    FExtensionHandlerDelegate AddAbilitiesDelegate
);
```

**作用**：当指定类型的 Actor 被创建或收到特定事件时，调用注册的委托。

**Lyra 中的使用（`GameFeatureAction_AddAbilities::AddToWorld`）**：

```cpp
// Source/LyraGame/GameFeatures/GameFeatureAction_AddAbilities.cpp
void UGameFeatureAction_AddAbilities::AddToWorld(
    const FWorldContext& WorldContext,
    const FGameFeatureStateChangeContext& ChangeContext)
{
    UWorld* World = WorldContext.World();
    UGameInstance* GameInstance = WorldContext.OwningGameInstance;
    FPerContextData& ActiveData = ContextData.FindOrAdd(ChangeContext);

    if ((GameInstance != nullptr) && (World != nullptr) && World->IsGameWorld())
    {
        if (UGameFrameworkComponentManager* ComponentMan =
            UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(GameInstance))
        {
            int32 EntryIndex = 0;
            for (const FGameFeatureAbilitiesEntry& Entry : AbilitiesList)
            {
                if (!Entry.ActorClass.IsNull())
                {
                    // 注册扩展处理器：当 Entry.ActorClass 类型的 Actor 出现时调用 HandleActorExtension
                    UGameFrameworkComponentManager::FExtensionHandlerDelegate AddAbilitiesDelegate =
                        UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
                            this,
                            &UGameFeatureAction_AddAbilities::HandleActorExtension,
                            EntryIndex,
                            ChangeContext);

                    TSharedPtr<FComponentRequestHandle> ExtensionRequestHandle =
                        ComponentMan->AddExtensionHandler(Entry.ActorClass, AddAbilitiesDelegate);

                    ActiveData.ComponentRequests.Add(ExtensionRequestHandle);
                    EntryIndex++;
                }
            }
        }
    }
}
```

### 3.2 SendGameFrameworkComponentExtensionEvent — 发送扩展事件

```cpp
// 引擎 API
static void SendGameFrameworkComponentExtensionEvent(
    AActor* OwningActor,
    const FName& EventName
);
```

**Lyra 中的使用**（`ALyraPlayerState::SetPawnData`）：

```cpp
// Source/LyraGame/Player/LyraPlayerState.cpp
void ALyraPlayerState::SetPawnData(const ULyraPawnData* InPawnData)
{
    // ... PawnData 设置逻辑 ...

    PawnData = InPawnData;

    // 授予 AbilitySet 中的技能
    for (const ULyraAbilitySet* AbilitySet : PawnData->AbilitySets)
    {
        if (AbilitySet)
        {
            AbilitySet->GiveToAbilitySystem(AbilitySystemComponent, nullptr);
        }
    }

    // 发送自定义扩展事件 "LyraAbilitiesReady"
    // 所有监听此事件的 GameFeatureAction 都会收到通知
    UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(
        this, NAME_LyraAbilityReady);

    ForceNetUpdate();
}
```

### 3.3 AddComponentRequest — 动态添加组件

```cpp
// 引擎 API
TSharedPtr<FComponentRequestHandle> AddComponentRequest(
    const TSoftClassPtr<AActor>& ReceiverClass,
    TSubclassOf<UActorComponent> ComponentClass
);
```

当请求的 Actor 类型被创建时，自动为其添加指定的组件。请求是**引用计数的**，当所有请求的 Handle 被释放时组件才会移除。

---

## 四、GameFeatureAction — 功能注入的执行者

### 4.1 基类架构

```
UGameFeatureAction (引擎基类)
└── UGameFeatureAction_WorldActionBase (Lyra 基类)
    ├── UGameFeatureAction_AddAbilities       ← 注入技能/属性
    ├── UGameFeatureAction_AddInputBinding    ← 注入输入绑定
    ├── UGameFeatureAction_AddInputContextMapping ← 注入输入映射
    ├── UGameFeatureAction_AddWidget          ← 注入 UI Widget
    └── UGameFeatureAction_SplitscreenConfig  ← 分屏配置
```

### 4.2 WorldActionBase — 世界感知的基类

`UGameFeatureAction_WorldActionBase` 在 GameFeature 激活时，监听 GameInstance 启动事件，对每个有效的游戏世界调用 `AddToWorld`：

```cpp
// Source/LyraGame/GameFeatures/GameFeatureAction_WorldActionBase.cpp
void UGameFeatureAction_WorldActionBase::OnGameFeatureActivating(FGameFeatureActivatingContext& Context)
{
    // 监听未来的 GameInstance 启动
    GameInstanceStartHandles.FindOrAdd(Context) = FWorldDelegates::OnStartGameInstance.AddUObject(
        this,
        &UGameFeatureAction_WorldActionBase::HandleGameInstanceStart,
        FGameFeatureStateChangeContext(Context));

    // 处理已经存在的世界
    for (const FWorldContext& WorldContext : GEngine->GetWorldContexts())
    {
        if (Context.ShouldApplyToWorldContext(WorldContext))
        {
            AddToWorld(WorldContext, Context);  // 纯虚函数，由子类实现
        }
    }
}

void UGameFeatureAction_WorldActionBase::OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context)
{
    FDelegateHandle* FoundHandle = GameInstanceStartHandles.Find(Context);
    if (ensure(FoundHandle))
    {
        FWorldDelegates::OnStartGameInstance.Remove(*FoundHandle);
    }
}
```

### 4.3 GameFeatureAction_AddAbilities — 技能/属性注入（最核心）

#### 数据结构

```cpp
// Source/LyraGame/GameFeatures/GameFeatureAction_AddAbilities.h
USTRUCT()
struct FGameFeatureAbilitiesEntry
{
    GENERATED_BODY()

    // 目标 Actor 类型（如 ALyraPlayerState）
    UPROPERTY(EditAnywhere, Category="Abilities")
    TSoftClassPtr<AActor> ActorClass;

    // 要授予的单个技能列表
    UPROPERTY(EditAnywhere, Category="Abilities")
    TArray<FLyraAbilityGrant> GrantedAbilities;

    // 要授予的属性集列表
    UPROPERTY(EditAnywhere, Category="Attributes")
    TArray<FLyraAttributeSetGrant> GrantedAttributes;

    // 要授予的技能集列表（打包多个技能+效果+属性）
    UPROPERTY(EditAnywhere, Category="Attributes", meta=(AssetBundles="Client,Server"))
    TArray<TSoftObjectPtr<const ULyraAbilitySet>> GrantedAbilitySets;
};
```

#### 完整注入流程

**Step 1: GameFeature 激活时注册扩展处理器**

```cpp
void UGameFeatureAction_AddAbilities::AddToWorld(...)
{
    // 对 AbilitiesList 中每个条目，注册一个扩展处理器
    // 当 Entry.ActorClass 类型的 Actor 出现/消失时会被回调
    ComponentMan->AddExtensionHandler(Entry.ActorClass, AddAbilitiesDelegate);
}
```

**Step 2: 收到扩展事件时分发**

```cpp
void UGameFeatureAction_AddAbilities::HandleActorExtension(
    AActor* Actor, FName EventName, int32 EntryIndex, FGameFeatureStateChangeContext ChangeContext)
{
    FPerContextData* ActiveData = ContextData.Find(ChangeContext);
    if (AbilitiesList.IsValidIndex(EntryIndex) && ActiveData)
    {
        const FGameFeatureAbilitiesEntry& Entry = AbilitiesList[EntryIndex];

        // Actor 被移除 → 回收技能
        if ((EventName == UGameFrameworkComponentManager::NAME_ExtensionRemoved) ||
            (EventName == UGameFrameworkComponentManager::NAME_ReceiverRemoved))
        {
            RemoveActorAbilities(Actor, *ActiveData);
        }
        // Actor 被添加 或 技能系统就绪 → 注入技能
        else if ((EventName == UGameFrameworkComponentManager::NAME_ExtensionAdded) ||
                 (EventName == ALyraPlayerState::NAME_LyraAbilityReady))
        {
            AddActorAbilities(Actor, Entry, *ActiveData);
        }
    }
}
```

**Step 3: 实际注入技能和属性**

```cpp
void UGameFeatureAction_AddAbilities::AddActorAbilities(
    AActor* Actor,
    const FGameFeatureAbilitiesEntry& AbilitiesEntry,
    FPerContextData& ActiveData)
{
    check(Actor);
    if (!Actor->HasAuthority()) return;  // 只在服务器端执行

    // 防止重复注入
    if (ActiveData.ActiveExtensions.Find(Actor) != nullptr) return;

    if (UAbilitySystemComponent* ASC = FindOrAddComponentForActor<UAbilitySystemComponent>(
            Actor, AbilitiesEntry, ActiveData))
    {
        FActorExtensions AddedExtensions;

        // 1. 注入单个技能
        for (const FLyraAbilityGrant& Ability : AbilitiesEntry.GrantedAbilities)
        {
            if (!Ability.AbilityType.IsNull())
            {
                FGameplayAbilitySpec NewAbilitySpec(Ability.AbilityType.LoadSynchronous());
                FGameplayAbilitySpecHandle AbilityHandle = ASC->GiveAbility(NewAbilitySpec);
                AddedExtensions.Abilities.Add(AbilityHandle);
            }
        }

        // 2. 注入属性集
        for (const FLyraAttributeSetGrant& Attributes : AbilitiesEntry.GrantedAttributes)
        {
            if (!Attributes.AttributeSetType.IsNull())
            {
                TSubclassOf<UAttributeSet> SetType = Attributes.AttributeSetType.LoadSynchronous();
                if (SetType)
                {
                    UAttributeSet* NewSet = NewObject<UAttributeSet>(ASC->GetOwner(), SetType);
                    if (!Attributes.InitializationData.IsNull())
                    {
                        UDataTable* InitData = Attributes.InitializationData.LoadSynchronous();
                        if (InitData) NewSet->InitFromMetaDataTable(InitData);
                    }
                    AddedExtensions.Attributes.Add(NewSet);
                    ASC->AddAttributeSetSubobject(NewSet);
                }
            }
        }

        // 3. 注入技能集（AbilitySet = 多个技能+效果+属性的打包）
        ULyraAbilitySystemComponent* LyraASC = CastChecked<ULyraAbilitySystemComponent>(ASC);
        for (const TSoftObjectPtr<const ULyraAbilitySet>& SetPtr : AbilitiesEntry.GrantedAbilitySets)
        {
            if (const ULyraAbilitySet* Set = SetPtr.Get())
            {
                Set->GiveToAbilitySystem(LyraASC,
                    &AddedExtensions.AbilitySetHandles.AddDefaulted_GetRef());
            }
        }

        // 记录已注入的扩展，用于后续清理
        ActiveData.ActiveExtensions.Add(Actor, AddedExtensions);
    }
}
```

**Step 4: GameFeature 停用时清理**

```cpp
void UGameFeatureAction_AddAbilities::RemoveActorAbilities(AActor* Actor, FPerContextData& ActiveData)
{
    if (FActorExtensions* ActorExtensions = ActiveData.ActiveExtensions.Find(Actor))
    {
        if (UAbilitySystemComponent* ASC = Actor->FindComponentByClass<UAbilitySystemComponent>())
        {
            // 移除属性集
            for (UAttributeSet* AttribSetInstance : ActorExtensions->Attributes)
            {
                ASC->RemoveSpawnedAttribute(AttribSetInstance);
            }

            // 移除技能
            for (FGameplayAbilitySpecHandle AbilityHandle : ActorExtensions->Abilities)
            {
                ASC->SetRemoveAbilityOnEnd(AbilityHandle);
            }

            // 移除技能集
            ULyraAbilitySystemComponent* LyraASC = CastChecked<ULyraAbilitySystemComponent>(ASC);
            for (FLyraAbilitySet_GrantedHandles& SetHandle : ActorExtensions->AbilitySetHandles)
            {
                SetHandle.TakeFromAbilitySystem(LyraASC);
            }
        }
        ActiveData.ActiveExtensions.Remove(Actor);
    }
}
```

### 4.4 GameFeatureAction_AddInputBinding — 输入绑定注入

```cpp
// Source/LyraGame/GameFeatures/GameFeatureAction_AddInputBinding.cpp
void UGameFeatureAction_AddInputBinding::AddToWorld(...)
{
    if (UGameFrameworkComponentManager* ComponentManager = ...)
    {
        // 监听所有 APawn 类型 Actor 的扩展事件
        UGameFrameworkComponentManager::FExtensionHandlerDelegate AddAbilitiesDelegate =
            UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
                this, &ThisClass::HandlePawnExtension, ChangeContext);

        TSharedPtr<FComponentRequestHandle> ExtensionRequestHandle =
            ComponentManager->AddExtensionHandler(APawn::StaticClass(), AddAbilitiesDelegate);

        ActiveData.ExtensionRequestHandles.Add(ExtensionRequestHandle);
    }
}

void UGameFeatureAction_AddInputBinding::HandlePawnExtension(
    AActor* Actor, FName EventName, FGameFeatureStateChangeContext ChangeContext)
{
    APawn* AsPawn = CastChecked<APawn>(Actor);

    if (EventName == UGameFrameworkComponentManager::NAME_ExtensionRemoved ||
        EventName == UGameFrameworkComponentManager::NAME_ReceiverRemoved)
    {
        RemoveInputMapping(AsPawn, ActiveData);
    }
    // 关键：监听 ULyraHeroComponent::NAME_BindInputsNow 自定义事件
    else if (EventName == UGameFrameworkComponentManager::NAME_ExtensionAdded ||
             EventName == ULyraHeroComponent::NAME_BindInputsNow)
    {
        AddInputMappingForPlayer(AsPawn, ActiveData);
    }
}

void UGameFeatureAction_AddInputBinding::AddInputMappingForPlayer(APawn* Pawn, FPerContextData& ActiveData)
{
    // 找到 HeroComponent，检查是否已准备好绑定输入
    ULyraHeroComponent* HeroComponent = Pawn->FindComponentByClass<ULyraHeroComponent>();
    if (HeroComponent && HeroComponent->IsReadyToBindInputs())
    {
        for (const TSoftObjectPtr<const ULyraInputConfig>& Entry : InputConfigs)
        {
            if (const ULyraInputConfig* BindSet = Entry.Get())
            {
                // 通过 HeroComponent 添加额外的输入配置
                HeroComponent->AddAdditionalInputConfig(BindSet);
            }
        }
    }
}
```

### 4.5 GameFeatureAction_AddWidget — UI Widget 注入

```cpp
// Source/LyraGame/GameFeatures/GameFeatureAction_AddWidget.cpp
void UGameFeatureAction_AddWidgets::AddToWorld(...)
{
    if (UGameFrameworkComponentManager* ComponentManager = ...)
    {
        // 监听 ALyraHUD 类型 Actor 的扩展事件
        TSoftClassPtr<AActor> HUDActorClass = ALyraHUD::StaticClass();
        TSharedPtr<FComponentRequestHandle> ExtensionRequestHandle =
            ComponentManager->AddExtensionHandler(
                HUDActorClass,
                UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
                    this, &ThisClass::HandleActorExtension, ChangeContext));
    }
}

void UGameFeatureAction_AddWidgets::AddWidgets(AActor* Actor, FPerContextData& ActiveData)
{
    ALyraHUD* HUD = CastChecked<ALyraHUD>(Actor);

    if (ULocalPlayer* LocalPlayer = Cast<ULocalPlayer>(HUD->GetOwningPlayerController()->Player))
    {
        FPerActorData& ActorData = ActiveData.ActorData.FindOrAdd(HUD);

        // 1. 推入布局 Widget 到指定 UI Layer
        for (const FLyraHUDLayoutRequest& Entry : Layout)
        {
            if (TSubclassOf<UCommonActivatableWidget> ConcreteWidgetClass = Entry.LayoutClass.Get())
            {
                ActorData.LayoutsAdded.Add(
                    UCommonUIExtensions::PushContentToLayer_ForPlayer(
                        LocalPlayer, Entry.LayerID, ConcreteWidgetClass));
            }
        }

        // 2. 注册 HUD 元素 Widget 到指定扩展点（SlotID）
        UUIExtensionSubsystem* ExtensionSubsystem = HUD->GetWorld()->GetSubsystem<UUIExtensionSubsystem>();
        for (const FLyraHUDElementEntry& Entry : Widgets)
        {
            ActorData.ExtensionHandles.Add(
                ExtensionSubsystem->RegisterExtensionAsWidgetForContext(
                    Entry.SlotID, LocalPlayer, Entry.WidgetClass.Get(), -1));
        }
    }
}
```

---

## 五、Experience 加载 — GameFeature 动态激活的驱动者

Experience 系统是驱动 GameFeature 插件激活的上层机制。

### 5.1 完整加载流程（代码级）

```
ALyraGameMode::InitGame()
    │
    ▼
ALyraGameMode::HandleMatchAssignmentIfNotExpectingOne()
    │  确定 ExperienceId（URL参数 > 开发设置 > 命令行 > WorldSettings > 默认）
    ▼
ALyraGameMode::OnMatchAssignmentGiven(ExperienceId, Source)
    │
    ▼
ULyraExperienceManagerComponent::SetCurrentExperience(ExperienceId)
    │  加载 ExperienceDefinition 资产
    ▼
ULyraExperienceManagerComponent::StartExperienceLoad()
    │  ① 异步加载 Experience 资产包（BundleAssetList）
    │  ② 根据 Client/Server 模式选择性加载
    ▼
ULyraExperienceManagerComponent::OnExperienceLoadComplete()
    │  ① 收集所有需要的 GameFeature 插件 URL
    │  ② 调用 UGameFeaturesSubsystem::LoadAndActivateGameFeaturePlugin()
    ▼
(等待所有 GameFeature 插件加载完成)
    │
    ▼
ULyraExperienceManagerComponent::OnExperienceFullLoadCompleted()
    │  ① 执行 Experience 的所有 GameFeatureAction
    │  ② 执行 ActionSet 的所有 GameFeatureAction
    │  ③ 广播 OnExperienceLoaded 委托
    ▼
各子系统收到 OnExperienceLoaded 通知，完成初始化
```

### 5.2 核心代码 — GameFeatureAction 的激活

```cpp
// Source/LyraGame/GameModes/LyraExperienceManagerComponent.cpp
void ULyraExperienceManagerComponent::OnExperienceFullLoadCompleted()
{
    // ...
    LoadState = ELyraExperienceLoadState::ExecutingActions;

    FGameFeatureActivatingContext Context;
    const FWorldContext* ExistingWorldContext = GEngine->GetWorldContextFromWorld(GetWorld());
    if (ExistingWorldContext)
    {
        Context.SetRequiredWorldContextHandle(ExistingWorldContext->ContextHandle);
    }

    // 激活所有 Action（统一走 GameFeature 的生命周期）
    auto ActivateListOfActions = [&Context](const TArray<UGameFeatureAction*>& ActionList)
    {
        for (UGameFeatureAction* Action : ActionList)
        {
            if (Action != nullptr)
            {
                Action->OnGameFeatureRegistering();    // 注册阶段
                Action->OnGameFeatureLoading();        // 加载阶段
                Action->OnGameFeatureActivating(Context); // 激活阶段
            }
        }
    };

    // 激活 Experience 自身的 Actions
    ActivateListOfActions(CurrentExperience->Actions);

    // 激活所有 ActionSet 中的 Actions
    for (const TObjectPtr<ULyraExperienceActionSet>& ActionSet : CurrentExperience->ActionSets)
    {
        if (ActionSet != nullptr)
        {
            ActivateListOfActions(ActionSet->Actions);
        }
    }

    LoadState = ELyraExperienceLoadState::Loaded;

    // 按优先级广播加载完成
    OnExperienceLoaded_HighPriority.Broadcast(CurrentExperience);
    OnExperienceLoaded.Broadcast(CurrentExperience);
    OnExperienceLoaded_LowPriority.Broadcast(CurrentExperience);
}
```

### 5.3 核心代码 — GameFeature 插件的加载与激活

```cpp
// Source/LyraGame/GameModes/LyraExperienceManagerComponent.cpp
void ULyraExperienceManagerComponent::OnExperienceLoadComplete()
{
    // 收集所有需要启用的 GameFeature 插件 URL
    auto CollectGameFeaturePluginURLs = [This=this](
        const UPrimaryDataAsset* Context,
        const TArray<FString>& FeaturePluginList)
    {
        for (const FString& PluginName : FeaturePluginList)
        {
            FString PluginURL;
            if (UGameFeaturesSubsystem::Get().GetPluginURLByName(PluginName, PluginURL))
            {
                This->GameFeaturePluginURLs.AddUnique(PluginURL);
            }
        }
    };

    // 从 Experience 和 ActionSet 中收集
    CollectGameFeaturePluginURLs(CurrentExperience, CurrentExperience->GameFeaturesToEnable);
    for (const TObjectPtr<ULyraExperienceActionSet>& ActionSet : CurrentExperience->ActionSets)
    {
        if (ActionSet != nullptr)
        {
            CollectGameFeaturePluginURLs(ActionSet, ActionSet->GameFeaturesToEnable);
        }
    }

    // 异步加载并激活每个 GameFeature 插件
    NumGameFeaturePluginsLoading = GameFeaturePluginURLs.Num();
    if (NumGameFeaturePluginsLoading > 0)
    {
        LoadState = ELyraExperienceLoadState::LoadingGameFeatures;
        for (const FString& PluginURL : GameFeaturePluginURLs)
        {
            UGameFeaturesSubsystem::Get().LoadAndActivateGameFeaturePlugin(
                PluginURL,
                FGameFeaturePluginLoadComplete::CreateUObject(
                    this, &ThisClass::OnGameFeaturePluginLoadComplete));
        }
    }
}
```

### 5.4 核心代码 — GameFeature 停用（EndPlay）

```cpp
// Source/LyraGame/GameModes/LyraExperienceManagerComponent.cpp
void ULyraExperienceManagerComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    Super::EndPlay(EndPlayReason);

    // 停用 GameFeature 插件
    for (const FString& PluginURL : GameFeaturePluginURLs)
    {
        if (ULyraExperienceManager::RequestToDeactivatePlugin(PluginURL))
        {
            UGameFeaturesSubsystem::Get().DeactivateGameFeaturePlugin(PluginURL);
        }
    }

    // 停用所有 Action
    if (LoadState == ELyraExperienceLoadState::Loaded)
    {
        LoadState = ELyraExperienceLoadState::Deactivating;

        FGameFeatureDeactivatingContext Context(...);

        auto DeactivateListOfActions = [&Context](const TArray<UGameFeatureAction*>& ActionList)
        {
            for (UGameFeatureAction* Action : ActionList)
            {
                if (Action)
                {
                    Action->OnGameFeatureDeactivating(Context);  // 停用
                    Action->OnGameFeatureUnregistering();         // 注销
                }
            }
        };

        DeactivateListOfActions(CurrentExperience->Actions);
        for (const TObjectPtr<ULyraExperienceActionSet>& ActionSet : CurrentExperience->ActionSets)
        {
            if (ActionSet != nullptr)
            {
                DeactivateListOfActions(ActionSet->Actions);
            }
        }
    }
}
```

---

## 六、Init State 系统 — 组件间协调初始化

### 6.1 状态链

```
(无状态) → InitState_Spawned → InitState_DataAvailable → InitState_DataInitialized → InitState_GameplayReady
```

### 6.2 ULyraPawnExtensionComponent — 协调者

```cpp
// Source/LyraGame/Character/LyraPawnExtensionComponent.cpp
bool ULyraPawnExtensionComponent::CanChangeInitState(
    UGameFrameworkComponentManager* Manager,
    FGameplayTag CurrentState,
    FGameplayTag DesiredState) const
{
    APawn* Pawn = GetPawn<APawn>();

    if (!CurrentState.IsValid() && DesiredState == LyraGameplayTags::InitState_Spawned)
    {
        // 只要 Pawn 存在就可以标记为已生成
        return Pawn != nullptr;
    }
    if (CurrentState == InitState_Spawned && DesiredState == InitState_DataAvailable)
    {
        // 需要 PawnData 已设置 + 有控制器
        return PawnData != nullptr && GetController<AController>() != nullptr;
    }
    else if (CurrentState == InitState_DataAvailable && DesiredState == InitState_DataInitialized)
    {
        // 所有注册的 Feature 都必须到达 DataAvailable 状态
        return Manager->HaveAllFeaturesReachedInitState(
            Pawn, LyraGameplayTags::InitState_DataAvailable);
    }
    else if (CurrentState == InitState_DataInitialized && DesiredState == InitState_GameplayReady)
    {
        return true;
    }
    return false;
}
```

### 6.3 ULyraHeroComponent — 响应者

```cpp
// Source/LyraGame/Character/LyraHeroComponent.cpp
void ULyraHeroComponent::HandleChangeInitState(
    UGameFrameworkComponentManager* Manager,
    FGameplayTag CurrentState,
    FGameplayTag DesiredState)
{
    if (CurrentState == InitState_DataAvailable && DesiredState == InitState_DataInitialized)
    {
        APawn* Pawn = GetPawn<APawn>();
        ALyraPlayerState* LyraPS = GetPlayerState<ALyraPlayerState>();

        if (ULyraPawnExtensionComponent* PawnExtComp =
            ULyraPawnExtensionComponent::FindPawnExtensionComponent(Pawn))
        {
            PawnData = PawnExtComp->GetPawnData<ULyraPawnData>();

            // 初始化技能系统：将 PlayerState 的 ASC 与当前 Pawn 关联
            PawnExtComp->InitializeAbilitySystem(
                LyraPS->GetLyraAbilitySystemComponent(), LyraPS);
        }

        // 初始化玩家输入
        if (ALyraPlayerController* LyraPC = GetController<ALyraPlayerController>())
        {
            if (Pawn->InputComponent != nullptr)
            {
                InitializePlayerInput(Pawn->InputComponent);
            }
        }

        // 绑定相机模式
        if (PawnData)
        {
            if (ULyraCameraComponent* CameraComponent =
                ULyraCameraComponent::FindCameraComponent(Pawn))
            {
                CameraComponent->DetermineCameraModeDelegate.BindUObject(
                    this, &ThisClass::DetermineCameraMode);
            }
        }
    }
}
```

### 6.4 HeroComponent 发出 BindInputsNow 事件

```cpp
// Source/LyraGame/Character/LyraHeroComponent.cpp
void ULyraHeroComponent::InitializePlayerInput(UInputComponent* PlayerInputComponent)
{
    // ... 绑定基础输入 ...

    bReadyToBindInputs = true;

    // 发送扩展事件，通知所有监听的 GameFeatureAction（如 AddInputBinding）
    // 此刻 GameFeature 插件可以注入额外的输入绑定
    UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(
        const_cast<APlayerController*>(PC), NAME_BindInputsNow);
    UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(
        const_cast<APawn*>(Pawn), NAME_BindInputsNow);
}
```

---

## 七、LyraGameFeaturePolicy — 插件策略管理

```cpp
// Source/LyraGame/GameFeatures/LyraGameFeaturePolicy.cpp
void ULyraGameFeaturePolicy::InitGameFeatureManager()
{
    // 注册全局观察者：监听所有 GameFeature 插件的状态变化
    Observers.Add(NewObject<ULyraGameFeature_HotfixManager>());
    Observers.Add(NewObject<ULyraGameFeature_AddGameplayCuePaths>());

    UGameFeaturesSubsystem& Subsystem = UGameFeaturesSubsystem::Get();
    for (UObject* Observer : Observers)
    {
        Subsystem.AddObserver(Observer,
            UGameFeaturesSubsystem::EObserverPluginStateUpdateMode::CurrentAndFuture);
    }

    Super::InitGameFeatureManager();
}

// 根据运行模式决定加载哪些数据
void ULyraGameFeaturePolicy::GetGameFeatureLoadingMode(
    bool& bLoadClientData, bool& bLoadServerData) const
{
    bLoadClientData = !IsRunningDedicatedServer();
    bLoadServerData = !IsRunningClientOnly();
}
```

**GameplayCue 路径动态注入**：

```cpp
void ULyraGameFeature_AddGameplayCuePaths::OnGameFeatureRegistering(
    const UGameFeatureData* GameFeatureData,
    const FString& PluginName,
    const FString& PluginURL)
{
    const FString PluginRootPath = TEXT("/") + PluginName;
    for (const UGameFeatureAction* Action : GameFeatureData->GetActions())
    {
        if (const UGameFeatureAction_AddGameplayCuePath* AddGameplayCueGFA =
            Cast<UGameFeatureAction_AddGameplayCuePath>(Action))
        {
            const TArray<FDirectoryPath>& DirsToAdd = AddGameplayCueGFA->GetDirectoryPathsToAdd();

            if (ULyraGameplayCueManager* GCM = ULyraGameplayCueManager::Get())
            {
                for (const FDirectoryPath& Directory : DirsToAdd)
                {
                    FString MutablePath = Directory.Path;
                    // 修复插件包路径
                    UGameFeaturesSubsystem::FixPluginPackagePath(
                        MutablePath, PluginRootPath, false);
                    GCM->AddGameplayCueNotifyPath(MutablePath, false);
                }

                if (!DirsToAdd.IsEmpty())
                {
                    GCM->InitializeRuntimeObjectLibrary(); // 重建运行时库
                }
            }
        }
    }
}
```

---

## 八、完整数据流时序图

```
┌──────────┐     ┌──────────────┐     ┌──────────────────┐     ┌───────────────┐
│GameMode  │     │ExperienceMgr │     │GameFeatureSubsys │     │ComponentMgr   │
└────┬─────┘     └──────┬───────┘     └────────┬─────────┘     └───────┬───────┘
     │                  │                      │                       │
     │ SetCurrentExp()  │                      │                       │
     │─────────────────>│                      │                       │
     │                  │                      │                       │
     │                  │ StartExperienceLoad()│                       │
     │                  │──────┐               │                       │
     │                  │      │ 异步加载资产   │                       │
     │                  │<─────┘               │                       │
     │                  │                      │                       │
     │                  │ LoadAndActivate()     │                       │
     │                  │─────────────────────>│                       │
     │                  │                      │ 加载插件              │
     │                  │                      │──────┐                │
     │                  │                      │<─────┘                │
     │                  │ PluginLoadComplete   │                       │
     │                  │<─────────────────────│                       │
     │                  │                      │                       │
     │                  │ ActivateListOfActions│                       │
     │                  │──────┐               │                       │
     │                  │      │               │                       │
     │                  │   Action.OnGameFeatureActivating(ctx)        │
     │                  │      │──────────────────────────────────────>│
     │                  │      │               │  AddExtensionHandler  │
     │                  │      │               │                       │──────┐
     │                  │      │               │                       │      │ 注册监听
     │                  │<─────┘               │                       │<─────┘
     │                  │                      │                       │
     │                  │ Broadcast Loaded     │                       │
     │                  │─────┐                │                       │
     │ OnExperienceLoaded    │                │                       │
     │<──────────────────────┘                │                       │
     │                  │                      │                       │
     │ RestartPlayer()  │                      │                       │
     │──────┐           │                      │                       │
     │      │ Spawn Pawn│                      │                       │
     │<─────┘           │                      │                       │
     │                  │                      │                       │
     │                  │                      │         Pawn 创建事件  │
     │                  │                      │                       │──────┐
     │                  │                      │                       │      │
     │                  │                      │                       │  回调所有注册的
     │                  │                      │                       │  ExtensionHandler
     │                  │                      │                       │      │
     │                  │                      │     ┌─────────────────│<─────┘
     │                  │                      │     │                 │
     │                  │                      │     ▼                 │
     │                  │                   AddAbilities/             │
     │                  │                   AddInputBinding/          │
     │                  │                   AddWidget                 │
     │                  │                      │                       │
```

---

## 九、设计模式总结

| 设计模式 | 实现方式 | 在 Lyra 中的体现 |
|----------|----------|------------------|
| **观察者模式** | `UGameFrameworkComponentManager` + Extension Handler | GameFeatureAction 监听 Actor 创建/销毁事件 |
| **策略模式** | `UGameFeatureAction` 多态 | 不同 Action 子类实现不同的注入策略 |
| **中介者模式** | `ULyraExperienceManagerComponent` | 协调 Experience、GameFeature、Action 三者的生命周期 |
| **状态机模式** | `IGameFrameworkInitStateInterface` | 组件间通过状态链协调初始化顺序 |
| **组合模式** | `ULyraExperienceActionSet` | Action 可以打包成集合，复用于多个 Experience |
| **依赖注入** | `AddExtensionHandler` + `AddComponentRequest` | 运行时动态为 Actor 注入组件和功能 |
| **引用计数** | `FComponentRequestHandle` (TSharedPtr) | Handle 释放后自动清理注入的组件 |

---

## 十、关键自定义事件名总结

| 事件名 | 定义位置 | 触发时机 | 监听者 |
|--------|----------|----------|--------|
| `NAME_LyraAbilityReady` | `ALyraPlayerState` | PawnData 设置完成、技能授予完成 | `GameFeatureAction_AddAbilities` |
| `NAME_BindInputsNow` | `ULyraHeroComponent` | 玩家输入绑定完成 | `GameFeatureAction_AddInputBinding` |
| `NAME_ActorFeatureName("Hero")` | `ULyraHeroComponent` | Init State 状态变化 | `ULyraPawnExtensionComponent` |
| `NAME_ActorFeatureName("PawnExtension")` | `ULyraPawnExtensionComponent` | Init State 状态变化 | `ULyraHeroComponent` |
| `NAME_ExtensionAdded` | 引擎 `UGameFrameworkComponentManager` | Actor 注册到 ComponentManager | 所有 GameFeatureAction |
| `NAME_ExtensionRemoved` | 引擎 `UGameFrameworkComponentManager` | Actor 从 ComponentManager 移除 | 所有 GameFeatureAction |
| `NAME_GameActorReady` | 引擎 `UGameFrameworkComponentManager` | Modular 基类 PreInitializeComponents | `GameFeatureAction_AddWidget` |

---

这套机制使得 Lyra 实现了真正的**功能热插拔**：游戏模式（如射击、俯视角竞技场）作为独立的 GameFeature 插件存在，通过 Experience 数据资产来组合使用，无需修改核心代码即可为 Actor 动态注入技能、输入、UI 等全套功能。
