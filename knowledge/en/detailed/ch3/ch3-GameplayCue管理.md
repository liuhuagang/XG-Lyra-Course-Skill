# GameplayCue Management

## Overview

ULyraGameplayCueManager inherits from UGameplayCueManager and is Lyra's customized GameplayCue management implementation. It provides lazy loading, preload management, and GameplayCue path registration.

## ULyraGameplayCueManager

Inherits from `UGameplayCueManager`, located in the `AbilitySystem/` directory.

### Key Methods

| Method | Purpose |
|------|------|
| OnCreated() | Initialization on creation |
| ShouldAsyncLoadRuntimeObjectLibraries() | Whether to async load runtime object libraries |
| ShouldSyncLoadMissingGameplayCues() | Whether to sync load missing GameplayCues |
| ShouldAsyncLoadMissingGameplayCues() | Whether to async load missing GameplayCues |
| LoadAlwaysLoadedCues() | Load GameplayCues that always need to be loaded |
| RefreshGameplayCuePrimaryAsset() | Refresh the GameplayCue Primary Asset's Bundle |

### Lazy Loading Mechanism

Lyra implements a lazy loading strategy for GameplayCues:

1. Not all GameplayCues are loaded immediately at startup
2. `ShouldAsyncLoadRuntimeObjectLibraries()` controls whether to async load
3. When a GameplayTag is referenced, on-demand loading is triggered via the `OnGameplayTagLoaded` callback
4. Preloaded Cues are stored in the `PreloadedCues` collection
5. Code-referenced or explicitly declared Cues are stored in the `AlwaysLoadedCues` collection

### Preload Mechanism

```cpp
// Preloaded Cues (loaded early because they are referenced by content)
UPROPERTY(transient)
TSet<TObjectPtr<UClass>> PreloadedCues;

// Always loaded Cues (code-referenced or explicitly declared)
UPROPERTY(transient)
TSet<TObjectPtr<UClass>> AlwaysLoadedCues;
```

### Post-Garbage Collection Handling

`HandlePostGarbageCollect()` processes GameplayTags that need loading after GC, performing actual loading via `ProcessLoadedTags()`.

## GameplayCue Path Registration

GameplayCue search paths are extended through two mechanisms:

### 1. GameFeatureAction_AddGameplayCuePath

GameFeature plugins register custom GameplayCue search paths during loading via `UGameFeatureAction_AddGameplayCuePath`:

```cpp
UPROPERTY(EditAnywhere, Category = "Game Feature | Gameplay Cues", meta = (RelativeToGameContentDir, LongPackageName))
TArray<FDirectoryPath> DirectoryPathsToAdd;
```

### 2. ULyraGameFeature_AddGameplayCuePaths

GameFeature policy Observer that listens for GameFeature plugin register/unregister events and automatically adds/removes GameplayCue paths:

```cpp
OnGameFeatureRegistering()   // Add path on registration
OnGameFeatureUnregistering() // Remove path on unregistration
```

## Debugging

```cpp
static void DumpGameplayCues(const TArray<FString>& Args);
```

Provides a console command for exporting the GameplayCue list to assist with debugging.
