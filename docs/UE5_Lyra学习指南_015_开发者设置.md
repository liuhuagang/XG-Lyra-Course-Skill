# UE5_Lyra学习指南_015_开发者设置

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_015\_开发者设置](#ue5_lyra学习指南_015_开发者设置)
	- [概述](#概述)
	- [作弊](#作弊)
	- [LyraDeveloperSettings](#lyradevelopersettings)
	- [修复](#修复)
	- [代码](#代码)
	- [总结](#总结)



## 概述
本节主要讲述在编辑器开发的时候,我们希望便捷的重写一些游戏选型,比如机器人设置,无敌指令之类的.
## 作弊

``` cpp
/*
 * 作弊执行时机
 */
UENUM()
enum class ECheatExecutionTime
{
	// When the cheat manager is created
	// 当作弊管理器被创建时
	OnCheatManagerCreated,

	// When a pawn is possessed by a player
	// 当一个兵卒被玩家控制时
	OnPlayerPawnPossession
};

```

``` cpp
/*
 * 需要执行的作弊描述指令的结构体
 */
USTRUCT()
struct FLyraCheatToRun
{
	GENERATED_BODY()

	UPROPERTY(EditAnywhere)
	ECheatExecutionTime Phase = ECheatExecutionTime::OnPlayerPawnPossession;

	UPROPERTY(EditAnywhere)
	FString Cheat;
};



```
这里定义的作弊需要执行的时机与命令.  
它是通过ULyraCheatManager来调用的.  
这个开发者类主要用于存储设置指令.
这部分内容需要结合到GAS,相机等.在后续讲这个类的时候详细讲解!  

``` cpp
void ULyraCheatManager::InitCheatManager()
{
	Super::InitCheatManager();

#if WITH_EDITOR
	if (GIsEditor)
	{
		APlayerController* PC = GetOuterAPlayerController();
		for (const FLyraCheatToRun& CheatRow : GetDefault<ULyraDeveloperSettings>()->CheatsToRun)
		{
			if (CheatRow.Phase == ECheatExecutionTime::OnCheatManagerCreated)
			{
				PC->ConsoleCommand(CheatRow.Cheat, /*bWriteToLog=*/ true);
			}
		}
	}
#endif

	if (LyraCheat::bStartInGodMode)
	{
		God();	
	}
}

```

## LyraDeveloperSettings

这个类继承的父类很有意思,它不是UDeveloperSettings,而是UDeveloperSettings的子类UDeveloperSettingsBackedByCVars.


```cpp
/**
 * The base class of auto discovered settings object where some or all of the settings
 * are stored in console variables instead of config variables.
 * 自动发现设置对象的基类，其中部分或全部设置是以控制台变量的形式存储，而非配置变量的形式。
 * 
 */
UCLASS(Abstract, MinimalAPI)
class UDeveloperSettingsBackedByCVars : public UDeveloperSettings
{
	GENERATED_BODY()

public:
	DEVELOPERSETTINGS_API UDeveloperSettingsBackedByCVars(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	DEVELOPERSETTINGS_API virtual void PostInitProperties() override;
#if WITH_EDITOR
	DEVELOPERSETTINGS_API virtual void PostEditChangeProperty(struct FPropertyChangedEvent& PropertyChangedEvent) override;
#endif

};

```
``` cpp
void UDeveloperSettingsBackedByCVars::PostInitProperties()
{
	Super::PostInitProperties();

#if WITH_EDITOR
	if (IsTemplate())
	{
		ImportConsoleVariableValues();
	}
#endif
}

#if WITH_EDITOR
void UDeveloperSettingsBackedByCVars::PostEditChangeProperty(struct FPropertyChangedEvent& PropertyChangedEvent)
{
	Super::PostEditChangeProperty(PropertyChangedEvent);

	if (PropertyChangedEvent.Property)
	{
		ExportValuesToConsoleVariables(PropertyChangedEvent.Property);
	}
}
#endif

```
在这里,我们看到它在这个资产PIE编辑的情况下,进行了导入导出到命令行.
让我们再看看具体的这两个导入导出方法:  
``` cpp
#if WITH_EDITOR
FText UDeveloperSettings::GetSectionText() const
{
	return GetClass()->GetDisplayNameText();
}

FText UDeveloperSettings::GetSectionDescription() const
{
	return GetClass()->GetToolTipText();
}

void UDeveloperSettings::PostEditChangeProperty(struct FPropertyChangedEvent& PropertyChangedEvent)
{
	SettingsChangedDelegate.Broadcast(this, PropertyChangedEvent);
}

UDeveloperSettings::FOnSettingsChanged& UDeveloperSettings::OnSettingChanged()
{
	return SettingsChangedDelegate;
}
#endif

TSharedPtr<SWidget> UDeveloperSettings::GetCustomSettingsWidget() const
{
	return TSharedPtr<SWidget>();
}


#if WITH_EDITOR

static FName DeveloperSettingsConsoleVariableMetaFName(TEXT("ConsoleVariable"));

void UDeveloperSettings::ImportConsoleVariableValues()
{
	for (FProperty* Property = GetClass()->PropertyLink; Property; Property = Property->PropertyLinkNext)
	{
		if (!Property->HasAnyPropertyFlags(CPF_Config))
		{
			continue;
		}

		const FString& CVarName = Property->GetMetaData(DeveloperSettingsConsoleVariableMetaFName);
		if (!CVarName.IsEmpty())
		{
			IConsoleVariable* CVar = IConsoleManager::Get().FindConsoleVariable(*CVarName);
			if (CVar)
			{
				if (Property->ImportText_InContainer(*CVar->GetString(), this, this, PPF_ConsoleVariable) == NULL)
				{
					UE_LOG(LogTemp, Error, TEXT("%s import failed for %s on console variable %s (=%s)"), *GetClass()->GetName(), *Property->GetName(), *CVarName, *CVar->GetString());
				}
			}
			else
			{
				UE_LOG(LogTemp, Fatal, TEXT("%s failed to find console variable %s for %s"), *GetClass()->GetName(), *CVarName, *Property->GetName());
			}
		}
	}
}

void UDeveloperSettings::ExportValuesToConsoleVariables(FProperty* PropertyThatChanged)
{
	if(PropertyThatChanged)
	{
		const FString& CVarName = PropertyThatChanged->GetMetaData(DeveloperSettingsConsoleVariableMetaFName);
		if (!CVarName.IsEmpty())
		{
			IConsoleVariable* CVar = IConsoleManager::Get().FindConsoleVariable(*CVarName);
			if (CVar && (CVar->GetFlags() & ECVF_ReadOnly) == 0)
			{
				FByteProperty* ByteProperty = CastField<FByteProperty>(PropertyThatChanged);
				if (ByteProperty != NULL && ByteProperty->Enum != NULL)
				{
					CVar->Set(ByteProperty->GetPropertyValue_InContainer(this), ECVF_SetByProjectSetting);
				}
				else if (FEnumProperty* EnumProperty = CastField<FEnumProperty>(PropertyThatChanged))
				{
					FNumericProperty* UnderlyingProp = EnumProperty->GetUnderlyingProperty();
					void* PropertyAddress = EnumProperty->ContainerPtrToValuePtr<void>(this);
					CVar->Set((int32)UnderlyingProp->GetSignedIntPropertyValue(PropertyAddress), ECVF_SetByProjectSetting);
				}
				else if (FBoolProperty* BoolProperty = CastField<FBoolProperty>(PropertyThatChanged))
				{
					CVar->Set((int32)BoolProperty->GetPropertyValue_InContainer(this), ECVF_SetByProjectSetting);
				}
				else if (FIntProperty* IntProperty = CastField<FIntProperty>(PropertyThatChanged))
				{
					CVar->Set(IntProperty->GetPropertyValue_InContainer(this), ECVF_SetByProjectSetting);
				}
				else if (FFloatProperty* FloatProperty = CastField<FFloatProperty>(PropertyThatChanged))
				{
					CVar->Set(FloatProperty->GetPropertyValue_InContainer(this), ECVF_SetByProjectSetting);
				}
				else if (FStrProperty* StringProperty = CastField<FStrProperty>(PropertyThatChanged))
				{
					CVar->Set(*StringProperty->GetPropertyValue_InContainer(this), ECVF_SetByProjectSetting);
				}
				else if (FNameProperty* NameProperty = CastField<FNameProperty>(PropertyThatChanged))
				{
					CVar->Set(*NameProperty->GetPropertyValue_InContainer(this).ToString(), ECVF_SetByProjectSetting);
				}

			}
			else
			{
				// Reduce the amount of log spam for read-only properties. 
				// We assume that if property requires restart it is very likely it needs to stay read-only and therefore no need to log a warning.
				static const FName ConfigRestartRequiredKey = TEXT("ConfigRestartRequired");
				if (!PropertyThatChanged->GetBoolMetaData(ConfigRestartRequiredKey))
				{
					UE_LOG(LogInit, Warning, TEXT("CVar named '%s' marked up in %s was not found or is set to read-only"), *CVarName, *GetClass()->GetName());
				}
			}
		}
	}
}

#endif


```

注意观察这一行:
``` cpp
static FName DeveloperSettingsConsoleVariableMetaFName(TEXT("ConsoleVariable"));
```
以及这一行:
``` cpp
		if (!Property->HasAnyPropertyFlags(CPF_Config))
		{
			continue;
		}
```

与我们的项目设置这一行
``` cpp
	/**
	* Should force feedback effects be played, even if the last input device was not a gamepad?
	* The default behavior in Lyra is to only play force feedback if the most recent input device was a gamepad.
	*
	* 如果要播放力反馈效果的话，即便上一个输入设备并非游戏手柄呢？
	* 在莱拉中，默认设置是仅在最近的输入设备为游戏手柄的情况下才播放力反馈效果。
	* 
	*/
	UPROPERTY(config, EditAnywhere, Category = Lyra, meta = (ConsoleVariable = "LyraPC.ShouldAlwaysPlayForceFeedback"))
	bool bShouldAlwaysPlayForceFeedback = false;

```
这个config,以及meta=(ConsoleVariable.... )从而实现了命令行联动
同时注意//~UObject interface的这几个方法的调用时机即可!
``` cpp
	//~UObject interface
#if WITH_EDITOR
	/**
    * 当此对象上的某个属性被外部修改时触发此事件*
    * @参数 改变的属性  被修改的属性
    * 
	*/
	virtual void PostEditChangeProperty(FPropertyChangedEvent& PropertyChangedEvent) override;
	/**
     * 在对象重新加载其配置数据之后，从“ReloadConfig”函数中被调用。
	 */
	virtual void PostReloadConfig(FProperty* PropertyThatWasLoaded) override;
	/**
	 * 在 C++ 构造函数之后、属性（包括从配置文件加载的属性）初始化完成之后调用。
	 * 此函数在任何序列化操作或其他设置操作尚未进行之前被调用。
	 * 
	 */
	virtual void PostInitProperties() override;
#endif
	//~End of UObject interface
```

然后就是这个元数据还有个允许编辑的判定
``` cpp
	// 是否重写机器人数量
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, config, Category=LyraBots, meta=(InlineEditConditionToggle))
	bool bOverrideBotCount = false;

	// 重写的机器人数量
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, config, Category=LyraBots, meta=(EditCondition=bOverrideBotCount))
	int32 OverrideNumPlayerBotsToSpawn = 0;

```
meta=(AllowedTypes="LyraExperienceDefinition")
meta=(AllowedClasses="/Script/Engine.World")
等等.这里可以参照大钊老师整理的一整套元数据标签.在GitHub或者官方分享中有详细提及.不再赘述.

## 修复
之前我们在EditorEngine里有注释掉一段代码现在可以对其进行修复:  
``` cpp
FGameInstancePIEResult ULyraEditorEngine::PreCreatePIEInstances(const bool bAnyBlueprintErrors, const bool bStartInSpectatorMode, const float PIEStartTime, const bool bSupportsOnlinePIE, int32& InNumOnlinePIEInstances)
{

	//...

	//@TODO: Should add delegates that a *non-editor* module could bind to for PIE start/stop instead of poking directly
	// 用于重写体验
	GetDefault<ULyraDeveloperSettings>()->OnPlayInEditorStarted();
	// 用于模拟平台各种特征
	//GetDefault<ULyraPlatformEmulationSettings>()->OnPlayInEditorStarted();
	
	FGameInstancePIEResult Result = Super::PreCreatePIEServerInstance(bAnyBlueprintErrors, bStartInSpectatorMode, PIEStartTime, bSupportsOnlinePIE, InNumOnlinePIEInstances);

	return Result;
}

```

现在编辑器下编译时成功的,但是运行时会报错!  
``` txt
Error        LogWindows                appError called: Fatal error: [File:D:\build\++UE5\Sync\Engine\Source\Runtime\DeveloperSettings\Private\Engine\DeveloperSettings.cpp] [Line: 132] 
Error        LogWindows                LyraDeveloperSettings failed to find console variable LyraPC.ShouldAlwaysPlayForceFeedback for bShouldAlwaysPlayForceFeedback
Error        LogWindows                
Error        LogWindows                
Log          LogWindows                Windows GetLastError: 操作成功完成。 (0)
Error        LogWindows                appError called: Fatal error: [File:D:\build\++UE5\Sync\Engine\Source\Runtime\DeveloperSettings\Private\Engine\DeveloperSettings.cpp] [Line: 132] 
Error        LogWindows                LyraDeveloperSettings failed to find console variable GameplayMessageSubsystem.LogMessages for LogGameplayMessages
Error        LogWindows                

```
这是因为我们还没有注册这个命令行变量.  

在LyraPlayerController.cpp的上部添加如下代码.  
注意,此处可能需要刷新项目,如果添加后仍不行,可以先尝试把UPROPERTY()给注释掉然后成功启动编辑器.然后关闭编辑器,在代码添加后编译再运行.
```cpp

namespace Lyra
{
	namespace Input
	{
		static int32 ShouldAlwaysPlayForceFeedback = 0;
		static FAutoConsoleVariableRef CVarShouldAlwaysPlayForceFeedback(TEXT("LyraPC.ShouldAlwaysPlayForceFeedback"),
			ShouldAlwaysPlayForceFeedback,
			TEXT("Should force feedback effects be played, even if the last input device was not a gamepad?"));
	}
}

```

此时仍然报错
``` txt
Error        LogWindows                appError called: Fatal error: [File:D:\build\++UE5\Sync\Engine\Source\Runtime\DeveloperSettings\Private\Engine\DeveloperSettings.cpp] [Line: 132] 
Error        LogWindows                LyraDeveloperSettings failed to find console variable GameplayMessageSubsystem.LogMessages for LogGameplayMessages
```
但是这个命令行变量已经注册了!
GameplayMessageSubsystem.cpp:
``` cpp
namespace UE
{
	namespace GameplayMessageSubsystem
	{
		static int32 ShouldLogMessages = 0;
		static FAutoConsoleVariableRef CVarShouldLogMessages(TEXT("GameplayMessageSubsystem.LogMessages"),
			ShouldLogMessages,
			TEXT("Should messages broadcast through the gameplay message subsystem be logged?"));
	}
}


```
我们只需要先临时这样处理即可.在外面初始化设置时,要确保这个模块已经加载进来了!这类错误是很经典的.
``` cpp
void ULyraDeveloperSettings::PostInitProperties()
{
	FModuleManager::Get().LoadModuleChecked(TEXT("GameplayMessageRuntime"));
	
	Super::PostInitProperties();

	ApplySettings();
}
```



## 代码
ULyraDeveloperSettings:  
``` cpp

/**
 * Developer settings / editor cheats
 * 开发者设置 / 编辑器作弊功能
 */
UCLASS(config=EditorPerProjectUserSettings, MinimalAPI)
class ULyraDeveloperSettings : public UDeveloperSettingsBackedByCVars
{
	GENERATED_BODY()

public:
	ULyraDeveloperSettings();

	//~UDeveloperSettings interface
	// 标识类别
	virtual FName GetCategoryName() const override;
	//~End of UDeveloperSettings interface

public:
	// The experience override to use for Play in Editor (if not set, the default for the world settings of the open map will be used)
	// 在编辑器中用于游戏运行的体验设置（若未进行设置，则将使用当前打开地图的世界设置的默认值）
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, config, Category=Lyra, meta=(AllowedTypes="LyraExperienceDefinition"))
	FPrimaryAssetId ExperienceOverride;

	// 是否重写机器人数量
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, config, Category=LyraBots, meta=(InlineEditConditionToggle))
	bool bOverrideBotCount = false;

	// 重写的机器人数量
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, config, Category=LyraBots, meta=(EditCondition=bOverrideBotCount))
	int32 OverrideNumPlayerBotsToSpawn = 0;

	// 是否允许机器人攻击
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, config, Category=LyraBots)
	bool bAllowPlayerBotsToAttack = true;

	// Do the full game flow when playing in the editor, or skip 'waiting for player' / etc... game phases?
	// 在编辑器中进行游戏时，是否要完整执行游戏流程，还是可以跳过“等待玩家”/等等等游戏阶段？
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, config, Category=Lyra)
	bool bTestFullGameFlowInPIE = false;

	/**
	* Should force feedback effects be played, even if the last input device was not a gamepad?
	* The default behavior in Lyra is to only play force feedback if the most recent input device was a gamepad.
	*
	* 如果要播放力反馈效果的话，即便上一个输入设备并非游戏手柄呢？
	* 在莱拉中，默认设置是仅在最近的输入设备为游戏手柄的情况下才播放力反馈效果。
	* 
	*/
	UPROPERTY(config, EditAnywhere, Category = Lyra, meta = (ConsoleVariable = "LyraPC.ShouldAlwaysPlayForceFeedback"))
	bool bShouldAlwaysPlayForceFeedback = false;

	// Should game logic load cosmetic backgrounds in the editor or skip them for iteration speed?
	// 在编辑器中，游戏逻辑应加载外观装饰背景还是跳过这部分以加快迭代速度？
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, config, Category=Lyra)
	bool bSkipLoadingCosmeticBackgroundsInPIE = false;

	// List of cheats to auto-run during 'play in editor'
	// 在“编辑器中游玩”模式下自动运行的作弊选项列表
	UPROPERTY(config, EditAnywhere, Category=Lyra)
	TArray<FLyraCheatToRun> CheatsToRun;
	
	// Should messages broadcast through the gameplay message subsystem be logged?
	// 在游戏玩法消息子系统中广播的消息是否需要进行记录？
	UPROPERTY(config, EditAnywhere, Category=GameplayMessages, meta=(ConsoleVariable="GameplayMessageSubsystem.LogMessages"))
	bool LogGameplayMessages = false;

#if WITH_EDITORONLY_DATA
	
	/** A list of common maps that will be accessible via the editor detoolbar */
	/** 一系列常见的地图列表，可通过编辑器工具栏进行访问 */
	UPROPERTY(config, EditAnywhere, BlueprintReadOnly, Category=Maps, meta=(AllowedClasses="/Script/Engine.World"))
	TArray<FSoftObjectPath> CommonEditorMaps;
#endif
	
#if WITH_EDITOR
public:
	// Called by the editor engine to let us pop reminder notifications when cheats are active
	// 由编辑引擎调用，以便在作弊功能激活时向我们发送提醒通知
	LYRAGAME_API void OnPlayInEditorStarted() const;

private:
	// 应用设置 目前没有使用 因为这里的设置都是外部读取 不需要自己去主动更新
	void ApplySettings();
#endif

public:
	//~UObject interface
#if WITH_EDITOR
	/**
    * 当此对象上的某个属性被外部修改时触发此事件*
    * @参数 改变的属性  被修改的属性
    * 
	*/
	virtual void PostEditChangeProperty(FPropertyChangedEvent& PropertyChangedEvent) override;
	/**
     * 在对象重新加载其配置数据之后，从“ReloadConfig”函数中被调用。
	 */
	virtual void PostReloadConfig(FProperty* PropertyThatWasLoaded) override;
	/**
	 * 在 C++ 构造函数之后、属性（包括从配置文件加载的属性）初始化完成之后调用。
	 * 此函数在任何序列化操作或其他设置操作尚未进行之前被调用。
	 * 
	 */
	virtual void PostInitProperties() override;
#endif
	//~End of UObject interface
};


```

## 总结
本节内容比较轻松,主要理解这个开发者设置就是方便我们开发使用的.值得注意的是它是如何与引擎进行交互,比如命令行读取修改变量,一些元数据标签的处理.  
这里面很多选型都是其他类来读取使用的,而非我们主动去调用.  
比如:作弊管理器,编辑器下的快捷地图栏访问,机器人的AI行为树等等.  
在下一节中,我们将讲解平台模拟设置,这个就需要设置变动后主动去修改引擎相关设置,就相对麻烦一些.但是本质不变.就是为了方便我们开发,避免硬编码,快速调试等操作!  
