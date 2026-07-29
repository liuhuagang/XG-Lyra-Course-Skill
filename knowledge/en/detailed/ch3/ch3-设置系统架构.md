# Settings System Architecture

## Overview

Lyra's settings system uses a two-tier architecture: local settings (ULyraSettingsLocal) and shared settings (ULyraSettingsShared). Local settings inherit from UGameUserSettings and store machine-bound configuration (audio devices, video resolution, graphics quality, etc.). Shared settings inherit from ULocalPlayerSaveGame and store user-bound configuration (language, color blindness, subtitles, sensitivity, etc.). Both tiers are registered with the settings UI through ULyraGameSettingRegistry.

## UGameUserSettings Base Class

UE5's UGameUserSettings is the engine base class for game user settings, with data stored in `DefaultEngine.ini` (`[Script/Engine.GameUserSettings]` section).

### Initialization Flow

```
Audio device initialization → GEngine->LoadConfig()
  → UEngine::CreateGameUserSettings()
    → Construct UGameUserSettings singleton
    → Read DefaultEngine.ini configuration
```

### Key Methods

| Method | Purpose |
|------|------|
| LoadSettings() | Load saved settings from config file |
| SaveSettings() | Save current settings to config file |
| ApplySettings() | Apply all settings (including non-resolution and resolution) |
| ApplyNonResolutionSettings() | Apply all settings except resolution |
| ValidateSettings() | Validate settings validity |
| GetOverallScalabilityLevel() | Get overall quality level (0~3: Low/Medium/High/Epic) |
| SetOverallScalabilityLevel() | Set overall quality level |

### Scalability System

Scalability::FQualityLevels contains 11 adjustable quality channels:

| Channel | Description |
|------|------|
| ResolutionQuality | Resolution quality (50~100) |
| ViewDistanceQuality | View distance |
| AntiAliasingQuality | Anti-aliasing |
| ShadowQuality | Shadows |
| GlobalIlluminationQuality | Global illumination |
| ReflectionQuality | Reflections |
| PostProcessQuality | Post-processing |
| TextureQuality | Textures |
| EffectsQuality | Effects |
| FoliageQuality | Foliage |
| ShadingQuality | Shading |

The overall quality level (OverallScalabilityLevel) automatically maps to corresponding values for each channel.

## ULyraSettingsLocal

Inherits from `UGameUserSettings`, replaces the engine's default GameUserSettings class via `DefaultEngine.ini`.

```ini
[SystemSettings]
GameUserSettingsClass=/Script/LyraGame.LyraSettingsLocal
```

### Settings Categories

| Category | Description |
|------|------|
| Audio | Volume control (Overall/Music/SoundFX/Dialogue/VoiceChat), audio output device switching, headphone mode (HRTF), HDR Audio |
| Video | Frame rate limits (Battery/Menu/Background/Always), display gamma, resolution mode confirmation |
| Graphics | Overall quality level, device profile suffix, mobile FPS mode, mobile resolution quality clamping |
| Frame Rate | Desktop frame rate limits, console frame sync, mobile frame rate mode |
| Performance Stats | PerfStatDisplayState system |
| Replay | Auto record replays, number of replays to keep |
| Controller | Controller platform, controller preset, input config name |
| Safe Zone | Safe zone scaling |

### Key Methods

| Method | Purpose |
|------|------|
| LoadSettings() | Load settings (Override) |
| SetToDefaults() | Reset to default settings |
| ApplyNonResolutionSettings() | Apply non-resolution settings (volume, quality, frame rate, etc.) |
| GetOverallScalabilityLevel() | Get overall quality level |
| SetOverallScalabilityLevel() | Set overall quality level |
| OnExperienceLoaded() | Callback when Experience loading completes |
| OnHotfixDeviceProfileApplied() | Callback when device profile hotfix is applied |

### Frame Rate Limit Tiers

ULyraSettingsLocal defines 4 frame rate limit modes:

| Property | Description |
|------|------|
| FrameRateLimit_OnBattery | Frame rate limit when on battery |
| FrameRateLimit_InMenu | Frame rate limit in menus |
| FrameRateLimit_WhenBackgrounded | Frame rate limit when running in background |
| FrameRateLimit_Always | Always active frame rate limit |

The actual frame rate limit is calculated by `GetEffectiveFrameRateLimit()` taking the minimum value based on the current state.

### Audio Volume Control

5 volume channels implemented via ControlBus system:

| Volume Channel | ControlBus Binding | Default Value |
|----------|----------------|--------|
| OverallVolume | OverallControlBus | 1.0 |
| MusicVolume | MusicControlBus | 1.0 |
| SoundFXVolume | SoundFXControlBus | 1.0 |
| DialogueVolume | DialogueControlBus | 1.0 |
| VoiceChatVolume | VoiceChatControlBus | 1.0 |

