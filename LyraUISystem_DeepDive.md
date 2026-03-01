# Lyra UI 系统深度分析

## 目录
1. [系统概述与架构总览](#1-系统概述与架构总览)
2. [插件基础层 - CommonGame](#2-插件基础层---commongame)
3. [插件基础层 - UIExtension](#3-插件基础层---uiextension)
4. [游戏UI核心层](#4-游戏ui核心层)
5. [前端流程状态机](#5-前端流程状态机)
6. [GameFeature与UI的桥梁](#6-gamefeature与ui的桥梁)
7. [Indicator系统（世界空间指示器）](#7-indicator系统世界空间指示器)
8. [武器UI系统](#8-武器ui系统)
9. [移动端触控UI](#9-移动端触控ui)
10. [基础UI组件](#10-基础ui组件)
11. [性能统计UI](#11-性能统计ui)
12. [设置系统UI关联](#12-设置系统ui关联)
13. [完整架构图](#13-完整架构图)
14. [核心设计模式总结](#14-核心设计模式总结)

---

## 1. 系统概述与架构总览

### 1.1 三层架构

Lyra的UI系统采用分层架构，由底向上分为三层：

```
┌──────────────────────────────────────────────────────┐
│           游戏UI层 (Source/LyraGame/UI/)              │
│   LyraHUD / LyraHUDLayout / WeaponUI / Indicator    │
├──────────────────────────────────────────────────────┤
│           插件基础层 (Plugins/)                        │
│   CommonGame / UIExtension / CommonLoadingScreen     │
├──────────────────────────────────────────────────────┤
│           引擎层 (CommonUI / UMG / Slate)             │
│   UCommonActivatableWidget / UCommonUserWidget       │
└──────────────────────────────────────────────────────┘
```

### 1.2 核心管理链

```
UGameInstance
  └── ULyraUIManagerSubsystem (GameInstanceSubsystem)
        └── ULyraUIPolicy (UGameUIPolicy子类)
              └── UPrimaryGameLayout (每个LocalPlayer一个)
                    ├── UI.Layer.Game     (游戏HUD)
                    ├── UI.Layer.GameMenu (游戏菜单)
                    ├── UI.Layer.Menu     (前端菜单)
                    └── UI.Layer.Modal    (模态对话框)
```

### 1.3 关键设计理念

- **CommonUI驱动**：所有UI基于Epic的CommonUI插件，统一处理键鼠/手柄/触控多输入
- **Layer Stack模型**：UI通过GameplayTag标识的Layer组织，每层维护激活栈
- **UIExtension发布-订阅**：GameFeature通过Tag动态注入Widget到布局扩展点
- **ControlFlow状态机**：前端流程由ControlFlow插件驱动状态切换

---

## 2. 插件基础层 - CommonGame

### 2.1 UGameUIManagerSubsystem

**文件**: `Plugins/CommonGame/Source/Public/GameUIManagerSubsystem.h/.cpp`

作为`UGameInstanceSubsystem`，是整个UI管理系统的入口：

```cpp
UCLASS()
class UGameUIManagerSubsystem : public UGameInstanceSubsystem
{
    // 当前UI策略类（由子类配置）
    UPROPERTY(Transient)
    TObjectPtr<UGameUIPolicy> CurrentPolicy;

    // 默认UI策略类
    UPROPERTY(config, EditAnywhere)
    TSoftClassPtr<UGameUIPolicy> DefaultUIPolicyClass;
};
```

**核心职责**：
- 维护`UGameUIPolicy`的生命周期
- 监听LocalPlayer的创建/移除，触发Layout创建
- 提供`GetCurrentUIPolicy()`全局访问点

**子类LyraUIManagerSubsystem扩展**：

```cpp
// LyraUIManagerSubsystem.cpp - Tick中同步HUD可见性
void ULyraUIManagerSubsystem::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    // 遍历所有LocalPlayer，将PrimaryGameLayout的可见性与AHUD::bShowHUD同步
    const UGameUIPolicy* Policy = GetCurrentUIPolicy();
    // ...
    for (const ULocalPlayer* LocalPlayer : GameInstance->GetLocalPlayers())
    {
        if (const AHUD* HUD = PC->GetHUD())
        {
            if (UPrimaryGameLayout* RootLayout = Policy->GetRootLayout(CastChecked<UCommonLocalPlayer>(LocalPlayer)))
            {
                const ESlateVisibility DesiredVisibility = HUD->bShowHUD 
                    ? ESlateVisibility::SelfHitTestInvisible 
                    : ESlateVisibility::Collapsed;
                if (RootLayout->GetVisibility() != DesiredVisibility)
                {
                    RootLayout->SetVisibility(DesiredVisibility);
                }
            }
        }
    }
}
```

### 2.2 UGameUIPolicy

**文件**: `Plugins/CommonGame/Source/Public/GameUIPolicy.h/.cpp`

为每个`ULocalPlayer`创建并管理`UPrimaryGameLayout`：

```cpp
UCLASS()
class UGameUIPolicy : public UObject
{
    // 每个LocalPlayer对应的RootLayout和ViewportClient
    UPROPERTY(Transient)
    TArray<FRootViewportLayoutInfo> RootViewportLayouts;

    // Layout的Widget类
    UPROPERTY(EditAnywhere)
    TSoftClassPtr<UPrimaryGameLayout> LayoutClass;
};
```

**Layout创建流程**：

```cpp
void UGameUIPolicy::NotifyPlayerAdded(UCommonLocalPlayer* LocalPlayer)
{
    // 获取或创建Layout
    FRootViewportLayoutInfo& LayoutInfo = RootViewportLayouts[..];
    if (!LayoutInfo.RootLayout)
    {
        TSubclassOf<UPrimaryGameLayout> LayoutWidgetClass = LayoutClass.LoadSynchronous();
        LayoutInfo.RootLayout = CreateWidget<UPrimaryGameLayout>(LocalPlayer, LayoutWidgetClass);
        LayoutInfo.RootLayout->SetPlayerContext(FLocalPlayerContext(LocalPlayer));
        LayoutInfo.RootLayout->AddToPlayerScreen(1000); // 高ZOrder确保在最上层
    }
}
```

**多人分屏支持**（`ELocalMultiplayerInteractionMode`）：
- `PrimaryOnly`：仅主玩家显示UI
- `SingleToggle`：单界面在玩家间切换
- `Simultaneous`：所有玩家同时显示

### 2.3 UPrimaryGameLayout

**文件**: `Plugins/CommonGame/Source/Public/PrimaryGameLayout.h/.cpp`

根UI Widget，维护Tag到Layer的映射：

```cpp
UCLASS()
class UPrimaryGameLayout : public UCommonUserWidget
{
    // Tag -> Layer Widget 映射
    UPROPERTY(Transient, meta=(Categories="UI.Layer"))
    TMap<FGameplayTag, TObjectPtr<UCommonActivatableWidgetContainerBase>> Layers;

    // Widget注册到Layer中
    void RegisterLayer(FGameplayTag LayerTag, UCommonActivatableWidgetContainerBase* LayerWidget);
};
```

**Push/Pop Widget到Layer**：

```cpp
// 同步推送
UCommonActivatableWidget* UPrimaryGameLayout::PushWidgetToLayerStack(
    FGameplayTag LayerName, 
    UClass* ActivatableWidgetClass)
{
    // 查找对应Layer的WidgetContainer
    if (UCommonActivatableWidgetContainerBase* Layer = Layers.FindRef(LayerName))
    {
        return Layer->AddWidget(*ActivatableWidgetClass);
    }
    return nullptr;
}

// 异步推送（等待类加载）
void UPrimaryGameLayout::PushWidgetToLayerStackAsync<AsyncCallerContext>(
    FGameplayTag LayerName,
    TSoftClassPtr<UCommonActivatableWidget> ActivatableWidgetClass)
{
    // 使用StreamableManager异步加载类
    // 加载完成后 suspend input 防止过渡期间操作
    FStreamableManager::RequestAsyncLoad(..., [this, LayerName]()
    {
        // 暂停输入
        SuspendInputForPlayer(...);
        // 推送Widget
        PushWidgetToLayerStack(LayerName, WidgetClass);
    });
}
```

**输入暂停机制**（防止Layer切换期间的误操作）：

```cpp
void UPrimaryGameLayout::SuspendInputForPlayer(ULocalPlayer* LocalPlayer, FName SuspendReason)
{
    // 通过CommonUIActionRouter暂停输入
}

void UPrimaryGameLayout::ResumeInputForPlayer(ULocalPlayer* LocalPlayer, FName SuspendReason)
{
    // 恢复输入
}
```

### 2.4 UCommonUIExtensions

**文件**: `Plugins/CommonGame/Source/Public/CommonUIExtensions.h/.cpp`

蓝图静态函数库，提供便捷的UI操作：

```cpp
UCLASS()
class UCommonUIExtensions : public UBlueprintFunctionLibrary
{
    // 将Widget推送到指定Layer
    UFUNCTION(BlueprintCallable)
    static UCommonActivatableWidget* PushContentToLayer_ForPlayer(
        ULocalPlayer* LocalPlayer,
        FGameplayTag LayerName,
        TSubclassOf<UCommonActivatableWidget> WidgetClass);

    // 推送流式内容（异步加载）
    UFUNCTION(BlueprintCallable)
    static void PushStreamedContentToLayer_ForPlayer(
        ULocalPlayer* LocalPlayer,
        FGameplayTag LayerName,
        TSoftClassPtr<UCommonActivatableWidget> WidgetClass);

    // 暂停/恢复输入
    UFUNCTION(BlueprintCallable)
    static void SuspendInputForPlayer(ULocalPlayer* LocalPlayer, FName SuspendReason);

    UFUNCTION(BlueprintCallable)
    static void ResumeInputForPlayer(ULocalPlayer* LocalPlayer, FName SuspendReason);
};
```

### 2.5 消息系统

**UCommonMessagingSubsystem** (`Plugins/CommonGame/Source/Public/Messaging/`)：

```cpp
UCLASS()
class UCommonMessagingSubsystem : public ULocalPlayerSubsystem
{
    virtual void ShowConfirmation(UCommonGameDialogDescriptor* DialogDescriptor,
        FCommonMessagingResultDelegate ResultCallback);
    virtual void ShowError(UCommonGameDialogDescriptor* DialogDescriptor,
        FCommonMessagingResultDelegate ResultCallback);
};
```

**UCommonGameDialogDescriptor** - 对话框描述数据：

```cpp
UCLASS()
class UCommonGameDialogDescriptor : public UObject
{
    UPROPERTY() FText Header;
    UPROPERTY() FText Body;
    UPROPERTY() TArray<FConfirmationDialogAction> ButtonActions;
    // 工厂方法
    static UCommonGameDialogDescriptor* CreateConfirmationOk(FText Header, FText Body);
    static UCommonGameDialogDescriptor* CreateConfirmationOkCancel(FText Header, FText Body);
    static UCommonGameDialogDescriptor* CreateConfirmationYesNo(FText Header, FText Body);
    static UCommonGameDialogDescriptor* CreateConfirmationYesNoCancel(FText Header, FText Body);
};
```

---

## 3. 插件基础层 - UIExtension

### 3.1 核心概念

**文件**: `Plugins/UIExtension/Source/Public/UIExtensionSystem.h/.cpp`

UIExtension系统实现了一个**发布-订阅模式**，允许GameFeature在运行时动态注入UI Widget：

- **ExtensionPoint**（订阅方）：布局中的"插槽"，通过GameplayTag标识
- **Extension**（发布方）：要注入的Widget，绑定到匹配的Tag

### 3.2 UUIExtensionSubsystem

```cpp
UCLASS()
class UUIExtensionSubsystem : public UWorldSubsystem
{
    // 注册扩展点（布局端调用）
    FUIExtensionPointHandle RegisterExtensionPoint(
        const FGameplayTag& ExtensionPointTag,
        EUIExtensionPointMatch ExtensionPointTagMatchType,
        const TArray<UClass*>& AllowedDataClasses,
        FExtendExtensionPointDelegate ExtensionCallback);

    // 注册扩展（GameFeature端调用）
    FUIExtensionHandle RegisterExtensionAsWidgetForContext(
        const FGameplayTag& ExtensionPointTag,
        UObject* ContextObject,
        TSubclassOf<UUserWidget> WidgetClass,
        int32 Priority);
    
    // 注册扩展（数据对象）
    FUIExtensionHandle RegisterExtensionAsData(
        const FGameplayTag& ExtensionPointTag,
        UObject* ContextObject,
        UObject* Data,
        int32 Priority);
};
```

### 3.3 Tag匹配机制

```cpp
UENUM()
enum class EUIExtensionPointMatch : uint8
{
    ExactMatch,      // 精确匹配
    PartialMatch     // 部分匹配（父Tag也可匹配）
};

// 匹配逻辑
bool FUIExtensionPoint::DoesExtensionPassContract(const FUIExtension* Extension) const
{
    // 1. Tag匹配检查
    if (TagMatchType == EUIExtensionPointMatch::ExactMatch)
    {
        if (Extension->ExtensionPointTag != ExtensionPointTag)
            return false;
    }
    else // PartialMatch
    {
        if (!Extension->ExtensionPointTag.MatchesTag(ExtensionPointTag))
            return false;
    }
    
    // 2. 数据类型契约检查
    if (AllowedDataClasses.Num() > 0)
    {
        for (const UClass* AllowedClass : AllowedDataClasses)
        {
            if (Extension->Data->IsA(AllowedClass))
                return true;
        }
        return false;
    }
    return true;
}
```

### 3.4 UUIExtensionPointWidget

**文件**: `Plugins/UIExtension/Source/Public/Widgets/UIExtensionPointWidget.h/.cpp`

UMG Widget，放入布局蓝图中作为扩展点：

```cpp
UCLASS()
class UUIExtensionPointWidget : public UDynamicEntryBoxBase
{
    UPROPERTY(EditAnywhere, meta=(Categories="UI.Extension"))
    FGameplayTag ExtensionPointTag;

    UPROPERTY(EditAnywhere)
    EUIExtensionPointMatch ExtensionPointTagMatch = EUIExtensionPointMatch::ExactMatch;

    UPROPERTY(EditAnywhere)
    TArray<TObjectPtr<UClass>> DataClasses;

protected:
    virtual void OnWidgetRebuilt() override
    {
        // 注册到UIExtensionSubsystem
        // 当有Extension注册时，回调创建子Widget
        ExtensionPointHandle = ExtensionSubsystem->RegisterExtensionPoint(
            ExtensionPointTag, ExtensionPointTagMatch, DataClasses,
            FExtendExtensionPointDelegate::CreateUObject(this, &ThisClass::OnAddOrRemoveExtension)
        );
    }

    void OnAddOrRemoveExtension(EUIExtensionAction Action, const FUIExtensionRequest& ExtensionRequest)
    {
        if (Action == EUIExtensionAction::Added)
        {
            UUserWidget* Widget = CreateEntryInternal(ExtensionRequest.WidgetClass);
            // 存储映射关系
        }
        else // Removed
        {
            // 移除对应Widget
        }
    }
};
```

---

## 4. 游戏UI核心层

### 4.1 ULyraUIManagerSubsystem

**文件**: `Source/LyraGame/UI/Subsystem/LyraUIManagerSubsystem.h/.cpp`

继承`UGameUIManagerSubsystem`，增加Tick同步HUD可见性（见2.1）。

### 4.2 ULyraUIMessaging

**文件**: `Source/LyraGame/UI/Subsystem/LyraUIMessaging.h/.cpp`

继承`UCommonMessagingSubsystem`，将对话框推送到Modal层：

```cpp
void ULyraUIMessaging::ShowConfirmation(
    UCommonGameDialogDescriptor* DialogDescriptor,
    FCommonMessagingResultDelegate ResultCallback)
{
    // 将ConfirmationDialog推送到UI.Layer.Modal
    if (UCommonActivatableWidget* Dialog = UCommonUIExtensions::PushContentToLayer_ForPlayer(
        GetLocalPlayer(), 
        FGameplayTag::RequestGameplayTag(FName("UI.Layer.Modal")),
        ConfirmationDialogClassPtr))
    {
        ULyraConfirmationScreen* Confirmation = CastChecked<ULyraConfirmationScreen>(Dialog);
        Confirmation->SetupDialog(DialogDescriptor, ResultCallback);
    }
}
```

### 4.3 ALyraHUD

**文件**: `Source/LyraGame/UI/LyraHUD.h/.cpp`

极简的HUD Actor，主要用途：
- 承载`UGameFrameworkComponentManager`的组件扩展
- 提供调试渲染接口

```cpp
UCLASS()
class ALyraHUD : public AHUD
{
    // 通过GameFrameworkComponentManager扩展
    // GameFeature可以向HUD添加组件
};
```

### 4.4 ULyraHUDLayout

**文件**: `Source/LyraGame/UI/LyraHUDLayout.h/.cpp`

HUD布局Widget，处理Escape菜单和控制器断连：

```cpp
UCLASS()
class ULyraHUDLayout : public ULyraActivatableWidget
{
    UPROPERTY(EditDefaultsOnly)
    TSoftClassPtr<UCommonActivatableWidget> EscapeMenuClass;

    // 按下Escape时推送菜单到GameMenu层
    void HandleEscapeAction()
    {
        if (ensure(!EscapeMenuClass.IsNull()))
        {
            UCommonUIExtensions::PushStreamedContentToLayer_ForPlayer(
                GetOwningLocalPlayer(),
                FGameplayTag::RequestGameplayTag(FName("UI.Layer.GameMenu")),
                EscapeMenuClass);
        }
    }

    // 蓝图事件：控制器断开连接
    UFUNCTION(BlueprintImplementableEvent)
    void OnControllerDisconnected();

    // 蓝图事件：控制器重连
    UFUNCTION(BlueprintImplementableEvent)
    void OnControllerReconnected();
};
```

### 4.5 ULyraActivatableWidget

**文件**: `Source/LyraGame/UI/LyraActivatableWidget.h/.cpp`

继承`UCommonActivatableWidget`，增加输入模式配置：

```cpp
UENUM()
enum class ELyraWidgetInputMode : uint8
{
    Default,       // 不改变输入模式
    GameAndMenu,   // 同时接受游戏和菜单输入
    Game,          // 仅游戏输入
    Menu           // 仅菜单输入
};

UCLASS()
class ULyraActivatableWidget : public UCommonActivatableWidget
{
    UPROPERTY(EditDefaultsOnly)
    ELyraWidgetInputMode InputConfig = ELyraWidgetInputMode::Default;

    virtual TOptional<FUIInputConfig> GetDesiredInputConfig() const override
    {
        switch (InputConfig)
        {
        case ELyraWidgetInputMode::GameAndMenu:
            return FUIInputConfig(ECommonInputMode::All, EMouseCaptureMode::NoCapture);
        case ELyraWidgetInputMode::Game:
            return FUIInputConfig(ECommonInputMode::Game, EMouseCaptureMode::CapturePermanently);
        case ELyraWidgetInputMode::Menu:
            return FUIInputConfig(ECommonInputMode::Menu, EMouseCaptureMode::NoCapture);
        default:
            return TOptional<FUIInputConfig>();
        }
    }
};
```

### 4.6 ULyraTaggedWidget

**文件**: `Source/LyraGame/UI/LyraTaggedWidget.h/.cpp`

基于GameplayTag驱动可见性的Widget：

```cpp
UCLASS()
class ULyraTaggedWidget : public UCommonUserWidget
{
    // 当玩家拥有这些Tag中的任意一个时隐藏
    UPROPERTY(EditAnywhere, meta=(Categories="UI.HiddenBy"))
    FGameplayTagContainer HiddenByTags;

    // 当前是否应该显示
    void OnWatchedTagsChanged()
    {
        // 检查AbilitySystemComponent上的Tags
        const bool bHasAnyHiddenTags = ASC->HasAnyMatchingGameplayTags(HiddenByTags);
        SetVisibility(bHasAnyHiddenTags ? ESlateVisibility::Collapsed : ESlateVisibility::SelfHitTestInvisible);
    }
};
```

> **注意**：源码中Tag监听部分标记了`@TODO`，实际的Tag变化回调注册尚未完成。

### 4.7 ULyraGameViewportClient

**文件**: `Source/LyraGame/UI/LyraGameViewportClient.h/.cpp`

按平台选择硬件/软件光标：

```cpp
UCLASS()
class ULyraGameViewportClient : public UGameViewportClient
{
    virtual bool GetUseMouseForTouch() const override;

    // 根据平台决定使用软件光标还是硬件光标
    // 主机平台使用软件光标以支持UI导航
};
```

---

## 5. 前端流程状态机

### 5.1 ULyraFrontendStateComponent

**文件**: `Source/LyraGame/UI/Frontend/LyraFrontendStateComponent.h/.cpp`

基于ControlFlow插件的前端状态机，管理从启动到进入主菜单的流程：

```cpp
UCLASS()
class ULyraFrontendStateComponent : public UGameFrameworkComponent
{
    UPROPERTY(EditAnywhere)
    TArray<FLyraFrontendStateEntry> FrontendStateEntries;

    // 状态流程：
    // 1. Startup → 用户初始化
    // 2. PressAnyKey → 按键继续
    // 3. JoinSession → 加入会话  
    // 4. MainScreen → 主菜单界面

    void FlowStep_TryShowPressStartScreen(FControlFlowNodeRef SubFlow, ...);
    void FlowStep_TryJoinRequestedSession(FControlFlowNodeRef SubFlow, ...);
    void FlowStep_TryShowMainScreen(FControlFlowNodeRef SubFlow, ...);
};
```

**状态条目定义**：

```cpp
USTRUCT()
struct FLyraFrontendStateEntry
{
    // 要推送到Layer的Widget类
    UPROPERTY(EditAnywhere)
    TSoftClassPtr<UCommonActivatableWidget> WidgetClass;

    // 目标Layer Tag
    UPROPERTY(EditAnywhere, meta=(Categories="UI.Layer"))
    FGameplayTag LayerTag;

    // 是否需要等待PlayerController
    UPROPERTY(EditAnywhere)
    bool bWaitForPlayerController = false;
};
```

**Flow执行逻辑**：

```cpp
void ULyraFrontendStateComponent::OnExperienceLoaded(const ULyraExperienceDefinition*)
{
    // 创建ControlFlow
    FControlFlow& Flow = FControlFlowStatics::Create(this, TEXT("FrontendFlow"));
    
    Flow.QueueStep(TEXT("Try Show Press Start Screen"), this, 
        &ThisClass::FlowStep_TryShowPressStartScreen);
    Flow.QueueStep(TEXT("Try Join Requested Session"), this, 
        &ThisClass::FlowStep_TryJoinRequestedSession);
    Flow.QueueStep(TEXT("Try Show Main Screen"), this, 
        &ThisClass::FlowStep_TryShowMainScreen);

    Flow.ExecuteFlow();
}

void ULyraFrontendStateComponent::FlowStep_TryShowMainScreen(
    FControlFlowNodeRef SubFlow, ...)
{
    // 推送配置的Widget到对应Layer
    for (const FLyraFrontendStateEntry& Entry : FrontendStateEntries)
    {
        UCommonUIExtensions::PushStreamedContentToLayer_ForPlayer(
            LocalPlayer, Entry.LayerTag, Entry.WidgetClass);
    }
    SubFlow->ContinueFlow();
}
```

---

## 6. GameFeature与UI的桥梁

### 6.1 UGameFeatureAction_AddWidgets

**文件**: `Source/LyraGame/GameFeatures/GameFeatureAction_AddWidget.h/.cpp`

GameFeature Action，在Experience加载时注入UI：

```cpp
USTRUCT()
struct FLyraHUDLayoutRequest
{
    // 要推送到Layer的Layout类
    UPROPERTY(EditAnywhere)
    TSoftClassPtr<UCommonActivatableWidget> LayoutClass;

    // 目标Layer
    UPROPERTY(EditAnywhere, meta=(Categories="UI.Layer"))
    FGameplayTag LayerID;
};

USTRUCT()
struct FLyraHUDElementEntry
{
    // 要注册为Extension的Widget类
    UPROPERTY(EditAnywhere)
    TSoftClassPtr<UUserWidget> WidgetClass;

    // 目标Extension Point Tag
    UPROPERTY(EditAnywhere, meta=(Categories="UI.Extension"))
    FGameplayTag SlotID;
};

UCLASS()
class UGameFeatureAction_AddWidgets : public UGameFeatureAction_WorldActionBase
{
    UPROPERTY(EditAnywhere)
    TArray<FLyraHUDLayoutRequest> Layout;

    UPROPERTY(EditAnywhere)
    TArray<FLyraHUDElementEntry> Widgets;
};
```

**注入流程**：

```cpp
void UGameFeatureAction_AddWidgets::AddToWorld(const FWorldContext& WorldContext, ...)
{
    // 监听ALyraHUD的创建（通过GameFrameworkComponentManager）
    // 当HUD Actor就绪时触发回调
    ComponentRequestHandles.Add(
        Manager->AddExtensionHandler(
            ALyraHUD::StaticClass(),
            UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
                this, &ThisClass::HandleActorExtension)));
}

void UGameFeatureAction_AddWidgets::HandleActorExtension(
    AActor* Actor, FName EventName, ...)
{
    if (EventName == UGameFrameworkComponentManager::NAME_GameActorReady)
    {
        ALyraHUD* HUD = CastChecked<ALyraHUD>(Actor);
        ULocalPlayer* LocalPlayer = HUD->GetOwningPlayerController()->GetLocalPlayer();

        // 1. 推送Layout到指定Layer
        for (const FLyraHUDLayoutRequest& LayoutEntry : Layout)
        {
            TSubclassOf<UCommonActivatableWidget> ConcreteWidgetClass = LayoutEntry.LayoutClass.Get();
            if (UPrimaryGameLayout* RootLayout = 
                UPrimaryGameLayout::GetPrimaryGameLayoutForPrimaryPlayer(LocalPlayer))
            {
                RootLayout->PushWidgetToLayerStack(LayoutEntry.LayerID, ConcreteWidgetClass);
            }
        }

        // 2. 注册Widget到UIExtension系统
        UUIExtensionSubsystem* ExtensionSubsystem = HUD->GetWorld()->GetSubsystem<UUIExtensionSubsystem>();
        for (const FLyraHUDElementEntry& Entry : Widgets)
        {
            ExtensionHandles.Add(
                ExtensionSubsystem->RegisterExtensionAsWidgetForContext(
                    Entry.SlotID, LocalPlayer, Entry.WidgetClass.Get(), -1));
        }
    }
}
```

### 6.2 完整注入流程

```
Experience加载完成
  → GameFeatureAction_AddWidgets激活
    → 监听ALyraHUD就绪
      → HUD Actor创建
        → 推送Layout(如W_HUDLayout)到UI.Layer.Game
          → Layout中包含UIExtensionPointWidget(如Tag: UI.Extension.HUD.Reticle)
            → UIExtensionSubsystem匹配已注册的Extension
              → 动态创建武器准心Widget
```

---

## 7. Indicator系统（世界空间指示器）

### 7.1 概述

Indicator系统将3D世界空间中的Actor投影到2D屏幕空间，支持屏幕边缘夹紧和箭头指示。

### 7.2 ULyraIndicatorManagerComponent

**文件**: `Source/LyraGame/UI/IndicatorSystem/LyraIndicatorManagerComponent.h/.cpp`

Controller组件，管理指示器描述符集合：

```cpp
UCLASS()
class ULyraIndicatorManagerComponent : public UControllerComponent
{
    UPROPERTY()
    TArray<TObjectPtr<UIndicatorDescriptor>> Indicators;

    UIndicatorDescriptor* AddIndicator(AActor* IndicatorActor);
    void RemoveIndicator(UIndicatorDescriptor* Indicator);

    // 委托通知添加/移除
    FIndicatorEvent OnIndicatorAdded;
    FIndicatorEvent OnIndicatorRemoved;
};
```

### 7.3 UIndicatorDescriptor

**文件**: `Source/LyraGame/UI/IndicatorSystem/IndicatorDescriptor.h/.cpp`

完整的指示器描述数据：

```cpp
UCLASS(BlueprintType)
class UIndicatorDescriptor : public UObject
{
    // 关联的Actor
    UPROPERTY() TObjectPtr<AActor> DataObject;

    // 附着的场景组件和Socket
    UPROPERTY() TObjectPtr<USceneComponent> Component;
    UPROPERTY() FName ComponentSocketName;

    // 屏幕空间偏移
    UPROPERTY() FVector BoundingBoxAnchor = FVector(0.5, 0.5, 1.0);
    UPROPERTY() FVector2D ScreenSpaceOffset;

    // 可见性控制
    UPROPERTY() bool bVisible = true;
    UPROPERTY() bool bClampToScreen = false;
    UPROPERTY() bool bShowClampToScreenArrow = false;
    UPROPERTY() bool bOverrideScreenPosition = false;

    // 自动移除
    UPROPERTY() bool bAutoRemoveWhenIndicatorComponentIsNull = true;

    // 投影优先级
    UPROPERTY() int32 Priority = 0;

    // HAlign / VAlign
    UPROPERTY() EHorizontalAlignment HAlignment = EHorizontalAlignment::HAlign_Center;
    UPROPERTY() EVerticalAlignment VAlignment = EVerticalAlignment::VAlign_Center;

    // 指示器Widget类
    UPROPERTY() TSoftClassPtr<UUserWidget> IndicatorWidgetClass;
};
```

### 7.4 SActorCanvas（核心Slate面板）

**文件**: `Source/LyraGame/UI/IndicatorSystem/SActorCanvas.h/.cpp`

自定义Slate面板，每帧执行投影计算和Widget管理：

```cpp
class SActorCanvas : public SPanel
{
    // Widget池 - 避免频繁创建/销毁
    struct FIndicatorPool
    {
        TArray<TSharedPtr<SWidget>> FreePool;
        TMap<UIndicatorDescriptor*, TSharedPtr<SWidget>> ActiveMap;

        TSharedPtr<SWidget> Acquire(UIndicatorDescriptor* Descriptor);
        void Release(UIndicatorDescriptor* Descriptor);
    };

    // 每种Widget类型一个池
    TMap<TSubclassOf<UUserWidget>, FIndicatorPool> IndicatorPools;

    // 每帧布局计算
    virtual void OnArrangeChildren(
        const FGeometry& AllottedGeometry, 
        FArrangedChildren& ArrangedChildren) const override;

    // 可选的箭头Widget（屏幕边缘指示）
    TSubclassOf<UUserWidget> ArrowWidgetClass;
};
```

**投影计算流程**（`OnArrangeChildren`中每帧执行）：

```cpp
void SActorCanvas::OnArrangeChildren(const FGeometry& AllottedGeometry, 
    FArrangedChildren& ArrangedChildren) const
{
    for (UIndicatorDescriptor* Indicator : Indicators)
    {
        if (!Indicator->GetIsVisible()) continue;

        // 1. 获取世界位置
        FVector WorldPosition;
        if (Indicator->GetComponent())
        {
            WorldPosition = Indicator->GetComponent()->GetSocketLocation(
                Indicator->GetComponentSocketName());
        }

        // 2. 使用BoundingBoxAnchor调整位置（基于Actor包围盒）
        // 3. WorldToScreen投影
        FVector2D ScreenPosition;
        bool bIsOnScreen = FSceneView::ProjectWorldToScreen(
            WorldPosition, ViewRect, ViewProjectionMatrix, ScreenPosition);

        // 4. 屏幕空间偏移
        ScreenPosition += Indicator->GetScreenSpaceOffset();

        // 5. 屏幕边缘夹紧
        if (Indicator->GetClampToScreen())
        {
            ClampToScreen(AllottedGeometry, ScreenPosition, WidgetSize, bIsOnScreen);
            
            // 显示箭头指向实际位置
            if (Indicator->GetShowClampToScreenArrow() && !bIsOnScreen)
            {
                // 计算箭头方向并显示箭头Widget
                UpdateArrowWidget(Indicator, ScreenPosition, ActualPosition);
            }
        }

        // 6. 深度排序（远处的先画，近处的后画）
        // 7. 排列Widget到计算出的位置
        ArrangedChildren.AddWidget(AllottedGeometry.MakeChild(
            WidgetSlot, FVector2D(ScreenPosition), WidgetSize));
    }
}
```

### 7.5 UIndicatorLayer

**文件**: `Source/LyraGame/UI/IndicatorSystem/IndicatorLayer.h/.cpp`

UMG包装器，在UMG系统中创建`SActorCanvas`：

```cpp
UCLASS()
class UIndicatorLayer : public UWidget
{
    UPROPERTY(EditAnywhere)
    TSoftClassPtr<UUserWidget> IndicatorWidgetClass;

    UPROPERTY(EditAnywhere)
    TSoftClassPtr<UUserWidget> ArrowWidgetClass;

    virtual TSharedRef<SWidget> RebuildWidget() override
    {
        return SNew(SActorCanvas, ...)
            .IndicatorPool(...)
            .ArrowWidgetClass(ArrowWidgetClass);
    }
};
```

### 7.6 IActorIndicatorWidget

**文件**: `Source/LyraGame/UI/IndicatorSystem/IActorIndicatorWidget.h`

蓝图接口，指示器Widget实现此接口接收绑定：

```cpp
UINTERFACE(BlueprintType)
class UActorIndicatorWidgetInterface : public UInterface {};

class IActorIndicatorWidgetInterface
{
    UFUNCTION(BlueprintNativeEvent)
    void BindIndicator(UIndicatorDescriptor* Indicator);
    
    UFUNCTION(BlueprintNativeEvent)
    void UnbindIndicator(UIndicatorDescriptor* Indicator);
};
```

---

## 8. 武器UI系统

### 8.1 ULyraWeaponUserInterface

**文件**: `Source/LyraGame/UI/Weapons/LyraWeaponUserInterface.h/.cpp`

每帧轮询检测装备变化的武器UI基类：

```cpp
UCLASS()
class ULyraWeaponUserInterface : public UUserWidget
{
    UPROPERTY(Transient)
    TObjectPtr<ULyraWeaponInstance> CurrentInstance;

    // 蓝图事件：武器变化
    UFUNCTION(BlueprintImplementableEvent)
    void OnWeaponChanged(ULyraWeaponInstance* OldWeapon, ULyraWeaponInstance* NewWeapon);

    virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override
    {
        Super::NativeTick(MyGeometry, InDeltaTime);

        // 通过Pawn → EquipmentManager → 查找当前武器
        if (APawn* Pawn = GetOwningPlayerPawn())
        {
            if (ULyraEquipmentManagerComponent* EquipMgr = 
                Pawn->FindComponentByClass<ULyraEquipmentManagerComponent>())
            {
                ULyraWeaponInstance* NewWeapon = EquipMgr->GetFirstInstanceOfType<ULyraWeaponInstance>();
                if (NewWeapon != CurrentInstance)
                {
                    ULyraWeaponInstance* OldWeapon = CurrentInstance;
                    CurrentInstance = NewWeapon;
                    OnWeaponChanged(OldWeapon, NewWeapon);
                    // 准心Widget也一并重建
                    RebuildReticle();
                }
            }
        }
    }

    // 重建准心Widget
    void RebuildReticle()
    {
        // 从WeaponInstance获取ReticleWidgetClass
        // 销毁旧准心，创建新准心
        if (CurrentInstance)
        {
            TSubclassOf<ULyraReticleWidgetBase> NewReticleClass = CurrentInstance->GetReticleClass();
            if (NewReticleClass != CurrentReticleClass)
            {
                // 移除旧Widget，创建新Widget，添加到ReticleContainer
            }
        }
    }
};
```

### 8.2 ULyraReticleWidgetBase

**文件**: `Source/LyraGame/UI/Weapons/LyraReticleWidgetBase.h/.cpp`

准心Widget基类，计算武器散布角度到屏幕空间：

```cpp
UCLASS()
class ULyraReticleWidgetBase : public UUserWidget
{
    // 当前散布角度（度）
    UPROPERTY(BlueprintReadOnly)
    float CurrentSpreadAngle = 0.0f;

    // 散布角度对应的屏幕空间半径（像素）
    UPROPERTY(BlueprintReadOnly)
    float ComputedSpreadRadius = 0.0f;

    // 散布角度范围（用于归一化）
    UPROPERTY(BlueprintReadOnly)
    float SpreadAngleMin = 0.0f;
    UPROPERTY(BlueprintReadOnly)
    float SpreadAngleMax = 0.0f;

    // 归一化散布值 [0,1]
    UFUNCTION(BlueprintCallable)
    float GetNormalizedSpread() const
    {
        if (SpreadAngleMax > SpreadAngleMin)
        {
            return FMath::Clamp((CurrentSpreadAngle - SpreadAngleMin) 
                / (SpreadAngleMax - SpreadAngleMin), 0.0f, 1.0f);
        }
        return 0.0f;
    }

    // 计算散布角到屏幕半径
    void ComputeSpreadRadius()
    {
        // 通过FOV和视口大小将角度转换为像素
        // SpreadRadius = ViewportHeight * tan(SpreadAngle) / tan(FOV/2) / 2
        if (APlayerController* PC = GetOwningPlayer())
        {
            if (APlayerCameraManager* CameraManager = PC->PlayerCameraManager)
            {
                float FOVAngle = CameraManager->GetFOVAngle();
                // ... 投影计算
                ComputedSpreadRadius = /* 计算结果 */;
            }
        }
    }
};
```

---

## 9. 移动端触控UI

### 9.1 ULyraSimulatedInputWidget

**文件**: `Source/LyraGame/UI/LyraSimulatedInputWidget.h`

移动端虚拟输入的基类，将触控操作转换为模拟的硬件输入注入EnhancedInput：

```cpp
UCLASS(Abstract)
class ULyraSimulatedInputWidget : public UCommonUserWidget
{
    // 关联的InputAction
    UPROPERTY(EditAnywhere)
    TObjectPtr<UInputAction> AssociatedAction;

    // 注入模拟输入
    void InjectSimulatedInput(FVector Value)
    {
        if (UEnhancedInputLocalPlayerSubsystem* InputSubsystem = 
            GetOwningLocalPlayer()->GetSubsystem<UEnhancedInputLocalPlayerSubsystem>())
        {
            InputSubsystem->InjectInputForAction(AssociatedAction, Value);
        }
    }

    // 触控按下/移动/抬起的虚方法
    virtual void OnTouchStarted(const FGeometry& Geometry, const FPointerEvent& Event);
    virtual void OnTouchMoved(const FGeometry& Geometry, const FPointerEvent& Event);
    virtual void OnTouchEnded(const FGeometry& Geometry, const FPointerEvent& Event);
};
```

### 9.2 ULyraJoystickWidget

**文件**: `Source/LyraGame/UI/LyraJoystickWidget.h`

虚拟摇杆，继承`ULyraSimulatedInputWidget`：

```cpp
UCLASS()
class ULyraJoystickWidget : public ULyraSimulatedInputWidget
{
    // 摇杆中心和当前位置
    FVector2D StickCenter;
    FVector2D StickPosition;

    // 死区和最大范围
    UPROPERTY(EditAnywhere)
    float DeadZone = 0.1f;

    UPROPERTY(EditAnywhere)
    float StickRange = 50.0f;

    // 触控处理：计算偏移方向并注入2D轴输入
    virtual void OnTouchMoved(const FGeometry& Geometry, const FPointerEvent& Event) override
    {
        FVector2D Delta = CurrentTouchPosition - StickCenter;
        float Distance = Delta.Size();
        if (Distance > DeadZone * StickRange)
        {
            FVector2D NormalizedDelta = Delta / StickRange;
            NormalizedDelta = NormalizedDelta.ClampAxes(-1.0f, 1.0f);
            InjectSimulatedInput(FVector(NormalizedDelta.X, NormalizedDelta.Y, 0.0f));
        }
    }
};
```

### 9.3 ULyraTouchRegion

**文件**: `Source/LyraGame/UI/LyraTouchRegion.h`

触控区域，用于瞄准/视角控制：

```cpp
UCLASS()
class ULyraTouchRegion : public ULyraSimulatedInputWidget
{
    // 触控移动时计算Delta并注入为视角输入
    virtual void OnTouchMoved(const FGeometry& Geometry, const FPointerEvent& Event) override
    {
        FVector2D Delta = CurrentPosition - LastPosition;
        // 将触控Delta转换为视角旋转输入
        InjectSimulatedInput(FVector(Delta.X, Delta.Y, 0.0f));
        LastPosition = CurrentPosition;
    }
};
```

---

## 10. 基础UI组件

### 10.1 ULyraButtonBase

**文件**: `Source/LyraGame/UI/Foundation/LyraButtonBase.h/.cpp`

按钮基类，输入方式变化时自动刷新文本样式：

```cpp
UCLASS()
class ULyraButtonBase : public UCommonButtonBase
{
    // 当输入方式改变时刷新按钮文本
    void NativeOnInitialized() override
    {
        Super::NativeOnInitialized();
        // 监听输入方式变化（键鼠 ↔ 手柄）
    }

    void UpdateButtonText()
    {
        // 根据当前输入设备刷新按钮显示的操作提示文本
    }

    // 蓝图可重写：更新按钮样式
    UFUNCTION(BlueprintImplementableEvent)
    void UpdateButtonStyle();
};
```

### 10.2 ULyraConfirmationScreen

**文件**: `Source/LyraGame/UI/Foundation/LyraConfirmationScreen.h/.cpp`

确认对话框，动态创建按钮：

```cpp
UCLASS()
class ULyraConfirmationScreen : public UCommonActivatableWidget
{
    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UTextBlock> Text_Title;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UTextBlock> Text_Message;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UDynamicEntryBox> EntryBox_Buttons;

    FCommonMessagingResultDelegate OnResultCallback;

    void SetupDialog(UCommonGameDialogDescriptor* Descriptor, 
        FCommonMessagingResultDelegate ResultCallback)
    {
        Text_Title->SetText(Descriptor->Header);
        Text_Message->SetText(Descriptor->Body);

        // 动态创建按钮
        EntryBox_Buttons->Reset<ULyraButtonBase>([](ULyraButtonBase& Button) {});
        for (const FConfirmationDialogAction& Action : Descriptor->ButtonActions)
        {
            ULyraButtonBase* Button = EntryBox_Buttons->CreateEntry<ULyraButtonBase>();
            Button->SetButtonText(Action.OptionalDisplayText);
            Button->OnClicked().AddLambda([this, Result = Action.Result]()
            {
                OnResultCallback.ExecuteIfBound(Result);
                DeactivateWidget();
            });
        }
    }
};
```

### 10.3 ULyraLoadingScreenSubsystem

**文件**: `Source/LyraGame/UI/Foundation/LyraLoadingScreenSubsystem.h/.cpp`

加载屏幕管理，跟踪是否需要显示加载界面：

```cpp
UCLASS()
class ULyraLoadingScreenSubsystem : public UGameInstanceSubsystem
{
    // 通过引用计数控制加载屏幕显示
    void SetLoadingScreenContentWidget(TSubclassOf<UUserWidget> NewWidgetClass);
    TSubclassOf<UUserWidget> GetLoadingScreenContentWidget() const;

    // 委托：通知加载屏幕状态变化
    FOnLoadingScreenVisibilityChanged OnLoadingScreenVisibilityChanged;
};
```

### 10.4 ULyraTabListWidgetBase

**文件**: `Source/LyraGame/UI/Common/LyraTabListWidgetBase.h`

Tab列表，支持预注册和动态Tab：

```cpp
USTRUCT(BlueprintType)
struct FLyraTabDescriptor
{
    UPROPERTY(EditAnywhere)
    FName TabId;

    UPROPERTY(EditAnywhere)
    FText TabText;

    UPROPERTY(EditAnywhere)
    bool bHidden = false;

    UPROPERTY(EditAnywhere)
    TSoftClassPtr<UCommonButtonBase> TabButtonType;

    UPROPERTY(EditAnywhere)
    TSoftClassPtr<UCommonActivatableWidget> TabContentType;

    // 创建时的数据传入
    UPROPERTY(EditAnywhere)
    UObject* CreatedTabContentWidget = nullptr;
};

UCLASS()
class ULyraTabListWidgetBase : public UCommonTabListWidgetBase
{
    // 预注册Tab列表（编辑器配置）
    UPROPERTY(EditAnywhere, meta=(TitleProperty="TabId"))
    TArray<FLyraTabDescriptor> PreregisteredTabInfoArray;

    // 运行时注册Tab
    UFUNCTION(BlueprintCallable)
    bool RegisterDynamicTab(const FLyraTabDescriptor& TabDescriptor);
};
```

### 10.5 ULyraWidgetFactory

**文件**: `Source/LyraGame/UI/Common/LyraWidgetFactory.h`

抽象Widget工厂：

```cpp
UCLASS(Abstract, BlueprintType, Blueprintable, EditInlineNew)
class ULyraWidgetFactory : public UObject
{
    UFUNCTION(BlueprintNativeEvent)
    TSubclassOf<UUserWidget> FindWidgetClassForData(const UObject* Data) const;
};
```

---

## 11. 性能统计UI

### 11.1 ULyraPerfStatWidgetBase

**文件**: `Source/LyraGame/UI/PerformanceStats/LyraPerfStatWidgetBase.h`

性能统计显示Widget，支持文本和可选图表：

```cpp
UCLASS()
class ULyraPerfStatWidgetBase : public UCommonUserWidget
{
    // 对应的性能统计类型
    UPROPERTY(EditAnywhere)
    ELyraDisplayablePerformanceStat StatToDisplay;

    // 是否显示图表
    UPROPERTY(EditAnywhere)
    bool bShowGraph = false;

    // 图表历史数据
    TArray<float> GraphValues;
    int32 GraphMaxHistory = 120; // 2秒 @ 60fps

    // 蓝图实现：格式化显示
    UFUNCTION(BlueprintImplementableEvent)
    void UpdateStatDisplay(float Value, const FText& FormattedValue);
};
```

### 11.2 ULyraPerfStatContainerBase

**文件**: `Source/LyraGame/UI/PerformanceStats/LyraPerfStatContainerBase.h`

性能统计容器，根据设置控制子Widget可见性：

```cpp
UCLASS()
class ULyraPerfStatContainerBase : public UCommonUserWidget
{
    // 缓存的stat Widget映射
    UPROPERTY(Transient)
    TMap<ELyraDisplayablePerformanceStat, TObjectPtr<ULyraPerfStatWidgetBase>> StatWidgets;

    void UpdateVisibilityOfChildren()
    {
        // 从ULyraPerformanceSettings读取用户配置
        // 根据配置显示/隐藏对应的StatWidget
        for (auto& [Stat, Widget] : StatWidgets)
        {
            ESlateVisibility Visibility = IsStatEnabled(Stat) 
                ? ESlateVisibility::HitTestInvisible 
                : ESlateVisibility::Collapsed;
            Widget->SetVisibility(Visibility);
        }
    }
};
```

---

## 12. 设置系统UI关联

Lyra的设置系统通过`Source/LyraGame/Settings/`目录提供丰富的游戏设置，与UI紧密关联：

### 12.1 关键设置类

| 类名 | 职责 |
|------|------|
| `ULyraGameSettingRegistry` | 设置注册表，构建所有设置页面 |
| `ULyraSettingsLocal` | 本地设置持久化（输入、音视频、性能等） |
| `ULyraSettingsShared` | 跨设备共享设置（通过SaveGame） |

### 12.2 设置与UI的交互

```cpp
// ULyraGameSettingRegistry 创建设置页面
UGameSettingCollection* ULyraGameSettingRegistry::InitializeVideoSettings(ULyraLocalPlayer* Player)
{
    UGameSettingCollection* Screen = NewObject<UGameSettingCollection>();
    Screen->SetDevName(TEXT("VideoCollection"));
    Screen->SetDisplayName(LOCTEXT("VideoCollection_Name", "Video"));

    // 添加各种视频设置项
    // 窗口模式、分辨率、VSync、帧率限制、画质预设...
    // 每个设置项关联一个GameplayTag用于UI绑定
    return Screen;
}
```

设置UI页面通过`UGameSettingCollection`树状结构组织，蓝图端使用CommonUI的设置面板Widget渲染。

---

## 13. 完整架构图

```
┌─────────────────────── GameInstance Scope ──────────────────────────────────────┐
│                                                                                 │
│  ULyraUIManagerSubsystem (UGameInstanceSubsystem, Tickable)                    │
│    ├── CurrentPolicy: ULyraUIPolicy                                            │
│    │     ├── LayoutClass: TSoftClassPtr<UPrimaryGameLayout>                    │
│    │     └── RootViewportLayouts[] (per LocalPlayer)                           │
│    │           └── UPrimaryGameLayout                                          │
│    │                 ├── Layers (TMap<FGameplayTag, WidgetContainer>)           │
│    │                 │     ├── "UI.Layer.Game"   → HUD/GameplayUI              │
│    │                 │     ├── "UI.Layer.GameMenu" → EscapeMenu等              │
│    │                 │     ├── "UI.Layer.Menu"   → 前端主菜单                   │
│    │                 │     └── "UI.Layer.Modal"  → 确认对话框                   │
│    │                 ├── PushWidgetToLayerStack()                               │
│    │                 ├── PushWidgetToLayerStackAsync()                          │
│    │                 └── SuspendInputForPlayer() / ResumeInputForPlayer()       │
│    └── Tick(): 同步 PrimaryGameLayout.Visibility ↔ AHUD.bShowHUD              │
│                                                                                 │
├─────────────────────── World Scope ────────────────────────────────────────────┤
│                                                                                 │
│  UUIExtensionSubsystem (UWorldSubsystem)                                       │
│    ├── RegisterExtensionPoint(Tag, MatchType, Callback)                        │
│    │     └── UUIExtensionPointWidget (放入Layout蓝图)                           │
│    │           → OnAddOrRemoveExtension → CreateEntry/RemoveEntry               │
│    └── RegisterExtensionAsWidgetForContext(Tag, Context, WidgetClass)           │
│          └── 由 GameFeatureAction_AddWidgets 调用                               │
│                                                                                 │
│  ALyraHUD (AHUD)                                                               │
│    └── GameFrameworkComponentManager扩展点                                      │
│          → GameFeatureAction_AddWidgets监听HUD就绪                              │
│            → 推送Layout到Layer                                                  │
│            → 注册Widget到UIExtension                                            │
│                                                                                 │
│  ULyraIndicatorManagerComponent (UControllerComponent)                          │
│    ├── Indicators[]  →  UIndicatorDescriptor                                   │
│    └── OnIndicatorAdded/Removed                                                │
│          └── SActorCanvas (Slate)                                              │
│                ├── Widget池 (per WidgetClass)                                  │
│                ├── OnArrangeChildren: WorldToScreen投影                         │
│                ├── 屏幕边缘夹紧 + 箭头                                          │
│                └── 深度排序                                                     │
│                                                                                 │
├─────────────────────── Frontend Flow ──────────────────────────────────────────┤
│                                                                                 │
│  ULyraFrontendStateComponent (ControlFlow状态机)                                │
│    ├── FlowStep: TryShowPressStartScreen                                       │
│    ├── FlowStep: TryJoinRequestedSession                                       │
│    └── FlowStep: TryShowMainScreen                                             │
│          → PushStreamedContentToLayer_ForPlayer(各FrontendStateEntry)            │
│                                                                                 │
├─────────────────────── Weapon UI ──────────────────────────────────────────────┤
│                                                                                 │
│  ULyraWeaponUserInterface                                                      │
│    ├── NativeTick: 轮询 EquipmentManager 检测武器变化                           │
│    ├── OnWeaponChanged(Old, New) → 蓝图事件                                    │
│    └── RebuildReticle()                                                         │
│          └── ULyraReticleWidgetBase                                            │
│                ├── CurrentSpreadAngle / ComputedSpreadRadius                   │
│                └── GetNormalizedSpread() → 蓝图驱动准心缩放                     │
│                                                                                 │
├─────────────────────── Mobile Touch ───────────────────────────────────────────┤
│                                                                                 │
│  ULyraSimulatedInputWidget (基类)                                              │
│    ├── InjectSimulatedInput → EnhancedInputSubsystem                           │
│    ├── ULyraJoystickWidget (虚拟摇杆)                                          │
│    └── ULyraTouchRegion (触控区域/视角控制)                                     │
│                                                                                 │
├─────────────────────── Messaging ──────────────────────────────────────────────┤
│                                                                                 │
│  ULyraUIMessaging (UCommonMessagingSubsystem)                                  │
│    └── ShowConfirmation/ShowError                                              │
│          → PushContentToLayer(UI.Layer.Modal, ConfirmationDialogClass)          │
│            → ULyraConfirmationScreen.SetupDialog()                             │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 14. 核心设计模式总结

### 14.1 Layer Stack 模式

**核心思想**：UI按层级组织，每层维护Widget激活栈。

| Layer Tag | 用途 | 输入模式 |
|-----------|------|----------|
| `UI.Layer.Game` | 游戏HUD（血条、弹药、准心） | Game |
| `UI.Layer.GameMenu` | 游戏内菜单（ESC菜单） | Menu |
| `UI.Layer.Menu` | 前端菜单（主菜单、大厅） | Menu |
| `UI.Layer.Modal` | 模态对话框（确认/错误） | Menu |

**优势**：
- 清晰的层级隔离
- 自动管理输入焦点
- Push/Pop语义简洁

### 14.2 UIExtension 发布-订阅模式

**核心思想**：布局定义"插槽"(ExtensionPoint)，GameFeature注册"内容"(Extension)，通过Tag自动匹配。

```
Layout蓝图 (订阅方):
  UIExtensionPointWidget [Tag: UI.Extension.HUD.Reticle]
      ↕ GameplayTag匹配
GameFeature (发布方):
  GameFeatureAction_AddWidgets [Widget: W_Reticle, Slot: UI.Extension.HUD.Reticle]
```

**优势**：
- 完全解耦：布局不知道具体Widget，GameFeature不知道布局结构
- 热插拔：GameFeature激活/停用时自动添加/移除Widget
- 数据驱动：通过配置而非代码定义UI组合

### 14.3 ControlFlow 状态机模式

**核心思想**：用ControlFlow插件的Step链式执行前端流程。

```
ExperienceLoaded → PressStart → JoinSession → MainScreen
                   (可跳过)     (可跳过)     (最终目标)
```

**优势**：
- 声明式流程定义
- 每步可独立条件判断和跳过
- 异步友好（等待加载、等待用户输入）

### 14.4 GameFrameworkComponent 扩展模式

**核心思想**：通过`UGameFrameworkComponentManager`让GameFeature向Actor（如ALyraHUD）动态添加组件和行为。

```
GameFeature激活
  → GameFeatureAction_AddWidgets注册ExtensionHandler
    → 监听ALyraHUD的GameActorReady事件
      → 注入Layout和Widget
```

**优势**：
- GameFeature可以不修改核心代码就扩展HUD
- 生命周期自动管理（GameFeature停用时清理）

### 14.5 每帧轮询 vs 事件驱动

Lyra UI中两种模式并存：

| 模式 | 使用场景 | 示例 |
|------|----------|------|
| **事件驱动** | UI层级管理、Widget生命周期 | Push/Pop Layer, UIExtension回调 |
| **每帧轮询** | 高频变化的运行时数据 | 武器切换检测(WeaponUserInterface), HUD可见性同步, Indicator投影(SActorCanvas) |

### 14.6 Widget池化

在SActorCanvas中实现的Widget池化机制避免了指示器频繁创建/销毁的开销：

```
Indicator出现 → 从FreePool获取Widget（或创建新的） → 放入ActiveMap
Indicator消失 → 从ActiveMap移除 → 放回FreePool
```

### 14.7 多输入统一处理

通过CommonUI + `ULyraActivatableWidget`的InputConfig：

- **键鼠**：直接鼠标控制，自动显示键位提示
- **手柄**：UI导航切换，按钮提示自动更新
- **触控**：`ULyraSimulatedInputWidget`将触控转为EnhancedInput

---

## 15. 文件索引

| 目录/文件 | 核心职责 |
|-----------|----------|
| `Plugins/CommonGame/` | UI管理框架（Policy/Layout/Layer） |
| `Plugins/UIExtension/` | 发布-订阅式UI扩展系统 |
| `Source/LyraGame/UI/Subsystem/` | LyraUIManagerSubsystem, LyraUIMessaging |
| `Source/LyraGame/UI/LyraHUD.*` | HUD Actor |
| `Source/LyraGame/UI/LyraHUDLayout.*` | HUD布局（ESC菜单、断连处理） |
| `Source/LyraGame/UI/LyraActivatableWidget.*` | 输入模式配置基类 |
| `Source/LyraGame/UI/LyraTaggedWidget.*` | Tag驱动可见性 |
| `Source/LyraGame/UI/Frontend/` | 前端状态机 |
| `Source/LyraGame/UI/IndicatorSystem/` | 世界空间指示器 |
| `Source/LyraGame/UI/Weapons/` | 武器UI和准心 |
| `Source/LyraGame/UI/Foundation/` | 按钮、确认框、加载屏 |
| `Source/LyraGame/UI/Common/` | Tab列表、Widget工厂 |
| `Source/LyraGame/UI/PerformanceStats/` | 性能统计显示 |
| `Source/LyraGame/UI/LyraSimulatedInputWidget.*` | 移动端虚拟输入 |
| `Source/LyraGame/UI/LyraJoystickWidget.*` | 虚拟摇杆 |
| `Source/LyraGame/UI/LyraTouchRegion.*` | 触控区域 |
| `Source/LyraGame/UI/LyraGameViewportClient.*` | 平台光标选择 |
| `Source/LyraGame/GameFeatures/GameFeatureAction_AddWidget.*` | GameFeature→UI桥梁 |
| `Source/LyraGame/Settings/` | 设置系统（与UI联动） |
