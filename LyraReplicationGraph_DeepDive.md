# Lyra ReplicationGraph 网络复制图 — 代码级深度分析

## 一、ReplicationGraph 原理概述

### 1.1 传统复制机制的问题

在 UE 的默认网络复制系统中，每帧服务器都需要对**所有已复制的 Actor** 调用 `AActor::IsNetRelevantFor()` 来判断该 Actor 是否对某个连接（客户端）"相关"。这是一个 **O(N × M)** 的操作（N = Actor 数量，M = 连接数量），在大量 Actor 和大量玩家的场景下开销巨大。

### 1.2 ReplicationGraph 解决方案

`UReplicationGraph` 是 UE5 提供的**替代复制驱动器**（`UReplicationDriver`），它用**图节点（Graph Node）**结构来管理 Actor 的复制。核心思想是：

1. **不使用 `AActor::IsNetRelevantFor`**：完全绕过传统的相关性检测
2. **持久化 Actor 列表**：大部分 Actor 列表在帧之间是持久的，避免每帧重建
3. **共享/复用收集工作**：多个连接可以共享同一组 Actor 列表
4. **分层节点**：通过不同类型的节点处理不同类型的 Actor

### 1.3 节点体系架构

```
UReplicationGraph
├── Global Nodes（全局节点，所有连接共享）
│   ├── UReplicationGraphNode_GridSpatialization2D    ← 空间网格化节点
│   ├── UReplicationGraphNode_ActorList               ← 始终相关 Actor 列表
│   └── ULyraReplicationGraphNode_PlayerStateFrequencyLimiter ← PlayerState 频率限制
│
├── Connection Nodes（每个连接独有）
│   ├── ULyraReplicationGraphNode_AlwaysRelevant_ForConnection ← 连接级始终相关
│   └── UReplicationGraphNode_TearOff_ForConnection            ← TearOff 处理（引擎内置）
│
└── ClassRepNodePolicies Map（Actor 类 → 节点路由策略映射）
```

### 1.4 核心工作流

```
每帧服务器复制周期：
┌────────────────────────────────────────────────────────┐
│ 1. PrepareForReplication()                              │
│    各节点准备其 Actor 列表                               │
│                                                         │
│ 2. 对每个 Connection：                                   │
│    ├── 遍历所有 Global Nodes                             │
│    │   └── GatherActorListsForConnection(Params)        │
│    │       → 输出该连接应复制的 Actor 列表                │
│    ├── 遍历该 Connection 的专属 Nodes                    │
│    │   └── GatherActorListsForConnection(Params)        │
│    │       → 输出该连接始终相关的 Actor 列表              │
│    └── 合并所有列表 → 优先级排序 → 序列化发送             │
│                                                         │
│ 3. Actor 动态变化时：                                    │
│    ├── RouteAddNetworkActorToNodes() → Actor 加入节点     │
│    └── RouteRemoveNetworkActorToNodes() → Actor 移出节点  │
└────────────────────────────────────────────────────────┘
```

---

## 二、Lyra 的 ReplicationGraph 实现架构

### 2.1 核心文件结构

| 文件 | 大小 | 职责 |
|------|------|------|
| `LyraReplicationGraph.h` | 4.73 KB | 主类声明 + 3 个自定义节点声明 |
| `LyraReplicationGraph.cpp` | 41.89 KB | 完整实现（项目最大单文件） |
| `LyraReplicationGraphTypes.h` | 3.08 KB | 路由枚举 + 配置数据结构 |
| `LyraReplicationGraphSettings.h/cpp` | 3.4 KB | 开发者设置（CVar 驱动） |

### 2.2 类继承关系

```
UReplicationGraph (引擎)
└── ULyraReplicationGraph

UReplicationGraphNode (引擎)
├── UReplicationGraphNode_GridSpatialization2D (引擎) ← Lyra 直接使用
├── UReplicationGraphNode_ActorList (引擎) ← Lyra 直接使用
├── UReplicationGraphNode_AlwaysRelevant_ForConnection (引擎)
│   └── ULyraReplicationGraphNode_AlwaysRelevant_ForConnection (Lyra 自定义)
└── ULyraReplicationGraphNode_PlayerStateFrequencyLimiter (Lyra 自定义)
```

### 2.3 ULyraReplicationGraph 类声明

```cpp
// Source/LyraGame/System/LyraReplicationGraph.h
UCLASS(transient, config=Engine)
class ULyraReplicationGraph : public UReplicationGraph
{
    GENERATED_BODY()

public:
    ULyraReplicationGraph();

    // 重写的核心虚函数
    virtual void ResetGameWorldState() override;
    virtual void InitGlobalActorClassSettings() override;    // 初始化 Actor 类→节点映射
    virtual void InitGlobalGraphNodes() override;            // 创建全局图节点
    virtual void InitConnectionGraphNodes(UNetReplicationGraphConnection*) override; // 创建连接级图节点
    virtual void RouteAddNetworkActorToNodes(const FNewReplicatedActorInfo&, FGlobalActorReplicationInfo&) override;
    virtual void RouteRemoveNetworkActorToNodes(const FNewReplicatedActorInfo&) override;

    UPROPERTY()
    TArray<TObjectPtr<UClass>> AlwaysRelevantClasses;

    UPROPERTY()
    TObjectPtr<UReplicationGraphNode_GridSpatialization2D> GridNode;  // 空间网格节点

    UPROPERTY()
    TObjectPtr<UReplicationGraphNode_ActorList> AlwaysRelevantNode;  // 始终相关节点

    TMap<FName, FActorRepListRefView> AlwaysRelevantStreamingLevelActors; // 流式关卡相关 Actor

private:
    TClassMap<EClassRepNodeMapping> ClassRepNodePolicies;  // 类→路由策略映射表
    TArray<UClass*> ExplicitlySetClasses;                  // 显式配置过的类列表
};
```

---

## 三、Actor 路由策略（EClassRepNodeMapping）

### 3.1 路由枚举定义

```cpp
// Source/LyraGame/System/LyraReplicationGraphTypes.h
UENUM()
enum class EClassRepNodeMapping : uint32
{
    NotRouted,                  // 不路由到任何节点（由特殊节点处理，如 PlayerState）
    RelevantAllConnections,     // 路由到 AlwaysRelevantNode 或流式关卡节点

    // 以下为空间化枚举 —— IsSpatialized() 判断依据
    Spatialize_Static,          // 静态 Actor：不移动，不需每帧更新
    Spatialize_Dynamic,         // 动态 Actor：频繁移动，每帧更新
    Spatialize_Dormancy,        // 休眠 Actor：休眠时当静态，唤醒时当动态
};
```

### 3.2 路由策略的自动推导

当一个 Actor 类没有被显式配置时，系统通过 CDO（Class Default Object）的属性**自动推导**路由策略：