Volume values are applied by loading a ControlBusMix via LoadUserControlBusMix and calling SetMixValue on USoundControlBusMix.

### Performance Stats Display

The PerfStatDisplayState system controls HUD performance widget display modes:

```cpp
ELyraStatDisplayMode // Hidden/TextOnly/GraphOnly/TextAndGraph
TMap<ELyraDisplayablePerformanceStat, ELyraStatDisplayMode> DisplayStatList
```

Settings are modified via SetPerfStatDisplayState(), triggering PerfStatSettingsChanged events on change.

## ULyraSettingsShared

Inherits from `ULocalPlayerSaveGame`, stored in the save system, supports cloud sync.

### Settings Categories

| Category | Properties |
|------|------|
| Color Blind Mode | ColorBlindMode (Off/Deuteranope/Protanope/Tritanope), ColorBlindStrength (0~10) |
| Subtitles | Enable/disable, text size/color/border, background opacity |
| Audio | AllowAudioInBackground (Off/AllSounds) |
| Language | PendingCulture, ResetToDefaultCulture |
| Mouse Sensitivity | MouseSensitivityX/Y, TargetingMultiplier, InvertVerticalAxis, InvertHorizontalAxis |
| Gamepad Sensitivity | GamepadLookSensitivityPreset, GamepadTargetingSensitivityPreset (Slow~Insane 10 levels) |
| Gamepad Dead Zones | GamepadMoveStickDeadZone, GamepadLookStickDeadZone |
| Gamepad Vibration | bForceFeedbackEnabled |
| Trigger Vibration | bTriggerHapticsEnabled, TriggerHapticStrength, TriggerHapticStartPosition |

### Lifecycle

```cpp
// Create temporary settings object
static ULyraSettingsShared* CreateTemporarySettings();

// Synchronous load (cannot be called before login)
static ULyraSettingsShared* LoadOrCreateSettings();

// Async load
static bool AsyncLoadOrCreateSettings(..., FOnSettingsLoadedEvent Delegate);

// Save
void SaveSettings(); // Async save

// Apply
void ApplySettings(); // Subtitles + background audio + culture + input
```

### Dirty Flag Mechanism

The ChangeValueAndDirty template method implements unified "value and dirty flag" management: when a property value changes, it updates the value, marks the dirty flag, and broadcasts the OnSettingChanged event.

## ULyraGameSettingRegistry

Settings registry, inheriting from `UGameSettingRegistry`, registers properties from ULyraSettingsLocal and ULyraSettingsShared as a UGameSetting collection.

### Registry Categories

| Category | Initialization Method | Data Source |
|------|-----------|--------|
| Video | InitializeVideoSettings() | ULyraSettingsLocal |
| Audio | InitializeAudioSettings() | ULyraSettingsLocal |
| Gameplay | InitializeGameplaySettings() | ULyraSettingsLocal + ULyraSettingsShared |
| MouseAndKeyboard | InitializeMouseAndKeyboardSettings() | ULyraSettingsShared |
| Gamepad | InitializeGamepadSettings() | ULyraSettingsShared |

### SaveChanges Flow

On save, both tiers are applied and saved sequentially:

```cpp
void ULyraGameSettingRegistry::SaveChanges() {
    LocalPlayer->GetLocalSettings()->ApplySettings(false);  // Apply local settings
    LocalPlayer->GetSharedSettings()->ApplySettings();       // Apply shared settings
    LocalPlayer->GetSharedSettings()->SaveSettings();        // Save shared settings
}
```

## Custom Setting Value Types

Located in `Settings/CustomSettings/` directory, inheriting from UGameSettingValue*:

| Class | Purpose |
|------|------|
| ULyraSettingValueDiscrete_Resolution | Resolution selector |
| ULyraSettingValueDiscrete_OverallQuality | Overall quality selector |
| ULyraSettingValueDiscrete_MobileFPSType | Mobile FPS selector |
| ULyraSettingValueDiscrete_Language | Language selector |
| ULyraSettingValueDiscrete_PerfStat | Performance stat selector |
| ULyraSettingValueDiscreteDynamic_AudioOutputDevice | Audio output device selector |
| ULyraSettingKeyboardInput | Keyboard mapping settings |
| ULyraSettingAction_SafeZoneEditor | Safe zone editor action |

## Settings Editor Screens

Located in `Settings/Screens/` directory:

| Class | Purpose |
|------|------|
| ULyraSafeZoneEditor | Safe zone visual editor |
| ULyraBrightnessEditor | Brightness/gamma visual editor |

## Settings List Widgets

Located in `Settings/Widgets/` directory:

| Class | Purpose |
|------|------|
| ULyraSettingsListEntrySetting_KeyboardInput | Keyboard mapping list entry widget |
