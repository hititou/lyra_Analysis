# Lyra StarterGame — 角色动画蓝图分层逻辑深度分析

> **项目版本**: UE5 Lyra StarterGame  
> **分析日期**: 2026-03-10  
> **分析范围**: `Source/LyraGame/`、`Plugins/`、`Content/` 中所有动画相关的 C++ 源码及蓝图资产

---

## 目录

- [1. 架构总览](#1-架构总览)
- [2. 分层体系详解](#2-分层体系详解)
  - [2.1 第1层：主动画蓝图（ABP_Mannequin_Base）](#21-第1层主动画蓝图abp_mannequin_base)
  - [2.2 第2层：动画层接口（ALI_ItemAnimLayers）](#22-第2层动画层接口ali_itemanimlayers)
  - [2.3 第3层：武器特定动画层（ABP_XXXAnimLayers）](#23-第3层武器特定动画层abp_xxxanimlayers)
  - [2.4 第4层：武器自身动画蓝图（ABP_Weap_XXX）](#24-第4层武器自身动画蓝图abp_weap_xxx)
  - [2.5 辅助层：后处理与重定向](#25-辅助层后处理与重定向)
- [3. C++ 核心类详解](#3-c-核心类详解)
  - [3.1 ULyraAnimInstance — 动画实例基类](#31-ulyraaniminstance--动画实例基类)
  - [3.2 FLyraAnimLayerSelectionSet — 动画层选择集](#32-flyraanimlayerselectionset--动画层选择集)
  - [3.3 FLyraAnimBodyStyleSelectionSet — 身体网格选择集](#33-flyraanimbodystyleselectionset--身体网格选择集)
  - [3.4 ULyraWeaponInstance — 武器实例](#34-ulyraweaponinstance--武器实例)
- [4. Cosmetic 外观系统与动画的协作](#4-cosmetic-外观系统与动画的协作)
  - [4.1 CosmeticTag 的产生链路](#41-cosmetictag-的产生链路)
  - [4.2 Controller 层组件](#42-controller-层组件)
  - [4.3 Pawn 层组件](#43-pawn-层组件)
  - [4.4 CosmeticTag 完整流转图](#44-cosmetictag-完整流转图)
- [5. 装备系统与动画层的联动](#5-装备系统与动画层的联动)
  - [5.1 装备定义与实例](#51-装备定义与实例)
  - [5.2 武器装备触发动画层切换的完整流程](#52-武器装备触发动画层切换的完整流程)
- [6. GAS（能力系统）与动画的集成](#6-gas能力系统与动画的集成)
  - [6.1 GameplayTag 自动映射](#61-gameplaytag-自动映射)
  - [6.2 Montage 失败消息机制](#62-montage-失败消息机制)
- [7. 蓝图资产目录结构与命名规范](#7-蓝图资产目录结构与命名规范)
  - [7.1 完整目录树](#71-完整目录树)
  - [7.2 动画资产命名规范](#72-动画资产命名规范)
  - [7.3 各武器动画资产清单](#73-各武器动画资产清单)
- [8. 动画通知——上下文效果系统](#8-动画通知上下文效果系统)
- [9. 网络复制与多人同步](#9-网络复制与多人同步)
- [10. 设计理念与架构优势总结](#10-设计理念与架构优势总结)

---

## 1. 架构总览

Lyra 的角色动画系统采用了 UE5 **Linked Anim Layers（链接动画层）** 架构，实现了高度解耦、数据驱动的动画分层设计。整体架构可概括为以下分层模型：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          ALyraCharacter                                  │
│  └─ SkeletalMeshComponent                                               │
│       │                                                                  │
│       ├─ AnimClass: ABP_Mannequin_Base (ULyraAnimInstance)              │
│       │     ├── 基础状态机 (Idle / Move / Jump / Fall / Death)           │
│       │     ├── GameplayTag → 动画变量自动映射                            │
│       │     ├── GroundDistance 实时计算                                   │
│       │     └── Linked Anim Layer 插槽                                   │
│       │           │                                                      │
│       │           ▼                                                      │
│       │     ALI_ItemAnimLayers (动画层接口)                               │
│       │           │  定义可替换的动画层函数签名                            │
│       │           ▼  运行时通过 LinkAnimClassLayers 动态链接              │
│       │     ┌─────────────────────────────────────────────┐              │
│       │     │ ABP_PistolAnimLayers      (手枪·Masculine)  │              │
│       │     │ ABP_PistolAnimLayers_Feminine (手枪·Feminine)│              │
│       │     │ ABP_RifleAnimLayers       (步枪·Masculine)  │              │
│       │     │ ABP_RifleAnimLayers_Feminine  (步枪·Feminine)│              │
│       │     │ ABP_ShotgunAnimLayers     (霰弹枪·Masculine)│              │
│       │     │ ABP_ShotgunAnimLayers_Feminine(霰弹枪·Fem.) │              │
│       │     │ ABP_UnarmedAnimLayers     (无武器·Masculine) │              │
│       │     │ ABP_UnarmedAnimLayers_Feminine(无武器·Fem.)  │              │
│       │     └─────────────────────────────────────────────┘              │
│       │                                                                  │
│       ├─ BodyMesh 动态切换 (SKM_Manny / SKM_Quinn)                      │
│       └─ PostProcess ABPs (ABP_Manny_PostProcess / ABP_Quinn_PostProcess)│
│                                                                          │
│  武器 Actor:                                                              │
│       └─ SkeletalMeshComponent                                           │
│            └─ AnimClass: ABP_Weap_XXX (武器自身动画)                      │
└─────────────────────────────────────────────────────────────────────────┘
```

**核心设计原则**：
- **C++ 层不硬编码任何 AnimBP 引用**（全部通过蓝图/数据资产配置）
- **PawnData 中不包含 AnimBP 字段**（动画蓝图配置在 PawnClass 蓝图的 Mesh 组件上）
- **动画层的切换完全由 CosmeticTag + 武器系统驱动**
- **`LinkAnimClassLayers` 的调用在蓝图端完成**，C++ 只负责选择逻辑

---

## 2. 分层体系详解

### 2.1 第1层：主动画蓝图（ABP_Mannequin_Base）

**资产路径**:
```
Content/Characters/Heroes/Mannequin/Animations/ABP_Mannequin_Base.uasset
```

**C++ 基类**: `ULyraAnimInstance`（继承自 `UAnimInstance`）

**源码位置**:
- `Source/LyraGame/Animation/LyraAnimInstance.h`
- `Source/LyraGame/Animation/LyraAnimInstance.cpp`

**核心职责**：

| 功能 | 实现方式 |
|------|----------|
| 基础状态机 | 蓝图中定义的 State Machine，处理 Idle/Move/Jump/Fall/Death 等状态切换 |
| GAS Tag 映射 | `FGameplayTagBlueprintPropertyMap` 自动将 GameplayTag 映射到动画蓝图变量 |
| 地面距离 | 每帧从 `LyraCharacterMovementComponent` 获取 `GroundDistance`，驱动落地动画 |
| Linked Layer 插槽 | 通过 `ALI_ItemAnimLayers` 接口声明可替换的动画层插槽 |

**关键代码**：

```cpp
// LyraAnimInstance.h
UCLASS(Config = Game)
class ULyraAnimInstance : public UAnimInstance
{
    // GameplayTag -> 蓝图变量的自动映射（在编辑器中配置）
    UPROPERTY(EditDefaultsOnly, Category = "GameplayTags")
    FGameplayTagBlueprintPropertyMap GameplayTagPropertyMap;

    // 地面距离（蓝图中用于落地/跳跃动画混合）
    UPROPERTY(BlueprintReadOnly, Category = "Character State Data")
    float GroundDistance = -1.0f;
};

// LyraAnimInstance.cpp
void ULyraAnimInstance::NativeInitializeAnimation()
{
    // 自动获取 ASC 并初始化 GameplayTag 属性映射
    if (UAbilitySystemComponent* ASC = 
        UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(GetOwningActor()))
    {
        InitializeWithAbilitySystem(ASC);
    }
}

void ULyraAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
    // 每帧更新地面距离
    const ALyraCharacter* Character = Cast<ALyraCharacter>(GetOwningActor());
    ULyraCharacterMovementComponent* CharMoveComp = 
        CastChecked<ULyraCharacterMovementComponent>(Character->GetCharacterMovement());
    GroundDistance = CharMoveComp->GetGroundInfo().GroundDistance;
}
```

**与 GAS 的绑定时机**：

在 `ULyraAbilitySystemComponent::InitAbilityActorInfo()` 中，当新的 Pawn Avatar 被设置时：

```cpp
// LyraAbilitySystemComponent.cpp
if (ULyraAnimInstance* LyraAnimInst = Cast<ULyraAnimInstance>(ActorInfo->GetAnimInstance()))
{
    LyraAnimInst->InitializeWithAbilitySystem(this);
}
```

### 2.2 第2层：动画层接口（ALI_ItemAnimLayers）

**资产路径**:
```
Content/Characters/Heroes/Mannequin/Animations/LinkedLayers/ALI_ItemAnimLayers.uasset
```

**性质**: UE5 的 **Anim Layer Interface**（动画层接口蓝图资产）

**职责**:
- 定义一组**动画层函数签名**（如 FullBody_IdleState、FullBody_StartState、UpperBody_AimOffset 等）
- 这些函数签名在主 ABP (`ABP_Mannequin_Base`) 中作为 **Linked Anim Layer 节点** 被引用
- 具体实现由第3层的武器 AnimLayer 蓝图提供
- 这是一个**纯接口层**，不包含任何动画逻辑实现

**关键设计意义**:
> 这一层将"接口定义"与"具体实现"完全分离。主 ABP 只依赖接口，不知道最终由哪个武器 AnimLayer 来填充内容。这使得新增武器类型时完全不需要修改主 ABP。

### 2.3 第3层：武器特定动画层（ABP_XXXAnimLayers）

**资产路径** 及其层级关系:

```
Content/Characters/Heroes/Mannequin/Animations/
├── LinkedLayers/
│   └── ABP_ItemAnimLayersBase.uasset        ← 所有武器 AnimLayer 的公共基类
│
└── Locomotion/
    ├── Pistol/
    │   ├── ABP_PistolAnimLayers.uasset           ← 手枪 Masculine 版本
    │   └── ABP_PistolAnimLayers_Feminine.uasset  ← 手枪 Feminine 版本
    ├── Rifle/
    │   ├── ABP_RifleAnimLayers.uasset            ← 步枪 Masculine 版本
    │   └── ABP_RifleAnimLayers_Feminine.uasset   ← 步枪 Feminine 版本
    ├── Shotgun/
    │   ├── ABP_ShotgunAnimLayers.uasset          ← 霰弹枪 Masculine 版本
    │   └── ABP_ShotgunAnimLayers_Feminine.uasset ← 霰弹枪 Feminine 版本
    └── Unarmed/
        ├── ABP_UnarmedAnimLayers.uasset          ← 无武器 Masculine 版本
        └── ABP_UnarmedAnimLayers_Feminine.uasset ← 无武器 Feminine 版本
```

**继承关系**:
```
UAnimInstance
  └── ABP_ItemAnimLayersBase (implements ALI_ItemAnimLayers)
        ├── ABP_PistolAnimLayers
        ├── ABP_PistolAnimLayers_Feminine
        ├── ABP_RifleAnimLayers
        ├── ABP_RifleAnimLayers_Feminine
        ├── ABP_ShotgunAnimLayers
        ├── ABP_ShotgunAnimLayers_Feminine
        ├── ABP_UnarmedAnimLayers
        └── ABP_UnarmedAnimLayers_Feminine
```

**各动画层包含的动画类型**（以手枪 Masculine 为例）:

| 分类 | 动画资产 | 说明 |
|------|---------|------|
| **站立空闲** | `MM_Pistol_Idle_Hipfire`, `MM_Pistol_Idle_ADS` | 腰射/瞄准状态 |
| **空闲打断** | `MM_Pistol_IdleBreak_Scan` | 长时间空闲时的自然动作 |
| **慢跑 (Jog)** | `MM_Pistol_Jog_Fwd/Bwd/Left/Right` | 四向慢跑循环 |
| **慢跑起步/停步** | `MM_Pistol_Jog_Fwd_Start/Stop`, ... | 起步加速和减速停止 |
| **慢跑转向** | `MM_Pistol_Jog_Pivot_Fwd/Bwd/Left/Right` | 方向转换过渡 |
| **行走 (Walk)** | `MM_Pistol_Walk_Fwd/Bwd/Left/Right` | 四向行走循环 |
| **行走起步/停步** | `MM_Pistol_Walk_Fwd_Start/Stop`, ... | 行走状态过渡 |
| **蹲伏空闲** | `MM_Pistol_Crouch_Idle` | 蹲伏静止 |
| **蹲伏进入/退出** | `MM_Pistol_Crouch_Entry/Exit` | 站蹲切换过渡 |
| **蹲伏移动** | `MM_Pistol_Crouch_Walk_Fwd/Bwd/Left/Right` | 四向蹲伏移动 |
| **蹲伏转向** | `MM_Pistol_Crouch_TurnLeft/Right_90/180` | 蹲伏原地转身 |
| **站立转向** | `MM_Pistol_TurnLeft/Right_90/180` | 站立原地转身 |
| **跳跃** | `MM_Pistol_Jump_Start/Start_Loop/Apex/Fall_Loop/Fall_Land` | 完整跳跃序列 |
| **跳跃恢复** | `MM_Pistol_Jump_RecoveryAdditive` | 落地恢复叠加动画 |

**Feminine（女性角色 Quinn）版本**的区别:
- 前缀从 `MM_` 变为 `MF_`（**M**annequin **M**asculine → **M**annequin **F**eminine）
- 动画资产数量和覆盖的状态完全相同，但动作风格适配女性体型

### 2.4 第4层：武器自身动画蓝图（ABP_Weap_XXX）

**资产路径**:
```
Content/Weapons/
├── Pistol/Animations/ABP_Weap_Pistol.uasset
├── Rifle/Animations/ABP_Weap_Rifle.uasset
└── Shotgun/Animations/ABP_Weap_Shotgun.uasset
```

**说明**:
这些是**武器模型自身的动画蓝图**（不是角色的），用于驱动武器骨骼网格的动画，例如：
- 开火时武器的后坐力动画
- 换弹时武器的动画配合
- 武器的空闲呼吸动画

武器 Actor 通过装备系统生成，挂载到角色的 `SkeletalMeshComponent` 上的 Socket。

### 2.5 辅助层：后处理与重定向

```
Content/Characters/Heroes/Mannequin/Rig/
├── ABP_Manny_PostProcess.uasset    ← Manny 后处理动画蓝图
├── ABP_Quinn_PostProcess.uasset    ← Quinn 后处理动画蓝图
├── CR_Mannequin_Body.uasset        ← Control Rig：身体
├── CR_Mannequin_FootPlant.uasset   ← Control Rig：脚部植入
├── CR_Mannequin_Procedural.uasset  ← Control Rig：程序化动画
└── IK_Mannequin.uasset             ← IK Rig

Content/Characters/Heroes/Mannequin/Animations/
├── ABP_Mannequin_CopyPose.uasset   ← Copy Pose 动画蓝图（用于观察者/镜像）
├── ABP_Mannequin_Retarget.uasset   ← 重定向动画蓝图
└── ABP_Mannequin_TopDown.uasset    ← 俯视角模式的简化动画蓝图
```

---

## 3. C++ 核心类详解

### 3.1 ULyraAnimInstance — 动画实例基类

**源码**: `Source/LyraGame/Animation/LyraAnimInstance.h/cpp`

```cpp
UCLASS(Config = Game)
class ULyraAnimInstance : public UAnimInstance
{
    GENERATED_BODY()

public:
    ULyraAnimInstance(const FObjectInitializer& ObjectInitializer);
    virtual void InitializeWithAbilitySystem(UAbilitySystemComponent* ASC);

protected:
    virtual void NativeInitializeAnimation() override;
    virtual void NativeUpdateAnimation(float DeltaSeconds) override;

    // GameplayTag -> 蓝图变量的自动映射
    // 在编辑器中配置，如：
    //   Tag "Status.Crouching" → bool bIsCrouching
    //   Tag "Status.ADS"       → bool bIsADS
    UPROPERTY(EditDefaultsOnly, Category = "GameplayTags")
    FGameplayTagBlueprintPropertyMap GameplayTagPropertyMap;

    // 角色到地面的距离
    UPROPERTY(BlueprintReadOnly, Category = "Character State Data")
    float GroundDistance = -1.0f;
};
```

**设计亮点**:
- 使用 `FGameplayTagBlueprintPropertyMap` 实现了 GAS Tag 到动画变量的**声明式映射**，避免了传统方式中在 `NativeUpdateAnimation` 里大量手动查询 Tag 的反模式
- 初始化在两个地方都有保障：`NativeInitializeAnimation()` 和 ASC 的 `InitAbilityActorInfo()`，确保不管哪个先就绪都能正确绑定

### 3.2 FLyraAnimLayerSelectionSet — 动画层选择集

**源码**: `Source/LyraGame/Cosmetics/LyraCosmeticAnimationTypes.h/cpp`

```cpp
// 单条选择规则
USTRUCT(BlueprintType)
struct FLyraAnimLayerSelectionEntry
{
    GENERATED_BODY()

    // 匹配时使用的动画层类
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSubclassOf<UAnimInstance> Layer;

    // 需要满足的所有 CosmeticTag
    UPROPERTY(EditAnywhere, BlueprintReadWrite, meta=(Categories="Cosmetic"))
    FGameplayTagContainer RequiredTags;
};

// 选择集（包含多条规则 + 默认值）
USTRUCT(BlueprintType)
struct FLyraAnimLayerSelectionSet
{
    GENERATED_BODY()

    // 按优先级排列的规则列表
    UPROPERTY(EditAnywhere, BlueprintReadWrite, meta=(TitleProperty=Layer))
    TArray<FLyraAnimLayerSelectionEntry> LayerRules;

    // 无规则匹配时的默认层
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSubclassOf<UAnimInstance> DefaultLayer;

    // 选择算法
    TSubclassOf<UAnimInstance> SelectBestLayer(const FGameplayTagContainer& CosmeticTags) const;
};
```

**选择算法实现**:

```cpp
TSubclassOf<UAnimInstance> FLyraAnimLayerSelectionSet::SelectBestLayer(
    const FGameplayTagContainer& CosmeticTags) const
{
    // 遍历规则列表，第一个满足条件的规则被选中
    for (const FLyraAnimLayerSelectionEntry& Rule : LayerRules)
    {
        if ((Rule.Layer != nullptr) && CosmeticTags.HasAll(Rule.RequiredTags))
        {
            return Rule.Layer;
        }
    }
    // 无匹配规则时返回默认层
    return DefaultLayer;
}
```

**典型配置示例**（概念性）:

| 优先级 | RequiredTags | Layer |
|--------|-------------|-------|
| 1 | `Cosmetic.BodyStyle.Feminine` | `ABP_PistolAnimLayers_Feminine` |
| 2 | *(空，匹配一切)* | `ABP_PistolAnimLayers` |
| 默认 | — | `ABP_PistolAnimLayers` |

### 3.3 FLyraAnimBodyStyleSelectionSet — 身体网格选择集

**源码**: 同文件 `LyraCosmeticAnimationTypes.h/cpp`

```cpp
USTRUCT(BlueprintType)
struct FLyraAnimBodyStyleSelectionSet
{
    GENERATED_BODY()

    // 身体网格规则列表
    UPROPERTY(EditAnywhere, BlueprintReadWrite, meta=(TitleProperty=Mesh))
    TArray<FLyraAnimBodyStyleSelectionEntry> MeshRules;

    // 默认网格
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TObjectPtr<USkeletalMesh> DefaultMesh = nullptr;

    // 强制使用的物理资产（覆盖网格自带的）
    UPROPERTY(EditAnywhere)
    TObjectPtr<UPhysicsAsset> ForcedPhysicsAsset = nullptr;

    USkeletalMesh* SelectBestBodyStyle(const FGameplayTagContainer& CosmeticTags) const;
};
```

选择逻辑与 `SelectBestLayer` 完全对称，用于根据 CosmeticTag 选择 `SKM_Manny` 或 `SKM_Quinn` 骨骼网格。

**可选网格**（`Content/Characters/Heroes/Mannequin/Meshes/`）:

| 网格 | 用途 |
|------|------|
| `SKM_Manny.uasset` | Masculine 体型（Manny） |
| `SKM_Quinn.uasset` | Feminine 体型（Quinn） |
| `SKM_Manny_Invis.uasset` | Masculine 不可见版本（第一人称等场景） |
| `SKM_Quinn_Invis.uasset` | Feminine 不可见版本 |
| `SK_Mannequin.uasset` | 通用 Skeleton 资产 |

### 3.4 ULyraWeaponInstance — 武器实例

**源码**: `Source/LyraGame/Weapons/LyraWeaponInstance.h/cpp`

```cpp
UCLASS(MinimalAPI)
class ULyraWeaponInstance : public ULyraEquipmentInstance
{
    GENERATED_BODY()

protected:
    // 装备时使用的动画层选择规则
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Animation)
    FLyraAnimLayerSelectionSet EquippedAnimSet;

    // 未装备时使用的动画层选择规则
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Animation)
    FLyraAnimLayerSelectionSet UneuippedAnimSet;

    // 根据装备状态和 CosmeticTags 选择最佳动画层
    UFUNCTION(BlueprintCallable, BlueprintPure=false, Category=Animation)
    TSubclassOf<UAnimInstance> PickBestAnimLayer(
        bool bEquipped, 
        const FGameplayTagContainer& CosmeticTags) const;
};
```

**实现**:
```cpp
TSubclassOf<UAnimInstance> ULyraWeaponInstance::PickBestAnimLayer(
    bool bEquipped, const FGameplayTagContainer& CosmeticTags) const
{
    const FLyraAnimLayerSelectionSet& SetToQuery = 
        (bEquipped ? EquippedAnimSet : UneuippedAnimSet);
    return SetToQuery.SelectBestLayer(CosmeticTags);
}
```

**设计说明**:
- 每把武器自带两套动画层选择规则：`EquippedAnimSet`（装备时，如手持手枪的移动/瞄准动画）和 `UneuippedAnimSet`（卸装时，回退到无武器动画）
- `PickBestAnimLayer` 被标记为 `BlueprintCallable`，意味着蓝图端在装备/卸装事件中调用此函数获取目标 AnimLayer 类，然后调用 `LinkAnimClassLayers` 完成动态链接

---

## 4. Cosmetic 外观系统与动画的协作

### 4.1 CosmeticTag 的产生链路

CosmeticTag 是驱动动画层选择和身体网格切换的**核心数据源**。其产生遵循以下链路：

```
Part Actor (如头部、躯体装饰)
    │  实现 IGameplayTagAssetInterface
    │  自带 GameplayTag（如 "Cosmetic.BodyStyle.Feminine"）
    ▼
CollectCombinedTags()
    │  收集所有已生成 Part Actor 的 Tag
    ▼
合并后的 FGameplayTagContainer (CosmeticTags)
    │
    ├──→ SelectBestBodyStyle() → 选择骨骼网格（Manny/Quinn）
    └──→ SelectBestLayer()    → 选择动画层（Masculine/Feminine 变体）
```

### 4.2 Controller 层组件

**源码**: `Source/LyraGame/Cosmetics/LyraControllerComponent_CharacterParts.h/cpp`

```cpp
UCLASS(meta = (BlueprintSpawnableComponent))
class ULyraControllerComponent_CharacterParts : public UControllerComponent
{
    // 在编辑器/蓝图中配置的角色外观部件列表
    UPROPERTY(EditAnywhere, Category=Cosmetics)
    TArray<FLyraControllerCharacterPartEntry> CharacterParts;
};
```

**工作流程**:
1. Controller 蓝图中预设 `CharacterParts`（如头部模型、身体装饰等）
2. 监听 `OnPossessedPawnChanged` 事件
3. Possess 新 Pawn 时，找到 Pawn 上的 `ULyraPawnComponent_CharacterParts`
4. 将所有 Part 逐个添加到 Pawn 组件
5. 每个 Part 来源有四种类型标记：
   - `Natural` — 正常配置的部件
   - `NaturalSuppressedViaCheat` — 被作弊命令压制的自然部件
   - `AppliedViaDeveloperSettingsCheat` — 开发者设置应用的
   - `AppliedViaCheatManager` — 作弊管理器添加的

### 4.3 Pawn 层组件

**源码**: `Source/LyraGame/Cosmetics/LyraPawnComponent_CharacterParts.h/cpp`

这是 CosmeticTag 系统的**核心执行者**。

**关键数据**:
```cpp
UCLASS(meta=(BlueprintSpawnableComponent))
class ULyraPawnComponent_CharacterParts : public UPawnComponent
{
    // 可网络复制的外观部件列表（FFastArraySerializer）
    UPROPERTY(Replicated, Transient)
    FLyraCharacterPartList CharacterPartList;

    // 根据 CosmeticTag 选择身体网格的规则
    UPROPERTY(EditAnywhere, Category=Cosmetics)
    FLyraAnimBodyStyleSelectionSet BodyMeshes;

    // 外观变化通知委托
    UPROPERTY(BlueprintAssignable, Category=Cosmetics)
    FLyraSpawnedCharacterPartsChanged OnCharacterPartsChanged;
};
```

**CosmeticTag 收集算法**:
```cpp
FGameplayTagContainer FLyraCharacterPartList::CollectCombinedTags() const
{
    FGameplayTagContainer Result;
    for (const FLyraAppliedCharacterPartEntry& Entry : Entries)
    {
        if (Entry.SpawnedComponent != nullptr)
        {
            // 从生成的 Part Actor 上收集 GameplayTag
            if (IGameplayTagAssetInterface* TagInterface = 
                Cast<IGameplayTagAssetInterface>(Entry.SpawnedComponent->GetChildActor()))
            {
                TagInterface->GetOwnedGameplayTags(/*inout*/ Result);
            }
        }
    }
    return Result;
}
```

**身体网格动态切换**（`BroadcastChanged()`）:
```cpp
void ULyraPawnComponent_CharacterParts::BroadcastChanged()
{
    if (USkeletalMeshComponent* MeshComponent = GetParentMeshComponent())
    {
        // 1. 收集所有 Part Actor 的 CosmeticTag
        const FGameplayTagContainer MergedTags = GetCombinedTags(FGameplayTag());
        
        // 2. 根据 Tag 选择最佳骨骼网格
        USkeletalMesh* DesiredMesh = BodyMeshes.SelectBestBodyStyle(MergedTags);
        
        // 3. 应用网格（如果没有变化则为空操作）
        MeshComponent->SetSkeletalMesh(DesiredMesh, /*bReinitPose=*/ true);
        
        // 4. 如有强制物理资产，也一并应用
        if (UPhysicsAsset* PhysicsAsset = BodyMeshes.ForcedPhysicsAsset)
        {
            MeshComponent->SetPhysicsAsset(PhysicsAsset, /*bForceReInit=*/ true);
        }
    }
    // 5. 通知所有观察者（如队伍着色系统）
    OnCharacterPartsChanged.Broadcast(this);
}
```

### 4.4 CosmeticTag 完整流转图

```
┌──────────────────────────────────┐
│  PlayerController                │
│  └─ ULyraControllerComponent_   │
│     CharacterParts               │
│     CharacterParts[] = [         │
│       { Part: BP_MannequinBody,  │
│         Socket: None }           │
│     ]                            │
└──────────┬───────────────────────┘
           │ OnPossessedPawnChanged
           │ AddCharacterPart()
           ▼
┌──────────────────────────────────┐
│  Character (Pawn)                │
│  └─ ULyraPawnComponent_         │
│     CharacterParts               │
│     │                            │
│     ├─ CharacterPartList         │
│     │   (FFastArraySerializer    │
│     │    网络复制)                │
│     │   └─ Entries[]             │
│     │       └─ SpawnedComponent  │
│     │           └─ ChildActor    │  ← Part Actor (带 IGameplayTagAssetInterface)
│     │               Tags: [      │     Tag 示例: "Cosmetic.BodyStyle.Feminine"
│     │                 "Cosmetic. │
│     │                  BodyStyle.│
│     │                  Feminine" │
│     │               ]            │
│     │                            │
│     ├─ CollectCombinedTags()     │  ← 遍历所有 Part Actor 收集 Tag
│     │   → FGameplayTagContainer  │
│     │                            │
│     ├─ BodyMeshes.SelectBest     │  ← 根据 Tag 选择 SKM_Manny/SKM_Quinn
│     │   BodyStyle()              │
│     │   → SetSkeletalMesh()      │
│     │                            │
│     └─ OnCharacterPartsChanged   │  ← 广播变更通知
│         .Broadcast()             │
└──────────────────────────────────┘
           │
           │  CosmeticTags 传递给武器系统
           ▼
┌──────────────────────────────────┐
│  ULyraWeaponInstance             │
│  PickBestAnimLayer(bEquipped,    │
│    CosmeticTags)                 │
│  → EquippedAnimSet.SelectBest    │
│    Layer(CosmeticTags)           │
│  → TSubclassOf<UAnimInstance>    │
│                                  │
│  蓝图端:                          │
│  LinkAnimClassLayers(LayerClass) │
└──────────────────────────────────┘
```

---

## 5. 装备系统与动画层的联动

### 5.1 装备定义与实例

**装备定义** (`Source/LyraGame/Equipment/LyraEquipmentDefinition.h`):

```cpp
UCLASS(Blueprintable, Const, Abstract, BlueprintType)
class ULyraEquipmentDefinition : public UObject
{
    // 装备实例类（武器为 ULyraWeaponInstance）
    UPROPERTY(EditDefaultsOnly, Category=Equipment)
    TSubclassOf<ULyraEquipmentInstance> InstanceType;

    // 授予的能力集（开火、换弹、近战等）
    UPROPERTY(EditDefaultsOnly, Category=Equipment)
    TArray<TObjectPtr<const ULyraAbilitySet>> AbilitySetsToGrant;

    // 要生成的 Actor（武器模型）
    UPROPERTY(EditDefaultsOnly, Category=Equipment)
    TArray<FLyraEquipmentActorToSpawn> ActorsToSpawn;
};

// 待生成的 Actor 定义
USTRUCT()
struct FLyraEquipmentActorToSpawn
{
    TSubclassOf<AActor> ActorToSpawn;  // 武器 Actor 类
    FName AttachSocket;                 // 挂载点（如 "weapon_r"）
    FTransform AttachTransform;         // 挂载变换
};
```

**装备实例** (`Source/LyraGame/Equipment/LyraEquipmentInstance.h/cpp`):

```cpp
UCLASS(BlueprintType, Blueprintable)
class ULyraEquipmentInstance : public UObject
{
    virtual void SpawnEquipmentActors(const TArray<FLyraEquipmentActorToSpawn>& ActorsToSpawn);
    virtual void OnEquipped();     // 装备时回调（C++ 端调用 K2_OnEquipped 蓝图事件）
    virtual void OnUnequipped();   // 卸装时回调
};
```

**武器 Actor 挂载**:
```cpp
void ULyraEquipmentInstance::SpawnEquipmentActors(...)
{
    USceneComponent* AttachTarget = OwningPawn->GetRootComponent();
    if (ACharacter* Char = Cast<ACharacter>(OwningPawn))
    {
        AttachTarget = Char->GetMesh();  // 优先挂载到骨骼网格组件
    }
    NewActor->AttachToComponent(AttachTarget, 
        FAttachmentTransformRules::KeepRelativeTransform, SpawnInfo.AttachSocket);
}
```

### 5.2 武器装备触发动画层切换的完整流程

```
步骤1: 玩家切换武器
    ↓
步骤2: ULyraEquipmentManagerComponent::EquipItem()
    │  → 创建 ULyraWeaponInstance
    │  → 授予 AbilitySets
    │  → SpawnEquipmentActors() (生成武器 Actor 并挂载到 Mesh)
    │  → 调用 OnEquipped()
    ↓
步骤3: ULyraWeaponInstance::OnEquipped()
    │  → 调用 K2_OnEquipped() (蓝图可实现事件)
    ↓
步骤4: 蓝图端（K2_OnEquipped 中）
    │  → 获取角色的 CosmeticTags
    │  → 调用 PickBestAnimLayer(bEquipped=true, CosmeticTags)
    │  → 返回匹配的 AnimLayer 类（如 ABP_PistolAnimLayers_Feminine）
    ↓
步骤5: 蓝图端
    │  → 获取角色 Mesh 的 AnimInstance
    │  → 调用 LinkAnimClassLayers(选中的 AnimLayer 类)
    ↓
步骤6: 引擎层
    │  → 主 ABP 中 ALI_ItemAnimLayers 接口的 Linked Layer 节点
    │     被替换为新的 AnimLayer 实例的实现
    ↓
步骤7: 角色动画立即切换为新武器的持枪姿势、移动动画等

------- 卸装时反向流程 -------

步骤A: ULyraEquipmentManagerComponent::UnequipItem()
    │  → 回收 AbilitySets
    │  → DestroyEquipmentActors() (销毁武器 Actor)
    │  → 调用 OnUnequipped()
    ↓
步骤B: 蓝图端 K2_OnUnequipped 中
    │  → PickBestAnimLayer(bEquipped=false, CosmeticTags)
    │  → 返回 UneuippedAnimSet 中匹配的层（通常是 ABP_UnarmedAnimLayers）
    │  → LinkAnimClassLayers(无武器动画层)
```

---

## 6. GAS（能力系统）与动画的集成

### 6.1 GameplayTag 自动映射

`ULyraAnimInstance` 使用 `FGameplayTagBlueprintPropertyMap` 实现了声明式的 Tag-to-Variable 映射：

```
┌──────────────────────────┐    自动同步    ┌──────────────────────────┐
│  GAS (AbilitySystem)     │ ────────────→  │  AnimBP Variables        │
│                          │                │                          │
│  Tag: Status.Crouching   │ ──→            │  bool bIsCrouching       │
│  Tag: Status.ADS         │ ──→            │  bool bIsADS             │
│  Tag: Status.Jumping     │ ──→            │  bool bIsJumping         │
│  Tag: Status.Firing      │ ──→            │  bool bIsFiring          │
│  Tag: Movement.Mode.*    │ ──→            │  enum MovementMode       │
└──────────────────────────┘                └──────────────────────────┘
```

**优势**:
- 无需在 `NativeUpdateAnimation` 中手动查询每个 Tag
- 当 Tag 被添加/移除时变量自动更新，响应即时
- 在编辑器中可视化配置，避免硬编码

### 6.2 Montage 失败消息机制

Lyra 的能力系统通过 **GameplayMessage** 广播 Montage 播放失败事件：

```cpp
// LyraGameplayAbility.h
struct FLyraAbilityMontageFailureMessage
{
    TObjectPtr<UAnimMontage> FailureMontage = nullptr;
};

// 每个 Ability 可以配置 Tag -> 失败 Montage 的映射
UPROPERTY(EditDefaultsOnly, Category = "Advanced")
TMap<FGameplayTag, TObjectPtr<UAnimMontage>> FailureTagToAnimMontage;
```

当能力激活失败时（如弹药不足），通过 `TAG_ABILITY_PLAY_MONTAGE_FAILURE_MESSAGE` 将失败 Montage 广播出去，由 UI 系统或角色蓝图接收并播放。这是一种**完全解耦的事件驱动架构**。

---

## 7. 蓝图资产目录结构与命名规范

### 7.1 完整目录树

```
Content/Characters/Heroes/Mannequin/
├── Animations/                              (628 个资产)
│   ├── ABP_Mannequin_Base.uasset            ← 主动画蓝图
│   ├── ABP_Mannequin_CopyPose.uasset        ← Copy Pose
│   ├── ABP_Mannequin_Retarget.uasset        ← 重定向
│   ├── ABP_Mannequin_TopDown.uasset         ← 俯视角
│   ├── AnimEnum_CardinalDirection.uasset     ← 自定义枚举: 方向
│   ├── AnimEnum_RootYawOffsetMode.uasset     ← 自定义枚举: 根骨骼偏转模式
│   ├── AnimStruct_CardinalDirections.uasset  ← 自定义结构: 方向集
│   ├── AnimStruct_TurnInPlaceEntry.uasset    ← 自定义结构: 原地转身
│   │
│   ├── Actions/                              (91 个资产)
│   │   ├── AM_MM_Pistol_Melee.uasset         ← AnimMontage
│   │   ├── AM_MM_Rifle_Melee.uasset
│   │   ├── AM_MM_Shotgun_Melee.uasset
│   │   ├── AM_MM_Death_Back_01.uasset        ← 死亡动画
│   │   ├── AM_MM_Dash_Forward.uasset         ← 冲刺动画
│   │   ├── AM_MM_HitReact_Front_*.uasset     ← 受击反馈
│   │   ├── MM_Pistol_Fire.uasset             ← 开火
│   │   ├── MM_Pistol_Reload.uasset           ← 换弹
│   │   ├── MM_Pistol_Equip.uasset            ← 装备
│   │   ├── MM_*_Additive.uasset              ← 叠加版本
│   │   └── ...
│   │
│   ├── AimOffsets/                            (98 个资产)
│   │   ├── AO_MM_Pistol_Idle_ADS.uasset      ← 手枪瞄准偏移
│   │   ├── AO_MM_Rifle_Idle_ADS.uasset       ← 步枪瞄准偏移
│   │   ├── AO_MM_Rifle_Crouch_Idle.uasset    ← 步枪蹲伏偏移
│   │   ├── AO_MM_Rifle_Idle_Hipfire.uasset   ← 步枪腰射偏移
│   │   ├── AO_MM_Unarmed_Idle_Ready.uasset   ← 无武器偏移
│   │   └── MM_*_AO_*.uasset                  ← 各方向 Pose 资产
│   │
│   ├── LinkedLayers/
│   │   ├── ABP_ItemAnimLayersBase.uasset      ← 动画层基类
│   │   └── ALI_ItemAnimLayers.uasset          ← 动画层接口
│   │
│   ├── Locomotion/
│   │   ├── Pistol/    (107 资产, 含 2 个 ABP)
│   │   ├── Rifle/     (118 资产, 含 2 个 ABP)
│   │   ├── Shotgun/   (5 资产, 含 2 个 ABP, 共享步枪部分动画)
│   │   └── Unarmed/   (109 资产, 含 2 个 ABP)
│   │
│   ├── Interactions/   (28 个资产)
│   └── Poses/          (55 个资产)
│
├── Meshes/
│   ├── SK_Mannequin.uasset
│   ├── SKM_Manny.uasset / SKM_Manny_Invis.uasset
│   └── SKM_Quinn.uasset / SKM_Quinn_Invis.uasset
│
├── Rig/
│   ├── ABP_Manny_PostProcess.uasset
│   ├── ABP_Quinn_PostProcess.uasset
│   ├── CR_Mannequin_Body.uasset
│   ├── CR_Mannequin_FootPlant.uasset
│   ├── CR_Mannequin_Procedural.uasset
│   ├── IK_Mannequin.uasset
│   └── PA_Mannequin.uasset
│
└── Materials/ & Textures/ (材质/贴图，非动画相关)

Content/Weapons/
├── Pistol/Animations/
│   ├── ABP_Weap_Pistol.uasset               ← 手枪武器动画蓝图
│   ├── AM_Weap_Pistol_Fire.uasset           ← 武器开火 Montage
│   └── AM_Weap_Pistol_Reload.uasset         ← 武器换弹 Montage
├── Rifle/Animations/
│   ├── ABP_Weap_Rifle.uasset
│   └── ...
└── Shotgun/Animations/
    ├── ABP_Weap_Shotgun.uasset
    └── ...
```

### 7.2 动画资产命名规范

| 前缀/格式 | 含义 | 示例 |
|-----------|------|------|
| `ABP_` | Animation Blueprint（动画蓝图） | `ABP_Mannequin_Base` |
| `ALI_` | Anim Layer Interface（动画层接口） | `ALI_ItemAnimLayers` |
| `MM_` | **M**annequin **M**asculine（男性角色动画序列） | `MM_Pistol_Jog_Fwd` |
| `MF_` | **M**annequin **F**eminine（女性角色动画序列） | `MF_Pistol_Jog_Fwd` |
| `AM_MM_` | AnimMontage for Masculine | `AM_MM_Pistol_Melee` |
| `AM_MF_` | AnimMontage for Feminine | `AM_MF_Emote_FingerGuns` |
| `AO_MM_` | AimOffset for Masculine | `AO_MM_Pistol_Idle_ADS` |
| `AO_MF_` | AimOffset for Feminine | `AO_MF_Pistol_Idle_ADS` |
| `CR_` | Control Rig | `CR_Mannequin_Body` |
| `SK_` / `SKM_` | Skeleton / Skeletal Mesh | `SKM_Manny`, `SKM_Quinn` |
| `PA_` | Physics Asset | `PA_Mannequin` |
| `IK_` | IK Rig | `IK_Mannequin` |
| `RTG_` | Retarget | `RTG_Mannequin` |
| `_Additive` | 叠加动画版本 | `MM_Pistol_Melee_Additive` |
| `_Feminine` | 女性体型变体 | `ABP_PistolAnimLayers_Feminine` |

**动画序列的命名模板**:
```
{体型前缀}_{武器类型}_{动作类别}_{方向}_{状态}.uasset

示例: MM_Pistol_Jog_Fwd_Start.uasset
      ├ MM       = Masculine
      ├ Pistol   = 手枪
      ├ Jog      = 慢跑
      ├ Fwd      = 前方
      └ Start    = 起步
```

### 7.3 各武器动画资产清单

#### 手枪 (Pistol) — 以 Masculine 为例

| 动作类别 | 资产列表 |
|---------|---------|
| **站立空闲** | Idle_Hipfire, Idle_ADS, IdleBreak_Scan |
| **慢跑循环** | Jog_Fwd, Jog_Bwd, Jog_Left, Jog_Right |
| **慢跑起步** | Jog_Fwd_Start, Jog_Bwd_Start, Jog_Left_Start, Jog_Right_Start |
| **慢跑停步** | Jog_Fwd_Stop, Jog_Bwd_Stop, Jog_Left_Stop, Jog_Right_Stop, Jog_Stop |
| **慢跑转向** | Jog_Pivot_Fwd, Jog_Pivot_Bwd, Jog_Pivot_Left, Jog_Pivot_Right |
| **行走循环** | Walk_Fwd, Walk_Bwd, Walk_Left, Walk_Right |
| **行走起/停** | Walk_Fwd_Start/Stop, Walk_Bwd_Start/Stop, Walk_Left_Stop, Walk_Right_Start/Stop |
| **行走转向** | Walk_Fwd_Pivot, Walk_Bwd_Pivot, Walk_Left_Pivot, Walk_Right_Pivot |
| **蹲伏** | Crouch_Idle, Crouch_Entry, Crouch_Exit, Crouch_OverridePose |
| **蹲伏移动** | Crouch_Walk_Fwd/Bwd/Left/Right + Start/Stop/Pivot |
| **蹲伏转向** | Crouch_TurnLeft/Right_90/180 |
| **站立转向** | TurnLeft/Right_90/180 |
| **跳跃** | Jump_Start, Jump_Start_Loop, Jump_Apex, Jump_Fall_Loop, Jump_Fall_Land, Jump_RecoveryAdditive |
| **战斗动作** | Fire, DryFire(+Additive), Reload(+Additive), Equip(+Additive), Melee(+Additive), Spawn |

#### 霰弹枪 (Shotgun) — 特殊情况

霰弹枪的 Locomotion 目录只有 5 个资产（`Idle_Hipfire`, `Idle_ADS` 及 2 个 ABP），说明霰弹枪的大部分移动动画**共享了步枪的资产**，仅自定义了空闲姿态等关键差异动画。这体现了基于继承的动画复用策略。

---

## 8. 动画通知——上下文效果系统

**源码**: `Source/LyraGame/Feedback/ContextEffects/AnimNotify_LyraContextEffects.h/cpp`

`UAnimNotify_LyraContextEffects` 是一个自定义的 AnimNotify，用于在动画播放的特定时机触发上下文相关的效果（音效 + VFX）。

**数据结构**:

```cpp
UCLASS(const, hidecategories=Object, CollapseCategories, Config=Game, 
       meta=(DisplayName="Play Context Effects"))
class UAnimNotify_LyraContextEffects : public UAnimNotify
{
    // 效果类型标签（如 "ContextEffect.Footstep.Walk"）
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify")
    FGameplayTag Effect;

    // 位置/旋转偏移
    FVector LocationOffset;
    FRotator RotationOffset;

    // VFX 属性（缩放）
    FLyraContextEffectAnimNotifyVFXSettings VFXProperties;

    // 音效属性（音量/音高乘数）
    FLyraContextEffectAnimNotifyAudioSettings AudioProperties;

    // 是否附着到骨骼
    uint32 bAttached : 1;
    FName SocketName;

    // 是否执行射线检测
    uint32 bPerformTrace : 1;
    FLyraContextEffectAnimNotifyTraceSettings TraceProperties;
};
```

**执行流程**:
```
动画播放到 Notify 时刻
    ↓
AnimNotify_LyraContextEffects::Notify()
    ↓
如果 bPerformTrace:
    射线检测 → 获取表面物理材质 → 转换为 Context GameplayTag
    ↓
查找实现 ILyraContextEffectsInterface 的 Actor/Component
    ↓
调用 AnimMotionEffect 事件
    ↓
ContextEffectsLibrary 根据 Effect Tag + Surface Tag 查找
    ↓
播放匹配的音效 (USoundBase) + VFX (UNiagaraSystem)
```

**典型应用场景**:
- 脚步声随地面材质变化（木地板、金属、泥土、水面等）
- 脚步粒子效果（灰尘、水花等）
- 编辑器中可直接预览效果

---

## 9. 网络复制与多人同步

Lyra 的动画系统在多人同步方面使用了以下策略：

| 系统 | 复制方式 | 说明 |
|------|---------|------|
| **CharacterPartList** | `FFastArraySerializer` (`DOREPLIFETIME`) | Part 的增删改自动复制到客户端，客户端收到后生成 Part Actor 并触发 `BroadcastChanged` |
| **EquipmentList** | `FFastArraySerializer` (`DOREPLIFETIME`) | 装备的增删自动复制，客户端收到后触发 `OnEquipped`/`OnUnequipped` → 切换 AnimLayer |
| **EquipmentInstance** | `IsSupportedForNetworking() = true` | 装备实例支持网络复制，`Instigator` 和 `SpawnedActors` 使用 `DOREPLIFETIME` |
| **GameplayTag** | 通过 GAS 的内置复制 | GAS 的 Tag 变化会自动同步，触发 `GameplayTagPropertyMap` 更新动画变量 |
| **加速度优化** | `FLyraReplicatedAcceleration` | 压缩表示的加速度信息，使用 `uint8` 量化方向/幅度，减少带宽 |
| **快速共享复制** | `FSharedRepMovement` + `NetMulticast` | 非标准帧使用 multicast RPC 广播运动状态 |

---

## 10. 设计理念与架构优势总结

### 核心设计模式

| 模式 | 体现 |
|------|------|
| **策略模式 (Strategy)** | `FLyraAnimLayerSelectionSet` 封装了动画层选择算法，武器实例持有不同的策略配置 |
| **组合模式 (Composite)** | CosmeticTag 由多个 Part Actor 的 Tag 组合而成，整体行为由部分决定 |
| **观察者模式 (Observer)** | `OnCharacterPartsChanged` 委托通知所有关心外观变化的系统 |
| **中介者模式 (Mediator)** | 装备管理器作为中介协调装备实例、能力系统、动画系统的交互 |
| **数据驱动 (Data-Driven)** | 所有 AnimBP 引用、选择规则、武器配置都在数据资产/蓝图中配置，C++ 零硬编码 |

### 架构优势

1. **高度模块化**：添加新武器只需创建对应的 AnimLayer 蓝图并在武器数据资产中配置规则，无需修改任何现有代码
2. **体型适配**：通过 CosmeticTag 自动选择 Masculine/Feminine 动画变体，同一套武器定义适配所有体型
3. **网格与动画一致性**：身体网格选择和动画层选择使用相同的 CosmeticTag 数据源和相同的匹配算法
4. **蓝图友好**：`LinkAnimClassLayers` 调用在蓝图端，动画师和设计师可以独立调整动画层切换逻辑
5. **网络高效**：使用 `FFastArraySerializer` 进行差量复制，仅同步变更部分
6. **GAS 深度集成**：避免了传统的 "每帧轮询 Tag" 反模式，使用声明式映射实现自动同步
7. **可测试性**：动画测试助手类 (`FShooterTestsAnimationTestHelper`) 提供了完整的自动化测试能力
8. **关注点分离**：
   - C++ 层：选择逻辑、数据结构、GAS 集成
   - 蓝图层：动画逻辑、状态机、层链接调用
   - 资产层：具体动画资源、规则配置

### 扩展新武器的步骤

```
1. 创建武器对应的动画序列资产（MM_NewWeapon_*）
2. （可选）创建 Feminine 变体动画资产（MF_NewWeapon_*）
3. 创建 ABP_NewWeaponAnimLayers（继承 ABP_ItemAnimLayersBase，实现 ALI_ItemAnimLayers）
4. （可选）创建 ABP_NewWeaponAnimLayers_Feminine
5. 在武器的 ULyraWeaponInstance 蓝图中配置：
   - EquippedAnimSet.LayerRules 添加 Feminine Tag → Feminine ABP
   - EquippedAnimSet.DefaultLayer = ABP_NewWeaponAnimLayers
   - UneuippedAnimSet 类似配置（通常指向 Unarmed）
6. 完成！无需修改主 ABP 或任何 C++ 代码
```

---

> **总结**：Lyra 的动画蓝图分层架构是 UE5 Linked Anim Layers 特性的典范实现，通过 "主 ABP → 动画层接口 → 武器特定 AnimLayer" 的三层结构，配合 CosmeticTag 驱动的动态选择机制，实现了一套高度可扩展、数据驱动、网络友好的角色动画系统。
