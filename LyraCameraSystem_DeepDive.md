# Lyra Camera System（相机系统）— 代码级深度分析

## 目录

1. [系统总览与架构设计](#1-系统总览与架构设计)
2. [核心类关系图](#2-核心类关系图)
3. [ULyraCameraMode — 相机模式基类](#3-ulyracameramode--相机模式基类)
4. [ULyraCameraModeStack — 相机模式栈](#4-ulyracameramodestacke--相机模式栈)
5. [ULyraCameraComponent — 相机组件](#5-ulyracameracomponent--相机组件)
6. [ULyraCameraMode_ThirdPerson — 第三人称相机模式](#6-ulyracameramode_thirdperson--第三人称相机模式)
7. [穿透避免系统（Penetration Avoidance）](#7-穿透避免系统penetration-avoidance)
8. [ALyraPlayerCameraManager — 玩家相机管理器](#8-alyraplayercameramanager--玩家相机管理器)
9. [ULyraUICameraManagerComponent — UI 相机管理组件](#9-ulyrauicameramanagercomponent--ui-相机管理组件)
10. [ILyraCameraAssistInterface — 相机辅助接口](#10-ilyracameraassistinterface--相机辅助接口)
11. [相机模式的确定与切换流程](#11-相机模式的确定与切换流程)
12. [GAS 能力系统与相机模式联动](#12-gas-能力系统与相机模式联动)
13. [ULyraCameraMode_TopDownArenaCamera — 俯视角竞技场相机](#13-ulyracameramode_topdownarenacamera--俯视角竞技场相机)
14. [ALyraDebugCameraController — 调试相机控制器](#14-alyradebugcameracontroller--调试相机控制器)
15. [数据驱动配置（ULyraPawnData）](#15-数据驱动配置ulyrapawndata)
16. [完整调用流程时序](#16-完整调用流程时序)
17. [关键设计模式总结](#17-关键设计模式总结)
18. [源码文件清单](#18-源码文件清单)

---

## 1. 系统总览与架构设计

Lyra 的相机系统是一个 **基于模式栈（CameraMode Stack）的可混合相机框架**，核心设计理念是：

- **模式驱动**：每种游戏状态对应一个 `ULyraCameraMode` 子类（第三人称、俯视角、瞄准镜等）
- **栈式混合**：通过 `ULyraCameraModeStack` 管理多个模式的堆叠和权重混合
- **委托解耦**：通过 `FLyraCameraModeDelegate` 委托决定当前应使用的相机模式，而非硬编码
- **数据驱动**：默认相机模式配置在 `ULyraPawnData` 数据资产中，能力可动态覆盖
- **不使用 SpringArm**：Lyra 完全放弃了 UE 的 `USpringArmComponent`，而是用自定义的多射线穿透避免系统

### 整体数据流

```
PlayerCameraManager::UpdateViewTarget()
  └─→ UCameraComponent::GetCameraView()  [多态: ULyraCameraComponent]
        ├─→ UpdateCameraModes()           // 通过委托确定当前 CameraMode
        │     └─→ DetermineCameraModeDelegate.Execute()
        │           └─→ ULyraHeroComponent::DetermineCameraMode()
        │                 ├─→ AbilityCameraMode (如果能力设置了相机模式)
        │                 └─→ PawnData->DefaultCameraMode (默认模式)
        ├─→ CameraModeStack->EvaluateStack()
        │     ├─→ UpdateStack()           // 更新每个模式的 View 和 BlendWeight
        │     └─→ BlendStack()            // 从栈底向栈顶混合所有模式的输出
        └─→ 输出 FMinimalViewInfo → 引擎渲染
```

---

## 2. 核心类关系图

```
APlayerCameraManager
  └── ALyraPlayerCameraManager
        ├── ULyraUICameraManagerComponent (UICamera)
        └── UpdateViewTarget() → 委托给下层

ULyraCameraComponent (挂在 ALyraCharacter 上)
  ├── ULyraCameraModeStack (CameraModeStack)
  │     ├── TArray<ULyraCameraMode*> CameraModeInstances  // 对象池
  │     └── TArray<ULyraCameraMode*> CameraModeStack      // 活跃栈
  ├── FLyraCameraModeDelegate DetermineCameraModeDelegate  // 绑到 HeroComponent
  └── GetCameraView() → 主更新入口

ULyraCameraMode (抽象基类, UObject)
  ├── FLyraCameraModeView View       // 输出: Location + Rotation + ControlRotation + FOV
  ├── BlendWeight / BlendAlpha       // 混合权重
  ├── UpdateView() / UpdateBlending()
  ├── ULyraCameraMode_ThirdPerson    // 第三人称实现
  │     ├── 曲线驱动的偏移
  │     ├── 蹲伏偏移
  │     └── 穿透避免（多 Feeler 射线）
  └── ULyraCameraMode_TopDownArenaCamera  // 俯视角实现（GameFeature 插件）

ULyraHeroComponent
  ├── DetermineCameraMode()          // 确定当前相机模式
  ├── SetAbilityCameraMode()         // 能力覆盖相机模式
  └── ClearAbilityCameraMode()       // 清除能力相机覆盖

ULyraGameplayAbility
  ├── SetCameraMode()                // 蓝图可调用
  ├── ClearCameraMode()              // EndAbility 时自动清除
  └── ActiveCameraMode               // 当前能力设定的相机模式类

ULyraPawnData (DataAsset)
  └── DefaultCameraMode              // 默认相机模式类

ILyraCameraAssistInterface           // 穿透避免辅助接口
FLyraPenetrationAvoidanceFeeler      // 穿透检测射线结构
```

---

## 3. ULyraCameraMode — 相机模式基类

> 源码: `Source/LyraGame/Camera/LyraCameraMode.h/.cpp`

### 3.1 类定义

`ULyraCameraMode` 继承自 `UObject`（而非 `UActorComponent`），它是一个纯逻辑对象，Outer 是 `ULyraCameraComponent`。

```cpp
// LyraCameraMode.h
UCLASS(MinimalAPI, Abstract, NotBlueprintable)
class ULyraCameraMode : public UObject
```

**关键设计决策**：使用 UObject 而非 Component，避免了 Actor 组件的 Tick 开销，也使得模式可以作为轻量级对象在栈中自由创建和管理。

### 3.2 FLyraCameraModeView — 输出数据结构

```cpp
// LyraCameraMode.h:45-59
struct FLyraCameraModeView
{
    FVector Location;          // 相机世界位置
    FRotator Rotation;         // 相机世界旋转
    FRotator ControlRotation;  // 控制器旋转（通常 == Rotation，但可分离）
    float FieldOfView;         // 视场角
};
```

混合逻辑在 `Blend()` 方法中：

```cpp
// LyraCameraMode.cpp:25-46
void FLyraCameraModeView::Blend(const FLyraCameraModeView& Other, float OtherWeight)
{
    if (OtherWeight <= 0.0f) return;
    if (OtherWeight >= 1.0f) { *this = Other; return; }

    Location = FMath::Lerp(Location, Other.Location, OtherWeight);

    // 旋转使用差值归一化后的加权混合（避免万向锁问题）
    const FRotator DeltaRotation = (Other.Rotation - Rotation).GetNormalized();
    Rotation = Rotation + (OtherWeight * DeltaRotation);

    const FRotator DeltaControlRotation = (Other.ControlRotation - ControlRotation).GetNormalized();
    ControlRotation = ControlRotation + (OtherWeight * DeltaControlRotation);

    FieldOfView = FMath::Lerp(FieldOfView, Other.FieldOfView, OtherWeight);
}
```

**要点**：
- `Location` 和 `FieldOfView` 使用简单线性插值
- `Rotation` 和 `ControlRotation` 使用差值归一化后的加权混合，确保最短路径旋转

### 3.3 核心属性

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `FieldOfView` | float | 80.0° | 水平视场角 |
| `ViewPitchMin` | float | -89.0° | 俯仰最小角度 |
| `ViewPitchMax` | float | 89.0° | 俯仰最大角度 |
| `BlendTime` | float | 0.5s | 混合过渡时间 |
| `BlendFunction` | ELyraCameraModeBlendFunction | EaseOut | 混合曲线类型 |
| `BlendExponent` | float | 4.0 | 曲线指数 |
| `CameraTypeTag` | FGameplayTag | — | 用于 Gameplay 查询的类型标签 |
| `bResetInterpolation` | bool | false | 是否跳过插值直接到位 |

### 3.4 混合函数枚举

```cpp
// LyraCameraMode.h:22-37
UENUM(BlueprintType)
enum class ELyraCameraModeBlendFunction : uint8
{
    Linear,     // 线性插值
    EaseIn,     // 立即加速，平滑减速
    EaseOut,    // 平滑加速，立即到达
    EaseInOut,  // 平滑加速减速
    COUNT
};
```

### 3.5 更新流程

`UpdateCameraMode()` 是每帧被调用的主入口：

```cpp
// LyraCameraMode.cpp:127-131
void ULyraCameraMode::UpdateCameraMode(float DeltaTime)
{
    UpdateView(DeltaTime);      // 1. 计算本帧的 View 数据
    UpdateBlending(DeltaTime);  // 2. 更新混合权重
}
```

#### GetPivotLocation() — 枢轴位置计算

```cpp
// LyraCameraMode.cpp:82-112
FVector ULyraCameraMode::GetPivotLocation() const
{
    const AActor* TargetActor = GetTargetActor();

    if (const ACharacter* TargetCharacter = Cast<ACharacter>(TargetPawn))
    {
        // 关键：使用 CDO 的默认半高和 BaseEyeHeight 来计算高度
        // 这样蹲伏时会自动调整
        const float DefaultHalfHeight = CapsuleCompCDO->GetUnscaledCapsuleHalfHeight();
        const float ActualHalfHeight = CapsuleComp->GetUnscaledCapsuleHalfHeight();
        const float HeightAdjustment = (DefaultHalfHeight - ActualHalfHeight) + TargetCharacterCDO->BaseEyeHeight;

        return TargetCharacter->GetActorLocation() + (FVector::UpVector * HeightAdjustment);
    }

    // 非 Character 的 Pawn 使用 GetPawnViewLocation()
    return TargetPawn->GetPawnViewLocation();
}
```

**关键设计**：通过比较 CDO（Class Default Object）的默认半高与当前实际半高的差值，加上 CDO 的 `BaseEyeHeight`，实现了蹲伏时的平滑高度调整，而不依赖角色自身的 `GetPawnViewLocation()`。

#### UpdateView() — 基类默认实现

```cpp
// LyraCameraMode.cpp:133-144
void ULyraCameraMode::UpdateView(float DeltaTime)
{
    FVector PivotLocation = GetPivotLocation();
    FRotator PivotRotation = GetPivotRotation();

    // 俯仰角钳制
    PivotRotation.Pitch = FMath::ClampAngle(PivotRotation.Pitch, ViewPitchMin, ViewPitchMax);

    View.Location = PivotLocation;
    View.Rotation = PivotRotation;
    View.ControlRotation = View.Rotation;
    View.FieldOfView = FieldOfView;
}
```

基类默认实现是一个第一人称视角（位置 = 枢轴位置，旋转 = 控制器视角旋转），子类通过 override 实现不同的相机行为。

#### UpdateBlending() — 混合权重更新

```cpp
// LyraCameraMode.cpp:177-213
void ULyraCameraMode::UpdateBlending(float DeltaTime)
{
    // BlendAlpha 随时间线性增长到 1.0
    if (BlendTime > 0.0f)
    {
        BlendAlpha += (DeltaTime / BlendTime);
        BlendAlpha = FMath::Min(BlendAlpha, 1.0f);
    }
    else
    {
        BlendAlpha = 1.0f;
    }

    // 然后通过 BlendFunction 将 BlendAlpha 映射到 BlendWeight
    switch (BlendFunction)
    {
    case Linear:   BlendWeight = BlendAlpha; break;
    case EaseIn:   BlendWeight = FMath::InterpEaseIn(0, 1, BlendAlpha, Exponent); break;
    case EaseOut:  BlendWeight = FMath::InterpEaseOut(0, 1, BlendAlpha, Exponent); break;
    case EaseInOut: BlendWeight = FMath::InterpEaseInOut(0, 1, BlendAlpha, Exponent); break;
    }
}
```

**两级映射**：
1. `BlendAlpha`：线性时间归一化 `[0, 1]`，表示已经过去了多少时间占 BlendTime 的比例
2. `BlendWeight`：通过缓动函数映射后的实际权重，控制混合效果的曲线形状

#### SetBlendWeight() — 逆向映射

```cpp
// LyraCameraMode.cpp:146-175
void ULyraCameraMode::SetBlendWeight(float Weight)
{
    BlendWeight = FMath::Clamp(Weight, 0.0f, 1.0f);

    // 逆向计算 BlendAlpha（使用逆指数）
    const float InvExponent = (BlendExponent > 0.0f) ? (1.0f / BlendExponent) : 1.0f;

    switch (BlendFunction)
    {
    case Linear:   BlendAlpha = BlendWeight; break;
    case EaseIn:   BlendAlpha = FMath::InterpEaseIn(0, 1, BlendWeight, InvExponent); break;
    case EaseOut:  BlendAlpha = FMath::InterpEaseOut(0, 1, BlendWeight, InvExponent); break;
    case EaseInOut: BlendAlpha = FMath::InterpEaseInOut(0, 1, BlendWeight, InvExponent); break;
    }
}
```

当栈中重新插入一个已存在的模式时，需要用 `SetBlendWeight()` 设置初始权重，此时需要逆向计算对应的 `BlendAlpha`，保证后续 `UpdateBlending()` 能正确继续。

---

## 4. ULyraCameraModeStack — 相机模式栈

> 源码: `Source/LyraGame/Camera/LyraCameraMode.h/.cpp`（同文件内定义）

### 4.1 类定义

```cpp
// LyraCameraMode.h:162-201
UCLASS()
class ULyraCameraModeStack : public UObject
{
    UPROPERTY()
    TArray<TObjectPtr<ULyraCameraMode>> CameraModeInstances;  // 对象池
    UPROPERTY()
    TArray<TObjectPtr<ULyraCameraMode>> CameraModeStack;      // 活跃模式栈
    bool bIsActive;
};
```

**双数组设计**：
- `CameraModeInstances`：对象池，缓存已创建的 CameraMode 实例，避免反复 `NewObject`
- `CameraModeStack`：活跃栈，栈顶（Index=0）是优先级最高的模式

### 4.2 GetCameraModeInstance() — 对象池获取

```cpp
// LyraCameraMode.cpp:343-363
ULyraCameraMode* ULyraCameraModeStack::GetCameraModeInstance(TSubclassOf<ULyraCameraMode> CameraModeClass)
{
    // 先从池中查找
    for (ULyraCameraMode* CameraMode : CameraModeInstances)
    {
        if (CameraMode && CameraMode->GetClass() == CameraModeClass)
            return CameraMode;
    }

    // 未找到则创建新实例（Outer 是 ULyraCameraComponent）
    ULyraCameraMode* NewCameraMode = NewObject<ULyraCameraMode>(GetOuter(), CameraModeClass);
    CameraModeInstances.Add(NewCameraMode);
    return NewCameraMode;
}
```

### 4.3 PushCameraMode() — 压入新模式（核心算法）

这是整个相机系统中最重要的方法之一：

```cpp
// LyraCameraMode.cpp:264-328
void ULyraCameraModeStack::PushCameraMode(TSubclassOf<ULyraCameraMode> CameraModeClass)
{
    ULyraCameraMode* CameraMode = GetCameraModeInstance(CameraModeClass);

    // 1. 如果已经在栈顶，直接返回
    if (CameraModeStack[0] == CameraMode) return;

    // 2. 如果已在栈中，计算其当前贡献度并移除
    float ExistingStackContribution = 1.0f;
    for (int32 StackIndex = 0; StackIndex < StackSize; ++StackIndex)
    {
        if (CameraModeStack[StackIndex] == CameraMode)
        {
            ExistingStackContribution *= CameraMode->GetBlendWeight();
            break;
        }
        else
        {
            // 上层模式"遮挡"了下层的部分贡献
            ExistingStackContribution *= (1.0f - CameraModeStack[StackIndex]->GetBlendWeight());
        }
    }

    // 3. 决定初始权重
    const bool bShouldBlend = (CameraMode->GetBlendTime() > 0.0f) && (StackSize > 0);
    const float BlendWeight = bShouldBlend ? ExistingStackContribution : 1.0f;
    CameraMode->SetBlendWeight(BlendWeight);

    // 4. 插入栈顶
    CameraModeStack.Insert(CameraMode, 0);

    // 5. 栈底始终 100% 权重
    CameraModeStack.Last()->SetBlendWeight(1.0f);

    // 6. 新模式首次入栈时触发 OnActivation
    if (ExistingStackIndex == INDEX_NONE)
        CameraMode->OnActivation();
}
```

**贡献度算法详解**：

当模式 A 在栈中位于 Index=2 时，其对最终混合结果的实际贡献度是：
```
Contribution(A) = (1 - Weight[0]) × (1 - Weight[1]) × Weight[A]
```
这正是上述循环计算的逻辑。当 A 被重新推到栈顶时，使用这个贡献度作为初始权重，确保切换时不会出现突变。

### 4.4 UpdateStack() — 栈更新与清理

```cpp
// LyraCameraMode.cpp:365-405
void ULyraCameraModeStack::UpdateStack(float DeltaTime)
{
    int32 RemoveCount = 0, RemoveIndex = INDEX_NONE;

    for (int32 StackIndex = 0; StackIndex < StackSize; ++StackIndex)
    {
        CameraModeStack[StackIndex]->UpdateCameraMode(DeltaTime);

        // 一旦某模式权重达到 100%，其下方的所有模式都不再可见
        if (CameraMode->GetBlendWeight() >= 1.0f)
        {
            RemoveIndex = StackIndex + 1;
            RemoveCount = StackSize - RemoveIndex;
            break;
        }
    }

    // 移除被完全遮盖的底层模式
    if (RemoveCount > 0)
    {
        for (int32 i = RemoveIndex; i < StackSize; ++i)
            CameraModeStack[i]->OnDeactivation();

        CameraModeStack.RemoveAt(RemoveIndex, RemoveCount);
    }
}
```

**自动栈清理**：当栈顶模式的 BlendWeight 达到 1.0 后，下方所有模式被移除并触发 `OnDeactivation()`。这防止了栈无限增长。

### 4.5 BlendStack() — 栈混合算法

```cpp
// LyraCameraMode.cpp:407-428
void ULyraCameraModeStack::BlendStack(FLyraCameraModeView& OutCameraModeView) const
{
    // 从栈底开始（权重永远是 1.0）
    OutCameraModeView = CameraModeStack[StackSize - 1]->GetCameraModeView();

    // 依次向栈顶混合
    for (int32 StackIndex = StackSize - 2; StackIndex >= 0; --StackIndex)
    {
        const ULyraCameraMode* CameraMode = CameraModeStack[StackIndex];
        OutCameraModeView.Blend(CameraMode->GetCameraModeView(), CameraMode->GetBlendWeight());
    }
}
```

**混合顺序**：从栈底到栈顶逐步覆盖。栈底的权重始终为 1.0（完全不透明底层），每层向上用自己的 BlendWeight 与下面混合后的结果做 Lerp。最终结果由栈顶（最新推入的模式）主导。

混合数学等价于：
```
Result = Stack[N-1].View
for i = N-2 downto 0:
    Result = Lerp(Result, Stack[i].View, Stack[i].BlendWeight)
```

### 4.6 GetBlendInfo() — 顶层信息查询

```cpp
// LyraCameraMode.cpp:449-464
void ULyraCameraModeStack::GetBlendInfo(float& OutWeightOfTopLayer, FGameplayTag& OutTagOfTopLayer) const
{
    if (CameraModeStack.Num() == 0) { ... return; }

    ULyraCameraMode* TopEntry = CameraModeStack.Last();
    OutWeightOfTopLayer = TopEntry->GetBlendWeight();
    OutTagOfTopLayer = TopEntry->GetCameraTypeTag();
}
```

注意这里用 `Last()`（栈底），这与直觉相反。`GetBlendInfo` 返回的是「**底层基础模式**」的信息，用于 Gameplay 系统查询当前基础相机状态（如 ADS 模式）。

---

## 5. ULyraCameraComponent — 相机组件

> 源码: `Source/LyraGame/Camera/LyraCameraComponent.h/.cpp`

### 5.1 类定义

```cpp
// LyraCameraComponent.h:27-70
UCLASS()
class ULyraCameraComponent : public UCameraComponent
{
    // 委托：询问外部"当前应该用哪个 CameraMode？"
    FLyraCameraModeDelegate DetermineCameraModeDelegate;

    // CameraMode 栈
    UPROPERTY()
    TObjectPtr<ULyraCameraModeStack> CameraModeStack;

    // FOV 偏移（单帧有效）
    float FieldOfViewOffset;
};
```

**委托定义**：
```cpp
DECLARE_DELEGATE_RetVal(TSubclassOf<ULyraCameraMode>, FLyraCameraModeDelegate);
```

### 5.2 初始化

```cpp
// LyraCameraComponent.cpp:21-29
void ULyraCameraComponent::OnRegister()
{
    Super::OnRegister();

    if (!CameraModeStack)
    {
        // CameraModeStack 的 Outer 是 this（ULyraCameraComponent）
        CameraModeStack = NewObject<ULyraCameraModeStack>(this);
    }
}
```

### 5.3 GetCameraView() — 主更新入口

这是引擎每帧调用的核心方法：

```cpp
// LyraCameraComponent.cpp:32-83
void ULyraCameraComponent::GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView)
{
    // 1. 确定并推入当前 CameraMode
    UpdateCameraModes();

    // 2. 执行栈求值（更新所有模式 + 混合）
    FLyraCameraModeView CameraModeView;
    CameraModeStack->EvaluateStack(DeltaTime, CameraModeView);

    // 3. 同步 PlayerController 的 ControlRotation
    if (APawn* TargetPawn = Cast<APawn>(GetTargetActor()))
    {
        if (APlayerController* PC = TargetPawn->GetController<APlayerController>())
        {
            PC->SetControlRotation(CameraModeView.ControlRotation);
        }
    }

    // 4. 应用 FOV 偏移（单帧有效）
    CameraModeView.FieldOfView += FieldOfViewOffset;
    FieldOfViewOffset = 0.0f;

    // 5. 同步组件自身的 Transform
    SetWorldLocationAndRotation(CameraModeView.Location, CameraModeView.Rotation);
    FieldOfView = CameraModeView.FieldOfView;

    // 6. 填充引擎需要的 FMinimalViewInfo
    DesiredView.Location = CameraModeView.Location;
    DesiredView.Rotation = CameraModeView.Rotation;
    DesiredView.FOV = CameraModeView.FieldOfView;
    DesiredView.OrthoWidth = OrthoWidth;
    DesiredView.OrthoNearClipPlane = OrthoNearClipPlane;
    DesiredView.OrthoFarClipPlane = OrthoFarClipPlane;
    DesiredView.AspectRatio = AspectRatio;
    DesiredView.bConstrainAspectRatio = bConstrainAspectRatio;
    DesiredView.bUseFieldOfViewForLOD = bUseFieldOfViewForLOD;
    DesiredView.ProjectionMode = ProjectionMode;

    // 7. PostProcess 混合
    DesiredView.PostProcessBlendWeight = PostProcessBlendWeight;
    if (PostProcessBlendWeight > 0.0f)
        DesiredView.PostProcessSettings = PostProcessSettings;

    // 8. XR 头部追踪特殊处理
    if (IsXRHeadTrackedCamera())
        Super::GetCameraView(DeltaTime, DesiredView);
}
```

**关键点**：
- 步骤 3 中同步 `ControlRotation` 非常重要 — 这确保了角色移动方向与相机视角一致
- 步骤 4 中 `FieldOfViewOffset` 是一个单帧临时偏移，用于武器开火时的 FOV 效果（由外部调用 `AddFieldOfViewOffset()`）
- 步骤 8 中对 XR 的兜底处理，确保 VR 模式也能正常工作

### 5.4 UpdateCameraModes() — 模式确定

```cpp
// LyraCameraComponent.cpp:85-99
void ULyraCameraComponent::UpdateCameraModes()
{
    if (CameraModeStack->IsStackActivate())
    {
        if (DetermineCameraModeDelegate.IsBound())
        {
            if (const TSubclassOf<ULyraCameraMode> CameraMode = DetermineCameraModeDelegate.Execute())
            {
                CameraModeStack->PushCameraMode(CameraMode);
            }
        }
    }
}
```

每帧都会通过委托询问当前需要什么相机模式。如果模式不变（已在栈顶），`PushCameraMode()` 内部会直接返回。

### 5.5 FOV 偏移接口

```cpp
// LyraCameraComponent.h:47
void AddFieldOfViewOffset(float FovOffset) { FieldOfViewOffset += FovOffset; }
```

这是一个累加式接口，可以在单帧内被多个来源调用（如后坐力、技能效果等），最终在 `GetCameraView()` 中被消费并清零。

---

## 6. ULyraCameraMode_ThirdPerson — 第三人称相机模式

> 源码: `Source/LyraGame/Camera/LyraCameraMode_ThirdPerson.h/.cpp`

### 6.1 类定义

```cpp
// LyraCameraMode_ThirdPerson.h:18-114
UCLASS(Abstract, Blueprintable)
class ULyraCameraMode_ThirdPerson : public ULyraCameraMode
```

**注意 `Blueprintable`**：与基类的 `NotBlueprintable` 不同，第三人称模式允许蓝图子类化，用于配置不同的偏移曲线和碰撞参数。

### 6.2 核心属性

#### 偏移曲线系统

```cpp
// LyraCameraMode_ThirdPerson.h:40-55
// 方式一：使用 CurveVector 资产（三维偏移随 Pitch 变化）
UPROPERTY(EditDefaultsOnly, Meta = (EditCondition = "!bUseRuntimeFloatCurves"))
TObjectPtr<const UCurveVector> TargetOffsetCurve;

// 方式二：使用 RuntimeFloatCurve（支持运行时编辑）
UPROPERTY(EditDefaultsOnly)
bool bUseRuntimeFloatCurves;

UPROPERTY(EditDefaultsOnly, Meta = (EditCondition = "bUseRuntimeFloatCurves"))
FRuntimeFloatCurve TargetOffsetX;
FRuntimeFloatCurve TargetOffsetY;
FRuntimeFloatCurve TargetOffsetZ;
```

**双重曲线方案**：
- `TargetOffsetCurve`：UCurveVector 资产引用，一个曲线同时输出 XYZ 三个分量
- `RuntimeFloatCurve`：三个独立 RuntimeFloatCurve（因 UE-103986 Bug，PIE 模式下 LiveEdit 不生效，所以保留了资产方式）

两种方式都以 **ViewPitch（视角俯仰角）** 为输入参数，在本地空间计算偏移后旋转到世界空间：

```cpp
// LyraCameraMode_ThirdPerson.cpp:51-68
if (!bUseRuntimeFloatCurves)
{
    if (TargetOffsetCurve)
    {
        const FVector TargetOffset = TargetOffsetCurve->GetVectorValue(PivotRotation.Pitch);
        View.Location = PivotLocation + PivotRotation.RotateVector(TargetOffset);
    }
}
else
{
    FVector TargetOffset(0.0f);
    TargetOffset.X = TargetOffsetX.GetRichCurveConst()->Eval(PivotRotation.Pitch);
    TargetOffset.Y = TargetOffsetY.GetRichCurveConst()->Eval(PivotRotation.Pitch);
    TargetOffset.Z = TargetOffsetZ.GetRichCurveConst()->Eval(PivotRotation.Pitch);

    View.Location = PivotLocation + PivotRotation.RotateVector(TargetOffset);
}
```

#### 蹲伏偏移

```cpp
// LyraCameraMode_ThirdPerson.h:57-112
float CrouchOffsetBlendMultiplier = 5.0f;  // 蹲伏偏移混合速度

FVector InitialCrouchOffset;   // 蹲伏切换时的起始偏移
FVector TargetCrouchOffset;    // 目标偏移
float CrouchOffsetBlendPct;    // 当前混合进度 [0, 1]
FVector CurrentCrouchOffset;   // 当前实际偏移
```

```cpp
// LyraCameraMode_ThirdPerson.cpp:74-91
void ULyraCameraMode_ThirdPerson::UpdateForTarget(float DeltaTime)
{
    if (const ACharacter* TargetCharacter = Cast<ACharacter>(GetTargetActor()))
    {
        if (TargetCharacter->IsCrouched())
        {
            // 计算蹲伏高度差 = CrouchedEyeHeight - BaseEyeHeight（通常是负值）
            const float CrouchedHeightAdjustment =
                TargetCharacterCDO->CrouchedEyeHeight - TargetCharacterCDO->BaseEyeHeight;
            SetTargetCrouchOffset(FVector(0, 0, CrouchedHeightAdjustment));
            return;
        }
    }
    SetTargetCrouchOffset(FVector::ZeroVector);
}

// 蹲伏偏移使用 EaseInOut 平滑混合
void ULyraCameraMode_ThirdPerson::UpdateCrouchOffset(float DeltaTime)
{
    if (CrouchOffsetBlendPct < 1.0f)
    {
        CrouchOffsetBlendPct = FMath::Min(CrouchOffsetBlendPct + DeltaTime * CrouchOffsetBlendMultiplier, 1.0f);
        CurrentCrouchOffset = FMath::InterpEaseInOut(InitialCrouchOffset, TargetCrouchOffset, CrouchOffsetBlendPct, 1.0f);
    }
}
```

### 6.3 UpdateView() — 完整流程

```cpp
// LyraCameraMode_ThirdPerson.cpp:35-72
void ULyraCameraMode_ThirdPerson::UpdateView(float DeltaTime)
{
    UpdateForTarget(DeltaTime);           // 1. 更新蹲伏偏移目标
    UpdateCrouchOffset(DeltaTime);        // 2. 平滑插值蹲伏偏移

    FVector PivotLocation = GetPivotLocation() + CurrentCrouchOffset;  // 3. 基础位置 + 蹲伏偏移
    FRotator PivotRotation = GetPivotRotation();

    PivotRotation.Pitch = FMath::ClampAngle(PivotRotation.Pitch, ViewPitchMin, ViewPitchMax);

    View.Location = PivotLocation;
    View.Rotation = PivotRotation;
    View.ControlRotation = View.Rotation;
    View.FieldOfView = FieldOfView;

    // 4. 应用曲线偏移（基于 Pitch）
    // ... (曲线计算，见上文)

    // 5. 穿透避免
    UpdatePreventPenetration(DeltaTime);
}
```

---

## 7. 穿透避免系统（Penetration Avoidance）

Lyra **完全不使用 USpringArmComponent**，而是实现了一套自定义的多射线穿透检测系统。

### 7.1 FLyraPenetrationAvoidanceFeeler — 探测射线结构

> 源码: `Source/LyraGame/Camera/LyraPenetrationAvoidanceFeeler.h`

```cpp
// LyraPenetrationAvoidanceFeeler.h:13-65
USTRUCT()
struct FLyraPenetrationAvoidanceFeeler
{
    FRotator AdjustmentRot;    // 相对主射线的偏转角
    float WorldWeight;         // 命中世界几何体时的影响权重
    float PawnWeight;          // 命中 Pawn 时的影响权重
    float Extent;              // 球形检测半径
    int32 TraceInterval;       // 检测间隔帧数（性能优化）
    int32 FramesUntilNextTrace; // 距下次检测的剩余帧数
};
```

### 7.2 默认 Feeler 配置

```cpp
// LyraCameraMode_ThirdPerson.cpp:26-32
// 构造函数中初始化 7 条射线
PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
    FRotator(+0, +0, 0),   1.00f, 1.00f, 14.f, 0));   // 主射线：每帧检测，14cm 半径
PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
    FRotator(+0, +16, 0),  0.75f, 0.75f, 0.f,  3));    // 右偏 16°：每 3 帧，点检测
PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
    FRotator(+0, -16, 0),  0.75f, 0.75f, 0.f,  3));    // 左偏 16°
PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
    FRotator(+0, +32, 0),  0.50f, 0.50f, 0.f,  5));    // 右偏 32°：每 5 帧
PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
    FRotator(+0, -32, 0),  0.50f, 0.50f, 0.f,  5));    // 左偏 32°
PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
    FRotator(+20, +0, 0),  1.00f, 1.00f, 0.f,  4));    // 上偏 20°：每 4 帧
PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(
    FRotator(-20, +0, 0),  0.50f, 0.50f, 0.f,  4));    // 下偏 20°
```

**射线分布示意**：
```
        上偏 20°
          |
左32° 左16° [主] 右16° 右32°
          |
        下偏 20°
```

- **Index 0（主射线）**：每帧检测，命中时 **立即** 拉近相机（HardBlocked）
- **Index 1+（预测射线）**：按间隔帧检测，命中时 **平滑** 拉近（SoftBlocked）

### 7.3 碰撞参数

```cpp
// LyraCameraMode_ThirdPerson.h:62-82
float PenetrationBlendInTime = 0.1f;       // 拉近混合时间（快速）
float PenetrationBlendOutTime = 0.15f;      // 恢复混合时间（稍慢）
bool bPreventPenetration = true;            // 是否启用穿透避免
bool bDoPredictiveAvoidance = true;         // 是否启用预测避免
float CollisionPushOutDistance = 2.f;       // 碰撞推出距离
float ReportPenetrationPercent = 0.f;       // 触发穿透回调的阈值
```

### 7.4 UpdatePreventPenetration() — 穿透避免入口

```cpp
// LyraCameraMode_ThirdPerson.cpp:111-171
void ULyraCameraMode_ThirdPerson::UpdatePreventPenetration(float DeltaTime)
{
    // 1. 通过 ILyraCameraAssistInterface 获取穿透避免目标
    ILyraCameraAssistInterface* TargetControllerAssist = Cast<ILyraCameraAssistInterface>(TargetController);
    ILyraCameraAssistInterface* TargetActorAssist = Cast<ILyraCameraAssistInterface>(TargetActor);

    // 允许目标 Actor 自定义穿透检测的基准 Actor
    TOptional<AActor*> OptionalPPTarget = TargetActorAssist
        ? TargetActorAssist->GetCameraPreventPenetrationTarget()
        : TOptional<AActor*>();
    AActor* PPActor = OptionalPPTarget.IsSet() ? OptionalPPTarget.GetValue() : TargetActor;

    // 2. 计算安全位置（SafeLocation）
    FVector SafeLocation = PPActor->GetActorLocation();
    // 在瞄准线上找最近点，限制在胶囊体内
    FMath::PointDistToLine(SafeLocation, View.Rotation.Vector(), View.Location, ClosestPointOnLineToCapsuleCenter);
    float MaxHalfHeight = PPActor->GetSimpleCollisionHalfHeight() - PushInDistance;
    SafeLocation.Z = FMath::Clamp(ClosestPointOnLineToCapsuleCenter.Z,
        SafeLocation.Z - MaxHalfHeight, SafeLocation.Z + MaxHalfHeight);

    // 3. 推入胶囊体内部
    PPActorRootComponent->GetSquaredDistanceToCollision(ClosestPointOnLineToCapsuleCenter, DistanceSqr, SafeLocation);
    SafeLocation += (SafeLocation - ClosestPointOnLineToCapsuleCenter).GetSafeNormal() * PushInDistance;

    // 4. 执行多射线检测
    PreventCameraPenetration(*PPActor, SafeLocation, View.Location, DeltaTime,
        AimLineToDesiredPosBlockedPct, !bDoPredictiveAvoidance);

    // 5. 穿透回调
    if (AimLineToDesiredPosBlockedPct < ReportPenetrationPercent)
    {
        for (ILyraCameraAssistInterface* Assist : AssistArray)
        {
            if (Assist) Assist->OnCameraPenetratingTarget();
        }
    }
}
```

### 7.5 PreventCameraPenetration() — 核心射线检测

```cpp
// LyraCameraMode_ThirdPerson.cpp:173-350
void ULyraCameraMode_ThirdPerson::PreventCameraPenetration(...)
{
    FVector BaseRay = CameraLoc - SafeLoc;
    FRotationMatrix BaseRayMatrix(BaseRay.Rotation());
    FVector BaseRayLocalUp, BaseRayLocalFwd, BaseRayLocalRight;
    BaseRayMatrix.GetScaledAxes(BaseRayLocalFwd, BaseRayLocalRight, BaseRayLocalUp);

    for (int32 RayIdx = 0; RayIdx < NumRaysToShoot; ++RayIdx)
    {
        FLyraPenetrationAvoidanceFeeler& Feeler = PenetrationAvoidanceFeelers[RayIdx];
        if (Feeler.FramesUntilNextTrace > 0)
        {
            --Feeler.FramesUntilNextTrace;
            continue;
        }

        // 将 Feeler 的偏转旋转应用到基础射线上
        FVector RotatedRay = BaseRay.RotateAngleAxis(Feeler.AdjustmentRot.Yaw, BaseRayLocalUp);
        RotatedRay = RotatedRay.RotateAngleAxis(Feeler.AdjustmentRot.Pitch, BaseRayLocalRight);
        FVector RayTarget = SafeLoc + RotatedRay;

        // 球形扫描
        SphereShape.Sphere.Radius = Feeler.Extent;
        bool bHit = World->SweepSingleByChannel(Hit, SafeLoc, RayTarget,
            FQuat::Identity, ECC_Camera, SphereShape, SphereParams);

        // 忽略带 "IgnoreCameraCollision" Tag 的 Actor
        if (HitActor->ActorHasTag(NAME_IgnoreCameraCollision))
        {
            bIgnoreHit = true;
            SphereParams.AddIgnoredActor(HitActor);
        }

        // 忽略在 ViewTarget 前方的 CameraBlockingVolume
        if (HitActor->IsA<ACameraBlockingVolume>())
        {
            // 只阻挡从后方穿过的情况
            if (FVector::DotProduct(ViewTargetForwardXY, HitDirectionXY) > 0.0f)
            {
                bIgnoreHit = true;
            }
        }

        if (!bIgnoreHit)
        {
            // 计算被遮挡百分比
            float NewBlockPct = ((Hit.Location - SafeLoc).Size() - CollisionPushOutDistance)
                / (RayTarget - SafeLoc).Size();
            DistBlockedPctThisFrame = FMath::Min(NewBlockPct, DistBlockedPctThisFrame);

            // 主射线命中 → 硬阻挡
            if (RayIdx == 0) HardBlockedPct = DistBlockedPctThisFrame;
            else             SoftBlockedPct = DistBlockedPctThisFrame;
        }
    }

    // 混合策略：
    // - 需要拉近时：主射线立即生效，辅助射线平滑混入
    // - 需要恢复时：平滑混出
    if (bResetInterpolation)
        DistBlockedPct = DistBlockedPctThisFrame;
    else if (DistBlockedPct < DistBlockedPctThisFrame)
        // 平滑恢复
        DistBlockedPct += DeltaTime / PenetrationBlendOutTime * (DistBlockedPctThisFrame - DistBlockedPct);
    else
    {
        if (DistBlockedPct > HardBlockedPct)
            DistBlockedPct = HardBlockedPct;   // 立即跳转到硬阻挡位置
        else if (DistBlockedPct > SoftBlockedPct)
            DistBlockedPct -= DeltaTime / PenetrationBlendInTime * (DistBlockedPct - SoftBlockedPct);
    }

    // 最终调整相机位置
    if (DistBlockedPct < (1.0f - ZERO_ANIMWEIGHT_THRESH))
    {
        CameraLoc = SafeLoc + (CameraLoc - SafeLoc) * DistBlockedPct;
    }
}
```

**穿透避免总结**：
| 特性 | 说明 |
|------|------|
| 多射线扫描 | 7 条射线（1主 + 6辅），不同方向覆盖 |
| 硬/软阻挡 | 主射线立即拉近，辅助射线平滑拉近 |
| 帧间隔优化 | 辅助射线每 3~5 帧检测一次，减少射线开销 |
| 双速混合 | 拉近快（0.1s），恢复慢（0.15s），避免弹跳 |
| 智能忽略 | 忽略 IgnoreCameraCollision Tag、忽略前方的 CameraBlockingVolume |
| 安全位置 | 从瞄准线上找胶囊体最近点，推入内部，保持瞄准稳定 |

---

## 8. ALyraPlayerCameraManager — 玩家相机管理器

> 源码: `Source/LyraGame/Camera/LyraPlayerCameraManager.h/.cpp`

### 8.1 全局常量

```cpp
// LyraPlayerCameraManager.h:14-16
#define LYRA_CAMERA_DEFAULT_FOV         (80.0f)
#define LYRA_CAMERA_DEFAULT_PITCH_MIN   (-89.0f)
#define LYRA_CAMERA_DEFAULT_PITCH_MAX   (89.0f)
```

### 8.2 类定义

```cpp
// LyraPlayerCameraManager.h:25-46
UCLASS(notplaceable, MinimalAPI)
class ALyraPlayerCameraManager : public APlayerCameraManager
{
    UPROPERTY(Transient)
    TObjectPtr<ULyraUICameraManagerComponent> UICamera;
};
```

### 8.3 UpdateViewTarget() — UI 相机优先级

```cpp
// LyraPlayerCameraManager.cpp:34-45
void ALyraPlayerCameraManager::UpdateViewTarget(FTViewTarget& OutVT, float DeltaTime)
{
    // UI 相机优先级最高
    if (UICamera->NeedsToUpdateViewTarget())
    {
        Super::UpdateViewTarget(OutVT, DeltaTime);
        UICamera->UpdateViewTarget(OutVT, DeltaTime);
        return;
    }

    Super::UpdateViewTarget(OutVT, DeltaTime);
}
```

**优先级体系**：
1. `ULyraUICameraManagerComponent`：最高优先级，UI 动画/聚焦时覆盖
2. `ULyraCameraComponent::GetCameraView()`：正常游戏中由引擎通过 `Super::UpdateViewTarget` 调用
3. `CameraMode Stack`：在 `GetCameraView` 内部完成模式混合

### 8.4 DisplayDebug()

```cpp
// LyraPlayerCameraManager.cpp:47-65
void ALyraPlayerCameraManager::DisplayDebug(UCanvas* Canvas, ...)
{
    Super::DisplayDebug(Canvas, DebugDisplay, YL, YPos);

    const APawn* Pawn = PCOwner ? PCOwner->GetPawn() : nullptr;
    if (const ULyraCameraComponent* CameraComponent = ULyraCameraComponent::FindCameraComponent(Pawn))
    {
        CameraComponent->DrawDebug(Canvas);
    }
}
```

通过 `showdebug camera` 控制台命令，可以在屏幕上显示当前相机模式栈信息、各模式权重、位置/旋转/FOV 等调试数据。

---

## 9. ULyraUICameraManagerComponent — UI 相机管理组件

> 源码: `Source/LyraGame/Camera/LyraUICameraManagerComponent.h/.cpp`

### 9.1 设计目的

`ULyraUICameraManagerComponent` 是一个挂载在 `ALyraPlayerCameraManager` 内部（`Within=LyraPlayerCameraManager`）的组件，负责在 UI 需要控制相机时接管视角。

```cpp
// LyraUICameraManagerComponent.h:18-45
UCLASS(Transient, Within=LyraPlayerCameraManager)
class ULyraUICameraManagerComponent : public UActorComponent
{
    AActor* ViewTarget;
    bool bUpdatingViewTarget;

    void SetViewTarget(AActor* InViewTarget, FViewTargetTransitionParams TransitionParams);
    bool NeedsToUpdateViewTarget() const;
    void UpdateViewTarget(struct FTViewTarget& OutVT, float DeltaTime);
};
```

### 9.2 关键实现

```cpp
// LyraUICameraManagerComponent.cpp:46-57
void ULyraUICameraManagerComponent::SetViewTarget(AActor* InViewTarget, FViewTargetTransitionParams TransitionParams)
{
    TGuardValue<bool> UpdatingViewTargetGuard(bUpdatingViewTarget, true);

    ViewTarget = InViewTarget;
    // 直接调用 CameraManager 的 SetViewTarget
    CastChecked<ALyraPlayerCameraManager>(GetOwner())->SetViewTarget(ViewTarget, TransitionParams);
}

bool ULyraUICameraManagerComponent::NeedsToUpdateViewTarget() const
{
    return false;  // 当前版本中 UI 相机暂未激活逻辑
}
```

**当前状态**：`NeedsToUpdateViewTarget()` 固定返回 `false`，说明 UI 相机功能尚未完全实现，但框架已预留。这是一个典型的 **预留扩展点** 设计模式。

### 9.3 静态获取方法

```cpp
// LyraUICameraManagerComponent.cpp:14-25
ULyraUICameraManagerComponent* ULyraUICameraManagerComponent::GetComponent(APlayerController* PC)
{
    if (ALyraPlayerCameraManager* PCCamera = Cast<ALyraPlayerCameraManager>(PC->PlayerCameraManager))
    {
        return PCCamera->GetUICameraComponent();
    }
    return nullptr;
}
```

---

## 10. ILyraCameraAssistInterface — 相机辅助接口

> 源码: `Source/LyraGame/Camera/LyraCameraAssistInterface.h`

```cpp
// LyraCameraAssistInterface.h:17-41
class ILyraCameraAssistInterface
{
    // 获取允许相机穿透的 Actor 列表
    virtual void GetIgnoredActorsForCameraPentration(TArray<const AActor*>& OutActorsAllowPenetration) const { }

    // 获取穿透避免的目标 Actor（默认是 ViewTarget 本身）
    virtual TOptional<AActor*> GetCameraPreventPenetrationTarget() const
    {
        return TOptional<AActor*>();  // 不设置 = 使用默认
    }

    // 相机过于接近目标时的回调
    virtual void OnCameraPenetratingTarget() { }
};
```

**用途**：
- **载具场景**：乘坐载具时，`GetCameraPreventPenetrationTarget()` 可以返回载具 Actor 而非 Pawn，使穿透避免基于载具形状
- **忽略列表**：角色携带的附属 Actor（如翅膀、背包）可以通过 `GetIgnoredActorsForCameraPentration()` 排除
- **穿透通知**：`OnCameraPenetratingTarget()` 可用于隐藏角色网格体等反馈

---

## 11. 相机模式的确定与切换流程

### 11.1 初始化绑定

在 `ULyraHeroComponent::HandleChangeInitState()` 中，当状态从 `DataAvailable` 变为 `DataInitialized` 时：

```cpp
// LyraHeroComponent.cpp:176-183
if (PawnData)
{
    if (ULyraCameraComponent* CameraComponent = ULyraCameraComponent::FindCameraComponent(Pawn))
    {
        CameraComponent->DetermineCameraModeDelegate.BindUObject(this, &ThisClass::DetermineCameraMode);
    }
}
```

### 11.2 DetermineCameraMode() — 模式决策逻辑

```cpp
// LyraHeroComponent.cpp:471-493
TSubclassOf<ULyraCameraMode> ULyraHeroComponent::DetermineCameraMode() const
{
    // 优先级 1：能力覆盖的相机模式
    if (AbilityCameraMode)
    {
        return AbilityCameraMode;
    }

    // 优先级 2：PawnData 中配置的默认模式
    const APawn* Pawn = GetPawn<APawn>();
    if (ULyraPawnExtensionComponent* PawnExtComp = ULyraPawnExtensionComponent::FindPawnExtensionComponent(Pawn))
    {
        if (const ULyraPawnData* PawnData = PawnExtComp->GetPawnData<ULyraPawnData>())
        {
            return PawnData->DefaultCameraMode;
        }
    }

    return nullptr;
}
```

**优先级链**：
1. `AbilityCameraMode`（由 GAS 能力动态设置）
2. `PawnData->DefaultCameraMode`（数据资产配置）

---

## 12. GAS 能力系统与相机模式联动

### 12.1 能力设置相机模式

```cpp
// LyraGameplayAbility.cpp:520-528
void ULyraGameplayAbility::SetCameraMode(TSubclassOf<ULyraCameraMode> CameraMode)
{
    ENSURE_ABILITY_IS_INSTANTIATED_OR_RETURN(SetCameraMode, );

    if (ULyraHeroComponent* HeroComponent = GetHeroComponentFromActorInfo())
    {
        HeroComponent->SetAbilityCameraMode(CameraMode, CurrentSpecHandle);
        ActiveCameraMode = CameraMode;
    }
}
```

### 12.2 能力清除相机模式

```cpp
// LyraGameplayAbility.cpp:531-544
void ULyraGameplayAbility::ClearCameraMode()
{
    if (ActiveCameraMode)
    {
        if (ULyraHeroComponent* HeroComponent = GetHeroComponentFromActorInfo())
        {
            HeroComponent->ClearAbilityCameraMode(CurrentSpecHandle);
        }
        ActiveCameraMode = nullptr;
    }
}
```

### 12.3 EndAbility 自动清除

```cpp
// LyraGameplayAbility.cpp:195-200
void ULyraGameplayAbility::EndAbility(...)
{
    ClearCameraMode();  // 能力结束时自动清除相机覆盖
    Super::EndAbility(...);
}
```

### 12.4 HeroComponent 的安全机制

```cpp
// LyraHeroComponent.cpp:495-511
void ULyraHeroComponent::SetAbilityCameraMode(TSubclassOf<ULyraCameraMode> CameraMode,
    const FGameplayAbilitySpecHandle& OwningSpecHandle)
{
    if (CameraMode)
    {
        AbilityCameraMode = CameraMode;
        AbilityCameraModeOwningSpecHandle = OwningSpecHandle;
    }
}

void ULyraHeroComponent::ClearAbilityCameraMode(const FGameplayAbilitySpecHandle& OwningSpecHandle)
{
    // 只有设置者才能清除（通过 SpecHandle 验证所有权）
    if (AbilityCameraModeOwningSpecHandle == OwningSpecHandle)
    {
        AbilityCameraMode = nullptr;
        AbilityCameraModeOwningSpecHandle = FGameplayAbilitySpecHandle();
    }
}
```

**所有权校验**：通过 `FGameplayAbilitySpecHandle` 确保只有设置相机模式的能力才能清除它，防止多个能力之间的冲突。

### 12.5 典型使用场景

```
1. 玩家按下 ADS 键
2. GAS 激活 ADS 能力
3. ADS 能力调用 SetCameraMode(ADS_CameraMode)
4. HeroComponent 记录 AbilityCameraMode
5. 下一帧 DetermineCameraMode() 返回 ADS_CameraMode
6. CameraModeStack 推入 ADS 模式，开始混合
7. 玩家松开 ADS 键
8. ADS 能力 EndAbility → ClearCameraMode()
9. DetermineCameraMode() 回退到 PawnData->DefaultCameraMode
10. CameraModeStack 推回默认模式，平滑混合
```

---

## 13. ULyraCameraMode_TopDownArenaCamera — 俯视角竞技场相机

> 源码: `Plugins/GameFeatures/TopDownArena/Source/TopDownArenaRuntime/`

### 13.1 类定义

```cpp
// LyraCameraMode_TopDownArenaCamera.h:18-45
UCLASS(Abstract, Blueprintable)
class ULyraCameraMode_TopDownArenaCamera : public ULyraCameraMode
{
    UPROPERTY(EditDefaultsOnly)
    float ArenaWidth;     // 竞技场宽度

    UPROPERTY(EditDefaultsOnly)
    float ArenaHeight;    // 竞技场高度

    UPROPERTY(EditDefaultsOnly)
    FRotator DefaultPivotRotation;  // 固定俯视角旋转

    UPROPERTY(EditDefaultsOnly)
    FRuntimeFloatCurve BoundsSizeToDistance;  // 场地大小 → 相机距离映射曲线
};
```

### 13.2 UpdateView() — 俯视角实现

```cpp
// LyraCameraMode_TopDownArenaCamera.cpp:14-31
void ULyraCameraMode_TopDownArenaCamera::UpdateView(float DeltaTime)
{
    // 1. 定义竞技场包围盒
    FBox ArenaBounds(FVector(-ArenaWidth, -ArenaHeight, 0),
                     FVector(ArenaWidth, ArenaHeight, 100));

    // 2. 用包围盒最大分量查曲线得到相机距离
    const double BoundsMaxComponent = ArenaBounds.GetSize().GetMax();
    const double CameraLoftDistance = BoundsSizeToDistance.GetRichCurveConst()->Eval(BoundsMaxComponent);

    // 3. 从竞技场中心沿俯视角方向后退
    FVector PivotLocation = ArenaBounds.GetCenter() - DefaultPivotRotation.Vector() * CameraLoftDistance;

    // 4. 直接设置（不跟踪玩家）
    View.Location = PivotLocation;
    View.Rotation = DefaultPivotRotation;
    View.ControlRotation = View.Rotation;
    View.FieldOfView = FieldOfView;
}
```

**设计特点**：
- **完全固定视角**：不跟踪任何玩家，直接俯瞰整个竞技场
- **自适应距离**：通过曲线映射，竞技场大小变化时自动调整相机高度
- **GameFeature 插件**：作为 `TopDownArena` 游戏功能的一部分，体现了 Lyra 的模块化架构

---

## 14. ALyraDebugCameraController — 调试相机控制器

> 源码: `Source/LyraGame/Player/LyraDebugCameraController.h/.cpp`

```cpp
// LyraDebugCameraController.h:17-29
UCLASS()
class ALyraDebugCameraController : public ADebugCameraController
{
    ALyraDebugCameraController(const FObjectInitializer& ObjectInitializer)
    {
        CheatClass = ULyraCheatManager::StaticClass();
    }

    virtual void AddCheats(bool bForce) override
    {
#if USING_CHEAT_MANAGER
        Super::AddCheats(true);  // 强制启用作弊管理器
#endif
    }
};
```

**用途**：按下 `~` 键后输入 `ToggleDebugCamera` 可以切换到自由飞行调试相机。使用 Lyra 的 `ULyraCheatManager` 确保在调试相机模式下也能使用游戏特定的作弊命令。

---

## 15. 数据驱动配置（ULyraPawnData）

> 源码: `Source/LyraGame/Character/LyraPawnData.h`

```cpp
// LyraPawnData.h:24-54
UCLASS(MinimalAPI, BlueprintType, Const, Meta = (DisplayName = "Lyra Pawn Data"))
class ULyraPawnData : public UPrimaryDataAsset
{
    // Pawn 类
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Pawn")
    TSubclassOf<APawn> PawnClass;

    // 能力集
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Abilities")
    TArray<TObjectPtr<ULyraAbilitySet>> AbilitySets;

    // 输入配置
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Input")
    TObjectPtr<ULyraInputConfig> InputConfig;

    // 默认相机模式
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Camera")
    TSubclassOf<ULyraCameraMode> DefaultCameraMode;
};
```

通过 `ULyraPawnData`，设计师可以为不同类型的 Pawn 配置不同的默认相机模式，而无需修改 C++ 代码。例如：
- 射击游戏角色 → `BP_CameraMode_ThirdPerson`
- 俯视角角色 → `BP_CameraMode_TopDown`
- 载具 → `BP_CameraMode_Vehicle`

---

## 16. 完整调用流程时序

### 16.1 初始化时序

```
ALyraCharacter 构造函数
  └─→ CreateDefaultSubobject<ULyraCameraComponent>("CameraComponent")
        └─→ SetRelativeLocation(-300, 0, 75)  // 初始偏移

ULyraCameraComponent::OnRegister()
  └─→ NewObject<ULyraCameraModeStack>(this)

ULyraHeroComponent::HandleChangeInitState(DataAvailable → DataInitialized)
  ├─→ InitializePlayerInput()
  └─→ CameraComponent->DetermineCameraModeDelegate.BindUObject(this, &DetermineCameraMode)
```

### 16.2 每帧更新时序

```
APlayerCameraManager::UpdateCamera()
  └─→ ALyraPlayerCameraManager::UpdateViewTarget()
        ├─→ UICamera->NeedsToUpdateViewTarget()?
        │     └─→ false（当前版本）
        └─→ Super::UpdateViewTarget()
              └─→ ViewTarget.CheckViewTarget()
                    └─→ ULyraCameraComponent::GetCameraView(DeltaTime, DesiredView)
                          │
                          ├─→ UpdateCameraModes()
                          │     └─→ DetermineCameraModeDelegate.Execute()
                          │           └─→ ULyraHeroComponent::DetermineCameraMode()
                          │                 ├─→ return AbilityCameraMode (if set)
                          │                 └─→ return PawnData->DefaultCameraMode
                          │
                          ├─→ CameraModeStack->EvaluateStack(DeltaTime, CameraModeView)
                          │     ├─→ UpdateStack(DeltaTime)
                          │     │     └─→ for each Mode:
                          │     │           ├─→ UpdateCameraMode(DeltaTime)
                          │     │           │     ├─→ UpdateView(DeltaTime)        [虚函数]
                          │     │           │     │     ├─→ GetPivotLocation()
                          │     │           │     │     ├─→ GetPivotRotation()
                          │     │           │     │     ├─→ 曲线偏移计算
                          │     │           │     │     └─→ UpdatePreventPenetration()
                          │     │           │     └─→ UpdateBlending(DeltaTime)
                          │     │           └─→ 清理 BlendWeight >= 1.0 下方的模式
                          │     └─→ BlendStack(OutView)
                          │           └─→ 从栈底向栈顶混合
                          │
                          ├─→ PC->SetControlRotation(CameraModeView.ControlRotation)
                          ├─→ FieldOfView += FieldOfViewOffset; FieldOfViewOffset = 0
                          ├─→ SetWorldLocationAndRotation(Location, Rotation)
                          └─→ 填充 DesiredView → 送渲染
```

### 16.3 能力切换相机时序

```
Ability::ActivateAbility()
  └─→ SetCameraMode(ADS_CameraMode)
        └─→ HeroComponent->SetAbilityCameraMode(ADS_CameraMode, SpecHandle)

--- 下一帧 ---

GetCameraView()
  └─→ DetermineCameraMode() → ADS_CameraMode
        └─→ PushCameraMode(ADS_CameraMode)
              ├─→ 计算初始 BlendWeight
              └─→ 开始混合（BlendTime 秒内权重从 0→1）

--- N 帧后 ---

Ability::EndAbility()
  └─→ ClearCameraMode()
        └─→ HeroComponent->ClearAbilityCameraMode(SpecHandle)

--- 下一帧 ---

GetCameraView()
  └─→ DetermineCameraMode() → PawnData->DefaultCameraMode
        └─→ PushCameraMode(DefaultCameraMode)
              ├─→ DefaultMode 重新到栈顶
              └─→ ADS_Mode 权重逐渐降到 0 后被清理
```

---

## 17. 关键设计模式总结

### 17.1 模式栈与混合

| 设计决策 | 说明 |
|----------|------|
| UObject 而非 Component | CameraMode 是轻量级逻辑对象，避免 Tick 开销 |
| 对象池 + 活跃栈 | CameraModeInstances 缓存，CameraModeStack 管理活跃模式 |
| 自底向上混合 | 栈底权重 100%，逐层向上覆盖 |
| 自动清理 | BlendWeight 达 100% 后自动移除下方所有模式 |
| 连续性保持 | 重新推入已有模式时保留贡献度作为初始权重 |

### 17.2 穿透避免 vs SpringArm

| 特性 | Lyra 穿透避免 | UE SpringArm |
|------|--------------|--------------|
| 射线数量 | 7 条（多方向） | 1 条 |
| 预测避免 | 支持（辅助射线预扫描） | 不支持 |
| 帧间隔优化 | 辅助射线 3~5 帧一次 | 每帧检测 |
| 硬/软混合 | 区分立即和平滑 | 单一响应 |
| 忽略逻辑 | Tag + CameraBlockingVolume 方向检测 | 仅通道/忽略列表 |
| 安全位置 | 基于瞄准线和胶囊体的智能计算 | 简单原点 |

### 17.3 委托驱动的解耦

```
ULyraCameraComponent ──(委托)──→ ULyraHeroComponent::DetermineCameraMode()
                                    ├──→ ULyraGameplayAbility (能力覆盖)
                                    └──→ ULyraPawnData (数据资产默认值)
```

组件之间通过委托通信，没有直接的类依赖，使得：
- CameraComponent 不需要知道 HeroComponent 的存在
- 相机模式可以从任何来源（数据资产、能力、蓝图）驱动
- 非玩家 Pawn 可以不绑委托，使用自己的相机逻辑

### 17.4 所有权安全机制

能力对相机模式的覆盖通过 `FGameplayAbilitySpecHandle` 确保了所有权：
- 只有设置者能清除
- EndAbility 自动清除
- 多个能力同时存在时，后设置的覆盖前一个，但不会错误清除

---

## 18. 源码文件清单

### Camera 核心目录 (`Source/LyraGame/Camera/`)

| 文件 | 说明 |
|------|------|
| `LyraCameraMode.h` | CameraMode 基类 + CameraModeStack + BlendFunction 枚举 + CameraModeView 结构 |
| `LyraCameraMode.cpp` | 枢轴计算、View 更新、混合权重更新、栈的 Push/Update/Blend/Clean 全部实现 |
| `LyraCameraComponent.h` | 挂在 Character 上的相机组件，持有 CameraModeStack 和 DetermineCameraMode 委托 |
| `LyraCameraComponent.cpp` | GetCameraView 主循环、UpdateCameraModes 委托调用、FOV 偏移、调试绘制 |
| `LyraCameraMode_ThirdPerson.h` | 第三人称相机模式，曲线偏移 + 蹲伏偏移 + 穿透避免参数 |
| `LyraCameraMode_ThirdPerson.cpp` | UpdateView 完整流程、蹲伏处理、穿透避免核心算法 |
| `LyraPlayerCameraManager.h` | 全局常量（默认 FOV/Pitch）、UI 相机优先级 |
| `LyraPlayerCameraManager.cpp` | UpdateViewTarget UI 优先级逻辑、DisplayDebug |
| `LyraUICameraManagerComponent.h` | UI 相机管理组件定义 |
| `LyraUICameraManagerComponent.cpp` | SetViewTarget、静态获取方法 |
| `LyraCameraAssistInterface.h` | 穿透避免辅助接口（忽略列表 + 自定义目标 + 穿透回调） |
| `LyraPenetrationAvoidanceFeeler.h` | Feeler 射线结构（方向 + 权重 + 半径 + 间隔） |

### 关联文件

| 文件 | 相机相关内容 |
|------|-------------|
| `Character/LyraPawnData.h` | `DefaultCameraMode` 配置 |
| `Character/LyraHeroComponent.h/.cpp` | DetermineCameraMode + 能力相机覆盖 |
| `Character/LyraCharacter.cpp` | 创建 ULyraCameraComponent，设置初始位置 |
| `AbilitySystem/Abilities/LyraGameplayAbility.h/.cpp` | SetCameraMode/ClearCameraMode，EndAbility 自动清理 |
| `Player/LyraDebugCameraController.h/.cpp` | 调试相机控制器 |

### GameFeature 插件

| 文件 | 说明 |
|------|------|
| `Plugins/GameFeatures/TopDownArena/.../LyraCameraMode_TopDownArenaCamera.h/.cpp` | 俯视角竞技场相机模式 |
