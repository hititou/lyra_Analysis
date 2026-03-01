# Lyra Equipment & Inventory（装备与物品系统）深度分析

## 目录

1. [系统总览与架构设计](#1-系统总览与架构设计)
2. [物品定义层 — ULyraInventoryItemDefinition](#2-物品定义层)
3. [Fragment 组合模式](#3-fragment-组合模式)
4. [物品实例 — ULyraInventoryItemInstance](#4-物品实例)
5. [物品统计标签栈 — FGameplayTagStackContainer](#5-物品统计标签栈)
6. [背包管理 — ULyraInventoryManagerComponent](#6-背包管理)
7. [装备定义 — ULyraEquipmentDefinition](#7-装备定义)
8. [装备实例 — ULyraEquipmentInstance](#8-装备实例)
9. [装备管理 — ULyraEquipmentManagerComponent](#9-装备管理)
10. [快捷栏 — ULyraQuickBarComponent](#10-快捷栏)
11. [装备能力 — ULyraGameplayAbility_FromEquipment](#11-装备能力)
12. [拾取系统 — IPickupable 接口](#12-拾取系统)
13. [拾取定义与武器生成器](#13-拾取定义与武器生成器)
14. [世界可收集物 — ALyraWorldCollectable](#14-世界可收集物)
15. [能力消耗集成](#15-能力消耗集成)
16. [武器实例 — ULyraWeaponInstance](#16-武器实例)
17. [网络复制架构](#17-网络复制架构)
18. [完整数据流分析](#18-完整数据流分析)
19. [类关系图谱与扩展指南](#19-类关系图谱与扩展指南)

---

## 1. 系统总览与架构设计

### 1.1 三层分离架构

```
┌──────────────────────────────────────────────────────────┐
│                QuickBar（快捷栏层）                         │
│  ULyraQuickBarComponent (ControllerComponent)            │
│  ─ 管理物品槽位、活跃武器切换、驱动装备/卸载               │
├──────────────────────────────────────────────────────────┤
│                Equipment（装备层）                         │
│  ULyraEquipmentDefinition → ULyraEquipmentInstance       │
│  ULyraEquipmentManagerComponent (PawnComponent)          │
│  ─ 管理装备实体化：Spawn Actor、Grant Ability              │
├──────────────────────────────────────────────────────────┤
│                Inventory（物品层）                         │
│  ULyraInventoryItemDefinition → ULyraInventoryItemInstance│
│  ULyraInventoryManagerComponent (ActorComponent)         │
│  ─ 管理物品持有、堆叠、统计标签栈                          │
└──────────────────────────────────────────────────────────┘
```

### 1.2 核心设计原则

| 原则 | 实现方式 |
|------|----------|
| **Definition/Instance 分离** | Definition 是不可变类模板（CDO），Instance 是运行时可变对象 |
| **Fragment 组合模式** | 物品定义通过组合多个 Fragment 描述能力，无需继承爆炸 |
| **职责分离** | Inventory 管持有，Equipment 管实体化，QuickBar 管切换 |
| **服务端权威** | 所有增删改操作均 `BlueprintAuthorityOnly`，通过 FFastArraySerializer 复制 |
| **GAS 深度集成** | 装备可授予 AbilitySet，物品可充当能力消耗资源 |

### 1.3 文件布局

```
Source/LyraGame/
├── Inventory/                              # 物品系统
│   ├── LyraInventoryItemDefinition.h/cpp   # 物品定义 + Fragment基类
│   ├── LyraInventoryItemInstance.h/cpp     # 物品实例
│   ├── LyraInventoryManagerComponent.h/cpp # 背包管理器
│   ├── IPickupable.h/cpp                   # 拾取接口
│   ├── InventoryFragment_EquippableItem.*  # Fragment: 可装备
│   ├── InventoryFragment_PickupIcon.*      # Fragment: 拾取图标
│   ├── InventoryFragment_QuickBarIcon.*    # Fragment: 快捷栏图标
│   └── InventoryFragment_SetStats.*        # Fragment: 初始统计
├── Equipment/                              # 装备系统
│   ├── LyraEquipmentDefinition.h/cpp       # 装备定义
│   ├── LyraEquipmentInstance.h/cpp         # 装备实例
│   ├── LyraEquipmentManagerComponent.h/cpp # 装备管理器
│   ├── LyraGameplayAbility_FromEquipment.* # 装备关联能力
│   ├── LyraPickupDefinition.h/cpp          # 拾取定义DataAsset
│   └── LyraQuickBarComponent.h/cpp         # 快捷栏
├── Weapons/                                # 武器扩展
│   ├── LyraWeaponInstance.h                # 武器实例(继承装备实例)
│   ├── LyraWeaponSpawner.h/cpp             # 武器生成器
│   └── InventoryFragment_ReticleConfig.*   # Fragment: 准星配置
└── AbilitySystem/Abilities/                # 能力消耗
    ├── LyraAbilityCost_InventoryItem.*     # 消耗物品(预留)
    └── LyraAbilityCost_ItemTagStack.*      # 消耗标签栈(弹药)
```

---

## 2. 物品定义层

### 2.1 ULyraInventoryItemDefinition

> 源码：`Source/LyraGame/Inventory/LyraInventoryItemDefinition.h`

物品定义是**不可变的类蓝图**（`Blueprintable, Const, Abstract`），作为 CDO 使用，所有同类型物品共享同一份定义数据。

```cpp
UCLASS(Blueprintable, Const, Abstract)
class ULyraInventoryItemDefinition : public UObject
{
    GENERATED_BODY()
public:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Display)
    FText DisplayName;

    // Fragment 列表 — 组合模式的核心
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Display, Instanced)
    TArray<TObjectPtr<ULyraInventoryItemFragment>> Fragments;

    const ULyraInventoryItemFragment* FindFragmentByClass(
        TSubclassOf<ULyraInventoryItemFragment> FragmentClass) const;
};
```

**关键设计点：**
- `Const` 标记强制不可变语义
- `Abstract` 标记必须创建蓝图子类才能使用
- 用 `TSubclassOf` 引用定义，运行时通过 `GetDefault<>()` 获取 CDO

FindFragmentByClass 线性查找，支持继承匹配（`IsA`）。物品 Fragment 通常 3-5 个，线性完全够用。

蓝图函数库提供 `FindItemDefinitionFragment`，使用 `DeterminesOutputType` 元数据实现蓝图类型安全查询。

---

## 3. Fragment 组合模式

### 3.1 ULyraInventoryItemFragment 基类

```cpp
UCLASS(MinimalAPI, DefaultToInstanced, EditInlineNew, Abstract)
class ULyraInventoryItemFragment : public UObject
{
    GENERATED_BODY()
public:
    virtual void OnInstanceCreated(ULyraInventoryItemInstance* Instance) const {}
};
```

| 标记 | 作用 |
|------|------|
| `DefaultToInstanced` | 每个定义蓝图持有自己的 Fragment 实例 |
| `EditInlineNew` | 可在属性面板中内联创建和编辑 |
| `OnInstanceCreated` | 物品实例创建时的初始化钩子 |

### 3.2 已有 Fragment 类型

**UInventoryFragment_EquippableItem** — 连接 Inventory 和 Equipment 的桥梁：
```cpp
UCLASS()
class UInventoryFragment_EquippableItem : public ULyraInventoryItemFragment
{
    UPROPERTY(EditAnywhere, Category=Lyra)
    TSubclassOf<ULyraEquipmentDefinition> EquipmentDefinition;
};
```

**UInventoryFragment_SetStats** — 初始化弹药等统计数据：
```cpp
UCLASS()
class UInventoryFragment_SetStats : public ULyraInventoryItemFragment
{
protected:
    UPROPERTY(EditDefaultsOnly, Category=Equipment)
    TMap<FGameplayTag, int32> InitialItemStats;
public:
    virtual void OnInstanceCreated(ULyraInventoryItemInstance* Instance) const override
    {
        for (const auto& KVP : InitialItemStats)
            Instance->AddStatTagStack(KVP.Key, KVP.Value);
    }
};
```

**UInventoryFragment_PickupIcon** — 地面拾取外观（SkeletalMesh + DisplayName + PadColor）

**UInventoryFragment_QuickBarIcon** — 快捷栏 UI（Brush + AmmoBrush + DisplayNameWhenEquipped）

**UInventoryFragment_ReticleConfig** — 武器准星 Widget 类列表

### 3.3 Fragment 组合 vs 传统继承

```
传统继承（不推荐）：UItem → UWeaponItem → URangedWeaponItem → URifleItem

Lyra Fragment 组合方式：
ULyraInventoryItemDefinition（通用物品定义）
  ├── Fragment_EquippableItem   → 可装备
  ├── Fragment_SetStats         → 初始弹药 30 发
  ├── Fragment_QuickBarIcon     → UI 图标
  ├── Fragment_PickupIcon       → 拾取外观
  └── Fragment_ReticleConfig    → 准星样式
```

---

## 4. 物品实例

### 4.1 ULyraInventoryItemInstance

> 源码：`Source/LyraGame/Inventory/LyraInventoryItemInstance.h`

```cpp
UCLASS(BlueprintType)
class ULyraInventoryItemInstance : public UObject
{
    GENERATED_BODY()
public:
    virtual bool IsSupportedForNetworking() const override { return true; }

    void AddStatTagStack(FGameplayTag Tag, int32 StackCount);
    void RemoveStatTagStack(FGameplayTag Tag, int32 StackCount);
    int32 GetStatTagStackCount(FGameplayTag Tag) const;
    bool HasStatTag(FGameplayTag Tag) const;

    TSubclassOf<ULyraInventoryItemDefinition> GetItemDef() const { return ItemDef; }

    const ULyraInventoryItemFragment* FindFragmentByClass(
        TSubclassOf<ULyraInventoryItemFragment> FragmentClass) const;

private:
    UPROPERTY(Replicated)
    FGameplayTagStackContainer StatTags;  // 可复制的统计标签栈

    UPROPERTY(Replicated)
    TSubclassOf<ULyraInventoryItemDefinition> ItemDef;
};
```

**Fragment 查找通过 CDO 代理**：实例不存储 Fragment，从定义类 CDO 动态查找：
```cpp
const ULyraInventoryItemFragment* ULyraInventoryItemInstance::FindFragmentByClass(...) const
{
    if ((ItemDef != nullptr) && (FragmentClass != nullptr))
        return GetDefault<ULyraInventoryItemDefinition>(ItemDef)->FindFragmentByClass(FragmentClass);
    return nullptr;
}
```

这意味着 Fragment 是**定义级**共享数据，实例级可变数据用 `StatTags` 存储。

---

## 5. 物品统计标签栈

### 5.1 FGameplayTagStackContainer

> 源码：`Source/LyraGame/System/GameplayTagStack.h`

通用的、可网络复制的 **GameplayTag → int32** 映射容器，基于 `FFastArraySerializer`。

```cpp
USTRUCT(BlueprintType)
struct FGameplayTagStack : public FFastArraySerializerItem
{
    FGameplayTag Tag;
    int32 StackCount = 0;
};

USTRUCT(BlueprintType)
struct FGameplayTagStackContainer : public FFastArraySerializer
{
    void AddStack(FGameplayTag Tag, int32 StackCount);
    void RemoveStack(FGameplayTag Tag, int32 StackCount);
    int32 GetStackCount(FGameplayTag Tag) const { return TagToCountMap.FindRef(Tag); }
    bool ContainsTag(FGameplayTag Tag) const { return TagToCountMap.Contains(Tag); }

private:
    TArray<FGameplayTagStack> Stacks;        // 复制用数组
    TMap<FGameplayTag, int32> TagToCountMap;  // 查询加速（不复制）
};
```

**双重存储策略：** `Stacks` 数组用于 FFastArraySerializer 增量复制，`TagToCountMap` 用于 O(1) 查询加速。客户端通过 `PostReplicatedAdd/Change/PreReplicatedRemove` 回调自动重建 Map。

**弹药系统典型流程：**
```
定义蓝图 Fragment_SetStats: {"Lyra.ShooterGame.Ammo": 30}
  → OnInstanceCreated → AddStatTagStack → StatTags = {Ammo: 30}
  → 每次射击 → RemoveStatTagStack(Ammo, 1)
  → CheckCost → GetStatTagStackCount(Ammo) >= 1?
```

---

## 6. 背包管理

### 6.1 ULyraInventoryManagerComponent

> 源码：`Source/LyraGame/Inventory/LyraInventoryManagerComponent.h`

```cpp
UCLASS(MinimalAPI, BlueprintType)
class ULyraInventoryManagerComponent : public UActorComponent
{
public:
    bool CanAddItemDefinition(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 StackCount = 1);
    ULyraInventoryItemInstance* AddItemDefinition(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 StackCount = 1);
    void AddItemInstance(ULyraInventoryItemInstance* ItemInstance);
    void RemoveItemInstance(ULyraInventoryItemInstance* ItemInstance);
    TArray<ULyraInventoryItemInstance*> GetAllItems() const;
    ULyraInventoryItemInstance* FindFirstItemStackByDefinition(TSubclassOf<ULyraInventoryItemDefinition> ItemDef) const;
    int32 GetTotalItemCountByDefinition(TSubclassOf<ULyraInventoryItemDefinition> ItemDef) const;
    bool ConsumeItemsByDefinition(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 NumToConsume);

private:
    UPROPERTY(Replicated)
    FLyraInventoryList InventoryList;
};
```

### 6.2 FLyraInventoryList::AddEntry 完整流程

```cpp
ULyraInventoryItemInstance* FLyraInventoryList::AddEntry(
    TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 StackCount)
{
    check(OwnerComponent->GetOwner()->HasAuthority());  // 仅服务端

    FLyraInventoryEntry& NewEntry = Entries.AddDefaulted_GetRef();
    NewEntry.Instance = NewObject<ULyraInventoryItemInstance>(OwnerComponent->GetOwner());
    NewEntry.Instance->SetItemDef(ItemDef);

    // 调用所有 Fragment 的初始化钩子（如设置初始弹药）
    for (ULyraInventoryItemFragment* Fragment : GetDefault<ULyraInventoryItemDefinition>(ItemDef)->Fragments)
    {
        if (Fragment != nullptr)
            Fragment->OnInstanceCreated(NewEntry.Instance);
    }

    NewEntry.StackCount = StackCount;
    MarkItemDirty(NewEntry);  // 触发复制
    return NewEntry.Instance;
}
```

### 6.3 变更消息广播

通过 `UGameplayMessageSubsystem` 广播 `Lyra.Inventory.Message.StackChanged`：

```cpp
struct FLyraInventoryChangeMessage
{
    TObjectPtr<UActorComponent> InventoryOwner;
    TObjectPtr<ULyraInventoryItemInstance> Instance;
    int32 NewCount;
    int32 Delta;  // NewCount - OldCount
};
```

增删改时均触发广播，客户端通过 PostReplicatedAdd/Change/PreReplicatedRemove 回调执行。

---

## 7. 装备定义

### 7.1 ULyraEquipmentDefinition

> 源码：`Source/LyraGame/Equipment/LyraEquipmentDefinition.h`

```cpp
USTRUCT()
struct FLyraEquipmentActorToSpawn
{
    TSubclassOf<AActor> ActorToSpawn;   // 要生成的 Actor 类
    FName AttachSocket;                  // 骨骼插槽名
    FTransform AttachTransform;          // 附着变换
};

UCLASS(Blueprintable, Const, Abstract, BlueprintType)
class ULyraEquipmentDefinition : public UObject
{
    // 装备实例类型（支持多态，如 ULyraWeaponInstance）
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<ULyraEquipmentInstance> InstanceType;

    // 装备时授予的能力集
    UPROPERTY(EditDefaultsOnly)
    TArray<TObjectPtr<const ULyraAbilitySet>> AbilitySetsToGrant;

    // 装备时在 Pawn 上生成的 Actor
    UPROPERTY(EditDefaultsOnly)
    TArray<FLyraEquipmentActorToSpawn> ActorsToSpawn;
};
```

构造函数中 `InstanceType` 默认为 `ULyraEquipmentInstance::StaticClass()`。

---

## 8. 装备实例

### 8.1 ULyraEquipmentInstance

> 源码：`Source/LyraGame/Equipment/LyraEquipmentInstance.h`

```cpp
UCLASS(BlueprintType, Blueprintable)
class ULyraEquipmentInstance : public UObject
{
public:
    virtual bool IsSupportedForNetworking() const override { return true; }

    UObject* GetInstigator() const { return Instigator; }
    void SetInstigator(UObject* InInstigator) { Instigator = InInstigator; }

    APawn* GetPawn() const;  // 通过 GetOuter() 获取
    TArray<AActor*> GetSpawnedActors() const { return SpawnedActors; }

    virtual void SpawnEquipmentActors(const TArray<FLyraEquipmentActorToSpawn>& ActorsToSpawn);
    virtual void DestroyEquipmentActors();
    virtual void OnEquipped();
    virtual void OnUnequipped();

private:
    UPROPERTY(ReplicatedUsing=OnRep_Instigator)
    TObjectPtr<UObject> Instigator;       // 关联的物品实例

    UPROPERTY(Replicated)
    TArray<TObjectPtr<AActor>> SpawnedActors;
};
```

**SpawnEquipmentActors 附着策略：**
- Character → 附着到 `GetMesh()`（骨骼网格体）
- 其他 Pawn → 附着到 `GetRootComponent()`
- 使用 `KeepRelativeTransform` + 指定 `AttachSocket`

**Pawn 获取：** `GetOuter()` 即为 Pawn（创建时 `NewObject<>(OwnerActor)` 设置）。

---

## 9. 装备管理

### 9.1 ULyraEquipmentManagerComponent

> 源码：`Source/LyraGame/Equipment/LyraEquipmentManagerComponent.h`

继承自 `UPawnComponent`，挂载在 **Pawn** 上。

```cpp
UCLASS(MinimalAPI, BlueprintType, Const)
class ULyraEquipmentManagerComponent : public UPawnComponent
{
public:
    ULyraEquipmentInstance* EquipItem(TSubclassOf<ULyraEquipmentDefinition> EquipmentDefinition);
    void UnequipItem(ULyraEquipmentInstance* ItemInstance);
    ULyraEquipmentInstance* GetFirstInstanceOfType(TSubclassOf<ULyraEquipmentInstance> InstanceType);
    TArray<ULyraEquipmentInstance*> GetEquipmentInstancesOfType(TSubclassOf<ULyraEquipmentInstance> InstanceType) const;

private:
    UPROPERTY(Replicated)
    FLyraEquipmentList EquipmentList;
};
```

### 9.2 EquipItem 内部流程

`FLyraEquipmentList::AddEntry` 核心逻辑：

```cpp
// 1. 获取 CDO，确定实例类型
const ULyraEquipmentDefinition* EquipmentCDO = GetDefault<>(EquipmentDefinition);
TSubclassOf<ULyraEquipmentInstance> InstanceType = EquipmentCDO->InstanceType;

// 2. 创建装备实例（Outer = Pawn Actor）
NewEntry.Instance = NewObject<ULyraEquipmentInstance>(OwnerComponent->GetOwner(), InstanceType);

// 3. 通过 ASC 授予能力集
for (const auto& AbilitySet : EquipmentCDO->AbilitySetsToGrant)
    AbilitySet->GiveToAbilitySystem(ASC, &NewEntry.GrantedHandles, Result);

// 4. 生成装备 Actor
Result->SpawnEquipmentActors(EquipmentCDO->ActorsToSpawn);
```

`GrantedHandles`（`FLyraAbilitySet_GrantedHandles`）标记为 `NotReplicated`，因为能力授予只在服务端执行。

### 9.3 UnequipItem 流程

```
1. RemoveReplicatedSubObject(Instance)
2. Instance->OnUnequipped()
3. EquipmentList.RemoveEntry(Instance)
   ├── GrantedHandles.TakeFromAbilitySystem(ASC)  // 回收能力
   ├── Instance->DestroyEquipmentActors()          // 销毁生成的Actor
   └── MarkArrayDirty()                            // 触发复制
```

### 9.4 客户端复制回调

```cpp
// 客户端收到新装备 → Entry.Instance->OnEquipped()
void FLyraEquipmentList::PostReplicatedAdd(const TArrayView<int32> AddedIndices, int32 FinalSize);

// 客户端收到装备移除 → Entry.Instance->OnUnequipped()
void FLyraEquipmentList::PreReplicatedRemove(const TArrayView<int32> RemovedIndices, int32 FinalSize);
```

---

## 10. 快捷栏

### 10.1 ULyraQuickBarComponent

> 源码：`Source/LyraGame/Equipment/LyraQuickBarComponent.h`

**ControllerComponent**，挂载在 PlayerController 上，连接 Inventory 和 Equipment。

```cpp
UCLASS(Blueprintable, meta=(BlueprintSpawnableComponent))
class ULyraQuickBarComponent : public UControllerComponent
{
public:
    void CycleActiveSlotForward();
    void CycleActiveSlotBackward();

    UFUNCTION(Server, Reliable, BlueprintCallable)
    void SetActiveSlotIndex(int32 NewIndex);  // Server RPC

    void AddItemToSlot(int32 SlotIndex, ULyraInventoryItemInstance* Item);
    ULyraInventoryItemInstance* RemoveItemFromSlot(int32 SlotIndex);

protected:
    int32 NumSlots = 3;

private:
    UPROPERTY(ReplicatedUsing=OnRep_Slots)
    TArray<TObjectPtr<ULyraInventoryItemInstance>> Slots;

    UPROPERTY(ReplicatedUsing=OnRep_ActiveSlotIndex)
    int32 ActiveSlotIndex = -1;

    TObjectPtr<ULyraEquipmentInstance> EquippedItem;
};
```

### 10.2 SetActiveSlotIndex — 武器切换核心

```cpp
void ULyraQuickBarComponent::SetActiveSlotIndex_Implementation(int32 NewIndex)
{
    if (Slots.IsValidIndex(NewIndex) && (ActiveSlotIndex != NewIndex))
    {
        UnequipItemInSlot();    // 卸载当前装备
        ActiveSlotIndex = NewIndex;
        EquipItemInSlot();      // 装备新物品
        OnRep_ActiveSlotIndex(); // 广播消息
    }
}
```

### 10.3 EquipItemInSlot — Inventory → Equipment 桥接

```cpp
void ULyraQuickBarComponent::EquipItemInSlot()
{
    if (ULyraInventoryItemInstance* SlotItem = Slots[ActiveSlotIndex])
    {
        // 1. 查找 EquippableItem Fragment → 获取 EquipmentDefinition
        if (const UInventoryFragment_EquippableItem* EquipInfo =
            SlotItem->FindFragmentByClass<UInventoryFragment_EquippableItem>())
        {
            // 2. 通过 EquipmentManager 装备
            if (ULyraEquipmentManagerComponent* EquipmentManager = FindEquipmentManager())
            {
                EquippedItem = EquipmentManager->EquipItem(EquipInfo->EquipmentDefinition);
                if (EquippedItem != nullptr)
                {
                    // 3. 建立反向引用：Equipment → Inventory
                    EquippedItem->SetInstigator(SlotItem);
                }
            }
        }
    }
}
```

**桥接链条：** `ItemInstance → Fragment_EquippableItem → EquipmentDefinition → EquipmentManager.EquipItem() → EquipmentInstance.SetInstigator(ItemInstance)`

### 10.4 消息广播

- `Lyra.QuickBar.Message.SlotsChanged` — 槽位内容变化
- `Lyra.QuickBar.Message.ActiveIndexChanged` — 活跃武器切换

CycleActiveSlotForward/Backward 会智能跳过空槽位。

---

## 11. 装备能力

### 11.1 ULyraGameplayAbility_FromEquipment

```cpp
UCLASS()
class ULyraGameplayAbility_FromEquipment : public ULyraGameplayAbility
{
public:
    ULyraEquipmentInstance* GetAssociatedEquipment() const;
    ULyraInventoryItemInstance* GetAssociatedItem() const;
};
```

**回溯链实现：**
```cpp
// Ability.Spec.SourceObject → EquipmentInstance
ULyraEquipmentInstance* GetAssociatedEquipment() const
{
    return Cast<ULyraEquipmentInstance>(GetCurrentAbilitySpec()->SourceObject.Get());
}

// EquipmentInstance.Instigator → InventoryItemInstance
ULyraInventoryItemInstance* GetAssociatedItem() const
{
    return Cast<ULyraInventoryItemInstance>(GetAssociatedEquipment()->GetInstigator());
}
```

**完整回溯链：** `Ability.Spec.SourceObject → EquipmentInstance → Instigator → InventoryItemInstance → StatTags`

编辑器验证强制装备能力必须是实例化的（NonInstanced 不允许）。

---

## 12. 拾取系统

### 12.1 IPickupable 接口

```cpp
USTRUCT(BlueprintType)
struct FInventoryPickup
{
    TArray<FPickupInstance> Instances;   // 已有实例（保留状态）
    TArray<FPickupTemplate> Templates;   // 模板（创建新实例）
};

class IPickupable
{
public:
    virtual FInventoryPickup GetPickupInventory() const = 0;
};
```

### 12.2 UPickupableStatics

```cpp
// 从 Actor 查找 IPickupable（先查 Actor 本身，再查组件）
static TScriptInterface<IPickupable> GetFirstPickupableFromActor(AActor* Actor);

// 将拾取物添加到背包
static void AddPickupToInventory(ULyraInventoryManagerComponent* Inv, TScriptInterface<IPickupable> Pickup)
{
    // Templates → AddItemDefinition（创建新实例）
    // Instances → AddItemInstance（直接转移）
}
```

| 拾取模式 | 适用场景 | 说明 |
|----------|----------|------|
| Template | 武器生成器重复生成 | 每次创建全新物品实例 |
| Instance | 掉落的已使用物品 | 保留当前状态（如剩余弹药） |

---

## 13. 拾取定义与武器生成器

### 13.1 ULyraPickupDefinition / ULyraWeaponPickupDefinition

DataAsset 配置拾取物的外观和效果：

```cpp
class ULyraPickupDefinition : public UDataAsset
{
    TSubclassOf<ULyraInventoryItemDefinition> InventoryItemDefinition;
    TObjectPtr<UStaticMesh> DisplayMesh;
    int32 SpawnCoolDownSeconds;
    TObjectPtr<USoundBase> PickedUpSound, RespawnedSound;
    TObjectPtr<UNiagaraSystem> PickedUpEffect, RespawnedEffect;
};

class ULyraWeaponPickupDefinition : public ULyraPickupDefinition
{
    FVector WeaponMeshOffset;
    FVector WeaponMeshScale = FVector(1.0f);
};
```

### 13.2 ALyraWeaponSpawner

关卡中放置的武器生成器，管理武器生成、拾取和冷却重生。

```cpp
UCLASS(MinimalAPI, Blueprintable, BlueprintType)
class ALyraWeaponSpawner : public AActor
{
    TObjectPtr<ULyraWeaponPickupDefinition> WeaponDefinition;
    UPROPERTY(ReplicatedUsing = OnRep_WeaponAvailability)
    bool bIsWeaponAvailable;
    float CoolDownTime;  // 默认 30 秒

    void AttemptPickUpWeapon(APawn* Pawn);          // BlueprintNativeEvent
    bool GiveWeapon(TSubclassOf<ULyraInventoryItemDefinition>, APawn*);  // BlueprintImplementableEvent
    void StartCoolDown();
    void ResetCoolDown();
};
```

**拾取流程：** Pawn 重叠 → `AttemptPickUpWeapon` → `GiveWeapon`（蓝图实现） → 设 `bIsWeaponAvailable=false` → 播放特效 → 启动冷却计时 → 冷却结束 → `ResetCoolDown` → 恢复可见 → 检查已重叠的 Pawn

`GetDefaultStatFromItemDef` 静态函数可从 ItemDef 查询默认弹药数等统计，供 UI 使用。

---

## 14. 世界可收集物

### 14.1 ALyraWorldCollectable

> 源码：`Plugins/GameFeatures/ShooterCore`

```cpp
UCLASS(Abstract, Blueprintable)
class ALyraWorldCollectable : public AActor, public IInteractableTarget, public IPickupable
{
protected:
    FInteractionOption Option;
    FInventoryPickup StaticInventory;

public:
    void GatherInteractionOptions(...) override { InteractionBuilder.AddInteractionOption(Option); }
    FInventoryPickup GetPickupInventory() const override { return StaticInventory; }
};
```

双接口设计：`IInteractableTarget`（可被交互系统发现）+ `IPickupable`（可拾取到背包）。

---

## 15. 能力消耗集成

### 15.1 ULyraAbilityCost_ItemTagStack — 弹药消耗

```cpp
// CheckCost: 通过 Ability → Equipment → Item 回溯链检查弹药
bool CheckCost(...) const
{
    ULyraInventoryItemInstance* Item = EquipmentAbility->GetAssociatedItem();
    return Item->GetStatTagStackCount(Tag) >= NumStacks;
}

// ApplyCost: 服务端扣除弹药
void ApplyCost(...)
{
    if (ActorInfo->IsNetAuthority())
        ItemInstance->RemoveStatTagStack(Tag, NumStacks);
}
```

失败时添加 `Ability.ActivateFail.Cost` 标签通知其他系统。

### 15.2 ULyraAbilityCost_InventoryItem — 物品消耗（预留）

`CheckCost` 和 `ApplyCost` 被 `#if 0` 禁用，设计用于消耗整个物品实例（如消耗品），当前未启用。

---

## 16. 武器实例

### 16.1 ULyraWeaponInstance

```cpp
class ULyraWeaponInstance : public ULyraEquipmentInstance
{
    FLyraAnimLayerSelectionSet EquippedAnimSet;
    FLyraAnimLayerSelectionSet UneuippedAnimSet;
    TArray<TObjectPtr<UInputDeviceProperty>> ApplicableDeviceProperties;
    double TimeLastEquipped, TimeLastFired;

    void UpdateFiringTime();
    float GetTimeSinceLastInteractedWith() const;
    TSubclassOf<UAnimInstance> PickBestAnimLayer(bool bEquipped, const FGameplayTagContainer& CosmeticTags) const;
};
```

**继承层次：**
```
ULyraEquipmentInstance          (通用装备)
  └── ULyraWeaponInstance       (武器：动画层、输入设备属性、交互时间)
       └── ULyraRangedWeaponInstance  (远程武器：射击参数)
```

---

## 17. 网络复制架构

### 17.1 复制策略总览

| 组件 | 复制方式 |
|------|----------|
| InventoryManagerComponent | FFastArraySerializer(InventoryList) + SubObject复制(ItemInstance) |
| EquipmentManagerComponent | FFastArraySerializer(EquipmentList) + SubObject复制(EquipInstance) |
| QuickBarComponent | 直接属性复制(Slots + ActiveSlotIndex) + OnRep回调 |
| ItemInstance.StatTags | FFastArraySerializer(GameplayTagStackContainer) |
| WeaponSpawner | OnRep_WeaponAvailability |

### 17.2 服务端权威模式

```
客户端                              服务端
  │  SetActiveSlotIndex(1)           │
  │  ─────(Server RPC)────────────→  │
  │                                  │  UnequipItemInSlot()
  │                                  │  EquipItemInSlot()
  │                                  │    → EquipmentManager.EquipItem()
  │                                  │      → 创建Instance、授予Ability、Spawn Actor
  │  ←──(FastArray Replication)────  │
  │  PostReplicatedAdd()             │
  │    → OnEquipped()                │
```

---

## 18. 完整数据流分析

### 18.1 武器拾取 → 装备 → 射击 → 弹药消耗 全流程

```
阶段 1：Pawn 走入 WeaponSpawner 的 CollisionVolume
  → AttemptPickUpWeapon(Pawn)
  → GiveWeapon(BP_Rifle_ItemDef, Pawn) [蓝图实现]

阶段 2：添加到背包
  InventoryManager->AddItemDefinition(BP_Rifle_ItemDef, 1)
    → 创建 ItemInstance，设置 ItemDef
    → Fragment_SetStats::OnInstanceCreated → AddStatTagStack("Ammo", 30)
    → MarkItemDirty → 复制到客户端
  QuickBar->AddItemToSlot(Slot, ItemInstance)
    → SetActiveSlotIndex(Slot)

阶段 3：装备（SetActiveSlotIndex 内部）
  QuickBar::EquipItemInSlot()
    → FindFragment<EquippableItem>() → EquipmentDefinition
    → EquipmentManager->EquipItem(EquipDef)
      → NewObject<ULyraRangedWeaponInstance>(Pawn)
      → GiveToAbilitySystem(AbilitySets) // 授予射击能力
      → SpawnEquipmentActors() // 生成武器模型
    → EquippedItem->SetInstigator(ItemInstance)

阶段 4：射击
  GA_Weapon_Fire::ActivateAbility()
    → CheckCost → GetAssociatedItem() → GetStatTagStackCount("Ammo") >= 1
    → ApplyCost → RemoveStatTagStack("Ammo", 1) // 30 → 29
    → 执行射击逻辑

阶段 5：弹药耗尽
  CheckCost → GetStatTagStackCount("Ammo") = 0
    → 添加 FailureTag("Ability.ActivateFail.Cost")
    → 能力激活失败
```

### 18.2 组件宿主关系

```
PlayerController
  ├── ULyraQuickBarComponent          (ControllerComponent)
  ├── ULyraInventoryManagerComponent  (ActorComponent, 挂在Controller上)
  │
  └── Possessed Pawn
        ├── ULyraEquipmentManagerComponent  (PawnComponent)
        └── ULyraAbilitySystemComponent
```

> 物品和快捷栏在 Controller 上（Pawn死亡重生时保留），装备在 Pawn 上（Pawn销毁时自动清理）。

---

## 19. 类关系图谱与扩展指南

### 19.1 核心类关系

```
ULyraInventoryItemDefinition (不可变类蓝图)
  └── Fragments[]
        ├── UInventoryFragment_EquippableItem
        │     └── EquipmentDefinition ─────────────┐
        ├── UInventoryFragment_SetStats             │
        ├── UInventoryFragment_PickupIcon           │
        ├── UInventoryFragment_QuickBarIcon         │
        └── UInventoryFragment_ReticleConfig        │
                                                     │
ULyraEquipmentDefinition (不可变类蓝图) ←───────────┘
  ├── InstanceType → ULyraEquipmentInstance
  │                    └── ULyraWeaponInstance
  │                         └── ULyraRangedWeaponInstance
  ├── AbilitySetsToGrant[] → ULyraAbilitySet
  └── ActorsToSpawn[] → FLyraEquipmentActorToSpawn

ULyraInventoryItemInstance (运行时对象)
  ├── ItemDef → ULyraInventoryItemDefinition (class ref)
  └── StatTags → FGameplayTagStackContainer

ULyraEquipmentInstance (运行时对象)
  ├── Instigator → ULyraInventoryItemInstance
  └── SpawnedActors[] → AActor

ULyraPickupDefinition (DataAsset)
  ├── InventoryItemDefinition
  └── ULyraWeaponPickupDefinition (子类)

IPickupable (接口)
  ├── ALyraWorldCollectable (Actor实现)
  └── 可由任何Actor/Component实现
```

### 19.2 扩展指南

**添加新物品类型：**
1. 创建 `ULyraInventoryItemDefinition` 蓝图子类
2. 添加所需 Fragment 组合（EquippableItem、SetStats 等）
3. 如需新 Fragment，继承 `ULyraInventoryItemFragment`

**添加新装备类型：**
1. 创建 `ULyraEquipmentDefinition` 蓝图子类
2. 如需自定义行为，继承 `ULyraEquipmentInstance`
3. 配置 AbilitySetsToGrant 和 ActorsToSpawn

**添加背包容量限制：**
- 扩展 `CanAddItemDefinition`（当前始终返回 true）

**添加物品掉落：**
- 使用 `FPickupInstance` 模式保留物品状态
- 实现 `IPickupable` 接口

**添加新的能力消耗类型：**
- 继承 `ULyraAbilityCost` 并实现 `CheckCost/ApplyCost`
- 可启用已预留的 `ULyraAbilityCost_InventoryItem`