```cpp
// Source/LyraGame/System/LyraReplicationGraph.cpp
EClassRepNodeMapping ULyraReplicationGraph::GetClassNodeMapping(UClass* Class) const
{
    // 1. 先查缓存
    if (const EClassRepNodeMapping* Ptr = ClassRepNodePolicies.FindWithoutClassRecursion(Class))
        return *Ptr;

    AActor* ActorCDO = Cast<AActor>(Class->GetDefaultObject());
    if (!ActorCDO || !ActorCDO->GetIsReplicated())
        return EClassRepNodeMapping::NotRouted;  // 不复制的 Actor 不路由

    // 2. 判断是否需要空间化
    auto ShouldSpatialize = [](const AActor* CDO)
    {
        return CDO->GetIsReplicated() &&
            (!(CDO->bAlwaysRelevant || CDO->bOnlyRelevantToOwner || CDO->bNetUseOwnerRelevancy));
    };

    // 3. 只有在子类与父类属性不同时才创建新映射
    UClass* SuperClass = Class->GetSuperClass();
    if (AActor* SuperCDO = Cast<AActor>(SuperClass->GetDefaultObject()))
    {
        if (SuperCDO->GetIsReplicated() == ActorCDO->GetIsReplicated()
            && SuperCDO->bAlwaysRelevant == ActorCDO->bAlwaysRelevant
            && SuperCDO->bOnlyRelevantToOwner == ActorCDO->bOnlyRelevantToOwner
            && SuperCDO->bNetUseOwnerRelevancy == ActorCDO->bNetUseOwnerRelevancy)
        {
            return GetClassNodeMapping(SuperClass);  // 继承父类策略
        }
    }

    // 4. 最终判断
    if (ShouldSpatialize(ActorCDO))
        return EClassRepNodeMapping::Spatialize_Dynamic;       // 默认空间化为动态
    else if (ActorCDO->bAlwaysRelevant && !ActorCDO->bOnlyRelevantToOwner)
        return EClassRepNodeMapping::RelevantAllConnections;   // 始终相关
    
    return EClassRepNodeMapping::NotRouted;
}
```

**推导规则总结**：

| Actor 属性 | 路由策略 |
|------------|----------|
| `bAlwaysRelevant = true` & `!bOnlyRelevantToOwner` | `RelevantAllConnections` |
| `bAlwaysRelevant = false` & `bOnlyRelevantToOwner = false` & `bNetUseOwnerRelevancy = false` | `Spatialize_Dynamic` |
| `bOnlyRelevantToOwner = true` 或其他特殊情况 | `NotRouted`（由专用节点处理）|

### 3.3 配置文件中的显式路由

```ini
; Config/DefaultGame.ini
[/Script/LyraGame.LyraReplicationGraphSettings]
bDisableReplicationGraph=True
DefaultReplicationGraphClass=/Script/LyraGame.LyraReplicationGraph

; PlayerState → NotRouted（由 PlayerStateFrequencyLimiter 和 ForConnection 节点处理）
+ClassSettings=(ActorClass="/Script/Engine.PlayerState",bAddClassRepInfoToMap=True,ClassNodeMapping=NotRouted)

; LevelScriptActor → NotRouted
+ClassSettings=(ActorClass="/Script/Engine.LevelScriptActor",bAddClassRepInfoToMap=True,ClassNodeMapping=NotRouted)

; ReplicationGraphDebugActor → NotRouted
+ClassSettings=(ActorClass="/Script/ReplicationGraph.ReplicationGraphDebugActor",bAddClassRepInfoToMap=True,ClassNodeMapping=NotRouted)

; LyraPlayerController → NotRouted（通过 ForConnection 节点处理）
+ClassSettings=(ActorClass="/Script/LyraGame.LyraPlayerController",bAddClassRepInfoToMap=True,ClassNodeMapping=NotRouted)
```

代码中加载这些配置：

```cpp
void ULyraReplicationGraph::InitGlobalActorClassSettings()
{
    const ULyraReplicationGraphSettings* LyraRepGraphSettings = GetDefault<ULyraReplicationGraphSettings>();

    // 从配置加载显式 Class → Node 映射
    for (const FRepGraphActorClassSettings& ActorClassSettings : LyraRepGraphSettings->ClassSettings)
    {
        if (ActorClassSettings.bAddClassRepInfoToMap)
        {
            if (UClass* StaticActorClass = ActorClassSettings.GetStaticActorClass())
            {
                AddClassRepInfo(StaticActorClass, ActorClassSettings.ClassNodeMapping);
            }
        }
    }

    // GameplayDebugger 特殊处理
    #if WITH_GAMEPLAY_DEBUGGER
    AddClassRepInfo(AGameplayDebuggerCategoryReplicator::StaticClass(), EClassRepNodeMapping::NotRouted);
    #endif
    // ...
}
```

---

## 四、ReplicationGraph 创建与初始化流程

### 4.1 创建 ReplicationDriver

Lyra 通过 `UReplicationDriver::CreateReplicationDriverDelegate()` 注册工厂函数：

```cpp
// Source/LyraGame/System/LyraReplicationGraph.cpp
ULyraReplicationGraph::ULyraReplicationGraph()
{
    if (!UReplicationDriver::CreateReplicationDriverDelegate().IsBound())
    {
        UReplicationDriver::CreateReplicationDriverDelegate().BindLambda(
            [](UNetDriver* ForNetDriver, const FURL& URL, UWorld* World) -> UReplicationDriver*
            {
                return Lyra::RepGraph::ConditionalCreateReplicationDriver(ForNetDriver, World);
            });
    }
}
```

工厂函数中进行条件检查：

```cpp
namespace Lyra::RepGraph
{
    UReplicationDriver* ConditionalCreateReplicationDriver(UNetDriver* ForNetDriver, UWorld* World)
    {
        // 只为 GameNetDriver 创建（不为 DemoNetDriver/BeaconDriver 创建）
        if (World && ForNetDriver && ForNetDriver->NetDriverName == NAME_GameNetDriver)
        {
            const ULyraReplicationGraphSettings* Settings = GetDefault<ULyraReplicationGraphSettings>();

            // 开发者设置中的开关
            if (Settings && Settings->bDisableReplicationGraph)
            {
                UE_LOG(LogLyraRepGraph, Display, TEXT("Replication graph is disabled via LyraReplicationGraphSettings."));
                return nullptr;
            }

            // 从设置加载 Graph 类（支持子类覆盖）
            TSubclassOf<ULyraReplicationGraph> GraphClass;
            if (Settings)
            {
                GraphClass = Settings->DefaultReplicationGraphClass.TryLoadClass<ULyraReplicationGraph>();
            }
            if (GraphClass.Get() == nullptr)
            {
                GraphClass = ULyraReplicationGraph::StaticClass();
            }

            return NewObject<ULyraReplicationGraph>(GetTransientPackage(), GraphClass.Get());
        }
        return nullptr;
    }
};
```

### 4.2 初始化全局 Actor 类设置（InitGlobalActorClassSettings）

这是整个 ReplicationGraph 初始化中**最核心、最复杂**的函数（~200 行），完成以下工作：

#### 4.2.1 设置懒加载初始化函数

