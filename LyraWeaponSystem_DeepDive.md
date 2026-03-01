# Lyra 武器系统（Weapon System）代码级深度分析

## 目录
- [一、系统架构总览](#一系统架构总览)
- [二、类继承体系与职责划分](#二类继承体系与职责划分)
- [三、核心层：Inventory 物品定义系统](#三核心层inventory-物品定义系统)
- [四、核心层：Equipment 装备系统](#四核心层equipment-装备系统)
- [五、武器实例层：ULyraWeaponInstance](#五武器实例层ulyraweaponinstance)
- [六、远程武器层：ULyraRangedWeaponInstance](#六远程武器层ulyrarangedweaponinstance)
- [七、射击能力：ULyraGameplayAbility_RangedWeapon](#七射击能力ulyragameplayability_rangedweapon)
- [八、武器状态管理：ULyraWeaponStateComponent](#八武器状态管理ulyraweaponstatecomponent)
- [九、快捷栏系统：ULyraQuickBarComponent](#九快捷栏系统ulyraquickbarcomponent)
- [十、伤害计算管线：LyraDamageExecution](#十伤害计算管线ulyradamageexecution)
- [十一、武器拾取系统：ALyraWeaponSpawner](#十一武器拾取系统alyraweaponspawner)
- [十二、武器 UI 系统](#十二武器-ui-系统)
- [十三、辅助瞄准系统（Aim Assist）](#十三辅助瞄准系统aim-assist)
- [十四、调试与诊断工具](#十四调试与诊断工具)
- [十五、网络复制策略总结](#十五网络复制策略总结)
- [十六、完整数据流：从拾取到伤害](#十六完整数据流从拾取到伤害)
- [十七、Lyra 武器系统设计亮点与启发](#十七lyra-武器系统设计亮点与启发)

---

## 一、系统架构总览

Lyra 的武器系统是一个 **多层解耦架构**，从最底层的物品定义到最顶层的伤害输出，每一层都通过清晰的接口衔接：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ★ 完整武器数据流 ★                              │
│                                                                         │
│  ULyraInventoryItemDefinition (物品定义，含 Fragment 插件)               │
│       │                                                                 │
│       ├─ UInventoryFragment_EquippableItem → 指向 EquipmentDefinition   │
│       ├─ UInventoryFragment_SetStats → 初始弹药等 Tag 栈              │
│       ├─ UInventoryFragment_ReticleConfig → 准心 Widget 配置           │
│       ├─ UInventoryFragment_PickupIcon → 拾取图标配置                  │
│       └─ UInventoryFragment_QuickBarIcon → 快捷栏图标                  │
│                                                                         │
│  ULyraEquipmentDefinition (装备定义)                                    │
│       │                                                                 │
│       ├─ InstanceType → ULyraRangedWeaponInstance (运行时实例类型)       │
│       ├─ AbilitySetsToGrant[] → 授予的 GA/GE 集合                      │
│       └─ ActorsToSpawn[] → 武器模型 Actor + 挂载 Socket/Transform       │
│                                                                         │
│  ULyraEquipmentManagerComponent (装备管理器，挂在 Pawn 上)              │
│       │                                                                 │
│       └─ FLyraEquipmentList (FastArraySerializer 网络复制)              │
│              ├─ 创建 EquipmentInstance (Outer = Pawn)                   │
│              ├─ 通过 AbilitySet 授予 GA/GE 到 ASC                      │
│              ├─ 调用 SpawnEquipmentActors() 生成武器模型                │
│              └─ 调用 OnEquipped() / OnUnequipped()                     │
│                                                                         │
│  ULyraQuickBarComponent (快捷栏，挂在 Controller 上)                   │
│       │                                                                 │
│       └─ 管理武器槽位切换、自动 Equip/Unequip                         │
│                                                                         │
│  ULyraWeaponInstance (武器实例基类)                                     │
│       │                                                                 │
│       ├─ AnimLayerSelection (装备/未装备动画层选择)                     │
│       ├─ InputDeviceProperty (手柄振动/触觉反馈)                       │
│       └─ 交互时间追踪 (TimeLastEquipped / TimeLastFired)               │
│                                                                         │
│  ULyraRangedWeaponInstance (远程武器实例)                               │
│       │                                                                 │
│       ├─ 散布系统 (Heat/Spread/CoolDown 曲线)                          │
│       ├─ 散布乘数系统 (站立/蹲伏/跳跃/瞄准 各自平滑过渡)             │
│       ├─ 首发精准 (First Shot Accuracy)                                │
│       ├─ 距离伤害衰减曲线 (DistanceDamageFalloff)                      │
│       ├─ 物理材质伤害乘数 (MaterialDamageMultiplier)                   │
│       └─ ILyraAbilitySourceInterface (供 DamageExecution 查询)         │
│                                                                         │
│  ULyraGameplayAbility_RangedWeapon (射击 GA)                           │
│       │                                                                 │
│       ├─ 射线检测 (Hitscan: Line + Sweep fallback)                     │
│       ├─ 散布方向计算 (VRandConeNormalDistribution)                    │
│       ├─ 多目标来源 (Camera/Pawn/Weapon × Forward/TowardsFocus)        │
│       ├─ TargetData 生成与服务器同步                                    │
│       └─ OnRangedWeaponTargetDataReady (蓝图处理命中效果)              │
│                                                                         │
│  ULyraWeaponStateComponent (武器状态，挂在 Controller 上)              │
│       │                                                                 │
│       ├─ 每帧 Tick 远程武器实例 (更新散布)                             │
│       ├─ 命中标记 (Hit Marker) 管理                                    │
│       └─ 服务端确认 → 客户端显示确认打击效果                           │
│                                                                         │
│  ULyraDamageExecution (GE 伤害计算)                                    │
│       │                                                                 │
│       ├─ BaseDamage × DistanceAttenuation × PhysicalMaterialAttenuation│
│       ├─ TeamSubsystem 友伤检查                                        │
│       └─ 写入 HealthSet.Damage 属性                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

**核心源文件分布**：

| 目录 | 文件数 | 职责 |
|------|:---:|------|
| `Source/LyraGame/Weapons/` | 16 | 武器核心逻辑：实例、GA、状态、拾取、调试 |
| `Source/LyraGame/Equipment/` | 12 | 装备定义、装备管理器、快捷栏 |
| `Source/LyraGame/Inventory/` | 14 | 物品定义、物品实例、Fragment 插件 |
| `Source/LyraGame/UI/Weapons/` | 12 | 准心、命中标记、散布可视化 |
| `Source/LyraGame/AbilitySystem/Executions/` | 4 | 伤害/治疗计算 |
| `Plugins/GameFeatures/ShooterCore/` | 29+ | 辅助瞄准、击杀处理器、Accolade |

---

## 二、类继承体系与职责划分

### 2.1 武器实例继承链

```
UObject
 └─ ULyraEquipmentInstance              // 装备实例基类
     └─ ULyraWeaponInstance             // 武器实例（动画层选择 + 设备属性）
         └─ ULyraRangedWeaponInstance   // 远程武器（散布 + 伤害衰减 + AbilitySource）
```

### 2.2 武器能力继承链

```
UGameplayAbility
 └─ ULyraGameplayAbility               // Lyra GA 基类
     └─ ULyraGameplayAbility_FromEquipment  // 从装备获取 GA（反查 Equipment + Item）
         └─ ULyraGameplayAbility_RangedWeapon  // 远程武器射击 GA（射线检测 + TargetData）
```

### 2.3 关键组件挂载关系

```
Pawn (角色)
 ├─ ULyraEquipmentManagerComponent    // 管理所有已装备的装备实例
 └─ ULyraInventoryManagerComponent    // 管理背包中所有物品实例（可选挂载位置）

Controller (控制器)
 ├─ ULyraQuickBarComponent            // 快捷栏（武器槽位切换）
 └─ ULyraWeaponStateComponent         // 武器状态（Tick 武器散布 + 命中标记）
```

> **设计要点**：快捷栏和武器状态挂在 **Controller** 而非 Pawn 上，因为这是玩家意图（槽位选择）和感知（命中反馈）的一部分，这样 Pawn 被替换时（如载具系统）不会丢失这些状态。

---

## 三、核心层：Inventory 物品定义系统

### 3.1 ULyraInventoryItemDefinition —— Fragment 插件架构

```cpp
// Source/LyraGame/Inventory/LyraInventoryItemDefinition.h
UCLASS(Blueprintable, Const, Abstract)
class ULyraInventoryItemDefinition : public UObject
{
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Display)
    FText DisplayName;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Display, Instanced)
    TArray<TObjectPtr<ULyraInventoryItemFragment>> Fragments;

    const ULyraInventoryItemFragment* FindFragmentByClass(TSubclassOf<ULyraInventoryItemFragment> FragmentClass) const;
};
```

**核心设计**：物品定义不使用传统的继承体系，而是通过 **Fragment（碎片）** 组合模式来扩展功能。每种武器的 `ItemDefinition` 蓝图子类中可自由组合以下 Fragment：

| Fragment 类 | 功能 |
|-------------|------|
| `UInventoryFragment_EquippableItem` | 指向 `ULyraEquipmentDefinition`，使物品可装备 |
| `UInventoryFragment_SetStats` | 通过 `TMap<FGameplayTag, int32>` 设置初始统计值（如弹药数） |
| `UInventoryFragment_ReticleConfig` | 配置准心 Widget 类列表 |
| `UInventoryFragment_PickupIcon` | 配置拾取时显示的图标 |
| `UInventoryFragment_QuickBarIcon` | 配置快捷栏中的图标 |

### 3.2 ULyraInventoryItemInstance —— GameplayTag 栈驱动的物品状态

```cpp
// Source/LyraGame/Inventory/LyraInventoryItemInstance.h
UCLASS(BlueprintType)
class ULyraInventoryItemInstance : public UObject
{
    UPROPERTY(Replicated)
    FGameplayTagStackContainer StatTags;   // Tag 栈容器（如弹药计数）

    UPROPERTY(Replicated)
    TSubclassOf<ULyraInventoryItemDefinition> ItemDef;
};
```

**特色**：武器的弹药、耐久等数值不是直接的 int/float 属性，而是存储在 `FGameplayTagStackContainer` 中。例如步枪的弹药可能是 `Lyra.ShooterGame.Weapon.MagazineAmmo` Tag 的栈计数。这使得弹药系统完全数据驱动，无需任何 C++ 代码修改即可添加新的武器统计项。

### 3.3 UInventoryFragment_SetStats —— 物品初始化时注入统计值

```cpp
// Source/LyraGame/Inventory/InventoryFragment_SetStats.cpp
void UInventoryFragment_SetStats::OnInstanceCreated(ULyraInventoryItemInstance* Instance) const
{
    for (const auto& KVP : InitialItemStats)
    {
        Instance->AddStatTagStack(KVP.Key, KVP.Value);
    }
}
```

物品实例创建时自动调用 `OnInstanceCreated`，把编辑器中配置的初始统计值写入物品实例，实现了 **定义（Definition）与实例（Instance）的分离**。

---

## 四、核心层：Equipment 装备系统

### 4.1 ULyraEquipmentDefinition —— 装备蓝图

```cpp
// Source/LyraGame/Equipment/LyraEquipmentDefinition.h
UCLASS(Blueprintable, Const, Abstract, BlueprintType)
class ULyraEquipmentDefinition : public UObject
{
    // 运行时要创建的实例类型（默认 ULyraEquipmentInstance，武器覆写为 ULyraRangedWeaponInstance）
    UPROPERTY(EditDefaultsOnly, Category=Equipment)
    TSubclassOf<ULyraEquipmentInstance> InstanceType;

    // 装备时授予的 Ability Set（包含 GA + GE + AttributeSet）
    UPROPERTY(EditDefaultsOnly, Category=Equipment)
    TArray<TObjectPtr<const ULyraAbilitySet>> AbilitySetsToGrant;

    // 装备时要 Spawn 的 Actor（武器模型）
    UPROPERTY(EditDefaultsOnly, Category=Equipment)
    TArray<FLyraEquipmentActorToSpawn> ActorsToSpawn;
};
```

每个 `FLyraEquipmentActorToSpawn` 包含：
- `ActorToSpawn`：要生成的 Actor 类（武器模型蓝图）
- `AttachSocket`：挂载骨骼插槽名（如 `weapon_r`）
- `AttachTransform`：挂载偏移变换

### 4.2 ULyraEquipmentInstance —— 装备运行时实例

```cpp
// Source/LyraGame/Equipment/LyraEquipmentInstance.cpp
void ULyraEquipmentInstance::SpawnEquipmentActors(const TArray<FLyraEquipmentActorToSpawn>& ActorsToSpawn)
{
    if (APawn* OwningPawn = GetPawn())
    {
        USceneComponent* AttachTarget = OwningPawn->GetRootComponent();
        if (ACharacter* Char = Cast<ACharacter>(OwningPawn))
        {
            AttachTarget = Char->GetMesh();  // 挂到骨骼网格上
        }

        for (const FLyraEquipmentActorToSpawn& SpawnInfo : ActorsToSpawn)
        {
            AActor* NewActor = GetWorld()->SpawnActorDeferred<AActor>(SpawnInfo.ActorToSpawn, FTransform::Identity, OwningPawn);
            NewActor->FinishSpawning(FTransform::Identity, true);
            NewActor->SetActorRelativeTransform(SpawnInfo.AttachTransform);
            NewActor->AttachToComponent(AttachTarget, FAttachmentTransformRules::KeepRelativeTransform, SpawnInfo.AttachSocket);
            SpawnedActors.Add(NewActor);
        }
    }
}
```

**关键细节**：
- `GetPawn()` 通过 `GetOuter()` 实现，因为装备实例的 Outer 就是 Pawn
- `Instigator` 和 `SpawnedActors` 都标记了 `UPROPERTY(Replicated)`，确保多人游戏同步
- 使用 `SpawnActorDeferred` + `FinishSpawning` 确保在附加前完成初始化

### 4.3 ULyraEquipmentManagerComponent —— FastArraySerializer 复制

```cpp
// Source/LyraGame/Equipment/LyraEquipmentManagerComponent.cpp
ULyraEquipmentInstance* FLyraEquipmentList::AddEntry(TSubclassOf<ULyraEquipmentDefinition> EquipmentDefinition)
{
    const ULyraEquipmentDefinition* EquipmentCDO = GetDefault<ULyraEquipmentDefinition>(EquipmentDefinition);
    TSubclassOf<ULyraEquipmentInstance> InstanceType = EquipmentCDO->InstanceType;

    FLyraAppliedEquipmentEntry& NewEntry = Entries.AddDefaulted_GetRef();
    NewEntry.EquipmentDefinition = EquipmentDefinition;
    NewEntry.Instance = NewObject<ULyraEquipmentInstance>(OwnerComponent->GetOwner(), InstanceType);

    // 通过 AbilitySet 授予 GA/GE
    if (ULyraAbilitySystemComponent* ASC = GetAbilitySystemComponent())
    {
        for (const TObjectPtr<const ULyraAbilitySet>& AbilitySet : EquipmentCDO->AbilitySetsToGrant)
        {
            AbilitySet->GiveToAbilitySystem(ASC, &NewEntry.GrantedHandles, Result);
        }
    }

    // 生成武器模型 Actor
    Result->SpawnEquipmentActors(EquipmentCDO->ActorsToSpawn);
    MarkItemDirty(NewEntry);  // 标记脏，触发 FastArray 网络复制
    return Result;
}
```

装备列表使用 `FFastArraySerializer`（与 GAS 的 AttributeSet 类似的高效增量复制）。每个 `FLyraAppliedEquipmentEntry` 包含：

```cpp
USTRUCT()
struct FLyraAppliedEquipmentEntry : public FFastArraySerializerItem
{
    TSubclassOf<ULyraEquipmentDefinition> EquipmentDefinition;
    TObjectPtr<ULyraEquipmentInstance> Instance;

    UPROPERTY(NotReplicated)
    FLyraAbilitySet_GrantedHandles GrantedHandles;  // 仅服务端维护
};
```

`GrantedHandles` 标记为 `NotReplicated`，因为 GA/GE 的授予由 GAS 自身的复制系统处理。

**客户端复制回调**：
- `PostReplicatedAdd` → 调用 `Entry.Instance->OnEquipped()`
- `PreReplicatedRemove` → 调用 `Entry.Instance->OnUnequipped()`

---

## 五、武器实例层：ULyraWeaponInstance

```cpp
// Source/LyraGame/Weapons/LyraWeaponInstance.h
UCLASS(MinimalAPI)
class ULyraWeaponInstance : public ULyraEquipmentInstance
{
protected:
    // 装备状态的动画层选择集
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Animation)
    FLyraAnimLayerSelectionSet EquippedAnimSet;

    // 未装备状态的动画层选择集
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Animation)
    FLyraAnimLayerSelectionSet UneuippedAnimSet;

    // 手柄触觉反馈设备属性
    UPROPERTY(EditDefaultsOnly, Instanced, BlueprintReadOnly, Category="Input Devices")
    TArray<TObjectPtr<UInputDeviceProperty>> ApplicableDeviceProperties;

private:
    TSet<FInputDevicePropertyHandle> DevicePropertyHandles;
    double TimeLastEquipped = 0.0;
    double TimeLastFired = 0.0;
};
```

### 5.1 动画层选择

```cpp
TSubclassOf<UAnimInstance> ULyraWeaponInstance::PickBestAnimLayer(bool bEquipped, const FGameplayTagContainer& CosmeticTags) const
{
    const FLyraAnimLayerSelectionSet& SetToQuery = (bEquipped ? EquippedAnimSet : UneuippedAnimSet);
    return SetToQuery.SelectBestLayer(CosmeticTags);
}
```

每把武器蓝图中配置两套 `FLyraAnimLayerSelectionSet`（装备/未装备），根据角色外观 Tag（体型/性别等）自动选择最匹配的动画层蓝图。蓝图端调用 `LinkAnimClassLayers()` 切换动画。

### 5.2 设备属性（手柄振动）

```cpp
void ULyraWeaponInstance::ApplyDeviceProperties()
{
    const FPlatformUserId UserId = GetOwningUserId();
    if (UserId.IsValid())
    {
        if (UInputDeviceSubsystem* InputDeviceSubsystem = UInputDeviceSubsystem::Get())
        {
            for (TObjectPtr<UInputDeviceProperty>& DeviceProp : ApplicableDeviceProperties)
            {
                FActivateDevicePropertyParams Params = {};
                Params.UserId = UserId;
                Params.bLooping = true;  // 循环播放，直到武器卸下
                DevicePropertyHandles.Emplace(InputDeviceSubsystem->ActivateDeviceProperty(DeviceProp, Params));
            }
        }
    }
}
```

**亮点**：
- 使用 UE5 的 `UInputDeviceSubsystem` API，支持 PS5 DualSense 等下一代手柄
- 设备属性以 `Looping` 模式激活，持续到 `OnUnequipped` 或角色死亡时移除
- 构造函数中监听 `HealthComponent->OnDeathStarted`，确保死亡时也清理振动效果

### 5.3 时间追踪

```cpp
float ULyraWeaponInstance::GetTimeSinceLastInteractedWith() const
{
    const double WorldTime = World->GetTimeSeconds();
    double Result = WorldTime - TimeLastEquipped;
    if (TimeLastFired > 0.0)
    {
        const double TimeSinceFired = WorldTime - TimeLastFired;
        Result = FMath::Min(Result, TimeSinceFired);
    }
    return Result;
}
```

返回距离最后一次交互（装备或开火）的时间，可用于"武器闲置放下"等动画逻辑。

---

## 六、远程武器层：ULyraRangedWeaponInstance

这是 Lyra 武器系统最核心、最复杂的类。

### 6.1 热度-散布曲线系统

```cpp
// 热度 → 散布角度的映射曲线（定义了武器的最小/最大散布范围）
UPROPERTY(EditAnywhere, Category="Spread|Fire Params")
FRuntimeFloatCurve HeatToSpreadCurve;

// 热度 → 每发增加的热度曲线（可实现过热惩罚机制）
UPROPERTY(EditAnywhere, Category="Spread|Fire Params")
FRuntimeFloatCurve HeatToHeatPerShotCurve;

// 热度 → 每秒冷却速率曲线（可实现过热后冷却减缓）
UPROPERTY(EditAnywhere, Category="Spread|Fire Params")
FRuntimeFloatCurve HeatToCoolDownPerSecondCurve;

// 停火后多久才开始冷却
UPROPERTY(EditAnywhere, Category="Spread|Fire Params")
float SpreadRecoveryCooldownDelay = 0.0f;

// 是否允许首发精准
UPROPERTY(EditAnywhere, Category="Spread|Fire Params")
bool bAllowFirstShotAccuracy = false;
```

**热度更新流程**（每帧在 `Tick` 中调用）：

```
Tick(DeltaSeconds)
 ├─ UpdateSpread(DeltaSeconds)
 │    ├─ if (停火时间 > SpreadRecoveryCooldownDelay)
 │    │    ├─ CooldownRate = HeatToCoolDownPerSecondCurve.Eval(CurrentHeat)
 │    │    ├─ CurrentHeat -= CooldownRate * DeltaSeconds
 │    │    └─ CurrentSpreadAngle = HeatToSpreadCurve.Eval(CurrentHeat)
 │    └─ return 散布是否已回到最小值
 │
 ├─ UpdateMultipliers(DeltaSeconds)
 │    ├─ 站立静止乘数 (速度 < 阈值时缩小散布)
 │    ├─ 蹲伏乘数
 │    ├─ 跳跃/坠落乘数
 │    ├─ 瞄准乘数 (通过 CameraComponent 的 BlendInfo 检测)
 │    └─ CombinedMultiplier = Aiming × StandingStill × Crouching × JumpFall
 │
 └─ bHasFirstShotAccuracy = bAllowFirstShotAccuracy && bMinMultipliers && bMinSpread
```

### 6.2 散布乘数系统 —— 平滑过渡

```cpp
bool ULyraRangedWeaponInstance::UpdateMultipliers(float DeltaSeconds)
{
    // 站立静止：速度在 [StandingStillSpeedThreshold, Threshold+Range] 间平滑映射
    const float MovementTargetValue = FMath::GetMappedRangeValueClamped(
        FVector2D(StandingStillSpeedThreshold, StandingStillSpeedThreshold + StandingStillToMovingSpeedRange),
        FVector2D(SpreadAngleMultiplier_StandingStill, 1.0f),
        PawnSpeed);
    StandingStillMultiplier = FMath::FInterpTo(StandingStillMultiplier, MovementTargetValue, DeltaSeconds, TransitionRate_StandingStill);

    // 蹲伏：FInterpTo 平滑过渡
    CrouchingMultiplier = FMath::FInterpTo(CrouchingMultiplier, CrouchingTargetValue, DeltaSeconds, TransitionRate_Crouching);

    // 跳跃/坠落
    JumpFallMultiplier = FMath::FInterpTo(JumpFallMultiplier, JumpFallTargetValue, DeltaSeconds, TransitionRate_JumpingOrFalling);

    // 瞄准：通过相机系统的 BlendInfo 获取瞄准权重
    if (const ULyraCameraComponent* CameraComponent = ULyraCameraComponent::FindCameraComponent(Pawn))
    {
        float TopCameraWeight;
        FGameplayTag TopCameraTag;
        CameraComponent->GetBlendInfo(TopCameraWeight, TopCameraTag);
        AimingAlpha = (TopCameraTag == TAG_Lyra_Weapon_SteadyAimingCamera) ? TopCameraWeight : 0.0f;
    }

    // 最终合并
    CurrentSpreadAngleMultiplier = AimingMultiplier * StandingStillMultiplier * CrouchingMultiplier * JumpFallMultiplier;
}
```

**设计精华**：
- 所有乘数使用 `FInterpTo` 做指数插值而非瞬间切换，带来丝滑的射击手感
- 瞄准乘数不是硬编码检查"是否在瞄准"，而是通过 **相机系统的 BlendInfo**（当前最高权重的相机 Tag 及其混合权重）来判断，完全解耦
- 配置表参数极为丰富：每个乘数都有独立的 `TransitionRate`，允许设计师分别调整蹲下/起身、起跳/落地时散布变化的速度

### 6.3 ILyraAbilitySourceInterface —— 伤害衰减接口

```cpp
class ULyraRangedWeaponInstance : public ULyraWeaponInstance, public ILyraAbilitySourceInterface
{
    // 距离衰减（查询 DistanceDamageFalloff 曲线）
    virtual float GetDistanceAttenuation(float Distance, ...) const override;

    // 物理材质衰减（查询 MaterialDamageMultiplier Tag 映射表）
    virtual float GetPhysicalMaterialAttenuation(const UPhysicalMaterial* PhysicalMaterial, ...) const override;
};
```

```cpp
float ULyraRangedWeaponInstance::GetPhysicalMaterialAttenuation(const UPhysicalMaterial* PhysicalMaterial, ...) const
{
    float CombinedMultiplier = 1.0f;
    if (const UPhysicalMaterialWithTags* PhysMatWithTags = Cast<const UPhysicalMaterialWithTags>(PhysicalMaterial))
    {
        for (const FGameplayTag MaterialTag : PhysMatWithTags->Tags)
        {
            if (const float* pTagMultiplier = MaterialDamageMultiplier.Find(MaterialTag))
            {
                CombinedMultiplier *= *pTagMultiplier;
            }
        }
    }
    return CombinedMultiplier;
}
```

**Tag 驱动的物理材质伤害**：
- `UPhysicalMaterialWithTags` 扩展了 UE 的 `UPhysicalMaterial`，添加了 `FGameplayTagContainer Tags`
- 武器蓝图中的 `MaterialDamageMultiplier` 映射 `FGameplayTag → float`
- 命中时检查被击中表面的物理材质 Tag，累乘所有匹配的伤害乘数
- 例如：`Gameplay.Zone.Head → 2.0`（爆头2倍伤害），`Gameplay.Zone.WeakSpot → 1.5`

### 6.4 弹药配置

```cpp
// 每次发射的弹丸数（步枪=1，霰弹枪>1）
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="Weapon Config")
int32 BulletsPerCartridge = 1;

// 最大伤害距离
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="Weapon Config")
float MaxDamageRange = 25000.0f;

// 子弹射线的扫描球半径（0=精确线检测）
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="Weapon Config")
float BulletTraceSweepRadius = 0.0f;
```

---

## 七、射击能力：ULyraGameplayAbility_RangedWeapon

这是整个射击流程的核心 GA，约 600 行代码，包含完整的射线检测、散布计算、多目标来源支持和网络同步。

### 7.1 目标来源枚举

```cpp
UENUM(BlueprintType)
enum class ELyraAbilityTargetingSource : uint8
{
    CameraTowardsFocus,    // 从相机向焦点 (最常用，FPS/TPS 标准模式)
    PawnForward,           // 从 Pawn 中心向前
    PawnTowardsFocus,      // 从 Pawn 中心向焦点
    WeaponForward,         // 从武器枪口向前
    WeaponTowardsFocus,    // 从武器枪口向焦点
    Custom                 // 蓝图自定义
};
```

### 7.2 激活流程

```cpp
void ULyraGameplayAbility_RangedWeapon::ActivateAbility(...)
{
    UAbilitySystemComponent* MyAbilityComponent = CurrentActorInfo->AbilitySystemComponent.Get();

    // 1. 绑定 TargetData 回调
    OnTargetDataReadyCallbackDelegateHandle = MyAbilityComponent->AbilityTargetDataSetDelegate(
        CurrentSpecHandle, CurrentActivationInfo.GetActivationPredictionKey()
    ).AddUObject(this, &ThisClass::OnTargetDataReadyCallback);

    // 2. 更新武器最后开火时间
    ULyraRangedWeaponInstance* WeaponData = GetWeaponInstance();
    WeaponData->UpdateFiringTime();

    // 3. 调用父类（最终到蓝图的 ActivateAbility）
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);
}
```

蓝图中的典型流程：`ActivateAbility → PlayMontageAndWait → StartRangedWeaponTargeting → 等待 OnRangedWeaponTargetDataReady → ApplyGameplayEffect → EndAbility`

### 7.3 射线检测核心

**`StartRangedWeaponTargeting()` → `PerformLocalTargeting()` → `TraceBulletsInCartridge()`**

```cpp
void ULyraGameplayAbility_RangedWeapon::TraceBulletsInCartridge(const FRangedWeaponFiringInput& InputData, TArray<FHitResult>& OutHits)
{
    ULyraRangedWeaponInstance* WeaponData = InputData.WeaponData;
    const int32 BulletsPerCartridge = WeaponData->GetBulletsPerCartridge();

    for (int32 BulletIndex = 0; BulletIndex < BulletsPerCartridge; ++BulletIndex)
    {
        // 获取当前散布
        const float BaseSpreadAngle = WeaponData->GetCalculatedSpreadAngle();
        const float SpreadAngleMultiplier = WeaponData->GetCalculatedSpreadAngleMultiplier();
        const float ActualSpreadAngle = BaseSpreadAngle * SpreadAngleMultiplier;
        const float HalfSpreadAngleInRadians = FMath::DegreesToRadians(ActualSpreadAngle * 0.5f);

        // 在锥形范围内按正态分布生成随机方向
        const FVector BulletDir = VRandConeNormalDistribution(InputData.AimDir, HalfSpreadAngleInRadians, WeaponData->GetSpreadExponent());

        // 执行单发子弹检测
        FHitResult Impact = DoSingleBulletTrace(InputData.StartTrace, EndTrace, WeaponData->GetBulletTraceSweepRadius(), false, AllImpacts);
        // ...
    }
}
```

### 7.4 VRandConeNormalDistribution —— 自定义散布分布

```cpp
FVector VRandConeNormalDistribution(const FVector& Dir, const float ConeHalfAngleRad, const float Exponent)
{
    if (ConeHalfAngleRad > 0.f)
    {
        // 使用指数分布而非均匀分布
        const float FromCenter = FMath::Pow(FMath::FRand(), Exponent);
        const float AngleFromCenter = FromCenter * ConeHalfAngleDegrees;
        const float AngleAround = FMath::FRand() * 360.0f;

        FRotator Rot = Dir.Rotation();
        FQuat DirQuat(Rot);
        FQuat FromCenterQuat(FRotator(0.0f, AngleFromCenter, 0.0f));
        FQuat AroundQuat(FRotator(0.0f, 0.0, AngleAround));
        FQuat FinalDirectionQuat = DirQuat * AroundQuat * FromCenterQuat;
        FinalDirectionQuat.Normalize();
        return FinalDirectionQuat.RotateVector(FVector::ForwardVector);
    }
    return Dir.GetSafeNormal();
}
```

**关键**：`SpreadExponent` 参数控制弹丸聚集度：
- `Exponent = 1.0`：均匀分布在锥体内
- `Exponent > 1.0`：弹丸更集中在中心（如步枪）
- `Exponent < 1.0`：弹丸更分散（如霰弹枪）

### 7.5 双层射线检测策略

```cpp
FHitResult ULyraGameplayAbility_RangedWeapon::DoSingleBulletTrace(
    const FVector& StartTrace, const FVector& EndTrace, float SweepRadius, bool bIsSimulated, TArray<FHitResult>& OutHits) const
{
    // 第一步：先尝试精确线检测（LineTrace）
    if (FindFirstPawnHitResult(OutHits) == INDEX_NONE)
    {
        Impact = WeaponTrace(StartTrace, EndTrace, 0.0f, bIsSimulated, OutHits);
    }

    // 第二步：如果线检测没命中 Pawn 且武器有扫描半径，尝试球形扫描（Sweep）
    if (FindFirstPawnHitResult(OutHits) == INDEX_NONE)
    {
        if (SweepRadius > 0.0f)
        {
            TArray<FHitResult> SweepHits;
            Impact = WeaponTrace(StartTrace, EndTrace, SweepRadius, bIsSimulated, SweepHits);

            // 仅在球形扫描命中 Pawn 且该 Pawn 前面没有阻挡物时才使用扫描结果
            const int32 FirstPawnIdx = FindFirstPawnHitResult(SweepHits);
            if (SweepHits.IsValidIndex(FirstPawnIdx))
            {
                bool bUseSweepHits = true;
                for (int32 Idx = 0; Idx < FirstPawnIdx; ++Idx)
                {
                    if (CurHitResult.bBlockingHit && OutHits.ContainsByPredicate(Pred))
                    {
                        bUseSweepHits = false;  // 线检测已经有阻挡物了
                        break;
                    }
                }
                if (bUseSweepHits) OutHits = SweepHits;
            }
        }
    }
}
```

这个 **Line → Sweep 回退策略** 非常巧妙：
1. 先做精确线检测，保证射击精度
2. 如果线检测没命中人，用更宽的球形扫描"容错"，提升手感
3. 但球形扫描的结果不能穿过线检测已经命中的阻挡物，避免"穿墙"

### 7.6 WeaponTrace —— 去重过滤

```cpp
FHitResult ULyraGameplayAbility_RangedWeapon::WeaponTrace(...) const
{
    FCollisionQueryParams TraceParams(SCENE_QUERY_STAT(WeaponTrace), true, GetAvatarActorFromActorInfo());
    TraceParams.bReturnPhysicalMaterial = true;  // 需要物理材质用于伤害计算

    const ECollisionChannel TraceChannel = DetermineTraceChannel(TraceParams, bIsSimulated);
    // → 返回 Lyra_TraceChannel_Weapon (ECC_GameTraceChannel2)

    // 执行多目标检测
    if (SweepRadius > 0.0f)
        GetWorld()->SweepMultiByChannel(HitResults, StartTrace, EndTrace, FQuat::Identity, TraceChannel, FCollisionShape::MakeSphere(SweepRadius), TraceParams);
    else
        GetWorld()->LineTraceMultiByChannel(HitResults, StartTrace, EndTrace, TraceChannel, TraceParams);

    // 去重：同一 Actor 只保留首次命中
    for (FHitResult& CurHitResult : HitResults)
    {
        auto Pred = [&CurHitResult](const FHitResult& Other) { return Other.HitObjectHandle == CurHitResult.HitObjectHandle; };
        if (!OutHitResults.ContainsByPredicate(Pred))
            OutHitResults.Add(CurHitResult);
    }
}
```

### 7.7 目标来源计算 —— GetTargetingTransform

```cpp
FTransform ULyraGameplayAbility_RangedWeapon::GetTargetingTransform(APawn* SourcePawn, ELyraAbilityTargetingSource Source) const
{
    // 对于 CameraTowardsFocus / PawnTowardsFocus / WeaponTowardsFocus 模式
    if (Controller != nullptr && (需要焦点的模式))
    {
        APlayerController* PC = Cast<APlayerController>(Controller);
        if (PC != nullptr)
        {
            PC->GetPlayerViewPoint(CamLoc, CamRot);
        }
        else // AI Controller
        {
            CamLoc = SourcePawn->GetActorLocation() + FVector(0, 0, SourcePawn->BaseEyeHeight);
        }

        FVector AimDir = CamRot.Vector().GetSafeNormal();
        FocalLoc = CamLoc + (AimDir * FocalDistance);

        // 关键：将起始点偏移到武器位置所在的瞄准线上
        if (PC)
        {
            const FVector WeaponLoc = GetWeaponTargetingSourceLocation();
            CamLoc = FocalLoc + (((WeaponLoc - FocalLoc) | AimDir) * AimDir);
            FocalLoc = CamLoc + (AimDir * FocalDistance);
        }

        if (Source == CameraTowardsFocus)
            return FTransform(CamRot, CamLoc);
    }
    // ... 其他模式
}
```

**AI 支持**：AI 控制器没有 PlayerCameraManager，直接使用 Pawn 头部位置作为射线起始点。

### 7.8 网络同步 —— TargetData 流

```
客户端:
  ActivateAbility
    → StartRangedWeaponTargeting()
      → PerformLocalTargeting() → 得到 HitResults
      → 打包为 FGameplayAbilityTargetDataHandle
      → AddUnconfirmedServerSideHitMarkers() (本地预测命中标记)
      → OnTargetDataReadyCallback()
        → CallServerSetReplicatedTargetData() (发送到服务器)
        → AddSpread() (增加散布)
        → OnRangedWeaponTargetDataReady() (蓝图应用效果)

服务器:
  收到 TargetData
    → OnTargetDataReadyCallback()
      → ClientConfirmTargetData() (Client RPC 确认命中标记)
      → CommitAbility() → AddSpread()
      → OnRangedWeaponTargetDataReady() (蓝图应用 GE)
```

### 7.9 自定义 TargetData

```cpp
// Source/LyraGame/AbilitySystem/LyraGameplayAbilityTargetData_SingleTargetHit.h
USTRUCT()
struct FLyraGameplayAbilityTargetData_SingleTargetHit : public FGameplayAbilityTargetData_SingleTargetHit
{
    UPROPERTY()
    int32 CartridgeID;  // 标识同一发弹药中的多颗弹丸（霰弹枪场景）

    bool bHitReplaced;  // 服务端标记，命中被替换
};
```

---

## 八、武器状态管理：ULyraWeaponStateComponent

挂载在 **PlayerController** 上的组件，承担两大职责：

### 8.1 每帧更新武器散布

```cpp
void ULyraWeaponStateComponent::TickComponent(float DeltaTime, ...)
{
    if (APawn* Pawn = GetPawn<APawn>())
    {
        if (ULyraEquipmentManagerComponent* EquipmentManager = Pawn->FindComponentByClass<ULyraEquipmentManagerComponent>())
        {
            if (ULyraRangedWeaponInstance* CurrentWeapon = Cast<ULyraRangedWeaponInstance>(
                    EquipmentManager->GetFirstInstanceOfType(ULyraRangedWeaponInstance::StaticClass())))
            {
                CurrentWeapon->Tick(DeltaTime);  // 更新热度、散布、乘数
            }
        }
    }
}
```

> 注意：`ULyraRangedWeaponInstance` 本身是 UObject 不能注册 Tick，所以由这个 Component 代为驱动。

### 8.2 命中标记（Hit Marker）管线

```
客户端射击 → 生成 UnconfirmedServerSideHitMarkers (预测)
    │
    ├─ 投影 HitResult.Location 到屏幕空间
    ├─ 检查 TeamSubsystem.CanCauseDamage() → bShowAsSuccess
    └─ 读取 PhysicalMaterialWithTags.Tags → HitZone (如 Gameplay.Zone.Head)

服务器确认 → ClientConfirmTargetData (Client RPC)
    │
    ├─ 匹配 UniqueId → 找到对应的 UnconfirmedBatch
    ├─ 移除被替换的命中 (HitReplaces)
    └─ bShowAsSuccess 的命中 → 加入 LastWeaponDamageScreenLocations → 更新 DamageInstigatedTime
```

```cpp
struct FLyraScreenSpaceHitLocation
{
    FVector2D Location;       // 屏幕空间命中位置
    FGameplayTag HitZone;     // 命中区域 Tag（用于区分爆头/普通命中标记样式）
    bool bShowAsSuccess;      // 是否造成了有效伤害
};
```

---

## 九、快捷栏系统：ULyraQuickBarComponent

### 9.1 武器切换流程

```cpp
void ULyraQuickBarComponent::SetActiveSlotIndex_Implementation(int32 NewIndex)
{
    if (Slots.IsValidIndex(NewIndex) && (ActiveSlotIndex != NewIndex))
    {
        UnequipItemInSlot();    // 卸下当前武器
        ActiveSlotIndex = NewIndex;
        EquipItemInSlot();      // 装备新武器
        OnRep_ActiveSlotIndex(); // 广播消息
    }
}
```

### 9.2 Equip 流程详细

```cpp
void ULyraQuickBarComponent::EquipItemInSlot()
{
    if (ULyraInventoryItemInstance* SlotItem = Slots[ActiveSlotIndex])
    {
        // 1. 从物品定义中查找 EquippableItem Fragment
        if (const UInventoryFragment_EquippableItem* EquipInfo = SlotItem->FindFragmentByClass<UInventoryFragment_EquippableItem>())
        {
            TSubclassOf<ULyraEquipmentDefinition> EquipDef = EquipInfo->EquipmentDefinition;
            if (EquipDef != nullptr)
            {
                // 2. 调用 EquipmentManager 装备
                if (ULyraEquipmentManagerComponent* EquipmentManager = FindEquipmentManager())
                {
                    EquippedItem = EquipmentManager->EquipItem(EquipDef);
                    if (EquippedItem != nullptr)
                    {
                        // 3. 将物品实例设为装备的 Instigator（建立反向引用）
                        EquippedItem->SetInstigator(SlotItem);
                    }
                }
            }
        }
    }
}
```

**完整引用链**：
```
QuickBar.Slot[i] = InventoryItemInstance
    → FindFragment<EquippableItem>() → EquipmentDefinition
        → EquipmentManager.EquipItem() → EquipmentInstance (= WeaponInstance)
            → SetInstigator(InventoryItemInstance) // 反向引用
```

这样 GA 中可以通过 `GetAssociatedEquipment()` 获取武器实例，通过 `GetAssociatedItem()` 获取物品实例（读取弹药等）。

### 9.3 网络复制与消息广播

```cpp
// Slots 和 ActiveSlotIndex 均使用 ReplicatedUsing
UPROPERTY(ReplicatedUsing=OnRep_Slots)
TArray<TObjectPtr<ULyraInventoryItemInstance>> Slots;

UPROPERTY(ReplicatedUsing=OnRep_ActiveSlotIndex)
int32 ActiveSlotIndex = -1;

// OnRep 时通过 GameplayMessageSubsystem 广播消息
void ULyraQuickBarComponent::OnRep_Slots()
{
    FLyraQuickBarSlotsChangedMessage Message;
    Message.Owner = GetOwner();
    Message.Slots = Slots;
    UGameplayMessageSubsystem::Get(this).BroadcastMessage(TAG_Lyra_QuickBar_Message_SlotsChanged, Message);
}
```

UI 组件监听这些 GameplayMessage 即可响应武器切换事件，完全解耦。

---

## 十、伤害计算管线：ULyraDamageExecution

### 10.1 GameplayEffect Context 扩展

```cpp
// Source/LyraGame/AbilitySystem/LyraGameplayEffectContext.h
struct FLyraGameplayEffectContext : public FGameplayEffectContext
{
    UPROPERTY()
    int32 CartridgeID = -1;  // 弹药 ID（用于聚合同一发霰弹的多颗弹丸）

    UPROPERTY()
    TWeakObjectPtr<const UObject> AbilitySourceObject;  // 武器实例（实现 ILyraAbilitySourceInterface）

    const ILyraAbilitySourceInterface* GetAbilitySource() const;
    const UPhysicalMaterial* GetPhysicalMaterial() const;
};
```

### 10.2 伤害公式

```cpp
void ULyraDamageExecution::Execute_Implementation(...) const
{
    // 1. 获取基础伤害（来自 CombatSet.BaseDamage 属性）
    float BaseDamage = 0.0f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().BaseDamageDef, EvaluateParameters, BaseDamage);

    // 2. 友伤检查
    float DamageInteractionAllowedMultiplier = 0.0f;
    if (TeamSubsystem)
        DamageInteractionAllowedMultiplier = TeamSubsystem->CanCauseDamage(EffectCauser, HitActor) ? 1.0 : 0.0;

    // 3. 距离衰减 + 物理材质衰减（从 ILyraAbilitySourceInterface 查询）
    float PhysicalMaterialAttenuation = 1.0f;
    float DistanceAttenuation = 1.0f;
    if (const ILyraAbilitySourceInterface* AbilitySource = TypedContext->GetAbilitySource())
    {
        PhysicalMaterialAttenuation = AbilitySource->GetPhysicalMaterialAttenuation(PhysMat, ...);
        DistanceAttenuation = AbilitySource->GetDistanceAttenuation(Distance, ...);
    }

    // 4. 最终伤害 = Base × Distance × Material × TeamCheck
    const float DamageDone = FMath::Max(BaseDamage * DistanceAttenuation * PhysicalMaterialAttenuation * DamageInteractionAllowedMultiplier, 0.0f);

    // 5. 写入 HealthSet.Damage 属性（通过 Additive 修改器）
    if (DamageDone > 0.0f)
        OutExecutionOutput.AddOutputModifier(FGameplayModifierEvaluatedData(ULyraHealthSet::GetDamageAttribute(), EGameplayModOp::Additive, DamageDone));
}
```

**伤害管线公式**：
```
FinalDamage = BaseDamage × DistanceAttenuation(distance) × PhysicalMaterialAttenuation(physMat.Tags) × TeamCheck(0 或 1)
```

---

## 十一、武器拾取系统：ALyraWeaponSpawner

### 11.1 场景中的武器刷新点

```cpp
UCLASS(MinimalAPI, Blueprintable, BlueprintType)
class ALyraWeaponSpawner : public AActor
{
    UPROPERTY(EditInstanceOnly, BlueprintReadOnly)
    TObjectPtr<ULyraWeaponPickupDefinition> WeaponDefinition;  // 数据资产配置

    UPROPERTY(EditDefaultsOnly, ReplicatedUsing=OnRep_WeaponAvailability)
    bool bIsWeaponAvailable;  // 复制属性：武器是否可拾取

    UPROPERTY(EditDefaultsOnly)
    float CoolDownTime;  // 拾取后的冷却时间
};
```

### 11.2 拾取流程

```
OnOverlapBegin (仅服务端, ROLE_Authority)
    → AttemptPickUpWeapon(Pawn)
        → 检查 ASC 存在
        → GiveWeapon(WeaponItemClass, Pawn)  [BlueprintImplementableEvent]
            → (蓝图中) InventoryManager.AddItemDefinition()
            → (蓝图中) QuickBar.AddItemToSlot()
        → bIsWeaponAvailable = false
        → SetWeaponPickupVisibility(false)
        → PlayPickupEffects() (音效 + Niagara 粒子)
        → StartCoolDown() (Timer)

冷却结束 → ResetCoolDown()
    → bIsWeaponAvailable = true
    → PlayRespawnEffects()
    → SetWeaponPickupVisibility(true)
    → CheckForExistingOverlaps() (延迟检查是否有人站在刷新点上)
```

### 11.3 ULyraWeaponPickupDefinition

```cpp
UCLASS(MinimalAPI, Blueprintable, BlueprintType, Const)
class ULyraPickupDefinition : public UDataAsset
{
    TSubclassOf<ULyraInventoryItemDefinition> InventoryItemDefinition;
    TObjectPtr<UStaticMesh> DisplayMesh;
    int32 SpawnCoolDownSeconds;
    TObjectPtr<USoundBase> PickedUpSound;
    TObjectPtr<USoundBase> RespawnedSound;
    TObjectPtr<UNiagaraSystem> PickedUpEffect;
    TObjectPtr<UNiagaraSystem> RespawnedEffect;
};

class ULyraWeaponPickupDefinition : public ULyraPickupDefinition
{
    FVector WeaponMeshOffset;   // 展示网格偏移
    FVector WeaponMeshScale;    // 展示网格缩放
};
```

---

## 十二、武器 UI 系统

### 12.1 ULyraWeaponUserInterface —— 武器 HUD 容器

```cpp
void ULyraWeaponUserInterface::NativeTick(const FGeometry& MyGeometry, float InDeltaTime)
{
    if (APawn* Pawn = GetOwningPlayerPawn())
    {
        if (ULyraEquipmentManagerComponent* EquipmentManager = Pawn->FindComponentByClass<ULyraEquipmentManagerComponent>())
        {
            if (ULyraWeaponInstance* NewInstance = EquipmentManager->GetFirstInstanceOfType<ULyraWeaponInstance>())
            {
                if (NewInstance != CurrentInstance && NewInstance->GetInstigator() != nullptr)
                {
                    ULyraWeaponInstance* OldWeapon = CurrentInstance;
                    CurrentInstance = NewInstance;
                    RebuildWidgetFromWeapon();
                    OnWeaponChanged(OldWeapon, CurrentInstance);  // 蓝图事件
                }
            }
        }
    }
}
```

每帧检测当前武器是否变化，变化时触发蓝图 `OnWeaponChanged` 事件，蓝图中据此更新弹药显示、准心等。

### 12.2 ULyraReticleWidgetBase —— 准心基类

```cpp
float ULyraReticleWidgetBase::ComputeMaxScreenspaceSpreadRadius() const
{
    // 将散布角度转换为屏幕空间像素半径
    const float SpreadRadiusRads = FMath::DegreesToRadians(ComputeSpreadAngle() * 0.5f);
    const float SpreadRadiusAtDistance = FMath::Tan(SpreadRadiusRads) * LongShotDistance;

    // 在远距离处构造一个偏移点，投影到屏幕
    FVector OffsetTargetAtDistance = CamPos + (CamForwDir * LongShotDistance) + (CamUpDir * SpreadRadiusAtDistance);
    PC->ProjectWorldLocationToScreen(OffsetTargetAtDistance, OffsetTargetInScreenspace, true);

    // 屏幕空间距离 = 散布半径
    return (OffsetTargetInScreenspace - ScreenSpaceCenter).Length();
}
```

### 12.3 SCircumferenceMarkerWidget —— Slate 底层准心绘制

自定义 `SLeafWidget`，在圆周上按指定角度绘制标记图片。核心是 `GetMarkerRenderTransform()` 的旋转+位移变换组合：

```cpp
FSlateRenderTransform SCircumferenceMarkerWidget::GetMarkerRenderTransform(const FCircumferenceMarkerEntry& Marker, float BaseRadius, float HUDScale) const
{
    // 先绕自身中心旋转 (ImageRotationAngle)
    FSlateRenderTransform RotateAboutOrigin(Concatenate(
        FVector2D(-MarkerBrush->ImageSize.X * 0.5f, -MarkerBrush->ImageSize.Y * 0.5f),
        FQuat2D(LocalRotationRadians),
        FVector2D(MarkerBrush->ImageSize.X * 0.5f, MarkerBrush->ImageSize.Y * 0.5f)));

    // 再按 PositionAngle 偏移到圆周位置
    return TransformCast<FSlateRenderTransform>(Concatenate(RotateAboutOrigin,
        FVector2D(XRadius * FMath::Sin(PositionAngleRadians) * HUDScale,
                  -YRadius * FMath::Cos(PositionAngleRadians) * HUDScale)));
}
```

### 12.4 UHitMarkerConfirmationWidget —— 命中确认

支持：
- `PerHitMarkerImage`：每个命中位置的标记图
- `PerHitMarkerZoneOverrideImages`：按区域 Tag（如 HeadShot）覆盖标记图
- `AnyHitsMarkerImage`：只要有命中就显示的中心标记
- `HitNotifyDuration`：标记淡出时间

### 12.5 UInventoryFragment_ReticleConfig —— 准心配置

```cpp
UCLASS()
class UInventoryFragment_ReticleConfig : public ULyraInventoryItemFragment
{
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Reticle)
    TArray<TSubclassOf<ULyraReticleWidgetBase>> ReticleWidgets;
};
```

每把武器可以在自己的 ItemDefinition 中配置专属的准心 Widget 列表。武器切换时 UI 系统自动替换准心。

---

## 十三、辅助瞄准系统（Aim Assist）

位于 `Plugins/GameFeatures/ShooterCore/` 中，作为 Game Feature 插件独立于核心武器系统。

### 13.1 UAimAssistInputModifier

作为 Enhanced Input 的 `UInputModifier`，挂接在手柄摇杆输入上：

```cpp
UCLASS()
class UAimAssistInputModifier : public UInputModifier
{
    UPROPERTY(EditInstanceOnly)
    FAimAssistSettings Settings;

    UPROPERTY(EditInstanceOnly)
    FAimAssistFilter Filter;

    UPROPERTY(EditInstanceOnly)
    TObjectPtr<const UInputAction> MoveInputAction;

    UPROPERTY(EditInstanceOnly)
    ELyraTargetingType TargetingType;
};
```

### 13.2 双重辅助机制

**Pull（吸附）**：当目标在准心附近时，自动旋转视角朝向目标
```
PullStrength = f(InnerReticle? OuterReticle? ADS? Hip?)
    → PullInnerStrengthHip / PullOuterStrengthHip / PullInnerStrengthAds / PullOuterStrengthAds
    → 通过 PullLerpInRate / PullLerpOutRate 做平滑过渡
    → 受 PullMaxRotationRate 限制
```

**Slow（减速）**：当准心经过目标时，降低转向速率使准心"粘住"
```
SlowStrength = f(InnerReticle? OuterReticle? ADS? Hip?)
    → SlowInnerStrengthHip / SlowOuterStrengthHip / SlowInnerStrengthAds / SlowOuterStrengthAds
    → 受 SlowMinRotationRate 保底
```

### 13.3 目标筛选系统

```cpp
struct FAimAssistFilter
{
    uint8 bIncludeSameFriendlyTargets : 1;  // 是否包含友方
    uint8 bExcludeInstigator : 1;           // 排除自己
    uint8 bExcludeDeadOrDying : 1;          // 排除已死亡目标
    uint8 bTraceComplexCollision : 1;       // 使用复杂碰撞检测

    TSet<TObjectPtr<UClass>> ExcludedClasses;      // 排除的类
    FGameplayTagContainer ExclusionGameplayTags;   // 排除的 Tag
    double TargetRange = 10000.0;                  // 最大范围
};
```

### 13.4 目标评分与帧间缓存

```cpp
struct FLyraAimAssistTarget
{
    TWeakObjectPtr<UShapeComponent> TargetShapeComponent;
    FVector Location;
    FVector DeltaMovement;        // 目标移动增量
    FBox2D ScreenBounds;          // 屏幕空间包围盒
    float ViewDistance;
    float SortScore;              // 综合评分
    float AssistTime;             // 累积辅助时间
    float AssistWeight;           // 辅助权重

    FTraceHandle VisibilityTraceHandle;  // 异步可见性射线
    uint8 bIsVisible : 1;
    uint8 bUnderAssistInnerReticle : 1;
    uint8 bUnderAssistOuterReticle : 1;
};
```

使用 **双缓冲目标缓存**（`TargetCache0` / `TargetCache1`，通过 `SwapTargetCaches()` 交替），支持异步可见性检测和帧间目标持续性追踪。

---

## 十四、调试与诊断工具

### 14.1 ULyraWeaponDebugSettings

```cpp
UCLASS(config=EditorPerProjectUserSettings)
class ULyraWeaponDebugSettings : public UDeveloperSettingsBackedByCVars
{
    // 控制台变量：lyra.Weapon.DrawBulletTraceDuration
    UPROPERTY(config, EditAnywhere, meta=(ConsoleVariable="lyra.Weapon.DrawBulletTraceDuration"))
    float DrawBulletTraceDuration;

    // lyra.Weapon.DrawBulletHitDuration
    float DrawBulletHitDuration;

    // lyra.Weapon.DrawBulletHitRadius
    float DrawBulletHitRadius;
};
```

可通过编辑器项目设置面板或控制台命令实时开关射击调试线和命中点绘制。

### 14.2 ULyraDamageLogDebuggerComponent

```cpp
UCLASS(Blueprintable, meta=(BlueprintSpawnableComponent))
class ULyraDamageLogDebuggerComponent : public UActorComponent
{
    UPROPERTY(EditAnywhere)
    double SecondsBetweenDamageBeforeLogging = 1.0;
};
```

**功能**：
- 监听 `TAG_Lyra_Damage_Message` GameplayMessage
- 按帧聚合伤害数据到 `TMap<int64, FFrameDamageEntry>`（key = GFrameCounter）
- 当停火超过阈值后输出统计日志：
  - 总命中次数、独立帧数、总时间跨度、总伤害
  - 帧间隔的 min/max/avg（毫秒）
  - DPS（每秒伤害）

---

## 十五、网络复制策略总结

| 数据 | 复制方式 | 说明 |
|------|---------|------|
| EquipmentList | `FFastArraySerializer` | 增量复制装备列表 |
| EquipmentInstance (Instigator + SpawnedActors) | `DOREPLIFETIME` + `ReplicateSubobjects` | UObject 子对象复制 |
| InventoryList | `FFastArraySerializer` | 增量复制物品列表 |
| InventoryItemInstance (StatTags + ItemDef) | `DOREPLIFETIME` + `ReplicateSubobjects` | UObject 子对象复制 |
| QuickBar Slots | `ReplicatedUsing=OnRep_Slots` | 标准属性复制 |
| QuickBar ActiveSlotIndex | `ReplicatedUsing=OnRep_ActiveSlotIndex` | 标准属性复制 |
| WeaponSpawner bIsWeaponAvailable | `ReplicatedUsing=OnRep_WeaponAvailability` | 标准属性复制 |
| SetActiveSlotIndex | `Server, Reliable` RPC | 客户端 → 服务器 |
| ClientConfirmTargetData | `Client, Reliable` RPC | 服务器 → 客户端 |
| TargetData | GAS 内置 `CallServerSetReplicatedTargetData` | 客户端 → 服务器 |
| GA/GE 授予 | GAS 内置复制 | GrantedHandles 标记 NotReplicated |

---

## 十六、完整数据流：从拾取到伤害

```
1. 世界拾取
   ALyraWeaponSpawner.OnOverlapBegin()
     → AttemptPickUpWeapon(Pawn)
       → GiveWeapon() [蓝图]
         → InventoryManager.AddItemDefinition(武器ItemDef)
           → 创建 InventoryItemInstance
           → Fragment_SetStats.OnInstanceCreated() → 注入初始弹药
         → QuickBar.AddItemToSlot(SlotIndex, ItemInstance)

2. 武器装备
   QuickBar.SetActiveSlotIndex(NewIndex) [Server RPC]
     → UnequipItemInSlot() → EquipmentManager.UnequipItem()
       → OnUnequipped() → RemoveDeviceProperties() + DestroyEquipmentActors() + TakeFromAbilitySystem()
     → EquipItemInSlot()
       → FindFragment<EquippableItem>() → EquipmentDefinition
       → EquipmentManager.EquipItem(EquipDef)
         → NewObject<ULyraRangedWeaponInstance>()
         → AbilitySet.GiveToAbilitySystem() → 授予射击 GA + 伤害 GE
         → SpawnEquipmentActors() → 生成武器模型
         → OnEquipped()
           → InitHeat → InitSpread → ApplyDeviceProperties()
       → SetInstigator(ItemInstance) → 建立反向引用

3. 射击
   玩家按下开火键 → Enhanced Input → GAS 激活 GA_RangedWeapon
     → ActivateAbility()
       → 绑定 TargetData 回调
       → UpdateFiringTime()
     → [蓝图] PlayMontageAndWait → StartRangedWeaponTargeting()
       → PerformLocalTargeting()
         → GetTargetingTransform(CameraTowardsFocus)
         → TraceBulletsInCartridge()
           → For each bullet: VRandConeNormalDistribution() → DoSingleBulletTrace()
             → WeaponTrace(Line) → if miss → WeaponTrace(Sweep)
       → 打包 TargetDataHandle (附带 CartridgeID)
       → WeaponStateComponent.AddUnconfirmedServerSideHitMarkers()
       → OnTargetDataReadyCallback()
         → CallServerSetReplicatedTargetData() [if client]
         → WeaponData.AddSpread() → CurrentHeat += HeatPerShot
         → OnRangedWeaponTargetDataReady() [蓝图应用 GE]

4. 伤害计算
   GE with LyraDamageExecution
     → Extract FLyraGameplayEffectContext
       → GetAbilitySource() → ULyraRangedWeaponInstance
       → GetPhysicalMaterial() → UPhysicalMaterialWithTags
     → BaseDamage (from CombatSet)
     × DistanceAttenuation (from weapon DistanceDamageFalloff curve)
     × PhysicalMaterialAttenuation (from weapon MaterialDamageMultiplier map)
     × TeamCheck (from TeamSubsystem.CanCauseDamage)
     → Write to HealthSet.Damage attribute
       → HealthSet.PostGameplayEffectExecute()
         → HealthComponent processes damage / death

5. 命中确认
   Server → ClientConfirmTargetData() [Client RPC]
     → 匹配 UnconfirmedBatch
     → 移除 HitReplaces
     → bShowAsSuccess → LastWeaponDamageScreenLocations
     → HitMarkerConfirmationWidget 显示命中标记
```

---

## 十七、Lyra 武器系统设计亮点与启发

### 17.1 Fragment 组合模式

**传统做法**：武器继承 → 步枪类、霰弹枪类、手枪类…
**Lyra 做法**：统一的 `InventoryItemDefinition` + `Fragment` 组合

优势：
- 添加新功能（如准心、图标）只需新建 Fragment 类，无需修改任何已有代码
- 同一把武器可以灵活组合不同 Fragment（有些武器有弹药、有些没有）
- 完全符合 **开闭原则（OCP）**

### 17.2 曲线驱动的散布系统

不是简单的"散布角度 = 基准 + 每发增量 - 冷却"的线性公式，而是：
- **三条独立曲线**（Heat→Spread, Heat→HeatPerShot, Heat→CoolDown）
- **四个独立乘数**（站立、蹲伏、跳跃、瞄准），各有独立的过渡速率
- **指数分布**（SpreadExponent）控制弹丸聚集度

这让设计师可以制作出各种手感：步枪前几发稳定后越打越飘、霰弹枪均匀散射、狙击枪首发精准等。

### 17.3 双层射线检测

Line → Sweep 回退策略在不牺牲精度的前提下提供了更好的命中手感，特别适合主机手柄玩家。

### 17.4 Tag 驱动的一切

- 弹药：`GameplayTagStackContainer` 中的 Tag 栈
- 物理材质伤害：`Tag → 伤害乘数` 映射
- 命中区域：物理材质上的 `Gameplay.Zone.xxx` Tag
- 武器状态：`Ability.Weapon.NoFiring` Tag 阻止开火
- 动画层选择：Cosmetic Tag 匹配

### 17.5 关注点分离

| 关注点 | 负责的类 | 挂载位置 |
|--------|---------|---------|
| 物品数据 | InventoryItemDefinition + Fragments | 资产 |
| 物品实例状态 | InventoryItemInstance | Pawn 上的 InventoryManager |
| 装备定义（GA/Actor） | EquipmentDefinition | 资产 |
| 装备运行时 | EquipmentInstance → WeaponInstance | Pawn 上的 EquipmentManager |
| 武器切换意图 | QuickBarComponent | Controller |
| 武器状态/散布 Tick | WeaponStateComponent | Controller |
| 射击逻辑 | GA_RangedWeapon | GAS |
| 伤害计算 | DamageExecution | GAS |
| 准心/命中 UI | ReticleWidget / HitMarkerWidget | HUD |
| 辅助瞄准 | AimAssistInputModifier | Enhanced Input |
| 武器拾取 | WeaponSpawner | World Actor |

### 17.6 网络架构

- **武器切换**：Server RPC（SetActiveSlotIndex_Implementation）→ 服务器装备 → FastArray 复制回客户端 → 客户端 PostReplicatedAdd 触发 OnEquipped
- **射击**：客户端本地检测 → TargetData 发到服务器 → 服务器确认 → Client RPC 回传命中标记
- **武器拾取**：仅服务端处理 → bIsWeaponAvailable 复制 → 客户端 OnRep 播放效果

这种架构确保了 **Authority 在服务端** 的同时，通过预测性命中标记和 GAS 的预测框架提供良好的客户端手感。
