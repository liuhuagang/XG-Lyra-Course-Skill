# UE5_Lyra学习指南_049_CommonUser

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_049\_CommonUser](#ue5_lyra学习指南_049_commonuser)
	- [概述](#概述)
	- [代码](#代码)
		- [AsyncAction\_CommonUserInitialize](#asyncaction_commonuserinitialize)
		- [CommonSessionSubsystem](#commonsessionsubsystem)
		- [CommonUserBasicPresence](#commonuserbasicpresence)
		- [CommonUserModule](#commonusermodule)
		- [CommonUserSubsytem](#commonusersubsytem)
		- [CommonUserTypes](#commonusertypes)
	- [总结](#总结)



## 概述
此处讲CommonUser的插件的头文件代码列出.以供参考!
## 代码
### AsyncAction_CommonUserInitialize
``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.
// Finished.
#pragma once

#include "CommonUserSubsystem.h"
#include "Engine/CancellableAsyncAction.h"

#include "AsyncAction_CommonUserInitialize.generated.h"

#define UE_API COMMONUSER_API

enum class ECommonUserOnlineContext : uint8;
enum class ECommonUserPrivilege : uint8;
struct FInputDeviceId;

class FText;
class UObject;
struct FFrame;

/**
 * Async action to handle different functions for initializing users
 * 异步操作，用于处理针对用户初始化的不同功能
 */
UCLASS(MinimalAPI)
class UAsyncAction_CommonUserInitialize : public UCancellableAsyncAction
{
	GENERATED_BODY()

public:
	/**
	 * Initializes a local player with the common user system, which includes doing platform-specific login and privilege checks.
	 * When the process has succeeded or failed, it will broadcast the OnInitializationComplete delegate.
	 *
	 * @param LocalPlayerIndex	Desired index of ULocalPlayer in Game Instance, 0 will be primary player and 1+ for local multiplayer
	 * @param PrimaryInputDevice Primary input device for the user, if invalid will use the system default
	 * @param bCanUseGuestLogin	If true, this player can be a guest without a real system net id
	 */
	/**
	 * 使用通用用户系统初始化本地玩家，这包括进行特定平台的登录和权限检查。
	 * 当该过程成功或失败时，它会广播“初始化完成”委托。
	 * @参数 LocalPlayerIndex：游戏实例中 ULocalPlayer 的期望索引值，0 表示为主玩家，1 及以上表示本地多人游戏模式
	 * @参数 PrimaryInputDevice：用户的主要输入设备，无效时将使用系统默认设置
	 * @参数 bCanUseGuestLogin：如果为真，则此玩家可以作为访客存在，无需真实的系统网络标识符
	 * 
	 */
	UFUNCTION(BlueprintCallable, Category = CommonUser, meta = (BlueprintInternalUseOnly = "true"))
	static UE_API UAsyncAction_CommonUserInitialize* InitializeForLocalPlay(UCommonUserSubsystem* Target, int32 LocalPlayerIndex, FInputDeviceId PrimaryInputDevice, bool bCanUseGuestLogin);

	/**
	 * Attempts to log an existing user into the platform-specific online backend to enable full online play
	 * When the process has succeeded or failed, it will broadcast the OnInitializationComplete delegate.
	 *
	 * @param LocalPlayerIndex	Index of existing LocalPlayer in Game Instance
	 */
	/**
	 * 尝试让现有用户登录到平台特定的在线后端，以实现完全的在线游戏功能
	 * 当该过程成功或失败时，会广播“OnInitializationComplete”委托。
	 * @参数 LocalPlayerIndex：游戏实例中现有本地玩家的索引号
	 * 
	 */
	UFUNCTION(BlueprintCallable, Category = CommonUser, meta = (BlueprintInternalUseOnly = "true"))
	static UE_API UAsyncAction_CommonUserInitialize* LoginForOnlinePlay(UCommonUserSubsystem* Target, int32 LocalPlayerIndex);

	/** Call when initialization succeeds or fails */
	/** 当初始化成功或失败时调用 */
	UPROPERTY(BlueprintAssignable)
	FCommonUserOnInitializeCompleteMulticast OnInitializationComplete;

	/** Fail and send callbacks if needed */
	/** 若有必要，则在失败时调用回调函数 */
	UE_API void HandleFailure();

	/** Wrapper delegate, will pass on to OnInitializationComplete if appropriate */
	/** 封装委托，若情况适宜则会将其传递给“初始化完成”事件 */
	UFUNCTION()
	UE_API virtual void HandleInitializationComplete(const UCommonUserInfo* UserInfo, bool bSuccess, FText Error, ECommonUserPrivilege RequestedPrivilege, ECommonUserOnlineContext OnlineContext);

protected:
	/** Actually start the initialization */
	/** 实际开始初始化操作 */
	UE_API virtual void Activate() override;

	TWeakObjectPtr<UCommonUserSubsystem> Subsystem;
	FCommonUserInitializeParams Params;
};

#undef UE_API


```
### CommonSessionSubsystem
``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.
// Finished.
#pragma once

#include "CommonUserTypes.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "UObject/ObjectPtr.h"
#include "UObject/StrongObjectPtr.h"
#include "UObject/PrimaryAssetId.h"
#include "UObject/WeakObjectPtr.h"
#include "PartyBeaconClient.h"
#include "PartyBeaconHost.h"
#include "PartyBeaconState.h"
#if! COMMONUSER_OSSV1
#include "Online/Sessions.h"
#endif



class APlayerController;
class AOnlineBeaconHost;
class ULocalPlayer;
namespace ETravelFailure { enum Type : int; }
struct FOnlineResultInformation;

#if COMMONUSER_OSSV1
#include "Interfaces/OnlineSessionInterface.h"
#include "OnlineSessionSettings.h"
#else
#include "Online/Lobbies.h"
#include "Online/OnlineAsyncOpHandle.h"
#endif // COMMONUSER_OSSV1

#include "CommonSessionSubsystem.generated.h"

class UWorld;
class FCommonSession_OnlineSessionSettings;

#if COMMONUSER_OSSV1
class FCommonOnlineSearchSettingsOSSv1;
using FCommonOnlineSearchSettings = FCommonOnlineSearchSettingsOSSv1;
#else
class FCommonOnlineSearchSettingsOSSv2;
using FCommonOnlineSearchSettings = FCommonOnlineSearchSettingsOSSv2;
#endif // COMMONUSER_OSSV1


//////////////////////////////////////////////////////////////////////
// UCommonSession_HostSessionRequest

/** Specifies the online features and connectivity that should be used for a game session */
/** 指定游戏会话中应采用的在线功能和连接方式 */
UENUM(BlueprintType)
enum class ECommonSessionOnlineMode : uint8
{
	Offline,
	LAN,
	Online
};

/** A request object that stores the parameters used when hosting a gameplay session */
/** 一个请求对象，用于存储举办游戏环节时所使用的参数 */
UCLASS(MinimalAPI, BlueprintType)
class UCommonSession_HostSessionRequest : public UObject
{
	GENERATED_BODY()

public:
	/** Indicates if the session is a full online session or a different type */
	/** 表示该会话是完整的在线会话还是其他类型的会话 */
	UPROPERTY(BlueprintReadWrite, Category=Session)
	ECommonSessionOnlineMode OnlineMode;

	/** True if this request should create a player-hosted lobbies if available */
	/** 如果当前请求能够在可用的情况下创建玩家主导型的游戏房间，则返回 true */
	UPROPERTY(BlueprintReadWrite, Category = Session)
	bool bUseLobbies;

	/** True if this request should create a lobby with enabled voice chat in available */
	/** 如果此请求需要创建一个具备启用语音聊天功能的房间，则返回 true */
	UPROPERTY(BlueprintReadWrite, Category = Session)
	bool bUseLobbiesVoiceChat;

	/** True if this request should create a session that will appear in the user's presence information */
	/** 如果此请求需要创建一个会话，并且该会话将出现在用户的个人资料中，则返回 true */
	UPROPERTY(BlueprintReadWrite, Category = Session)
	bool bUsePresence;

	/** String used during matchmaking to specify what type of game mode this is */
	/** 在匹配过程中使用的字符串，用于指定此游戏模式的类型 */
	UPROPERTY(BlueprintReadWrite, Category=Session)
	FString ModeNameForAdvertisement;

	/** The map that will be loaded at the start of gameplay, this needs to be a valid Primary Asset top-level map */
	/** 游戏开始时将加载的地图，此地图必须是有效的主资产层级地图 */
	UPROPERTY(BlueprintReadWrite, Category=Session, meta=(AllowedTypes="World"))
	FPrimaryAssetId MapID;

	/** Extra arguments passed as URL options to the game */
	/** 作为游戏的 URL 选项传递的额外参数 */
	UPROPERTY(BlueprintReadWrite, Category=Session)
	TMap<FString, FString> ExtraArgs;

	/** Maximum players allowed per gameplay session */
	/** 每次游戏时段允许的最大玩家数量 */
	UPROPERTY(BlueprintReadWrite, Category=Session)
	int32 MaxPlayerCount = 16;

public:
	/** Returns the maximum players that should actually be used, could be overridden in child classes */
	/** 返回实际应使用的最大玩家数量，该值可在子类中进行重写 */
	COMMONUSER_API virtual int32 GetMaxPlayers() const;

	/** Returns the full map name that will be used during gameplay */
	/** 返回在游戏过程中将使用的完整地图名称 */
	COMMONUSER_API virtual FString GetMapName() const;

	/** Constructs the full URL that will be passed to ServerTravel */
	/** 构建将传递给“服务器导航”功能的完整 URL */
	COMMONUSER_API virtual FString ConstructTravelURL() const;

	/** Returns true if this request is valid, returns false and logs errors if it is not */
	/** 如果此请求有效则返回 true，否则返回 false 并记录错误信息 */
	COMMONUSER_API virtual bool ValidateAndLogErrors(FText& OutError) const;
};


//////////////////////////////////////////////////////////////////////
// UCommonSession_SearchResult

/** A result object returned from the online system that describes a joinable game session */
/** 从在线系统返回的结果对象，该对象描述了一个可参与的游戏时段 */
UCLASS(MinimalAPI, BlueprintType)
class UCommonSession_SearchResult : public UObject
{
	GENERATED_BODY()

public:
	/** Returns an internal description of the session, not meant to be human readable */
	/** 返回会话的内部描述，此描述不供人类阅读 */
	UFUNCTION(BlueprintCallable, Category=Session)
	COMMONUSER_API FString GetDescription() const;

	/** Gets an arbitrary string setting, bFoundValue will be false if the setting does not exist */
	/** 获取任意字符串设置，若该设置不存在，则变量 bFoundValue 的值将为 false */
	UFUNCTION(BlueprintPure, Category=Sessions)
	COMMONUSER_API void GetStringSetting(FName Key, FString& Value, bool& bFoundValue) const;

	/** Gets an arbitrary integer setting, bFoundValue will be false if the setting does not exist */
	/** 获取任意整数值设置，若该设置不存在，则变量 bFoundValue 的值为 false */
	UFUNCTION(BlueprintPure, Category = Sessions)
	COMMONUSER_API void GetIntSetting(FName Key, int32& Value, bool& bFoundValue) const;

	/** The number of private connections that are available */
	/** 当前可用的私有连接数量 */
	UFUNCTION(BlueprintPure, Category=Sessions)
	COMMONUSER_API int32 GetNumOpenPrivateConnections() const;

	/** The number of publicly available connections that are available */
	/** 公开可用的连接数量 */
	UFUNCTION(BlueprintPure, Category=Sessions)
	COMMONUSER_API int32 GetNumOpenPublicConnections() const;

	/** The maximum number of publicly available connections that could be available, including already filled connections */
	/** 可供公众使用的最大连接数量，包括已占用的连接数量 */
	UFUNCTION(BlueprintPure, Category = Sessions)
	COMMONUSER_API int32 GetMaxPublicConnections() const;

	/** Ping to the search result, MAX_QUERY_PING is unreachable */
	/** 向搜索结果发送 Ping 操作，但 MAX_QUERY_PING 无法到达 */
	UFUNCTION(BlueprintPure, Category=Sessions)
	COMMONUSER_API int32 GetPingInMs() const;

public:
	/** Pointer to the platform-specific implementation */
	/** 指向平台特定实现的指针 */
#if COMMONUSER_OSSV1
	FOnlineSessionSearchResult Result;
#else
	TSharedPtr<const UE::Online::FLobby> Lobby;

	UE::Online::FOnlineSessionId SessionID;
#endif // COMMONUSER_OSSV1

};


//////////////////////////////////////////////////////////////////////
// UCommonSession_SearchSessionRequest

/** Delegates called when a session search completes */
/** 当会话搜索完成时所调用的回调函数 */
DECLARE_MULTICAST_DELEGATE_TwoParams(FCommonSession_FindSessionsFinished, bool bSucceeded, const FText& ErrorMessage);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FCommonSession_FindSessionsFinishedDynamic, bool, bSucceeded, FText, ErrorMessage);

/** Request object describing a session search, this object will be updated once the search has completed */
/** 描述会话搜索的请求对象，该对象在搜索完成后会进行更新 */
UCLASS(MinimalAPI, BlueprintType)
class UCommonSession_SearchSessionRequest : public UObject
{
	GENERATED_BODY()

public:
	/** Indicates if the this is looking for full online games or a different type like LAN */
	/** 表示此功能是用于查找完整的在线游戏，还是查找诸如局域网游戏之类的其他类型的游戏 */
	UPROPERTY(BlueprintReadWrite, Category = Session)
	ECommonSessionOnlineMode OnlineMode;

	/** True if this request should look for player-hosted lobbies if they are available, false will only search for registered server sessions */
	/** 如果存在玩家自建的房间，则应执行此请求以查找这些房间；若为假，则仅搜索已注册的服务器会话 */
	UPROPERTY(BlueprintReadWrite, Category = Session)
	bool bUseLobbies;

	/** List of all found sessions, will be valid when OnSearchFinished is called */
	/** 所有找到的会话列表，当调用“OnSearchFinished”时该列表仍有效 */
	UPROPERTY(BlueprintReadOnly, Category=Session)
	TArray<TObjectPtr<UCommonSession_SearchResult>> Results;

	/** Native Delegate called when a session search completes */
	/** 当会话搜索完成时所调用的原生委托函数 */
	FCommonSession_FindSessionsFinished OnSearchFinished;

	/** Called by subsystem to execute finished delegates */
	/** 由子系统调用以执行已完成的委托 */
	COMMONUSER_API void NotifySearchFinished(bool bSucceeded, const FText& ErrorMessage);

private:
	/** Delegate called when a session search completes */
	/** 当会话搜索完成时所调用的委托函数 */
	UPROPERTY(BlueprintAssignable, Category = "Events", meta = (DisplayName = "On Search Finished", AllowPrivateAccess = true))
	FCommonSession_FindSessionsFinishedDynamic K2_OnSearchFinished;
};


//////////////////////////////////////////////////////////////////////
// CommonSessionSubsystem Events

/**
 * Event triggered when the local user has requested to join a session from an external source, for example from a platform overlay.
 * Generally, the game should transition the player into the session.
 * @param LocalPlatformUserId the local user id that accepted the invitation. This is a platform user id because the user might not be signed in yet.
 * @param RequestedSession the requested session. Can be null if there was an error processing the request.
 * @param RequestedSessionResult result of the requested session processing
 */
/**
* 当本地用户从外部来源（例如平台覆盖层）请求加入会话时触发此事件。
* 通常，游戏应将玩家引导进入该会话。
* @参数 LocalPlatformUserId 接受邀请的本地用户 ID。这是一个平台用户 ID，因为用户可能尚未登录。
* @参数 RequestedSession 被请求的会话。如果处理请求时出现错误，则可以为 null。
* @参数 RequestedSessionResult 请求会话处理的结果*/
DECLARE_MULTICAST_DELEGATE_ThreeParams(FCommonSessionOnUserRequestedSession, const FPlatformUserId& /*LocalPlatformUserId*/, UCommonSession_SearchResult* /*RequestedSession*/, const FOnlineResultInformation& /*RequestedSessionResult*/);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_ThreeParams(FCommonSessionOnUserRequestedSession_Dynamic, const FPlatformUserId&, LocalPlatformUserId, UCommonSession_SearchResult*, RequestedSession, const FOnlineResultInformation&, RequestedSessionResult);

/**
 * Event triggered when a session join has completed, after joining the underlying session and before traveling to the server if it was successful.
 * The event parameters indicate if this was successful, or if there was an error that will stop it from traveling.
 * @param Result result of the session join
 */
/**
* 当会话加入操作完成时触发此事件，即在加入底层会话之后且在前往服务器之前（如果操作成功的话）。
* 该事件参数表明此次操作是否成功，或者是否存在会导致无法前往服务器的错误。
* @参数 结果：会话加入的结果*/
DECLARE_MULTICAST_DELEGATE_OneParam(FCommonSessionOnJoinSessionComplete, const FOnlineResultInformation& /*Result*/);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FCommonSessionOnJoinSessionComplete_Dynamic, const FOnlineResultInformation&, Result);

/**
 * Event triggered when a session creation for hosting has completed, right before it travels to the map.
 * The event parameters indicate if this was successful, or if there was an error that will stop it from traveling.
 * @param Result result of the session join
 */
/**
* 当用于托管的会话创建完成并即将传输至地图时触发此事件。
* 该事件参数表明此次操作是否成功，或者是否存在会导致传输中断的错误。
* @参数 结果：会话加入的结果*/
DECLARE_MULTICAST_DELEGATE_OneParam(FCommonSessionOnCreateSessionComplete, const FOnlineResultInformation& /*Result*/);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FCommonSessionOnCreateSessionComplete_Dynamic, const FOnlineResultInformation&, Result);

/**
 * Event triggered when the local user has requested to destroy a session from an external source, for example from a platform overlay.
 * The game should transition the player out of the session.
 * @param LocalPlatformUserId the local user id that made the destroy request. This is a platform user id because the user might not be signed in yet.
 * @param SessionName the name identifier for the session.
 */
/**
* 当本地用户从外部来源（例如平台覆盖层）请求销毁会话时触发此事件。
* 游戏应将玩家从该会话中移出。
* @参数 LocalPlatformUserId 发出销毁请求的本地用户 ID。这是一个平台用户 ID，因为用户可能尚未登录。
* @参数 SessionName 会话的名称标识符。*/
DECLARE_MULTICAST_DELEGATE_TwoParams(FCommonSessionOnDestroySessionRequested, const FPlatformUserId& /*LocalPlatformUserId*/, const FName& /*SessionName*/);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FCommonSessionOnDestroySessionRequested_Dynamic, const FPlatformUserId&, LocalPlatformUserId, const FName&, SessionName);

/**
 * Event triggered when a session join has completed, after resolving the connect string and prior to the client traveling.
 * @param URL resolved connection string for the session with any additional arguments
 */
/**
* 当会话加入操作完成时触发此事件，此时已解析连接字符串，且客户端尚未开始移动。
* @参数 URL 该会话的已解析连接字符串（包含任何附加参数）*/
DECLARE_MULTICAST_DELEGATE_OneParam(FCommonSessionOnPreClientTravel, FString& /*URL*/);

/**
 * Event triggered at different points in the session ecosystem that represent a user-presentable state of the session.
 * This should not be used for online functionality (use OnCreateSessionComplete or OnJoinSessionComplete for those) but for features such as rich presence
 */
/**
* 在会话生态系统的不同环节触发的事件，这些事件代表了会话的用户可查看的状态。
* 不应将其用于在线功能（应使用 OnCreateSessionComplete 或 OnJoinSessionComplete 来实现此类功能），而应用于诸如丰富状态显示等特性。*/
UENUM(BlueprintType)
enum class ECommonSessionInformationState : uint8
{
	OutOfGame,
	Matchmaking,
	InGame
};
DECLARE_MULTICAST_DELEGATE_ThreeParams(FCommonSessionOnSessionInformationChanged, ECommonSessionInformationState /*SessionStatus*/, const FString& /*GameMode*/, const FString& /*MapName*/);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_ThreeParams(FCommonSessionOnSessionInformationChanged_Dynamic, ECommonSessionInformationState, SessionStatus, const FString&, GameMode, const FString&, MapName);

//////////////////////////////////////////////////////////////////////
// UCommonSessionSubsystem

/** 
 * Game subsystem that handles requests for hosting and joining online games.
 * One subsystem is created for each game instance and can be accessed from blueprints or C++ code.
 * If a game-specific subclass exists, this base subsystem will not be created.
 */
/**
* 负责处理在线游戏的托管和加入请求的游戏子系统。
* 每个游戏实例都会创建一个独立的子系统，并且可以从蓝图或 C++ 代码中访问该子系统。
* 如果存在特定于游戏的子类，则不会创建这个基础子系统。*/
UCLASS(MinimalAPI, BlueprintType, Config=Engine)
class UCommonSessionSubsystem : public UGameInstanceSubsystem
{
	GENERATED_BODY()

public:
	UCommonSessionSubsystem() { }

	COMMONUSER_API virtual void Initialize(FSubsystemCollectionBase& Collection) override;
	COMMONUSER_API virtual void Deinitialize() override;
	COMMONUSER_API virtual bool ShouldCreateSubsystem(UObject* Outer) const override;

	/** Creates a host session request with default options for online games, this can be modified after creation */
	/** 创建一个适用于在线游戏的主机会话请求，该请求默认包含相关选项，创建后可对其进行修改 */
	UFUNCTION(BlueprintCallable, Category = Session)
	COMMONUSER_API virtual UCommonSession_HostSessionRequest* CreateOnlineHostSessionRequest();

	/** Creates a session search object with default options to look for default online games, this can be modified after creation */
	/** 创建一个会话搜索对象，使用默认选项以查找默认的在线游戏，创建后可对其进行修改 */
	UFUNCTION(BlueprintCallable, Category = Session)
	COMMONUSER_API virtual UCommonSession_SearchSessionRequest* CreateOnlineSearchSessionRequest();

	/** Creates a new online game using the session request information, if successful this will start a hard map transfer */
	/** 根据会话请求信息创建一个新的在线游戏，如果操作成功，将启动一次硬地图传输 */
	UFUNCTION(BlueprintCallable, Category=Session)
	COMMONUSER_API virtual void HostSession(APlayerController* HostingPlayer, UCommonSession_HostSessionRequest* Request);

	/** Starts a process to look for existing sessions or create a new one if no viable sessions are found */
	/** 启动一个进程，以查找现有的会话，若未找到可用的会话则创建一个新的会话 */
	UFUNCTION(BlueprintCallable, Category=Session)
	COMMONUSER_API virtual void QuickPlaySession(APlayerController* JoiningOrHostingPlayer, UCommonSession_HostSessionRequest* Request);

	/** Starts process to join an existing session, if successful this will connect to the specified server */
	/** 启动进程以加入现有会话，如果操作成功，将连接到指定的服务器 */
	UFUNCTION(BlueprintCallable, Category=Session)
	COMMONUSER_API virtual void JoinSession(APlayerController* JoiningPlayer, UCommonSession_SearchResult* Request);

	/** Queries online system for the list of joinable sessions matching the search request */
	/** 通过在线系统查询与搜索请求相匹配的可加入会话的列表 */
	UFUNCTION(BlueprintCallable, Category=Session)
	COMMONUSER_API virtual void FindSessions(APlayerController* SearchingPlayer, UCommonSession_SearchSessionRequest* Request);

	/** Clean up any active sessions, called from cases like returning to the main menu */
	/** 清理所有正在运行的会话，此操作在诸如返回主菜单等情况下会被调用 */
	UFUNCTION(BlueprintCallable, Category=Session)
	COMMONUSER_API virtual void CleanUpSessions();

	//////////////////////////////////////////////////////////////////////
	// Events

	/** Native Delegate when a local user has accepted an invite */
	/** 当本地用户接受邀请时的原生委托函数 */
	FCommonSessionOnUserRequestedSession OnUserRequestedSessionEvent;
	/** Event broadcast when a local user has accepted an invite */
	/** 当本地用户接受邀请时会触发此事件 */
	UPROPERTY(BlueprintAssignable, Category = "Events", meta = (DisplayName = "On User Requested Session"))
	FCommonSessionOnUserRequestedSession_Dynamic K2_OnUserRequestedSessionEvent;

	/** Native Delegate when a JoinSession call has completed */
	/** 当调用 JoinSession 操作完成后所使用的原生委托 */
	FCommonSessionOnJoinSessionComplete OnJoinSessionCompleteEvent;
	/** Event broadcast when a JoinSession call has completed */
	/** 当调用 JoinSession 操作完成后会触发此事件 */
	UPROPERTY(BlueprintAssignable, Category = "Events", meta = (DisplayName = "On Join Session Complete"))
	FCommonSessionOnJoinSessionComplete_Dynamic K2_OnJoinSessionCompleteEvent;

	/** Native Delegate when a CreateSession call has completed */
	/** 当创建会话的调用完成时的原生委托 */
	FCommonSessionOnCreateSessionComplete OnCreateSessionCompleteEvent;
	/** Event broadcast when a CreateSession call has completed */
	/** 当创建会话的调用完成时触发的事件 */
	UPROPERTY(BlueprintAssignable, Category = "Events", meta = (DisplayName = "On Create Session Complete"))
	FCommonSessionOnCreateSessionComplete_Dynamic K2_OnCreateSessionCompleteEvent;

	/** Native Delegate when the presentable session information has changed */
	/** 当可展示的会话信息发生更改时的原生委托函数 */
	FCommonSessionOnSessionInformationChanged OnSessionInformationChangedEvent;
	/** Event broadcast when the presentable session information has changed */
	/** 当可展示的会话信息发生更改时触发的事件 */
	UPROPERTY(BlueprintAssignable, Category = "Events", meta = (DisplayName = "On Session Information Changed"))
	FCommonSessionOnSessionInformationChanged_Dynamic K2_OnSessionInformationChangedEvent;

	/** Native Delegate when a platform session destroy has been requested */
	/** 当请求销毁平台会话时触发的事件 */
	FCommonSessionOnDestroySessionRequested OnDestroySessionRequestedEvent;
	/** Event broadcast when a platform session destroy has been requested */
	/** 当请求销毁平台会话时触发的事件 */
	UPROPERTY(BlueprintAssignable, Category = "Events", meta = (DisplayName = "On Leave Session Requested"))
	FCommonSessionOnDestroySessionRequested_Dynamic K2_OnDestroySessionRequestedEvent;

	/** Native Delegate for modifying the connect URL prior to a client travel */
	/** 用于在客户端移动前修改连接 URL 的原生委托函数 */
	FCommonSessionOnPreClientTravel OnPreClientTravelEvent;

	// Config settings, these can overridden in child classes or config files
	// 配置设置，这些设置可以在子类或配置文件中进行更改以进行覆盖。
	
	/** Sets the default value of bUseLobbies for session search and host requests */
	/** 设置用于会话搜索和主机请求的“bUseLobbies”属性的默认值 */
	UPROPERTY(Config)
	bool bUseLobbiesDefault = true;

	/** Sets the default value of bUseLobbiesVoiceChat for session host requests */
	/** 设置用于会话主持人请求的“bUseLobbiesVoiceChat”属性的默认值 */
	UPROPERTY(Config)
	bool bUseLobbiesVoiceChatDefault = false;

	/** Enables reservation beacon flow prior to server travel when creating or joining a game session */
	/** 在创建或加入游戏会话时，在服务器移动之前启用预订信标流 */
	UPROPERTY(Config)
	bool bUseBeacons = true;

protected:
	// Functions called during the process of creating or joining a session, these can be overidden for game-specific behavior
	// 在创建或加入会话的过程中所调用的函数，这些函数可针对特定游戏的需求进行重写以实现个性化功能。

	/** Called to fill in a session request from quick play host settings, can be overridden for game-specific behavior */
	/** 被调用以根据快速游戏主机设置填充会话请求，可针对特定游戏需求进行覆盖 */
	COMMONUSER_API virtual TSharedRef<FCommonOnlineSearchSettings> CreateQuickPlaySearchSettings(UCommonSession_HostSessionRequest* Request, UCommonSession_SearchSessionRequest* QuickPlayRequest);

	/** Called when a quick play search finishes, can be overridden for game-specific behavior */
	/** 当快速播放搜索完成时会调用此函数，可针对特定游戏需求进行自定义处理 */
	COMMONUSER_API virtual void HandleQuickPlaySearchFinished(bool bSucceeded, const FText& ErrorMessage, TWeakObjectPtr<APlayerController> JoiningOrHostingPlayer, TStrongObjectPtr<UCommonSession_HostSessionRequest> HostRequest);

	/** Called when traveling to a session fails */
	/** 当前往某个会话时出现错误时会调用此函数 */
	COMMONUSER_API virtual void TravelLocalSessionFailure(UWorld* World, ETravelFailure::Type FailureType, const FString& ReasonString);

	/** Called when a new session is either created or fails to be created */
	/** 当新会话被创建或创建失败时会调用此函数 */
	COMMONUSER_API virtual void OnCreateSessionComplete(FName SessionName, bool bWasSuccessful);

	/** Called to finalize session creation */
	/** 被调用以完成会话创建的最终步骤 */
	COMMONUSER_API virtual void FinishSessionCreation(bool bWasSuccessful);

	/** Called after traveling to the new hosted session map */
	/** 在前往新的托管会话地图后被调用 */
	COMMONUSER_API virtual void HandlePostLoadMap(UWorld* World);

protected:
	// Internal functions for initializing and handling results from the online systems

	COMMONUSER_API void BindOnlineDelegates();
	COMMONUSER_API void CreateOnlineSessionInternal(ULocalPlayer* LocalPlayer, UCommonSession_HostSessionRequest* Request);
	COMMONUSER_API void FindSessionsInternal(APlayerController* SearchingPlayer, const TSharedRef<FCommonOnlineSearchSettings>& InSearchSettings);
	COMMONUSER_API void JoinSessionInternal(ULocalPlayer* LocalPlayer, UCommonSession_SearchResult* Request);
	COMMONUSER_API void InternalTravelToSession(const FName SessionName);
	COMMONUSER_API void NotifyUserRequestedSession(const FPlatformUserId& PlatformUserId, UCommonSession_SearchResult* RequestedSession, const FOnlineResultInformation& RequestedSessionResult);
	COMMONUSER_API void NotifyJoinSessionComplete(const FOnlineResultInformation& Result);
	COMMONUSER_API void NotifyCreateSessionComplete(const FOnlineResultInformation& Result);
	COMMONUSER_API void NotifySessionInformationUpdated(ECommonSessionInformationState SessionStatusStr, const FString& GameMode = FString(), const FString& MapName = FString());
	COMMONUSER_API void NotifyDestroySessionRequested(const FPlatformUserId& PlatformUserId, const FName& SessionName);
	COMMONUSER_API void SetCreateSessionError(const FText& ErrorText);

#if COMMONUSER_OSSV1
	COMMONUSER_API void BindOnlineDelegatesOSSv1();
	COMMONUSER_API void CreateOnlineSessionInternalOSSv1(ULocalPlayer* LocalPlayer, UCommonSession_HostSessionRequest* Request);
	COMMONUSER_API void FindSessionsInternalOSSv1(ULocalPlayer* LocalPlayer);
	COMMONUSER_API void JoinSessionInternalOSSv1(ULocalPlayer* LocalPlayer, UCommonSession_SearchResult* Request);
	COMMONUSER_API TSharedRef<FCommonOnlineSearchSettings> CreateQuickPlaySearchSettingsOSSv1(UCommonSession_HostSessionRequest* Request, UCommonSession_SearchSessionRequest* QuickPlayRequest);
	COMMONUSER_API void CleanUpSessionsOSSv1();

	COMMONUSER_API void HandleSessionFailure(const FUniqueNetId& NetId, ESessionFailure::Type FailureType);
	COMMONUSER_API void HandleSessionUserInviteAccepted(const bool bWasSuccessful, const int32 LocalUserIndex, FUniqueNetIdPtr AcceptingUserId, const FOnlineSessionSearchResult& SearchResult);
	COMMONUSER_API void OnStartSessionComplete(FName SessionName, bool bWasSuccessful);
	COMMONUSER_API void OnRegisterLocalPlayerComplete_CreateSession(const FUniqueNetId& PlayerId, EOnJoinSessionCompleteResult::Type Result);
	COMMONUSER_API void OnUpdateSessionComplete(FName SessionName, bool bWasSuccessful);
	COMMONUSER_API void OnEndSessionComplete(FName SessionName, bool bWasSuccessful);
	COMMONUSER_API void OnDestroySessionComplete(FName SessionName, bool bWasSuccessful);
	COMMONUSER_API void OnDestroySessionRequested(int32 LocalUserNum, FName SessionName);
	COMMONUSER_API void OnFindSessionsComplete(bool bWasSuccessful);
	COMMONUSER_API void OnJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result);
	COMMONUSER_API void OnRegisterJoiningLocalPlayerComplete(const FUniqueNetId& PlayerId, EOnJoinSessionCompleteResult::Type Result);
	COMMONUSER_API void FinishJoinSession(EOnJoinSessionCompleteResult::Type Result);

#else
	COMMONUSER_API void BindOnlineDelegatesOSSv2();
	COMMONUSER_API void CreateOnlineSessionInternalOSSv2(ULocalPlayer* LocalPlayer, UCommonSession_HostSessionRequest* Request);
	COMMONUSER_API void FindSessionsInternalOSSv2(ULocalPlayer* LocalPlayer);
	COMMONUSER_API void JoinSessionInternalOSSv2(ULocalPlayer* LocalPlayer, UCommonSession_SearchResult* Request);
	COMMONUSER_API TSharedRef<FCommonOnlineSearchSettings> CreateQuickPlaySearchSettingsOSSv2(UCommonSession_HostSessionRequest* HostRequest, UCommonSession_SearchSessionRequest* SearchRequest);
	COMMONUSER_API void CleanUpSessionsOSSv2();

	/** Process a join request originating from the online service */
	COMMONUSER_API void OnLobbyJoinRequested(const UE::Online::FUILobbyJoinRequested& EventParams);

	/** Process a SESSION join request originating from the online service */
	COMMONUSER_API void OnSessionJoinRequested(const UE::Online::FUISessionJoinRequested& EventParams);

	/** Get the local user id for a given controller */
	COMMONUSER_API UE::Online::FAccountId GetAccountId(APlayerController* PlayerController) const;
	/** Get the lobby id for a given session name */
	COMMONUSER_API UE::Online::FLobbyId GetLobbyId(const FName SessionName) const;
	/** Event handle for UI lobby join requested */
	UE::Online::FOnlineEventDelegateHandle LobbyJoinRequestedHandle;

	/** Event handle for UI lobby session requested */
	UE::Online::FOnlineEventDelegateHandle SessionJoinRequestedHandle;

#endif // COMMONUSER_OSSV1

	COMMONUSER_API void CreateHostReservationBeacon();
	COMMONUSER_API void ConnectToHostReservationBeacon();
	COMMONUSER_API void DestroyHostReservationBeacon();

protected:
	/** The travel URL that will be used after session operations are complete */
	/** 在完成会话操作后将使用的旅行相关网址 */
	FString PendingTravelURL;

	/** Most recent result information for a session creation attempt, stored here to allow storing error codes for later */
	/** 本次会话创建尝试的最新结果信息，存储于此以便日后保存错误代码 */
	FOnlineResultInformation CreateSessionResult;

	/** True if we want to cancel the session after it is created */
	/** 如果我们希望在创建会话后取消该会话，则返回 true */
	bool bWantToDestroyPendingSession = false;

	/** True if this is a dedicated server, which doesn't require a LocalPlayer to create a session */
	/** 如果这是一个专用服务器，则返回 true，因为此类服务器无需由本地玩家来创建会话 */
	bool bIsDedicatedServer = false;

	/** Settings for the current search */
	/** 当前搜索的设置 */
	TSharedPtr<FCommonOnlineSearchSettings> SearchSettings;

	/** General beacon listener for registering beacons with */
	/** 用于注册信标的一般性信标监听器 */
	UPROPERTY(Transient)
	TWeakObjectPtr<AOnlineBeaconHost> BeaconHostListener;
	/** State of the beacon host */
	/** 蜂窝设备主机的状态 */
	UPROPERTY(Transient)
	TObjectPtr<UPartyBeaconState> ReservationBeaconHostState;
	/** Beacon controlling access to this game. */
	/** 用于控制对本游戏访问权限的信标。*/
	UPROPERTY(Transient)
	TWeakObjectPtr<APartyBeaconHost> ReservationBeaconHost;
	/** Common class object for beacon communication */
	/** 用于蜂窝通信的通用类对象 */
	UPROPERTY(Transient)
	TWeakObjectPtr<APartyBeaconClient> ReservationBeaconClient;

	/** Number of teams for beacon reservation */
	/** 用于信标预订的团队数量 */
	UPROPERTY(Config)
	int32 BeaconTeamCount = 2;
	/** Size of a team for beacon reservation */
	/** 用于信标预订的团队规模 */
	UPROPERTY(Config)
	int32 BeaconTeamSize = 8;
	/** Max number of beacon reservations */
	/** 最大信标预留数量 */
	UPROPERTY(Config)
	int32 BeaconMaxReservations = 16;
};


```
### CommonUserBasicPresence
``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.
// Finished.
#pragma once

#include "Subsystems/GameInstanceSubsystem.h"
#include "CommonUserBasicPresence.generated.h"

#define UE_API COMMONUSER_API

class UCommonSessionSubsystem;
enum class ECommonSessionInformationState : uint8;

//////////////////////////////////////////////////////////////////////
// UCommonUserBasicPresence

/**
 * This subsystem plugs into the session subsystem and pushes its information to the presence interface.
 * It is not intended to be a full featured rich presence implementation, but can be used as a proof-of-concept
 * for pushing information from the session subsystem to the presence system
 *
 * 该子系统与会话子系统相连接，并将自身信息推送到状态显示界面。
 * 它并非旨在成为功能完备的丰富状态显示实现，但可以作为将会话子系统中的信息推送到状态显示系统的一个概念验证示例。
 */
UCLASS(MinimalAPI, BlueprintType, Config = Engine)
class UCommonUserBasicPresence : public UGameInstanceSubsystem
{
	GENERATED_BODY()

public:
	UE_API UCommonUserBasicPresence();


	/** Implement this for initialization of instances of the system */
	/** 请实现此功能以对系统实例进行初始化 */
	UE_API virtual void Initialize(FSubsystemCollectionBase& Collection) override;

	/** Implement this for deinitialization of instances of the system */
	/** 为系统实例的销毁操作实现此功能 */
	UE_API virtual void Deinitialize() override;

	/** False is a general purpose killswitch to stop this class from pushing presence*/
	/** 假值是一个通用的终止开关，用于阻止此类发送状态信息 */
	UPROPERTY(Config)
	bool bEnableSessionsBasedPresence = false;

	/** Maps the presence status "In-game" to a backend key*/
	/** 将“游戏内”这一在线状态映射为后端的某个键值 */
	UPROPERTY(Config)
	FString PresenceStatusInGame;

	/** Maps the presence status "Main Menu" to a backend key*/
	/** 将“主菜单”这一显示状态映射到后端的一个键值上 */
	UPROPERTY(Config)
	FString PresenceStatusMainMenu;

	/** Maps the presence status "Matchmaking" to a backend key*/
	/** 将“匹配中”这一在线状态映射为后端的标识键 */
	UPROPERTY(Config)
	FString PresenceStatusMatchmaking;

	/** Maps the "Game Mode" rich presence entry to a backend key*/
	/** 将“游戏模式”丰富的状态提示项映射到后端的键值上 */
	UPROPERTY(Config)
	FString PresenceKeyGameMode;

	/** Maps the "Map Name" rich presence entry to a backend key*/
	/** 将“地图名称”这一丰富的状态信息条目映射到后端的某个键上 */
	UPROPERTY(Config)
	FString PresenceKeyMapName;

	UE_API void OnNotifySessionInformationChanged(ECommonSessionInformationState SessionStatus, const FString& GameMode, const FString& MapName);
	UE_API FString SessionStateToBackendKey(ECommonSessionInformationState SessionStatus);
};

#undef UE_API


```

### CommonUserModule
``` cpp

// Copyright Epic Games, Inc. All Rights Reserved.
// Finished.
#pragma once

#include "Modules/ModuleInterface.h"

class FCommonUserModule : public IModuleInterface
{
public:

	/** IModuleInterface implementation */
	virtual void StartupModule() override;
	virtual void ShutdownModule() override;
};

```
### CommonUserSubsytem
``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "CommonUserTypes.h"
#include "Engine/GameViewportClient.h"
#include "GameFramework/OnlineReplStructs.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "UObject/WeakObjectPtr.h"
#include "GameplayTagContainer.h"
#include "CommonUserSubsystem.generated.h"

#if COMMONUSER_OSSV1
#include "Interfaces/OnlineIdentityInterface.h"
#include "OnlineError.h"
#else
#include "Online/OnlineAsyncOpHandle.h"
#endif

class FNativeGameplayTag;
class IOnlineSubsystem;

/** List of tags used by the common user subsystem */
/** 常用用户子系统所使用的标签列表 */
struct FCommonUserTags
{
	// General severity levels and specific system messages
	// 基本的严重程度级别及具体的系统消息

	static COMMONUSER_API FNativeGameplayTag SystemMessage_Error;	// SystemMessage.Error
	static COMMONUSER_API FNativeGameplayTag SystemMessage_Warning; // SystemMessage.Warning
	static COMMONUSER_API FNativeGameplayTag SystemMessage_Display; // SystemMessage.Display

	/** All attempts to initialize a player failed, user has to do something before trying again */
	/** 所有尝试初始化玩家的操作均未成功，用户必须先采取一些行动，然后再重新尝试 */
	static COMMONUSER_API FNativeGameplayTag SystemMessage_Error_InitializeLocalPlayerFailed; // SystemMessage.Error.InitializeLocalPlayerFailed


	// Platform trait tags, it is expected that the game instance or other system calls SetTraitTags with these tags for the appropriate platform
	// 平台特有标签，建议游戏实例或其他系统使用这些标签调用 SetTraitTags 方法，以针对相应的平台进行设置。
	
	/** This tag means it is a console platform that directly maps controller IDs to different system users. If false, the same user can have multiple controllers */
	/** 此标签表示这是一个直接将控制器 ID 映射到不同系统用户的控制台平台。若为假值，则同一用户可以拥有多个控制器 */
	static COMMONUSER_API FNativeGameplayTag Platform_Trait_RequiresStrictControllerMapping; // Platform.Trait.RequiresStrictControllerMapping

	/** This tag means the platform has a single online user and all players use index 0 */
	/** 此标签表示该平台仅有一个在线用户，所有玩家均使用索引 0 */
	static COMMONUSER_API FNativeGameplayTag Platform_Trait_SingleOnlineUser; // Platform.Trait.SingleOnlineUser
};

/** Logical representation of an individual user, one of these will exist for all initialized local players */
/** 代表单个用户的逻辑结构，所有初始化后的本地玩家都将拥有这样的结构 */
UCLASS(MinimalAPI, BlueprintType)
class UCommonUserInfo : public UObject
{
	GENERATED_BODY()

public:
	/** Primary controller input device for this user, they could also have additional secondary devices */
	/** 本用户的主控制器输入设备，他们还可以拥有其他辅助设备 */
	UPROPERTY(BlueprintReadOnly, Category = UserInfo)
	FInputDeviceId PrimaryInputDevice;

	/** Specifies the logical user on the local platform, guest users will point to the primary user */
	/** 指定本地平台上的逻辑用户，访客用户将指向主用户 */
	UPROPERTY(BlueprintReadOnly, Category = UserInfo)
	FPlatformUserId PlatformUser;
	
	/** If this user is assigned a LocalPlayer, this will match the index in the GameInstance localplayers array once it is fully created */
	/** 如果此用户被分配了一个本地玩家角色，那么在游戏实例的本地玩家数组完全创建完成后，此条件将匹配该数组中的索引值 */
	UPROPERTY(BlueprintReadOnly, Category = UserInfo)
	int32 LocalPlayerIndex = -1;

	/** If true, this user is allowed to be a guest */
	/** 如果为真，则表示此用户可被视为访客 */
	UPROPERTY(BlueprintReadOnly, Category = UserInfo)
	bool bCanBeGuest = false;

	/** If true, this is a guest user attached to primary user 0 */
	/** 如果为真，则表示这是一个与主用户 0 相关联的访客用户 */
	UPROPERTY(BlueprintReadOnly, Category = UserInfo)
	bool bIsGuest = false;

	/** Overall state of the user's initialization process */
	/** 用户初始化过程的整体状态 */
	UPROPERTY(BlueprintReadOnly, Category = UserInfo)
	ECommonUserInitializationState InitializationState = ECommonUserInitializationState::Invalid;

	/** Returns true if this user has successfully logged in */
	/** 如果当前用户已成功登录，则返回 true */
	UFUNCTION(BlueprintCallable, Category = UserInfo)
	COMMONUSER_API bool IsLoggedIn() const;

	/** Returns true if this user is in the middle of logging in */
	/** 如果当前用户正在登录过程中，则返回 true */
	UFUNCTION(BlueprintCallable, Category = UserInfo)
	COMMONUSER_API bool IsDoingLogin() const;

	/** Returns the most recently queries result for a specific privilege, will return unknown if never queried */
	/** 返回针对特定权限的最近查询结果，若从未进行过查询则返回“未知” */
	UFUNCTION(BlueprintCallable, Category = UserInfo)
	COMMONUSER_API ECommonUserPrivilegeResult GetCachedPrivilegeResult(ECommonUserPrivilege Privilege, ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;

	/** Ask about the general availability of a feature, this combines cached results with state */
	/** 询问某一功能的总体可用性情况，此操作会将缓存结果与状态信息相结合 */
	UFUNCTION(BlueprintCallable, Category = UserInfo)
	COMMONUSER_API ECommonUserAvailability GetPrivilegeAvailability(ECommonUserPrivilege Privilege) const;

	/** Returns the net id for the given context */
	/** 返回给定上下文的净标识符 */
	UFUNCTION(BlueprintCallable, Category = UserInfo)
	COMMONUSER_API FUniqueNetIdRepl GetNetId(ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;

	/** Returns the user's human readable nickname, this will return the value that was cached during UpdateCachedNetId or SetNickname */
	/** 返回用户的可读人性别化昵称，此值将为在“更新缓存网络标识”或“设置昵称”操作中缓存的值 */
	UFUNCTION(BlueprintCallable, Category = UserInfo)
	COMMONUSER_API FString GetNickname(ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;

	/** Modify the user's human readable nickname, this can be used when setting up multiple guests but will get overwritten with the platform nickname for real users */
	/** 修改用户的可读性人名，此功能可用于为多个访客设置昵称，但实际用户使用的平台昵称会覆盖此设置 */
	UFUNCTION(BlueprintCallable, Category = UserInfo)
	COMMONUSER_API void SetNickname(const FString& NewNickname, ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game);

	/** Returns an internal debug string for this player */
	/** 返回此玩家的内部调试字符串 */
	UFUNCTION(BlueprintCallable, Category = UserInfo)
	COMMONUSER_API FString GetDebugString() const;

	/** Accessor for platform user id */
	/** 平台用户 ID 访问器 */
	COMMONUSER_API FPlatformUserId GetPlatformUserId() const;

	/** Gets the platform user index for older functions expecting an integer */
	/** 获取用于较旧函数的平台用户索引（这些函数期望输入的是一个整数） */
	COMMONUSER_API int32 GetPlatformUserIndex() const;

	// Internal data, only intended to be accessed by online subsystems
	// 内部数据，仅供在线子系统访问使用

	/** Cached data for each online system */
	/** 每个在线系统的缓存数据 */
	struct FCachedData
	{
		/** Cached net id per system */
		/** 每个系统的缓存网络标识 */
		FUniqueNetIdRepl CachedNetId;

		/** Cached nickanem, updated whenever net ID might change */
		/** 缓存的用户名，每当网络标识可能发生变更时会进行更新 */
		FString CachedNickname;

		/** Cached values of various user privileges */
		/** 不同用户权限的缓存值 */
		TMap<ECommonUserPrivilege, ECommonUserPrivilegeResult> CachedPrivileges;
	};

	/** Per context cache, game will always exist but others may not */
	/** 按照不同的环境设置，游戏始终会存在，但其他内容可能不会 */
	TMap<ECommonUserOnlineContext, FCachedData> CachedDataMap;
	
	/** Looks up cached data using resolution rules */
	/** 根据解析规则查找缓存数据 */
	COMMONUSER_API FCachedData* GetCachedData(ECommonUserOnlineContext Context);
	COMMONUSER_API const FCachedData* GetCachedData(ECommonUserOnlineContext Context) const;

	/** Updates cached privilege results, will propagate to game if needed */
	/** 更新缓存的权限结果，如有需要将同步至游戏端 */
	COMMONUSER_API void UpdateCachedPrivilegeResult(ECommonUserPrivilege Privilege, ECommonUserPrivilegeResult Result, ECommonUserOnlineContext Context);

	/** Updates cached privilege results, will propagate to game if needed */
	/** 更新缓存的权限结果，如有需要将同步至游戏端 */
	COMMONUSER_API void UpdateCachedNetId(const FUniqueNetIdRepl& NewId, ECommonUserOnlineContext Context);

	/** Return the subsystem this is owned by */
	/** 返回此系统所归属的子系统 */
	COMMONUSER_API class UCommonUserSubsystem* GetSubsystem() const;
};


/** Delegates when initialization processes succeed or fail */
/** 用于记录初始化过程成功或失败的情况 */
DECLARE_DYNAMIC_MULTICAST_DELEGATE_FiveParams(FCommonUserOnInitializeCompleteMulticast, const UCommonUserInfo*, UserInfo, bool, bSuccess, FText, Error, ECommonUserPrivilege, RequestedPrivilege, ECommonUserOnlineContext, OnlineContext);
DECLARE_DYNAMIC_DELEGATE_FiveParams(FCommonUserOnInitializeComplete, const UCommonUserInfo*, UserInfo, bool, bSuccess, FText, Error, ECommonUserPrivilege, RequestedPrivilege, ECommonUserOnlineContext, OnlineContext);

/** Delegate when a system error message is sent, the game can choose to display it to the user using the type tag */
/** 当系统错误消息被发送时，该方法将被调用。游戏可以根据类型标签选择向用户显示该错误消息 */
DECLARE_DYNAMIC_MULTICAST_DELEGATE_ThreeParams(FCommonUserHandleSystemMessageDelegate, FGameplayTag, MessageType, FText, TitleText, FText, BodyText);

/** Delegate when a privilege changes, this can be bound to see if online status/etc changes during gameplay */
/** 当权限发生变更时会触发此回调函数，此回调函数可用于检测游戏过程中在线状态等的变化情况 */
DECLARE_DYNAMIC_MULTICAST_DELEGATE_FourParams(FCommonUserAvailabilityChangedDelegate, const UCommonUserInfo*, UserInfo, ECommonUserPrivilege, Privilege, ECommonUserAvailability, OldAvailability, ECommonUserAvailability, NewAvailability);


/** Parameter struct for initialize functions, this would normally be filled in by wrapper functions like async nodes */
/** 用于初始化函数的参数结构体，通常由诸如异步节点之类的封装函数来填充 */
USTRUCT(BlueprintType)
struct FCommonUserInitializeParams
{
	GENERATED_BODY()
	
	/** What local player index to use, can specify one above current if can create player is enabled */
	/** 指定使用的本地玩家索引，如果“创建玩家”功能已启用，则可以指定高于当前索引的值 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = Default)
	int32 LocalPlayerIndex = 0;

	/** Deprecated method of selecting platform user and input device */
	/** 已弃用的用于选择平台用户和输入设备的方法 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = Default)
	int32 ControllerId = -1;

	/** Primary controller input device for this user, they could also have additional secondary devices */
	/** 本用户的主控制器输入设备，他们还可以拥有其他辅助设备 */
	UPROPERTY(BlueprintReadOnly, Category = UserInfo)
	FInputDeviceId PrimaryInputDevice;

	/** Specifies the logical user on the local platform */
	/** 指定本地平台上的逻辑用户 */
	UPROPERTY(BlueprintReadOnly, Category = UserInfo)
	FPlatformUserId PlatformUser;
	
	/** Generally either CanPlay or CanPlayOnline, specifies what level of privilege is required */
	/** 通常为“CanPlay”或“CanPlayOnline”，用于指定所需的权限级别 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = Default)
	ECommonUserPrivilege RequestedPrivilege = ECommonUserPrivilege::CanPlay;

	/** What specific online context to log in to, game means to login to all relevant ones */
	/** 指定要登录的具体在线环境，游戏模式意味着登录所有相关环境 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = Default)
	ECommonUserOnlineContext OnlineContext = ECommonUserOnlineContext::Game;

	/** True if this is allowed to create a new local player for initial login */
	/** 如果允许在首次登录时创建新的本地玩家，则返回 true */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = Default)
	bool bCanCreateNewLocalPlayer = false;

	/** True if this player can be a guest user without an actual online presence */
	/** 如果此玩家可以作为访客用户存在，而无需具备实际的在线活动记录，则返回 true */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = Default)
	bool bCanUseGuestLogin = false;

	/** True if we should not show login errors, the game will be responsible for displaying them */
	/** 如果我们不应显示登录错误信息，则应设为“真”，此时游戏将负责显示这些错误信息 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = Default)
	bool bSuppressLoginErrors = false;

	/** If bound, call this dynamic delegate at completion of login */
	/** 如果已绑定，则在登录完成时调用此动态委托 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = Default)
	FCommonUserOnInitializeComplete OnUserInitializeComplete;
};

/**
 * Game subsystem that handles queries and changes to user identity and login status.
 * One subsystem is created for each game instance and can be accessed from blueprints or C++ code.
 * If a game-specific subclass exists, this base subsystem will not be created.
 *
 * 该子系统负责处理用户身份和登录状态的查询及更改操作。
 * 每个游戏实例都会创建一个独立的子系统，并且可以从蓝图或 C++ 代码中访问该子系统。
 * 如果存在特定于游戏的子类，则不会创建这个基础子系统。
 * 
 */
UCLASS(MinimalAPI, BlueprintType, Config=Engine)
class UCommonUserSubsystem : public UGameInstanceSubsystem
{
	GENERATED_BODY()

public:
	UCommonUserSubsystem() { }

	COMMONUSER_API virtual void Initialize(FSubsystemCollectionBase& Collection) override;
	COMMONUSER_API virtual void Deinitialize() override;
	COMMONUSER_API virtual bool ShouldCreateSubsystem(UObject* Outer) const override;


	/** BP delegate called when any requested initialization request completes */
	/** 当任何请求的初始化操作完成时会调用此 BP（业务处理）委托函数 */
	UPROPERTY(BlueprintAssignable, Category = CommonUser)
	FCommonUserOnInitializeCompleteMulticast OnUserInitializeComplete;

	/** BP delegate called when the system sends an error/warning message */
	/** 当系统发送错误/警告消息时调用的 BP（业务处理）委托函数 */
	UPROPERTY(BlueprintAssignable, Category = CommonUser)
	FCommonUserHandleSystemMessageDelegate OnHandleSystemMessage;

	/** BP delegate called when privilege availability changes for a user  */
	/** 当用户的权限可用性发生变化时调用的 BP（业务流程）委托函数 */
	UPROPERTY(BlueprintAssignable, Category = CommonUser)
	FCommonUserAvailabilityChangedDelegate OnUserPrivilegeChanged;

	/** Send a system message via OnHandleSystemMessage */
	/** 通过“OnHandleSystemMessage”方法发送系统消息 */
	UFUNCTION(BlueprintCallable, Category = CommonUser)
	COMMONUSER_API virtual void SendSystemMessage(FGameplayTag MessageType, FText TitleText, FText BodyText);

	/** Sets the maximum number of local players, will not destroy existing ones */
	/** 设置本地玩家的最大数量（不会删除已存在的玩家） */
	UFUNCTION(BlueprintCallable, Category = CommonUser)
	COMMONUSER_API virtual void SetMaxLocalPlayers(int32 InMaxLocalPLayers);

	/** Gets the maximum number of local players */
	/** 获取本地玩家的最大数量 */
	UFUNCTION(BlueprintPure, Category = CommonUser)
	COMMONUSER_API int32 GetMaxLocalPlayers() const;

	/** Gets the current number of local players, will always be at least 1 */
	/** 获取当前本地玩家的数量，该数量始终至少为 1 */
	UFUNCTION(BlueprintPure, Category = CommonUser)
	COMMONUSER_API int32 GetNumLocalPlayers() const;

	/** Returns the state of initializing the specified local player */
	/** 返回指定本地玩家初始化状态 */
	UFUNCTION(BlueprintPure, Category = CommonUser)
	COMMONUSER_API ECommonUserInitializationState GetLocalPlayerInitializationState(int32 LocalPlayerIndex) const;

	/** Returns the user info for a given local player index in game instance, 0 is always valid in a running game */
	/** 返回游戏实例中指定本地玩家索引对应的用户信息，运行中的游戏中的 0 索引始终有效 */
	UFUNCTION(BlueprintCallable, BlueprintPure = False, Category = CommonUser)
	COMMONUSER_API const UCommonUserInfo* GetUserInfoForLocalPlayerIndex(int32 LocalPlayerIndex) const;

	/** Deprecated, use PlatformUserId when available */
	/** 已弃用，如可用时请使用 PlatformUserId */
	UFUNCTION(BlueprintCallable, BlueprintPure = False, Category = CommonUser)
	COMMONUSER_API const UCommonUserInfo* GetUserInfoForPlatformUserIndex(int32 PlatformUserIndex) const;

	/** Returns the primary user info for a given platform user index. Can return null */
	/** 返回给定平台用户索引对应的主用户信息。可能返回 null  */
	UFUNCTION(BlueprintCallable, BlueprintPure = False, Category = CommonUser)
	COMMONUSER_API const UCommonUserInfo* GetUserInfoForPlatformUser(FPlatformUserId PlatformUser) const;

	/** Returns the user info for a unique net id. Can return null */
	/** 返回特定网络账号对应的用户信息。可能返回 null  */
	UFUNCTION(BlueprintCallable, BlueprintPure = False, Category = CommonUser)
	COMMONUSER_API const UCommonUserInfo* GetUserInfoForUniqueNetId(const FUniqueNetIdRepl& NetId) const;

	/** Deprecated, use InputDeviceId when available */
	/** 已弃用，如可用时请使用 InputDeviceId */
	UFUNCTION(BlueprintCallable, BlueprintPure = False, Category = CommonUser)
	COMMONUSER_API const UCommonUserInfo* GetUserInfoForControllerId(int32 ControllerId) const;

	/** Returns the user info for a given input device. Can return null */
	/** 返回给定输入设备的用户信息。可能返回 null  */
	UFUNCTION(BlueprintCallable, BlueprintPure = False, Category = CommonUser)
	COMMONUSER_API const UCommonUserInfo* GetUserInfoForInputDevice(FInputDeviceId InputDevice) const;

	/**
	 * Tries to start the process of creating or updating a local player, including logging in and creating a player controller.
	 * When the process has succeeded or failed, it will broadcast the OnUserInitializeComplete delegate.
	 *
	 * @param LocalPlayerIndex	Desired index of LocalPlayer in Game Instance, 0 will be primary player and 1+ for local multiplayer
	 * @param PrimaryInputDevice The physical controller that should be mapped to this user, will use the default device if invalid
	 * @param bCanUseGuestLogin	If true, this player can be a guest without a real Unique Net Id
	 *
	 * @returns true if the process was started, false if it failed before properly starting
	 */
	/**
	 * 尝试启动创建或更新本地玩家的流程，包括登录操作和创建玩家控制器。
	 * 当该流程成功或失败时，会广播“OnUserInitializeComplete”委托。
	 * @参数 LocalPlayerIndex：游戏实例中本地玩家的期望索引值，0 表示为主玩家，1 及以上表示本地多人游戏模式
	 * @参数 PrimaryInputDevice：应与该用户关联的物理控制器，无效时将使用默认设备
	 * @参数 bCanUseGuestLogin：如果为真，则此玩家可以作为访客存在，无需拥有真实的唯一网络标识符
	 * @返回值：如果进程已启动则返回 true，否则如果在正确启动之前就失败则返回 false
	 * 
	 */
	UFUNCTION(BlueprintCallable, Category = CommonUser)
	COMMONUSER_API virtual bool TryToInitializeForLocalPlay(int32 LocalPlayerIndex, FInputDeviceId PrimaryInputDevice, bool bCanUseGuestLogin);

	/**
	 * Starts the process of taking a locally logged in user and doing a full online login including account permission checks.
	 * When the process has succeeded or failed, it will broadcast the OnUserInitializeComplete delegate.
	 *
	 * @param LocalPlayerIndex	Index of existing LocalPlayer in Game Instance
	 *
	 * @returns true if the process was started, false if it failed before properly starting
	 */
	/**
	 * 开始对本地登录的用户进行完整在线登录的操作，包括账户权限检查。
	 * 当该过程成功或失败时，会广播“OnUserInitializeComplete”委托。
	 * @参数 LocalPlayerIndex：游戏实例中现有本地玩家的索引号*
	 * @返回值：如果进程已启动则返回 true，若在正确启动之前失败则返回 false
	 * 
	 */
	UFUNCTION(BlueprintCallable, Category = CommonUser)
	COMMONUSER_API virtual bool TryToLoginForOnlinePlay(int32 LocalPlayerIndex);

	/**
	 * Starts a general user login and initialization process, using the params structure to determine what to log in to.
	 * When the process has succeeded or failed, it will broadcast the OnUserInitializeComplete delegate.
	 * AsyncAction_CommonUserInitialize provides several wrapper functions for using this in an Event graph.
	 *
	 * @returns true if the process was started, false if it failed before properly starting
	 */
	/**
	 * 启动通用用户登录及初始化流程，通过参数结构来确定要登录的账号。
	 * 当该流程成功或失败时，会广播“OnUserInitializeComplete”委托。
	 * AsyncAction_CommonUserInitialize 提供了几个用于在事件图中使用此功能的封装函数。
	 * @返回值：如果进程已启动则返回 true，若在正确启动之前失败则返回 false
	 * 
	 */
	UFUNCTION(BlueprintCallable, Category = CommonUser)
	COMMONUSER_API virtual bool TryToInitializeUser(FCommonUserInitializeParams Params);

	/** 
	 * Starts the process of listening for user input for new and existing controllers and logging them.
	 * This will insert a key input handler on the active GameViewportClient and is turned off by calling again with empty key arrays.
	 *
	 * @param AnyUserKeys		Listen for these keys for any user, even the default user. Set this for an initial press start screen or empty to disable
	 * @param NewUserKeys		Listen for these keys for a new user without a player controller. Set this for splitscreen/local multiplayer or empty to disable
	 * @param Params			Params passed to TryToInitializeUser after detecting key input
	 */
	/**
	 * 开始监听用户对新控制器和现有控制器的操作输入，并将这些输入记录下来。
	 * 这会将一个按键输入处理程序插入到活动的游戏视图客户端中，并可通过再次传入空的按键数组来关闭该处理程序。
	 * @参数 AnyUserKeys：监听任何用户的这些按键，包括默认用户。将此设置为初始启动屏幕的参数值或留空可禁用该功能。
	 * @参数 NewUserKeys：监听新用户的这些按键（该用户没有玩家控制器）。将此设置为分屏/本地多人游戏的参数值或留空可禁用该功能。
	 * @参数 Params：在检测到按键输入后传递给 TryToInitializeUser 的参数值。
	 * 
	 */	
	UFUNCTION(BlueprintCallable, Category = CommonUser)
	COMMONUSER_API virtual void ListenForLoginKeyInput(TArray<FKey> AnyUserKeys, TArray<FKey> NewUserKeys, FCommonUserInitializeParams Params);

	/** Attempts to cancel an in-progress initialization attempt, this may not work on all platforms but will disable callbacks */
	/** 尝试取消正在进行的初始化操作，此操作可能并非在所有平台上都能生效，但会禁用回调函数 */
	UFUNCTION(BlueprintCallable, Category = CommonUser)
	COMMONUSER_API virtual bool CancelUserInitialization(int32 LocalPlayerIndex);

	/** Logs a player out of any online systems, and optionally destroys the player entirely if it's not the first one */
	/** 会将玩家从任何在线系统中退出，并且如果这不是首次操作，还会选择性地彻底删除该玩家的账号 */
	UFUNCTION(BlueprintCallable, Category = CommonUser)
	COMMONUSER_API virtual bool TryToLogOutUser(int32 LocalPlayerIndex, bool bDestroyPlayer = false);

	/** Resets the login and initialization state when returning to the main menu after an error */
	/** 在出现错误后返回主菜单时，重置登录和初始化状态 */
	UFUNCTION(BlueprintCallable, Category = CommonUser)
	COMMONUSER_API virtual void ResetUserState();

	/** Returns true if this this could be a real platform user with a valid identity (even if not currently logged in)  */
	/** 如果此对象代表的是拥有有效身份的真正平台用户（即使当前未登录）则返回 true  */
	COMMONUSER_API virtual bool IsRealPlatformUserIndex(int32 PlatformUserIndex) const;

	/** Returns true if this this could be a real platform user with a valid identity (even if not currently logged in) */
	/** 如果此对象可能是拥有有效身份的真正平台用户（即便当前未登录）则返回 true */
	COMMONUSER_API virtual bool IsRealPlatformUser(FPlatformUserId PlatformUser) const;

	/** Converts index to id */
	/** 将索引转换为 ID */
	COMMONUSER_API virtual FPlatformUserId GetPlatformUserIdForIndex(int32 PlatformUserIndex) const;

	/** Converts id to index */
	/** 将“id”转换为“索引” */
	COMMONUSER_API virtual int32 GetPlatformUserIndexForId(FPlatformUserId PlatformUser) const;

	/** Gets the user for an input device */
	/** 获取输入设备的用户信息 */
	COMMONUSER_API virtual FPlatformUserId GetPlatformUserIdForInputDevice(FInputDeviceId InputDevice) const;

	/** Gets a user's primary input device id */
	/** 获取用户的主输入设备 ID */
	COMMONUSER_API virtual FInputDeviceId GetPrimaryInputDeviceForPlatformUser(FPlatformUserId PlatformUser) const;

	/** Call from game code to set the cached trait tags when platform state or options changes */
	/** 从游戏代码中调用，用于在平台状态或选项发生变化时设置缓存的特征标签 */
	COMMONUSER_API virtual void SetTraitTags(const FGameplayTagContainer& InTags);

	/** Gets the current tags that affect feature avialability */
	/** 获取当前影响功能可用性的标签 */
	const FGameplayTagContainer& GetTraitTags() const { return CachedTraitTags; }

	/** Checks if a specific platform/feature tag is enabled */
	/** 检查特定的平台/功能标签是否已启用 */
	UFUNCTION(BlueprintPure, Category=CommonUser)
	bool HasTraitTag(const FGameplayTag TraitTag) const { return CachedTraitTags.HasTag(TraitTag); }

	/** Checks to see if we should display a press start/input confirmation screen at startup. Games can call this or check the trait tags directly */
	/** 检查是否应在启动时显示“开始游戏/输入确认”界面。游戏可以调用此方法，也可以直接检查特性标签来确定是否显示该界面 */
	UFUNCTION(BlueprintPure, BlueprintPure, Category=CommonUser)
	COMMONUSER_API virtual bool ShouldWaitForStartInput() const;


	// Functions for accessing low-level online system information
	// 用于访问底层在线系统信息的函数
	
#if COMMONUSER_OSSV1
	/** Returns OSS interface of specific type, will return null if there is no type */
	/** 返回特定类型的 OSS 接口，若不存在该类型则返回 null */
	COMMONUSER_API IOnlineSubsystem* GetOnlineSubsystem(ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;

	/** Returns identity interface of specific type, will return null if there is no type */
	/** 返回特定类型的标识接口，若不存在该类型则返回 null */
	COMMONUSER_API IOnlineIdentity* GetOnlineIdentity(ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;

	/** Returns human readable name of OSS system */
	/** 返回 OSS 系统的可读名称 */
	COMMONUSER_API FName GetOnlineSubsystemName(ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;

	/** Returns the current online connection status */
	/** 返回当前的在线连接状态 */
	COMMONUSER_API EOnlineServerConnectionStatus::Type GetConnectionStatus(ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;
#else
	/** Get the services provider type, or None if there isn't one. */
	/** 获取服务提供商类型，若无则返回 None 。*/
	COMMONUSER_API UE::Online::EOnlineServices GetOnlineServicesProvider(ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;
	
	/** Returns auth interface of specific type, will return null if there is no type */
	/** 返回特定类型的认证接口，若不存在该类型则返回 null */
	COMMONUSER_API UE::Online::IAuthPtr GetOnlineAuth(ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;

	/** Returns the current online connection status */
	/** 返回当前的在线连接状态 */
	COMMONUSER_API UE::Online::EOnlineServicesConnectionStatus GetConnectionStatus(ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;
#endif

	/** Returns true if we are currently connected to backend servers */
	/** 返回值为真表示我们当前已与后端服务器建立连接 */
	COMMONUSER_API bool HasOnlineConnection(ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;

	/** Returns the current login status for a player on the specified online system, only works for real platform users */
	/** 返回指定在线系统中玩家的当前登录状态，仅适用于真实平台上的用户 */
	COMMONUSER_API ELoginStatusType GetLocalUserLoginStatus(FPlatformUserId PlatformUser, ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;

	/** Returns the unique net id for a local platform user */
	/** 返回本地平台用户唯一的网络标识符 */
	COMMONUSER_API FUniqueNetIdRepl GetLocalUserNetId(FPlatformUserId PlatformUser, ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;

	/** Returns the nickname for a local platform user, this is cached in common user Info */
	/** 返回本地平台用户的昵称，该昵称已缓存在通用用户信息中 */
	COMMONUSER_API FString GetLocalUserNickname(FPlatformUserId PlatformUser, ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;

	/** Convert a user id to a debug string */
	/** 将用户 ID 转换为调试字符串 */
	COMMONUSER_API FString PlatformUserIdToString(FPlatformUserId UserId);

	/** Convert a context to a debug string */
	/** 将上下文转换为调试字符串 */
	COMMONUSER_API FString ECommonUserOnlineContextToString(ECommonUserOnlineContext Context);

	/** Returns human readable string for privilege checks */
	/** 返回用于权限检查的可读性较强的字符串 */
	COMMONUSER_API virtual FText GetPrivilegeDescription(ECommonUserPrivilege Privilege) const;
	COMMONUSER_API virtual FText GetPrivilegeResultDescription(ECommonUserPrivilegeResult Result) const;

	/** 
	 * Starts the process of login for an existing local user, will return false if callback was not scheduled 
	 * This activates the low level state machine and does not modify the initialization state on user info
	 */
	/**
	 * 开始为现有本地用户进行登录操作，若未安排回调则会返回 false
	 * 这会激活底层状态机，并且不会修改用户信息的初始化状态
	 * 
	 */
	DECLARE_DELEGATE_FiveParams(FOnLocalUserLoginCompleteDelegate, const UCommonUserInfo* /*UserInfo*/, ELoginStatusType /*NewStatus*/, FUniqueNetIdRepl /*NetId*/, const TOptional<FOnlineErrorType>& /*Error*/, ECommonUserOnlineContext /*Type*/);
	COMMONUSER_API virtual bool LoginLocalUser(const UCommonUserInfo* UserInfo, ECommonUserPrivilege RequestedPrivilege, ECommonUserOnlineContext Context, FOnLocalUserLoginCompleteDelegate OnComplete);

	/** Assign a local player to a specific local user and call callbacks as needed */
	/** 将本地玩家分配给特定的本地用户，并根据需要调用回调函数 */
	COMMONUSER_API virtual void SetLocalPlayerUserInfo(ULocalPlayer* LocalPlayer, const UCommonUserInfo* UserInfo);

	/** Resolves a context that has default behavior into a specific context */
	/** 将具有默认行为的上下文转换为特定的上下文 */
	COMMONUSER_API ECommonUserOnlineContext ResolveOnlineContext(ECommonUserOnlineContext Context) const;

	/** True if there is a separate platform and service interface */
	/** 如果存在独立的平台及服务接口，则为真 */
	COMMONUSER_API bool HasSeparatePlatformContext() const;

protected:
	/** Internal structure that caches status and pointers for each online context */
	/** 用于缓存每个在线会话的状态和指针的内部结构 */
	struct FOnlineContextCache
	{
#if COMMONUSER_OSSV1
		/** Pointer to base subsystem, will stay valid as long as game instance does */
		/** 指向基础子系统的指针，只要游戏实例存在，该指针就会一直有效 */
		IOnlineSubsystem* OnlineSubsystem = nullptr;

		/** Cached identity system, this will always be valid */
		/** 缓存的身份系统，此系统始终有效 */
		IOnlineIdentityPtr IdentityInterface;

		/** Last connection status that was passed into the HandleNetworkConnectionStatusChanged hander */
		/** 传入“处理网络连接状态变化”处理程序的最后连接状态 */
		EOnlineServerConnectionStatus::Type	CurrentConnectionStatus = EOnlineServerConnectionStatus::Normal;
#else
		/** Online services, accessor to specific services */
		/** 在线服务，特定服务的访问器 */
		UE::Online::IOnlineServicesPtr OnlineServices;
		/** Cached auth service */
		/** 缓存认证服务 */
		UE::Online::IAuthPtr AuthService;
		/** Login status changed event handle */
		/** 登录状态变化事件处理程序 */
		UE::Online::FOnlineEventDelegateHandle LoginStatusChangedHandle;
		/** Connection status changed event handle */
		/** 连接状态变化事件处理程序 */
		UE::Online::FOnlineEventDelegateHandle ConnectionStatusChangedHandle;
		/** Last connection status that was passed into the HandleNetworkConnectionStatusChanged hander */
		/** 传入“处理网络连接状态变化”处理程序的最后连接状态 */
		UE::Online::EOnlineServicesConnectionStatus CurrentConnectionStatus = UE::Online::EOnlineServicesConnectionStatus::NotConnected;
#endif

		/** Resets state, important to clear all shared ptrs */
		/** 重置状态，此操作对于清除所有共享指针至关重要 */
		void Reset()
		{
#if COMMONUSER_OSSV1
			OnlineSubsystem = nullptr;
			IdentityInterface.Reset();
			CurrentConnectionStatus = EOnlineServerConnectionStatus::Normal;
#else
			OnlineServices.Reset();
			AuthService.Reset();
			CurrentConnectionStatus = UE::Online::EOnlineServicesConnectionStatus::NotConnected;
#endif
		}
	};

	/** Internal structure to represent an in-progress login request */
	/** 用于表示正在进行的登录请求的内部结构 */
	struct FUserLoginRequest : public TSharedFromThis<FUserLoginRequest>
	{
		FUserLoginRequest(UCommonUserInfo* InUserInfo, ECommonUserPrivilege InPrivilege, ECommonUserOnlineContext InContext, FOnLocalUserLoginCompleteDelegate&& InDelegate)
			: UserInfo(TWeakObjectPtr<UCommonUserInfo>(InUserInfo))
			, DesiredPrivilege(InPrivilege)
			, DesiredContext(InContext)
			, Delegate(MoveTemp(InDelegate))
			{}

		/** Which local user is trying to log on */
		/** 正在尝试登录的本地用户是谁 */
		TWeakObjectPtr<UCommonUserInfo> UserInfo;

		/** Overall state of login request, could come from many sources */
		/** 登录请求的整体状态，可能来自多个来源 */
		ECommonUserAsyncTaskState OverallLoginState = ECommonUserAsyncTaskState::NotStarted;

		/** State of attempt to use platform auth. When started, this immediately transitions to Failed for OSSv1, as we do not support platform auth there. */
		/** 表示尝试使用平台认证的状态。当启动该操作时，对于 OSSv1 来说，此状态会立即变为“失败”，因为我们在该版本中不支持平台认证。*/
		ECommonUserAsyncTaskState TransferPlatformAuthState = ECommonUserAsyncTaskState::NotStarted;

		/** State of attempt to use AutoLogin */
		/** 使用自动登录的尝试状态 */
		ECommonUserAsyncTaskState AutoLoginState = ECommonUserAsyncTaskState::NotStarted;

		/** State of attempt to use external login UI */
		/** 使用外部登录界面的尝试状态 */
		ECommonUserAsyncTaskState LoginUIState = ECommonUserAsyncTaskState::NotStarted;

		/** Final privilege to that is requested */
		/** 最终所要求的权限 */
		ECommonUserPrivilege DesiredPrivilege = ECommonUserPrivilege::Invalid_Count;

		/** State of attempt to request the relevant privilege */
		/** 请求相关权限的尝试状态 */
		ECommonUserAsyncTaskState PrivilegeCheckState = ECommonUserAsyncTaskState::NotStarted;

		/** The final context to log into */
		/** 最终需要记录的上下文信息 */
		ECommonUserOnlineContext DesiredContext = ECommonUserOnlineContext::Invalid;

		/** What online system we are currently logging into */
		/** 我们当前登录的是哪个在线系统 */
		ECommonUserOnlineContext CurrentContext = ECommonUserOnlineContext::Invalid;

		/** User callback for completion */
		/** 用户完成操作的回调函数 */
		FOnLocalUserLoginCompleteDelegate Delegate;

		/** Most recent/relevant error to display to user */
		/** 显示给用户的最新/相关错误信息 */
		TOptional<FOnlineErrorType> Error;
	};


	/** Create a new user info object */
	/** 创建一个新的用户信息对象 */
	COMMONUSER_API virtual UCommonUserInfo* CreateLocalUserInfo(int32 LocalPlayerIndex);

	/** Deconst wrapper for const getters */
	/** 对常量获取器的解常量包装器 */
	FORCEINLINE UCommonUserInfo* ModifyInfo(const UCommonUserInfo* Info) { return const_cast<UCommonUserInfo*>(Info); }

	/** Refresh user info from OSS */
	/** 从 OSS 中刷新用户信息 */
	COMMONUSER_API virtual void RefreshLocalUserInfo(UCommonUserInfo* UserInfo);

	/** Possibly send privilege availability notification, compares current value to cached old value */
	/** 可能会发送权限可用性通知，将当前值与缓存的旧值进行比较 */
	COMMONUSER_API virtual void HandleChangedAvailability(UCommonUserInfo* UserInfo, ECommonUserPrivilege Privilege, ECommonUserAvailability OldAvailability);

	/** Updates the cached privilege on a user and notifies delegate */
	/** 更新用户的缓存权限并通知委托方 */
	COMMONUSER_API virtual void UpdateUserPrivilegeResult(UCommonUserInfo* UserInfo, ECommonUserPrivilege Privilege, ECommonUserPrivilegeResult Result, ECommonUserOnlineContext Context);

	/** Gets internal data for a type of online system, can return null for service */
	/** 获取某种在线系统的内部数据，对于某些服务可能会返回 null  */
	COMMONUSER_API const FOnlineContextCache* GetContextCache(ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game) const;
	COMMONUSER_API FOnlineContextCache* GetContextCache(ECommonUserOnlineContext Context = ECommonUserOnlineContext::Game);

	/** Create and set up system objects before delegates are bound */
	/** 在委托绑定之前创建并设置系统对象 */
	COMMONUSER_API virtual void CreateOnlineContexts();
	COMMONUSER_API virtual void DestroyOnlineContexts();

	/** Bind online delegates */
	/** 绑定在线委托 */
	COMMONUSER_API virtual void BindOnlineDelegates();

	/** Forcibly logs out and deinitializes a single user */
	/** 强制退出并解除单个用户的登录状态 */
	COMMONUSER_API virtual void LogOutLocalUser(FPlatformUserId PlatformUser);

	/** Performs the next step of a login request, which could include completing it. Returns true if it's done */
	/** 执行登录请求的下一步操作，这可能包括完成整个操作。如果已完成则返回 true */
	COMMONUSER_API virtual void ProcessLoginRequest(TSharedRef<FUserLoginRequest> Request);

	/** Call login on OSS, with platform auth from the platform OSS. Return true if AutoLogin started */
	/** 调用 OSS 中的登录功能，使用来自平台 OSS 的平台认证信息。如果自动登录已启动，则返回 true */
	COMMONUSER_API virtual bool TransferPlatformAuth(FOnlineContextCache* System, TSharedRef<FUserLoginRequest> Request, FPlatformUserId PlatformUser);

	/** Call AutoLogin on OSS. Return true if AutoLogin started. */
	/** 在 OSS 上调用自动登录功能。如果自动登录已启动，则返回 true。*/
	COMMONUSER_API virtual bool AutoLogin(FOnlineContextCache* System, TSharedRef<FUserLoginRequest> Request, FPlatformUserId PlatformUser);

	/** Call ShowLoginUI on OSS. Return true if ShowLoginUI started. */
	/** 在 OSS 上调用 ShowLoginUI 函数。如果 ShowLoginUI 已启动，则返回 true。*/
	COMMONUSER_API virtual bool ShowLoginUI(FOnlineContextCache* System, TSharedRef<FUserLoginRequest> Request, FPlatformUserId PlatformUser);

	/** Call QueryUserPrivilege on OSS. Return true if QueryUserPrivilege started. */
	/** 调用 OSS 上的“查询用户权限”函数。如果“查询用户权限”操作已开始，则返回 true。*/
	COMMONUSER_API virtual bool QueryUserPrivilege(FOnlineContextCache* System, TSharedRef<FUserLoginRequest> Request, FPlatformUserId PlatformUser);

	/** OSS-specific functions */
	/** 与 OSS 相关的函数 */
#if COMMONUSER_OSSV1
	COMMONUSER_API virtual ECommonUserPrivilege ConvertOSSPrivilege(EUserPrivileges::Type Privilege) const;
	COMMONUSER_API virtual EUserPrivileges::Type ConvertOSSPrivilege(ECommonUserPrivilege Privilege) const;
	COMMONUSER_API virtual ECommonUserPrivilegeResult ConvertOSSPrivilegeResult(EUserPrivileges::Type Privilege, uint32 Results) const;

	COMMONUSER_API void BindOnlineDelegatesOSSv1();
	COMMONUSER_API bool AutoLoginOSSv1(FOnlineContextCache* System, TSharedRef<FUserLoginRequest> Request, FPlatformUserId PlatformUser);
	COMMONUSER_API bool ShowLoginUIOSSv1(FOnlineContextCache* System, TSharedRef<FUserLoginRequest> Request, FPlatformUserId PlatformUser);
	COMMONUSER_API bool QueryUserPrivilegeOSSv1(FOnlineContextCache* System, TSharedRef<FUserLoginRequest> Request, FPlatformUserId PlatformUser);
#else
	COMMONUSER_API virtual ECommonUserPrivilege ConvertOnlineServicesPrivilege(UE::Online::EUserPrivileges Privilege) const;
	COMMONUSER_API virtual UE::Online::EUserPrivileges ConvertOnlineServicesPrivilege(ECommonUserPrivilege Privilege) const;
	COMMONUSER_API virtual ECommonUserPrivilegeResult ConvertOnlineServicesPrivilegeResult(UE::Online::EUserPrivileges Privilege, UE::Online::EPrivilegeResults Results) const;

	COMMONUSER_API void BindOnlineDelegatesOSSv2();
	COMMONUSER_API void CacheConnectionStatus(ECommonUserOnlineContext Context);
	COMMONUSER_API bool TransferPlatformAuthOSSv2(FOnlineContextCache* System, TSharedRef<FUserLoginRequest> Request, FPlatformUserId PlatformUser);
	COMMONUSER_API bool AutoLoginOSSv2(FOnlineContextCache* System, TSharedRef<FUserLoginRequest> Request, FPlatformUserId PlatformUser);
	COMMONUSER_API bool ShowLoginUIOSSv2(FOnlineContextCache* System, TSharedRef<FUserLoginRequest> Request, FPlatformUserId PlatformUser);
	COMMONUSER_API bool QueryUserPrivilegeOSSv2(FOnlineContextCache* System, TSharedRef<FUserLoginRequest> Request, FPlatformUserId PlatformUser);
	COMMONUSER_API TSharedPtr<UE::Online::FAccountInfo> GetOnlineServiceAccountInfo(UE::Online::IAuthPtr AuthService, FPlatformUserId InUserId) const;
#endif

	/** Callbacks for OSS functions */
	/** 用于 OSS 功能的回调函数 */
#if COMMONUSER_OSSV1
	COMMONUSER_API virtual void HandleIdentityLoginStatusChanged(int32 PlatformUserIndex, ELoginStatus::Type OldStatus, ELoginStatus::Type NewStatus, const FUniqueNetId& NewId, ECommonUserOnlineContext Context);
	COMMONUSER_API virtual void HandleUserLoginCompleted(int32 PlatformUserIndex, bool bWasSuccessful, const FUniqueNetId& NetId, const FString& Error, ECommonUserOnlineContext Context);
	COMMONUSER_API virtual void HandleControllerPairingChanged(int32 PlatformUserIndex, FControllerPairingChangedUserInfo PreviousUser, FControllerPairingChangedUserInfo NewUser);
	COMMONUSER_API virtual void HandleNetworkConnectionStatusChanged(const FString& ServiceName, EOnlineServerConnectionStatus::Type LastConnectionStatus, EOnlineServerConnectionStatus::Type ConnectionStatus, ECommonUserOnlineContext Context);
	COMMONUSER_API virtual void HandleOnLoginUIClosed(TSharedPtr<const FUniqueNetId> LoggedInNetId, const int PlatformUserIndex, const FOnlineError& Error, ECommonUserOnlineContext Context);
	COMMONUSER_API virtual void HandleCheckPrivilegesComplete(const FUniqueNetId& UserId, EUserPrivileges::Type Privilege, uint32 PrivilegeResults, ECommonUserPrivilege RequestedPrivilege, TWeakObjectPtr<UCommonUserInfo> CommonUserInfo, ECommonUserOnlineContext Context);
#else
	COMMONUSER_API virtual void HandleAuthLoginStatusChanged(const UE::Online::FAuthLoginStatusChanged& EventParameters, ECommonUserOnlineContext Context);
	COMMONUSER_API virtual void HandleUserLoginCompletedV2(const UE::Online::TOnlineResult<UE::Online::FAuthLogin>& Result, FPlatformUserId PlatformUser, ECommonUserOnlineContext Context);
	COMMONUSER_API virtual void HandleOnLoginUIClosedV2(const UE::Online::TOnlineResult<UE::Online::FExternalUIShowLoginUI>& Result, FPlatformUserId PlatformUser, ECommonUserOnlineContext Context);
	COMMONUSER_API virtual void HandleNetworkConnectionStatusChanged(const UE::Online::FConnectionStatusChanged& EventParameters, ECommonUserOnlineContext Context);
	COMMONUSER_API virtual void HandleCheckPrivilegesComplete(const UE::Online::TOnlineResult<UE::Online::FQueryUserPrivilege>& Result, TWeakObjectPtr<UCommonUserInfo> CommonUserInfo, UE::Online::EUserPrivileges DesiredPrivilege, ECommonUserOnlineContext Context);
#endif

	/**
	 * Callback for when an input device (i.e. a gamepad) has been connected or disconnected.
	 * 当输入设备（例如游戏手柄）被连接或断开时的回调函数。
	 * 
	 */
	COMMONUSER_API virtual void HandleInputDeviceConnectionChanged(EInputDeviceConnectionState NewConnectionState, FPlatformUserId PlatformUserId, FInputDeviceId InputDeviceId);

	COMMONUSER_API virtual void HandleLoginForUserInitialize(const UCommonUserInfo* UserInfo, ELoginStatusType NewStatus, FUniqueNetIdRepl NetId, const TOptional<FOnlineErrorType>& Error, ECommonUserOnlineContext Context, FCommonUserInitializeParams Params);
	COMMONUSER_API virtual void HandleUserInitializeFailed(FCommonUserInitializeParams Params, FText Error);
	COMMONUSER_API virtual void HandleUserInitializeSucceeded(FCommonUserInitializeParams Params);

	/** Callback for handling press start/login logic */
	/** 用于处理启动/登录逻辑的回调函数 */
	COMMONUSER_API virtual bool OverrideInputKeyForLogin(FInputKeyEventArgs& EventArgs);


	/** Previous override handler, will restore on cancel */
	/** 上次的覆盖处理程序，取消操作时将恢复该处理程序 */
	FOverrideInputKeyHandler WrappedInputKeyHandler;

	/** List of keys to listen for from any user */
	/** 从任何用户那里监听的键列表 */
	TArray<FKey> LoginKeysForAnyUser;

	/** List of keys to listen for a new unmapped user */
	/** 用于监听新未映射用户的键列表 */
	TArray<FKey> LoginKeysForNewUser;

	/** Params to use for a key-triggered login */
	/** 用于触发基于密钥的登录的参数 */
	FCommonUserInitializeParams ParamsForLoginKey;

	/** Maximum number of local players */
	/** 最多可容纳的本地玩家数量 */
	int32 MaxNumberOfLocalPlayers = 0;
	
	/** True if this is a dedicated server, which doesn't require a LocalPlayer */
	/** 如果这是一个专用服务器（即不需要本地玩家的服务器），则返回 true */
	bool bIsDedicatedServer = false;

	/** List of current in progress login requests */
	/** 当前正在进行的登录请求列表 */
	TArray<TSharedRef<FUserLoginRequest>> ActiveLoginRequests;

	/** Information about each local user, from local player index to user */
	/** 每个本地用户的相关信息，从本地玩家索引到用户本身 */
	UPROPERTY()
	TMap<int32, TObjectPtr<UCommonUserInfo>> LocalUserInfos;
	
	/** Cached platform/mode trait tags */
	/** 缓存的平台/模式特征标签 */
	FGameplayTagContainer CachedTraitTags;

	/** Do not access this outside of initialization */
	/** 在初始化阶段之外请勿访问此内容 */
	FOnlineContextCache* DefaultContextInternal = nullptr;
	FOnlineContextCache* ServiceContextInternal = nullptr;
	FOnlineContextCache* PlatformContextInternal = nullptr;

	friend UCommonUserInfo;
};

```
### CommonUserTypes
``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.
// Finished.
#pragma once


#if COMMONUSER_OSSV1

// Online Subsystem (OSS v1) includes and forward declares
// 在线子系统（OSS 版本 1）包含并预先声明了
#include "OnlineSubsystemTypes.h"
class IOnlineSubsystem;
struct FOnlineError;
using FOnlineErrorType = FOnlineError;
using ELoginStatusType = ELoginStatus::Type;

#else

// Online Services (OSS v2) includes and forward declares
// 在线服务（OSS v2）包含并预先声明了
#include "Online/Connectivity.h"
#include "Online/OnlineError.h"
namespace UE::Online
{
	enum class ELoginStatus : uint8;
	enum class EPrivilegeResults : uint32;
	enum class EUserPrivileges : uint8;
	using IAuthPtr = TSharedPtr<class IAuth>;
	using IOnlineServicesPtr = TSharedPtr<class IOnlineServices>;
	template <typename OpType>
	class TOnlineResult;
	struct FAuthLogin;
	struct FConnectionStatusChanged;
	struct FExternalUIShowLoginUI;
	struct FAuthLoginStatusChanged;
	struct FQueryUserPrivilege;
	struct FAccountInfo;
}
using FOnlineErrorType = UE::Online::FOnlineError;
using ELoginStatusType = UE::Online::ELoginStatus;

#endif

#include "CommonUserTypes.generated.h"


/** Enum specifying where and how to run online queries */
/** 用于指定在线查询的运行位置及方式的枚举类型 */
UENUM(BlueprintType)
enum class ECommonUserOnlineContext : uint8
{
	/** Called from game code, this uses the default system but with special handling that could merge results from multiple contexts */
	/** 由游戏代码调用，此函数使用默认系统，但会进行特殊处理，以便将来自多个环境的结果进行合并 */
	Game,

	/** The default engine online system, this will always exist and will be the same as either Service or Platform */
	/** 默认的在线引擎系统，此系统始终存在，并且与“服务”或“平台”系统相同 */
	Default,
	
	/** Explicitly ask for the external service, which may not exist */
	/** 明确要求使用外部服务，但该服务可能并不存在 */
	Service,

	/** Looks for external service first, then falls back to default */
	/** 首先查找外部服务，若未找到则回退至默认设置 */
	ServiceOrDefault,
	
	/** Explicitly ask for the platform system, which may not exist */
	/** 明确要求获取平台系统信息，但该系统可能并不存在 */
	Platform,

	/** Looks for platform system first, then falls back to default */
	/** 首先查找平台系统，若未找到则转而使用默认设置 */
	PlatformOrDefault,

	/** Invalid system */
	/** 无效系统 */

	Invalid
};

/** Enum describing the state of initialization for a specific user */
/** 描述特定用户初始化状态的枚举 */
UENUM(BlueprintType)
enum class ECommonUserInitializationState : uint8
{
	/** User has not started login process */
	/** 用户尚未开始登录流程 */
	Unknown,

	/** Player is in the process of acquiring a user id with local login */
	/** 玩家正在通过本地登录的方式获取用户 ID */
	DoingInitialLogin,

	/** Player is performing the network login, they have already logged in locally */
	/** 玩家正在进行网络登录操作，他们已经在本地完成了登录 */
	DoingNetworkLogin,

	/** Player failed to log in at all */
	/** 玩家完全无法登录 */
	FailedtoLogin,

	
	/** Player is logged in and has access to online functionality */
	/** 玩家已登录并拥有使用在线功能的权限 */
	LoggedInOnline,

	/** Player is logged in locally (either guest or real user), but cannot perform online actions */
	/** 玩家已在本地登录（可能是访客用户，也可能是真实用户），但无法执行在线操作 */
	LoggedInLocalOnly,


	/** Invalid state or user */
	/** 无效状态或用户 */
	Invalid,
};

/** Enum specifying different privileges and capabilities available to a user */
/** 定义了用户可享有的不同权限和功能的枚举 */
UENUM(BlueprintType)
enum class ECommonUserPrivilege : uint8
{
	/** Whether the user can play at all, online or offline */
	/** 用户是否能够进行游戏操作（无论是在线还是离线状态） */
	CanPlay,

	/** Whether the user can play in online modes */
	/** 用户是否能够在在线模式下进行游戏 */
	CanPlayOnline,

	/** Whether the user can use text chat */
	/** 用户是否可以使用文字聊天 */
	CanCommunicateViaTextOnline,

	/** Whether the user can use voice chat */
	/** 用户是否可以使用语音聊天功能 */
	CanCommunicateViaVoiceOnline,

	/** Whether the user can access content generated by other users */
	/** 用户是否能够访问其他用户生成的内容 */
	CanUseUserGeneratedContent,

	/** Whether the user can ever participate in cross-play */
	/** 用户是否能够参与跨平台游戏活动 */
	CanUseCrossPlay,

	/** Invalid privilege (also the count of valid ones) */
	/** 无效权限（以及有效权限的数量） */
	Invalid_Count					UMETA(Hidden)
};

/** Enum specifying the general availability of a feature or privilege, this combines information from multiple sources */
/** 该枚举用于指定一项功能或权限的总体可用性，它整合了来自多个来源的信息 */
UENUM(BlueprintType)
enum class ECommonUserAvailability : uint8
{
	/** State is completely unknown and needs to be queried */
	/** 状态完全未知，需要进行查询 */
	Unknown,

	/** This feature is fully available for use right now */
	/** 该功能目前可完全正常使用 */
	NowAvailable,

	/** This might be available after the completion of normal login procedures */
	/** 这项功能可能在完成常规登录流程后才会可用 */
	PossiblyAvailable,

	/** This feature is not available now because of something like network connectivity but may be available in the future */
	/** 目前此功能无法使用，可能是由于网络连接等原因所致，但未来可能会恢复可用。 */
	CurrentlyUnavailable,

	/** This feature will never be available for the rest of this session due to hard account or platform restrictions */
	/** 由于账户或平台的严格限制，此功能在本次会话结束后将无法再使用 */
	AlwaysUnavailable,

	/** Invalid feature */
	/** 无效特性 */
	Invalid,
};

/** Enum giving specific reasons why a user may or may not use a certain privilege */
/** 一个枚举类型，用于说明用户可能或可能不会使用某项特权的具体原因 */
UENUM(BlueprintType)
enum class ECommonUserPrivilegeResult : uint8
{
	/** State is unknown and needs to be queried */
	/** 状态未知，需要进行查询 */
	Unknown,

	/** This privilege is fully available for use */
	/** 此权限可完全自由使用 */
	Available,

	/** User has not fully logged in */
	/** 用户尚未完成登录 */
	UserNotLoggedIn,

	/** User does not own the game or content */
	/** 用户并不拥有该游戏或相关内容 */
	LicenseInvalid,

	/** The game needs to be updated or patched before this will be available */
	/** 在此功能可用之前，游戏需要进行更新或补丁操作 */
	VersionOutdated,

	/** No network connection, this may be resolved by reconnecting */
	/** 没有网络连接，这种情况可能通过重新连接来解决 */
	NetworkConnectionUnavailable,

	/** Parental control failure */
	/** 父母控制失败 */
	AgeRestricted,

	/** Account does not have a required subscription or account type */
	/** 账户未拥有必需的订阅服务或账户类型 */
	AccountTypeRestricted,

	/** Another account/user restriction such as being banned by the service */
	/** 另一种账户/用户限制，例如因违反服务规定而被封禁 */
	AccountUseRestricted,

	/** Other platform-specific failure */
	/** 其他平台特定的故障 */
	PlatformFailure,
};

/** Used to track the progress of different asynchronous operations */
/** 用于记录不同异步操作的执行进度 */
enum class ECommonUserAsyncTaskState : uint8
{
	/** The task has not been started */
	/** 该任务尚未开始执行 */
	NotStarted,
	/** The task is currently being processed */
	/** 当前任务正在处理中 */
	InProgress,
	/** The task has completed successfully */
	/** 该任务已成功完成 */
	Done,
	/** The task failed to complete */
	/** 该任务未能完成 */
	Failed
};

/** Detailed information about the online error. Effectively a wrapper for FOnlineError. */
/** 关于在线错误的详细信息。实际上是对 FOnlineError 的一种封装。*/
USTRUCT(BlueprintType)
struct FOnlineResultInformation
{
	GENERATED_BODY()

	/** Whether the operation was successful or not. If it was successful, the error fields of this struct will not contain extra information. */
	/** 表示操作是否成功。如果操作成功，则此结构体中的错误字段将不包含任何额外信息。*/
	UPROPERTY(BlueprintReadOnly)
	bool bWasSuccessful = true;

	/** The unique error id. Can be used to compare against specific handled errors. */
	/** 该唯一的错误标识符。可用于与已处理的特定错误进行比较。*/
	UPROPERTY(BlueprintReadOnly)
	FString ErrorId;

	/** Error text to display to the user. */
	/** 显示给用户的错误信息文本。*/
	UPROPERTY(BlueprintReadOnly)
	FText ErrorText;

	/**
	 * Initialize this from an FOnlineErrorType
	 * @param InOnlineError the online error to initialize from
	 */
	/**
	 * 从 FOnlineErrorType 中初始化此值
	 * @参数 InOnlineError 用于初始化的在线错误信息
	 * 
	 */
	void COMMONUSER_API FromOnlineError(const FOnlineErrorType& InOnlineError);
};

```
## 总结
本节列举了CommonUser插件中的代码.不需要深究.以黑盒的方式掌握蓝图使用即可!
因为这东西更新很快,还没有定论!属于未来可期的类型.而且一般我们也不会写这块的代码,它是属于开发环境架构相关的内容了.