```cpp
void ULyraReplicationGraph::InitGlobalActorClassSettings()
{
    // 为运行时新发现的类设置延迟初始化函数
    GlobalActorReplicationInfoMap.SetInitClassInfoFunc(
        [this](UClass* Class, FClassReplicationInfo& ClassInfo)
        {
            RegisterClassRepNodeMapping(Class);  // 先注册路由策略
            const bool bHandled = ConditionalInitClassReplicationInfo(Class, ClassInfo);
            // Debug 日志...
            return bHandled;
        });

    // 为 ClassRepNodePolicies 设置新元素初始化函数
    ClassRepNodePolicies.InitNewElement = [this](UClass* Class, EClassRepNodeMapping& NodeMapping)
    {
        NodeMapping = GetClassNodeMapping(Class);
        return true;
    };
```

#### 4.2.2 遍历所有已加载的可复制类

```cpp
    TArray<UClass*> AllReplicatedClasses;

    for (TObjectIterator<UClass> It; It; ++It)
    {
        UClass* Class = *It;
        AActor* ActorCDO = Cast<AActor>(Class->GetDefaultObject());
        if (!ActorCDO || !ActorCDO->GetIsReplicated())
            continue;

        // 跳过编辑器临时类
        if (Class->GetName().StartsWith(TEXT("SKEL_")) || Class->GetName().StartsWith(TEXT("REINST_")))
            continue;

        AllReplicatedClasses.Add(Class);
        RegisterClassRepNodeMapping(Class);  // 为每个类注册路由策略
    }
```

#### 4.2.3 配置 Character 类的复制信息

```cpp
    // Character 类的显式配置
    FClassReplicationInfo CharacterClassRepInfo;
    CharacterClassRepInfo.DistancePriorityScale = 1.f;     // 距离优先级权重
    CharacterClassRepInfo.StarvationPriorityScale = 1.f;   // 饥饿优先级权重
    CharacterClassRepInfo.ActorChannelFrameTimeout = 4;     // 通道超时帧数
    CharacterClassRepInfo.SetCullDistanceSquared(
        ALyraCharacter::StaticClass()->GetDefaultObject<ALyraCharacter>()->GetNetCullDistanceSquared());

    SetClassInfo(ACharacter::StaticClass(), CharacterClassRepInfo);
```

#### 4.2.4 配置 FastShared 路径

```cpp
    // FastShared 复制：角色移动数据的高效共享序列化
    CharacterClassRepInfo.FastSharedReplicationFunc = [](AActor* Actor)
    {
        bool bSuccess = false;
        if (ALyraCharacter* Character = Cast<ALyraCharacter>(Actor))
        {
            bSuccess = Character->UpdateSharedReplication();
        }
        return bSuccess;
    };

    CharacterClassRepInfo.FastSharedReplicationFuncName = FName(TEXT("FastSharedReplication"));

    // FastShared 带宽限制：每帧最大 bit 数 = (目标KB/s × 1024 × 8) / Tick率
    FastSharedPathConstants.MaxBitsPerFrame = (int32)(
        (float)(Lyra::RepGraph::TargetKBytesSecFastSharedPath * 1024 * 8) / NetDriver->GetNetServerMaxTickRate());
    FastSharedPathConstants.DistanceRequirementPct = Lyra::RepGraph::FastSharedPathCullDistPct;

    SetClassInfo(ALyraCharacter::StaticClass(), CharacterClassRepInfo);
```

#### 4.2.5 配置频率桶

```cpp
    // 动态空间化 Actor 的频率桶设置
    UReplicationGraphNode_ActorListFrequencyBuckets::DefaultSettings.ListSize = 12;
    UReplicationGraphNode_ActorListFrequencyBuckets::DefaultSettings.NumBuckets =
        Lyra::RepGraph::DynamicActorFrequencyBuckets;  // 默认 3 桶
    UReplicationGraphNode_ActorListFrequencyBuckets::DefaultSettings.EnableFastPath =
        (Lyra::RepGraph::EnableFastSharedPath > 0);
    UReplicationGraphNode_ActorListFrequencyBuckets::DefaultSettings.FastPathFrameModulo = 1;
```

#### 4.2.6 配置 Multicast RPC 通道策略

```cpp
    // Multicast RPC 是否应自动打开通道
    RPC_Multicast_OpenChannelForClass.Reset();
    RPC_Multicast_OpenChannelForClass.Set(AActor::StaticClass(), true);       // 默认：是
    RPC_Multicast_OpenChannelForClass.Set(AController::StaticClass(), false);  // Controller：否（避免破坏 Owner 复制）
    RPC_Multicast_OpenChannelForClass.Set(AServerStatReplicator::StaticClass(), false);

    // 从配置中读取额外设置
    for (const FRepGraphActorClassSettings& ActorClassSettings : LyraRepGraphSettings->ClassSettings)
    {
        if (ActorClassSettings.bAddToRPC_Multicast_OpenChannelForClassMap)
        {
            if (UClass* StaticActorClass = ActorClassSettings.GetStaticActorClass())
            {
                RPC_Multicast_OpenChannelForClass.Set(
                    StaticActorClass, ActorClassSettings.bRPC_Multicast_OpenChannelForClass);
            }
        }
    }
```

#### 4.2.7 配置销毁信息距离

```cpp
    // 销毁信息的最大复制距离
    DestructInfoMaxDistanceSquared = Lyra::RepGraph::DestructionInfoMaxDist * Lyra::RepGraph::DestructionInfoMaxDist;
    // 默认值：30000cm × 30000cm = 900,000,000
```

### 4.3 初始化全局图节点（InitGlobalGraphNodes）

```cpp
void ULyraReplicationGraph::InitGlobalGraphNodes()
{
    // ① 空间网格化节点 — 处理基于距离的相关性
    GridNode = CreateNewNode<UReplicationGraphNode_GridSpatialization2D>();
    GridNode->CellSize = Lyra::RepGraph::CellSize;                           // 默认 10000cm = 100m
    GridNode->SpatialBias = FVector2D(Lyra::RepGraph::SpatialBiasX,          // 默认 (-150000, -200000)
                                      Lyra::RepGraph::SpatialBiasY);

    if (Lyra::RepGraph::DisableSpatialRebuilds)
    {
        GridNode->AddToClassRebuildDenyList(AActor::StaticClass());  // 禁用空间重建（优化）
    }
    AddGlobalGraphNode(GridNode);

    // ② 始终相关节点 — 对所有连接始终相关的 Actor
    AlwaysRelevantNode = CreateNewNode<UReplicationGraphNode_ActorList>();
    AddGlobalGraphNode(AlwaysRelevantNode);

    // ③ PlayerState 频率限制节点 — 限制 PlayerState 的复制频率
    ULyraReplicationGraphNode_PlayerStateFrequencyLimiter* PlayerStateNode =
        CreateNewNode<ULyraReplicationGraphNode_PlayerStateFrequencyLimiter>();
    AddGlobalGraphNode(PlayerStateNode);
}
```

### 4.4 初始化连接级图节点（InitConnectionGraphNodes）

