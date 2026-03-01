# Lyra GamePhase（游戏阶段）子系统 — 代码级深度分析

## 目录

1. [系统概览与架构总图](#1-系统概览与架构总图)
2. [核心类继承体系](#2-核心类继承体系)
3. [GamePhase 核心三件套详解](#3-gamephase-核心三件套详解)
   - 3.1 [ULyraGamePhaseAbility — 阶段能力基类](#31-ulyragamephaseability--阶段能力基类)
   - 3.2 [ULyraGamePhaseSubsystem — 阶段管理子系统](#32-ulyragamephasesubsystem--阶段管理子系统)
   - 3.3 [GamePhaseTag 嵌套层级机制](#33-gamephasetag-嵌套层级机制)
4. [Experience 系统 — GamePhase 的触发源](#4-experience-系统--gamephase-的触发源)
   - 4.1 [ULyraExperienceDefinition — 体验定义](#41-ulyraexperiencedefinition--体验定义)
   - 4.2 [ULyraExperienceManagerComponent — 体验加载状态机](#42-ulyraexperiencemanagercomponent--体验加载状态机)
   - 4.3 [ULyraExperienceActionSet — 可组合的动作集](#43-ulyraexperienceactionset--可组合的动作集)
   - 4.4 [ULyraExperienceManager — PIE 多实例仲裁](#44-ulyraexperiencemanager--pie-多实例仲裁)
5. [GameMode/GameState — 基础设施层](#5-gamemodegamestate--基础设施层)
   - 5.1 [ALyraGameMode — 游戏模式](#51-alyragamemode--游戏模式)
   - 5.2 [ALyraGameState — 游戏状态](#52-alyragamestate--游戏状态)
   - 5.3 [ALyraWorldSettings — 世界设置](#53-alyraworldsettings--世界设置)
6. [GameStateComponent 组件体系](#6-gamestatecomponent-组件体系)
   - 6.1 [ULyraTeamCreationComponent — 队伍创建](#61-ulyrateamcreationcomponent--队伍创建)
   - 6.2 [ULyraBotCreationComponent — Bot 创建](#62-ulyrabotcreationcomponent--bot-创建)
   - 6.3 [ULyraPlayerSpawningManagerComponent — 出生点管理](#63-ulyraplayerspawningmanagercomponent--出生点管理)
   - 6.4 [ULyraFrontendStateComponent — 前端 UI 状态流](#64-ulyrafrontendstatecomponent--前端-ui-状态流)
7. [异步等待机制 — AsyncAction_ExperienceReady](#7-异步等待机制--asyncaction_experienceready)
8. [用户面向的体验定义](#8-用户面向的体验定义)
9. [完整生命周期流程图](#9-完整生命周期流程图)
10. [阶段切换的核心算法详解](#10-阶段切换的核心算法详解)
11. [网络复制策略](#11-网络复制策略)
12. [关键设计模式总结](#12-关键设计模式总结)
13. [源码文件索引](#13-源码文件索引)

---

## 1. 系统概览与架构总图

Lyra 的 GamePhase 子系统是一套**基于 GAS (Gameplay Ability System) + GameplayTag 层级匹配**的游戏阶段管理框架。它将传统游戏中的"等待开始 → 游戏进行中 → 结算"等阶段概念，**用 GameplayAbility 的激活/结束来表达**，实现了以下独特设计：

- **阶段即 Ability**：每个游戏阶段是一个 `ULyraGamePhaseAbility` 的激活实例
- **Tag 层级互斥**：利用 GameplayTag 的层级结构，自动管理兄弟阶段互斥、父子阶段共存
- **观察者模式**：通过 `WhenPhaseStartsOrIsActive` / `WhenPhaseEnds` 提供事件订阅
- **Experience 驱动**：阶段由 Experience（体验定义）+ GameFeatureAction 驱动启动

```
架构总图：

┌─────────────────────────────────────────────────────────────────┐
│                        ALyraGameMode                            │
│  ┌─────────────────┐                                            │
│  │ InitGame()      │──→ HandleMatchAssignment ──→ 确定 ExperienceId │
│  └─────────────────┘                                            │
│          │                                                      │
│          ▼                                                      │
│  ┌───────────────────────────────────────────────────┐          │
│  │        ALyraGameState                              │          │
│  │  ┌─────────────────────────────┐                   │          │
│  │  │ ULyraAbilitySystemComponent │ ← Phase Ability   │          │
│  │  │ (GameState 级 ASC)          │   在此 ASC 上激活  │          │
│  │  └─────────────────────────────┘                   │          │
│  │  ┌─────────────────────────────────────┐           │          │
│  │  │ ULyraExperienceManagerComponent     │           │          │
│  │  │  - 加载 Experience                   │           │          │
│  │  │  - 激活 GameFeatureActions           │           │          │
│  │  │  - 广播 OnExperienceLoaded           │           │          │
│  │  └─────────────────────────────────────┘           │          │
│  │  ┌─────────────────────┐ ┌──────────────────────┐  │          │
│  │  │ TeamCreationComp    │ │ BotCreationComp      │  │          │
│  │  │ PlayerSpawningComp  │ │ FrontendStateComp    │  │          │
│  │  └─────────────────────┘ └──────────────────────┘  │          │
│  └───────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              ULyraGamePhaseSubsystem (WorldSubsystem)           │
│  ┌──────────────────────────────────────────────────┐           │
│  │ StartPhase(PhaseAbilityClass)                    │           │
│  │   → GameState_ASC->GiveAbilityAndActivateOnce() │           │
│  │   → OnBeginPhase() 检测并取消互斥的兄弟阶段       │           │
│  │   → 通知 PhaseStartObservers                     │           │
│  │                                                  │           │
│  │ ActivePhaseMap: Handle → {PhaseTag, Callback}    │           │
│  │ PhaseStartObservers: [{Tag, MatchType, Cb}]      │           │
│  │ PhaseEndObservers: [{Tag, MatchType, Cb}]        │           │
│  └──────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心类继承体系

```
UWorldSubsystem
  └── ULyraGamePhaseSubsystem          // 阶段管理子系统（World 级单例）

ULyraGameplayAbility
  └── ULyraGamePhaseAbility            // 阶段能力基类（Abstract）

AModularGameModeBase
  └── ALyraGameMode                    // 游戏模式

AModularGameStateBase + IAbilitySystemInterface
  └── ALyraGameState                   // 游戏状态（持有 GameState 级 ASC）

AWorldSettings
  └── ALyraWorldSettings               // 世界设置（持有默认 Experience）

UPrimaryDataAsset
  ├── ULyraExperienceDefinition        // 体验定义
  ├── ULyraExperienceActionSet         // 体验动作集
  └── ULyraUserFacingExperienceDefinition  // 用户面向体验定义

UGameStateComponent
  ├── ULyraExperienceManagerComponent  // 体验管理器
  ├── ULyraTeamCreationComponent       // 队伍创建
  ├── ULyraBotCreationComponent        // Bot 创建
  ├── ULyraPlayerSpawningManagerComponent  // 出生点管理
  └── ULyraFrontendStateComponent      // 前端 UI 状态

UEngineSubsystem
  └── ULyraExperienceManager           // PIE 多实例插件仲裁

UBlueprintAsyncActionBase
  └── UAsyncAction_ExperienceReady     // 蓝图异步等待 Experience 就绪
```

---

## 3. GamePhase 核心三件套详解

### 3.1 ULyraGamePhaseAbility — 阶段能力基类

**文件**：`Source/LyraGame/AbilitySystem/Phases/LyraGamePhaseAbility.h/.cpp`

#### 类定义

```cpp
// LyraGamePhaseAbility.h
UCLASS(Abstract, HideCategories = Input)
class ULyraGamePhaseAbility : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    ULyraGamePhaseAbility(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

    const FGameplayTag& GetGamePhaseTag() const { return GamePhaseTag; }

protected:
    virtual void ActivateAbility(...) override;
    virtual void EndAbility(...) override;

    // 定义此能力所代表的游戏阶段
    // 例如：GamePhase.Playing.NormalPlay
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Game Phase")
    FGameplayTag GamePhaseTag;
};
```

#### 构造函数 — 关键的复制策略

```cpp
ULyraGamePhaseAbility::ULyraGamePhaseAbility(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    ReplicationPolicy = EGameplayAbilityReplicationPolicy::ReplicateNo;    // 不复制
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor; // 每 Actor 一个实例
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerInitiated; // 仅服务器发起
    NetSecurityPolicy = EGameplayAbilityNetSecurityPolicy::ServerOnly;      // 仅服务器执行
}
```

**设计要点**：
- **ServerOnly**：游戏阶段管理是纯权威端逻辑，客户端不参与阶段决策
- **InstancedPerActor**：因为挂载在 GameState 的 ASC 上，全局只有一个 GameState，所以本质上是全局单例
- **ReplicateNo**：阶段信息不通过 Ability 复制走（如需客户端感知，靠其他机制如 GameplayTag 的复制）

#### ActivateAbility — 通知子系统阶段开始

```cpp
void ULyraGamePhaseAbility::ActivateAbility(...)
{
    if (ActorInfo->IsNetAuthority())
    {
        UWorld* World = ActorInfo->AbilitySystemComponent->GetWorld();
        ULyraGamePhaseSubsystem* PhaseSubsystem = UWorld::GetSubsystem<ULyraGamePhaseSubsystem>(World);
        PhaseSubsystem->OnBeginPhase(this, Handle);  // ← 关键：通知子系统
    }
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);
}
```

#### EndAbility — 通知子系统阶段结束

```cpp
void ULyraGamePhaseAbility::EndAbility(...)
{
    if (ActorInfo->IsNetAuthority())
    {
        UWorld* World = ActorInfo->AbilitySystemComponent->GetWorld();
        ULyraGamePhaseSubsystem* PhaseSubsystem = UWorld::GetSubsystem<ULyraGamePhaseSubsystem>(World);
        PhaseSubsystem->OnEndPhase(this, Handle);  // ← 关键：通知子系统
    }
    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}
```

#### 编辑器数据校验

```cpp
#if WITH_EDITOR
EDataValidationResult ULyraGamePhaseAbility::IsDataValid(FDataValidationContext& Context) const
{
    EDataValidationResult Result = CombineDataValidationResults(Super::IsDataValid(Context), EDataValidationResult::Valid);
    if (!GamePhaseTag.IsValid())
    {
        Result = EDataValidationResult::Invalid;
        Context.AddError(LOCTEXT("GamePhaseTagNotSet", "GamePhaseTag must be set to a tag representing the current phase."));
    }
    return Result;
}
#endif
```

**设计亮点**：编辑器时强制校验 `GamePhaseTag` 必须设置，防止运行时出错。

---

### 3.2 ULyraGamePhaseSubsystem — 阶段管理子系统

**文件**：`Source/LyraGame/AbilitySystem/Phases/LyraGamePhaseSubsystem.h/.cpp`

这是整个 GamePhase 系统的**中枢控制器**，作为 `UWorldSubsystem` 在 World 级别存在。

#### 类头部定义

```cpp
// 委托声明 — 4 种回调签名
DECLARE_DYNAMIC_DELEGATE_OneParam(FLyraGamePhaseDynamicDelegate, const ULyraGamePhaseAbility*, Phase);
DECLARE_DELEGATE_OneParam(FLyraGamePhaseDelegate, const ULyraGamePhaseAbility* Phase);
DECLARE_DYNAMIC_DELEGATE_OneParam(FLyraGamePhaseTagDynamicDelegate, const FGameplayTag&, PhaseTag);
DECLARE_DELEGATE_OneParam(FLyraGamePhaseTagDelegate, const FGameplayTag& PhaseTag);

// Tag 匹配方式枚举
UENUM(BlueprintType)
enum class EPhaseTagMatchType : uint8
{
    ExactMatch,    // 精确匹配：注册 "A.B" 只匹配 "A.B"
    PartialMatch   // 部分匹配：注册 "A.B" 同时匹配 "A.B" 和 "A.B.C"
};
```

#### 核心数据结构

```cpp
// 活跃阶段条目
struct FLyraGamePhaseEntry
{
    FGameplayTag PhaseTag;                    // 阶段 Tag
    FLyraGamePhaseDelegate PhaseEndedCallback; // 阶段结束回调
};

TMap<FGameplayAbilitySpecHandle, FLyraGamePhaseEntry> ActivePhaseMap;  // Handle → 阶段映射

// 阶段观察者
struct FPhaseObserver
{
    bool IsMatch(const FGameplayTag& ComparePhaseTag) const;  // 匹配检测
    FGameplayTag PhaseTag;
    EPhaseTagMatchType MatchType = EPhaseTagMatchType::ExactMatch;
    FLyraGamePhaseTagDelegate PhaseCallback;
};

TArray<FPhaseObserver> PhaseStartObservers;  // 阶段开始观察者列表
TArray<FPhaseObserver> PhaseEndObservers;    // 阶段结束观察者列表
```

#### StartPhase — 启动新阶段

```cpp
void ULyraGamePhaseSubsystem::StartPhase(TSubclassOf<ULyraGamePhaseAbility> PhaseAbility, 
                                          FLyraGamePhaseDelegate PhaseEndedCallback)
{
    UWorld* World = GetWorld();
    // ① 获取 GameState 上的 ASC
    ULyraAbilitySystemComponent* GameState_ASC = 
        World->GetGameState()->FindComponentByClass<ULyraAbilitySystemComponent>();
    
    if (ensure(GameState_ASC))
    {
        // ② 创建 AbilitySpec 并立即激活
        FGameplayAbilitySpec PhaseSpec(PhaseAbility, 1, 0, this);
        FGameplayAbilitySpecHandle SpecHandle = GameState_ASC->GiveAbilityAndActivateOnce(PhaseSpec);
        
        FGameplayAbilitySpec* FoundSpec = GameState_ASC->FindAbilitySpecFromHandle(SpecHandle);
        
        if (FoundSpec && FoundSpec->IsActive())
        {
            // ③ 注册到 ActivePhaseMap
            FLyraGamePhaseEntry& Entry = ActivePhaseMap.FindOrAdd(SpecHandle);
            Entry.PhaseEndedCallback = PhaseEndedCallback;
        }
        else
        {
            // ④ 激活失败，立即回调
            PhaseEndedCallback.ExecuteIfBound(nullptr);
        }
    }
}
```

**关键设计**：
- 阶段 Ability 运行在 **GameState 的 ASC** 上，不是任何 Player 的 ASC
- 使用 `GiveAbilityAndActivateOnce` 一步完成赋予+激活
- 通过 `PhaseEndedCallback` 支持链式阶段（一个阶段结束后启动下一个）

#### OnBeginPhase — 阶段开始时的互斥处理（核心算法）

```cpp
void ULyraGamePhaseSubsystem::OnBeginPhase(const ULyraGamePhaseAbility* PhaseAbility, 
                                            const FGameplayAbilitySpecHandle PhaseAbilityHandle)
{
    const FGameplayTag IncomingPhaseTag = PhaseAbility->GetGamePhaseTag();

    ULyraAbilitySystemComponent* GameState_ASC = 
        World->GetGameState()->FindComponentByClass<ULyraAbilitySystemComponent>();
    
    if (ensure(GameState_ASC))
    {
        // ① 收集当前所有活跃阶段
        TArray<FGameplayAbilitySpec*> ActivePhases;
        for (const auto& KVP : ActivePhaseMap)
        {
            if (FGameplayAbilitySpec* Spec = GameState_ASC->FindAbilitySpecFromHandle(KVP.Key))
            {
                ActivePhases.Add(Spec);
            }
        }

        // ② 检查每个活跃阶段，结束非祖先关系的阶段
        for (const FGameplayAbilitySpec* ActivePhase : ActivePhases)
        {
            const ULyraGamePhaseAbility* ActivePhaseAbility = 
                CastChecked<ULyraGamePhaseAbility>(ActivePhase->Ability);
            const FGameplayTag ActivePhaseTag = ActivePhaseAbility->GetGamePhaseTag();
            
            // 关键判断：IncomingPhaseTag.MatchesTag(ActivePhaseTag)
            // 如果新阶段 Tag 是旧阶段 Tag 的子 Tag（即旧阶段是新阶段的祖先），
            // 则旧阶段保留；否则结束旧阶段
            if (!IncomingPhaseTag.MatchesTag(ActivePhaseTag))
            {
                FGameplayAbilitySpecHandle HandleToEnd = ActivePhase->Handle;
                GameState_ASC->CancelAbilitiesByFunc(
                    [HandleToEnd](const ULyraGameplayAbility* LyraAbility, FGameplayAbilitySpecHandle Handle) {
                        return Handle == HandleToEnd;
                    }, true);
            }
        }

        // ③ 将新阶段加入 ActivePhaseMap
        FLyraGamePhaseEntry& Entry = ActivePhaseMap.FindOrAdd(PhaseAbilityHandle);
        Entry.PhaseTag = IncomingPhaseTag;

        // ④ 通知所有阶段开始观察者
        for (const FPhaseObserver& Observer : PhaseStartObservers)
        {
            if (Observer.IsMatch(IncomingPhaseTag))
            {
                Observer.PhaseCallback.ExecuteIfBound(IncomingPhaseTag);
            }
        }
    }
}
```

#### OnEndPhase — 阶段结束处理

```cpp
void ULyraGamePhaseSubsystem::OnEndPhase(const ULyraGamePhaseAbility* PhaseAbility, 
                                          const FGameplayAbilitySpecHandle PhaseAbilityHandle)
{
    const FGameplayTag EndedPhaseTag = PhaseAbility->GetGamePhaseTag();

    // ① 执行阶段结束回调
    const FLyraGamePhaseEntry& Entry = ActivePhaseMap.FindChecked(PhaseAbilityHandle);
    Entry.PhaseEndedCallback.ExecuteIfBound(PhaseAbility);

    // ② 从活跃阶段映射中移除
    ActivePhaseMap.Remove(PhaseAbilityHandle);

    // ③ 通知所有阶段结束观察者
    for (const FPhaseObserver& Observer : PhaseEndObservers)
    {
        if (Observer.IsMatch(EndedPhaseTag))
        {
            Observer.PhaseCallback.ExecuteIfBound(EndedPhaseTag);
        }
    }
}
```

#### WhenPhaseStartsOrIsActive — 注册阶段开始/已活跃观察

```cpp
void ULyraGamePhaseSubsystem::WhenPhaseStartsOrIsActive(FGameplayTag PhaseTag, 
    EPhaseTagMatchType MatchType, const FLyraGamePhaseTagDelegate& WhenPhaseActive)
{
    // 注册观察者
    FPhaseObserver Observer;
    Observer.PhaseTag = PhaseTag;
    Observer.MatchType = MatchType;
    Observer.PhaseCallback = WhenPhaseActive;
    PhaseStartObservers.Add(Observer);

    // 如果该阶段已经活跃，立即触发一次
    if (IsPhaseActive(PhaseTag))
    {
        WhenPhaseActive.ExecuteIfBound(PhaseTag);
    }
}
```

**设计亮点**：遵循"CallOrRegister"模式——如果阶段已经在运行，立即回调；否则注册等待。

#### IsPhaseActive — 查询阶段是否活跃

```cpp
bool ULyraGamePhaseSubsystem::IsPhaseActive(const FGameplayTag& PhaseTag) const
{
    for (const auto& KVP : ActivePhaseMap)
    {
        const FLyraGamePhaseEntry& PhaseEntry = KVP.Value;
        if (PhaseEntry.PhaseTag.MatchesTag(PhaseTag))  // 使用 Tag 层级匹配
        {
            return true;
        }
    }
    return false;
}
```

**注意**：使用 `MatchesTag` 而非 `==`，这意味着查询 `Game.Playing` 时，即使当前活跃的是 `Game.Playing.SuddenDeath`，也会返回 `true`。

#### FPhaseObserver::IsMatch — 观察者匹配逻辑

```cpp
bool ULyraGamePhaseSubsystem::FPhaseObserver::IsMatch(const FGameplayTag& ComparePhaseTag) const
{
    switch(MatchType)
    {
    case EPhaseTagMatchType::ExactMatch:
        return ComparePhaseTag == PhaseTag;        // 完全相等
    case EPhaseTagMatchType::PartialMatch:
        return ComparePhaseTag.MatchesTag(PhaseTag); // Tag 层级匹配
    }
    return false;
}
```

#### 蓝图 API 封装

子系统提供了完整的蓝图 API，通过 `K2_` 前缀函数包装 C++ 委托到动态委托：

```cpp
UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Game Phase",
    meta = (DisplayName="Start Phase"))
void K2_StartPhase(TSubclassOf<ULyraGamePhaseAbility> Phase, 
                   const FLyraGamePhaseDynamicDelegate& PhaseEnded);

UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Game Phase",
    meta = (DisplayName = "When Phase Starts or Is Active"))
void K2_WhenPhaseStartsOrIsActive(FGameplayTag PhaseTag, EPhaseTagMatchType MatchType, 
                                   FLyraGamePhaseTagDynamicDelegate WhenPhaseActive);

UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category = "Game Phase",
    meta = (DisplayName = "When Phase Ends"))
void K2_WhenPhaseEnds(FGameplayTag PhaseTag, EPhaseTagMatchType MatchType, 
                       FLyraGamePhaseTagDynamicDelegate WhenPhaseEnd);

UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, BlueprintPure = false)
bool IsPhaseActive(const FGameplayTag& PhaseTag) const;
```

所有蓝图 API 标记 `BlueprintAuthorityOnly`——仅服务器权威端可调用。

#### 子系统生命周期

```cpp
bool ULyraGamePhaseSubsystem::ShouldCreateSubsystem(UObject* Outer) const
{
    // 始终创建（有注释掉的 GameMode 检查）
    return Super::ShouldCreateSubsystem(Outer) ? true : false;
}

bool ULyraGamePhaseSubsystem::DoesSupportWorldType(const EWorldType::Type WorldType) const
{
    return WorldType == EWorldType::Game || WorldType == EWorldType::PIE;
    // 仅在游戏世界和 PIE 世界中创建，编辑器预览等不创建
}
```

---

### 3.3 GamePhaseTag 嵌套层级机制

这是整个系统的**最大设计亮点**。源码注释中给出了清晰说明：

```
Tag 层级示例：
  GamePhase.WaitingToStart
  GamePhase.Playing
  GamePhase.Playing.NormalPlay
  GamePhase.Playing.SuddenDeath
  GamePhase.Playing.ActualSuddenDeath
  GamePhase.ShowingScore
  GamePhase.GameOver

规则：
1. 启动 GamePhase.Playing 时 → 结束 GamePhase.WaitingToStart（兄弟互斥）
2. 启动 GamePhase.Playing.NormalPlay 时 → GamePhase.Playing 保持（父子共存）
3. 启动 GamePhase.Playing.SuddenDeath 时 → 结束 GamePhase.Playing.NormalPlay（兄弟互斥）
   → GamePhase.Playing 保持（父子共存）
4. 启动 GamePhase.GameOver 时 → 结束所有 GamePhase.Playing.* 和 GamePhase.Playing
```

#### 互斥判断核心公式

```cpp
// OnBeginPhase 中的关键判断：
if (!IncomingPhaseTag.MatchesTag(ActivePhaseTag))
{
    // 结束 ActivePhase
}
```

`IncomingPhaseTag.MatchesTag(ActivePhaseTag)` 的语义是：**新 Tag 是否 "匹配" 旧 Tag**。

在 UE 的 GameplayTag 系统中，`A.B.C.MatchesTag(A.B)` 返回 `true`（子 Tag 匹配父 Tag）。

所以：
| 新 Tag (Incoming) | 旧 Tag (Active) | MatchesTag 结果 | 旧阶段是否保留 |
|---|---|---|---|
| `Game.Playing.SuddenDeath` | `Game.Playing` | `true`（子匹配父） | ✅ 保留 |
| `Game.Playing.SuddenDeath` | `Game.Playing.NormalPlay` | `false`（兄弟不匹配） | ❌ 结束 |
| `Game.GameOver` | `Game.Playing` | `false`（兄弟不匹配） | ❌ 结束 |
| `Game.Playing` | `Game.Playing` | `true`（自匹配） | ✅ 保留 |

**总结规则**：新阶段启动时，只保留作为其"祖先"的阶段，所有非祖先阶段（兄弟或不相关的）都会被取消。

---

## 4. Experience 系统 — GamePhase 的触发源

Experience 系统是 GamePhase 的上游驱动力。GamePhase 的 Ability 通常由 Experience 中配置的 GameFeatureAction 来启动。

### 4.1 ULyraExperienceDefinition — 体验定义

**文件**：`Source/LyraGame/GameModes/LyraExperienceDefinition.h/.cpp`

```cpp
UCLASS(BlueprintType, Const)
class ULyraExperienceDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 需要启用的 GameFeature 插件列表
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TArray<FString> GameFeaturesToEnable;

    // 默认 Pawn 数据
    UPROPERTY(EditDefaultsOnly, Category=Gameplay)
    TObjectPtr<const ULyraPawnData> DefaultPawnData;

    // 体验加载/激活时执行的 Actions
    UPROPERTY(EditDefaultsOnly, Instanced, Category="Actions")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;

    // 额外的动作集（组合模式）
    UPROPERTY(EditDefaultsOnly, Category=Gameplay)
    TArray<TObjectPtr<ULyraExperienceActionSet>> ActionSets;
};
```

**设计模式**：
- **组合优于继承**：通过 `Actions` + `ActionSets` 组合功能，而非创建子类
- **不允许 BP 二次继承**：编辑器验证中明确禁止 `Blueprint subclasses of Blueprint experiences`
- 每个 Action 遵循 `UGameFeatureAction` 的 `Registering → Loading → Activating → Deactivating → Unregistering` 生命周期

### 4.2 ULyraExperienceManagerComponent — 体验加载状态机

**文件**：`Source/LyraGame/GameModes/LyraExperienceManagerComponent.h/.cpp`

这是挂载在 `ALyraGameState` 上的核心组件，管理 Experience 的完整生命周期。

#### 加载状态枚举

```cpp
enum class ELyraExperienceLoadState
{
    Unloaded,                // 未加载
    Loading,                 // 加载资源中
    LoadingGameFeatures,     // 加载 GameFeature 插件中
    LoadingChaosTestingDelay,// 混沌测试延迟中
    ExecutingActions,        // 执行 Actions 中
    Loaded,                  // 已加载完成
    Deactivating             // 正在停用
};
```

#### 加载流程详解

```
状态机流转：

  Unloaded
     │ SetCurrentExperience()
     ▼
  Loading ──────────────────────────────► 异步加载 Bundle 资源
     │ OnExperienceLoadComplete()
     ▼
  LoadingGameFeatures ──────────────────► 逐个 LoadAndActivate GameFeature 插件
     │ 全部插件加载完成
     ▼
  LoadingChaosTestingDelay ─────────────► (可选) 随机延迟用于压力测试
     │
     ▼
  ExecutingActions ─────────────────────► 依次调用每个 Action 的:
     │                                    OnGameFeatureRegistering()
     │                                    OnGameFeatureLoading()
     │                                    OnGameFeatureActivating(Context)
     ▼
  Loaded ──────────────────────────────► 广播三级回调:
     │                                    1. OnExperienceLoaded_HighPriority
     │                                    2. OnExperienceLoaded
     │                                    3. OnExperienceLoaded_LowPriority
     │
     │ EndPlay()
     ▼
  Deactivating ────────────────────────► 反向停用所有 Actions:
     │                                    OnGameFeatureDeactivating()
     │                                    OnGameFeatureUnregistering()
     ▼
  Unloaded
```

#### 三级优先级回调

```cpp
// 高优先级（子系统初始化等）
FOnLyraExperienceLoaded OnExperienceLoaded_HighPriority;

// 标准优先级（大多数游戏逻辑）
FOnLyraExperienceLoaded OnExperienceLoaded;

// 低优先级（Bot 创建等后期逻辑）
FOnLyraExperienceLoaded OnExperienceLoaded_LowPriority;
```

**使用实例**：
- `ULyraTeamCreationComponent` → 注册 `HighPriority`（先创建队伍）
- `ALyraGameMode` → 注册标准优先级（然后生成玩家）
- `ULyraBotCreationComponent` → 注册 `LowPriority`（最后创建 Bot）
- `ULyraFrontendStateComponent` → 注册 `HighPriority`（UI 流程最先启动）

#### CallOrRegister 模式

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

这是 Lyra 中极为重要的设计模式——**无论何时注册，都能保证回调被执行**。避免了注册时序问题。

#### 网络复制

```cpp
UPROPERTY(ReplicatedUsing=OnRep_CurrentExperience)
TObjectPtr<const ULyraExperienceDefinition> CurrentExperience;

void ULyraExperienceManagerComponent::OnRep_CurrentExperience()
{
    StartExperienceLoad();  // 客户端收到复制后，开始本地加载流程
}
```

#### 混沌测试支持

```cpp
namespace LyraConsoleVariables
{
    static float ExperienceLoadRandomDelayMin = 0.0f;    // lyra.chaos.ExperienceDelayLoad.MinSecs
    static float ExperienceLoadRandomDelayRange = 0.0f;  // lyra.chaos.ExperienceDelayLoad.RandomSecs
    
    float GetExperienceLoadDelayDuration()
    {
        return FMath::Max(0.0f, ExperienceLoadRandomDelayMin + FMath::FRand() * ExperienceLoadRandomDelayRange);
    }
}
```

通过控制台变量注入随机延迟，用于测试在不同加载时序下的系统稳健性。

### 4.3 ULyraExperienceActionSet — 可组合的动作集

**文件**：`Source/LyraGame/GameModes/LyraExperienceActionSet.h/.cpp`

```cpp
UCLASS(BlueprintType, NotBlueprintable)
class ULyraExperienceActionSet : public UPrimaryDataAsset
{
    // 可复用的 Action 列表
    UPROPERTY(EditAnywhere, Instanced, Category="Actions to Perform")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;

    // 此 ActionSet 需要的 GameFeature 插件
    UPROPERTY(EditAnywhere, Category="Feature Dependencies")
    TArray<FString> GameFeaturesToEnable;
};
```

**设计目的**：将常用的 Action 组合（如"射击核心 Action 集"）封装为可复用资产，多个 Experience 可以共享同一个 ActionSet。

### 4.4 ULyraExperienceManager — PIE 多实例仲裁

**文件**：`Source/LyraGame/GameModes/LyraExperienceManager.h/.cpp`

```cpp
UCLASS(MinimalAPI)
class ULyraExperienceManager : public UEngineSubsystem
{
    // 引用计数：防止 PIE 多窗口下同一个 GameFeature 被重复停用
    TMap<FString, int32> GameFeaturePluginRequestCountMap;

public:
    static void NotifyOfPluginActivation(const FString PluginURL);   // 激活时 +1
    static bool RequestToDeactivatePlugin(const FString PluginURL);  // 停用时 -1，为0时真正停用
};
```

**解决的问题**：在 PIE 多客户端模式下，多个 World 可能激活同一个 GameFeature 插件。使用引用计数确保"最后一个使用者离开时才真正停用"。

---

## 5. GameMode/GameState — 基础设施层

### 5.1 ALyraGameMode — 游戏模式

**文件**：`Source/LyraGame/GameModes/LyraGameMode.h/.cpp`

#### Experience 确定优先级

```cpp
void ALyraGameMode::HandleMatchAssignmentIfNotExpectingOne()
{
    // 优先级从高到低：
    // 1. URL Options (?Experience=XXX)
    // 2. Developer Settings (PIE only)
    // 3. Command Line (-Experience=XXX)
    // 4. World Settings (DefaultGameplayExperience)
    // 5. Dedicated Server 登录流程
    // 6. 默认体验 (B_LyraDefaultExperience)
    
    // ... 确定后 →
    OnMatchAssignmentGiven(ExperienceId, ExperienceIdSource);
}
```

#### Experience → GameMode 初始化链

```cpp
void ALyraGameMode::InitGameState()
{
    Super::InitGameState();
    
    ULyraExperienceManagerComponent* ExperienceComponent = 
        GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
    check(ExperienceComponent);
    
    // 注册 Experience 加载完成回调
    ExperienceComponent->CallOrRegister_OnExperienceLoaded(
        FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}

void ALyraGameMode::OnExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience)
{
    // Experience 加载完成后，重新尝试生成之前等待中的玩家
    for (FConstPlayerControllerIterator Iterator = GetWorld()->GetPlayerControllerIterator(); 
         Iterator; ++Iterator)
    {
        APlayerController* PC = Cast<APlayerController>(*Iterator);
        if ((PC != nullptr) && (PC->GetPawn() == nullptr))
        {
            if (PlayerCanRestart(PC))
            {
                RestartPlayer(PC);
            }
        }
    }
}

// Experience 加载前，阻止玩家生成
void ALyraGameMode::HandleStartingNewPlayer_Implementation(APlayerController* NewPlayer)
{
    if (IsExperienceLoaded())
    {
        Super::HandleStartingNewPlayer_Implementation(NewPlayer);
    }
    // 未加载时不做任何事——OnExperienceLoaded 回调中会补上
}
```

#### 委托管理器

```cpp
// 通用的玩家初始化委托（供 TeamCreation 等组件监听）
DECLARE_MULTICAST_DELEGATE_TwoParams(FOnLyraGameModePlayerInitialized, 
    AGameModeBase*, AController*);
FOnLyraGameModePlayerInitialized OnGameModePlayerInitialized;

void ALyraGameMode::GenericPlayerInitialization(AController* NewPlayer)
{
    Super::GenericPlayerInitialization(NewPlayer);
    OnGameModePlayerInitialized.Broadcast(this, NewPlayer);
}
```

### 5.2 ALyraGameState — 游戏状态

**文件**：`Source/LyraGame/GameModes/LyraGameState.h/.cpp`

```cpp
UCLASS(MinimalAPI, Config = Game)
class ALyraGameState : public AModularGameStateBase, public IAbilitySystemInterface
{
    // GameState 级 ASC —— GamePhase Ability 在此运行
    UPROPERTY(VisibleAnywhere)
    TObjectPtr<ULyraAbilitySystemComponent> AbilitySystemComponent;

    // Experience 管理器组件
    UPROPERTY()
    TObjectPtr<ULyraExperienceManagerComponent> ExperienceManagerComponent;

    // 复制的服务器 FPS
    UPROPERTY(Replicated)
    float ServerFPS;
};
```

#### 构造函数

```cpp
ALyraGameState::ALyraGameState(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    PrimaryActorTick.bCanEverTick = true;
    PrimaryActorTick.bStartWithTickEnabled = true;

    // 创建 GameState 级 ASC
    AbilitySystemComponent = ObjectInitializer.CreateDefaultSubobject<ULyraAbilitySystemComponent>(
        this, TEXT("AbilitySystemComponent"));
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);

    // 创建 Experience 管理器
    ExperienceManagerComponent = CreateDefaultSubobject<ULyraExperienceManagerComponent>(
        TEXT("ExperienceManagerComponent"));
}
```

**关键点**：
- GameState 持有 **游戏全局级 ASC**，与玩家级 ASC 分开
- ASC 使用 `Mixed` 复制模式（Owner 用 Full，其他用 Minimal）
- GamePhase Ability 就是在这个 ASC 上激活/取消的

### 5.3 ALyraWorldSettings — 世界设置

**文件**：`Source/LyraGame/GameModes/LyraWorldSettings.h/.cpp`

```cpp
UCLASS(MinimalAPI)
class ALyraWorldSettings : public AWorldSettings
{
protected:
    // 此地图的默认 Experience
    UPROPERTY(EditDefaultsOnly, Category=GameMode)
    TSoftClassPtr<ULyraExperienceDefinition> DefaultGameplayExperience;

public:
    FPrimaryAssetId GetDefaultGameplayExperience() const;

#if WITH_EDITORONLY_DATA
    // 前端关卡强制 Standalone 模式
    UPROPERTY(EditDefaultsOnly, Category=PIE)
    bool ForceStandaloneNetMode = false;
#endif
};
```

每张地图通过 WorldSettings 指定默认的 Experience，GameMode 在确定 Experience 时会查询这个值。

---

## 6. GameStateComponent 组件体系

Lyra 大量使用 `UGameStateComponent`（挂载在 GameState 上的模块化组件）来实现各种游戏功能。这些组件都遵循"监听 Experience 加载完成 → 执行逻辑"的模式。

### 6.1 ULyraTeamCreationComponent — 队伍创建

**文件**：`Source/LyraGame/Teams/LyraTeamCreationComponent.h/.cpp`

```cpp
UCLASS(Blueprintable)
class ULyraTeamCreationComponent : public UGameStateComponent
{
protected:
    // 配置：队伍 ID → 显示资产 映射
    UPROPERTY(EditDefaultsOnly, Category = Teams)
    TMap<uint8, TObjectPtr<ULyraTeamDisplayAsset>> TeamsToCreate;
};
```

#### 初始化时序

```cpp
void ULyraTeamCreationComponent::BeginPlay()
{
    Super::BeginPlay();
    ULyraExperienceManagerComponent* ExperienceComponent = ...;
    // 注册 HighPriority —— 队伍必须在玩家生成之前创建
    ExperienceComponent->CallOrRegister_OnExperienceLoaded_HighPriority(
        FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}

void ULyraTeamCreationComponent::OnExperienceLoaded(const ULyraExperienceDefinition* Experience)
{
    if (HasAuthority())
    {
        ServerCreateTeams();        // 创建所有队伍
        ServerAssignPlayersToTeams(); // 分配已有玩家 + 监听后续加入的玩家
    }
}
```

#### 负载均衡的队伍分配

```cpp
void ULyraTeamCreationComponent::ServerChooseTeamForPlayer(ALyraPlayerState* PS)
{
    if (PS->IsOnlyASpectator())
    {
        PS->SetGenericTeamId(FGenericTeamId::NoTeam);  // 观战者无队伍
    }
    else
    {
        const FGenericTeamId TeamID = IntegerToGenericTeamId(GetLeastPopulatedTeamID());
        PS->SetGenericTeamId(TeamID);  // 分配到人数最少的队伍
    }
}

int32 ULyraTeamCreationComponent::GetLeastPopulatedTeamID() const
{
    // 统计每个队伍的活跃玩家数，返回人数最少（相同时选 ID 最小）的队伍
    // 排除未分配(INDEX_NONE)和已断线(IsInactive)的玩家
}
```

### 6.2 ULyraBotCreationComponent — Bot 创建

**文件**：`Source/LyraGame/GameModes/LyraBotCreationComponent.h/.cpp`

```cpp
UCLASS(Blueprintable, Abstract)
class ULyraBotCreationComponent : public UGameStateComponent
{
protected:
    UPROPERTY(EditDefaultsOnly, Category=Gameplay)
    int32 NumBotsToCreate = 5;

    UPROPERTY(EditDefaultsOnly, Category=Gameplay)
    TSubclassOf<AAIController> BotControllerClass;

    UPROPERTY(EditDefaultsOnly, Category=Gameplay)
    TArray<FString> RandomBotNames;
};
```

#### 初始化时序

```cpp
void ULyraBotCreationComponent::BeginPlay()
{
    Super::BeginPlay();
    ULyraExperienceManagerComponent* ExperienceComponent = ...;
    // 注册 LowPriority —— Bot 最后创建（在队伍、出生点等都准备好之后）
    ExperienceComponent->CallOrRegister_OnExperienceLoaded_LowPriority(
        FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}
```

#### Bot 数量覆盖链

```cpp
void ULyraBotCreationComponent::ServerCreateBots_Implementation()
{
    int32 EffectiveBotCount = NumBotsToCreate;  // 默认值

    // 1. 开发者设置覆盖（仅编辑器）
    if (GIsEditor)
    {
        const ULyraDeveloperSettings* DeveloperSettings = GetDefault<ULyraDeveloperSettings>();
        if (DeveloperSettings->bOverrideBotCount)
            EffectiveBotCount = DeveloperSettings->OverrideNumPlayerBotsToSpawn;
    }

    // 2. URL 选项覆盖（?NumBots=XX）
    if (AGameModeBase* GameModeBase = GetGameMode<AGameModeBase>())
        EffectiveBotCount = UGameplayStatics::GetIntOption(
            GameModeBase->OptionsString, TEXT("NumBots"), EffectiveBotCount);

    // 3. 创建指定数量的 Bot
    for (int32 Count = 0; Count < EffectiveBotCount; ++Count)
        SpawnOneBot();
}
```

### 6.3 ULyraPlayerSpawningManagerComponent — 出生点管理

**文件**：`Source/LyraGame/Player/LyraPlayerSpawningManagerComponent.h/.cpp`

此组件被 `ALyraGameMode` 代理调用，处理玩家出生点选择：

```cpp
// ALyraGameMode 中的代理调用链
AActor* ALyraGameMode::ChoosePlayerStart_Implementation(AController* Player)
{
    if (ULyraPlayerSpawningManagerComponent* Comp = 
        GameState->FindComponentByClass<ULyraPlayerSpawningManagerComponent>())
    {
        return Comp->ChoosePlayerStart(Player);
    }
    return Super::ChoosePlayerStart_Implementation(Player);
}
```

#### 出生点选择策略

```cpp
AActor* ULyraPlayerSpawningManagerComponent::ChoosePlayerStart(AController* Player)
{
    // 1. PIE 模式下优先使用 "Play From Here" 点
    // 2. 观战者使用随机点（不占用）
    // 3. 调用 OnChoosePlayerStart 虚函数（子类可重写）
    // 4. 回退到 GetFirstRandomUnoccupiedPlayerStart（空闲 > 部分占用 > 随机）
    // 5. 成功选择后调用 LyraStart->TryClaim(Controller) 占用出生点
}
```

### 6.4 ULyraFrontendStateComponent — 前端 UI 状态流

**文件**：`Source/LyraGame/UI/Frontend/LyraFrontendStateComponent.h/.cpp`

这是前端主菜单的状态机组件，通过 `FControlFlow` 实现流式步骤管理：

```cpp
void ULyraFrontendStateComponent::OnExperienceLoaded(const ULyraExperienceDefinition* Experience)
{
    FControlFlow& Flow = FControlFlowStatics::Create(this, TEXT("FrontendFlow"))
        .QueueStep(TEXT("Wait For User Initialization"), this, 
                   &ThisClass::FlowStep_WaitForUserInitialization)
        .QueueStep(TEXT("Try Show Press Start Screen"), this, 
                   &ThisClass::FlowStep_TryShowPressStartScreen)
        .QueueStep(TEXT("Try Join Requested Session"), this, 
                   &ThisClass::FlowStep_TryJoinRequestedSession)
        .QueueStep(TEXT("Try Show Main Screen"), this, 
                   &ThisClass::FlowStep_TryShowMainScreen);

    Flow.ExecuteFlow();
    FrontEndFlow = Flow.AsShared();
}
```

#### 前端 Flow 四步流程

```
Step 1: WaitForUserInitialization
  - 硬断线时重置用户状态
  - 始终重置会话状态
  
Step 2: TryShowPressStartScreen
  - 检查是否已登录 → 跳过
  - 检查平台是否需要 Press Start → 自动登录
  - 显示 Press Start 界面，等待用户操作
  
Step 3: TryJoinRequestedSession
  - 检查是否有待加入的会话
  - 有 → 尝试加入，成功则 CancelFlow（直接进游戏）
  - 无 → 继续到主菜单
  
Step 4: TryShowMainScreen
  - 推送主菜单 Widget 到 UI.Layer.Menu 层
  - 关闭加载画面
```

---

## 7. 异步等待机制 — AsyncAction_ExperienceReady

**文件**：`Source/LyraGame/GameModes/AsyncAction_ExperienceReady.h/.cpp`

提供蓝图友好的异步等待节点：

```cpp
UCLASS()
class UAsyncAction_ExperienceReady : public UBlueprintAsyncActionBase
{
    UFUNCTION(BlueprintCallable, meta=(WorldContext = "WorldContextObject", BlueprintInternalUseOnly="true"))
    static UAsyncAction_ExperienceReady* WaitForExperienceReady(UObject* WorldContextObject);

    UPROPERTY(BlueprintAssignable)
    FExperienceReadyAsyncDelegate OnReady;
};
```

#### 四步内部流程

```cpp
// Step1: 等待 GameState 设置完成
void Step1_HandleGameStateSet(AGameStateBase* GameState);

// Step2: 监听 Experience 加载
void Step2_ListenToExperienceLoading(AGameStateBase* GameState)
{
    ULyraExperienceManagerComponent* ExperienceComponent = 
        GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
    
    if (ExperienceComponent->IsExperienceLoaded())
    {
        // 已加载但延迟一帧广播，避免依赖时序
        World->GetTimerManager().SetTimerForNextTick(
            FTimerDelegate::CreateUObject(this, &ThisClass::Step4_BroadcastReady));
    }
    else
    {
        ExperienceComponent->CallOrRegister_OnExperienceLoaded(
            FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::Step3_HandleExperienceLoaded));
    }
}

// Step3: Experience 加载完成
void Step3_HandleExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience)
{
    Step4_BroadcastReady();
}

// Step4: 广播就绪并销毁
void Step4_BroadcastReady()
{
    OnReady.Broadcast();
    SetReadyToDestroy();
}
```

**设计要点**：即使 Experience 已加载，也延迟一帧通知，保证使用方不会写出依赖同步时序的脆弱代码。

---

## 8. 用户面向的体验定义

**文件**：`Source/LyraGame/GameModes/LyraUserFacingExperienceDefinition.h/.cpp`

```cpp
UCLASS(BlueprintType)
class ULyraUserFacingExperienceDefinition : public UPrimaryDataAsset
{
    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
    FPrimaryAssetId MapID;           // 关联地图

    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
    FPrimaryAssetId ExperienceID;    // 关联 Experience

    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
    TMap<FString, FString> ExtraArgs; // 额外参数

    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
    FText TileTitle;                 // UI 标题
    FText TileSubTitle;              // 副标题
    FText TileDescription;           // 描述
    TObjectPtr<UTexture2D> TileIcon; // 图标

    bool bIsDefaultExperience = false; // 是否为默认体验
    bool bShowInFrontEnd = true;       // 是否在前端显示
    bool bRecordReplay = false;        // 是否录制回放
    int32 MaxPlayerCount = 16;         // 最大玩家数

    // 创建会话请求
    UCommonSession_HostSessionRequest* CreateHostingRequest(const UObject* WorldContextObject) const;
};
```

这是面向玩家的 Experience 描述，用于前端 UI 展示和创建在线会话。它将 `MapID` + `ExperienceID` 包装为可配置、可展示的数据资产。

---

## 9. 完整生命周期流程图

```
服务器启动地图
      │
      ▼
ALyraGameMode::InitGame()
      │ 下一帧
      ▼
HandleMatchAssignmentIfNotExpectingOne()
      │ 按优先级确定 ExperienceId
      ▼
OnMatchAssignmentGiven()
      │
      ▼
ULyraExperienceManagerComponent::SetCurrentExperience()
      │ 设置 CurrentExperience (Replicated)
      │ 开始 StartExperienceLoad()
      ▼
┌─────────────────────────────────┐
│  异步加载 Experience 资源 Bundle │
│  (PawnData, Abilities, etc.)    │
└──────────────┬──────────────────┘
               │ OnExperienceLoadComplete()
               ▼
┌─────────────────────────────────┐
│  Load & Activate GameFeature    │
│  Plugins (ShooterCore, etc.)    │
└──────────────┬──────────────────┘
               │ 全部插件就绪
               ▼
┌─────────────────────────────────┐
│  执行所有 GameFeatureActions     │
│  - AddAbilities                 │
│  - AddInputBinding              │
│  - AddWidget                    │
│  - Phase 相关的自定义 Actions    │
└──────────────┬──────────────────┘
               │ LoadState = Loaded
               ▼
    广播 OnExperienceLoaded
      │
      ├──[HighPriority]──→ ULyraTeamCreationComponent::OnExperienceLoaded()
      │                        → ServerCreateTeams()
      │                        → ServerAssignPlayersToTeams()
      │
      ├──[HighPriority]──→ ULyraFrontendStateComponent::OnExperienceLoaded()
      │                        → 启动前端 ControlFlow
      │
      ├──[Normal]────────→ ALyraGameMode::OnExperienceLoaded()
      │                        → RestartPlayer() 为等待的玩家生成 Pawn
      │
      ├──[LowPriority]──→ ULyraBotCreationComponent::OnExperienceLoaded()
      │                        → ServerCreateBots()
      │
      └──[其他订阅者]───→ 自定义逻辑...
               │
               ▼
    GameFeatureAction 触发 Phase Ability
               │
               ▼
ULyraGamePhaseSubsystem::StartPhase(PhaseAbilityClass)
      │
      ▼
GameState_ASC->GiveAbilityAndActivateOnce(PhaseSpec)
      │
      ▼
ULyraGamePhaseAbility::ActivateAbility()
      │
      ▼
ULyraGamePhaseSubsystem::OnBeginPhase()
      │ 检查并取消互斥的兄弟阶段
      │ 注册到 ActivePhaseMap
      │ 通知 PhaseStartObservers
      ▼
    游戏进行中...
               │
               ▼  (某个条件满足，切换到下一阶段)
ULyraGamePhaseSubsystem::StartPhase(NextPhaseAbilityClass)
      │
      ▼
    上一阶段被自动取消（如果是兄弟关系）
      │
      ▼
ULyraGamePhaseAbility::EndAbility()
      │
      ▼
ULyraGamePhaseSubsystem::OnEndPhase()
      │ 执行 PhaseEndedCallback
      │ 从 ActivePhaseMap 移除
      │ 通知 PhaseEndObservers
      ▼
    新阶段继续运行...
```

---

## 10. 阶段切换的核心算法详解

### 算法伪代码

```
FUNCTION OnBeginPhase(IncomingPhaseAbility, Handle):
    IncomingTag = IncomingPhaseAbility.GetGamePhaseTag()
    
    // 收集所有活跃阶段的 AbilitySpec
    ActiveSpecs = []
    FOR EACH (Handle, Entry) IN ActivePhaseMap:
        Spec = GameState_ASC.FindAbilitySpecFromHandle(Handle)
        IF Spec != null:
            ActiveSpecs.Add(Spec)
    
    // 对每个活跃阶段执行互斥检查
    FOR EACH ActiveSpec IN ActiveSpecs:
        ActiveTag = ActiveSpec.Ability.GetGamePhaseTag()
        
        // 核心判断：新 Tag 是否是旧 Tag 的后代（或相同）
        IF NOT IncomingTag.MatchesTag(ActiveTag):
            // 不是后代关系 → 互斥 → 取消旧阶段
            GameState_ASC.CancelAbilitiesByFunc(
                Lambda: Handle == ActiveSpec.Handle, 
                bReplicateCancel = true)
    
    // 注册新阶段
    ActivePhaseMap[Handle] = { Tag = IncomingTag }
    
    // 通知观察者
    FOR EACH Observer IN PhaseStartObservers:
        IF Observer.IsMatch(IncomingTag):
            Observer.Callback.Execute(IncomingTag)
```

### 具体场景推演

#### 场景 1：标准射击游戏阶段流

```
初始状态：ActivePhaseMap = {}

1. StartPhase(Phase_WaitingToStart)
   IncomingTag = "Game.WaitingToStart"
   ActiveSpecs = [] (空)
   → 无需取消任何阶段
   ActivePhaseMap = { H1: "Game.WaitingToStart" }

2. StartPhase(Phase_Playing)
   IncomingTag = "Game.Playing"
   ActiveSpecs = [H1: "Game.WaitingToStart"]
   
   检查 H1: "Game.Playing".MatchesTag("Game.WaitingToStart") = false
   → 取消 Game.WaitingToStart ✓
   
   ActivePhaseMap = { H2: "Game.Playing" }

3. StartPhase(Phase_CaptureTheFlag)
   IncomingTag = "Game.Playing.CaptureTheFlag"
   ActiveSpecs = [H2: "Game.Playing"]
   
   检查 H2: "Game.Playing.CaptureTheFlag".MatchesTag("Game.Playing") = true (子→父)
   → Game.Playing 保留 ✓
   
   ActivePhaseMap = { H2: "Game.Playing", H3: "Game.Playing.CaptureTheFlag" }

4. StartPhase(Phase_PostGame)
   IncomingTag = "Game.Playing.PostGame"
   ActiveSpecs = [H2: "Game.Playing", H3: "Game.Playing.CaptureTheFlag"]
   
   检查 H2: "Game.Playing.PostGame".MatchesTag("Game.Playing") = true (子→父)
   → Game.Playing 保留 ✓
   
   检查 H3: "Game.Playing.PostGame".MatchesTag("Game.Playing.CaptureTheFlag") = false
   → 取消 Game.Playing.CaptureTheFlag ✓
   
   ActivePhaseMap = { H2: "Game.Playing", H4: "Game.Playing.PostGame" }

5. StartPhase(Phase_GameOver)
   IncomingTag = "Game.GameOver"
   ActiveSpecs = [H2: "Game.Playing", H4: "Game.Playing.PostGame"]
   
   检查 H2: "Game.GameOver".MatchesTag("Game.Playing") = false
   → 取消 Game.Playing ✓
   
   检查 H4: "Game.GameOver".MatchesTag("Game.Playing.PostGame") = false
   → 取消 Game.Playing.PostGame ✓
   
   ActivePhaseMap = { H5: "Game.GameOver" }
```

#### 场景 2：同级多阶段同时激活

```
ActivePhaseMap = { H1: "Game.Playing" }

同时两个 Ability 使用 Tag "Game.Playing"：
StartPhase(Phase_PlayingA)  → IncomingTag = "Game.Playing"
  检查 H1: "Game.Playing".MatchesTag("Game.Playing") = true (自匹配)
  → H1 保留
  ActivePhaseMap = { H1: "Game.Playing", H2: "Game.Playing" }

这允许同一 Tag 下有多个活跃 Ability，适用于"多个系统都关注 Playing 阶段"的场景。
```

---

## 11. 网络复制策略

### GamePhase 本身：ServerOnly

```cpp
// ULyraGamePhaseAbility 构造函数
NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerInitiated;
NetSecurityPolicy = EGameplayAbilityNetSecurityPolicy::ServerOnly;
ReplicationPolicy = EGameplayAbilityReplicationPolicy::ReplicateNo;
```

- Phase Ability 仅在服务器执行
- 不通过 GAS 的 Ability 复制机制复制到客户端
- 客户端如需感知阶段变化，可通过：
  - GameState ASC 上的 GameplayTag 变化（ASC 自带 Tag 复制）
  - 自定义 RPC
  - GameState 上的 Replicated 属性

### Experience：通过属性复制

```cpp
// ULyraExperienceManagerComponent
UPROPERTY(ReplicatedUsing=OnRep_CurrentExperience)
TObjectPtr<const ULyraExperienceDefinition> CurrentExperience;

void OnRep_CurrentExperience()
{
    StartExperienceLoad();  // 客户端收到后自行加载
}
```

### GameState ASC：Mixed 模式

```cpp
AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
```

- Owner（GameState 本身）使用 Full Replication
- 其他客户端使用 Minimal Replication
- GameplayTag 变化会被复制（可用于客户端感知当前 Phase Tag）

---

## 12. 关键设计模式总结

### 模式 1：CallOrRegister（注册即回调）

```
核心思想：无论注册发生在事件之前还是之后，都保证回调执行

应用位置：
  - ULyraExperienceManagerComponent::CallOrRegister_OnExperienceLoaded
  - ULyraGamePhaseSubsystem::WhenPhaseStartsOrIsActive

实现：
  if (已完成) 
      立即执行回调
  else
      注册到回调列表，等待事件触发
```

### 模式 2：Tag 层级互斥

```
核心思想：利用 GameplayTag 的树状结构自动管理状态互斥

规则：
  - 父阶段与子阶段可共存（Game.Playing + Game.Playing.SuddenDeath）
  - 兄弟阶段互斥（Game.Playing.NormalPlay vs Game.Playing.SuddenDeath）
  - 新阶段启动时自动取消所有非祖先阶段

优势：
  - 无需手写状态转移矩阵
  - 通过 Tag 层级天然描述阶段之间的关系
  - 添加新阶段只需定义 Tag，无需修改现有逻辑
```

### 模式 3：Ability as State（能力即状态）

```
核心思想：用 GAS Ability 的激活/结束来表达游戏阶段

优势：
  - 复用 GAS 基础设施（激活、取消、Tag、网络）
  - Phase Ability 可以包含自己的逻辑（定时、条件判断）
  - Phase Ability 可以被其他 Ability 取消/阻止
  - 天然支持并发控制（GAS 的并发策略）
```

### 模式 4：组合化 GameStateComponent

```
核心思想：GameState 功能通过可插拔的组件实现

应用：
  - ExperienceManagerComponent → Experience 加载
  - TeamCreationComponent → 队伍管理
  - BotCreationComponent → Bot 管理
  - PlayerSpawningManagerComponent → 出生点管理
  - FrontendStateComponent → 前端 UI 流程

优势：
  - 每个组件由 Experience 的 Action 动态添加
  - 不同 Experience 可以有不同的组件组合
  - 无需修改 GameState 基类即可扩展功能
```

### 模式 5：三级优先级广播

```
核心思想：Experience 加载完成的通知分三级优先级

  HighPriority → 基础设施初始化（队伍、前端 UI）
  Normal       → 标准游戏逻辑（玩家生成）
  LowPriority  → 后期初始化（Bot 创建）

优势：
  - 解决组件间的初始化依赖问题
  - 无需显式声明依赖关系
  - 新组件只需选择合适的优先级即可
```

### 模式 6：ControlFlow 状态流

```
核心思想：前端 UI 使用流式步骤管理器

  FControlFlowStatics::Create()
    .QueueStep("Step1", ...)
    .QueueStep("Step2", ...)
    .QueueStep("Step3", ...)
    .ExecuteFlow();

优势：
  - 每步可以异步完成（SubFlow->ContinueFlow()）
  - 支持取消流程（SubFlow->CancelFlow()）
  - 调试名称自描述
  - 配合 LoadingScreen 显示当前步骤名称
```

---

## 13. 源码文件索引

### GamePhase 核心（3 个文件）

| 文件 | 说明 |
|------|------|
| `Source/LyraGame/AbilitySystem/Phases/LyraGamePhaseAbility.h/.cpp` | 阶段 Ability 基类 |
| `Source/LyraGame/AbilitySystem/Phases/LyraGamePhaseSubsystem.h/.cpp` | 阶段管理 WorldSubsystem |
| `Source/LyraGame/AbilitySystem/Phases/LyraGamePhaseLog.h` | 日志分类声明 |

### Experience 系统（5 个文件）

| 文件 | 说明 |
|------|------|
| `Source/LyraGame/GameModes/LyraExperienceDefinition.h/.cpp` | 体验定义数据资产 |
| `Source/LyraGame/GameModes/LyraExperienceManagerComponent.h/.cpp` | 体验管理器组件（加载状态机） |
| `Source/LyraGame/GameModes/LyraExperienceActionSet.h/.cpp` | 可组合动作集 |
| `Source/LyraGame/GameModes/LyraExperienceManager.h/.cpp` | PIE 多实例插件仲裁 |
| `Source/LyraGame/GameModes/LyraUserFacingExperienceDefinition.h/.cpp` | 用户面向体验定义 |

### GameMode/GameState 基础设施（3 个文件）

| 文件 | 说明 |
|------|------|
| `Source/LyraGame/GameModes/LyraGameMode.h/.cpp` | 游戏模式（Experience 确定逻辑） |
| `Source/LyraGame/GameModes/LyraGameState.h/.cpp` | 游戏状态（持有全局 ASC） |
| `Source/LyraGame/GameModes/LyraWorldSettings.h/.cpp` | 世界设置（默认 Experience） |

### GameStateComponent 组件（4 个文件）

| 文件 | 说明 |
|------|------|
| `Source/LyraGame/Teams/LyraTeamCreationComponent.h/.cpp` | 队伍创建（HighPriority） |
| `Source/LyraGame/GameModes/LyraBotCreationComponent.h/.cpp` | Bot 创建（LowPriority） |
| `Source/LyraGame/Player/LyraPlayerSpawningManagerComponent.h/.cpp` | 出生点管理 |
| `Source/LyraGame/UI/Frontend/LyraFrontendStateComponent.h/.cpp` | 前端 UI 状态流 |

### 辅助工具（1 个文件）

| 文件 | 说明 |
|------|------|
| `Source/LyraGame/GameModes/AsyncAction_ExperienceReady.h/.cpp` | 蓝图异步等待 Experience 就绪 |
