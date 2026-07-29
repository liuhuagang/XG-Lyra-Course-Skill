# Developer Settings and Simulation Platform

## UDeveloperSettings

UE's `UDeveloperSettings` is an engine-level configuration class system used to present configurable items in the Project Settings editor.

### Usage

```cpp
UCLASS(Config="Game", DefaultConfig)
class ULyraSomeSettings : public UDeveloperSettings
{
    GENERATED_BODY()

public:
    UPROPERTY(Config, EditDefaultsOnly, Category = "Settings")
    bool bSomeFlag;
};
```

### Example in Lyra

`ULyraAudioSettings` inherits from `UDeveloperSettings`, providing visual editing for audio configuration:

- `DefaultSoundClass` — Default sound class
- `DefaultSoundConcurrencyName` — Default sound concurrency settings
- `DefaultBaseSoundMix` — Base sound mix
- `MasterSubmix` — Master output submix

## Simulation Platform Settings

Lyra supports simulating different platform behaviors through editor tools for platform behavior verification during development:

### Simulation Scope

| Setting | Description |
|--------|------|
| Platform Input | Simulate target platform input mapping |
| Platform Performance | Simulate target platform performance limits |
| Platform Resolution | Simulate target platform screen resolution |
| Platform Storage | Simulate target platform file system |

### Implementation

Based on `UGameInstanceSubsystem` or editor tools, platform simulation options are added in developer settings. When simulation is enabled:

1. Override platform detection methods in `UGameInstanceSubsystem`
2. Replace platform-specific input/display settings in the development environment
3. Add toggle interface in the editor

## ULyraUserFacingExperienceDefinition

[ULyraUserFacingExperienceDefinition](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraUserFacingExperienceDefinition.h) inherits from `UPrimaryDataAsset`, used to present selectable game modes in the front-end UI.

### Definition

```cpp
UCLASS()
class ULyraUserFacingExperienceDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // Display name (shown in UI)
    UPROPERTY(EditDefaultsOnly, Category = Display)
    FText DisplayName;

    // Description text
    UPROPERTY(EditDefaultsOnly, Category = Display)
    FText Description;

    // Associated Experience
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TSoftObjectPtr<ULyraExperienceDefinition> Experience;

    // Associated map
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TSoftObjectPtr<UWorld> Map;

    // Extra parameters
    UPROPERTY(EditDefaultsOnly, Category = Gameplay)
    TMap<FString, FString> ExtraOptions;
};
```

### Usage

- List selectable game modes in the main menu UI (Control, Elimination, etc.)
- Provide Experience + Map + parameter combinations
- After user selection, trigger loading via `UGameInstance`'s `TryLoadExperience()`
