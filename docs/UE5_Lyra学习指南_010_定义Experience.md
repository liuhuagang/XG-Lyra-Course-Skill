# UE5_Lyra学习指南_010_定义Experience

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)

文章内容由小刚撰写,采用了以下多种方式:

1.口述转文字
2.AI重构
3.参考引擎源码
4.Lyra工程源码
5.结合社区论坛各位大佬的解析

- [UE5\_Lyra学习指南\_010\_定义Experience](#ue5_lyra学习指南_010_定义experience)
	- [创建Experience的定义](#创建experience的定义)
	- [ULyraExperienceActionSet](#ulyraexperienceactionset)
	- [ULyraPawnData](#ulyrapawndata)
	- [指定WorldSettings](#指定worldsettings)
	- [修改EditorEngine](#修改editorengine)
	- [验证效果](#验证效果)
	- [总结](#总结)


## 创建Experience的定义
什么是Experience?
它的功能和GameMode很像.但是因为GameMode实在太重要了.所以单独针对每一个玩法进行体验设计.这个设计可以在不同Level层级进行复用.
此处直接给出代码即可.
``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.
// Finished.
#pragma once

#include "Engine/DataAsset.h"
#include "LyraExperienceDefinition.generated.h"

class UGameFeatureAction;
class ULyraPawnData;
class ULyraExperienceActionSet;

/**
 * Definition of an experience
 * 游戏体验的定义
 */
UCLASS(BlueprintType, Const)
class ULyraExperienceDefinition : public UPrimaryDataAsset
{
	GENERATED_BODY()

public:
	ULyraExperienceDefinition();

	//~UObject interface
#if WITH_EDITOR
	/**
	 * Generic function to validate objects during changelist validations, etc.
	 * 用于在变更列表验证等过程中验证对象的通用函数。
	 *
	 * @param	Context	the context holding validation warnings/errors.
	 *			上下文：包含验证警告/错误的信息区域。
	 * @return Valid if this object has data validation rules set up for it and the data for this object is valid. Returns Invalid if it does not pass 
	 *         the rules. Returns NotValidated if no rules are set for this object.
	 *         如果此对象已设置数据验证规则且该对象的数据有效，则返回有效；否则返回无效。规则。如果此对象未设置任何规则，则返回“NotValidated”。
	 */	
	virtual EDataValidationResult IsDataValid(class FDataValidationContext& Context) const override;
#endif
	//~End of UObject interface


	
	//~UPrimaryDataAsset interface
#if WITH_EDITORONLY_DATA
	/** This scans the class for AssetBundles metadata on asset properties and initializes the AssetBundleData with InitializeAssetBundlesFromMetadata */
	/** 此函数会遍历该类中的资产属性，查找资产包的元数据，并使用“InitializeAssetBundlesFromMetadata”方法来初始化 AssetBundleData 对象 */ 
	virtual void UpdateAssetBundleData() override;
#endif
	//~End of UPrimaryDataAsset interface

public:
	// List of Game Feature Plugins this experience wants to have active
	// 此体验所需的已激活的游戏功能插件列表
	UPROPERTY(EditDefaultsOnly, Category = Gameplay)
	TArray<FString> GameFeaturesToEnable;

	/** The default pawn class to spawn for players */
	/** 用于生成玩家默认兵种的兵种类 */
	//@TODO: Make soft?
	//@待办事项：做成软引用？
	UPROPERTY(EditDefaultsOnly, Category=Gameplay)
	TObjectPtr<const ULyraPawnData> DefaultPawnData;

	// List of actions to perform as this experience is loaded/activated/deactivated/unloaded
	// 当此体验被加载/激活/停用/卸载时要执行的操作列表
	/** Represents an action to be taken when a game feature is activated */
	/** 表示当某个游戏功能被激活时应采取的操作 */
	UPROPERTY(EditDefaultsOnly, Instanced, Category="Actions")
	TArray<TObjectPtr<UGameFeatureAction>> Actions;

	// List of additional action sets to compose into this experience
	// 用于构成此体验的附加操作集列表
	UPROPERTY(EditDefaultsOnly, Category=Gameplay)
	TArray<TObjectPtr<ULyraExperienceActionSet>> ActionSets;
};

```

注意ULyraExperienceDefinition引入了两个新的类分别是:
ULyraPawnData
ULyraExperienceActionSet

## ULyraExperienceActionSet
此处直接给出文件.
这个操作集很好理解.
就把不同体验的中可以复用的操作提取出来.
所以它的写法和ExperienceDefinition的写法基本一致.
``` cpp


// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "Engine/DataAsset.h"
#include "LyraExperienceActionSet.generated.h"

class UGameFeatureAction;

/**
 * Definition of a set of actions to perform as part of entering an experience
 * 作为进入某一体验过程中所需执行的一系列动作的定义
 */
UCLASS(BlueprintType, NotBlueprintable)
class ULyraExperienceActionSet : public UPrimaryDataAsset
{
	GENERATED_BODY()

public:
	ULyraExperienceActionSet();

	//~UObject interface
#if WITH_EDITOR
	virtual EDataValidationResult IsDataValid(class FDataValidationContext& Context) const override;
#endif
	//~End of UObject interface

	//~UPrimaryDataAsset interface
#if WITH_EDITORONLY_DATA
	virtual void UpdateAssetBundleData() override;
#endif
	//~End of UPrimaryDataAsset interface

public:
	// List of actions to perform as this experience is loaded/activated/deactivated/unloaded
	// 当此体验被加载/激活/停用/卸载时要执行的操作列表
	UPROPERTY(EditAnywhere, Instanced, Category="Actions to Perform")
	TArray<TObjectPtr<UGameFeatureAction>> Actions;

	// List of Game Feature Plugins this experience wants to have active
	// 用于构成此体验的附加操作集列表
	UPROPERTY(EditAnywhere, Category="Feature Dependencies")
	TArray<FString> GameFeaturesToEnable;
};

```

## ULyraPawnData
PawnData是用来描述一个角色具体的操作的类,GAS能力集,输入映射,相机模式的一个类.
我们需要将AbilitySets,TagRelationshipMapping,InputConfig,DefaultCameraMode这几行代码删掉或注释掉.
因为它们每一个都会去引入到另一套系统.目前还用不上.
``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "Engine/DataAsset.h"

#include "LyraPawnData.generated.h"

#define UE_API LYRAGAME_API

class APawn;
class ULyraAbilitySet;
class ULyraAbilityTagRelationshipMapping;
class ULyraCameraMode;
class ULyraInputConfig;
class UObject;


/**
 * ULyraPawnData
 *
 *	Non-mutable data asset that contains properties used to define a pawn.
 *	一种不可变的数据资产，其中包含用于定义棋子的属性。
 */
UCLASS(MinimalAPI, BlueprintType, Const, Meta = (DisplayName = "Lyra Pawn Data", ShortTooltip = "Data asset used to define a Pawn."))
class ULyraPawnData : public UPrimaryDataAsset
{
	GENERATED_BODY()

public:

	UE_API ULyraPawnData(const FObjectInitializer& ObjectInitializer);

public:

	// Class to instantiate for this pawn (should usually derive from ALyraPawn or ALyraCharacter).
	// 用于创建此兵种实例的类（通常应派生自 ALyraPawn 或 ALyraCharacter）。
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Pawn")
	TSubclassOf<APawn> PawnClass;

	// Ability sets to grant to this pawn's ability system.
	// 为该兵种赋予的能力组。
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Abilities")
	TArray<TObjectPtr<ULyraAbilitySet>> AbilitySets;

	// What mapping of ability tags to use for actions taking by this pawn
	// 此棋子执行行动时应采用何种能力标签的映射方式
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Abilities")
	TObjectPtr<ULyraAbilityTagRelationshipMapping> TagRelationshipMapping;

	// Input configuration used by player controlled pawns to create input mappings and bind input actions.
	// 玩家控制的兵种所使用的输入配置，用于创建输入映射并绑定输入操作。
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Input")
	TObjectPtr<ULyraInputConfig> InputConfig;

	// Default camera mode used by player controlled pawns.
	// 玩家控制的兵种所采用的默认摄像机模式。
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Lyra|Camera")
	TSubclassOf<ULyraCameraMode> DefaultCameraMode;
};

#undef UE_API

```
## 指定WorldSettings
当我们有了这个定义之后,我们需要将其嵌入到引擎流程中.
这个时候我们创建LyraWorldSettings.
此处直接贴出代码.
注意:
这个世界设置类还去检查了玩家起始点是否是Lyra项目自定义的起始点
并且可以在PIE模式下锁定网络启动情况ForceStandaloneNetMode,这个是通过编辑器引擎实现的.
``` cpp


// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "GameFramework/WorldSettings.h"
#include "LyraWorldSettings.generated.h"

#define UE_API LYRAGAME_API

class ULyraExperienceDefinition;

/**
 * The default world settings object, used primarily to set the default gameplay experience to use when playing on this map
 * 默认的世界设置对象，主要用于设定在该地图上游玩时所使用的默认游戏体验设置。
 */
UCLASS(MinimalAPI)
class ALyraWorldSettings : public AWorldSettings
{
	GENERATED_BODY()

public:

	UE_API ALyraWorldSettings(const FObjectInitializer& ObjectInitializer);

#if WITH_EDITOR
	/**
	 * Function that gets called from within Map_Check to allow this actor to check itself
	 * for any potential errors and register them with map check dialog.
	 *
	 * 该函数在“Map_Check”内部被调用，以便此角色能够检查自身是否存在任何潜在错误，并将这些错误信息通过“地图检查对话框”进行记录。
	 */
	UE_API virtual void CheckForErrors() override;
#endif

public:
	// Returns the default experience to use when a server opens this map if it is not overridden by the user-facing experience
	// 返回服务器在打开此地图时所使用的默认体验设置，若该设置未被用户界面中的体验所覆盖则使用此默认设置。
	UE_API FPrimaryAssetId GetDefaultGameplayExperience() const;

protected:
	// The default experience to use when a server opens this map if it is not overridden by the user-facing experience
	// 当服务器打开此地图时（若用户界面体验未对其进行覆盖），所采用的默认体验方式
	UPROPERTY(EditDefaultsOnly, Category=GameMode)
	TSoftClassPtr<ULyraExperienceDefinition> DefaultGameplayExperience;

public:

#if WITH_EDITORONLY_DATA
	// Is this level part of a front-end or other standalone experience?
	// When set, the net mode will be forced to Standalone when you hit Play in the editor
	// 这个级别是属于前端部分还是独立的独立体验的一部分呢？
	// 如果设置了此项，那么在编辑器中点击“播放”时，网络模式将强制切换为“独立模式”
	UPROPERTY(EditDefaultsOnly, Category=PIE)
	bool ForceStandaloneNetMode = false;
#endif
};

#undef UE_API

``` 
记得在ini文件中进行配置:
``` ini
[/Script/Engine.Engine]
WorldSettingsClassName=/Script/LyraGame.LyraWorldSettings
```

## 修改EditorEngine
LyraEditorEngine.cpp:

``` cpp
FGameInstancePIEResult ULyraEditorEngine::PreCreatePIEInstances(const bool bAnyBlueprintErrors, const bool bStartInSpectatorMode, const float PIEStartTime, const bool bSupportsOnlinePIE, int32& InNumOnlinePIEInstances)
{
	
	if (const ALyraWorldSettings* LyraWorldSettings = Cast<ALyraWorldSettings>(EditorWorld->GetWorldSettings()))
	{
		if (LyraWorldSettings->ForceStandaloneNetMode)
		{
			EPlayNetMode OutPlayNetMode;
			PlaySessionRequest->EditorPlaySettings->GetPlayNetMode(OutPlayNetMode);
			if (OutPlayNetMode != PIE_Standalone)
			{
				PlaySessionRequest->EditorPlaySettings->SetPlayNetMode(PIE_Standalone);

				FNotificationInfo Info(LOCTEXT("ForcingStandaloneForFrontend", "Forcing NetMode: Standalone for the Frontend"));
				Info.ExpireDuration = 2.0f;
				FSlateNotificationManager::Get().AddNotification(Info);
			}
		}
	}
	
	
	//@TODO: Should add delegates that a *non-editor* module could bind to for PIE start/stop instead of poking directly
	//@待办事项：应当添加一些委托机制，使得非编辑模块能够绑定这些委托来实现程序启动/停止的功能，而非直接进行操作。
	//GetDefault<ULyraDeveloperSettings>()->OnPlayInEditorStarted();
	//GetDefault<ULyraPlatformEmulationSettings>()->OnPlayInEditorStarted();



	FGameInstancePIEResult Result = Super::PreCreatePIEServerInstance(bAnyBlueprintErrors, bStartInSpectatorMode, PIEStartTime, bSupportsOnlinePIE, InNumOnlinePIEInstances);

	return Result;
}
```

## 验证效果
此时启动编辑器进行验证:
1.在ProjectSettings的中WorldSettings已发生更改.
2.可以强制PIE的关卡为单机模式.
3.可以在关卡上指定Experience.
4.可以创建PawnData,ExperienceDefinition,ExperienceActionSet.并互相指定
5.如果出现不合规要求,MessageLog会弹出提示.

## 总结
本节我们创建ExperenceDefinition这个特殊的蓝图资产,它很特殊,它是蓝色的.....
一般我们创建资产都是红色的.
ExperenceDefinition在很多地方都会用到.
本节只展示了通过EditorEngine来强制使用前端为单机模式.


