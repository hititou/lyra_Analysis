# Lyra Experience 系统（体验系统）深度代码分析

## 目录

1. [系统架构总览](#1-系统架构总览)
2. [ULyraExperienceDefinition — 体验定义资产](#2-ulyraexperiencedefinition--体验定义资产)
3. [ULyraExperienceActionSet — 动作集合](#3-ulyraexperienceactionset--动作集合)
4. [ULyraExperienceManagerComponent — 核心加载管理器](#4-ulyraexperiencemanagercomponent--核心加载管理器)
5. [ALyraGameMode — Experience 选择与启动](#5-alyragamemode--experience-选择与启动)
6. [Experience 加载状态机详解](#6-experience-加载状态机详解)
7. [ULyraExperienceManager — PIE 多实例协调](#7-ulyraexperiencemanager--pie-多实例协调)
8. [ULyraUserFacingExperienceDefinition — 用户界面体验描述](#8-ulyrauserfacingexperiencedefinition--用户界面体验描述)
9. [UAsyncAction_ExperienceReady — 蓝图异步等待](#9-uasyncaction_experienceready--蓝图异步等待)
10. [Experience 消费者 — 各子系统如何监听](#10-experience-消费者--各子系统如何监听)
11. [ALyraWorldSettings — 地图默认体验](#11-alyraworldsettings--地图默认体验)
12. [ULyraAssetManager — 资产加载基础设施](#12-ulyraassetmanager--资产加载基础设施)
13. [完整架构图与数据流](#13-完整架构图与数据流)
14. [核心设计模式总结](#14-核心设计模式总结)
15. [文件索引](#15-文件索引)

---

## 1. 系统架构总览

Experience 系统是 Lyra 最具创新性的架构设计，它将传统引擎中"硬编码在 GameMode 里的游戏规则"转变为**数据驱动的、可异步加载的、模块化组合的体验定义**。一个 Experience 定义了"玩家进入某张地图后，游戏应该是什么样子"。

### 1.1 核心设计理念

- **数据驱动**：游戏规则不写在 GameMode 代码里，而是配置在 `ULyraExperienceDefinition` 数据资产中
- **组合优于继承**：通过 `Actions` + `ActionSets` 组合功能，而非为每种模式写一个 GameMode 子类
- **异步加载**：Experience 及其依赖的 GameFeature 插件全部异步加载，配合 Loading Screen
- **优先级回调**：加载完成后按 High/Normal/Low 三级优先级通知消费者
- **网络复制**：服务器决定 Experience，通过属性复制同步到客户端
- **FILO 插件管理**：PIE 多实例场景下，GameFeature 插件采用引用计数管理

### 1.2 核心类一览

```
Source/LyraGame/GameModes/
├── LyraExperienceDefinition.h/cpp           // Experience 数据资产定义
├── LyraExperienceActionSet.h/cpp            // 可复用的 Action 集合
├── LyraExperienceManagerComponent.h/cpp     // 挂在 GameState 上的加载管理器（核心）
├── LyraExperienceManager.h/cpp              // EngineSubsystem，PIE 多实例协调
├── LyraUserFacingExperienceDefinition.h/cpp // UI 展示用的体验描述
├── AsyncAction_ExperienceReady.h/cpp        // 蓝图异步等待节点
├── LyraGameMode.h/cpp                       // Experience 选择逻辑
├── LyraGameState.h/cpp                      // ExperienceManagerComponent 的宿主
├── LyraWorldSettings.h/cpp                  // 地图级默认 Experience 配置
└── LyraBotCreationComponent.h/cpp           // Experience 消费者示例（Bot 创建）
```

### 1.3 类层次结构

```
UPrimaryDataAsset
├── ULyraExperienceDefinition        // 核心：定义 PawnData + Actions + ActionSets + GameFeatures
├── ULyraExperienceActionSet          // 可复用的 Action 打包集合
└── ULyraUserFacingExperienceDefinition // UI 展示层：标题/图标/地图/最大人数等

UGameStateComponent
└── ULyraExperienceManagerComponent   // 核心：异步加载 + 状态机 + 优先级回调

UEngineSubsystem
└── ULyraExperienceManager            // PIE 多实例 GameFeature 引用计数

UBlueprintAsyncActionBase
└── UAsyncAction_ExperienceReady      // 蓝图异步等待 Experience 加载完成

AModularGameModeBase
└── ALyraGameMode                     // Experience 选择（优先级链）+ Pawn 生成

AModularGameStateBase
└── ALyraGameState                    // ExperienceManagerComponent 的宿主

AWorldSettings
└── ALyraWorldSettings                // 地图级默认 Experience 配置
```

---

## 2. ULyraExperienceDefinition — 体验定义资产

**文件**: `Source/LyraGame/GameModes/LyraExperienceDefinition.h/.cpp`

`ULyraExperienceDefinition` 是 Experience 系统的核心数据结构，它是一个 **不可变的蓝图数据资产**（`Const`），集中定义了一个游戏体验所需的所有配置。

### 2.1 数据结构

```cpp
UCLASS(BlueprintType, Const)
class ULyraExperienceDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 需要激活的 GameFeature 插件名称列表
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TArray<FString> GameFeaturesToEnable;

    // 默认 Pawn 数据（定义角色类、技能集、输入、相机模式等）
    UPROPERTY(EditDefaultsOnly, Category=Gameplay)
    TObjectPtr<const ULyraPawnData> DefaultPawnData;

    // 需要执行的 GameFeatureAction 列表（Instanced = 内联编辑实例）
    UPROPERTY(EditDefaultsOnly, Instanced, Category="Actions")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;

    // 附加的 ActionSet 引用列表（组合模式）
    UPROPERTY(EditDefaultsOnly, Category=Gameplay)
    TArray<TObjectPtr<ULyraExperienceActionSet>> ActionSets;
};
```

### 2.2 四大核心字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `GameFeaturesToEnable` | `TArray<FString>` | 插件名列表，如 "ShooterCore"、"TopDownArena" |
| `DefaultPawnData` | `ULyraPawnData*` | 定义 PawnClass、AbilitySets、InputConfig、DefaultCameraMode |
| `Actions` | `TArray<UGameFeatureAction*>` | 内联的 Action 实例（AddAbilities、AddWidget 等） |
| `ActionSets` | `TArray<ULyraExperienceActionSet*>` | 引用的共享 ActionSet（可被多个 Experience 复用） |

### 2.3 数据验证（编辑器）

```cpp
#if WITH_EDITOR
EDataValidationResult ULyraExperienceDefinition::IsDataValid(FDataValidationContext& Context) const
{
    // 1. 验证每个 Action 不为 null，且各自的 IsDataValid() 通过
    for (const UGameFeatureAction* Action : Actions)
    {
        if (Action)
            Result = CombineDataValidationResults(Result, Action->IsDataValid(Context));
        else
            Context.AddError("Null entry at index N in Actions");
    }

    // 2. 禁止蓝图二级继承（只允许从 C++ 基类直接继承一次蓝图）
    if (!GetClass()->IsNative())
    {
        const UClass* ParentClass = GetClass()->GetSuperClass();
        const UClass* FirstNativeParent = FindFirstNativeParent(ParentClass);
        if (FirstNativeParent != ParentClass)
        {
            Context.AddError("Blueprint subclasses of Blueprint experiences is not currently supported"
                             "(use composition via ActionSets instead)");
        }
    }
}
#endif
```

**关键设计决策**：
- **禁止蓝图多级继承** — 强制使用 `ActionSets` 组合而非继承，保证架构可维护性
- **`Const` 类修饰** — `ULyraExperienceDefinition` 标记为 `Const`，表示运行时不可修改
- **`Instanced` Actions** — Actions 是内联实例化的，每个 Experience 有自己的 Action 副本

### 2.4 Asset Bundle 更新

```cpp
#if WITH_EDITORONLY_DATA
void ULyraExperienceDefinition::UpdateAssetBundleData()
{
    Super::UpdateAssetBundleData();
    
    // 让每个 Action 也能注册自己的 asset bundle 依赖
    for (UGameFeatureAction* Action : Actions)
    {
        if (Action)
            Action->AddAdditionalAssetBundleData(AssetBundleData);
    }
}
#endif
```

这确保了 Action 引用的资产（如 Widget 类、Ability 类）能被正确加入异步加载的 bundle 中。

---

## 3. ULyraExperienceActionSet — 动作集合

**文件**: `Source/LyraGame/GameModes/LyraExperienceActionSet.h/.cpp`

`ULyraExperienceActionSet` 是**可复用的 Action 打包集合**，是 Experience 组合模式的关键。

### 3.1 数据结构

```cpp
UCLASS(BlueprintType, NotBlueprintable)
class ULyraExperienceActionSet : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 需要执行的 GameFeatureAction 列表
    UPROPERTY(EditAnywhere, Instanced, Category="Actions to Perform")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;

    // 附加的 GameFeature 插件依赖
    UPROPERTY(EditAnywhere, Category="Feature Dependencies")
    TArray<FString> GameFeaturesToEnable;
};
```

### 3.2 设计意图

```
Experience A (射击 TDM)            Experience B (射击 FFA)
├── DefaultPawnData: ShooterPawn    ├── DefaultPawnData: ShooterPawn
├── ActionSets:                     ├── ActionSets:
│   ├── AS_ShooterBase ────────────►│   ├── AS_ShooterBase        ← 复用！
│   └── AS_TeamDeathmatch           │   └── AS_FreeForAll
└── Actions: [...]                  └── Actions: [...]
```

- **`NotBlueprintable`**：不允许蓝图继承，只能作为数据资产使用
- ActionSet 也有自己的 `GameFeaturesToEnable`，加载时会与 Experience 的合并
- 同样支持 `IsDataValid()` 和 `UpdateAssetBundleData()`

---

## 4. ULyraExperienceManagerComponent — 核心加载管理器

**文件**: `Source/LyraGame/GameModes/LyraExperienceManagerComponent.h/.cpp`

这是 Experience 系统的**大脑**，负责整个加载生命周期的状态机管理。它挂在 `ALyraGameState` 上，实现了 `ILoadingProcessInterface` 以控制 Loading Screen。

### 4.1 加载状态枚举

```cpp
enum class ELyraExperienceLoadState
{
    Unloaded,                 // 初始状态
    Loading,                  // 正在异步加载 Experience 资产
    LoadingGameFeatures,      // 正在加载 GameFeature 插件
    LoadingChaosTestingDelay, // 混沌测试延迟（可选）
    ExecutingActions,         // 正在执行 GameFeatureAction
    Loaded,                   // 加载完成
    Deactivating              // 正在卸载
};
```

### 4.2 类声明

```cpp
UCLASS(MinimalAPI)
class ULyraExperienceManagerComponent final : public UGameStateComponent, public ILoadingProcessInterface
{
    GENERATED_BODY()

public:
    // 设置当前 Experience（由 GameMode 调用）
    void SetCurrentExperience(FPrimaryAssetId ExperienceId);

    // 三级优先级回调注册
    void CallOrRegister_OnExperienceLoaded_HighPriority(FOnLyraExperienceLoaded::FDelegate&& Delegate);
    void CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate&& Delegate);
    void CallOrRegister_OnExperienceLoaded_LowPriority(FOnLyraExperienceLoaded::FDelegate&& Delegate);

    // 获取已加载的 Experience（未加载时 assert）
    const ULyraExperienceDefinition* GetCurrentExperienceChecked() const;
    bool IsExperienceLoaded() const;

    // ILoadingProcessInterface — 控制 Loading Screen
    virtual bool ShouldShowLoadingScreen(FString& OutReason) const override;

private:
    UPROPERTY(ReplicatedUsing=OnRep_CurrentExperience)
    TObjectPtr<const ULyraExperienceDefinition> CurrentExperience;

    ELyraExperienceLoadState LoadState = ELyraExperienceLoadState::Unloaded;
    int32 NumGameFeaturePluginsLoading = 0;
    TArray<FString> GameFeaturePluginURLs;

    // 三级优先级委托
    FOnLyraExperienceLoaded OnExperienceLoaded_HighPriority;
    FOnLyraExperienceLoaded OnExperienceLoaded;
    FOnLyraExperienceLoaded OnExperienceLoaded_LowPriority;
};
```

### 4.3 SetCurrentExperience — 入口点

```cpp
void ULyraExperienceManagerComponent::SetCurrentExperience(FPrimaryAssetId ExperienceId)
{
    ULyraAssetManager& AssetManager = ULyraAssetManager::Get();
    
    // 通过 PrimaryAssetId 获取资产路径，同步加载 Class（非实例）
    FSoftObjectPath AssetPath = AssetManager.GetPrimaryAssetPath(ExperienceId);
    TSubclassOf<ULyraExperienceDefinition> AssetClass = Cast<UClass>(AssetPath.TryLoad());
    
    // 获取 CDO（Class Default Object）作为 Experience 定义
    const ULyraExperienceDefinition* Experience = GetDefault<ULyraExperienceDefinition>(AssetClass);

    check(Experience != nullptr);
    check(CurrentExperience == nullptr);  // 不允许重复设置
    CurrentExperience = Experience;       // 赋值触发网络复制
    StartExperienceLoad();                // 开始异步加载流程
}
```

**关键细节**：
- Experience 使用**蓝图 CDO**作为数据源，而非实例化对象
- `CurrentExperience` 是 `ReplicatedUsing`，赋值后自动复制到客户端
- 客户端收到复制后调用 `OnRep_CurrentExperience()` → `StartExperienceLoad()`

### 4.4 StartExperienceLoad — 资产异步加载

```cpp
void ULyraExperienceManagerComponent::StartExperienceLoad()
{
    check(CurrentExperience != nullptr);
    check(LoadState == ELyraExperienceLoadState::Unloaded);
    LoadState = ELyraExperienceLoadState::Loading;

    ULyraAssetManager& AssetManager = ULyraAssetManager::Get();

    // 1. 收集需要加载的 PrimaryAsset
    TSet<FPrimaryAssetId> BundleAssetList;
    BundleAssetList.Add(CurrentExperience->GetPrimaryAssetId());
    for (const auto& ActionSet : CurrentExperience->ActionSets)
    {
        if (ActionSet != nullptr)
            BundleAssetList.Add(ActionSet->GetPrimaryAssetId());
    }

    // 2. 确定加载 bundle（根据 Client/Server 角色）
    TArray<FName> BundlesToLoad;
    BundlesToLoad.Add(FLyraBundles::Equipped);
    
    const ENetMode OwnerNetMode = GetOwner()->GetNetMode();
    const bool bLoadClient = GIsEditor || (OwnerNetMode != NM_DedicatedServer);
    const bool bLoadServer = GIsEditor || (OwnerNetMode != NM_Client);
    if (bLoadClient) BundlesToLoad.Add(UGameFeaturesSubsystemSettings::LoadStateClient);
    if (bLoadServer) BundlesToLoad.Add(UGameFeaturesSubsystemSettings::LoadStateServer);

    // 3. 异步加载 Asset Bundle
    TSharedPtr<FStreamableHandle> BundleLoadHandle = 
        AssetManager.ChangeBundleStateForPrimaryAssets(
            BundleAssetList.Array(), BundlesToLoad, {}, false, 
            FStreamableDelegate(), FStreamableManager::AsyncLoadHighPriority);

    // 4. 绑定完成回调
    FStreamableDelegate OnAssetsLoadedDelegate = 
        FStreamableDelegate::CreateUObject(this, &ThisClass::OnExperienceLoadComplete);
    
    if (!Handle.IsValid() || Handle->HasLoadCompleted())
    {
        FStreamableHandle::ExecuteDelegate(OnAssetsLoadedDelegate);  // 已加载，立即回调
    }
    else
    {
        Handle->BindCompleteDelegate(OnAssetsLoadedDelegate);
        Handle->BindCancelDelegate(/* 取消时也执行回调 */);
    }
}
```

**加载策略**：
- **Bundle 分角色**：Editor 加载全部，DedicatedServer 跳过 Client bundle，纯 Client 跳过 Server bundle
- **AsyncLoadHighPriority**：使用最高优先级异步加载
- **取消安全**：即使异步加载被取消，也会执行回调

### 4.5 OnExperienceLoadComplete — GameFeature 插件加载

```cpp
void ULyraExperienceManagerComponent::OnExperienceLoadComplete()
{
    check(LoadState == ELyraExperienceLoadState::Loading);

    // 收集所有 GameFeature 插件 URL（去重）
    GameFeaturePluginURLs.Reset();

    auto CollectGameFeaturePluginURLs = [This=this](const UPrimaryDataAsset* Context, 
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

    // 收集 Experience 自身 + 所有 ActionSet 的插件依赖
    CollectGameFeaturePluginURLs(CurrentExperience, CurrentExperience->GameFeaturesToEnable);
    for (const auto& ActionSet : CurrentExperience->ActionSets)
    {
        if (ActionSet != nullptr)
            CollectGameFeaturePluginURLs(ActionSet, ActionSet->GameFeaturesToEnable);
    }

    // 加载并激活每个 GameFeature 插件
    NumGameFeaturePluginsLoading = GameFeaturePluginURLs.Num();
    if (NumGameFeaturePluginsLoading > 0)
    {
        LoadState = ELyraExperienceLoadState::LoadingGameFeatures;
        for (const FString& PluginURL : GameFeaturePluginURLs)
        {
            ULyraExperienceManager::NotifyOfPluginActivation(PluginURL);  // PIE 引用计数 +1
            UGameFeaturesSubsystem::Get().LoadAndActivateGameFeaturePlugin(
                PluginURL, 
                FGameFeaturePluginLoadComplete::CreateUObject(this, &ThisClass::OnGameFeaturePluginLoadComplete));
        }
    }
    else
    {
        OnExperienceFullLoadCompleted();  // 无插件，直接完成
    }
}
```

**关键流程**：
1. 插件名（如 "ShooterCore"）→ 通过 `GetPluginURLByName` 转为 URL
2. `AddUnique` 去重（Experience + ActionSets 可能引用相同插件）
3. 并行加载所有插件，计数器跟踪完成数

### 4.6 OnExperienceFullLoadCompleted — Action 执行与回调广播

```cpp
void ULyraExperienceManagerComponent::OnExperienceFullLoadCompleted()
{
    // 可选的混沌测试延迟
    if (LoadState != ELyraExperienceLoadState::LoadingChaosTestingDelay)
    {
        const float DelaySecs = LyraConsoleVariables::GetExperienceLoadDelayDuration();
        if (DelaySecs > 0.0f)
        {
            LoadState = ELyraExperienceLoadState::LoadingChaosTestingDelay;
            GetWorld()->GetTimerManager().SetTimer(DummyHandle, this, 
                &ThisClass::OnExperienceFullLoadCompleted, DelaySecs, false);
            return;  // 延迟后重入此函数
        }
    }

    LoadState = ELyraExperienceLoadState::ExecutingActions;

    // 构建 World 上下文
    FGameFeatureActivatingContext Context;
    const FWorldContext* ExistingWorldContext = GEngine->GetWorldContextFromWorld(GetWorld());
    if (ExistingWorldContext)
        Context.SetRequiredWorldContextHandle(ExistingWorldContext->ContextHandle);

    // 执行所有 Action 的三阶段激活
    auto ActivateListOfActions = [&Context](const TArray<UGameFeatureAction*>& ActionList)
    {
        for (UGameFeatureAction* Action : ActionList)
        {
            if (Action != nullptr)
            {
                Action->OnGameFeatureRegistering();   // 第一阶段：注册
                Action->OnGameFeatureLoading();        // 第二阶段：加载
                Action->OnGameFeatureActivating(Context); // 第三阶段：激活
            }
        }
    };

    // 先执行 Experience 自身的 Actions
    ActivateListOfActions(CurrentExperience->Actions);
    // 再执行所有 ActionSet 的 Actions
    for (const auto& ActionSet : CurrentExperience->ActionSets)
    {
        if (ActionSet != nullptr)
            ActivateListOfActions(ActionSet->Actions);
    }

    // 标记为已加载
    LoadState = ELyraExperienceLoadState::Loaded;

    // 按优先级顺序广播回调
    OnExperienceLoaded_HighPriority.Broadcast(CurrentExperience);
    OnExperienceLoaded_HighPriority.Clear();

    OnExperienceLoaded.Broadcast(CurrentExperience);
    OnExperienceLoaded.Clear();

    OnExperienceLoaded_LowPriority.Broadcast(CurrentExperience);
    OnExperienceLoaded_LowPriority.Clear();
}
```

**Action 三阶段激活**：
1. `OnGameFeatureRegistering()` — 注册全局数据（如 GameplayCue 路径）
2. `OnGameFeatureLoading()` — 加载阶段准备
3. `OnGameFeatureActivating(Context)` — 实际激活（注入组件、添加 Widget 等）

### 4.7 CallOrRegister 模式 — 时序安全的委托注册

```cpp
void ULyraExperienceManagerComponent::CallOrRegister_OnExperienceLoaded(
    FOnLyraExperienceLoaded::FDelegate&& Delegate)
{
    if (IsExperienceLoaded())
    {
        Delegate.Execute(CurrentExperience);  // 已加载：立即执行
    }
    else
    {
        OnExperienceLoaded.Add(MoveTemp(Delegate));  // 未加载：注册等待
    }
}
```

这是 Lyra 中广泛使用的 **RegisterAndCall 模式**：
- 解决注册时序问题：无论在加载前还是加载后注册，都能正确触发
- 三个优先级变体共享相同模式

### 4.8 EndPlay — 卸载流程

```cpp
void ULyraExperienceManagerComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    Super::EndPlay(EndPlayReason);

    // 1. 通知 ExperienceManager 减少插件引用计数
    for (const FString& PluginURL : GameFeaturePluginURLs)
    {
        if (ULyraExperienceManager::RequestToDeactivatePlugin(PluginURL))
        {
            UGameFeaturesSubsystem::Get().DeactivateGameFeaturePlugin(PluginURL);
        }
    }

    // 2. 反向执行 Action 的停用
    if (LoadState == ELyraExperienceLoadState::Loaded)
    {
        LoadState = ELyraExperienceLoadState::Deactivating;
        
        FGameFeatureDeactivatingContext Context(TEXT(""), 
            [this](FStringView) { this->OnActionDeactivationCompleted(); });

        auto DeactivateListOfActions = [&Context](const TArray<UGameFeatureAction*>& ActionList)
        {
            for (UGameFeatureAction* Action : ActionList)
            {
                if (Action)
                {
                    Action->OnGameFeatureDeactivating(Context);  // 停用
                    Action->OnGameFeatureUnregistering();          // 注销
                }
            }
        };

        DeactivateListOfActions(CurrentExperience->Actions);
        for (const auto& ActionSet : CurrentExperience->ActionSets)
        {
            if (ActionSet != nullptr)
                DeactivateListOfActions(ActionSet->Actions);
        }
    }
}
```

### 4.9 网络复制

```cpp
// 构造函数
ULyraExperienceManagerComponent::ULyraExperienceManagerComponent(...)
{
    SetIsReplicatedByDefault(true);  // 默认开启复制
}

// 复制属性
void ULyraExperienceManagerComponent::GetLifetimeReplicatedProps(...)
{
    DOREPLIFETIME(ThisClass, CurrentExperience);  // 复制 Experience 指针
}

// 客户端收到复制后
void ULyraExperienceManagerComponent::OnRep_CurrentExperience()
{
    StartExperienceLoad();  // 客户端也执行完整加载流程
}
```

**网络流程**：
1. 服务器 `SetCurrentExperience()` → 设置 `CurrentExperience` → 触发网络复制
2. 客户端 `OnRep_CurrentExperience()` → `StartExperienceLoad()` → 完整异步加载
3. 客户端和服务器**独立加载**，但加载相同的 Experience 定义

### 4.10 Loading Screen 集成

```cpp
bool ULyraExperienceManagerComponent::ShouldShowLoadingScreen(FString& OutReason) const
{
    if (LoadState != ELyraExperienceLoadState::Loaded)
    {
        OutReason = TEXT("Experience still loading");
        return true;   // 未加载完成时保持 Loading Screen
    }
    return false;
}
```

通过实现 `ILoadingProcessInterface`，Experience 加载过程中会自动显示 Loading Screen。

### 4.11 混沌测试支持

```cpp
namespace LyraConsoleVariables
{
    static float ExperienceLoadRandomDelayMin = 0.0f;
    static FAutoConsoleVariableRef CVarExperienceLoadRandomDelayMin(
        TEXT("lyra.chaos.ExperienceDelayLoad.MinSecs"), ...);

    static float ExperienceLoadRandomDelayRange = 0.0f;
    static FAutoConsoleVariableRef CVarExperienceLoadRandomDelayRange(
        TEXT("lyra.chaos.ExperienceDelayLoad.RandomSecs"), ...);
}
```

通过控制台变量可注入随机延迟，用于测试异步加载下的竞态条件。

---

## 5. ALyraGameMode — Experience 选择与启动

**文件**: `Source/LyraGame/GameModes/LyraGameMode.h/.cpp`

`ALyraGameMode` 是 Experience 系统的**入口**，负责确定使用哪个 Experience。

### 5.1 Experience 选择优先级链

```cpp
void ALyraGameMode::HandleMatchAssignmentIfNotExpectingOne()
{
    FPrimaryAssetId ExperienceId;
    FString ExperienceIdSource;

    // 优先级从高到低：
    // 1. URL Options (如 ?Experience=B_ShooterExperience)
    if (UGameplayStatics::HasOption(OptionsString, TEXT("Experience")))
    {
        const FString ExperienceFromOptions = UGameplayStatics::ParseOption(OptionsString, TEXT("Experience"));
        ExperienceId = FPrimaryAssetId(FPrimaryAssetType(ULyraExperienceDefinition::StaticClass()->GetFName()), 
                                        FName(*ExperienceFromOptions));
        ExperienceIdSource = TEXT("OptionsString");
    }

    // 2. Developer Settings (PIE only)
    if (!ExperienceId.IsValid() && World->IsPlayInEditor())
    {
        ExperienceId = GetDefault<ULyraDeveloperSettings>()->ExperienceOverride;
        ExperienceIdSource = TEXT("DeveloperSettings");
    }

    // 3. Command Line (如 -Experience=B_ShooterExperience)
    if (!ExperienceId.IsValid())
    {
        FString ExperienceFromCommandLine;
        if (FParse::Value(FCommandLine::Get(), TEXT("Experience="), ExperienceFromCommandLine))
        {
            ExperienceId = ...;
            ExperienceIdSource = TEXT("CommandLine");
        }
    }

    // 4. World Settings (地图级默认)
    if (!ExperienceId.IsValid())
    {
        if (ALyraWorldSettings* TypedWorldSettings = Cast<ALyraWorldSettings>(GetWorldSettings()))
        {
            ExperienceId = TypedWorldSettings->GetDefaultGameplayExperience();
            ExperienceIdSource = TEXT("WorldSettings");
        }
    }

    // 5. Dedicated Server 登录流程
    // 6. 硬编码默认值 B_LyraDefaultExperience
    if (!ExperienceId.IsValid())
    {
        ExperienceId = FPrimaryAssetId(FPrimaryAssetType("LyraExperienceDefinition"), 
                                        FName("B_LyraDefaultExperience"));
        ExperienceIdSource = TEXT("Default");
    }

    OnMatchAssignmentGiven(ExperienceId, ExperienceIdSource);
}
```

**优先级总结**：

| 优先级 | 来源 | 适用场景 |
|--------|------|----------|
| 1（最高） | URL Options | 在线匹配、Session 参数传递 |
| 2 | DeveloperSettings | PIE 开发调试 |
| 3 | Command Line | 启动参数测试 |
| 4 | World Settings | 地图级默认配置 |
| 5 | Dedicated Server | 服务器自动选择 |
| 6（最低） | 硬编码默认 | 兜底 `B_LyraDefaultExperience` |

### 5.2 InitGame — 延迟一帧启动

```cpp
void ALyraGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
    Super::InitGame(MapName, Options, ErrorMessage);
    
    // 等到下一帧才开始选择 Experience，给其他初始化一个机会
    GetWorld()->GetTimerManager().SetTimerForNextTick(this, &ThisClass::HandleMatchAssignmentIfNotExpectingOne);
}
```

### 5.3 InitGameState — 监听 Experience 加载完成

```cpp
void ALyraGameMode::InitGameState()
{
    Super::InitGameState();

    // 注册 Experience 加载完成回调
    ULyraExperienceManagerComponent* ExperienceComponent = 
        GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
    ExperienceComponent->CallOrRegister_OnExperienceLoaded(
        FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}
```

### 5.4 OnExperienceLoaded — 生成等待中的玩家

```cpp
void ALyraGameMode::OnExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience)
{
    // 为所有已连接但还没有 Pawn 的玩家生成 Pawn
    for (FConstPlayerControllerIterator Iterator = GetWorld()->GetPlayerControllerIterator(); Iterator; ++Iterator)
    {
        APlayerController* PC = Cast<APlayerController>(*Iterator);
        if (PC && PC->GetPawn() == nullptr && PlayerCanRestart(PC))
        {
            RestartPlayer(PC);
        }
    }
}
```

### 5.5 HandleStartingNewPlayer — 延迟生成

```cpp
void ALyraGameMode::HandleStartingNewPlayer_Implementation(APlayerController* NewPlayer)
{
    // Experience 未加载时不生成玩家，等 OnExperienceLoaded 统一生成
    if (IsExperienceLoaded())
    {
        Super::HandleStartingNewPlayer_Implementation(NewPlayer);
    }
}
```

### 5.6 Pawn 生成流程

```cpp
APawn* ALyraGameMode::SpawnDefaultPawnAtTransform_Implementation(AController* NewPlayer, const FTransform& SpawnTransform)
{
    if (UClass* PawnClass = GetDefaultPawnClassForController(NewPlayer))
    {
        FActorSpawnParameters SpawnInfo;
        SpawnInfo.bDeferConstruction = true;  // 延迟构造

        if (APawn* SpawnedPawn = GetWorld()->SpawnActor<APawn>(PawnClass, SpawnTransform, SpawnInfo))
        {
            // 在 FinishSpawning 之前设置 PawnData
            if (ULyraPawnExtensionComponent* PawnExtComp = 
                    ULyraPawnExtensionComponent::FindPawnExtensionComponent(SpawnedPawn))
            {
                if (const ULyraPawnData* PawnData = GetPawnDataForController(NewPlayer))
                {
                    PawnExtComp->SetPawnData(PawnData);  // 在构造完成前注入数据
                }
            }
            SpawnedPawn->FinishSpawning(SpawnTransform);
            return SpawnedPawn;
        }
    }
    return nullptr;
}
```

**关键技巧**：使用 `bDeferConstruction = true` 延迟构造，在 `FinishSpawning` 前注入 `PawnData`，确保组件初始化时数据已就位。

### 5.7 GetPawnDataForController — PawnData 获取优先级

```cpp
const ULyraPawnData* ALyraGameMode::GetPawnDataForController(const AController* InController) const
{
    // 1. 优先使用 PlayerState 上已设置的 PawnData
    if (const ALyraPlayerState* LyraPS = InController->GetPlayerState<ALyraPlayerState>())
    {
        if (const ULyraPawnData* PawnData = LyraPS->GetPawnData<ULyraPawnData>())
            return PawnData;
    }

    // 2. 使用 Experience 的 DefaultPawnData
    if (ExperienceComponent->IsExperienceLoaded())
    {
        const ULyraExperienceDefinition* Experience = ExperienceComponent->GetCurrentExperienceChecked();
        if (Experience->DefaultPawnData != nullptr)
            return Experience->DefaultPawnData;

        // 3. 兜底：AssetManager 的全局默认
        return ULyraAssetManager::Get().GetDefaultPawnData();
    }

    return nullptr;  // Experience 未加载
}
```

---

## 6. Experience 加载状态机详解

### 6.1 完整状态转换图

```
                    SetCurrentExperience()
                           │
                           ▼
                    ┌──────────────┐
                    │   Unloaded   │ ← 初始状态 / OnAllActionsDeactivated
                    └──────┬───────┘
                           │ StartExperienceLoad()
                           ▼
                    ┌──────────────┐
                    │   Loading    │ 异步加载 Experience + ActionSet Asset Bundles
                    └──────┬───────┘
                           │ OnExperienceLoadComplete()
                           ▼
               ┌───────────────────────┐
               │  LoadingGameFeatures  │ 并行加载所有 GameFeature 插件
               └───────────┬───────────┘
                           │ 所有插件加载完成 (NumGameFeaturePluginsLoading == 0)
                           ▼
          ┌────────────────────────────────────┐
          │  LoadingChaosTestingDelay (可选)    │ 混沌测试随机延迟
          └────────────────┬───────────────────┘
                           │ 延迟结束
                           ▼
               ┌───────────────────────┐
               │   ExecutingActions    │ 执行 Action: Registering → Loading → Activating
               └───────────┬───────────┘
                           │ 所有 Action 执行完毕
                           ▼
                    ┌──────────────┐
                    │    Loaded    │ 广播三级优先级回调
                    └──────┬───────┘
                           │ EndPlay()
                           ▼
               ┌───────────────────────┐
               │    Deactivating       │ Action: Deactivating → Unregistering
               └───────────┬───────────┘
                           │ OnAllActionsDeactivated()
                           ▼
                    ┌──────────────┐
                    │   Unloaded   │
                    └──────────────┘
```

### 6.2 服务器 vs 客户端流程对比

| 阶段 | 服务器 | 客户端 |
|------|--------|--------|
| Experience 选择 | `GameMode::HandleMatchAssignment` | — |
| Experience 设置 | `SetCurrentExperience()` | `OnRep_CurrentExperience()` |
| 资产加载 | `StartExperienceLoad()` | `StartExperienceLoad()` |
| GameFeature 加载 | `LoadAndActivateGameFeaturePlugin` | `LoadAndActivateGameFeaturePlugin` |
| Action 执行 | `Registering→Loading→Activating` | `Registering→Loading→Activating` |
| 回调广播 | `Broadcast(High→Normal→Low)` | `Broadcast(High→Normal→Low)` |

服务器和客户端**独立执行完整加载流程**，但共享相同的 Experience 定义。

### 6.3 回调优先级使用惯例

| 优先级 | 使用者 | 用途 |
|--------|--------|------|
| **High** | `LyraFrontendStateComponent` | 前端 UI 流程初始化（需要最先执行） |
| **Normal** | `LyraGameMode` / `LyraPlayerState` / `LyraTeamCreationComponent` | 核心游戏逻辑初始化 |
| **Low** | `LyraBotCreationComponent` | Bot 创建（依赖团队等已初始化） |

---

## 7. ULyraExperienceManager — PIE 多实例协调

**文件**: `Source/LyraGame/GameModes/LyraExperienceManager.h/.cpp`

`ULyraExperienceManager` 是一个 **EngineSubsystem**（进程级别单例），专门解决 PIE（Play In Editor）多实例场景下的 GameFeature 插件管理问题。

### 7.1 问题背景

PIE 中可以同时运行多个 World（Server + Client1 + Client2...），它们可能使用相同的 Experience，因此会请求激活相同的 GameFeature 插件。但 GameFeature 系统是进程级的，不能在 World1 卸载后就直接停用插件——因为 World2 可能还在使用。

### 7.2 引用计数机制

```cpp
UCLASS(MinimalAPI)
class ULyraExperienceManager : public UEngineSubsystem
{
    // 每个插件 URL → 活跃请求计数
    TMap<FString, int32> GameFeaturePluginRequestCountMap;
};

void ULyraExperienceManager::NotifyOfPluginActivation(const FString PluginURL)
{
    if (GIsEditor)
    {
        int32& Count = ExperienceManagerSubsystem->GameFeaturePluginRequestCountMap.FindOrAdd(PluginURL);
        ++Count;  // 引用计数 +1
    }
}

bool ULyraExperienceManager::RequestToDeactivatePlugin(const FString PluginURL)
{
    if (GIsEditor)
    {
        int32& Count = ExperienceManagerSubsystem->GameFeaturePluginRequestCountMap.FindChecked(PluginURL);
        --Count;  // 引用计数 -1
        
        if (Count == 0)
        {
            ExperienceManagerSubsystem->GameFeaturePluginRequestCountMap.Remove(PluginURL);
            return true;   // 最后一个引用：允许真正停用
        }
        return false;  // 还有其他引用：不停用
    }
    return true;  // 非编辑器环境：总是停用
}
```

### 7.3 FILO（先进后出）语义

```
PIE Server 启动 → 激活 ShooterCore (Count: 1)
PIE Client 启动 → 激活 ShooterCore (Count: 2)
PIE Client 结束 → RequestToDeactivate → Count=1 → 不停用
PIE Server 结束 → RequestToDeactivate → Count=0 → 真正停用
```

---

## 8. ULyraUserFacingExperienceDefinition — 用户界面体验描述

**文件**: `Source/LyraGame/GameModes/LyraUserFacingExperienceDefinition.h/.cpp`

这是面向**玩家 UI** 的 Experience 描述，用于前端菜单展示和 Session 创建。

### 8.1 数据结构

```cpp
UCLASS(BlueprintType)
class ULyraUserFacingExperienceDefinition : public UPrimaryDataAsset
{
    // 地图 ID
    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience, meta=(AllowedTypes="Map"))
    FPrimaryAssetId MapID;

    // 对应的 Experience 定义 ID
    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience, meta=(AllowedTypes="LyraExperienceDefinition"))
    FPrimaryAssetId ExperienceID;

    // 额外 URL 参数
    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
    TMap<FString, FString> ExtraArgs;

    // UI 展示信息
    UPROPERTY(BlueprintReadWrite, EditAnywhere) FText TileTitle;
    UPROPERTY(BlueprintReadWrite, EditAnywhere) FText TileSubTitle;
    UPROPERTY(BlueprintReadWrite, EditAnywhere) FText TileDescription;
    UPROPERTY(BlueprintReadWrite, EditAnywhere) TObjectPtr<UTexture2D> TileIcon;

    // 加载界面 Widget
    UPROPERTY(BlueprintReadWrite, EditAnywhere) TSoftClassPtr<UUserWidget> LoadingScreenWidget;

    // 配置
    UPROPERTY() bool bIsDefaultExperience = false;
    UPROPERTY() bool bShowInFrontEnd = true;
    UPROPERTY() bool bRecordReplay = false;
    UPROPERTY() int32 MaxPlayerCount = 16;
};
```

### 8.2 CreateHostingRequest — 创建 Session 请求

```cpp
UCommonSession_HostSessionRequest* ULyraUserFacingExperienceDefinition::CreateHostingRequest(
    const UObject* WorldContextObject) const
{
    UCommonSession_HostSessionRequest* Result = Subsystem->CreateOnlineHostSessionRequest();
    
    Result->MapID = MapID;
    Result->ModeNameForAdvertisement = UserFacingExperienceName;
    Result->ExtraArgs = ExtraArgs;
    Result->ExtraArgs.Add(TEXT("Experience"), ExperienceName);  // 关键：将 Experience 名注入 URL
    Result->MaxPlayerCount = MaxPlayerCount;

    if (bRecordReplay)
        Result->ExtraArgs.Add(TEXT("DemoRec"), FString());

    return Result;
}
```

**流程链**：
```
前端菜单 → 选择 UserFacingExperience → CreateHostingRequest()
    → Session 参数中包含 ?Experience=xxx
        → ServerTravel 到目标地图
            → GameMode::HandleMatchAssignment() → 从 URL Options 提取 Experience
```

---

## 9. UAsyncAction_ExperienceReady — 蓝图异步等待

**文件**: `Source/LyraGame/GameModes/AsyncAction_ExperienceReady.h/.cpp`

这是一个 **蓝图异步动作**，提供给蓝图和 Widget 使用的 Experience 等待机制。

### 9.1 类声明

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FExperienceReadyAsyncDelegate);

UCLASS()
class UAsyncAction_ExperienceReady : public UBlueprintAsyncActionBase
{
    GENERATED_UCLASS_BODY()

public:
    // 蓝图节点：等待 Experience 就绪
    UFUNCTION(BlueprintCallable, meta=(WorldContext = "WorldContextObject", BlueprintInternalUseOnly="true"))
    static UAsyncAction_ExperienceReady* WaitForExperienceReady(UObject* WorldContextObject);

    virtual void Activate() override;

    // 蓝图输出引脚
    UPROPERTY(BlueprintAssignable)
    FExperienceReadyAsyncDelegate OnReady;

private:
    void Step1_HandleGameStateSet(AGameStateBase* GameState);
    void Step2_ListenToExperienceLoading(AGameStateBase* GameState);
    void Step3_HandleExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience);
    void Step4_BroadcastReady();

    TWeakObjectPtr<UWorld> WorldPtr;
};
```

### 9.2 四步异步流程

**Step 0: 工厂函数**

```cpp
UAsyncAction_ExperienceReady* UAsyncAction_ExperienceReady::WaitForExperienceReady(UObject* InWorldContextObject)
{
    UAsyncAction_ExperienceReady* Action = nullptr;
    if (UWorld* World = GEngine->GetWorldFromContextObject(InWorldContextObject, EGetWorldErrorMode::LogAndReturnNull))
    {
        Action = NewObject<UAsyncAction_ExperienceReady>();
        Action->WorldPtr = World;
        Action->RegisterWithGameInstance(World);  // 生命周期绑定到 GameInstance
    }
    return Action;
}
```

**Step 1: 等待 GameState 就绪**

```cpp
void UAsyncAction_ExperienceReady::Activate()
{
    if (UWorld* World = WorldPtr.Get())
    {
        if (AGameStateBase* GameState = World->GetGameState())
        {
            Step2_ListenToExperienceLoading(GameState);  // GameState 已存在，直接进入下一步
        }
        else
        {
            // GameState 尚未创建，监听创建事件
            World->GameStateSetEvent.AddUObject(this, &ThisClass::Step1_HandleGameStateSet);
        }
    }
    else
    {
        SetReadyToDestroy();  // 无 World，直接销毁
    }
}

void UAsyncAction_ExperienceReady::Step1_HandleGameStateSet(AGameStateBase* GameState)
{
    if (UWorld* World = WorldPtr.Get())
    {
        World->GameStateSetEvent.RemoveAll(this);  // 取消监听
    }
    Step2_ListenToExperienceLoading(GameState);
}
```

**Step 2: 监听 Experience 加载**

```cpp
void UAsyncAction_ExperienceReady::Step2_ListenToExperienceLoading(AGameStateBase* GameState)
{
    check(GameState);
    ULyraExperienceManagerComponent* ExperienceComponent = 
        GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
    check(ExperienceComponent);

    if (ExperienceComponent->IsExperienceLoaded())
    {
        UWorld* World = GameState->GetWorld();
        check(World);

        // 【关键设计】即使已加载完成，也延迟一帧广播
        // 防止使用者依赖"总是同步返回"的假设
        World->GetTimerManager().SetTimerForNextTick(
            FTimerDelegate::CreateUObject(this, &ThisClass::Step4_BroadcastReady));
    }
    else
    {
        // 尚未加载，使用 RegisterAndCall 模式监听
        ExperienceComponent->CallOrRegister_OnExperienceLoaded(
            FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::Step3_HandleExperienceLoaded));
    }
}
```

**Step 3 & 4: 广播就绪**

```cpp
void UAsyncAction_ExperienceReady::Step3_HandleExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience)
{
    Step4_BroadcastReady();  // 直接转发
}

void UAsyncAction_ExperienceReady::Step4_BroadcastReady()
{
    OnReady.Broadcast();    // 触发蓝图输出引脚
    SetReadyToDestroy();    // 自动清理
}
```

### 9.3 设计亮点分析

| 设计点 | 说明 |
|--------|------|
| **四步防御式编程** | 每一步都处理"目标可能还不存在"的情况 |
| **延迟一帧广播** | 即使 Experience 已加载，也 `SetTimerForNextTick`，确保调用者永远在异步路径上收到回调 |
| **RegisterWithGameInstance** | 将 AsyncAction 生命周期绑定到 GameInstance，防止中途被 GC |
| **SetReadyToDestroy** | 广播后立即标记可销毁，防止内存泄漏 |
| **弱指针 WorldPtr** | 使用 `TWeakObjectPtr<UWorld>` 防止 World 被销毁后访问野指针 |

### 9.4 蓝图使用方式

```
[WaitForExperienceReady] ──── OnReady ────→ [初始化 UI/逻辑]
```

蓝图中只需拖出一个 `WaitForExperienceReady` 节点，从 `OnReady` 引脚连接后续逻辑即可。这是 Lyra 中**蓝图与 Experience 系统交互的标准方式**。

---

## 10. Experience 消费者 — 各子系统如何监听

Experience 系统的核心价值在于：多个子系统可以**安全地等待 Experience 加载完成后再初始化**。下面分析所有主要消费者的接入模式。

### 10.1 通用接入模式

```cpp
// 标准模式：在 BeginPlay/PostInitializeComponents 中注册
void SomeComponent::BeginPlay()
{
    Super::BeginPlay();
    
    AGameStateBase* GameState = GetWorld()->GetGameState();
    ULyraExperienceManagerComponent* ExperienceComp = 
        GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
    
    ExperienceComp->CallOrRegister_OnExperienceLoaded[_优先级](
        FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}
```

### 10.2 消费者清单

#### （1）ALyraPlayerState — Normal 优先级

```cpp
// Source/LyraGame/Player/LyraPlayerState.cpp
void ALyraPlayerState::PostInitializeComponents()
{
    Super::PostInitializeComponents();

    check(AbilitySystemComponent);
    AbilitySystemComponent->InitAbilityActorInfo(this, GetPawn());

    UWorld* World = GetWorld();
    if (World && World->IsGameWorld() && World->GetNetMode() != NM_Client)
    {
        AGameStateBase* GameState = GetWorld()->GetGameState();
        check(GameState);
        ULyraExperienceManagerComponent* ExperienceComponent = 
            GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
        check(ExperienceComponent);
        // Normal 优先级（默认）
        ExperienceComponent->CallOrRegister_OnExperienceLoaded(
            FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
    }
}

void ALyraPlayerState::OnExperienceLoaded(const ULyraExperienceDefinition* /*CurrentExperience*/)
{
    if (ALyraGameMode* LyraGameMode = GetWorld()->GetAuthGameMode<ALyraGameMode>())
    {
        // 从 GameMode 获取 PawnData 并设置到 PlayerState 上
        if (const ULyraPawnData* NewPawnData = LyraGameMode->GetPawnDataForController(GetOwningController()))
        {
            SetPawnData(NewPawnData);
        }
    }
}
```

**关键点**：
- 仅在**服务器端**注册（`GetNetMode() != NM_Client`）
- `SetPawnData()` 会立即将 PawnData 的 AbilitySets 注入到 ASC 中
- PawnData 通过网络复制传递给客户端

#### （2）ULyraBotCreationComponent — Low 优先级

```cpp
// Source/LyraGame/GameModes/LyraBotCreationComponent.cpp
void ULyraBotCreationComponent::BeginPlay()
{
    Super::BeginPlay();

    AGameStateBase* GameState = GetGameStateChecked<AGameStateBase>();
    ULyraExperienceManagerComponent* ExperienceComponent = 
        GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
    check(ExperienceComponent);
    // Low 优先级 — 确保团队系统等已初始化
    ExperienceComponent->CallOrRegister_OnExperienceLoaded_LowPriority(
        FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}

void ULyraBotCreationComponent::OnExperienceLoaded(const ULyraExperienceDefinition* Experience)
{
#if WITH_SERVER_CODE
    if (HasAuthority())
    {
        ServerCreateBots();  // 创建 AI Bot
    }
#endif
}
```

**关键点**：
- 使用 **Low 优先级**，因为 Bot 生成依赖团队分配等已完成
- 仅在**有 Authority 的服务器**上执行
- Bot 数量可被 DeveloperSettings 或 URL Options 覆盖
- `ServerCreateBots()` 是 `BlueprintNativeEvent`，子类可在蓝图中覆写

#### （3）ALyraGameMode — Normal 优先级

```cpp
// Source/LyraGame/GameModes/LyraGameMode.cpp
void ALyraGameMode::InitGameState()
{
    Super::InitGameState();

    ULyraExperienceManagerComponent* ExperienceComponent = 
        GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
    // Normal 优先级
    ExperienceComponent->CallOrRegister_OnExperienceLoaded(
        FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}

void ALyraGameMode::OnExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience)
{
    // 遍历所有已连接的 PlayerController，为没有 Pawn 的玩家生成 Pawn
    for (FConstPlayerControllerIterator Iterator = GetWorld()->GetPlayerControllerIterator(); Iterator; ++Iterator)
    {
        APlayerController* PC = Cast<APlayerController>(*Iterator);
        if (PC && PC->GetPawn() == nullptr && PlayerCanRestart(PC))
        {
            RestartPlayer(PC);
        }
    }
}
```

**关键点**：
- Experience 加载完成前，`HandleStartingNewPlayer` 会**跳过**玩家生成
- 加载完成后，统一为所有等待中的玩家 `RestartPlayer`

#### （4）ULyraFrontendStateComponent — High 优先级

```cpp
// Source/LyraGame/UI/Frontend/LyraFrontendStateComponent.cpp
void ULyraFrontendStateComponent::BeginPlay()
{
    Super::BeginPlay();

    ULyraExperienceManagerComponent* ExperienceComponent = ...;
    // High 优先级 — 前端 UI 需要最先初始化
    ExperienceComponent->CallOrRegister_OnExperienceLoaded_HighPriority(
        FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}
```

**关键点**：
- 使用 **High 优先级**，确保前端 UI 流程在其他系统之前启动
- 负责启动 ControlFlow：用户初始化 → 按下开始 → 加入 Session → 主菜单

### 10.3 消费者优先级汇总表

| 消费者 | 优先级 | 注册位置 | 回调行为 |
|--------|--------|----------|----------|
| `LyraFrontendStateComponent` | **High** | `BeginPlay` | 启动前端 ControlFlow |
| `LyraGameMode` | Normal | `InitGameState` | RestartPlayer 为等待中的玩家生成 Pawn |
| `LyraPlayerState` | Normal | `PostInitializeComponents` | 获取并设置 PawnData + AbilitySets |
| `LyraTeamCreationComponent` | Normal | `BeginPlay` | 创建团队 |
| `LyraBotCreationComponent` | **Low** | `BeginPlay` | 创建 AI Bot |

### 10.4 时序图

```
Experience Loaded!
    │
    ├── [High Priority] ─────────────────────────────────────┐
    │   └── FrontendStateComponent::OnExperienceLoaded()     │
    │       └── 启动前端 ControlFlow                          │
    │                                                        │
    ├── [Normal Priority] ───────────────────────────────────┤
    │   ├── GameMode::OnExperienceLoaded()                   │
    │   │   └── RestartPlayer() for waiting players          │
    │   ├── PlayerState::OnExperienceLoaded()                │
    │   │   └── SetPawnData() + GiveAbilities()              │
    │   └── TeamCreationComponent::OnExperienceLoaded()      │
    │       └── CreateTeams()                                │
    │                                                        │
    └── [Low Priority] ─────────────────────────────────────┘
        └── BotCreationComponent::OnExperienceLoaded()
            └── ServerCreateBots()
```

---

## 11. ALyraWorldSettings — 地图默认体验

**文件**: `Source/LyraGame/GameModes/LyraWorldSettings.h/.cpp`

### 11.1 类定义

```cpp
UCLASS(MinimalAPI)
class ALyraWorldSettings : public AWorldSettings
{
    GENERATED_BODY()

public:
    UE_API ALyraWorldSettings(const FObjectInitializer& ObjectInitializer);

#if WITH_EDITOR
    UE_API virtual void CheckForErrors() override;
#endif

    // 返回地图默认 Experience 的 PrimaryAssetId
    UE_API FPrimaryAssetId GetDefaultGameplayExperience() const;

protected:
    // 地图级默认 Experience（软引用，避免强依赖）
    UPROPERTY(EditDefaultsOnly, Category=GameMode)
    TSoftClassPtr<ULyraExperienceDefinition> DefaultGameplayExperience;

#if WITH_EDITORONLY_DATA
    // 前端/独立体验的地图：PIE 中强制 Standalone 网络模式
    UPROPERTY(EditDefaultsOnly, Category=PIE)
    bool ForceStandaloneNetMode = false;
#endif
};
```

### 11.2 GetDefaultGameplayExperience

```cpp
FPrimaryAssetId ALyraWorldSettings::GetDefaultGameplayExperience() const
{
    FPrimaryAssetId Result;
    if (!DefaultGameplayExperience.IsNull())
    {
        // 通过 AssetManager 将软引用转换为 PrimaryAssetId
        Result = UAssetManager::Get().GetPrimaryAssetIdForPath(
            DefaultGameplayExperience.ToSoftObjectPath());

        if (!Result.IsValid())
        {
            UE_LOG(LogLyraExperience, Error, 
                TEXT("%s.DefaultGameplayExperience is %s but that failed to resolve into an asset ID "
                     "(you might need to add a path to the Asset Rules in your game feature plugin or project settings"),
                *GetPathNameSafe(this), *DefaultGameplayExperience.ToString());
        }
    }
    return Result;
}
```

**关键设计**：
- 使用 `TSoftClassPtr` 而非硬引用，避免地图加载时强制加载 Experience 资产
- 通过 `AssetManager::GetPrimaryAssetIdForPath()` 转换，与 Experience 系统的 `FPrimaryAssetId` 体系对齐
- 这是 Experience 选择优先级链中**第 4 级**来源

### 11.3 编辑器错误检查

```cpp
#if WITH_EDITOR
void ALyraWorldSettings::CheckForErrors()
{
    Super::CheckForErrors();

    FMessageLog MapCheck("MapCheck");
    
    for (TActorIterator<APlayerStart> PlayerStartIt(GetWorld()); PlayerStartIt; ++PlayerStartIt)
    {
        APlayerStart* PlayerStart = *PlayerStartIt;
        if (IsValid(PlayerStart) && PlayerStart->GetClass() == APlayerStart::StaticClass())
        {
            MapCheck.Warning()
                ->AddToken(FUObjectToken::Create(PlayerStart))
                ->AddToken(FTextToken::Create(
                    FText::FromString("is a normal APlayerStart, replace with ALyraPlayerStart.")));
        }
    }
}
#endif
```

在编辑器中会检查地图中的 PlayerStart 是否使用了 `ALyraPlayerStart`（支持 Experience 标签匹配），而非原生 `APlayerStart`。

---

## 12. ULyraAssetManager — 资产加载基础设施

**文件**: `Source/LyraGame/System/LyraAssetManager.h/.cpp`

### 12.1 类概述

```cpp
UCLASS(MinimalAPI, Config = Game)
class ULyraAssetManager : public UAssetManager
{
    GENERATED_BODY()

public:
    static UE_API ULyraAssetManager& Get();  // 单例访问

    // 同步加载（带日志）
    template<typename AssetType>
    static AssetType* GetAsset(const TSoftObjectPtr<AssetType>& AssetPointer, bool bKeepInMemory = true);

    // 子类加载
    template<typename AssetType>
    static TSubclassOf<AssetType> GetSubclass(const TSoftClassPtr<AssetType>& AssetPointer, bool bKeepInMemory = true);

    UE_API const ULyraGameData& GetGameData();
    UE_API const ULyraPawnData* GetDefaultPawnData() const;

protected:
    virtual void StartInitialLoading() override;

    // 全局默认 PawnData（Config 配置，DefaultGame.ini 中指定）
    UPROPERTY(Config)
    TSoftObjectPtr<ULyraPawnData> DefaultPawnData;

private:
    // 启动任务列表
    TArray<FLyraAssetManagerStartupJob> StartupJobs;
    
    // 已加载资产追踪（线程安全）
    UPROPERTY()
    TSet<TObjectPtr<const UObject>> LoadedAssets;
    FCriticalSection LoadedAssetsCritical;
};
```

### 12.2 与 Experience 系统的关系

#### （1）DefaultPawnData — 最终兜底

```cpp
const ULyraPawnData* ULyraAssetManager::GetDefaultPawnData() const
{
    return GetAsset(DefaultPawnData);  // 同步加载 Config 中配置的默认 PawnData
}
```

这是 `GetPawnDataForController()` 三级优先级的**最后一级兜底**：
```
PlayerState::PawnData → Experience::DefaultPawnData → AssetManager::DefaultPawnData
```

#### （2）StartInitialLoading — 引擎启动阶段

```cpp
void ULyraAssetManager::StartInitialLoading()
{
    SCOPED_BOOT_TIMING("ULyraAssetManager::StartInitialLoading");
    Super::StartInitialLoading();  // 执行资产扫描

    STARTUP_JOB(InitializeGameplayCueManager());  // GAS 初始化
    STARTUP_JOB_WEIGHTED(GetGameData(), 25.f);     // 加载全局游戏数据

    DoAllStartupJobs();  // 执行所有启动任务
}
```

#### （3）Asset Bundle 加载

Experience 加载流程中，`ULyraExperienceManagerComponent::StartExperienceLoad()` 会调用：

```cpp
// 在 ExperienceManagerComponent 中调用
TArray<FPrimaryAssetId> BundleAssetList;
BundleAssetList.Add(CurrentExperience->GetPrimaryAssetId());
// 加入所有 ActionSet 的 AssetId
for (const auto* ActionSet : CurrentExperience->ActionSets)
{
    BundleAssetList.Add(ActionSet->GetPrimaryAssetId());
}

// 按角色加载不同的 Asset Bundle
TArray<FName> BundlesToLoad;
const ENetMode OwnerNetMode = GetOwner()->GetNetMode();
const bool bLoadClient = (OwnerNetMode != NM_DedicatedServer);
const bool bLoadServer = (OwnerNetMode != NM_Client);
if (bLoadClient) BundlesToLoad.Add(UGameFeaturesSubsystemSettings::LoadStateClient);
if (bLoadServer) BundlesToLoad.Add(UGameFeaturesSubsystemSettings::LoadStateServer);

// 异步加载 Asset Bundle
ChangeBundleStateForPrimaryAssets(BundleAssetList, BundlesToLoad, ...);
```

**Asset Bundle 机制**允许 Experience 和 ActionSet 声明依赖的资产，按 Client/Server 角色选择性加载，避免不必要的内存占用。

#### （4）同步加载工具

```cpp
template<typename AssetType>
AssetType* ULyraAssetManager::GetAsset(const TSoftObjectPtr<AssetType>& AssetPointer, bool bKeepInMemory)
{
    AssetType* LoadedAsset = AssetPointer.Get();
    if (!LoadedAsset)
    {
        LoadedAsset = Cast<AssetType>(SynchronousLoadAsset(AssetPath));
    }
    if (LoadedAsset && bKeepInMemory)
    {
        Get().AddLoadedAsset(Cast<UObject>(LoadedAsset));  // 防止 GC
    }
    return LoadedAsset;
}
```

`bKeepInMemory = true` 时，资产会被加入 `LoadedAssets` 集合中，防止被垃圾回收。

#### （5）PIE 预加载

```cpp
#if WITH_EDITOR
void ULyraAssetManager::PreBeginPIE(bool bStartSimulate)
{
    Super::PreBeginPIE(bStartSimulate);

    // PIE 开始前预加载全局数据
    const ULyraGameData& LocalGameDataCommon = GetGameData();
    
    // 可在此处根据 DeveloperSettings 预加载 Experience 所需资产
}
#endif
```

---

## 13. 完整架构图与数据流

### 13.1 系统全景图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Experience System Architecture                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────┐    ┌────────────────────────────────┐         │
│  │  UserFacingExperience   │    │     LyraWorldSettings          │         │
│  │  (前端菜单展示)          │    │  (DefaultGameplayExperience)   │         │
│  │  MapID + ExperienceID   │    └──────────────┬─────────────────┘         │
│  │  TileTitle/Icon/...     │                   │                           │
│  └──────────┬──────────────┘                   │ 优先级 4                   │
│             │                                  │                           │
│             │ CreateHostingRequest()            │                           │
│             ▼                                  ▼                           │
│  ┌──────────────────────────────────────────────────────────────┐         │
│  │              ALyraGameMode                                    │         │
│  │  HandleMatchAssignmentIfNotExpectingOne()                     │         │
│  │  ┌──────────────────────────────────────────────────────┐    │         │
│  │  │ 优先级链: URL > DevSettings > CmdLine > World > Default│    │         │
│  │  └──────────────────────────┬───────────────────────────┘    │         │
│  │                             │ OnMatchAssignmentGiven()        │         │
│  └─────────────────────────────┼────────────────────────────────┘         │
│                                ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐         │
│  │         ExperienceManagerComponent (on GameState)             │         │
│  │                                                               │         │
│  │  SetCurrentExperience() → StartExperienceLoad()               │         │
│  │      │                                                        │         │
│  │      ▼                                                        │         │
│  │  [Loading] → [LoadingGameFeatures] → [ChaosDelay]            │         │
│  │      → [ExecutingActions] → [Loaded]                          │         │
│  │                                │                              │         │
│  │                                ▼                              │         │
│  │     ┌──────────────────────────────────────────────┐         │         │
│  │     │  Three-Priority Callback Broadcast            │         │         │
│  │     │  High → Normal → Low                          │         │         │
│  │     └──────────────────────────────────────────────┘         │         │
│  └──────────────────────────────────────────────────────────────┘         │
│                                │                                           │
│         ┌──────────────────────┼──────────────────────┐                   │
│         ▼                      ▼                      ▼                   │
│  ┌─────────────┐  ┌────────────────────┐  ┌─────────────────────┐       │
│  │ [High]      │  │ [Normal]           │  │ [Low]               │       │
│  │ Frontend    │  │ GameMode           │  │ BotCreation         │       │
│  │ StateComp   │  │ PlayerState        │  │ Component           │       │
│  │             │  │ TeamCreation       │  │                     │       │
│  └─────────────┘  └────────────────────┘  └─────────────────────┘       │
│                                                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                          数据资产层                                         │
│                                                                             │
│  ┌────────────────────────────┐     ┌────────────────────────────┐        │
│  │  ExperienceDefinition      │────▶│  ExperienceActionSet       │        │
│  │  ┌──────────────────────┐  │     │  ┌──────────────────────┐  │        │
│  │  │ GameFeaturesToEnable │  │     │  │ GameFeaturesToEnable │  │        │
│  │  │ DefaultPawnData      │  │     │  │ Actions[]            │  │        │
│  │  │ Actions[]            │  │     │  └──────────────────────┘  │        │
│  │  │ ActionSets[]  ───────│──┘     └────────────────────────────┘        │
│  │  └──────────────────────┘  │                                           │
│  └────────────────────────────┘                                           │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                        基础设施层                                           │
│                                                                             │
│  ┌────────────────────────┐   ┌──────────────────────────────────┐        │
│  │  LyraAssetManager      │   │  LyraExperienceManager           │        │
│  │  (DefaultPawnData)      │   │  (PIE 引用计数)                   │        │
│  │  (StartupJobs)          │   │  (GameFeaturePlugin 生命周期)     │        │
│  │  (Asset Bundle 加载)    │   │                                  │        │
│  └────────────────────────┘   └──────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 13.2 完整生命周期时序

```
[引擎启动]
    │
    ▼
ULyraAssetManager::StartInitialLoading()
    ├── Asset 扫描
    ├── GameplayCueManager 初始化
    └── GameData 加载
    
[地图加载]
    │
    ▼
ALyraGameMode::InitGame()
    └── SetTimerForNextTick → HandleMatchAssignmentIfNotExpectingOne()
        └── 6 级优先级链确定 ExperienceId
            └── OnMatchAssignmentGiven(ExperienceId)
                └── ExperienceManagerComponent::SetCurrentExperience()
                    │
                    ├── [Server] CurrentExperience = Experience
                    │   └── 触发网络复制
                    │
                    └── StartExperienceLoad()
                        │
                        ▼
                    [Loading] 异步加载 Asset Bundles (Client/Server)
                        │
                        ▼
                    [LoadingGameFeatures] 并行加载 GameFeature 插件
                        │ NumGameFeaturePluginsLoading → 0
                        ▼
                    [ChaosTestingDelay] 可选随机延迟
                        │
                        ▼
                    [ExecutingActions] 
                        ├── OnGameFeatureRegistering()
                        ├── OnGameFeatureLoading()
                        └── OnGameFeatureActivating(Context)
                        │
                        ▼
                    [Loaded]
                        ├── Broadcast High   → FrontendState 启动 UI
                        ├── Broadcast Normal → GameMode.RestartPlayer()
                        │                   → PlayerState.SetPawnData()
                        │                   → TeamCreation.CreateTeams()
                        └── Broadcast Low    → BotCreation.ServerCreateBots()
                        
[玩家进入]
    │
    ▼
GameMode::SpawnDefaultPawnAtTransform()
    ├── bDeferConstruction = true
    ├── PawnExtComp->SetPawnData(PawnData)
    └── FinishSpawning()

[地图卸载 / PIE 结束]
    │
    ▼
ExperienceManagerComponent::EndPlay()
    └── [Deactivating]
        ├── OnGameFeatureDeactivating()
        ├── OnGameFeatureUnregistering()
        └── ExperienceManager::RequestToDeactivatePlugin()
            └── 引用计数 == 0 → 真正停用
                └── [Unloaded]
```

---

## 14. 核心设计模式总结

### 14.1 RegisterAndCall（注册并立即调用）模式

```cpp
void CallOrRegister_OnExperienceLoaded(FDelegate&& Delegate)
{
    if (IsExperienceLoaded())
    {
        Delegate.Execute(CurrentExperience);  // 已加载：立即执行
    }
    else
    {
        OnExperienceLoaded.Add(MoveTemp(Delegate));  // 未加载：注册等待
    }
}
```

**问题**：组件初始化时序不确定，Experience 可能已加载也可能未加载。
**解决**：统一入口，无论何时注册都能正确执行。

### 14.2 三级优先级回调

```cpp
// 三个独立委托，按顺序广播
OnExperienceLoaded_HighPriority.Broadcast(CurrentExperience);  // UI 先行
OnExperienceLoaded.Broadcast(CurrentExperience);               // 核心逻辑
OnExperienceLoaded_LowPriority.Broadcast(CurrentExperience);   // 依赖前两级
```

**问题**：不同子系统有初始化依赖关系（如 Bot 依赖团队已创建）。
**解决**：三级广播确保执行顺序，无需子系统之间显式耦合。

### 14.3 FILO 引用计数（PIE 协调）

```cpp
NotifyOfPluginActivation()   → Count++
RequestToDeactivatePlugin()  → Count-- → Count==0 时真正停用
```

**问题**：PIE 多 World 共享 GameFeature 插件，某个 World 退出不应影响其他 World。
**解决**：进程级 EngineSubsystem 管理引用计数，最后一个退出者才执行停用。

### 14.4 延迟构造注入（Deferred Construction）

```cpp
SpawnInfo.bDeferConstruction = true;      // 延迟构造
PawnExtComp->SetPawnData(PawnData);       // 注入数据
SpawnedPawn->FinishSpawning(SpawnTransform);  // 完成构造
```

**问题**：Pawn 组件在 `BeginPlay` 中需要 PawnData，但标准 Spawn 流程中数据注入在 BeginPlay 之后。
**解决**：推迟 `FinishSpawning`，在构造和 BeginPlay 之间注入数据。

### 14.5 组合优于继承（Composition over Inheritance）

```
ExperienceDefinition
    ├── DefaultPawnData
    ├── Actions[] (GameFeatureAction 实例)
    └── ActionSets[] (可复用 Action 集合)
            ├── Actions[]
            └── GameFeaturesToEnable[]
```

**问题**：传统 GameMode 继承导致代码膨胀和耦合。
**解决**：Experience 通过组合 Actions 和 ActionSets 构建游戏规则，不同 Experience 可共享 ActionSets。

### 14.6 数据驱动规则（Data-Driven Rules）

```
传统方式：AMyShooterGameMode extends AMyGameMode  (硬编码)
Lyra 方式：B_ShooterExperience.uasset → Actions[] + ActionSets[]  (数据驱动)
```

**问题**：每种游戏模式都需要一个 C++ GameMode 子类。
**解决**：一个 `ALyraGameMode` + 不同的 Experience 数据资产 = 任意数量的游戏模式。

### 14.7 网络透明加载（Network-Transparent Loading）

```
Server: SetCurrentExperience() → 完整加载流程
        ↓ 网络复制 CurrentExperience
Client: OnRep_CurrentExperience() → 完整加载流程（独立执行）
```

**问题**：客户端和服务器需要加载相同的 Experience，但时序不同。
**解决**：利用 UE 属性复制，客户端收到复制值后**独立执行完整加载管线**。

### 14.8 防御式异步编程

```cpp
// AsyncAction_ExperienceReady 中：
// 即使已加载，也延迟一帧广播
if (ExperienceComponent->IsExperienceLoaded())
{
    World->GetTimerManager().SetTimerForNextTick(...Step4_BroadcastReady);
}
```

**问题**：如果有时同步返回、有时异步返回，调用者很难处理两种情况。
**解决**：**始终异步**返回，消除时序不确定性。

---

## 15. 文件索引

### 15.1 核心文件

| 文件 | 类 | 职责 |
|------|------|------|
| `Source/LyraGame/GameModes/LyraExperienceDefinition.h/.cpp` | `ULyraExperienceDefinition` | Experience 数据资产定义 |
| `Source/LyraGame/GameModes/LyraExperienceActionSet.h/.cpp` | `ULyraExperienceActionSet` | 可复用动作集合 |
| `Source/LyraGame/GameModes/LyraExperienceManagerComponent.h/.cpp` | `ULyraExperienceManagerComponent` | **核心**：加载状态机、回调管理 |
| `Source/LyraGame/GameModes/LyraExperienceManager.h/.cpp` | `ULyraExperienceManager` | PIE 多实例插件引用计数 |
| `Source/LyraGame/GameModes/AsyncAction_ExperienceReady.h/.cpp` | `UAsyncAction_ExperienceReady` | 蓝图异步等待 |
| `Source/LyraGame/GameModes/LyraUserFacingExperienceDefinition.h/.cpp` | `ULyraUserFacingExperienceDefinition` | UI 展示与 Session 创建 |

### 15.2 入口与协调文件

| 文件 | 类 | 与 Experience 的关系 |
|------|------|------|
| `Source/LyraGame/GameModes/LyraGameMode.h/.cpp` | `ALyraGameMode` | Experience 选择优先级链、Pawn 生成 |
| `Source/LyraGame/GameModes/LyraGameState.h/.cpp` | `ALyraGameState` | 持有 ExperienceManagerComponent |
| `Source/LyraGame/GameModes/LyraWorldSettings.h/.cpp` | `ALyraWorldSettings` | 地图级默认 Experience |

### 15.3 消费者文件

| 文件 | 类 | 消费方式 |
|------|------|------|
| `Source/LyraGame/Player/LyraPlayerState.h/.cpp` | `ALyraPlayerState` | Normal 回调 → SetPawnData |
| `Source/LyraGame/GameModes/LyraBotCreationComponent.h/.cpp` | `ULyraBotCreationComponent` | Low 回调 → ServerCreateBots |
| `Source/LyraGame/UI/Frontend/LyraFrontendStateComponent.h/.cpp` | `ULyraFrontendStateComponent` | High 回调 → 启动前端流程 |

### 15.4 基础设施文件

| 文件 | 类 | 职责 |
|------|------|------|
| `Source/LyraGame/System/LyraAssetManager.h/.cpp` | `ULyraAssetManager` | 资产管理、DefaultPawnData、Asset Bundle |
| `Source/LyraGame/Development/LyraDeveloperSettings.h` | `ULyraDeveloperSettings` | PIE ExperienceOverride、Bot 数量覆盖 |

### 15.5 加载状态枚举

| 文件 | 枚举 | 值 |
|------|------|------|
| `Source/LyraGame/GameModes/LyraExperienceManagerComponent.h` | `ELyraExperienceLoadState` | `Unloaded`, `Loading`, `LoadingGameFeatures`, `LoadingChaosTestingDelay`, `ExecutingActions`, `Loaded`, `Deactivating` |
