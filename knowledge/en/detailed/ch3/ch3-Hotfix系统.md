# Hotfix System

## Overview

Lyra's hotfix system implements runtime configuration patching through ULyraHotfixManager. Inheriting from UOnlineHotfixManager, it supports downloading hotfix files from online services and dynamically updating configuration and asset references.

## ULyraHotfixManager

Inherits from `UOnlineHotfixManager`, responsible for managing the online hotfix process.

### Key Methods

| Method | Purpose |
|------|------|
| StartHotfixProcess() | Start the hotfix process |
| OnHotfixCompleted() | Hotfix completion callback |
| WantsHotfixProcessing() | Determine whether to process a specified file |
| ApplyHotfixProcessing() | Apply hotfix for a file |
| HotfixIniFile() | Apply .ini file hotfix |
| PatchAssetsFromIniFiles() | Patch asset references from .ini files |
| RequestPatchAssetsFromIniFiles() | Request asset patching from .ini |
| OnHotfixAvailablityCheck() | Hotfix availability check callback |

### Hotfix Flow

```
StartHotfixProcess()
  → Download hotfix file list from online service
  → Process files one by one:
    → WantsHotfixProcessing() filtering
    → ApplyHotfixProcessing() application
      → .ini file → HotfixIniFile() update config
      → Asset file → PatchAssetsFromIniFiles() update references
  → OnHotfixCompleted() completion callback
    → Broadcast OnPendingGameHotfixChanged event
```

### Cache Directory

```cpp
virtual FString GetCachedDirectory() override
{
    return FPaths::ProjectPersistentDownloadDir() / TEXT("Hotfix/");
}
```

Hotfix files are cached in the `Hotfix/` subdirectory of the project's persistent download directory.

### Hotfix State Tracking

| Property | Purpose |
|------|------|
| bHasPendingGameHotfix | Whether there is a pending game hotfix |
| bHasPendingDeviceProfileHotfix | Whether there is a pending device profile hotfix |
| GameHotfixCounter | Game hotfix counter |

### ShouldWarnAboutMissing

When patching assets from .ini, `ShouldWarnAboutMissingWhenPatchingFromIni()` controls whether to warn about missing assets.

## LyraTextHotfixConfig

Located in `Hotfix/` directory, handles text content hotfix configuration.

## LyraRuntimeOptions

Located in `Hotfix/` directory, runtime option hotfix support.

## Integration with Settings

After hotfix completion, `ULyraSettingsLocal::OnHotfixDeviceProfileApplied()` is triggered to reapply device profile and frame rate settings:

```
Hotfix complete
  → OnHotfixCompleted()
    → OnPendingGameHotfixChanged event
      → ULyraSettingsLocal::OnHotfixDeviceProfileApplied()
        → ReapplyThingsDueToPossibleDeviceProfileChange()
          → UpdateGameModeDeviceProfileAndFps()
            → Update device profile suffix
            → Update frame rate mode
```
