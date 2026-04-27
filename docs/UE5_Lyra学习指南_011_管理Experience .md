# UE5_Lyra学习指南_011_管理Experience 

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)

文章内容由小刚撰写,采用了以下多种方式:

1.口述转文字
2.AI重构
3.参考引擎源码
4.Lyra工程源码
5.结合社区论坛各位大佬的解析

- [UE5\_Lyra学习指南\_011\_管理Experience](#ue5_lyra学习指南_011_管理experience)
	- [概述](#概述)
	- [1.创建LyraExperienceManager](#1创建lyraexperiencemanager)
	- [2.修改LyraEditor.h](#2修改lyraeditorh)
	- [3.编写ULyraExperienceManagerComponent](#3编写ulyraexperiencemanagercomponent)
		- [启动时机](#启动时机)
		- [内部加载流程](#内部加载流程)
		- [后调用流程](#后调用流程)
	- [总结](#总结)



## 概述

本节代码量非常大,请不要走神,并一步一步跟着做.
我们并没有从启动开始,也并没有从结束倒推.我们选择从从中间层进行呈上起来.
因为这个地方它是一个比较完整的中间层,只有进和出.解析起来比较方便.逻辑顺序上和下一节是一个倒装!

## 1.创建LyraExperienceManager

LyraExperienceManager,它是一个用来体验管理的引擎子系统,主要负责多个 PIE 会议之间的协调工作.

``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.
// 001 Finished.
#pragma once

#include "Subsystems/EngineSubsystem.h"
#include "LyraExperienceManager.generated.h"

/**
 * Manager for experiences - primarily for arbitration between multiple PIE sessions
 * 体验管理的引擎子系统 - 主要负责多个 PIE 会议之间的协调工作
 */
UCLASS(MinimalAPI)
class ULyraExperienceManager : public UEngineSubsystem
{
	GENERATED_BODY()

public:
#if WITH_EDITOR
	//在编辑器模块中的StartupModule()进行调用,用于初始化GameFeaturePluginRequestCountMap
	LYRAGAME_API void OnPlayInEditorBegun();

	//通知有插件被激活了,增加计数,由LyraExperienceManagerComponent调用
	static void NotifyOfPluginActivation(const FString PluginURL);

	//要求取消插件的激活计数,减少计数,由LyraExperienceManagerComponent调用
	static bool RequestToDeactivatePlugin(const FString PluginURL);

	//运行时不需要这个功能
#else
	static void NotifyOfPluginActivation(const FString PluginURL) {}
	static bool RequestToDeactivatePlugin(const FString PluginURL) { return true; }
#endif

private:
	// The map of requests to active count for a given game feature plugin
	// (to allow first in, last out activation management during PIE)
	// 指定游戏功能插件的请求量与激活次数的关系图
	// （以便在 PIE 过程中实现[先入后出]的激活管理）
	TMap<FString, int32> GameFeaturePluginRequestCountMap;
};


```
## 2.修改LyraEditor.h
我们需要把这个引擎子系统嵌入到编辑器使用的流程,它需要在编辑器启动的时候初始化GameFeature插件的映射.
这里为了省事儿,直接把所有头文件都导入了.
这个EditorModule文件写了很多编辑拓展功能的代码,我们将随着课程进度进行完善.
```cpp
// Copyright Epic Games, Inc. All Rights Reserved.

#include "LyraEditor.h"

#include "AbilitySystemGlobals.h"
#include "DataValidationModule.h"
#include "Development/LyraDeveloperSettings.h"
#include "Editor/UnrealEdEngine.h"
#include "Engine/GameInstance.h"
#include "Framework/Application/SlateApplication.h"
#include "Framework/MultiBox/MultiBoxBuilder.h"
#include "GameEditorStyle.h"
#include "GameModes/LyraExperienceManager.h"
#include "GameplayAbilitiesEditorModule.h"
#include "GameplayCueInterface.h"
#include "GameplayCueNotify_Burst.h"
#include "GameplayCueNotify_BurstLatent.h"
#include "GameplayCueNotify_Looping.h"
#include "Private/AssetTypeActions_LyraContextEffectsLibrary.h"
#include "ToolMenu.h"
#include "ToolMenus.h"
#include "UObject/UObjectIterator.h"
#include "UnrealEdGlobals.h"
#include "Validation/EditorValidator.h"

class SWidget;

#define LOCTEXT_NAMESPACE "LyraEditor"

DEFINE_LOG_CATEGORY(LogLyraEditor);


/**
 * FLyraEditorModule
 */
class FLyraEditorModule : public FDefaultGameModuleImpl
{
	//必须要在这里进行定义 否则用不了ThisClass
	typedef FLyraEditorModule ThisClass;
	
	virtual void StartupModule() override
	{
		if (!IsRunningGame())
		{


			FEditorDelegates::BeginPIE.AddRaw(this, &ThisClass::OnBeginPIE);
			FEditorDelegates::EndPIE.AddRaw(this, &ThisClass::OnEndPIE);
		}
	}



	virtual void ShutdownModule() override
	{
	
	
	}
	void OnBeginPIE(bool bIsSimulating)
	{
		ULyraExperienceManager* ExperienceManager = GEngine->GetEngineSubsystem<ULyraExperienceManager>();
		check(ExperienceManager);
		ExperienceManager->OnPlayInEditorBegun();
	}

	void OnEndPIE(bool bIsSimulating)
	{
		
	}

	
};

IMPLEMENT_MODULE(FLyraEditorModule, LyraEditor);


#undef LOCTEXT_NAMESPACE

```


## 3.编写ULyraExperienceManagerComponent
这个GameStateComponent很重要.
它是管理加载使用Experience的组件类.
注意,在网络游戏情况下,它是通过GameMode进行服务器初始化,然后通过网络属性同步来进行客户端的初始化.

### 启动时机
这里我们掐头去尾了!!!!
我们没有写它的初始化位置,但它实际上是在GameMode的
void ALyraGameMode::OnMatchAssignmentGiven(FPrimaryAssetId ExperienceId, const FString& ExperienceIdSource)调用.
继续往上面翻void ALyraGameMode::HandleMatchAssignmentIfNotExpectingOne()
继续往上面是在void ALyraGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage);

以下是启动位置:
``` cpp
	/**
	 * Initialize the game.
	 * The GameMode's InitGame() event is called before any other functions (including PreInitializeComponents() )
	 * and is used by the GameMode to initialize parameters and spawn its helper classes.
	 * @warning: this is called before actors' PreInitializeComponents.
	 */
	/**
	 * 初始化游戏。
	 * 在调用任何其他函数（包括 PreInitializeComponents() 函数）之前，会调用 GameMode 的 InitGame() 事件。
	 * 这个事件由 GameMode 使用来初始化参数并生成其辅助类。
	 * 注意：此事件在角色的 PreInitializeComponents 事件之前被调用。
	 */
	ENGINE_API virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage);
```

``` cpp
void ALyraGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
	Super::InitGame(MapName, Options, ErrorMessage);

	// Wait for the next frame to give time to initialize startup settings
	GetWorld()->GetTimerManager().SetTimerForNextTick(this, &ThisClass::HandleMatchAssignmentIfNotExpectingOne);
}
``` 
以下解决使用哪个Experience.
``` cpp
void ALyraGameMode::HandleMatchAssignmentIfNotExpectingOne()
{
	FPrimaryAssetId ExperienceId;
	FString ExperienceIdSource;

	// Precedence order (highest wins)
	//  - Matchmaking assignment (if present)
	//  - URL Options override
	//  - Developer Settings (PIE only)
	//  - Command Line override
	//  - World Settings
	//  - Dedicated server
	//  - Default experience
	// 省略若干代码
	OnMatchAssignmentGiven(ExperienceId, ExperienceIdSource);
}
```
``` cpp
void ALyraGameMode::OnMatchAssignmentGiven(FPrimaryAssetId ExperienceId, const FString& ExperienceIdSource)
{
	if (ExperienceId.IsValid())
	{
		UE_LOG(LogLyraExperience, Log, TEXT("Identified experience %s (Source: %s)"), *ExperienceId.ToString(), *ExperienceIdSource);

		ULyraExperienceManagerComponent* ExperienceComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
		check(ExperienceComponent);
		ExperienceComponent->SetCurrentExperience(ExperienceId);
	}
	else
	{
		UE_LOG(LogLyraExperience, Error, TEXT("Failed to identify experience, loading screen will stay up forever"));
	}
}
```

### 内部加载流程
注意这个函数有服务器直接启动,但是客户端时时通过网络同步做的.

``` cpp
void ULyraExperienceManagerComponent::SetCurrentExperience(FPrimaryAssetId ExperienceId)
{
	//@XGTODO:这里使用的Lyra项目自定义的资产管理器
	//ULyraAssetManager& AssetManager = ULyraAssetManager::Get();
	UAssetManager& AssetManager = UAssetManager::Get();

	/** 获取指定主资产类型及名称的 FSoftObjectPath，若未找到则返回无效值 */
	FSoftObjectPath AssetPath = AssetManager.GetPrimaryAssetPath(ExperienceId);

	//尝试加载该资源时，这将调用“LoadObject”函数，而该函数的执行可能会非常缓慢。
	TSubclassOf<ULyraExperienceDefinition> AssetClass = Cast<UClass>(AssetPath.TryLoad());

	//必须要拿到!
	check(AssetClass);

	//获取一个类的默认对象。
	//在大多数情况下，类的默认对象不应被修改。因此，此方法返回的是一个不可变的指针。如果您需要修改默认对象，请使用 GetMutableDefault 替代。
	const ULyraExperienceDefinition* Experience = GetDefault<ULyraExperienceDefinition>(AssetClass);

	check(Experience != nullptr);
	check(CurrentExperience == nullptr);

	//这里由服务器属性同步到到客户端进行加载
	CurrentExperience = Experience;

	//开始加载
	StartExperienceLoad();
}

```
``` cpp
void ULyraExperienceManagerComponent::OnRep_CurrentExperience()
{
	//从服务器同步过来,开启加载
	StartExperienceLoad();
}

```

``` cpp
	void StartExperienceLoad();
```

在本节中,我们还没有实现自定义的资产管理器等,我们就先用引擎原生的,随后我们将进行实现.
其次,我们也没有实现用户设置,随后再进行实现.

总结一下这类的源文件,把Experience所有用到的GameFeature插件,以及这个Experience的ActionSets所用到的GameFeature插件都进行了加载卸载管理,并对所有GameFeature的Action进行了注释加载激活等操作.
这里涉及到了大量同步和异步的资产流程,以及先后操作顺序,较为复杂,一定要耐心地跟着我的思路过一遍.具体流程和注释都写到了代码里面了.此处不贴出全文,因为太长了.
还要注意这里有大量的TODO
``` txt
//@TODO: Async load the experience definition itself
//@TODO: 异步加载Experience本身,我们现在是在设置Experience时直接进行加载的Tryload.

//@TODO: Handle failures explicitly (go into a 'completed but failed' state rather than check()-ing)
//@待办事项：明确处理失败情况（进入“已完成但失败”的状态，而非通过检查来处理）

//@TODO: Do the action phases at the appropriate times instead of all at once
//@注意事项：应在恰当的时间执行各个行动阶段，而非一次性全部完成。

//@TODO: Support deactivating an experience and do the unloading actions
//@待办事项：支持停用体验，并执行卸载操作

//@TODO: Think about what deactivation/cleanup means for preloaded assets
//@待办事项：思考一下预加载资源的停用/清理工作意味着什么

//@TODO: Handle deactivating game features, right now we 'leak' them enabled
//@待办事项：处理游戏功能的停用问题，目前我们存在功能未停用而仍处于启用状态的情况。
// (for a client moving from experience to experience we actually want to diff the requirements and only unload some, not unload everything for them to just be immediately reloaded)
// （对于那些从一个项目转向另一个项目的客户而言，我们实际上希望对需求进行差异处理，只卸载一部分内容，而不是一次性全部卸载，以免导致他们需要重新加载所有内容）

//@TODO: Handle both built-in and URL-based plugins (search for colon?)
//@待办事项：处理内置插件和基于 URL 的插件（查找冒号？）
```

此处给出头文件定义:
``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.
// 001  Not Finished. 需要替换资产管理器 需要更新用户设置
#pragma once

#include "Components/GameStateComponent.h"
#include "LoadingProcessInterface.h"

#include "LyraExperienceManagerComponent.generated.h"

#define UE_API LYRAGAME_API

namespace UE::GameFeatures { struct FResult; }

class ULyraExperienceDefinition;
// Experience各个阶段的加载代理,是一个多播.
DECLARE_MULTICAST_DELEGATE_OneParam(FOnLyraExperienceLoaded, const ULyraExperienceDefinition* /*Experience*/);

// 用来表明Experience的加载状态的枚举,整个状态比较复杂,同时包含异步和同步的过程
enum class ELyraExperienceLoadState
{
	Unloaded,
	Loading,
	LoadingGameFeatures,
	LoadingChaosTestingDelay,
	ExecutingActions,
	Loaded,
	Deactivating
};

/*
 * 管理体验的游戏状态组件,非常重要
 * 它在GameState的构造函数中创建,开启了网络同步的功能用来传递Experience
 * 
 */
UCLASS(MinimalAPI)
class ULyraExperienceManagerComponent final : public UGameStateComponent, public ILoadingProcessInterface
{
	GENERATED_BODY()

public:

	UE_API ULyraExperienceManagerComponent(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	//~UActorComponent interface
	/**
	 * Ends gameplay for this component.
	 * Called from AActor::EndPlay only if bHasBegunPlay is true
	 * 
	 * 结束此组件的游戏进程。
	 * 仅在 bHasBegunPlay 为真时，从 AActor::EndPlay 中调用此函数。
	 */
	UE_API virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
	//~End of UActorComponent interface

	//~ILoadingProcessInterface interface
	// Checks to see if this object implements the interface, and if so asks whether or not we should be currently showing a loading screen
	// 检查该对象是否实现了该接口，如果实现了则询问是否当前应显示加载界面
	UE_API virtual bool ShouldShowLoadingScreen(FString& OutReason) const override;
	//~End of ILoadingProcessInterface

	// Tries to set the current experience, either a UI or gameplay one
	// 尝试设置当前的体验，可以是用户界面体验，也可以是游戏体验。
	UE_API void SetCurrentExperience(FPrimaryAssetId ExperienceId);

	// Ensures the delegate is called once the experience has been loaded,
	// before others are called.
	// However, if the experience has already loaded, calls the delegate immediately.
	// 确保在体验加载完成后调用该委托函数，
	// 而不会在其他函数被调用之前再调用它。
	// 但若体验已加载完成，则会立即调用该委托函数。
	UE_API void CallOrRegister_OnExperienceLoaded_HighPriority(FOnLyraExperienceLoaded::FDelegate&& Delegate);

	// Ensures the delegate is called once the experience has been loaded
	// If the experience has already loaded, calls the delegate immediately
	// 确保在体验加载完成后调用委托函数
	// 如果体验已经加载完成，则立即调用该委托函数
	UE_API void CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate&& Delegate);

	// Ensures the delegate is called once the experience has been loaded
	// If the experience has already loaded, calls the delegate immediately
	// 确保在体验加载完成后调用委托函数
	// 如果体验已经加载完成，则立即调用该委托函数
	UE_API void CallOrRegister_OnExperienceLoaded_LowPriority(FOnLyraExperienceLoaded::FDelegate&& Delegate);

	// This returns the current experience if it is fully loaded, asserting otherwise
	// (i.e., if you called it too soon)
	// 此函数会返回当前的体验状态，如果该体验已完全加载则返回该状态，否则会抛出错误（即，如果您过早调用此函数的话）
	UE_API const ULyraExperienceDefinition* GetCurrentExperienceChecked() const;

	// Returns true if the experience is fully loaded
	// 若体验已完全加载，则返回 true
	UE_API bool IsExperienceLoaded() const;

private:
	//由网络同步过来的Experience从而启动加载,这是客户端的Experience加载启动如果
	UFUNCTION()
	void OnRep_CurrentExperience();

	//开始加载
	void StartExperienceLoad();

	//加载完成
	void OnExperienceLoadComplete();

	//当一个GameFeature插件加载完毕,从而减少需要加载GameFeature插件计数.在Experience加载过程中用于计数
	void OnGameFeaturePluginLoadComplete(const UE::GameFeatures::FResult& Result);

	//当Experience完全加载完毕时,需要开启对应的Action列表,并在Action列表执行完毕后,启动之前注册得高中低优先级代理,最后重置用户设置.
	void OnExperienceFullLoadCompleted();

	//当Experience退出时,需要卸载对应得Action,其中一个卸载完成时,增加观察计数.
	void OnActionDeactivationCompleted();

	//当Experience退出时,所有Action都卸载后,对后续内容进行处理,比如垃圾回收,卸载等.
	void OnAllActionsDeactivated();

private:

	//当前正在使用的体验
	UPROPERTY(ReplicatedUsing=OnRep_CurrentExperience)
	TObjectPtr<const ULyraExperienceDefinition> CurrentExperience;

	//目前体验的工作状态
	ELyraExperienceLoadState LoadState = ELyraExperienceLoadState::Unloaded;

	//正在加载的游戏特性插件数
	int32 NumGameFeaturePluginsLoading = 0;

	//游戏特性插件对应的URL数组
	TArray<FString> GameFeaturePluginURLs;

	//观察到的停留数,用于Action计数
	int32 NumObservedPausers = 0;
	//期望的停留数,用于Action计数
	int32 NumExpectedPausers = 0;

	/**
	 * Delegate called when the experience has finished loading just before others
	 * (e.g., subsystems that set up for regular gameplay)
	 */
	/**
	 * 当体验在其他部分加载完成之前就已经完成加载时会触发此委托。
	 * （例如，那些为常规游戏流程做准备的子系统)
	*/
	FOnLyraExperienceLoaded OnExperienceLoaded_HighPriority;

	/** Delegate called when the experience has finished loading */
	/** 当体验加载完成时所调用的委托函数 */
	FOnLyraExperienceLoaded OnExperienceLoaded;

	/** Delegate called when the experience has finished loading */
	/** 当体验加载完成时所调用的委托函数 */
	FOnLyraExperienceLoaded OnExperienceLoaded_LowPriority;
};

#undef UE_API

```


### 后调用流程
当我们游戏体验加载完毕后就会调用三个优先级的代理.如果注册代理的时候已经是加载完毕状态,则直接调用.
我们可以通过优先级来处理不同代理之间的时序问题.





``` cpp
void ULyraExperienceManagerComponent::OnExperienceFullLoadCompleted()
{
	//省略若干代码.

	//到这里加载完成了.
	LoadState = ELyraExperienceLoadState::Loaded;

	// 呼叫执行各个级别的代理 这里通过优先级的控制 使得代理事件之间可以进行时序的区分
	
	OnExperienceLoaded_HighPriority.Broadcast(CurrentExperience);
	OnExperienceLoaded_HighPriority.Clear();

	OnExperienceLoaded.Broadcast(CurrentExperience);
	OnExperienceLoaded.Clear();

	OnExperienceLoaded_LowPriority.Broadcast(CurrentExperience);
	OnExperienceLoaded_LowPriority.Clear();

	// Apply any necessary scalability settings
	// 应用任何必要的扩展性设置

#if !UE_SERVER
	//@XGTODO:这里需要去变更以下对应的用户设置,目前我们还没有写整个类.所以注释掉了.
	//ULyraSettingsLocal::Get()->OnExperienceLoaded();
#endif
}
```

此处给出几个使用示例,我们会在课程后面写到这些代码

这个是前端页面的GameState组件,它需要在Experience进行ControllFlow流程
``` cpp
void ULyraFrontendStateComponent::BeginPlay()
{
	Super::BeginPlay();

	// Listen for the experience load to complete
	// 监听体验加载完成的进程

	AGameStateBase* GameState = GetGameStateChecked<AGameStateBase>();
	ULyraExperienceManagerComponent* ExperienceComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
	check(ExperienceComponent);

	// This delegate is on a component with the same lifetime as this one, so no need to unhook it in
	// 此委托与当前组件具有相同的生命周期，因此无需对其进行解绑操作。
	ExperienceComponent->CallOrRegister_OnExperienceLoaded_HighPriority(FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}
void ULyraFrontendStateComponent::OnExperienceLoaded(const ULyraExperienceDefinition* Experience)
{
	FControlFlow& Flow = FControlFlowStatics::Create(this, TEXT("FrontendFlow"))
		.QueueStep(TEXT("Wait For User Initialization"), this, &ThisClass::FlowStep_WaitForUserInitialization)
		.QueueStep(TEXT("Try Show Press Start Screen"), this, &ThisClass::FlowStep_TryShowPressStartScreen)
		.QueueStep(TEXT("Try Join Requested Session"), this, &ThisClass::FlowStep_TryJoinRequestedSession)
		.QueueStep(TEXT("Try Show Main Screen"), this, &ThisClass::FlowStep_TryShowMainScreen);

	Flow.ExecuteFlow();

	FrontEndFlow = Flow.AsShared();
}


```

以下是异步蓝图节点,主要是为了方便在蓝图中使用,当我们需要在角色或者其他蓝图类的Beinplay进行相关操作时,必须确认Experience已经加载了!
``` cpp
void UAsyncAction_ExperienceReady::Step2_ListenToExperienceLoading(AGameStateBase* GameState)
{
	check(GameState);
	ULyraExperienceManagerComponent* ExperienceComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
	check(ExperienceComponent);

	if (ExperienceComponent->IsExperienceLoaded())
	{
		UWorld* World = GameState->GetWorld();
		check(World);

		// The experience happened to be already loaded, but still delay a frame to
		// make sure people don't write stuff that relies on this always being true
		//@TODO: Consider not delaying for dynamically spawned stuff / any time after the loading screen has dropped?
		//@TODO: Maybe just inject a random 0-1s delay in the experience load itself?
		World->GetTimerManager().SetTimerForNextTick(FTimerDelegate::CreateUObject(this, &ThisClass::Step4_BroadcastReady));
	}
	else
	{
		ExperienceComponent->CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::Step3_HandleExperienceLoaded));
	}
}


```
其余地方还有GameMode,PlayerState,ULyraTeamCreationComponent,ULyraBotCreationComponent等等位置.
此处不再列出.

## 总结
本节我们将Experience的核心加载流程进行缜密的剖析,难度非常高.
因为涉及到了资产的同步异步管理,网络初始化的时序,GameFeature是如何更加优雅地嵌入到引擎流程中.
关于AssetManger和Settings,我们还留了一点尾巴,将在后面处理.

