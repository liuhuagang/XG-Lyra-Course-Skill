# Chapter 1: Project Architecture & Project Setup

## Overview

This chapter sets up the Lyra project from scratch, covering project creation, module splitting, compilation configuration (Target/Build files), asset import, logging system, engine class customization, and other infrastructure. The goal is to establish a compilable, packagable, and extensible Lyra development environment.

## Lecture Scope

| Lecture ID | Title | Corresponding Knowledge File |
|------------|-------|------------------------------|
| 001 | Project Introduction | [ch1-项目概述](../detailed/ch1/ch1-项目概述.md) |
| 002 | Source Code Overview | [ch1-源码架构概览](../detailed/ch1/ch1-源码架构概览.md) |
| 003 | Blueprint Overview | [ch1-源码架构概览](../detailed/ch1/ch1-源码架构概览.md) |
| 004 | Creating the Project | [ch1-工程搭建流程](../detailed/ch1/ch1-工程搭建流程.md) |
| 005 | Target Files | [ch1-Target文件配置](../detailed/ch1/ch1-Target文件配置.md) |
| 006 | Build Files & Module Dependencies | [ch1-源码架构概览](../detailed/ch1/ch1-源码架构概览.md) |
| 007 | Importing Assets | [ch1-资产导入与项目设置](../detailed/ch1/ch1-资产导入与项目设置.md) |
| 008 | Log Categories | [ch1-日志系统](../detailed/ch1/ch1-日志系统.md) |
| 009 | Engine Classes | [ch1-引擎类体系](../detailed/ch1/ch1-引擎类体系.md) |
| 009_补_01 | SPathView's bDisplayPluginFolders | [ch1-编辑器定制](../detailed/ch1/ch1-编辑器定制.md) |

## Source Directories

- `LyraGame/System/` — Engine-level infrastructure (GameEngine, AssetManager)
- `LyraGame/GameModes/` — World settings, Experience management
- `LyraGame/Player/` — Local player
- `LyraGame/Settings/` — Game settings
- `LyraGame/UI/` — Viewport client
- `LyraGame/Audio/` — Audio settings and subsystems
- `LyraEditor/` — Editor module

## Core Concepts

### Module Architecture

Lyra project adopts a two-layer module architecture:
- **LyraGame (Runtime Module)**: Game runtime core, contains all runtime logic
- **LyraEditor (Editor Module)**: Editor-specific functionality, loaded only in the editor

### Compilation Configuration System

UE projects manage compilation through three levels of configuration:
- **`.uproject`** — Project-level configuration, declares module and plugin dependencies
- **`.Target.cs`** — Target platform configuration, declares build target type and edit/run mode
- **`.Build.cs`** — Module-level configuration, declares header file paths and dependency modules

### Engine Class Customization

Lyra replaces 6 engine core classes via `DefaultEngine.ini`, completed early in engine initialization (`FEngineLoop::PreInit`):
- `UGameEngine` → `ULyraGameEngine`
- `UUnrealEdEngine` → `ULyraEditorEngine`
- `UGameViewportClient` → `ULyraGameViewportClient`
- `UAssetManager` → `ULyraAssetManager`
- `AWorldSettings` → `ALyraWorldSettings`
- `ULocalPlayer` → `ULyraLocalPlayer`
- `UGameUserSettings` → `ULyraSettingsLocal`

## Key Class List

| Class | Inherits From | File | Purpose |
|-------|---------------|------|---------|
| `ULyraGameEngine` | `UGameEngine` | [LyraGameEngine.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraGameEngine.h) | Game runtime engine |
| `ULyraEditorEngine` | `UUnrealEdEngine` | [LyraEditorEngine.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/LyraEditorEngine.h) | Editor engine |
| `ULyraGameViewportClient` | `UCommonGameViewportClient` | [LyraGameViewportClient.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/UI/LyraGameViewportClient.h) | Game viewport client |
| `ULyraAssetManager` | `UAssetManager` | [LyraAssetManager.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraAssetManager.h) | Global asset manager |
| `ALyraWorldSettings` | `AWorldSettings` | [LyraWorldSettings.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraWorldSettings.h) | World settings |
| `ULyraLocalPlayer` | `UCommonLocalPlayer` | [LyraLocalPlayer.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Player/LyraLocalPlayer.h) | Local player |
| `ULyraSettingsLocal` | `UGameUserSettings` | [LyraSettingsLocal.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Settings/LyraSettingsLocal.h) | Game user settings |
| `ULyraExperienceManagerComponent` | `UGameStateComponent` | [LyraExperienceManagerComponent.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraExperienceManagerComponent.h) | Experience loading management |
| `ULyraAudioSettings` | `UDeveloperSettings` | [LyraAudioSettings.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Audio/LyraAudioSettings.h) | Audio configuration |
| `ULyraAudioMixEffectsSubsystem` | `UWorldSubsystem` | [LyraAudioMixEffectsSubsystem.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Audio/LyraAudioMixEffectsSubsystem.h) | Audio mix effect overrides |

## Log Categories

Lyra defines dedicated log categories in multiple header files:

| Log Category | Definition Location | Default Level |
|--------------|-------------------|----------------|
| `LogLyra` | LyraLogChannels.h | Log |
| `LogLyraExperience` | LyraLogChannels.h | Log |
| `LogLyraAbilitySystem` | LyraLogChannels.h | Log |
| `LogLyraTeams` | LyraLogChannels.h | Log |
| `LogLyraEditor` | LyraEditor.h | Log |
| `LogLyraGamePhase` | LyraGamePhaseLog.h | Log |
| `LogLyraCheat` | LyraCheatManager.h | Log |
| `LogLyraGameSettingRegistry` | LyraGameSettingRegistry.h | Log |
| `LogLyraRepGraph` | LyraReplicationGraph.h | Display |

## Dependency Relationships

The infrastructure classes covered in this chapter sit at the bottom of the Lyra architecture and are depended upon by all upper-layer subsystems:

```
ULyraGameEngine ← ULyraAssetManager ← ALyraWorldSettings
       ↓
ULyraEditorEngine (Editor only)
       ↓
ULyraExperienceManagerComponent (loads Experience at runtime)
```