```cpp
void ULyraReplicationGraph::InitConnectionGraphNodes(UNetReplicationGraphConnection* RepGraphConnection)
{
    Super::InitConnectionGraphNodes(RepGraphConnection);

    // 为每个连接创建一个"始终相关"节点
    ULyraReplicationGraphNode_AlwaysRelevant_ForConnection* AlwaysRelevantConnectionNode =
        CreateNewNode<ULyraReplicationGraphNode_AlwaysRelevant_ForConnection>();

    // 监听客户端流式关卡可见性变化
    RepGraphConnection->OnClientVisibleLevelNameAdd.AddUObject(
        AlwaysRelevantConnectionNode,
        &ULyraReplicationGraphNode_AlwaysRelevant_ForConnection::OnClientLevelVisibilityAdd);
    RepGraphConnection->OnClientVisibleLevelNameRemove.AddUObject(
        AlwaysRelevantConnectionNode,
        &ULyraReplicationGraphNode_AlwaysRelevant_ForConnection::OnClientLevelVisibilityRemove);

    AddConnectionGraphNode(AlwaysRelevantConnectionNode, RepGraphConnection);
}
```

---

## 五、Actor 路由机制（RouteAddNetworkActorToNodes / RouteRemoveNetworkActorToNodes）

当一个新的需要复制的 Actor 进入世界时，`RouteAddNetworkActorToNodes` 被调用，根据路由策略将其加入对应节点：

```cpp
void ULyraReplicationGraph::RouteAddNetworkActorToNodes(
    const FNewReplicatedActorInfo& ActorInfo,
    FGlobalActorReplicationInfo& GlobalInfo)
{
    EClassRepNodeMapping Policy = GetMappingPolicy(ActorInfo.Class);

    switch(Policy)
    {
        case EClassRepNodeMapping::NotRouted:
            break;  // 什么都不做，由特殊节点（ForConnection / PlayerStateLimiter）自行处理

        case EClassRepNodeMapping::RelevantAllConnections:
            if (ActorInfo.StreamingLevelName == NAME_None)
            {
                // 持久关卡 Actor → 加入全局 AlwaysRelevant 列表
                AlwaysRelevantNode->NotifyAddNetworkActor(ActorInfo);
            }
            else
            {
                // 流式关卡 Actor → 加入对应关卡的列表
                FActorRepListRefView& RepList =
                    AlwaysRelevantStreamingLevelActors.FindOrAdd(ActorInfo.StreamingLevelName);
                RepList.ConditionalAdd(ActorInfo.Actor);
            }
            break;

        case EClassRepNodeMapping::Spatialize_Static:
            GridNode->AddActor_Static(ActorInfo, GlobalInfo);   // 静态格子：放入后不更新位置
            break;

        case EClassRepNodeMapping::Spatialize_Dynamic:
            GridNode->AddActor_Dynamic(ActorInfo, GlobalInfo);  // 动态格子：每帧更新位置
            break;

        case EClassRepNodeMapping::Spatialize_Dormancy:
            GridNode->AddActor_Dormancy(ActorInfo, GlobalInfo); // 休眠感知：根据休眠状态切换
            break;
    };
}
```

移除时的对称操作：

```cpp
void ULyraReplicationGraph::RouteRemoveNetworkActorToNodes(const FNewReplicatedActorInfo& ActorInfo)
{
    EClassRepNodeMapping Policy = GetMappingPolicy(ActorInfo.Class);

    switch(Policy)
    {
        case EClassRepNodeMapping::NotRouted:
            break;

        case EClassRepNodeMapping::RelevantAllConnections:
            if (ActorInfo.StreamingLevelName == NAME_None)
            {
                AlwaysRelevantNode->NotifyRemoveNetworkActor(ActorInfo);
            }
            else
            {
                FActorRepListRefView& RepList =
                    AlwaysRelevantStreamingLevelActors.FindChecked(ActorInfo.StreamingLevelName);
                RepList.RemoveFast(ActorInfo.Actor);
            }
            // 设置销毁信息忽略距离裁剪（确保所有客户端都知道此 Actor 被销毁）
            SetActorDestructionInfoToIgnoreDistanceCulling(ActorInfo.GetActor());
            break;

        case EClassRepNodeMapping::Spatialize_Static:
            GridNode->RemoveActor_Static(ActorInfo);
            break;

        case EClassRepNodeMapping::Spatialize_Dynamic:
            GridNode->RemoveActor_Dynamic(ActorInfo);
            break;

        case EClassRepNodeMapping::Spatialize_Dormancy:
            GridNode->RemoveActor_Dormancy(ActorInfo);
            break;
    };
}
```

**路由决策流程图**：

```
新 Actor 需要复制
    │
    ▼
查找 ClassRepNodePolicies[ActorClass]
    │
    ├─── NotRouted ────────────────────────────► 不加入任何节点
    │                                             （PlayerState/Controller 等由专用节点直接处理）
    │
    ├─── RelevantAllConnections ───┬── 持久关卡 → AlwaysRelevantNode
    │                              └── 流式关卡 → AlwaysRelevantStreamingLevelActors[LevelName]
    │
    ├─── Spatialize_Static ────────────────────► GridNode.AddActor_Static()
    │
    ├─── Spatialize_Dynamic ───────────────────► GridNode.AddActor_Dynamic()
    │
    └─── Spatialize_Dormancy ──────────────────► GridNode.AddActor_Dormancy()
```

---

## 六、自定义图节点详解

### 6.1 ULyraReplicationGraphNode_AlwaysRelevant_ForConnection

**职责**：为每个客户端连接收集该连接始终需要复制的 Actor（如自己的 PlayerController、Pawn、PlayerState 等）。

**关键特性**：
- **不维护持久列表**：每帧重建列表，因为这些 Actor 可以直接从 PlayerController 获取
- **流式关卡支持**：跟踪客户端可见的流式关卡，复制其中的始终相关 Actor
- **GameplayDebugger 支持**：可选地包含调试复制器

