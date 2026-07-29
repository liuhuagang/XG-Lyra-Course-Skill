# AssetManager and Async Loading

## ULyraAssetManager

[ULyraAssetManager](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraAssetManager.h) inherits from `UAssetManager` and is Lyra's global asset manager. It is responsible for:

1. Async asset loading (via `FSoftObjectPath`)
2. Asset Bundle management
3. Asset pre-warming at initialization

## Async Asset Loading

### TSoftObjectPtr and FSoftObjectPath

Lyra extensively uses `TSoftObjectPtr` and `FSoftObjectPath` for async loading:

- **`TSoftObjectPtr<UObject>`** — Type-safe soft pointer, used for asset references
- **`FSoftObjectPath`** — Underlying path string representation, no type constraints
- **`FPrimaryAssetId`** — Primary asset ID, composed of `PrimaryAssetType:PrimaryAssetName`

Differences between the two:

| Feature | TSoftObjectPtr\<T\> | FSoftObjectPath |
|------|---------------------|----------------|
| Type Safety | Yes | No |
| Runtime Sync/Async Loading | Yes | Yes |
| Memory Usage | Larger | Smaller |
| Serialization | Stable | Stable |

Use cases:
- When certain resources are not needed during initialization, use `TSoftObjectPtr` instead of hard pointers
- If only storing a path string, `FSoftObjectPath` is more lightweight

### Core Loading Methods

```cpp
// Synchronous loading (should generally be avoided)
TSoftObjectPtr<UObject> MyObject;
UObject* Loaded = MyObject.LoadSynchronous();

// Async loading (recommended)
TSoftObjectPtr<UObject> MyObject;
TStreamableHandlePtr Handle = UAssetManager::GetStreamableManager().RequestAsyncLoad(
    MyObject.ToSoftObjectPath(),
    FStreamableDelegate::CreateUObject(this, &ThisClass::OnLoadComplete));
```

### Usage in Experience

`ULyraExperienceDefinition` uses `TSoftClassPtr` to reference GA/GE/AttributeSet:

```cpp
UPROPERTY(EditDefaultsOnly, Category = Gameplay)
TArray<TSoftClassPtr<ULyraGameplayAbility>> GameplayAbilities;

UPROPERTY(EditDefaultsOnly, Category = Gameplay)
TArray<TSoftClassPtr<ULyraAttributeSet>> AttributeSets;

UPROPERTY(EditDefaultsOnly, Category = Gameplay)
TArray<TSoftClassPtr<UGameplayEffect>> GameplayEffects;
```

## Asset Bundle

Asset Bundle is UE's asset grouping mechanism, packaging multiple assets into a Bundle for on-demand loading.

In Lyra, Asset Bundle works with the Experience system:
1. Experience definition declares required assets (GA/GE/AttributeSet)
2. AssetManager categorizes these assets into Bundles
3. When Experience loads, it notifies AssetManager to request the Bundle
4. AssetManager asynchronously loads all assets in the Bundle
5. After loading completes, Experience notifies GameMode to allow player login

## Performance Optimization Macros

Macros used in Lyra:

### UE_INLINE_GENERATED_CPP_BY_NAME

Inlines the `_gen.cpp` content into the header file, reducing indirect calls. Note:
- Can only be used once per class
- Remove the corresponding `#include` statement after use (since the header already contains generated code)

### UE_DISABLE_OPTIMIZATION / UE_ENABLE_OPTIMIZATION

Used to disable/enable compiler optimization for specific code:

```cpp
UE_DISABLE_OPTIMIZATION
// Code that does not need optimization
UE_ENABLE_OPTIMIZATION
```

### UE_AUTORTTR_END_SCOPE

Used to automatically end RTT tracing markers within a scope:
- Typically search for `TRACE_CPUPROFILER_EVENT_SCOPE` markers in hot functions
- Searching for the `RTT` abbreviation helps quickly locate

### Reference Marker Search Tips

Search for the following markers in VS for quick navigation:

| Marker | RTT Abbrev | Description |
|------|---------|------|
| `RTT` | R | Hot function marker |
| `TRACETAG` | T | CPU trace tag |
| `TRACE_CPUPROFILER_EVENT_SCOPE` | T | Scope CPU tracing |

## Common Performance Pitfalls

Directly using `StaticLoadObject` / `StaticConstructObject` / `LoadObject` causes synchronous loading that blocks the game thread. The correct approach is to use `UAssetManager::GetStreamableManager().RequestAsyncLoad` for async loading.
