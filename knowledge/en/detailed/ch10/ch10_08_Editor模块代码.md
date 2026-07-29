# Editor Module Code

## Overview

The LyraEditor module provides editor extensions, asset validation, command line tools, asset factories, and style systems. It serves as the editor-side infrastructure for the Lyra project.

## File Structure

- [LyraEditor.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/LyraEditor.cpp) — Module entry StartupModule
- [GameEditorStyle.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/Private/GameEditorStyle.h) / [GameEditorStyle.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/Private/GameEditorStyle.cpp) — Slate style
- [LyraEditorEngine.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/LyraEditorEngine.h) / [LyraEditorEngine.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/LyraEditorEngine.cpp) — Editor engine override
- [Validation/](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/Validation/) — Asset validation system
- [Commandlets/](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/Commandlets/) — ContentValidationCommandlet
- [Utilities/](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/Utilities/) — Editor utility commands
- [Private/](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/Private/) — AssetTypeActions / Factory

## FLyraEditorModule StartupModule

[LyraEditor.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/LyraEditor.cpp#L208-L234)

Executes in sequence on module startup:

1. **Initialize Style**: `FGameEditorStyle::Initialize()`
2. **GameplayAbilitiesEditor Delegate Binding**: Listens for the `GameplayAbilitiesEditor` module via module load callback, binds three GameplayCue editor delegates:
   - `GetGameplayCueNotifyClassesDelegate`: Returns `UGameplayCueNotify_Burst`, `AGameplayCueNotify_BurstLatent`, `AGameplayCueNotify_Looping`
   - `GetGameplayCueInterfaceClassesDelegate`: Scans all AActor subclasses implementing `UGameplayCueInterface`
   - `GetGameplayCueNotifyPathDelegate`: Gets path from `UAbilitySystemGlobals::GameplayCueNotifyPaths`
3. **PIE Event Binding**: Binds `FEditorDelegates::BeginPIE` / `EndPIE`, notifies `ULyraExperienceManager` on PIE start
4. **Register Asset Type**: Registers `FAssetTypeActions_LyraContextEffectsLibrary`
5. **Register Toolbar Menu**: `RegisterGameEditorMenus()`

## FGameEditorStyle Slate Style

[GameEditorStyle.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/Private/GameEditorStyle.h) / [GameEditorStyle.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/Private/GameEditorStyle.cpp)

Pure static class, manages an `FSlateStyleSet` named `GameEditorStyle`:

- Defines `IMAGE_BRUSH` / `BOX_BRUSH` / `BORDER_BRUSH` (engine path) macros
- Defines `GAME_IMAGE_BRUSH` / `GAME_IMAGE_BRUSH_SVG` (project path) macros
- Registers icon resources: `"GameEditor.CheckContent"` uses project SVG icon

## Toolbar Menu RegisterGameEditorMenus

Inserts three extensions after the `LevelEditor.LevelEditorToolBar.PlayToolBar` toolbar:

- **Check Content Button**: Calls `UEditorValidator::ValidateCheckedOutContent()`
- **Common Maps Dropdown**: Reads common map list from `ULyraDeveloperSettings::CommonEditorMaps`

## ULyraEditorEngine

[LyraEditorEngine.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/LyraEditorEngine.h) / [LyraEditorEngine.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/LyraEditorEngine.cpp)

Inherits from `UUnrealEdEngine`, overrides key virtual functions:

- **FirstTickSetup**: Single execution, forces `UContentBrowserSettings::DisplayPluginFolders` to enabled
- **PreCreatePIEInstances**: Intercepts before PIE creation; if `ALyraWorldSettings::ForceStandaloneNetMode` is true, forces NetMode to `PIE_Standalone`; then notifies `ULyraDeveloperSettings` and `ULyraPlatformEmulationSettings` of `OnPlayInEditorStarted()`

## PIE Experience Management Forwarding

[LyraExperienceManager.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraExperienceManager.h) / [LyraExperienceManager.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraExperienceManager.cpp)

`OnPlayInEditorBegun` method is called by `FLyraEditorModule` at PIE start, notifying ExperienceManager for initialization:

```cpp
void ULyraExperienceManager::OnPlayInEditorBegun()
{
    // Experience management initialization at PIE start
    NotifyOfPluginActivation(...);
    RequestToDeactivatePlugin(...);
}
```

## Validation System

Path: [LyraEditor/Validation/](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/Validation/)

Uses the strategy pattern, base class `UEditorValidator` inherits from `UEditorValidatorBase`.

### UEditorValidator Base Class

[EditorValidator.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/Validation/EditorValidator.h) / [EditorValidator.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/Validation/EditorValidator.cpp)

- **FLyraValidationMessageGatherer**: Internal class, inherits `FOutputDevice`, hooks into `GLog` to capture Warning and lower level logs
- **ValidateCheckedOutContent**: Gets changed files from source control; for .h changes, uses `GetChangedAssetsForCode()` to find affected Blueprints
- **ValidatePackages**: Loads and validates asset packages, pre-loads to listen for Load warnings, then calls `UEditorValidatorSubsystem::ValidateAssetsWithSettings()`
- **ValidateProjectSettings**: Checks if Python `bDeveloperMode` is enabled (should not be committed)
- **GetChangedAssetsForCode**: Finds affected Blueprints via header file changes (native class in header → derived Blueprint)

### UEditorValidator_SourceControl

[EditorValidator_SourceControl.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/Validation/EditorValidator_SourceControl.h) / [EditorValidator_SourceControl.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra/Source/LyraEditor/Validation/EditorValidator_SourceControl.cpp)

Validates that assets committed to source control do not reference assets not added to source control.

### UEditorValidator_Blueprints

[EditorValidator_Blueprints.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra/Source/LyraEditor/Validation/EditorValidator_Blueprints.h) / [EditorValidator_Blueprints.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra/Source/LyraEditor/Validation/EditorValidator_Blueprints.cpp)

When a non-data-only Blueprint changes, cascade checks all non-data-only Blueprints that hard-referenced it for compilation errors.

### UEditorValidator_Load

[EditorValidator_Load.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra/Source/LyraEditor/Validation/EditorValidator_Load.h) / [EditorValidator_Load.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra/Source/LyraEditor/Validation/EditorValidator_Load.cpp)

Validates whether assets produce warnings or errors when loaded. Core strategy: if the asset is already in memory, copy it to a temporary path and reload to capture load warnings.

### UEditorValidator_MaterialFunctions

[EditorValidator_MaterialFunctions.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra/Source/LyraEditor/Validation/EditorValidator_MaterialFunctions.h) / [EditorValidator_MaterialFunctions.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra/Source/LyraEditor/Validation/EditorValidator_MaterialFunctions.cpp)

When a `UMaterialFunction` changes, cascade checks all `UMaterial` instances that hard-referenced it for compilation errors.

## Command Line Tools

### UContentValidationCommandlet

[ContentValidationCommandlet.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra/Source/LyraEditor/Commandlets/ContentValidationCommandlet.cpp)

Content validation Commandlet, integrates with Perforce source control:

- Executes batch asset validation from the command line
- Checks assets in the changed file list
- Integrates with P4 to obtain changeset information

### Editor Utility Commands

[Utilities/](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra/Source/LyraEditor/Utilities/)

- **Lyra.CheckChaosMeshCollision**: Checks Chaos mesh collision settings
- **Lyra.CreateRedirectorPackage**: Creates redirector packages
- **Lyra.DiffCollectionReferenceSupport**: Collection reference diff comparison tool

These commands are registered via `FAutoConsoleCommand` and are available in the editor console.

## ContextEffects Editor Support

- [AssetTypeActions_LyraContextEffectsLibrary.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra/Source/LyraEditor/Private/AssetTypeActions_LyraContextEffectsLibrary.h) — Asset type actions, categorized as `EAssetTypeCategories::Gameplay`
- [LyraContextEffectsLibraryFactory.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra/Source/LyraEditor/Private/LyraContextEffectsLibraryFactory.h) — Right-click create `ULyraContextEffectsLibrary` asset in Content Browser
