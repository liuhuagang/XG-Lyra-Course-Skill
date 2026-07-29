# Chapter 3: Settings System

## Overview

This chapter covers the core of Lyra's settings system. The settings system adopts a two-layer architecture: local settings (ULyraSettingsLocal) inherit from UGameUserSettings, stored in local configuration files, managing audio, video, performance, frame rate, and other machine-bound settings; shared settings (ULyraSettingsShared) inherit from ULocalPlayerSaveGame, stored in the save game system, managing language, color blindness, subtitles, sensitivity, and other user-bound settings. Both layers are registered with the settings UI through ULyraGameSettingRegistry.

## Lecture Scope

| Lecture ID | Title | Corresponding Knowledge File |
|------------|-------|------------------------------|
| 028 | GameUserSettings | [ch3-设置系统架构](../detailed/ch3/ch3-设置系统架构.md) |
| 029 | Lyra Local Game Settings | [ch3-设置系统架构](../detailed/ch3/ch3-设置系统架构.md) |
| 029_泽_01 | ControlBus | [ch3-音频系统](../detailed/ch3/ch3-音频系统.md) |
| 029_泽_02 | HRTF | [ch3-音频系统](../detailed/ch3/ch3-音频系统.md) |
| 029_泽_03 | ILatencyMarkerModule | [ch3-Latency与NVIDIA Reflex](../detailed/ch3/ch3-Latency与NVIDIAReflex.md) |
| 029_泽_04 | SetCustomLatencyMarker | [ch3-Latency与NVIDIA Reflex](../detailed/ch3/ch3-Latency与NVIDIAReflex.md) |
| 029_泽_05 | LyraSettingsHelpers | [ch3-性能与渲染设置](../detailed/ch3/ch3-性能与渲染设置.md) |
| 030 | Audio Mix Subsystem | [ch3-音频系统](../detailed/ch3/ch3-音频系统.md) |
| 031 | Platform Rendering Settings | [ch3-性能与渲染设置](../detailed/ch3/ch3-性能与渲染设置.md) |
| 032 | SaveGame and LocalPlayerSaveGame | [ch3-设置系统架构](../detailed/ch3/ch3-设置系统架构.md) |
| 033 | Lyra Shared Game Settings | [ch3-设置系统架构](../detailed/ch3/ch3-设置系统架构.md) |
| 034 | GameFeature | [ch3-GameFeature系统](../detailed/ch3/ch3-GameFeature系统.md) |
| 034_AI_ActorFactory | AI ActorFactory | [ch3-GameFeature系统](../detailed/ch3/ch3-GameFeature系统.md) |
| 034_AI_UContentBundleEngineSubsystem | ContentBundle Engine Subsystem | [ch3-GameFeature系统](../detailed/ch3/ch3-GameFeature系统.md) |
| 035 | LyraGameFeature | [ch3-GameFeature系统](../detailed/ch3/ch3-GameFeature系统.md) |
| 036 | LyraHotFix | [ch3-Hotfix系统](../detailed/ch3/ch3-Hotfix系统.md) |
| 037 | LyraGameplayCueManager | [ch3-GameplayCue管理](../detailed/ch3/ch3-GameplayCue管理.md) |
| 038 | Preloading Screen (CommonStartupLoadingScreen) | [docs/..._038_预加载界面.txt](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/docs/UE5_Lyra学习指南_038_预加载界面.txt) — Extracted from .docx lecture, includes editor screenshots, text is for structural reference only |

## Source Directories

- `Settings/` — Local settings (ULyraSettingsLocal), shared settings (ULyraSettingsShared), settings registry (ULyraGameSettingRegistry)
- `Settings/CustomSettings/` — Custom setting value types (resolution, quality, mobile FPS, language, audio output device, safe zone, key mapping)
- `Settings/Screens/` — Settings editor interfaces (safe zone editor, brightness editor)
- `Settings/Widgets/` — Settings list entry widgets
- `Audio/` — Audio settings (ULyraAudioSettings), audio mix subsystem (ULyraAudioMixEffectsSubsystem)
- `Hotfix/` — Hotfix manager (ULyraHotfixManager), text hotfix config (LyraTextHotfixConfig)
- `GameFeatures/` — GameFeature policy (ULyraGameFeaturePolicy), various GameFeatureActions
- `Performance/` — Platform rendering settings (ULyraPlatformSpecificRenderingSettings), performance settings
- `AbilitySystem/` — GameplayCue manager (ULyraGameplayCueManager)

## Core Architecture

