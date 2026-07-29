# GameState and PlayerState

## ALyraGameState

[ALyraGameState](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/GameModes/LyraGameState.h) inherits from `AModularGameStateBase` and also implements `IAbilitySystemInterface`.

### Responsibilities

1. **Holds ExperienceManagerComponent** — GameState hosts `ULyraExperienceManagerComponent` to manage Experience loading
2. **Network Synchronization** — Game state shared across all clients (e.g., team scores, game phase)
3. **AbilitySystem Interface** — `IAbilitySystemInterface` provides global `UAbilitySystemComponent` access

### ModularGameplay Support

`AModularGameStateBase` is a base class provided by the `ModularGameplayActors` plugin, supporting GameFeature plugins to dynamically add components to GameState without modifying the base class code.

```cpp
UCLASS()
class ALyraGameState : public AModularGameStateBase, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    // IAbilitySystemInterface
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

    // Experience manager component
    UPROPERTY()
    TObjectPtr<ULyraExperienceManagerComponent> ExperienceManagerComponent;

    // Player spawning manager
    UPROPERTY()
    TObjectPtr<ULyraPlayerSpawningManagerComponent> PlayerSpawningManagerComponent;
};
```

## ALyraPlayerState

[ALyraPlayerState](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Player/LyraPlayerState.h) inherits from `AModularPlayerState` and implements both `IAbilitySystemInterface` and `ILyraTeamAgentInterface`.

### Responsibilities

1. **Holds individual ASC** — Each player has their own `UAbilitySystemComponent`
2. **TagStack holder** — Uses `FGameplayTagStack` to manage player Tag counts
3. **Team interface** — Implements `ILyraTeamAgentInterface`, identifies the player's team

### Core Properties

```cpp
UCLASS()
class ALyraPlayerState : public AModularPlayerState, 
    public IAbilitySystemInterface,
    public ILyraTeamAgentInterface
{
    // Player's ASC
    UPROPERTY()
    TObjectPtr<ULyraAbilitySystemComponent> AbilitySystemComponent;

    // TagStack container
    UPROPERTY(Replicated)
    FGameplayTagStackContainer TagStacks;

    // Team ID
    UPROPERTY(ReplicatedUsing = OnRep_TeamId)
    ELyraTeamType TeamType;

    // Team display assets (color, materials, etc.)
    UPROPERTY(ReplicatedUsing = OnRep_TeamDisplayAsset)
    TObjectPtr<ULyraTeamDisplayAsset> TeamDisplayAsset;
};
```

## FGameplayTagStack Container

[FGameplayTagStack](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/GameplayTagStack.h) is a Tag counting mechanism based on `FFastArraySerializer`.

### FastArray Principle

Uses `FFastArraySerializer` for efficient network synchronization, transmitting only changed elements rather than the entire array.

```cpp
USTRUCT()
struct FGameplayTagStack : public FFastArraySerializerItem
{
    GENERATED_BODY()

    UPROPERTY()
    FGameplayTag Tag;

    UPROPERTY()
    int32 StackCount = 0;
};

USTRUCT()
struct FGameplayTagStackContainer : public FFastArraySerializer
{
    GENERATED_BODY()

    // Increase the count for a specified Tag
    void AddStack(FGameplayTag Tag, int32 Count);

    // Get the current count for a specified Tag
    int32 GetStackCount(FGameplayTag Tag) const;

    // Check if a specified Tag exists
    bool ContainsTag(FGameplayTag Tag) const;

    UPROPERTY()
    TArray<FGameplayTagStack> Stacks;
};
```

### Use Cases

- **Resource counting** — Ammo count, currency, etc.
- **Status markers** — Whether a certain state is active
- **Permission checking** — Whether a specific Tag is held

### FastArray Key Functions

Key functions in `FFastArraySerializer`:

| Function | Description |
|------|------|
| `MarkItemDirty(Item)` | Mark an item as dirty, triggering incremental sync |
| `MarkArrayDirty()` | Force full synchronization |
| `SetItemDeltaReplicated(Item)` | Incremental update |
| `PreReplicatedRemove` | Callback before item removal |
| `PostReplicatedAdd` | Callback after item addition |
| `PostReplicatedChange` | Callback after item modification |

## Common G-prefix Global Variables

Common G-prefix global variables in UE engine:

| Global Variable | Type | Description |
|---------|------|------|
| `GEngine` | `UEngine*` | Engine instance |
| `GWorld` | `UWorld*` | Current active world |
| `GConfig` | `FConfigCacheIni*` | Configuration system |
| `GEditor` | `UEditorEngine*` | Editor engine instance |
| `GEditorToDiscard` | `bool` | Editor restart marker |
| `GFileManager` | `IFileManager*` | File manager |
| `GLog` | `FOutputDevice*` | Log system |
| `GIsEditor` | `bool` | Whether in editor mode |
| `GIsGameThread` | `bool` | Whether currently on game thread |
| `GIsServer` | `bool` | Whether currently a server |
| `GIsClient` | `bool` | Whether currently a client |
| `GScreen` | `FViewport*` | Screen viewport |
| `GInputKeyframeInformation` | Collection | Input keyframe information |
| `GNearClippingPlane` | `float` | Near clipping plane distance |

### GIsServer vs GIsClient

| Environment | GIsServer | GIsClient |
|------|-----------|-----------|
| Standalone Game | true | true |
| Dedicated Server (DS) | true | false |
| Client | false | true |
| Editor PIE Server | true | true |
| Editor PIE Client | false | true |
