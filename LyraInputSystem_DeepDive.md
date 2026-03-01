# Lyra Input System（输入系统）深度代码分析

## 目录

1. [系统概览与架构](#1-系统概览与架构)
2. [核心数据结构：ULyraInputConfig](#2-核心数据结构ulyra inputconfig)
3. [输入组件：ULyraInputComponent](#3-输入组件ulyra inputcomponent)
4. [输入初始化流程：ULyraHeroComponent](#4-输入初始化流程ulyra herocomponent)
5. [InputTag 与 GameplayTag 的桥接](#5-inputtag-与-gameplaytag-的桥接)
6. [能力系统输入处理：ULyraAbilitySystemComponent](#6-能力系统输入处理ulyra abilitysystemcomponent)
7. [自定义输入修改器（InputModifier）](#7-自定义输入修改器inputmodifier)
8. [瞄准灵敏度数据：ULyraAimSensitivityData](#8-瞄准灵敏度数据ulyra aimsensitivitydata)
9. [玩家输入类：ULyraPlayerInput](#9-玩家输入类ulyra playerinput)
10. [输入用户设置与键位映射配置](#10-输入用户设置与键位映射配置)
11. [GameFeature 输入注入机制](#11-gamefeature-输入注入机制)
12. [瞄准辅助系统（Aim Assist）](#12-瞄准辅助系统aim-assist)
13. [触摸模拟输入：ULyraSimulatedInputWidget](#13-触摸模拟输入ulyra simulatedinputwidget)
14. [PawnData 中的输入配置引用](#14-pawndata-中的输入配置引用)
15. [完整数据流总结](#15-完整数据流总结)
16. [关键源文件清单](#16-关键源文件清单)

---

## 1. 系统概览与架构

Lyra 的输入系统构建在 **UE5 Enhanced Input System** 之上，通过多层自定义封装，实现了 **输入动作与 GameplayTag 的解耦绑定**、**模块化 GameFeature 输入注入**、**设置驱动的输入修改器**、**手柄瞄准辅助** 等高级功能。

### 架构层次图

```
┌──────────────────────────────────────────────────────────────────┐
│                     硬件输入 (键盘/鼠标/手柄/触摸)                  │
└───────────────────────────┬──────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│              UE5 Enhanced Input System                           │
│  ┌─────────────────────┐  ┌──────────────────────────────────┐  │
│  │ UInputMappingContext │  │ UInputAction                     │  │
│  │ (IMC: 按键→动作映射)  │  │ (IA: 抽象的输入动作)              │  │
│  └─────────────────────┘  └──────────────────────────────────┘  │
│  ┌─────────────────────┐  ┌──────────────────────────────────┐  │
│  │ UInputModifier       │  │ UInputTrigger                    │  │
│  │ (修改器: 死区/灵敏度)  │  │ (触发器: 按下/释放/持续)          │  │
│  └─────────────────────┘  └──────────────────────────────────┘  │
└───────────────────────────┬──────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                   Lyra 自定义输入层                                │
│                                                                  │
│  ┌──────────────────┐    ┌───────────────────────────────────┐  │
│  │ ULyraInputConfig │───▶│ FLyraInputAction                  │  │
│  │ (数据资产)        │    │ { InputAction + InputTag }        │  │
│  │ ├ NativeActions   │    │ 将 IA 映射到 GameplayTag           │  │
│  │ └ AbilityActions  │    └───────────────────────────────────┘  │
│  └──────────────────┘                                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ULyraInputComponent : UEnhancedInputComponent            │   │
│  │ ├ BindNativeAction()  → 直接绑定 C++ 函数               │   │
│  │ └ BindAbilityActions()→ 批量绑定能力输入 (Tag传递)        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ULyraHeroComponent                                       │   │
│  │ ├ InitializePlayerInput()  ← 初始化入口                  │   │
│  │ ├ 注册 DefaultInputMappings (IMC)                        │   │
│  │ ├ 绑定 NativeAction (移动/视角/蹲伏/自动跑)             │   │
│  │ ├ 绑定 AbilityActions (能力输入)                         │   │
│  │ └ 广播 NAME_BindInputsNow 事件                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 自定义 InputModifier                                     │   │
│  │ ├ ULyraSettingBasedScalar (设置属性驱动缩放)             │   │
│  │ ├ ULyraInputModifierDeadZone (设置驱动死区)              │   │
│  │ ├ ULyraInputModifierGamepadSensitivity (手柄灵敏度)      │   │
│  │ ├ ULyraInputModifierAimInversion (轴反转)                │   │
│  │ └ UAimAssistInputModifier (瞄准辅助)                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└───────────────────────────┬──────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│              能力系统 (GAS) 输入处理                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ULyraAbilitySystemComponent                              │   │
│  │ ├ AbilityInputTagPressed(Tag)  → 收集本帧按下的能力      │   │
│  │ ├ AbilityInputTagReleased(Tag) → 收集本帧释放的能力      │   │
│  │ └ ProcessAbilityInput()        → 统一激活/取消能力       │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### 设计哲学

1. **Tag 驱动，完全解耦**：输入动作（InputAction）与游戏逻辑通过 `FGameplayTag` 桥接，而非硬编码的 InputID 整数
2. **数据驱动**：通过 `ULyraInputConfig` 数据资产配置 InputAction↔InputTag 映射，无需修改 C++ 代码
3. **模块化注入**：通过 GameFeature Action 动态添加 InputMappingContext 和 InputConfig，支持插件式功能扩展
4. **设置驱动的输入处理**：死区、灵敏度、轴反转等全部从玩家设置读取，运行时可调

---

## 2. 核心数据结构：ULyraInputConfig

**文件**: `Source/LyraGame/Input/LyraInputConfig.h/.cpp`

### 2.1 FLyraInputAction 结构体

```cpp
// LyraInputConfig.h:19-31
USTRUCT(BlueprintType)
struct FLyraInputAction
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TObjectPtr<const UInputAction> InputAction = nullptr;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Meta = (Categories = "InputTag"))
    FGameplayTag InputTag;
};
```

这是 Lyra 输入系统的基本映射单元，将一个 UE5 `UInputAction` 与一个 `FGameplayTag` 关联。`Meta = (Categories = "InputTag")` 限制编辑器中只能选择 `InputTag` 层级下的 Tag。

### 2.2 ULyraInputConfig 数据资产

```cpp
// LyraInputConfig.h:38-61
UCLASS(BlueprintType, Const)
class ULyraInputConfig : public UDataAsset
{
    GENERATED_BODY()

public:
    const UInputAction* FindNativeInputActionForTag(const FGameplayTag& InputTag, bool bLogNotFound = true) const;
    const UInputAction* FindAbilityInputActionForTag(const FGameplayTag& InputTag, bool bLogNotFound = true) const;

    // 原生输入动作列表 - 直接绑定到 C++ 函数（如移动、视角）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TArray<FLyraInputAction> NativeInputActions;

    // 能力输入动作列表 - 自动绑定到拥有匹配 InputTag 的能力
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TArray<FLyraInputAction> AbilityInputActions;
};
```

**关键设计：两类输入分离**

| 类别 | NativeInputActions | AbilityInputActions |
|------|-------------------|---------------------|
| 用途 | 移动、视角、蹲伏等基础操作 | 射击、跳跃、技能等能力触发 |
| 绑定方式 | `BindNativeAction()` → C++ 回调 | `BindAbilityActions()` → ASC Tag 路由 |
| 触发事件 | 指定 ETriggerEvent | 固定 Triggered + Completed |

### 2.3 查找函数实现

```cpp
// LyraInputConfig.cpp:14-30
const UInputAction* ULyraInputConfig::FindNativeInputActionForTag(
    const FGameplayTag& InputTag, bool bLogNotFound) const
{
    for (const FLyraInputAction& Action : NativeInputActions)
    {
        if (Action.InputAction && (Action.InputTag == InputTag))
        {
            return Action.InputAction;
        }
    }
    // ... 找不到则 Log 报错
    return nullptr;
}
```

线性遍历查找，由于配置数量通常很少（<20），这个 O(n) 查找完全可接受，且只在初始化时调用一次。

---

## 3. 输入组件：ULyraInputComponent

**文件**: `Source/LyraGame/Input/LyraInputComponent.h/.cpp`

### 3.1 类定义

```cpp
// LyraInputComponent.h:20-39
UCLASS(Config = Input)
class ULyraInputComponent : public UEnhancedInputComponent
{
    GENERATED_BODY()

public:
    void AddInputMappings(const ULyraInputConfig* InputConfig,
                          UEnhancedInputLocalPlayerSubsystem* InputSubsystem) const;
    void RemoveInputMappings(const ULyraInputConfig* InputConfig,
                             UEnhancedInputLocalPlayerSubsystem* InputSubsystem) const;

    template<class UserClass, typename FuncType>
    void BindNativeAction(const ULyraInputConfig* InputConfig,
                          const FGameplayTag& InputTag,
                          ETriggerEvent TriggerEvent,
                          UserClass* Object, FuncType Func, bool bLogIfNotFound);

    template<class UserClass, typename PressedFuncType, typename ReleasedFuncType>
    void BindAbilityActions(const ULyraInputConfig* InputConfig,
                            UserClass* Object,
                            PressedFuncType PressedFunc,
                            ReleasedFuncType ReleasedFunc,
                            TArray<uint32>& BindHandles);

    void RemoveBinds(TArray<uint32>& BindHandles);
};
```

继承自 `UEnhancedInputComponent`，在其基础上提供了两个核心模板方法。

### 3.2 BindNativeAction —— 原生动作绑定

```cpp
// LyraInputComponent.h:42-50
template<class UserClass, typename FuncType>
void ULyraInputComponent::BindNativeAction(
    const ULyraInputConfig* InputConfig,
    const FGameplayTag& InputTag,
    ETriggerEvent TriggerEvent,
    UserClass* Object, FuncType Func, bool bLogIfNotFound)
{
    check(InputConfig);
    if (const UInputAction* IA = InputConfig->FindNativeInputActionForTag(InputTag, bLogIfNotFound))
    {
        BindAction(IA, TriggerEvent, Object, Func);
    }
}
```

**执行流程**：
1. 通过 Tag 从 InputConfig 的 `NativeInputActions` 中找到对应的 `UInputAction`
2. 调用父类 `UEnhancedInputComponent::BindAction()` 绑定到指定的 C++ 回调函数
3. 支持指定 `ETriggerEvent` 类型（Triggered、Started、Completed 等）

### 3.3 BindAbilityActions —— 能力动作批量绑定

```cpp
// LyraInputComponent.h:52-72
template<class UserClass, typename PressedFuncType, typename ReleasedFuncType>
void ULyraInputComponent::BindAbilityActions(
    const ULyraInputConfig* InputConfig,
    UserClass* Object,
    PressedFuncType PressedFunc,
    ReleasedFuncType ReleasedFunc,
    TArray<uint32>& BindHandles)
{
    check(InputConfig);

    for (const FLyraInputAction& Action : InputConfig->AbilityInputActions)
    {
        if (Action.InputAction && Action.InputTag.IsValid())
        {
            if (PressedFunc)
            {
                BindHandles.Add(BindAction(Action.InputAction, ETriggerEvent::Triggered,
                    Object, PressedFunc, Action.InputTag).GetHandle());
            }
            if (ReleasedFunc)
            {
                BindHandles.Add(BindAction(Action.InputAction, ETriggerEvent::Completed,
                    Object, ReleasedFunc, Action.InputTag).GetHandle());
            }
        }
    }
}
```

**关键机制**：
- 遍历 `AbilityInputActions` 数组中的每一个映射
- 对每个 InputAction 绑定两次：`Triggered` → PressedFunc，`Completed` → ReleasedFunc
- **InputTag 作为额外参数传入回调**，使得一个统一的回调函数可以区分是哪个能力被触发
- 返回 `BindHandles` 用于后续解绑

### 3.4 RemoveBinds —— 解绑

```cpp
// LyraInputComponent.cpp:33-40
void ULyraInputComponent::RemoveBinds(TArray<uint32>& BindHandles)
{
    for (uint32 Handle : BindHandles)
    {
        RemoveBindingByHandle(Handle);
    }
    BindHandles.Reset();
}
```

通过句柄精确移除绑定，支持 GameFeature 动态添加/移除输入配置。

---

## 4. 输入初始化流程：ULyraHeroComponent

**文件**: `Source/LyraGame/Character/LyraHeroComponent.h/.cpp`

`ULyraHeroComponent` 是 Lyra 中玩家操控 Pawn 的核心组件，负责 **输入初始化** 和 **摄像机管理**。它是输入系统与角色系统的桥梁。

### 4.1 初始化状态机

HeroComponent 遵循 `IGameFrameworkInitStateInterface` 接口的状态链：

```
[无状态] → Spawned → DataAvailable → DataInitialized → GameplayReady
```

**输入初始化发生在 `DataAvailable → DataInitialized` 转换时**：

```cpp
// LyraHeroComponent.cpp:145-183
void ULyraHeroComponent::HandleChangeInitState(
    UGameFrameworkComponentManager* Manager,
    FGameplayTag CurrentState, FGameplayTag DesiredState)
{
    if (CurrentState == LyraGameplayTags::InitState_DataAvailable
        && DesiredState == LyraGameplayTags::InitState_DataInitialized)
    {
        APawn* Pawn = GetPawn<APawn>();
        ALyraPlayerState* LyraPS = GetPlayerState<ALyraPlayerState>();

        // 1. 初始化能力系统
        if (ULyraPawnExtensionComponent* PawnExtComp = ...)
        {
            PawnExtComp->InitializeAbilitySystem(LyraPS->GetLyraAbilitySystemComponent(), LyraPS);
        }

        // 2. 初始化输入 ← 关键步骤
        if (ALyraPlayerController* LyraPC = GetController<ALyraPlayerController>())
        {
            if (Pawn->InputComponent != nullptr)
            {
                InitializePlayerInput(Pawn->InputComponent);
            }
        }

        // 3. 设置摄像机
        if (PawnData)
        {
            if (ULyraCameraComponent* CameraComponent = ...)
            {
                CameraComponent->DetermineCameraModeDelegate.BindUObject(
                    this, &ThisClass::DetermineCameraMode);
            }
        }
    }
}
```

**进入 DataAvailable 的前提条件**（`CanChangeInitState` 检查）：
- Pawn 已存在
- PlayerState 已存在
- 本地控制且非 Bot 时：`InputComponent` 和 `LocalPlayer` 必须就绪
- Authority/Autonomous 时：Controller 必须已与 PlayerState 配对

### 4.2 InitializePlayerInput —— 核心初始化函数

```cpp
// LyraHeroComponent.cpp:225-302
void ULyraHeroComponent::InitializePlayerInput(UInputComponent* PlayerInputComponent)
{
    const APawn* Pawn = GetPawn<APawn>();
    const APlayerController* PC = GetController<APlayerController>();
    const ULyraLocalPlayer* LP = Cast<ULyraLocalPlayer>(PC->GetLocalPlayer());
    UEnhancedInputLocalPlayerSubsystem* Subsystem =
        LP->GetSubsystem<UEnhancedInputLocalPlayerSubsystem>();

    // Step 1: 清除所有现有映射
    Subsystem->ClearAllMappings();

    if (const ULyraPawnData* PawnData = PawnExtComp->GetPawnData<ULyraPawnData>())
    {
        if (const ULyraInputConfig* InputConfig = PawnData->InputConfig)
        {
            // Step 2: 注册 DefaultInputMappings（IMC + 优先级）
            for (const FInputMappingContextAndPriority& Mapping : DefaultInputMappings)
            {
                if (UInputMappingContext* IMC = Mapping.InputMapping.LoadSynchronous())
                {
                    if (Mapping.bRegisterWithSettings)
                    {
                        // 注册到用户设置系统（用于设置页面显示）
                        if (UEnhancedInputUserSettings* Settings = Subsystem->GetUserSettings())
                        {
                            Settings->RegisterInputMappingContext(IMC);
                        }
                        // 添加到 EnhancedInput 子系统
                        FModifyContextOptions Options = {};
                        Options.bIgnoreAllPressedKeysUntilRelease = false;
                        Subsystem->AddMappingContext(IMC, Mapping.Priority, Options);
                    }
                }
            }

            // Step 3: 获取 LyraInputComponent
            ULyraInputComponent* LyraIC = Cast<ULyraInputComponent>(PlayerInputComponent);

            // Step 4: 添加自定义输入映射
            LyraIC->AddInputMappings(InputConfig, Subsystem);

            // Step 5: 绑定能力输入动作 → ASC Tag 路由
            TArray<uint32> BindHandles;
            LyraIC->BindAbilityActions(InputConfig, this,
                &ThisClass::Input_AbilityInputTagPressed,
                &ThisClass::Input_AbilityInputTagReleased,
                BindHandles);

            // Step 6: 绑定原生输入动作
            LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Move,
                ETriggerEvent::Triggered, this, &ThisClass::Input_Move, false);
            LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Look_Mouse,
                ETriggerEvent::Triggered, this, &ThisClass::Input_LookMouse, false);
            LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Look_Stick,
                ETriggerEvent::Triggered, this, &ThisClass::Input_LookStick, false);
            LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_Crouch,
                ETriggerEvent::Triggered, this, &ThisClass::Input_Crouch, false);
            LyraIC->BindNativeAction(InputConfig, LyraGameplayTags::InputTag_AutoRun,
                ETriggerEvent::Triggered, this, &ThisClass::Input_AutoRun, false);
        }
    }

    // Step 7: 标记输入绑定就绪
    bReadyToBindInputs = true;

    // Step 8: 广播 BindInputsNow 事件（通知 GameFeature 添加额外绑定）
    UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(
        const_cast<APlayerController*>(PC), NAME_BindInputsNow);
    UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(
        const_cast<APawn*>(Pawn), NAME_BindInputsNow);
}
```

**完整初始化时序**：

```
1. ClearAllMappings()         - 清空 EnhancedInput 子系统的所有映射
2. 遍历 DefaultInputMappings  - 注册 IMC 到子系统和用户设置
3. AddInputMappings()         - 添加自定义映射（预留扩展点）
4. BindAbilityActions()       - 绑定能力输入，回调传递 InputTag
5. BindNativeAction() × 5     - 绑定移动/鼠标视角/手柄视角/蹲伏/自动跑
6. bReadyToBindInputs = true  - 标记就绪
7. 广播 NAME_BindInputsNow   - 触发 GameFeature 添加额外输入
```

### 4.3 原生输入处理函数

#### 移动输入

```cpp
// LyraHeroComponent.cpp:374-402
void ULyraHeroComponent::Input_Move(const FInputActionValue& InputActionValue)
{
    APawn* Pawn = GetPawn<APawn>();
    AController* Controller = Pawn ? Pawn->GetController() : nullptr;

    // 移动时取消自动跑
    if (ALyraPlayerController* LyraController = Cast<ALyraPlayerController>(Controller))
    {
        LyraController->SetIsAutoRunning(false);
    }

    if (Controller)
    {
        const FVector2D Value = InputActionValue.Get<FVector2D>();
        const FRotator MovementRotation(0.0f, Controller->GetControlRotation().Yaw, 0.0f);

        if (Value.X != 0.0f)
        {
            const FVector MovementDirection = MovementRotation.RotateVector(FVector::RightVector);
            Pawn->AddMovementInput(MovementDirection, Value.X);
        }
        if (Value.Y != 0.0f)
        {
            const FVector MovementDirection = MovementRotation.RotateVector(FVector::ForwardVector);
            Pawn->AddMovementInput(MovementDirection, Value.Y);
        }
    }
}
```

- 获取 2D 轴值，基于 Controller 的 Yaw 旋转将输入变换为世界空间方向
- X 轴→左右（Right），Y 轴→前后（Forward）

#### 鼠标视角 vs 手柄视角

```cpp
// 鼠标视角 - 直接使用原始值
void ULyraHeroComponent::Input_LookMouse(const FInputActionValue& InputActionValue)
{
    const FVector2D Value = InputActionValue.Get<FVector2D>();
    Pawn->AddControllerYawInput(Value.X);
    Pawn->AddControllerPitchInput(Value.Y);
}

// 手柄视角 - 乘以转速和 DeltaTime
void ULyraHeroComponent::Input_LookStick(const FInputActionValue& InputActionValue)
{
    const FVector2D Value = InputActionValue.Get<FVector2D>();
    const UWorld* World = GetWorld();
    Pawn->AddControllerYawInput(Value.X * LyraHero::LookYawRate * World->GetDeltaSeconds());
    Pawn->AddControllerPitchInput(Value.Y * LyraHero::LookPitchRate * World->GetDeltaSeconds());
}
```

鼠标和手柄的视角输入被分为 **两个独立的 InputAction**（`InputTag.Look.Mouse` 和 `InputTag.Look.Stick`），各有不同的 InputModifier 链：
- **鼠标**：直接传入原始像素偏移
- **手柄**：乘以固定转速（Yaw: 300°/s, Pitch: 165°/s）再乘以 DeltaTime，加上死区/灵敏度/瞄准辅助等 Modifier

### 4.4 能力输入回调

```cpp
// LyraHeroComponent.cpp:343-372
void ULyraHeroComponent::Input_AbilityInputTagPressed(FGameplayTag InputTag)
{
    if (const APawn* Pawn = GetPawn<APawn>())
    {
        if (const ULyraPawnExtensionComponent* PawnExtComp = ...)
        {
            if (ULyraAbilitySystemComponent* LyraASC = PawnExtComp->GetLyraAbilitySystemComponent())
            {
                LyraASC->AbilityInputTagPressed(InputTag);
            }
        }
    }
}

void ULyraHeroComponent::Input_AbilityInputTagReleased(FGameplayTag InputTag)
{
    // ... 同理，调用 LyraASC->AbilityInputTagReleased(InputTag);
}
```

这是 `BindAbilityActions` 绑定的统一回调。当任何 AbilityInputAction 被触发时，回调函数收到对应的 `InputTag`，然后转发给 ASC。

### 4.5 动态添加/移除额外输入配置

```cpp
// LyraHeroComponent.cpp:304-336
void ULyraHeroComponent::AddAdditionalInputConfig(const ULyraInputConfig* InputConfig)
{
    TArray<uint32> BindHandles;
    const APawn* Pawn = GetPawn<APawn>();
    // ... 获取 LyraInputComponent
    ULyraInputComponent* LyraIC = Pawn->FindComponentByClass<ULyraInputComponent>();
    LyraIC->BindAbilityActions(InputConfig, this,
        &ThisClass::Input_AbilityInputTagPressed,
        &ThisClass::Input_AbilityInputTagReleased, BindHandles);
}

void ULyraHeroComponent::RemoveAdditionalInputConfig(const ULyraInputConfig* InputConfig)
{
    //@TODO: Implement me!  // 注意：尚未实现
}
```

此机制被 `GameFeatureAction_AddInputBinding` 调用，允许 GameFeature 在运行时注入新的能力输入绑定。

---

## 5. InputTag 与 GameplayTag 的桥接

**文件**: `Source/LyraGame/LyraGameplayTags.h/.cpp`

### 5.1 预定义的 InputTag

```cpp
// LyraGameplayTags.h:22-26
UE_DECLARE_GAMEPLAY_TAG_EXTERN(InputTag_Move);
UE_DECLARE_GAMEPLAY_TAG_EXTERN(InputTag_Look_Mouse);
UE_DECLARE_GAMEPLAY_TAG_EXTERN(InputTag_Look_Stick);
UE_DECLARE_GAMEPLAY_TAG_EXTERN(InputTag_Crouch);
UE_DECLARE_GAMEPLAY_TAG_EXTERN(InputTag_AutoRun);

// LyraGameplayTags.cpp:21-25
UE_DEFINE_GAMEPLAY_TAG_COMMENT(InputTag_Move, "InputTag.Move", "Move input.");
UE_DEFINE_GAMEPLAY_TAG_COMMENT(InputTag_Look_Mouse, "InputTag.Look.Mouse", "Look (mouse) input.");
UE_DEFINE_GAMEPLAY_TAG_COMMENT(InputTag_Look_Stick, "InputTag.Look.Stick", "Look (stick) input.");
UE_DEFINE_GAMEPLAY_TAG_COMMENT(InputTag_Crouch, "InputTag.Crouch", "Crouch input.");
UE_DEFINE_GAMEPLAY_TAG_COMMENT(InputTag_AutoRun, "InputTag.AutoRun", "Auto-run input.");
```

### 5.2 Tag 层级设计

```
InputTag
├── InputTag.Move            → 原生：移动
├── InputTag.Look
│   ├── InputTag.Look.Mouse  → 原生：鼠标视角
│   └── InputTag.Look.Stick  → 原生：手柄视角
├── InputTag.Crouch          → 原生：蹲伏
├── InputTag.AutoRun         → 原生：自动跑
├── InputTag.Weapon.Fire     → 能力：射击（在 AbilityInputActions 中配置）
├── InputTag.Ability.Jump    → 能力：跳跃
├── InputTag.Ability.Dash    → 能力：冲刺
└── ...                      → 可通过数据资产自由扩展
```

**原生 Tag 用于 `BindNativeAction`**，它们在 C++ 中硬编码绑定到特定函数。
**能力 Tag 用于 `BindAbilityActions`**，它们通过数据驱动的方式绑定到 GAS 能力。

---

## 6. 能力系统输入处理：ULyraAbilitySystemComponent

**文件**: `Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.h/.cpp`

### 6.1 输入收集

```cpp
// LyraAbilitySystemComponent.cpp:186-214
void ULyraAbilitySystemComponent::AbilityInputTagPressed(const FGameplayTag& InputTag)
{
    if (InputTag.IsValid())
    {
        for (const FGameplayAbilitySpec& AbilitySpec : ActivatableAbilities.Items)
        {
            if (AbilitySpec.Ability &&
                AbilitySpec.GetDynamicSpecSourceTags().HasTagExact(InputTag))
            {
                InputPressedSpecHandles.AddUnique(AbilitySpec.Handle);
                InputHeldSpecHandles.AddUnique(AbilitySpec.Handle);
            }
        }
    }
}

void ULyraAbilitySystemComponent::AbilityInputTagReleased(const FGameplayTag& InputTag)
{
    if (InputTag.IsValid())
    {
        for (const FGameplayAbilitySpec& AbilitySpec : ActivatableAbilities.Items)
        {
            if (AbilitySpec.Ability &&
                AbilitySpec.GetDynamicSpecSourceTags().HasTagExact(InputTag))
            {
                InputReleasedSpecHandles.AddUnique(AbilitySpec.Handle);
                InputHeldSpecHandles.Remove(AbilitySpec.Handle);
            }
        }
    }
}
```

**关键**：使用 `GetDynamicSpecSourceTags()` 匹配 InputTag。当能力通过 `ULyraAbilitySet::GrantToAbilitySystem()` 授予时，InputTag 被设置为 `DynamicSpecSourceTags`，这样 ASC 就能将输入事件路由到正确的能力。

### 6.2 输入三缓冲区

```cpp
// LyraAbilitySystemComponent.h:97-104
// 本帧按下的能力句柄
TArray<FGameplayAbilitySpecHandle> InputPressedSpecHandles;
// 本帧释放的能力句柄
TArray<FGameplayAbilitySpecHandle> InputReleasedSpecHandles;
// 持续按住的能力句柄
TArray<FGameplayAbilitySpecHandle> InputHeldSpecHandles;
```

### 6.3 ProcessAbilityInput —— 统一处理

```cpp
// LyraAbilitySystemComponent.cpp:216-311
void ULyraAbilitySystemComponent::ProcessAbilityInput(float DeltaTime, bool bGamePaused)
{
    // 检查输入是否被阻塞（TAG_Gameplay_AbilityInputBlocked）
    if (HasMatchingGameplayTag(TAG_Gameplay_AbilityInputBlocked))
    {
        ClearAbilityInput();
        return;
    }

    static TArray<FGameplayAbilitySpecHandle> AbilitiesToActivate;
    AbilitiesToActivate.Reset();

    // 1. 处理持续按住的能力（WhileInputActive 策略）
    for (const FGameplayAbilitySpecHandle& SpecHandle : InputHeldSpecHandles)
    {
        if (const FGameplayAbilitySpec* AbilitySpec = FindAbilitySpecFromHandle(SpecHandle))
        {
            if (AbilitySpec->Ability && !AbilitySpec->IsActive())
            {
                const ULyraGameplayAbility* LyraAbilityCDO = Cast<ULyraGameplayAbility>(AbilitySpec->Ability);
                if (LyraAbilityCDO && LyraAbilityCDO->GetActivationPolicy() == ELyraAbilityActivationPolicy::WhileInputActive)
                {
                    AbilitiesToActivate.AddUnique(AbilitySpec->Handle);
                }
            }
        }
    }

    // 2. 处理本帧按下的能力（OnInputTriggered 策略）
    for (const FGameplayAbilitySpecHandle& SpecHandle : InputPressedSpecHandles)
    {
        if (FGameplayAbilitySpec* AbilitySpec = FindAbilitySpecFromHandle(SpecHandle))
        {
            if (AbilitySpec->Ability)
            {
                AbilitySpec->InputPressed = true;

                if (AbilitySpec->IsActive())
                {
                    // 已激活 → 转发 InputPressed 事件（用于 WaitInputPress 任务）
                    AbilitySpecInputPressed(*AbilitySpec);
                }
                else
                {
                    const ULyraGameplayAbility* LyraAbilityCDO = Cast<ULyraGameplayAbility>(AbilitySpec->Ability);
                    if (LyraAbilityCDO && LyraAbilityCDO->GetActivationPolicy() == ELyraAbilityActivationPolicy::OnInputTriggered)
                    {
                        AbilitiesToActivate.AddUnique(AbilitySpec->Handle);
                    }
                }
            }
        }
    }

    // 3. 统一激活所有待激活能力
    for (const FGameplayAbilitySpecHandle& Handle : AbilitiesToActivate)
    {
        TryActivateAbility(Handle);
    }

    // 4. 处理本帧释放的能力
    for (const FGameplayAbilitySpecHandle& SpecHandle : InputReleasedSpecHandles)
    {
        if (FGameplayAbilitySpec* AbilitySpec = FindAbilitySpecFromHandle(SpecHandle))
        {
            if (AbilitySpec->Ability)
            {
                AbilitySpec->InputPressed = false;
                if (AbilitySpec->IsActive())
                {
                    AbilitySpecInputReleased(*AbilitySpec);
                }
            }
        }
    }

    // 5. 清空本帧缓冲
    InputPressedSpecHandles.Reset();
    InputReleasedSpecHandles.Reset();
}
```

### 6.4 能力激活策略与输入的关系

| ELyraAbilityActivationPolicy | 输入行为 |
|-------------------------------|---------|
| `OnInputTriggered` | 按下即激活一次 |
| `WhileInputActive` | 按住期间持续尝试激活（每帧检查） |
| `OnSpawn` | 不依赖输入，生成时自动激活 |

### 6.5 输入阻塞机制

```cpp
UE_DEFINE_GAMEPLAY_TAG(TAG_Gameplay_AbilityInputBlocked, "Gameplay.AbilityInputBlocked");
```

当 ASC 拥有 `Gameplay.AbilityInputBlocked` 标签时，`ProcessAbilityInput` 会直接清空所有输入缓冲并返回。这可以通过 GameplayEffect 动态添加，实现如"眩晕时禁止输入"的效果。

---

## 7. 自定义输入修改器（InputModifier）

**文件**: `Source/LyraGame/Input/LyraInputModifiers.h/.cpp`

Lyra 定义了 4 个自定义 InputModifier，它们全部从 `ULyraLocalPlayer` → `ULyraSettingsShared` 读取玩家设置。

### 7.1 ULyraSettingBasedScalar —— 设置属性驱动缩放

```cpp
// LyraInputModifiers.h:20-52
UCLASS(NotBlueprintable, MinimalAPI, meta = (DisplayName = "Setting Based Scalar"))
class ULyraSettingBasedScalar : public UInputModifier
{
    UPROPERTY(EditInstanceOnly)
    FName XAxisScalarSettingName = NAME_None;
    UPROPERTY(EditInstanceOnly)
    FName YAxisScalarSettingName = NAME_None;
    UPROPERTY(EditInstanceOnly)
    FName ZAxisScalarSettingName = NAME_None;

    UPROPERTY(EditInstanceOnly)
    FVector MaxValueClamp = FVector(10.0, 10.0, 10.0);
    UPROPERTY(EditInstanceOnly)
    FVector MinValueClamp = FVector::ZeroVector;

protected:
    virtual FInputActionValue ModifyRaw_Implementation(...) override;
    TArray<const FProperty*> PropertyCache;  // 反射属性缓存
};
```

**工作原理**：
1. 通过 `FName` 指定 `ULyraSettingsShared` 上的属性名
2. 运行时通过 **反射**（`FindPropertyByName` + `ContainerPtrToValuePtr`）读取 double 值
3. 将读取到的值作为缩放因子乘以输入值
4. 使用 `PropertyCache` 缓存 `FProperty*` 指针避免每帧查找

```cpp
// LyraInputModifiers.cpp:38-84
FInputActionValue ULyraSettingBasedScalar::ModifyRaw_Implementation(...)
{
    ULyraLocalPlayer* LocalPlayer = LyraInputModifiersHelpers::GetLocalPlayer(PlayerInput);
    const UClass* SettingsClass = ULyraSettingsShared::StaticClass();
    ULyraSettingsShared* SharedSettings = LocalPlayer->GetSharedSettings();

    // 反射读取属性值
    const FProperty* XAxisValue = bHasCachedProperty ? PropertyCache[0]
        : SettingsClass->FindPropertyByName(XAxisScalarSettingName);
    // ... Y、Z 同理

    // 缓存属性指针
    if (PropertyCache.IsEmpty()) { PropertyCache.Emplace(...); }

    FVector ScalarToUse = FVector(1.0, 1.0, 1.0);
    // 根据值类型读取并 Clamp
    ScalarToUse.X = *XAxisValue->ContainerPtrToValuePtr<double>(SharedSettings);
    ScalarToUse.X = FMath::Clamp(ScalarToUse.X, MinValueClamp.X, MaxValueClamp.X);

    return CurrentValue.Get<FVector>() * ScalarToUse;
}
```

### 7.2 ULyraInputModifierDeadZone —— 设置驱动死区

```cpp
// LyraInputModifiers.h:68-92
UCLASS(NotBlueprintable, MinimalAPI, meta = (DisplayName = "Lyra Settings Driven Dead Zone"))
class ULyraInputModifierDeadZone : public UInputModifier
{
    UPROPERTY(EditInstanceOnly)
    EDeadZoneType Type = EDeadZoneType::Radial;

    UPROPERTY(EditInstanceOnly)
    float UpperThreshold = 1.0f;

    UPROPERTY(EditInstanceOnly)
    EDeadzoneStick DeadzoneStick = EDeadzoneStick::MoveStick;
};
```

区分移动摇杆和视角摇杆，从 `ULyraSettingsShared` 读取不同的 `LowerThreshold`：

```cpp
// LyraInputModifiers.cpp:89-139
float LowerThreshold =
    (DeadzoneStick == EDeadzoneStick::MoveStick) ?
    Settings->GetGamepadMoveStickDeadZone() :
    Settings->GetGamepadLookStickDeadZone();

auto DeadZoneLambda = [LowerThreshold, this](const float AxisVal)
{
    return FMath::Min(1.f,
        (FMath::Max(0.f, FMath::Abs(AxisVal) - LowerThreshold)
        / (UpperThreshold - LowerThreshold)))
        * FMath::Sign(AxisVal);
};
```

支持 **Axial**（各轴独立死区）和 **Radial**（径向死区）两种模式。

### 7.3 ULyraInputModifierGamepadSensitivity —— 手柄灵敏度

```cpp
// LyraInputModifiers.cpp:154-171
FInputActionValue ULyraInputModifierGamepadSensitivity::ModifyRaw_Implementation(...)
{
    const ELyraGamepadSensitivity Sensitivity =
        (TargetingType == ELyraTargetingType::Normal) ?
        Settings->GetGamepadLookSensitivityPreset() :
        Settings->GetGamepadTargetingSensitivityPreset();

    const float Scalar = SensitivityLevelTable->SensitivtyEnumToFloat(Sensitivity);
    return CurrentValue.Get<FVector>() * Scalar;
}
```

区分 **Normal**（正常视角）和 **ADS**（瞄准镜下）两种灵敏度，从 `ULyraAimSensitivityData` 数据表查询对应的浮点缩放因子。

### 7.4 ULyraInputModifierAimInversion —— 轴反转

```cpp
// LyraInputModifiers.cpp:176-200
FInputActionValue ULyraInputModifierAimInversion::ModifyRaw_Implementation(...)
{
    FVector NewValue = CurrentValue.Get<FVector>();

    if (Settings->GetInvertVerticalAxis())
        NewValue.Y *= -1.0f;

    if (Settings->GetInvertHorizontalAxis())
        NewValue.X *= -1.0f;

    return NewValue;
}
```

### 7.5 InputModifier 链处理顺序

在 InputMappingContext 的每个 InputAction 映射中，Modifier 按配置顺序依次执行。Lyra 手柄视角的典型 Modifier 链：

```
原始手柄输入
  → ULyraInputModifierDeadZone (死区过滤)
  → ULyraInputModifierGamepadSensitivity (灵敏度缩放)
  → ULyraInputModifierAimInversion (轴反转)
  → UAimAssistInputModifier (瞄准辅助：拉拽 + 减速)
  → 最终输出值
```

---

## 8. 瞄准灵敏度数据：ULyraAimSensitivityData

**文件**: `Source/LyraGame/Input/LyraAimSensitivityData.h/.cpp`

```cpp
// LyraAimSensitivityData.h:16-30
UCLASS(MinimalAPI, BlueprintType, Const)
class ULyraAimSensitivityData : public UPrimaryDataAsset
{
    GENERATED_BODY()
public:
    const float SensitivtyEnumToFloat(const ELyraGamepadSensitivity InSensitivity) const;

protected:
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TMap<ELyraGamepadSensitivity, float> SensitivityMap;
};
```

默认灵敏度映射表：

| ELyraGamepadSensitivity | 缩放因子 |
|--------------------------|---------|
| Slow | 0.5 |
| SlowPlus | 0.75 |
| SlowPlusPlus | 0.9 |
| Normal | 1.0 |
| NormalPlus | 1.1 |
| NormalPlusPlus | 1.25 |
| Fast | 1.5 |
| FastPlus | 1.75 |
| FastPlusPlus | 2.0 |
| Insane | 2.5 |

---

## 9. 玩家输入类：ULyraPlayerInput

**文件**: `Source/LyraGame/Input/LyraPlayerInput.h/.cpp`

```cpp
// LyraPlayerInput.h:16-35
UCLASS(config = Input, transient)
class ULyraPlayerInput : public UEnhancedPlayerInput
{
    GENERATED_BODY()
public:
    ULyraPlayerInput();
    virtual ~ULyraPlayerInput() override;

protected:
    virtual bool InputKey(const FInputKeyEventArgs& Params) override;
    void ProcessInputEventForLatencyMarker(const FInputKeyEventArgs& Params);
    void BindToLatencyMarkerSettingChange();
    void UnbindLatencyMarkerSettingChangeListener();
    void HandleLatencyMarkerSettingChanged();

    bool bShouldTriggerLatencyFlash = false;
};
```

### 功能：输入延迟标记（Latency Marker）

继承自 `UEnhancedPlayerInput`，重写 `InputKey()` 以在鼠标左键按下时触发延迟闪烁标记：

```cpp
// LyraPlayerInput.cpp:27-57
bool ULyraPlayerInput::InputKey(const FInputKeyEventArgs& Params)
{
    const bool bResult = Super::InputKey(Params);
    ProcessInputEventForLatencyMarker(Params);
    return bResult;
}

void ULyraPlayerInput::ProcessInputEventForLatencyMarker(const FInputKeyEventArgs& Params)
{
    if (!bShouldTriggerLatencyFlash)
        return;

    if (Params.Key == EKeys::LeftMouseButton)
    {
        TArray<ILatencyMarkerModule*> LatencyMarkerModules =
            IModularFeatures::Get().GetModularFeatureImplementations<ILatencyMarkerModule>(...);

        for (ILatencyMarkerModule* Module : LatencyMarkerModules)
        {
            Module->SetCustomLatencyMarker(7, GFrameCounter);  // TRIGGER_FLASH = 7
        }
    }
}
```

此功能配合 NVIDIA Reflex 等插件使用，用于测量从按下鼠标到屏幕画面变化的端到端延迟。通过 `ULyraSettingsLocal` 控制开关。

---

## 10. 输入用户设置与键位映射配置

### 10.1 ULyraInputUserSettings

**文件**: `Source/LyraGame/Input/LyraInputUserSettings.h/.cpp`

```cpp
// LyraInputUserSettings.h:17-35
UCLASS(MinimalAPI)
class ULyraInputUserSettings : public UEnhancedInputUserSettings
{
    GENERATED_BODY()
public:
    virtual void ApplySettings() override;
};
```

继承 UE5 的 `UEnhancedInputUserSettings`，负责序列化和应用输入设置。当前为扩展预留点，可添加如"切换 vs 按住"等自定义设置。

### 10.2 ULyraPlayerMappableKeySettings

```cpp
// LyraInputUserSettings.h:42-56
UCLASS(MinimalAPI)
class ULyraPlayerMappableKeySettings : public UPlayerMappableKeySettings
{
    GENERATED_BODY()
public:
    const FText& GetTooltipText() const;

protected:
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Settings")
    FText Tooltip = FText::GetEmpty();
};
```

为每个可映射的按键操作提供额外的 Tooltip 元数据，用于设置界面显示。

### 10.3 ULyraPlayerMappableKeyProfile

**文件**: `Source/LyraGame/Input/LyraPlayerMappableKeyProfile.h/.cpp`

```cpp
UCLASS(MinimalAPI)
class ULyraPlayerMappableKeyProfile : public UEnhancedPlayerMappableKeyProfile
{
    GENERATED_BODY()
protected:
    virtual void EquipProfile() override;
    virtual void UnEquipProfile() override;
};
```

继承 UE5 的按键配置档案系统，支持多套按键配置的切换。`EquipProfile`/`UnEquipProfile` 是切换配置档案时的钩子。

---

## 11. GameFeature 输入注入机制

### 11.1 GameFeatureAction_AddInputContextMapping

**文件**: `Source/LyraGame/GameFeatures/GameFeatureAction_AddInputContextMapping.h/.cpp`

此 GameFeature Action 在功能激活时，向所有 PlayerController 的 EnhancedInput 子系统添加 IMC。

#### 数据结构

```cpp
// GameFeatureAction_AddInputContextMapping.h:15-30
USTRUCT()
struct FInputMappingContextAndPriority
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category="Input")
    TSoftObjectPtr<UInputMappingContext> InputMapping;

    UPROPERTY(EditAnywhere, Category="Input")
    int32 Priority = 0;

    UPROPERTY(EditAnywhere, Category="Input")
    bool bRegisterWithSettings = true;
};
```

#### 生命周期

```
OnGameFeatureRegistering()
  → RegisterInputMappingContexts()
    → 绑定 FWorldDelegates::OnStartGameInstance
    → 遍历所有 GameInstance
      → 绑定 OnLocalPlayerAdded/Removed
      → 遍历所有 LocalPlayer
        → RegisterInputMappingContext(IMC) 到 EnhancedInputUserSettings

OnGameFeatureActivating()
  → AddToWorld()
    → 注册 ComponentExtensionHandler 到 APlayerController
    → HandleControllerExtension()
      → AddInputMappingForPlayer()
        → Subsystem->AddMappingContext(IMC, Priority)

OnGameFeatureDeactivating()
  → Reset()
    → RemoveInputMapping()
      → Subsystem->RemoveMappingContext(IMC)

OnGameFeatureUnregistering()
  → UnregisterInputMappingContexts()
    → UnregisterInputMappingContext(IMC)
```

**双重注册**：
1. **Registering 阶段**：注册到 `EnhancedInputUserSettings`（用于设置 UI 显示所有可映射的按键）
2. **Activating 阶段**：添加到 `EnhancedInputLocalPlayerSubsystem`（实际生效）

### 11.2 GameFeatureAction_AddInputBinding

**文件**: `Source/LyraGame/GameFeatures/GameFeatureAction_AddInputBinding.h/.cpp`

此 GameFeature Action 在功能激活时，向 Pawn 的 `ULyraHeroComponent` 添加额外的 `ULyraInputConfig`。

```cpp
// GameFeatureAction_AddInputBinding.h:37-38
UPROPERTY(EditAnywhere, Category="Input")
TArray<TSoftObjectPtr<const ULyraInputConfig>> InputConfigs;
```

#### 核心逻辑

```cpp
// GameFeatureAction_AddInputBinding.cpp:107-120
void UGameFeatureAction_AddInputBinding::HandlePawnExtension(
    AActor* Actor, FName EventName, FGameFeatureStateChangeContext ChangeContext)
{
    APawn* AsPawn = CastChecked<APawn>(Actor);

    if (EventName == UGameFrameworkComponentManager::NAME_ExtensionRemoved
        || EventName == UGameFrameworkComponentManager::NAME_ReceiverRemoved)
    {
        RemoveInputMapping(AsPawn, ActiveData);
    }
    else if (EventName == UGameFrameworkComponentManager::NAME_ExtensionAdded
             || EventName == ULyraHeroComponent::NAME_BindInputsNow)
    {
        AddInputMappingForPlayer(AsPawn, ActiveData);
    }
}
```

**关键**：监听 `NAME_BindInputsNow` 事件。当 `ULyraHeroComponent::InitializePlayerInput()` 完成后广播此事件，GameFeature 收到后调用 `HeroComponent->AddAdditionalInputConfig(BindSet)` 注入额外能力绑定。

**竞态处理**：如果 GameFeature 在 HeroComponent 初始化之前就激活了，它通过 `ExtensionAdded` 事件也能被通知到，但会先检查 `HeroComponent->IsReadyToBindInputs()` 确保时机正确。

---

## 12. 瞄准辅助系统（Aim Assist）

**文件目录**: `Plugins/GameFeatures/ShooterCore/Source/ShooterCoreRuntime/Public/Input/` 和 `Private/Input/`

瞄准辅助是 Lyra 输入系统中最复杂的子模块，作为 `UInputModifier` 实现，完全在客户端的 Enhanced Input 处理管线中运行。

### 12.1 架构组成

```
┌────────────────────────────────────────────────────────────┐
│  UAimAssistInputModifier : UInputModifier                  │
│  (挂在手柄 Look Stick 的 InputAction 上)                    │
│  ├── FAimAssistSettings (参数配置)                          │
│  ├── FAimAssistFilter   (目标过滤规则)                      │
│  ├── FAimAssistOwnerViewData (视口数据)                     │
│  ├── TargetCache0 / TargetCache1 (目标双缓冲)              │
│  └── 核心方法:                                             │
│      ├ ModifyRaw_Implementation()  - 修改输入值             │
│      ├ UpdateTargetData()          - 更新目标信息           │
│      ├ UpdateRotationalVelocity()  - 计算辅助旋转           │
│      └ CalculateTargetStrengths()  - 计算拉拽/减速强度      │
└────────────────────────┬───────────────────────────────────┘
                         │ 查询目标
                         ▼
┌────────────────────────────────────────────────────────────┐
│  UAimAssistTargetManagerComponent : UGameStateComponent    │
│  (挂在 GameState 上，全局单例)                              │
│  ├ GetVisibleTargets()           - 空间查询+屏幕投影+过滤   │
│  ├ DoesTargetPassFilter()        - 团队/距离/标签过滤       │
│  └ DetermineTargetVisibility()   - 异步射线检测可见性       │
└────────────────────────┬───────────────────────────────────┘
                         │ 查询碰撞
                         ▼
┌────────────────────────────────────────────────────────────┐
│  UAimAssistTargetComponent : UCapsuleComponent             │
│  (挂在需要被瞄准辅助的 Actor 上)                            │
│  └ GatherTargetOptions() → 返回 FAimAssistTargetOptions    │
│     { ShapeComponent, AssociatedTags, bIsActive }          │
│                                                            │
│  IAimAssistTaget (接口)                                    │
│  └ 任何 Actor/Component 都可以实现此接口成为辅助目标        │
└────────────────────────────────────────────────────────────┘
```

### 12.2 FAimAssistOwnerViewData —— 视口数据容器

```cpp
struct FAimAssistOwnerViewData
{
    const APlayerController* PlayerController;
    const ULocalPlayer* LocalPlayer;
    FMatrix ProjectionMatrix;
    FMatrix ViewProjectionMatrix;
    FIntRect ViewRect;
    FTransform ViewTransform;
    FVector ViewForward;
    FTransform PlayerTransform;
    FTransform PlayerInverseTransform;
    FVector DeltaMovement;  // 帧间位移差
    int32 TeamID;
};
```

每帧更新，缓存视图矩阵、投影矩阵、视口矩形等数据，用于将 3D 目标投影到屏幕空间。

### 12.3 FLyraAimAssistTarget —— 目标状态

```cpp
struct FLyraAimAssistTarget
{
    TWeakObjectPtr<UShapeComponent> TargetShapeComponent;
    FVector Location;
    FVector DeltaMovement;   // 目标帧间位移
    FBox2D ScreenBounds;     // 屏幕空间包围盒
    float ViewDistance;
    float SortScore;
    float AssistTime;        // 累计被辅助时间
    float AssistWeight;      // 归一化权重
    FTraceHandle VisibilityTraceHandle;  // 异步可见性检测句柄
    uint8 bIsVisible : 1;
    uint8 bUnderAssistInnerReticle : 1;
    uint8 bUnderAssistOuterReticle : 1;
};
```

### 12.4 核心处理流程

```cpp
// AimAssistInputModifier.cpp:454-529
FInputActionValue UAimAssistInputModifier::ModifyRaw_Implementation(
    const UEnhancedPlayerInput* PlayerInput,
    FInputActionValue CurrentValue, float DeltaTime)
{
    // 1. 更新视口数据
    OwnerViewData.UpdateViewData(PC);

    // 2. 更新目标数据（双缓冲交换 + 空间查询 + 屏幕投影）
    UpdateTargetData(DeltaTime);

    // 3. 获取基线输入和移动输入
    FVector BaselineInput = CurrentValue.Get<FVector>();
    FVector CurrentMoveInput = MoveInputAction ?
        PlayerInput->GetActionValue(MoveInputAction).Get<FVector>() : FVector::ZeroVector;

    // 4. 计算辅助后的旋转速度
    FRotator LookRates = GetLookRates(BaselineInput);
    const FRotator RotationalVelocity = UpdateRotationalVelocity(
        PC, DeltaTime, BaselineInput, CurrentMoveInput);

    // 5. 反向从旋转速度计算输出值（归一化到 [-1, 1]）
    FVector OutAssistedInput = BaselineInput;
    if (LookRates.Yaw > 0.0f)
        OutAssistedInput.X = FMath::Clamp(RotationalVelocity.Yaw / LookRates.Yaw, -1.0f, 1.0f);
    if (LookRates.Pitch > 0.0f)
        OutAssistedInput.Y = FMath::Clamp(RotationalVelocity.Pitch / LookRates.Pitch, -1.0f, 1.0f);

    return OutAssistedInput;
}
```

### 12.5 目标查询与评分

`UpdateTargetData()` 调用 `UAimAssistTargetManagerComponent::GetVisibleTargets()` 执行以下步骤：

1. **空间查询**：以玩家位置为中心、视椎体为范围，使用 `OverlapMultiByChannel` 在 AimAssist 碰撞通道上检测

2. **接口查询**：对每个命中的 Actor/Component 检查是否实现 `IAimAssistTaget` 接口

3. **过滤**：
   - 排除自身/Instigator
   - 排除同队目标（除非 `bIncludeSameFriendlyTargets`）
   - 排除死亡/濒死目标
   - 排除超出范围的目标
   - 排除被 GameplayTag 排除的目标

4. **屏幕投影**：将目标的 3D 碰撞形状（Box/Sphere/Capsule）投影到屏幕 2D 包围盒

5. **准星判定**：检测屏幕包围盒是否与内/外准星矩形相交

6. **评分排序**：`SortScore = AssistWeightScore + ViewDotScore + ViewDistanceScore`

7. **数量限制**：按评分排序后截取前 `MaxNumberOfTargets`（默认 6）个

8. **可见性检测**：支持 **异步射线检测**（`AsyncLineTraceByChannel`），利用帧间缓存减少延迟

### 12.6 拉拽（Pull）与减速（Slow）

```cpp
void UAimAssistInputModifier::UpdateRotationalVelocity(...)
{
    // 1. 计算每个目标因移动产生的跟踪旋转
    for (const FLyraAimAssistTarget& Target : TargetCache)
    {
        if (Target.bUnderAssistOuterReticle && Target.bIsVisible)
        {
            RotationNeeded += Target.GetRotationFromMovement(OwnerViewData) * Target.AssistWeight;
            CalculateTargetStrengths(Target, TargetPullStrength, TargetSlowStrength);
            PullStrength += TargetPullStrength;
            SlowStrength += TargetSlowStrength;
        }
    }

    // 2. 应用平滑插值
    PullStrength = FMath::FInterpConstantTo(LastPullStrength, PullStrength, DeltaTime, PullLerpRate);
    SlowStrength = FMath::FInterpConstantTo(LastSlowStrength, SlowStrength, DeltaTime, SlowLerpRate);

    // 3. 拉拽：按 RotationNeeded 的百分比拉向目标
    if (Settings.bApplyPull && bIsApplyingAnyInput)
    {
        FRotator PullRotation = RotationNeeded * PullStrength;
        // 横移时仅按横移比例缩放拉拽（防止跑过目标时视角急转）
        if (!bIsApplyingLookInput && Settings.bApplyStrafePullScale)
        {
            float StrafePullScale = FMath::Abs(CurrentMoveInputValue.Y);
            PullRotation.Yaw *= StrafePullScale;
        }
        // FOV 缩放 + 最大转速限制
        RotationalVelocity += PullRotation * (1.0f / DeltaTime);
    }

    // 4. 减速：降低视角转速
    if (Settings.bApplySlowing && bIsApplyingLookInput)
    {
        FRotator SlowRates = LookRates * (1.0f - SlowStrength);
        // 动态减速：向目标方向看时，减少减速强度
        if (Settings.bUseDynamicSlow)
        {
            const float YawDynamicBoost = BoostRotation.Yaw * FMath::Sign(CurrentLookInputValue.X);
            if (YawDynamicBoost > 0.0f)
                SlowRates.Yaw += YawDynamicBoost;
        }
        RotationalVelocity.Yaw += CurrentLookInputValue.X * SlowRates.Yaw;
    }
}
```

**拉拽与减速的强度配置**（区分 Hip 和 ADS）：

| 参数 | Hip（腰射） | ADS（瞄准） |
|------|-----------|-----------|
| PullInnerStrength | 0.6 | 0.7 |
| PullOuterStrength | 0.5 | 0.4 |
| SlowInnerStrength | 0.6 | 0.7 |
| SlowOuterStrength | 0.5 | 0.4 |

### 12.7 IAimAssistTaget 接口

```cpp
// IAimAssistTargetInterface.h
class IAimAssistTaget
{
    GENERATED_BODY()
public:
    virtual void GatherTargetOptions(OUT FAimAssistTargetOptions& TargetData) = 0;
};
```

任何 Actor 或 Component 实现此接口即可成为瞄准辅助目标。`UAimAssistTargetComponent`（继承自 `UCapsuleComponent`）是默认实现，只需添加此组件到角色即可。

---

## 13. 触摸模拟输入：ULyraSimulatedInputWidget

**文件**: `Source/LyraGame/UI/LyraSimulatedInputWidget.h/.cpp`

```cpp
UCLASS(MinimalAPI)
class ULyraSimulatedInputWidget : public UCommonUserWidget
{
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TObjectPtr<const UInputAction> AssociatedAction = nullptr;

    UPROPERTY(BlueprintReadOnly, EditAnywhere)
    FKey FallbackBindingKey = EKeys::Gamepad_Right2D;

    FKey KeyToSimulate;
};
```

### 功能说明

为移动平台提供虚拟摇杆/按钮：

1. **关联 InputAction**：指定此 Widget 对应的 `UInputAction`
2. **查询映射按键**：`QueryKeyToSimulate()` 查询当前映射到该 Action 的实际按键
3. **注入输入值**：通过两种路径注入：
   - 如果有 `AssociatedAction`：`InjectInputVectorForAction()` 直接注入向量
   - 否则：`InputKey()` 模拟按键事件

```cpp
void ULyraSimulatedInputWidget::InputKeyValue(const FVector& Value)
{
    if (AssociatedAction)
    {
        // 直接注入到 Enhanced Input 子系统
        System->InjectInputVectorForAction(AssociatedAction, Value, Modifiers, Triggers);
    }
    else
    {
        // 模拟按键事件
        Input->InputKey(Args);
    }
}
```

支持处理"组合键"（如 `Mouse2D` 由 `MouseX` + `MouseY` 组成），通过 `EKeys::GetPairedKeyDetails()` 分解为子键后分别注入。

---

## 14. PawnData 中的输入配置引用

**文件**: `Source/LyraGame/Character/LyraPawnData.h`

```cpp
UCLASS(MinimalAPI, BlueprintType, Const)
class ULyraPawnData : public UPrimaryDataAsset
{
    // Pawn 蓝图类
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Pawn")
    TSubclassOf<APawn> PawnClass;

    // 能力集
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Abilities")
    TArray<TObjectPtr<ULyraAbilitySet>> AbilitySets;

    // 输入配置 ← 决定此 Pawn 使用哪套 InputTag↔InputAction 映射
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Input")
    TObjectPtr<ULyraInputConfig> InputConfig;

    // 默认摄像机模式
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Camera")
    TSubclassOf<ULyraCameraMode> DefaultCameraMode;
};
```

`ULyraPawnData` 是 Lyra 数据驱动架构的核心，不同的游戏模式可以指定不同的 PawnData，从而让不同模式的角色拥有完全不同的输入配置、能力集和摄像机模式。

---

## 15. 完整数据流总结

### 15.1 初始化阶段

```
Experience 加载完成
  → PawnData 确定 (包含 InputConfig)
  → Pawn Spawn
    → LyraHeroComponent::BeginPlay()
      → 状态链推进: Spawned → DataAvailable → DataInitialized
        → HandleChangeInitState(DataAvailable → DataInitialized)
          → InitializePlayerInput()
            → ClearAllMappings()
            → 注册 DefaultInputMappings (IMC)
            → BindAbilityActions() (能力输入)
            → BindNativeAction() × 5 (原生输入)
            → bReadyToBindInputs = true
            → 广播 NAME_BindInputsNow
              → GameFeature 收到事件
                → AddInputContextMapping → 追加 IMC
                → AddInputBinding → 追加 InputConfig
```

### 15.2 运行时输入处理（每帧）

```
硬件输入事件
  → UEnhancedPlayerInput::InputKey()
    → ULyraPlayerInput::InputKey()  (延迟标记)
  → Enhanced Input System 处理
    → InputMappingContext 匹配
    → InputModifier 链处理
      → DeadZone → Sensitivity → AimInversion → AimAssist
    → InputTrigger 判定
    → 触发 ETriggerEvent

对于 NativeAction:
  ETriggerEvent::Triggered
    → ULyraHeroComponent::Input_Move()
    → 或 Input_LookMouse() / Input_LookStick() / Input_Crouch() / Input_AutoRun()
    → Pawn->AddMovementInput() / AddControllerYawInput() 等

对于 AbilityAction:
  ETriggerEvent::Triggered
    → ULyraHeroComponent::Input_AbilityInputTagPressed(InputTag)
      → ULyraAbilitySystemComponent::AbilityInputTagPressed(InputTag)
        → 遍历 ActivatableAbilities，匹配 DynamicSpecSourceTags
        → 加入 InputPressedSpecHandles + InputHeldSpecHandles

  ETriggerEvent::Completed
    → ULyraHeroComponent::Input_AbilityInputTagReleased(InputTag)
      → ULyraAbilitySystemComponent::AbilityInputTagReleased(InputTag)
        → 加入 InputReleasedSpecHandles，移出 InputHeldSpecHandles

  每帧 Tick:
    → ULyraAbilitySystemComponent::ProcessAbilityInput()
      → 检查 TAG_Gameplay_AbilityInputBlocked
      → 处理 Held (WhileInputActive)
      → 处理 Pressed (OnInputTriggered)
      → TryActivateAbility()
      → 处理 Released
      → 清空缓冲
```

### 15.3 瞄准辅助处理流程

```
手柄右摇杆输入
  → Enhanced Input Modifier 链
    → ... (DeadZone, Sensitivity, AimInversion)
    → UAimAssistInputModifier::ModifyRaw_Implementation()
      → OwnerViewData.UpdateViewData(PC)
      → UpdateTargetData(DeltaTime)
        → SwapTargetCaches()
        → TargetManager->GetVisibleTargets()
          → OverlapMultiByChannel() (AimAssist 碰撞通道)
          → GatherTargetOptions() (IAimAssistTaget 接口)
          → DoesTargetPassFilter() (团队/距离/生死/标签过滤)
          → ProjectShapeToScreen() (3D→2D 投影)
          → 准星交叉判定 (Inner/Outer)
          → 评分排序 + 数量截断
          → DetermineTargetVisibility() (异步射线)
        → 更新权重 (时间加权 + 归一化)
      → UpdateRotationalVelocity()
        → 计算跟踪旋转 (目标移动 + 玩家移动)
        → CalculateTargetStrengths() (Inner/Outer × Hip/ADS)
        → 平滑插值 Pull/Slow
        → 应用 Pull (视角拉向目标)
        → 应用 Slow (减慢经过目标的转速)
      → 输出辅助后的归一化输入值
```

---

## 16. 关键源文件清单

### 核心输入模块

| 文件 | 职责 |
|------|------|
| `Source/LyraGame/Input/LyraInputConfig.h/.cpp` | InputAction↔InputTag 映射数据资产 |
| `Source/LyraGame/Input/LyraInputComponent.h/.cpp` | 扩展的输入组件，提供 Tag 绑定模板方法 |
| `Source/LyraGame/Input/LyraInputModifiers.h/.cpp` | 4 个自定义 InputModifier（死区/灵敏度/反转/属性缩放） |
| `Source/LyraGame/Input/LyraAimSensitivityData.h/.cpp` | 灵敏度枚举→浮点映射数据表 |
| `Source/LyraGame/Input/LyraPlayerInput.h/.cpp` | 自定义 PlayerInput，延迟标记功能 |
| `Source/LyraGame/Input/LyraInputUserSettings.h/.cpp` | 输入用户设置基类 + 可映射按键设置 |
| `Source/LyraGame/Input/LyraPlayerMappableKeyProfile.h/.cpp` | 键位配置档案 |

### 输入初始化与桥接

| 文件 | 职责 |
|------|------|
| `Source/LyraGame/Character/LyraHeroComponent.h/.cpp` | 输入初始化、原生输入处理、能力输入路由 |
| `Source/LyraGame/Character/LyraPawnData.h` | PawnData 中持有 InputConfig 引用 |
| `Source/LyraGame/LyraGameplayTags.h/.cpp` | 定义 InputTag 命名空间下的 Tag |
| `Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.h/.cpp` | 能力输入收集、处理、激活 |

### GameFeature 输入注入

| 文件 | 职责 |
|------|------|
| `Source/LyraGame/GameFeatures/GameFeatureAction_AddInputContextMapping.h/.cpp` | 动态注入 IMC |
| `Source/LyraGame/GameFeatures/GameFeatureAction_AddInputBinding.h/.cpp` | 动态注入 InputConfig |

### 瞄准辅助系统

| 文件 | 职责 |
|------|------|
| `Plugins/.../ShooterCore/.../Input/AimAssistInputModifier.h/.cpp` | 瞄准辅助 InputModifier |
| `Plugins/.../ShooterCore/.../Input/IAimAssistTargetInterface.h` | 辅助目标接口 |
| `Plugins/.../ShooterCore/.../Input/AimAssistTargetComponent.h/.cpp` | 辅助目标默认组件 |
| `Plugins/.../ShooterCore/.../Input/AimAssistTargetManagerComponent.h/.cpp` | 全局目标管理器 |

### 其他

| 文件 | 职责 |
|------|------|
| `Source/LyraGame/UI/LyraSimulatedInputWidget.h/.cpp` | 触摸平台虚拟输入 Widget |
| `Source/LyraGame/Player/LyraLocalPlayer.h` | 持有 SharedSettings，提供输入设置访问 |
