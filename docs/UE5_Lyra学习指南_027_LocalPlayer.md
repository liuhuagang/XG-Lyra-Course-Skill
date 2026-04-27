# UE5_Lyra学习指南_027_LocalPlayer

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_027\_LocalPlayer](#ue5_lyra学习指南_027_localplayer)
	- [概述](#概述)
	- [LyraLocalPlayer](#lyralocalplayer)
		- [设置队伍](#设置队伍)
		- [读取设置](#读取设置)
		- [加载共享设置](#加载共享设置)
		- [音频设备变更](#音频设备变更)
			- [绑定位置](#绑定位置)
			- [调用位置](#调用位置)
			- [执行位置](#执行位置)
	- [修复代码](#修复代码)
		- [GameMode](#gamemode)
		- [ExperienceManagerComponent](#experiencemanagercomponent)
		- [GameInstance](#gameinstance)
		- [PlayerController](#playercontroller)
	- [代码](#代码)
		- [LyraLocalPlayer](#lyralocalplayer-1)
		- [LyraSettingsLocal](#lyrasettingslocal)
		- [LyraSettingsShared](#lyrasettingsshared)
	- [总结](#总结)



## 概述
本节主要完成了LyraLocalPlayer的定义.
这里引入了两个新的配置类.分别是ULyraSettingsLocal以及ULyraSettingsShared.
游戏设置将单独用章节章节.此处仅修复各类对它的调用.
## LyraLocalPlayer
### 设置队伍
这个类同样具有队伍属性.并且是从PlayerController传递过来的.
``` cpp
	// 无作用 我们只从PlayerController里面获取队伍信息
	UE_API virtual void SetGenericTeamId(const FGenericTeamId& NewTeamID) override;
	// 从控制器中获取队伍信息
	UE_API virtual FGenericTeamId GetGenericTeamId() const override;
	// 获取队伍变更代理
	UE_API virtual FOnLyraTeamIndexChangedDelegate* GetOnTeamIndexChangedDelegate() override;
```
为了监听PlayerController的变化,需要嵌入到引擎变更控制器的流程中去.
``` cpp
void ULyraLocalPlayer::SwitchController(class APlayerController* PC)
{
	Super::SwitchController(PC);

	OnPlayerControllerChanged(PlayerController);
}

bool ULyraLocalPlayer::SpawnPlayActor(const FString& URL, FString& OutError, UWorld* InWorld)
{
	const bool bResult = Super::SpawnPlayActor(URL, OutError, InWorld);

	OnPlayerControllerChanged(PlayerController);

	return bResult;
}

void ULyraLocalPlayer::InitOnlineSession()
{
	OnPlayerControllerChanged(PlayerController);

	Super::InitOnlineSession();
}

```
当控制器切换之后,重新进行队伍的绑定即可!
``` cpp
void ULyraLocalPlayer::OnPlayerControllerChanged(APlayerController* NewController)
{
	// Stop listening for changes from the old controller
	// 停止监听旧控制器的更改情况
	FGenericTeamId OldTeamID = FGenericTeamId::NoTeam;
	if (ILyraTeamAgentInterface* ControllerAsTeamProvider = Cast<ILyraTeamAgentInterface>(LastBoundPC.Get()))
	{
		OldTeamID = ControllerAsTeamProvider->GetGenericTeamId();
		ControllerAsTeamProvider->GetTeamChangedDelegateChecked().RemoveAll(this);
	}

	// Grab the current team ID and listen for future changes
	FGenericTeamId NewTeamID = FGenericTeamId::NoTeam;
	// 获取当前团队的 ID，并监听未来的变化情况
	if (ILyraTeamAgentInterface* ControllerAsTeamProvider = Cast<ILyraTeamAgentInterface>(NewController))
	{
		NewTeamID = ControllerAsTeamProvider->GetGenericTeamId();
		ControllerAsTeamProvider->GetTeamChangedDelegateChecked().AddDynamic(this, &ThisClass::OnControllerChangedTeam);
		LastBoundPC = NewController;
	}

	ConditionalBroadcastTeamChanged(this, OldTeamID, NewTeamID);
}

```
### 读取设置
本地设置直接读取即可.云端设置需要等待登录完成之后才能读取到.但根据平台可以先创建一个临时的
``` cpp
	/** Gets the local settings for this player, this is read from config files at process startup and is always valid */
	/** 获取此玩家的本地设置，该设置在程序启动时从配置文件中读取，并且始终有效 */
	UFUNCTION()
	UE_API ULyraSettingsLocal* GetLocalSettings() const;

	/** Gets the shared setting for this player, this is read using the save game system so may not be correct until after user login */
	/** 获取此玩家的共享设置，此操作是通过保存游戏系统来实现的，因此在用户登录之后才能确保其准确性 */
	UFUNCTION()
	UE_API ULyraSettingsShared* GetSharedSettings() const;
```
### 加载共享设置
在GameInstance中的用户登录之后调用
``` cpp
void ULyraGameInstance::HandlerUserInitialized(const UCommonUserInfo* UserInfo, bool bSuccess, FText Error, ECommonUserPrivilege RequestedPrivilege, ECommonUserOnlineContext OnlineContext)
{
	Super::HandlerUserInitialized(UserInfo, bSuccess, Error, RequestedPrivilege, OnlineContext);

	// If login succeeded, tell the local player to load their settings
	// 如果登录成功，就告知本地玩家加载他们的设置
	if (bSuccess && ensure(UserInfo))
	{
		ULyraLocalPlayer* LocalPlayer = Cast<ULyraLocalPlayer>(GetLocalPlayerByIndex(UserInfo->LocalPlayerIndex));

		// There will not be a local player attached to the dedicated server user
		// 专用服务器用户不会关联本地玩家。
		if (LocalPlayer)
		{
			LocalPlayer->LoadSharedSettingsFromDisk();
		}
	}
}
```
加载完成之后需要进行缓存,避免重复加载
``` cpp
void ULyraLocalPlayer::LoadSharedSettingsFromDisk(bool bForceLoad)
{
	FUniqueNetIdRepl CurrentNetId = GetCachedUniqueNetId();
	if (!bForceLoad && SharedSettings && CurrentNetId == NetIdForSharedSettings)
	{
		// Already loaded once, don't reload
		// 已经加载过一次，不要再重新加载了
		return;
	}

	ensure(ULyraSettingsShared::AsyncLoadOrCreateSettings(this, ULyraSettingsShared::FOnSettingsLoadedEvent::CreateUObject(this, &ULyraLocalPlayer::OnSharedSettingsLoaded)));
}

void ULyraLocalPlayer::OnSharedSettingsLoaded(ULyraSettingsShared* LoadedOrCreatedSettings)
{
	// The settings are applied before it gets here
	// 这些设置在到达此处之前就已经生效了
	if (ensure(LoadedOrCreatedSettings))
	{
		// This will replace the temporary or previously loaded object which will GC out normally
		// 这将替换临时对象或之前加载的对象，这些对象将按照正常流程进行垃圾回收。
		SharedSettings = LoadedOrCreatedSettings;

		NetIdForSharedSettings = GetCachedUniqueNetId();
	}
}

```
### 音频设备变更
#### 绑定位置
``` cpp
void ULyraLocalPlayer::PostInitProperties()
{
	Super::PostInitProperties();

	if (ULyraSettingsLocal* LocalSettings = GetLocalSettings())
	{
		LocalSettings->OnAudioOutputDeviceChanged.AddUObject(this, &ULyraLocalPlayer::OnAudioOutputDeviceChanged);
	}
}

```
#### 调用位置
``` cpp
UCLASS()
class ULyraSettingsLocal : public UGameUserSettings
{
	GENERATED_BODY()
public:
	DECLARE_EVENT_OneParam(ULyraSettingsLocal, FAudioDeviceChanged, const FString& /*DeviceId*/);
	FAudioDeviceChanged OnAudioOutputDeviceChanged;

}
```
``` cpp
void ULyraSettingsLocal::SetAudioOutputDeviceId(const FString& InAudioOutputDeviceId)
{
	AudioOutputDeviceId = InAudioOutputDeviceId;
	OnAudioOutputDeviceChanged.Broadcast(InAudioOutputDeviceId);
}

```
#### 执行位置
``` cpp
void ULyraLocalPlayer::OnAudioOutputDeviceChanged(const FString& InAudioOutputDeviceId)
{
	FOnCompletedDeviceSwap DevicesSwappedCallback;
	DevicesSwappedCallback.BindUFunction(this, FName("OnCompletedAudioDeviceSwap"));

	/**
	 * 将音频输出设备更改为所请求的设备
	 * @参数 新设备标识 - 要更换的设备标识
	 * @参数 完成设备更换事件 - 当音频端点设备获取完毕时触发的事件
	 * 
	 */
	UAudioMixerBlueprintLibrary::SwapAudioOutputDevice(GetWorld(), InAudioOutputDeviceId, DevicesSwappedCallback);
}

void ULyraLocalPlayer::OnCompletedAudioDeviceSwap(const FSwapAudioOutputResult& SwapResult)
{
	if (SwapResult.Result == ESwapAudioOutputDeviceResultState::Failure)
	{
	}
}
```


## 修复代码
### GameMode
修复了从PlayerState的中直接获取PawnData.
这里录制PlayerState的时候应该已经修复了.
但是我制作课件的时候,还没有录制.所以再次强调下.避免漏了TODO.
``` cpp
const ULyraPawnData* ALyraGameMode::GetPawnDataForController(const AController* InController) const
{
	// See if pawn data is already set on the player state
	// 检查是否已为玩家状态设置了棋子数据

	// 控制器不应该为空!
	if (InController != nullptr)
	{
		// AI控制器也会有PlayerState,通过其的构造函数开启
		if (const ALyraPlayerState* LyraPS = InController->GetPlayerState<ALyraPlayerState>())
		{
			if (const ULyraPawnData* PawnData = LyraPS->GetPawnData<ULyraPawnData>())
			{
				return PawnData;
			}
		}
	}
}
```

### ExperienceManagerComponent
体验加载设置变更
``` cpp
void ULyraExperienceManagerComponent::OnExperienceFullLoadCompleted()
{
	// Apply any necessary scalability settings
#if !UE_SERVER
	ULyraSettingsLocal::Get()->OnExperienceLoaded();
#endif
}

```

### GameInstance
初始化设置以及获取玩家控制器
这里需要在玩家登陆后,读取共享设置
``` cpp
void ULyraGameInstance::HandlerUserInitialized(const UCommonUserInfo* UserInfo, bool bSuccess, FText Error, ECommonUserPrivilege RequestedPrivilege, ECommonUserOnlineContext OnlineContext)
{
	Super::HandlerUserInitialized(UserInfo, bSuccess, Error, RequestedPrivilege, OnlineContext);

	// If login succeeded, tell the local player to load their settings
	// 如果登录成功，就告知本地玩家加载他们的设置
	if (bSuccess && ensure(UserInfo))
	{
		ULyraLocalPlayer* LocalPlayer = Cast<ULyraLocalPlayer>(GetLocalPlayerByIndex(UserInfo->LocalPlayerIndex));

		// There will not be a local player attached to the dedicated server user
		// 专用服务器用户不会关联本地玩家。
		if (LocalPlayer)
		{
			LocalPlayer->LoadSharedSettingsFromDisk();
		}
	}
}

```

可能之前漏了这个玩家控制前的获取方法,也修复一下
``` cpp
ALyraPlayerController* ULyraGameInstance::GetPrimaryPlayerController() const
{
	return Cast<ALyraPlayerController>(Super::GetPrimaryPlayerController(false));
}


```

### PlayerController
恢复了从游戏设置获取是否开启记录回放.
恢复了从共享设置里面绑定设备变更 以及力反馈.
``` cpp
bool ALyraPlayerController::ShouldRecordClientReplay()
{	
	{
		// If this is possible, now check the settings
		// 如果可行的话，现在就检查一下设置吧
		if (const ULyraLocalPlayer* LyraLocalPlayer = Cast<ULyraLocalPlayer>(GetLocalPlayer()))
		{
			//004:从本地游戏设置从读取是否自动记录回放功能
			if (LyraLocalPlayer->GetLocalSettings()->ShouldAutoRecordReplays())
			{
				return true;
			}
		}
	}
	return false;
}

```
``` cpp
void ALyraPlayerController::SetPlayer(UPlayer* InPlayer)
{
	Super::SetPlayer(InPlayer);

	if (const ULyraLocalPlayer* LyraLocalPlayer = Cast<ULyraLocalPlayer>(InPlayer))
	{
		// 005 触发玩家对应的设置变更
		ULyraSettingsShared* UserSettings = LyraLocalPlayer->GetSharedSettings();
		UserSettings->OnSettingChanged.AddUObject(this, &ThisClass::OnSettingsChanged);

		OnSettingsChanged(UserSettings);
	}
}

void ALyraPlayerController::OnSettingsChanged(ULyraSettingsShared* InSettings)
{
	// 005 从共享设置中读取力反馈
	bForceFeedbackEnabled = InSettings->GetForceFeedbackEnabled();
}


```
## 代码

### LyraLocalPlayer
``` cpp

/**
 * ULyraLocalPlayer
 * Lyra项目的本地玩家类
 */
UCLASS(MinimalAPI)
class ULyraLocalPlayer : public UCommonLocalPlayer, public ILyraTeamAgentInterface
{
	GENERATED_BODY()

public:

	// 构造函数 无作用
	UE_API ULyraLocalPlayer();

	//~UObject interface
	// 绑定音频设备变更代理
	UE_API virtual void PostInitProperties() override;
	//~End of UObject interface

	//~UPlayer interface

	/**
	 * 	动态地为玩家分配控制器并设置视口。
	 *	触发控制器变更代理
	 * 	
	 * 	@参数  PC - 用于分配给玩家的新玩家控制器
	 */
	UE_API virtual void SwitchController(class APlayerController* PC) override;
	//~End of UPlayer interface

	//~ULocalPlayer interface

	/**
	 * 为该玩家创建一个角色。
	 * 触发控制器变更
	 * @参数 URL - 玩家加入时所使用的 URL。
	 * @参数 OutError - 若出现错误，则返回错误描述。
	 * @参数 InWorld - 用于生成玩家角色的场景
	 * @返回值 若出现错误则为 False，若成功生成玩家角色则为 True。*
	 * 
	 */
	UE_API virtual bool SpawnPlayActor(const FString& URL, FString& OutError, UWorld* InWorld) override;

	/**
	 * 被要求初始化在线代表人员
	 * 
	 */
	UE_API virtual void InitOnlineSession() override;
	//~End of ULocalPlayer interface

	//~ILyraTeamAgentInterface interface
	
	// 无作用 我们只从PlayerController里面获取队伍信息
	UE_API virtual void SetGenericTeamId(const FGenericTeamId& NewTeamID) override;
	// 从控制器中获取队伍信息
	UE_API virtual FGenericTeamId GetGenericTeamId() const override;
	// 获取队伍变更代理
	UE_API virtual FOnLyraTeamIndexChangedDelegate* GetOnTeamIndexChangedDelegate() override;
	
	//~End of ILyraTeamAgentInterface interface

	/** Gets the local settings for this player, this is read from config files at process startup and is always valid */
	/** 获取此玩家的本地设置，该设置在程序启动时从配置文件中读取，并且始终有效 */
	UFUNCTION()
	UE_API ULyraSettingsLocal* GetLocalSettings() const;

	/** Gets the shared setting for this player, this is read using the save game system so may not be correct until after user login */
	/** 获取此玩家的共享设置，此操作是通过保存游戏系统来实现的，因此在用户登录之后才能确保其准确性 */
	UFUNCTION()
	UE_API ULyraSettingsShared* GetSharedSettings() const;

	/** Starts an async request to load the shared settings, this will call OnSharedSettingsLoaded after loading or creating new ones */
	/** 启动一个异步请求以加载共享设置，加载或创建新设置完成后会调用 OnSharedSettingsLoaded 方法 */
	// 从ULyraGameInstance::HandlerUserInitialized中调用.
	UE_API void LoadSharedSettingsFromDisk(bool bForceLoad = false);

protected:


	// 当设置加载完成后调用,用以存储对象,避免回收,避免重复加载
	UE_API void OnSharedSettingsLoaded(ULyraSettingsShared* LoadedOrCreatedSettings);

	// 用于触发音频设备变更的函数
	UE_API void OnAudioOutputDeviceChanged(const FString& InAudioOutputDeviceId);

	// 用于音频设备变更完成后的回调函数
	UFUNCTION()
	UE_API void OnCompletedAudioDeviceSwap(const FSwapAudioOutputResult& SwapResult);

	// 玩家控制器的变更函数,主要用于重新绑定传递队伍信息
	UE_API void OnPlayerControllerChanged(APlayerController* NewController);

	// 向下级传递队伍变更信息
	UFUNCTION()
	UE_API void OnControllerChangedTeam(UObject* TeamAgent, int32 OldTeam, int32 NewTeam);

private:
	// 共享设置
	UPROPERTY(Transient)
	mutable TObjectPtr<ULyraSettingsShared> SharedSettings;
	
	/**
	 * FUniqueNetIdRepl
	 * 对不透明类型 FUniqueNetId 的包装器*
	 * 确保 FUniqueNetId 中的不透明部分在通过网络远程过程调用（RPC）以及角色复制过程中能够得到正确处理/序列化
	 * 
	 * FUniqueNetId
	 * 基于在线标识的用户资料服务的抽象
	 * 该类应保持不可见性
	 * 
	 */
	FUniqueNetIdRepl NetIdForSharedSettings;

	// 临时对象 输入映射的上下文 目前没用到
	UPROPERTY(Transient)
	mutable TObjectPtr<const UInputMappingContext> InputMappingContext;

	// 用于传递给下级队伍信息的代理
	UPROPERTY()
	FOnLyraTeamIndexChangedDelegate OnTeamChangedDelegate;

	// 上次绑定的玩家控制器
	UPROPERTY()
	TWeakObjectPtr<APlayerController> LastBoundPC;
};
```
### LyraSettingsLocal
``` cpp
/**
 * UGameUserSettings:
 * Stores user settings for a game (for example graphics and sound settings), with the ability to save and load to and from a file.
 *
 * 为游戏保存用户设置（例如图形和声音设置），并具备将设置保存至文件、从文件加载设置以及从文件中加载并应用设置的功能。
 * 
 */
/**
 * ULyraSettingsLocal
 * 本地设置
 */
UCLASS()
class ULyraSettingsLocal : public UGameUserSettings
{
	GENERATED_BODY()

public:

	ULyraSettingsLocal();

	static ULyraSettingsLocal* Get();
	//////////////////////////////////////////////////////////////////
	// Audio - Volume
public:
	DECLARE_EVENT_OneParam(ULyraSettingsLocal, FAudioDeviceChanged, const FString& /*DeviceId*/);
	FAudioDeviceChanged OnAudioOutputDeviceChanged;

	//////////////////////////////////////////////////////////////////
	// Audio - Sound

	// 这个方法在ExperienceManagerComponent里面体验完全加载后调用
	void OnExperienceLoaded();


public:
	UFUNCTION()
	bool ShouldAutoRecordReplays() const { return false; }
};
``` 
### LyraSettingsShared
``` cpp
/**
 * ULyraSettingsShared - The "Shared" settings are stored as part of the USaveGame system, these settings are not machine
 * specific like the local settings, and are safe to store in the cloud - and 'share' them.  Using the save game system
 * we can also store settings per player, so things like controller keybind preferences should go here, because if those
 * are stored in the local settings all users would get them.
 *
 * ULyraSettingsShared - “共享”设置被存储在 USaveGame 系统中，与本地设置不同，这些设置并非针对特定机器的，而且可以安全地存储在云端并进行“共享”。通过使用保存游戏系统，我们还可以为每个玩家存储设置，比如控制器按键绑定偏好就应存放在这里，因为如果这些设置存放在本地设置中，那么所有用户都会获取到它们。
 */
UCLASS()
class ULyraSettingsShared : public ULocalPlayerSaveGame
{
	GENERATED_BODY()
	
public:
	
	DECLARE_EVENT_OneParam(ULyraSettingsShared, FOnSettingChangedEvent, ULyraSettingsShared* Settings);
	FOnSettingChangedEvent OnSettingChanged;

public:

	ULyraSettingsShared();

	/** Creates a temporary settings object, this will be replaced by one loaded from the user's save game */
	/** 创建一个临时的设置对象，该对象将被从用户的游戏存档中加载的设置对象所替换 */
	static ULyraSettingsShared* CreateTemporarySettings(const ULyraLocalPlayer* LocalPlayer);


	/** Synchronously loads a settings object, this is not valid to call before login */
	/** 异步加载一个设置对象，此操作在登录之前不可执行 */
	static ULyraSettingsShared* LoadOrCreateSettings(const ULyraLocalPlayer* LocalPlayer);

	DECLARE_DELEGATE_OneParam(FOnSettingsLoadedEvent, ULyraSettingsShared* Settings);

	/** Starts an async load of the settings object, calls Delegate on completion */
	/** 启动对设置对象的异步加载过程，完成时调用委托函数 */
	static bool AsyncLoadOrCreateSettings(const ULyraLocalPlayer* LocalPlayer, FOnSettingsLoadedEvent Delegate);

	////////////////////////////////////////////////////////
	// Gamepad Vibration
public:
	UFUNCTION()
	bool GetForceFeedbackEnabled() const { return false; }

};
```

## 总结
本节也是一次性搞了LocalPlayer.
但同时又引入了两个新的配置类.一个是嵌入到引擎的.另一个是通过存档来做的.后续会结合UI的使用来一起讲解.