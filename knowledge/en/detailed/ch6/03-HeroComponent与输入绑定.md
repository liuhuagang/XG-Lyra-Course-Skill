# HeroComponent & Input Binding

## Overview

`ULyraHeroComponent` is the character's "Hero" component, responsible for input binding, ASC initialization, and camera mode proxying. It also participates in the init state machine, performing final input binding setup in the GameplayReady state.

## Class Declaration

```cpp
class ULyraHeroComponent : public UPawnComponent, public IGameFrameworkInitStateInterface
```

## Init State Machine Participation

- Feature Name: `"Hero"` (different from PawnExtension's `"PawnExtension"`)
- `RegisterInitStateFeature()` registers itself
- Listens to PawnExtension state changes
- In `DataAvailable → DataInitialized` phase:
  - Initializes ASC (calls PawnExtension->InitializeAbilitySystem)
  - Binds AbilityActivated event
- In `DataInitialized → GameplayReady` phase:
  - OnSetupInputComponents binds input

## ASC Initialization

- Creates or obtains ASC via `ULyraAbilitySystemComponent*`
- ASC's OwnerActor is PlayerState, AvatarActor is Pawn
- Binds `AbilityActivated` event to check if the activated Ability should be handled by the camera
- If it is a camera Ability, sends event to the camera component

## Input Binding (SetupPlayerInputComponent)

`OnSetupInputComponents` is the input binding entry point, triggered when the init state machine reaches GameplayReady:

1. Ensures valid PlayerController exists
2. Creates `ULyraInputComponent`
3. Gets `InputConfig` from PawnData
4. Calls `InputConfig->BindAbilityActivation(...)` to bind input actions to GA
5. Binds native input actions

## Camera Mode Proxy

HeroComponent acts as a proxy layer for camera mode switching:

- Handles camera-responsive Abilities (e.g., aiming)
- Forwards Ability activation/cancel events to `ULyraCameraComponent`
- Calls `CameraComponent->ChangeCameraMode(...)` to switch camera mode

## Relationships with Other Components

```
ULyraHeroComponent
  ├── Depends on PawnExtensionComponent's init state
  ├── Creates and initializes ULyraAbilitySystemComponent
  ├── Controls ULyraInputComponent binding
  ├── Proxies camera mode switching to ULyraCameraComponent
  └── Listens to GameplayAbility activation/cancel events
```

## Code References

| Class | File |
|----|------|
| `ULyraHeroComponent` | [LyraHeroComponent.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Character/LyraHeroComponent.h) |
