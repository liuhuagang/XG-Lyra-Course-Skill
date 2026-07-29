# Experience Definition and Loading

## Overview

Experience is Lyra's core design pattern. It uses `ULyraExperienceDefinition` data assets to replace the traditional hardcoded GameMode gameplay loading logic, allowing each map/game mode to independently declare the GameFeature plugins, GameplayAbilities, AttributeSets, GameplayEffects, PawnData, etc., that need to be loaded.

## ExperienceDefinition Data Asset

[ULyraExperienceDefinition](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraExperienceDefinition.h) inherits from `UPrimaryDataAsset` and defines what an Experience contains:

```cpp
UCLASS()
class ULyraExperienceDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // List of GameFeature plugins associated with this Experience
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TArray<FString> GameFeaturesToEnable;

    // Default PawnData (defines character attributes, animation, camera, etc.)
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TObjectPtr<const ULyraPawnData> DefaultPawnData;

    // GameplayAbilities granted when Experience loads
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TArray<TSoftClassPtr<ULyraGameplayAbility>> GameplayAbilities;

    // AttributeSets granted when Experience loads
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TArray<TSoftClassPtr<ULyraAttributeSet>> AttributeSets;

    // GameplayEffects applied when Experience loads
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TArray<TSoftClassPtr<UGameplayEffect>> GameplayEffects;

    // Experience action sets (further encapsulation of GA/GE/Attribute collections)
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TArray<TObjectPtr<ULyraExperienceActionSet>> Actions;

    // Custom actions when Experience loads (deprecated)
    UPROPERTY(EditDefaultsOnly, Instanced, Category = Actions)
    TArray<TObjectPtr<ULyraExperienceAction>> Actions_DEPRECATED;

    // Custom actions when Experience loads (new version)
    UPROPERTY(EditDefaultsOnly, Instanced, Category = Actions)
    TArray<TObjectPtr<ULyraExperienceAction>> ExperienceActions;
};
```

### ExperienceActionSet

[ULyraExperienceActionSet](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraExperienceActionSet.h) is a `UPrimaryDataAsset` that packages multiple GA/GE/AttributeSets into a reusable collection. Multiple Experiences can share the same ActionSet.

## ExperienceManagerComponent Load Flow

[ULyraExperienceManagerComponent](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraExperienceManagerComponent.h) is attached to `ALyraGameState`, inherits from `UGameStateComponent`, and implements `ILoadingProcessInterface`.

### Load States

The Experience loading process tracks state via the `ELyraExperienceLoadState` enum:

- `Unloaded` — Not loaded
- `Loading` — Loading
- `LoadingGameFeatures` — Activating GameFeature plugins
- `LoadingChaosTestingWait` — Waiting for chaos testing
- `ExecutingActions` — Executing Experience actions
- `Loaded` — Loading complete
- `Deactivate` — Deactivating

### Core Load Flow

```cpp
// Request to start loading Experience
void ULyraExperienceManagerComponent::StartExperienceLoad(ULyraExperienceDefinition* Experience)
{
    CurrentExperience = Experience;
    SetCurrentState(ELyraExperienceLoadState::Loading);

    // Load and activate GameFeature plugins
    // Async—calls UE's GameFeature plugin API
    for (FString& PluginURL : Experience->GameFeaturesToEnable)
    {
        // Convert plugin name to URL
        FString PluginURL;
        UGameFeaturesSubsystem::Get().GetPluginURLByName(PluginName, PluginURL);
        // Load and activate...
    }

    // Async load all GameplayAbilities, AttributeSets, GameplayEffects
    // LoadPackages uses FSoftObjectPath to check and async load each
}
```

### After Loading Completes

When all plugins are loaded and assets are ready:

```cpp
void ULyraExperienceManagerComponent::OnExperienceLoadComplete()
{
    // Grant abilities and attributes defined in the Experience
    // Register GA/GE/AttributeSet with the global ability system
    // Broadcast OnExperienceLoaded event
    SetCurrentState(ELyraExperienceLoadState::ExecutingActions);
    // Execute ExperienceActions...
}
```

## Experience Management in GameInstance

[ULyraGameInstance](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraGameInstance.h) inherits from `UCommonGameInstance` and is responsible for user login and Experience lifecycle management:

```cpp
void ULyraGameInstance::BeginLoginAttempt(ULocalPlayer* LocalPlayer, ...)
{
    // Check if online subsystem login is needed
    // Call OnUserLoginComplete callback
}
```

### Async Experience Loading

`ULyraUserFacingExperienceDefinition` in `ULyraGameInstance` provides the ability to trigger Experience loading from the front-end UI—via `TryLoadExperience()` after the user selects a mode/map.

## Default Experience in WorldSettings

[ALyraWorldSettings](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraWorldSettings.h) inherits from `AWorldSettings` and defines the default Experience:

```cpp
UCLASS()
class ALyraWorldSettings : public AWorldSettings
{
    // Default Experience used by the map
    UPROPERTY(EditDefaultsOnly, Category = Game)
    TSoftObjectPtr<ULyraExperienceDefinition> DefaultExperience;
};
```

### OnPacketHandling Error Handling

`ALyraWorldSettings` also overrides the error handling logic for `OnPacketHandling`:

```cpp
virtual void OnPacketHandling(EPacketHandling& OutHandling, bool& OutRemoveNetworkActors, bool& bForceFastArrayDev) override;
```
