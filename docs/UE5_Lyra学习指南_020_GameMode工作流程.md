# UE5_Lyra学习指南_020_GameMode工作流程

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_020\_GameMode工作流程](#ue5_lyra学习指南_020_gamemode工作流程)
	- [概述](#概述)
	- [GameMode初始化流程](#gamemode初始化流程)
		- [a. InitGame的时机](#a-initgame的时机)
			- [创建GameMode](#创建gamemode)
			- [初始化GameMode](#初始化gamemode)
		- [b.InitGameState的时机](#binitgamestate的时机)
	- [玩家登录流程](#玩家登录流程)
		- [控制器创建流程](#控制器创建流程)
		- [Pawn创建流程](#pawn创建流程)
			- [寻找出生点](#寻找出生点)
			- [生成Pawn](#生成pawn)
			- [控制Pawn](#控制pawn)
		- [BeginPlay的时机](#beginplay的时机)
	- [DS的初始化启动](#ds的初始化启动)
	- [代码](#代码)
	- [总结](#总结)



## 概述
本节主要讲解LyraGameMode的工作流程.  
本节非常难,非常复杂!!!  
但是呢,我们之前已经把Experience的加载流程讲了!  
PlayerStart及其生成组件也讲了.  
## GameMode初始化流程
这一帧!
### a. InitGame的时机

在InitGame里面绑定下一帧去初始化体验!

``` cpp
void ALyraGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
	Super::InitGame(MapName, Options, ErrorMessage);

	// Wait for the next frame to give time to initialize startup settings
	// 等待下一帧的到来，以便有足够的时间来初始化启动设置
	GetWorld()->GetTimerManager().SetTimerForNextTick(this, &ThisClass::HandleMatchAssignmentIfNotExpectingOne);
}
```

这个是调用的堆栈:
``` txt
ALyraGameMode::InitGame(const FString &, const FString &, FString &) LyraGameMode.cpp:96
UWorld::InitializeActorsForPlay(const FURL &, bool, FRegisterComponentContext *) World.cpp:5777
UGameInstance::StartPlayInEditorGameInstance(ULocalPlayer *, const FGameInstancePIEParameters &) GameInstance.cpp:528
UEditorEngine::CreateInnerProcessPIEGameInstance(FRequestPlaySessionParams &, const FGameInstancePIEParameters &, int) PlayLevel.cpp:3182
UEditorEngine::OnLoginPIEComplete_Deferred(int, bool, FString, FPieLoginStruct) PlayLevel.cpp:1626
UEditorEngine::CreateNewPlayInEditorInstance(FRequestPlaySessionParams &, const bool, EPlayNetMode) PlayLevel.cpp:1890
UEditorEngine::StartPlayInEditorSession(FRequestPlaySessionParams &) PlayLevel.cpp:2909
UEditorEngine::StartQueuedPlaySessionRequestImpl() PlayLevel.cpp:1204
UEditorEngine::StartQueuedPlaySessionRequest() PlayLevel.cpp:1101
UEditorEngine::Tick(float, bool) EditorEngine.cpp:2037
UUnrealEdEngine::Tick(float, bool) UnrealEdEngine.cpp:530
ULyraEditorEngine::Tick(float, bool) LyraEditorEngine.cpp:37
FEngineLoop::Tick() LaunchEngineLoop.cpp:5619
[内联] EngineTick() Launch.cpp:60
GuardedMain(const wchar_t *) Launch.cpp:189
LaunchWindowsStartup(HINSTANCE__ *, HINSTANCE__ *, char *, int, const wchar_t *) LaunchWindows.cpp:271
WinMain(HINSTANCE__ *, HINSTANCE__ *, char *, int) LaunchWindows.cpp:339
``` 
重点是这个函数:
FGameInstancePIEResult UGameInstance::StartPlayInEditorGameInstance(ULocalPlayer* LocalPlayer, const FGameInstancePIEParameters& Params)

```cpp
FGameInstancePIEResult UGameInstance::StartPlayInEditorGameInstance(ULocalPlayer* LocalPlayer, const FGameInstancePIEParameters& Params)
{
	// 497
		if (!PlayWorld->SetGameMode(URL))
		{
			// Setting the game mode failed so bail 
			return FGameInstancePIEResult::Failure(NSLOCTEXT("UnrealEd", "Error_FailedCreateEditorPreviewWorld", "Failed to create editor preview world."));
		}
	//528
		SlowTask.EnterProgressFrame(10, NSLOCTEXT("UnrealEd", "PIEInitializingActors", "Starting PIE (Initializing Actors)..."));
		{
			FRegisterComponentContext Context(PlayWorld);
			PlayWorld->InitializeActorsForPlay(URL, true, &Context);
		}


}

```
#### 创建GameMode
```cpp
bool UWorld::SetGameMode(const FURL& InURL)
{
	if (!IsNetMode(NM_Client) && !AuthorityGameMode)
	{
		AuthorityGameMode = GetGameInstance()->CreateGameModeForURL(InURL, this);
		if( AuthorityGameMode != NULL )
		{
			return true;
		}
		else
		{
			UE_LOG(LogWorld, Error, TEXT("Failed to spawn GameMode actor."));
			return false;
		}
	}

	return false;
}
```

``` cpp
AGameModeBase* UGameInstance::CreateGameModeForURL(FURL InURL, UWorld* InWorld)
{
	// ...	
	return World->SpawnActor<AGameModeBase>(GameClass, SpawnInfo);
}

```

#### 初始化GameMode

``` cpp
void UWorld::InitializeActorsForPlay(const FURL& InURL, bool bResetTime, FRegisterComponentContext* Context)
{
		//....
		// 5774
		// Init the game mode.
		if (AuthorityGameMode && !AuthorityGameMode->IsActorInitialized())
		{
			AuthorityGameMode->InitGame( FPaths::GetBaseFilename(InURL.Map), Options, Error );
		}
		// Route various initialization functions and set volumes.
		const int32 ProcessAllRouteActorInitializationGranularity = 0;
		for (int32 LevelIndex = 0; LevelIndex < Levels.Num(); LevelIndex++)
		{	//5784
			ULevel* const Level = Levels[LevelIndex];
			Level->RouteActorInitialize(ProcessAllRouteActorInitializationGranularity);
		}
		// ...

}
``` 


这个OptionsString可以关注一下.
``` cpp
void AGameModeBase::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
	UWorld* World = GetWorld();

	// Save Options for future use
	OptionsString = Options;

	FActorSpawnParameters SpawnInfo;
	SpawnInfo.Instigator = GetInstigator();
	SpawnInfo.ObjectFlags |= RF_Transient;	// We never want to save game sessions into a map
	GameSession = World->SpawnActor<AGameSession>(GetGameSessionClass(), SpawnInfo);
	GameSession->InitOptions(Options);

	FGameModeEvents::GameModeInitializedEvent.Broadcast(this);
	if (GetNetMode() != NM_Standalone)
	{
		// Attempt to login, returning true means an async login is in flight
		if (!UOnlineEngineInterface::Get()->DoesSessionExist(World, GameSession->SessionName) && 
			!GameSession->ProcessAutoLogin())
		{
			GameSession->RegisterServer();
		}
	}
}

```

### b.InitGameState的时机
调用堆栈
``` txt
ALyraGameMode::InitGameState() LyraGameMode.cpp:531
AGameModeBase::PreInitializeComponents() GameModeBase.cpp:143
ULevel::RouteActorInitialize(int) Level.cpp:3803
UWorld::InitializeActorsForPlay(const FURL &, bool, FRegisterComponentContext *) World.cpp:5785
UGameInstance::StartPlayInEditorGameInstance(ULocalPlayer *, const FGameInstancePIEParameters &) GameInstance.cpp:528
UEditorEngine::CreateInnerProcessPIEGameInstance(FRequestPlaySessionParams &, const FGameInstancePIEParameters &, int) PlayLevel.cpp:3182
UEditorEngine::OnLoginPIEComplete_Deferred(int, bool, FString, FPieLoginStruct) PlayLevel.cpp:1626
UEditorEngine::CreateNewPlayInEditorInstance(FRequestPlaySessionParams &, const bool, EPlayNetMode) PlayLevel.cpp:1890
UEditorEngine::StartPlayInEditorSession(FRequestPlaySessionParams &) PlayLevel.cpp:2909
UEditorEngine::StartQueuedPlaySessionRequestImpl() PlayLevel.cpp:1204
UEditorEngine::StartQueuedPlaySessionRequest() PlayLevel.cpp:1101
UEditorEngine::Tick(float, bool) EditorEngine.cpp:2037
UUnrealEdEngine::Tick(float, bool) UnrealEdEngine.cpp:530
ULyraEditorEngine::Tick(float, bool) LyraEditorEngine.cpp:37
FEngineLoop::Tick() LaunchEngineLoop.cpp:5619
[内联] EngineTick() Launch.cpp:60
GuardedMain(const wchar_t *) Launch.cpp:189
LaunchWindowsStartup(HINSTANCE__ *, HINSTANCE__ *, char *, int, const wchar_t *) LaunchWindows.cpp:271
WinMain(HINSTANCE__ *, HINSTANCE__ *, char *, int) LaunchWindows.cpp:339

```



在InitGameState()绑定Experience加载回调
``` cpp
void ALyraGameMode::InitGameState()
{
	Super::InitGameState();

	// Listen for the experience load to complete
	// 监听体验加载完成的进程
	ULyraExperienceManagerComponent* ExperienceComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
	check(ExperienceComponent);
	
	ExperienceComponent->CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
}

```
还是在UWorld::InitializeActorsForPlay()中调用.
``` cpp
void UWorld::InitializeActorsForPlay(const FURL& InURL, bool bResetTime, FRegisterComponentContext* Context)
{
		//....
		// 5774
		// Init the game mode.
		if (AuthorityGameMode && !AuthorityGameMode->IsActorInitialized())
		{
			AuthorityGameMode->InitGame( FPaths::GetBaseFilename(InURL.Map), Options, Error );
		}
		// Route various initialization functions and set volumes.
		const int32 ProcessAllRouteActorInitializationGranularity = 0;
		for (int32 LevelIndex = 0; LevelIndex < Levels.Num(); LevelIndex++)
		{	//5784
			ULevel* const Level = Levels[LevelIndex];
			Level->RouteActorInitialize(ProcessAllRouteActorInitializationGranularity);
		}
		// ...

}
``` 
``` cpp
void ULevel::RouteActorInitialize(int32 NumActorsToProcess)
{
	const bool bFullProcessing = (NumActorsToProcess <= 0);
	switch (RouteActorInitializationState)
	{
		case ERouteActorInitializationState::Preinitialize:
		{
			// Actor pre-initialization may spawn new actors so we need to incrementally process until actor count stabilizes
			while (RouteActorInitializationIndex < Actors.Num())
			{
				AActor* const Actor = Actors[RouteActorInitializationIndex];
				if (Actor && !Actor->IsActorInitialized())
				{	//3803
					Actor->PreInitializeComponents();
				}
				//......
			}
		}
	

	}		
}
```

GameState的创建时机

``` cpp
void AGameModeBase::PreInitializeComponents()
{
	Super::PreInitializeComponents();

	FActorSpawnParameters SpawnInfo;
	SpawnInfo.Instigator = GetInstigator();
	SpawnInfo.ObjectFlags |= RF_Transient;	// We never want to save game states or network managers into a map									
											
	// Fallback to default GameState if none was specified.
	if (GameStateClass == nullptr)
	{
		UE_LOG(LogGameMode, Warning, TEXT("No GameStateClass was specified in %s (%s)"), *GetName(), *GetClass()->GetName());
		GameStateClass = AGameStateBase::StaticClass();
	}

	UWorld* World = GetWorld();
	// 132
	GameState = World->SpawnActor<AGameStateBase>(GameStateClass, SpawnInfo);
	World->SetGameState(GameState);
	if (GameState)
	{
		GameState->AuthorityGameMode = this;
	}

	// Only need NetworkManager for servers in net games
	AWorldSettings* WorldSettings = World->GetWorldSettings();
	World->NetworkManager = WorldSettings->GameNetworkManagerClass ? World->SpawnActor<AGameNetworkManager>(WorldSettings->GameNetworkManagerClass, SpawnInfo) : nullptr;

	// 143
	InitGameState();
}

```




## 玩家登录流程
堆栈
``` txt
ALyraGameMode::GenericPlayerInitialization(AController *) LyraGameMode.cpp:543
AGameModeBase::PostLogin(APlayerController *) GameModeBase.cpp:1011
UWorld::SpawnPlayActor(UPlayer *, ENetRole, const FURL &, const FUniqueNetIdRepl &, FString &, unsigned char) LevelActor.cpp:1109
ULocalPlayer::SpawnPlayActor(const FString &, FString &, UWorld *) LocalPlayer.cpp:296
ULyraLocalPlayer::SpawnPlayActor(const FString &, FString &, UWorld *) LyraLocalPlayer.cpp:39
UGameInstance::StartPlayInEditorGameInstance(ULocalPlayer *, const FGameInstancePIEParameters &) GameInstance.cpp:537
UEditorEngine::CreateInnerProcessPIEGameInstance(FRequestPlaySessionParams &, const FGameInstancePIEParameters &, int) PlayLevel.cpp:3182
UEditorEngine::OnLoginPIEComplete_Deferred(int, bool, FString, FPieLoginStruct) PlayLevel.cpp:1626
UEditorEngine::CreateNewPlayInEditorInstance(FRequestPlaySessionParams &, const bool, EPlayNetMode) PlayLevel.cpp:1890
UEditorEngine::StartPlayInEditorSession(FRequestPlaySessionParams &) PlayLevel.cpp:2909
UEditorEngine::StartQueuedPlaySessionRequestImpl() PlayLevel.cpp:1204
UEditorEngine::StartQueuedPlaySessionRequest() PlayLevel.cpp:1101
UEditorEngine::Tick(float, bool) EditorEngine.cpp:2037
UUnrealEdEngine::Tick(float, bool) UnrealEdEngine.cpp:530
ULyraEditorEngine::Tick(float, bool) LyraEditorEngine.cpp:37
FEngineLoop::Tick() LaunchEngineLoop.cpp:5619
[内联] EngineTick() Launch.cpp:60
GuardedMain(const wchar_t *) Launch.cpp:189
LaunchWindowsStartup(HINSTANCE__ *, HINSTANCE__ *, char *, int, const wchar_t *) LaunchWindows.cpp:271
WinMain(HINSTANCE__ *, HINSTANCE__ *, char *, int) LaunchWindows.cpp:339

```

### 控制器创建流程
注意专属服务器的情况下,一开始没有玩家.所以这部分流程并不是和初始化流程顺序执行.虽然此时debug是在一起.
这里会走的我们的ULyraLocalPlayer.此时还没有写可以先不管
```cpp
FGameInstancePIEResult UGameInstance::StartPlayInEditorGameInstance(ULocalPlayer* LocalPlayer, const FGameInstancePIEParameters& Params)
{
	// ...

	// 537
		// @todo, just use WorldContext.GamePlayer[0]?
		if (LocalPlayer)
		{
			FString Error;
			if (!LocalPlayer->SpawnPlayActor(URL.ToString(1), Error, PlayWorld))
			{
				return FGameInstancePIEResult::Failure(FText::Format(NSLOCTEXT("UnrealEd", "Error_CouldntSpawnPlayer", "Couldn't spawn player: {0}"), FText::FromString(Error)));
			}

			if (GameViewport != NULL && GameViewport->Viewport != NULL)
			{
				SlowTask.EnterProgressFrame(25, NSLOCTEXT("UnrealEd", "PIEWaitingForLevelStreaming", "Starting PIE (Waiting for level streaming)..."));
				// Stream any levels now that need to be loaded before the game starts as a result of spawning the local player
				GEngine->BlockTillLevelStreamingCompleted(PlayWorld);
			}
		}
	//


}

```

``` cpp
bool ULyraLocalPlayer::SpawnPlayActor(const FString& URL, FString& OutError, UWorld* InWorld)
{
	const bool bResult = Super::SpawnPlayActor(URL, OutError, InWorld);

	OnPlayerControllerChanged(PlayerController);

	return bResult;
}


```
玩家控制器会通过ULocalPlayer::SpawnPlayActor进行生成:
``` cpp
bool ULocalPlayer::SpawnPlayActor(const FString& URL,FString& OutError, UWorld* InWorld)
{
	check(InWorld);
	if (!InWorld->IsNetMode(NM_Client))
	{
		// ...

		PlayerController = InWorld->SpawnPlayActor(this, ROLE_SimulatedProxy, PlayerURL, UniqueId, OutError, GEngine->GetGamePlayers(InWorld).Find(this));
	}
	else
	{
		// ....
	}
	return PlayerController != NULL;
}


```
而这部分的生成是通过世界去生成的.此时会触发GameMode的登录函数!  Login,PostLogin
``` cpp
APlayerController* UWorld::SpawnPlayActor(UPlayer* NewPlayer, ENetRole RemoteRole, const FURL& InURL, const FUniqueNetIdRepl& UniqueId, FString& Error, uint8 InNetPlayerIndex)
{
	Error = TEXT("");

	// Make the option string.
	FString Options;
	for (int32 i = 0; i < InURL.Op.Num(); i++)
	{
		Options += TEXT('?');
		Options += InURL.Op[i];
	}

	if (AGameModeBase* const GameMode = GetAuthGameMode())
	{
		// Give the GameMode a chance to accept the login
		APlayerController* const NewPlayerController = GameMode->Login(NewPlayer, RemoteRole, *InURL.Portal, Options, UniqueId, Error);
		if (NewPlayerController == NULL)
		{
			UE_LOG(LogSpawn, Warning, TEXT("Login failed: %s"), *Error);
			return NULL;
		}

		if (UNetConnection* Connection = Cast<UNetConnection>(NewPlayer))
		{
			NewPlayerController->SetClientHandshakeId(Connection->GetClientHandshakeId());
		}

		UE_LOG(LogSpawn, Log, TEXT("%s got player %s [%s]"), *NewPlayerController->GetName(), *NewPlayer->GetName(), UniqueId.IsValid() ? *UniqueId->ToString() : TEXT("Invalid"));

		// Possess the newly-spawned player.
		NewPlayerController->NetPlayerIndex = InNetPlayerIndex;
		NewPlayerController->SetRole(ROLE_Authority);
		NewPlayerController->SetReplicates(RemoteRole != ROLE_None);
		if (RemoteRole == ROLE_AutonomousProxy)
		{
			NewPlayerController->SetAutonomousProxy(true);
		}
		NewPlayerController->SetPlayer(NewPlayer);
		GameMode->PostLogin(NewPlayerController);
		return NewPlayerController;
	}

	UE_LOG(LogSpawn, Warning, TEXT("Login failed: No game mode set."));
	return nullptr;
}
```
到这一步玩家控制器已经生成好了.接下来我们要生成HUD.生成玩家控制的Pawn.
Lyra在HandleStartingNewPlayer()里面断掉了Pawn的这部分流程.必须要等Experience加载完毕才行!

PostLogin
``` cpp

void AGameModeBase::PostLogin(APlayerController* NewPlayer)
{
	// Runs shared initialization that can happen during seamless travel as well

	GenericPlayerInitialization(NewPlayer);

	// Perform initialization that only happens on initially joining a server

	UWorld* World = GetWorld();

	NewPlayer->ClientCapBandwidth(NewPlayer->Player->CurrentNetSpeed);

	if (MustSpectate(NewPlayer))
	{
		NewPlayer->ClientGotoState(NAME_Spectating);
	}
	else
	{
		// If NewPlayer is not only a spectator and has a valid ID, add it as a user to the replay.
		const FUniqueNetIdRepl& NewPlayerStateUniqueId = NewPlayer->PlayerState->GetUniqueId();
		if (NewPlayerStateUniqueId.IsValid() && NewPlayerStateUniqueId.IsV1())
		{
			GetGameInstance()->AddUserToReplay(NewPlayerStateUniqueId.ToString());
		}
	}

	if (GameSession)
	{
		GameSession->PostLogin(NewPlayer);
	}

	OnPostLogin(NewPlayer);

	// Now that initialization is done, try to spawn the player's pawn and start match
	HandleStartingNewPlayer(NewPlayer);
}

```
这里需要看一下PlayerState的生成时机.
``` cpp
void APlayerController::PostInitializeComponents()
{
	Super::PostInitializeComponents();

	if (IsValidChecked(this) && (GetNetMode() != NM_Client) )
	{
		// create a new player replication info
		InitPlayerState();
	}

	SpawnPlayerCameraManager();
	ResetCameraMode(); 

	if ( GetNetMode() == NM_Client )
	{
		SpawnDefaultHUD();
	}

	AddCheats();

	bPlayerIsWaiting = true;
	StateName = NAME_Spectating; // Don't use ChangeState, because we want to defer spawning the SpectatorPawn until the Player is received
}
```



我们先看一下HUD在哪里生成的.
``` cpp
void AGameModeBase::GenericPlayerInitialization(AController* C)
{
	APlayerController* PC = Cast<APlayerController>(C);
	if (PC != nullptr)
	{
		InitializeHUDForPlayer(PC);

		// Notify the game that we can now be muted and mute others
		UpdateGameplayMuteList(PC);

		if (GameSession != nullptr)
		{
			// Tell the player to enable voice by default or use the push to talk method
			PC->ClientEnableNetworkVoice(!GameSession->RequiresPushToTalk());
		}
		//....
	}
}
``` 
注意这里Lyra的多添加了一个触发代理
``` cpp
void ALyraGameMode::GenericPlayerInitialization(AController* NewPlayer)
{
	Super::GenericPlayerInitialization(NewPlayer);

	OnGameModePlayerInitialized.Broadcast(this, NewPlayer);
}

```

Lyra是在这里把Pawn的生成流程断掉的.这个Super不会执行.直至体验加载完成.以下代码在刚开始启动时不会执行.
``` cpp
void ALyraGameMode::HandleStartingNewPlayer_Implementation(APlayerController* NewPlayer)
{
	// Delay starting new players until the experience has been loaded
	// (players who log in prior to that will be started by OnExperienceLoaded)
	// 在加载游戏体验之前，先延迟启动新玩家
	// （在该过程之前登录的玩家将由“加载体验完成时启动”功能来启动）
	if (IsExperienceLoaded())
	{
		Super::HandleStartingNewPlayer_Implementation(NewPlayer);
	}
}

```
```cpp

void AGameModeBase::HandleStartingNewPlayer_Implementation(APlayerController* NewPlayer)
{
	// If players should start as spectators, leave them in the spectator state
	if (!bStartPlayersAsSpectators && !MustSpectate(NewPlayer) && PlayerCanRestart(NewPlayer))
	{
		// Otherwise spawn their pawn immediately
		RestartPlayer(NewPlayer);
	}
}
```





### Pawn创建流程
下一帧!!!
``` txt
ALyraGameMode::HandleMatchAssignmentIfNotExpectingOne() LyraGameMode.cpp:108
TBaseUObjectMethodDelegateInstance::Execute() DelegateInstancesImpl.h:672
[内联] TDelegate::Execute() DelegateSignatureImpl.inl:614
FTimerUnifiedDelegate::Execute() TimerManager.cpp:348
FTimerManager::Tick(float) TimerManager.cpp:1062
UWorld::Tick(ELevelTick, float) LevelTick.cpp:1577
UEditorEngine::Tick(float, bool) EditorEngine.cpp:2149
UUnrealEdEngine::Tick(float, bool) UnrealEdEngine.cpp:530
ULyraEditorEngine::Tick(float, bool) LyraEditorEngine.cpp:37
FEngineLoop::Tick() LaunchEngineLoop.cpp:5619
[内联] EngineTick() Launch.cpp:60
GuardedMain(const wchar_t *) Launch.cpp:189
LaunchWindowsStartup(HINSTANCE__ *, HINSTANCE__ *, char *, int, const wchar_t *) LaunchWindows.cpp:271
WinMain(HINSTANCE__ *, HINSTANCE__ *, char *, int) LaunchWindows.cpp:339
```
现在我们要开始创建我们的Pawn了.
实际上我们的调用流程在体验回调里面执行的.而非上述流程.
``` cpp
void ALyraGameMode::OnExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience)
{
	// Spawn any players that are already attached
	// 生成所有已附着的玩家对象
	
	//@TODO: Here we're handling only *player* controllers, but in GetDefaultPawnClassForController_Implementation we skipped all controllers
	//@待办事项：在这里我们只处理玩家控制器，但在 GetDefaultPawnClassForController_Implementation 函数中我们忽略了所有控制器。
	// GetDefaultPawnClassForController_Implementation might only be getting called for players anyways
	// “GetDefaultPawnClassForController_Implementation” 方法实际上可能只是在为玩家调用的，所以这种情况是完全有可能发生的。

	// 比如AI控制呢?
	
	for (FConstPlayerControllerIterator Iterator = GetWorld()->GetPlayerControllerIterator(); Iterator; ++Iterator)
	{
		APlayerController* PC = Cast<APlayerController>(*Iterator);
		if ((PC != nullptr) && (PC->GetPawn() == nullptr))
		{
			if (PlayerCanRestart(PC))
			{
				RestartPlayer(PC);
			}
		}
	}
}
```

#### 寻找出生点
``` cpp
void AGameModeBase::RestartPlayer(AController* NewPlayer)
{
	if (NewPlayer == nullptr || NewPlayer->IsPendingKillPending())
	{
		return;
	}

	AActor* StartSpot = FindPlayerStart(NewPlayer);

	// If a start spot wasn't found,
	if (StartSpot == nullptr)
	{
		// Check for a previously assigned spot
		if (NewPlayer->StartSpot != nullptr)
		{
			StartSpot = NewPlayer->StartSpot.Get();
			UE_LOG(LogGameMode, Warning, TEXT("RestartPlayer: Player start not found, using last start spot"));
		}	
	}

	RestartPlayerAtPlayerStart(NewPlayer, StartSpot);
}
```
``` cpp
AActor* AGameModeBase::FindPlayerStart_Implementation(AController* Player, const FString& IncomingName)
{
	UWorld* World = GetWorld();

	// ...
	AActor* BestStart = ChoosePlayerStart(Player);
	// ...
	return BestStart;
}

```
我们重写了选择玩家出生点这个方法.转发给了我们的组件执行.
``` cpp
AActor* ALyraGameMode::ChoosePlayerStart_Implementation(AController* Player)
{
	if (ULyraPlayerSpawningManagerComponent* PlayerSpawningComponent = GameState->FindComponentByClass<ULyraPlayerSpawningManagerComponent>())
	{
		return PlayerSpawningComponent->ChoosePlayerStart(Player);
	}
	
	return Super::ChoosePlayerStart_Implementation(Player);
}

```
#### 生成Pawn
此时我们的玩家控制器时没有Pawn.所以需要重新生成.
``` cpp
void AGameModeBase::RestartPlayerAtPlayerStart(AController* NewPlayer, AActor* StartSpot)
{
	if (NewPlayer == nullptr || NewPlayer->IsPendingKillPending())
	{
		return;
	}

	if (!StartSpot)
	{
		UE_LOG(LogGameMode, Warning, TEXT("RestartPlayerAtPlayerStart: Player start not found"));
		return;
	}

	FRotator SpawnRotation = StartSpot->GetActorRotation();

	UE_LOG(LogGameMode, Verbose, TEXT("RestartPlayerAtPlayerStart %s"), (NewPlayer && NewPlayer->PlayerState) ? *NewPlayer->PlayerState->GetPlayerName() : TEXT("Unknown"));

	if (MustSpectate(Cast<APlayerController>(NewPlayer)))
	{
		UE_LOG(LogGameMode, Verbose, TEXT("RestartPlayerAtPlayerStart: Tried to restart a spectator-only player!"));
		return;
	}

	if (NewPlayer->GetPawn() != nullptr)
	{
		// If we have an existing pawn, just use it's rotation
		SpawnRotation = NewPlayer->GetPawn()->GetActorRotation();
	}
	else if (GetDefaultPawnClassForController(NewPlayer) != nullptr)
	{
		// Try to create a pawn to use of the default class for this player
		APawn* NewPawn = SpawnDefaultPawnFor(NewPlayer, StartSpot);
		if (IsValid(NewPawn))
		{
			NewPlayer->SetPawn(NewPawn);
		}
	}
	
	if (!IsValid(NewPlayer->GetPawn()))
	{
		FailedToRestartPlayer(NewPlayer);
	}
	else
	{
		// Tell the start spot it was used
		InitStartSpot(StartSpot, NewPlayer);

		FinishRestartPlayer(NewPlayer, SpawnRotation);
	}
}

```

我们重写了获取玩家操作对面默认类的方法,从PawnData中获取:
SpawnDefaultPawnAtTransform_Implementation
GetDefaultPawnClassForController_Implementation

``` cpp
UClass* ALyraGameMode::GetDefaultPawnClassForController_Implementation(AController* InController)
{
	if (const ULyraPawnData* PawnData = GetPawnDataForController(InController))
	{
		if (PawnData->PawnClass)
		{
			return PawnData->PawnClass;
		}
	}

	return Super::GetDefaultPawnClassForController_Implementation(InController);
}

```

注意 在这个地方将我们的PawnData传递Pawn的拓展组件!非常重要
``` cpp
APawn* ALyraGameMode::SpawnDefaultPawnAtTransform_Implementation(AController* NewPlayer, const FTransform& SpawnTransform)
{
	FActorSpawnParameters SpawnInfo;
	SpawnInfo.Instigator = GetInstigator();
	SpawnInfo.ObjectFlags |= RF_Transient;	// Never save the default player pawns into a map.// 请勿将默认的玩家棋子保存到地图中。
	/* 决定是否运行构建脚本。如果为真，则不会在生成的角色上运行构建脚本。仅在角色是从蓝图中生成的情况下适用。*/
	// 延迟构造Actor
	SpawnInfo.bDeferConstruction = true;

	if (UClass* PawnClass = GetDefaultPawnClassForController(NewPlayer))
	{
		if (APawn* SpawnedPawn = GetWorld()->SpawnActor<APawn>(PawnClass, SpawnTransform, SpawnInfo))
		{
			if (ULyraPawnExtensionComponent* PawnExtComp = ULyraPawnExtensionComponent::FindPawnExtensionComponent(SpawnedPawn))
			{
				if (const ULyraPawnData* PawnData = GetPawnDataForController(NewPlayer))
				{
					// 很重要 传递了PawnData
					PawnExtComp->SetPawnData(PawnData);
				}
				else
				{
					UE_LOG(LogLyra, Error, TEXT("Game mode was unable to set PawnData on the spawned pawn [%s]."), *GetNameSafe(SpawnedPawn));
				}
			}

			SpawnedPawn->FinishSpawning(SpawnTransform);

			return SpawnedPawn;
		}
		else
		{
			UE_LOG(LogLyra, Error, TEXT("Game mode was unable to spawn Pawn of class [%s] at [%s]."), *GetNameSafe(PawnClass), *SpawnTransform.ToHumanReadableString());
		}
	}
	else
	{
		UE_LOG(LogLyra, Error, TEXT("Game mode was unable to spawn Pawn due to NULL pawn class."));
	}

	return nullptr;
}

```

#### 控制Pawn
在RestartPlayerAtPlayerStart中 生成好Pawn之后调用.
如果失败了也会调用FailedToRestartPlayer.
``` cpp
void ALyraGameMode::FinishRestartPlayer(AController* NewPlayer, const FRotator& StartRotation)
{
	if (ULyraPlayerSpawningManagerComponent* PlayerSpawningComponent = GameState->FindComponentByClass<ULyraPlayerSpawningManagerComponent>())
	{
		PlayerSpawningComponent->FinishRestartPlayer(NewPlayer, StartRotation);
	}

	Super::FinishRestartPlayer(NewPlayer, StartRotation);
}
```
``` cpp
void AGameModeBase::FinishRestartPlayer(AController* NewPlayer, const FRotator& StartRotation)
{
	NewPlayer->Possess(NewPlayer->GetPawn());

	// ...
}
``` 

### BeginPlay的时机
``` cpp
FGameInstancePIEResult UGameInstance::StartPlayInEditorGameInstance(ULocalPlayer* LocalPlayer, const FGameInstancePIEParameters& Params)
{
		//564
		SlowTask.EnterProgressFrame(10, NSLOCTEXT("UnrealEd", "PIEBeginPlay", "Starting PIE (Begin play)..."));
		PlayWorld->BeginPlay();
}

```
``` cpp

void UWorld::BeginPlay()
{
	// ...
	AGameModeBase* const GameMode = GetAuthGameMode();
	if (GameMode)
	{
		GameMode->StartPlay();
		if (GetAISystem())
		{
			GetAISystem()->StartPlay();
		}
	}
	// ...
}
```
``` cpp
void AGameModeBase::StartPlay()
{
	GameState->HandleBeginPlay();
}

```
``` cpp
void AGameStateBase::HandleBeginPlay()
{
	bReplicatedHasBegunPlay = true;

	GetWorldSettings()->NotifyBeginPlay();
	GetWorldSettings()->NotifyMatchStarted();
}

```

``` cpp
void AWorldSettings::NotifyBeginPlay()
{
	UWorld* World = GetWorld();
	if (!World->GetBegunPlay())
	{
		World->OnWorldPreBeginPlay.Broadcast();

		for (FActorIterator It(World); It; ++It)
		{
			SCOPE_CYCLE_COUNTER(STAT_ActorBeginPlay);
			const bool bFromLevelLoad = true;
			It->DispatchBeginPlay(bFromLevelLoad);
		}

		World->SetBegunPlay(true);
	}
}

```

如果是一个动态生成的Actor,它的BeginPlay时机是什么时候?
``` cpp
void AActor::PostActorConstruction()
{
				if (bRunBeginPlay)
				{
					SCOPE_CYCLE_COUNTER(STAT_ActorBeginPlay);
					DispatchBeginPlay();
				}
}

```


## DS的初始化启动
``` cpp
void ALyraGameMode::HandleMatchAssignmentIfNotExpectingOne()
{
		// 7.专属服务器体验覆盖
		if (TryDedicatedServerLogin())
		{
			// This will start to host as a dedicated server
			// 这将开始作为独立服务器进行运行
			return;
		}
}
```
主要通过以下两个函数实现.  
因为DS上没有玩家.  
所以初始化体验得流程不太一样.  
在视频内详细讲解内部实现代码.
``` cpp
	UE_API bool TryDedicatedServerLogin();
	UE_API void HostDedicatedServerMatch(ECommonSessionOnlineMode OnlineMode);

```
## 代码
``` cpp
/**
 * Post login event, triggered when a player or bot joins the game as well as after seamless and non seamless travel
 *
 * 登录后事件，当玩家或机器人加入游戏时触发，以及在无缝和非无缝移动之后触发。
 *
 * This is called after the player has finished initialization
 * 这是在玩家完成初始化操作之后所执行的程序。
 */
DECLARE_MULTICAST_DELEGATE_TwoParams(FOnLyraGameModePlayerInitialized, AGameModeBase* /*GameMode*/, AController* /*NewPlayer*/);

/**
 * ALyraGameMode
 *
 *	The base game mode class used by this project.
 *	该项目所使用的基础游戏模式类。
 */
UCLASS(MinimalAPI, Config = Game, Meta = (ShortTooltip = "The base game mode class used by this project."))
class ALyraGameMode : public AModularGameModeBase
{
	GENERATED_BODY()

public:
	//构造函数 用于指定项目使用的默认类
	UE_API ALyraGameMode(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	/*
	 * 从控制器拿到PawnData
	 * 可以AI控制器,也可以是玩家控制器
	 */
	UFUNCTION(BlueprintCallable, Category = "Lyra|Pawn")
	UE_API const ULyraPawnData* GetPawnDataForController(const AController* InController) const;

	//~AGameModeBase interface
	/**
	 * 初始化游戏。
	 * 在调用任何其他函数（包括 PreInitializeComponents() 函数）之前，会调用 GameMode 的 InitGame() 事件。
	 * 这个事件由 GameMode 使用来初始化参数并生成其辅助类。
	 * 注意：此事件在角色的 PreInitializeComponents 事件之前被调用。
	 */
	UE_API virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage) override;

	
	/**
	 * 返回给定控制器的默认兵类
	 * 我们重写了这个方法,当我们加载体验完成之后PawnData就应该可用了.通过它获取
	 * 初始化的时候,可能没有加载完成,那就现用默认的
	 * 
	 */
	UE_API virtual UClass* GetDefaultPawnClassForController_Implementation(AController* InController) override;
	
	/**
	 * 在“重启玩家”过程中调用此函数以实际生成玩家的兵种，当使用变换时
	 * @参数 新玩家 - 生成兵种的控制器
	 * @参数 生成变换 - 用于生成兵种的变换体
	 * @返回 一个默认兵种类的兵种
	 * 
	 */
	UE_API virtual APawn* SpawnDefaultPawnAtTransform_Implementation(AController* NewPlayer, const FTransform& SpawnTransform) override;
	
	/** 如果“查找玩家起点”操作应使用玩家身上所保存的“起点位置”而非调用“选择玩家起点”功能，则返回 true */
	UE_API virtual bool ShouldSpawnAtStartSpot(AController* Player) override;
	
	/** 表示玩家已准备好进入游戏，此时游戏可能会启动 */
	UE_API virtual void HandleStartingNewPlayer_Implementation(APlayerController* NewPlayer) override;
	
	/**
	 * 返回此玩家可从何处开始生成的“最佳”位置
	 * 默认实现会寻找一个随机且未被占用的位置*
	 * @参数 Player：我们为它所选择的控制角色
	 * @返回值：被选作玩家起始位置的 AActor 对象（通常为 PlayerStart 类型）
	 * 转发给LyraPlayerSpawningManagerComponent
	 *	 
	 */
	UE_API virtual AActor* ChoosePlayerStart_Implementation(AController* Player) override;

	/**
	 * 处理重新播放玩家的后半部分内容
	 * 转发给LyraPlayerSpawningManagerComponent
	 */
	UE_API virtual void FinishRestartPlayer(AController* NewPlayer, const FRotator& StartRotation) override;

	/**
	 * 若调用“重启玩家”操作是有效的，则返回 true。默认情况下会调用“玩家”的“可以重启玩家”方法
	 * 转发给LyraPlayerSpawningManagerComponent
	 */
	UE_API virtual bool PlayerCanRestart_Implementation(APlayerController* Player) override;
	

	// 绑定体验加载完成之后回调
	UE_API virtual void InitGameState() override;

	/**
	 * 尝试初始化玩家的“起始位置”。
	 * @参数  Player  玩家的控制器。
	 * @参数  OutErrorMessage  任何错误信息。
	 * @返回  bool  如果更新了起始位置则返回 true，否则返回 false。
	 * 
	 */
	UE_API virtual bool UpdatePlayerStartSpot(AController* Player, const FString& Portal, FString& OutErrorMessage) override;
	/**
	 * 负责所有在旅行方式之间共享的玩家初始化工作
	 * （即从 PostLogin() 和 HandleSeamlessTravelPlayer() 中调用）*
	 * 
	 */
	UE_API virtual void GenericPlayerInitialization(AController* NewPlayer) override;

	UE_API virtual void FailedToRestartPlayer(AController* NewPlayer) override;
	//~End of AGameModeBase interface

	// Restart (respawn) the specified player or bot next frame
	// - If bForceReset is true, the controller will be reset this frame (abandoning the currently possessed pawn, if any)
	UFUNCTION(BlueprintCallable)
	UE_API void RequestPlayerRestartNextFrame(AController* Controller, bool bForceReset = false);

	/*
	 * Agnostic version of PlayerCanRestart that can be used for both player bots and players
	 * “无神论版”的“玩家可重置”功能，可用于玩家机器人和玩家自身。
	 * 转发给LyraPlayerSpawningManagerComponent
	 */
	UE_API virtual bool ControllerCanRestart(AController* Controller);

	// Delegate called on player initialization, described above
	// 在玩家初始化时调用的委托函数，如上文所述
	FOnLyraGameModePlayerInitialized OnGameModePlayerInitialized;

protected:
	/*
	 * 体验加载完成后回调
	 * 它的调用可能晚于原引擎的玩家生成流程,所以需要重设玩家状态
	 * 在InitGameState中绑定.
	 */
	UE_API void OnExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience);
	// 体验是否加载完成 通过访问GameState的体验管理插件得知
	UE_API bool IsExperienceLoaded() const;

	/*
	 * 确定体验之后调用GameState的体验管理组件,在服务端设置体验,开始体验的加载流程.
	 * 
	 */
	UE_API void OnMatchAssignmentGiven(FPrimaryAssetId ExperienceId, const FString& ExperienceIdSource);

	/*
	 * 重载体验的选型
	 * 在InitGame的绑定到下一帧调用执行
	 * 包含两种路径,一种是用户自己拉起来游戏,还有一种是DS举行在线服务.
	 * 
	 */
	UE_API void HandleMatchAssignmentIfNotExpectingOne();

	// 专属服务器情况开始自动登录 等待登录结果 去触发体验重写开始主持游戏
	UE_API bool TryDedicatedServerLogin();
	UE_API void HostDedicatedServerMatch(ECommonSessionOnlineMode OnlineMode);

	/*
	 * 专属服务器情况下,本地用户注册登录的结果回调
	 */
	UFUNCTION()
	UE_API void OnUserInitializedForDedicatedServer(const UCommonUserInfo* UserInfo,
		bool bSuccess, FText Error, ECommonUserPrivilege RequestedPrivilege, ECommonUserOnlineContext OnlineContext);
};

#undef UE_API

```

## 总结
本节我们阅读了关卡启动时,GameMode做了哪些工作.  
这些都在UGameInstance::StartPlayInEditorGameInstance这个函数里面!  
关于HUD,PlayerController,PlayerState,GameState是如何在这个流程中被整合启动的.  
然后我们强调了以下BeginPlay的执行时机.  
总之,LyraGameMode主要是确保体验完成之后,加载我们的PawnData给我们的角色即可!