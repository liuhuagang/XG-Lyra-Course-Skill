# Chapter 2: Experience & Gameplay Framework

## Overview

This chapter is the core of the Lyra architecture. The Experience system replaces the traditional hardcoded GameMode gameplay logic approach, using data assets to dynamically declare the various assets, GameplayAbilities, GameFeature plugins, and action sets that a game mode needs to load. During initialization, GameMode automatically triggers Experience loading, connecting the key objects in the Gameplay framework: GameState, PlayerState, PlayerController, LocalPlayer, and player spawning management.

## Lecture Scope

| Lecture ID | Title | Corresponding Knowledge File |
|------------|-------|------------------------------|
| 010 | Defining Experience | [ch2-Experience定义与加载](../detailed/ch2/ch2-Experience定义与加载.md) |
| 011 | Managing Experience | [ch2-Experience定义与加载](../detailed/ch2/ch2-Experience定义与加载.md) |
| 011_泽_01 | Async Asset Loading and Bundle | [ch2-AssetManager与异步加载](../detailed/ch2/ch2-AssetManager与异步加载.md) |
| 012 | AssetManager Class | [ch2-AssetManager与异步加载](../detailed/ch2/ch2-AssetManager与异步加载.md) |
| 012_泽_01 | TSoftObjectPtr and FSoftObjectPath | [ch2-AssetManager与异步加载](../detailed/ch2/ch2-AssetManager与异步加载.md) |
| 012_泽_02 | Performance Optimization Macros | [ch2-AssetManager与异步加载](../detailed/ch2/ch2-AssetManager与异步加载.md) |
| 013 | Game Instance | [ch2-GameInstance与登录流程](../detailed/ch2/ch2-GameInstance与登录流程.md) |
| 013_泽_01 | DTLS Concept | [ch2-GameInstance与登录流程](../detailed/ch2/ch2-GameInstance与登录流程.md) |
| 013_泽_02 | Token Verification Flow | [ch2-GameInstance与登录流程](../detailed/ch2/ch2-GameInstance与登录流程.md) |
| 014 | Creating Game Base Classes | [ch2-GameMode工作流程](../detailed/ch2/ch2-GameMode工作流程.md) |
| 014_泽_01 | NearClipPlane | [ch2-GameMode工作流程](../detailed/ch2/ch2-GameMode工作流程.md) |
| 015 | Developer Settings | [ch2-开发者设置与模拟平台](../detailed/ch2/ch2-开发者设置与模拟平台.md) |
| 016 | Simulated Platform Settings | [ch2-开发者设置与模拟平台](../detailed/ch2/ch2-开发者设置与模拟平台.md) |
| 017 | User Experience Definition | [ch2-开发者设置与模拟平台](../detailed/ch2/ch2-开发者设置与模拟平台.md) |
| 018 | Player Spawn Points | [ch2-玩家生成系统](../detailed/ch2/ch2-玩家生成系统.md) |
| 019 | Player Spawning Manager Component | [ch2-玩家生成系统](../detailed/ch2/ch2-玩家生成系统.md) |
| 020 | GameMode Workflow | [ch2-GameMode工作流程](../detailed/ch2/ch2-GameMode工作流程.md) |
| 020_泽_01 | OptionsString | [ch2-GameMode工作流程](../detailed/ch2/ch2-GameMode工作流程.md) |
| 021 | GameState Workflow | [ch2-GameState与PlayerState](../detailed/ch2/ch2-GameState与PlayerState.md) |
| 021_泽_01 | Common G-prefixed Global Variables | [ch2-GameState与PlayerState](../detailed/ch2/ch2-GameState与PlayerState.md) |
| 022 | Fixing UI Strategy Loading | [ch2-GameMode工作流程](../detailed/ch2/ch2-GameMode工作流程.md) |
| 023 | Async Experience Loading Inquiry | [ch2-Experience定义与加载](../detailed/ch2/ch2-Experience定义与加载.md) |
| 024 | PlayerState Workflow | [ch2-GameState与PlayerState](../detailed/ch2/ch2-GameState与PlayerState.md) |
| 025 | Tag FastArray Container | [ch2-GameState与PlayerState](../detailed/ch2/ch2-GameState与PlayerState.md) |
| 026 | PlayerController Workflow | [ch2-PlayerController与LocalPlayer](../detailed/ch2/ch2-PlayerController与LocalPlayer.md) |
| 027 | LocalPlayer | [ch2-PlayerController与LocalPlayer](../detailed/ch2/ch2-PlayerController与LocalPlayer.md) |