```cpp
void ULyraReplicationGraphNode_AlwaysRelevant_ForConnection::GatherActorListsForConnection(
    const FConnectionGatherActorListParameters& Params)
{
    ULyraReplicationGraph* LyraGraph = CastChecked<ULyraReplicationGraph>(GetOuter());
    ReplicationActorList.Reset();  // 每帧清空重建

    for (const FNetViewer& CurViewer : Params.Viewers)
    {
        // ① 添加 Viewer（通常是 PlayerController）和 ViewTarget
        ReplicationActorList.ConditionalAdd(CurViewer.InViewer);
        ReplicationActorList.ConditionalAdd(CurViewer.ViewTarget);

        if (ALyraPlayerController* PC = Cast<ALyraPlayerController>(CurViewer.InViewer))
        {
            // ② PlayerState 50% 节流：奇偶帧交替复制
            const bool bReplicatePS =
                (Params.ConnectionManager.ConnectionOrderNum % 2) == (Params.ReplicationFrameNum % 2);
            if (bReplicatePS)
            {
                if (APlayerState* PS = PC->PlayerState)
                {
                    if (!bInitializedPlayerState)
                    {
                        bInitializedPlayerState = true;
                        // 为自己的 PlayerState 设置更高频率（每帧1次）
                        FConnectionReplicationActorInfo& ConnectionActorInfo =
                            Params.ConnectionManager.ActorInfoMap.FindOrAdd(PS);
                        ConnectionActorInfo.ReplicationPeriodFrame = 1;
                    }
                    ReplicationActorList.ConditionalAdd(PS);
                }
            }

            // ③ 添加 Pawn 及其缓存的相关 Actor
            FCachedAlwaysRelevantActorInfo& LastData = PastRelevantActorMap.FindOrAdd(CurViewer.Connection);

            if (ALyraCharacter* Pawn = Cast<ALyraCharacter>(PC->GetPawn()))
            {
                UpdateCachedRelevantActor(Params, Pawn, LastData.LastViewer);
                if (Pawn != CurViewer.ViewTarget)
                {
                    ReplicationActorList.ConditionalAdd(Pawn);
                }
            }

            if (ALyraCharacter* ViewTargetPawn = Cast<ALyraCharacter>(CurViewer.ViewTarget))
            {
                UpdateCachedRelevantActor(Params, ViewTargetPawn, LastData.LastViewTarget);
            }
        }
    }

    CleanupCachedRelevantActors(PastRelevantActorMap);

    // ④ 处理流式关卡中的始终相关 Actor
    TMap<FName, FActorRepListRefView>& StreamingLevelActors =
        LyraGraph->AlwaysRelevantStreamingLevelActors;

    for (int32 Idx = AlwaysRelevantStreamingLevelsNeedingReplication.Num() - 1; Idx >= 0; --Idx)
    {
        const FName& StreamingLevel = AlwaysRelevantStreamingLevelsNeedingReplication[Idx];
        FActorRepListRefView* Ptr = StreamingLevelActors.Find(StreamingLevel);

        if (Ptr == nullptr)
        {
            // 关卡没有始终相关 Actor，移除
            AlwaysRelevantStreamingLevelsNeedingReplication.RemoveAtSwap(Idx, EAllowShrinking::No);
            continue;
        }

        FActorRepListRefView& RepList = *Ptr;
        if (RepList.Num() > 0)
        {
            // 检查是否所有 Actor 都处于休眠状态
            bool bAllDormant = true;
            for (FActorRepListType Actor : RepList)
            {
                FConnectionReplicationActorInfo& ConnectionActorInfo =
                    ConnectionActorInfoMap.FindOrAdd(Actor);
                if (ConnectionActorInfo.bDormantOnConnection == false)
                {
                    bAllDormant = false;
                    break;
                }
            }

            if (bAllDormant)
            {
                // 全部休眠 → 不再复制该关卡
                AlwaysRelevantStreamingLevelsNeedingReplication.RemoveAtSwap(Idx, EAllowShrinking::No);
            }
            else
            {
                // 有活跃 Actor → 加入复制列表
                Params.OutGatheredReplicationLists.AddReplicationActorList(RepList);
            }
        }
    }

    // ⑤ GameplayDebugger
    #if WITH_GAMEPLAY_DEBUGGER
    if (GameplayDebugger)
    {
        ReplicationActorList.ConditionalAdd(GameplayDebugger);
    }
    #endif

    // 输出最终列表
    Params.OutGatheredReplicationLists.AddReplicationActorList(ReplicationActorList);
}
```

**流式关卡可见性管理**：

```cpp
void ULyraReplicationGraphNode_AlwaysRelevant_ForConnection::OnClientLevelVisibilityAdd(
    FName LevelName, UWorld* StreamingWorld)
{
    // 客户端加载了新关卡 → 需要复制该关卡的始终相关 Actor
    AlwaysRelevantStreamingLevelsNeedingReplication.Add(LevelName);
}

void ULyraReplicationGraphNode_AlwaysRelevant_ForConnection::OnClientLevelVisibilityRemove(FName LevelName)
{
    // 客户端卸载关卡 → 停止复制
    AlwaysRelevantStreamingLevelsNeedingReplication.Remove(LevelName);
}
```

### 6.2 ULyraReplicationGraphNode_PlayerStateFrequencyLimiter

**职责**：限制 PlayerState 对非拥有者连接的复制频率。在大量玩家的场景下，不需要每帧为每个连接复制所有 PlayerState。

**核心原理**：将所有 PlayerState 分成多个桶（bucket），每帧只返回一个桶的 Actor。

```cpp
ULyraReplicationGraphNode_PlayerStateFrequencyLimiter::ULyraReplicationGraphNode_PlayerStateFrequencyLimiter()
{
    bRequiresPrepareForReplicationCall = true;  // 需要在每帧复制前调用 PrepareForReplication
}

void ULyraReplicationGraphNode_PlayerStateFrequencyLimiter::PrepareForReplication()
{
    ReplicationActorLists.Reset();
    ForceNetUpdateReplicationActorList.Reset();

    ReplicationActorLists.AddDefaulted();
    FActorRepListRefView* CurrentList = &ReplicationActorLists[0];

    // 每帧重建 PlayerState 列表（处理玩家断线的简单方式）
    for (TActorIterator<APlayerState> It(GetWorld()); It; ++It)
    {
        APlayerState* PS = *It;
        if (IsActorValidForReplicationGather(PS) == false)
            continue;

        // 当前桶满了 → 创建新桶
        if (CurrentList->Num() >= TargetActorsPerFrame)  // 默认每桶 2 个
        {
            ReplicationActorLists.AddDefaulted();
            CurrentList = &ReplicationActorLists.Last();
        }

        CurrentList->Add(PS);
    }
}

void ULyraReplicationGraphNode_PlayerStateFrequencyLimiter::GatherActorListsForConnection(
    const FConnectionGatherActorListParameters& Params)
{
    // 轮询桶：帧号 % 桶数 = 本帧使用的桶
    const int32 ListIdx = Params.ReplicationFrameNum % ReplicationActorLists.Num();
    Params.OutGatheredReplicationLists.AddReplicationActorList(ReplicationActorLists[ListIdx]);

    // ForceNetUpdate 的 Actor 不受频率限制
    if (ForceNetUpdateReplicationActorList.Num() > 0)
    {
        Params.OutGatheredReplicationLists.AddReplicationActorList(ForceNetUpdateReplicationActorList);
    }
}
```

**效果示例**（假设 8 个 PlayerState，TargetActorsPerFrame=2）：

```
桶分配:
  Bucket[0] = [PS_0, PS_1]
  Bucket[1] = [PS_2, PS_3]
  Bucket[2] = [PS_4, PS_5]
  Bucket[3] = [PS_6, PS_7]

帧  0 → 返回 Bucket[0] (PS_0, PS_1)
帧  1 → 返回 Bucket[1] (PS_2, PS_3)
帧  2 → 返回 Bucket[2] (PS_4, PS_5)
帧  3 → 返回 Bucket[3] (PS_6, PS_7)
帧  4 → 返回 Bucket[0] (PS_0, PS_1)  ← 周期重复
...

效果：每个 PlayerState 每 4 帧复制一次，而非每帧，带宽降低 75%
```

---

## 七、FastShared Replication（快速共享复制）

### 7.1 原理

FastShared Replication 是 ReplicationGraph 提供的一种**序列化共享**优化机制，专为高频更新的数据（如角色移动）设计。核心思想是：

1. 服务器每帧对每个需要更新的 Actor **只序列化一次** 移动数据
2. 将序列化后的 **同一份二进制数据** 发送给所有相关客户端
3. 与传统属性复制独立运行，使用独立的带宽预算

### 7.2 FSharedRepMovement 数据结构

