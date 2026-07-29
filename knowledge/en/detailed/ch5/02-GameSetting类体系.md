# GameSetting Class Hierarchy

## Overview

The GameSetting class hierarchy defines the core data structure of the settings system. All settings (volume sliders, dropdowns, button actions, etc.) inherit from the `UGameSetting` base class, using a polymorphic hierarchy to differentiate different types of behavior.

## Complete Class Hierarchy

```
UGameSetting (abstract base class)
  ├── UGameSettingCollection (setting collection container)
  │     └── UGameSettingCollectionPage (navigable sub-page)
  ├── UGameSettingValue (modifiable value)
  │     ├── UGameSettingValueScalar (continuous value slider)
  │     │     └── UGameSettingValueScalarDynamic (dynamic data source slider)
  │     └── UGameSettingValueDiscrete (discrete value selector)
  │           ├── UGameSettingValueDiscreteDynamic (dynamic discrete value)
  │           │     ├── UGameSettingValueDiscreteDynamic_Bool
  │           │     ├── UGameSettingValueDiscreteDynamic_Number
  │           │     ├── UGameSettingValueDiscreteDynamic_Enum
  │           │     ├── UGameSettingValueDiscreteDynamic_Color
  │           │     └── UGameSettingValueDiscreteDynamic_Vector2D
  │           └── Lyra custom discrete values:
  │                 ├── ULyraSettingValueDiscrete_Resolution
  │                 ├── ULyraSettingValueDiscrete_Language
  │                 ├── ULyraSettingValueDiscrete_PerfStat
  │                 ├── ULyraSettingValueDiscrete_MobileFPSType
  │                 ├── ULyraSettingValueDiscrete_OverallQuality
  │                 └── ULyraSettingValueDiscreteDynamic_AudioOutputDevice
  └── UGameSettingAction (executable action button)
        ├── ULyraSettingAction_SafeZoneEditor (safe zone editor)
        └── ULyraSettingKeyboardInput (key binding)
```

## UGameSetting Base Class

- **Header**: [GameSetting.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameSettings/Source/Public/GameSetting.h)
- **Inheritance**: `UObject` → `UGameSetting`

Core properties:

| Property | Type | Description |
|------|------|------|
| `DevName` | `FName` | Developer identifier name, used for lookup |
| `DisplayName` | `FText` | UI display name |
| `DescriptionRichText` | `FText` | Rich text description |
| `WarningRichText` | `FText` | Warning text |
| `Tags` | `FGameplayTagContainer` | Tags |
| `DynamicDetails` | Script delegate | Dynamic detail text |
| `SettingParent` | `UGameSetting*` | Parent setting |
| `OwningRegistry` | `UGameSettingRegistry*` | Owning registry |
| `bReady` | `bool` | Whether initialization is complete |
| `EditableStateCache` | `FGameSettingEditableState` | Editable state cache |
| `EditConditions` | `TArray<FGameSettingEditCondition*>` | List of edit conditions |

Event delegates:

| Event | Parameters | Description |
|------|------|------|
| `OnSettingChanged` | `(Setting, EGameSettingChangeReason)` | Triggered when value changes |
| `OnSettingApplied` | `(Setting)` | Triggered when value is applied |
| `OnSettingEditConditionChanged` | `(Setting)` | Triggered when edit condition changes |
| `OnNamedAction` | `(Setting, FGameplayTag)` | Named action triggered |

### Lifecycle

```
Initialize(LocalPlayer)
  ├── Check DevName is not empty (ensure in !UE_BUILD_SHIPPING)
  ├── Initialize child Settings (recursive for collections)
  └── Startup()
        ├── StartupComplete()
        └── OnInitialized()
              └── ComputeEditableState()
```

Override for `UGameSettingValueScalarDynamic`:

```
Startup()
  └── Getter->Startup(LocalPlayer, OnDataSourcesReady)
        └── OnDataSourcesReady()
              └─ Setter->Startup(LocalPlayer, nullptr)
              └─ StartupComplete()
```

### Editable State

`ComputeEditableState()` calculates the current setting's editable state:

```
ComputeEditableState()
  └── Create new FGameSettingEditableState
        ├── Iterate all EditConditions → GatherEditState(EditableState)
        └── Cache to EditableStateCache
```

## UGameSettingCollection

- **Header**: [GameSettingCollection.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameSettings/Source/Public/GameSettingCollection.h)
- **Responsibility**: Container for child settings, implements setting hierarchy

| Method | Description |
|------|------|
| `AddSetting(Setting)` | Add a child setting |
| `GetSettingsForFilter(FilterState)` | Get visible child settings by filter criteria |

### UGameSettingCollectionPage

`IsSelectable=true`, can independently navigate to a child setting page. Defines navigation text via `NavigationText`, and handles navigation via `OnExecuteNavigationEvent`.

## UGameSettingValue

- **Header**: [GameSettingValue.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameSettings/Source/Public/GameSettingValue.h)
- **Pure virtual methods**:

| Method | Description |
|------|------|
| `StoreInitial()` | Store initial value after Startup |
| `ResetToDefault()` | Reset to default value |
| `RestoreToInitial()` | Restore to initial value |

### UGameSettingValueScalar

- **Header**: [GameSettingValueScalar.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameSettings/Source/Public/GameSettingValueScalar.h)
- Range slider value, uses `SetValueNormalized(n)` / `GetValueNormalized()` for normalized value access
- `SourceRange` / `SourceStep` define the original value range and step size

### UGameSettingValueScalarDynamic

