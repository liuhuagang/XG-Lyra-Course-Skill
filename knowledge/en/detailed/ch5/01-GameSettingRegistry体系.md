# GameSettingRegistry System

## Overview

`ULyraGameSettingRegistry` is the central registry of the settings system. It inherits from `UGameSettingRegistry` (GameSettings plugin) and is responsible for creating and managing all game settings. The registry is created and held by `ULyraSettingScreen`, serving as the bridge between the settings UI and the underlying data.

## Core Classes

### UGameSettingRegistry (Base)

- **Header**: [GameSettingRegistry.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameSettings/Source/Public/GameSettingRegistry.h)
- **Responsibility**: Settings registry base class, manages setting registration, filtering, and event dispatching

Core members:

| Member | Type | Description |
|------|------|------|
| `TopLevelSettings` | `TArray<UGameSetting*>` | Top-level setting collection, added via `RegisterSetting` |
| `RegisteredSettings` | `TArray<UGameSetting*>` | Flat list of all registered settings |
| `OwningLocalPlayer` | `ULocalPlayer*` | Owning local player |
| `OnSettingChangedEvent` | Multicast delegate | Triggered when setting value changes, params: (Setting, ChangeReason) |
| `OnSettingEditConditionChangedEvent` | Multicast delegate | Triggered when edit condition changes |
| `OnSettingNamedActionEvent` | Multicast delegate | Called when a named action is triggered |
| `OnExecuteNavigationEvent` | Multicast delegate | Triggered when navigating to a sub-setting page |

Lifecycle methods:

| Method | Description |
|------|------|
| `Initialize(LocalPlayer)` | Entry point: sets OwningLocalPlayer → calls `OnInitialize` |
| `OnInitialize(LocalPlayer)` | Pure virtual function, subclass implements specific setting registration |
| `Regenerate()` | Regenerates all settings |
| `SaveChanges()` | Saves all setting changes |
| `GetSettingsForFilter(FilterState)` | Gets the setting list filtered by criteria |
| `FindSettingByDevName(DevName)` | Finds a setting by developer name |

### ULyraGameSettingRegistry (Concrete Implementation)

- **Header**: [LyraGameSettingRegistry.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Settings/LyraGameSettingRegistry.h)
- **Implementation**: [LyraGameSettingRegistry.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Settings/LyraGameSettingRegistry.cpp)
- **Additional Files**: `LyraGameSettingRegistry_Audio.cpp`, `_Gamepad.cpp`, `_Gameplay.cpp`, `_MouseAndKeyboard.cpp`, `_PerfStats.cpp`, `_Video.cpp`
- **Inheritance**: `UObject` → `UGameSettingRegistry` → `ULyraGameSettingRegistry`

## OnInitialize Flow

```
ULyraGameSettingRegistry::OnInitialize(LocalPlayer)
  │
  ├── VideoSettings = InitializeVideoSettings(LyraLocalPlayer)
  │     ├── InitializeVideoSettings_FrameRates(VideoSettings, LyraLocalPlayer)
  │     └── RegisterSetting(VideoSettings)
  │
  ├── AudioSettings = InitializeAudioSettings(LyraLocalPlayer)
  │     └── RegisterSetting(AudioSettings)
  │
  ├── GameplaySettings = InitializeGameplaySettings(LyraLocalPlayer)
  │     └── RegisterSetting(GameplaySettings)
  │
  ├── MouseAndKeyboardSettings = InitializeMouseAndKeyboardSettings(LyraLocalPlayer)
  │     └── RegisterSetting(MouseAndKeyboardSettings)
  │
  └── GamepadSettings = InitializeGamepadSettings(LyraLocalPlayer)
        └── RegisterSetting(GamepadSettings)
```

Each `Initialize*Settings` function creates and returns a `UGameSettingCollection` containing the hierarchical structure of settings.

## Registration Flow

```
UGameSettingRegistry::RegisterSetting(InSetting)
  ├── TopLevelSettings.Add(InSetting)           // Add to top-level list
  ├── InSetting->SetRegistry(this)              // Assign owning registry
  └── RegisterInnerSettings(InSetting)          // Recursive registration
        ├── RegisteredSettings.Add(InSetting)   // Add to flat list
        ├── Bind events:
        │     ├── OnSettingChanged → HandleSettingChanged
        │     ├── OnSettingApplied → HandleSettingApplied
        │     ├── OnSettingEditConditionChanged → HandleSettingEditConditionsChanged
        │     ├── OnSettingNamedAction → HandleSettingNamedAction
        │     └── OnExecuteNavigation → HandleSettingNavigation
        └── If UGameSettingCollection:
              └── Recursively call RegisterInnerSettings for each child Setting
```

