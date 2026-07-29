# Engine Class Hierarchy

## Engine Class Inheritance

```
UEngine (Base)
├─ UGameEngine (Game Runtime Engine)
└─ UEditorEngine (Generic Editor Engine Base)
   └─ UUnrealEdEngine (Unreal Dedicated Editor Engine)
```

Engine class location: `Engine/Source/Runtime/Engine/Classes/Engine/Engine.h`

## Engine Class Differences

| Feature | UGameEngine | UEditorEngine (Base) | UUnrealEdEngine (Subclass) |
|------|-------------|---------------------|----------------------|
| Purpose | Pure game runtime | Editor base framework | Complete Unreal Editor implementation |
| Module Dependencies | Core game modules only | Basic editor tools | Full editor toolchain |
| Runtime Scenario | Packaged game | Custom editor tool development | Unreal Editor |
| PIE Support | No | Yes (via subclass) | Yes (full PIE/SIE support) |

## Lyra's Engine Class Configuration

Lyra replaces 7 core engine classes in the `[/Script/Engine.Engine]` section of `DefaultEngine.ini`:

```ini
[/Script/Engine.Engine]
GameEngine=/Script/LyraGame.LyraGameEngine
UnrealEdEngine=/Script/LyraEditor.LyraEditorEngine
EditorEngine=/Script/LyraEditor.LyraEditorEngine
GameViewportClientClassName=/Script/LyraGame.LyraGameViewportClient
AssetManagerClassName=/Script/LyraGame.LyraAssetManager
WorldSettingsClassName=/Script/LyraGame.LyraWorldSettings
LocalPlayerClassName=/Script/LyraGame.LyraLocalPlayer
GameUserSettingsClassName=/Script/LyraGame.LyraSettingsLocal
```

## UEngine Configurable Classes

UEngine exposes replaceable classes via `FSoftClassPath` properties:

| Property | Type | Description |
|------|------|------|
| `GameViewportClientClassName` | `FSoftClassPath` | Game viewport client class |
| `AssetManagerClassName` | `FSoftClassPath` | Global AssetManager class |
| `WorldSettingsClassName` | `FSoftClassPath` | WorldSettings class |
| `LocalPlayerClassName` | `FSoftClassPath` | Local player class |
| `GameUserSettingsClassName` | `FSoftClassPath` | Game user settings class |

## Engine Loading Timing

Engine initialization occurs very early (before `main()`), with core code located in `LaunchEngineLoop.cpp` (approx. 4530 lines).

Key flow:

```cpp
if (!GIsEditor) {
    // Game mode: load GameEngine
    GConfig->GetString(TEXT("/Script/Engine.Engine"), TEXT("GameEngine"), 
        GameEngineClassName, GEngineIni);
    EngineClass = StaticLoadClass(UGameEngine::StaticClass(), nullptr, *GameEngineClassName);
    GEngine = NewObject<UEngine>(GetTransientPackage(), EngineClass);
} else {
    // Editor mode: load UnrealEdEngine
    GConfig->GetString(TEXT("/Script/Engine.Engine"), TEXT("UnrealEdEngine"), 
        UnrealEdEngineClassName, GEngineIni);
    EngineClass = StaticLoadClass(UUnrealEdEngine::StaticClass(), nullptr, *UnrealEdEngineClassName);
    GEngine = GEditor = GUnrealEd = NewObject<UUnrealEdEngine>(GetTransientPackage(), EngineClass);
}
```

### Configuration Source Priority

| Config File | Description |
|---------|------|
| `BaseEngine.ini` | Engine default configuration (read-only) |
| `DefaultEngine.ini` | Project-level configuration (overrides base) |
| Platform-specific INI | Configuration under Windows/Android directories |
| `Saved/Config/...` | User runtime override configuration |

## Verifying Current Engine Class

Method 1 — Console command:
```
obj list class=Engine
```

Method 2 — Code debugging:
Set breakpoints in `UGameEngine::Init()` or `UUnrealEdEngine::Init()`.

## Lyra Custom Engine Interfaces

| Class | File | Inherits From |
|------|---------|--------|
| [ULyraGameEngine](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraGameEngine.h) | `LyraGame/System/LyraGameEngine.h` | `UGameEngine` |
| [ULyraEditorEngine](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/LyraEditorEngine.h) | `LyraEditor/LyraEditorEngine.h` | `UUnrealEdEngine` |
| [ULyraGameViewportClient](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/UI/LyraGameViewportClient.h) | `LyraGame/UI/LyraGameViewportClient.h` | `UCommonGameViewportClient` |
| [ULyraAssetManager](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraAssetManager.h) | `LyraGame/System/LyraAssetManager.h` | `UAssetManager` |
| [ALyraWorldSettings](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraWorldSettings.h) | `LyraGame/GameModes/LyraWorldSettings.h` | `AWorldSettings` |
| [ULyraLocalPlayer](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Player/LyraLocalPlayer.h) | `LyraGame/Player/LyraLocalPlayer.h` | `UCommonLocalPlayer` |
| [ULyraSettingsLocal](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Settings/LyraSettingsLocal.h) | `LyraGame/Settings/LyraSettingsLocal.h` | `UGameUserSettings` |