- **Header**: [GameSettingValueScalarDynamic.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameSettings/Source/Public/GameSettingValueScalarDynamic.h)
- Reads/writes values via dynamic data sources (Getter/Setter `FGameSettingDataSource`)
- Custom `DisplayFormat` lambda controls display formatting
- `SetDefaultValue(Value)` sets the default value
- Async `Startup`: Getter → OnDataSourcesReady → Setter → StartupComplete

### UGameSettingValueDiscrete

- **Header**: [GameSettingValueDiscrete.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameSettings/Source/Public/GameSettingValueDiscrete.h)
- Discrete value selector (dropdown/radio button)

| Method | Description |
|------|------|
| `SetDiscreteOptionByIndex(Index)` | Set option by index |
| `GetDiscreteOptions()` | Get all option texts |
| `GetDiscreteOptionDefaultIndex()` | Get default option index |

## UGameSettingAction

- **Header**: [GameSettingAction.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameSettings/Source/Public/GameSettingAction.h)
- Executable button action

| Property | Description |
|------|------|
| `ActionText` | Button text |
| `NamedAction` / `CustomAction` | Action callbacks (named actions triggered via FGameplayTag) |

## Filter State

### FGameSettingFilterState

- **Header**: [GameSettingFilterState.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameSettings/Source/Public/GameSettingFilterState.h)

| Property | Default | Description |
|------|--------|------|
| `bIncludeDisabled` | `true` | Whether to include disabled settings |
| `bIncludeHidden` | `false` | Whether to include hidden settings |
| `bIncludeResetable` | `true` | Whether to include resettable settings |
| `bIncludeNestedPages` | `false` | Whether to include nested pages |
| `SearchText` | Empty | Search text |
| `SettingRootList` | Empty | Setting root list whitelist |
| `SettingAllowList` | Empty | Setting allow list |

`DoesSettingPassFilter(Setting)` checks sequentially: visibility → enabled → resettable → allow list → search text match.

## Editable State

### FGameSettingEditableState

- **Header**: [GameSettingFilterState.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameSettings/Source/Public/GameSettingFilterState.h)

| Property | Default | Description |
|------|--------|------|
| `bVisible` | `true` | Whether visible |
| `bEnabled` | `true` | Whether interactive |
| `bResetable` | `true` | Whether resettable |
| `bHideFromAnalytics` | `false` | Whether to hide from analytics |

Methods:

| Method | Description |
|------|------|
| `Hide(DevReason)` | Hide (with developer reason) |
| `Disable(Reason)` | Disable |
| `DisableOption(Option)` | Disable specific option |
| `UnableToReset()` | Cannot reset |
| `HideFromAnalytics()` | Hide from analytics |
| `Kill(DevReason)` | Hide + HideFromAnalytics + UnableToReset simultaneously |

## Edit Conditions

### FGameSettingEditCondition (Base)

- **Header**: [GameSettingFilterState.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameSettings/Source/Public/GameSettingFilterState.h)

| Virtual Method | Description |
|--------|------|
| `Initialize(Setting)` | Initialize, can bind events |
| `GatherEditState(EditableState)` | Collect editable state |
| `SettingApplied(Setting)` | Callback when setting is applied |
| `SettingChanged(Value, ChangeReason)` | Callback when setting changes |

### Built-in Edit Conditions

| Class | Description |
|----|------|
| `FWhenCondition` | Condition check via lambda |
| `FWhenPlatformHasTrait` | Control by platform trait GameplayTag |
| `FWhenPlayingAsPrimaryPlayer` | Only primary player (singleton) |

#### FWhenPlatformHasTrait

- **Header**: [WhenPlatformHasTrait.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameSettings/Source/Public/EditCondition/WhenPlatformHasTrait.h)

Factory methods (all static):

| Method | Behavior |
|------|------|
| `KillIfMissing(Tag)` | Kill if tag is missing |
| `DisableIfMissing(Tag)` | Disable if tag is missing |
| `KillIfPresent(Tag)` | Kill if tag is present |
| `DisableIfPresent(Tag)` | Disable if tag is present |

### Edit Dependencies

Cross-setting linkage via `AddEditDependency(Setting)`. When a dependent setting's EditableState changes, the current setting's EditableState is automatically recalculated.

## Lyra Custom Types

### ULyraSettingValueDiscrete_Resolution

- **Header**: [LyraSettingValueDiscrete_Resolution.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Settings/CustomSettings/LyraSettingValueDiscrete_Resolution.h)
- Gets the available resolution list from `ULyraSettingsLocal`
- Depends on WindowMode setting (resolution can only be switched in fullscreen mode)

### ULyraSettingValueDiscrete_Language

- **Header**: [LyraSettingValueDiscrete_Language.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Settings/CustomSettings/LyraSettingValueDiscrete_Language.h)
- Gets available languages via `FTextLocalizationManager`
- Uses `ULyraSettingsShared::SetPendingCulture` to stage language changes

### ULyraSettingValueDiscrete_PerfStat

- **Header**: [LyraSettingValueDiscrete_PerfStat.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Settings/CustomSettings/LyraSettingValueDiscrete_PerfStat.h)
- 4 display modes: Hidden / TextOnly / GraphOnly / TextAndGraph

### ULyraSettingAction_SafeZoneEditor

- **Header**: [LyraSettingAction_SafeZoneEditor.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Settings/CustomSettings/LyraSettingAction_SafeZoneEditor.h)
- Navigates to the safe zone editor interface
- Embeds `ULyraSettingValueScalarDynamic_SafeZoneValue` child setting

### ULyraSettingKeyboardInput

- **Header**: [LyraSettingKeyboardInput.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Settings/CustomSettings/LyraSettingKeyboardInput.h)
- Key binding setting, integrates with Enhanced Input's `UEnhancedPlayerMappableKeyProfile`
