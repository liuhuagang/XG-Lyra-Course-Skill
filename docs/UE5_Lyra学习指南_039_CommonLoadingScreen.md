# UE5_Lyra学习指南_039_CommonLoadingScreen

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_039\_CommonLoadingScreen](#ue5_lyra学习指南_039_commonloadingscreen)
	- [概述](#概述)
	- [配置](#配置)
	- [接口](#接口)
	- [管理类](#管理类)
		- [开发者设置](#开发者设置)
		- [蓝图使用](#蓝图使用)
		- [继承使用](#继承使用)
	- [LoadingScreenManager](#loadingscreenmanager)
		- [输入屏蔽](#输入屏蔽)
		- [更新加载屏幕可视性](#更新加载屏幕可视性)
			- [当下计算可视性](#当下计算可视性)
			- [延时补充可视性](#延时补充可视性)
		- [调用执行](#调用执行)
		- [实例的生命周期](#实例的生命周期)
		- [外部闻讯](#外部闻讯)
			- [代理](#代理)
			- [访问](#访问)
	- [总结](#总结)



## 概述
本节主要讲一下这个加载界面是如何实现的!
因为太优雅了!所以必须单独提出来讲一下!!!!!
因为它的这个写法就是很标准的虚幻插件模块写法.
在上套课程中给大家讲了一下标准的虚幻引擎插件开发.
这节算是一个额外的补充讲解!
流程:
1.定义一个接口,让需要使用到这个功能的实例继承自它.
2.定义了一个开发者设置可以用于在项目中设置,将其变量暴露到命令行中,可以快捷调试
3.定义了一个管理类,负责监听所有接口实例的生命周期,并通过该类分发具体的功能
4.暴露了蓝图静态函数,异步蓝图节点等方式提供便捷地使用.
## 配置
``` ini
[/Script/CommonLoadingScreen.CommonLoadingScreenSettings]
; 默认的加载界面
LoadingScreenWidget=/Game/UI/Foundation/LoadingScreen/W_LoadingScreen_Host.W_LoadingScreen_Host_C
; 强制Tick加载界面即使是在编辑器
ForceTickLoadingScreenEvenInEditor=False
```
## 接口
``` cpp
/** Interface for things that might cause loading to happen which requires a loading screen to be displayed */
/** 用于表示那些可能会触发加载操作（且该操作需要显示加载界面）的接口 */
UINTERFACE(MinimalAPI, BlueprintType)
class ULoadingProcessInterface : public UInterface
{
	GENERATED_BODY()
};

class ILoadingProcessInterface
{
	GENERATED_BODY()

public:
	// Checks to see if this object implements the interface, and if so asks whether or not we should
	// be currently showing a loading screen
	// 检查该对象是否实现了该接口，如果实现了则询问是否当前应显示加载界面
	static UE_API bool ShouldShowLoadingScreen(UObject* TestObject, FString& OutReason);

	virtual bool ShouldShowLoadingScreen(FString& OutReason) const
	{
		return false;
	}
};


```
## 管理类
``` cpp

/**
 * Handles showing/hiding the loading screen
 * 控制显示/隐藏加载界面
 */
UCLASS(MinimalAPI)
class ULoadingScreenManager : public UGameInstanceSubsystem, public FTickableGameObject
{
	GENERATED_BODY()

public:
	//~USubsystem interface
	// 绑定地图的变动
	UE_API virtual void Initialize(FSubsystemCollectionBase& Collection) override;
	// 移除加载屏幕的功能 
	UE_API virtual void Deinitialize() override;
	// 客户端才创建
	UE_API virtual bool ShouldCreateSubsystem(UObject* Outer) const override;
	//~End of USubsystem interface

	//~FTickableObjectBase interface
	// 每帧检测是否要显示加载屏幕
	UE_API virtual void Tick(float DeltaTime) override;
	// 决定Tick的开关类型
	UE_API virtual ETickableTickType GetTickableTickType() const override;
	// 补充是否需要显示加载屏幕
	UE_API virtual bool IsTickable() const override;
	// ID
	UE_API virtual TStatId GetStatId() const override;
	// 应用的世界
	UE_API virtual UWorld* GetTickableGameObjectWorld() const override;
	//~End of FTickableObjectBase interface

	// 暴露给蓝图询问
	UFUNCTION(BlueprintCallable, Category=LoadingScreen)
	FString GetDebugReasonForShowingOrHidingLoadingScreen() const
	{
		return DebugReasonForShowingOrHidingLoadingScreen;
	}

	/** Returns True when the loading screen is currently being shown */
	/** 当加载界面正在显示时返回 True */
	bool GetLoadingScreenDisplayStatus() const
	{
		return bCurrentlyShowingLoadingScreen;
	}

	/** Called when the loading screen visibility changes  */
	/** 当加载界面的可见性发生变化时调用 */
	DECLARE_MULTICAST_DELEGATE_OneParam(FOnLoadingScreenVisibilityChangedDelegate, bool);
	FORCEINLINE FOnLoadingScreenVisibilityChangedDelegate& OnLoadingScreenVisibilityChangedDelegate() { return LoadingScreenVisibilityChanged; }

	// 由具体的接口实例进行生命周期的注册
	UE_API void RegisterLoadingProcessor(TScriptInterface<ILoadingProcessInterface> Interface);
	UE_API void UnregisterLoadingProcessor(TScriptInterface<ILoadingProcessInterface> Interface);
	
private:
	// 触发更新
	UE_API void HandlePreLoadMap(const FWorldContext& WorldContext, const FString& MapName);
	// 触发更新
	UE_API void HandlePostLoadMap(UWorld* World);

	/** Determines if we should show or hide the loading screen. Called every frame. */
	/** 用于确定是否应显示或隐藏加载界面。每帧都会调用此函数。*/
	UE_API void UpdateLoadingScreen();

	/** Returns true if we need to be showing the loading screen. */
	/** 若需要显示加载界面，则返回 true 。*/
	UE_API bool CheckForAnyNeedToShowLoadingScreen();

	/** Returns true if we want to be showing the loading screen (if we need to or are artificially forcing it on for other reasons). */
	/** 若我们希望显示加载界面，则返回 true（这意味着我们需要显示该界面，或者是因为其他人为原因而强制显示）。*/
	UE_API bool ShouldShowLoadingScreen();

	/** Returns true if we are in the initial loading flow before this screen should be used */
	/** 如果我们正处于初始加载流程中，且在此屏幕投入使用之前，返回 true */
	UE_API bool IsShowingInitialLoadingScreen() const;

	/** Shows the loading screen. Sets up the loading screen widget on the viewport */
	/** 显示加载界面。在视口上设置加载界面组件 */
	UE_API void ShowLoadingScreen();

	/** Hides the loading screen. The loading screen widget will be destroyed */
	/** 隐藏加载界面。加载界面的组件将会被销毁 */
	UE_API void HideLoadingScreen();

	/** Removes the widget from the viewport */
	/** 将该控件从视口中移除 */
	UE_API void RemoveWidgetFromViewport();

	/** Prevents input from being used in-game while the loading screen is visible */
	/** 在加载界面显示期间，防止输入内容被用于游戏之中 */
	UE_API void StartBlockingInput();

	/** Resumes in-game input, if blocking */
	/** 若存在阻塞情况，则恢复游戏中的输入操作 */
	UE_API void StopBlockingInput();

	// 变更由于切换加载屏幕的性能影响
	UE_API void ChangePerformanceSettings(bool bEnabingLoadingScreen);

private:
	/** Delegate broadcast when the loading screen visibility changes */
	/** 当加载界面的可见性发生变化时，执行广播操作 */
	FOnLoadingScreenVisibilityChangedDelegate LoadingScreenVisibilityChanged;

	/** A reference to the loading screen widget we are displaying (if any) */
	/** 指向我们正在显示的加载屏幕组件的引用（如果有的话） */
	TSharedPtr<SWidget> LoadingScreenWidget;

	/** Input processor to eat all input while the loading screen is shown */
	/** 用于在加载界面显示期间接收所有输入的输入处理器 */
	TSharedPtr<IInputProcessor> InputPreProcessor;

	/** External loading processors, components maybe actors that delay the loading. */
	/** 外部加载处理器、组件可能包括那些会延迟加载过程的执行者。*/
	TArray<TWeakInterfacePtr<ILoadingProcessInterface>> ExternalLoadingProcessors;

	/** The reason why the loading screen is up (or not) */
	/** 加载界面是否显示（或未显示）的原因 */
	FString DebugReasonForShowingOrHidingLoadingScreen;

	/** The time when we started showing the loading screen */
	/** 我们开始显示加载界面的时间 */
	double TimeLoadingScreenShown = 0.0;

	/** The time the loading screen most recently wanted to be dismissed (might still be up due to a min display duration requirement) **/
	/** 加载界面最后一次想要被关闭的时间（由于最小显示时长要求，该界面可能仍处于显示状态） **/
	double TimeLoadingScreenLastDismissed = -1.0;

	/** The time until the next log for why the loading screen is still up */
	/** 下一次记录加载屏幕仍处于显示状态的原因所需的时间 */
	double TimeUntilNextLogHeartbeatSeconds = 0.0;

	/** True when we are between PreLoadMap and PostLoadMap */
	/** 当我们处于“预加载地图”与“后加载地图”之间时为真 */
	bool bCurrentlyInLoadMap = false;

	/** True when the loading screen is currently being shown */
	/** 当加载界面正在显示时为真 */
	bool bCurrentlyShowingLoadingScreen = false;
};

```

### 开发者设置
``` cpp
/**
 * Settings for a loading screen system.
 * 加载屏幕系统的设置。
 */
UCLASS(config=Game, defaultconfig, meta=(DisplayName="Common Loading Screen"))
class UCommonLoadingScreenSettings : public UDeveloperSettingsBackedByCVars
{
	GENERATED_BODY()

public:
	UCommonLoadingScreenSettings();

public:
	
	// The widget to load for the loading screen.
	// 用于加载屏幕的加载组件。
	UPROPERTY(config, EditAnywhere, Category=Display, meta=(MetaClass="/Script/UMG.UserWidget"))
	FSoftClassPath LoadingScreenWidget;

	// The z-order of the loading screen widget in the viewport stack
	// 视图堆栈中加载屏幕组件的层级顺序
	UPROPERTY(config, EditAnywhere, Category=Display)
	int32 LoadingScreenZOrder = 10000;

	// How long to hold the loading screen up after other loading finishes (in seconds) to
	// try to give texture streaming a chance to avoid blurriness
	//
	// Note: This is not normally applied in the editor for iteration time, but can be 
	// enabled via HoldLoadingScreenAdditionalSecsEvenInEditor
	// 其他加载完成之后，要将加载界面保持显示多长时间（以秒为单位），以期让纹理流加载有机会避免出现模糊现象//
	// 注意：通常情况下，此功能不适用于编辑器中的迭代时间，但可以通过“HoldLoadingScreenAdditionalSecsEvenInEditor”选项来启用。
 	UPROPERTY(config, EditAnywhere, Category=Configuration, meta=(ForceUnits=s, ConsoleVariable="CommonLoadingScreen.HoldLoadingScreenAdditionalSecs"))
	float HoldLoadingScreenAdditionalSecs = 2.0f;

	// The interval in seconds beyond which the loading screen is considered permanently hung (if non-zero).
	// 超过此时间间隔（以秒为单位）后，加载界面将被视为永久性卡住（若该值非零）。
 	UPROPERTY(config, EditAnywhere, Category=Configuration, meta=(ForceUnits=s))
	float LoadingScreenHeartbeatHangDuration = 0.0f;

	// The interval in seconds between each log of what is keeping a loading screen up (if non-zero).
	// 每次记录加载屏幕保持状态变化的时间间隔（若不为零）。单位：秒。
 	UPROPERTY(config, EditAnywhere, Category=Configuration, meta=(ForceUnits=s))
	float LogLoadingScreenHeartbeatInterval = 5.0f;

	// When true, the reason the loading screen is shown or hidden will be printed to the log every frame.
	// 若为真，则每次渲染时都会将加载界面显示或隐藏的原因记录到日志中。
	UPROPERTY(Transient, EditAnywhere, Category=Debugging, meta=(ConsoleVariable="CommonLoadingScreen.LogLoadingScreenReasonEveryFrame"))
	bool LogLoadingScreenReasonEveryFrame = 0;

	// Force the loading screen to be displayed (useful for debugging)
	// 强制显示加载界面（用于调试之用）
	UPROPERTY(Transient, EditAnywhere, Category=Debugging, meta=(ConsoleVariable="CommonLoadingScreen.AlwaysShow"))
	bool ForceLoadingScreenVisible = false;

	// Should we apply the additional HoldLoadingScreenAdditionalSecs delay even in the editor
	// (useful when iterating on loading screens)
	// 我们是否应该在编辑器中也使用额外的“保持加载界面显示时间（额外秒数）”这一设置呢？
	// （在对加载界面进行改进时，此设置很有用）
	UPROPERTY(Transient, EditAnywhere, Category=Debugging)
	bool HoldLoadingScreenAdditionalSecsEvenInEditor = false;

	// Should we apply the additional HoldLoadingScreenAdditionalSecs delay even in the editor
	// (useful when iterating on loading screens)
	// 我们是否应该在编辑器中也使用额外的“保持加载界面显示时间（额外秒数）”这一设置呢？
	// （在对加载界面进行改进时，此设置很有用）
	UPROPERTY(config, EditAnywhere, Category=Configuration)
	bool ForceTickLoadingScreenEvenInEditor = true;
};


```

### 蓝图使用
``` cpp
// 创建一个接口实例,用来临时显示加载屏幕.
// 注意一定要持有这个对象,防止GC
// 和异步蓝图节点的写法很像.
UCLASS(MinimalAPI, BlueprintType)
class ULoadingProcessTask : public UObject, public ILoadingProcessInterface
{
	GENERATED_BODY()
	
public:
	UFUNCTION(BlueprintCallable, meta=(WorldContext = "WorldContextObject"))
	static UE_API ULoadingProcessTask* CreateLoadingScreenProcessTask(UObject* WorldContextObject, const FString& ShowLoadingScreenReason);

public:
	ULoadingProcessTask() { }

	UFUNCTION(BlueprintCallable)
	UE_API void Unregister();

	UFUNCTION(BlueprintCallable)
	UE_API void SetShowLoadingScreenReason(const FString& InReason);

	UE_API virtual bool ShouldShowLoadingScreen(FString& OutReason) const override;
	
	FString Reason;
};

```
![RandomLobbyBPBP](./Pictures/005前端界面/P_LoadRandomBP.png)
### 继承使用
``` cpp
UCLASS(MinimalAPI)
class ULyraExperienceManagerComponent final : public UGameStateComponent, public ILoadingProcessInterface
{
	GENERATED_BODY()

public:
};
```
``` cpp
UCLASS(Abstract)
class ULyraFrontendStateComponent : public UGameStateComponent, public ILoadingProcessInterface
{
	GENERATED_BODY()

public:
}
```
## LoadingScreenManager

### 输入屏蔽
``` cpp
	/** Prevents input from being used in-game while the loading screen is visible */
	/** 在加载界面显示期间，防止输入内容被用于游戏之中 */
	UE_API void StartBlockingInput();

	/** Resumes in-game input, if blocking */
	/** 若存在阻塞情况，则恢复游戏中的输入操作 */
	UE_API void StopBlockingInput();
```

``` cpp
// Input processor to throw in when loading screen is shown
// This will capture any inputs, so active menus under the loading screen will not interact
// 在加载界面显示时使用的输入处理器
// 此处理器将捕获所有输入操作，因此加载界面下的活动菜单将不会与之交互
class FLoadingScreenInputPreProcessor : public IInputProcessor
{
public:
	FLoadingScreenInputPreProcessor() { }
	virtual ~FLoadingScreenInputPreProcessor() { }

	bool CanEatInput() const
	{
		return !GIsEditor;
	}

	//~IInputProcess interface
	virtual void Tick(const float DeltaTime, FSlateApplication& SlateApp, TSharedRef<ICursor> Cursor) override { }

	virtual bool HandleKeyDownEvent(FSlateApplication& SlateApp, const FKeyEvent& InKeyEvent) override { return CanEatInput(); }
	virtual bool HandleKeyUpEvent(FSlateApplication& SlateApp, const FKeyEvent& InKeyEvent) override { return CanEatInput(); }
	virtual bool HandleAnalogInputEvent(FSlateApplication& SlateApp, const FAnalogInputEvent& InAnalogInputEvent) override { return CanEatInput(); }
	virtual bool HandleMouseMoveEvent(FSlateApplication& SlateApp, const FPointerEvent& MouseEvent) override { return CanEatInput(); }
	virtual bool HandleMouseButtonDownEvent(FSlateApplication& SlateApp, const FPointerEvent& MouseEvent) override { return CanEatInput(); }
	virtual bool HandleMouseButtonUpEvent(FSlateApplication& SlateApp, const FPointerEvent& MouseEvent) override { return CanEatInput(); }
	virtual bool HandleMouseButtonDoubleClickEvent(FSlateApplication& SlateApp, const FPointerEvent& MouseEvent) override { return CanEatInput(); }
	virtual bool HandleMouseWheelOrGestureEvent(FSlateApplication& SlateApp, const FPointerEvent& InWheelEvent, const FPointerEvent* InGestureEvent) override { return CanEatInput(); }
	virtual bool HandleMotionDetectedEvent(FSlateApplication& SlateApp, const FMotionEvent& MotionEvent) override { return CanEatInput(); }
	//~End of IInputProcess interface
};

```

### 更新加载屏幕可视性
``` cpp
void ULoadingScreenManager::UpdateLoadingScreen()
{
	bool bLogLoadingScreenStatus = LoadingScreenCVars::LogLoadingScreenReasonEveryFrame;

	if (ShouldShowLoadingScreen())
	{
		const UCommonLoadingScreenSettings* Settings = GetDefault<UCommonLoadingScreenSettings>();
		
		// If we don't make it to the specified checkpoint in the given time will trigger the hang detector so we can better determine where progress stalled.
		// 如果我们未能在规定时间内到达指定的检查点，就会触发挂起检测机制，这样我们就能更清楚地了解进度停滞的具体位置。
 		FThreadHeartBeat::Get().MonitorCheckpointStart(GetFName(), Settings->LoadingScreenHeartbeatHangDuration);

		ShowLoadingScreen();

 		if ((Settings->LogLoadingScreenHeartbeatInterval > 0.0f) && (TimeUntilNextLogHeartbeatSeconds <= 0.0))
 		{
			bLogLoadingScreenStatus = true;
 			TimeUntilNextLogHeartbeatSeconds = Settings->LogLoadingScreenHeartbeatInterval;
 		}
	}
	else
	{
		HideLoadingScreen();
		/* 当检查点结束时，由线程调用 */
 		FThreadHeartBeat::Get().MonitorCheckpointEnd(GetFName());
	}

	if (bLogLoadingScreenStatus)
	{
		UE_LOG(LogLoadingScreen, Log, TEXT("Loading screen showing: %d. Reason: %s"), bCurrentlyShowingLoadingScreen ? 1 : 0, *DebugReasonForShowingOrHidingLoadingScreen);
	}
}
```
#### 当下计算可视性
``` cpp

bool ULoadingScreenManager::CheckForAnyNeedToShowLoadingScreen()
{
	// Start out with 'unknown' reason in case someone forgets to put a reason when changing this in the future.
	// 一开始将原因设为“未知”，以防日后有人在修改此项内容时忘记填写原因。
	DebugReasonForShowingOrHidingLoadingScreen = TEXT("Reason for Showing/Hiding LoadingScreen is unknown!");

	const UGameInstance* LocalGameInstance = GetGameInstance();

	// 是否强制显示
	if (LoadingScreenCVars::ForceLoadingScreenVisible)
	{
		DebugReasonForShowingOrHidingLoadingScreen = FString(TEXT("CommonLoadingScreen.AlwaysShow is true"));
		return true;
	}

	const FWorldContext* Context = LocalGameInstance->GetWorldContext();
	if (Context == nullptr)
	{
		// We don't have a world context right now... better show a loading screen
		// 目前我们还没有全球性的背景信息……最好显示一个加载界面
		DebugReasonForShowingOrHidingLoadingScreen = FString(TEXT("The game instance has a null WorldContext"));
		return true;
	}

	UWorld* World = Context->World();
	// 无世界
	if (World == nullptr)
	{
		DebugReasonForShowingOrHidingLoadingScreen = FString(TEXT("We have no world (FWorldContext's World() is null)"));
		return true;
	}

	AGameStateBase* GameState = World->GetGameState<AGameStateBase>();
	// 无GameState
	if (GameState == nullptr)
	{
		// The game state has not yet replicated.
		DebugReasonForShowingOrHidingLoadingScreen = FString(TEXT("GameState hasn't yet replicated (it's null)"));
		return true;
	}

	// 加载地图中
	if (bCurrentlyInLoadMap)
	{
		// Show a loading screen if we are in LoadMap
		// 如果处于“加载地图”状态，则显示加载界面
		DebugReasonForShowingOrHidingLoadingScreen = FString(TEXT("bCurrentlyInLoadMap is true"));
		return true;
	}

	/** 用于处理待连接客户端的跳转 URL */
	if (!Context->TravelURL.IsEmpty())
	{
		// Show a loading screen when pending travel
		// 当有待处理的行程时显示加载界面
		DebugReasonForShowingOrHidingLoadingScreen = FString(TEXT("We have pending travel (the TravelURL is not empty)"));
		return true;
	}

	if (Context->PendingNetGame != nullptr)
	{
		// Connecting to another server
		// 连接到另一台服务器
		DebugReasonForShowingOrHidingLoadingScreen = FString(TEXT("We are connecting to another server (PendingNetGame != nullptr)"));
		return true;
	}

	// 世界还没开始
	if (!World->HasBegunPlay())
	{
		DebugReasonForShowingOrHidingLoadingScreen = FString(TEXT("World hasn't begun play"));
		return true;
	}

	if (World->IsInSeamlessTravel())
	{
		// Show a loading screen during seamless travel
		// 在无缝切换过程中显示加载界面
		DebugReasonForShowingOrHidingLoadingScreen = FString(TEXT("We are in seamless travel"));
		return true;
	}

	// Ask the game state if it needs a loading screen
	// 调用游戏状态对象，询问其是否需要加载界面
	if (ILoadingProcessInterface::ShouldShowLoadingScreen(GameState, /*out*/ DebugReasonForShowingOrHidingLoadingScreen))
	{
		return true;
	}

	// Ask any game state components if they need a loading screen
	// 要求任何游戏状态组件告知自身是否需要加载界面
	for (UActorComponent* TestComponent : GameState->GetComponents())
	{
		if (ILoadingProcessInterface::ShouldShowLoadingScreen(TestComponent, /*out*/ DebugReasonForShowingOrHidingLoadingScreen))
		{
			return true;
		}
	}

	// Ask any of the external loading processors that may have been registered.  These might be actors or components
	// that were registered by game code to tell us to keep the loading screen up while perhaps something finishes
	// streaming in.
	// 联系任何可能已注册的外部加载处理器。这些处理器可能是由游戏代码注册的演员或组件，它们会告知我们应保持加载界面处于显示状态，以便在某些内容完成加载传输时能够显示出来。
	for (const TWeakInterfacePtr<ILoadingProcessInterface>& Processor : ExternalLoadingProcessors)
	{
		if (ILoadingProcessInterface::ShouldShowLoadingScreen(Processor.GetObject(), /*out*/ DebugReasonForShowingOrHidingLoadingScreen))
		{
			return true;
		}
	}

	// Check each local player
	// 检查每个本地玩家
	bool bFoundAnyLocalPC = false;
	bool bMissingAnyLocalPC = false;

	for (ULocalPlayer* LP : LocalGameInstance->GetLocalPlayers())
	{
		if (LP != nullptr)
		{
			if (APlayerController* PC = LP->PlayerController)
			{
				bFoundAnyLocalPC = true;

				// Ask the PC itself if it needs a loading screen
				// 要求电脑自身确认是否需要加载画面
				if (ILoadingProcessInterface::ShouldShowLoadingScreen(PC, /*out*/ DebugReasonForShowingOrHidingLoadingScreen))
				{
					return true;
				}

				// Ask any PC components if they need a loading screen
				// 询问任何电脑组件是否需要加载画面
				for (UActorComponent* TestComponent : PC->GetComponents())
				{
					if (ILoadingProcessInterface::ShouldShowLoadingScreen(TestComponent, /*out*/ DebugReasonForShowingOrHidingLoadingScreen))
					{
						return true;
					}
				}
			}
			else
			{
				bMissingAnyLocalPC = true;
			}
		}
	}

	UGameViewportClient* GameViewportClient = LocalGameInstance->GetGameViewportClient();
	const bool bIsInSplitscreen = GameViewportClient->GetCurrentSplitscreenConfiguration() != ESplitScreenType::None;

	// In splitscreen we need all player controllers to be present
	// 在分屏模式下，我们需要确保所有玩家控制器都处于开启状态
	if (bIsInSplitscreen && bMissingAnyLocalPC)
	{
		DebugReasonForShowingOrHidingLoadingScreen = FString(TEXT("At least one missing local player controller in splitscreen"));
		return true;
	}

	// And in non-splitscreen we need at least one player controller to be present
	// 在非分屏模式下，我们至少需要有一个玩家控制器在场。
	if (!bIsInSplitscreen && !bFoundAnyLocalPC)
	{
		DebugReasonForShowingOrHidingLoadingScreen = FString(TEXT("Need at least one local player controller"));
		return true;
	}

	// Victory! The loading screen can go away now
	// 胜利！加载画面现在可以消失了
	DebugReasonForShowingOrHidingLoadingScreen = TEXT("(nothing wants to show it anymore)");
	return false;
}
```
#### 延时补充可视性


``` cpp
	const UCommonLoadingScreenSettings* Settings = GetDefault<UCommonLoadingScreenSettings>();

	// Check debugging commands that force the state one way or another
	// 检查那些能够强制改变状态的调试命令
#if !UE_BUILD_SHIPPING
	// 命令行无加载界面
	static bool bCmdLineNoLoadingScreen = FParse::Param(FCommandLine::Get(), TEXT("NoLoadingScreen"));
	if (bCmdLineNoLoadingScreen)
	{
		DebugReasonForShowingOrHidingLoadingScreen = FString(TEXT("CommandLine has 'NoLoadingScreen'"));
		return false;
	}
#endif

	// Can't show a loading screen if there's no game viewport
	// 若没有游戏视图区域，则无法显示加载屏幕
	UGameInstance* LocalGameInstance = GetGameInstance();
	if (LocalGameInstance->GetGameViewportClient() == nullptr)
	{
		return false;
	}

	// Check for a need to show the loading screen
	// 检查是否需要显示加载界面
	const bool bNeedToShowLoadingScreen = CheckForAnyNeedToShowLoadingScreen();

	// Keep the loading screen up a bit longer if desired
	// 如果需要的话，可以让加载界面多显示一段时间
	bool bWantToForceShowLoadingScreen = false;
	if (bNeedToShowLoadingScreen)
	{
		// Still need to show it
		// 仍需展示它
		TimeLoadingScreenLastDismissed = -1.0;
	}
	else
	{
		// Don't *need* to show the screen anymore, but might still want to for a bit
		// 无需再显示该屏幕了，但可能仍需要展示一小段时间。
		const double CurrentTime = FPlatformTime::Seconds();
		const bool bCanHoldLoadingScreen = (!GIsEditor || Settings->HoldLoadingScreenAdditionalSecsEvenInEditor);
		const double HoldLoadingScreenAdditionalSecs = bCanHoldLoadingScreen ? LoadingScreenCVars::HoldLoadingScreenAdditionalSecs : 0.0;

		if (TimeLoadingScreenLastDismissed < 0.0)
		{
			TimeLoadingScreenLastDismissed = CurrentTime;
		}
		const double TimeSinceScreenDismissed = CurrentTime - TimeLoadingScreenLastDismissed;

		// hold for an extra X seconds, to cover up streaming
		// 延长等待 X 秒，以弥补数据传输的延迟
		if ((HoldLoadingScreenAdditionalSecs > 0.0) && (TimeSinceScreenDismissed < HoldLoadingScreenAdditionalSecs))
		{
			// Make sure we're rendering the world at this point, so that textures will actually stream in
			//@TODO: If bNeedToShowLoadingScreen bounces back true during this window, we won't turn this off again...
			
			// 确保此时我们正在渲染整个世界，这样纹理才能真正加载进来
			//@待办事项：如果在这一窗口期间 bNeedToShowLoadingScreen 的值反向变为 true，我们就不会再次将其关闭……
			
			UGameViewportClient* GameViewportClient = GetGameInstance()->GetGameViewportClient();
			/** 是否设置为禁用世界渲染 */
			// 开始渲染3D画面
			GameViewportClient->bDisableWorldRendering = false;

			DebugReasonForShowingOrHidingLoadingScreen = FString::Printf(TEXT("Keeping loading screen up for an additional %.2f seconds to allow texture streaming"), HoldLoadingScreenAdditionalSecs);
			bWantToForceShowLoadingScreen = true;
		}
	}

	return bNeedToShowLoadingScreen || bWantToForceShowLoadingScreen;
```



### 调用执行



``` cpp
void ULoadingScreenManager::Initialize(FSubsystemCollectionBase& Collection)
{
	/** 在加载地图操作之初发送 */
	FCoreUObjectDelegates::PreLoadMapWithContext.AddUObject(this, &ThisClass::HandlePreLoadMap);

	/** 在加载地图操作完成后发送 */
	FCoreUObjectDelegates::PostLoadMapWithWorld.AddUObject(this, &ThisClass::HandlePostLoadMap);

	const UGameInstance* LocalGameInstance = GetGameInstance();
	check(LocalGameInstance);
}


void ULoadingScreenManager::HandlePreLoadMap(const FWorldContext& WorldContext, const FString& MapName)
{
	if (WorldContext.OwningGameInstance == GetGameInstance())
	{
		bCurrentlyInLoadMap = true;

		// Update the loading screen immediately if the engine is initialized
		// 若引擎已初始化，则立即更新加载界面
		if (GEngine->IsInitialized())
		{
			UpdateLoadingScreen();
		}
	}
}
```
``` cpp
void ULoadingScreenManager::Tick(float DeltaTime)
{
	UpdateLoadingScreen();

	TimeUntilNextLogHeartbeatSeconds = FMath::Max(TimeUntilNextLogHeartbeatSeconds - DeltaTime, 0.0);
}


```

### 实例的生命周期
注意此处只是用于手动的.实际上会自动访问GameState及其组件是否需要这个功能!
``` cpp
	// 由具体的接口实例进行生命周期的注册
	UE_API void RegisterLoadingProcessor(TScriptInterface<ILoadingProcessInterface> Interface);
	UE_API void UnregisterLoadingProcessor(TScriptInterface<ILoadingProcessInterface> Interface);
```
``` cpp
/*static*/ ULoadingProcessTask* ULoadingProcessTask::CreateLoadingScreenProcessTask(UObject* WorldContextObject, const FString& ShowLoadingScreenReason)
{
	UWorld* World = GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::LogAndReturnNull);
	UGameInstance* GameInstance = World ? World->GetGameInstance() : nullptr;
	ULoadingScreenManager* LoadingScreenManager = GameInstance ? GameInstance->GetSubsystem<ULoadingScreenManager>() : nullptr;

	if (LoadingScreenManager)
	{
		ULoadingProcessTask* NewLoadingTask = NewObject<ULoadingProcessTask>(LoadingScreenManager);
		NewLoadingTask->SetShowLoadingScreenReason(ShowLoadingScreenReason);

		LoadingScreenManager->RegisterLoadingProcessor(NewLoadingTask);
		
		return NewLoadingTask;
	}

	return nullptr;
}

```
### 外部闻讯
#### 代理
``` cpp
private:
	/** Delegate broadcast when the loading screen visibility changes */
	/** 当加载界面的可见性发生变化时，执行广播操作 */
	FOnLoadingScreenVisibilityChangedDelegate LoadingScreenVisibilityChanged;
```
#### 访问
``` cpp
	// 暴露给蓝图询问
	UFUNCTION(BlueprintCallable, Category=LoadingScreen)
	FString GetDebugReasonForShowingOrHidingLoadingScreen() const
	{
		return DebugReasonForShowingOrHidingLoadingScreen;
	}
```
## 总结
虽然我们使用的时候只配置了几个蓝图.但是它里面的逻辑真的很优秀!