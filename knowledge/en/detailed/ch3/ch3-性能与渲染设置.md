# Performance and Rendering Settings

## Overview

Lyra's performance and rendering system handles frame rate limits, quality levels, device profiles, platform-specific rendering settings, etc. Core classes include ULyraPlatformSpecificRenderingSettings (platform rendering settings), ULyraPerformanceSettings (desktop performance settings), and LyraSettingsHelpers (mobile quality helpers).

## ULyraPlatformSpecificRenderingSettings

Inherits from `UPlatformSettings`, configured per-platform via the `PerPlatformSettings` mechanism in `DefaultEngine.ini`.

### Properties

| Property | Type | Purpose |
|------|------|------|
| DefaultDeviceProfileSuffix | FString | Default device profile suffix |
| UserFacingDeviceProfileOptions | TArray<FLyraQualityDeviceProfileVariant> | User-selectable device profile variants |
| bSupportsGranularVideoQualitySettings | bool | Whether to support granular video quality settings |
| bSupportsAutomaticVideoQualityBenchmark | bool | Whether to support automatic quality benchmark |
| FramePacingMode | ELyraFramePacingMode | Frame pacing mode |
| MobileFrameRateLimits | TArray<int32> | Available mobile frame rate list |

### ELyraFramePacingMode

| Mode | Description | Target Platform |
|------|------|---------|
| DesktopStyle | Manual frame rate limit, user can choose v-sync | Desktop |
| ConsoleStyle | Frame rate controlled via device profile presentation interval | Console |
| MobileStyle | User selects from device profile allowed frame rates | Mobile |

### FLyraQualityDeviceProfileVariant

| Property | Description |
|------|------|
| DisplayName | UI display name |
| DeviceProfileSuffix | Suffix appended to base device profile name |
| MinRefreshRate | Minimum refresh rate required for enabling |

## ULyraPerformanceSettings

Inherits from `UDeveloperSettingsBackedByCVars`, configured in `DefaultGame.ini` under `[LyraPerformanceSettings]`.

### Properties

| Property | Purpose |
|------|------|
| DesktopFrameRateLimits | List of available frame rate limits for desktop |
| UserFacingPerformanceStats | User-enableable performance stat groups |

### FLyraPerformanceStatGroup

| Property | Purpose |
|------|------|
| VisibilityQuery | Visibility query based on platform features (GameplayTag query) |
| AllowedStats | Set of stats available when the query passes |

## LyraSettingsHelpers

Located in the `LyraSettingsLocal.cpp` namespace, provides mobile quality management helper functions.

### TMobileQualityWrapper\<T\>

Template wrapper class for parsing CVar-driven threshold specifications. CVar format is `"FPS:Value,FPS2:Value2,..."`, representing quality limit values applied at different frame rate thresholds.

### Key Functions

| Function | Purpose |
|------|------|
| HasPlatformTrait | Check if the platform has a specified trait |
| GetHighestLevelOfAnyScalabilityChannel | Get the highest level of any scalability channel |
| FillScalabilitySettingsFromDeviceProfile | Fill scalability settings from device profile |
| ConstrainFrameRateToBeCompatibleWithOverallQuality | Constrain frame rate to be compatible with overall quality |

### Mobile CVars

| CVar | Format | Purpose |
|------|------|------|
| Lyra.DeviceProfile.Mobile.OverallQualityLimits | FPS:MaxQuality,FPS2:MaxQuality2 | Maximum quality limit per frame rate |
| Lyra.DeviceProfile.Mobile.ResolutionQualityLimits | FPS:MaxResQuality | Maximum resolution quality per frame rate |
| Lyra.DeviceProfile.Mobile.ResolutionQualityRecommendation | FPS:Recommendation | Recommended resolution per frame rate |
| Lyra.DeviceProfile.Mobile.DefaultFrameRate | int | Default mobile frame rate (30) |
| Lyra.DeviceProfile.Mobile.MaxFrameRate | int | Maximum mobile frame rate (30) |

## Frame Rate Control

### Desktop Frame Rate

Desktop platforms calculate the actual frame rate limit via `GetEffectiveFrameRateLimit()`. This function takes the minimum value among limits based on the current state (foreground/background/menu/battery).

### Console Frame Sync

Console platforms use device profiles to drive target frame rate and frame sync type:

```cpp
CVarDeviceProfileDrivenTargetFps     // Lyra.DeviceProfile.Console.TargetFPS
CVarDeviceProfileDrivenFrameSyncType // Lyra.DeviceProfile.Console.FrameSyncType
```

### Mobile Frame Rate

Mobile frame rate control is implemented via `UpdateMobileFramePacing()`:

1. Read default and maximum frame rates from device profile
2. User selects desired frame rate (`DesiredMobileFrameRateLimit`)
3. Automatically clamp resolution quality based on frame rate (`ClampMobileResolutionQuality`)
4. Set platform frame rate accelerator

```cpp
void ClampMobileResolutionQuality(int32 TargetFPS);  // Clamp resolution quality based on target FPS
void RemapMobileResolutionQuality(int32 FromFPS, int32 ToFPS); // Remap when switching frame rates
```

### Dynamic Resolution

Update dynamic resolution frame time via `UpdateDynamicResFrameTime(float TargetFPS)` based on target frame rate.

## Device Profiles

### Device Profile Suffix

Console and mobile platforms use the `DeviceProfileSuffix` mechanism to switch quality presets:

1. `UserChosenDeviceProfileSuffix` persists user choice
2. `OnHotfixDeviceProfileApplied()` is called during hotfix
3. `UpdateGameModeDeviceProfileAndFps()` applies the device profile for the current mode

### Frontend Performance Settings

`SetShouldUseFrontendPerformanceSettings(bool bInFrontEnd)` controls performance settings under the frontend UI (lower quality to optimize UI responsiveness).