```cpp
// Source/LyraGame/Character/LyraCharacter.h
USTRUCT()
struct FSharedRepMovement
{
    GENERATED_BODY()

    UPROPERTY(Transient)
    FRepMovement RepMovement;          // 位置、旋转、速度（引擎标准格式）

    UPROPERTY(Transient)
    float RepTimeStamp = 0.0f;         // 时间戳

    UPROPERTY(Transient)
    uint8 RepMovementMode = 0;         // 移动模式（行走/飞行/游泳等）

    UPROPERTY(Transient)
    bool bProxyIsJumpForceApplied = false;  // 跳跃力状态

    UPROPERTY(Transient)
    bool bIsCrouched = false;           // 蹲伏状态
};

template<>
struct TStructOpsTypeTraits<FSharedRepMovement> : public TStructOpsTypeTraitsBase2<FSharedRepMovement>
{
    enum
    {
        WithNetSerializer = true,           // 自定义网络序列化
        WithNetSharedSerialization = true,   // 启用共享序列化（关键！）
    };
};
```

**关键标记** `WithNetSharedSerialization = true`：告诉引擎此结构体可以在多个连接间共享同一份序列化缓冲区，避免为每个客户端重复序列化。

### 7.3 自定义 NetSerialize

```cpp
bool FSharedRepMovement::NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess)
{
    bOutSuccess = true;
    RepMovement.NetSerialize(Ar, Map, bOutSuccess);  // 标准 FRepMovement 序列化
    Ar << RepMovementMode;
    Ar << bProxyIsJumpForceApplied;
    Ar << bIsCrouched;

    // 时间戳优化：非零时才序列化
    uint8 bHasTimeStamp = (RepTimeStamp != 0.f);
    Ar.SerializeBits(&bHasTimeStamp, 1);
    if (bHasTimeStamp)
    {
        Ar << RepTimeStamp;
    }
    else
    {
        RepTimeStamp = 0.f;
    }

    return true;
}
```

### 7.4 服务器端：UpdateSharedReplication

在 `InitGlobalActorClassSettings` 中注册的 `FastSharedReplicationFunc` Lambda 每帧被 ReplicationGraph 调用：

```cpp
bool ALyraCharacter::UpdateSharedReplication()
{
    if (GetLocalRole() == ROLE_Authority)
    {
        FSharedRepMovement SharedMovement;
        if (SharedMovement.FillForCharacter(this))
        {
            // 只在数据变化时发送 → 避免冗余序列化
            if (!SharedMovement.Equals(LastSharedReplication, this))
            {
                LastSharedReplication = SharedMovement;
                SetReplicatedMovementMode(SharedMovement.RepMovementMode);
                
                // 发送 NetMulticast RPC（unreliable）
                FastSharedReplication(SharedMovement);
            }
            return true;
        }
    }
    return false;
}
```

**FillForCharacter** — 从角色获取当前移动数据：

```cpp
bool FSharedRepMovement::FillForCharacter(ACharacter* Character)
{
    if (USceneComponent* PawnRootComponent = Character->GetRootComponent())
    {
        UCharacterMovementComponent* CharacterMovement = Character->GetCharacterMovement();

        RepMovement.Location = FRepMovement::RebaseOntoZeroOrigin(
            PawnRootComponent->GetComponentLocation(), Character);
        RepMovement.Rotation = PawnRootComponent->GetComponentRotation();
        RepMovement.LinearVelocity = CharacterMovement->Velocity;
        RepMovementMode = CharacterMovement->PackNetworkMovementMode();
        bProxyIsJumpForceApplied = Character->GetProxyIsJumpForceApplied()
            || (Character->JumpForceTimeRemaining > 0.0f);
        bIsCrouched = Character->IsCrouched();

        // 时间戳：仅在线性平滑模式或显式要求时发送
        if ((CharacterMovement->NetworkSmoothingMode == ENetworkSmoothingMode::Linear)
            || CharacterMovement->bNetworkAlwaysReplicateTransformUpdateTimestamp)
        {
            RepTimeStamp = CharacterMovement->GetServerLastTransformUpdateTimeStamp();
        }
        else
        {
            RepTimeStamp = 0.f;  // 不发送（节省带宽）
        }
        return true;
    }
    return false;
}
```

### 7.5 客户端：接收 FastSharedReplication

```cpp
// NetMulticast, unreliable — 不可靠多播 RPC
UFUNCTION(NetMulticast, unreliable)
void FastSharedReplication(const FSharedRepMovement& SharedRepMovement);

void ALyraCharacter::FastSharedReplication_Implementation(const FSharedRepMovement& SharedRepMovement)
{
    if (GetWorld()->IsPlayingReplay())
        return;

    if (GetLocalRole() == ROLE_SimulatedProxy)
    {
        // 设置时间戳
        SetReplicatedServerLastTransformUpdateTimeStamp(SharedRepMovement.RepTimeStamp);

        // 更新移动模式
        if (GetReplicatedMovementMode() != SharedRepMovement.RepMovementMode)
        {
            SetReplicatedMovementMode(SharedRepMovement.RepMovementMode);
            GetCharacterMovement()->bNetworkMovementModeChanged = true;
            GetCharacterMovement()->bNetworkUpdateReceived = true;
        }

        // 更新位置/旋转/速度
        FRepMovement& MutableRepMovement = GetReplicatedMovement_Mutable();
        MutableRepMovement = SharedRepMovement.RepMovement;
        OnRep_ReplicatedMovement();  // 触发移动平滑

        // 更新跳跃/蹲伏状态
        SetProxyIsJumpForceApplied(SharedRepMovement.bProxyIsJumpForceApplied);
        if (IsCrouched() != SharedRepMovement.bIsCrouched)
        {
            SetIsCrouched(SharedRepMovement.bIsCrouched);
            OnRep_IsCrouched();
        }
    }
}
```

### 7.6 FastShared 带宽控制

```cpp
// 在 InitGlobalActorClassSettings 中设置
FastSharedPathConstants.MaxBitsPerFrame = (int32)(
    (float)(TargetKBytesSecFastSharedPath * 1024 * 8) / NetDriver->GetNetServerMaxTickRate()
);
// 默认：(10 * 1024 * 8) / 30 ≈ 2730 bits/帧 ≈ 341 bytes/帧

FastSharedPathConstants.DistanceRequirementPct = 0.80f;
// 只有在 CullDistance 的 80% 范围内的连接才接收 FastShared 更新
```

---

## 八、压缩加速度复制（FLyraReplicatedAcceleration）

### 8.1 数据结构

```cpp
// Source/LyraGame/Character/LyraCharacter.h
USTRUCT()
struct FLyraReplicatedAcceleration
{
    GENERATED_BODY()

    UPROPERTY()
    uint8 AccelXYRadians = 0;     // XY 方向角，量化为 [0, 2π] → [0, 255]（1 字节）

    UPROPERTY()
    uint8 AccelXYMagnitude = 0;   // XY 幅度，量化为 [0, MaxAccel] → [0, 255]（1 字节）

    UPROPERTY()
    int8 AccelZ = 0;              // Z 轴加速度，量化为 [-MaxAccel, MaxAccel] → [-127, 127]（1 字节）
};
```

**总共只需 3 字节**即可表示完整的 3D 加速度向量（原始 `FVector` 需要 12 字节），压缩率 75%。

