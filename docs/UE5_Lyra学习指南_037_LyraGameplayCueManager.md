# UE5_Lyra学习指南_037_LyraGameplayCueManager

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_037\_LyraGameplayCueManager](#ue5_lyra学习指南_037_lyragameplaycuemanager)
	- [概述](#概述)
	- [配置](#配置)
	- [加载模式](#加载模式)
	- [核心逻辑](#核心逻辑)
		- [绑定代理](#绑定代理)
		- [Tag引用加载](#tag引用加载)
		- [实际加载](#实际加载)
		- [PIE产生的问题](#pie产生的问题)
	- [Degbug日志](#degbug日志)
	- [外部调用](#外部调用)
	- [代码](#代码)
	- [总结](#总结)



## 概述
这节我们主要讲解ULyraGameplayCueManager,它是专门管理游戏玩法提示的工具类
## 配置
DefaultGame.ini
``` ini
[/Script/GameplayAbilities.AbilitySystemGlobals]
; GAS的全局数据类
AbilitySystemGlobalsClassName=/Script/LyraGame.LyraAbilitySystemGlobals
; 使用HUD下的调试目标
bUseDebugTargetFromHud=True
; 存储有关有效属性的最小值、最大值以及堆叠规则的信息 
GlobalAttributeMetaDataTableName=None
; 视觉特效指定所使用的类
GlobalGameplayCueManagerClass=/Script/LyraGame.LyraGameplayCueManager
; 管理类是使用的名字
GlobalGameplayCueManagerName=None
; 视觉特效的路径
+GameplayCueNotifyPaths=/Game/GameplayCueNotifies
; 视觉特效的路径
+GameplayCueNotifyPaths=/Game/GameplayCues
;;;;;

```
## 加载模式
``` cpp
enum class ELyraEditorLoadMode
{
	// Loads all cues upfront; longer loading speed in the editor but short PIE times and effects never fail to play
	// 全部预加载提示音；编辑器中的加载速度更快，但 PIE 时间较短，且所有效果都能正常播放
	LoadUpfront,

	// Outside of editor: Async loads as cue tag are registered
	// In editor: Async loads when cues are invoked
	//   Note: This can cause some 'why didn't I see the effect for X' issues in PIE and is good for iteration speed but otherwise bad for designers
	// 在编辑器外：当注册了异步加载的提示标签时会进行异步加载
	// 在编辑器内：在调用提示时会进行异步加载
	// 请注意：这可能会导致在 PIE 中出现“为什么我没有看到针对 X 的效果”这样的问题，对于迭代速度而言是有好处的，但对于设计师而言则不太理想。
	PreloadAsCuesAreReferenced_GameOnly,

	// Async loads as cue tag are registered
	// 异步加载的提示标签已注册
	PreloadAsCuesAreReferenced
};
```
``` cpp
namespace LyraGameplayCueManagerCvars
{
	// 命令行激活方法 显示所有与已已经加载的GameplayCue通过LyraGameplayCueManager.
	static FAutoConsoleCommand CVarDumpGameplayCues(
		TEXT("Lyra.DumpGameplayCues"),
		TEXT("Shows all assets that were loaded via LyraGameplayCueManager and are currently in memory."),
		FConsoleCommandWithArgsDelegate::CreateStatic(ULyraGameplayCueManager::DumpGameplayCues));

	static ELyraEditorLoadMode LoadMode = ELyraEditorLoadMode::LoadUpfront;
}
```
这个加载模式理解起来是有一点歧义的.
简而言之.默认情况下就是LoadUpfront,全部预先加载,也就是走引擎默认的加载流程.
其次PreloadAsCuesAreReferenced,异步加载受到引用的.这个就是走我们自己定义的加载流程!
最特殊的是PreloadAsCuesAreReferenced_GameOnly,游戏的时候走我们自己定义的加载流程,然后编辑器下需要调用的时候才去加载,这样迭代起来很快,但是会产生修改资产后未生效的问题!

## 核心逻辑
### 绑定代理
如果是模式1,全部走引擎的,那么此处无作用.直接被返回了
``` cpp
void ULyraGameplayCueManager::UpdateDelayLoadDelegateListeners()
{
	// 移除GameTag加载完成后的代理
	UGameplayTagsManager::Get().OnGameplayTagLoadedDelegate.RemoveAll(this);
	
	/** 在垃圾回收完成后被调用（如果启用了增量清理，则在清理阶段之前；若未启用则在清理阶段之后） */
	FCoreUObjectDelegates::GetPostGarbageCollect().RemoveAll(this);
	
	/** 在加载地图操作完成后发送 */
	FCoreUObjectDelegates::PostLoadMapWithWorld.RemoveAll(this);

	switch (LyraGameplayCueManagerCvars::LoadMode)
	{
		// 已经是走的引擎流程了.不需要做任何的自定义
	case ELyraEditorLoadMode::LoadUpfront:
		return;
	case ELyraEditorLoadMode::PreloadAsCuesAreReferenced_GameOnly:
#if WITH_EDITOR
		if (GIsEditor)
		{
			// 编辑器下直接返回. 不需要做按照引用关系加载 这里就会导致
			// Note: This can cause some 'why didn't I see the effect for X' issues in PIE and is good for iteration speed but otherwise bad for designers
			return;
		}
		// 只在打包后运行 启用按引用关系加载
#endif
		break;
		// 全都走自定义流程 启动按引用关系加载
	case ELyraEditorLoadMode::PreloadAsCuesAreReferenced:
		break;
	}

	UGameplayTagsManager::Get().OnGameplayTagLoadedDelegate.AddUObject(this, &ThisClass::OnGameplayTagLoaded);
	FCoreUObjectDelegates::GetPostGarbageCollect().AddUObject(this, &ThisClass::HandlePostGarbageCollect);
	/** 在加载地图操作完成后发送 */
	FCoreUObjectDelegates::PostLoadMapWithWorld.AddUObject(this, &ThisClass::HandlePostLoadMap);
}
```
### Tag引用加载
``` cpp

void ULyraGameplayCueManager::OnGameplayTagLoaded(const FGameplayTag& Tag)
{
	// 锁 保护队列
	FScopeLock ScopeLock(&LoadedGameplayTagsToProcessCS);
	bool bStartTask = LoadedGameplayTagsToProcess.Num() == 0;
	/**
	 *    这里尝试获取当前对象序列化的上下文（SerializeContext）。
	 *    在UE中，当对象被序列化（加载）时，会有一个上下文记录当前正在序列化的对象。
	 *    `FUObjectThreadContext`是每个线程的UObject相关上下文，通过`GetSerializeContext()`获取序列化上下文。
	 *    如果存在序列化上下文，那么我们可以获取到当前正在被序列化的对象（`SerializedObject`），这个对象通常是拥有这个Tag的对象（或者与这个Tag加载相关的对象）。
	 *    如果上下文不存在，则`OwningObject`为nullptr。
	 */
	FUObjectSerializeContext* LoadContext = FUObjectThreadContext::Get().GetSerializeContext();
	UObject* OwningObject = LoadContext ? LoadContext->SerializedObject : nullptr;
	LoadedGameplayTagsToProcess.Emplace(Tag, OwningObject);
	// 如果是首个任务就需要创建Task 如果不是直接塞到队列里面即可.
	if (bStartTask)
	{
		TGraphTask<FGameplayCueTagThreadSynchronizeGraphTask>::CreateTask().ConstructAndDispatchWhenReady([]()
			{
				/* 此操作是否在 MainLoop() 函数内部进行 */
				if (GIsRunning)
				{
					if (ULyraGameplayCueManager* StrongThis = Get())
					{
						// If we are garbage collecting we cannot call StaticFindObject (or a few other static uobject functions), so we'll just wait until the GC is over and process the tags then
						// 如果正在进行垃圾回收操作，我们不能调用 StaticFindObject 函数（或者一些其他与静态 UObject 相关的函数），因此我们只能等到垃圾回收结束之后，再处理这些标签。
						if (IsGarbageCollecting())
						{
							StrongThis->bProcessLoadedTagsAfterGC = true;
						}
						else
						{
							StrongThis->ProcessLoadedTags();
						}
					}
				}
			});
	}
}


```
``` cpp
void ULyraGameplayCueManager::ProcessLoadedTags()
{
	// 把需要处理的标签进行拷贝
	TArray<FLoadedGameplayTagToProcessData> TaskLoadedGameplayTagsToProcess;
	{
		// Lock LoadedGameplayTagsToProcess just long enough to make a copy and clear
		// 对待要处理的游戏玩法标签进行锁定，锁定时间仅足够进行复制操作并完成清理工作。
		FScopeLock TaskScopeLock(&LoadedGameplayTagsToProcessCS);
		TaskLoadedGameplayTagsToProcess = LoadedGameplayTagsToProcess;
		LoadedGameplayTagsToProcess.Empty();
	}

	// This might return during shutdown, and we don't want to proceed if that is the case
	// 这种情况可能会在系统关闭时发生，如果出现这种情况，我们就不应继续操作了。
	if (GIsRunning)
	{
		if (RuntimeGameplayCueObjectLibrary.CueSet)
		{
			for (const FLoadedGameplayTagToProcessData& LoadedTagData : TaskLoadedGameplayTagsToProcess)
			{
				// 是否包含我们对应的Tag
				if (RuntimeGameplayCueObjectLibrary.CueSet->GameplayCueDataMap.Contains(LoadedTagData.Tag))
				{
					/**
					 与 ！IsValid() 的实现略有不同：如果此指针曾经指向一个 UObject，但现在已经不再指向任何对象，并且在此期间也没有被赋值或重置，那么该函数将返回 true。
					 * @参数 bIncludingIfPendingKill 如果为 true，则将挂起销毁对象视为过时对象。
					 * @参数 bThreadsafeTest 在非游戏线程中测试时将其设置为 true。如果 WeakObjPtr 指向现有对象（不检查标志），则结果为 false。
					 * @返回 如果此指针曾经指向真实对象，但现在不再指向任何对象，则返回 true。
					 * 
					 */
					// 已经加载过.或者对象是有效的.就没必要去处理它了.
					if (!LoadedTagData.WeakOwner.IsStale())
					{
						ProcessTagToPreload(LoadedTagData.Tag, LoadedTagData.WeakOwner.Get());
					}
				}
			}
		}
		else
		{
			// 我们预期存放的地方是空的 无法处理所以跳过.
			UE_LOG(LogLyra, Warning, TEXT("ULyraGameplayCueManager::OnGameplayTagLoaded processed loaded tag(s) but RuntimeGameplayCueObjectLibrary.CueSet was null. Skipping processing."));
		}
	}
}


```
### 实际加载
``` cpp

void ULyraGameplayCueManager::ProcessTagToPreload(const FGameplayTag& Tag, UObject* OwningObject)
{
	// 判断一下我们的管理器执行模型
	switch (LyraGameplayCueManagerCvars::LoadMode)
	{
		// 走引擎流程 直接跳过
	case ELyraEditorLoadMode::LoadUpfront:
		return;
	case ELyraEditorLoadMode::PreloadAsCuesAreReferenced_GameOnly:
#if WITH_EDITOR
		if (GIsEditor)
		{
			// 编辑器下走引擎流程.直接跳过
			// 实际引擎也没有走 所以会导致PIE下的问题.
			
			return;
		}
#endif
		break;
		// 打包后运行走自定义流程
	case ELyraEditorLoadMode::PreloadAsCuesAreReferenced:
		// 无论编辑器还是运行时都自自定义流程
		break;
	}

	check(RuntimeGameplayCueObjectLibrary.CueSet);

	// 必须要有对应的Tag
	int32* DataIdx = RuntimeGameplayCueObjectLibrary.CueSet->GameplayCueDataMap.Find(Tag);
	if (DataIdx && RuntimeGameplayCueObjectLibrary.CueSet->GameplayCueData.IsValidIndex(*DataIdx))
	{
		const FGameplayCueNotifyData& CueData = RuntimeGameplayCueObjectLibrary.CueSet->GameplayCueData[*DataIdx];

		UClass* LoadedGameplayCueClass = FindObject<UClass>(nullptr, *CueData.GameplayCueNotifyObj.ToString());
		// 看看能不能知道对应的类类型 如果找不到就要先去加载
		if (LoadedGameplayCueClass)
		{
			RegisterPreloadedCue(LoadedGameplayCueClass, OwningObject);
		}
		else
		{
			// 它应当是空的
			bool bAlwaysLoadedCue = OwningObject == nullptr;
			TWeakObjectPtr<UObject> WeakOwner = OwningObject;
			StreamableManager.RequestAsyncLoad(CueData.GameplayCueNotifyObj,
				FStreamableDelegate::CreateUObject(this, &ThisClass::OnPreloadCueComplete, CueData.GameplayCueNotifyObj, WeakOwner, bAlwaysLoadedCue),
				FStreamableManager::DefaultAsyncLoadPriority,
				false,
				false,
				TEXT("GameplayCueManager"));
		}
	}
}


```
### PIE产生的问题
``` cpp

bool ULyraGameplayCueManager::ShouldSyncLoadMissingGameplayCues() const
{
	return false;
}

bool ULyraGameplayCueManager::ShouldAsyncLoadMissingGameplayCues() const
{
	return true;
}
bool ULyraGameplayCueManager::ShouldAsyncLoadRuntimeObjectLibraries() const
{
	switch (LyraGameplayCueManagerCvars::LoadMode)
	{
	case ELyraEditorLoadMode::LoadUpfront:
		// 无论是服务器还是客户端,因为我们已经全部加载好了 这里就直接走引擎的流程
		return true;
	case ELyraEditorLoadMode::PreloadAsCuesAreReferenced_GameOnly:
#if WITH_EDITOR
		if (GIsEditor)
		{	// 这种是 编辑器下 会有我们自己按需进行加载的资产 所以需要我们自己去加载 就不要走引擎的流程了.
			// 编辑器器 无论客户端服务端都走自定义加载 false
			return false;
		}

#endif
		// 打包后 客户端 false 走自定义加载流程
		// 打包后 服务端 true 走引擎流程
		break;
	case ELyraEditorLoadMode::PreloadAsCuesAreReferenced:
		// 打包后 客户端 false 走自定义加载流程
		// 打包后 服务端 true 走引擎流程
		break;
	}

	// 客户端会延迟加载特效
	// 客户端不应当但异步去加载运行时的对象库
	// 所以这里会返回false 走我们自定义的异步加载对象库流程

	// 服务端不会延迟加载特效
	// 那么服务端直接走引擎的流程就可以了.
	
	return !ShouldDelayLoadGameplayCues();
}

```
父类因为这个函数返回的是false
``` cpp
void UGameplayCueManager::InitializeRuntimeObjectLibrary()
{
	FAssetRegistryModule& AssetRegistryModule = FModuleManager::LoadModuleChecked<FAssetRegistryModule>(TEXT("AssetRegistry"));
	if (AssetRegistryModule.Get().IsGathering() && ShouldDeferScanningRuntimeLibraries())
	{
		ABILITY_LOG(Display, TEXT("Asset registry still gathering. Deferring InitializeRuntimeObjectLibrary."));
		if (!AssetRegistryModule.Get().OnKnownGathersComplete().IsBoundToObject(this))
		{
			AssetRegistryModule.Get().OnKnownGathersComplete().AddUObject(this, &UGameplayCueManager::InitializeRuntimeObjectLibrary);
		}
		return;
	}

	ABILITY_LOG(Display, TEXT("Processing InitializeRuntimeObjectLibrary."))

	AssetRegistryModule.Get().OnKnownGathersComplete().RemoveAll(this);

	UE_SCOPED_ENGINE_ACTIVITY(TEXT("Initializing GameplayCueManager Runtime Object Library"));

	RuntimeGameplayCueObjectLibrary.Paths = GetAlwaysLoadedGameplayCuePaths();
	if (RuntimeGameplayCueObjectLibrary.CueSet == nullptr)
	{
		RuntimeGameplayCueObjectLibrary.CueSet = NewObject<UGameplayCueSet>(this, TEXT("GlobalGameplayCueSet"));
	}

	RuntimeGameplayCueObjectLibrary.CueSet->Empty();
	RuntimeGameplayCueObjectLibrary.bHasBeenInitialized = true;
	
	RuntimeGameplayCueObjectLibrary.bShouldSyncScan = ShouldSyncScanRuntimeObjectLibraries();
	RuntimeGameplayCueObjectLibrary.bShouldSyncLoad = ShouldSyncLoadRuntimeObjectLibraries();
	RuntimeGameplayCueObjectLibrary.bShouldAsyncLoad = ShouldAsyncLoadRuntimeObjectLibraries();

	InitObjectLibrary(RuntimeGameplayCueObjectLibrary);
}
```
``` cpp

TSharedPtr<FStreamableHandle> UGameplayCueManager::InitObjectLibrary(FGameplayCueObjectLibrary& Lib)
{
	TSharedPtr<FStreamableHandle> RetVal;

	DECLARE_SCOPE_CYCLE_COUNTER(TEXT("Loading Library"), STAT_ObjectLibrary, STATGROUP_LoadTime);

	// Instantiate the UObjectLibraries if they aren't there already
	if (!Lib.StaticObjectLibrary)
	{
		Lib.StaticObjectLibrary = UObjectLibrary::CreateLibrary(UGameplayCueNotify_Static::StaticClass(), true, GIsEditor && !IsRunningCommandlet());
		if (GIsEditor)
		{
			Lib.StaticObjectLibrary->bIncludeOnlyOnDiskAssets = false;
		}
	}
	if (!Lib.ActorObjectLibrary)
	{
		Lib.ActorObjectLibrary = UObjectLibrary::CreateLibrary(AGameplayCueNotify_Actor::StaticClass(), true, GIsEditor && !IsRunningCommandlet());
		if (GIsEditor)
		{
			Lib.ActorObjectLibrary->bIncludeOnlyOnDiskAssets = false;
		}
	}	

	Lib.bHasBeenInitialized = true;

#if WITH_EDITOR
	bAccelerationMapOutdated = false;
#endif

	FScopeCycleCounterUObject PreloadScopeActor(Lib.ActorObjectLibrary);

	// ------------------------------------------------------------------------------------------------------------------
	//	Scan asset data. If bShouldSyncScan is false, whatever state the asset registry is in will be what is returned.
	// ------------------------------------------------------------------------------------------------------------------

	{
		//SCOPE_LOG_TIME_IN_SECONDS(*FString::Printf(TEXT("UGameplayCueManager::InitObjectLibraries    Actors. Paths: %s"), *FString::Join(Lib.Paths, TEXT(", "))), nullptr)
		Lib.ActorObjectLibrary->LoadBlueprintAssetDataFromPaths(Lib.Paths, Lib.bShouldSyncScan);
	}
	{
		//SCOPE_LOG_TIME_IN_SECONDS(*FString::Printf(TEXT("UGameplayCueManager::InitObjectLibraries    Objects")), nullptr)
		Lib.StaticObjectLibrary->LoadBlueprintAssetDataFromPaths(Lib.Paths, Lib.bShouldSyncScan);
	}

	// ---------------------------------------------------------
	// Sync load if told to do so	
	// ---------------------------------------------------------
	if (Lib.bShouldSyncLoad)
	{
#if STATS
		FString PerfMessage = FString::Printf(TEXT("Fully Loaded GameplayCueNotify object library"));
		SCOPE_LOG_TIME_IN_SECONDS(*PerfMessage, nullptr)
#endif
		Lib.ActorObjectLibrary->LoadAssetsFromAssetData();
		Lib.StaticObjectLibrary->LoadAssetsFromAssetData();
	}

	// ---------------------------------------------------------
	// Look for GameplayCueNotifies that handle events
	// ---------------------------------------------------------
	
	TArray<FAssetData> ActorAssetDatas;
	Lib.ActorObjectLibrary->GetAssetDataList(ActorAssetDatas);

	TArray<FAssetData> StaticAssetDatas;
	Lib.StaticObjectLibrary->GetAssetDataList(StaticAssetDatas);

	TArray<FGameplayCueReferencePair> CuesToAdd;
	TArray<FSoftObjectPath> AssetsToLoad;

	// ------------------------------------------------------------------------------------------------------------------
	// Build Cue lists for loading. Determines what from the obj library needs to be loaded
	// ------------------------------------------------------------------------------------------------------------------
	BuildCuesToAddToGlobalSet(ActorAssetDatas, GET_MEMBER_NAME_CHECKED(AGameplayCueNotify_Actor, GameplayCueName), CuesToAdd, AssetsToLoad, Lib.ShouldLoad);
	BuildCuesToAddToGlobalSet(StaticAssetDatas, GET_MEMBER_NAME_CHECKED(UGameplayCueNotify_Static, GameplayCueName), CuesToAdd, AssetsToLoad, Lib.ShouldLoad);

	const FName PropertyName = GET_MEMBER_NAME_CHECKED(AGameplayCueNotify_Actor, GameplayCueName);
	check(PropertyName == GET_MEMBER_NAME_CHECKED(UGameplayCueNotify_Static, GameplayCueName));

	// ------------------------------------------------------------------------------------------------------------------------------------
	// Add these cues to the set. The UGameplayCueSet is the data structure used in routing the gameplay cue events at runtime.
	// ------------------------------------------------------------------------------------------------------------------------------------
	UGameplayCueSet* SetToAddTo = Lib.CueSet;
	if (!SetToAddTo)
	{
		SetToAddTo = RuntimeGameplayCueObjectLibrary.CueSet;
	}
	check(SetToAddTo);
	SetToAddTo->AddCues(CuesToAdd);

	// --------------------------------------------
	// Start loading them if necessary
	// --------------------------------------------
	if (Lib.bShouldAsyncLoad)
	{
		auto ForwardLambda = [](TArray<FSoftObjectPath> AssetList, FOnGameplayCueNotifySetLoaded OnLoadedDelegate)
		{
			OnLoadedDelegate.ExecuteIfBound(AssetList);
		};

		if (AssetsToLoad.Num() > 0)
		{
			FStreamableDelegate Del = FStreamableDelegate::CreateStatic(ForwardLambda, AssetsToLoad, Lib.OnLoaded);
			GameplayCueAssetHandle = StreamableManager.RequestAsyncLoad(MoveTemp(AssetsToLoad), MoveTemp(Del), Lib.AsyncPriority);
			RetVal = GameplayCueAssetHandle;
		}
		else
		{
			// Still fire the delegate even if nothing was found to load
			Lib.OnLoaded.ExecuteIfBound(MoveTemp(AssetsToLoad));
		}
	}

	// Build Tag Translation table
	TranslationManager.BuildTagTranslationTable();
	return RetVal;
}
```

## Degbug日志
``` cpp
void ULyraGameplayCueManager::DumpGameplayCues(const TArray<FString>& Args)
{
	ULyraGameplayCueManager* GCM = Cast<ULyraGameplayCueManager>(UAbilitySystemGlobals::Get().GetGameplayCueManager());
	// 类型不匹配 失败
	if (!GCM)
	{
		UE_LOG(LogLyra, Error, TEXT("DumpGameplayCues failed. No ULyraGameplayCueManager found."));
		return;
	}

	// 获取是否需要开启引用打印
	const bool bIncludeRefs = Args.Contains(TEXT("Refs"));

	UE_LOG(LogLyra, Log, TEXT("=========== Dumping Always Loaded Gameplay Cue Notifies ==========="));
	// 打印总是加载的提示
	for (UClass* CueClass : GCM->AlwaysLoadedCues)
	{
		UE_LOG(LogLyra, Log, TEXT("  %s"), *GetPathNameSafe(CueClass));
	}
	// 打印由于内容引用而预先加载的提示
	UE_LOG(LogLyra, Log, TEXT("=========== Dumping Preloaded Gameplay Cue Notifies ==========="));
	for (UClass* CueClass : GCM->PreloadedCues)
	{
		TSet<FObjectKey>* ReferencerSet = GCM->PreloadedCueReferencers.Find(CueClass);
		int32 NumRefs = ReferencerSet ? ReferencerSet->Num() : 0;
		// 打印引用数量
		UE_LOG(LogLyra, Log, TEXT("  %s (%d refs)"), *GetPathNameSafe(CueClass), NumRefs);

		// 如果开启了引用打印 把引用关系也打出来
		if (bIncludeRefs && ReferencerSet)
		{
			for (const FObjectKey& Ref : *ReferencerSet)
			{
				UObject* RefObject = Ref.ResolveObjectPtr();
				UE_LOG(LogLyra, Log, TEXT("    ^- %s"), *GetPathNameSafe(RefObject));
			}
		}
	}
	// 打印需要加载的但是还没加载的提示
	UE_LOG(LogLyra, Log, TEXT("=========== Dumping Gameplay Cue Notifies loaded on demand ==========="));
	int32 NumMissingCuesLoaded = 0;
	if (GCM->RuntimeGameplayCueObjectLibrary.CueSet)
	{
		for (const FGameplayCueNotifyData& CueData : GCM->RuntimeGameplayCueObjectLibrary.CueSet->GameplayCueData)
		{
			if (CueData.LoadedGameplayCueClass && !GCM->AlwaysLoadedCues.Contains(CueData.LoadedGameplayCueClass) && !GCM->PreloadedCues.Contains(CueData.LoadedGameplayCueClass))
			{
				NumMissingCuesLoaded++;
				UE_LOG(LogLyra, Log, TEXT("  %s"), *CueData.LoadedGameplayCueClass->GetPathName());
			}
		}
	}
	// 汇总输出
	UE_LOG(LogLyra, Log, TEXT("=========== Gameplay Cue Notify summary ==========="));
	UE_LOG(LogLyra, Log, TEXT("  ... %d cues in always loaded list"), GCM->AlwaysLoadedCues.Num());
	UE_LOG(LogLyra, Log, TEXT("  ... %d cues in preloaded list"), GCM->PreloadedCues.Num());
	UE_LOG(LogLyra, Log, TEXT("  ... %d cues loaded on demand"), NumMissingCuesLoaded);
	UE_LOG(LogLyra, Log, TEXT("  ... %d cues in total"), GCM->AlwaysLoadedCues.Num() + GCM->PreloadedCues.Num() + NumMissingCuesLoaded);
}
```
## 外部调用
``` cpp
void ULyraAssetManager::InitializeGameplayCueManager()
{
	// 专门用于 UE 启动阶段性能分析的工具，适合优化游戏启动时间
	SCOPED_BOOT_TIMING("ULyraAssetManager::InitializeGameplayCueManager");

	// 这里先在本节注释掉
	ULyraGameplayCueManager* GCM = ULyraGameplayCueManager::Get();
	check(GCM);
	GCM->LoadAlwaysLoadedCues();
}

```

``` cpp

void ULyraGameplayCueManager::LoadAlwaysLoadedCues()
{
	// 客户端是true 从资产管理器调用过来
	if (ShouldDelayLoadGameplayCues())
	{
		UGameplayTagsManager& TagManager = UGameplayTagsManager::Get();
	
		//@TODO: Try to collect these by filtering GameplayCue. tags out of native gameplay tags?
		//@待办事项：尝试通过从原生游戏标签中剔除“游戏提示”标签的方式来收集这些内容？
		TArray<FName> AdditionalAlwaysLoadedCueTags;

		for (const FName& CueTagName : AdditionalAlwaysLoadedCueTags)
		{
			FGameplayTag CueTag = TagManager.RequestGameplayTag(CueTagName, /*ErrorIfNotFound=*/ false);
			if (CueTag.IsValid())
			{
				ProcessTagToPreload(CueTag, nullptr);
			}
			else
			{
				UE_LOG(LogLyra, Warning, TEXT("ULyraGameplayCueManager::AdditionalAlwaysLoadedCueTags contains invalid tag %s"), *CueTagName.ToString());
			}
		}
	}
}

```

## 代码
``` cpp
/**
 * ULyraGameplayCueManager
 *
 * Game-specific manager for gameplay cues
 * 专门针对游戏玩法提示的管理工具
 */
UCLASS()
class ULyraGameplayCueManager : public UGameplayCueManager
{
	GENERATED_BODY()

public:
	// 构造函数
	ULyraGameplayCueManager(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	// 单例获取方法 注意需要进行配置默认类
	static ULyraGameplayCueManager* Get();

	//~UGameplayCueManager interface
	/** 当管理器首次创建时调用 */
	virtual void OnCreated() override;

	// 是否应当异步加载运行时所使用的对象库 父类默认时true 我们需要根据实际情况进行调整
	virtual bool ShouldAsyncLoadRuntimeObjectLibraries() const override;
	
	/** 若为真，则会同步加载缺失的游戏提示内容  默认返回False 不应该同步加载*/ 
	virtual bool ShouldSyncLoadMissingGameplayCues() const override;

	/** 若为真，则会异步加载缺失的游戏提示信息，并在加载完成时执行该提示信息  返回true*/
	virtual bool ShouldAsyncLoadMissingGameplayCues() const override;
	//~End of UGameplayCueManager interface

	// 日志方法 打印出来 总是加载,由于引用被加载,需要但是还没加载  的游戏提示信息.
	static void DumpGameplayCues(const TArray<FString>& Args);

	// When delay loading cues, this will load the cues that must be always loaded anyway
	// 在延迟加载提示信息时，此功能会加载那些无论何时都必须加载的提示信息。 从资产管理器的初始化过程中调用void ULyraAssetManager::InitializeGameplayCueManager()
	void LoadAlwaysLoadedCues();

	// Updates the bundles for the singular gameplay cue primary asset
	// 更新单个游戏玩法提示主资源的包文件
	void RefreshGameplayCuePrimaryAsset();

private:
	// 在Tag加载完成后调用
	void OnGameplayTagLoaded(const FGameplayTag& Tag);

	// 在GC之后处理Tag的加载事件 避免和GC冲突
	void HandlePostGarbageCollect();

	// 处理被加载的Tag
	void ProcessLoadedTags();

	// 通过Tag去加载对象
	void ProcessTagToPreload(const FGameplayTag& Tag, UObject* OwningObject);

	// 在加载LoadedGameplayCueClass后调用 然后处理加载完成对应的对象
	void OnPreloadCueComplete(FSoftObjectPath Path, TWeakObjectPtr<UObject> OwningObject, bool bAlwaysLoadedCue);
	// 记录下来被加载的Cue,它可能是总是加载的,或者收到OwningObject的引用被加载了
	void RegisterPreloadedCue(UClass* LoadedGameplayCueClass, UObject* OwningObject);

	// 在地图加载完成后调用
	void HandlePostLoadMap(UWorld* NewWorld);

	// 更新延迟加载代理的监听回调 如果是全部提前加载好 就不需要管 如果有自定义流程 就需要监听各个回调的接口
	void UpdateDelayLoadDelegateListeners();
	
	// 是否应该延迟加载有些特效 客户端应该这样做
	bool ShouldDelayLoadGameplayCues() const;

private:
	struct FLoadedGameplayTagToProcessData
	{
		FGameplayTag Tag;
		TWeakObjectPtr<UObject> WeakOwner;

		FLoadedGameplayTagToProcessData() {}
		FLoadedGameplayTagToProcessData(const FGameplayTag& InTag, const TWeakObjectPtr<UObject>& InWeakOwner) : Tag(InTag), WeakOwner(InWeakOwner) {}
	};

private:
	// Cues that were preloaded on the client due to being referenced by content
	// 由于内容引用而预先加载在客户端上的提示信息
	UPROPERTY(transient)
	TSet<TObjectPtr<UClass>> PreloadedCues;
	TMap<FObjectKey, TSet<FObjectKey>> PreloadedCueReferencers;

	// Cues that were preloaded on the client and will always be loaded (code referenced or explicitly always loaded)
	// 已预先加载在客户端上的、且永远会进行加载的提示信息（通过代码引用或明确指定始终会加载）
	UPROPERTY(transient)
	TSet<TObjectPtr<UClass>> AlwaysLoadedCues;

	// 已经加载的需要进行处理的Tag
	TArray<FLoadedGameplayTagToProcessData> LoadedGameplayTagsToProcess;
	FCriticalSection LoadedGameplayTagsToProcessCS;
	bool bProcessLoadedTagsAfterGC = false;
};

```
## 总结
该类也是属于可写 可不写的类.主要是用于优化GameplayCue的加载方式.