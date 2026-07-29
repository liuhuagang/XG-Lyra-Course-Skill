# GameMode Workflow

## ALyraGameMode

[ALyraGameMode](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraGameMode.h) inherits from `AModularGameModeBase` and is the Lyra game mode base class.

## Initialization Flow

### InitGame

UE engine entry point for starting GameMode. Lyra binds Experience loading here:

```cpp
void ALyraGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
    Super::InitGame(MapName, Options, ErrorMessage);

    // Wait until next frame to give initialization startup settings time
    GetWorld()->GetTimerManager().SetTimerForNextTick(
        this, &ThisClass::HandleMatchAssignmentIfNotExpectingOne);
}
```

Call stack:
```
ALyraGameMode::InitGame
UWorld::InitializeActorsForPlay
UGameInstance::StartPlayInEditorGameInstance
UEditorEngine::CreateInnerProcessPIEGameInstance
UEditorEngine::CreateNewPlayInEditorInstance
UEditorEngine::StartPlayInEditorSession
UEditorEngine::Tick
UUnrealEdEngine::Tick
ULyraEditorEngine::Tick
FEngineLoop::Tick
```

### InitGameState

Called during the `RouteActorInitialize` phase. `ALyraGameMode` performs Experience-related state initialization in `InitGameState`.

### HandleMatchAssignmentIfNotExpectingOne

This function executes on the next frame after InitGame. Core logic:

```cpp
void ALyraGameMode::HandleMatchAssignmentIfNotExpectingOne()
{
    // Skip if there is already a pending Experience request
    if (bPendingAssignment) return;

    // Get default Experience from OptionsString or WorldSettings
    FString ExperienceName;
    if (!FParse::Value(OptionsString, TEXT("Experience="), ExperienceName))
    {
        // If not specified, get default from WorldSettings
        if (ALyraWorldSettings* WS = Cast<ALyraWorldSettings>(GetWorld()->GetWorldSettings()))
        {
            ExperienceName = WS->DefaultExperience.GetAssetName();
        }
    }

    // Check readiness (waiting for hotfix, GSI, and other subsystems)
    if (!IsReadyToProcessMatchAssignment())
    {
        // Not ready, retry next frame
        GetWorld()->GetTimerManager().SetTimerForNextTick(
            this, &ThisClass::HandleMatchAssignmentIfNotExpectingOne);
        return;
    }

    // Ready, start loading Experience
    bPendingAssignment = true;
    StartExperienceLoading(ExperienceName);
}
```

### IsReadyToProcessMatchAssignment

Checks whether the following subsystems are ready:
- Hotfix manager (`ULyraHotfixManager`) initialization complete
- GSI (GameStateInterface) available
- Other systems that need to complete before Experience loading

### OnExperienceLoaded Callback

When `ULyraExperienceManagerComponent` completes Experience loading, it notifies GameMode:

```cpp
void ALyraGameMode::OnExperienceLoaded(const ULyraExperienceDefinition* Experience)
{
    // Register Actor types from GameFeature plugins
    // Initialize bot creation component
    // Mark Experience loading as complete
}
```

From this point onward, `PreLogin` allows players to enter.

## OptionsString

`OptionsString` is the way GameMode passes startup parameters, stored in `AGameModeBase::OptionsString`. Used for:

- Passing Experience selection: `?Experience=ExperienceName`
- Passing map loading parameters
- Passing debug options

## Player Login Flow

### Creating Controller

```
PreLogin → Login → PostLogin
```

1. **PreLogin** — Verify if the player can join (requires Experience loading complete)
2. **Login** — Create PlayerController
3. **PostLogin** — Initialize player state, trigger player spawn

### Pawn Creation Flow

```
ALyraGameMode::HandleStartingNewPlayer
  → Call ULyraPlayerSpawningManagerComponent to get spawn point
  → Call SpawnDefaultPawnAtTransform
  → Create character (ALyraCharacter / ALyraPawn)
  → Possess Pawn
```

### Finding Spawn Point

Player spawn points use subclasses of `APlayerStart` (Lyra custom player spawn point types), centrally managed by `ULyraPlayerSpawningManagerComponent`.

## DS Initialization Startup

Dedicated Server startup flow:

```
Server → InitGame → Load Experience → Wait for players
       → Does not need to load UI resources
       → Does not need audio and rendering resources
```

## UI Loading Strategy During Experience

UI loading strategy needs proper handling during Experience loading:

- Use `AsyncMixin` to show a loading screen before loading completes
- Avoid rendering the main menu before Experience loading is complete
- `CommonLoadingScreen` plugin provides a unified loading screen

## Creating Game Base Classes

Game class overrides are registered via module load callback at the top of `LyraGameMode.cpp`:

```cpp
// Register default classes via StaticClass or FCoreDelegates
// Similar to UE GameMode default value settings
// Includes default GameState, PlayerController, PlayerState, HUD, etc.
```

### NearClipPlane

`GNearClippingPlane` is the engine's global clipping plane variable for setting the near clip plane distance:

```cpp
// Set near clip plane (smaller value allows seeing objects closer)
GNearClippingPlane = 15.0f;
```

In `ALyraPlayerCameraManager`, this value may be adjusted based on the current camera mode to avoid objects clipping through the camera.

## Final Flow Summary

```
1. UWorld::SetGameMode() → Create ALyraGameMode
2. UWorld::InitializeActorsForPlay() → Call InitGame()
3. InitGame() → SetTimerForNextTick → HandleMatchAssignmentIfNotExpectingOne
4. HandleMatchAssignment → Check readiness → Start loading Experience
5. ExperienceManagerComponent → Async load plugins and assets
6. OnExperienceLoaded → GameMode receives notification
7. RouteActorInitialize → InitGameState / PreInitializeComponents
8. Player login: PreLogin → Login → PostLogin
9. PostLogin → HandleStartingNewPlayer → Spawn Pawn → Possess
```