### 8.2 服务器端压缩

```cpp
// Source/LyraGame/Character/LyraCharacter.cpp — PreReplication
void ALyraCharacter::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
    // ...
    if (UCharacterMovementComponent* MovementComponent = GetCharacterMovement())
    {
        const double MaxAccel = MovementComponent->MaxAcceleration;
        const FVector CurrentAccel = MovementComponent->GetCurrentAcceleration();

        double AccelXYRadians, AccelXYMagnitude;
        // 将 XY 分量转为极坐标
        FMath::CartesianToPolar(CurrentAccel.X, CurrentAccel.Y, AccelXYMagnitude, AccelXYRadians);

        // 量化压缩
        ReplicatedAcceleration.AccelXYRadians   = FMath::FloorToInt((AccelXYRadians / TWO_PI) * 255.0);
        ReplicatedAcceleration.AccelXYMagnitude = FMath::FloorToInt((AccelXYMagnitude / MaxAccel) * 255.0);
        ReplicatedAcceleration.AccelZ           = FMath::FloorToInt((CurrentAccel.Z / MaxAccel) * 127.0);
    }
}
```

### 8.3 客户端解压缩

```cpp
void ALyraCharacter::OnRep_ReplicatedAcceleration()
{
    if (ULyraCharacterMovementComponent* LyraMovementComponent =
        Cast<ULyraCharacterMovementComponent>(GetCharacterMovement()))
    {
        const double MaxAccel = LyraMovementComponent->MaxAcceleration;

        // 反量化
        const double AccelXYMagnitude = double(ReplicatedAcceleration.AccelXYMagnitude) * MaxAccel / 255.0;
        const double AccelXYRadians   = double(ReplicatedAcceleration.AccelXYRadians) * TWO_PI / 255.0;

        FVector UnpackedAcceleration(FVector::ZeroVector);
        FMath::PolarToCartesian(AccelXYMagnitude, AccelXYRadians,
            UnpackedAcceleration.X, UnpackedAcceleration.Y);
        UnpackedAcceleration.Z = double(ReplicatedAcceleration.AccelZ) * MaxAccel / 127.0;

        // 设置到移动组件
        LyraMovementComponent->SetReplicatedAcceleration(UnpackedAcceleration);
    }
}
```

### 8.4 SimulateMovement 中使用

```cpp
// Source/LyraGame/Character/LyraCharacterMovementComponent.cpp
void ULyraCharacterMovementComponent::SimulateMovement(float DeltaTime)
{
    if (bHasReplicatedAcceleration)
    {
        // 保留我们收到的复制加速度
        const FVector OriginalAcceleration = Acceleration;
        Super::SimulateMovement(DeltaTime);
        // Super 可能会清零加速度，恢复它
        Acceleration = OriginalAcceleration;
    }
    else
    {
        Super::SimulateMovement(DeltaTime);
    }
}
```

---

## 九、CVar 参数系统

Lyra 的 ReplicationGraph 通过大量 CVar 实现运行时调参：

```cpp
namespace Lyra::RepGraph
{
    // 销毁信息最大复制距离（默认 300m）
    float DestructionInfoMaxDist = 30000.f;
    static FAutoConsoleVariableRef CVarDestructMaxDist(
        TEXT("Lyra.RepGraph.DestructInfo.MaxDist"), DestructionInfoMaxDist, TEXT("..."));

    // 空间网格单元大小（默认 100m）
    float CellSize = 10000.f;
    static FAutoConsoleVariableRef CVarCellSize(
        TEXT("Lyra.RepGraph.CellSize"), CellSize, TEXT("..."));

    // 空间偏移（网格起始点）
    float SpatialBiasX = -150000.f;   // -1500m
    float SpatialBiasY = -200000.f;   // -2000m

    // 动态 Actor 频率桶数（默认 3）
    int32 DynamicActorFrequencyBuckets = 3;

    // 是否禁用空间重建（默认是）
    int32 DisableSpatialRebuilds = 1;

    // FastShared 路径目标带宽（默认 10KB/s）
    int32 TargetKBytesSecFastSharedPath = 10;

    // FastShared 距离阈值百分比（默认 80%）
    float FastSharedPathCullDistPct = 0.80f;

    // 是否启用 FastShared（默认启用）
    int32 EnableFastSharedPath = 1;

    // 是否记录延迟初始化的类
    int32 LogLazyInitClasses = 0;

    // 是否显示客户端流式关卡信息
    int32 DisplayClientLevelStreaming = 0;
}
```

### 开发者设置类（ULyraReplicationGraphSettings）

```cpp
UCLASS(config=Game, MinimalAPI)
class ULyraReplicationGraphSettings : public UDeveloperSettingsBackedByCVars
{
    UPROPERTY(config) bool bDisableReplicationGraph = true;       // 是否禁用 RepGraph
    UPROPERTY(config) FSoftClassPath DefaultReplicationGraphClass; // RepGraph 类路径
    UPROPERTY() bool bEnableFastSharedPath = true;                // FastShared 开关
    UPROPERTY() int32 TargetKBytesSecFastSharedPath = 10;         // FastShared 目标带宽
    UPROPERTY() float FastSharedPathCullDistPct = 0.80f;          // FastShared 距离百分比
    UPROPERTY() float DestructionInfoMaxDist = 30000.f;           // 销毁信息最大距离
    UPROPERTY() float SpatialGridCellSize = 10000.0f;             // 网格单元大小
    UPROPERTY() float SpatialBiasX = -200000.0f;                  // 网格起始X
    UPROPERTY() float SpatialBiasY = -200000.0f;                  // 网格起始Y
    UPROPERTY() bool bDisableSpatialRebuilds = true;              // 禁用空间重建
    UPROPERTY() int32 DynamicActorFrequencyBuckets = 3;           // 动态频率桶数
    UPROPERTY(config) TArray<FRepGraphActorClassSettings> ClassSettings; // 类级配置
};
```

---

## 十、GameplayDebugger 集成

Lyra 的 ReplicationGraph 集成了 GameplayDebugger 的网络复制：

```cpp
#if WITH_GAMEPLAY_DEBUGGER
void ULyraReplicationGraph::OnGameplayDebuggerOwnerChange(
    AGameplayDebuggerCategoryReplicator* Debugger, APlayerController* OldOwner)
{
    CHECK_WORLDS(Debugger);

    auto GetAlwaysRelevantForConnectionNode = [this](APlayerController* Controller)
        -> ULyraReplicationGraphNode_AlwaysRelevant_ForConnection*
    {
        if (Controller)
        {
            if (UNetConnection* NetConnection = Controller->GetNetConnection())
            {
                if (NetConnection->GetDriver() == NetDriver)
                {
                    if (UNetReplicationGraphConnection* GraphConnection =
                        FindOrAddConnectionManager(NetConnection))
                    {
                        for (UReplicationGraphNode* ConnectionNode : GraphConnection->GetConnectionGraphNodes())
                        {
                            if (auto* AlwaysRelevantNode = Cast<
                                ULyraReplicationGraphNode_AlwaysRelevant_ForConnection>(ConnectionNode))
                            {
                                return AlwaysRelevantNode;
                            }
                        }
                    }
                }
            }
        }
        return nullptr;
    };

    // 从旧的 Owner 连接节点中移除
    if (auto* OldNode = GetAlwaysRelevantForConnectionNode(OldOwner))
    {
        OldNode->GameplayDebugger = nullptr;
    }

    // 添加到新的 Owner 连接节点
    if (auto* NewNode = GetAlwaysRelevantForConnectionNode(Debugger->GetReplicationOwner()))
    {
        NewNode->GameplayDebugger = Debugger;
    }
}
#endif
```

