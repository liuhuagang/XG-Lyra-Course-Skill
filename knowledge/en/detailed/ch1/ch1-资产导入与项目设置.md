# Asset Import and Project Settings

## Packaging Verification

Errors that may occur during the first packaging after creating a blank level and their fixes:

### Error 1: GameFeatureData Asset Type Not Registered

```
Asset manager settings do not include a rule for assets of type GameFeatureData
```

**Fix**: Add to `PrimaryAssetTypesToScan` in Game → AssetManager:

```ini
(PrimaryAssetType="GameFeatureData",
 AssetBaseClass="/Script/GameFeatures.GameFeatureData",
 bHasBlueprintClasses=False,
 bIsEditorOnly=False,
 Directories=((Path="/Game/Unused")),
 Rules=(Priority=-1, ChunkId=-1, bApplyRecursively=True, CookRule=AlwaysCook))
```

### Error 2: Water Body Collision Preset Missing

```
Collision Profile settings do not include an entry for the Water Body Collision profile
```

**Fix**: Add the `WaterBodyCollision` collision preset in the `[/Script/Engine.CollisionProfile]` section of `DefaultEngine.ini`.

## Collision Configuration

Lyra defines 5 custom trace channels:

| Channel Name | Purpose |
|--------|------|
| `Lyra_TraceChannel_Interaction` | Interaction system raycast |
| `Lyra_TraceChannel_Weapon` | Weapon raycast |
| `Lyra_TraceChannel_Weapon_Capsule` | Weapon capsule detection |
| `Lyra_TraceChannel_Weapon_Multi` | Multi-segment weapon detection |
| `Lyra_TraceChannel_AimAssist` | Aim assist detection |

And multiple custom collision profiles:

| Profile Name | Purpose |
|--------|------|
| `LyraPawnMesh` | Character mesh collision |
| `LyraPawnCapsule` | Character capsule collision |
| `Interactable_OverlapDynamic` | Interactable object overlap detection |
| `Interactable_BlockDynamic` | Interactable object physics blocking |
| `AimAssist_OverlapDynamic` | Aim assist area detection |

## Asset Import

Copy the Content directory from the Lyra sample project directly into the current project.

Post-import status:
- All Blueprints referencing C++ classes will be broken (C++ classes not yet written)
- Some assets can still be used:
  - Audio assets (MetaSound, SoundWave)
  - Some animation assets (AnimSequence)
  - All materials, textures, and meshes

## Audio Settings

Project audio is primarily managed through two classes:

| Class | Inherits From | Purpose |
|----|--------|------|
| `ULyraAudioSettings` | `UDeveloperSettings` | Audio configuration, defines default sound classes, concurrency settings, submix channels |
| `ULyraAudioMixEffectsSubsystem` | `UWorldSubsystem` | Runtime audio mix overrides |

Key audio settings properties (from `AudioSettings.h`):
- `DefaultSoundClass` — Sound class for newly created sounds
- `DefaultSoundConcurrencyName` — Default sound concurrency settings
- `DefaultBaseSoundMix` — Base sound mix
- `MasterSubmix` — Default submix channel (root submix for audio output to hardware)
- `ReverbSubmix` — Submix channel for reverb effects