## Source Directories

- `LyraGame/GameModes/` — Experience definition, GameMode, GameState, WorldSettings
- `LyraGame/Player/` — PlayerController, PlayerState, LocalPlayer, spawning management
- `LyraGame/System/` — GameInstance, AssetManager

## Core Architecture

### Experience-Driven Pattern

Lyra's Experience design pattern:
1. `ALyraWorldSettings` specifies the default `ULyraExperienceDefinition`
2. `ALyraGameMode::InitGame()` triggers Experience assignment
3. `ULyraExperienceManagerComponent` (mounted on GameState) manages the loading process
4. When Experience loading completes, notifies GameMode via the `OnExperienceLoaded` event
5. GameMode allows player login after receiving the event (blocks until before `PreLogin`)

### Game Startup Flow

```
UWorld::SetGameMode()
  → ALyraGameMode::InitGame()
    → HandleMatchAssignmentIfNotExpectingOne()  // Next frame
      → Reads default Experience from WorldSettings
      → Calls IsReadyToProcessMatchAssignment()
        → Waits for subsystems like hotfix, GSI, etc.
      → Begins loading Experience
        → ExperienceManagerComponent executes loading
        → Activates GameFeature plugins
        → Grants GA/GE/Attribute, etc.
      → OnExperienceLoaded_Failed/K2
        → GameMode notified, allows player login
  → APlayerController::PreLogin/Login/PostLogin
  → Player spawns
```

## Key Class List

| Class | Inherits From | File | Purpose |
|-------|---------------|------|---------|
| `ULyraExperienceDefinition` | `UPrimaryDataAsset` | [LyraExperienceDefinition.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraExperienceDefinition.h) | Experience data asset definition |
| `ULyraExperienceManagerComponent` | `UGameStateComponent` | [LyraExperienceManagerComponent.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraExperienceManagerComponent.h) | Experience loading and state management |
| `ULyraExperienceActionSet` | `UPrimaryDataAsset` | [LyraExperienceActionSet.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraExperienceActionSet.h) | Experience action set |
| `ULyraGameInstance` | `UCommonGameInstance` | [LyraGameInstance.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraGameInstance.h) | Game instance, manages sessions and login |
| `ALyraGameMode` | `AModularGameModeBase` | [LyraGameMode.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraGameMode.h) | Game mode base class |
| `ALyraGameState` | `AModularGameStateBase` | [LyraGameState.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraGameState.h) | Game state, holds ExperienceManager |
| `ALyraPlayerState` | `AModularPlayerState` | [LyraPlayerState.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Player/LyraPlayerState.h) | Player state, TagStack holder |
| `ALyraPlayerController` | `ACommonPlayerController` | [LyraPlayerController.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Player/LyraPlayerController.h) | Player controller |
| `ULyraLocalPlayer` | `UCommonLocalPlayer` | [LyraLocalPlayer.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Player/LyraLocalPlayer.h) | Local player |
| `ULyraPlayerSpawningManagerComponent` | `UGameStateComponent` | [LyraPlayerSpawningManagerComponent.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Player/LyraPlayerSpawningManagerComponent.h) | Player spawning management |
| `ALyraWorldSettings` | `AWorldSettings` | [LyraWorldSettings.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraWorldSettings.h) | World settings, specifies default Experience |
| `ULyraUserFacingExperienceDefinition` | `UPrimaryDataAsset` | [LyraUserFacingExperienceDefinition.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraUserFacingExperienceDefinition.h) | Front-end user experience definition |
| `ULyraBotCreationComponent` | `UGameStateComponent` | [LyraBotCreationComponent.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraBotCreationComponent.h) | Bot creation component |

## Dependency Relationships

```
ALyraWorldSettings (default Experience)
       ↓
ALyraGameMode → ULyraGameInstance → ULyraAssetManager
       │                                  │
       │                                  └─ Async asset loading
       │
       └─→ ALyraGameState
              ├─ ULyraExperienceManagerComponent (loads Experience)
              ├─ ULyraPlayerSpawningManagerComponent
              └─→ ALyraPlayerState (TagStack holder)
                        │
                        └─→ ALyraPlayerController
                               └─→ ULyraLocalPlayer
```
