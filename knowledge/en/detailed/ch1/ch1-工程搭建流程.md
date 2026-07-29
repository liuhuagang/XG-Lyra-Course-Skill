# Project Setup Process

## Core Steps

1. Create a UE5.6 blank C++ project
2. Split the single module into LyraGame (runtime) + LyraEditor (editor) dual modules
3. Configure `.uproject` file to declare modules and plugins
4. Configure `.Target.cs` files to declare build targets
5. Configure `.Build.cs` files to declare module dependencies
6. Copy Lyra sample project's Content assets
7. Complete project settings (collision configuration, audio settings, etc.)

## Module Separation

Split from a single module into runtime + editor modules:

| Aspect | LyraGame (Runtime) | LyraEditor (Editor) |
|------|-------------------|-------------------|
| LoadingPhase | Default | Default |
| Type | Runtime | Editor |
| Dependencies | Runtime modules | LyraGame + Editor modules |

## `.uproject` Configuration

Core structure of `Lyra.uproject`:

```json
{
  "FileVersion": 3,
  "EngineAssociation": "5.6",
  "Category": "",
  "Description": "",
  "Modules": [
    {
      "Name": "LyraGame",
      "Type": "Runtime",
      "LoadingPhase": "Default"
    },
    {
      "Name": "LyraEditor",
      "Type": "Editor",
      "LoadingPhase": "Default"
    }
  ],
  "Plugins": [
    { "Name": "CommonGame", "Enabled": true },
    { "Name": "CommonUser", "Enabled": true },
    { "Name": "CommonLoadingScreen", "Enabled": true },
    { "Name": "GameSettings", "Enabled": true },
    { "Name": "GameplayMessageRouter", "Enabled": true },
    { "Name": "ModularGameplayActors", "Enabled": true },
    { "Name": "UIExtension", "Enabled": true },
    { "Name": "PocketWorlds", "Enabled": true },
    { "Name": "AsyncMixin", "Enabled": true },
    { "Name": "ControlFlows", "Enabled": true },
    { "Name": "ShooterCore", "Enabled": true },
    { "Name": "ShooterMaps", "Enabled": true },
    { "Name": "TopDownArena", "Enabled": true },
    { "Name": "ShooterExplorer", "Enabled": true },
    { "Name": "ShooterTests", "Enabled": true },
    { "Name": "OpenImageDenoise", "Enabled": false }
  ]
}
```

## LyraGame.Build.cs Core Dependencies

```csharp
PublicDependencyModuleNames.AddRange(new string[] {
    "Core", "CoreUObject", "CoreOnline", "ApplicationCore",
    "Engine", "PhysicsCore",
    "GameplayTags", "GameplayTasks", "GameplayAbilities",
    "AIModule",
    "ModularGameplay", "ModularGameplayActors",
    "DataRegistry", "ReplicationGraph", "GameFeatures",
    "Niagara",
    "CommonLoadingScreen", "AsyncMixin", "ControlFlows"
});

PrivateDependencyModuleNames.AddRange(new string[] {
    "InputCore", "Slate", "SlateCore",
    "EnhancedInput",
    "UMG", "CommonUI", "CommonInput",
    "CommonGame", "CommonUser",
    "GameplayMessageRuntime",
    "AudioMixer", "NetworkReplayStreaming",
    "UIExtension", "AudioModulation",
    "DTLSHandlerComponent", "Json"
});
```

## LyraEditor.Build.cs Core Dependencies

```csharp
PublicDependencyModuleNames.AddRange(new string[] {
    "Core", "CoreUObject", "Engine", "EditorFramework", "UnrealEd",
    "GameplayTagsEditor", "GameplayTasksEditor",
    "GameplayAbilities", "GameplayAbilitiesEditor",
    "StudioTelemetry",
    "LyraGame"
});

PrivateDependencyModuleNames.AddRange(new string[] {
    "InputCore", "Slate", "SlateCore", "ToolMenus",
    "EditorStyle", "DataValidation", "MessageLog",
    "Projects", "DeveloperToolSettings",
    "CollectionManager", "SourceControl", "Chaos"
});
```

## Build Options

- `WITH_RPC_REGISTRY` — For RPC registry
- `SHIPPING_DRAW_DEBUG_ERROR` — Treat DrawDebug as error in Shipping configuration
- `SetupGameplayDebuggerSupport()` — Enable GameplayDebugger support
- `SetupIrisSupport()` — Enable Iris network replication framework
