# GameFeature System

## Overview

GameFeature is UE5's plugin-level dynamic content loading system, allowing game features to be split into independent plugins that can be loaded/unloaded at runtime on demand. Lyra implements ULyraGameFeaturePolicy as the project-level loading policy and uses multiple GameFeatureAction subclasses to define specific behaviors during loading.

## GameFeature Fundamentals

### Plugin Structure

GameFeature plugins are standard UE plugins located in `Plugins/GameFeatures/`. Each plugin contains:

- `.uplugin` plugin descriptor file
- `Content/` asset directory
- `Source/` source code directory

### Chunking and Content Packaging

GameFeature plugins support:
- **Chunking**: Distribute plugin assets into independent Patches/Chunks for on-demand downloading
- **ContentBundle**: Content bundle mechanism, manages loading/unloading of plugin assets
- **DataRegistry**: Data registry, manages table data in plugins
- **InputConfig**: Input configuration, registers plugin-specific input mappings
- **GameplayCueManager**: GameplayCue path registration
- **CommonUILayout**: Common UI layout activation

### ContentBundle

UContentBundleEngineSubsystem is the engine subsystem for content bundles, managing the lifecycle of ContentBundles in GameFeature plugins:

```
ContentBundle registration → Bundle registered when plugin is installed
  → Bundle activation → Activated when Experience loads
    → Asset loading → Bundle's associated assets loaded into memory
  → Bundle deactivation → Deactivated when plugin is unloaded
```

## ULyraGameFeaturePolicy

Inherits from `UDefaultGameFeaturesProjectPolicies`, the GameFeature loading policy singleton for the Lyra project.

### Key Methods

| Method | Purpose |
|------|------|
| InitGameFeatureManager() | Initialize the GameFeature manager |
| ShutdownGameFeatureManager() | Shutdown the GameFeature manager |
| GetPreloadAssetListForGameFeature() | Get the list of assets to preload for a GameFeature plugin |
| IsPluginAllowed() | Determine if a specified plugin is allowed to load |
| GetPreloadBundleStateForGameFeature() | Get the preload Bundle state |
| GetGameFeatureLoadingMode() | Get the loading mode (client/server) |

### Observer Mechanism

ULyraGameFeaturePolicy manages a set of Observers that listen for GameFeature plugin state changes:

| Observer Class | Purpose |
|-------------|------|
| ULyraGameFeature_HotfixManager | Execute hotfix when GameFeature loads |
| ULyraGameFeature_AddGameplayCuePaths | Add/remove GameplayCue paths when GameFeature registers/unregisters |

## GameFeatureAction

GameFeatureAction defines specific behaviors when a GameFeature plugin loads/activates.

### GameFeatureActions in the Main Project

| Class | Inherits From | Purpose |
|------|--------|------|
| GameFeatureAction_WorldActionBase | UGameFeatureAction | World action base class |
| GameFeatureAction_AddAbilities | UGameFeatureAction | Add GameplayAbility, effects, attribute sets |
| GameFeatureAction_AddInputBinding | UGameFeatureAction | Add input binding mappings |
| GameFeatureAction_AddInputContextMapping | UGameFeatureAction | Add input context mappings |
| GameFeatureAction_AddWidget | UGameFeatureAction | Add UI widgets |
| GameFeatureAction_AddGameplayCuePath | UGameFeatureAction | Add GameplayCue paths |
| GameFeatureAction_SplitscreenConfig | UGameFeatureAction | Splitscreen configuration |

### GameFeatureAction Lifecycle

```
Plugin registering → OnGameFeatureRegistering
  → Plugin loading → OnGameFeatureLoading
    → Plugin activating → OnGameFeatureActivating
      → GameFeatureAction execution (AddAbilities/AddWidget, etc.)
    → Plugin deactivating → OnGameFeatureDeactivating
      → GameFeatureAction rollback
  → Plugin unloading → OnGameFeatureUnload
→ Plugin unregistering → OnGameFeatureUnregistering
```

## UGameFeatureAction_AddGameplayCuePath

Used to register additional GameplayCue search paths with the GameplayCue manager:

```cpp
UPROPERTY(EditAnywhere, Category = "Game Feature | Gameplay Cues", meta = (RelativeToGameContentDir, LongPackageName))
TArray<FDirectoryPath> DirectoryPathsToAdd;
```

Paths are relative to the game content directory, using long package name format.

## AI ActorFactory

GameFeature plugins can create custom AI characters via the ActorFactory mechanism, encapsulating character definition and spawning logic within the plugin.

## Plugin Example: TopDownArena

Located in `Plugins/GameFeatures/TopDownArena/`, demonstrates a complete GameFeature plugin implementation:

| Class | Purpose |
|------|------|
| TopDownArenaRuntimeModule | Plugin module |
| ALyraCameraMode_TopDownArenaCamera | Top-down camera mode |
| ATopDownArenaMovementComponent | Top-down movement component |
| UTopDownArenaAttributeSet | Top-down attribute set |
| UTopDownArenaPickupUIData | Pickup UI data |
