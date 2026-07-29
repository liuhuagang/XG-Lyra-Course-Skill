# Log System

## Log Category Definition

Log categories are defined using a pair of macros:

**Declaration in header (.h)**:
```cpp
LYRAGAME_API DECLARE_LOG_CATEGORY_EXTERN(LogLyra, Log, All);
```

**Definition in source (.cpp)**:
```cpp
DEFINE_LOG_CATEGORY(LogLyra);
```

## Log Category Locations

| Log Category | Default Level | File |
|----------|---------|---------|
| `LogLyra` | Log | [LyraLogChannels.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/LyraLogChannels.h) |
| `LogLyraExperience` | Log | [LyraLogChannels.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/LyraLogChannels.h) |
| `LogLyraAbilitySystem` | Log | [LyraLogChannels.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/LyraLogChannels.h) |
| `LogLyraTeams` | Log | [LyraLogChannels.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/LyraLogChannels.h) |
| `LogLyraEditor` | Log | LyraEditor.h |
| `LogLyraGamePhase` | Log | LyraGamePhaseLog.h |
| `LogLyraCheat` | Log | LyraCheatManager.h |
| `LogLyraGameSettingRegistry` | Log | LyraGameSettingRegistry.h |
| `LogLyraRepGraph` | Display | LyraReplicationGraph.h |

## Log Verbosity Levels

| Level | Console Output | Editor Log | Text Color | Description |
|------|-----------|-----------|---------|------|
| Fatal | Yes | N/A | N/A | Session crash |
| Error | Yes | Yes | Red | — |
| Warning | Yes | Yes | Yellow | — |
| Display | Yes | Yes | Gray | — |
| Log | No | Yes | Gray | Default level |
| Verbose | No | No | N/A | — |
| VeryVerbose | No | No | N/A | Log mask can set color |

## Static vs Regular Log Categories

```cpp
// Only accessible within the file it is defined in (file-local static)
DEFINE_LOG_CATEGORY_STATIC(SourceFilterPresets, Display, Display);

// Accessible via extern in other files
DEFINE_LOG_CATEGORY(LogLyra);
```

## Context Object Retrieval Methods

[LyraLogChannels.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/LyraLogChannels.h) defines a helper function:

```cpp
FString GetClientServerContextString(UObject* ContextObject = nullptr);
```

This function is called in `ULyraExperienceManagerComponent` for printing Experience loading logs. The object passed is typically the `GameState`, which has network synchronization properties.

### FWorldContext

`FWorldContext` is the engine-level context environment that manages `UWorld`:

- Each `WorldContext` can be thought of as a track
- Game engine: typically has only one `WorldContext`
- Editor engine: may have one for the editor world and one for the PIE world
- `GPlayInEditorContextString`: debug helper variable in PIE mode, updated during world switching/map loading
- `UGameplayMessageSubsystem::BroadcastMessageInternal` also accesses this variable to determine the world type where the message occurred
