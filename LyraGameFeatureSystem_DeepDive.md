# Lyra GameFeature 插件体系深度分析

## 目录
1. [系统概述与架构总览](#1-系统概述与架构总览)
2. [ModularGameplayActors 插件 — 组件注入基础设施](#2-modulargameplayactors-插件--组件注入基础设施)
3. [Experience 系统 — GameFeature 的驱动核心](#3-experience-系统--gamefeature-的驱动核心)
4. [GameFeatureAction 基类体系](#4-gamefeatureaction-基类体系)
5. [GameFeatureAction_AddAbilities — 能力注入](#5-gamefeatureaction_addabilities--能力注入)
6. [GameFeatureAction_AddInputBinding — 输入绑定注入](#6-gamefeatureaction_addinputbinding--输入绑定注入)
7. [GameFeatureAction_AddInputContextMapping — 输入映射注入](#7-gamefeatureaction_addinputcontextmapping--输入映射注入)
8. [GameFeatureAction_AddWidgets — UI注入](#8-gamefeatureaction_addwidgets--ui注入)
9. [GameFeatureAction_AddGameplayCuePath — GameplayCue路径注入](#9-gamefeatureaction_addgameplaycuepath--gameplaycue路径注入)
10. [GameFeatureAction_SplitscreenConfig — 分屏配置](#10-gamefeatureaction_splitscreenconfig--分屏配置)
11. [ApplyFrontendPerfSettingsAction — 前端性能设置](#11-applyfrontendperfsettingsaction--前端性能设置)
12. [LyraGameFeaturePolicy — 加载策略与观察者](#12-lyragamefeaturepolicy--加载策略与观察者)
13. [具体GameFeature插件实例分析](#13-具体gamefeature插件实例分析)
14. [完整架构图与核心设计模式总结](#14-完整架构图与核心设计模式总结)

---

## 1. 系统概述与架构总览

### 1.1 Lyra的模块化哲学

Lyra项目的核心哲学是**"一切皆可插拔"**。游戏的功能不是硬编码在单一工程中，而是通过 GameFeature 插件动态组合。一个"射击游戏"和一个"俯视竞技场"可以共享同一套核心框架，仅通过加载不同的 GameFeature 插件来切换玩法。

### 1.2 四层架构

```
┌─────────────────────────────────────────────────────────────────┐
│  GameFeature 插件实例层 (Plugins/GameFeatures/)                  │
│  ShooterCore / ShooterMaps / ShooterExplorer / TopDownArena     │
├─────────────────────────────────────────────────────────────────┤
│  自定义 GameFeatureAction 层 (Source/LyraGame/GameFeatures/)     │
│  AddAbilities / AddInputBinding / AddWidget / AddIMC / ...      │
├─────────────────────────────────────────────────────────────────┤
│  Experience 驱动层 (Source/LyraGame/GameModes/)                  │
│  ExperienceDefinition → ExperienceManagerComponent              │
├─────────────────────────────────────────────────────────────────┤
│  模块化基础设施层                                                 │
│  ModularGameplayActors + UGameFrameworkComponentManager (引擎)   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 核心运行流程

```
地图加载 → ALyraGameMode::InitGame
  → HandleMatchAssignmentIfNotExpectingOne()
    → 确定 ExperienceId（URL > 开发者设置 > 命令行 > WorldSettings > 默认）
      → ExperienceManagerComponent::SetCurrentExperience(ExperienceId)
        → StartExperienceLoad()
          → 1. 异步加载 Experience 资产和 ActionSet 资产
          → 2. 收集 GameFeaturesToEnable URL 列表
          → 3. LoadAndActivateGameFeaturePlugin() 逐个加载激活
          → 4. 所有插件加载完成后执行 Actions
            → Action::OnGameFeatureRegistering()
            → Action::OnGameFeatureLoading()
            → Action::OnGameFeatureActivating(Context)
          → 5. 广播 OnExperienceLoaded 委托
            → GameMode 开始生成玩家 Pawn
            → 各系统组件初始化
```

---

## 2. ModularGameplayActors 插件 — 组件注入基础设施

### 2.1 概述

**位置**: `Plugins/ModularGameplayActors/`

该插件提供一组**模块化的基础 Actor 类**，它们在生命周期的关键节点调用 `UGameFrameworkComponentManager`，从而允许 GameFeature 动态注入组件。

### 2.2 Modular Actor 三步协议

每个 Modular Actor 遵循相同的三步模式：

```cpp
// 以 AModularCharacter 为例
// 文件: Plugins/ModularGameplayActors/Source/.../ModularCharacter.cpp

void AModularCharacter::PreInitializeComponents()
{
    Super::PreInitializeComponents();
    // 步骤1: 注册为组件接收者
    UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver(this);
}

void AModularCharacter::BeginPlay()
{
    // 步骤2: 发送"Actor就绪"事件，触发已注册的 ExtensionHandler
    UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(
        this, UGameFrameworkComponentManager::NAME_GameActorReady);
    Super::BeginPlay();
}

void AModularCharacter::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    // 步骤3: 注销接收者
    UGameFrameworkComponentManager::RemoveGameFrameworkComponentReceiver(this);
    Super::EndPlay(EndPlayReason);
}
```

### 2.3 完整的 Modular Actor 家族

| 类名 | 基类 | 额外行为 |
|------|------|----------|
| `AModularCharacter` | `ACharacter` | 标准三步协议 |
| `AModularPawn` | `APawn` | 标准三步协议 |
| `AModularPlayerController` | `APlayerController` | 三步协议 + `ReceivedPlayer`时发送Ready + 转发`PlayerTick`到`ControllerComponent` |
| `AModularPlayerState` | `APlayerState` | 三步协议 + 转发`Reset`和`CopyProperties`到`PlayerStateComponent` |
| `AModularGameStateBase` | `AGameStateBase` | 标准三步协议 |
| `AModularGameState` | `AGameState` | 三步协议 + 转发`HandleMatchHasStarted`到`GameStateComponent` |
| `AModularGameModeBase` | `AGameModeBase` | 设置默认Modular类 |
| `AModularGameMode` | `AGameMode` | 设置默认Modular类 |
| `AModularAIController` | `AAIController` | 标准三步协议 |

### 2.4 PlayerController 的特殊处理

`AModularPlayerController` 在 `ReceivedPlayer` 而非 `BeginPlay` 中发送 `GameActorReady`，因为 PlayerController 需要等到 Player 分配完毕才真正可用：

```cpp
void AModularPlayerController::ReceivedPlayer()
{
    UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(
        this, UGameFrameworkComponentManager::NAME_GameActorReady);
    Super::ReceivedPlayer();

    // 转发到所有模块化组件
    TArray<UControllerComponent*> ModularComponents;
    GetComponents(ModularComponents);
    for (UControllerComponent* Component : ModularComponents)
    {
        Component->ReceivedPlayer();
    }
}
```

### 2.5 Lyra 对 Modular Actor 的继承

```
AModularGameModeBase
  └── ALyraGameMode (设置 LyraGameState / LyraPlayerController / LyraHUD 等)

AModularGameStateBase → AModularGameState
  └── ALyraGameState

AModularPlayerController
  └── ALyraPlayerController

AModularPlayerState
  └── ALyraPlayerState

AModularCharacter
  └── ALyraCharacter
```

### 2.6 UGameFrameworkComponentManager 的工作原理

`UGameFrameworkComponentManager`（引擎提供的 `UGameInstanceSubsystem`）是组件注入的核心引擎：

```
GameFeature 激活时:
  → Action 调用 ComponentManager->AddExtensionHandler(ActorClass, Delegate)
    → ComponentManager 记录：对于 ActorClass 类型的 Actor，执行此 Delegate
    → 如果已有该类型的 Actor 存在，立即回调

Modular Actor 创建时:
  → PreInitializeComponents: AddGameFrameworkComponentReceiver(this)
    → ComponentManager 为该 Actor 执行所有已注册的 ComponentRequest（动态添加组件）
  → BeginPlay: SendGameFrameworkComponentExtensionEvent(this, NAME_GameActorReady)
    → ComponentManager 为该 Actor 触发所有已注册的 ExtensionHandler
```

---

## 3. Experience 系统 — GameFeature 的驱动核心

### 3.1 ULyraExperienceDefinition

**文件**: `Source/LyraGame/GameModes/LyraExperienceDefinition.h/.cpp`

Experience 是 Lyra 中组织游戏玩法的核心数据资产：

```cpp
UCLASS(BlueprintType, Const)
class ULyraExperienceDefinition : public UPrimaryDataAsset
{
    // 需要激活的 GameFeature 插件名列表
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TArray<FString> GameFeaturesToEnable;

    // 默认 Pawn 数据
    UPROPERTY(EditDefaultsOnly, Category=Gameplay)
    TObjectPtr<const ULyraPawnData> DefaultPawnData;

    // 直接内联的 Action 列表（不需要 GameFeature 插件的 Action）
    UPROPERTY(EditDefaultsOnly, Instanced, Category="Actions")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;

    // 组合的 ActionSet 列表（可复用的 Action 集合）
    UPROPERTY(EditDefaultsOnly, Category=Gameplay)
    TArray<TObjectPtr<ULyraExperienceActionSet>> ActionSets;
};
```

**关键设计点**：
- `GameFeaturesToEnable` 指定需要加载的插件（如 `"ShooterCore"`, `"ShooterMaps"`）
- `Actions` 是直接内联在 Experience 中的 `UGameFeatureAction` 实例
- `ActionSets` 是可复用的 Action 集合，支持组合模式
- 禁止蓝图二次继承（通过 `IsDataValid` 验证）

### 3.2 ULyraExperienceActionSet

**文件**: `Source/LyraGame/GameModes/LyraExperienceActionSet.h/.cpp`

可复用的 Action 集合，多个 Experience 可以共享同一个 ActionSet：

```cpp
UCLASS(BlueprintType, NotBlueprintable)
class ULyraExperienceActionSet : public UPrimaryDataAsset
{
    // Action 列表
    UPROPERTY(EditAnywhere, Instanced, Category="Actions to Perform")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;

    // 额外的 GameFeature 插件依赖
    UPROPERTY(EditAnywhere, Category="Feature Dependencies")
    TArray<FString> GameFeaturesToEnable;
};
```

### 3.3 ULyraExperienceManagerComponent

**文件**: `Source/LyraGame/GameModes/LyraExperienceManagerComponent.h/.cpp`

Experience 加载的状态机，挂在 `GameState` 上，是整个 GameFeature 系统的心脏：

#### 状态机

```cpp
enum class ELyraExperienceLoadState
{
    Unloaded,                   // 初始状态
    Loading,                    // 加载 Experience 资产
    LoadingGameFeatures,        // 加载 GameFeature 插件
    LoadingChaosTestingDelay,   // 混沌测试延迟（可选）
    ExecutingActions,           // 执行 Action
    Loaded,                     // 完成
    Deactivating                // 停用中（EndPlay）
};
```

#### 加载流程详细分析

**阶段1: 设置 Experience (服务器)**

```cpp
void ULyraExperienceManagerComponent::SetCurrentExperience(FPrimaryAssetId ExperienceId)
{
    // 通过 AssetManager 加载 Experience CDO
    ULyraAssetManager& AssetManager = ULyraAssetManager::Get();
    FSoftObjectPath AssetPath = AssetManager.GetPrimaryAssetPath(ExperienceId);
    TSubclassOf<ULyraExperienceDefinition> AssetClass = Cast<UClass>(AssetPath.TryLoad());
    const ULyraExperienceDefinition* Experience = GetDefault<ULyraExperienceDefinition>(AssetClass);

    CurrentExperience = Experience;  // 网络复制属性
    StartExperienceLoad();
}
```

**阶段2: 异步加载资产**

```cpp
void ULyraExperienceManagerComponent::StartExperienceLoad()
{
    LoadState = ELyraExperienceLoadState::Loading;

    // 1. 收集需要加载的 PrimaryAssetId（Experience 本身 + 所有 ActionSet）
    TSet<FPrimaryAssetId> BundleAssetList;
    BundleAssetList.Add(CurrentExperience->GetPrimaryAssetId());
    for (const auto& ActionSet : CurrentExperience->ActionSets)
    {
        BundleAssetList.Add(ActionSet->GetPrimaryAssetId());
    }

    // 2. 按 Client/Server 加载 AssetBundle
    TArray<FName> BundlesToLoad;
    BundlesToLoad.Add(FLyraBundles::Equipped);
    if (bLoadClient) BundlesToLoad.Add(UGameFeaturesSubsystemSettings::LoadStateClient);
    if (bLoadServer) BundlesToLoad.Add(UGameFeaturesSubsystemSettings::LoadStateServer);

    // 3. 异步加载
    BundleLoadHandle = AssetManager.ChangeBundleStateForPrimaryAssets(
        BundleAssetList.Array(), BundlesToLoad, {}, false, ...);
    // 完成后回调 OnExperienceLoadComplete
}
```

**阶段3: 加载 GameFeature 插件**

```cpp
void ULyraExperienceManagerComponent::OnExperienceLoadComplete()
{
    // 1. 收集所有 GameFeature 插件 URL（Experience + 所有 ActionSet）
    GameFeaturePluginURLs.Reset();
    auto CollectURLs = [this](const UPrimaryDataAsset* Context, const TArray<FString>& FeaturePluginList)
    {
        for (const FString& PluginName : FeaturePluginList)
        {
            FString PluginURL;
            if (UGameFeaturesSubsystem::Get().GetPluginURLByName(PluginName, PluginURL))
            {
                GameFeaturePluginURLs.AddUnique(PluginURL);
            }
        }
    };
    CollectURLs(CurrentExperience, CurrentExperience->GameFeaturesToEnable);
    for (const auto& ActionSet : CurrentExperience->ActionSets)
    {
        CollectURLs(ActionSet, ActionSet->GameFeaturesToEnable);
    }

    // 2. 逐个加载并激活插件
    NumGameFeaturePluginsLoading = GameFeaturePluginURLs.Num();
    LoadState = ELyraExperienceLoadState::LoadingGameFeatures;
    for (const FString& PluginURL : GameFeaturePluginURLs)
    {
        ULyraExperienceManager::NotifyOfPluginActivation(PluginURL);
        UGameFeaturesSubsystem::Get().LoadAndActivateGameFeaturePlugin(
            PluginURL, 
            FGameFeaturePluginLoadComplete::CreateUObject(
                this, &ThisClass::OnGameFeaturePluginLoadComplete));
    }
}
```

**阶段4: 执行 Actions**

```cpp
void ULyraExperienceManagerComponent::OnExperienceFullLoadCompleted()
{
    LoadState = ELyraExperienceLoadState::ExecutingActions;

    // 创建 ActivatingContext，绑定到当前 World
    FGameFeatureActivatingContext Context;
    const FWorldContext* ExistingWorldContext = GEngine->GetWorldContextFromWorld(GetWorld());
    if (ExistingWorldContext)
    {
        Context.SetRequiredWorldContextHandle(ExistingWorldContext->ContextHandle);
    }

    // 对每个 Action 依次调用三个生命周期方法
    auto ActivateListOfActions = [&Context](const TArray<UGameFeatureAction*>& ActionList)
    {
        for (UGameFeatureAction* Action : ActionList)
        {
            if (Action)
            {
                Action->OnGameFeatureRegistering();  // 注册阶段
                Action->OnGameFeatureLoading();       // 加载阶段
                Action->OnGameFeatureActivating(Context); // 激活阶段
            }
        }
    };

    // 先激活 Experience 自身的 Actions，再激活 ActionSet 中的 Actions
    ActivateListOfActions(CurrentExperience->Actions);
    for (const auto& ActionSet : CurrentExperience->ActionSets)
    {
        ActivateListOfActions(ActionSet->Actions);
    }

    // 广播三个优先级的委托
    LoadState = ELyraExperienceLoadState::Loaded;
    OnExperienceLoaded_HighPriority.Broadcast(CurrentExperience);
    OnExperienceLoaded.Broadcast(CurrentExperience);
    OnExperienceLoaded_LowPriority.Broadcast(CurrentExperience);
}
```

**阶段5: 停用清理（EndPlay）**

```cpp
void ULyraExperienceManagerComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    // 1. 停用 GameFeature 插件（引用计数，最后一个使用者才真正停用）
    for (const FString& PluginURL : GameFeaturePluginURLs)
    {
        if (ULyraExperienceManager::RequestToDeactivatePlugin(PluginURL))
        {
            UGameFeaturesSubsystem::Get().DeactivateGameFeaturePlugin(PluginURL);
        }
    }

    // 2. 反向停用所有 Actions
    FGameFeatureDeactivatingContext Context(TEXT(""), 
        [this](FStringView) { this->OnActionDeactivationCompleted(); });

    auto DeactivateListOfActions = [&Context](const TArray<UGameFeatureAction*>& ActionList)
    {
        for (UGameFeatureAction* Action : ActionList)
        {
            if (Action)
            {
                Action->OnGameFeatureDeactivating(Context);
                Action->OnGameFeatureUnregistering();
            }
        }
    };

    DeactivateListOfActions(CurrentExperience->Actions);
    for (const auto& ActionSet : CurrentExperience->ActionSets)
    {
        DeactivateListOfActions(ActionSet->Actions);
    }
}
```

### 3.4 三优先级委托

`CallOrRegister_OnExperienceLoaded` 模式确保不管 Experience 是否已加载都能正确执行：

```cpp
void CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate&& Delegate)
{
    if (IsExperienceLoaded())
    {
        Delegate.Execute(CurrentExperience);  // 已加载，立即执行
    }
    else
    {
        OnExperienceLoaded.Add(MoveTemp(Delegate));  // 未加载，注册回调
    }
}
```

三个优先级：`HighPriority` → `Normal` → `LowPriority`，确保系统初始化顺序。

### 3.5 网络复制

```cpp
UPROPERTY(ReplicatedUsing=OnRep_CurrentExperience)
TObjectPtr<const ULyraExperienceDefinition> CurrentExperience;

void OnRep_CurrentExperience()
{
    StartExperienceLoad();  // 客户端收到复制后也开始加载
}
```

### 3.6 ULyraExperienceManager

**文件**: `Source/LyraGame/GameModes/LyraExperienceManager.h/.cpp`

`UEngineSubsystem`，仅在编辑器模式下管理 PIE 中多 World 的 GameFeature 插件引用计数：

```cpp
UCLASS()
class ULyraExperienceManager : public UEngineSubsystem
{
    // 引用计数 Map: PluginURL → 活跃请求数
    TMap<FString, int32> GameFeaturePluginRequestCountMap;

    // PIE 场景: World A 和 World B 都请求了 ShooterCore
    // NotifyOfPluginActivation("ShooterCore") → Count = 2
    // RequestToDeactivatePlugin("ShooterCore") → Count = 1, return false (不真正停用)
    // RequestToDeactivatePlugin("ShooterCore") → Count = 0, return true (可以停用了)
};
```

### 3.7 AsyncAction_ExperienceReady

**文件**: `Source/LyraGame/GameModes/AsyncAction_ExperienceReady.h/.cpp`

蓝图异步节点，等待 Experience 加载完成：

```cpp
// 蓝图用法: WaitForExperienceReady → OnReady 委托
// 执行步骤:
// Step1: 等待 GameState 创建
// Step2: 获取 ExperienceManagerComponent，检查是否已加载
// Step3: 如果未加载，注册 CallOrRegister_OnExperienceLoaded 回调
// Step4: 广播 OnReady，SetReadyToDestroy
```

### 3.8 Experience 选择优先级

在 `ALyraGameMode::HandleMatchAssignmentIfNotExpectingOne` 中：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 (最高) | URL Options | `?Experience=XXX` |
| 2 | DeveloperSettings | PIE 时的 `ExperienceOverride` |
| 3 | CommandLine | `-Experience=XXX` |
| 4 | WorldSettings | `ALyraWorldSettings::DefaultGameplayExperience` |
| 5 | DedicatedServer | 专服登录流程 |
| 6 (最低) | Default | `B_LyraDefaultExperience` |

### 3.9 ALyraWorldSettings

**文件**: `Source/LyraGame/GameModes/LyraWorldSettings.h/.cpp`

每个地图的默认 Experience 配置：

```cpp
UCLASS()
class ALyraWorldSettings : public AWorldSettings
{
    // 该地图的默认 Experience
    UPROPERTY(EditDefaultsOnly, Category=GameMode)
    TSoftClassPtr<ULyraExperienceDefinition> DefaultGameplayExperience;

    FPrimaryAssetId GetDefaultGameplayExperience() const;

    // PIE 专用: 强制 Standalone 模式（前端关卡用）
    UPROPERTY(EditDefaultsOnly, Category=PIE)
    bool ForceStandaloneNetMode = false;
};
```

---

## 4. GameFeatureAction 基类体系

### 4.1 UGameFeatureAction（引擎基类）

引擎提供的抽象基类，定义了 GameFeature 的完整生命周期：

```
OnGameFeatureRegistering()     ← 插件注册时（最早）
OnGameFeatureLoading()         ← 插件加载时
OnGameFeatureActivating()      ← 插件激活时（最常用）
OnGameFeatureDeactivating()    ← 插件停用时
OnGameFeatureUnregistering()   ← 插件注销时（最晚）
```

### 4.2 UGameFeatureAction_WorldActionBase

**文件**: `Source/LyraGame/GameFeatures/GameFeatureAction_WorldActionBase.h/.cpp`

Lyra 自定义的中间基类，将 Action 限定到特定 World：

```cpp
UCLASS(Abstract)
class UGameFeatureAction_WorldActionBase : public UGameFeatureAction
{
    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) override;
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override;

    // 子类实现: 对每个适用的 World 执行操作
    virtual void AddToWorld(const FWorldContext& WorldContext, 
        const FGameFeatureStateChangeContext& ChangeContext) PURE_VIRTUAL(...);

private:
    TMap<FGameFeatureStateChangeContext, FDelegateHandle> GameInstanceStartHandles;
};
```

**激活逻辑**：

```cpp
void UGameFeatureAction_WorldActionBase::OnGameFeatureActivating(FGameFeatureActivatingContext& Context)
{
    // 1. 注册 GameInstance 启动委托（处理未来创建的 World）
    GameInstanceStartHandles.FindOrAdd(Context) = 
        FWorldDelegates::OnStartGameInstance.AddUObject(this, 
            &ThisClass::HandleGameInstanceStart, FGameFeatureStateChangeContext(Context));

    // 2. 对已存在的 World 立即执行
    for (const FWorldContext& WorldContext : GEngine->GetWorldContexts())
    {
        if (Context.ShouldApplyToWorldContext(WorldContext))
        {
            AddToWorld(WorldContext, Context);
        }
    }
}
```

**关键设计**: `Context.ShouldApplyToWorldContext(WorldContext)` 确保在 PIE 多 World 场景下，Action 只应用到正确的 World。

### 4.3 Lyra 所有自定义 Action 一览

| Action 类 | 基类 | 职责 |
|-----------|------|------|
| `GameFeatureAction_AddAbilities` | `WorldActionBase` | 向 Actor 注入 GAS 能力/属性集 |
| `GameFeatureAction_AddInputBinding` | `WorldActionBase` | 向 Pawn 注入输入配置 |
| `GameFeatureAction_AddInputContextMapping` | `WorldActionBase` | 向 PlayerController 注入 InputMappingContext |
| `GameFeatureAction_AddWidgets` | `WorldActionBase` | 向 HUD 注入 Layout 和 Widget |
| `GameFeatureAction_AddGameplayCuePath` | `UGameFeatureAction`（直接） | 添加 GameplayCue 通知路径 |
| `GameFeatureAction_SplitscreenConfig` | `WorldActionBase` | 配置分屏开关 |
| `ApplyFrontendPerfSettingsAction` | `UGameFeatureAction`（直接） | 切换前端性能设置 |

---

## 5. GameFeatureAction_AddAbilities — 能力注入

**文件**: `Source/LyraGame/GameFeatures/GameFeatureAction_AddAbilities.h/.cpp`

### 5.1 数据结构

```cpp
USTRUCT()
struct FGameFeatureAbilitiesEntry
{
    // 目标 Actor 类（如 ALyraPlayerState）
    TSoftClassPtr<AActor> ActorClass;

    // 授予的 GameplayAbility 列表
    TArray<FLyraAbilityGrant> GrantedAbilities;

    // 授予的 AttributeSet 列表（带初始化数据表）
    TArray<FLyraAttributeSetGrant> GrantedAttributes;

    // 授予的 AbilitySet 列表（组合包）
    TArray<TSoftObjectPtr<const ULyraAbilitySet>> GrantedAbilitySets;
};
```

### 5.2 注入流程

```cpp
void UGameFeatureAction_AddAbilities::AddToWorld(const FWorldContext& WorldContext, ...)
{
    if (UGameFrameworkComponentManager* ComponentMan = ...)
    {
        for (const FGameFeatureAbilitiesEntry& Entry : AbilitiesList)
        {
            // 注册 ExtensionHandler: 当 Entry.ActorClass 类型的 Actor 就绪时回调
            auto Delegate = FExtensionHandlerDelegate::CreateUObject(
                this, &ThisClass::HandleActorExtension, EntryIndex, ChangeContext);
            ComponentMan->AddExtensionHandler(Entry.ActorClass, Delegate);
        }
    }
}
```

### 5.3 Actor 就绪时的回调

```cpp
void UGameFeatureAction_AddAbilities::HandleActorExtension(
    AActor* Actor, FName EventName, int32 EntryIndex, ...)
{
    if (EventName == NAME_ExtensionRemoved || EventName == NAME_ReceiverRemoved)
    {
        RemoveActorAbilities(Actor, ActiveData);
    }
    // 注意：使用自定义事件名 NAME_LyraAbilityReady（而非仅 NAME_ExtensionAdded）
    else if (EventName == NAME_ExtensionAdded || EventName == ALyraPlayerState::NAME_LyraAbilityReady)
    {
        AddActorAbilities(Actor, Entry, ActiveData);
    }
}
```

### 5.4 能力授予实现

```cpp
void UGameFeatureAction_AddAbilities::AddActorAbilities(AActor* Actor, ...)
{
    // 仅在有权限端执行
    if (!Actor->HasAuthority()) return;

    // 防止重复
    if (ActiveData.ActiveExtensions.Find(Actor)) return;

    if (UAbilitySystemComponent* ASC = FindOrAddComponentForActor<UAbilitySystemComponent>(Actor, ...))
    {
        // 1. 授予 Abilities
        for (const FLyraAbilityGrant& Ability : AbilitiesEntry.GrantedAbilities)
        {
            FGameplayAbilitySpec NewAbilitySpec(Ability.AbilityType.LoadSynchronous());
            FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(NewAbilitySpec);
            AddedExtensions.Abilities.Add(Handle);
        }

        // 2. 授予 AttributeSets（带可选初始化数据表）
        for (const FLyraAttributeSetGrant& Attributes : AbilitiesEntry.GrantedAttributes)
        {
            UAttributeSet* NewSet = NewObject<UAttributeSet>(ASC->GetOwner(), SetType);
            if (InitData) NewSet->InitFromMetaDataTable(InitData);
            ASC->AddAttributeSetSubobject(NewSet);
        }

        // 3. 授予 AbilitySets（组合包）
        for (const auto& SetPtr : AbilitiesEntry.GrantedAbilitySets)
        {
            if (const ULyraAbilitySet* Set = SetPtr.Get())
            {
                Set->GiveToAbilitySystem(LyraASC, &AddedExtensions.AbilitySetHandles.AddDefaulted_GetRef());
            }
        }
    }
}
```

### 5.5 清理机制

```cpp
void UGameFeatureAction_AddAbilities::RemoveActorAbilities(AActor* Actor, ...)
{
    if (UAbilitySystemComponent* ASC = Actor->FindComponentByClass<UAbilitySystemComponent>())
    {
        // 移除 AttributeSets
        for (UAttributeSet* Set : ActorExtensions->Attributes)
            ASC->RemoveSpawnedAttribute(Set);

        // 标记 Abilities 在结束时移除
        for (FGameplayAbilitySpecHandle Handle : ActorExtensions->Abilities)
            ASC->SetRemoveAbilityOnEnd(Handle);

        // 撤回 AbilitySet 授予
        for (auto& SetHandle : ActorExtensions->AbilitySetHandles)
            SetHandle.TakeFromAbilitySystem(LyraASC);
    }
}
```

### 5.6 FindOrAddComponent 模式

当 Actor 没有 ASC 时，通过 `ComponentManager->AddComponentRequest` 请求动态添加：

```cpp
UActorComponent* FindOrAddComponentForActor(UClass* ComponentType, AActor* Actor, ...)
{
    UActorComponent* Component = Actor->FindComponentByClass(ComponentType);
    
    if (Component == nullptr || /* 需要追加请求 */)
    {
        ComponentMan->AddComponentRequest(AbilitiesEntry.ActorClass, ComponentType);
        Component = Actor->FindComponentByClass(ComponentType); // 立即可用
    }
    return Component;
}
```

---

## 6. GameFeatureAction_AddInputBinding — 输入绑定注入

**文件**: `Source/LyraGame/GameFeatures/GameFeatureAction_AddInputBinding.h/.cpp`

### 6.1 数据结构

```cpp
UCLASS(meta = (DisplayName = "Add Input Binds"))
class UGameFeatureAction_AddInputBinding final : public UGameFeatureAction_WorldActionBase
{
    // 要注入的 InputConfig 列表
    UPROPERTY(EditAnywhere, Category="Input")
    TArray<TSoftObjectPtr<const ULyraInputConfig>> InputConfigs;
};
```

### 6.2 注入流程

```cpp
void AddToWorld(...)
{
    // 监听所有 APawn 类型的 Actor
    ComponentManager->AddExtensionHandler(APawn::StaticClass(), 
        FExtensionHandlerDelegate::CreateUObject(this, &ThisClass::HandlePawnExtension, ...));
}

void HandlePawnExtension(AActor* Actor, FName EventName, ...)
{
    // 监听 HeroComponent 的 NAME_BindInputsNow 自定义事件
    if (EventName == NAME_ExtensionAdded || EventName == ULyraHeroComponent::NAME_BindInputsNow)
    {
        AddInputMappingForPlayer(CastChecked<APawn>(Actor), ActiveData);
    }
}

void AddInputMappingForPlayer(APawn* Pawn, ...)
{
    ULyraHeroComponent* HeroComponent = Pawn->FindComponentByClass<ULyraHeroComponent>();
    if (HeroComponent && HeroComponent->IsReadyToBindInputs())
    {
        for (const auto& Entry : InputConfigs)
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

**关键点**: 使用 `ULyraHeroComponent::NAME_BindInputsNow` 自定义事件确保在 HeroComponent 完成输入初始化后才注入。

---

## 7. GameFeatureAction_AddInputContextMapping — 输入映射注入

**文件**: `Source/LyraGame/GameFeatures/GameFeatureAction_AddInputContextMapping.h/.cpp`

### 7.1 数据结构

```cpp
USTRUCT()
struct FInputMappingContextAndPriority
{
    TSoftObjectPtr<UInputMappingContext> InputMapping;
    int32 Priority = 0;
    bool bRegisterWithSettings = true;  // 是否注册到用户设置
};
```

### 7.2 双阶段生命周期

该 Action 同时使用 `Registering` 和 `Activating` 两个阶段：

```cpp
// 注册阶段：将 IMC 注册到 EnhancedInput 用户设置（全局设置面板可见）
void OnGameFeatureRegistering()
{
    RegisterInputMappingContexts();
    // 监听 GameInstance 启动，为每个 LocalPlayer 注册 IMC 到 Settings
}

// 注销阶段：从用户设置移除
void OnGameFeatureUnregistering()
{
    UnregisterInputMappingContexts();
}

// 激活阶段：将 IMC 添加到 EnhancedInput 系统（实际生效）
void AddToWorld(...)
{
    // 监听 APlayerController 类型的 Actor
    ComponentManager->AddExtensionHandler(APlayerController::StaticClass(), ...);
}

void AddInputMappingForPlayer(UPlayer* Player, ...)
{
    if (UEnhancedInputLocalPlayerSubsystem* InputSystem = LocalPlayer->GetSubsystem<...>())
    {
        for (const auto& Entry : InputMappings)
        {
            InputSystem->AddMappingContext(Entry.InputMapping.Get(), Entry.Priority);
        }
    }
}
```

**关键设计**: 分离"注册到设置"（全局，影响设置UI）和"添加到运行时"（每World，实际生效）两个层面。

---

## 8. GameFeatureAction_AddWidgets — UI注入

**文件**: `Source/LyraGame/GameFeatures/GameFeatureAction_AddWidget.h/.cpp`

### 8.1 数据结构

```cpp
USTRUCT()
struct FLyraHUDLayoutRequest
{
    TSoftClassPtr<UCommonActivatableWidget> LayoutClass;  // Layout Widget 类
    FGameplayTag LayerID;  // 目标 UI Layer（如 UI.Layer.Game）
};

USTRUCT()
struct FLyraHUDElementEntry
{
    TSoftClassPtr<UUserWidget> WidgetClass;  // Widget 类
    FGameplayTag SlotID;  // UIExtension 扩展点 Tag
};
```

### 8.2 注入流程

```cpp
void AddToWorld(...)
{
    // 监听 ALyraHUD 类型的 Actor
    ComponentManager->AddExtensionHandler(ALyraHUD::StaticClass(), ...);
}

void AddWidgets(AActor* Actor, FPerContextData& ActiveData)
{
    ALyraHUD* HUD = CastChecked<ALyraHUD>(Actor);
    ULocalPlayer* LocalPlayer = HUD->GetOwningPlayerController()->GetLocalPlayer();
    FPerActorData& ActorData = ActiveData.ActorData.FindOrAdd(HUD);

    // 1. 推送 Layout 到指定 Layer
    for (const FLyraHUDLayoutRequest& Entry : Layout)
    {
        ActorData.LayoutsAdded.Add(
            UCommonUIExtensions::PushContentToLayer_ForPlayer(
                LocalPlayer, Entry.LayerID, Entry.LayoutClass.Get()));
    }

    // 2. 注册 Widget 到 UIExtension 系统
    UUIExtensionSubsystem* ExtensionSubsystem = HUD->GetWorld()->GetSubsystem<UUIExtensionSubsystem>();
    for (const FLyraHUDElementEntry& Entry : Widgets)
    {
        ActorData.ExtensionHandles.Add(
            ExtensionSubsystem->RegisterExtensionAsWidgetForContext(
                Entry.SlotID, LocalPlayer, Entry.WidgetClass.Get(), -1));
    }
}
```

### 8.3 清理

```cpp
void RemoveWidgets(AActor* Actor, FPerContextData& ActiveData)
{
    // 1. DeactivateWidget（从 Layer 栈弹出）
    for (auto& AddedLayout : ActorData->LayoutsAdded)
    {
        if (AddedLayout.IsValid())
            AddedLayout->DeactivateWidget();
    }

    // 2. Unregister Extension（从 UIExtension 系统移除）
    for (FUIExtensionHandle& Handle : ActorData->ExtensionHandles)
    {
        Handle.Unregister();
    }
}
```

### 8.4 AssetBundle 预加载

```cpp
#if WITH_EDITORONLY_DATA
void AddAdditionalAssetBundleData(FAssetBundleData& AssetBundleData)
{
    for (const FLyraHUDElementEntry& Entry : Widgets)
    {
        // 将 Widget 类标记为 Client 端 Bundle 资产，确保预加载
        AssetBundleData.AddBundleAsset(
            UGameFeaturesSubsystemSettings::LoadStateClient, 
            Entry.WidgetClass.ToSoftObjectPath().GetAssetPath());
    }
}
#endif
```

---

## 9. GameFeatureAction_AddGameplayCuePath — GameplayCue路径注入

**文件**: `Source/LyraGame/GameFeatures/GameFeatureAction_AddGameplayCuePath.h/.cpp`

### 9.1 设计特点

该 Action **直接继承 `UGameFeatureAction`**（不继承 WorldActionBase），因为 GameplayCue 路径是全局的，不分 World：

```cpp
UCLASS(meta = (DisplayName = "Add Gameplay Cue Path"))
class UGameFeatureAction_AddGameplayCuePath final : public UGameFeatureAction
{
    // 相对于 GameContent 目录的路径列表
    UPROPERTY(EditAnywhere, Category = "Game Feature | Gameplay Cues", 
              meta = (RelativeToGameContentDir, LongPackageName))
    TArray<FDirectoryPath> DirectoryPathsToAdd;
};

// 构造函数中添加默认路径
UGameFeatureAction_AddGameplayCuePath::UGameFeatureAction_AddGameplayCuePath()
{
    DirectoryPathsToAdd.Add(FDirectoryPath{ TEXT("/GameplayCues") });
}
```

### 9.2 实际加载由 Observer 完成

该 Action 本身不实现 `OnGameFeatureActivating`，实际的 GameplayCue 路径注册由 `ULyraGameFeature_AddGameplayCuePaths` 观察者在 `OnGameFeatureRegistering` 时完成（见第12节）。

---

## 10. GameFeatureAction_SplitscreenConfig — 分屏配置

**文件**: `Source/LyraGame/GameFeatures/GameFeatureAction_SplitscreenConfig.h/.cpp`

### 10.1 投票机制

使用全局引用计数（投票）控制分屏开关，支持多个 GameFeature 同时禁用分屏：

```cpp
UCLASS(meta = (DisplayName = "Splitscreen Config"))
class UGameFeatureAction_SplitscreenConfig final : public UGameFeatureAction_WorldActionBase
{
    UPROPERTY(EditAnywhere, Category=Action)
    bool bDisableSplitscreen = true;

    TArray<FObjectKey> LocalDisableVotes;              // 本 Action 的投票
    static TMap<FObjectKey, int32> GlobalDisableVotes;  // 全局投票计数
};
```

### 10.2 激活/停用

```cpp
void AddToWorld(const FWorldContext& WorldContext, ...)
{
    if (bDisableSplitscreen)
    {
        UGameViewportClient* VC = GameInstance->GetGameViewportClient();
        FObjectKey ViewportKey(VC);
        
        LocalDisableVotes.Add(ViewportKey);
        int32& VoteCount = GlobalDisableVotes.FindOrAdd(ViewportKey);
        VoteCount++;
        if (VoteCount == 1)  // 第一个投票者实际禁用
        {
            VC->SetForceDisableSplitscreen(true);
        }
    }
}

void OnGameFeatureDeactivating(...)
{
    for (FObjectKey ViewportKey : LocalDisableVotes)
    {
        int32& VoteCount = GlobalDisableVotes[ViewportKey];
        if (VoteCount <= 1)
        {
            GlobalDisableVotes.Remove(ViewportKey);
            GVP->SetForceDisableSplitscreen(false);  // 最后一个投票者恢复
        }
        else
        {
            --VoteCount;
        }
    }
}
```

---

## 11. ApplyFrontendPerfSettingsAction — 前端性能设置

**文件**: `Source/LyraGame/UI/Frontend/ApplyFrontendPerfSettingsAction.h/.cpp`

直接继承 `UGameFeatureAction`，使用静态引用计数管理前端性能模式：

```cpp
UCLASS(meta = (DisplayName = "Use Frontend Perf Settings"))
class UApplyFrontendPerfSettingsAction final : public UGameFeatureAction
{
    static int32 ApplicationCounter;
};

void UApplyFrontendPerfSettingsAction::OnGameFeatureActivating(...)
{
    ApplicationCounter++;
    if (ApplicationCounter == 1)
    {
        // 第一个前端 Experience 激活时，启用前端性能设置（降低画质提高帧率）
        ULyraSettingsLocal::Get()->SetShouldUseFrontendPerformanceSettings(true);
    }
}

void UApplyFrontendPerfSettingsAction::OnGameFeatureDeactivating(...)
{
    ApplicationCounter--;
    if (ApplicationCounter == 0)
    {
        // 离开所有前端 Experience 时，恢复游戏性能设置
        ULyraSettingsLocal::Get()->SetShouldUseFrontendPerformanceSettings(false);
    }
}
```

---

## 12. LyraGameFeaturePolicy — 加载策略与观察者

**文件**: `Source/LyraGame/GameFeatures/LyraGameFeaturePolicy.h/.cpp`

### 12.1 ULyraGameFeaturePolicy

继承 `UDefaultGameFeaturesProjectPolicies`，自定义 GameFeature 管理策略：

```cpp
UCLASS(Config = Game)
class ULyraGameFeaturePolicy : public UDefaultGameFeaturesProjectPolicies
{
    UPROPERTY(Transient)
    TArray<TObjectPtr<UObject>> Observers;  // 观察者列表
};
```

**初始化时注册观察者**：

```cpp
void ULyraGameFeaturePolicy::InitGameFeatureManager()
{
    // 创建两个全局观察者
    Observers.Add(NewObject<ULyraGameFeature_HotfixManager>());
    Observers.Add(NewObject<ULyraGameFeature_AddGameplayCuePaths>());

    // 注册到 GameFeaturesSubsystem，监听当前和未来的插件状态变化
    UGameFeaturesSubsystem& Subsystem = UGameFeaturesSubsystem::Get();
    for (UObject* Observer : Observers)
    {
        Subsystem.AddObserver(Observer, 
            UGameFeaturesSubsystem::EObserverPluginStateUpdateMode::CurrentAndFuture);
    }
    Super::InitGameFeatureManager();
}
```

**加载模式控制**：

```cpp
void ULyraGameFeaturePolicy::GetGameFeatureLoadingMode(bool& bLoadClientData, bool& bLoadServerData) const
{
    bLoadClientData = !IsRunningDedicatedServer();  // 非专服加载客户端数据
    bLoadServerData = !IsRunningClientOnly();        // 非纯客户端加载服务器数据
}
```

### 12.2 ULyraGameFeature_HotfixManager（观察者1）

```cpp
UCLASS()
class ULyraGameFeature_HotfixManager : public UObject, public IGameFeatureStateChangeObserver
{
    virtual void OnGameFeatureLoading(const UGameFeatureData* GameFeatureData, const FString& PluginURL) override
    {
        // 每当 GameFeature 加载时，请求 HotfixManager 修补资产
        if (ULyraHotfixManager* HotfixManager = Cast<ULyraHotfixManager>(UOnlineHotfixManager::Get(nullptr)))
        {
            HotfixManager->RequestPatchAssetsFromIniFiles();
        }
    }
};
```

### 12.3 ULyraGameFeature_AddGameplayCuePaths（观察者2）

```cpp
UCLASS()
class ULyraGameFeature_AddGameplayCuePaths : public UObject, public IGameFeatureStateChangeObserver
{
    // 插件注册时：扫描 GameFeatureData 中的 AddGameplayCuePath Action，
    // 将路径添加到 GameplayCueManager
    virtual void OnGameFeatureRegistering(const UGameFeatureData* GameFeatureData, 
        const FString& PluginName, const FString& PluginURL) override
    {
        const FString PluginRootPath = TEXT("/") + PluginName;
        for (const UGameFeatureAction* Action : GameFeatureData->GetActions())
        {
            if (const auto* AddCueGFA = Cast<UGameFeatureAction_AddGameplayCuePath>(Action))
            {
                if (ULyraGameplayCueManager* GCM = ULyraGameplayCueManager::Get())
                {
                    for (const FDirectoryPath& Directory : AddCueGFA->GetDirectoryPathsToAdd())
                    {
                        FString MutablePath = Directory.Path;
                        UGameFeaturesSubsystem::FixPluginPackagePath(MutablePath, PluginRootPath, false);
                        GCM->AddGameplayCueNotifyPath(MutablePath, false);
                    }
                    GCM->InitializeRuntimeObjectLibrary();
                }
            }
        }
    }

    // 插件注销时：移除路径
    virtual void OnGameFeatureUnregistering(...) override
    {
        // 对称地 RemoveGameplayCueNotifyPath
    }
};
```

---

## 13. 具体 GameFeature 插件实例分析

### 13.1 插件依赖关系

```
ShooterCore (核心射击玩法)
  ├── ShooterMaps (射击地图) ──依赖→ ShooterCore
  ├── ShooterExplorer (探索模式) ──依赖→ ShooterCore
  └── ShooterTests (测试) ──依赖→ (隐式)

TopDownArena (俯视竞技场) ──独立──
```

### 13.2 .uplugin 关键配置

所有 GameFeature 插件共享以下配置模式：

```json
{
    "ExplicitlyLoaded": true,            // 必须由 Experience 显式加载
    "EnabledByDefault": false,            // 默认不启用
    "BuiltInInitialFeatureState": "Registered",  // 初始状态为"已注册"
    "CanContainContent": true,            // 包含内容资产
    "Category": "Game Features"
}
```

### 13.3 ShooterCore — 射击核心

**依赖插件**: GameplayAbilities, ModularGameplay, GameplayMessageRouter, AsyncMixin, CommonUI, CommonGame, GameSubtitles, EnhancedInput, LyraExampleContent

**Runtime 模块**: `ShooterCoreRuntime`

**提供的功能**：
- **荣誉系统**: `LyraAccoladeDefinition`/`LyraAccoladeHostWidget` — 连杀、助攻等荣誉奖励
- **辅助瞄准**: `AimAssistInputModifier`/`AimAssistTargetComponent`/`AimAssistTargetManagerComponent` — 手柄辅助瞄准
- **消息处理器**: `AssistProcessor`/`ElimChainProcessor`/`ElimStreakProcessor` — 击杀链、连杀统计
- **控制点消息**: `ControlPointStatusMessage` — 占领模式消息
- **TDM生成管理**: `TDM_PlayerSpawningManagmentComponent` — 团队死斗出生点管理
- **可收集物**: `LyraWorldCollectable` — 世界中的可拾取物

### 13.4 ShooterMaps — 射击地图

**依赖插件**: ShooterCore, LyraExampleContent

**无 Runtime 模块**（纯内容插件），包含射击模式的地图和 Experience 配置资产。

### 13.5 ShooterExplorer — 探索模式

**依赖插件**: ShooterCore, LyraExampleContent

**无 Runtime 模块**（纯内容插件），在 ShooterCore 基础上添加探索元素。

### 13.6 TopDownArena — 俯视竞技场

**依赖插件**: GameplayAbilities, LyraExampleContent, Niagara

**Runtime 模块**: `TopDownArenaRuntime`

**提供的功能**（完全独立于 ShooterCore）：
- **自定义相机**: `LyraCameraMode_TopDownArenaCamera` — 俯视固定相机模式
- **自定义移动**: `TopDownArenaMovementComponent` — 俯视视角移动逻辑
- **自定义属性**: `TopDownArenaAttributeSet` — 竞技场专用属性
- **拾取物UI**: `TopDownArenaPickupUIData` — 拾取物UI数据

### 13.7 插件组合示例

一个典型的射击 TDM Experience 的配置：

```
B_ShooterGame_TDM (ULyraExperienceDefinition)
├── GameFeaturesToEnable:
│     ├── "ShooterCore"     → 射击核心玩法
│     └── "ShooterMaps"     → 射击地图
├── DefaultPawnData: HeroData_ShooterGame
├── Actions: (内联)
│     ├── GameFeatureAction_AddWidgets → HUD Layout
│     ├── GameFeatureAction_AddInputContextMapping → 射击输入
│     └── GameFeatureAction_SplitscreenConfig → 禁用分屏
└── ActionSets:
      ├── AS_ShooterGame_SharedHUD → 通用 HUD Action
      └── AS_ShooterGame_SharedInput → 通用输入 Action
```

一个俯视竞技场 Experience：

```
B_TopDownArena (ULyraExperienceDefinition)
├── GameFeaturesToEnable:
│     └── "TopDownArena"    → 俯视竞技场（独立于 ShooterCore）
├── DefaultPawnData: HeroData_TopDownArena
├── Actions:
│     ├── GameFeatureAction_AddWidgets → 俯视 HUD
│     ├── GameFeatureAction_AddAbilities → 竞技场能力
│     └── GameFeatureAction_AddInputContextMapping → 俯视输入
└── ActionSets: ...
```

---

## 14. 完整架构图与核心设计模式总结

### 14.1 完整架构图

```
┌─────────────── 地图加载 ──────────────────────────────────────────────────────┐
│                                                                                │
│  ALyraWorldSettings                                                           │
│    └── DefaultGameplayExperience: TSoftClassPtr<ULyraExperienceDefinition>    │
│                                                                                │
│  ALyraGameMode (继承 AModularGameModeBase)                                    │
│    ├── InitGame() → HandleMatchAssignmentIfNotExpectingOne()                  │
│    │     → 确定 ExperienceId (URL > DevSettings > CmdLine > WorldSettings)   │
│    │     → ExperienceManagerComponent->SetCurrentExperience(ExperienceId)     │
│    ├── InitGameState() → 注册 OnExperienceLoaded 回调                        │
│    └── OnExperienceLoaded() → RestartPlayer (生成 Pawn)                      │
│                                                                                │
├─────────────── Experience 加载状态机 ─────────────────────────────────────────┤
│                                                                                │
│  ULyraExperienceManagerComponent (挂在 GameState 上, Replicated)              │
│    ├── Unloaded → Loading                                                     │
│    │     → 异步加载 Experience + ActionSet 资产 (AssetBundle)                  │
│    ├── Loading → LoadingGameFeatures                                          │
│    │     → 收集所有 GameFeaturesToEnable URL                                  │
│    │     → LoadAndActivateGameFeaturePlugin() 逐个加载                        │
│    ├── LoadingGameFeatures → ExecutingActions                                 │
│    │     → Action::OnGameFeatureRegistering()                                 │
│    │     → Action::OnGameFeatureLoading()                                     │
│    │     → Action::OnGameFeatureActivating(Context)                           │
│    ├── ExecutingActions → Loaded                                              │
│    │     → Broadcast: HighPriority → Normal → LowPriority                    │
│    └── Loaded → Deactivating (EndPlay)                                       │
│          → Action::OnGameFeatureDeactivating()                                │
│          → Action::OnGameFeatureUnregistering()                               │
│          → DeactivateGameFeaturePlugin() (引用计数)                            │
│                                                                                │
├─────────────── Action 执行层 ─────────────────────────────────────────────────┤
│                                                                                │
│  UGameFeatureAction_WorldActionBase                                           │
│    ├── OnGameFeatureActivating(Context)                                       │
│    │     ├── 注册 FWorldDelegates::OnStartGameInstance 委托                   │
│    │     └── 遍历现有 WorldContext → AddToWorld()                             │
│    └── AddToWorld() [子类实现]                                                │
│          → ComponentManager->AddExtensionHandler(ActorClass, Delegate)        │
│                                                                                │
│  ┌─ AddAbilities ────────────────────────────────────────────────────────┐    │
│  │  监听 ActorClass → NAME_LyraAbilityReady                             │    │
│  │  → GiveAbility / AddAttributeSet / GiveAbilitySet                    │    │
│  └───────────────────────────────────────────────────────────────────────┘    │
│  ┌─ AddInputBinding ────────────────────────────────────────────────────┐    │
│  │  监听 APawn → NAME_BindInputsNow                                     │    │
│  │  → HeroComponent->AddAdditionalInputConfig()                         │    │
│  └───────────────────────────────────────────────────────────────────────┘    │
│  ┌─ AddInputContextMapping ─────────────────────────────────────────────┐    │
│  │  监听 APlayerController → NAME_BindInputsNow                         │    │
│  │  → EnhancedInputSubsystem->AddMappingContext()                        │    │
│  │  + OnRegistering: 注册到 EnhancedInputUserSettings                   │    │
│  └───────────────────────────────────────────────────────────────────────┘    │
│  ┌─ AddWidgets ─────────────────────────────────────────────────────────┐    │
│  │  监听 ALyraHUD → NAME_GameActorReady                                 │    │
│  │  → PushContentToLayer(Layout) + RegisterExtensionAsWidget(Widget)     │    │
│  └───────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
├─────────────── 模块化 Actor 基础设施 ─────────────────────────────────────────┤
│                                                                                │
│  UGameFrameworkComponentManager (UGameInstanceSubsystem)                       │
│    ├── AddComponentRequest(ActorClass, ComponentClass)                         │
│    │     → 当目标 Actor 创建时动态添加组件                                     │
│    ├── AddExtensionHandler(ActorClass, Delegate)                              │
│    │     → 当目标 Actor 就绪时回调                                             │
│    └── SendGameFrameworkComponentExtensionEvent(Actor, EventName)              │
│          → 触发已注册的 Handler                                                │
│                                                                                │
│  ModularGameplayActors (所有 Lyra Actor 的基类)                               │
│    → PreInitializeComponents: AddGameFrameworkComponentReceiver                │
│    → BeginPlay/ReceivedPlayer: SendExtensionEvent(NAME_GameActorReady)        │
│    → EndPlay: RemoveGameFrameworkComponentReceiver                             │
│                                                                                │
├─────────────── 全局观察者 ────────────────────────────────────────────────────┤
│                                                                                │
│  ULyraGameFeaturePolicy                                                       │
│    ├── Observer: ULyraGameFeature_HotfixManager                              │
│    │     → OnGameFeatureLoading: 请求 HotfixManager 修补资产                  │
│    └── Observer: ULyraGameFeature_AddGameplayCuePaths                        │
│          → OnGameFeatureRegistering: 扫描 AddGameplayCuePath Action          │
│          → 添加路径到 GameplayCueManager                                      │
│                                                                                │
├─────────────── GameFeature 插件实例 ──────────────────────────────────────────┤
│                                                                                │
│  Plugins/GameFeatures/                                                        │
│    ├── ShooterCore/    (射击核心: 荣誉/辅助瞄准/消息处理)                      │
│    ├── ShooterMaps/    (射击地图: 纯内容) → 依赖 ShooterCore                  │
│    ├── ShooterExplorer/(探索模式: 纯内容) → 依赖 ShooterCore                  │
│    ├── ShooterTests/   (测试)                                                 │
│    └── TopDownArena/   (俯视竞技场: 相机/移动/属性) → 独立                    │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 14.2 核心设计模式

#### 模式1: ExtensionHandler 回调模式

所有 WorldActionBase 子类共享的注入模式：

```
GameFeature 激活
  → Action::AddToWorld()
    → ComponentManager->AddExtensionHandler(目标ActorClass, 回调Delegate)
      → 当目标 Actor 发送 ExtensionEvent 时
        → 回调执行注入逻辑（添加能力/输入/Widget）
      → 当目标 Actor 销毁时
        → 回调执行清理逻辑
```

#### 模式2: 自定义事件名扩展

除了引擎默认的 `NAME_ExtensionAdded` / `NAME_GameActorReady`，Lyra 定义了自定义事件名来控制更精确的时序：

| 自定义事件名 | 来源 | 用途 |
|-------------|------|------|
| `ALyraPlayerState::NAME_LyraAbilityReady` | PlayerState | ASC 就绪后才注入能力 |
| `ULyraHeroComponent::NAME_BindInputsNow` | HeroComponent | 输入系统就绪后才绑定 |

#### 模式3: PerContextData 隔离

每个 Action 使用 `TMap<FGameFeatureStateChangeContext, FPerContextData>` 隔离不同 World 的数据：

```cpp
struct FPerContextData
{
    TArray<TSharedPtr<FComponentRequestHandle>> ComponentRequests;
    TMap<AActor*, FActorExtensions> ActiveExtensions;
};
TMap<FGameFeatureStateChangeContext, FPerContextData> ContextData;
```

这确保了 PIE 多 World 场景下的正确性。

#### 模式4: 投票/引用计数

分屏配置和性能设置使用引用计数模式，支持多个 GameFeature 对同一全局状态投票：

```
Feature A 激活 → Count = 1 → 禁用分屏
Feature B 激活 → Count = 2 → 保持禁用
Feature A 停用 → Count = 1 → 保持禁用
Feature B 停用 → Count = 0 → 恢复分屏
```

#### 模式5: Experience 组合模式

Experience 通过三个维度组合功能：

| 维度 | 来源 | 生命周期 |
|------|------|----------|
| `GameFeaturesToEnable` | 激活整个 GameFeature 插件 | 插件级别 |
| `Actions` | 内联 Action 实例 | Experience 级别 |
| `ActionSets` | 可复用 Action 集合 | 跨 Experience 复用 |

#### 模式6: Observer 全局观察

`IGameFeatureStateChangeObserver` 允许在不修改 Action 的情况下，对所有 GameFeature 的生命周期事件做全局响应（如自动注册 GameplayCue 路径、触发热修复）。

---

## 15. 文件索引

| 目录/文件 | 核心职责 |
|-----------|----------|
| **基础设施层** | |
| `Plugins/ModularGameplayActors/` | 模块化 Actor 基类（组件注入三步协议） |
| **Experience 驱动层** | |
| `Source/LyraGame/GameModes/LyraExperienceDefinition.h/.cpp` | Experience 数据定义 |
| `Source/LyraGame/GameModes/LyraExperienceActionSet.h/.cpp` | 可复用 Action 集合 |
| `Source/LyraGame/GameModes/LyraExperienceManagerComponent.h/.cpp` | Experience 加载状态机（核心） |
| `Source/LyraGame/GameModes/LyraExperienceManager.h/.cpp` | PIE 插件引用计数 |
| `Source/LyraGame/GameModes/AsyncAction_ExperienceReady.h/.cpp` | 蓝图异步等待 Experience |
| `Source/LyraGame/GameModes/LyraUserFacingExperienceDefinition.h/.cpp` | 用户界面 Experience 描述 |
| `Source/LyraGame/GameModes/LyraGameMode.h/.cpp` | 游戏模式（Experience 选择与启动） |
| `Source/LyraGame/GameModes/LyraWorldSettings.h/.cpp` | 地图默认 Experience 配置 |
| **自定义 Action 层** | |
| `Source/LyraGame/GameFeatures/GameFeatureAction_WorldActionBase.h/.cpp` | World 限定 Action 基类 |
| `Source/LyraGame/GameFeatures/GameFeatureAction_AddAbilities.h/.cpp` | 能力/属性/AbilitySet 注入 |
| `Source/LyraGame/GameFeatures/GameFeatureAction_AddInputBinding.h/.cpp` | InputConfig 注入 |
| `Source/LyraGame/GameFeatures/GameFeatureAction_AddInputContextMapping.h/.cpp` | InputMappingContext 注入 |
| `Source/LyraGame/GameFeatures/GameFeatureAction_AddWidget.h/.cpp` | UI Layout/Widget 注入 |
| `Source/LyraGame/GameFeatures/GameFeatureAction_AddGameplayCuePath.h/.cpp` | GameplayCue 路径声明 |
| `Source/LyraGame/GameFeatures/GameFeatureAction_SplitscreenConfig.h/.cpp` | 分屏投票控制 |
| `Source/LyraGame/UI/Frontend/ApplyFrontendPerfSettingsAction.h/.cpp` | 前端性能设置切换 |
| `Source/LyraGame/GameFeatures/LyraGameFeaturePolicy.h/.cpp` | 加载策略 + 全局观察者 |
| **GameFeature 插件实例** | |
| `Plugins/GameFeatures/ShooterCore/` | 射击核心（荣誉/辅助瞄准/消息处理） |
| `Plugins/GameFeatures/ShooterMaps/` | 射击地图（纯内容） |
| `Plugins/GameFeatures/ShooterExplorer/` | 探索模式（纯内容） |
| `Plugins/GameFeatures/ShooterTests/` | 自动化测试 |
| `Plugins/GameFeatures/TopDownArena/` | 俯视竞技场（相机/移动/属性） |
