# Experience Framework & Loading Flow

## Overview

The Experience framework is the orchestration core of Lyra's startup flow. It decouples "game mode initialization" from the hardcoded GameMode, making it a configurable flow driven by data assets (ExperienceDefinition).

---

## Three-Tier Architecture

```
ExperienceDefinition (Data Layer)
    ↓ References
ExperienceManagerComponent (State Machine Layer)
    ↓ Executes
UGameFeatureAction (Action Layer, Engine Native Class)
```

---

## Tier 1: ExperienceDefinition

**File**: `Source/LyraGame/GameModes/LyraExperienceDefinition.h`

```cpp
UCLASS(BlueprintType, Meta = (DisplayName = "Lyra Experience Definition"))
class LYRAGAME_API ULyraExperienceDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // List of GameFeature plugins this Experience needs to load
    UPROPERTY(EditDefaultsOnly, Category = "Experience")
    TArray<FString> GameFeaturesToEnable;

    // Collection of GameFeatureActions this Experience needs to execute
    UPROPERTY(EditDefaultsOnly, Category = "Experience")
    TArray<TObjectPtr<ULyraExperienceActionSet>> ActionSets;

    // Default Gameplay classes
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay")
    TSubclassOf<APawn> PawnClass;

    UPROPERTY(EditDefaultsOnly, Category = "Gameplay")
    TSubclassOf<AHUD> HUDClass;

    UPROPERTY(EditDefaultsOnly, Category = "Gameplay")
    TSubclassOf<APlayerController> PlayerControllerClass;

    UPROPERTY(EditDefaultsOnly, Category = "Gameplay")
    TSubclassOf<APlayerState> PlayerStateClass;

    UPROPERTY(EditDefaultsOnly, Category = "Gameplay")
    TSubclassOf<AGameStateBase> GameStateClass;
};
```

ExperienceDefinition is stored as a data asset in the Content directory and can be configured in the editor. By switching different Experiences, you can completely change the GameFeature set and default Gameplay classes.

---

## Tier 2: ExperienceManagerComponent

**File**: `Source/LyraGame/GameModes/LyraExperienceManagerComponent.h`

ExperienceManagerComponent is mounted on GameState and manages the Experience loading state machine.

### State Transitions

```
Inactive → WaitingForAction → LoadingActions → Loaded → Deactivating
    ↑                                                     │
    └─────────────────────────────────────────────────────┘
```

### Key Methods

```cpp
UCLASS()
class LYRAGAME_API ULyraExperienceManagerComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    // Start loading an Experience
    void StartExperience(ULyraExperienceDefinition* Experience);

    // Current loading progress (0~1)
    float GetExperienceLoadProgress() const;

    // Check if loading is complete
    bool IsExperienceLoaded() const;

protected:
    // State machine advancement
    void OnExperienceLoaded();
    void OnExperienceFullLoadCompleted();

    // Action completion callback
    void OnActionActivationCompleted();
};
```

### Loading Flow

1. GameMode calls `StartExperience` during `InitGame`
2. ManagerComponent sets state to `WaitingForAction`
3. Iterates through the Experience's `ActionSets`, collecting all `UGameFeatureAction`
4. Activates each GameFeatureAction one by one, advancing state to `LoadingActions`
5. After all Actions complete, state advances to `Loaded`
6. GameMode receives `OnExperienceFullLoadCompleted` and begins Spawn Player

---

## Tier 3: UGameFeatureAction

Lyra does not define custom ExperienceAction classes; it directly uses the Engine's native `UGameFeatureAction` mechanism. `ULyraExperienceActionSet` is a `UDataAsset` that holds `TArray<TObjectPtr<UGameFeatureAction>>`, aggregating all Actions into a single asset.

**File**: `Source/LyraGame/GameModes/LyraExperienceActionSet.h`

```cpp
UCLASS()
class LYRAGAME_API ULyraExperienceActionSet : public UDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category = "Experience")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;
};
```

Standard `UGameFeatureAction` subclasses provided by the Engine include:

| Action | Function |
|--------|----------|
| UGameFeatureAction_AddComponents | Adds components to specified Actors |
| UGameFeatureAction_DataRegistry | Registers DataRegistry |
| UGameFeatureAction_WorldActionBase | World action base class |

### GameFeature Plugin Activation Flow

```
ExperienceManagerComponent::StartExperience()
    → Iterates ActionSets, collects all UGameFeatureAction
    → Calls OnGameFeatureActivating() for each GameFeatureAction
    → Engine loads .uplugin from the plugin
    → Executes plugin registration logic (AssetManager, PrimaryAsset)
    → State advances to Loaded
```

---

## Custom GameFeatureAction

Inherit `UGameFeatureAction` and inject custom logic in `OnGameFeatureActivating`/`OnGameFeatureDeactivating`:

```cpp
UCLASS()
class UMyGameFeatureAction_CustomInit : public UGameFeatureAction
{
    GENERATED_BODY()

protected:
    virtual void OnGameFeatureActivating() override
    {
        // Initialize custom manager after GameFeature loading completes
    }

    virtual void OnGameFeatureDeactivating() override
    {
        // Cleanup
    }
};
```

Custom GameFeatureActions can be referenced through the Actions array of `ULyraExperienceActionSet`, requiring no extra Lyra layer wrapper.

---

## GameMode Workflow

**File**: `Source/LyraGame/GameModes/LyraGameMode.h`

```
ALyraGameMode::InitGame()
    → Loads default ExperienceDefinition
    → ManagerComponent on GameState::StartExperience()

ALyraGameMode::OnExperienceLoaded()
    → Sets GameState, PlayerState classes, etc.
    → Subsequent SpawnPlayer uses Experience-configured classes
```

Complete flow:
```
InitGame
    → Server loads ExperienceDefinition
    → Activates GameFeature plugins
    → OnExperienceFullLoadCompleted
    → GameMode sets default classes
    → Waits for player join
    → SpawnPlayer → Assigns Pawn/Controller/HUD
```

---

## Key Design Points

1. **Experience can be switched at runtime** — ManagerComponent goes through Deactivating → Inactive cycle when switching
2. **Dynamic loading of GameFeature plugins** — Feature modules can be loaded on demand, reducing initial load time
3. **Serial activation of GameFeatureActions** — Each Action completes activation before the next executes, ensuring dependency order
4. **Data-driven initialization** — Configured via data assets, game behavior can be changed without modifying C++ code