### Two-Layer Settings Architecture

Lyra splits game settings into two tiers:

1. **Local Settings (ULyraSettingsLocal)**: Inherits from UGameUserSettings, data stored in local config files (Config tag). Machine-bound, includes audio device, video resolution, graphics quality, frame rate limit, controller platform, etc. Replaces the engine default UGameUserSettings via `DefaultEngine.ini`.

2. **Shared Settings (ULyraSettingsShared)**: Inherits from ULocalPlayerSaveGame, data stored in the save game system. User-bound, cross-platform synced, includes color blind mode, subtitle options, language/culture, controller sensitivity, background audio strategy, etc.

### Settings Registry Pattern

ULyraGameSettingRegistry acts as the settings registration center, registering properties from local settings and shared settings as UGameSetting objects, grouped by category (Video, Audio, Gameplay, Mouse & Keyboard, Controller), and provided to the settings UI.

### Settings Lifecycle

```
Initialization phase:
DefaultEngine.ini → UGameUserSettings replaced with ULyraSettingsLocal
  → LoadSettings() reads local config
  → ULyraGameSettingRegistry registers all settings

Runtime:
Settings UI modified → Write property → MarkDirty
  → SaveChanges() → ApplySettings()
    → ApplyNonResolutionSettings() applies non-resolution settings
    → ApplyResolutionSettings() applies resolution settings
  → SaveSettings() persists

Shared settings:
AsyncLoadOrCreateSettings() async load
  → ApplySettings() applies (subtitles, background audio, culture, input)
  → SaveSettings() async saves to save file
```

## Key Class List

| Class | Inherits From | File | Purpose |
|-------|---------------|------|---------|
| `ULyraSettingsLocal` | `UGameUserSettings` | [LyraSettingsLocal.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Settings/LyraSettingsLocal.h) | Local game settings (machine-bound) |
| `ULyraSettingsShared` | `ULocalPlayerSaveGame` | [LyraSettingsShared.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Settings/LyraSettingsShared.h) | Shared game settings (user-bound) |
| `ULyraGameSettingRegistry` | `UGameSettingRegistry` | [LyraGameSettingRegistry.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Settings/LyraGameSettingRegistry.h) | Settings registry |
| `ULyraAudioMixEffectsSubsystem` | `UWorldSubsystem` | [LyraAudioMixEffectsSubsystem.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Audio/LyraAudioMixEffectsSubsystem.h) | Audio mix effects subsystem |
| `ULyraAudioSettings` | `UDeveloperSettings` | [LyraAudioSettings.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Audio/LyraAudioSettings.h) | Audio configuration (ControlBus references) |
| `ULyraPlatformSpecificRenderingSettings` | `UPlatformSettings` | [LyraPerformanceSettings.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Performance/LyraPerformanceSettings.h) | Platform rendering settings |
| `ULyraPerformanceSettings` | `UDeveloperSettingsBackedByCVars` | [LyraPerformanceSettings.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Performance/LyraPerformanceSettings.h) | Performance settings (desktop frame rate limits, etc.) |
| `ULyraHotfixManager` | `UOnlineHotfixManager` | [LyraHotfixManager.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Hotfix/LyraHotfixManager.h) | Hotfix manager |
| `ULyraGameFeaturePolicy` | `UDefaultGameFeaturesProjectPolicies` | [LyraGameFeaturePolicy.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameFeatures/LyraGameFeaturePolicy.h) | GameFeature plugin loading policy |
| `ULyraGameplayCueManager` | `UGameplayCueManager` | [LyraGameplayCueManager.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/LyraGameplayCueManager.h) | GameplayCue manager |

## Dependency Relationships

```
ULyraSettingsLocal (machine-bound, Config serialized)
       │
       ├──→ ULyraAudioSettings (ControlBus path config)
       │         └──→ ULyraAudioMixEffectsSubsystem (loads ControlBus at runtime)
       │
       ├──→ ULyraPlatformSpecificRenderingSettings (platform frame rate mode)
       │
       └──→ ULyraGameSettingRegistry (registers all settings to UI)
                  │
                  └──→ ULyraSettingsShared (user-bound, save file serialized)

ULyraHotfixManager (runtime config hotfix)
       │
       └──→ Modifies ULyraSettingsLocal, etc.

ULyraGameFeaturePolicy (GameFeature plugin lifecycle management)
       │
       ├──→ UGameFeatureAction_* (specific actions)
       └──→ ULyraGameplayCueManager (GameplayCue path registration)
```
