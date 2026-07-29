# Init State Machine & PawnExtension

## Overview

`ULyraPawnExtensionComponent` is the central component for character initialization. It implements a 4-state initialization chain based on `IGameFrameworkInitStateInterface`, coordinating the initialization timing of various subsystems.

## Base Class Hierarchy

```
UGameFrameworkComponent
  └── UPawnComponent
        ├── GetPawn()
        ├── GetPlayerState()
        ├── GetController()
        └── ...
              └── ULyraPawnExtensionComponent : IGameFrameworkInitStateInterface
```

- `UPawnComponent` — Provides Pawn/PlayerState/Controller access
- `UGameFrameworkComponent` — Provides GameInstance/HasAuthority/Timer access
- `IGameFrameworkInitStateInterface` — Init state machine interface

## Init State Machine (IGameFrameworkInitStateInterface)

### State Chain

```
Spawned → DataAvailable → DataInitialized → GameplayReady
```

| State | Meaning | Entry Condition |
|------|------|---------|
| Spawned | Spawned, component available | Valid Pawn |
| DataAvailable | PawnData loaded | Requires PawnData set + Server/local needs Controller |
| DataInitialized | All components reached DataAvailable | Manager->HaveAllFeaturesReachedInitState(DataAvailable) |
| GameplayReady | Game logic ready to run | Always true |

### Core Interface Methods

| Method | Responsibility |
|------|------|
| `GetFeatureName()` | Returns `"PawnExtension"` (`NAME_ActorFeatureName`) |
| `RegisterInitStateFeature()` | Registers self with ComponentManager |
| `UnregisterInitStateFeature()` | Unregisters from ComponentManager |
| `CanChangeInitState()` | Checks if state transition is reachable |
| `HandleChangeInitState()` | Executes logic during state transition (currently no-op for DataInitialized) |
| `CheckDefaultInitialization()` | Attempts to advance along the state chain |
| `ContinueInitStateChain()` | Iterates through the state chain step by step |
| `BindOnActorInitStateChanged()` | Listens to state changes of other features |

### Triggering Moments

PawnExtension state advancement is triggered by the following events:

1. **BeginPlay** — Binds listeners to all features, attempts to advance to Spawned
2. **Other component state changes** — `OnActorInitStateChanged` retries when other features reach DataAvailable
3. **PawnData set** — Server `SetPawnData` triggers ForceNetUpdate and calls CheckDefaultInitialization
4. **ControllerChanged** — Advances when Controller becomes available
5. **PlayerStateReplicated** — Client advances after receiving PlayerState
6. **SetupPlayerInputComponent** — Advances when Input becomes available

### CanChangeInitState Logic

```
None → Spawned:
   Validate Pawn is valid

Spawned → DataAvailable:
   Requires PawnData != nullptr
   Server or locally controlled: Requires Controller != nullptr

DataAvailable → DataInitialized:
   Manager->HaveAllFeaturesReachedInitState(Pawn, DataAvailable)
   // Waits for all features (HeroComponent, CameraComponent, etc.) to reach DataAvailable

DataInitialized → GameplayReady:
   Always returns true
```

## PawnData Management

- `SetPawnData(InPawnData)`: Server-only, can only be set once, triggers `ForceNetUpdate` and calls `CheckDefaultInitialization()`
- `OnRep_PawnData`: Client calls `CheckDefaultInitialization()` after receiving the replicated data
- PawnData contains: `AbilitySets` (GA/GES/AttributeSet list), `TagRelationshipMapping` (Tag relationship data asset), `CameraMode` and other configurations

## ASC Management

### InitializeAbilitySystem

```cpp
void ULyraPawnExtensionComponent::InitializeAbilitySystem(
    ULyraAbilitySystemComponent* InASC, AActor* InOwnerActor)
```

1. **Avatar Conflict Handling**: If the current ASC already has a different avatar, evicts the old one via `OtherExtensionComponent->UninitializeAbilitySystem()`
2. `InASC->InitAbilityActorInfo(InOwnerActor, Pawn)` — Sets OwnerActor and AvatarActor
3. `InASC->SetTagRelationshipMapping(PawnData->TagRelationshipMapping)` — Sets Tag Relationship Mapping
4. `OnAbilitySystemInitialized.Broadcast()` — Broadcasts initialization complete event

### UninitializeAbilitySystem

- Clears ASC bindings
- Broadcasts `OnAbilitySystemUninitialized`

### RegisterAndCall Pattern

```cpp
void OnAbilitySystemInitialized_RegisterAndCall(FOnAbilitySystemInitialized::FDelegate Delegate);
void OnAbilitySystemUninitialized_Register(FOnAbilitySystemUninitialized::FDelegate Delegate);
```

- `_RegisterAndCall` — Registers delegate, calls immediately if already initialized
- `_Register` — Registers only, does not call immediately

## Health Component Binding

In `ALyraCharacter` constructor, the following binding order is used:

1. `PawnExtComponent->OnAbilitySystemInitialized_RegisterAndCall(...)`
   - Callback calls `HealthComponent->InitializeWithAbilitySystem(ASC)`
   - `InitializeGameplayTags()` sets initial GameplayTags
2. `PawnExtComponent->OnAbilitySystemUninitialized_Register(...)`
   - Callback calls `HealthComponent->UninitializeFromAbilitySystem()`

## Overall Flow

```
Actor Spawned
  → OnRegister: RegisterInitStateFeature
  → BeginPlay: BindOnActorInitStateChanged, TryToChangeInitState(Spawned), CheckDefaultInitialization
  → PawnData Set: CheckDefaultInitialization
  → Controller Attached: CheckDefaultInitialization
  → All features reach DataAvailable: CheckDefaultInitialization
  → GameplayReady: System ready
```

## Code References

| Class/File | Path |
|---------|------|
| `ULyraPawnExtensionComponent` | [LyraPawnExtensionComponent.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Character/LyraPawnExtensionComponent.h) |
| `ULyraPawnData` | [LyraPawnData.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Character/LyraPawnData.h) |
| `ULyraAbilityTagRelationshipMapping` | [LyraPawnExtensionComponent.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Character/LyraPawnExtensionComponent.h) |
| `IGameFrameworkInitStateInterface` | GameFramework plugin |
