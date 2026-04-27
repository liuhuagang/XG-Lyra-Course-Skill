# UE5_Lyra学习指南_030_平台渲染设置

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_030\_平台渲染设置](#ue5_lyra学习指南_030_平台渲染设置)
	- [概述](#概述)
		- [音频设置LyraAudioSettings](#音频设置lyraaudiosettings)
		- [音频混合子系统LyraAudioMixEffectsSubsystem](#音频混合子系统lyraaudiomixeffectssubsystem)
			- [加载并持有对象防止GC](#加载并持有对象防止gc)
			- [加载界面覆写音频控制总线混合](#加载界面覆写音频控制总线混合)
	- [调试设置](#调试设置)
	- [代码](#代码)
		- [LyraAudioSettings.h](#lyraaudiosettingsh)
		- [LyraAudioMixEffectsSubsystem.h](#lyraaudiomixeffectssubsystemh)
	- [总结](#总结)



## 概述
本节主要介绍了项目对于音频控制总线混合的处理.
### 音频设置LyraAudioSettings

通过一个开发者设置保存软引用来指定对应的控制总线和控制总线的混合.
控制总线的混合主要有

``` txt
/** 默认基础控制总线混合模式 */
DefaultControlBusMix
/** 加载屏幕控制总线组合 - 在加载屏幕期间调用，用于覆盖背景音频事件 */
LoadingScreenControlBusMix
/** 默认基础控制总线混合模式 */
UserSettingsControlBusMix
```
控制总线主要有:
``` txt
/** 分配给整体音量调节的控制总线 */
OverallVolumeControlBus;

/** 分配给音乐音量设置的控制总线 */
FSoftObjectPath MusicVolumeControlBus;

/** 分配给音效声音音量设置的控制总线 */
FSoftObjectPath SoundFXVolumeControlBus;

/** 分配给对话音量设置的控制总线 */
DialogueVolumeControlBus;

/** 分配给语音聊天音量设置的控制总线 */
VoiceChatVolumeControlBus;

```
以及一些关于高动态和低动态的音频输出
``` txt
/** 亚混音处理链用于实现高动态范围音频输出 */
HDRAudioSubmixEffectChain
/** 亚混音处理链用于实现低动态范围音频输出 */
LDRAudioSubmixEffectChain
```


``` cpp
UCLASS(MinimalAPI, config = Game, defaultconfig, meta = (DisplayName = "LyraAudioSettings"))
class ULyraAudioSettings : public UDeveloperSettings
{
	GENERATED_BODY()

public:
	/** The Default Base Control Bus Mix */
	/** 默认基础控制总线混合模式 */
	UPROPERTY(config, EditAnywhere, Category = MixSettings, meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBusMix"))
	FSoftObjectPath DefaultControlBusMix;
	// ...
	/** Control Bus assigned to the Overall sound volume setting */
	/** 分配给整体音量调节的控制总线 */
	UPROPERTY(config, EditAnywhere, Category = UserMixSettings, meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBus"))
	FSoftObjectPath OverallVolumeControlBus;
}


```


### 音频混合子系统LyraAudioMixEffectsSubsystem
这个子系统主要是把我们的音频设置在世界上进行应用.
它的级别是世界子系统.跟随世界的生命周期.

#### 加载并持有对象防止GC
``` cpp
class ULyraAudioMixEffectsSubsystem : public UWorldSubsystem
{
	GENERATED_BODY()

	// ...
	// Default Sound Control Bus Mix retrieved from the Lyra Audio Settings
	// 从Lyra音频设置中获取的默认声音控制总线混音设置
	UPROPERTY(Transient)
	TObjectPtr<USoundControlBusMix> DefaultBaseMix = nullptr;

	// Loading Screen Sound Control Bus Mix retrieved from the Lyra Audio Settings
	// 从Lyra音频设置中获取加载屏幕声音控制总线混音设置
	UPROPERTY(Transient)
	TObjectPtr<USoundControlBusMix> LoadingScreenMix = nullptr;

	// User Sound Control Bus Mix retrieved from the Lyra Audio Settings
	// 从Lyra音频设置中获取的用户声音控制总线混合模式
	UPROPERTY(Transient)
	TObjectPtr<USoundControlBusMix> UserMix = nullptr;
	// ...

}
```


``` cpp
void ULyraAudioMixEffectsSubsystem::PostInitialize()
{
	if (const ULyraAudioSettings* LyraAudioSettings = GetDefault<ULyraAudioSettings>())
	{
		if (UObject* ObjPath = LyraAudioSettings->DefaultControlBusMix.TryLoad())
		{
			if (USoundControlBusMix* SoundControlBusMix = Cast<USoundControlBusMix>(ObjPath))
			{
				// 设置默认基础控制总线混合
				DefaultBaseMix = SoundControlBusMix;
			}
			else
			{
				ensureMsgf(SoundControlBusMix, TEXT("Default Control Bus Mix reference missing from Lyra Audio Settings."));
			}
		}
		// ...
				// Load HDR Submix Effect Chain
		// 加载高动态范围混合效果链
		for (const FLyraSubmixEffectChainMap& SoftSubmixEffectChain : LyraAudioSettings->HDRAudioSubmixEffectChain)
		{
			// ....
		}
	}
}
```

#### 加载界面覆写音频控制总线混合
``` cpp
void ULyraAudioMixEffectsSubsystem::PostInitialize()
{
		// ...
	// Register with the loading screen manager
	// 向加载屏幕管理器注册
	if (ULoadingScreenManager* LoadingScreenManager = UGameInstance::GetSubsystem<ULoadingScreenManager>(GetWorld()->GetGameInstance()))
	{
		LoadingScreenManager->OnLoadingScreenVisibilityChangedDelegate().AddUObject(this, &ThisClass::OnLoadingScreenStatusChanged);
		ApplyOrRemoveLoadingScreenMix(LoadingScreenManager->GetLoadingScreenDisplayStatus());
	}

}
void ULyraAudioMixEffectsSubsystem::ApplyOrRemoveLoadingScreenMix(bool bWantsLoadingScreenMix)
{
	UWorld* World = GetWorld();

	if (bAppliedLoadingScreenMix != bWantsLoadingScreenMix && LoadingScreenMix && World)
	{
		if (bWantsLoadingScreenMix)
		{
			// Apply the mix
			UAudioModulationStatics::ActivateBusMix(World, LoadingScreenMix);
		}
		else
		{
			// Remove the mix
			UAudioModulationStatics::DeactivateBusMix(World, LoadingScreenMix);
		}
		bAppliedLoadingScreenMix = bWantsLoadingScreenMix;
	}
}
```
## 调试设置
此处需要演示以下日志的产生:
``` txt
[2025.07.28-05.11.37:196][  0]LogAudioModulation: Display: Initialized Audio Modulation Parameter 'BitDepth'
[2025.07.28-05.11.37:197][  0]LogAudioModulation: Display: Initialized Audio Modulation Parameter 'HPFCutoffFrequency'
[2025.07.28-05.11.37:197][  0]LogAudioModulation: Display: Initialized Audio Modulation Parameter 'LowRateFrequency'
[2025.07.28-05.11.37:197][  0]LogAudioModulation: Display: Initialized Audio Modulation Parameter 'LPFCutoffFrequency'
[2025.07.28-05.11.37:198][  0]LogAudioModulation: Display: Initialized Audio Modulation Parameter 'Pan'
[2025.07.28-05.11.37:198][  0]LogAudioModulation: Display: Initialized Audio Modulation Parameter 'Pitch'
[2025.07.28-05.11.37:198][  0]LogAudioModulation: Display: Initialized Audio Modulation Parameter 'SampleRate'
[2025.07.28-05.11.37:198][  0]LogAudioModulation: Display: Initialized Audio Modulation Parameter 'TimeOfDay'
[2025.07.28-05.11.37:199][  0]LogAudioModulation: Display: Initialized Audio Modulation Parameter 'Volume'
```

``` txt
[2025.07.28-05.22.40:680][696]LogAudioMixer: Display: Audio Buffer Underrun (starvation) detected. InstanceID=9
[2025.07.28-05.22.45:359][696]LogHttpServerModule: Starting all listeners...
[2025.07.28-05.22.45:359][696]LogHttpServerModule: All listeners started
[2025.07.28-05.22.45:359][696]LogAudioModulation: Display: Could not update mix 'CBM_UserMix' because the mix is not currently active.
[2025.07.28-05.22.45:360][696]PIE: Server logged in
[2025.07.28-05.22.45:363][696]PIE: Play in editor total start time 15.7 seconds.
[2025.07.28-05.22.45:364][696]LogBackChannel: Listening on FBackChannelConnection Client Socket (localport: 2049)
[2025.07.28-05.22.45:364][696]LogRemoteSession: Started listening on port 2049
[2025.07.28-05.22.45:366][696]LogLyraExperience: Identified experience LyraExperienceDefinition:B_ShooterGame_Elimination (Source: WorldSettings)
[2025.07.28-05.22.45:366][696]LogLyraExperience: EXPERIENCE: StartExperienceLoad(CurrentExperience = LyraExperienceDefinition:B_ShooterGame_Elimination, Server)
[2025.07.28-05.22.45:367][696]LogLoadingScreen: Showing loading screen when 'IsShowingInitialLoadingScreen()' is false.
```
``` txt
LogAudioModulation: Warning: Bus 'Overall' Not currently applied to Bus Mix 'CBM_UserMix'. Please ensure that all your Mix Profiles have the same Control Buses.
LogAudioModulation: Warning: Bus 'Music' Not currently applied to Bus Mix 'CBM_UserMix'. Please ensure that all your Mix Profiles have the same Control Buses.
LogAudioModulation: Warning: Bus 'SoundFX' Not currently applied to Bus Mix 'CBM_UserMix'. Please ensure that all your Mix Profiles have the same Control Buses.
LogAudioModulation: Warning: Bus 'Dialogue' Not currently applied to Bus Mix 'CBM_UserMix'. Please ensure that all your Mix Profiles have the same Control Buses.
LogAudioModulation: Warning: Bus 'VoiceChat' Not currently applied to Bus Mix 'CBM_UserMix'. Please ensure that all your Mix Profiles have the same Control Buses.
```


## 代码
### LyraAudioSettings.h
``` cpp

USTRUCT()
struct FLyraSubmixEffectChainMap
{
	GENERATED_BODY()
	
	/**
	 * “声音混音分轨类”用于对多个音频源的混合总和效果进行应用。
	 * 
	 */
	UPROPERTY(EditAnywhere, meta = (AllowedClasses = "/Script/Engine.SoundSubmix"))
	TSoftObjectPtr<USoundSubmix> Submix = nullptr;

	/** 一种可作为多种声音混合效果基础的预设设置。该预设可被多个声音所共享。*/
	UPROPERTY(EditAnywhere, meta = (AllowedClasses = "/Script/Engine.SoundEffectSubmixPreset"))
	TArray<TSoftObjectPtr<USoundEffectSubmixPreset>> SubmixEffectChain;

};

/**
 * Lyra项目的音频设置
 * 
 */
UCLASS(MinimalAPI, config = Game, defaultconfig, meta = (DisplayName = "LyraAudioSettings"))
class ULyraAudioSettings : public UDeveloperSettings
{
	GENERATED_BODY()

public:

	/** The Default Base Control Bus Mix */
	/** 默认基础控制总线混合模式 */
	UPROPERTY(config, EditAnywhere, Category = MixSettings, meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBusMix"))
	FSoftObjectPath DefaultControlBusMix;

	/** The Loading Screen Control Bus Mix - Called during loading screens to cover background audio events */
	/** 加载屏幕控制总线组合 - 在加载屏幕期间调用，用于覆盖背景音频事件 */
	UPROPERTY(config, EditAnywhere, Category = MixSettings, meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBusMix"))
	FSoftObjectPath LoadingScreenControlBusMix;

	/** The Default Base Control Bus Mix */
	/** 默认基础控制总线混合模式 */
	UPROPERTY(config, EditAnywhere, Category = UserMixSettings, meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBusMix"))
	FSoftObjectPath UserSettingsControlBusMix;

	/** Control Bus assigned to the Overall sound volume setting */
	/** 分配给整体音量调节的控制总线 */
	UPROPERTY(config, EditAnywhere, Category = UserMixSettings, meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBus"))
	FSoftObjectPath OverallVolumeControlBus;

	/** Control Bus assigned to the Music sound volume setting */
	/** 分配给音乐音量设置的控制总线 */
	UPROPERTY(config, EditAnywhere, Category = UserMixSettings, meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBus"))
	FSoftObjectPath MusicVolumeControlBus;

	/** Control Bus assigned to the SoundFX sound volume setting */
	/** 分配给音效声音音量设置的控制总线 */
	UPROPERTY(config, EditAnywhere, Category = UserMixSettings, meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBus"))
	FSoftObjectPath SoundFXVolumeControlBus;

	/** Control Bus assigned to the Dialogue sound volume setting */
	/** 分配给对话音量设置的控制总线 */
	UPROPERTY(config, EditAnywhere, Category = UserMixSettings, meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBus"))
	FSoftObjectPath DialogueVolumeControlBus;

	/** Control Bus assigned to the VoiceChat sound volume setting */
	/** 分配给语音聊天音量设置的控制总线 */
	UPROPERTY(config, EditAnywhere, Category = UserMixSettings, meta = (AllowedClasses = "/Script/AudioModulation.SoundControlBus"))
	FSoftObjectPath VoiceChatVolumeControlBus;

	/** Submix Processing Chains to achieve high dynamic range audio output */
	/** 亚混音处理链用于实现高动态范围音频输出 */
	UPROPERTY(config, EditAnywhere, Category = EffectSettings)
	TArray<FLyraSubmixEffectChainMap> HDRAudioSubmixEffectChain;
	
	/** Submix Processing Chains to achieve low dynamic range audio output */
	/** 亚混音处理链用于实现低动态范围音频输出 */
	UPROPERTY(config, EditAnywhere, Category = EffectSettings)
	TArray<FLyraSubmixEffectChainMap> LDRAudioSubmixEffectChain;

private:


};

```
### LyraAudioMixEffectsSubsystem.h
``` cpp

// 子混音音频效果链
USTRUCT()
struct FLyraAudioSubmixEffectsChain
{
	GENERATED_BODY()

	// Submix on which to apply the Submix Effect Chain Override
	// 用于应用子混音效果链覆盖功能的子混音通道
	UPROPERTY(Transient)
	TObjectPtr<USoundSubmix> Submix = nullptr;

	// Submix Effect Chain Override (Effects processed in Array index order)
	// 子混音效果链覆盖（按数组索引顺序处理的效果）
	UPROPERTY(Transient)
	TArray<TObjectPtr<USoundEffectSubmixPreset>> SubmixEffectChain;
};

/**
 * This subsystem is meant to automatically engage default and user control bus mixes
 * to retrieve previously saved user settings and apply them to the activated user mix.
 * Additionally, this subsystem will automatically apply HDR/LDR Audio Submix Effect Chain Overrides
 * based on the user's preference for HDR Audio. Submix Effect Chain Overrides are defined in the
 * Lyra Audio Settings.
 *
 * 该子系统旨在自动启用默认和用户控制的音频混合模式，
 * 以获取先前保存的用户设置，并将其应用到激活的用户模式中。
 * 此外，该子系统还将根据用户对 HDR 音频的偏好自动应用 HDR/LDR 音频子混合效果链的覆盖设置。
 * 效果链覆盖设置在莱拉音频设置中定义。
 * 
 */
UCLASS(MinimalAPI)
class ULyraAudioMixEffectsSubsystem : public UWorldSubsystem
{
	GENERATED_BODY()

public:
	// USubsystem implementation Begin
	UE_API virtual void Initialize(FSubsystemCollectionBase& Collection) override;
	UE_API virtual void Deinitialize() override;
	// USubsystem implementation End

	// 是否需要创建这类子系统
	UE_API virtual bool ShouldCreateSubsystem(UObject* Outer) const override;

	/** Called once all UWorldSubsystems have been initialized */
	/** 在所有 UWorld 子系统初始化完毕后被调用一次 */
	UE_API virtual void PostInitialize() override;

	/** Called when world is ready to start gameplay before the game mode transitions to the correct state and call BeginPlay on all actors */
	/** 当游戏世界准备好开始游戏流程（在游戏模式转换至正确状态之前）时调用此函数，并对所有角色调用 BeginPlay 函数 */
	UE_API virtual void OnWorldBeginPlay(UWorld& InWorld) override;

	/** Set whether the HDR Audio Submix Effect Chain Override settings are applied */
	/** 设置是否应用 HDR 音频子混音效果链覆盖设置 */
	UE_API void ApplyDynamicRangeEffectsChains(bool bHDRAudio);
	
protected:

	// 加载屏幕发生变化时调用
	UE_API void OnLoadingScreenStatusChanged(bool bShowingLoadingScreen);

	// 添加或移除加载屏幕时的混音效果
	UE_API void ApplyOrRemoveLoadingScreenMix(bool bWantsLoadingScreenMix);
	
	// Called when determining whether to create this Subsystem
	// 在确定是否创建此子系统时被调用
	UE_API virtual bool DoesSupportWorldType(const EWorldType::Type WorldType) const override;

	// Default Sound Control Bus Mix retrieved from the Lyra Audio Settings
	// 从Lyra音频设置中获取的默认声音控制总线混音设置
	UPROPERTY(Transient)
	TObjectPtr<USoundControlBusMix> DefaultBaseMix = nullptr;

	// Loading Screen Sound Control Bus Mix retrieved from the Lyra Audio Settings
	// 从Lyra音频设置中获取加载屏幕声音控制总线混音设置
	UPROPERTY(Transient)
	TObjectPtr<USoundControlBusMix> LoadingScreenMix = nullptr;

	// User Sound Control Bus Mix retrieved from the Lyra Audio Settings
	// 从Lyra音频设置中获取的用户声音控制总线混合模式
	UPROPERTY(Transient)
	TObjectPtr<USoundControlBusMix> UserMix = nullptr;

	// Overall Sound Control Bus retrieved from the Lyra Audio Settings and linked to the UI and game settings in LyraSettingsLocal
	// 从Lyra音频设置中获取了整体音效控制总线，并将其与Lyra设置本地中的用户界面和游戏设置相连接。
	UPROPERTY(Transient)
	TObjectPtr<USoundControlBus> OverallControlBus = nullptr;

	// Music Sound Control Bus retrieved from the Lyra Audio Settings and linked to the UI and game settings in LyraSettingsLocal
	// 从Lyra音频设置中获取音乐音效控制总线，并将其与Lyra设置本地中的用户界面和游戏设置相连接
	UPROPERTY(Transient)
	TObjectPtr<USoundControlBus> MusicControlBus = nullptr;

	// SoundFX Sound Control Bus retrieved from the Lyra Audio Settings and linked to the UI and game settings in LyraSettingsLocal
	// 从Lyra音频设置中获取了音效音量控制总线，并将其与Lyra设置本地中的用户界面和游戏设置相连接。
	UPROPERTY(Transient)
	TObjectPtr<USoundControlBus> SoundFXControlBus = nullptr;

	// Dialogue Sound Control Bus retrieved from the Lyra Audio Settings and linked to the UI and game settings in LyraSettingsLocal
	// 从Lyra音频设置中获取对话音效控制总线，并将其与Lyra设置本地中的用户界面和游戏设置相连接
	UPROPERTY(Transient)
	TObjectPtr<USoundControlBus> DialogueControlBus = nullptr;

	// VoiceChat Sound Control Bus retrieved from the Lyra Audio Settings and linked to the UI and game settings in LyraSettingsLocal
	// 语音聊天声音控制总线从Lyra音频设置中获取，并与Lyra设置本地中的用户界面和游戏设置相连接。
	UPROPERTY(Transient)
	TObjectPtr<USoundControlBus> VoiceChatControlBus = nullptr;

	// Submix Effect Chain Overrides to apply when HDR Audio is turned on
	// 当启用高动态范围音频时，子混音效果链的优先设置规则
	UPROPERTY(Transient)
	TArray<FLyraAudioSubmixEffectsChain> HDRSubmixEffectChain;

	// Submix Effect hain Overrides to apply when HDR Audio is turned off
	// 联合混音效果：在关闭高动态范围音频时启用此效果以进行调整设置
	UPROPERTY(Transient)
	TArray<FLyraAudioSubmixEffectsChain> LDRSubmixEffectChain;

	// 是否运用了加载屏幕混音
	bool bAppliedLoadingScreenMix = false;
};

```


## 总结
此节作为Lyra本地游戏设置的音频外部调用补充.