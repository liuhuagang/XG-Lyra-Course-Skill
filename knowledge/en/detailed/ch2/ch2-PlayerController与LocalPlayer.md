# PlayerController and LocalPlayer

## ALyraPlayerController

[ALyraPlayerController](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Player/LyraPlayerController.h) inherits from `ACommonPlayerController` and implements both `ILyraCameraAssistInterface` and `ILyraTeamAgentInterface`.

### Constructor

```cpp
ALyraPlayerController::ALyraPlayerController(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    // Specify camera manager class
    PlayerCameraManagerClass = ALyraPlayerCameraManager::StaticClass();

#if USING_CHEAT_MANAGER
    // Specify cheat manager class
    CheatClass = ULyraCheatManager::StaticClass();
#endif
}
```

### Core Responsibilities

1. **Input forwarding** — Receive player input and forward to `ULyraHeroComponent`
2. **Camera management** — Coordinate with `ALyraPlayerCameraManager` to manage camera modes
3. **GAS timing** — Ensure ASC is initialized at the correct time
4. **Team binding** — Implement `ILyraTeamAgentInterface`
5. **Spectator synchronization** — Manage network replication of `TargetViewRotation`
6. **HttpServer registration** — Start RPC listener (testing purposes)
7. **Replay recording** — Control replay start/stop
8. **Force feedback** — Handle controller force feedback settings

### HttpServer Registration

Register HTTP routes in `BeginPlay()` for automated testing:

```cpp
void ALyraPlayerController::BeginPlay()
{
    Super::BeginPlay();

#if WITH_RPC_REGISTRY
    FHttpServerModule::Get().StartAllListeners();

    int32 RpcPort = 0;
    if (FParse::Value(FCommandLine::Get(), TEXT("rpcport="), RpcPort))
    {
        ULyraGameplayRpcRegistrationComponent* ObjectInstance = 
            ULyraGameplayRpcRegistrationComponent::GetInstance();
        if (ObjectInstance && ObjectInstance->IsValidLowLevel())
        {
            ObjectInstance->RegisterAlwaysOnHttpCallbacks();
            ObjectInstance->RegisterInMatchHttpCallbacks();
        }
    }
#endif
    SetActorHiddenInGame(false);
}
```

Registered routes:

| Route | Method | Purpose |
|------|------|------|
| `/core/cheatcommand` | POST | Execute cheat commands remotely |
| `/player/status` | GET | Get player status |
| `/player/status` | POST | Remote fire |

### Spectator View Rotation

Use `TargetViewRotation` replicated property to sync non-controlled view rotation:

```cpp
UPROPERTY(replicated)
FRotator TargetViewRotation;

// Only replicate to owner
DOREPLIFETIME_WITH_PARAMS_FAST(APlayerController, TargetViewRotation, Params);
```

### Ensuring ASC Timing

`ALyraPlayerController` handles GAS initialization timing:

1. **When PC is created** — Bind `OnPossess` / `OnUnPossess` events
2. **Possess Pawn** — Check if ASC on Pawn is ready
3. **Dedicated Server timing** — Server side needs ASC initialized immediately after Pawn creation
4. **Input processing** — Input handling in `ULyraAbilitySystemComponent`

### Setting Team Binding

Implemented via `ILyraTeamAgentInterface`:

```cpp
void ALyraPlayerController::SetGenericTeamId(const FGenericTeamId& NewTeamID)
{
    // Update team information
    // Notify team subsystem
    // Set force feedback
}
```

### Force Feedback Settings

Force feedback intensity controlled via console variable `CVarForceFeedbackScale`:

```cpp
static float CVarForceFeedbackScale = 1.0f;
FAutoConsoleVariableRef CVarForceFeedback(
    TEXT("PlayerController.ForceFeedbackScale"),
    CVarForceFeedbackScale,
    TEXT("Force feedback scale factor"));
```

Override `APlayerController::PlayDynamicForceFeedback` method to support custom force feedback logic.

### Camera Penetration

Handling camera penetration when objects block the view:

1. `ALyraPlayerCameraManager` detects occlusion
2. Call `ILyraCameraAssistInterface` interface methods
3. Determine handling based on occluder type

### Replay

```cpp
void ALyraPlayerController::OnPossess(APawn* InPawn)
{
    Super::OnPossess(InPawn);

    // Check if replay should be recorded
    if (ShouldRecordReplay())
    {
        StartRecordingReplay();
    }
}
```

## ULyraLocalPlayer

[ULyraLocalPlayer](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Player/LyraLocalPlayer.h) inherits from `UCommonLocalPlayer`.

### Responsibilities

1. **Holds player settings** — Manages `ULyraSettingsShared` (cross-platform synced settings)
2. **Local player identity** — Identifies the local player on the current machine
3. **Platform binding** — Handles platform-level player operations

### UCommonLocalPlayer vs ULocalPlayer

| Feature | ULocalPlayer | UCommonLocalPlayer |
|------|-------------|-------------------|
| Platform Player | Basic platform abstraction | Added CommonUI integration |
| Input Handling | Basic input | Enhanced CommonInput support |
| Settings | Basic settings | Cross-platform settings sync |

## ALyraReplayPlayerController

Replay-specific PlayerController, inheriting from `ALyraPlayerController`, for replay scenarios:
- Updated ViewTarget logic
- Disable input during playback
- Listen for replay state changes
