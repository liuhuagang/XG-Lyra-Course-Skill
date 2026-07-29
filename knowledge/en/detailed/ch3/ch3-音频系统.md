# Audio System

## Overview

Lyra's audio system uses ULyraAudioSettings to configure ControlBus references, and ULyraAudioMixEffectsSubsystem to load ControlBuses at runtime and apply user volume settings. The audio system supports HDR/LDR dynamic range switching, headphone mode (HRTF), and multi-channel independent volume control.

## ULyraAudioSettings

Inherits from `UDeveloperSettings`, path references configured in `DefaultGame.ini` under `[LyraAudioSettings]`.

### ControlBus Path Configuration

| Property | Type | Purpose |
|------|------|------|
| DefaultControlBusMix | FSoftObjectPath | Default base ControlBusMix |
| LoadingScreenControlBusMix | FSoftObjectPath | Loading screen ControlBusMix |
| UserSettingsControlBusMix | FSoftObjectPath | User settings ControlBusMix |
| OverallVolumeControlBus | FSoftObjectPath | Overall volume ControlBus |
| MusicVolumeControlBus | FSoftObjectPath | Music volume ControlBus |
| SoundFXVolumeControlBus | FSoftObjectPath | Sound effects volume ControlBus |
| DialogueVolumeControlBus | FSoftObjectPath | Dialogue volume ControlBus |
| VoiceChatVolumeControlBus | FSoftObjectPath | Voice chat volume ControlBus |

### Dynamic Range Effect Chains

| Property | Purpose |
|------|------|
| HDRAudioSubmixEffectChain | Submix effect chain for HDR audio mode |
| LDRAudioSubmixEffectChain | Submix effect chain for LDR audio mode |

## ULyraAudioMixEffectsSubsystem

Inherits from `UWorldSubsystem`, automatically loads ControlBuses at level start and applies user volume settings.

### Runtime References

| Property | Type | Source |
|------|------|------|
| DefaultBaseMix | USoundControlBusMix | Loaded from ULyraAudioSettings |
| LoadingScreenMix | USoundControlBusMix | Loaded from ULyraAudioSettings |
| UserMix | USoundControlBusMix | Loaded from ULyraAudioSettings |
| OverallControlBus | USoundControlBus | Loaded from ULyraAudioSettings |
| MusicControlBus | USoundControlBus | Loaded from ULyraAudioSettings |
| SoundFXControlBus | USoundControlBus | Loaded from ULyraAudioSettings |
| DialogueControlBus | USoundControlBus | Loaded from ULyraAudioSettings |
| VoiceChatControlBus | USoundControlBus | Loaded from ULyraAudioSettings |

### Lifecycle

```
Initialize() → Load ControlBus paths from ULyraAudioSettings
  → PostInitialize() → Complete initialization
  → OnWorldBeginPlay() → Apply default BaseMix
  → ApplyDynamicRangeEffectsChains(bHDRAudio) → Switch HDR/LDR effect chains
  → Loading screen: ApplyOrRemoveLoadingScreenMix()
  → Deinitialize() → Cleanup
```

### ApplyDynamicRangeEffectsChains

Applies HDRSubmixEffectChain or LDRSubmixEffectChain to the corresponding Submix based on whether HDR Audio is enabled. When the user toggles the HDR Audio setting, ULyraSettingsLocal calls this method to update the effect chain.

### Loading Screen Audio

The `OnLoadingScreenStatusChanged` callback listens for loading screen state changes, applying `LoadingScreenMix` when the loading screen opens and removing it when closed.

## ControlBus Audio Modulation

The UE5 Audio Modulation system uses ControlBus for parameterized audio control.

### Core Concepts

| Concept | Description |
|------|------|
| ControlBus | Parameterized control bus, defines value range and unit |
| ControlBusMix | Collection of multiple ControlBus values, activated/deactivated as a whole |

### Audio Modulation Chain

```
MetaSound/SoundClass → ControlBus binding
  → ControlBusMix activation
    → Modulation parameters (volume/pitch/filter, etc.)
      → Final audio output
```

### Mixing Rules

| Rule | Description |
|------|------|
| Union Mode | When multiple Mixes are active simultaneously, takes the maximum value |
| Override Mode | Directly override to specified value |

## HRTF Headphone Mode

HRTF (Head-Related Transfer Function) is used for audio spatialization on headphones.

### Implementation Principles

| Mechanism | Description |
|------|------|
| Spectral Filtering | Filters the ear spectrum based on sound source direction |
| ITD (Interaural Time Difference) | Time difference between ears, determines horizontal position |
| IID (Interaural Intensity Difference) | Intensity difference between ears, determines vertical position |

### Implementation in Lyra

```cpp
// ULyraSettingsLocal
bool IsHeadphoneModeEnabled() const;   // Whether headphone mode is enabled
void SetHeadphoneModeEnabled(bool bEnabled);  // Enable/disable
bool CanModifyHeadphoneModeEnabled() const;   // Whether modifiable (platform forced)
```

- `bUseHeadphoneMode` (Config flag) persisted in storage
- `bDesiredHeadphoneMode` (Transient flag) runtime desired value
- If the platform forces a setting via CVar `au.DisableBinauralSpatialization`, `CanModifyHeadphoneModeEnabled()` returns false

## Audio Device Switching

```cpp
// ULyraSettingsLocal
FString GetAudioOutputDeviceId() const;
void SetAudioOutputDeviceId(const FString& InAudioOutputDeviceId);
```

Switching triggers the `OnAudioOutputDeviceChanged` event, and the audio subsystem responds by switching the output device.
