# UE5_Lyra学习指南_026_PlayerController

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_026\_PlayerController](#ue5_lyra学习指南_026_playercontroller)
	- [概述](#概述)
	- [指定相机管理类和作弊器的类](#指定相机管理类和作弊器的类)
	- [注册HTTPServer](#注册httpserver)
	- [旁观者角度同步](#旁观者角度同步)
		- [生命周期](#生命周期)
		- [Tick从主端获取,客端同步](#tick从主端获取客端同步)
	- [Pawn的作弊时机](#pawn的作弊时机)
	- [自动运行](#自动运行)
	- [确保ASC的时序正常](#确保asc的时序正常)
		- [解除时机](#解除时机)
		- [专属服务器下的时序问题](#专属服务器下的时序问题)
		- [ASC处理输入](#asc处理输入)
	- [设置队伍的绑定](#设置队伍的绑定)
	- [力反馈设置](#力反馈设置)
		- [命令行变量](#命令行变量)
		- [绑定游戏时设置变更](#绑定游戏时设置变更)
		- [重写力反馈的方法](#重写力反馈的方法)
	- [相机穿透](#相机穿透)
		- [CameraMode调用位置](#cameramode调用位置)
		- [接口定义](#接口定义)
		- [执行时机](#执行时机)
	- [回放](#回放)
		- [是否录制回放](#是否录制回放)
		- [尝试开始录制](#尝试开始录制)
	- [更新回放播放时的ViewTarget](#更新回放播放时的viewtarget)
		- [监听回放状态类的变化](#监听回放状态类的变化)
		- [更新视图](#更新视图)
	- [代码](#代码)
		- [LyraPlayerController](#lyraplayercontroller)
		- [LyraReplayPlayerController](#lyrareplayplayercontroller)
	- [总结](#总结)



## 概述
LyraPlayerController是一个很重要的类.
它嵌套了很多业务逻辑进行转发.
比如GAS,回放,相机穿透,队伍,作弊,响应输入,手柄力反馈等等.
本节还是先把大部分逻辑过一遍.对应的板块逻辑在后面再详细讲解.一个板块就会抽象出一堆的功能类.

## 指定相机管理类和作弊器的类
```cpp
ALyraPlayerController::ALyraPlayerController(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
{
	// 指定相机管理类
	PlayerCameraManagerClass = ALyraPlayerCameraManager::StaticClass();

#if USING_CHEAT_MANAGER
	// 指定作弊器的类
	CheatClass = ULyraCheatManager::StaticClass();
#endif // #if USING_CHEAT_MANAGER
}
```
## 注册HTTPServer
这部分类似的代码,我们在上套课程通过详细探讨如何使用独立程序编写登录注册服务器.
通过注册HttpServer可以接受网络的指令从而执行测试.
``` cpp
void ALyraPlayerController::BeginPlay()
{
	Super::BeginPlay();
	
	#if WITH_RPC_REGISTRY
	// 开启所有的监听
	FHttpServerModule::Get().StartAllListeners();

	// 监听的端口
	int32 RpcPort = 0;

	// 从命令行读取监听端口
	if (FParse::Value(FCommandLine::Get(), TEXT("rpcport="), RpcPort))
	{
		ULyraGameplayRpcRegistrationComponent* ObjectInstance = ULyraGameplayRpcRegistrationComponent::GetInstance();
		if (ObjectInstance && ObjectInstance->IsValidLowLevel())
		{
			// 注册总是调用
			ObjectInstance->RegisterAlwaysOnHttpCallbacks();

			// 注册比赛中调用
			ObjectInstance->RegisterInMatchHttpCallbacks();
		}
	}
	#endif
	SetActorHiddenInGame(false);
}
```
这里是绑定的路由地址.
``` cpp
void ULyraGameplayRpcRegistrationComponent::RegisterAlwaysOnHttpCallbacks()
{
	Super::RegisterAlwaysOnHttpCallbacks();	
	const FExternalRpcArgumentDesc CommandDesc(TEXT("command"), TEXT("string"), TEXT("The command to tell the executable to run."));

	RegisterHttpCallback(FName(TEXT("CheatCommand")),
		FHttpPath("/core/cheatcommand"),
		EHttpServerRequestVerbs::VERB_POST,
		FHttpRequestHandler::CreateUObject(this, &ThisClass::HttpExecuteCheatCommand),
		true,
		TEXT("Cheats"),
		TEXT("raw"),
		{ CommandDesc });
}

void ULyraGameplayRpcRegistrationComponent::RegisterInMatchHttpCallbacks()
{
	RegisterHttpCallback(FName(TEXT("GetPlayerStatus")),
		FHttpPath("/player/status"),
		EHttpServerRequestVerbs::VERB_GET,
		FHttpRequestHandler::CreateUObject(this, &ThisClass::HttpGetPlayerVitalsCommand),
		true);

	RegisterHttpCallback(FName(TEXT("PlayerFireOnce")),
		FHttpPath("/player/status"),
		EHttpServerRequestVerbs::VERB_POST,
		FHttpRequestHandler::CreateUObject(this, &ThisClass::HttpFireOnceCommand),
		true);
	
}

```
这里我们简单看一个 执行的过程.后续会专门有一节来讲解这个功能的实操.不在这里赘述.
	LPC->ConsoleCommand(*CheatCommand, true);
``` cpp
bool ULyraGameplayRpcRegistrationComponent::HttpExecuteCheatCommand(const FHttpServerRequest& Request, const FHttpResultCallback& OnComplete)
{
	TSharedPtr<FJsonObject> BodyObject = GetJsonObjectFromRequestBody(Request.Body);

	if (!BodyObject.IsValid())
	{
		TUniquePtr<FHttpServerResponse>Response = CreateSimpleResponse(false, TEXT("Invalid body object"));
		OnComplete(MoveTemp(Response));
		return true;
	}
	if (BodyObject->GetStringField(TEXT("command")).IsEmpty())
	{
		TUniquePtr<FHttpServerResponse>Response = CreateSimpleResponse(false, TEXT("command not found in json body"));
		OnComplete(MoveTemp(Response));
		return true;
	}
	ALyraPlayerController* LPC = GetPlayerController();
	if (!LPC)
	{
		TUniquePtr<FHttpServerResponse>Response = CreateSimpleResponse(false, TEXT("player controller not found"));
		OnComplete(MoveTemp(Response));
		return true;
	}
	FString CheatCommand = FString::Printf(TEXT("%s"), *BodyObject->GetStringField(TEXT("command")));
	LPC->ConsoleCommand(*CheatCommand, true);

	TUniquePtr<FHttpServerResponse>Response = CreateSimpleResponse(true);
	OnComplete(MoveTemp(Response));
	return true;
}
```
## 旁观者角度同步
### 生命周期
此处需要比对PlayerController的源码

``` cpp
	/** Used to replicate the view rotation of targets not owned/possessed by this PlayerController. */ 
	/** 用于复制不属于/未被本玩家控制器所拥有的目标的视角旋转效果。*/
	UPROPERTY(replicated)
	FRotator TargetViewRotation; 

void APlayerController::GetLifetimeReplicatedProps(TArray< FLifetimeProperty > & OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);
	
	FDoRepLifetimeParams Params;
	Params.bIsPushBased = UE::Gameplay::CVars::bIsPlayerControllerPushBased;
	Params.Condition = COND_OwnerOnly;
	// These used to only replicate if PlayerCameraManager->GetViewTargetPawn() != GetPawn()
	// But, since they also don't update unless that condition is true, these values won't change, thus won't send
	// This is a little less efficient, but fits into the new condition system well, and shouldn't really add much overhead
	// 这些原本只有在 PlayerCameraManager->GetViewTargetPawn() 不等于 GetPawn() 的情况下才会进行复制
	// 但因为除非上述条件成立，否则这些值不会更新，所以它们不会发生变化，也就不会发送
	// 这的效率稍低一些，但与新的条件系统相契合得很好，而且实际上也不会增加太多开销
	DOREPLIFETIME_WITH_PARAMS_FAST(APlayerController, TargetViewRotation, Params);

	// Replicate SpawnLocation for remote spectators
	// 为远程观众复制“起始位置”设置
	DOREPLIFETIME_WITH_PARAMS_FAST(APlayerController, SpawnLocation, Params);
}


```


``` cpp
void ALyraPlayerController::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	// Disable replicating the PC target view as it doesn't work well for replays or client-side spectating.
	// The engine TargetViewRotation is only set in APlayerController::TickActor if the server knows ahead of time that 
	// a specific pawn is being spectated and it only replicates down for COND_OwnerOnly.
	// In client-saved replays, COND_OwnerOnly is never true and the target pawn is not always known at the time of recording.
	// To support client-saved replays, the replication of this was moved to ReplicatedViewRotation and updated in PlayerTick.

	// 禁用复制 PC 目标视图，因为这对于回放或客户端旁观模式来说效果不佳。
	// 在 APlayerController::TickActor 中，只有当服务器事先知道某个特定角色正在被旁观时，才会设置引擎的 TargetViewRotation 属性，并且只有在 COND_OwnerOnly 条件下才会进行复制。
	// 在客户端保存的回放中，COND_OwnerOnly 从不为真，而且在录制时也不总是知道目标角色的具体信息。
	// 为了支持客户端保存的回放，此属性的复制已移至 ReplicatedViewRotation，并在 PlayerTick 中进行更新。
	
	DISABLE_REPLICATED_PROPERTY(APlayerController, TargetViewRotation);
}
```
### Tick从主端获取,客端同步

因为玩家控制器只存在于服务器和本地机器上,所以这里需要从服务器或者本机上去设置,

``` cpp
	/**
	 * Processes player input (immediately after PlayerInput gets ticked) and calls UpdateRotation().
	 * PlayerTick is only called if the PlayerController has a PlayerInput object. Therefore, it will only be called for locally controlled PlayerControllers.
	 * 对玩家输入进行处理（在“PlayerInput”完成计时后立即执行）并调用“UpdateRotation()”函数。
	 * “PlayerTick”函数仅在“PlayerController”拥有“PlayerInput”对象时才会被调用。因此，它只会针对本地控制的“PlayerController”进行调用
	 */
	ENGINE_API virtual void PlayerTick(float DeltaTime);

```

这里调用了 LyraPlayerState的设置同步观看的角度.从而实现了都同步到.
``` cpp
void ALyraPlayerController::PlayerTick(float DeltaTime)
{
	// ...
ALyraPlayerState* LyraPlayerState = GetLyraPlayerState();

	if (PlayerCameraManager && LyraPlayerState)
	{
		APawn* TargetPawn = PlayerCameraManager->GetViewTargetPawn();

		if (TargetPawn)
		{
			// Update view rotation on the server so it replicates
			// 在服务器上更新视图旋转角度，以便实现同步复制
			if (HasAuthority() || TargetPawn->IsLocallyControlled())
			{
				LyraPlayerState->SetReplicatedViewRotation(TargetPawn->GetViewRotation());
			}

			// Update the target view rotation if the pawn isn't locally controlled
			// 如果兵不是本地控制的，则更新目标视图的旋转角度
			if (!TargetPawn->IsLocallyControlled())
			{
				LyraPlayerState = TargetPawn->GetPlayerState<ALyraPlayerState>();
				if (LyraPlayerState)
				{
					// Get it from the spectated pawn's player state, which may not be the same as the PC's playerstate
					// 从旁观者棋子的玩家状态中获取该信息，而该玩家状态可能与电脑玩家的状态不同。
					TargetViewRotation = LyraPlayerState->GetReplicatedViewRotation();
				}
			}
		}



}

```

## Pawn的作弊时机
这里针对的是对于一个Pawn被控制的时候 执行对应的作弊指令
``` cpp
void ALyraPlayerController::OnPossess(APawn* InPawn)
{
	Super::OnPossess(InPawn);

#if WITH_SERVER_CODE && WITH_EDITOR
	if (GIsEditor && (InPawn != nullptr) && (GetPawn() == InPawn))
	{
		for (const FLyraCheatToRun& CheatRow : GetDefault<ULyraDeveloperSettings>()->CheatsToRun)
		{
			if (CheatRow.Phase == ECheatExecutionTime::OnPlayerPawnPossession)
			{
				ConsoleCommand(CheatRow.Cheat, /*bWriteToLog=*/ true);
			}
		}
	}
#endif
	// 关闭自动运行
	SetIsAutoRunning(false);
}
```
## 自动运行
通过ASC的Tag确认是否需要自动运行.
``` cpp
void ALyraPlayerController::SetIsAutoRunning(const bool bEnabled)
{
	const bool bIsAutoRunning = GetIsAutoRunning();
	if (bEnabled != bIsAutoRunning)
	{
		if (!bEnabled)
		{
			OnEndAutoRun();
		}
		else
		{
			OnStartAutoRun();
		}
	}
}

bool ALyraPlayerController::GetIsAutoRunning() const
{
	bool bIsAutoRunning = false;
	if (const ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		bIsAutoRunning = LyraASC->GetTagCount(LyraGameplayTags::Status_AutoRunning) > 0;
	}
	return bIsAutoRunning;
}

void ALyraPlayerController::OnStartAutoRun()
{
	if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		LyraASC->SetLooseGameplayTagCount(LyraGameplayTags::Status_AutoRunning, 1);
		K2_OnStartAutoRun();
	}	
}

void ALyraPlayerController::OnEndAutoRun()
{
	if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		LyraASC->SetLooseGameplayTagCount(LyraGameplayTags::Status_AutoRunning, 0);
		K2_OnEndAutoRun();
	}
}Player

```
如果需要自动运行,那么就在PlayerTick里面模拟输入
``` cpp
void ALyraPlayerController::PlayerTick(float DeltaTime)
{
	Super::PlayerTick(DeltaTime);

	// If we are auto running then add some player input
	// 如果我们处于自动运行模式，则添加一些玩家操作输入
	if (GetIsAutoRunning())
	{
		if (APawn* CurrentPawn = GetPawn())
		{
			const FRotator MovementRotation(0.0f, GetControlRotation().Yaw, 0.0f);
			const FVector MovementDirection = MovementRotation.RotateVector(FVector::ForwardVector);
			CurrentPawn->AddMovementInput(MovementDirection, 1.0f);	
		}
	}
}
```
## 确保ASC的时序正常
### 解除时机
当失去控制一个Pawn时,确保我们的Avatar与其对应,并置为空.
``` cpp
void ALyraPlayerController::OnUnPossess()
{
	// Make sure the pawn that is being unpossessed doesn't remain our ASC's avatar actor
	// 确保被解除控制的棋子不会仍然是我们 ASC 的角色扮演者角色。
	if (APawn* PawnBeingUnpossessed = GetPawn())
	{
		if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(PlayerState))
		{
			if (ASC->GetAvatarActor() == PawnBeingUnpossessed)
			{
				ASC->SetAvatarActor(nullptr);
			}
		}
	}

	Super::OnUnPossess();
}
```
### 专属服务器下的时序问题
``` cpp
void ALyraPlayerController::OnRep_PlayerState()
{
	Super::OnRep_PlayerState();
	BroadcastOnPlayerStateChanged();

	// When we're a client connected to a remote server, the player controller may replicate later than the PlayerState and AbilitySystemComponent.
	// However, TryActivateAbilitiesOnSpawn depends on the player controller being replicated in order to check whether on-spawn abilities should
	// execute locally. Therefore once the PlayerController exists and has resolved the PlayerState, try once again to activate on-spawn abilities.
	// On other net modes the PlayerController will never replicate late, so LyraASC's own TryActivateAbilitiesOnSpawn calls will succeed. The handling 
	// here is only for when the PlayerState and ASC replicated before the PC and incorrectly thought the abilities were not for the local player.

	// 当我们作为客户端连接到远程服务器时，玩家控制器的同步可能会晚于玩家状态和能力系统组件的同步。
	// 但是，TryActivateAbilitiesOnSpawn 函数依赖于玩家控制器已进行同步，以便检查是否应在游戏启动时本地执行初始能力。因此，一旦玩家控制器存在并已解析玩家状态，就再次尝试激活初始能力。
	// 在其他网络模式下，玩家控制器永远不会同步较晚，所以 LyraASC 的自身 TryActivateAbilitiesOnSpawn 调用将成功。此处的处理方式仅适用于当玩家状态和 ASC 在玩家控制器同步之前已进行同步，并且错误地认为这些能力并非针对本地玩家时的情况。
	
	if (GetWorld()->IsNetMode(NM_Client))
	{
		if (ALyraPlayerState* LyraPS = GetPlayerState<ALyraPlayerState>())
		{
			if (ULyraAbilitySystemComponent* LyraASC = LyraPS->GetLyraAbilitySystemComponent())
			{
				LyraASC->RefreshAbilityActorInfo();
				LyraASC->TryActivateAbilitiesOnSpawn();
			}
		}
	}
}

```
### ASC处理输入
因为输入涉及到一个Tag的双向绑定 增强型输入系统 和GAS系统.
这里进行转发让ASC去处理输入的句柄
``` cpp
void ALyraPlayerController::PostProcessInput(const float DeltaTime, const bool bGamePaused)
{
	if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		LyraASC->ProcessAbilityInput(DeltaTime, bGamePaused);
	}

	Super::PostProcessInput(DeltaTime, bGamePaused);
}

```
## 设置队伍的绑定
一旦PlayerState发生了变化,就去重新绑定,并触发队伍变更的判定!
``` cpp
private:

	// 广播玩家状态发生了变化,需要重新绑定队伍代理
	void BroadcastOnPlayerStateChanged();
```

``` cpp
void ALyraPlayerController::BroadcastOnPlayerStateChanged()
{
	// 先调用拓展接口
	OnPlayerStateChanged();

	// Unbind from the old player state, if any
	// 从旧的玩家状态中解除绑定（如果存在的话）
	FGenericTeamId OldTeamID = FGenericTeamId::NoTeam;
	if (LastSeenPlayerState != nullptr)
	{
		if (ILyraTeamAgentInterface* PlayerStateTeamInterface = Cast<ILyraTeamAgentInterface>(LastSeenPlayerState))
		{
			OldTeamID = PlayerStateTeamInterface->GetGenericTeamId();
			PlayerStateTeamInterface->GetTeamChangedDelegateChecked().RemoveAll(this);
		}
	}

	// Bind to the new player state, if any
	// 将其绑定到新的玩家状态（如果有新的玩家状态的话）
	FGenericTeamId NewTeamID = FGenericTeamId::NoTeam;
	if (PlayerState != nullptr)
	{
		if (ILyraTeamAgentInterface* PlayerStateTeamInterface = Cast<ILyraTeamAgentInterface>(PlayerState))
		{
			NewTeamID = PlayerStateTeamInterface->GetGenericTeamId();
			PlayerStateTeamInterface->GetTeamChangedDelegateChecked().AddDynamic(this, &ThisClass::OnPlayerStateChangedTeam);
		}
	}

	// Broadcast the team change (if it really has)
	// 发布团队变更信息（如果确实有变更的话）
	ConditionalBroadcastTeamChanged(this, OldTeamID, NewTeamID);

	LastSeenPlayerState = PlayerState;
}

```

需要注意PlayerController只接收来自PlayerState的变更,而不会主动发起变更
``` cpp
	//~ILyraTeamAgentInterface interface

	// 设置队伍,此函数不应该被调用,因为它是由PlayerState来驱动修改的.
	UE_API virtual void SetGenericTeamId(const FGenericTeamId& NewTeamID) override;

	// 从PlayerState获取队伍ID
	UE_API virtual FGenericTeamId GetGenericTeamId() const override;

	// 用于绑定,主要是Character来这里进行绑定
	UE_API virtual FOnLyraTeamIndexChangedDelegate* GetOnTeamIndexChangedDelegate() override;
	//~End of ILyraTeamAgentInterface interface

```
``` cpp
void ALyraPlayerController::SetGenericTeamId(const FGenericTeamId& NewTeamID)
{
	UE_LOG(LogLyraTeams, Error, TEXT("You can't set the team ID on a player controller (%s); it's driven by the associated player state"), *GetPathNameSafe(this));
}

```
## 力反馈设置
首先是我们之前的命令行变量,可以去同步我们的编辑器设置
### 命令行变量
``` cpp
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
### 绑定游戏时设置变更
这里的LyraLocalPlayer和游戏设置,我们还没有实现可以先注释掉.
``` cpp
void ALyraPlayerController::SetPlayer(UPlayer* InPlayer)
{
	Super::SetPlayer(InPlayer);

	if (const ULyraLocalPlayer* LyraLocalPlayer = Cast<ULyraLocalPlayer>(InPlayer))
	{
		ULyraSettingsShared* UserSettings = LyraLocalPlayer->GetSharedSettings();
		UserSettings->OnSettingChanged.AddUObject(this, &ThisClass::OnSettingsChanged);

		OnSettingsChanged(UserSettings);
	}
}

void ALyraPlayerController::OnSettingsChanged(ULyraSettingsShared* InSettings)
{
	bForceFeedbackEnabled = InSettings->GetForceFeedbackEnabled();
}

```
### 重写力反馈的方法
``` cpp
void ALyraPlayerController::UpdateForceFeedback(IInputInterface* InputInterface, const int32 ControllerId)
{
	if (bForceFeedbackEnabled)
	{
		if (const UCommonInputSubsystem* CommonInputSubsystem = UCommonInputSubsystem::Get(GetLocalPlayer()))
		{
			const ECommonInputType CurrentInputType = CommonInputSubsystem->GetCurrentInputType();
			if (Lyra::Input::ShouldAlwaysPlayForceFeedback || CurrentInputType == ECommonInputType::Gamepad || CurrentInputType == ECommonInputType::Touch)
			{
				InputInterface->SetForceFeedbackChannelValues(ControllerId, ForceFeedbackValues);
				return;
			}
		}
	}
	
	InputInterface->SetForceFeedbackChannelValues(ControllerId, FForceFeedbackValues());
}

```

## 相机穿透
这里涉及到一个CameraMode.可以先不管.
其实就是涉及到,当我们控制的角色被侵入的时候,让这个角色在这一帧隐藏即可.

### CameraMode调用位置
调用的位置是在
``` cpp

void ULyraCameraMode_ThirdPerson::UpdatePreventPenetration(float DeltaTime)
{
	// ...
	
		// Then aim line to desired camera position
		bool const bSingleRayPenetrationCheck = !bDoPredictiveAvoidance;
		PreventCameraPenetration(*PPActor, SafeLocation, View.Location, DeltaTime, AimLineToDesiredPosBlockedPct, bSingleRayPenetrationCheck);

		ILyraCameraAssistInterface* AssistArray[] = { TargetControllerAssist, TargetActorAssist, PPActorAssist };

		if (AimLineToDesiredPosBlockedPct < ReportPenetrationPercent)
		{
			for (ILyraCameraAssistInterface* Assist : AssistArray)
			{
				if (Assist)
				{
					// camera is too close, tell the assists
					Assist->OnCameraPenetratingTarget();
				}
			}
		}
}

```
### 接口定义
``` cpp
/** */
UINTERFACE(BlueprintType)
class ULyraCameraAssistInterface : public UInterface
{
	GENERATED_BODY()
};

class ILyraCameraAssistInterface
{
	GENERATED_BODY()

public:
	/**
	 * Get the list of actors that we're allowing the camera to penetrate. Useful in 3rd person cameras
	 * when you need the following camera to ignore things like the a collection of view targets, the pawn,
	 * a vehicle..etc.+
	 *
	 * 获取我们允许摄像机穿透的演员列表。在使用第三人称摄像机时非常有用。
	 * 当您需要以下摄像机忽略诸如一系列视点目标、角色、车辆等元素时，此功能非常有用。
	 * 
	 */
	virtual void GetIgnoredActorsForCameraPentration(TArray<const AActor*>& OutActorsAllowPenetration) const { }

	/**
	 * The target actor to prevent penetration on.  Normally, this is almost always the view target, which if
	 * unimplemented will remain true.  However, sometimes the view target, isn't the same as the root actor 
	 * you need to keep in frame.
	 *
	 * 需要防范入侵的目标对象。通常情况下，这几乎总是视图目标，即如果视图目标未实现，则该值始终为真。然而，有时视图目标与您需要保持在画面中的根对象并不相同。
	 * 
	 */
	virtual TOptional<AActor*> GetCameraPreventPenetrationTarget() const
	{
		return TOptional<AActor*>();
	}

	/** Called if the camera penetrates the focal target.  Useful if you want to hide the target actor when being overlapped. */
	/** 若相机穿透了焦点目标，则会调用此函数。若您希望在目标被覆盖时隐藏目标角色，则此功能非常有用。*/
	virtual void OnCameraPenetratingTarget() { }
};

```
### 执行时机
``` cpp


	/**
	 * 根据游戏玩法构建一个隐藏组件的列表
	 * @参数 视图位置：需要隐藏/取消隐藏的视点
	 * @参数 隐藏组件列表：要添加到列表中或从列表中移除的组件列表
	 * 
	 */	
	UE_API virtual void UpdateHiddenComponents(const FVector& ViewLocation, TSet<FPrimitiveComponentId>& OutHiddenComponents) override;

	//~ILyraCameraAssistInterface interface
	// 当被相机穿透的时候需要调用此函数去设置一帧的隐藏
	UE_API virtual void OnCameraPenetratingTarget() override;
	//~End of ILyraCameraAssistInterface interface

```
注意这是一个重写的方法.只在这帧添加.添加完毕后消耗我们的bHideViewTargetPawnNextFrame变量.
``` cpp
void ALyraPlayerController::OnCameraPenetratingTarget()
{
	bHideViewTargetPawnNextFrame = true;
}
``` cpp


``` cpp
void ALyraPlayerController::UpdateHiddenComponents(const FVector& ViewLocation, TSet<FPrimitiveComponentId>& OutHiddenComponents)
{
	Super::UpdateHiddenComponents(ViewLocation, OutHiddenComponents);

	if (bHideViewTargetPawnNextFrame)
	{
		AActor* const ViewTargetPawn = PlayerCameraManager ? Cast<AActor>(PlayerCameraManager->GetViewTarget()) : nullptr;
		if (ViewTargetPawn)
		{
			// internal helper func to hide all the components
			// 内部辅助函数，用于隐藏所有组件
			auto AddToHiddenComponents = [&OutHiddenComponents](const TInlineComponentArray<UPrimitiveComponent*>& InComponents)
			{
				// add every component and all attached children
				// 添加每个组件及其所有附属子元素
				for (UPrimitiveComponent* Comp : InComponents)
				{
					if (Comp->IsRegistered())
					{
						OutHiddenComponents.Add(Comp->GetPrimitiveSceneId());

						for (USceneComponent* AttachedChild : Comp->GetAttachChildren())
						{
							static FName NAME_NoParentAutoHide(TEXT("NoParentAutoHide"));
							UPrimitiveComponent* AttachChildPC = Cast<UPrimitiveComponent>(AttachedChild);
							if (AttachChildPC && AttachChildPC->IsRegistered() && !AttachChildPC->ComponentTags.Contains(NAME_NoParentAutoHide))
							{
								OutHiddenComponents.Add(AttachChildPC->GetPrimitiveSceneId());
							}
						}
					}
				}
			};

			//TODO Solve with an interface.  Gather hidden components or something.
			//TODO Hiding isn't awesome, sometimes you want the effect of a fade out over a proximity, needs to bubble up to designers.

			// 请使用接口来解决这个问题。可以整合一些隐藏功能或者采取其他措施。
			// 请不要将隐藏操作视为完美解决方案，有时您可能希望在接近目标时产生逐渐淡出的效果，这需要与设计师进行沟通。
			
			// hide pawn's components
			TInlineComponentArray<UPrimitiveComponent*> PawnComponents;
			ViewTargetPawn->GetComponents(PawnComponents);
			AddToHiddenComponents(PawnComponents);

			//// hide weapon too
			//if (ViewTargetPawn->CurrentWeapon)
			//{
			//	TInlineComponentArray<UPrimitiveComponent*> WeaponComponents;
			//	ViewTargetPawn->CurrentWeapon->GetComponents(WeaponComponents);
			//	AddToHiddenComponents(WeaponComponents);
			//}
		}

		// we consumed it, reset for next frame
		bHideViewTargetPawnNextFrame = false;
	}
}

```

## 回放
### 是否录制回放

``` cpp

bool ALyraPlayerController::ShouldRecordClientReplay()
{
	UWorld* World = GetWorld();
	UGameInstance* GameInstance = GetGameInstance();
	
	if (GameInstance != nullptr &&
		World != nullptr &&
		!World->IsPlayingReplay() &&
		!World->IsRecordingClientReplay() &&
		NM_DedicatedServer != GetNetMode() &&
		IsLocalPlayerController())
	{
		// 游戏实例不能为空
		// 世界不能为空
		// 不能正在播放回放
		// 不能正在记录录像
		// 不能是专属服务器
		// 必须是本地控制的玩家控制器

		
		FString DefaultMap = UGameMapsSettings::GetGameDefaultMap();
		FString CurrentMap = World->URL.Map;

#if WITH_EDITOR
		CurrentMap = UWorld::StripPIEPrefixFromPackageName(CurrentMap, World->StreamingLevelsPrefix);
#endif
		if (CurrentMap == DefaultMap)
		{
			// Never record demos on the default frontend map, this could be replaced with a better check for being in the main menu
			// 请勿在默认的前端地图上录制演示内容，这一操作可以改为采用更有效的检查方式，以确认是否处于主菜单界面。
			return false;
		}

		if (UReplaySubsystem* ReplaySubsystem = GameInstance->GetSubsystem<UReplaySubsystem>())
		{
			if (ReplaySubsystem->IsRecording() || ReplaySubsystem->IsPlaying())
			{
				// Only one at a time
				// 一次只能一个
				return false;
			}
		}

		// If this is possible, now check the settings
		// 如果可行的话，现在就检查一下设置吧
		if (const ULyraLocalPlayer* LyraLocalPlayer = Cast<ULyraLocalPlayer>(GetLocalPlayer()))
		{
			if (LyraLocalPlayer->GetLocalSettings()->ShouldAutoRecordReplays())
			{
				return true;
			}
		}
	}
	return false;
}
```
回放播送控制器不可以记录录像.
``` cpp
bool ALyraReplayPlayerController::ShouldRecordClientReplay()
{
	return false;
}
```

### 尝试开始录制

回放子系统 现在还没有写 可以注释掉.后续章节再详细讲.

``` cpp

bool ALyraPlayerController::TryToRecordClientReplay()
{
	// See if we should record a replay
	// 查看是否需要录制回放内容
	if (ShouldRecordClientReplay())
	{
		if (ULyraReplaySubsystem* ReplaySubsystem = GetGameInstance()->GetSubsystem<ULyraReplaySubsystem>())
		{
			APlayerController* FirstLocalPlayerController = GetGameInstance()->GetFirstLocalPlayerController();
			if (FirstLocalPlayerController == this)
			{
				// If this is the first player, update the spectator player for local replays and then record
				// 如果这是第一个玩家，则为本地回放更新旁观者玩家信息，然后进行记录
				if (ALyraGameState* GameState = Cast<ALyraGameState>(GetWorld()->GetGameState()))
				{
					GameState->SetRecorderPlayerState(PlayerState);

					ReplaySubsystem->RecordClientReplay(this);
					return true;
				}
			}
		}
	}
	return false;
}

```
## 更新回放播放时的ViewTarget
### 监听回放状态类的变化
``` cpp
void ALyraReplayPlayerController::Tick(float DeltaSeconds)
{
	Super::Tick(DeltaSeconds);

	// The state may go invalid at any time due to scrubbing during a replay
	// 由于在重放过程中进行的数据清除操作，该状态可能会随时失效。
	if (!IsValid(FollowedPlayerState))
	{
		UWorld* World = GetWorld();

		// Listen for changes for both recording and playback
		// 监听录制和播放过程中的任何变化
		if (ALyraGameState* GameState = Cast<ALyraGameState>(World->GetGameState()))
		{
			if (!GameState->OnRecorderPlayerStateChangedEvent.IsBoundToObject(this))
			{
				GameState->OnRecorderPlayerStateChangedEvent.AddUObject(this, &ThisClass::RecorderPlayerStateUpdated);
			}
			if (APlayerState* RecorderState = GameState->GetRecorderPlayerState())
			{
				RecorderPlayerStateUpdated(RecorderState);
			}
		}
	}
}
```
### 更新视图

``` cpp
void ALyraReplayPlayerController::RecorderPlayerStateUpdated(APlayerState* NewRecorderPlayerState)
{
	if (NewRecorderPlayerState)
	{
		FollowedPlayerState = NewRecorderPlayerState;

		// Bind to when pawn changes and call now
		// 当兵种发生变化时绑定此操作，并立即执行
		NewRecorderPlayerState->OnPawnSet.AddUniqueDynamic(this, &ALyraReplayPlayerController::OnPlayerStatePawnSet);
		OnPlayerStatePawnSet(NewRecorderPlayerState, NewRecorderPlayerState->GetPawn(), nullptr);
	}
}

void ALyraReplayPlayerController::OnPlayerStatePawnSet(APlayerState* ChangedPlayerState, APawn* NewPlayerPawn, APawn* OldPlayerPawn)
{
	if (ChangedPlayerState == FollowedPlayerState)
	{
		SetViewTarget(NewPlayerPawn);
	}
}


```

## 代码
### LyraPlayerController
``` cpp

/**
 * ALyraPlayerController
 *
 *	The base player controller class used by this project.
 * 本项目所使用的基础玩家控制器类。
 */
UCLASS(MinimalAPI, Config = Game, Meta = (ShortTooltip = "The base player controller class used by this project."))
class ALyraPlayerController : public ACommonPlayerController, public ILyraCameraAssistInterface, public ILyraTeamAgentInterface
{
	GENERATED_BODY()

public:
/*
 * 构造函数
 * 设置相机管理组件类
 * 设置作弊左键类
 */
	UE_API ALyraPlayerController(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	// 获取玩家状态
	UFUNCTION(BlueprintCallable, Category = "Lyra|PlayerController")
	UE_API ALyraPlayerState* GetLyraPlayerState() const;

	// 获取ASC组件
	UFUNCTION(BlueprintCallable, Category = "Lyra|PlayerController")
	UE_API ULyraAbilitySystemComponent* GetLyraAbilitySystemComponent() const;
	
	// 获取玩家HUD
	UFUNCTION(BlueprintCallable, Category = "Lyra|PlayerController")
	UE_API ALyraHUD* GetLyraHUD() const;

	// Call from game state logic to start recording an automatic client replay if ShouldRecordClientReplay returns true
	// 在游戏状态逻辑中，如果“应录制客户端回放”这一条件返回为真，则启动自动录制客户端回放的过程
	UFUNCTION(BlueprintCallable, Category = "Lyra|PlayerController")
	UE_API bool TryToRecordClientReplay();

	// Call to see if we should record a replay, subclasses could change this
	// 调用此方法以确定是否需要录制回放，子类可以对此进行修改
	UE_API virtual bool ShouldRecordClientReplay();

	// Run a cheat command on the server.
	// 在服务器上运行作弊指令。
	UFUNCTION(Reliable, Server, WithValidation)
	UE_API void ServerCheat(const FString& Msg);

	// Run a cheat command on the server for all players.
	// 在服务器上为所有玩家执行作弊指令。
	UFUNCTION(Reliable, Server, WithValidation)
	UE_API void ServerCheatAll(const FString& Msg);

	//~AActor interface

	// 暂无功能
	UE_API virtual void PreInitializeComponents() override;

	// 注册一个HTTP Server的回调
	UE_API virtual void BeginPlay() override;

	// 暂无功能
	UE_API virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

	// 用来同步TargetViewRotation
	UE_API virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
	//~End of AActor interface

	//~AController interface
	
	// 嵌入作弊时机
	UE_API virtual void OnPossess(APawn* InPawn) override;

	// 确保ASC的解除成功
	UE_API virtual void OnUnPossess() override;

	// 初始化PlayerState 并广播出去 在广播中绑定到了PlayerState的队伍变更
	UE_API virtual void InitPlayerState() override;

	// 清理PlayerState 并广播出去
	UE_API virtual void CleanupPlayerState() override;

	// 确保专属服务器下的ASC执行时序问题
	UE_API virtual void OnRep_PlayerState() override;
	//~End of AController interface

	//~APlayerController interface
	// 暂无功能
	UE_API virtual void ReceivedPlayer() override;

	// 确保TargetViewRotation的获取正确
	UE_API virtual void PlayerTick(float DeltaTime) override;

	// 绑定本地玩家类的变更,从而响应设置的更改.
	UE_API virtual void SetPlayer(UPlayer* InPlayer) override;

	// 是否开启作弊
	UE_API virtual void AddCheats(bool bForce) override;

	// 更新力反馈
	UE_API virtual void UpdateForceFeedback(IInputInterface* InputInterface, const int32 ControllerId) override;

	/**
	 * 根据游戏玩法构建一个隐藏组件的列表
	 * @参数 视图位置：需要隐藏/取消隐藏的视点
	 * @参数 隐藏组件列表：要添加到列表中或从列表中移除的组件列表
	 * 
	 */	
	UE_API virtual void UpdateHiddenComponents(const FVector& ViewLocation, TSet<FPrimitiveComponentId>& OutHiddenComponents) override;

	// 暂无功能
	UE_API virtual void PreProcessInput(const float DeltaTime, const bool bGamePaused) override;

	// 这里去主动激活ASC
	UE_API virtual void PostProcessInput(const float DeltaTime, const bool bGamePaused) override;
	//~End of APlayerController interface

	//~ILyraCameraAssistInterface interface
	// 当被相机穿透的时候需要调用此函数去设置一帧的隐藏
	UE_API virtual void OnCameraPenetratingTarget() override;
	//~End of ILyraCameraAssistInterface interface
	
	//~ILyraTeamAgentInterface interface

	// 设置队伍,此函数不应该被调用,因为它是由PlayerState来驱动修改的.
	UE_API virtual void SetGenericTeamId(const FGenericTeamId& NewTeamID) override;

	// 从PlayerState获取队伍ID
	UE_API virtual FGenericTeamId GetGenericTeamId() const override;

	// 用于绑定,主要是Character来这里进行绑定
	UE_API virtual FOnLyraTeamIndexChangedDelegate* GetOnTeamIndexChangedDelegate() override;
	//~End of ILyraTeamAgentInterface interface

	// 开启自动运行
	UFUNCTION(BlueprintCallable, Category = "Lyra|Character")
	UE_API void SetIsAutoRunning(const bool bEnabled);

	// 获取是否自动运行
	UFUNCTION(BlueprintCallable, Category = "Lyra|Character")
	UE_API bool GetIsAutoRunning() const;

private:

	// 获取切换的代理,用于下级单位进行绑定,如Character
	UPROPERTY()
	FOnLyraTeamIndexChangedDelegate OnTeamChangedDelegate;

	// 记录上次看见的PlayerState
	UPROPERTY()
	TObjectPtr<APlayerState> LastSeenPlayerState;

private:
	// 用于绑定在PlayerState上的队伍接口
	UFUNCTION()
	void OnPlayerStateChangedTeam(UObject* TeamAgent, int32 OldTeam, int32 NewTeam);

protected:
	// Called when the player state is set or cleared
	// 当玩家状态被设置或取消时触发此事件
	UE_API virtual void OnPlayerStateChanged();

private:

	// 广播玩家状态发生了变化,需要重新绑定队伍代理
	void BroadcastOnPlayerStateChanged();

protected:

	//~APlayerController interface

	//~End of APlayerController interface
	
	// 用于绑定到本地玩家上面的设置,实现读取是否开启力反馈
	UE_API void OnSettingsChanged(ULyraSettingsShared* Settings);

	// 开启自动运行
	UE_API void OnStartAutoRun();
	// 关闭自动运行
	UE_API void OnEndAutoRun();

	// 蓝图拓展事件
	UFUNCTION(BlueprintImplementableEvent, meta=(DisplayName="OnStartAutoRun"))
	UE_API void K2_OnStartAutoRun();

	// 蓝图拓展事件
	UFUNCTION(BlueprintImplementableEvent, meta=(DisplayName="OnEndAutoRun"))
	UE_API void K2_OnEndAutoRun();

	// 用于被相机穿透时去隐藏一帧 使用后即复原
	bool bHideViewTargetPawnNextFrame = false;

};


```
### LyraReplayPlayerController

``` cpp

// A player controller used for replay capture and playback
// 用于录制和回放的玩家控制器
UCLASS()
class ALyraReplayPlayerController : public ALyraPlayerController
{
	GENERATED_BODY()

	virtual void Tick(float DeltaSeconds) override;
	virtual void SmoothTargetViewRotation(APawn* TargetPawn, float DeltaSeconds) override;
	virtual bool ShouldRecordClientReplay() override;

	// Callback for when the game state's RecorderPlayerState gets replicated during replay playback
	// 当在回放过程中游戏状态的“记录玩家状态”被复制时的回调函数
	void RecorderPlayerStateUpdated(APlayerState* NewRecorderPlayerState);

	// Callback for when the followed player state changes pawn
	// 当关注的玩家状态发生变化时（例如角色状态改变）的回调函数
	UFUNCTION()
	void OnPlayerStatePawnSet(APlayerState* ChangedPlayerState, APawn* NewPlayerPawn, APawn* OldPlayerPawn);

	// The player state we are currently following */
	// 我们当前所遵循的玩家状态 */
	UPROPERTY(Transient)
	TObjectPtr<APlayerState> FollowedPlayerState;
};


```
## 总结
本节内容也是极其重要的.通过简单的梳理.
玩家控制器实现了众多的功能.其中关于回放,相机模式,作弊器,ASC等等可在后续章节深入探讨.