# Target File Configuration

## Target File Types

UE projects support four Target files, corresponding to different build targets:

| Target File | Build Target | Purpose |
|-------------|---------|------|
| `LyraGame.Target.cs` | Game | Game main executable |
| `LyraEditor.Target.cs` | Editor | Editor mode |
| `LyraClient.Target.cs` | Client | Client-only (no server logic) |
| `LyraServer.Target.cs` | Server | Dedicated server |

## Shared Configuration Method

Lyra uses the `ApplySharedLyraTargetSettings()` method to centrally manage shared configuration across Targets. All Target files call this method to reduce duplication.

Shared configuration includes:

### Build Configuration

```csharp
// Shadow variable warning level
ShadowVariableWarningLevel = WarningLevel.Error;

// Non-Shipping crash reporting
bUseLoggingInShipping = true;

// Generate INI files only in development configuration
bAllowGeneratedIniWhenCooked = true;

// Allow unverified certificates only in non-Shipping configurations
bDisableUnverifiedCertificates = !bHasShippingConfig;

// AssetRegistry pointer optimization
UE_ASSETREGISTRY_INDIRECT_ASSETDATA_POINTERS = 1;
```

### Plugin Management

Use the `ConfigureGameFeaturePlugins()` method to declare GameFeature plugin relationships with build targets:

```csharp
// Disable specific plugins on Server target
if (Target.Type == TargetRules.TargetType.Server)
{
    DisablePlugin(Target, "SomeClientPlugin");
}
```

## LyraGame.Target.cs Core Structure

```csharp
public class LyraGameTarget : TargetRules
{
    public LyraGameTarget(TargetInfo Target) : base(Target)
    {
        Type = TargetType.Game;
        DefaultBuildSettings = BuildSettingsVersion.V4;
        IncludeOrderVersion = EngineIncludeOrderVersion.Unreal5_4;
        ExtraModuleNames.AddRange(new string[] { "LyraGame", "LyraEditor" });
        
        ApplySharedLyraTargetSettings(this);
        ConfigureGameFeaturePlugins(this);
    }
}
```

## Target File vs Build File

| | `.Target.cs` | `.Build.cs` |
|--|-------------|-------------|
| Scope | Build target level | Module level |
| Configuration Content | Target type, editor/server mode, platform settings | Module dependencies, header paths, libraries |
| Typical Configuration | `Type = TargetType.Game` | `PublicDependencyModuleNames` |
| When to Modify | New target platform or new module | New dependencies added |

## Notes

- The Editor Target (LyraEditor) does not need `ApplySharedLyraTargetSettings` because the editor has its own independent configuration path
- The Server Target (LyraServer) should disable unnecessary client plugins to reduce build size
