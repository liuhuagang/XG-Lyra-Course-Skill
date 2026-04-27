# UE5_Lyra学习指南_012_AssetManager类

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)

文章内容由小刚撰写,采用了以下多种方式:

1.口述转文字
2.AI重构
3.参考引擎源码
4.Lyra工程源码
5.结合社区论坛各位大佬的解析

- [UE5\_Lyra学习指南\_012\_AssetManager类](#ue5_lyra学习指南_012_assetmanager类)
	- [概述](#概述)
	- [配置位置](#配置位置)
	- [实例化时机](#实例化时机)
		- [初始化流程](#初始化流程)
		- [GIsEditor = ture](#giseditor--ture)
		- [if WITH\_EDITOR 和 #if WITH\_EDITORONLY\_DATA](#if-with_editor-和-if-with_editoronly_data)
			- [1. `#if WITH_EDITOR`](#1-if-with_editor)
			- [2. `#if WITH_EDITORONLY_DATA`](#2-if-with_editoronly_data)
			- [关键区别](#关键区别)
			- [何时使用？](#何时使用)
			- [注意事项](#注意事项)
	- [ULyraGameData](#ulyragamedata)
	- [LyraAssetManger](#lyraassetmanger)
	- [LyraAssetManagerStartupJob](#lyraassetmanagerstartupjob)
	- [修复关于AssetManger的引用](#修复关于assetmanger的引用)
	- [总结](#总结)



## 概述

本节课主要讲解AssetManger这个类,并讲解它在引擎初始化过程中嵌入执行的任务.
## 配置位置
本节课主要讲解AssetManger这个类资产管理器的实现，该实现会覆盖原有功能并存储游戏特定类型的数据。
预计大多数游戏都会希望重写 AssetManager 类，因为它为游戏特定的加载逻辑提供了一个理想的位置。
此类通过在 DefaultEngine.ini 中设置 'AssetManagerClassName' 来使用。
``` ini
; 游戏资产管理类
AssetManagerClassName=/Script/LyraGame.LyraAssetManager
```
因为启动流程控制的原因,一下DefaultGame.ini里面必须配置有效的蓝图资产
``` ini
[/Script/LyraGame.LyraAssetManager]
LyraGameDataPath=/Game/DefaultGameData.DefaultGameData
DefaultPawnData=/Game/Characters/Heroes/EmptyPawnData/DefaultPawnData_EmptyPawn.DefaultPawnData_EmptyPawn
```
## 实例化时机
### 初始化流程
E:\Epic\UE\UE_5.6\Engine\Source\Runtime\Engine\Private\UnrealEngine.cpp
3465行左右UEngine::InitializeObjectReferences()调用
``` cpp
/**
* Loads all Engine object references from their corresponding config entries.
*/
void UEngine::InitializeObjectReferences()
{
	UE_SCOPED_ENGINE_ACTIVITY(TEXT("Initializing Object References"));
	TRACE_LOADTIME_REQUEST_GROUP_SCOPE(TEXT("UEngine::InitializeObjectReferences"));
	SCOPED_BOOT_TIMING("UEngine::InitializeObjectReferences");
	DECLARE_SCOPE_CYCLE_COUNTER(TEXT("UEngine::InitializeObjectReferences"), STAT_InitializeObjectReferences, STATGROUP_LoadTime);

	// This initializes the tag data if it hasn't been already, we need to do this before loading any game data
	UGameplayTagsManager::Get();
	
	// 省略若干代码
	LoadEngineClass<UConsole>(ConsoleClassName, ConsoleClass);
	LoadEngineClass<UGameViewportClient>(GameViewportClientClassName, GameViewportClientClass);
	LoadEngineClass<ULocalPlayer>(LocalPlayerClassName, LocalPlayerClass);
	LoadEngineClass<AWorldSettings>(WorldSettingsClassName, WorldSettingsClass);
	LoadEngineClass<UNavigationSystemBase>(NavigationSystemClassName, NavigationSystemClass);
	LoadEngineClass<UNavigationSystemConfig>(NavigationSystemConfigClassName, NavigationSystemConfigClass);
	if (AvoidanceManagerClassName.IsValid())
	{
		LoadEngineClass<UAvoidanceManager>(AvoidanceManagerClassName, AvoidanceManagerClass);
	}
	LoadEngineClass<UGameUserSettings>(GameUserSettingsClassName, GameUserSettingsClass);
	LoadEngineClass<ALevelScriptActor>(LevelScriptActorClassName, LevelScriptActorClass);

	//省略若干代码

	if (AssetManager == nullptr)
	{
		UClass* SingletonClass = nullptr;
		if (AssetManagerClassName.ToString().Len() > 0)
		{
			SingletonClass = LoadClass<UObject>(nullptr, *AssetManagerClassName.ToString());
		}
		if (!SingletonClass)
		{
			UE_LOG(LogEngine, Fatal, TEXT("Engine config value AssetManagerClassName '%s' is not a valid class name."), *AssetManagerClassName.ToString());
		}

		AssetManager = NewObject<UAssetManager>(this, SingletonClass);
		check(AssetManager);
		if (!FParse::Param(FCommandLine::Get(), TEXT("SkipAssetScan")))
		{
			AssetManager->StartInitialLoading();
		}
	}

	const UUserInterfaceSettings* UISettings = GetDefault<UUserInterfaceSettings>(UUserInterfaceSettings::StaticClass());
	UISettings->ForceLoadResources();
}


```

---

2200行 void UEngine::Init(IEngineLoop* InEngineLoop)调用InitializeObjectReferences()
```cpp
void UEngine::Init(IEngineLoop* InEngineLoop)
{
	TRACE_CPUPROFILER_EVENT_SCOPE(UEngine::Init);
	UE_LOG(LogEngine, Log, TEXT("Initializing Engine..."));
	DECLARE_SCOPE_CYCLE_COUNTER(TEXT("Engine Initialized"), STAT_EngineStartup, STATGROUP_LoadTime);
	// ....

	InitializeObjectReferences();
	
	// ...

}
```

---

在EditorEngine.cpp 第950行调用UEngine::Init(IEngineLoop* InEngineLoop)
``` cpp
void UEditorEngine::InitEditor(IEngineLoop* InEngineLoop)
{
	// Allow remote execution of derived data builds from this point
	// TODO: This needs to be enabled earlier to allow early data builds to be remote executed.
	if (FParse::Param(FCommandLine::Get(), TEXT("ExecuteBuildsLocally")))
	{
		InitDerivedDataBuildLocalExecutor();
	}
	else
	{
		InitDerivedDataBuildRemoteExecutor();
	}
	InitDerivedDataBuildWorkers();

	// Startup UnrealEdMisc
	FUnrealEdMisc::Get();

	// Call base.
	UEngine::Init(InEngineLoop);
	
	// 省略若干代码

}
```

---

void UEditorEngine::Init(IEngineLoop* InEngineLoop) 第1311行调用InitEditor()

``` cpp
void UEditorEngine::Init(IEngineLoop* InEngineLoop)
{
	FScopedSlowTask SlowTask(100);

	DECLARE_SCOPE_CYCLE_COUNTER(TEXT("Editor Engine Initialized"), STAT_EditorEngineStartup, STATGROUP_LoadTime);

	//省略若干代码

	FEditorDelegates::BeginPIE.AddLambda([](bool)
	{
	// ...
	});

	FEditorDelegates::PrePIEEnded.AddLambda([this](bool)
	{
	// ...
	});

	FEditorDelegates::EndPIE.AddLambda([](bool)
	{
		// ...

	});

	// ...

	// Init editor.
	SlowTask.EnterProgressFrame(40);
	GEditor = this;
	InitEditor(InEngineLoop);

	//省略若干代码

```

---

在LaunchEngineLoop中的4571行调用Init();
``` cpp
int32 FEngineLoop::Init()
{

	SlowTask.EnterProgressFrame(60);

	{
		SCOPED_BOOT_TIMING("GEngine->Init");
		GEngine->Init(this);
	}

	// Call init callbacks
	{
		SCOPED_BOOT_TIMING("OnPostEngineInit.Broadcast");
		FCoreDelegates::OnPostEngineInit.Broadcast();
	}

}
```

---

在Launch.cpp中的第164行调用 
注意这里我们必须去FEngineLoop找实现,因为它是一个接口IEngineLoop

``` cpp

FEngineLoop	GEngineLoop;


int32 GuardedMain( const TCHAR* CmdLine )
{
		// 省略若干代码

		int32 ErrorLevel = EnginePreInit( CmdLine );

		// 省略若干代码

#if WITH_EDITOR
		if (GIsEditor)
		{
			ErrorLevel = EditorInit(GEngineLoop);
		}
		else
#endif
		{
			ErrorLevel = EngineInit();
		}
	}

		//省略若干代码
}
```

---

这里注意在EnginePreInit函数中
``` cpp
int32 FEngineLoop::PreInit(const TCHAR* CmdLine)
{

	// ...
	/** First part of PreInit. */
	const int32 rv1 = PreInitPreStartupScreen(CmdLine);
	if (rv1 != 0)
	{
		PreInitContext.Cleanup();
		return rv1;
	}
	/** Second part of PreInit. */
	const int32 rv2 = PreInitPostStartupScreen(CmdLine);
	if (rv2 != 0)
	{
		PreInitContext.Cleanup();
		return rv2;
	}

	return 0;
}

```



### GIsEditor = ture

在第一部分中找到对于GIsEditor的赋值.

```cpp
int32 FEngineLoop::PreInitPreStartupScreen(const TCHAR* CmdLine)
{

	// ...

#if WITH_EDITORONLY_DATA
	auto SetIsRunningAsEditor = [&IsModeSelected, &bHasEditorToken]()
	{
		checkf(!IsModeSelected(), TEXT("SetIsRunningAsEditor should not be called after mode has been selected."));
		bHasEditorToken = true;

		GIsClient = true;
		GIsServer = true;
		GIsEditor = true;
#if WITH_ENGINE
		checkf(!PRIVATE_GIsRunningCommandlet, TEXT("It should not be possible for PRIVATE_GIsRunningCommandlet to have been set when calling SetIsRunningAsEditor"));
#endif
	};

	// ...
	if (!IsModeSelected())
	{
#if UE_EDITOR
		if (!Switches.Contains(TEXT("GAME")))
		{
			SetIsRunningAsEditor();
		}
#elif WITH_ENGINE && WITH_EDITOR && WITH_EDITORONLY_DATA 
		// If a non-editor target build w/ WITH_EDITOR and WITH_EDITORONLY_DATA, use the old token check...
		//@todo. Is this something we need to support?
		if (TokenArray.Contains(TEXT("EDITOR")))
		{
			SetIsRunningAsEditor();
		}
#endif
		// Game, server and non-engine programs never run as the editor
	}

	// ...
}
```

### if WITH_EDITOR 和 #if WITH_EDITORONLY_DATA
在Unreal Engine中，`#if WITH_EDITOR` 和 `#if WITH_EDITORONLY_DATA` 是两个常用的预处理器宏，它们的用途和区别如下：

---

#### 1. `#if WITH_EDITOR`
- **作用范围**：  
  用于标记仅在编辑器环境下（如Unreal Editor中）需要编译或执行的代码，包括工具、编辑器功能、开发辅助逻辑等。
- **典型场景**：
  - 编辑器专用的模块或工具代码（如自定义编辑器按钮、细节面板扩展）。
  - 开发时的调试或可视化逻辑（如绘制调试线、编辑器预览）。
  - 依赖编辑器API的功能（如`GEditor`、`FEditorDelegates`）。
- **运行时行为**：  
  在非编辑器构建（如打包后的游戏）中，这些代码会被完全移除，不参与编译。

**示例**：
```cpp
#if WITH_EDITOR
    // 只有编辑器下才会执行的代码
    GEditor->GetEditorSubsystem<UAssetEditorSubsystem>()->OpenEditorForAsset(Asset);
#endif
```

---

#### 2. `#if WITH_EDITORONLY_DATA`
- **作用范围**：  
  用于标记仅在编辑器中需要的数据成员（如`UPROPERTY`），这些数据在打包后的游戏中不需要保留。
- **典型场景**：
  - 编辑器专用的资产元数据（如预览缩略图、开发注释）。
  - 临时或辅助性变量（如编辑器中的可视化配置参数）。
- **运行时行为**：  
  这些数据在非编辑器构建中会被完全剥离，不占用内存，也不会被序列化到打包后的资产中。

**示例**：
```cpp
UPROPERTY(EditAnywhere, Category="EditorOnly")
#if WITH_EDITORONLY_DATA
    UTexture2D* PreviewThumbnail; // 仅编辑器可见的缩略图
#endif
```

---

#### 关键区别
| 特性                | `WITH_EDITOR`                     | `WITH_EDITORONLY_DATA`            |
|---------------------|-----------------------------------|-----------------------------------|
| **用途**            | 编辑器逻辑代码                   | 编辑器专用数据成员               |
| **影响范围**        | 代码块                           | 变量、UPROPERTY等数据定义        |
| **打包后是否保留**  | 代码不编译                       | 数据不包含在资产中               |
| **常见位置**        | 函数、工具逻辑                   | 类的成员变量声明                 |

---

#### 何时使用？
- 如果需要编写**编辑器工具或功能**（如扩展编辑器按钮、自定义面板），用 `WITH_EDITOR`。
- 如果定义**仅在编辑器中使用的数据**（如临时变量、预览资源），用 `WITH_EDITORONLY_DATA`。

---

#### 注意事项
- 在 `WITH_EDITORONLY_DATA` 中引用的变量，如果在运行时被访问（如未用 `WITH_EDITOR` 包裹的代码），会导致崩溃。
- 两者可以结合使用，例如：
  ```cpp
  #if WITH_EDITOR
      void UpdateThumbnail() {
  #if WITH_EDITORONLY_DATA
          PreviewThumbnail = GenerateThumbnail(); // 安全访问编辑器数据
  #endif
      }
  #endif
  ```

通过合理使用这两个宏，可以确保编辑器专用代码和数据不会影响运行时性能或增加包体大小。

## ULyraGameData
这个数据是为了方便我们在任意位置获取到全局使用的统一资产.
此处直接给出代码.
这里放置的都是一些我们GAS使用的GameplayEffect的默认类的指针.
``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "Engine/DataAsset.h"

#include "LyraGameData.generated.h"

#define UE_API LYRAGAME_API

class UGameplayEffect;
class UObject;

/**
 * ULyraGameData
 *
 *	Non-mutable data asset that contains global game data.
 *	不可变的数据资产，其中包含全局游戏数据。
 */
UCLASS(MinimalAPI, BlueprintType, Const, Meta = (DisplayName = "Lyra Game Data", ShortTooltip = "Data asset containing global game data."))
class ULyraGameData : public UPrimaryDataAsset
{
	GENERATED_BODY()

public:

	UE_API ULyraGameData();

	// Returns the loaded game data.
	// 返回已加载的游戏数据。
	static UE_API const ULyraGameData& Get();

public:

	// Gameplay effect used to apply damage.  Uses SetByCaller for the damage magnitude.
	// 用于造成伤害的游戏效果。其伤害值由调用者设定。
	// 比如坠落超出高度
	UPROPERTY(EditDefaultsOnly, Category = "Default Gameplay Effects", meta = (DisplayName = "Damage Gameplay Effect (SetByCaller)"))
	TSoftClassPtr<UGameplayEffect> DamageGameplayEffect_SetByCaller;

	// Gameplay effect used to apply healing.  Uses SetByCaller for the healing magnitude.
	// 用于施加治疗效果的玩法特效。治疗量使用“由调用者设定”来确定。
	UPROPERTY(EditDefaultsOnly, Category = "Default Gameplay Effects", meta = (DisplayName = "Heal Gameplay Effect (SetByCaller)"))
	TSoftClassPtr<UGameplayEffect> HealGameplayEffect_SetByCaller;

	// Gameplay effect used to add and remove dynamic tags.
	// 用于添加和移除动态标签的游戏玩法效果。
	UPROPERTY(EditDefaultsOnly, Category = "Default Gameplay Effects")
	TSoftClassPtr<UGameplayEffect> DynamicTagGameplayEffect;
};

#undef UE_API


```


## LyraAssetManger
这个类主要是用于加载我们的游戏数据,如PawnData和GameData,你当然可以拥有其他数据.这个根据需要是做了缓存的, 避免重复加载.
其中GameData是嵌入到引擎的加载流程中的.
关于GameplayCue的预加载也是嵌入到引擎流程中的.
这节先不讲GameplayCue的管理.
你可以在这里根据重写的方法进行编辑器重载设置.
此处,我们直接给出代码


---

LyraAssetManager.h:
```cpp
// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "Engine/AssetManager.h"
#include "LyraAssetManagerStartupJob.h"
#include "Templates/SubclassOf.h"
#include "LyraAssetManager.generated.h"

#define UE_API LYRAGAME_API

class UPrimaryDataAsset;

class ULyraGameData;
class ULyraPawnData;

//一个约定的Bundles的命名
struct FLyraBundles
{
	static const FName Equipped;
};


/**
 * ULyraAssetManager
 *
 *	Game implementation of the asset manager that overrides functionality and stores game-specific types.
 *	It is expected that most games will want to override AssetManager as it provides a good place for game-specific loading logic.
 *	This class is used by setting 'AssetManagerClassName' in DefaultEngine.ini.
 *  资产管理器的实现，该实现会覆盖原有功能并存储游戏特定类型的数据。
 *  预计大多数游戏都会希望重写 AssetManager 类，因为它为游戏特定的加载逻辑提供了一个理想的位置。
 *  此类通过在 DefaultEngine.ini 中设置 'AssetManagerClassName' 来使用。
 * 
 */
UCLASS(MinimalAPI, Config = Game)
class ULyraAssetManager : public UAssetManager
{
	GENERATED_BODY()

public:

	//构造函数 初始化PawnData
	UE_API ULyraAssetManager();

	// Returns the AssetManager singleton object.
	// 返回资产管理器的单例对象。
	static UE_API ULyraAssetManager& Get();

	// TSoftObjectPtr is templatized wrapper of the generic FSoftObjectPtr, it can be used in UProperties
	// TSoftObjectPtr 是通用 FSoftObjectPtr 的模板化封装类，可用于 UProperties 中。

	// Returns the asset referenced by a TSoftObjectPtr.  This will synchronously load the asset if it's not already loaded.
	// 返回由 TSoftObjectPtr 引用的资产。如果该资产尚未加载，则会同步加载该资产。
	template<typename AssetType>
	static AssetType* GetAsset(const TSoftObjectPtr<AssetType>& AssetPointer, bool bKeepInMemory = true);

	// TSoftClassPtr is a templatized wrapper around FSoftObjectPtr that works like a TSubclassOf, it can be used in UProperties for blueprint subclasses
	// 返回由 TSoftClassPtr 指向的子类。如果该资产尚未加载，则会同步进行加载操作。

	// Returns the subclass referenced by a TSoftClassPtr.  This will synchronously load the asset if it's not already loaded.
	// 返回由 TSoftClassPtr 指向的子类。如果该资产尚未加载，则会同步进行加载操作。
	template<typename AssetType>
	static TSubclassOf<AssetType> GetSubclass(const TSoftClassPtr<AssetType>& AssetPointer, bool bKeepInMemory = true);

	// Logs all assets currently loaded and tracked by the asset manager.
	// 记录当前由资产管理器加载并跟踪的所有资产的信息。
	// 可以通过命令行调用
	static UE_API void DumpLoadedAssets();

	// 获取游戏数据
	UE_API const ULyraGameData& GetGameData();
	
	// 获取默认的玩家数据
	UE_API const ULyraPawnData* GetDefaultPawnData() const;

protected:
	// 获取或加载指定的游戏数据
	template <typename GameDataClass>
	const GameDataClass& GetOrLoadTypedGameData(const TSoftObjectPtr<GameDataClass>& DataPath)
	{
		// 如果已经缓存了直接获取
		if (TObjectPtr<UPrimaryDataAsset> const * pResult = GameDataMap.Find(GameDataClass::StaticClass()))
		{
			return *CastChecked<GameDataClass>(*pResult);
		}

		// Does a blocking load if needed
		// 如有需要则进行阻塞式加载
		return *CastChecked<const GameDataClass>(LoadGameDataOfClass(GameDataClass::StaticClass(),
			DataPath, GameDataClass::StaticClass()->GetFName()));
	}

	// 同步加载资产
	static UE_API UObject* SynchronousLoadAsset(const FSoftObjectPath& AssetPath);

	// 读取命令行参数,是否应当打印资产加载的日志
	static UE_API bool ShouldLogAssetLoads();

	// Thread safe way of adding a loaded asset to keep in memory.
	// 一种线程安全的添加已加载资源到内存中的方法。
	UE_API void AddLoadedAsset(const UObject* Asset);

	//~UAssetManager interface
	/** 开始初始加载，由“初始化对象引用”函数调用 */
	UE_API virtual void StartInitialLoading() override;
#if WITH_EDITOR
	/** Called right before PIE starts, will refresh asset directory and can be overriden to preload assets */
	/** 在 PIE 开始之前被调用，会刷新资源目录，并且可以在此处重写以实现预加载资源 */
	UE_API virtual void PreBeginPIE(bool bStartSimulate) override;
#endif
	//~End of UAssetManager interface
	
	/**
	 * FPrimaryAssetType
	 * A primary asset type, represented as an FName internally and implicitly convertible back and forth
	 * This exists so the blueprint API can understand it's not a normal FName
	 *
	 * 一种主要的资产类型，内部以 FName 格式表示，并且可以自动进行双向转换
	 * 此设置的存在是为了让蓝图 API 能够明白这不是一个普通的 FName 类型
	 * 
	 */
	UE_API UPrimaryDataAsset* LoadGameDataOfClass(TSubclassOf<UPrimaryDataAsset> DataClass, const TSoftObjectPtr<UPrimaryDataAsset>& DataClassPath, FPrimaryAssetType PrimaryAssetType);

protected:

	// Global game data asset to use.
	// 所需的全局游戏数据资源。
	// 这里是通过int配置.
	// 不好好配置会崩溃噢!
	UPROPERTY(Config)
	TSoftObjectPtr<ULyraGameData> LyraGameDataPath;

	// Loaded version of the game data
	// 已加载的游戏数据版本.
	UPROPERTY(Transient)
	TMap<TObjectPtr<UClass>, TObjectPtr<UPrimaryDataAsset>> GameDataMap;

	// Pawn data used when spawning player pawns if there isn't one set on the player state.
	// 当玩家状态中未设置相关数据时，用于生成玩家兵卒的兵卒数据。
	// 这里是通过int配置.
	// 不好好配置会崩溃噢!
	UPROPERTY(Config)
	TSoftObjectPtr<ULyraPawnData> DefaultPawnData;

private:
	// Flushes the StartupJobs array. Processes all startup work.
	// 清空“启动任务”数组。处理所有启动工作。
	UE_API void DoAllStartupJobs();

	// Sets up the ability system
	// 设置能力系统
	UE_API void InitializeGameplayCueManager();

	// Called periodically during loads, could be used to feed the status to a loading screen
	// 在加载过程中会定期调用此函数，可用于将状态信息传递给加载界面
	UE_API void UpdateInitialGameContentLoadPercent(float GameContentPercent);

	// The list of tasks to execute on startup. Used to track startup progress.
	// 启动时要执行的任务列表。用于跟踪启动过程的进度。
	TArray<FLyraAssetManagerStartupJob> StartupJobs;

private:
	
	// Assets loaded and tracked by the asset manager.
	// 资源已由资源管理器加载并进行跟踪。
	UPROPERTY()
	TSet<TObjectPtr<const UObject>> LoadedAssets;

	// Used for a scope lock when modifying the list of load assets.
	// 用于在修改加载资源列表时进行范围锁定。
	FCriticalSection LoadedAssetsCritical;
};

// 获取资产的模板实现
template<typename AssetType>
AssetType* ULyraAssetManager::GetAsset(const TSoftObjectPtr<AssetType>& AssetPointer, bool bKeepInMemory)
{
	AssetType* LoadedAsset = nullptr;

	const FSoftObjectPath& AssetPath = AssetPointer.ToSoftObjectPath();

	if (AssetPath.IsValid())
	{
		/**
		* 解除软指针的引用。
		* @返回值 若此对象已不存在或延迟指针为 null，则返回 nullptr；否则返回有效的 UObject 指针。
		* 
		 */
		LoadedAsset = AssetPointer.Get();
		if (!LoadedAsset)
		{
			LoadedAsset = Cast<AssetType>(SynchronousLoadAsset(AssetPath));
			ensureAlwaysMsgf(LoadedAsset, TEXT("Failed to load asset [%s]"), *AssetPointer.ToString());
		}

		if (LoadedAsset && bKeepInMemory)
		{
			// Added to loaded asset list.
			// 已添加至已加载资源列表。
			Get().AddLoadedAsset(Cast<UObject>(LoadedAsset));
		}
	}

	return LoadedAsset;
}

// 获取资产的类的模板实现
template<typename AssetType>
TSubclassOf<AssetType> ULyraAssetManager::GetSubclass(const TSoftClassPtr<AssetType>& AssetPointer, bool bKeepInMemory)
{
	TSubclassOf<AssetType> LoadedSubclass;

	const FSoftObjectPath& AssetPath = AssetPointer.ToSoftObjectPath();

	if (AssetPath.IsValid())
	{
		LoadedSubclass = AssetPointer.Get();
		if (!LoadedSubclass)
		{
			LoadedSubclass = Cast<UClass>(SynchronousLoadAsset(AssetPath));
			ensureAlwaysMsgf(LoadedSubclass, TEXT("Failed to load asset class [%s]"), *AssetPointer.ToString());
		}

		if (LoadedSubclass && bKeepInMemory)
		{
			// Added to loaded asset list.
			// 已添加至已加载资源列表。
			Get().AddLoadedAsset(Cast<UObject>(LoadedSubclass));
		}
	}

	return LoadedSubclass;
}

#undef UE_API

```



## LyraAssetManagerStartupJob

注意嵌入到引擎流程这个是通过定义宏一个启动任务去实现,相关代码部分在LyraAssetManagerStartupJob.h
LyraAssetManagerStartupJob.h:

``` cpp

// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "Engine/StreamableManager.h"

//单播代理
DECLARE_DELEGATE_OneParam(FLyraAssetManagerStartupJobSubstepProgress, float /*NewProgress*/);

/** Handles reporting progress from streamable handles */
/** 处理来自可流式句柄的进度报告 */
struct FLyraAssetManagerStartupJob
{
	//进度代理
	FLyraAssetManagerStartupJobSubstepProgress SubstepProgressDelegate;
	TFunction<void(const FLyraAssetManagerStartupJob&, TSharedPtr<FStreamableHandle>&)> JobFunc;
	// 任务名称
	FString JobName;
	// 任务权重
	float JobWeight;
	
	mutable double LastUpdate = 0;

	/** Simple job that is all synchronous */
	/** 简单的同步型任务 */
	// 任务名,任务函数,权重
	FLyraAssetManagerStartupJob(const FString& InJobName, const TFunction<void(const FLyraAssetManagerStartupJob&, TSharedPtr<FStreamableHandle>&)>& InJobFunc, float InJobWeight)
		: JobFunc(InJobFunc)
		, JobName(InJobName)
		, JobWeight(InJobWeight)
	{}

	/** Perform actual loading, will return a handle if it created one */
	/** 执行实际加载操作，如果创建了处理对象则会返回该处理对象的句柄 */
	TSharedPtr<FStreamableHandle> DoJob() const;

	// 更新进度 没有用到 因为资产管理器里面并没有注册复杂任务
	void UpdateSubstepProgress(float NewProgress) const
	{
		SubstepProgressDelegate.ExecuteIfBound(NewProgress);
	}

	// 根据流式加载句柄 没有用到 因为资产管理器里面并没有注册复杂任务
	void UpdateSubstepProgressFromStreamable(TSharedRef<FStreamableHandle> StreamableHandle) const
	{
		//先判断是否绑定了
		if (SubstepProgressDelegate.IsBound())
		{
			// StreamableHandle::GetProgress traverses() a large graph and is quite expensive
			// StreamableHandle::GetProgress 方法会遍历一个庞大的图结构，其执行效率较低。
			double Now = FPlatformTime::Seconds();
			// 比较时间 每16ms去获取下进度并传递出去
			if (LastUpdate - Now > 1.0 / 60)
			{
				SubstepProgressDelegate.Execute(StreamableHandle->GetProgress());
				LastUpdate = Now;
			}
		}
	}
};
```

---
## 修复关于AssetManger的引用
在上节课我们使用的是引擎自定义的AssetManger.
一共三处.第三处是关于一个Bundles的规则引用.
``` cpp
void ULyraExperienceManagerComponent::SetCurrentExperience(FPrimaryAssetId ExperienceId)
{
	//@XGTODO:这里使用的Lyra项目自定义的资产管理器
	//ULyraAssetManager& AssetManager = ULyraAssetManager::Get();
	UAssetManager& AssetManager = UAssetManager::Get();
}
void ULyraExperienceManagerComponent::StartExperienceLoad()
{
	//@XGTODO:这里使用的Lyra项目自定义的资产管理器
	//ULyraAssetManager& AssetManager = ULyraAssetManager::Get();
	UAssetManager& AssetManager = UAssetManager::Get();

	//@XGTODO:这里添加了资产管理类里面的一个全局变量,它是一个Bundle规则.
	//BundlesToLoad.Add(FLyraBundles::Equipped);
	//目前我们没有定义它,所以我们直接使用字符串即可,后面编写资产管理器的时候再改回来.
	BundlesToLoad.Add(TEXT("Equipped"));
}
```


## 总结
本节简单讲解了Lyra项目中关于资产管理器的定义.
其中的资产加载方法很重要,在项目其他地方都有调用.
本节的课程的难度非常大,但它属于框架类型的内容,只要搞定了,一般就不会怎么变动.
重要的是在这个学习过程中,要不断递进,与引擎的底层的流程关联起来,不能孤立式的学习!