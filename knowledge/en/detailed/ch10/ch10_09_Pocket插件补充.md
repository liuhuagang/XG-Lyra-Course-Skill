# Pocket Plugin Supplement

## Overview

PocketWorlds is an independent plugin in the Lyra project (`Plugins/PocketWorlds`), providing two orthogonal capabilities:

1. **Pocket Level Streaming** — Independently streams sub-levels for each local client, used for UI/menu scenes
2. **Pocket Capture** — Scene capture system based on `USceneCaptureComponent2D`, used for generating thumbnails

Each is managed by its own independent `UWorldSubsystem`.

## Pocket Level Streaming

### Pocket Level Definition

[UPocketLevel](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/PocketWorlds/Source/Public/PocketLevel.h) inherits from `UDataAsset`, serving as the definition asset for a pocket world:

- `Level` (`TSoftObjectPtr<UWorld>`): Level asset to stream
- `Bounds` (`FVector`): Boundary size of the pocket world, used to compute vertical offset for multiple instances to avoid overlap

### Pocket Level Subsystem

[UPocketLevelSubsystem](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/PocketWorlds/Source/Public/PocketLevelSystem.h) inherits from `UWorldSubsystem`, the global manager maintaining an array of `UPocketLevelInstance`.

`GetOrCreatePocketLevelFor(LocalPlayer, PocketLevel, DesiredSpawnPoint)` is the core entry point:
- Iterates existing instances, reuses if matching `LocalPlayer + PocketLevel`
- Otherwise, accumulates `Bounds.Z` to compute vertical offset and creates a new instance

### Pocket Level Instance

[UPocketLevelInstance](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/PocketWorlds/Source/Public/PocketLevelInstance.h) inherits from `UObject` (`Within=PocketLevelSubsystem`), responsible for the lifecycle of a single pocket world.

**Core Methods:**

| Method | Description |
|------|------|
| `Initialize` | Calls `ULevelStreamingDynamic::LoadLevelInstanceBySoftObjectPtr`, binds `OnLevelLoaded` / `OnLevelShown` callbacks |
| `StreamIn` / `StreamOut` | Controls streaming level visibility and load state |
| `AddReadyCallback` | Callback when level is ready; executes immediately if already ready |
| `BeginDestroy` | Cleans up streaming level, removes bindings |

**`HandlePocketLevelLoaded()` Key Pattern:**
- `bClientOnlyVisible = true`: Marked as client-only visible
- `bExchangedRoles = true`: Avoids server sync expectations
- `SetOwner(PlayerController)`: All Actors owned by the local player controller

## Pocket Capture System

### Capture Subsystem

[UPocketCaptureSubsystem](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/PocketWorlds/Source/Public/PocketCaptureSubsystem.h) inherits from `UWorldSubsystem`, manages a pool of `UPocketCapture` instances.

**Core Features:**
- `CreateThumbnailRenderer(Class)`: Creates `UPocketCapture`, uses `nullptr` placeholder for slot reuse
- `DestroyThumbnailRenderer(Renderer)`: Nullifies slot and uninitializes
- `StreamThisFrame(Components)`: Marks components `bForceMipStreaming = true`, adds to streaming queue
- Per-frame Tick: Clears previous frame's `bForceMipStreaming`, swaps `StreamNextFrame`

**Two-Frame Mip Streaming Strategy:** Current frame marks components → capture → next frame Tick clears marks, ensuring capture frame gets high-precision Mip textures.

### Pocket Capture Renderer

[UPocketCapture](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/PocketWorlds/Source/Public/PocketCapture.h) inherits from `UObject` (`Abstract, Within=PocketCaptureSubsystem`), responsible for scene captures of target Actors.

**Three Render Targets:**
- `DiffuseRT` (`RTF_RGBA8`): Diffuse color capture
- `AlphaMaskRT` (`RTF_R8`): Alpha mask capture
- `EffectsRT` (`RTF_R8`): Effects capture (TODO not implemented)

**`CaptureScene()` Core Flow:**
1. Gets `UCameraComponent` from `CaptureTarget`
2. Calls `GetThumbnailSystem()->StreamThisFrame()` to trigger Mip streaming
3. If `OverrideMaterial` is provided, temporarily replaces materials on all PrimitiveComponents
4. Disables many ShowFlags (DOF, MotionBlur, TAA, AO, VolumetricFog, etc.) for clean capture
5. Sets `CaptureSource`, calls `CaptureScene()`
6. Restores original materials

## Reference Source

- [PocketWorlds plugin directory](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/PocketWorlds/)
- [UPocketLevel](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/PocketWorlds/Source/Public/PocketLevel.h)
- [UPocketLevelSubsystem](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/PocketWorlds/Source/Public/PocketLevelSystem.h)
- [UPocketLevelInstance](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/PocketWorlds/Source/Public/PocketLevelInstance.h)
- [UPocketCaptureSubsystem](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/PocketWorlds/Source/Public/PocketCaptureSubsystem.h)
- [UPocketCapture](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/PocketWorlds/Source/Public/PocketCapture.h)
