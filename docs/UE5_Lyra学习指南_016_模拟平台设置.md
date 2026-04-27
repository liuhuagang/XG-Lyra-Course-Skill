# UE5_Lyra学习指南_016_模拟平台设置

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_016\_模拟平台设置](#ue5_lyra学习指南_016_模拟平台设置)
	- [概述](#概述)
	- [LyraPlatformEmulationSettings](#lyraplatformemulationsettings)
		- [平台特征](#平台特征)
		- [平台名称](#平台名称)
		- [平台配置文件](#平台配置文件)
		- [触发时机](#触发时机)
	- [修复](#修复)
	- [代码](#代码)
	- [总结](#总结)



## 概述
这个类主要用于将标识各类平台的tag和配置名称进行选择传递给引擎.



## LyraPlatformEmulationSettings
这个类的父类是DeveloperSettingsBackedByCVars,那么同样的它也有需要去动态读取命令行的参数.  
``` cpp
	// Do we apply desktop-style frame rate settings in PIE?
	// (frame rate limits are an engine-wide setting so it's not always desirable to have enabled in the editor)
	// You may also want to disable the editor preference "Use Less CPU when in Background" if testing background frame rate limits
	// 在 PIE 中，我们是否采用桌面式的帧率设置？
	// （帧率限制是引擎级别的设置，所以在编辑器中并非总是需要将其启用）
	// 如果要测试后台的帧率限制，您可能还需要禁用编辑器的偏好设置“在后台使用较少的 CPU”
	UPROPERTY(EditAnywhere, config, Category=PlatformEmulation, meta=(ConsoleVariable="Lyra.Settings.ApplyFrameRateSettingsInPIE"))
	bool bApplyFrameRateSettingsInPIE = false;
```



### 平台特征

``` cpp
	// 添加的平台特征开启
	UPROPERTY(EditAnywhere, config, Category=PlatformEmulation, meta=(Categories="Input,Platform.Trait"))
	FGameplayTagContainer AdditionalPlatformTraitsToEnable;

	// 添加的平台特征压制
	UPROPERTY(EditAnywhere, config, Category=PlatformEmulation, meta=(Categories="Input,Platform.Trait"))
	FGameplayTagContainer AdditionalPlatformTraitsToSuppress;
```
在这一步注意观察Project Settings-Plugins-CommonUIFrameWork-Visibility-Platform.
``` cpp
void ULyraPlatformEmulationSettings::ApplySettings()
{
	// 是遏制CommonUI对象调试Tag
	UCommonUIVisibilitySubsystem::SetDebugVisibilityConditions(AdditionalPlatformTraitsToEnable, AdditionalPlatformTraitsToSuppress);

	if (GIsEditor && PretendPlatform != LastAppliedPretendPlatform)
	{
		ChangeActivePretendPlatform(PretendPlatform);
	}

	PickReasonableBaseDeviceProfile();
}
```
### 平台名称
``` cpp
	// 假装的平台名称
	UPROPERTY(EditAnywhere, config, Category=PlatformEmulation, meta=(GetOptions=GetKnownPlatformIds))
	FName PretendPlatform;
```
None  
Andriod  
IOS  
Linux  
Mac  
TVDS  
Windows  

通过下面这个函数修改平台
``` cpp
	// 设置编辑器模拟的平台
	UPlatformSettingsManager::SetEditorSimulatedPlatform(PretendPlatform);
```

### 平台配置文件
``` cpp
	// The base device profile to pretend we are using when emulating device-specific device profiles applied from ULyraSettingsLocal
	// 在模拟来自“ULyraSettingsLocal”中的设备特定设备配置文件时所使用的基础设备配置文件。
	UPROPERTY(EditAnywhere, config, Category=PlatformEmulation, meta=(GetOptions=GetKnownDeviceProfiles, EditCondition=bApplyDeviceProfilesInPIE))
	FName PretendBaseDeviceProfile;
```
``` txt
 iPadAir3, 
 iPadAir4, 
 iPadAir5, 
 iPadMini4, 
 iPadMini5, 
 iPadMini6, 
 iPhone6S, 
 iPhone7, 
 iPodTouch7, 
 iPhone6SPlus, 
 iPhone7Plus, 
 iPhoneSE, 
 iPhone8, 
 iPhone8Plus, 
 iPhoneX, 
 iPhoneXS, 
 iPhoneXSMax, 
 iPhoneXR, 
 iPhone11, 
 iPhone11Pro, 
 iPhone11ProMax, 
 iPhoneSE2, 
 iPhone12Mini, 
 iPhone12, 
 iPhone12Pro, 
 iPhone12ProMax, 
 iPhone13Mini, 
 iPhone13, 
 iPhone13Pro, 
 iPhone13ProMax, 
 iPhoneSE3, 
 iPhone14, 
 iPhone14Plus, 
 iPhone14Pro, 
 iPhone14ProMax, 
 iPadPro105, 
 iPadPro129, 
 iPadPro97, 
 iPadPro2_129, 
 iPad5, 
 iPad6, 
 iPad7, 
 iPad8, 
 iPad9, 
 iPad10, 
 iPadPro11, 
 iPadPro2_11, 
 iPadPro3_11, 
 iPadPro4_11, 
 iPadPro3_129, 
 iPadPro4_129, 
 iPadPro5_129, 
 iPadPro6_129, 
 AppleTV, 
 AppleTV4K, 
 AppleTV2_4K, 
 TVOS, 
 Mac, 
 MacEditor, 
 MacClient, 
 MacServer, 
 Linux, 
 LinuxEditor, 
 LinuxArm64Editor, 
 LinuxArm64, 
 LinuxClient, 
 LinuxArm64Client, 
 LinuxServer, 
 LinuxArm64Server, 
 Android, 
 Android_Preview_OpenGL, 
 Android_Preview_Vulkan, 
 Android_Low, 
 Android_Mid, 
 Android_High, 
 Android_Default, 
 Android_Adreno4xx, 
 Android_Adreno5xx_Low, 
 Android_Adreno5xx, 
 Android_Adreno6xx, 
 Android_Adreno6xx_Vulkan, 
 Android_Adreno7xx, 
 Android_Adreno7xx_Vulkan, 
 Android_Mali_T6xx, 
 Android_Mali_T7xx, 
 Android_Mali_T8xx, 
 Android_Mali_G71, 
 Android_Mali_G72, 
 Android_Mali_G72_Vulkan, 
 Android_Mali_G76, 
 Android_Mali_G76_Vulkan, 
 Android_Mali_G77, 
 Android_Mali_G77_Vulkan, 
 Android_Mali_G78, 
 Android_Mali_G78_Vulkan, 
 Android_Mali_G710, 
 Android_Mali_G710_Vulkan, 
 Android_Xclipse_920, 
 Android_Xclipse_920_Vulkan, 
 Android_Vulkan_SM5, 
 Android_PowerVR_G6xxx, 
 Android_PowerVR_GT7xxx, 
 Android_PowerVR_GE8xxx, 
 Android_PowerVR_GM9xxx, 
 Android_PowerVR_GM9xxx_Vulkan, 
 Android_TegraK1, 
 Android_Unknown_Vulkan, 
 Oculus_Quest, 
 Oculus_Quest2, 
 HoloLens, 
 PS4, 
 PS4_Neo, 
 PS4_Neo_4k, 
 Switch, 
 SwitchConsole, 
 SwitchHandheld, 
 Switch_Console_Forward, 
 Switch_Handheld_Forward, 
 Switch_Console_Deferred, 
 Switch_Handheld_Deferred, 
 WinGDK, 
 WinGDKClient, 
 XSX, 
 XSX_Lockhart_Base, 
 XSX_Lockhart, 
 XSX_Anaconda_Base, 
 XSX_Anaconda, 
 PS5_Base, 
 PS5, 
 XB1, 
 XB1_S, 
 XB1_Scorpio, 
 XboxOneGDK, 
 XboxOneGDK_S, 
 XboxOneGDK_Scorpio, 
``` 


### 触发时机
每次在编辑器修改设置后,对应的这个类就会触发UObject的接口
``` cpp
void ULyraPlatformEmulationSettings::PostEditChangeProperty(FPropertyChangedEvent& PropertyChangedEvent)
{
	Super::PostEditChangeProperty(PropertyChangedEvent);

	ApplySettings();
}

void ULyraPlatformEmulationSettings::PostReloadConfig(FProperty* PropertyThatWasLoaded)
{
	Super::PostReloadConfig(PropertyThatWasLoaded);

	ApplySettings();
}

void ULyraPlatformEmulationSettings::PostInitProperties()
{
	Super::PostInitProperties();

	ApplySettings();
}
```
在ApplySetting里面就去修改对应的平台,顺带把平台特征传递给CommonUI.


``` cpp
void ULyraPlatformEmulationSettings::ApplySettings()
{
	// 是遏制CommonUI对象调试Tag
	UCommonUIVisibilitySubsystem::SetDebugVisibilityConditions(AdditionalPlatformTraitsToEnable, AdditionalPlatformTraitsToSuppress);

	if (GIsEditor && PretendPlatform != LastAppliedPretendPlatform)
	{
		ChangeActivePretendPlatform(PretendPlatform);
	}

	PickReasonableBaseDeviceProfile();
}
```

## 修复
之前我们在EditorEngine里有注释掉一段代码现在可以对其进行修复:  
``` cpp
FGameInstancePIEResult ULyraEditorEngine::PreCreatePIEInstances(const bool bAnyBlueprintErrors, const bool bStartInSpectatorMode, const float PIEStartTime, const bool bSupportsOnlinePIE, int32& InNumOnlinePIEInstances)
{
	// ...
	//@TODO: Should add delegates that a *non-editor* module could bind to for PIE start/stop instead of poking directly
	// 用于重写体验
	GetDefault<ULyraDeveloperSettings>()->OnPlayInEditorStarted();
	// 用于模拟平台各种特征
	GetDefault<ULyraPlatformEmulationSettings>()->OnPlayInEditorStarted();
	
	FGameInstancePIEResult Result = Super::PreCreatePIEServerInstance(bAnyBlueprintErrors, bStartInSpectatorMode, PIEStartTime, bSupportsOnlinePIE, InNumOnlinePIEInstances);

	return Result;
}

```

接下来,我们继续修复命令行变量定义的问题:  

``` cpp
#if WITH_EDITOR
static TAutoConsoleVariable<bool> CVarApplyFrameRateSettingsInPIE(TEXT("Lyra.Settings.ApplyFrameRateSettingsInPIE"),
	false,
	TEXT("Should we apply frame rate settings in PIE?"),
	ECVF_Default);

static TAutoConsoleVariable<bool> CVarApplyFrontEndPerformanceOptionsInPIE(TEXT("Lyra.Settings.ApplyFrontEndPerformanceOptionsInPIE"),
	false,
	TEXT("Do we apply front-end specific performance options in PIE?"),
	ECVF_Default);

static TAutoConsoleVariable<bool> CVarApplyDeviceProfilesInPIE(TEXT("Lyra.Settings.ApplyDeviceProfilesInPIE"),
	false,
	TEXT("Should we apply experience/platform emulated device profiles in PIE?"),
	ECVF_Default);
#endif

```

## 代码
ULyraPlatformEmulationSettings:  

``` cpp

/**
 * Platform emulation settings
 * 平台模拟设置
 */
UCLASS(config=EditorPerProjectUserSettings, MinimalAPI)
class ULyraPlatformEmulationSettings : public UDeveloperSettingsBackedByCVars
{
	GENERATED_BODY()

public:
	ULyraPlatformEmulationSettings();

	//~UDeveloperSettings interface
	virtual FName GetCategoryName() const override;
	//~End of UDeveloperSettings interface

	// 获取假装的平台配置文件
	FName GetPretendBaseDeviceProfile() const;

	// 获取假装的平台名称
	FName GetPretendPlatformName() const;

private:
	// 添加的平台特征开启
	UPROPERTY(EditAnywhere, config, Category=PlatformEmulation, meta=(Categories="Input,Platform.Trait"))
	FGameplayTagContainer AdditionalPlatformTraitsToEnable;

	// 添加的平台特征压制
	UPROPERTY(EditAnywhere, config, Category=PlatformEmulation, meta=(Categories="Input,Platform.Trait"))
	FGameplayTagContainer AdditionalPlatformTraitsToSuppress;

	// 假装的平台名称
	UPROPERTY(EditAnywhere, config, Category=PlatformEmulation, meta=(GetOptions=GetKnownPlatformIds))
	FName PretendPlatform;

	// The base device profile to pretend we are using when emulating device-specific device profiles applied from ULyraSettingsLocal
	// 在模拟来自“ULyraSettingsLocal”中的设备特定设备配置文件时所使用的基础设备配置文件。
	UPROPERTY(EditAnywhere, config, Category=PlatformEmulation, meta=(GetOptions=GetKnownDeviceProfiles, EditCondition=bApplyDeviceProfilesInPIE))
	FName PretendBaseDeviceProfile;

	// Do we apply desktop-style frame rate settings in PIE?
	// (frame rate limits are an engine-wide setting so it's not always desirable to have enabled in the editor)
	// You may also want to disable the editor preference "Use Less CPU when in Background" if testing background frame rate limits
	// 在 PIE 中，我们是否采用桌面式的帧率设置？
	// （帧率限制是引擎级别的设置，所以在编辑器中并非总是需要将其启用）
	// 如果要测试后台的帧率限制，您可能还需要禁用编辑器的偏好设置“在后台使用较少的 CPU”
	UPROPERTY(EditAnywhere, config, Category=PlatformEmulation, meta=(ConsoleVariable="Lyra.Settings.ApplyFrameRateSettingsInPIE"))
	bool bApplyFrameRateSettingsInPIE = false;

	// Do we apply front-end specific performance options in PIE?
	// Most engine performance/scalability settings they drive are global, so if one PIE window
	// is in the front-end and the other is in-game one will win and the other gets stuck with those settings
	// 在 PIE 中，我们是否应用前端特定的性能选项？
	// 大多数引擎的性能/可扩展性设置都是全局性的，所以如果一个 PIE 窗口在前端运行，而另一个在游戏内运行，那么前者会占优，而后者则会一直受这些设置的限制。
	UPROPERTY(EditAnywhere, config, Category=PlatformEmulation, meta=(ConsoleVariable="Lyra.Settings.ApplyFrontEndPerformanceOptionsInPIE"))
	bool bApplyFrontEndPerformanceOptionsInPIE = false;

	// Should we apply experience/platform emulated device profiles in PIE?
	// 我们是否应在 PIE 中应用经验/平台模拟设备配置文件？
	UPROPERTY(EditAnywhere, config, Category=PlatformEmulation, meta=(InlineEditConditionToggle, ConsoleVariable="Lyra.Settings.ApplyDeviceProfilesInPIE"))
	bool bApplyDeviceProfilesInPIE = false;

#if WITH_EDITOR
public:
	// Called by the editor engine to let us pop reminder notifications when cheats are active
	// 由编辑引擎调用，以便在作弊功能激活时向我们发送提醒通知
	LYRAGAME_API void OnPlayInEditorStarted() const;

private:
	// The last pretend platform we applied
	// 我们所使用的最后一个模拟平台
	FName LastAppliedPretendPlatform;

private:
	// 应用设置 将Tag传递给CommonUI系统
	void ApplySettings();
	// 更改激活的平台名称
	void ChangeActivePretendPlatform(FName NewPlatformName);
#endif

public:
	//~UObject interface
#if WITH_EDITOR
	virtual void PostEditChangeProperty(FPropertyChangedEvent& PropertyChangedEvent) override;
	virtual void PostReloadConfig(FProperty* PropertyThatWasLoaded) override;
	virtual void PostInitProperties() override;
#endif
	//~End of UObject interface

private:
	// 获取已知平台的名称
	UFUNCTION()
	TArray<FName> GetKnownPlatformIds() const;

	// 获取已知的配置文件
	UFUNCTION()
	TArray<FName> GetKnownDeviceProfiles() const;

	// 找到合理的基础配置文件
	void PickReasonableBaseDeviceProfile();
};

```

通过下面这个函数来找到对应的配置文件
``` cpp
	// 找到合理的基础配置文件
	void PickReasonableBaseDeviceProfile();
```
## 总结
本节讲了关于平台特征,平台名称,平台配置文件如何在编辑器进行覆写调用.  