## Data Source Macros

Used to bind setting widgets to properties of `ULyraSettingsLocal` or `ULyraSettingsShared`:

```cpp
// Local settings data source: ULyraLocalPlayer → GetLocalSettings() → Property
#define GET_LOCAL_SETTINGS_FUNCTION_PATH(FunctionOrPropertyName) \
    MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({ \
        GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetLocalSettings), \
        GET_FUNCTION_NAME_STRING_CHECKED(ULyraSettingsLocal, FunctionOrPropertyName) \
    }))

// Shared settings data source: ULyraLocalPlayer → GetSharedSettings() → Property
#define GET_SHARED_SETTINGS_FUNCTION_PATH(FunctionOrPropertyName) \
    MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({ \
        GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetSharedSettings), \
        GET_FUNCTION_NAME_STRING_CHECKED(ULyraSettingsShared, FunctionOrPropertyName) \
    }))
```

### GET_FUNCTION_NAME_STRING_CHECKED

- **Header**: `LyraGameSettingRegistry.h`
- **Implementation**: `((void)sizeof(&ClassName::FunctionName), TEXT(#FunctionName))`

Uses `sizeof(&ClassName::FunctionName)` to detect at compile time whether the function exists. If the function does not exist, the compiler reports an error. The comma operator ensures the value is the `TEXT(#FunctionName)` string on the right. Zero runtime overhead.

### FGameSettingDataSourceDynamic

- **Header**: [GameSettingDataSourceDynamic.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameSettings/Source/Public/DataSource/GameSettingDataSourceDynamic.h)

Accepts an array of property path strings, internally using `FCachedPropertyPath` for property resolution. Supports the following property types:
- `FObjectProperty` / `UObjectProperty`: Object/weak object references
- `FStructProperty`: Structs
- `FArrayProperty`: Arrays

## GC Lifecycle

```
// Find existing registry from LocalPlayer
ULyraGameSettingRegistry* Registry = FindObject<ULyraGameSettingRegistry>(
    InLocalPlayer, TEXT("LyraGameSettingRegistry"), true);

// Create if not found
if (Registry == nullptr)
{
    // Outer = InLocalPlayer — but Outer cannot prevent GC
    Registry = NewObject<ULyraGameSettingRegistry>(InLocalPlayer,
        TEXT("LyraGameSettingRegistry"));
    Registry->Initialize(InLocalPlayer);
}

// Actually held by ULyraSettingScreen::CreateRegistry()
UGameSettingRegistry* ULyraSettingScreen::CreateRegistry()
{
    // Note: Outer is nullptr, held by member variable Registry
    ULyraGameSettingRegistry* NewRegistry = NewObject<ULyraGameSettingRegistry>();
    NewRegistry->Initialize(LocalPlayer);
    return NewRegistry;
}
```

Outer cannot prevent GC. The registry must be held by a `UPROPERTY()` member (such as `ULyraSettingScreen::Registry`) to avoid being collected.

## IsFinishedInitializing

`ULyraGameSettingRegistry::IsFinishedInitializing()` additionally waits for shared settings (`ULyraSettingsShared`) async loading to complete, beyond the base class check:

```cpp
bool ULyraGameSettingRegistry::IsFinishedInitializing() const
{
    if (Super::IsFinishedInitializing())
    {
        if (ULyraLocalPlayer* LocalPlayer = Cast<ULyraLocalPlayer>(OwningLocalPlayer))
        {
            if (LocalPlayer->GetSharedSettings() == nullptr)
                return false;
        }
        return true;
    }
    return false;
}
```

## Data Flow

```
User UI operation → Setting value change
  → OnSettingChanged(Setting, ChangeReason)
    → MarkDirty + Registry records change
      → SaveChanges()
        → Call each Setting's Apply()
          → ULyraSettingsLocal::ApplySettings()
          → ULyraSettingsShared::SaveSettings()
            → Config persistence / Save persistence
```