在 `InitGlobalActorClassSettings` 中注册事件：

```cpp
#if WITH_GAMEPLAY_DEBUGGER
    AGameplayDebuggerCategoryReplicator::NotifyDebuggerOwnerChange.AddUObject(
        this, &ThisClass::OnGameplayDebuggerOwnerChange);
#endif
```

---

## 十一、调试工具

### 11.1 控制台命令

```cpp
// 打印所有类的路由策略
FAutoConsoleCommandWithWorldAndArgs LyraPrintRepNodePoliciesCmd(
    TEXT("Lyra.RepGraph.PrintRouting"),
    TEXT("Prints how actor classes are routed to RepGraph nodes"),
    FConsoleCommandWithWorldAndArgsDelegate::CreateLambda([](const TArray<FString>& Args, UWorld* World)
    {
        for (TObjectIterator<ULyraReplicationGraph> It; It; ++It)
        {
            It->PrintRepNodePolicies();
        }
    })
);

// 修改频率桶数
FAutoConsoleCommandWithWorldAndArgs ChangeFrequencyBucketsCmd(
    TEXT("Lyra.RepGraph.FrequencyBuckets"),
    TEXT("Resets frequency bucket count."),
    // ...
);
```

### 11.2 引擎内置调试命令

```
Net.RepGraph.PrintGraph              — 打印完整图结构
Net.RepGraph.PrintGraph class        — 按类分组打印
Net.RepGraph.PrintGraph nclass       — 按原生类分组打印
Net.RepGraph.PrintAll <帧数> <连接> — 打印完整的收集和优先级信息
Net.RepGraph.PrintAllActorInfo       — 打印 Actor 的复制信息
```

---

## 十二、完整数据流时序图

```
┌───────────┐      ┌──────────────────┐     ┌─────────────┐     ┌──────────────┐
│NetDriver  │      │LyraRepGraph      │     │GridNode     │     │ForConnection │
│(Server)   │      │                  │     │(2D Grid)    │     │Node          │
└─────┬─────┘      └────────┬─────────┘     └──────┬──────┘     └──────┬───────┘
      │                     │                      │                    │
      │ ServerReplicateActors()                    │                    │
      │────────────────────>│                      │                    │
      │                     │                      │                    │
      │                     │ PrepareForReplication │                    │
      │                     │─────────────────────>│                    │
      │                     │                      │ 更新动态Actor位置   │
      │                     │                      │──────┐             │
      │                     │                      │<─────┘             │
      │                     │                      │                    │
      │                     │ For each Connection:  │                    │
      │                     │─────────┐             │                    │
      │                     │         │             │                    │
      │                     │ GatherActorLists      │                    │
      │                     │────────────────────>  │                    │
      │                     │                      │                    │
      │                     │    返回:在连接视野内  │                    │
      │                     │    的空间化Actor列表  │                    │
      │                     │<────────────────────  │                    │
      │                     │                      │                    │
      │                     │ GatherActorLists      │                    │
      │                     │───────────────────────────────────────────>│
      │                     │                      │                    │
      │                     │    返回:PC/Pawn/PS +  │                    │
      │                     │    流式关卡Actor      │                    │
      │                     │<──────────────────────────────────────────│
      │                     │                      │                    │
      │                     │ FastSharedReplication │                    │
      │                     │ (Character移动数据)   │                    │
      │                     │──────┐               │                    │
      │                     │      │ 序列化一次     │                    │
      │                     │      │ 发给所有连接   │                    │
      │                     │<─────┘               │                    │
      │                     │                      │                    │
      │ 合并列表→优先级排序  │                      │                    │
      │ →序列化→发送        │                      │                    │
      │<────────────────────│                      │                    │
```

---

## 十三、性能优化策略总结

| 优化策略 | 实现方式 | 性能收益 |
|----------|----------|----------|
| **空间分区** | `GridSpatialization2D` 将世界分为 100m×100m 网格 | 从 O(N) 降到 O(N/K)，K 为网格数 |
| **持久列表** | Actor 列表在帧间持久化 | 避免每帧重建，降低 GC 压力 |
| **频率桶** | 动态 Actor 分 3 个桶轮询复制 | 每帧只处理 1/3 动态 Actor |
| **PlayerState 频率限制** | 每帧只复制 2 个 PlayerState | 64 人服务器从 64/帧降到 2/帧 |
| **PlayerState 50% 节流** | 自己的 PS 也只在奇/偶帧复制 | 连接级 PS 更新频率再减半 |
| **FastShared 序列化** | 移动数据序列化一次发给所有人 | N 个连接只序列化 1 次（而非 N 次） |
| **差异检测** | `FSharedRepMovement::Equals` | 数据不变时跳过序列化和 RPC |
| **压缩加速度** | 3D 向量 → 3 字节极坐标 | 带宽节省 75%（12B → 3B） |
| **禁用空间重建** | `AddToClassRebuildDenyList` | 避免 Actor 跨网格时的重建开销 |
| **距离裁剪** | `FastSharedPathCullDistPct = 0.80` | FastShared 只发给 80% CullDist 内的连接 |
| **独立带宽预算** | `TargetKBytesSecFastSharedPath = 10KB/s` | FastShared 不占用主通道带宽 |
| **销毁信息距离** | `DestructionInfoMaxDist = 30000cm` | 超远距离不发送销毁通知 |
| **休眠优化** | `bAllDormant` 检查流式关卡 | 全部休眠的关卡跳过复制 |
| **Multicast 通道策略** | Controller 不开通道 | 避免 Multicast RPC 破坏 Owner 复制 |

---

## 十四、与传统复制系统的对比

| 维度 | 传统复制 | ReplicationGraph |
|------|----------|------------------|
| **相关性判断** | `AActor::IsNetRelevantFor()` 每帧 O(N×M) | 图节点预分类 + 空间查询 |
| **Actor 列表** | 每帧从零遍历所有 Actor | 持久化列表，增量更新 |
| **序列化** | 每个连接独立序列化 | FastShared 共享序列化 |
| **PlayerState** | 每帧全部复制 | 频率限制（2/帧/连接） |
| **配置灵活性** | 仅 `NetCullDistanceSquared` | CVar + 配置文件 + 类级路由策略 |
| **扩展性** | 难以优化 | 自定义节点可处理任意逻辑 |
| **调试** | 有限 | 丰富的控制台命令 + 日志 |

---

这套 ReplicationGraph 实现使 Lyra 能够在大量玩家的网络游戏场景下实现高效的 Actor 复制，通过空间分区、频率限制、序列化共享和压缩传输等多层优化，显著降低了服务器的 CPU 和带宽开销。
