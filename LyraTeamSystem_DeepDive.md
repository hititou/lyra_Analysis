# Lyra Teams（团队系统）深度代码分析

## 目录

1. [系统概述](#1-系统概述)
2. [架构总览与类图](#2-架构总览与类图)
3. [核心类详解](#3-核心类详解)
4. [异步观察者系统](#4-异步观察者系统)
5. [团队接口的实现者](#5-团队接口的实现者)
6. [团队系统完整生命周期](#6-团队系统完整生命周期)
7. [网络复制架构](#7-网络复制架构)
8. [伤害系统中的团队判定](#8-伤害系统中的团队判定)
9. [团队ID传播链路图](#9-团队id传播链路图)
10. [设计模式总结](#10-设计模式总结)
11. [关键源码文件索引](#11-关键源码文件索引)

---

## 1. 系统概述

Lyra 的 Teams 系统是一套完整的**多团队管理框架**，支持：

- **团队创建与注册**：通过 GameStateComponent 在 Experience 加载后自动创建团队
- **玩家自动分配**：基于最少人数算法自动将新玩家分配到对应团队
- **团队身份传播**：通过 `ILyraTeamAgentInterface` 接口在 PlayerState → Controller → Pawn → LocalPlayer 整条链路上同步团队 ID
- **团队视觉表现**：通过 `ULyraTeamDisplayAsset` 将颜色、材质参数、纹理等应用到角色渲染
- **团队关系判定**：子系统提供 `CompareTeams()` 和 `CanCauseDamage()` 用于友军/敌军判断
- **团队级 Tag 堆栈**：每个团队拥有独立的 `FGameplayTagStackContainer`，可用于存储团队级游戏状态
- **蓝图异步观察**：`AsyncAction_ObserveTeam` 和 `AsyncAction_ObserveTeamColors` 支持蓝图异步监听团队变更

### 核心设计理念

```
1. PlayerState 是团队 ID 的唯一权威来源
2. Controller 和 Pawn 通过委托链自动同步
3. Public/Private Info 实现团队数据的分级复制
4. WorldSubsystem 作为全局查询入口
5. 接口驱动 - 所有参与者实现 ILyraTeamAgentInterface
```

---

## 2. 架构总览与类图

```
                        ┌─────────────────────────┐
                        │   ULyraTeamSubsystem    │  (WorldSubsystem)
                        │  - TeamMap<int32, Info>  │
                        │  - CompareTeams()        │
                        │  - CanCauseDamage()      │
                        │  - FindTeamFromObject()  │
                        │  - ChangeTeamForActor()  │
                        │  - Tag Stack APIs        │
                        └────────┬────────────────┘
                                 │ RegisterTeamInfo / UnregisterTeamInfo
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                   ▼
    ┌─────────────────┐  ┌───────────────┐  ┌───────────────────┐
    │ALyraTeamPublicInfo│ │ALyraTeamPrivate│ │FLyraTeamTrackingInfo│
    │(AInfo, Replicated)│ │Info (AInfo)    │ │ (per-team struct)   │
    │- TeamDisplayAsset │ │- TeamTags     │  │ - PublicInfo        │
    │- TeamTags         │ │               │  │ - PrivateInfo       │
    └────────┬──────────┘ └──────┬────────┘  │ - DisplayAsset      │
             └────────────────────┘           └─────────────────────┘
                       ▲
                       │ Spawn & Configure
              ┌────────┴──────────────┐
              │ULyraTeamCreationComponent│  (GameStateComponent)
              │- TeamsToCreate map       │
              │- ServerCreateTeams()     │
              │- ServerAssignPlayers()   │
              │- GetLeastPopulatedTeam() │
              └──────────────────────────┘

    ┌─────────────────────────────────────────────────────────────┐
    │              ILyraTeamAgentInterface 实现者                   │
    │  ALyraPlayerState (权威来源)                                  │
    │  ALyraPlayerController (委托给PS)                            │
    │  ALyraPlayerBotController (委托给PS)                         │
    │  ALyraCharacter (从Controller同步)                           │
    │  ALyraPawn (从Controller同步)                                │
    │  ULyraLocalPlayer (从PC同步)                                 │
    └─────────────────────────────────────────────────────────────┘
```

---

## 3. 核心类详解

### 3.1 ILyraTeamAgentInterface — 团队代理接口

**文件**: `Source/LyraGame/Teams/LyraTeamAgentInterface.h/.cpp`

整个团队系统的核心接口，继承自 UE 引擎的 `IGenericTeamAgentInterface`。

```cpp
// 团队变更委托
DECLARE_DYNAMIC_MULTICAST_DELEGATE_ThreeParams(FOnLyraTeamIndexChangedDelegate,
    UObject*, ObjectChangingTeam, int32, OldTeamID, int32, NewTeamID);

// FGenericTeamId 与 int32 的双向转换
inline int32 GenericTeamIdToInteger(FGenericTeamId ID)
{
    return (ID == FGenericTeamId::NoTeam) ? INDEX_NONE : (int32)ID;
}
inline FGenericTeamId IntegerToGenericTeamId(int32 ID)
{
    return (ID == INDEX_NONE) ? FGenericTeamId::NoTeam : FGenericTeamId((uint8)ID);
}

UINTERFACE(MinimalAPI, meta=(CannotImplementInterfaceInBlueprint))
class ULyraTeamAgentInterface : public UGenericTeamAgentInterface {};

class ILyraTeamAgentInterface : public IGenericTeamAgentInterface
{
    virtual FOnLyraTeamIndexChangedDelegate* GetOnTeamIndexChangedDelegate() { return nullptr; }

    // 仅在团队真正变更时广播
    static void ConditionalBroadcastTeamChanged(TScriptInterface<ILyraTeamAgentInterface> This,
        FGenericTeamId OldTeamID, FGenericTeamId NewTeamID);

    FOnLyraTeamIndexChangedDelegate& GetTeamChangedDelegateChecked()
    {
        FOnLyraTeamIndexChangedDelegate* Result = GetOnTeamIndexChangedDelegate();
        check(Result);
        return *Result;
    }
};
```

`ConditionalBroadcastTeamChanged` 的实现只在 OldTeamID != NewTeamID 时才触发广播，避免冗余通知。

**关键设计点**：
- `CannotImplementInterfaceInBlueprint` — 仅 C++ 实现，确保类型安全
- `ConditionalBroadcast` — 防止不必要的广播
- 委托采用动态多播 — 支持蓝图绑定

### 3.2 ULyraTeamSubsystem — 团队子系统

**文件**: `Source/LyraGame/Teams/LyraTeamSubsystem.h/.cpp`

团队系统的全局中枢，作为 `UWorldSubsystem` 自动随 World 创建/销毁。

#### 核心数据结构

```cpp
USTRUCT()
struct FLyraTeamTrackingInfo
{
    TObjectPtr<ALyraTeamPublicInfo> PublicInfo = nullptr;
    TObjectPtr<ALyraTeamPrivateInfo> PrivateInfo = nullptr;
    TObjectPtr<ULyraTeamDisplayAsset> DisplayAsset = nullptr;
    FOnLyraTeamDisplayAssetChangedDelegate OnTeamDisplayAssetChanged;

    void SetTeamInfo(ALyraTeamInfoBase* Info);   // 自动区分 Public/Private
    void RemoveTeamInfo(ALyraTeamInfoBase* Info);
};

UENUM(BlueprintType)
enum class ELyraTeamComparison : uint8
{
    OnSameTeam,       // 同一团队
    DifferentTeams,   // 不同团队
    InvalidArgument   // 无效参数
};

class ULyraTeamSubsystem : public UWorldSubsystem
{
    TMap<int32, FLyraTeamTrackingInfo> TeamMap;  // TeamId → TrackingInfo
};
```

`SetTeamInfo` 自动判别类型：对 `ALyraTeamPublicInfo` 设置 PublicInfo 并更新 DisplayAsset；对 `ALyraTeamPrivateInfo` 设置 PrivateInfo。当 DisplayAsset 变更时立即广播 `OnTeamDisplayAssetChanged`。

#### 从对象查找团队 ID — 多路径查找

```cpp
int32 ULyraTeamSubsystem::FindTeamFromObject(const UObject* TestObject) const
{
    // 路径1: 直接实现了 ILyraTeamAgentInterface
    if (const ILyraTeamAgentInterface* Obj = Cast<ILyraTeamAgentInterface>(TestObject))
        return GenericTeamIdToInteger(Obj->GetGenericTeamId());

    if (const AActor* TestActor = Cast<const AActor>(TestObject))
    {
        // 路径2: 通过 Instigator 查找
        if (const ILyraTeamAgentInterface* Instigator = Cast<ILyraTeamAgentInterface>(TestActor->GetInstigator()))
            return GenericTeamIdToInteger(Instigator->GetGenericTeamId());
        // 路径3: 特殊处理 TeamInfo Actor
        if (const ALyraTeamInfoBase* TeamInfo = Cast<ALyraTeamInfoBase>(TestActor))
            return TeamInfo->GetTeamId();
        // 路径4: 通过关联的 PlayerState 查找
        if (const ALyraPlayerState* LyraPS = FindPlayerStateFromActor(TestActor))
            return LyraPS->GetTeamId();
    }
    return INDEX_NONE;
}
```

`FindPlayerStateFromActor` 支持从 Pawn、Controller、或直接 PlayerState 获取。

#### 团队比较与伤害判定

```cpp
ELyraTeamComparison ULyraTeamSubsystem::CompareTeams(const UObject* A, const UObject* B,
    int32& TeamIdA, int32& TeamIdB) const
{
    TeamIdA = FindTeamFromObject(Cast<const AActor>(A));
    TeamIdB = FindTeamFromObject(Cast<const AActor>(B));
    if ((TeamIdA == INDEX_NONE) || (TeamIdB == INDEX_NONE))
        return ELyraTeamComparison::InvalidArgument;
    return (TeamIdA == TeamIdB) ? ELyraTeamComparison::OnSameTeam : ELyraTeamComparison::DifferentTeams;
}

bool ULyraTeamSubsystem::CanCauseDamage(const UObject* Instigator, const UObject* Target,
    bool bAllowDamageToSelf) const
{
    // 自伤: 允许 (通过比较 PlayerState 判断是否同一玩家)
    if (bAllowDamageToSelf && ((Instigator == Target) ||
        (FindPlayerStateFromActor(Cast<AActor>(Instigator)) == FindPlayerStateFromActor(Cast<AActor>(Target)))))
        return true;

    int32 InstigatorTeamId, TargetTeamId;
    const ELyraTeamComparison Relationship = CompareTeams(Instigator, Target, InstigatorTeamId, TargetTeamId);
    if (Relationship == ELyraTeamComparison::DifferentTeams)
        return true;   // 不同团队 → 可以伤害
    else if ((Relationship == ELyraTeamComparison::InvalidArgument) && (InstigatorTeamId != INDEX_NONE))
        return UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(Cast<const AActor>(Target)) != nullptr;
    return false;      // 同一团队 → 不能伤害（无友伤）
}
```

#### 切换团队

```cpp
bool ULyraTeamSubsystem::ChangeTeamForActor(AActor* ActorToChange, int32 NewTeamIndex)
{
    const FGenericTeamId NewTeamID = IntegerToGenericTeamId(NewTeamIndex);
    // 优先通过 PlayerState 设置（它是权威来源）
    if (ALyraPlayerState* LyraPS = const_cast<ALyraPlayerState*>(FindPlayerStateFromActor(ActorToChange)))
    { LyraPS->SetGenericTeamId(NewTeamID); return true; }
    // 其次直接设置（用于无 PlayerState 的 Actor）
    else if (ILyraTeamAgentInterface* TeamActor = Cast<ILyraTeamAgentInterface>(ActorToChange))
    { TeamActor->SetGenericTeamId(NewTeamID); return true; }
    return false;
}
```

#### 团队 Tag 堆栈操作

通过 `ALyraTeamPublicInfo` 或 `ALyraTeamPrivateInfo` 上的 `FGameplayTagStackContainer` 操作，仅 Authority 可修改。查询时合并 Public + Private 的堆栈计数：

```cpp
int32 ULyraTeamSubsystem::GetTeamTagStackCount(int32 TeamId, FGameplayTag Tag) const
{
    if (const FLyraTeamTrackingInfo* Entry = TeamMap.Find(TeamId))
    {
        const int32 PublicCount = Entry->PublicInfo ? Entry->PublicInfo->TeamTags.GetStackCount(Tag) : 0;
        const int32 PrivateCount = Entry->PrivateInfo ? Entry->PrivateInfo->TeamTags.GetStackCount(Tag) : 0;
        return PublicCount + PrivateCount;
    }
    return 0;
}
```

### 3.3 ALyraTeamInfoBase — 团队信息基类

**文件**: `Source/LyraGame/Teams/LyraTeamInfoBase.h/.cpp`

团队的**网络复制实体**，每个团队由 Public + Private 两个 Info Actor 表示。

```cpp
UCLASS(Abstract)
class ALyraTeamInfoBase : public AInfo
{
    friend ULyraTeamCreationComponent;
public:
    int32 GetTeamId() const { return TeamId; }

    UPROPERTY(Replicated)
    FGameplayTagStackContainer TeamTags;  // 团队级 Tag 堆栈

private:
    UPROPERTY(ReplicatedUsing=OnRep_TeamId)
    int32 TeamId;  // COND_InitialOnly
};
```

关键特性：
- `bReplicates = true; bAlwaysRelevant = true; NetPriority = 3.0f` — 高优先级复制
- `TeamId` 使用 `COND_InitialOnly` — 只在首次复制
- `SetTeamId()` 带三重 check：必须 Authority + TeamId 当前为 INDEX_NONE + 新值非 INDEX_NONE
- BeginPlay/OnRep 都调用 `TryRegisterWithTeamSubsystem()` 注册到子系统
- EndPlay 时从子系统注销

### 3.4 ALyraTeamPublicInfo — 公开团队信息

继承 `ALyraTeamInfoBase`，增加 `TeamDisplayAsset` 属性（`COND_InitialOnly` 复制）。所有客户端可见。

### 3.5 ALyraTeamPrivateInfo — 私有团队信息

继承 `ALyraTeamInfoBase`，当前为空实现。代码注释 `@TODO: Actually make private (using replication graph)` 表明计划通过 ReplicationGraph 限制只对同团队客户端可见，但尚未实现。

### 3.6 ULyraTeamCreationComponent — 团队创建组件

**文件**: `Source/LyraGame/Teams/LyraTeamCreationComponent.h/.cpp`

`Blueprintable` 的 GameStateComponent，负责创建团队和分配玩家。

```cpp
UCLASS(Blueprintable)
class ULyraTeamCreationComponent : public UGameStateComponent
{
protected:
    UPROPERTY(EditDefaultsOnly, Category = Teams)
    TMap<uint8, TObjectPtr<ULyraTeamDisplayAsset>> TeamsToCreate;  // 团队ID → 显示资产

    UPROPERTY(EditDefaultsOnly, Category=Teams)
    TSubclassOf<ALyraTeamPublicInfo> PublicTeamInfoClass;

    UPROPERTY(EditDefaultsOnly, Category=Teams)
    TSubclassOf<ALyraTeamPrivateInfo> PrivateTeamInfoClass;

    virtual void ServerCreateTeams();
    virtual void ServerAssignPlayersToTeams();
    virtual void ServerChooseTeamForPlayer(ALyraPlayerState* PS);
};
```

**生命周期**：

1. BeginPlay → 注册 `OnExperienceLoaded_HighPriority` 回调
2. Experience 加载完成 → `ServerCreateTeams()` 遍历 `TeamsToCreate`，为每个团队 Spawn Public + Private Info Actor
3. `ServerAssignPlayersToTeams()` → 遍历已有玩家 + 监听新玩家加入
4. `ServerChooseTeamForPlayer()` → 观察者设为 NoTeam，其他玩家分配到人数最少的团队

**最少人数算法** `GetLeastPopulatedTeamID()`：
- 统计每个团队的活跃（非断线、已分配）玩家数
- 选人数最少的团队，人数相同时选 ID 更小的（确保确定性）

**可扩展性**：`Blueprintable` + `virtual` 方法，子类可覆盖 `ServerChooseTeamForPlayer()` 实现自定义分配逻辑。

### 3.7 ULyraTeamDisplayAsset — 团队显示资产

**文件**: `Source/LyraGame/Teams/LyraTeamDisplayAsset.h/.cpp`

数据驱动的团队视觉配置（UDataAsset）：

```cpp
UCLASS(BlueprintType)
class ULyraTeamDisplayAsset : public UDataAsset
{
    TMap<FName, float> ScalarParameters;
    TMap<FName, FLinearColor> ColorParameters;
    TMap<FName, TObjectPtr<UTexture>> TextureParameters;
    FText TeamShortName;

    void ApplyToMaterial(UMaterialInstanceDynamic* Material);
    void ApplyToMeshComponent(UMeshComponent* MeshComponent);
    void ApplyToNiagaraComponent(UNiagaraComponent* NiagaraComponent);
    void ApplyToActor(AActor* TargetActor, bool bIncludeChildActors = true);
};
```

- `ApplyToMeshComponent` 自动创建动态材质实例来设置纹理参数
- `ApplyToActor` 遍历所有 Mesh 和 Niagara 组件并应用参数
- 编辑器中修改属性时，通过 `PostEditChangeProperty` 通知所有活跃的 `ULyraTeamSubsystem`，实现编辑器实时预览

### 3.8 FGameplayTagStackContainer

**文件**: `Source/LyraGame/System/GameplayTagStack.h`

基于 `FFastArraySerializer` 的可复制 Tag 堆栈。使用增量复制只同步变化的元素。内部维护 `TMap<FGameplayTag, int32>` 加速查询（O(1)）。被团队系统（TeamTags）和玩家统计系统（StatTags）共用。

### 3.9 ULyraTeamStatics — 蓝图静态函数库

**文件**: `Source/LyraGame/Teams/LyraTeamStatics.h/.cpp`

为蓝图提供便捷查询函数：`FindTeamFromObject`（一次返回 TeamId + DisplayAsset）、`GetTeamDisplayAsset`、`GetTeamColorWithFallback`、`GetTeamScalarWithFallback`、`GetTeamTextureWithFallback`。

### 3.10 ULyraTeamCheats — 调试作弊命令

**文件**: `Source/LyraGame/Teams/LyraTeamCheats.h/.cpp`

通过 `UCheatManagerExtension` 注入：
- `CycleTeam` — 环形切换到下一个团队
- `SetTeam <ID>` — 设置到指定团队
- `ListTeams` — 列出所有团队

---

## 4. 异步观察者系统

### 4.1 UAsyncAction_ObserveTeam

**文件**: `Source/LyraGame/Teams/AsyncAction_ObserveTeam.h/.cpp`

蓝图异步节点，监听指定 Agent 的团队变更。

- 激活时立即广播一次当前状态（确保观察者获得初始值）
- 绑定 `GetTeamChangedDelegateChecked()`，未来团队变更时再次广播
- 取消时自动 RemoveAll 清理

```cpp
void UAsyncAction_ObserveTeam::Activate()
{
    int32 CurrentTeamIndex = INDEX_NONE;
    if (ILyraTeamAgentInterface* TeamInterface = TeamInterfacePtr.Get())
    {
        CurrentTeamIndex = GenericTeamIdToInteger(TeamInterface->GetGenericTeamId());
        TeamInterface->GetTeamChangedDelegateChecked().AddDynamic(this, &ThisClass::OnWatchedAgentChangedTeam);
    }
    OnTeamChanged.Broadcast(CurrentTeamIndex != INDEX_NONE, CurrentTeamIndex);  // 立即广播
}
```

### 4.2 UAsyncAction_ObserveTeamColors

**文件**: `Source/LyraGame/Teams/AsyncAction_ObserveTeamColors.h/.cpp`

比 `ObserveTeam` 更高级，同时监听**团队变更**和**显示资产变更**。

关键的 `BroadcastChange` 处理团队切换时的委托重绑定：

```cpp
void UAsyncAction_ObserveTeamColors::BroadcastChange(int32 NewTeam, const ULyraTeamDisplayAsset* DisplayAsset)
{
    const bool bTeamChanged = (LastBroadcastTeamId != NewTeam);
    // 解绑旧团队的 DisplayAsset 监听
    if (TeamSubsystem && bTeamChanged && (LastBroadcastTeamId != INDEX_NONE))
        TeamSubsystem->GetTeamDisplayAssetChangedDelegate(LastBroadcastTeamId).RemoveAll(this);

    LastBroadcastTeamId = NewTeam;
    OnTeamChanged.Broadcast(NewTeam != INDEX_NONE, NewTeam, DisplayAsset);

    // 绑定新团队的 DisplayAsset 监听
    if (TeamSubsystem && bTeamChanged && (NewTeam != INDEX_NONE))
        TeamSubsystem->GetTeamDisplayAssetChangedDelegate(NewTeam).AddDynamic(this, &ThisClass::OnDisplayAssetChanged);
}
```

---

## 5. 团队接口的实现者

### 5.1 ALyraPlayerState — 团队 ID 的权威来源

**文件**: `Source/LyraGame/Player/LyraPlayerState.h/.cpp`

唯一可写的团队 ID 存储点。`MyTeamID` 使用 Push-Based 复制（`MARK_PROPERTY_DIRTY`）优化带宽。

```cpp
void ALyraPlayerState::SetGenericTeamId(const FGenericTeamId& NewTeamID)
{
    if (HasAuthority())
    {
        const FGenericTeamId OldTeamID = MyTeamID;
        MARK_PROPERTY_DIRTY_FROM_NAME(ThisClass, MyTeamID, this);
        MyTeamID = NewTeamID;
        ConditionalBroadcastTeamChanged(this, OldTeamID, NewTeamID);
    }
}

void ALyraPlayerState::OnRep_MyTeamID(FGenericTeamId OldTeamID)
{
    ConditionalBroadcastTeamChanged(this, OldTeamID, MyTeamID);  // 客户端复制后广播
}
```

### 5.2 ALyraPlayerController — 委托给 PlayerState

**不存储团队 ID**。`SetGenericTeamId` 直接打印错误日志拒绝调用。`GetGenericTeamId` 从 PlayerState 获取。

PlayerState 变更时自动重绑定（在 `InitPlayerState`/`CleanupPlayerState`/`OnRep_PlayerState` 中调用 `BroadcastOnPlayerStateChanged`）：解绑旧 PS 的委托 → 绑定新 PS 的委托 → 广播团队变更。

### 5.3 ALyraPlayerBotController

与 PlayerController 完全一致的团队委托逻辑。额外实现了 `GetTeamAttitudeTowards`：不同团队返回 Hostile，同团队返回 Friendly，无法判断返回 Neutral。

### 5.4 ALyraPawn / ALyraCharacter

团队 ID **从 Controller 同步获得**，通过 `ReplicatedUsing` 复制到客户端。

- `PossessedBy` — 从新 Controller 获取团队 ID，绑定 Controller 的团队变更委托
- `UnPossessed` — 解绑旧 Controller，调用 `DetermineNewTeamAfterPossessionEnds`（默认返回 NoTeam）
- `SetGenericTeamId` — 仅在无 Controller 且为 Authority 时允许直接设置
- `OnControllerChangedTeam` — Controller 团队变更时同步更新自身
- `OnRep_MyTeamID` — 客户端复制后广播

### 5.5 ULyraLocalPlayer

从 PlayerController 同步团队 ID。通过 `OnControllerChangedTeam` 监听 PC 的团队变更。

---

## 6. 团队系统完整生命周期

```
1. World 创建
   └→ ULyraTeamSubsystem::Initialize() → 注册 TeamCheats

2. GameMode::InitGame()
   └→ 确定 ExperienceId → ExperienceManagerComponent 开始异步加载

3. ULyraTeamCreationComponent::BeginPlay()
   └→ 注册 OnExperienceLoaded_HighPriority 回调

4. Experience 加载完成 → 三级广播
   [HighPriority] ULyraTeamCreationComponent::OnExperienceLoaded()
     ├→ ServerCreateTeams()
     │   └→ 遍历 TeamsToCreate
     │       ├→ SpawnActor<ALyraTeamPublicInfo> → SetTeamId() + SetTeamDisplayAsset()
     │       └→ SpawnActor<ALyraTeamPrivateInfo> → SetTeamId()
     │       (TeamSubsystem.TeamMap 被填充)
     └→ ServerAssignPlayersToTeams()
         ├→ 遍历已有 PlayerState → ServerChooseTeamForPlayer()
         │   └→ GetLeastPopulatedTeamID() → PS->SetGenericTeamId()
         └→ 绑定 OnGameModePlayerInitialized (监听新玩家)

5. 新玩家加入
   └→ OnGameModePlayerInitialized → ServerChooseTeamForPlayer(NewPS)

6. 客户端接收复制
   ├→ ALyraTeamPublicInfo::OnRep_TeamId() → TryRegisterWithTeamSubsystem()
   ├→ ALyraPlayerState::OnRep_MyTeamID() → ConditionalBroadcast → Controller → Pawn
   └→ Character/Pawn::OnRep_MyTeamID() → ConditionalBroadcast
```

---

## 7. 网络复制架构

### 7.1 复制属性一览

| 类 | 属性 | 复制条件 | 复制模式 |
|---|------|---------|---------|
| `ALyraTeamInfoBase` | `TeamId` | `COND_InitialOnly` | 标准 |
| `ALyraTeamInfoBase` | `TeamTags` | 始终 | FastArraySerializer |
| `ALyraTeamPublicInfo` | `TeamDisplayAsset` | `COND_InitialOnly` | 标准 |
| `ALyraPlayerState` | `MyTeamID` | Push-Based | `MARK_PROPERTY_DIRTY` |
| `ALyraPlayerState` | `MySquadID` | Push-Based | `MARK_PROPERTY_DIRTY` |
| `ALyraPlayerState` | `StatTags` | 始终 | FastArraySerializer |
| `ALyraPawn` | `MyTeamID` | 始终 | 标准 |
| `ALyraCharacter` | `MyTeamID` | 始终 | 标准 |

### 7.2 Public/Private 信息分离

- `ALyraTeamPublicInfo` — `bAlwaysRelevant=true` → 所有客户端可见
- `ALyraTeamPrivateInfo` — 当前也是 `bAlwaysRelevant=true`（`@TODO` 计划通过 ReplicationGraph 限制为仅同队可见）

---

## 8. 伤害系统中的团队判定

**文件**: `Source/LyraGame/AbilitySystem/Executions/LyraDamageExecution.cpp`

```cpp
void ULyraDamageExecution::Execute_Implementation(...) const
{
    // ... 计算基础伤害 ...
    float DamageInteractionAllowedMultiplier = 0.0f;
    if (HitActor)
    {
        ULyraTeamSubsystem* TeamSubsystem = HitActor->GetWorld()->GetSubsystem<ULyraTeamSubsystem>();
        DamageInteractionAllowedMultiplier = TeamSubsystem->CanCauseDamage(EffectCauser, HitActor) ? 1.0 : 0.0;
    }
    // 最终伤害 = BaseDamage × DistanceAttenuation × PhysMaterialAttenuation × TeamMultiplier
    const float DamageDone = FMath::Max(BaseDamage * DistanceAttenuation *
        PhysicalMaterialAttenuation * DamageInteractionAllowedMultiplier, 0.0f);
}
```

| 关系 | 乘数 | 结果 |
|------|------|------|
| 不同团队 | 1.0 | 全额伤害 |
| 同一团队 | 0.0 | 无伤害 |
| 自伤 | 1.0 | 允许 |
| 目标无团队但有ASC | 1.0 | 允许 |

---

## 9. 团队ID传播链路图

```
服务器端传播:
  TeamCreationComponent
    → PlayerState.SetGenericTeamId()
      → PS.ConditionalBroadcast()
        → Controller.OnPlayerStateChangedTeam()
          → Controller.ConditionalBroadcast()
            → Pawn.OnControllerChangedTeam()
              → Pawn.ConditionalBroadcast()
                → LocalPlayer.OnControllerChangedTeam()

客户端传播 (通过复制):
  PlayerState.MyTeamID 复制 → OnRep_MyTeamID() → ConditionalBroadcast()
  Character.MyTeamID 复制   → OnRep_MyTeamID() → ConditionalBroadcast()

权威模型:
  PlayerState (可写)  ← 唯一权威来源
  Controller (只读)   ← 拒绝 SetGenericTeamId, 从 PS 获取
  Pawn (只读)         ← 仅在无 Controller 时可直接设置
  LocalPlayer (只读)  ← 从 Controller 同步
```

---

## 10. 设计模式总结

### 10.1 接口驱动 (Interface-Driven)

`ILyraTeamAgentInterface` 统一 6 种 Actor 的团队协议。`ConditionalBroadcast` 仅在真正变更时触发。

### 10.2 单一权威来源 (Single Source of Truth)

`ALyraPlayerState` 是唯一可写的团队 ID 存储点。Controller、Pawn、LocalPlayer 全部拒绝直接设置，只从 PS 获取。

### 10.3 Public/Private 信息分离

设计为 Public（全部可见）+ Private（仅同队可见），支持团队级数据的差异化复制。

### 10.4 Subsystem 全局查询中枢

`ULyraTeamSubsystem::FindTeamFromObject` 支持从任何对象（Pawn/Controller/PS/TeamInfo/带Instigator的Actor）查找团队。

### 10.5 数据驱动视觉表现

`ULyraTeamDisplayAsset` 通过 Map<FName, 参数值> 驱动材质、Niagara、网格体的视觉参数，完全配置化。

### 10.6 蓝图异步观察者

`ObserveTeam` / `ObserveTeamColors` 采用"立即广播当前值 + 持续监听未来变更"模式，确保观察者无论何时注册都能获得正确状态。

### 10.7 最少人数自动平衡

`GetLeastPopulatedTeamID` 排除断线玩家，确定性选择（人数相同时选小ID）。`Blueprintable` 支持子类自定义分配策略。

---

## 11. 关键源码文件索引

| 文件 | 说明 |
|------|------|
| `Teams/LyraTeamAgentInterface.h/.cpp` | 团队代理接口，GenericTeamId 转换 |
| `Teams/LyraTeamSubsystem.h/.cpp` | WorldSubsystem 中枢 |
| `Teams/LyraTeamInfoBase.h/.cpp` | 团队信息基类 (AInfo) |
| `Teams/LyraTeamPublicInfo.h/.cpp` | 公开信息 + DisplayAsset |
| `Teams/LyraTeamPrivateInfo.h/.cpp` | 私有信息 (预留) |
| `Teams/LyraTeamCreationComponent.h/.cpp` | 团队创建 + 玩家分配 |
| `Teams/LyraTeamDisplayAsset.h/.cpp` | 视觉参数数据资产 |
| `Teams/LyraTeamStatics.h/.cpp` | 蓝图静态函数库 |
| `Teams/LyraTeamCheats.h/.cpp` | 控制台调试命令 |
| `Teams/AsyncAction_ObserveTeam.h/.cpp` | 异步观察团队变更 |
| `Teams/AsyncAction_ObserveTeamColors.h/.cpp` | 异步观察团队颜色 |
| `Player/LyraPlayerState.h/.cpp` | 团队ID权威来源 |
| `Player/LyraPlayerController.h/.cpp` | PC团队委托 |
| `Player/LyraPlayerBotController.h/.cpp` | Bot团队委托+AI态度 |
| `Character/LyraPawn.h/.cpp` | Pawn团队同步 |
| `Character/LyraCharacter.h/.cpp` | Character团队同步 |
| `Player/LyraLocalPlayer.h` | LocalPlayer团队同步 |
| `System/GameplayTagStack.h` | Tag堆栈容器 |
| `AbilitySystem/Executions/LyraDamageExecution.cpp` | 伤害中的团队判定 |
